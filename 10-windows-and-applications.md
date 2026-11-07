# 10. Windows and Applications

A component normally fills one window. An `Application` can declare several windows and
notification icons that share one state root.

## Declaring window appearance

Inside `view`, call these methods on `ViewContext`:

| Method | Effect |
| --- | --- |
| `window_title(text)` | Sets the native window title |
| `window_visuals(visuals)` | Sets backdrop, theme, size, constraints, icon, and initial placement |
| `window_frame(title, content)` | Sets the title and wraps content in a `TitleBar` + `Border` grid |

Each may be declared only once per render. `window_visuals` replaces the whole visual
declaration, so build one chain.

`WindowVisuals` fields:

| Builder | Meaning |
| --- | --- |
| `backdrop(WindowBackdrop)` | `None`, `Mica`, `MicaAlt`, `Acrylic` |
| `theme(WindowTheme)` | `System`, `Light`, `Dark` |
| `client_size(width, height)` | Client size in DIPs, including a custom title bar |
| `constraints(WindowConstraints)` | Optional min/max width and height |
| `icon(path)` | Window icon file |
| `initial_position(ScreenPoint)` | Outer top-left in physical screen pixels |
| `initial_placement(WindowPlacement)` | Restore outer bounds and maximized state |

`WindowPolicy`, used with `App::run_component_with_policy` and `open_window_with_policy`, covers
the subset known before the component exists: `title`, `theme`, `client_size`, and
`minimum_client_size`. Prefer `WindowVisuals` for everything else.

Restoring a saved placement:

```rust
use windows_reactor::*;

pub fn restored_visuals() -> WindowVisuals {
    WindowVisuals::new()
        .initial_position(ScreenPoint::new(120, 80))
        .initial_placement(WindowPlacement {
            x: 100,
            y: 60,
            width: 1280,
            height: 800,
            maximized: false,
        })
}
```

`initial_placement` overrides `initial_position` and `client_size`. Both are read only from the
initial declaration; changing them in a later render does not move or resize the window. Load a
saved placement synchronously before the window opens, not from an asynchronous update.

## Observing the window

`ViewContext` exposes three observers:

| Method | Payload | Fires |
| --- | --- | --- |
| `on_window_size` | `WindowSize` (`width`, `height` in DIPs) | Client size changes |
| `on_window_placement` | `WindowPlacement` (`x`, `y`, `width`, `height`, `maximized`) | Restored outer bounds change; minimized state excluded |
| `on_color_scheme` | `ColorScheme` (`Light`, `Dark`) | Current scheme on subscribe, then changes |

Each observer may be declared only once per view. Declaring the same observer twice panics with
`window size observer declared more than once` (or the corresponding message). Replacing the
callback does not replay the current value; remove and re-add the observer to start a new
subscription.

A window that reports its own geometry:

```rust
use windows_reactor::*;

#[derive(Clone)]
pub enum WindowMessage {
    Resized(WindowSize),
    Moved(WindowPlacement),
    Scheme(ColorScheme),
}

pub struct ObserverWindow {
    size: WindowSize,
    placement: Option<WindowPlacement>,
    scheme: ColorScheme,
}

impl Component for ObserverWindow {
    type Input = ();
    type Message = WindowMessage;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            size: WindowSize::default(),
            placement: None,
            scheme: ColorScheme::Light,
        }
    }

    fn update(&mut self, message: WindowMessage, _context: &ComponentContext<Self>) {
        match message {
            WindowMessage::Resized(size) => self.size = size,
            WindowMessage::Moved(placement) => self.placement = Some(placement),
            WindowMessage::Scheme(scheme) => self.scheme = scheme,
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        context.window_title("Window observatory");
        context.window_visuals(
            WindowVisuals::new()
                .backdrop(WindowBackdrop::Mica)
                .theme(WindowTheme::System)
                .client_size(800.0, 600.0)
                .constraints(WindowConstraints {
                    min_width: Some(400.0),
                    min_height: Some(300.0),
                    max_width: Some(1400.0),
                    max_height: Some(1000.0),
                }),
        );
        context.on_window_size(context.callback(WindowMessage::Resized));
        context.on_window_placement(context.callback(WindowMessage::Moved));
        context.on_color_scheme(context.callback(WindowMessage::Scheme));

        let geometry = self.placement.map_or_else(
            || "not moved".to_string(),
            |placement| {
                format!(
                    "{}x{} at ({}, {})",
                    placement.width, placement.height, placement.x, placement.y
                )
            },
        );

        context.window_frame(
            "Window observatory",
            Border::new().padding(24.0).content(
                StackPanel::new().spacing(8.0).children((
                    format!("Client size: {} x {}", self.size.width, self.size.height),
                    format!("Placement: {geometry}"),
                    format!("Color scheme: {:?}", self.scheme),
                )),
            ),
        )
    }
}
```

Persist `WindowPlacement` in the `Moved` handler. Because placement notifications are coalesced
and exclude minimized state, save the latest value rather than trying to infer a final state at
shutdown.

## Secondary windows

`ComponentContext` can open independent windows:

| Method | Result |
| --- | --- |
| `open_window::<W>(input)` | `true` if the request was accepted |
| `open_window_with_policy::<W>(input, policy)` | Same, with a `WindowPolicy` |
| `activate_window()` | Brings the current window forward |
| `close_window()` | Closes the current window |

A secondary window is another component. It has its own state, its own view, and its own
`window_title` / `window_visuals` declarations:

```rust
use windows_reactor::*;

#[derive(Clone)]
pub enum ChildMessage {
    Activate,
    Close,
}

pub struct ChildWindow {
    id: u32,
}

impl Component for ChildWindow {
    type Input = u32;
    type Message = ChildMessage;

    fn create(input: &Self::Input, _context: &ComponentContext<Self>) -> Self {
        Self { id: *input }
    }

    fn update(&mut self, message: ChildMessage, context: &ComponentContext<Self>) {
        match message {
            ChildMessage::Activate => {
                let _ = context.activate_window();
            }
            ChildMessage::Close => {
                let _ = context.close_window();
            }
        }
    }

    fn view(&self, _input: &Self::Input, context: &mut ViewContext<Self>) -> View {
        context.window_title(format!("Child window #{}", self.id));
        context.window_visuals(WindowVisuals::new().client_size(360.0, 220.0));
        Border::new()
            .padding(24.0)
            .content(
                StackPanel::new().spacing(8.0).children((
                    TextBlock::new().text(format!("Independent window #{}", self.id)),
                    Button::new()
                        .content("Activate this window")
                        .on_click(context.message(ChildMessage::Activate)),
                    Button::new()
                        .content("Close this window")
                        .on_click(context.message(ChildMessage::Close)),
                )),
            )
            .into()
    }
}
```

Closing a secondary window drops its component and its state. Opening it again creates a fresh
component from the input. If you need shared state across windows, keep it in
`App::run_application` (next section) or in an external store, not in window-local fields.

## Application roots

`App::run_application` runs one root component that owns windows and icons. The root is not
rendered into a window; its `view` returns an `ApplicationView` declaration:

```rust
use windows_reactor::*;

#[derive(Clone, PartialEq)]
pub struct WorkspaceInput {
    pub name: &'static str,
    pub dark: bool,
    pub on_theme: Callback<()>,
    pub on_open: Callback<()>,
}

pub struct Workspace;

impl Component for Workspace {
    type Input = WorkspaceInput;
    type Message = ();

    fn create(_input: &Self::Input, _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, input: &Self::Input, context: &mut ViewContext<Self>) -> View {
        context.window_title(input.name);
        context.window_visuals(WindowVisuals::new().theme(if input.dark {
            WindowTheme::Dark
        } else {
            WindowTheme::Light
        }));
        Border::new()
            .padding(24.0)
            .content(
                StackPanel::new().spacing(8.0).children((
                    TextBlock::new().text(input.name).font_size(24.0),
                    Button::new()
                        .content("Toggle theme")
                        .on_click(input.on_theme.clone()),
                    Button::new()
                        .content("Open secondary workspace")
                        .on_click(input.on_open.clone()),
                )),
            )
            .into()
    }
}

#[derive(Clone)]
pub enum AppMessage {
    Open(&'static str),
    ToggleTheme,
}

pub struct Workspaces {
    dark: bool,
}

impl Application for Workspaces {
    type Input = ();
    type Message = AppMessage;

    fn create(_input: &(), context: &ApplicationContext<Self>) -> Self {
        context.show_window("primary");
        Self { dark: false }
    }

    fn update(&mut self, message: AppMessage, context: &ApplicationContext<Self>) {
        match message {
            AppMessage::Open(key) => context.show_window(key),
            AppMessage::ToggleTheme => self.dark = !self.dark,
        }
    }

    fn view(&self, _input: &(), context: &ApplicationViewContext<Self>) -> ApplicationView {
        let input = |name, other| WorkspaceInput {
            name,
            dark: self.dark,
            on_theme: context.message(AppMessage::ToggleTheme),
            on_open: context.message(AppMessage::Open(other)),
        };

        ApplicationView::new()
            .window::<Workspace>("primary", input("Primary workspace", "secondary"))
            .window::<Workspace>("secondary", input("Secondary workspace", "primary"))
    }
}
```

Rules for application roots:

- `Application::view` runs on every application update. It must declare **all** resources: a
  window omitted from the declaration is closed, and an icon omitted is removed.
- `ApplicationView::window::<C>(key, input)` declares a reopenable window. Declaration alone does
  not open it.
- An open window receives updated input without being recreated. Closing it drops its component
  state; `show_window` creates a new component next time.
- `ApplicationContext::show_window(key)` opens, restores, or activates a declared window. Requests
  are applied after the next declaration is committed. An unknown key is an application error.
- `ApplicationContext::exit()` closes windows, menus, and notification icons.
- `ApplicationViewContext` is a narrower context: it can create callbacks and messages but cannot
  issue `show_window` or `exit`.

The application exits when no user windows and no declared notification icons remain, and pending
lifecycle work has finished.

## Notification icons

`ApplicationView::notify_icon(key, icon)` declares a tray icon. `NotifyIcon` takes an `.ico` path,
a tooltip, an activation callback that receives `ScreenPoint`, and an optional menu:

```rust
use windows_reactor::*;

#[derive(Clone)]
pub enum TrayMessage {
    Open,
    ToggleIcon,
    Exit,
    Menu(Key),
}

pub struct TrayApp {
    icon: bool,
}

impl Application for TrayApp {
    type Input = ();
    type Message = TrayMessage;

    fn create(_input: &(), _context: &ApplicationContext<Self>) -> Self {
        Self { icon: true }
    }

    fn update(&mut self, message: TrayMessage, context: &ApplicationContext<Self>) {
        match message {
            TrayMessage::Open => context.show_window("main"),
            TrayMessage::ToggleIcon => self.icon = !self.icon,
            TrayMessage::Exit => context.exit(),
            TrayMessage::Menu(key) => match key.as_str() {
                Some("open") => context.show_window("main"),
                Some("exit") => context.exit(),
                _ => {}
            },
        }
    }

    fn view(&self, _input: &(), context: &ApplicationViewContext<Self>) -> ApplicationView {
        let view = ApplicationView::new().window::<TrayWindow>(
            "main",
            TrayWindowInput {
                icon: self.icon,
                on_toggle: context.message(TrayMessage::ToggleIcon),
                on_exit: context.message(TrayMessage::Exit),
            },
        );

        if self.icon {
            view.notify_icon(
                "tray",
                NotifyIcon::new(concat!(env!("CARGO_MANIFEST_DIR"), "\\..\\icon\\icon.ico"))
                    .tooltip("Open the window")
                    .on_activate(context.callback(|_: ScreenPoint| TrayMessage::Open))
                    .menu(Menu::new(
                        [
                            MenuItem::item("open", "Open"),
                            MenuItem::separator("separator"),
                            MenuItem::item("exit", "Exit"),
                        ],
                        context.callback(TrayMessage::Menu),
                    )),
            )
        } else {
            view
        }
    }
}

#[derive(Clone, PartialEq)]
pub struct TrayWindowInput {
    pub icon: bool,
    pub on_toggle: Callback<()>,
    pub on_exit: Callback<()>,
}

pub struct TrayWindow;

impl Component for TrayWindow {
    type Input = TrayWindowInput;
    type Message = ();

    fn create(_input: &Self::Input, _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, input: &Self::Input, context: &mut ViewContext<Self>) -> View {
        context.window_frame(
            "Tray application",
            StackPanel::new().spacing(8.0).children((
                "This window is independent of the notification icon.",
                Button::new()
                    .on_click(input.on_toggle.clone())
                    .content(if input.icon {
                        "Remove notification icon"
                    } else {
                        "Add notification icon"
                    }),
                Button::new()
                    .on_click(input.on_exit.clone())
                    .content("Exit application"),
            )),
        )
    }
}
```

A notification icon does not keep a window open. The app can keep running with only the icon and
no windows; `show_window` recreates the window from its declaration.

The icon path must exist and be a valid `.ico`. A missing file panics when the icon is applied,
like other contract violations.

## Window vs application at a glance

| Need | Use |
| --- | --- |
| One window | `App::run_component` |
| One window with a title/theme/size | `App::run_component_with_policy` |
| Independent windows with separate state | `open_window` from a component |
| Shared state across windows | `App::run_application` |
| Tray icon | `ApplicationView::notify_icon` |
| Remember size and position | `on_window_placement` + `initial_placement` |

Next: [Async, Tasks, and Effects](11-async-tasks-and-timers.md) covers work that outlives a single
render.
