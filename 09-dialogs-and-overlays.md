# 9. Dialogs and Overlays

Overlays are declarative. You keep a `bool` in state, attach the overlay to a control (or place it
in the view), and clear the flag when the overlay closes.

## ContentDialog

`ContentDialog` is attached to the control that opens it with `.content_dialog(...)`. The dialog
itself carries the title, body, buttons, and default focus:

| Method | Purpose |
| --- | --- |
| `title`, `content` | Header and body |
| `primary_button_text`, `secondary_button_text`, `close_button_text` | Button labels |
| `is_primary_button_enabled`, `is_secondary_button_enabled` | Button enablement |
| `default_button` | `ContentDialogButton::Primary`, `Secondary`, `Close`, `None` |
| `is_open` | Whether the dialog is showing |
| `on_closed` | Receives `ContentDialogResult` |

```rust
use windows_reactor::*;

#[derive(Clone)]
pub enum OverlayMessage {
    OpenDialog,
    DialogClosed(ContentDialogResult),
}

pub struct SaveDialog {
    open: bool,
    result: Option<ContentDialogResult>,
}

impl Component for SaveDialog {
    type Input = ();
    type Message = OverlayMessage;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            open: false,
            result: None,
        }
    }

    fn update(&mut self, message: OverlayMessage, _context: &ComponentContext<Self>) {
        match message {
            OverlayMessage::OpenDialog => self.open = true,
            OverlayMessage::DialogClosed(result) => {
                self.open = false;
                self.result = Some(result);
            }
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        let result_text = match self.result {
            None => "No result",
            Some(ContentDialogResult::Primary) => "Saved",
            Some(ContentDialogResult::Secondary) => "Discarded",
            Some(ContentDialogResult::None) => "Cancelled",
            Some(_) => "Other",
        };

        StackPanel::new()
            .spacing(8.0)
            .children((
                Button::new()
                    .content("Close document")
                    .on_click(context.message(OverlayMessage::OpenDialog))
                    .content_dialog(
                        ContentDialog::new()
                            .title("Save changes?")
                            .content("You have unsaved changes.")
                            .primary_button_text("Save")
                            .secondary_button_text("Discard")
                            .close_button_text("Cancel")
                            .default_button(ContentDialogButton::Primary)
                            .is_open(self.open)
                            .on_closed(context.callback(OverlayMessage::DialogClosed)),
                    ),
                format!("Result: {result_text}"),
            ))
            .into()
    }
}
```

`ContentDialogResult` is `#[non_exhaustive]`, so the wildcard arm is required. The same is true for
`ContentDialogButton`, `InfoBarSeverity`, `ButtonStyle`, and several other enums. Omitting the
wildcard produces a compile error when Reactor adds a variant.

Only one content dialog can be open at a time in a window. If you need sequential dialogs, close
one before setting the next `is_open` to true.

## Flyouts

A flyout is a light-dismiss surface anchored to a control. Use the shorthand for text:

```rust
use windows_reactor::*;

pub fn simple_flyout() -> View {
    Button::new().content("Simple flyout").flyout("Flyout text").into()
}
```

For rich content, build a `Flyout` and attach it with `flyout_with`:

```rust
use windows_reactor::*;

pub fn rich_flyout() -> View {
    Button::new()
        .content("Rich flyout")
        .flyout_with(
            Flyout::rich(
                StackPanel::new().spacing(4.0).children((
                    TextBlock::new()
                        .text("Flyout title")
                        .font_weight(FontWeight::BOLD),
                    TextBlock::new().text("Flyout body").opacity(0.7),
                )),
            )
            .placement(FlyoutPlacement::Bottom),
        )
        .into()
}
```

`FlyoutPlacement` covers `Auto`, `Top`, `Bottom`, `Left`, `Right`, `Full`, and the edge-aligned
variants (`TopEdgeAlignedLeft`, `BottomEdgeAlignedRight`, and so on). `Auto` lets WinUI choose a
side that fits on screen.

## Tooltips

`Tooltip` follows the same text/rich split:

```rust
use windows_reactor::*;

pub fn tooltip_button() -> View {
    Button::new()
        .content("Hover me")
        .tooltip_with(Tooltip::text("Helpful hint").placement(TooltipPlacement::Top))
        .into()
}
```

`TooltipPlacement` has `Top`, `Bottom`, `Left`, `Right`, and `Mouse`. `Mouse` follows the pointer;
the others anchor to the control. Use `.tooltip("text")` for the short form.

Tooltips supplement, but do not replace, accessible names. An icon-only button still needs an
`automation_name`; a tooltip alone is not announced consistently by every screen reader.

## Context menus

Attach a `Menu` to a control with `.menu(...)`. The menu opens on right-click (or the keyboard
menu key):

```rust
use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    Menu(Key),
}

pub struct ContextMenuDemo;

impl Component for ContextMenuDemo {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn update(&mut self, _message: Message, _context: &ComponentContext<Self>) {}

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        Button::new()
            .content("Right-click me")
            .menu(Menu::new(
                [
                    MenuItem::item("rename", "Rename"),
                    MenuItem::item("duplicate", "Duplicate"),
                    MenuItem::separator("menu-separator"),
                    MenuItem::item("delete", "Delete"),
                ],
                context.callback(Message::Menu),
            ))
            .into()
    }
}
```

`MenuItem::item`, `submenu`, `separator`, and `disabled` build the menu entries. The callback
receives the invoked item's `Key`.

## Command bar flyout

`CommandBarFlyout` combines a small command bar with a menu. It is useful for contextual actions
on a card or list row:

```rust
use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    Command(Key),
}

pub struct CommandFlyoutDemo;

impl Component for CommandFlyoutDemo {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn update(&mut self, _message: Message, _context: &ComponentContext<Self>) {}

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        Button::new()
            .content("More")
            .command_bar_flyout(CommandBarFlyout::new(
                [CommandBarCommand::button_with_icon(
                    "share",
                    "Share",
                    Symbol::Share,
                )],
                [CommandBarCommand::button("delete", "Delete")],
                context.callback(Message::Command),
            ))
            .into()
    }
}
```

`CommandBarFlyout::new(primary, secondary, callback)` takes the same `CommandBarCommand` values as
`CommandBar`. The callback receives the invoked command's `Key`.

## TeachingTip

`TeachingTip` is a regular control placed in the view tree. It is not modal; it can be light
dismissed and it participates in layout:

```rust
use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    Show,
    Action,
    Closed,
}

pub struct TipDemo {
    open: bool,
    accepted: bool,
}

impl Component for TipDemo {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            open: false,
            accepted: false,
        }
    }

    fn update(&mut self, message: Message, _context: &ComponentContext<Self>) {
        match message {
            Message::Show => self.open = true,
            Message::Action => {
                self.accepted = true;
                self.open = false;
            }
            Message::Closed => self.open = false,
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(8.0)
            .children((
                Button::new()
                    .content("Show tip")
                    .on_click(context.message(Message::Show)),
                TeachingTip::new()
                    .title("Did you know?")
                    .subtitle("Tips are regular controls and can appear anywhere.")
                    .is_open(self.open)
                    .preferred_placement(TeachingTipPlacementMode::Bottom)
                    .action_button_content("Got it")
                    .close_button_content("Dismiss")
                    .on_action_button_click(context.message(Message::Action))
                    .on_closed(context.message(Message::Closed)),
                TextBlock::new().text(format!("accepted = {}", self.accepted)),
            ))
            .into()
    }
}
```

`preferred_placement` accepts the `TeachingTipPlacementMode` variants: `Auto`, `Top`, `Bottom`,
`Left`, `Right`, `Center`, and the corner-aligned variants (`TopLeft`, `BottomRight`, and so on).
`is_light_dismiss_enabled` controls whether clicking elsewhere closes the tip.

## InfoBar

`InfoBar` is a non-modal banner for status, warnings, and errors. It stays in flow and can be
closable:

```rust
use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    Close,
}

pub struct Banner {
    open: bool,
}

impl Component for Banner {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self { open: true }
    }

    fn update(&mut self, message: Message, _context: &ComponentContext<Self>) {
        match message {
            Message::Close => self.open = false,
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        InfoBar::new()
            .title("Saved")
            .message("Your changes were stored.")
            .severity(InfoBarSeverity::Success)
            .is_open(self.open)
            .is_closable(true)
            .on_closed(context.message(Message::Close))
            .into()
    }
}
```

`InfoBarSeverity` is `Informational`, `Success`, `Warning`, or `Error` (non-exhaustive). Use
`Informational` for neutral messages, `Warning` for recoverable problems, and `Error` for failed
actions. `is_closable(false)` turns the banner into a permanent status strip.

## Choosing an overlay

| Need | Use |
| --- | --- |
| Blocking decision with buttons | `ContentDialog` |
| Small contextual surface | `Flyout` |
| Hint on hover | `Tooltip` |
| Right-click actions | `Menu` |
| Contextual command bar | `CommandBarFlyout` |
| First-run or feature hint | `TeachingTip` |
| Inline status or error | `InfoBar` |

Next: [Windows and Applications](10-windows-and-applications.md) moves beyond a single window.
