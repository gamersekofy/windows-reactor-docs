# Templated Lists (ListView, GridView, FlipView)

The virtualized, templated list widgets — the right tool for scrollable
collections of rich rows.

## Factories

```rust
list_view(items, row_renderer)     // vertical list
grid_view(items, cell_renderer)    // grid of cells
flip_view(items, page_renderer)    // flip/carousel pages
virtual_list(items, row_renderer)  // alias for list_view
```

All return a `TemplatedListBuilder<T>`.

## The Row Renderer

```rust
list_view(contacts, |contact, index| {
    // &Contact, usize → anything that is Into<Element>
    hstack((
        PersonPicture::new()
            .display_name(&name)
            .initials(get_initials(&name))
            .width(40.0)
            .height(40.0),
        vstack((
            text_block(&name).bold(),
            text_block(&contact.email).opacity(0.6),
        ))
        .spacing(2.0),
    ))
    .spacing(12.0)
})
```

The renderer is `impl Fn(&T, usize) -> R` where `R: Into<Element>` — you can
return **any** element tree (vstack/hstack/border/etc.), not just text.

## Builder Methods

| Method | Purpose |
|--------|---------|
| `.with_key_selector(fn(&T) -> String)` | Stable item identity for reconciliation |
| `.selected_index(i32)` | Controlled selection (`-1` = none) |
| `.on_selection_changed(fn(i32))` | Selection callback (receives index) |
| `.selection_mode(SelectionMode)` | `None` / `Single` (default) / `Multiple` / `Extended` |
| `.with_key(name)` | Key for the list element itself |
| `.width(f64)` / `.height(f64)` | Explicit sizing |
| `.build()` | → `Element` |

Example:

```rust
list_view(contacts.clone(), |contact, _idx| {
    text_block(contact.name.clone()).padding(Thickness::uniform(10.0))
})
.with_key_selector(|c| format!("{}-{}", c.id, c.last_name))
.selected_index(selected_index)
.on_selection_changed({
    let set_selected = set_selected.clone();
    move |idx| set_selected.call(idx)
})
.build()              // ← needed before .grid_column() etc.
.grid_column(0)
```

## Selection

- **`-1` means "nothing selected"** — use it as the initial value so the
  detail panel shows a placeholder until the user picks someone.
- The callback receives the new `i32` index. Guard reads with
  `idx >= 0 && (idx as usize) < contacts.len()` before indexing.
- In `Multiple`/`Extended` mode, the callback still only gives the index
  (item-level). For full multi-selection, keep track of a set yourself.

## ⚠️ Height behavior

- **Without `.height()`**: the ListView grows to show *all* items. Simple and
  visually nice for small lists, but no virtualization (all rows realized).
- **With `.height(f64)`**: fixed viewport + virtualization — off-screen rows
  are recycled. Use this for large collections.

Both are valid; pick based on list size.

## ⚠️ `eager_templated_realization` (critical)

See [Getting Started](01-getting-started.md). Without
`.eager_templated_realization(true)` on `App`, a ListView inside a
NavigationView may render as an empty box with zero errors. Symptom: other UI
(header, labels) renders, but the list rows never appear.

## ⚠️ Keyed lists & `contacts.clone()`

`list_view` takes `Vec<T>` **by value** (moves it). If you need the data
elsewhere in the same render (e.g. to compute a label), either:

- `.clone()` the vec when passing to `list_view`, or
- compute derived values *before* the move.

## ⚠️ Known issue: selection state inside `component()`

If a page that owns a ListView's selection state is wrapped in
`component(...)`, the selection callback may fire (visible via `println!`)
but the UI won't re-render. Workaround: keep the state on the root render
function and pass it down as parameters. See
[Architecture](02-architecture.md#the-component-boundary-gotcha-important).

## `GridView` specifics

```rust
grid_view(items, |item, _idx| {
    border(vstack((/* tile content */)))
        .width(150.0)
        .height(80.0)
})
.selection_mode(SelectionMode::Single)
.on_selection_changed(...)
.with_key_selector(|item| item.key.clone())
.build()
```

The gallery's `card_grid` helper is a good reference for a full grid_view
implementation with selection → navigation.
