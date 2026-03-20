# AGENTS.md

## Cursor Cloud specific instructions

This is a **static documentation site** (single `index.html` with inline CSS/JS). There are no build tools, package managers, test frameworks, or linters.

### Running the site

Serve `index.html` with any HTTP server from the repository root:

```
python3 -m http.server 8080
```

Then open `http://localhost:8080/index.html` in a browser.

### Key notes

- The site references `logo.png` and `qrcode.png` which are not committed to the repo; these will show as broken images.
- All configuration (QQ group link, pricing) is managed via the `CONFIG` object in the inline `<script>` block at the bottom of `index.html`.
- No dependencies to install, no build step, no tests, no lint checks.
