# Setup: toolchain, project scaffold, structure

Getting a GPUI project compiling and structured.

## Toolchain

GPUI requires **nightly Rust**:

```sh
rustup install nightly
```

Pin it per-project with `rust-toolchain.toml` in the project root:

```toml
[toolchain]
channel = "nightly"
```

## Cargo.toml

GPUI is consumed from the Zed repository (there is no stable crates.io release of core `gpui`):

```toml
[package]
name = "my-gpui-app"
version = "0.1.0"
edition = "2024"

[dependencies]
gpui = { git = "https://github.com/zed-industries/zed" }

# General-purpose additions that GPUI apps commonly need:
anyhow = "1"
log = "0.4"
env_logger = "0.11"

[dev-dependencies]
criterion = "0.5"

[[bench]]
name = "rendering"
harness = false
```

The checkout used by `Cargo.lock` defines which GPUI revision you are pinned to.
If a maintenance window pins an older git SHA, examples in these docs may need
small adjustments — check `crates/gpui` at that SHA.

## Standard imports

```rust
use gpui::{
    actions, div, prelude::*, px, rgb, size, App, Application, Bounds, Context,
    FocusHandle, Focusable, MouseButton, Render, SharedString, Window, WindowBounds,
    WindowOptions,
};
```

## Starter entry point

This skeleton uses the current API: `Application::new().run(...)` with a root
view built through `cx.new(...)` inside `open_window`.

```rust
use gpui::{
    div, prelude::*, px, rgb, size, App, Application, Bounds, Context, Render,
    Window, WindowBounds, WindowOptions,
};

struct MainView;

impl Render for MainView {
    fn render(&mut self, _window: &mut Window, _cx: &mut Context<Self>) -> impl IntoElement {
        div().flex().flex_col().bg(rgb(0x1e1e2e)).child("Hello, GPUI!")
    }
}

fn main() {
    Application::new().run(|cx: &mut App| {
        let bounds = Bounds::centered(None, size(px(800.), px(600.)), cx);
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(bounds)),
                ..Default::default()
            },
            |_window, cx| cx.new(|_cx| MainView),
        )
        .expect("failed to open window");
    })
}
```

Build and run:

```sh
cargo run
```

First compile builds GPUI and all dependencies (minutes on a cold build);
subsequent builds are fast. Incremental rebuilds of `gpui` itself can still be
slow; prefer `cargo build` at a crate boundary over the whole workspace when
iterating on app code.

## Recommended project layout

Small to medium apps:

```
my-gpui-app/
├── Cargo.toml
├── rust-toolchain.toml
├── src/
│   ├── main.rs            # entry point, window setup
│   ├── app.rs             # root view / app-level wiring
│   ├── ui/
│   │   ├── mod.rs
│   │   ├── views/         # high-level views (one file each)
│   │   ├── components/    # reusable presentational components
│   │   └── theme.rs       # theme definitions
│   ├── models/            # Entity-backed state
│   └── services/          # file I/O, HTTP, external integrations
├── examples/
└── tests/
```

Larger apps organize **by feature** instead of by layer:

```
src/
├── features/
│   ├── editor/
│   │   ├── model.rs
│   │   ├── view.rs
│   │   ├── actions.rs
│   │   └── components/
│   ├── sidebar/
│   └── statusbar/
```

Feature folders keep related model, view, actions and components together and
scale better as the team and codebase grow. See [`architecture.md`](architecture.md)
for the full treatment.

## .gitignore

```
/target/
*.swp
*.swo
.DS_Store
```

## Next steps

- First interactive view + window sizing: see [`fundamentals.md`](fundamentals.md).
- Adding state that multiple views share: [`state.md`](state.md).
- Layout flows through `div()` chains; the styling API (spacing, colors,
  flex, grid) is in [`styling.md`](styling.md).