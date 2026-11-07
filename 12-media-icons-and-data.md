# 12. Media, Icons, and Data

This chapter covers the non-text media types: images, icons, date/time values, and drag-and-drop
payloads.

## Images

`Image` displays a raster or SVG image. The source builders differ in what they accept and
whether they return a `Result`:

| Builder | Input | Returns |
| --- | --- | --- |
| `source_file(path)` | Any `AsRef<Path>` | `Result<Image>` |
| `source(uri)` | `Into<Rc<str>>` (file or `ms-appx` URI) | `Result<Image>` |
| `source_optional(value)` | `Option<T>` | `Result<Image>` |
| `source_data(encoded)` | `EncodedImage` | `Image` |
| `source_svg(encoded)` | `EncodedImage` | `Image` |

`source_file` and `source` validate the path/URI when the view is built, so they return a
`Result`. Use `.unwrap()` for a constant asset path, or propagate the error. `source_data` and
`source_svg` cannot fail at construction because they take bytes you already own.

```rust
use std::path::PathBuf;

use windows_reactor::*;

fn asset(name: &str) -> PathBuf {
    std::path::Path::new(env!("CARGO_MANIFEST_DIR"))
        .join("assets")
        .join(name)
}

pub struct Media {
    image_failed: bool,
}

#[derive(Clone)]
pub enum MediaMessage {
    Failed,
    Opened,
}

impl Component for Media {
    type Input = ();
    type Message = MediaMessage;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            image_failed: false,
        }
    }

    fn update(&mut self, message: MediaMessage, _context: &ComponentContext<Self>) {
        match message {
            MediaMessage::Failed => self.image_failed = true,
            MediaMessage::Opened => self.image_failed = false,
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(12.0)
            .children((
                Image::new()
                    .source_file(asset("logo.png"))
                    .unwrap()
                    .width(160.0)
                    .height(90.0)
                    .stretch(Stretch::Uniform)
                    .on_opened(context.message(MediaMessage::Opened))
                    .on_failed(context.message(MediaMessage::Failed)),
                Image::new()
                    .source_data(EncodedImage::from_static(&[0x89, b'P', b'N', b'G']))
                    .width(48.0)
                    .height(48.0),
                Image::new()
                    .source_svg(EncodedImage::from_static(
                        br#"<svg xmlns="http://www.w3.org/2000/svg" width="16" height="16"/>"#,
                    ))
                    .width(48.0)
                    .height(48.0),
                if self.image_failed {
                    "Image failed to load"
                } else {
                    "Image loaded from a file"
                },
            ))
            .into()
    }
}
```

`on_opened` and `on_failed` are unit callbacks; use `context.message(...)` when the component has
a non-unit message type. The image bytes in the example are illustrative — a real PNG or SVG
needs valid contents.

`EncodedImage::new(bytes)` takes an `Arc<[u8]>` for owned data; `EncodedImage::from_static(bytes)`
borrows a `'static` slice and avoids a copy. `ImageSource` has the lower-level constructors
`ImageSource::file`, `ImageSource::uri`, `ImageSource::encoded`, and `ImageSource::svg`.

`Image::stretch` accepts `Stretch::None`, `Fill`, `Uniform`, and `UniformToFill`. Set both
`width`/`height` or let the parent panel size the image; an unconstrained image reports its
natural size.

## Icons

| Type | Use |
| --- | --- |
| `SymbolIcon` | A built-in `Symbol` glyph |
| `FontIcon` | A glyph from a font, with `glyph`, `font_size`, `foreground` |
| `BitmapIcon` | A bitmap or SVG URI, with `show_as_monochrome` |
| `PathIcon` | Vector path data |
| `ImageIcon` | An image source, with the same builders as `Image` |
| `PersonPicture` | Avatar with `display_name` and `initials` |

```rust
use std::path::PathBuf;

use windows_reactor::*;

fn asset(name: &str) -> PathBuf {
    std::path::Path::new(env!("CARGO_MANIFEST_DIR"))
        .join("assets")
        .join(name)
}

pub fn icons() -> View {
    StackPanel::new()
        .orientation(Orientation::Horizontal)
        .spacing(8.0)
        .children((
            SymbolIcon::new().symbol(Symbol::Home),
            FontIcon::new().glyph("\u{E80F}").font_size(20.0),
            BitmapIcon::new()
                .uri_source("ms-appx:///Assets/icon.png")
                .unwrap()
                .show_as_monochrome(true),
            PathIcon::new().data("M 0 0 L 16 0 L 8 16 Z"),
            ImageIcon::new().source_file(asset("icon.png")).unwrap(),
            PersonPicture::new()
                .display_name("Avery Chen")
                .initials("AC")
                .width(48.0)
                .height(48.0),
        ))
        .into()
}
```

`BitmapIcon::uri_source` and `ImageIcon::source_file` return `Result` for the same reason as
`Image`: the URI or path is validated eagerly.

`Icon` is the type used by command bars, navigation items, and `AppBarButton`. Its constructors
cover the same sources:

| Constructor | Returns |
| --- | --- |
| `Icon::symbol(Symbol)` | `Icon` |
| `Icon::font(glyph)` | `Icon` |
| `Icon::path(data)` | `Icon` |
| `Icon::bitmap(uri, monochrome)` | `Result<Icon>` |
| `Icon::image(ImageSource)` | `Icon` |
| `Icon::image_file(path)` | `Result<Icon>` |
| `Icon::image_uri(uri)` | `Result<Icon>` |
| `Icon::image_data(encoded)` | `Icon` |
| `Icon::image_svg(encoded)` | `Icon` |

Use `Icon::symbol(Symbol::Add)` in `CommandBarCommand::button_with_icon` and
`NavigationViewItem::icon`. Use a `Result`-returning constructor only when the source is
dynamic.

## DateTime

`DateTime` is a UTC instant with 100-nanosecond ticks. It is re-exported by `windows_reactor`, so
`use windows_reactor::*` is enough.

| Method | Meaning |
| --- | --- |
| `DateTime::now()` | Current UTC instant |
| `from_ticks`, `from_unix_secs`, `from_unix_millis` | Construct from an epoch value |
| `ticks`, `unix_secs`, `unix_millis`, `unix_nanos` | Convert back |
| `to_local()` | Shift to the machine's local clock |
| `year`, `month`, `day`, `hour`, `minute`, `second`, `milliseconds` | Calendar fields |
| `day_of_week` | Day index (`0` = Sunday) |
| `checked_add(TimeSpan)`, `checked_sub(TimeSpan)` | Fallible arithmetic |
| `saturating_add`, `saturating_sub` | Clamp on overflow |
| `checked_duration_since(other)` | Difference as `Option<TimeSpan>` |
| `DateTime::MIN`, `DateTime::MAX` | Representable range |

`DateTime` implements `Display` as an RFC 3339-style UTC string such as
`2026-11-07T14:30:00Z`. It does not implement `Default`; use `Option<DateTime>` for an unset date.

## TimeSpan

`TimeSpan` is a signed duration in the same tick units:

| Constructor | Unit |
| --- | --- |
| `from_ticks`, `from_micros`, `from_millis` | Small units |
| `from_seconds`, `from_minutes`, `from_hours`, `from_days` | Larger units |

`whole_seconds`, `whole_minutes`, `whole_hours`, and `whole_days` truncate toward zero.
`checked_add`, `checked_sub`, `checked_mul`, `checked_div`, and `checked_neg` return `Option`;
the `saturating_*` variants clamp. `abs`, `signum`, `is_zero`, `is_negative`, and `is_positive`
cover the common predicates. `TimeSpan` also implements `Display`.

```rust
use windows_reactor::*;

pub struct Dates {
    created: DateTime,
}

impl Component for Dates {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            created: DateTime::now(),
        }
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        let local = self.created.to_local();
        let deadline = self
            .created
            .checked_add(TimeSpan::from_hours(24))
            .unwrap_or(DateTime::MAX);
        let since_epoch = self.created.unix_secs();
        let label = format!(
            "{}-{:02}-{:02} {:02}:{:02}",
            local.year(),
            local.month(),
            local.day(),
            local.hour(),
            local.minute()
        );

        StackPanel::new()
            .spacing(4.0)
            .children((
                format!("Created: {local}"),
                label,
                format!("Deadline: {deadline}"),
                format!("Unix seconds: {since_epoch}"),
                format!("Timeout: {}", TimeSpan::from_minutes(90)),
                format!("Day of week: {}", local.day_of_week()),
            ))
            .into()
    }
}
```

`DateTime::now()` reads the system clock, which is an effect only in the sense that it is not a
pure function of state. Call it in `create` or `update`, not on every render, so the displayed
time is stable.

## Drag and drop

Drop support is declared on controls with `drop_policy`. The policy states which data kinds are
accepted and which operation each implies:

| Type | Meaning |
| --- | --- |
| `DragDropPolicy` | `storage_items(action)` and `text(action)` |
| `DragDropAction` | `new(DragDropOperation)` plus `caption(...)` |
| `DragDropOperation` | `Copy`, `Move`, `Link` |
| `DragKind` | `StorageItems`, `Text`, `Unsupported` |
| `DroppedData` | `StorageItems(Vec<DroppedStorageItem>)`, `Text(String)`, `Unsupported` |
| `DroppedStorageItem` | `name` and `path` |

Handlers:

| Handler | Payload |
| --- | --- |
| `on_drag_enter` | `DragKind` |
| `on_drag_over` | `DragKind` |
| `on_drag_leave` | `()` |
| `on_drop` | `DroppedData` |

```rust
use windows_reactor::*;

#[derive(Clone)]
pub enum DropMessage {
    Hover(DragKind),
    Leave,
    Drop(DroppedData),
}

pub struct DragDropDemo {
    hover: Option<DragKind>,
    dropped: Option<String>,
}

impl Component for DragDropDemo {
    type Input = ();
    type Message = DropMessage;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            hover: None,
            dropped: None,
        }
    }

    fn update(&mut self, message: DropMessage, _context: &ComponentContext<Self>) {
        match message {
            DropMessage::Hover(kind) => self.hover = Some(kind),
            DropMessage::Leave => self.hover = None,
            DropMessage::Drop(data) => {
                self.hover = None;
                self.dropped = Some(match data {
                    DroppedData::StorageItems(items) => items
                        .iter()
                        .map(|item| item.name.clone())
                        .collect::<Vec<_>>()
                        .join(", "),
                    DroppedData::Text(text) => text,
                    DroppedData::Unsupported => "Unsupported data".to_string(),
                });
            }
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        let background = match self.hover {
            None => Color::rgb(255, 255, 255),
            Some(DragKind::StorageItems) => Color::rgb(92, 202, 221),
            Some(DragKind::Text) => Color::rgb(155, 219, 90),
            Some(DragKind::Unsupported) => Color::rgb(255, 180, 180),
        };

        Border::new()
            .padding(24.0)
            .margin(24.0)
            .background(background)
            .border_brush(Color::rgb(224, 224, 224))
            .border_thickness(Thickness::uniform(1.5))
            .corner_radius(CornerRadius::uniform(12.0))
            .drop_policy(
                DragDropPolicy::new()
                    .storage_items(
                        DragDropAction::new(DragDropOperation::Link)
                            .caption("Drop to link files"),
                    )
                    .text(
                        DragDropAction::new(DragDropOperation::Copy)
                            .caption("Drop to paste"),
                    ),
            )
            .on_drag_enter(context.callback(DropMessage::Hover))
            .on_drag_over(context.callback(DropMessage::Hover))
            .on_drag_leave(context.message(DropMessage::Leave))
            .on_drop(context.callback(DropMessage::Drop))
            .content(
                TextBlock::new()
                    .text(
                        self.dropped
                            .as_deref()
                            .unwrap_or("Drop files or text here"),
                    )
                    .text_wrapping(TextWrapping::Wrap)
                    .font_size(20.0)
                    .horizontal_alignment(HorizontalAlignment::Center)
                    .vertical_alignment(VerticalAlignment::Center),
            )
            .into()
    }
}
```

`DragKind` reports what the source offers, not what the drop target accepts. The policy decides
whether the cursor shows a copy/link/move affordance. If the policy has no action for a kind,
the control does not accept that kind.

Dropped storage items expose both `name` and `path`. Use `path` for file operations and `name`
for display; both are strings supplied by the shell.

## Formatting and state

- Store `DateTime` as UTC. Convert with `to_local()` only for display.
- Keep `Option<DateTime>` for unset calendar values; `None` round-trips through the pickers.
- Format dates and times with `Display` or explicit field access; there is no locale-aware
  formatter in `windows-time`.
- For durations, prefer `TimeSpan` in state and format it once in `view`.
- The path- and URI-taking image and icon builders validate their argument immediately and
  return `Result`; handle the error where the source is constructed rather than expecting a
  blank control.

Next: [Troubleshooting](13-gotchas.md) collects the errors these APIs produce.
