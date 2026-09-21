# Architecture: structure, layers, state ownership

How to organize GPUI projects so they scale from prototype to product. This
file is deliberately API-light — the guidance is structural. Pair it with
[`state.md`](state.md) for the shape of entities and subscriptions.

## Project structure

Two proven organizations (see [`setup.md`](setup.md) for full trees):

- **Layer-based** (`ui/`, `models/`, `services/`, `domain/`): great for small
  and medium apps; easy to scan.
- **Feature-based** (`features/editor/{model,view,actions,components}`):
  better once a codebase crosses a few thousand lines or a team shares it —
  everything for a feature lives together and can be understood, tested, and
  shipped independently.

## Four-layer architecture

Keep layers depending only *downward*:

```
UI Layer        GPUI views: render + user interactions + layout
Application     Entity-backed models: app state, orchestration
Service         File I/O, network, external APIs (behind traits)
Domain          Pure business logic + types; NO gpui imports
```

```rust
// domain/ — pure logic, zero gpui dependencies
#[derive(Clone, Debug)]
pub struct Document {
    pub content: String,
}

impl Document {
    pub fn word_count(&self) -> usize { self.content.split_whitespace().count() }
}

// services/ — external world behind a trait (swap for mocks in tests)
pub trait FileService: Send + Sync {
    fn read(&self, path: &Path) -> anyhow::Result<String>;
    fn write(&self, path: &Path, content: &str) -> anyhow::Result<()>;
}

// models/ (Application layer) — maps domain + services into reactive state
pub struct DocumentModel {
    document: Document,
    file_service: Arc<dyn FileService>,
    is_modified: bool,
}

impl DocumentModel {
    pub fn update_content(&mut self, content: String) {
        self.document.content = content;
        self.is_modified = true;
    }
}

// ui/ (views) — observes the model
struct DocumentView {
    model: Entity<DocumentModel>,
    _subscription: Subscription,
}
```

The payoff: `Document` and `FileService` are testable without ever opening a
window, and swapping the real file service for a mock takes one type.

## State management architecture

### Unidirectional data flow

```
User Action → handler/action → Entity update → cx.notify() → render
     ↑                                                          ↓
     └──────────  observe/subscribe callbacks  ────────────────┘
```

Role each view method: an action handler changes state, then notifies; `render()`
is a pure description of the current state. Avoid mutating state directly from
`render()`.

### State ownership

- **Single source of truth**: each piece of state lives in exactly one `Entity`.
  Multiple views *observe* it; none own a private copy.
- **Hierarchical ownership**: parents own children's state, children receive
  data + callbacks (see Container/Presenter in [`state.md`](state.md)).
- **Shared app-wide state**: `Global` (theme, settings) — low churn, read-only
  in most views (see [`styling.md`](styling.md)).
- **Local UI state**: keep purely-visual state (hover, open/closed) inside the
  view struct, not in shared entities.

## Separation of concerns

Domain logic, application logic, and UI logic belong in different modules with
no circular imports:

```rust
// domain: pure logic
pub struct Document { /* content, insert() */ }

// application: models that own domain objects + services
pub struct EditorModel { document: Document, cursor: usize }

// ui: views that render EditorModel
pub struct EditorView { model: Entity<EditorModel>, /* Render */ }
```

Rules of thumb:

- A view file should contain rendering, layout, and event *wiring* — not
  business logic.
- An entity (model) should expose mutation methods and be UI-free.
- `services` glue external I/O; nothing above should call the OS directly.

## Testability

Design for tests from the start:

- **Inject dependencies through traits** (`Arc<dyn FileService>`), not concrete
  types.
- Provide a `#[cfg(test)]` mock implementation in the service module.
- Keep `render()` free of side effects so heads-up tests can build a view over
  a known-state entity (see [`testing.md`](testing.md)).

## SOLID, GPUI-flavored

1. **Single responsibility** — one view renders one concern; one entity owns
   one slice of state.
2. **Open/closed** — extend via composition (`div()` + handlers), not by
   mutating shared components.
3. **Liskov** — component variants are swappable (the `ButtonVariant` pattern in
   [`styling.md`](styling.md)); callers shouldn't branch on which variant.
4. **Interface segregation** — small service traits (`FileService`, `ApiClient`)
   over one god-trait.
5. **Dependency inversion** — depend on traits and data, not on concrete views.

Also GPUI-specific: **reactive by default** (observe/subscribe rather than
polling), **explicit dependencies** (pass `Entity` handles or data in rather
than reaching for globals), **type-safe state** (leverage Rust's type system to
make invalid states unrepresentable).

## Scaling strategies

- **Small (<5k LOC)**: flat modules, models in one file, minimal layers.
- **Medium (5–20k LOC)**: feature folders, a `services/` layer, a shared
  component library, shared state utilities.
- **Large (>20k LOC)**: workspace/crate boundaries, plugin hooks, full
  dependency injection, CI-test gate, performance monitoring.

## Plugin / extension systems

At the large-app tier, decouple features with a plugin trait:

```rust
pub trait EditorPlugin: Send + Sync {
    fn name(&self) -> &str;
    fn on_document_saved(&self, path: &Path);
}

pub struct PluginManager {
    plugins: Vec<Box<dyn EditorPlugin>>,
}

impl PluginManager {
    pub fn register(&mut self, plugin: Box<dyn EditorPlugin>) { /* … */ }
}
```

A manager that notifies registered plugins keeps core logic open for extension
without modifying it.

## Architecture review checklist

When reviewing an app's structure:

- [ ] UI / application / service / domain layers exist and depend downward only
- [ ] Component boundaries are clear; no god-views doing everything
- [ ] Consistent state management (unidirectional flow) throughout
- [ ] No prop-drilling through more than a couple of levels (use container/view
      composition instead)
- [ ] Dependencies injectable via traits where I/O or time is involved
- [ ] Global state limited to genuinely app-wide, low-churn values
- [ ] No circular entity references (see [`performance.md`](performance.md))
- [ ] Feature boundaries match how the product/team actually works

## Anti-patterns to avoid

- **God components** — a view that renders, fetches, computes, and handles all
  events. Split it.
- **Prop drilling** — pass a parent view's `Entity` down layers instead of
  threading 6 scalar props.
- **Global mutable state** used as a convenience for view-local data.
- **Copy-paste architecture** — duplicate view families; abstract the repeated
  `div()` pattern into a component recipe.
- **Fighting the borrow checker** — clone values out of `read(cx)` before
  closures instead of holding borrows across `update`/spawn boundaries.