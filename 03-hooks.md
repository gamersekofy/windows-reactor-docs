# Hooks

All hooks are methods on `&mut RenderCx`. They must be called in a **stable
order** every render (no conditionals around them — same as React's rules of
hooks).

## State Hooks

### `use_state(initial)` → `(T, SetState<T>)`

```rust
let (count, set_count) = cx.use_state(0_i32);
let (name, set_name) = cx.use_state(String::new());
```

- Requires **`T: Clone + PartialEq`** — `use_state` skips re-rendering if the
  new value equals the old one. Add `#[derive(Clone, PartialEq)]` to your
  structs.
- `SetState<T>` has `.call(value)`: replaces the value, then requests a
  re-render. Calling with an equal value is a no-op.
- `SetState<T>` is **not `Copy`** — `.clone()` it for every closure that needs
  it.

```rust
let (count, set_count) = cx.use_state(0_i32);
button("Increment").on_click(move || set_count.call(count + 1))
```

### `use_reducer(initial)` → `(T, Updater<T>)`

Like `use_state` but the setter takes a function `f(prev) -> next`:

```rust
let (count, updater) = cx.use_reducer(0_i32);
updater.call(|prev| prev + 1);
```

### `use_reducer_fn(reducer, initial)` → `(S, Dispatch<A>)`

Classic Redux-style reducer:

```rust
let (state, dispatch) = cx.use_reducer_fn(
    |state: i32, action: Action| match action {
        Action::Increment => state + 1,
        Action::Decrement => state - 1,
    },
    0,
);
dispatch.call(Action::Increment);
```

### `use_ref(initial)` → `HookRef<T>`

Identity-stable mutable reference, survives re-renders without triggering
them:

```rust
let counter = cx.use_ref(0);
counter.set(counter.get_cloned() + 1);
```

## Derived State

### `use_memo(deps, factory)` → `T`

Recomputes only when dependencies change. `deps` can be any
`PartialEq + Clone + 'static` type (a tuple works great):

```rust
let filtered = cx.use_memo((query.clone(), contacts.clone()), || {
    contacts.iter().filter(|c| c.name.contains(&query)).collect::<Vec<_>>()
});
```

### `use_callback(deps, f)` → `Callback<A>`

Stable function reference, recreated only when deps change:

```rust
let on_click = cx.use_callback((), move |_: ()| { /* do something */ });
```

## Effects

### `use_effect(deps, f)`

Runs after render when deps change. `f` is `FnOnce()`:

```rust
cx.use_effect((), move || {
    println!("Mounted!");
});
```

### `use_effect_with_cleanup(deps, f)`

`f` returns an optional cleanup closure, run before the next effect run and
on unmount:

```rust
cx.use_effect_with_cleanup((), move || {
    let token = some_event_listener();
    Some(move || token.revoke())
});
```

## Context

### `use_context<T>(&Context<T>)` → `T`

Reads a value from the nearest `Provider` up the tree, or the default:

```rust
let theme = cx.use_context(&APP_THEME);
```

## Environment / Host Info

| Hook | Returns | Notes |
|------|---------|-------|
| `use_color_scheme()` | `ColorScheme` | `Light` / `Dark`; re-renders on theme change |
| `use_inner_size()` | `Size` | Window client size in DIPs; re-renders on resize |
| `use_dpi()` | `u32` | Per-monitor DPI |
| `use_ui_marshaller()` | `UiMarshaller` | Marshal work onto the UI thread |

## Async / Thread-Safe State

### `use_async_state(initial)` → `(T, AsyncSetState<T>)`

Like `use_state` but the setter is `Send + Sync` — safe to call from any
thread; writes marshal back to the UI thread automatically:

```rust
let (data, set_data) = cx.use_async_state(Vec::new());
std::thread::spawn(move || {
    let results = fetch_something();
    set_data.call(results); // safe from background thread
});
```

Requires `T: Send + Clone + PartialEq`.

## Type Requirements Cheat Sheet

| Hook | Bound on `T` |
|------|--------------|
| `use_state` | `Clone + PartialEq + 'static` |
| `use_reducer` | `Clone + PartialEq + 'static` |
| `use_reducer_fn` | state `S: Clone + PartialEq`, action `A: 'static` |
| `use_async_state` | `Send + Clone + PartialEq + 'static` |
| `use_memo` | result `T: Clone + 'static`, deps `PartialEq + Clone + 'static` |
| `use_context` | `Clone + 'static` |
