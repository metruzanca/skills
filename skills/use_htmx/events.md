# Events, scripting, and the JavaScript API

htmx is attribute-driven, but events are its integration surface and its
logging system. This file covers the request lifecycle events, the `hx-on*`
attributes, and the small JavaScript API. Everything is checked against
`reference/htmx/www/content/events.md`, `api.md`, and
`attributes/hx-on.md`.

## Event naming

All htmx events fire under **two names**: camelCase and kebab-case. Listen for
`htmx:afterSwap` *or* `htmx:after-swap`. Kebab-case matters because HTML
attributes are case-insensitive (see `hx-on` below) and some libraries (e.g.
Alpine.js) only see kebab-case.

Attach listeners on `document.body` (events bubble from the triggering
element) — or use the `htmx.on` helper:

```js
document.body.addEventListener('htmx:load', function(evt) {
  myJavascriptLib.init(evt.detail.elt);
});
```

## The request lifecycle (in order)

Every event's `detail` carries `elt`, `xhr`, `target`, and `requestConfig`
unless noted. `reference/htmx/www/content/events.md` is the authoritative
list; the full request order of operations is in `docs.md#request-operations`.

| Event | When | Notes |
|---|---|---|
| `htmx:configRequest` | params collected, before request | mutate `evt.detail.parameters` / `.headers` to add data |
| `htmx:beforeRequest` | just before the request issues | `preventDefault()` cancels it |
| `htmx:beforeSend` | right before the XHR sends | cannot cancel |
| `htmx:beforeSwap` | before content swaps | set `shouldSwap`, `target`, `swapOverride`, `selectOverride`; `preventDefault()` cancels the swap |
| `htmx:beforeTransition` | before a View-Transition swap | `preventDefault()` falls back to a normal swap |
| `htmx:afterSwap` | content swapped | |
| `htmx:afterSettle` | DOM settled | |
| `htmx:afterRequest` | request finished (success or error) | `detail.successful` / `.failed` |
| `htmx:responseError` | HTTP error (non-2xx/3xx) | |
| `htmx:sendError` | network error | |
| `htmx:load` | new content added to the DOM | the hook for initializing 3rd-party libs in swapped content |
| `htmx:beforeProcessNode` / `htmx:afterProcessNode` | before/after htmx initializes a node | |
| `htmx:abort` | *sent to* an element | abort its in-flight request |

Two commonly needed patterns:

**Reject `422` and swap it** (see [`server.md`](server.md#response-handling)):

```js
document.body.addEventListener('htmx:beforeSwap', (evt) => {
  if (evt.detail.xhr.status === 422) {
    evt.detail.shouldSwap = true;
    evt.detail.isError = false;
  }
});
```

**Add a token to every request**:

```js
document.body.addEventListener('htmx:configRequest', (evt) => {
  evt.detail.headers['X-CSRF-TOKEN'] = getToken();
});
```

## Inline scripting: `hx-on*`

Respond to any event inline, preserving Locality of Behaviour. The event name
is part of the attribute, after a colon:

```html
<div hx-on:click="alert('Clicked!')">Click</div>
<button hx-get="/info" hx-on:htmx:before-request="alert('Making a request!')">
  Get Info!
</button>
```

- **Gotcha:** DOM attributes are case-insensitive, so `hx-on:htmx:beforeRequest`
  **does not work** — use the kebab-case `hx-on:htmx:before-request`.
- Shorthand `hx-on::before-request` = `hx-on:htmx:before-request`.
- For a plain `click`, the standard `onclick` is fine; use `hx-on:` for
  htmx/custom events.
- These are a lightweight scripting mechanism, not a replacement for Alpine.js
  or hyperscript. CamelCase-only custom events need one of those.
- `hx-on` requires eval-like functionality; disable with
  `htmx.config.allowEval = false` if you need strict CSP.

## The JavaScript API

htmx ships a small API (not its focus; reach for attributes first). Full
signatures: `reference/htmx/www/content/api.md`. The essentials:

- `htmx.ajax(verb, path, target)` — issue an htmx-style request; returns a
  Promise. `htmx.ajax('GET', '/example', '#myDiv')`, or with a context object
  `{target, swap, values, headers, push, select, ...}`. **If you find yourself
  using this heavily, consider whether attributes fit better** (QUIRKS.md).
- `htmx.trigger(elt, name, detail)` — fire an event; also used to abort:
  `htmx.trigger('#button', 'htmx:abort')`.
- `htmx.process(elt)` — initialize htmx attributes on content you added to the
  DOM yourself (e.g. after `innerHTML =` from `fetch`).
- `htmx.on(elt, event, handler)` / `htmx.off(...)` — add/remove listeners.
- `htmx.onLoad(callback)` — `htmx:load` handler; initialize 3rd-party libs on
  swapped content (e.g. SortableJS).
- `htmx.find(sel|elt, sel)` / `htmx.findAll(...)` — query helpers.
- `htmx.addClass/removeClass/toggleClass/takeClass(elt, cls)` — class helpers.
- `htmx.values(elt)` — resolve the values htmx would submit for an element.
- `htmx.swap(target, content, swapSpec)` — swap HTML programmatically.
- `htmx.remove(elt, delay)` — remove an element (optionally after a delay).
- `htmx.defineExtension(name, ext)` / `htmx.removeExtension(name)` — extension
  management (see [`extensions.md`](extensions.md)).
- `htmx.parseInterval('3s')` — parse htmx timing strings (milliseconds).
- `htmx.config` — the runtime config object (see
  [`setup.md`](setup.md#configuring-the-meta-tag) for the `meta` tag form).
  Notable flags: `selfRequestsOnly`, `allowEval`, `allowScriptTags`,
  `historyCacheSize`, `defaultSwapStyle`, `globalViewTransitions`,
  `methodsThatUseUrlParams` (default `["get","delete"]` — `DELETE` params go in
  the URL by spec, body for others).

## Logging and debugging

- `htmx.logAll()` — log every htmx event to the console; `htmx.logNone()` stops it.
- `htmx.logger = function(elt, event, data){...}` — custom logger.
- `monitorEvents(htmx.find("#theElement"))` — browser-console helper to see
  what events an element fires (console only, not embeddable in a page).
- For demos/repros, `<script src="https://demo.htmx.org"></script>` installs
  htmx + hyperscript + a request-mocking layer (`<template url="/foo">…</template>`).
- When content isn't behaving, confirm the element was actually processed
  (`htmx.process` for hand-inserted DOM) before debugging the request.