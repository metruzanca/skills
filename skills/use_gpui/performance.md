# Performance: render, layout, memory, profiling

Optimizing GPUI apps. The render cycle is the mental model:

```
state change → cx.notify() → Render → Layout → Paint → Display
```

GPUI diffs the scene tree and only re-paints what changed — the framework is
fast by default. Optimization is about *not fighting it*: fewer notifications,
cheaper renders, flatter trees, and measuring before tuning.

## Rendering

### Notify only when something actually changed

```rust
fn set_count(&mut self, v: i32, cx: &mut Context<Self>) {
    if self.count == v {
        return; // no-op: don't schedule a re-render
    }
    self.count = v;
    cx.notify();
}
```

Bad patterns that force renders:

- A `loop { cx.notify(); sleep(16ms) }` animation ticker when a timer/change
  event would do.
- Subscriptions that blindly `cx.notify()` on every model touch — use selective
  updates (below).
- `cx.notify()` inside `render()` itself.

### Selective subscription updates

```rust
let _subscription = cx.observe(&state, |this, state, cx| {
    let v = state.read(cx).relevant_field.clone();
    if v != this.cached_value {
        this.cached_value = v;
        cx.notify(); // only when the field you render changed
    }
});
```

### Memoize expensive work in render

If computing something costs real time, cache it keyed by the inputs:

```rust
struct Component {
    cache: Option<(u64, String)>, // (hash of inputs, computed result)
}

fn compute(&mut self, inputs: &[Item]) -> String {
    let hash = hash_inputs(inputs);
    match &self.cache {
        Some((h, cached)) if *h == hash => cached.clone(),
        _ => {
            let result = expensive(inputs);
            self.cache = Some((hash, result.clone()));
            result
        }
    }
}
```

Storing the cache in the struct (recomputed on state change) beats recomputing
inside `render()` every frame.

## Layout

- **Prefer flat trees.** Deeply nested flex containers multiply layout work and
  make diffing costlier. Aim for `header / content / footer` at one level, not
  ten nested `div()`s.
- **Fixed sizes beat dynamic when they don't matter.** `w_full()`/flex sizing
  forces layout measurement; `size(px(..))` on stable regions skips it.
- **Don't read layout in render.** Reading `window.bounds()` during `render()`
  causes layout thrashing. Cache dimensions via `observe_window_bounds`
  (see [`styling.md`](styling.md)) and read the cached value instead.
- **Virtualize long lists.** Rendering thousands of rows every frame is the #1
  list jank source. GPUI ships `list` and `uniform_list` elements in
  `crates/gpui/src/elements/` — a virtualized `uniform_list` only lays out and
  paints visible rows. The constructor API differs across GPUI revisions, so
  read that element's docs in the *pinned* version before using it; don't hand-roll
  `on_scroll_wheel` + offset math.

## Memory

- **Store subscriptions.** A `Subscription` created but not stored dies
  instantly (`observe` callbacks silently stop); creating one inside `render()`
  leaks a new subscription every frame. See [`state.md`](state.md).
- **Avoid circular owner relationships.** An `Entity` owning a strong `Entity`
  of its own parent, or `Arc<Mutex<…>>` cycles, prevent teardown. Prefer
  `WeakEntity` for back-references.
- **Bound collections.** History/log lists grow forever; cap with a
  `VecDeque` + pop-front.
- **Reuse allocations.** For per-frame formatting, keep a scratch `String` in
  the struct and `clear()`/`push_str` into it instead of allocating fresh.

## Batching updates

Multiple state writes in a loop trigger a re-render each:

```rust
// BAD: notify on every item
for item in items {
    self.state.update(cx, |s, cx| { s.add(item); cx.notify(); });
}

// GOOD: one update, one notify
self.state.update(cx, |s, cx| {
    for item in items { s.add(item); }
    cx.notify();
});
```

## Async loading

Show loading state immediately, update once the data arrives — never block the
UI thread on I/O. See [`state.md`](state.md) for the `cx.spawn` /
`.detach()` pattern with `Entity::update` after `await`.

## Profiling

Measure first; optimize what the profiler says, not what you guessed.

```bash
# CPU: flame graph of the app
cargo install flamegraph
cargo flamegraph --bin your-app

# Memory: heap snapshots
valgrind --tool=massif --massif-out-file=massif.out ./target/release/your-app
ms_print massif.out
# or
heaptrack ./target/release/your-app

# macOS allocations
cargo build --release
instruments -t "Allocations" ./target/release/your-app

# micro-benchmarks of hot logic
cargo bench  # with criterion dev-dependency, harness = false
```

Profile **release builds** on target hardware; debug assertions and an unbounded
GPU animating profiles skew results.

## Budgets to aim for

- Frame: 60 FPS → 16.7 ms budget; warn when render+layout exceeds ~10 ms.
- Startup: window visible < 100 ms; fully interactive < 500 ms.
- Memory: stable after initialization; steady growth = leak.

## Anti-pattern checklist

- [ ] Allocating fresh `Vec`s/Strings inside `render()`
- [ ] Deep element nesting (10+ levels)
- [ ] Expensive computation inside `render()`
- [ ] Unnecessary `cx.notify()` (every model touch or timer tick)
- [ ] Subscriptions created at render time / not stored
- [ ] Unbounded collections (history, logs, caches)
- [ ] Reading layout during render
- [ ] Rendering thousands of rows without virtualization
- [ ] Blocking the UI thread on sync I/O

Order of attack: profile → identify (render vs layout vs paint vs your code) →
fix the top item → re-profile. Ship the fix, then move to the next.