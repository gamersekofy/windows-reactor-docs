# 5. Styling and Theming

Reactor exposes the WinUI theme resources directly. The main decision is whether a value should
follow the user's light/dark setting (`ThemeBrush`) or stay fixed (`Color`).

## Theme brushes versus colors

| Type | Behavior |
| --- | --- |
| `ThemeBrush` | Resolves against the active app theme; changes when the theme changes |
| `Color` | A fixed ARGB value; identical in light and dark mode |

Use `ThemeBrush` for surfaces and text. Use `Color` for data, accents that must not change, or
drawing on a `Canvas`.

### ThemeBrush values

| Brush | Typical use |
| --- | --- |
| `ThemeBrush::Accent` | Accent-colored fills and strokes |
| `ThemeBrush::AccentText` | Text drawn on an accent fill |
| `ThemeBrush::PrimaryText` | Primary body and heading text |
| `ThemeBrush::SolidBackground` | Page background |
| `ThemeBrush::CardBackground` | Card and pane surfaces |
| `ThemeBrush::CardStroke` | Card borders and separators |
| `ThemeBrush::SystemCritical` | Error text and error icons |
| `ThemeBrush::SystemCriticalBackground` | Error banners and badges |

### Colors and shapes

```rust
use windows_reactor::*;

pub fn swatches() -> View {
    StackPanel::new()
        .spacing(8.0)
        .children((
            Border::new()
                .background(Color::rgb(0, 120, 212))
                .height(32.0),
            Border::new()
                .background(Color::argb(128, 255, 0, 0))
                .height(32.0),
            Border::new()
                .background(Color::transparent())
                .border_brush(ThemeBrush::Accent)
                .border_thickness(Thickness::uniform(2.0))
                .height(32.0),
            TextBlock::new().text("Accent").foreground(ThemeBrush::Accent),
            TextBlock::new()
                .text("Critical")
                .foreground(ThemeBrush::SystemCritical),
        ))
        .into()
}
```

`Color::rgb(r, g, b)` is opaque; `Color::argb(a, r, g, b)` takes alpha; `Color::transparent()` is
fully transparent. `Thickness` and `CornerRadius` both implement `From<f64>`, so a uniform value
can be passed directly where either is expected.

## Typography

`TextBlock` is the text primitive. Its commonly used methods are:

| Method | Purpose |
| --- | --- |
| `text` | Content (`impl AsRef<str>`) |
| `font_family` | Font family name |
| `font_size` | Size in DIPs |
| `font_weight` | `FontWeight::NORMAL`, `SEMI_BOLD`, `BOLD`, etc. |
| `foreground` | Brush, usually a `ThemeBrush` |
| `text_wrapping` | `Wrap`, `NoWrap`, `WrapWholeWords` |
| `text_trimming` | `None`, `CharacterEllipsis`, `WordEllipsis`, `Clip` |
| `max_lines` | Clamp visible lines |
| `line_height` | Fixed line height in DIPs |
| `character_spacing` | Extra spacing in thousandths of an em |
| `text_alignment` | `Left`, `Center`, `Right`, `Justify`, and others |
| `padding` | Inner padding for the text block |
| `is_text_selection_enabled` | Allow the user to select the text |

`FontWeight` is a struct with associated constants:
`THIN`, `EXTRA_LIGHT`, `LIGHT`, `SEMI_LIGHT`, `NORMAL`, `MEDIUM`, `SEMI_BOLD`, `BOLD`,
`EXTRA_BOLD`, `BLACK`, and `EXTRA_BLACK`.

```rust
use windows_reactor::*;

pub struct Card;

impl Component for Card {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        Border::new()
            .background(ThemeBrush::CardBackground)
            .border_brush(ThemeBrush::CardStroke)
            .border_thickness(Thickness::uniform(1.0))
            .corner_radius(CornerRadius::new(8.0, 8.0, 0.0, 0.0))
            .padding(Thickness::new(16.0, 12.0, 16.0, 12.0))
            .content(
                StackPanel::new().spacing(4.0).children((
                    TextBlock::new()
                        .text("Card title")
                        .font_family("Segoe UI Variable")
                        .font_size(20.0)
                        .font_weight(FontWeight::SEMI_BOLD)
                        .foreground(ThemeBrush::PrimaryText),
                    TextBlock::new()
                        .text("Secondary line")
                        .font_weight(FontWeight::NORMAL)
                        .foreground(ThemeBrush::PrimaryText)
                        .opacity(0.7)
                        .text_wrapping(TextWrapping::Wrap),
                )),
            )
            .into()
    }
}
```

Using `PrimaryText` plus `opacity(0.7)` keeps secondary text readable in both themes. A hard-coded
gray usually fails in one of them.

## Button styles

`ButtonStyle` selects one of the built-in Button styles:

| Style | Appearance |
| --- | --- |
| `ButtonStyle::Default` | Standard button with a border |
| `ButtonStyle::Accent` | Accent fill; one per surface |
| `ButtonStyle::Subtle` | Borderless until hovered |
| `ButtonStyle::TextLink` | Looks like a hyperlink |

`ButtonStyle` is `#[non_exhaustive]`, so match it with a wildcard arm if you branch on it.

```rust
use windows_reactor::*;

pub fn button_styles() -> View {
    StackPanel::new()
        .spacing(8.0)
        .children((
            Button::new()
                .content("Accent")
                .style(ButtonStyle::Accent),
            Button::new()
                .content("Subtle")
                .style(ButtonStyle::Subtle),
            Button::new()
                .content("Text link")
                .style(ButtonStyle::TextLink),
            Button::new()
                .content("Custom background")
                .resource_overrides(
                    ResourceOverrides::new()
                        .set("ButtonBackground", Color::rgb(32, 32, 32))
                        .set("ButtonBorderBrush", Color::rgb(90, 90, 90)),
                ),
        ))
        .into()
}
```

## Resource overrides

`resource_overrides` replaces named WinUI theme resources on that control and its descendants.
It accepts `Color`, `CornerRadius`, and `Thickness` values through `ResourceValue`:

```rust
use windows_reactor::*;

pub fn custom_button() -> View {
    Button::new()
        .content("Overridden")
        .resource_overrides(
            ResourceOverrides::new()
                .set("ButtonBackground", Color::rgb(24, 24, 24))
                .set("ButtonBorderBrush", Color::rgb(0, 120, 212))
                .set("ControlCornerRadius", CornerRadius::uniform(2.0))
                .set("ButtonBorderThemeThickness", Thickness::uniform(2.0)),
        )
        .into()
}
```

Overrides are local. Prefer them over mutating global resources: they are declarative, removed
when the control unmounts, and cannot leak into unrelated windows.

`resource_overrides` is available on `Button` and `Expander`. For other controls, style through
`background`, `foreground`, `border_brush`, and typography methods.

## Window theme and backdrop

The window's theme is separate from control brushes. Declare it once from `view`:

```rust
use windows_reactor::*;

pub struct ThemedWindow;

impl Component for ThemedWindow {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        context.window_visuals(
            WindowVisuals::new()
                .backdrop(WindowBackdrop::Mica)
                .theme(WindowTheme::Dark)
                .client_size(960.0, 640.0),
        );
        context.window_frame(
            "Themed",
            Border::new()
                .padding(24.0)
                .background(ThemeBrush::SolidBackground)
                .content(TextBlock::new().text("Mica behind a dark window")),
        )
    }
}
```

| Backdrop | Effect |
| --- | --- |
| `WindowBackdrop::None` | Solid background |
| `WindowBackdrop::Mica` | Desktop-tinted material |
| `WindowBackdrop::MicaAlt` | Stronger Mica tint |
| `WindowBackdrop::Acrylic` | Blurred, translucent material |

| Theme | Behavior |
| --- | --- |
| `WindowTheme::System` | Follow the Windows setting (default) |
| `WindowTheme::Light` | Force light |
| `WindowTheme::Dark` | Force dark |

`WindowTheme::System` is the right default for most apps. A forced theme is useful for previews,
media apps, and kiosk surfaces. Changing `WindowVisuals` later updates the existing window; it
does not recreate the component.

`window_visuals` may be declared only once per view. Calling it twice panics with
`window visuals declared more than once`. Set all fields on one builder chain.

## Quick reference

| Need | Reach for |
| --- | --- |
| Page background | `ThemeBrush::SolidBackground` |
| Card surface | `ThemeBrush::CardBackground` + `CardStroke` border |
| Body text | `ThemeBrush::PrimaryText` |
| Error text | `ThemeBrush::SystemCritical` |
| Fixed brand color | `Color::rgb(...)` |
| Rounded card | `Border` + `corner_radius` |
| Accent action | `ButtonStyle::Accent` |
| Per-control theme tweak | `ResourceOverrides` |
| Light/dark window | `WindowVisuals::theme` |
| Material background | `WindowVisuals::backdrop` |

Next: [Input Controls](06-input-controls.md) builds a full form from these primitives.
