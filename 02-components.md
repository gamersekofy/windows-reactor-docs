# 2. Components

A Reactor app is a tree of components. Each component owns its state, receives typed messages,
and returns a `View` that describes the controls for its current state. Reactor compares the new
view with the previous one and applies the minimal native changes.

## The `Component` trait

```rust,ignore
pub trait Component: Sized + 'static {
    type Input: Clone + PartialEq + 'static;
    type Message: 'static;

    fn create(input: &Self::Input, context: &ComponentContext<Self>) -> Self;
    fn input_changed(&mut self, _input: &Self::Input, _context: &ComponentContext<Self>) {}
    fn update(&mut self, _message: Self::Message, _context: &ComponentContext<Self>) {}
    fn view(&self, input: &Self::Input, context: &mut ViewContext<Self>) -> View;
}
```

Only `create` and `view` are required. A stateless component can leave `Message = ()` and
`update` unimplemented.

`Input` must be `Clone + PartialEq` because Reactor keeps the latest input and compares it
before calling `input_changed`. `Message` only has to be `'static`; it does not have to be
`Send` unless you send it from a background thread.

## Lifecycle

| Call | When it runs | What to do |
| --- | --- | --- |
| `create` | Once, when the component is first mounted | Initialize state from input |
| `input_changed` | When a parent supplies a new input that differs from the old one | Recompute derived state |
| `update` | When a message arrives | Change state; do not render UI here |
| `view` | After `create`, and after every `update` or accepted input | Describe the current UI |

`update` and `view` run on the UI thread. Keep both short: do blocking work in
`spawn_background` (chapter 11) and mutate state through messages.

`view` takes `&self`. State changes belong in `update`; side effects belong in effects. The
exception is state that is only meaningful to the view, which is usually a sign that it should
be a separate component.

## State lives in fields

Reactor has no hook-based state API. State is a field on the component struct:

```rust
use windows_reactor::*;

struct Counter {
    count: u32,
}

impl Component for Counter {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self { count: 0 }
    }

    fn update(&mut self, _message: (), _context: &ComponentContext<Self>) {
        self.count += 1;
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(8.0)
            .children((
                TextBlock::new().text(format!("Count: {}", self.count)),
                Button::new()
                    .on_click(context.forward())
                    .content("Increment"),
            ))
            .into()
    }
}
```

`context.forward()` works here because `Message` is `()`. When a component has a non-unit
message type, `on_click` needs a `Callback<()>`, so use `context.message(Msg::X)` instead. See
[State and Messages](03-state-and-messages.md).

## Input and `input_changed`

Use input for values that a parent owns: the selected record, a filter string, a theme. Reactor
re-renders the child when input changes, but does not recreate it — local state survives.

```rust
use windows_reactor::*;

#[derive(Clone, PartialEq)]
pub struct GreetingInput {
    pub name: String,
    pub clicks: u32,
}

pub struct Greeting;

impl Component for Greeting {
    type Input = GreetingInput;
    type Message = ();

    fn create(_input: &Self::Input, _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, input: &Self::Input, _context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(4.0)
            .children((
                TextBlock::new()
                    .text(format!("Hello, {}!", input.name))
                    .font_size(20.0),
                format!("You have clicked the button {} times.", input.clicks),
            ))
            .into()
    }
}
```

If a child caches something derived from input, implement `input_changed`:

```rust
use windows_reactor::*;

#[derive(Clone, PartialEq)]
pub struct PanelInput {
    pub title: String,
    pub count: u32,
}

pub struct Panel {
    last_seen: String,
}

impl Component for Panel {
    type Input = PanelInput;
    type Message = ();

    fn create(input: &Self::Input, _context: &ComponentContext<Self>) -> Self {
        Self {
            last_seen: input.title.clone(),
        }
    }

    fn input_changed(&mut self, input: &Self::Input, _context: &ComponentContext<Self>) {
        self.last_seen = input.title.clone();
    }

    fn view(&self, input: &Self::Input, _context: &mut ViewContext<Self>) -> View {
        TextBlock::new()
            .text(format!("{} / {} / {}", input.title, input.count, self.last_seen))
            .into()
    }
}
```

Reactor only calls `input_changed` when the new input compares unequal to the previous one. If
the parent rebuilds an identical value every render, nothing happens.

## Composition

Insert a child component in one of two ways:

- `View::component::<Greeting>(input)` gives the child **positional identity**. It keeps its
  state while this position exists.
- `component::<Greeting>("greeting-keyed", input)` gives the child an **explicit key**. Two nodes
  with different keys never share state.

`View::component` returns a `View`. `component` returns a `ComponentNode`, which converts into a
`View` and can be passed directly to `children`, or wrapped with `.keyed()`.

A complete parent:

```rust
use windows_reactor::*;

#[derive(Clone, PartialEq)]
pub struct GreetingInput {
    pub name: String,
    pub clicks: u32,
}

pub struct Greeting;

impl Component for Greeting {
    type Input = GreetingInput;
    type Message = ();

    fn create(_input: &Self::Input, _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, input: &Self::Input, _context: &mut ViewContext<Self>) -> View {
        TextBlock::new()
            .text(format!("Hello, {}! ({} clicks)", input.name, input.clicks))
            .into()
    }
}

pub struct Parent {
    clicks: u32,
}

impl Component for Parent {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self { clicks: 0 }
    }

    fn update(&mut self, _message: (), _context: &ComponentContext<Self>) {
        self.clicks += 1;
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(8.0)
            .children((
                View::component::<Greeting>(GreetingInput {
                    name: "world".into(),
                    clicks: self.clicks,
                }),
                Button::new()
                    .on_click(context.forward())
                    .content("Click me"),
            ))
            .into()
    }
}
```

## Keys and reconciliation

Reactor keeps a component instance when its position in the tree still contains the same
component type. Two rules matter:

1. **Positional identity** is stable while the surrounding structure is stable. Inserting an
   item at the front of an unkeyed list shifts every later position, so every row receives the
   next row's state.
2. **Explicit keys** decouple identity from position. A keyed node with key `"alice"` keeps its
   state when other rows are inserted, removed, or reordered.

For a list of components or keyed rows, build `KeyedView` values:

```rust
use windows_reactor::*;

pub struct KeyedList {
    items: Vec<String>,
}

impl Component for KeyedList {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            items: vec!["alpha".into(), "beta".into(), "gamma".into()],
        }
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        let rows: Vec<KeyedView> = self
            .items
            .iter()
            .map(|name| keyed(name.clone(), TextBlock::new().text(name)))
            .collect();

        StackPanel::new().spacing(4.0).keyed_children(rows).into()
    }
}
```

Keys can be strings, integers, or `Key` values. Use a stable identifier from the data — a
database id, a file path, a slug — not the row index. `Key` implements `From<&str>`,
`From<String>`, and `From<usize>`/`From<u32>`/`From<u64>`.

Reactor panics when a key is used twice in the same collection. That is a programming error, not
a recoverable condition.

## Context available while rendering

`ComponentContext` is passed to `create`, `input_changed`, and `update`. `ViewContext` is passed
to `view` and gives access to the same messaging helpers plus rendering-only APIs.

| Method | Available on | Purpose |
| --- | --- | --- |
| `input()` | `ComponentContext` | Borrow the current input |
| `sender()` | both | Local sender for messages |
| `completion()` | both | Completion handle used by async work |
| `callback(map)` | both | Adapt an event payload into a message |
| `forward()` | both | Forward a payload identical to `Message` |
| `message(msg)` | both | Callback that sends a clone of `msg` |
| `root()` | both | Element reference to the component root |
| `use_context(&ctx)` | `ViewContext` | Read a provided context value |
| `use_effect(...)` | `ViewContext` | Run work after the view is applied |
| `use_effect_guard(...)` | `ViewContext` | Run work and drop a guard on rerun/unmount |
| `on_window_size/placement/color_scheme` | `ViewContext` | Observe window changes |
| `window_title/visuals/frame` | `ViewContext` | Declare window appearance |
| `open_window`, `activate_window`, `close_window` | `ComponentContext` | Manage secondary windows |
| `spawn_background`, `set_timeout`, `set_local_timeout` | `ComponentContext` | Off-thread work and timers |

All of these are covered in later chapters. The important lifecycle rule is that
`ComponentContext` is available where state changes, and `ViewContext` is available where the
view is described.

Next: [State and Messages](03-state-and-messages.md) turns these callbacks into a working
message flow.
