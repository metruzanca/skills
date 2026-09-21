# Fundamentals: the core htmx model

The heart of htmx: attributes that issue requests, what triggers them, where
the response goes, and how it is swapped. Everything here is checked against
`reference/htmx/www/content/docs.md`, `reference.md`, and the `attributes/*`
files. The full list of every attribute, class, header, event, and config
option lives in `reference/htmx/www/content/reference.md`.

## The request attributes

Five attributes issue a request of a given HTTP verb to a URL:

```html
<button hx-get="/thing">   <!-- GET    -->
<button hx-post="/things"> <!-- POST   -->
<button hx-put="/thing">   <!-- PUT    -->
<button hx-patch="/thing"> <!-- PATCH  -->
<button hx-delete="/thing"><!-- DELETE -->
```

The element issues the request when its **natural event** fires: `click` for
most elements, `change` for `input`/`textarea`/`select`, `submit` for `form`.
Override this with `hx-trigger`.

The `data-` prefix works everywhere: `data-hx-post="/click"` is the same as
`hx-post="/click"`. Use it when you want HTML-validity or are running other
tooling that dislikes non-standard attributes.

## Triggers (`hx-trigger`)

Specify what fires the request. Full reference:
`reference/htmx/www/content/attributes/hx-trigger.md`.

- **Event name**: `hx-trigger="click"`, `hx-trigger="mouseenter"`,
  `hx-trigger="keyup"`, or any custom event name.
- **Modifiers**, space-separated after the event:
  - `once` — fire only on the first event.
  - `changed` — only fire if the element's value changed.
  - `delay:1s` — wait, resetting the timer if the event fires again (debounce).
  - `throttle:1s` — wait, dropping events that arrive during the wait (leading + cooldown).
  - `from:<selector>` — listen on another element (e.g. `from:body` for hotkeys).
  - `target:<selector>` — filter by event target.
  - `consume` — stop the event from triggering parent htmx requests.
  - `queue:first|last|all|none` — how to handle events arriving while a request is in flight.
- **Filters** in square brackets, a JS expression: `hx-trigger="click[ctrlKey&&shiftKey]"`.
  Symbols resolve against the event first, then the global scope; `this` is the element.
- **Special events**:
  - `load` — fires once when the element loads (lazy-load content).
  - `revealed` — fires once when the element scrolls into the viewport.
  - `intersect` — fires once when the element intersects the viewport
    (`root:<sel>`, `threshold:<float>` options).
- **Polling**: `hx-trigger="every 2s"` re-issues the request on an interval.
  Respond with HTTP `286` to stop polling.

Debounced search:

```html
<input name="q" hx-get="/search" hx-trigger="keyup changed delay:500ms"
       hx-target="#search-results">
<div id="search-results"></div>
```

Multiple triggers, comma-separated, each with its own options:
`hx-trigger="load, click delay:1s"`.

`hx-trigger` is **not inherited**.

## Targets (`hx-target`)

The response is swapped into the target element — default is the element that
issued the request. `hx-target` takes a CSS selector or an **extended selector**
(also used by `hx-include`, `hx-indicator`, `hx-disabled-elt`, and others):

- `this` — the element itself.
- `closest <sel>` — the nearest ancestor (or self) matching.
- `find <sel>` — the first descendant matching.
- `next <sel>` / `previous <sel>` — the next/previous element matching (or
  bare `next`/`previous` for the sibling).
- A selector in `<` `/>` (hyperscript query-literal style).

```html
<div>
  <div id="response-div"></div>
  <button hx-post="/register" hx-target="#response-div" hx-swap="beforeend">
    Register!
  </button>
</div>
<a hx-post="/new-link" hx-target="this" hx-swap="outerHTML">New link</a>
```

`hx-target` is **inherited**.

## Swapping (`hx-swap`)

Controls how the response replaces the target. Full reference:
`reference/htmx/www/content/attributes/hx-swap.md`.

| Value | Behaviour |
|---|---|
| `innerHTML` | default — puts content inside the target |
| `outerHTML` | replaces the target element itself |
| `afterbegin` | prepends inside the target |
| `beforebegin` | inserts before the target |
| `beforeend` | appends inside the target |
| `afterend` | inserts after the target |
| `delete` | deletes the target, ignores response |
| `none` | no swap (OOB swaps and response headers still processed) |
| `textContent` | replaces text without parsing HTML |

**Modifiers** (colon-separated after the strategy):

- `swap:1s` — delay between receiving the response and swapping.
- `settle:1s` — delay between the swap and settling (CSS transitions).
- `ignoreTitle:true` — don't update the document title from a `<title>` in the response.
- `scroll:top|bottom` and `show:top|bottom` (plus `scroll:#sel:top`, `show:window:top`, `show:none`).
- `focus-scroll:true` — scroll the focused input into view after the request.
- `transition:true` — use the View Transitions API for this swap.

`hx-swap` is **inherited**.

**Morph swaps** (`morph`, `morph:innerHTML`, `morph:outerHTML`) merge new DOM
into existing nodes instead of replacing them, preserving focus/video state —
via the idiomorph extension, see [`extensions.md`](extensions.md).

**CSS transitions**: keep the element's `id` identical between old and new
content; htmx copies old attributes onto the new element, swaps, then settles
to the new values — so a `transition` on the class change animates without JS:

```html
<!-- old --> <div id="div1">Original Content</div>
<!-- new --> <div id="div1" class="red">New Content</div>
```

## Selecting content from the response (`hx-select`, `hx-select-oob`)

Pull a subset of the response instead of the whole body:

```html
<button hx-get="/info" hx-select="#info-detail" hx-swap="outerHTML">Get Info!</button>
```

`hx-select-oob="#alert"` picks elements out of the response for out-of-band
swaps (see [`state.md`](state.md)). Both are inherited.

## Request indicators

While a request is in flight, htmx adds the `htmx-request` class to the
triggering element (or to the element named by `hx-indicator`). Elements with
the `htmx-indicator` class are shown via injected CSS:

```html
<button hx-post="/example" hx-indicator="#spinner">Post It!</button>
<img id="spinner" class="htmx-indicator" src="/img/bars.svg" alt="Loading..."/>
```

The other lifecycle classes (also on the target): `htmx-swapping`,
`htmx-settling`, and `htmx-added` on new content. These are what make CSS
transitions work. Full request order-of-operations:
`reference/htmx/www/content/docs.md#request-operations`.

`hx-indicator` is inherited.

## Parameters

By default a request includes the element's value if it has one; a form
includes all its named inputs. `name` attributes are the parameter names.

- `hx-include` — add values from other elements (`hx-include="[name='email']"`,
  `hx-include="closest form"`, `inherit, ...` to merge with a parent).
  **Inherited.**
- `hx-params` — filter what's sent: `*`, `none`, `not a,b`, or `a,b`.
  **Inherited.**
- `hx-vals` — add fixed values in JSON: `hx-vals='{"myVal": "My Value"}'`.
  Prefix `js:` to evaluate: `hx-vals='js:{lastKey: event.key}'`. **Inherited.**
  Malformed JSON is ignored with a console error.
- `hx-vars` — **deprecated**, the older dynamic form. Use `hx-vals`.
- `hx-headers` — add request headers in JSON (`'{"X-CSRF-TOKEN": "…"}'`).
  **Inherited.**
- `hx-encoding` — set to `multipart/form-data` for file uploads. **Inherited.**
- `hx-request` — per-request options in JSON: `{"timeout":100}`,
  `credentials`, `noHeaders`. Merge-inherited.
- `hx-confirm` — show a `confirm()` before issuing; **inherited**.
- `hx-prompt` — show a `prompt()`; the answer is sent in the `HX-Prompt` header. **Inherited.**

Form submit semantics: a `GET` on a **non-form** element does **not** include
the enclosing form's values by default — use `hx-include="closest form"` if
needed. Non-`GET` requests from inside a form do include the form's inputs.

## Boosting (`hx-boost`)

Turns ordinary `<a href>` and `<form>` into AJAX requests that swap the
`body`'s innerHTML and push the URL into history — a progressive-enhancement
layer that still works with JS disabled:

```html
<div hx-boost="true">
  <a href="/blog">Blog</a>
</div>
```

- Inherited; disable per-child with `hx-boost="false"` (or `unset`).
- Only same-domain links and non-local anchors are boosted.
- Detect boosted requests server-side via the `HX-Boosted` request header.
- Tradeoffs: only `body` swaps (styles/scripts in the response `<head>` are
  discarded), the global JS scope isn't refreshed, and history can get tricky
  (see [`architecture.md`](architecture.md) and `QUIRKS.md`).

## Inheritance

Most attributes are inherited: `hx-target`, `hx-swap`, `hx-confirm`,
`hx-boost`, `hx-vals`, `hx-include`, `hx-params`, `hx-headers`, `hx-indicator`,
`hx-disabled-elt`, `hx-sync`, `hx-encoding`, `hx-request`, `hx-select`. Not
inherited: `hx-get`/`hx-post`/etc, `hx-trigger`, `hx-swap-oob`, `hx-preserve`,
`hx-history-elt`, `hx-validate`, `hx-ext` is inherited *and merged*.

Undo inheritance per-element with `unset` (`hx-confirm="unset"`), carve out
specific attributes with `hx-disinherit="hx-target"` (or `hx-disinherit="*"`),
or flip the whole system with `htmx.config.disableInheritance` and opt in with
`hx-inherit="hx-target"`.

## Inline scripting: `hx-on*`

`hx-on:<event>` runs inline JS for any event — htmx's generalized replacement
for the fixed HTML `on*` attributes. Because HTML attributes are
case-insensitive, use **kebab-case** event names:

```html
<button hx-post="/example"
        hx-on:htmx:config-request="event.detail.parameters.example = 'Hello Scripting!'">
  Post Me!
</button>
```

`hx-on::before-request` is shorthand for `hx-on:htmx:before-request`. Standard
`onclick` is fine for standard events; use `hx-on:` for htmx/custom events.
More in [`events.md`](events.md).

## Common patterns

The reference checkout has runnable examples in
`reference/htmx/www/content/examples/` — click-to-edit, click-to-load,
active-search, lazy-load, infinite-scroll, inline-validation, edit-row,
bulk-update, delete-row, progress-bar, tabs, dialogs, and more. Reach for the
matching example when building a known pattern.