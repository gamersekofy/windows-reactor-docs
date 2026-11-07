# 14. Samples

The upstream repository contains complete applications built with Reactor. They are the best
reference for combining components, windows, and controls in a real codebase.

Clone the repository and run samples from its root:

```sh
git clone https://github.com/microsoft/windows-rs
cd windows-rs
cargo run -p reactor-gallery
```

Each sample has its own `Cargo.toml` and uses workspace dependencies, so run it from the
repository root rather than from inside the sample directory.

## Learn the component model

| Sample | Package | What it demonstrates |
| --- | --- | --- |
| [counter](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/counter) | `reactor-counter` | The smallest component and message loop |
| [component-input](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/component-input) | `reactor-component-input` | Parent input and `input_changed` |
| [controlled](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/controlled) | `reactor-controlled` | Controlled text and value inputs |
| [keyed-list-reorder](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/keyed-list-reorder) | `reactor-keyed-list-reorder` | Stable keys while reordering a list |
| [use-effect](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/use-effect) | `reactor-use-effect` | Effects, cleanup, and focus |
| [element-ref](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/element-ref) | `reactor-element-ref` | Typed `ElementRef` and focus results |
| [context](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/context) | `reactor-context` | `Context<T>`, `provide`, and `use_context` |
| [virtual](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/virtual) | `reactor-virtual` | `VirtualSource` and virtualized rows |
| [async-state](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/async-state) | `reactor-async-state` | Background work returning a message |
| [workbench](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/workbench) | `reactor-workbench` | Keyed components with per-row work |

## Windows and deployment

| Sample | Package | What it demonstrates |
| --- | --- | --- |
| [framework_dependent](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/framework_dependent) | `reactor-framework-dependent` | Uses the installed Windows App Runtime |
| [self_contained](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/self_contained) | `reactor-self-contained` | Stages the runtime with `windows-reactor-setup` |
| [window](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/window) | `reactor-window` | Title, size, backdrop, and theme |
| [window-observers](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/window-observers) | `reactor-window-observers` | Size, placement, and color-scheme observers |
| [window-theme](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/window-theme) | `reactor-window-theme` | Switching light and dark themes |
| [window-placement-registry](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/window-placement-registry) | `reactor-window-placement-registry` | Saving and restoring placement in the registry |
| [secondary-window](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/secondary-window) | `reactor-secondary-window` | `open_window` with independent state |
| [navigation](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/navigation) | `reactor-navigation` | Application root and window navigation |
| [notifyicon](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/notifyicon) | `reactor-notifyicon` | Tray icon, menu, and activation |
| [startup_perf](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/startup_perf) | `reactor-startup-perf` | Startup tracing and measurement |

## Controls and interaction

| Sample | Package | What it demonstrates |
| --- | --- | --- |
| [form](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/form) | `reactor-form` | A form built from the input controls |
| [image-sources](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/image-sources) | `reactor-image-sources` | File, URI, encoded, and SVG image sources |
| [icon](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/icon) | `reactor-icon` | Window title-bar and taskbar icon |
| [lightweight-resources](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/lightweight-resources) | `reactor-lightweight-resources` | Per-control resource overrides |
| [transitions](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/transitions) | `reactor-transitions` | Theme and exit transitions |
| [drag-drop](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/drag-drop) | `reactor-drag-drop` | File and text drop targets |
| [pointer](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/pointer) | `reactor-pointer` | Pointer events and capture |
| [message-box](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/message-box) | `reactor-message-box` | A native Win32 message box via `run_window` |

## Applications and games

| Sample | What it demonstrates |
| --- | --- |
| [gallery](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/gallery) | Every major WinUI control, categorized, with search and a theme toggle |
| [calculator](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/calculator) | A grid of buttons and a small expression state machine |
| [tictactoe](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/tictactoe) | A keyed grid, alternating turns, and win detection |
| [minesweeper](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/minesweeper) | A grid game with pointer input and recursive reveal |
| [dotsweeper](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/dotsweeper) | A visual variant of minesweeper with custom drawing |
| [solitaire](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/solitaire) | A full card game built from standard controls |
| [stacker](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/stacker) | `Canvas`, composition, and a timing loop |
| [explorer](https://github.com/microsoft/windows-rs/tree/master/crates/samples/reactor/explorer) | A file explorer shell with navigation, lists, and a tree |

## What to copy from each sample

| Need | Look at |
| --- | --- |
| A minimal `main.rs` | `counter`, `framework_dependent` |
| Build script for shipping | `self_contained` |
| A shell with a title bar and nav pane | `gallery`, `explorer` |
| Persisting window placement | `window-placement-registry` |
| A tray-only app | `notifyicon` |
| A form with many inputs | `form`, `controlled` |
| A long list that stays responsive | `virtual`, `explorer` |
| A game loop driven by messages | `stacker`, `minesweeper` |
| Saving state across a reorder | `keyed-list-reorder` |
| Effects and focus | `use-effect`, `element-ref` |

## Running a sample

Most samples are binaries:

```sh
cargo run -p reactor-gallery
cargo run -p reactor-solitaire
cargo run -p reactor-stacker
```

The counter sample declares a binary named `component`:

```sh
cargo run -p reactor-counter --bin component
```

If a sample fails at startup with an install prompt, the machine is missing the Windows App
Runtime. Install it, or run the `self_contained` sample to see the build-script setup.

## Repository map

| Path | Contents |
| --- | --- |
| `crates/libs/reactor` | The crate source and internal tests |
| `crates/samples/reactor` | The samples listed above |
| `crates/libs/reactor/setup` | `windows-reactor-setup` |
| `docs/reactor-gallery.png` | Gallery screenshot used by the crate readme |

Back to the [README](README.md), or start from
[Getting Started](01-getting-started.md).
