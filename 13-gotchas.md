# 13. Troubleshooting

Most Reactor mistakes surface as one of a handful of compiler errors or as a panic during a view
commit. This chapter lists the common ones with the fix.

## Compiler errors

### `Callback<T>: IntoUnitCallback` is not satisfied

`on_click` and other unit events accept `impl IntoUnitCallback`, which is implemented by
`Callback<()>` — not `Callback<Message>`. If `Message` is an enum, build a unit callback:

```rust
use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    Increment,
}

pub struct Counter {
    count: u32,
}

impl Component for Counter {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self { count: 0 }
    }

    fn update(&mut self, message: Message, _context: &ComponentContext<Self>) {
        match message {
            Message::Increment => self.count += 1,
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        Button::new()
            .on_click(context.message(Message::Increment))
            .content(format!("Count: {}", self.count))
            .into()
    }
}
```

`context.forward()` is only for `Message = ()`, or for events whose payload type equals the
message type.

### `expected View, found TextBlock`

A control is not a `View`. Convert at the boundary that expects `View`:

```rust
use windows_reactor::*;

fn label() -> View {
    TextBlock::new().text("Hello").into()
}
```

Inside `children`, `content`, and `items`, conversion happens automatically; `.into()` is only
needed when the value is returned or stored as a `View`.

### `the trait bound ... is not satisfied` for a payload callback

The control defines the payload type. For example `on_text_changed` sends `Rc<str>`, so a
`Message::Name(String)` variant needs an adapter:

```rust
use std::rc::Rc;

use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    Name(String),
}

pub struct Adapter {
    name: String,
}

impl Component for Adapter {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            name: String::new(),
        }
    }

    fn update(&mut self, message: Message, _context: &ComponentContext<Self>) {
        match message {
            Message::Name(name) => self.name = name,
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        TextBox::new()
            .text(&self.name)
            .on_text_changed(context.callback(|value: Rc<str>| {
                Message::Name(value.to_string())
            }))
            .into()
    }
}
```

Keep the message payload as `Rc<str>` unless you need an owned `String`; the adapter allocates on
every keystroke.

### `ElementRef<A>` cannot be attached to control `B`

`ElementRef<T>` is typed and enforced. Use the control's type:

```rust
use windows_reactor::*;

pub struct Refs {
    input: ElementRef<TextBox>,
}

impl Component for Refs {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            input: ElementRef::new(),
        }
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        TextBox::new().element_ref(&self.input).into()
    }
}
```

`ElementRef` without a type parameter is `ElementRef<AnyElement>` and can be attached to any
control, but it only exposes the generic focus and notification APIs.

### `DateTime` has no `Default`

`DateTime` is a value type with no meaningful zero. Represent an unset date as
`Option<DateTime>`:

```rust
use windows_reactor::*;

pub struct Birthday {
    date: Option<DateTime>,
}

impl Component for Birthday {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self { date: None }
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        CalendarDatePicker::new().date(self.date).into()
    }
}
```

The pickers accept `Option<DateTime>` and report `Option<DateTime>`.

## Match errors from non-exhaustive enums

`ContentDialogResult`, `ContentDialogButton`, `ButtonStyle`, `InfoBarSeverity`,
`TeachingTipPlacementMode`, `ListViewSelectionMode`, and several other enums are
`#[non_exhaustive]`. Always include a wildcard:

```rust
use windows_reactor::*;

pub fn describe(result: ContentDialogResult) -> &'static str {
    match result {
        ContentDialogResult::Primary => "confirmed",
        ContentDialogResult::Secondary => "secondary",
        ContentDialogResult::None => "cancelled",
        _ => "other",
    }
}
```

Omitting `_` compiles until Reactor adds a variant, then breaks downstream code. Add the wildcard
now.

## Runtime panics

### `effect "key" declared more than once`

Effect keys must be unique within a single `view` call. Do not register the same key in two
branches or loop iterations:

```rust
use windows_reactor::*;

pub struct OneEffect {
    value: u32,
}

impl Component for OneEffect {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self { value: 0 }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        context.use_effect("publish-value", self.value, || None);
        TextBlock::new().text(format!("{}", self.value)).into()
    }
}
```

If several effects are dynamic, derive the key from the item (`format!("row-{id}")`) so each key
is unique and stable across renders.

### `window title declared more than once`

`window_title`, `window_visuals`, `on_window_size`, `on_window_placement`, and `on_color_scheme`
may each be declared once per view. `window_frame` already sets the title:

```rust
use windows_reactor::*;

pub struct FrameOnly;

impl Component for FrameOnly {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        // Do not also call context.window_title here.
        context.window_frame("Title", TextBlock::new().text("Content"))
    }
}
```

Set all window visuals on one `WindowVisuals` chain, and all observers once.

### A timer never fires

`ComponentTimer` is `#[must_use]`; dropping it cancels delivery. A temporary like
`let _ = context.set_timeout(...)` is dropped immediately, and even a local
`let _timer = ...` is dropped at the end of `update` — both cancel the timer. Store it in a field:

```rust
use std::time::Duration;

use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    Tick,
}

pub struct TimerHolder {
    timer: Option<ComponentTimer>,
}

impl Component for TimerHolder {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self { timer: None }
    }

    fn update(&mut self, message: Message, context: &ComponentContext<Self>) {
        match message {
            Message::Tick => {
                self.timer =
                    Some(context.set_timeout(Duration::from_secs(1), Message::Tick));
            }
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        Button::new()
            .on_click(context.message(Message::Tick))
            .content("Start")
            .into()
    }
}
```

Calling `timer.cancel()` stops delivery but does not interrupt a running background closure.

### A component loses state when its type changes

A slot that renders one component type in one state and another type in the next state remounts
the child, so local state is lost. This is expected: the two components are different types. If
the slot must keep identity, keep the component type stable and pass the mode as input. If it
genuinely swaps pages, give each page a distinct key so the intent is explicit:

```rust
use windows_reactor::*;

pub struct PageA;
pub struct PageB;

impl Component for PageA {
    type Input = ();
    type Message = ();
    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }
    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        TextBlock::new().text("A").into()
    }
}

impl Component for PageB {
    type Input = ();
    type Message = ();
    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }
    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        TextBlock::new().text("B").into()
    }
}

pub fn page(show_a: bool) -> View {
    let child: View = if show_a {
        View::component::<PageA>(())
    } else {
        View::component::<PageB>(())
    };
    let key = if show_a { "page-a" } else { "page-b" };

    StackPanel::new().keyed_children([keyed(key, child)]).into()
}
```

### Keyed state jumps to the wrong row

Keys must identify the data, not the position. Index keys (`keyed(index, row)`) reproduce the
positional bug that keys are meant to solve. Use a stable id:

```rust
use windows_reactor::*;

pub struct Row {
    pub id: u32,
    pub label: String,
}

pub fn rows(items: &[Row]) -> View {
    StackPanel::new()
        .keyed_children(
            items
                .iter()
                .map(|item| keyed(item.id, TextBlock::new().text(&item.label))),
        )
        .into()
}
```

### `duplicate key` in a collection

Two siblings with the same key panic. This commonly happens when a key is derived from a display
string that is not unique. Include the id in the key:

```rust
use windows_reactor::*;

pub struct Entry {
    pub id: u32,
    pub name: String,
}

pub fn entries(items: &[Entry]) -> View {
    StackPanel::new()
        .keyed_children(
            items
                .iter()
                .map(|item| keyed(item.id, TextBlock::new().text(&item.name))),
        )
        .into()
}
```

### A window size or constraint panics

Window sizes and constraints are validated: width and height must be finite and positive, and
`WindowConstraints` minima must not exceed maxima. Several control-specific measurements
(`font_size`, `line_height`, `padding`, `border_thickness`, `corner_radius`) are checked in the
same way. Compute dynamic values defensively:

```rust
use windows_reactor::*;

pub fn safe_width(value: f64) -> f64 {
    if value.is_finite() && value > 0.0 {
        value
    } else {
        1.0
    }
}

pub fn sized() -> View {
    Border::new().width(safe_width(0.0)).into()
}
```

The same rule applies to `ResourceOverrides::set`: the key must not be empty, and thickness and
radius values must be finite and non-negative.

### The window closes immediately

`App::run_component` returns when the last window closes. A component that calls
`context.close_window()` in `create`, or an application whose `view` declares no windows and no
icons, exits right away. Keep at least one window or icon declared.

If the process exits before any UI appears, remove `#![windows_subsystem = "windows"]` and run
from a terminal to see the startup error.

## Layout and styling surprises

| Symptom | Cause | Fix |
| --- | --- | --- |
| Padding does nothing | `StackPanel` and `Grid` have no `padding` | Wrap in `Border`, or use `TextBlock::padding` |
| Text is clipped | Fixed height or `max_lines` | Remove the height or allow wrapping |
| Child fills the whole grid cell | Default alignment is stretch | Set `horizontal_alignment` / `vertical_alignment` |
| Card has square corners | `corner_radius` on the wrong element | Put it on `Border`, not the child |
| Dark mode text is unreadable | Hard-coded `Color` text | Use `ThemeBrush::PrimaryText` |
| Overrides leak to siblings | Global resource mutation | Use `resource_overrides` on one control |

## Selection payloads

| Control | Payload |
| --- | --- |
| `ListView` | `Option<usize>` |
| `GridView` | `Option<usize>` |
| `FlipView` | `Option<usize>` |
| `ComboBox` | `Option<usize>` |
| `RadioButtons` | `Option<usize>` |
| `TabView` | `Option<usize>` |
| `Pivot` | `Option<usize>` |
| `ListBox` | `Option<Rc<str>>` (tag) |
| `NavigationView` | `Option<Rc<str>>` (tag) |
| `SelectorBar` | `Option<Rc<str>>` (text) |
| `TreeView` | `Rc<str>` (invoked key) |
| `BreadcrumbBar` | `Rc<str>` (clicked text) |

`ListView`, `GridView`, and `FlipView` also expose `on_reordered` with `Vec<Key>`.

## Threading mistakes

- Never block the UI thread in `update` or `view`. `std::thread::sleep`, synchronous HTTP, and
  large file reads freeze the window.
- `spawn_background` requires `Message: Send`. `Rc<str>`, `Rc<RefCell<_>>`, and `Callback` are not
  `Send`; keep them on the UI thread or convert to owned `String`/ids before crossing threads.
- `set_timeout` also requires `Message: Send`; use `set_local_timeout` when the message contains
  UI-only values.
- Dropping a `ComponentTimer` cancels it; dropping a `ComponentTask` does not cancel the worker.
  Call `task.cancel()` explicitly.

## Version and deployment mistakes

| Symptom | Cause |
| --- | --- |
| Build error about edition 2024 | Rust older than 1.95 |
| Install prompt at startup | Windows App Runtime missing; install it or use a self-contained build |
| `windows-reactor-setup` version conflict | Setup crate and Reactor versions differ |
| Self-contained app fails before `main` | Runtime files were not copied next to the executable |
| Linker error about WinUI | App and runtime target architectures differ |

Next: [Samples](14-samples.md) points at working applications that combine these pieces.
