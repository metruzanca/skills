# Reviewing GPUI code

A structured pass for idiomatic, correct, and fast GPUI code. Run this checklist
when asked to review a GPUI change or codebase; report findings in the format at
the bottom.

## Scope & discovery

- Find `.rs` files importing `gpui` (`use gpui::…`).
- Categorize: views (`impl Render`), models/state (`Entity`-backed structs),
  entry points, utilities, tests.
- Review whole files, then cross-cutting patterns (all views using a model).

## Component structure

- [ ] One concern per view; state lives in an `Entity`, not ad-hoc fields that
      fight re-renders
- [ ] Presentational pieces are reusable (function/component), not copy-pasted
- [ ] Dependencies are explicit (handles/data passed in) — no global-by-default
- [ ] `render()` is pure: no side effects, no subscriptions, no I/O

## State management

- [ ] Mutable state is touched through `entity.update(cx, …)`, not borrowed
      across boundaries
- [ ] `cx.observe`/`cx.subscribe` happen in the **constructor** and the returned
      `Subscription` is **stored** in a field
      - ❌ *Leak/breakage #1*: a `Subscription` created in `render()` — a new
        subscription every frame
      - ❌ *Leak/breakage #2*: `cx.observe(...)` discarded — the observer silently
        never fires
- [ ] Every mutation handler calls `cx.notify()` (and only then)
- [ ] Data actions handle their payload properly (`#[derive(Action)]` + parse);
      keybindings reach handlers only when the focus chain is complete
  (`Focusable` + `track_focus` + `on_action` + `bind_keys`)

## Context & API usage

- [ ] Uses current API: `Context<T>` / `Entity<T>` / `cx.new`/`read`/`update`;
      no `ViewContext`, `Model<T>`, `cx.new_view`/`new_model`, `impl_actions!`,
      `blue_500()`, `input()` (see `fundamentals.md` → *API compatibility*)
- [ ] Colors/spacing come from theme or `rgb(0x…)`/`Hsla {}` — no invented helpers
- [ ] Window-aware code passes `&mut Window` where handlers require it
  (`cx.listener(|this, event, window, cx| …)`)

## Performance

- [ ] No expensive computation inside `render()` — cached/memoized instead
- [ ] No `cx.notify()` storming (per-item in a loop, per-tick timers)
- [ ] Element trees reasonably flat; dynamic lists use `.children(iter)`
- [ ] Long lists virtualized (`list`/`uniform_list`), not thousands of rows
- [ ] Layout is not read during render (cache via `observe_window_bounds`)
- [ ] No per-frame allocations in hot paths

## Anti-patterns to flag

```rust
// ❌ Circular ownership — prevents teardown
struct Circular { self_entity: Entity<Self> }

// ❌ Unbounded accumulation
struct History { entries: Vec<Entry> } // cap it

// ❌ Storing a context (won't compile; pass `&mut Context` to methods instead)
struct Bad { cx: Context<'static, Self> }
```

- [ ] No `Entity` cycles (use `WeakEntity` for back-references)
- [ ] No unbounded logs/caches (bounded collections)
- [ ] No stale-closure captures in dynamic lists — item identity captured as an
      owned value before the closure (see *fundamentals.md*)
- [ ] Event handlers that should not bubble use `emitter.capture_*`/`stop` at the
      right phase, not mutation of globals in handlers

## Framework best practices

- [ ] Actions defined via `actions!`/`#[derive(Action)]`; keybindings registered
      once at startup
- [ ] Focusable views expose `FocusHandle` via `Focusable` and are focused on
      startup (`window.focus(&handle, cx)` / `cx.focus_self(window)`)
- [ ] Theme is app-global (`Global`) and components read from it
- [ ] Async work uses `cx.spawn`/`cx.spawn_in` + `.detach()`, updating an `Entity`
      after `await` — no blocking in handlers
- [ ] Public component APIs documented; error paths surfaced to the user

## Report format

For each issue:

```
📁 src/ui/views/main_view.rs:45   ❌ [Critical] Subscription created in render()

Problem:  cx.observe(...) called inside render() allocates a new subscription
          every frame; they are never cleaned up.
Fix:      Move to the constructor and store it:
          struct MainView { state: Entity<AppState>, _sub: Subscription }
          fn new(state, cx) -> Self {
              let _sub = cx.observe(&state, |_, _, cx| cx.notify());
              Self { state, _sub }
          }
```

Severity guide: **Critical** (leak/incorrect behavior) · **High** (perf or
correctness risk) · **Medium** (structure/idiom) · **Low** (style/docs).

End with a summary: files reviewed, issues by severity, top 3 fixes to make
first, and **positive patterns to keep** (e.g. "good container/presenter split in
editor views", "subscriptions stored on all models").