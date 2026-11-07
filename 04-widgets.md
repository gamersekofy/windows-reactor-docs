# Widgets Catalog

Widgets are organized in `crates/libs/reactor/src/core/widgets/<name>.rs` —
one file per widget containing the struct, builder methods, and factory
function. Everything is re-exported at the crate root, so
`use windows_reactor::*;` gets you all of them.

## Universal Pattern

```rust
factory(args).modifier1().modifier2()...into()
```

Every widget chain ends with `.into()` (or is consumed by a parent widget's
children tuple, which calls `.into()` implicitly).

## Layout Widgets

| Factory | Purpose |
|---------|---------|
| `vstack((...))` | Vertical StackPanel (children top-to-bottom) |
| `hstack((...))` | Horizontal StackPanel (children left-to-right) |
| `stack_panel(orientation)` | StackPanel with explicit orientation |
| `grid((...))` | Grid with `.rows([...])` / `.columns([...])` |
| `border(child)` | Styling container: padding, background, corner radius |
| `scroll_view(content)` | Modern ScrollView wrapper |
| `scroll_viewer(child)` | Classic ScrollViewer |
| `canvas((...))` | Absolute positioning (`.canvas_left()`, `.canvas_top()`, `.canvas_z_index()`) |
| `relative_panel((...))` | Relative positioning (`relative_align_*`) |
| `split_view(pane, content)` | Master/detail split |
| `viewbox(child)` | Scale content to fit |

> ⚠️ **StackPanels have no `.padding()`** — WinUI's `StackPanel` lacks a
> Padding property. You get the runtime warning
> `set_prop: ignoring Padding for StackPanel` and the padding is silently
> dropped. **Wrap in `border(...).padding(...)` instead** — see
> [Layout & Styling](05-layout.md).

## Text & Display

| Factory | Notes |
|---------|-------|
| `text_block(text)` | `.font_size()`, `.bold()`, `.semibold()`, `.italic()`, `.underline()`, `.opacity()`, `.wrap()` |
| `text_box(text)` | Text input; `.header("Label")`, `.multiline()` for notes |
| `rich_edit_box(text)` | Rich text input |
| `rich_text_block(text)` | Rich text display |
| `image(uri)` | Image from URI |
| `person_picture()` | Avatar; see below |
| `progress_bar(value)` / `progress_ring()` | Progress indicators |
| `info_bar(...)` / `info_badge(...)` | Status messages |
| `shape(kind)` | Rectangles, ellipses, etc. |

### `PersonPicture`

```rust
PersonPicture::new()
    .display_name("Lubbes Billi") // drives initials if not overridden
    .initials(get_initials(&name)) // ← takes a String parameter!
    .width(40.0)
    .height(40.0)
```

A helper to derive initials from a name:

```rust
fn get_initials(name: &str) -> String {
    name.split_whitespace()
        .filter_map(|word| word.chars().next())
        .map(|c| c.to_uppercase().to_string())
        .collect::<String>()
}
```

## Input Widgets

| Factory | Notes |
|---------|-------|
| `button(label)` | `.on_click(closure)` |
| `check_box(label)` | `.is_checked(bool)`, `.on_checked(...)` |
| `toggle_switch(label)` | On/off switch |
| `toggle_button(content)` | Toggleable button |
| `radio_button(label)` / `radio_buttons(items)` | Radio selection |
| `slider(value)` | Numeric slider |
| `number_box(value)` | Numeric input |
| `combo_box(items)` | Dropdown |
| `list_box(items)` | Single-selection list (non-virtualized) |
| `auto_suggest_box(text)` | Search box with suggestions |
| `date_picker()` / `time_picker()` | Date / time pickers |
| `calendar_date_picker()` / `calendar_view()` | Calendar widgets |
| `rating_control(value)` | Star rating |
| `color_picker()` | Color selection |
| `password_box(...)` | Password input |

## Actions & Toolbars

| Factory | Notes |
|---------|-------|
| `command_bar(commands)` | Toolbar of commands — see [Navigation](07-navigation.md) |
| `menu_bar(items)` | Menu bar |
| `split_button(content)` / `drop_down_button(content)` | Split buttons |
| `repeat_button(content)` | Auto-repeating button |
| `hyperlink_button(text)` | Hyperlink-style button |
| `expander(content)` | Collapsible section (`.header("...")`) |
| `title_bar(...)` | Custom title bar (`.content()`, `.footer()`, `.tall(true)`) |
| `breadcrumb_bar(items)` | Breadcrumb navigation |

## Navigation & Containers

| Factory | Notes |
|---------|-------|
| `navigation_view(items, content)` | NavView with pane — see [Navigation](07-navigation.md) |
| `tab_view(tabs)` | Tab strip |
| `pivot(items)` | Pivot control |
| `tree_view(nodes)` | Tree hierarchy |
| `selector_bar(items)` | Segmented selection |
| `teaching_tip(...)` | In-app teaching popup |
| `content_dialog(...)` | Modal dialog — see [Navigation](07-navigation.md) |

## Virtualized Lists

`list_view`, `grid_view`, `flip_view`, `virtual_list` (alias for list_view).
These are the power tools — see [Templated Lists](06-lists.md).

## Modifiers (the styling language)

Widgets accept modifier methods that map to WinUI properties:

**Sizing & layout:**
`.width(f64)`, `.height(f64)`, `.min_width()`, `.max_width()`, `.min_height()`,
`.max_height()`, `.horizontal_alignment(HorizontalAlignment::*)`,
`.vertical_alignment(VerticalAlignment::*)`, `.margin(Thickness)`,
`.padding(Thickness)`

**Appearance:**
`.background(color | ThemeRef)`, `.foreground(color | ThemeRef)`,
`.opacity(f64)`, `.corner_radius(f64)`, `.border_brush(color)`,
`.border_thickness(Thickness)`, `.font_family("Segoe UI Variable")`

**Text:**
`.font_size(f64)`, `.bold()`, `.semibold()`, `.italic()`, `.underline()`,
`.strikethrough()`, `.wrap()`

**Grid placement (on Element or grid-aware builders):**
`.grid_row(i32)`, `.grid_column(i32)`, `.grid_row_span(i32)`,
`.grid_column_span(i32)`

**Container spacing:**
`.spacing(f64)` (stacks), `.row_spacing(f64)`, `.column_spacing(f64)` (grids)

**Accessibility:**
`.automation_name(...)`, `.automation_id(...)`, `.help_text(...)`,
`.heading_level(...)`, `.keyboard_accelerator(...)`

## `Thickness`

```rust
Thickness::uniform(8.0)                     // all sides equal
Thickness { left: 8.0, top: 12.0, right: 8.0, bottom: 12.0 }
```

## `Color`

```rust
Color { a: 255, r: 255, g: 0, b: 0 }   // RGBA, 0-255 per channel
```
