# State: history, out-of-band swaps, preserving, synchronizing

How htmx handles state that spans requests: browser history, "out of band"
updates that bypass the request target, elements preserved across swaps, and
coordinating concurrent requests. Checked against `reference/htmx/www/content/
docs.md` (History Support, OOB swaps), `attributes/hx-swap-oob.md`,
`hx-push-url.md`, `hx-replace-url.md`, `hx-history-elt.md`, `hx-history.md`,
`hx-preserve.md`, `hx-sync.md`, `hx-disabled-elt.md`, `hx-select-oob.md`.

## History

### Pushing URLs (`hx-push-url`)

To make an htmx request create a browser-history entry, add `hx-push-url`:

```html
<a hx-get="/blog" hx-push-url="true">Blog</a>
```

Before the request, htmx snapshots the current DOM to its history cache (in
`localStorage`); on back/forward it restores the snapshot. On a cache miss it
issues an AJAX request to the URL with `HX-History-Restore-Request: true` and
expects the **full page HTML** back. Values: `true` (push the fetched URL),
`false` (disable an inherited push), or an explicit URL.
**Inherited.** The `HX-Push-Url` response header overrides the attribute.

**You must be able to serve a full page at any URL you push** — a user can
copy-paste it, and htmx needs the whole page for history restoration on a miss.

`hx-replace-url` does the same but via `replaceState` (no new history entry);
the `HX-Replace-Url` header overrides it.

### History snapshot element (`hx-history-elt`)

By default htmx snapshots/restores the `body`. Narrow it to a child when
needed (e.g. `#content`), but the element must be present on every page or
restoration breaks. **Not inherited.** In most cases leave it at `body`.

### Excluding sensitive pages (`hx-history`)

Set `hx-history="false"` anywhere in the document to keep that page out of the
`localStorage` cache — history navigation then re-fetches from the server
instead. Also disable caching entirely with
`<meta name="htmx-config" content='{"historyCacheSize": 0}'>`.

### History gotchas

- Third-party libs that mutate the DOM need cleanup before a snapshot is taken.
- `hx-preserve` elements have their state preserved across history navigation.
- `htmx.config.historyRestoreAsHxRequest` defaults to `true`; when your server
  uses `HX-Request` to decide between full page and fragment, disable it so the
  full-history restore isn't cached as a fragment (see [`server.md`](server.md#caching)).

## Out-of-band swaps (`hx-swap-oob`)

Piggyback updates to **other** elements on a response, outside the request
target. The *response* marks an element to be swapped elsewhere:

```html
<!-- response body -->
<div>
  This goes into the request target.
</div>
<div id="alerts" hx-swap-oob="true">Saved!</div>
```

`div#alerts` is swapped into the existing `#alerts` in the page, not into the
target. Values: `true` (or `outerHTML`) for inline replacement, any `hx-swap`
value (`hx-swap-oob="beforeend:#list"` appends), optionally `:selector` to
target matching elements. The wrapping tags of the OOB element are stripped
except for `outerHTML`/`true`.

**Troublesome tables/SVG**: `<tr>`, `<td>`, `<li>`, `<circle>` etc. can't stand
alone in the DOM — wrap them in a `template` tag (which htmx strips):

```html
<template>
  <tr id="row" hx-swap-oob="true">...</tr>
</template>
```

**Nested OOB**: by default OOB elements *nested inside* the main response
element are still processed (`htmx.config.allowNestedOobSwaps`). If your
template fragments double as OOB targets and main fragments, set that config to
`false` so only OOB elements *adjacent* to the main response are processed.
**Not inherited.**

`hx-select-oob` is the client-side pair: pick elements out of a response for
OOB swaps without the server marking them, e.g. `hx-select-oob="#alert"` or
`hx-select-oob="#alert:afterbegin"`. **Inherited.**

## Preserving content (`hx-preserve`)

Keep an element's state (video playback, a form, focus) across swaps that would
replace its ancestor:

```html
<div id="video" hx-preserve></div>
```

The element is preserved by **`id`** (you must keep the `id` stable), and the
response must still contain an element with the same `id`. **Not inherited.**
Caveats: `<input type="text">` focus/caret and iframes can't be perfectly
preserved — prefer a morph swap (idiomorph, [`extensions.md`](extensions.md))
for those; avoid `hx-swap="none"` when a response may carry a `hx-preserve`
element; a preserved element can be *relocated* if the response nests it
elsewhere.

## Synchronization (`hx-sync`)

Coordinate requests between elements to resolve races. Form:
`hx-sync="<selector>:<strategy>"` — a selector (often `this` or
`closest form`) plus one of:

- `drop` (default) — ignore this request if a request is already in flight.
- `abort` — drop this request if one is in flight; also abort this request if
  another arrives while it runs.
- `replace` — abort the in-flight request and replace it with this one.
- `queue` (with `first`/`last`/`all`) — queue requests instead of dropping.

The canonical case is a form whose submit races an input's validation request:

```html
<form hx-post="/store">
  <input id="title" name="title" type="text"
         hx-post="/validate" hx-trigger="change"
         hx-sync="closest form:abort">
  <button type="submit">Submit</button>
</form>
```

`hx-sync` is **inherited**. Requests can also be cancelled programmatically by
dispatching `htmx:abort` to the element (see [`events.md`](events.md)).

## Disabling during requests (`hx-disabled-elt`)

Add the `disabled` attribute to specified elements for the duration of a
request — prevents double-submits:

```html
<button hx-post="/example" hx-disabled-elt="this">Post It!</button>
```

Value: a CSS selector, `this`, extended selectors, or a comma-separated list
(no `this` in lists); `inherit, ...` merges with a parent's list.
**Inherited.**