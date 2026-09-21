# Server side: the backend contract

htmx is language-agnostic because the contract between client and server is
pure HTTP + HTML. This file is the neutral description of what your backend —
Django, Rails, Phoenix, ASP.NET, Go, anything — must do. The one hard rule:

> **Respond with HTML fragments, not JSON.**

htmx expects the response body to be HTML (typically a fragment) and swaps it
into the DOM. A full document works too when combined with `hx-select`. All
headers here are checked against `reference/htmx/www/content/docs.md#requests`,
`reference.md#request_headers` / `#response_headers`, and the `headers/*` files.

## Detecting an htmx request

Every htmx request sends `HX-Request: true`. Use it to decide between returning
a fragment (htmx request) and a full page (plain navigation):

| Request header | Meaning |
|---|---|
| `HX-Request` | always `"true"` for htmx requests (except history-restore requests when `historyRestoreAsHxRequest` is disabled) |
| `HX-Boosted` | present if the request came from an `hx-boost` element |
| `HX-Current-URL` | the browser's current URL |
| `HX-Target` | `id` of the target element, if it has one |
| `HX-Trigger` | `id` of the triggering element, if it has one |
| `HX-Trigger-Name` | `name` of the triggering element, if it has one |
| `HX-Prompt` | the user's answer to an `hx-prompt` |
| `HX-History-Restore-Request` | `"true"` when restoring history after a cache miss |

## Responding

Return the new HTML for the target. There is **no Post/Redirect/Get needed**:
after a successful `POST`, return the replacement fragment directly (200) rather
than a redirect. If you want no swap at all (e.g. the interaction is handled by
a response header), return **`204 No Content`** and htmx ignores the body.

## Response headers

Set any of these on a successful (non-3xx) response. Full details in
`reference/htmx/www/content/headers/`:

| Header | Effect |
|---|---|
| `HX-Location` | client-side "boost-like" navigation to a URL — issues an AJAX request and pushes history (JSON form: `{"path":"/x","target":"#div"}`) |
| `HX-Redirect` | full browser redirect to a new location (use for non-htmx endpoints / when `<head>` must reload) |
| `HX-Push-Url` | push a URL into history (overrides `hx-push-url`); `false` to prevent |
| `HX-Replace-Url` | replace the current history URL (no new entry) |
| `HX-Refresh` | `"true"` → full page refresh |
| `HX-Retarget` | CSS selector: swap into a different element than the request target |
| `HX-Reswap` | override the swap strategy for this response (see `hx-swap` values) |
| `HX-Reselect` | CSS selector: choose which part of the response to swap |
| `HX-Trigger` | trigger client events immediately |
| `HX-Trigger-After-Swap` | trigger client events after the swap |
| `HX-Trigger-After-Settle` | trigger client events after settling |

`HX-Trigger` accepts a bare event name (`HX-Trigger: myEvent`) or a JSON map
whose values become `event.detail`:

```
HX-Trigger: {"showMessage":{"level":"info","message":"Here Is A Message"}}
```

Events bubble up from the triggering element, so listen on `body` — and for an
`hx-trigger` firing from a header, use `hx-trigger="myEvent from:body"`.
(Details: `headers/hx-trigger.md`, [`events.md`](events.md).)

**Gotcha:** response headers are **not** processed on `3xx` redirect responses —
the browser intercepts the redirect and returns the redirected URL's headers
instead. Prefer `200` + a header (or `HX-Redirect`) over `302`.

## Status codes and response handling {#response-handling}

By default htmx treats status codes via `htmx.config.responseHandling`:

```js
[
  {code:"204", swap:false},            // 204: no swap, not an error
  {code:"[23]..", swap:true},          // 2xx, 3xx: swap, not an error
  {code:"[45]..", swap:false, error:true}, // 4xx, 5xx: no swap, error
  {code:"...", swap:false}             // anything else: no swap
]
```

Consequences: **`4xx`/`5xx` responses are not swapped by default**, and
`204` isn't swapped. This surprises people whose framework returns `422` for
validation errors — the server's error HTML never appears. Two fixes:

1. **Per-request**, handle the event (see [`events.md`](events.md)):
   ```js
   document.body.addEventListener('htmx:beforeSwap', (evt) => {
     if (evt.detail.xhr.status === 422) {
       evt.detail.shouldSwap = true;
       evt.detail.isError = false;  // keep it out of the console error logs
     }
   });
   ```
2. **Globally**, configure `responseHandling` in a `meta` tag:
   ```html
   <meta name="htmx-config" content='{
     "responseHandling":[
       {"code":"204","swap":false},
       {"code":"[23]..","swap":true},
       {"code":"422","swap":true},
       {"code":"[45]..","swap":false,"error":true},
       {"code":"...","swap":false}
     ]}' />
   ```
   Or via the **response-targets extension**, which targets per-status-code
   declaratively: `<button hx-target-404="#not-found">` — see
   [`extensions.md`](extensions.md).

Network errors trigger `htmx:sendError`; HTTP errors trigger
`htmx:responseError`.

## CORS

When calling cross-origin endpoints, the server must send
`Access-Control-Allow-Headers` (for the `HX-*` request headers) and
`Access-Control-Expose-Headers` (for the `HX-*` response headers) or htmx's
headers won't be visible to the client. Note `htmx.config.selfRequestsOnly`
defaults to `true` — cross-domain htmx requests are blocked unless set to
`false` or allowed via the `htmx:validateUrl` event.

## Caching

htmx uses standard HTTP caching. The important gotcha: if the same URL renders
a full page for plain navigation and a fragment for `HX-Request`, the two
responses must not share a cache slot — the server needs `Vary: HX-Request`
(and distinct `ETag`s), plus you should disable
`htmx.config.historyRestoreAsHxRequest` so full-history restores aren't cached
with fragment responses. Alternatively `getCacheBusterParam: true` adds a
cache-busting parameter to htmx `GET`s. Details:
`reference/htmx/www/content/docs.md#caching`.

## CSRF

CSRF token handling is a backend responsibility; htmx supports sending a token
on every request via `hx-headers` on an ancestor (e.g. the `<body>`):

```html
<body hx-headers='{"X-CSRF-TOKEN": "TOKEN_GOES_HERE"}'>
```

Note `hx-boost` does not replace `<html>`/`<body>`, so a token placed there
survives boosted navigation; a hidden form input is the framework-idiomatic
alternative. Full guidance:
`reference/htmx/www/content/docs.md#csrf-prevention`.

## Scripting / security boundary

If your backend injects raw (untrusted) HTML, it must be scrubbed — htmx
increases what injected HTML can do (`hx-*` attributes, `<script>` tags in
responses). Wrap untrusted content in `hx-disable` as a backstop, and see the
security section of [`architecture.md`](architecture.md).