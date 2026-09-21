# Extensions

htmx's extension mechanism adds behaviour without touching core. Extensions
are JavaScript, defined via `htmx.defineExtension`, loaded like any script,
and **enabled declaratively** with the `hx-ext` attribute. Checked against
`reference/htmx/www/content/extensions/*`.

## Installing and enabling

Load core htmx first, then the extension script. Enable with `hx-ext="name"`
on `<body>` (applies to all children) or on any element (applies to that
subtree). `hx-ext` is inherited **and merged** with parents; carve out a
subtree with `hx-ext="ignore:name"`. Multiple: `hx-ext="preload,morph"`.

```html
<head>
  <script src="https://cdn.jsdelivr.net/npm/htmx.org@2.0.10/dist/htmx.min.js" integrity="sha384-H5SrcfygHmAuTDZphMHqBJLc3FhssKjG7w/CeCpFReSfwBWDTKpkzPP8c+cLsK+V" crossorigin="anonymous"></script>
  <script src="https://cdn.jsdelivr.net/npm/htmx-ext-response-targets@2.0.4" integrity="sha384-T41oglUPvXLGBVyRdZsVRxNWnOOqCynaPubjUVjxhsjFTKrFJGEMm3/0KGmNQ+Pg" crossorigin="anonymous"></script>
</head>
<body hx-ext="response-targets">
```

Most extensions are on npm as `htmx-ext-<name>` (except idiomorph) and on
jsDelivr as `https://cdn.jsdelivr.net/npm/htmx-ext-<name>`. The docs suggest
vendoring instead of CDN in production.

## Core extensions

| Extension | `hx-ext` name | What it does |
|---|---|---|
| **idiomorph** | `morph` | Adds `morph` / `morph:innerHTML` / `morph:outerHTML` swap strategies that merge new DOM into existing nodes (preserves focus, video, state). npm package is `idiomorph`, script `idiomorph-ext.min.js` |
| **response-targets** | `response-targets` | Per-status-code targets: `hx-target-404="#not-found"`, `hx-target-error="…"`; swaps error responses that normally wouldn't swap |
| **preload** | `preload` | Loads content into cache before the user asks (`<a preload>`, `<button hx-get="/x" preload>`); adds `HX-Preloaded: true` header. Use sparingly — it costs bandwidth |
| **head-support** | `head-support` | Merges `<head>` content (styles, etc.) from htmx responses — core htmx only handles `<title>` |
| **sse** | `sse` | Server-Sent Events from HTML: `sse-connect="<url>"`, `sse-swap="<message>"`, `hx-trigger="sse:<name>"`, `sse-close` |
| **ws** | `ws` | WebSockets from HTML: `ws-connect="<url>"`, `ws-send` on a form |
| **htmx-1-compat** | `htmx-1-compat` | Restores most htmx 1.x defaults/attributes (`hx-ws`, `hx-sse`, `hx-on`, smooth scroll, DELETE body params, cross-domain) |

### SSE (`hx-ext="sse"`)

One-way server→client over plain HTTP. Configure with attributes:
`reference/htmx/www/content/extensions/sse.md`.

```html
<div hx-ext="sse" sse-connect="/chatroom" sse-swap="message">
  Updated in real time by every SSE `message`.
</div>
```

- `sse-connect="<url>"` — the EventSource URL.
- `sse-swap="<message-name>"` — which SSE event's data to swap in.
- `hx-trigger="sse:<name>"` — have an SSE message *trigger an HTTP request*.
- `sse-close="<name>"` — close the stream when that message arrives.
- SSE is uni-directional; use WebSockets for two-way.

### WebSocket (`hx-ext="ws"`)

Two-way. `reference/htmx/www/content/extensions/ws.md`.

```html
<div hx-ext="ws" ws-connect="/chatroom">
  <div id="notifications"></div>
  <form id="form" ws-send>
    <input name="chat_message">
  </form>
</div>
```

- `ws-connect="<url>"` (or `ws-connect="wss:<url>"`; defaults to same-origin,
  sending cookies).
- `ws-send` — submit the element's values as a WebSocket message on its trigger.
- Incoming messages that look like HTML swap into the element.

### Idiomorph / morph swaps

```html
<body hx-ext="morph">
  <button hx-get="/example" hx-swap="morph">Morph Me</button>
</body>
```

- `morph` / `morph:outerHTML` — morph the target and its children.
- `morph:innerHTML` — morph only the target's children.
- Morphing reuses existing DOM nodes, so focus, video, and element state
  survive swaps — at the cost of more CPU. Prefer it over `hx-preserve` for
  focus-heavy inputs. Requires `htmx-ext-sse` ≥ 2.2.4 if combined with SSE.

## Notable community extensions

The full list is in `reference/htmx/www/content/extensions/_index.md`. Commonly
useful:

- **class-tools** — declarative CSS-class toggling (`classes="toggle red:1s"`).
- **multi-swap** — swap several marked elements from one response, each with
  its own strategy.
- **json-enc** / **client-side-templates** — send/receive JSON with client-side
  templating (when you can't use pure HTML).
- **loading-states** — manage loading UI (disable, class toggles) during requests.
- **remove-me** — `remove-me="1s"` removes an element after a delay.
- **debug** — log all htmx events for an element; `htmx.logAll()` is often enough.

Community extensions are in the `bigskysoftware/htmx-extensions` repo
(`src/<name>/README.md`) or third-party repos — verify their READMEs rather
than guessing install paths.

## Building an extension

Define one with `htmx.defineExtension(name, def)` — typically in a standalone
JS file:

```js
htmx.defineExtension('my-ext', {
  init: function(api) { return null; },
  getSelectors: function() { return null; },
  onEvent: function(name, evt) { return true; },
  transformResponse: function(text, xhr, elt) { return text; },
  isInlineSwap: function(swapStyle) { return false; },
  handleSwap: function(swapStyle, target, fragment, settleInfo) { return false; },
  encodeParameters: function(xhr, parameters, elt) { return null; }
});
```

Then enable with `hx-ext="my-ext"`. Extension names should be short,
dash-separated. Full contract: `reference/htmx/www/content/extensions/building.md`.