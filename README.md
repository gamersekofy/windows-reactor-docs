# windows-reactor Knowledge Base

A collection of everything we've learned about the
[`windows-reactor`](https://github.com/microsoft/windows-rs/pull/4479) crate —
a React-like UI library for building native WinUI 3 apps in Rust.

> **Note:** This crate has very sparse official documentation. Everything here
> was discovered through a mix of reading the crate source
> (`crates/libs/reactor` in the windows-rs repo), studying the
> [gallery sample app](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/gallery),
> and educated guessing. Some information may become stale as the crate
> evolves. When in doubt, **read the source** — every widget, hook, and builder
> method lives in the crate itself.

## Table of Contents

1. [Getting Started](01-getting-started.md) — dependencies, build system, entry point
2. [Architecture & The Component Model](02-architecture.md) — Element, RenderCx, reconciler
3. [Hooks](03-hooks.md) — use_state, use_reducer, use_effect, and friends
4. [Widgets Catalog](04-widgets.md) — every widget family and how to use them
5. [Layout & Styling](05-layout.md) — stacking, grids, borders, spacing, theming
6. [Templated Lists](06-lists.md) — ListView, GridView, FlipView, virtualization
7. [Navigation & Dialogs](07-navigation.md) — NavigationView, CommandBar, ContentDialog
8. [Data & Models](08-data.md) — DateTime, optional fields, state patterns
9. [Pitfalls & Gotchas](09-gotchas.md) — every trap we fell into (and you can avoid)

## Cheat Sheet

```rust
// Entry point — render fn MUST return Element, not impl Into<Element>
fn main() -> Result<()> {
    App::new()
        .title("My App")
        .backdrop(Backdrop::Mica)
        .eager_templated_realization(true) // ← required for ListView inside NavView
        .render(shell)
}

// Component render function
fn shell(cx: &mut RenderCx) -> Element {
    let (count, set_count) = cx.use_state(0);
    button(format!("Count: {count}"))
        .on_click(move || set_count.call(count + 1))
        .into() // ← always end with .into()
}
```
