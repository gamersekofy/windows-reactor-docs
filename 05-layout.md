# Layout & Styling

## The Two Layout Workhorses

### StackPanels: `vstack` / `hstack`

```rust
vstack((
    text_block("Title").font_size(28.0).bold(),
    text_block("Subtitle").opacity(0.6),
))
.spacing(8.0)
.margin(Thickness {
    left: 36.0,
    top: 24.0,
    right: 36.0,
    bottom: 36.0,
})
```

- Children are tuples: `vstack((a, b, c))`. Larger children lists use
  `vstack(vec![a, b, c])`.
- `.spacing(f64)` sets the gap between children.
- `.horizontal_alignment(HorizontalAlignment::Stretch)` makes children fill
  the width.

### ⚠️ StackPanels do NOT support `.padding()`

```rust
// ❌ Runtime warning: "ignoring Padding for StackPanel"
vstack((...)).padding(Thickness::uniform(8.0))

// ✅ Wrap in a border instead
border(vstack((...)))
    .padding(Thickness::uniform(8.0))
```

WinUI's `StackPanel` simply has no Padding property. The crate emits a runtime
warning and silently drops it. `border` does support padding.

## Grids

```rust
grid((
    list_view(items, row).grid_column(0),
    border(detail_panel).grid_column(1),
))
.columns([GridLength::Star(1.0), GridLength::Star(2.0)])
.rows([GridLength::Star(1.0)])
.row_spacing(16.0)
.column_spacing(16.0)
```

### `GridLength`

| Variant | Meaning | Example |
|---------|---------|---------|
| `GridLength::Star(f64)` | Proportional share of remaining space | `Star(1.0)` + `Star(2.0)` → ⅓ / ⅔ split |
| `GridLength::Auto` | Size to content | header rows |
| `GridLength::Pixels(f64)` | Fixed size in DIPs | `Pixels(200.0)` |

### Placement

Children of a grid get placed with `.grid_row()` / `.grid_column()`:

```rust
vstack((...)).grid_row(0).grid_column(1)
```

- These methods live on `Element`, so builders that lack them need `.build()`
  or `.into()` first (e.g. `list_view(...).build().grid_column(0)`).
- Default placement is row 0, column 0. Span with `.grid_row_span()` /
  `.grid_column_span()`.

## `border` — the styling container

`border` is how you add padding, backgrounds, and rounded corners:

```rust
border(
    hstack((
        person_picture,
        vstack((name, email)).spacing(2.0),
    ))
    .spacing(12.0),
)
.padding(Thickness {
    left: 12.0,
    top: 10.0,
    right: 12.0,
    bottom: 10.0,
})
.background(ThemeRef::CardBackground)
.border_brush(ThemeRef::CardStroke)
.border_thickness(Thickness::uniform(1.0))
.corner_radius(8.0)
```

This is the standard "card" pattern used by the gallery.

## Scrolling

```rust
scroll_view(
    vstack((...))
        .spacing(16.0)
        .margin(Thickness { left: 36.0, top: 24.0, right: 36.0, bottom: 36.0 })
        .horizontal_alignment(HorizontalAlignment::Stretch),
)
```

`scroll_view` takes one child element. It's the modern WinUI 3 ScrollView
(use this one over `scroll_viewer` unless you have a reason not to).

## Theming

### Theme references

`ThemeRef` values adapt to light/dark automatically:

- `ThemeRef::SolidBackground`
- `ThemeRef::CardBackground`
- `ThemeRef::CardStroke`
- `ThemeRef::SecondaryText`

Use them for backgrounds/borders/text instead of hardcoded colors:

```rust
text_block("Detail").foreground(ThemeRef::SecondaryText)
border(card).background(ThemeRef::CardBackground)
```

### Runtime theme switching

```rust
let is_dark = matches!(cx.use_color_scheme(), ColorScheme::Dark);

button(if is_dark { "\u{E706}" } else { "\u{E708}" })
    .on_click(move || {
        set_requested_theme(if is_dark {
            RequestedTheme::Light
        } else {
            RequestedTheme::Dark
        });
    })
    .font_family("Segoe MDL2 Assets")
```

## Window-level backdrop

```rust
App::new().backdrop(Backdrop::Mica)  // or Backdrop::Acrylic
```

## Fonts

```rust
.font_family("Segoe UI Variable")  // modern Windows font
```

## Margins vs Padding — quick reference

| Modifier | WinUI property | Works on |
|----------|----------------|----------|
| `.margin(Thickness)` | `Margin` | everything (incl. StackPanel) |
| `.padding(Thickness)` | `Padding` | `border`, buttons, text boxes — **NOT StackPanel** |
