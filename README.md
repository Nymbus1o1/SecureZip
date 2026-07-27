# SecureZip

**Compress and encrypt files entirely on your own device — no server, no upload, no accounts.**

SecureZip is a single self-contained HTML file that runs a full-featured file encryption and compression tool directly in the browser, using the native **Web Crypto API** and **Compression Streams API**. There is no backend, no build step, and no third-party network calls — everything (files, passwords, keyfiles) stays in the browser tab and is never transmitted anywhere.

> **Current build:** `securezip-11.html` — see [Versioning](#versioning) below.

---

## Table of contents

- [Overview](#overview)
- [Features](#features)
- [How it works](#how-it-works)
- [Security notes](#security-notes)
- [Getting started](#getting-started)
- [Deployment](#deployment)
  - [GitHub Pages](#option-1-github-pages-recommended)
  - [Netlify](#option-2-netlify)
  - [Vercel](#option-3-vercel)
  - [Cloudflare Pages](#option-4-cloudflare-pages)
  - [Any static host / self-hosted](#option-5-any-static-host--self-hosted)
  - [Running locally](#running-locally)
- [Repository structure](#repository-structure)
- [Browser support](#browser-support)
- [Versioning](#versioning)
- [Known limitations](#known-limitations)
- [Contributing](#contributing)
- [License](#license)
- [Disclaimer](#disclaimer)

---

## Overview

SecureZip lets you compress, encrypt, and decrypt files without installing anything or trusting a remote server with your data. Open the page (locally or hosted), drop in a file, choose Compress / Encrypt / Both, set a password, and download the result. Decryption works the same way in reverse.

It's built as one static HTML document — the CSS and JavaScript are inline, there are no external scripts, fonts, or CDN calls, and it works completely offline once loaded.

## Features

**Core operations**
- Compress-only, encrypt-only, or both in one pass
- **AES-256-GCM** authenticated encryption
- **PBKDF2-SHA-256** key derivation with an adjustable iteration count (100,000 – 1,000,000; default **250,000**)
- Gzip compression/decompression via the native `CompressionStream` / `DecompressionStream` APIs
- Reads plain `.zip` and `.gz` files too, not just its own format — useful for inspecting or decompressing archives created elsewhere

**Sharing & multi-party access**
- **Multi-recipient encryption** — encrypt once, unlock with any of several independent passwords/keyfiles (each recipient gets their own password without ever seeing anyone else's)
- **Decoy / deniable content** — a second password reveals separate, innocuous content instead of the real file (clearly labeled in the UI as basic misdirection, not true plausible deniability — archive size can still hint that more than one payload exists)
- **Keyfiles** — an optional second factor; both the password *and* the exact keyfile are required to decrypt
- **Split output into parts** (~10/25/50/100 MB) for email limits or unstable connections — drop all the parts back into the Decrypt panel and they reassemble automatically
- **Copy as text** — base64-encodes the encrypted output so it can be pasted anywhere file attachments aren't allowed

**Input options**
- Drag-and-drop, multi-file picker, folder picker, camera capture (mobile), clipboard paste, or raw text input
- Batch processing of multiple files/jobs in a single run, with per-file or "download all as .zip" output

**Trust & safety tools**
- **Verify only** mode — confirm a password/keyfile is correct without writing or downloading anything
- SHA-256 fingerprint shown on results for integrity verification
- Password strength meter and cryptographically-random password generator
- Common-password detection
- **Auto-clear** of passwords, keyfiles, and decrypted results after 5 minutes of inactivity, or shortly after the browser tab is hidden
- Light / dark theme toggle

**Zero-dependency by design**
- One HTML file — no `npm install`, no bundler, no framework, no external requests
- Works fully offline once the page has loaded

## How it works

SecureZip defines its own lightweight container format, identified by the magic bytes `SXZ1` and a version byte, so the tool can always tell what kind of file it's looking at and stay backward-compatible as it evolves:

| Version | Purpose |
|---|---|
| `1` (legacy) | Original single-password format, still readable for backward compatibility |
| `2` (current default) | Single-volume format: header stores flags, PBKDF2 iteration count, salt, and IV |
| `3` (multi-volume) | Powers **both** multi-recipient encryption and decoy volumes with one mechanism |

The v3 multi-volume layout:

```
MAGIC(4) version(1)=3 flags(1) volumeCount(1)
  for each volume:
    slotCount(1)
    for each slot: salt(16) iterations(4) slotIv(12) wrappedCek(48)
    volumeIv(12) cipherLen(4) paddedLen(4) cipher+padding(paddedLen)
```

Each volume gets its own random **content encryption key (CEK)**. Every recipient "slot" independently wraps that same CEK under a key derived from its own password (and optional keyfile) via PBKDF2 — so the (potentially large) payload is only ever encrypted once per volume, no matter how many recipients or how many decoys share the file. When decrypting, every slot in every volume is tried against the supplied password concurrently, and volume sizes are padded to match so an observer can't tell from ciphertext length alone whether a decoy is present.

Split parts get their own small header (`group ID`, `total parts`, `part index`) so the Decrypt panel can detect and reassemble a group as soon as all parts have been dropped in, regardless of the order they were added.

## Security notes

- **No password recovery.** If a password (and any required keyfile) is lost, the encrypted data is unrecoverable by design — there is no backdoor and no server-side copy.
- **Secure context required for encryption.** The Web Crypto API only runs in a "secure context" — HTTPS, or `http://localhost`. Opening the file directly (`file://`) works for compression in most browsers, but some browsers (notably Firefox) will block encryption/decryption in that mode. Deploying over HTTPS (see [Deployment](#deployment)) avoids this entirely.
- **Everything stays local.** There is no server component. No files, passwords, or keyfiles are ever sent over the network by this tool.
- **Decoys are misdirection, not deniability.** The UI is explicit about this: unlike a fixed-size hidden-volume scheme, an archive containing a decoy can still be a different size than a single-volume one, which may hint to a careful examiner that more than one payload exists.
- **Strength depends on your password.** PBKDF2 with a high iteration count slows down brute-forcing, but a weak or reused password is still the weakest link — use the built-in generator or strength meter as a guide.

This is a personal/utility-grade encryption tool built with standard, well-reviewed primitives (AES-GCM, PBKDF2-SHA256) via the browser's native crypto implementation. It has not undergone a third-party security audit — treat it accordingly for anything high-stakes.

## Getting started

No installation is required — SecureZip is a single HTML file.

```bash
git clone https://github.com/<your-username>/SecureZip.git
cd SecureZip
```

Then either open the file directly in a browser, or serve it locally (recommended, so encryption works in every browser):

```bash
# Python 3
python3 -m http.server 8080

# Node (no install, via npx)
npx serve .
```

Visit `http://localhost:8080/securezip-11.html` (or `index.html`, if you've renamed it — see below).

## Deployment

Because SecureZip is a static, dependency-free HTML file, it can be deployed to essentially any static hosting provider. Pick whichever fits your workflow:

### Option 1: GitHub Pages (recommended)

1. Rename the file to `index.html` at the repository root (or keep the versioned name and add a small `index.html` that redirects to it — see [Repository structure](#repository-structure)).
2. Commit and push to your default branch.
3. In your repo on GitHub, go to **Settings → Pages**.
4. Under **Build and deployment → Source**, choose **Deploy from a branch**.
5. Select your branch (e.g. `main`) and the `/ (root)` folder, then **Save**.
6. GitHub will publish the site at `https://<your-username>.github.io/SecureZip/` within a minute or two.

GitHub Pages serves everything over HTTPS by default, which satisfies the "secure context" requirement for the Web Crypto API — encryption will work for all visitors out of the box.

### Option 2: Netlify

1. [Sign in to Netlify](https://app.netlify.com) and choose **Add new site → Import an existing project**.
2. Connect your GitHub account and select the `SecureZip` repository.
3. Leave the build command empty and set the publish directory to the repository root (`.`).
4. Deploy — Netlify will serve the file over HTTPS at a generated `*.netlify.app` URL (or a custom domain you attach).

### Option 3: Vercel

1. [Import the repository](https://vercel.com/new) into Vercel.
2. Framework preset: **Other** (no framework/build step needed).
3. Leave build and output settings at their defaults for a static site, or set the output directory to the repo root.
4. Deploy — Vercel serves it over HTTPS automatically.

### Option 4: Cloudflare Pages

1. In the Cloudflare dashboard, go to **Workers & Pages → Create → Pages → Connect to Git**.
2. Select the `SecureZip` repository.
3. Build command: none. Build output directory: `/` (repo root).
4. Deploy — Cloudflare Pages serves the site over HTTPS on a `*.pages.dev` subdomain (or a custom domain).

### Option 5: Any static host / self-hosted

Since there's no build step, you can also drop the HTML file onto:
- An S3 bucket (with static website hosting or CloudFront in front, for HTTPS)
- Nginx, Apache, or Caddy on your own server — just serve the file as static content
- Any object storage / static file host that supports custom uploads

Only requirement: serve it over **HTTPS** (or access it at `http://localhost`) so the Web Crypto API is available in every browser.

### Running locally

For local development or offline use without any hosting:

```bash
python3 -m http.server 8080
# or
npx serve .
```

Opening the file with a plain double-click (`file://…`) also works for most operations in Chrome/Edge/Safari; only Firefox is known to block Web Crypto in that mode (compression-only still works there).

## Repository structure

SecureZip ships as a single file, so the repository can stay minimal:

```
SecureZip/
├── index.html          # SecureZip app (rename from securezip-11.html for deployment)
├── securezip-11.html    # optional: keep the versioned filename as a source-of-truth copy
└── README.md
```

If you want to preserve a history of iterations (as the `securezip-11` naming suggests this isn't the first build), consider a simple structure like:

```
SecureZip/
├── index.html            # always the latest release, deployed by Pages/Netlify/etc.
└── releases/
    ├── securezip-10.html
    ├── securezip-11.html # current
    └── ...
```

and update `index.html` (or a redirect inside it) each time you promote a new build.

## Browser support

SecureZip needs two modern browser APIs:

- **Web Crypto API** (`crypto.subtle`) — for AES-256-GCM and PBKDF2
- **Compression Streams API** (`CompressionStream` / `DecompressionStream`) — for gzip

Both are available in recent versions of Chrome, Edge, Firefox, and Safari. If either is missing, the app shows a banner explaining what's unsupported rather than failing silently; compression-only workflows can still work in more browsers than encryption, since encryption additionally requires a secure context.

## Versioning

This repository's HTML file is versioned by filename (e.g. `securezip-11.html` = the 11th build). There's no separate changelog embedded in the file today — if you want one, consider adding a `CHANGELOG.md` and tagging each promoted build as a GitHub release (e.g. `v11`) so the deployed `index.html` always traces back to a specific tagged source file.

## Known limitations

- Decoy volumes are explicitly **not** true plausible deniability (see [Security notes](#security-notes)).
- No password recovery mechanism exists, by design.
- Very large files are limited by available browser memory, since processing happens client-side.
- No automated test suite is included in this build; verify changes manually (encrypt → decrypt round trip, verify-only, multi-recipient, decoy, and split/reassemble flows) before deploying.

## Contributing

Issues and pull requests are welcome. Since this is a single-file app, please:
1. Keep it dependency-free (no new CDN scripts, frameworks, or build steps) unless there's a strong reason to introduce one.
2. Test encryption, decryption, multi-recipient, decoy, split/reassemble, and verify-only flows before submitting changes that touch the crypto or container-format code.
3. Note the container format version you're touching (v1/v2/v3) and preserve backward-compatible decryption for existing files.

## License

No license file is currently included in this repository. Until one is added, all rights are reserved by default under standard copyright — which means others technically can't legally reuse, modify, or redistribute the code.

## Disclaimer

SecureZip is provided as-is, without warranty of any kind. It has not undergone a formal third-party security audit. Use your own judgment about whether it's appropriate for your threat model, especially for high-stakes or regulated data.
