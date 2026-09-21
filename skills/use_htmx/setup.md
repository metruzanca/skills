# Setup: installing htmx, config, versions

Getting htmx loaded and configured. There is **no build step**: htmx is a
single dependency-free JavaScript file, and a standard blocking `<script>` tag
is the recommended way to load it.

## Load via CDN

The official docs recommend jsDelivr. Always load core htmx **before** any
extensions, and keep the `integrity`/`crossorigin` attributes:

```html
<script src="https://cdn.jsdelivr.net/npm/htmx.org@2.0.10/dist/htmx.min.js"
        integrity="sha384-H5SrcfygHmAuTDZphMHqBJLc3FhssKjG7w/CeCpFReSfwBWDTKpkzPP8c+cLsK+V"
        crossorigin="anonymous"></script>
<button hx-post="/clicked" hx-swap="outerHTML">Click Me</button>
```

Note the CDN URLs pin a version (`htmx.org@2.0.10`). If you write a `<script>`
tag, pin an explicit version and match the user's target (see [versions](#versions)).

## Via npm / a bundler

```sh
npm install htmx.org
```

htmx 2.x ships browser-loadable `dist/htmx.js` plus module files: ESM
`dist/htmx.esm.js`, AMD `dist/htmx.amd.js`, CJS `dist/htmx.cjs.js`, and the
types file `dist/htmx.esm.d.ts`.

```js
import 'htmx.org'; // or import * as htmx from 'htmx.org'
```

## Loading asynchronously is unreliable

htmx is designed to be loaded with a **standard, blocking** `<script>` tag —
not `defer`, not a module, not dynamically injected. Loading it asynchronously
works only as a best-effort and can silently fail. Recommend the blocking tag.

## Configuring: the `meta` tag

Runtime config is read from a `<meta name="htmx-config">` tag (or set
programmatically on `htmx.config`, see [`events.md`](events.md)). The value is
JSON; the tag must appear before htmx is used:

```html
<meta name="htmx-config" content='{"defaultSwapStyle":"outerHTML"}'>
```

Common one-liners:

- Default swap style: `{"defaultSwapStyle":"outerHTML"}`
- Disable history caching: `{"historyCacheSize": 0}`
- Allow `422` responses to swap (validation errors): see
  [`server.md`](server.md#response-handling).
- Disable attribute inheritance entirely: `{"disableInheritance": true}` (then
  opt in per-element with `hx-inherit`, see [`fundamentals.md`](fundamentals.md#inheritance)).
- Disable htmx's injected indicator CSS: `{"includeIndicatorStyles": false}`.
- Add a nonce for inline scripts/styles under a strict CSP:
  `{"inlineScriptNonce": "nonce-value", "inlineStyleNonce": "nonce-value"}`.

## Versions

- This skill documents the **2.x line**, which is what the reference checkout
  covers and what `latest` on npm resolves to.
- **htmx 4.0** was released in 2026 but is not yet marked `latest` on npm; its
  docs are on the `four` branch (`four.htmx.org`). Before writing version
  specific code, check which line the user's app is on (the pinned `<script>`
  URL or the `htmx.org` version in `package.json`).
- **htmx 1.x** still exists for IE11 support and is frozen. The
  `htmx-1-compat` extension rolls most 2.x behaviour back to 1.x defaults —
  see [`extensions.md`](extensions.md).
- Migrating from 1.x to 2.x or from Turbo/Hotwire / intercooler.js? The
  reference checkout has `migration-guide-htmx-1.md`,
  `migration-guide-hotwire-turbo.md`, and `migration-guide-intercooler.md` in
  `reference/htmx/www/content/`.

## Serving the assets

Your server must serve the htmx script (static file, CDN, or bundled). htmx
itself is browser-side; any backend works. The only server-side requirement is
that your endpoints respond with the **HTML fragments** htmx swaps in — see
[`server.md`](server.md).