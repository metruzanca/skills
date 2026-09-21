---
name: use_gpui
description: Build GPU-accelerated, cross-platform native desktop apps in Rust with GPUI (Zed's UI framework). Use when the user is writing or reviewing GPUI code — views, elements, state, subscriptions, styling, theming, events, actions, focus/keybindings, testing, or performance — or when they mention GPUI, gpui, Zed UI, or a native Rust desktop UI.
---

# GPUI Skill

Guidance for writing, reviewing, and tuning GPUI applications. GPUI is a hybrid
immediate/retained, GPU-accelerated UI framework for Rust by the Zed Industries
team. This directory is split into focused topic files; read the one(s) that
match the task at hand instead of this whole file.

## API correctness rule (read first)

All code in these files is checked against the GPUI source at
`crates/gpui` in [zed-industries/zed](https://github.com/zed-industries/zed)
(**GPUI 0.2.2**). The API is:

- One context type: **`Context<'a, T>`** (derefs to `App`). Old names
  `ViewContext`, `WindowContext`, `ModelContext` are gone.
- One entity type: **`Entity<T>`** (created with `cx.new(|cx| …)`), replacing
  `View<T>` and `Model<T>`.
- `Render::render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement`.
- Colors are built with `rgb(0x…)`, `rgba(0x…, alpha)`, or an `Hsla { h, s, l, a }`
  literal. There are **no** `blue_500()`-style palette helpers in core gpui.
- There are **no** `input()`, `textarea()`, or `button()` elements in core gpui.
  Use `div()` + handlers, or widgets from the `ui` crate (Zed application crate,
  not `gpui`).

If you encounter older-style GPUI code (`ViewContext`, `Model<T>`,
`cx.new_view`/`cx.new_model`, `impl_actions!`, `blue_500()`, `input()`,
`window.focus(&handle)`), translate it per the compatibility table in
[`fundamentals.md`](#api-compatibility), or ask the user to confirm the GPUI
revision they target before keeping old forms. Never emit unverified APIs.

## Topic index

| When the user asks about…                        | Read |
|--------------------------------------------------|------|
| Toolchain, Cargo.toml, project layout, scaffolding a new app | [`setup.md`](setup.md) |
| First app, views/entities, elements, events, focus, actions, text input, todo/counter examples, old→new API translation | [`fundamentals.md`](fundamentals.md) |
| State, models, observe/subscribe, globals, async, container/presenter, action system | [`state.md`](state.md) |
| Styling, layout, colors, themes, responsive design, reusable component recipes | [`styling.md`](styling.md) |
| Project architecture, layer separation, state ownership, SOLID, DI, plugins | [`architecture.md`](architecture.md) |
| Render/layout/memory optimization, profiling, benchmarks | [`performance.md`](performance.md) |
| Testing views/state/interactions | [`testing.md`](testing.md) |
| Reviewing GPUI code (idiomaticity, perf, correctness) | [`review.md`](review.md) |

## How to use

1. Identify the task category above and read the matching file(s). A task like
   "build a settings panel" usually needs `styling.md` + `state.md`;
   "why is my list janky" needs `performance.md`.
2. Follow the code patterns exactly as written — they use the current, verified
   GPUI API.
3. When a snippet depends on the running GPUI version (animation, widgets,
   test helpers), verify against the crate that is actually in the user's
   `Cargo.lock` before copying.

## Working style

- Show complete, runnable examples (imports included), then explain.
- Point out pitfalls proactively: missing `cx.notify()`, subscriptions not
  stored, focus chain incomplete, non-compiling legacy snippets.
- If API drift is suspected (the user's repo targets an older GPUI),
  surface it and normalize rather than guessing.
- The `gpui` crate lives in the Zed repo; read its source
  (`crates/gpui/src/…`) when a behavior needs confirming.