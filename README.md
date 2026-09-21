# Skills

A unified repository of agent skills. Each skill lives in
`skills/<name>/` with `SKILL.md` as the entry point and focused topic
files for progressive disclosure.

## Skills

| Skill | Covers |
|-------|--------|
| [`use_gleam`](skills/use_gleam/) | Writing, debugging, and reviewing Gleam code — a functional language compiling to Erlang and JavaScript that runs on the BEAM |
| [`use_gpui`](skills/use_gpui/) | Building GPU-accelerated, cross-platform native desktop apps in Rust with GPUI (Zed's UI framework) |

## Install

Install individual skills with the Skills CLI:

```sh
npx skills add metruzanca/skills --skill use_gleam
npx skills add metruzanca/skills --skill use_gpui
```

For local development:

```sh
npx skills add ./skills/use_gleam
npx skills add ./skills/use_gpui
```