# Skills

A unified repository of agent skills. Each skill lives in
`skills/<name>/` with `SKILL.md` as the entry point and focused topic
files for progressive disclosure.

## Skills

| Skill | Covers |
|-------|--------|
| [`use_gleam`](skills/use_gleam/) | Writing, debugging, and reviewing Gleam code — a functional language compiling to Erlang and JavaScript that runs on the BEAM |
| [`use_gpui`](skills/use_gpui/) | Building GPU-accelerated, cross-platform native desktop apps in Rust with GPUI (Zed's UI framework) |
| [`use_htmx`](skills/use_htmx/) | Writing, debugging, and reviewing htmx — AJAX, CSS transitions, WebSockets, and SSE driven by `hx-*` attributes in HTML, with any backend serving HTML fragments |

## Install

Install individual skills with the Skills CLI:

```sh
npx skills add metruzanca/skills --skill use_gleam
npx skills add metruzanca/skills --skill use_gpui
npx skills add metruzanca/skills --skill use_htmx
```

For local development:

```sh
npx skills add ./skills/use_gleam
npx skills add ./skills/use_gpui
npx skills add ./skills/use_htmx
```