# Reviewing htmx code

A structured pass for correct, idiomatic, and safe htmx — both the markup and
the server contract behind it. Run this checklist when asked to review an htmx
change or app; report findings in the format at the bottom. Reference the topic
files and the official docs in `reference/htmx/www/content/` for each item.

## Scope & discovery

- Find the HTML with `hx-*` attributes (and `data-hx-*`), plus the server
  endpoints they hit.
- Identify the htmx version the app pins (the `<script>` URL or `htmx.org` in
  `package.json`). If it's not a 2.x line, note it and verify version-specific
  claims (see [`setup.md`](setup.md#versions)).
- List each request attribute and its `hx-trigger`, `hx-target`, `hx-swap`
  trio — most bugs live in the mismatch between them.

## Markup correctness

- [ ] Every `hx-get`/`hx-post`/`hx-put`/`hx-patch`/`hx-delete` has a target
      that actually exists at request time and a swap that matches intent.
- [ ] `hx-target` values use correct extended selectors (`this`, `closest …`,
      `find …`, `next …`, `previous …`) — and resolve from the *triggering*
      element, not the element that declares an inherited `hx-include`/etc.
- [ ] `hx-swap` strategy matches the response shape: an `outerHTML` swap
      against a fragment that doesn't include the element, or `innerHTML` when
      the response is a full element, are the classic mistakes.
- [ ] `hx-trigger` modifiers are sensible: `delay` debounces, `throttle`
      drops; `changed` avoids redundant requests; `every Ns` polling actually
      terminates (via `286` or load polling) when it should.
- [ ] `hx-trigger` is not inherited — a request "that used to work" that
      depends on a parent trigger needs its own trigger.
- [ ] Inherited attributes (`hx-target`, `hx-swap`, `hx-confirm`, `hx-boost`,
      `hx-vals`, `hx-include`, `hx-headers`, `hx-indicator`, `hx-sync`,
      `hx-disabled-elt`, `hx-select`) are not accidentally hoisted to affect
      elements that shouldn't inherit; `hx-disinherit`/`unset` used where intended.
- [ ] `hx-on:` handlers use **kebab-case** event names
      (`hx-on:htmx:before-request`), not camelCase — which silently fails.
- [ ] No deprecated `hx-vars` — normalized to `hx-vals` (with `js:` where a
      dynamic value is intended).
- [ ] `hx-vals`/`hx-headers`/`hx-request` values are valid JSON (single-quoted
      inner strings silently drop).
- [ ] Elements that must survive swaps use `hx-preserve` with a stable `id`,
      or a morph swap for focus/caret-sensitive inputs.
- [ ] `hx-encoding="multipart/form-data"` present for file uploads.
- [ ] `data-*` prefix used consistently if the codebase has chosen it.

## Server contract

- [ ] Every htmx endpoint returns **HTML fragments** (or uses `hx-select`), not JSON.
- [ ] Error responses that should render are handled: `responseHandling`
      config, `htmx:beforeSwap` per-status handling, or the response-targets
      extension — not left silently unswapped (see [`server.md`](server.md#response-handling)).
- [ ] `HX-Request` / `HX-Boosted` used correctly to branch between full page
      and fragment when the same URL serves both.
- [ ] Response headers (`HX-Trigger`, `HX-Redirect`, `HX-Location`,
      `HX-Push-Url`, `HX-Reswap`, `HX-Retarget`) used rather than a full-page
      redirect where a fragment + header would do; no `3xx` carrying htmx
      response headers (they're ignored on 3xx).
- [ ] `204 No Content` used for "no swap needed" responses.
- [ ] Caching: `Vary: HX-Request` (or `getCacheBusterParam`) when a URL serves
      both full and partial content; `historyRestoreAsHxRequest` disabled when
      `HX-Request` drives response shape.
- [ ] CSRF token sent via `hx-headers` or a hidden form input.
- [ ] Any URL pushed via `hx-push-url` (or `HX-Push-Url`) serves a full page
      on direct navigation.

## State & coordination

- [ ] OOB swaps (`hx-swap-oob`) have a matching `id` in the page; table/SVG
      elements wrapped in `template` tags; nested-OOB behaviour is intended.
- [ ] Concurrent requests are coordinated with `hx-sync` (submit racing an
      input's validation is the canonical case).
- [ ] Destructive actions use `hx-confirm`; double-submits prevented with
      `hx-disabled-elt`.
- [ ] History features (`hx-push-url`, `hx-history-elt`) are used deliberately,
      given their gotchas (see [`state.md`](state.md)).

## Security

- [ ] No unescaped user content injected into htmx-bearing markup; if raw HTML
      is injected, `hx-disable` wraps it or tags/attributes are whitelisted.
- [ ] No `js:`/`javascript:` values built from untrusted input in
      `hx-vals`/`hx-headers` (XSS vector).
- [ ] Sensitive pages excluded from the history cache (`hx-history="false"`
      or `historyCacheSize: 0`).
- [ ] Cross-domain requests restricted (`selfRequestsOnly`) or explicitly
      allowlisted via `htmx:validateUrl`.
- [ ] `allowScriptTags`/`allowEval` considered under strict CSP; nonce config
      set if CSP requires it.
- [ ] CSRF protection actually present (header or hidden input).

## Idioms

- [ ] Behaviour is attribute-driven near the element (Locality of Behaviour),
      not scattered `htmx.ajax()` calls in a script — unless there's a genuine
      reason.
- [ ] The smallest reasonable fragment is returned (server-side template
      fragments where available), not the whole page.
- [ ] The full-page vs fragment decision lives server-side on `HX-Request`,
      not duplicated client-side.
- [ ] No over-engineering: plain links/forms where a request isn't needed;
      `hx-boost` chosen deliberately given its tradeoffs.

## Report format

For each finding, report:

- `Severity` — `blocking` (won't work / security) | `warning` (fragile /
  likely bug) | `suggestion` (style/idiom).
- `Location` — file and the offending element/attribute.
- `Issue` — one line.
- `Fix` — the corrected markup/server behaviour, from the relevant topic file.

Finish with a one-line verdict on whether the htmx usage is sound overall.