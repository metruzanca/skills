# Testing GPUI

`gpui` supports synchronous and async tests through the `#[gpui::test]`
attribute macro and `App::test`. Tests run against `Entity` state directly —
driving *render* from a test is possible but fiddly; prefer asserting on state
and view methods, then smoke-test render for panics.

> Verify the precise `#[gpui::test]` / `App::test` / executor-draining helpers
> against the `crates/gpui/src/test.rs` of your **pinned** GPUI revision before
> leaning on call names that differ between versions. Everything below uses the
> stable core.

## The basic shape

```rust
use gpui::prelude::*;

#[gpui::test]
fn counter_increments() {
    App::test(|cx: &mut App| {
        let state = cx.new(|_cx| CounterState { count: 0 });
        let view = cx.new(|_cx| Counter::new(state.clone(), cx));

        view.update(cx, |view, cx| view.increment(cx));

        state.update(cx, |s, _cx| {
            assert_eq!(s.count, 1);
        });
    });
}
```

- `cx.new(...)` builds entities just like a real app.
- `Entity::update(cx, |value, cx| …)` lets you drive the same methods users
  trigger through events.
- Assert on **state**, not on rendered text, wherever possible.

## Initialization & state-reactivity tests

```rust
#[gpui::test]
fn observes_state_changes() {
    App::test(|cx: &mut App| {
        let state = cx.new(|_cx| CounterState { count: 0 });
        let view = cx.new(|cx| Counter::new(state.clone(), cx));

        // Subscription set up in the constructor fires on notify.
        state.update(cx, |s, cx| { s.count += 1; cx.notify(); });

        view.read(cx).assert_consistent_with(state.read(cx)); // your own helper
    });
}
```

If a test proves a subscription is *not* updating, suspect the "subscription not
stored" bug from [`state.md`](state.md) — the entity updates but the view never
heard about it.

## Interaction and flow tests

Drive the same methods event handlers call:

```rust
#[gpui::test]
fn complete_user_flow() {
    App::test(|cx: &mut App| {
        let state = cx.new(|_cx| TodoState::default());
        let view = cx.new(|cx| TodoList::new(state.clone(), cx));

        view.update(cx, |v, cx| { v.add("buy milk", cx); v.toggle(0, cx); v.delete(0, cx); });

        state.update(cx, |s, _cx| {
            assert!(s.todos.is_empty());
        });
    });
}
```

### Edge cases

```rust
#[gpui::test]
fn empty_state_renders_without_panicking() {
    App::test(|cx: &mut App| {
        let view = cx.new(|cx| MyView::new(cx.new(|_| AppState::default()), cx));
        // Rendering needs a real Window (open_windows + render into it). The
        // exact call shape varies between GPUI revisions — copy the pattern from
        // crates/gpui/src/test.rs of YOUR pinned version rather than inventing it.
        // For most tests, skip render entirely and assert on state directly.
    });
}
```

## Async behavior

GPUI's test runtime executes spawned futures. The exact helper to "park the
runloop until pending tasks finish" appears in `crates/gpui/src/test.rs` in
your pinned revision — read that file rather than guessing a name that drifted.
Preferences that keep async tests robust:

- Injected `fetch`/service traits (see [`architecture.md`](architecture.md)) let
  you hand back an immediate future and avoid real I/O.
- Structure handlers as small methods (`fn on_data(&mut self, data: Data, cx)`)
  so the *logic* after an await is testable without the await.

## Test doubles for services

Prefer trait injection over fakes of gpui internals:

```rust
#[cfg(test)]
struct MockFileService { written: RefCell<Vec<(PathBuf, String)>> }

impl FileService for MockFileService {
    fn read(&self, _: &Path) -> anyhow::Result<String> { Ok(String::new()) }
    fn write(&self, path: &Path, content: &str) -> anyhow::Result<()> {
        self.written.borrow_mut().push((path.into(), content.into()));
        Ok(())
    }
}
```

Then assert on the mock's captured calls instead of the UI.

## Property-based tests

For invariants (e.g. "counter never negative"), use `proptest` wrapping the
same state-level assertions:

```rust
proptest::proptest! {
    #[test]
    fn counter_never_negative(incs in 0usize..100, decs in 0usize..100) {
        App::test(|cx: &mut App| {
            let state = cx.new(|_cx| CounterState { count: 0 });
            state.update(cx, |s, cx| {
                for _ in 0..incs { s.count = s.count.saturating_add(1); }
                for _ in 0..decs { s.count = s.count.saturating_sub(1); }
                cx.notify();
            });
            state.update(cx, |s, _cx| {
                prop_assert!(s.count >= 0);
                Ok(())
            }).unwrap();
        });
    }
}
```

## Benchmarks

With `criterion` as a dev-dependency and `[[bench]] harness = false`, benchmark
the hot path — typically a render or a reducer:

```rust
// benches/render.rs
use criterion::{criterion_group, criterion_main, Criterion};
use gpui::prelude::*;

fn render_bench(c: &mut Criterion) {
    c.bench_function("render list-1000", |b| {
        b.iter(|| {
            App::test(|cx: &mut App| {
                let view = cx.new(|cx| LargeList::new(cx.new(|_| make_state()), cx));
                // view.update(cx, |v, cx| { black_box(v.render(cx)); });
            });
        });
    });
}
criterion_group!(benches, render_bench);
criterion_main!(benches);
```

## Best practices

- Test behavior, not implementation; assert on state and results.
- Name tests by behavior (`empty_state_renders`, `overflow_bounded`), not by
  function.
- Cover edge cases: empty, max values, duplicate ids, rapid repeated actions.
- Keep tests independent — build fresh state per test.
- Factor shared setup into a `test_utils` module.
- Aim for >80% coverage of models/state; views are cheap to smoke-test.
- Run `cargo test` in CI; keep benches out of the normal test run (`cargo bench`).