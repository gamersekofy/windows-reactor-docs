# Getting Started

## Dependencies

```toml
[dependencies]
windows-reactor = { git = "https://github.com/microsoft/windows-rs" }

[build-dependencies]
windows-reactor-setup = { git = "https://github.com/microsoft/windows-rs" }
```

- The crate lives at **`crates/libs/reactor`** inside the windows-rs repo (not
  `crates/windows-reactor`, which doesn't exist).
- `windows-reactor-setup` is a **build-time** helper that stages Windows App
  Runtime files next to your executable.

## Build Script (`build.rs`)

Cargo automatically runs `build.rs` at the project root before compiling.

```rust
// Framework-dependent: ~2.7 MB, requires runtime installed on target machine
fn main() {
    windows_reactor_setup::as_framework_dependent();
}

// Self-contained: ~55 MB, no runtime needed on target machine
fn main() {
    windows_reactor_setup::as_self_contained();
}
```

Framework-dependent apps also need an explicit bootstrap handle held alive for
the lifetime of the process:

```rust
fn main() -> windows_core::Result<()> {
    let _bootstrap_handle = windows_reactor::bootstrap::initialize()?;
    App::new().render(...)
}
```

## Entry Point

```rust
use windows_reactor::*;

fn main() -> Result<()> {
    App::new()
        .title("My App")
        .inner_size(1200.0, 800.0)
        .backdrop(Backdrop::Mica)
        .eager_templated_realization(true)
        .render(shell)
}
```

### `App` builder options

| Method | Description |
|--------|-------------|
| `.title(impl Into<String>)` | Window title |
| `.inner_size(w, h)` | Initial window size in DIPs |
| `.backdrop(Backdrop)` | `Mica`, `Acrylic`, or `Default` |
| `.presenter(PresenterKind)` | Window presenter — see below |
| `.fullscreen(bool)` | Shortcut for `presenter(FullScreen)` / back to `Default` |
| `.eager_templated_realization(bool)` | **See below** |
| `.render(fn)` | Register a render function (see below) |
| `.run(factory)` | Register a full `Component` factory |

### Window types (`PresenterKind`)

Three window presentation modes are supported:

| Variant | Description |
|---------|-------------|
| `PresenterKind::Default` | Standard floating, resizable window (the default) |
| `PresenterKind::FullScreen` | Fullscreen window |
| `PresenterKind::CompactOverlay` | Always-on-top compact window (picture-in-picture style, like a mini-player) |

```rust
App::new().presenter(PresenterKind::CompactOverlay)

// or the fullscreen convenience shortcut:
App::new().fullscreen(true)  // → FullScreen
App::new().fullscreen(false) // → Default
```

### ⚠️ The `render` function signature trap

`App::render()` requires:

```rust
pub fn render<F>(self, f: F) -> Result<()>
where
    F: Fn(&mut RenderCx) -> Element + Send + 'static,
```

It wants a concrete **`Element`**, *not* `impl Into<Element>` — even though the
official intro post and some docs show the latter. The result is error E0271:

```
expected `app` to return `Element`, but it returns `impl Into<Element>`
```

**The fix:** return `Element` and end your builder chain with `.into()`:

```rust
fn app(cx: &mut RenderCx) -> Element {   // ← Element, not impl Into<Element>
    vstack((text_block("Hi"),)).into()   // ← .into() at the end
}
```

### ⚠️ `eager_templated_realization(true)`

WinUI's `ListView`/`GridView` virtualize items — they create rows lazily based
on their measured size. Inside a `NavigationView`, the content area's size is
determined late in the layout pass, and the lazy list may never receive the
signal to realize its items. Result: **the list renders as an empty box** (or
not at all) with zero errors.

Set `.eager_templated_realization(true)` on `App` to force all items to be
created immediately. The gallery app does this. **Always enable it** if you use
templated lists.

## Where to Find API Documentation

The best "documentation" is the crate source:

- **Widgets:** `crates/libs/reactor/src/core/widgets/<widget>.rs` — one file per
  widget containing the struct, all builder methods, and the factory function.
- **Element enum:** `crates/libs/reactor/src/core/element.rs` — the `define_element!`
  macro lists every widget variant.
- **Hooks:** `crates/libs/reactor/src/core/render_context.rs`
- **Samples:** `crates/samples/reactor/` — `gallery`, `minimal`, plus example files.
