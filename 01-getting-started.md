# 1. Getting Started

This chapter gets a Reactor window on screen and explains the pieces Cargo needs to make that
work.

## Prerequisites

| Requirement | Notes |
| --- | --- |
| Windows 10 version 1809 or newer | Windows 11 recommended |
| Rust 1.95+ with edition 2024 | `rustup update stable` |
| Windows App Runtime | Installed by most Windows App SDK apps; Reactor shows an install prompt if it is missing |
| MSVC toolchain | `x86_64-pc-windows-msvc` or `aarch64-pc-windows-msvc` |

Reactor links WinUI 3 directly. It does not use `windows` crate bindings in your code — the
`windows_reactor` prelude re-exports every control, enum, and helper you need.

## Add the dependency

For the latest published preview:

```toml
[dependencies]
windows-reactor = "0.100"
```

These chapters describe the newer `master` API. To pin exactly the revision they were verified
against:

```toml
[dependencies]
windows-reactor = { git = "https://github.com/microsoft/windows-rs", rev = "9769d429fa2bd3a8b5641443c080aa6f166d342e" }
```

If you pin a git revision, run `cargo update -p windows-reactor` after changing the revision;
Cargo keeps the previous checkout until the lockfile changes.

## Runtime deployment

By default an app is **framework-dependent**: it uses the Windows App Runtime already installed
on the machine. No build script is required. Reactor looks for Windows App Runtime `2.6.4` (or
newer in the same family) at startup and displays an install dialog if it cannot find one.

To ship the runtime with your app, add the setup crate as a build dependency:

```toml
[build-dependencies]
windows-reactor-setup = "0.100"
```

and call it from `build.rs`:

```rust,ignore
fn main() {
    windows_reactor_setup::as_self_contained();
}
```

The build script stages the runtime files next to the executable. Distribute that whole output
directory, not just the `.exe`. Self-contained builds are larger but do not require a system-wide
runtime install.

Keep the setup crate's version aligned with the Reactor version you use. A mismatch produces a
runtime version conflict at startup rather than a compile error.

## The three entry points

| Function | Use it for |
| --- | --- |
| `App::run_component::<C>(input)` | One window driven by one component |
| `App::run_component_with_policy::<C>(input, policy)` | One window with a title, theme, and size |
| `App::run_application::<A>(input)` | An application root that owns multiple windows and tray icons |

All three return `Result<()>`, so `main` can return the error directly or call
`.unwrap()`.

## A first window

```rust
use windows_reactor::*;

struct Hello;

impl Component for Hello {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        context.window_frame(
            "Hello, Reactor",
            StackPanel::new()
                .spacing(8.0)
                .children((TextBlock::new().text("Hello, world."),)),
        )
    }
}

fn main() {
    App::run_component::<Hello>(()).unwrap();
}
```

`window_frame` is a convenience: it sets the window title and wraps your content in a `Grid` with
a `TitleBar` in row 0 and your content in row 1. Use `context.window_title(...)` plus a plain view
if you want to draw your own title area.

`create` runs once. `view` runs whenever state changes. Chapter 2 covers the full lifecycle.

## Setting the window policy

`WindowPolicy` controls the native window before the component is created:

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

fn main() {
    let policy = WindowPolicy::new()
        .title("Counter")
        .theme(WindowTheme::Dark)
        .client_size(420.0, 240.0)
        .minimum_client_size(320.0, 200.0);

    App::run_component_with_policy::<Counter>((), policy).unwrap();
}
```

`WindowPolicy` supports `title`, `theme`, `client_size`, and `minimum_client_size`. For backdrop,
icon, initial position, and placement constraints, use `WindowVisuals` from inside the component
instead — see [Windows and Applications](10-windows-and-applications.md).

## Hide the console window

Add this to the top of `main.rs` when you build a release app:

```rust
#![windows_subsystem = "windows"]
```

Leave it off while developing so `println!` and panic messages remain visible in the terminal.

## Suggested project layout

```text
my-app/
  Cargo.toml
  build.rs            # only for self-contained builds
  src/
    main.rs
    components/
      mod.rs
```

Reactor components are ordinary Rust types. Put each component in its own module and export the
ones `main` needs. A single `main.rs` is fine for small samples.

## Troubleshooting startup

| Symptom | Likely cause |
| --- | --- |
| Install prompt at startup | Windows App Runtime is missing; install it or switch to a self-contained build |
| `STATUS_DLL_NOT_FOUND` before `main` | Self-contained output directory was copied incompletely |
| Linker error mentioning `Microsoft.ui.xaml.dll` | Runtime and app target architectures do not match |
| Panic in `bootstrap` | Installed runtime is older than the version Reactor requires |
| Window flashes and closes | `main` returned an error; remove `#![windows_subsystem = "windows"]` to see it |

Next: [Components](02-components.md) explains what happens between `create`, `update`, and
`view`.
