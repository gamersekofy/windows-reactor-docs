# 8. Navigation and Shell

A desktop shell is usually a `Grid` with an auto-sized title bar in row 0 and a star-sized
navigation surface in row 1. Reactor's shell controls slot into that structure.

## Shell layout

| Row | Content |
| --- | --- |
| 0 (`Auto`) | `TitleBar` |
| 1 (`Star`) | `NavigationView`, or a `Grid` that contains the navigation pane and page |

Keep navigation state (`selected` tag, history, pane visibility) in the component and feed it back
into the controls. The controls are declarative; they do not navigate for you.

## TitleBar

`TitleBar` draws a custom title area and owns the back button and pane toggle. Declare the
handlers and the current state:

| Method | Purpose |
| --- | --- |
| `title`, `subtitle` | Text in the title area |
| `preferred_height` | `WindowTitleBarHeight::Tall`, `Standard`, or `Collapsed` |
| `is_back_button_visible`, `is_back_button_enabled` | Back button state |
| `on_back_requested` | Back button clicked |
| `is_pane_toggle_button_visible` | Show the pane toggle |
| `on_pane_toggle_requested` | Pane toggle clicked |
| `content` | Center content, often a search box |
| `left_header`, `right_header` | Extra controls on either side |
| `icon` | Icon shown in the title area |

Use `preferred_height(WindowTitleBarHeight::Tall)` when the bar contains inputs; `Standard` is
designed for text-only bars.

## NavigationView

`NavigationView` owns the pane, the header, and the content area:

| Method | Purpose |
| --- | --- |
| `menu_items` / `keyed_menu_items` | Primary navigation items |
| `footer_menu_items` / `keyed_footer_menu_items` | Footer items, often Settings |
| `pane_title`, `header` | Pane and content titles |
| `pane_display_mode` | `Left`, `LeftCompact`, `LeftMinimal`, `Top`, `Auto` |
| `is_pane_open` | Expanded or compact pane |
| `on_is_pane_open_changed` | Pane state changed (`bool`) |
| `on_selected_tag_changed` | Selection changed (`Option<Rc<str>>`) |
| `is_settings_visible` | Show the built-in Settings item |
| `pane_header`, `pane_footer`, `pane_custom_content` | Custom pane regions |
| `content` | The page area |

`NavigationViewItem` supplies `tag`, `content`, `icon`, `is_selected`, `selects_on_invoked`,
`is_expanded`, and nested `menu_items`. Use `keyed_menu_items` so item identity survives
reordering.

A complete shell:

```rust
use std::rc::Rc;

use windows_reactor::*;

#[derive(Clone)]
pub enum ShellMessage {
    Back,
    Navigate(Option<Rc<str>>),
    PaneOpen(bool),
    TogglePane,
}

pub struct ShellDemo {
    selected: String,
    pane_open: bool,
    history: Vec<String>,
}

impl Component for ShellDemo {
    type Input = ();
    type Message = ShellMessage;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            selected: "home".into(),
            pane_open: true,
            history: Vec::new(),
        }
    }

    fn update(&mut self, message: ShellMessage, _context: &ComponentContext<Self>) {
        match message {
            ShellMessage::Back => {
                if let Some(previous) = self.history.pop() {
                    self.selected = previous;
                }
            }
            ShellMessage::Navigate(Some(tag)) => {
                self.history
                    .push(std::mem::replace(&mut self.selected, tag.to_string()));
            }
            ShellMessage::Navigate(None) => {}
            ShellMessage::PaneOpen(open) => self.pane_open = open,
            ShellMessage::TogglePane => self.pane_open = !self.pane_open,
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        context.window_title("Shell");
        context.window_visuals(WindowVisuals::new().client_size(1100.0, 760.0));

        let nav_item = |tag: &'static str, label: &'static str, icon: Symbol, selected: bool| {
            KeyedView::new(
                tag,
                NavigationViewItem::new()
                    .tag(tag)
                    .is_selected(selected)
                    .selects_on_invoked(true)
                    .content(label)
                    .icon(icon),
            )
        };

        let content: View = Border::new()
            .padding(24.0)
            .content(TextBlock::new().text(format!("Page: {}", self.selected)))
            .into();

        let navigation = NavigationView::new()
            .pane_display_mode(NavigationViewPaneDisplayMode::Left)
            .pane_title("Demo")
            .is_settings_visible(false)
            .is_pane_open(self.pane_open)
            .on_is_pane_open_changed(context.callback(ShellMessage::PaneOpen))
            .on_selected_tag_changed(context.callback(ShellMessage::Navigate))
            .keyed_menu_items([
                nav_item("home", "Home", Symbol::Home, self.selected == "home"),
                nav_item("browse", "Browse", Symbol::Find, self.selected == "browse"),
                nav_item(
                    "settings",
                    "Settings",
                    Symbol::Setting,
                    self.selected == "settings",
                ),
            ])
            .keyed_footer_menu_items([KeyedView::new(
                "about",
                NavigationViewItem::new().tag("about").content("About"),
            )])
            .grid_row(1)
            .content(Grid::new().keyed_children([keyed(self.selected.clone(), content)]));

        let title_bar = TitleBar::new()
            .preferred_height(WindowTitleBarHeight::Tall)
            .title("Demo")
            .subtitle(&self.selected)
            .is_back_button_visible(true)
            .is_back_button_enabled(!self.history.is_empty())
            .is_pane_toggle_button_visible(true)
            .on_back_requested(context.message(ShellMessage::Back))
            .on_pane_toggle_requested(context.message(ShellMessage::TogglePane))
            .grid_row(0)
            .content(TextBox::new().placeholder_text("Search"))
            .right_header(
                Button::new()
                    .on_click(context.message(ShellMessage::TogglePane))
                    .content("Menu"),
            );

        Grid::new()
            .rows([GridLength::Auto, GridLength::Star(1.0)])
            .children((title_bar, navigation))
            .into()
    }
}
```

The page content is keyed by the selected tag. That makes each page a fresh component when the
user switches sections, while returning to a section starts it again. If pages should retain
state across navigation, key them by a constant and pass the page id as input instead.

## CommandBar

`CommandBar` takes primary and secondary command lists. `CommandBarCommand` has four
constructors:

| Constructor | Appearance |
| --- | --- |
| `button(key, label)` | Text button |
| `button_with_icon(key, label, icon)` | Icon and label |
| `disabled(key, label)` | Disabled text button |
| `separator(key)` | Divider |

Commands report their `Key` to the callback passed to `owned_commands`, or to
`on_click`-style handlers if you use the keyed builder forms.

## MenuBar and Menu

A menu bar is a row of drop-down menus. Each `MenuBarItem` has a title and a `Menu`:

```rust
use std::rc::Rc;

use windows_reactor::*;

#[derive(Clone)]
pub enum ToolbarMessage {
    Command(String),
    Menu(Key),
}

pub struct Toolbars {
    last_command: String,
}

impl Component for Toolbars {
    type Input = ();
    type Message = ToolbarMessage;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            last_command: "(none)".into(),
        }
    }

    fn update(&mut self, message: ToolbarMessage, _context: &ComponentContext<Self>) {
        match message {
            ToolbarMessage::Command(command) => self.last_command = command,
            ToolbarMessage::Menu(key) => {
                self.last_command = key.as_str().unwrap_or("unknown").to_string();
            }
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        let menu_callback = context.callback(ToolbarMessage::Menu);
        StackPanel::new()
            .spacing(12.0)
            .children((
                CommandBar::new().owned_commands(
                    [
                        CommandBarCommand::button_with_icon("add", "Add", Symbol::Add),
                        CommandBarCommand::button_with_icon("edit", "Edit", Symbol::Edit),
                        CommandBarCommand::separator("separator"),
                        CommandBarCommand::disabled("delete", "Delete"),
                    ],
                    [
                        CommandBarCommand::button("select-all", "Select All"),
                        CommandBarCommand::button("share", "Share"),
                    ],
                    context.callback(|key: Key| {
                        ToolbarMessage::Command(key.as_str().unwrap_or("unknown").to_string())
                    }),
                ),
                MenuBar::new().items((
                    MenuBarItem::new().title("File").menu(Menu::new(
                        [
                            MenuItem::item("new", "New"),
                            MenuItem::item("open", "Open"),
                            MenuItem::separator("file-separator"),
                            MenuItem::disabled("save-as", "Save As"),
                        ],
                        menu_callback.clone(),
                    )),
                    MenuBarItem::new().title("Edit").menu(Menu::new(
                        [
                            MenuItem::item("undo", "Undo"),
                            MenuItem::submenu(
                                "copy-special",
                                "Copy Special",
                                [
                                    MenuItem::item("copy-plain", "Plain text"),
                                    MenuItem::item("copy-markdown", "Markdown"),
                                ],
                            ),
                        ],
                        menu_callback,
                    )),
                )),
                BreadcrumbBar::new()
                    .items_source(["Home", "Documents", "Reports", "Q4"])
                    .on_item_clicked(context.callback(|value: Rc<str>| {
                        ToolbarMessage::Command(value.to_string())
                    })),
                TextBlock::new().text(format!("Last command: {}", self.last_command)),
            ))
            .into()
    }
}
```

`MenuItem` constructors are `item(key, label)`, `submenu(key, label, items)`, `separator(key)`,
and `disabled(key, label)`. The `Menu` callback receives the invoked item's `Key`.

`BreadcrumbBar::on_item_clicked` reports the clicked item's text as `Rc<str>`. It does not carry a
key, so keep the displayed labels stable or map them back to ids yourself.

## Tabs, selector bars, and pivots

`TabView` is the full tab control: closable tabs, an add button, reordering, and selection.
`SelectorBar` is a lightweight segmented switcher. `Pivot` is the touch-oriented page switcher.

```rust
use std::rc::Rc;

use windows_reactor::*;

#[derive(Clone)]
pub enum TabMessage {
    Select(Option<usize>),
    Close(Key),
    Add,
}

pub struct Tabs {
    selected: Option<usize>,
}

impl Component for Tabs {
    type Input = ();
    type Message = TabMessage;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            selected: Some(0),
        }
    }

    fn update(&mut self, message: TabMessage, _context: &ComponentContext<Self>) {
        match message {
            TabMessage::Select(index) => self.selected = index,
            TabMessage::Close(_) | TabMessage::Add => {}
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(12.0)
            .children((
                TabView::new()
                    .is_add_tab_button_visible(true)
                    .can_reorder_tabs(true)
                    .selected_index(self.selected)
                    .on_selection_changed(context.callback(TabMessage::Select))
                    .on_close_requested(context.callback(TabMessage::Close))
                    .on_add_tab_button_click(context.message(TabMessage::Add))
                    .keyed_tab_items([
                        KeyedView::new(
                            "tab-1",
                            TabViewItem::new()
                                .header("First")
                                .tag("first")
                                .content("First tab content"),
                        ),
                        KeyedView::new(
                            "tab-2",
                            TabViewItem::new()
                                .header("Second")
                                .tag("second")
                                .is_closable(true)
                                .content("Second tab content"),
                        ),
                    ]),
                SelectorBar::new()
                    .on_selected_text_changed(context.callback(|value: Option<Rc<str>>| {
                        TabMessage::Select(value.map(|_| 0))
                    }))
                    .keyed_items([
                        KeyedView::new(
                            "recent",
                            SelectorBarItem::new().text("Recent").is_selected(true),
                        ),
                        KeyedView::new("shared", SelectorBarItem::new().text("Shared")),
                        KeyedView::new(
                            "favorites",
                            SelectorBarItem::new()
                                .text("Favorites")
                                .icon(Symbol::Favorite),
                        ),
                    ]),
                Pivot::new()
                    .title("Pivot")
                    .selected_index(self.selected)
                    .on_selection_changed(context.callback(TabMessage::Select))
                    .items([
                        PivotItem::new().header("Overview").content("Overview content"),
                        PivotItem::new().header("Details").content("Details content"),
                    ]),
                SplitView::new()
                    .is_pane_open(true)
                    .open_pane_length(180.0)
                    .display_mode(SplitViewDisplayMode::Inline)
                    .pane(Border::new().padding(16.0).content("Pane"))
                    .content(Border::new().padding(16.0).content("Main content")),
                Expander::new()
                    .header("Advanced")
                    .is_expanded(false)
                    .on_is_expanded_changed(context.callback(|_| TabMessage::Add))
                    .content(TextBlock::new().text("Hidden until expanded")),
            ))
            .into()
    }
}
```

Notes:

- `TabView::selected_index` is `Option<usize>`; `on_close_requested` reports the closed tab's
  `Key`, so translating a key to an index is the app's job.
- `SelectorBar::on_selected_text_changed` reports `Option<Rc<str>>`. Give each item stable text or
  key items and map the text back to an id.
- `Pivot` has the same `Option<usize>` selection shape as `TabView`.
- `SplitView::on_pane_closed` reports `bool`. `display_mode` accepts `Inline`, `CompactInline`,
  `Overlay`, and `CompactOverlay`.
- `Expander::on_is_expanded_changed` reports `bool`.

## Cross-control payload reference

| Control | Event | Payload |
| --- | --- | --- |
| `TitleBar` | `on_back_requested`, `on_pane_toggle_requested` | `()` |
| `NavigationView` | `on_selected_tag_changed` | `Option<Rc<str>>` |
| `NavigationView` | `on_is_pane_open_changed`, `on_display_mode_changed` | `bool`, `NavigationViewDisplayMode` |
| `CommandBar` | command callback | `Key` |
| `Menu` | callback | `Key` |
| `MenuBar` | item selection | `Key` (via each menu's callback) |
| `BreadcrumbBar` | `on_item_clicked` | `Rc<str>` |
| `TabView` | `on_selection_changed` | `Option<usize>` |
| `TabView` | `on_close_requested` | `Key` |
| `TabView` | `on_reordered` | `Vec<Key>` |
| `TabView` | `on_add_tab_button_click` | `()` |
| `SelectorBar` | `on_selected_text_changed` | `Option<Rc<str>>` |
| `Pivot` | `on_selection_changed` | `Option<usize>` |
| `SplitView` | `on_pane_closed` | `bool` |
| `Expander` | `on_is_expanded_changed` | `bool` |

Next: [Dialogs and Overlays](09-dialogs-and-overlays.md) adds temporary surfaces above the shell.
