# 🛡️ parapet

> **The defensive barrier for static sites.**  
> Zero-browser link auditing, dead DOM anchor detection, and CSP drift checking in under 30ms.

[![Rust](https://img.shields.io/badge/Rust-2021%2B-orange.svg?style=flat-square)](https://www.rust-lang.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](https://opensource.org/licenses/MIT)
[![Speed](https://img.shields.io/badge/Speed-%3C30ms-brightgreen.svg?style=flat-square)]()
[![CI](https://img.shields.io/badge/CI-GitHub%20Actions-2088FF.svg?style=flat-square)](https://github.com/nirjxr26/parapet/actions)

---

## 💡 The Problem: Static Sites Fail Silently

Static builds are blazing fast to serve, but they fail quietly:

1. **Dead DOM Anchors**: Renaming a heading from `id="architecture-review"` to `id="architecture"` silently breaks any link pointing to `#architecture-review`. Browsers don't 404—they shrug and scroll to the top of the page.
2. **Stale Feeds & Sitemaps**: A route gets slugified differently during a refactor, but `sitemap.xml` and `llms.txt` keep advertising the old URL. Search crawlers and AI agents hit a 404 on a file whose sole job is to be accurate.
3. **CSP Drift**: Strict Content Security Policies require whitelisting inline scripts via SHA-256 hashes (`'sha256-...'`). Changing a single whitespace character in a blocking theme script leaves your build passing, but modern browsers will refuse to execute it in production.

Booting headless Chromium via Playwright or Cypress in CI costs **30 to 60 seconds**, hundreds of megabytes of RAM, and introduces flaky network timeouts just to verify text files sitting on a disk.

**`parapet` replaces the browser with a 28ms deterministic static scan.**

---

## ⚡ Benchmark

Tested on a production static site with 17 prerendered pages, 272 links and assets, sitemaps, and strict CSP manifests:

| Metric | `parapet` (Rust) | Playwright / Cypress | Node regex script |
| :--- | :--- | :--- | :--- |
| **Execution Time** | **28 ms** | 30 – 60 s | ~1,800 ms |
| **Memory Footprint** | **< 12 MB** | 400 – 800 MB | ~150 MB |
| **Dependencies** | **0 (Native binary)** | Chromium + Node runtime | Node + `node_modules` |
| **Flakiness** | **0% (Pure deterministic I/O)** | 5 – 10% (Timeouts) | Low |
| **CSP Hash Audit** | **Built-in** | Requires plugins | Custom script |

---

## 🚀 Quick Start

### 1. Run without installation (Cargo)

```bash
cargo run --release -- dist
```

### 2. Install globally via Cargo

```bash
cargo install parapet
parapet dist
```

### 3. Usage Examples

```bash
# Basic check against your dist folder (domain auto-detected from sitemap.xml)
parapet dist

# Explicit canonical domain
parapet dist --domain https://example.com

# Non-blocking CI warning mode (always exits with code 0)
parapet dist --warn-only

# Skip CSP or sitemap checks if not applicable
parapet dist --skip-csp --skip-sitemap
```

---

## 🛠️ CLI Reference

```text
parapet 0.1.0
Blazing-fast zero-browser static site integrity auditor & CSP drift checker.

USAGE:
    parapet [OPTIONS] [DIST]

ARGS:
    <DIST>                       Path to built static directory [default: dist]

OPTIONS:
    -r, --root <ROOT>            Project root directory [default: auto-detected]
    -d, --domain <DOMAIN>        Canonical domain (e.g. https://example.com)
                                 [default: auto-detected from sitemap.xml]
    -m, --manifests <FILES>      Comma-separated list of security manifests to check
                                 [default: public/_headers, vercel.json, netlify.toml]
        --skip-csp               Skip Content Security Policy inline script hash audit
        --skip-sitemap           Skip sitemap.xml route auditing
        --skip-llms              Skip llms.txt route auditing
        --warn-only              Exit with code 0 even if integrity errors are found
    -q, --quiet                  Silence non-error output
    -h, --help                   Print help information
    -V, --version                Print version information
```

---

## 🔍 What `parapet` Audits

### 1. Two-Pass DOM Anchor Verification
* **Pass 1**: Indexes all `id="..."` and `<a name="...">` targets across every `.html` file into an in-memory hash map.
* **Pass 2**: Scans every `<a href>`, `<link href>`, `<script src>`, and `<img src>`. If a target contains a `#fragment`, it verifies that the destination file has an element with that exact ID.

### 2. Content Security Policy (CSP) Drift
* Extracts executable inline scripts from `index.html` (skipping `application/ld+json`, `importmap`, etc.).
* Computes the SHA-256 base64 digest in standard W3C CSP format (`sha256-<hash>`).
* Confirms that every active inline script's hash is declared in your deploy manifests (`_headers`, `vercel.json`, `netlify.toml`).

### 3. Sitemaps & `llms.txt` Routes
* Validates every `<loc>` in `sitemap.xml` against actual generated routes.
* Audits URLs listed in `llms.txt` so AI agents and crawlers never hit a dead route.

### 4. Clean URLs & Rewrites
* Understands directory indexes (`/services` &rarr; `services/index.html` or `services.html`).
* Reads native rewrite and redirect rules from `vercel.json` and `_redirects` (Netlify / Cloudflare Pages).

---

## 🤖 CI / CD Integration

### GitHub Actions

Add `parapet` immediately following your build step:

```yaml
name: CI

on: [push, pull_request]

jobs:
  build-and-verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Build Static Site
        run: |
          npm ci
          npm run build

      - name: Cache Parapet
        uses: actions/cache@v4
        with:
          path: ~/.cargo/bin/parapet
          key: ${{ runner.os }}-parapet

      - name: Run Parapet
        run: |
          which parapet || cargo install parapet
          parapet dist
```

### Pre-commit / Package.json

```json
{
  "scripts": {
    "build": "vite build && prerender",
    "verify:site": "parapet dist",
    "test": "npm run build && npm run verify:site"
  }
}
```

---

## 🏛️ What is a Parapet?

> In military architecture, a **parapet** is a protective barrier placed at the edge of a rampart or roof to guard against falls and external threats.

In web architecture, `parapet` is the last deterministic wall before deployment. It guarantees that what you ship is integral, consistent, and secure.

---

## 📄 License

MIT © [Nirjar Goswami](https://github.com/nirjxr26)
