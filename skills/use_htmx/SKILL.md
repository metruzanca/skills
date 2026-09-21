---
name: use_htmx
description: Write, debug, and review htmx — a dependency-free JavaScript library that brings AJAX, CSS transitions, WebSockets, and Server-Sent Events directly into HTML via `hx-*` attributes, so the server (any backend) responds with HTML fragments instead of JSON. Use when the user is writing or reviewing HTML with htmx attributes — hx-get, hx-post, hx-put, hx-patch, hx-delete, hx-boost, hx-trigger, hx-target, hx-swap, hx-select, hx-vals, hx-include, hx-on, hx-swap-oob, hx-sync, hx-push-url, hx-history-elt, hx-ext — or building the server side of an htmx app (responding with partial HTML, the HX-Request/HX-Trigger/HX-Redirect/HX-Location and other headers, response handling, history, SSE, WebSockets), or when they mention htmx, hypermedia-driven applications, HDA, HATEOAS, partials/fragments instead of JSON, or migrating from Turbo/Hotwire or intercooler.
---

# htmx Skill

Guidance for writing, debugging, and reviewing htmx — "high power tools for
HTML." htmx is a small (~16k min.gz), dependency-free, browser-side library
that generalizes HTML's built-in hypermedia: any element can issue any HTTP
request, any event can trigger one, and any element can be the swap target.
Because htmx lives entirely in attributes on your HTML, it is **language
agnostic**: the backend is whatever you already use, and it responds with HTML
fragments rather than JSON. This directory is split into focused topic files;
read the one(s) that match the task at hand instead of this whole file.

## Verification rule (read first)

All code and claims in these files are checked against the official htmx
docs at `reference/htmx/www/content/` in this repo — a checkout of the
[bigskysoftware/htmx](https://github.com/bigskysoftware/htmx) repo, **htmx
2.x line** (`docs.md`, `reference.md`, `attributes/*`, `headers/*`,
`events.md`, `api.md`, `extensions/*`, `examples/*`, `QUIRKS.md`).

Version facts that matter:

- This skill documents **htmx 2.x** (the `latest` npm tag). htmx 4.0 was
  released in 2026 but is **not yet marked `latest`** on npm, and its docs
  live on the separate `four` branch / `four.htmx.org`. Do not emit 4.x-only
  APIs as if they are current.
- The reference checkout is a live repo: `git -C reference/htmx status`
  will show whether it has moved. If you are writing for an app that pins an
  exact version, check the user's actual version (the `<script>` tag or
  `package.json`) and verify against that line before relying on a snippet.

If a claim or attribute cannot be confirmed in the reference, rewrite it to a
confirmed equivalent or delete it. **Never invent htmx attributes, events, or
headers.**

## Topic index

| When the user asks about… | Read |
|---|---|
| Installing htmx (CDN, npm, module formats), the `meta` config tag, version pinning, no-build-step setup | [`setup.md`](setup.md) |
| The core model: `hx-get`/`hx-post`/`hx-put`/`hx-patch`/`hx-delete`, triggers & modifiers & filters & polling, targets & extended selectors, swapping & swap modifiers, indicators, parameters, `hx-boost`, inheritance, `hx-on`, CSS transitions | [`fundamentals.md`](fundamentals.md) |
| What the server must do: returning HTML fragments, `HX-Request` detection, all request & response headers, `responseHandling`, 204/error codes, caching, CSRF | [`server.md`](server.md) |
| History (`hx-push-url`, `hx-history-elt`), out-of-band swaps (`hx-swap-oob`, `hx-select-oob`), `hx-preserve`, `hx-sync`, `hx-disabled-elt` | [`state.md`](state.md) |
| Events (`htmx:*` lifecycle), the JavaScript API (`htmx.ajax`, `htmx.trigger`, `htmx.process`, `htmx.config`, …), `hx-on:` scripting, logging | [`events.md`](events.md) |
| Extensions: idiomorph/morph swaps, response-targets, preload, head-support, SSE, WebSocket, community extensions, building your own | [`extensions.md`](extensions.md) |
| Architecture: hypermedia-driven apps, HATEOAS, Locality of Behaviour, template fragments, progressive enhancement, security (XSS, CSP, CSRF), when htmx fits | [`architecture.md`](architecture.md) |
| Reviewing htmx markup and server code (correctness, idioms, security) | [`review.md`](review.md) |

## How to use

1. Identify the task category and read the matching file(s). Writing a new
   interaction usually needs `fundamentals.md` (+ `server.md` for the
   backend contract); debugging a non-firing request needs `events.md`;
   "is this safe?" needs `architecture.md`.
2. Follow the patterns exactly as written — they use the current, verified
   htmx 2.x API. Snippets are complete enough to copy.
3. The official docs in `reference/htmx/www/content/` are the source of
   truth; open the linked file there when a snippet's edge cases matter
   (e.g. `attributes/hx-trigger.md` for the full modifier list).

## Working style

- Show complete, runnable HTML (the `hx-*` attributes + the surrounding
  element), then explain what the server must return.
- Keep the backend language-neutral: say "the server returns this HTML
  fragment" or "the server sets the `HX-Trigger` header", not how a given
  framework does it, unless the user names their stack.
- Point out htmx pitfalls proactively: the `body` target always swaps
  `innerHTML`, `4xx`/`5xx` responses don't swap by default, `GET` on a
  non-form element doesn't include enclosing form values, `hx-on:htmx:event`
  camelCase breaks (attributes are case-insensitive), `hx-vars` is deprecated
  in favor of `hx-vals`, attributes like `hx-trigger` are not inherited while
  `hx-target`/`hx-swap`/`hx-confirm`/`hx-boost`/`hx-vals`/`hx-include` are.
- When behaviour depends on version (2.x vs 4.x), say so and verify against
  the user's actual version rather than assuming parity.