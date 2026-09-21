# Architecture: hypermedia-driven apps, security, when htmx fits

The mental model behind htmx and the design guidance for building with it.
Checked against the essays and security docs in
`reference/htmx/www/content/essays/` and `docs.md#security`.

## The HDA mental model

htmx is the tool for the **Hypermedia-Driven Application (HDA)** architecture —
the synthesis of the MPA and SPA approaches. Two constraints define it
(`essays/hypermedia-driven-applications.md`):

1. **Declarative, HTML-embedded syntax** for front-end interactivity
   (attributes, not imperative JS).
2. **Hypermedia (HTML) as the server communication format** — not JSON/RPC.

So an HDA stays within the original REST architecture of the web and continues
to use HATEOAS (hypermedia as the engine of application state), which SPAs
abandon. Practically: the server renders fragments, htmx swaps them, and the
server's HTML *is* the application logic — no duplicated client-side model.

Consequences that shape every htmx app:

- **The server owns state.** There is no client-side model to keep in sync;
  the response to a request contains the new state.
- **Template fragments** (`essays/template-fragments.md`) let a server-side
  template render a partial for a swap without splitting into many files. When
  the same view serves both full page and fragment, branch on the `HX-Request`
  header (see [`server.md`](server.md#detecting-an-htmx-request)).
- **Locality of Behaviour** (`essays/locality-of-behaviour.md`): a unit's
  behaviour should be obvious by looking at that unit. `hx-*` attributes put
  behaviour next to the element — that's the design payoff. Prefer attributes
  near the element over hoisting them far up the tree, and over `htmx.ajax()`
  calls in a distant script.
- **Scripting augments, doesn't supersede** (`essays/hypermedia-friendly-scripting.md`):
  use events to communicate, keep state server-side, isolate genuinely
  client-heavy components ("islands") rather than rebuilding everything in JS.

## Progressive enhancement and accessibility

`hx-boost` is the native progressive-enhancement tool: links and forms keep
working with JS disabled because they degrade to ordinary navigation. It has
real tradeoffs though (`QUIRKS.md`): only the `body` swaps (response `<head>`
styles/scripts are dropped), the global JS scope isn't refreshed, and history
can get tricky. Some htmx core members recommend plain links/forms over
`hx-boost`. Decide deliberately.

Non-boosted htmx can also be made progressively enhanced — e.g. wrapping an
`hx-post` active-search input in a real `<form>` that still works on Enter —
at the cost of more thought and server branching on `HX-Request`.

Accessibility: htmx apps are HTML apps, so standard HTML accessibility rules
apply — semantic elements, visible focus states, labels on fields, good
contrast.

## Security

htmx makes HTML more expressive, so **injecting unescaped user content into an
htmx page is more dangerous** than into a static one: `hx-*` attributes and
`<script>` tags in a response are executed. The layered defenses:

1. **Escape all user content** (the first rule; your templating language's
   auto-escaping). If you inject raw HTML, whitelist allowed tags/attributes —
   scrub `hx-*`/`data-hx-*` and `<script>`.
2. **`hx-disable` as a backstop** — wrap untrusted raw HTML so htmx won't
   process any htmx behaviour inside it. It cannot be overridden by content
   beneath it:
   ```html
   <div hx-disable><%= raw(user_content) %></div>
   ```
3. **Config flags** (`htmx.config`):
   - `selfRequestsOnly: true` (default) — block cross-domain requests.
   - `allowScriptTags: false` — stop processing `<script>` in new content.
   - `historyCacheSize: 0` — don't store page HTML in `localStorage`.
   - `allowEval: false` — disables everything eval-based: event filters,
     `hx-on:*`, `js:`-prefixed `hx-vals`/`hx-headers`. Also enables stricter CSP.
   - `htmx:validateUrl` event — allowlist extra hosts beyond the current one
     (`evt.detail.url`, `evt.detail.sameHost`; `preventDefault()` to block).
4. **CSP** — e.g. `default-src 'self'` restricts connections; combined with the
   config flags for a layered posture. Under a nonce-based CSP, set
   `inlineScriptNonce`/`inlineStyleNonce` or disable `includeIndicatorStyles`
   and host the indicator CSS yourself.
5. **CSRF** — backend responsibility; send the token via `hx-headers` on an
   ancestor (see [`server.md`](server.md#csrf)). Remember `hx-boost` doesn't
   replace `<html>`/`<body>`, so tokens there survive boosted navigation.

Details: `docs.md#security`, `essays/web-security-basics-with-htmx.md`.

## When htmx fits

Reach for htmx when:

- The interaction is fundamentally server-rendered hypermedia: CRUD, search,
  pagination, partial updates, forms, navigation — the vast majority of web UIs.
- You want no build step, tiny payloads (~16k min.gz), and no framework lock-in,
  across any backend.
- You're on a server-side framework that already renders HTML and want richer
  interactivity without writing a JSON API + client model.

It's the wrong tool when you have a genuinely client-side model that must be
kept in sync with the server (real-time collaborative editing, complex local
state, offline-first), or when you need imperative, fine-grained DOM control
everywhere. Those components can be "islands" inside an htmx page rather than
the whole app. The essays `spa-alternative.md`, `when-to-use-hypermedia.md`,
`hypermedia-clients.md`, and the `a-real-world-*-to-htmx-port.md` series in the
reference cover the tradeoffs concretely.

## Migrating from other tools

- **1.x → 2.x**: `migration-guide-htmx-1.md` in the reference (module files,
  extensions moved out of core, `hx-on:` replaces `hx-on`, `makeFragment`
  always returns a `DocumentFragment`, IE dropped).
- **Turbo / Hotwire**: `migration-guide-hotwire-turbo.md` (Turbo Drive ≈
  `hx-boost`, Turbo Frames ≈ `hx-target`, Turbo Streams ≈ OOB swaps,
  Stimulus ≈ Alpine.js/hyperscript).
- **intercooler.js**: `migration-guide-intercooler.md`.
- **htmx 4.0**: released 2026, not yet `latest`; docs on the `four` branch.
  Verify against the user's actual version before assuming 4.x behaviour.