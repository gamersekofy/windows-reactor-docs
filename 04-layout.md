# 4. Layout

Reactor uses the WinUI layout system. Panels arrange their children; attached properties such as
`grid_row` and `canvas_left` position a child inside its parent panel.

## Passing children

Every child-taking builder accepts `impl IntoViews`. The supported forms are:

| Form | Example |
| --- | --- |
| `()` | `.children(())` |
| A tuple of 1–16 values | `.children((text, button, border))` |
| `[View; N]` or an array of controls | `.children([a, b, c])` |
| `Vec<View>` | `.children(rows)` |

Tuple members may be different control types because each is converted independently. A
`Vec<View>` is the right choice when the number of children is dynamic:

```rust
use windows_reactor::*;

fn labels(names: &[&str]) -> View {
    let rows: Vec<View> = names
        .iter()
        .map(|name| TextBlock::new().text(*name).into())
        .collect();

    StackPanel::new().spacing(4.0).children(rows).into()
}
```

When children are components or rows that need stable identity, use `keyed_children` with
`KeyedView` values (chapter 2).

## Panels at a glance

| Panel | Arranges children |
| --- | --- |
| `StackPanel` | One after another, vertically or horizontally, with `spacing` |
| `Grid` | In rows and columns defined by `GridLength` |
| `Border` | A single child with padding, background, border, and corner radius |
| `Canvas` | At absolute `canvas_left` / `canvas_top` coordinates |
| `RelativePanel` | By alignment relative to the panel |
| `VariableSizedWrapGrid` | In a wrapping grid with fixed cell size |
| `ScrollView` / `ScrollViewer` | One child with scrolling |
| `Viewbox` | One child, scaled to fit |

`StackPanel` and `Grid` have no `padding`. Wrap content in a `Border` when you need padding.

## Grid

`Grid` is the workhorse for page layout. Define `rows` and `columns`, then attach each child to a
cell:

```rust
use windows_reactor::*;

pub struct LayoutDemo;

impl Component for LayoutDemo {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        Grid::new()
            .rows([
                GridLength::Auto,
                GridLength::Pixel(56.0),
                GridLength::Star(1.0),
            ])
            .columns([GridLength::Pixel(200.0), GridLength::Star(1.0)])
            .row_spacing(8.0)
            .column_spacing(8.0)
            .children((
                TextBlock::new()
                    .text("Header")
                    .font_size(24.0)
                    .grid_row(0)
                    .grid_column(0)
                    .grid_column_span(2),
                Border::new()
                    .padding(Thickness::new(12.0, 8.0, 12.0, 8.0))
                    .background(ThemeBrush::CardBackground)
                    .corner_radius(CornerRadius::uniform(8.0))
                    .content(TextBlock::new().text("Toolbar"))
                    .grid_row(1)
                    .grid_column(0)
                    .grid_column_span(2),
                StackPanel::new()
                    .spacing(4.0)
                    .children((TextBlock::new().text("Sidebar"),))
                    .grid_row(2)
                    .grid_column(0),
                ScrollViewer::new()
                    .content(Border::new().padding(16.0).content(
                        TextBlock::new()
                            .text("A long body that wraps.")
                            .text_wrapping(TextWrapping::Wrap)
                            .text_trimming(TextTrimming::WordEllipsis)
                            .max_lines(3)
                            .line_height(20.0),
                    ))
                    .grid_row(2)
                    .grid_column(1),
            ))
            .into()
    }
}
```

`GridLength` has three shapes:

| Value | Meaning |
| --- | --- |
| `GridLength::Auto` | Size to content |
| `GridLength::Pixel(120.0)` | Fixed device-independent pixels |
| `GridLength::Star(1.0)` | Share remaining space proportionally |
| `GridLength::STAR` | Shortcut for `Star(1.0)` |

`GridLength` also has `.min(value)` and `.max(value)` to bound a star row or column. Row and
column indices are zero-based; spans default to 1.

## StackPanel

```rust
use windows_reactor::*;

pub struct Stacks;

impl Component for Stacks {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(12.0)
            .children((
                StackPanel::new()
                    .orientation(Orientation::Horizontal)
                    .spacing(8.0)
                    .horizontal_alignment(HorizontalAlignment::Stretch)
                    .children((
                        Button::new().content("One"),
                        Button::new().content("Two"),
                    )),
                Border::new()
                    .padding(Thickness::uniform(16.0))
                    .content(StackPanel::new().spacing(4.0).children((
                        TextBlock::new().text("Card"),
                        TextBlock::new().text("Details").opacity(0.6),
                    ))),
            ))
            .into()
    }
}
```

`Orientation::Vertical` is the default. `spacing` only affects the gap between children; it does
not add outer padding.

## Border

`Border` wraps one child and is the standard way to create a card:

| Method | Effect |
| --- | --- |
| `padding` | Space between the border and the child |
| `background` | Fill behind the child (`Color` or `ThemeBrush`) |
| `border_brush` / `border_thickness` | Stroke around the border |
| `corner_radius` | Rounded corners |
| `content` | The single child |

`Thickness::uniform(16.0)`, `Thickness::xy(horizontal, vertical)`, and
`Thickness::new(left, top, right, bottom)` cover the common cases. `CornerRadius::uniform(8.0)`
and `CornerRadius::new(top_left, top_right, bottom_right, bottom_left)` do the same for corners.

## Canvas

`Canvas` positions children at absolute coordinates:

```rust
use windows_reactor::*;

pub struct CanvasDemo;

impl Component for CanvasDemo {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        Canvas::new()
            .height(200.0)
            .children((
                Border::new()
                    .canvas_left(16.0)
                    .canvas_top(16.0)
                    .width(120.0)
                    .height(80.0)
                    .background(Color::rgb(70, 130, 200))
                    .corner_radius(CornerRadius::uniform(8.0)),
                TextBlock::new()
                    .canvas_left(24.0)
                    .canvas_top(24.0)
                    .text("Positioned on a Canvas"),
            ))
            .into()
    }
}
```

Canvas children are not measured by the canvas, so give each one an explicit size or let it size
to content. Layering follows declaration order: later children draw on top.

## Wrapping grid

```rust
use windows_reactor::*;

pub struct WrapDemo;

impl Component for WrapDemo {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        VariableSizedWrapGrid::new()
            .item_width(100.0)
            .item_height(64.0)
            .orientation(Orientation::Horizontal)
            .children((
                Border::new().background(Color::rgb(210, 70, 70)),
                Border::new().background(Color::rgb(70, 170, 110)),
                Border::new().background(Color::rgb(70, 130, 200)),
                Border::new().background(Color::rgb(230, 180, 60)),
            ))
            .into()
    }
}
```

`VariableSizedWrapGrid` gives every cell the same width and height. Use `GridView` instead when
cells contain interactive items with selection (chapter 7).

## Scrolling and scaling

```rust
use windows_reactor::*;

pub struct SizeDemo;

impl Component for SizeDemo {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        ScrollView::new()
            .horizontal_scroll_bar_visibility(ScrollingScrollBarVisibility::Auto)
            .vertical_scroll_bar_visibility(ScrollingScrollBarVisibility::Auto)
            .content(
                Viewbox::new().stretch(Stretch::Uniform).content(
                    TextBlock::new()
                        .text("Scaled text")
                        .font_size(48.0)
                        .min_width(200.0)
                        .max_width(600.0)
                        .margin(Thickness::uniform(8.0)),
                ),
            )
            .into()
    }
}
```

`ScrollView` is the modern WinUI scrolling control and uses `ScrollingScrollBarVisibility`.
`ScrollViewer` is the older control and uses `ScrollBarVisibility`; both accept `.content(...)`.
`Stretch::Uniform` preserves aspect ratio; `UniformToFill` fills and crops.

## RelativePanel

`RelativePanel` aligns each child relative to the panel itself. The available alignment methods
are `relative_align_left`, `relative_align_right`, `relative_align_top`,
`relative_align_bottom`, `relative_align_horizontal_center`, and
`relative_align_vertical_center`:

```rust
use windows_reactor::*;

pub struct RelativeDemo;

impl Component for RelativeDemo {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        RelativePanel::new()
            .height(200.0)
            .children((
                TextBlock::new()
                    .text("Top left")
                    .relative_align_left()
                    .relative_align_top(),
                TextBlock::new()
                    .text("Bottom right")
                    .relative_align_right()
                    .relative_align_bottom(),
            ))
            .into()
    }
}
```

## Methods shared by every control

Every control builder also accepts these declaration methods:

| Category | Methods |
| --- | --- |
| Size | `width`, `height`, `min_width`, `min_height`, `max_width`, `max_height` |
| Spacing | `margin(impl Into<Thickness>)` |
| Alignment | `horizontal_alignment`, `vertical_alignment` |
| Visibility | `opacity` |
| Grid | `grid_row`, `grid_column`, `grid_row_span`, `grid_column_span` |
| Canvas | `canvas_left`, `canvas_top` |
| Focus | Focus, keyboard, and pointer handlers on controls that support them |
| Automation | `automation_name`, `automation_id`, `automation_help_text`, and friends |
| Transitions | `transitions`, `transitions_optional`, `exit_fade`, `exit_transition` |
| Reference | `element_ref` |

Sizes are device-independent pixels. In WinUI, `NaN` means "auto"; other non-finite or negative
values are invalid and can fail when the property is applied. Several control-specific
measurements (font size, padding, border thickness, corner radius) are asserted when the view is
built, so compute dynamic values defensively.

Next: [Styling and Theming](05-styling-and-theming.md) covers brushes, typography, and control
styles.
