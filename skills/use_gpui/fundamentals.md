# Fundamentals: core concepts, views, elements, events, focus

The foundations every GPUI app builds on. Read [`setup.md`](setup.md) first if
the project is not yet compiling.

## GPUI's rendering model

GPUI is a **hybrid immediate and retained mode**, GPU-accelerated UI framework.

- **Immediate mode**: every frame, your code describes the entire UI from scratch.
- **Retained mode**: GPUI remembers the previous scene, diffs it against the new
  description, and re-renders only what changed (like a virtual DOM, but for a
  GPU painter).

Consequence: write `render()` to return a *fresh description* of the UI each
call, and drive updates by calling `cx.notify()` when state changes. GPUI does
the diffing; you do not mutate elements imperatively.

```rust
// Imperative (not GPUI): mutate an existing object
// button.setText("New Label");

// Declarative (GPUI): describe what should appear
div().child("New Label")
```

## Application lifecycle

`Application::new().run()` starts everything; its closure receives `&mut App`
(the app context). From `App` (or any `Context<T>` by deref) you open windows
and create entities.

```rust
Application::new().run(|cx: &mut App| {
    let bounds = Bounds::centered(None, size(px(500.), px(500.)), cx);
    cx.open_window(
        WindowOptions {
            window_bounds: Some(WindowBounds::Windowed(bounds)),
            ..Default::default()
        },
        |_window, cx| cx.new(|_cx| HelloWorld::new()),
    )
    .expect("failed to open window");
});
```

`cx.open_window` builds a root **view** (an `Entity<V>` where `V: Render`) and
returns a `WindowHandle<V>` you can update or read later.

## Views, entities, and the Render trait

UI is made of Rust structs that implement `Render`. When you create one through
`cx.new(...)` (rather than as a bare struct), you get an `Entity<V>` that GPUI
can re-render and that can own shared state. The render method receives the
window and a context for the view:

```rust
use gpui::{div, prelude::*, px, rgb, Context, IntoElement, Render, SharedString, Window};

struct HelloWorld {
    name: SharedString, // SharedString is cheap to share across views
}

impl HelloWorld {
    fn new() -> Self {
        Self { name: "GPUI World".into() }
    }
}

impl Render for HelloWorld {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex()
            .flex_col()
            .bg(rgb(0x2e3440))
            .size(px(500.0))
            .justify_center()
            .items_center()
            .text_xl()
            .text_color(rgb(0xd8dee9))
            .child(format!("Hello, {}!", &self.name))
    }
}
```

Use `SharedString` instead of `String` for text that may cross view boundaries;
it avoids extra allocations without the ergonomic cost of `Arc<String>`.

## Elements and the component tree

`render()` returns `impl IntoElement`. The building blocks:

- `div()` — the workhorse container (layout + styling + interactivity)
- `img(...)`, `svg(...)` — images and icons
- `list` / `uniform_list` — virtualized scrolling lists (see [`performance.md`](performance.md))
- `anchored(...)`, `canvas(...)`, `surface(...)` — advanced containers
- Text is embedded through `.child("…")` / `.child(format!(…))` (rendered as
  `Text` elements); styled runs use `TextStyle`/font helpers

There is **no** `button()`, `input()`, or `textarea()` element in core gpui:
build interactive controls from `div()` + handlers (see below), or pull widgets
from the Zed `ui` crate (`ui::Button`, `ui::TextInput`, …) if you take that
dependency.

Most element APIs are **method chains** that read left-to-right:

```rust
div()
    .flex()                                  // flexbox layout
    .flex_col()                              // vertical stacking
    .gap_4()                                 // 4px gaps between children
    .bg(rgb(0x2e3440))                       // background color
    .size(px(500.0))                         // dimensions
    .justify_center()                        // main axis alignment
    .items_center()                          // cross axis alignment
    .text_xl()                               // text size
    .text_color(rgb(0xd8dee9))               // text color
    .child("Content")                        // add a child
```

`.child(x)` takes anything `IntoElement` (a `&str`, a `String`, another element,
an `Entity<V: Render>`); `.children(iter)` takes an iterator of elements — the
pattern for dynamic lists.

## Interactivity and events

### The focus system

Keyboard input needs the full focus chain. Never skip a step:

1. **Create the handle** in the constructor: `cx.focus_handle()`.
2. **Implement `Focusable`** for the view.
3. **Connect it in render** with `.track_focus(&self.focus_handle)` on the
   element that should receive focus.
4. **Register handlers** with `.on_action(cx.listener(Self::method))`.
5. **Bind keys** to actions with `cx.bind_keys([...])`.

```rust
use gpui::{actions, div, prelude::*, px, rgb, size, App, Application, Bounds,
           Context, FocusHandle, Focusable, KeyDownEvent, MouseButton, MouseUpEvent,
           Render, SharedString, Window, WindowBounds, WindowOptions};

actions!(counter, [Increment, Reset]);

struct Counter {
    count: i32,
    focus_handle: FocusHandle,
}

impl Counter {
    fn new(cx: &mut Context<Self>) -> Self {
        Self { count: 0, focus_handle: cx.focus_handle() }
    }

    fn increment(&mut self, _: &Increment, _: &mut Window, cx: &mut Context<Self>) {
        self.count += 1;
        cx.notify();
    }

    fn reset(&mut self, _: &Reset, _: &mut Window, cx: &mut Context<Self>) {
        self.count = 0;
        cx.notify();
    }

    fn on_increment_click(&mut self, _: &MouseUpEvent, _: &mut Window, cx: &mut Context<Self>) {
        self.count += 1;
        cx.notify();
    }
}

impl Focusable for Counter {
    fn focus_handle(&self, _: &App) -> FocusHandle {
        self.focus_handle.clone()
    }
}

impl Render for Counter {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex()
            .flex_col()
            .gap_4()
            .bg(rgb(0x2e3440))
            .size(px(400.0))
            .justify_center()
            .items_center()
            .text_xl()
            .text_color(rgb(0xd8dee9))
            .track_focus(&self.focus_handle)
            .on_action(cx.listener(Self::increment))
            .on_action(cx.listener(Self::reset))
            .child(div().text_2xl().child(format!("Count: {}", self.count)))
            .child(
                div()
                    .flex()
                    .flex_row()
                    .gap_3()
                    .child(
                        div()
                            .bg(rgb(0x4c566a))
                            .hover(|style| style.bg(rgb(0x5e81ac)).cursor_pointer())
                            .border_1()
                            .border_color(rgb(0x88c0d0))
                            .rounded_lg()
                            .px_6()
                            .py_3()
                            .child("Increment")
                            .on_mouse_up(MouseButton::Left, cx.listener(Self::on_increment_click)),
                    ),
            )
    }
}

fn main() {
    Application::new().run(|cx: &mut App| {
        let bounds = Bounds::centered(None, size(px(400.), px(300.)), cx);

        cx.bind_keys([
            gpui::KeyBinding::new("space", Increment, None),
            gpui::KeyBinding::new("r", Reset, None),
        ]);

        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(bounds)),
                ..Default::default()
            },
            |_window, cx| cx.new(|_cx| Counter::new(cx)),
        )
        .expect("failed to open window");
    });
}
```

Notes:

- `actions!(counter, [Increment, Reset])` defines unit action structs in a
  `counter` namespace. For actions that carry data, derive `Action` instead —
  see [`state.md`](state.md).
- Handlers always receive `(&Action, &mut Window, &mut Context<Self>)` and must
  call `cx.notify()` after changing state.
- Mouse handlers (`on_mouse_up`, `on_click`, `on_mouse_down`, `on_mouse_move`,
  `on_scroll_wheel`) fire **regardless of focus**.
- Keyboard shortcuts (actions) require the focus chain above and, for immediate
  startup access, focus the view on launch:
  ```rust
  let window = cx.open_window(window_options, |_, cx| cx.new(Counter::new)).unwrap();
  window
      .update(cx, |view, window, cx| {
          window.focus(&view.focus_handle(cx), cx);
          cx.activate(true); // bring the app to the foreground
      })
      .unwrap();
  ```

## Text input (core-gpui style)

Since there is no `input()` element, capture text from `on_key_down`:

```rust
div()
    .on_key_down(cx.listener(|this, event: &KeyDownEvent, _window, cx| {
        if let Some(key_char) = &event.keystroke.key_char {
            if key_char.len() == 1 && !event.keystroke.modifiers.control {
                this.buffer.push_str(key_char);
                cx.notify();
            }
        }
    }))
```

Use actions for special keys (Enter to submit, Backspace) and raw
`on_key_down` for character input.

## Dynamic lists: capture values before closures

When a list has per-item buttons, capture the item's identity as an owned value
**before** the closure. Each `cx.listener(move |…| …)` closure owns its own id;
otherwise every button closes over the last item.

```rust
.children(self.todos.iter().map(|todo| {
    let todo_id = todo.id; // owned copy; the closure takes ownership
    div()
        .child(todo.text.clone())
        .on_mouse_up(MouseButton::Left, cx.listener(move |this, _e, _w, cx| {
            this.delete_todo(todo_id, cx);
        }))
}))
```

## Conditional rendering

`.when(condition, |el| ...)` and `.when_some(option, |el, value| ...)` keep
render trees declarative:

```rust
div()
    .when(self.todos.is_empty(), |el| {
        el.child(div().text_center().text_color(rgb(0x888888)).py_8()
            .child("No todos yet. Press Enter to add one."))
    })
```

## Closure patterns

```rust
// Handler is a method on the view:
cx.listener(Self::my_method)

// Handler captures local state (must be 'static + Clone or owned):
let local_id = item.id;
cx.listener(move |this, _event, _window, cx| this.delete_item(local_id, cx))
```

The `cx.listener(...)` callback signature is
`Fn(&mut T, &Event, &mut Window, &mut Context<T>)` — every listener gets the
window as well.

## API compatibility

Code online and in older tutorials frequently targets the *previous* GPUI API.
Translate as follows (confirmed against current `crates/gpui` source):

| Older (removed)                                                    | Current                                                                                     |
|--------------------------------------------------------------------|---------------------------------------------------------------------------------------------|
| `View<T>` / `Model<T>` / `WeakView` / `WeakModel`                  | `Entity<T>` / `WeakEntity<T>`                                                                |
| `ViewContext<T>` / `ModelContext<T>` / `WindowContext`             | `Context<T>`; window-less ops take `&mut Window` and `&mut App` or `&mut Context<T>`         |
| `cx.new_view(\|cx\| …)` / `cx.new_model(\|_\| …)`                  | `cx.new(\|cx\| …)` (views and models are both `Entity<T>`)                                   |
| `cx.observe(&model, \|_, _, cx\| …)`                               | same, but callback is `FnMut(&mut T, Entity<W>, &mut Context<T>)`                            |
| `cx.listener(\|this, event, cx\| …)` (no window)                    | `cx.listener(\|this, event, window, cx\| …)` (one extra `&mut Window` arg)                   |
| `cx.on_action(cx.listener(…))` on view context                     | element `.on_action(cx.listener(…))` inside `render()`; context-level is `cx.on_action(TypeId, window, handler)` |
| `impl_actions!`                                                     | `#[derive(Clone, PartialEq, serde::Deserialize, schemars::JsonSchema, Action)] #[action(namespace = …)]` |
| `window.focus(&handle)`                                             | `window.focus(&handle, cx)`                                                                 |
| `white()` / `black()` / `hsla(h, s, l, a)`                          | `rgb(0xffffff)` / `rgb(0x000000)` / `Hsla { h, s, l, a }`                                    |
| `blue_500()` palette helpers                                        | `rgb(0x…)` (no palette in core gpui)                                                          |
| `input()` / `textarea()` / `button()` elements                      | `div()` + handlers, or the `ui` crate's widgets                                              |
| `on_scroll`                                                        | `on_scroll_wheel`, or the `list`/`uniform_list` elements for scrolling lists                 |