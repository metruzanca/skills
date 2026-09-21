# State: entities, subscriptions, globals, async, composition

Patterns for managing state in GPUI and structuring stateful components.

## Entities: the state unit

Both "models" (state) and "views" (rendered UI) are the same type: **`Entity<T>`**,
created with `cx.new(...)`. A view observing shared state holds an `Entity` field.

```rust
use gpui::{Context, Entity, Render, Window};

#[derive(Clone)]
struct AppState {
    count: usize,
    items: Vec<String>,
}

struct AppView {
    state: Entity<AppState>,
    _subscription: Subscription, // keep the subscription alive (see below)
}

impl AppView {
    fn new(state: Entity<AppState>, cx: &mut Context<Self>) -> Self {
        let _subscription = cx.observe(&state, |_, _, cx| cx.notify());
        Self { state, _subscription }
    }

    fn increment(&mut self, cx: &mut Context<Self>) {
        self.state.update(cx, |state, cx| {
            state.count += 1;
            cx.notify();
        });
    }
}

impl Render for AppView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let state = self.state.read(cx);
        div().child(format!("Count: {}", state.count))
    }
}
```

Read with `entity.read(cx)`, mutate with `entity.update(cx, |value, cx| ...)`.

## Reactivity: observe, subscribe, notify

- `cx.notify()` marks *this* entity dirty; GPUI re-renders observers.
  Call it after every visible state change.
- `cx.observe(&other, |this, other, cx| ...)` fires when `other` is notified.
- `cx.subscribe(&emitter, |this, emitter, event, cx| ...)` receives typed events
  that the emitter sent with `cx.emit(...)` (requires `EventEmitter<Evt>`).
- `cx.observe_self(...)` / `cx.subscribe_self(...)` react to your own updates.

### Store subscriptions — always

A `Subscription` only stays active while held. Discarding it silently disables
the callback and observers get stale UI. Store it in a field:

```rust
// BAD: subscription dropped as soon as `new` returns
impl AppView {
    fn new(state: Entity<AppState>, cx: &mut Context<Self>) -> Self {
        cx.observe(&state, |_, _, cx| cx.notify()); // dead on arrival
        Self { state }
    }
}

// GOOD: field keeps it alive; dropped with the view
struct AppView {
    state: Entity<AppState>,
    _subscription: Subscription,
}
```

A common footgun is creating subscriptions *inside* `render()` — that leaks a
new subscription every frame. Create them in the constructor.

### Selective updates

Re-render only when the data you actually read changed:

```rust
let _subscription = cx.observe(&state, |this, state, cx| {
    let new_value = state.read(cx).important_field.clone();
    if new_value != this.cached_field {
        this.cached_field = new_value;
        cx.notify();
    }
});
```

## Global state

App-wide state (theme, settings, selected language) lives in a `Global`.

```rust
#[derive(Clone, Copy, PartialEq)]
enum ThemeMode { Dark, Light }

#[derive(Clone, Copy)]
struct AppTheme {
    mode: ThemeMode,
    primary: Hsla,
}
impl Global for AppTheme {}

// Set once at startup, from App/Context:
cx.set_global(AppTheme { mode: ThemeMode::Dark, primary: rgb(0x3b82f6) });

// Read anywhere a context is available:
let theme = cx.global::<AppTheme>();

// Update:
cx.update_global(|theme, _cx| {
    theme.mode = match theme.mode {
        ThemeMode::Dark => ThemeMode::Light,
        ThemeMode::Light => ThemeMode::Dark,
    };
});

// Views that display global values react by subscribing with
// `cx.observe_global::<AppTheme>(...)` or by reading the global in render and
// being notified through their own state; force a repaint with
// `window.refresh(cx)` where a global change must redraw immediately.
```

Prefer explicit `Entity` passing between co-located views over a global; globals
are for genuinely app-wide, low-churn values.

## Async work

Spawn background work with `cx.spawn`. You receive a `WeakEntity<Self>` (never a
strong one, to avoid leaks) and an async context that survives awaits.

```rust
fn load_data(&mut self, cx: &mut Context<Self>) {
    let state = self.state.clone();

    cx.spawn(|this, cx| async move {
        let data = fetch_data().await?;

        // Update shared state from the async context. Strong Entity handles
        // return R directly (not Result); use WeakEntity::update for the `?`.
        state.update(cx, |state, cx| {
            state.items = data;
            cx.notify();
        });

        Ok::<_, anyhow::Error>(())
    })
    .detach();
}
```

- `.detach()` on the returned `Task` runs it fire-and-forget; keeping the `Task`
  lets you cancel/observe completion.
- Prefer updating an `Entity` you already hold over `this` (the weak handle),
  unless you specifically need to touch the view itself.
- Window-scoped work (layout-aware, per-window timers) uses
  `cx.spawn_in(window, |this, cx: &mut AsyncWindowContext| async move { … })`.

## Action system (deep dive)

### Defining actions

Unit actions (no data):

```rust
actions!(app, [Increment, Decrement, Reset]);
```

Actions that carry data derive `Action` (note: needs `Clone`, `PartialEq`,
`serde::Deserialize`, `schemars::JsonSchema`):

```rust
use gpui::Action;

#[derive(Clone, PartialEq, serde::Deserialize, schemars::JsonSchema, Action)]
#[action(namespace = app)]
pub struct SetValue {
    pub value: i32,
}
```

### Handling actions

On elements inside `render()`:

```rust
div()
    .on_action(cx.listener(|this, action: &SetValue, _window, cx| {
        this.state.update(cx, |state, cx| { state.count = action.value; cx.notify(); });
    }))
```

Method-style handler (matches `fn(&mut T, &Action, &mut Window, &mut Context<T>)`):

```rust
.on_action(cx.listener(Self::set_value_from))
```

### Dispatching actions

Programmatically trigger actions (they flow through the focus/dispatch tree):

```rust
cx.dispatch_action(Increment);      // dispatch to focused element
cx.dispatch_action(SetValue { value: 5 });

// Window-scoped dispatch (from a context that has the window):
window.dispatch_action(&Reset, window, cx);
```

### Keybindings

Bind keys globally at startup; the *focused* element's handlers receive them:

```rust
cx.bind_keys([
    KeyBinding::new("cmd-+", Increment, None),
    KeyBinding::new("cmd--", Decrement, None),
    KeyBinding::new("cmd-0", Reset, None),
]);
```

## Component composition

### Container / presenter

**Container** owns state and logic; **presenter** is pure rendering fed by data
and callbacks. Keeps `render()` free of business logic.

```rust
struct EditorContainer {
    document: Entity<DocumentModel>,
    _subscription: Subscription,
}

impl EditorContainer {
    fn new(document: Entity<DocumentModel>, cx: &mut Context<Self>) -> Self {
        let _subscription = cx.observe(&document, |_, _, cx| cx.notify());
        Self { document, _subscription }
    }
}

impl Render for EditorContainer {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let doc = self.document.read(cx);
        EditorPresenter::new(
            doc.content.clone(),
            cx.listener(|this, new_content: &String, _window, cx| {
                this.document.update(cx, |doc, cx| {
                    doc.update_content(new_content.clone());
                    cx.notify();
                });
            }),
        )
    }
}

// Presenter: pure rendering; receives data + a callback, owns no state.
struct EditorPresenter {
    content: String,
    on_change: Box<dyn Fn(&String, &mut Window, &mut App) + 'static>,
}
```

Passing `cx.listener(...)` closures down as props mirrors unidirectional data
flow: children render, parents own mutable state.

### Compound components

Related components share a parent; the parent wires their data and actions:

```rust
struct Tabs {
    items: Vec<(SharedString, Entity<Panel>)>,
    active: usize,
}
```

Keep child components purely presentational and connect them from the parent.

## Lifecycle and cleanup

- Constructor: set up fields, subscriptions, spawned tasks.
- Notify: `cx.notify()` in every mutation handler.
- Cleanup: subscriptions drop automatically with the view. Implement `Drop`
  only for cleanup *beyond* subscription/task teardown (e.g. flushing a buffer).

## Unidirectional data flow

```
User Action → dispatch_action/handler → state.change + cx.notify() → render
```

Wherever possible, route mutations through a small set of view methods or
actions rather than scattering `update()` calls across components; it keeps the
data-flow auditable as the app grows. See [`architecture.md`](architecture.md)
for ownership hierarchies and larger-scale guidance.