# windows-reactor

Build native Windows apps with Rust. `windows-reactor` is a React-style UI library for
[WinUI 3](https://learn.microsoft.com/windows/apps/winui/winui3/): you describe the UI with
typed Rust builders, keep state in components, and send typed messages when something happens.
Reactor reconciles each new view against the previous one and updates only the controls that
changed.

> **Preview software.** Reactor is published as a preview and its public API may change between
> releases. The API in these chapters is the one on `master` at **0.101.0**.

## Which version are these docs for?

| Source | Version | Notes |
| --- | --- | --- |
| `master` at `9769d42` | `0.101.0` | What these chapters describe |
| [crates.io](https://crates.io/crates/windows-reactor) | `0.100.0` | The latest published release |

The 0.100 release uses the same component model but predates some of the builders documented
here. When in doubt, read the source in
[`crates/libs/reactor`](https://github.com/microsoft/windows-rs/tree/master/crates/libs/reactor).

## Install

```toml
[dependencies]
windows-reactor = "0.100"
```

To track the API in these docs before the next release, pin the git revision:

```toml
[dependencies]
windows-reactor = { git = "https://github.com/microsoft/windows-rs", rev = "9769d429fa2bd3a8b5641443c080aa6f166d342e" }
```

Reactor requires **Rust 1.95 or newer** (edition 2024) and a Windows App Runtime installation.
Framework-dependent apps need no build script: the crate resolves the installed Windows App
Runtime at startup. To ship the runtime with your app, use `windows-reactor-setup` from a build
script — see [Getting Started](01-getting-started.md).

## Smallest useful app

```rust
use windows_reactor::*;

#[derive(Clone, Copy)]
enum Message {
    Increment,
    Reset,
}

struct Counter {
    count: i32,
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
            Message::Reset => self.count = 0,
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        let content = StackPanel::new().spacing(8.0).children((
            format!("Count: {}", self.count),
            Button::new()
                .on_click(context.message(Message::Increment))
                .content("Increment"),
            Button::new()
                .on_click(context.message(Message::Reset))
                .content("Reset"),
        ));
        context.window_frame("Counter", content)
    }
}

fn main() {
    App::run_component::<Counter>(()).unwrap();
}
```

A click sends a message to `update`. `update` changes state, Reactor calls `view` again, and
the changed `TextBlock` is updated in place.

## Chapters

1. [Getting Started](01-getting-started.md) — toolchain, Cargo setup, runtime deployment, and a
   first window.
2. [Components](02-components.md) — the `Component` lifecycle, input, composition, and
   reconciliation.
3. [State and Messages](03-state-and-messages.md) — typed messages, controlled inputs,
   context, and element references.
4. [Layout](04-layout.md) — stacks, grids, borders, canvases, scrolling, and sizing.
5. [Styling and Theming](05-styling-and-theming.md) — theme brushes, colors, typography, button
   styles, and resource overrides.
6. [Input Controls](06-input-controls.md) — text, toggles, sliders, pickers, ratings, and the
   button family.
7. [Collections](07-collections.md) — list and grid controls, keys, selection, and
   virtualization.
8. [Navigation and Shell](08-navigation-and-shell.md) — title bars, navigation views, command
   bars, tabs, and panes.
9. [Dialogs and Overlays](09-dialogs-and-overlays.md) — content dialogs, flyouts, tooltips,
   menus, teaching tips, and info bars.
10. [Windows and Applications](10-windows-and-applications.md) — window configuration,
    observers, secondary windows, shared application state, and tray icons.
11. [Async, Tasks, and Effects](11-async-tasks-and-timers.md) — background work, timers, and
    post-render effects.
12. [Media, Icons, and Data](12-media-icons-and-data.md) — images, icon types, `DateTime` /
    `TimeSpan`, and drag-and-drop.
13. [Troubleshooting](13-gotchas.md) — the compile errors and runtime panics you are most
    likely to hit.
14. [Samples](14-samples.md) — a map of the upstream sample applications.

## Reading the code examples

Every Rust example in these chapters was compiled against the pinned `master` revision. Most are
complete components or complete functions; a few show a single builder chain inside a function
that returns `View`. Two conventions appear throughout:

- A control converts into a `View` with `.into()`. Methods such as `children`, `content`, and
  `items` accept `impl Into<View>` directly, so `.into()` is only needed at a boundary that
  expects `View` — typically a `view` method's return value.
- Event handlers take callbacks produced by the context. `context.message(Msg::X)` is a
  `Callback<()>`; `context.callback(map)` adapts a payload into a message; `context.forward()`
  forwards a payload whose type is exactly the component's message type.

## Useful links

- [Source](https://github.com/microsoft/windows-rs/tree/master/crates/libs/reactor)
- [Samples](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor)
- [API docs on docs.rs](https://docs.rs/windows-reactor)
- [WinUI 3 controls](https://learn.microsoft.com/windows/apps/design/controls/)
