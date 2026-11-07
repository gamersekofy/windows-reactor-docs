# Architecture & The Component Model

## The Big Picture

`windows-reactor` brings a React-like model to native Windows:

1. You write **render functions**: `fn(cx: &mut RenderCx) -> Element`
2. The render function is a *pure function of state* — it declares what the UI
   should look like given the current hook values
3. State changes (via hooks) trigger a **re-render**
4. A **reconciler** diffs the old element tree against the new one and applies
   only the minimal changes to the real WinUI controls

## `Element`

`Element` is a sum type (Rust enum) where every variant maps to a WinUI
control class:

```rust
pub enum Element {
    TextBlock(TextBlock),
    Button(Button),
    StackPanel(StackPanel),
    Grid(Grid),
    NavigationView(NavigationView),
    TemplatedList(TemplatedListElement),  // list_view / grid_view / flip_view
    Component(ComponentElement),          // sub-component boundary
    Provider(...),                        // context provider
    ErrorBoundary(...),
    Group(GroupElement),                  // fragments
    Custom(CustomElementHandle),
    Empty,
    // ... ~50 more widget variants
}
```

Widget factories (`text_block(...)`, `button(...)`, `vstack(...)`) return
**builder structs** that implement `Into<Element>`. You convert with:

- `.into()` — idiomatic, used everywhere
- `.build()` — exists on `TemplatedListBuilder` (and some others)

### Universal `Element` methods

These work on any widget **after** it's an `Element` (or on builders that
expose them):

```rust
element.grid_row(0)          // grid placement
element.grid_column(1)
element.grid_row_span(2)
element.grid_column_span(2)
```

> ⚠️ **Gotcha:** `TemplatedListBuilder` does NOT have `.grid_column()`.
> You must call `.build()` first:
> ```rust
> list_view(items, |item, _| text_block(item.clone()))
>     .build()          // → Element
>     .grid_column(0)   // now works
> ```

## `RenderCx` (Render Context)

Passed to every render function. Holds:

- The **hooks vector** (state slots) with a cursor
- A **rerender request** closure
- Ambient **context providers** stack
- **UI thread** info, marshaller handle, window size, DPI

Hooks must be called in a **stable order** across renders — no conditional
hook calls (same rule as React). Violating this panics at runtime with a
"hook called in different order" message.

## The Reconciler

The reconciler (in `core/reconciler.rs`) is the engine that syncs your
declarative tree to real WinUI controls. It:

- **Mounts** new elements (creates the WinUI control, applies modifiers)
- **Updates** existing ones when properties change
- **Unmounts** removed elements
- Uses **keys** (`.with_key(...)`, `.with_key_selector(...)`) to match items
  across renders instead of positional identity
- Optimizes via equality: if old == new element (and no theme bindings), it
  skips the diff entirely

### The `component()` boundary gotcha (important!)

`component(fn, props)` wraps a render function in a `ComponentElement` — a
sub-component with its **own** hook vector and re-render lifecycle. This is
the intended pattern for pages:

```rust
component(pages::settings::settings_page, ())
```

However, in our contacts app, hook-driven state *inside* a `component()` that
owned a `ListView`'s selection did **not** propagate to the UI — the
`on_selection_changed` callback fired (debug prints worked) but the UI never
updated.

**The workaround we used:** lift state up to the **root** render function
(registered via `App::render()`) and pass values + setters down to plain
layout functions as parameters (the "props" pattern). See
[Data & Models → State management patterns](08-data.md).

## File Organization (from our app)

```
src/
├── main.rs              # App::new() + root render fn (owns all hooks/state)
├── models/
│   ├── mod.rs
│   ├── contact.rs       # Contact struct
│   └── address.rs       # Address struct
└── pages/
    ├── mod.rs
    └── contacts.rs      # layout functions, widget helpers, initial data
```

`main.rs` stays thin: declare hooks, call a layout function from a page
module, build the NavigationView. All widget/layout code lives in page
modules.
