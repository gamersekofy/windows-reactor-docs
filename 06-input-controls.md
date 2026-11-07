# 6. Input Controls

Reactor's input controls follow the same pattern as HTML controlled inputs: the value comes from
state and the change event sends a message. The event payload is defined by the control, so start
by choosing the message variant that matches the payload.

## Text inputs

| Control | Value builder | Change event payload |
| --- | --- | --- |
| `TextBox` | `text` | `Rc<str>` |
| `PasswordBox` | `password` | `Rc<str>` |
| `RichEditBox` | `text` | `Rc<str>` |
| `AutoSuggestBox` | `text` | `Rc<str>` (plus `on_suggestion_chosen` → `Rc<str>`) |

All text inputs support `header`, `placeholder_text`, and `is_enabled`. `TextBox` and
`RichEditBox` support `is_read_only`; `TextBox` also supports `accepts_return`,
`text_alignment`, and `text_wrapping`.

A text field in a form:

```rust
use std::rc::Rc;

use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    NameChanged(Rc<str>),
}

pub struct NameForm {
    name: Rc<str>,
}

impl Component for NameForm {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            name: Rc::from(""),
        }
    }

    fn update(&mut self, message: Message, _context: &ComponentContext<Self>) {
        match message {
            Message::NameChanged(name) => self.name = name,
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(8.0)
            .children((
                TextBox::new()
                    .header("Name")
                    .placeholder_text("Type your name")
                    .text(self.name.clone())
                    .on_text_changed(context.callback(Message::NameChanged)),
                format!("Hello, {}", self.name),
            ))
            .into()
    }
}
```

`AutoSuggestBox::items_source` accepts any iterator of `impl Into<Rc<str>>`. Filtering is the
app's job: keep the source in state and rebuild it in `view` from the current query.

## Toggles

| Control | Value builder | Event | Payload |
| --- | --- | --- | --- |
| `CheckBox` | `is_checked` | `on_is_checked_changed` | `Option<bool>` |
| `ToggleButton` | `is_checked` | `on_is_checked_changed` | `Option<bool>` |
| `ToggleSwitch` | `is_on` | `on_toggled` | `bool` |
| `RadioButton` | `is_checked` | `on_checked` | `Option<bool>` |
| `RadioButtons` | `selected_index` | `on_selection_changed` | `Option<usize>` |

`CheckBox` and `ToggleButton` accept a `bool` for `is_checked` because `bool` converts into
`Option<bool>`. `RadioButtons` is the convenient choice for a group: give it
`items_source([...])`, `max_columns`, and a `selected_index`; it renders the radio buttons for
you. Use individual `RadioButton`s with the same `group_name` when you need custom layout.

## Ranges and numeric values

| Control | Builders | Event | Payload |
| --- | --- | --- | --- |
| `Slider` | `minimum`, `maximum`, `step_frequency`, `value` | `on_value_changed` | `f64` |
| `NumberBox` | `minimum`, `maximum`, `small_change`, `value` | `on_value_changed` | `Option<f64>` |
| `RatingControl` | `caption`, `max_rating`, `value` | `on_value_changed` | `Option<f64>` |
| `ProgressBar` | `minimum`, `maximum`, `value`, `is_indeterminate` | — | — |
| `ProgressRing` | `minimum`, `maximum`, `value`, `is_active` | — | — |
| `InfoBadge` | `value` | — | — |

`NumberBox` reports `None` when the field is empty or cannot be parsed. That is why its message
variant should be `Option<f64>`, not `f64`. `Slider` always reports a number.

## Pickers

| Control | Value builder | Event | Payload |
| --- | --- | --- | --- |
| `ComboBox` | `selected_index`, `items_source` | `on_selection_changed` | `Option<usize>` |
| `CalendarDatePicker` | `date` | `on_date_changed` | `Option<DateTime>` |
| `DatePicker` | — | `on_date_changed` | `Option<DateTime>` |
| `TimePicker` | `minute_increment` | `on_time_changed` | `Option<TimeSpan>` |
| `ColorPicker` | `color` | `on_color_changed` | `Color` |

Calendar-backed pickers use `Option<DateTime>` because an unset date is represented by `None`.
`DateTime` has no `Default` implementation, so `Option<DateTime>` is the natural state shape.
`TimePicker` uses `Option<TimeSpan>`.

`CalendarDatePicker` also supports `min_date`, `max_date`, `is_calendar_open`, and
`is_today_highlighted`. `DatePicker` exposes `day_visible`, `month_visible`, and `year_visible`
for its individual fields; `TimePicker` supports `clock_identifier`.

## A complete form

```rust
use std::rc::Rc;

use windows_reactor::*;

#[derive(Clone)]
pub enum FormMessage {
    Name(Rc<str>),
    Password(Rc<str>),
    Subscribe(Option<bool>),
    Volume(f64),
    Amount(Option<f64>),
    Choice(Option<usize>),
    Birthday(Option<DateTime>),
    Reminder(Option<TimeSpan>),
    Rating(Option<f64>),
    Color(Color),
    Suggest(Rc<str>),
    CloseInfoBar,
}

pub struct ProfileForm {
    name: Rc<str>,
    password: Rc<str>,
    subscribe: bool,
    volume: f64,
    amount: Option<f64>,
    choice: Option<usize>,
    birthday: Option<DateTime>,
    rating: Option<f64>,
    color: Color,
    info_open: bool,
}

impl Component for ProfileForm {
    type Input = ();
    type Message = FormMessage;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            name: Rc::from(""),
            password: Rc::from(""),
            subscribe: true,
            volume: 40.0,
            amount: Some(1.0),
            choice: Some(0),
            birthday: None,
            rating: Some(3.0),
            color: Color::rgb(0, 120, 212),
            info_open: true,
        }
    }

    fn update(&mut self, message: FormMessage, _context: &ComponentContext<Self>) {
        match message {
            FormMessage::Name(value) => self.name = value,
            FormMessage::Password(value) => self.password = value,
            FormMessage::Subscribe(value) => self.subscribe = value.unwrap_or(false),
            FormMessage::Volume(value) => self.volume = value,
            FormMessage::Amount(value) => self.amount = value,
            FormMessage::Choice(value) => self.choice = value,
            FormMessage::Birthday(value) => self.birthday = value,
            FormMessage::Rating(value) => self.rating = value,
            FormMessage::Color(value) => self.color = value,
            FormMessage::Suggest(_) => {}
            FormMessage::Reminder(_) => {}
            FormMessage::CloseInfoBar => self.info_open = false,
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(12.0)
            .children((
                TextBox::new()
                    .header("Name")
                    .text(self.name.clone())
                    .placeholder_text("Display name")
                    .on_text_changed(context.callback(FormMessage::Name)),
                PasswordBox::new()
                    .header("Password")
                    .password(self.password.clone())
                    .on_password_changed(context.callback(FormMessage::Password)),
                AutoSuggestBox::new()
                    .header("Search")
                    .placeholder_text("Type to filter")
                    .items_source(["Alpha", "Beta", "Gamma"])
                    .on_text_changed(context.callback(FormMessage::Suggest)),
                CheckBox::new()
                    .content("Subscribe to updates")
                    .is_checked(self.subscribe)
                    .on_is_checked_changed(context.callback(FormMessage::Subscribe)),
                ToggleSwitch::new()
                    .header("Notifications")
                    .is_on(self.subscribe)
                    .on_toggled(context.callback(|value: bool| {
                        FormMessage::Subscribe(Some(value))
                    })),
                Slider::new()
                    .header("Volume")
                    .minimum(0.0)
                    .maximum(100.0)
                    .step_frequency(1.0)
                    .value(self.volume)
                    .on_value_changed(context.callback(FormMessage::Volume)),
                NumberBox::new()
                    .header("Amount")
                    .minimum(0.0)
                    .maximum(1000.0)
                    .small_change(0.5)
                    .value(self.amount)
                    .on_value_changed(context.callback(FormMessage::Amount)),
                ComboBox::new()
                    .header("Choice")
                    .items_source(["First", "Second", "Third"])
                    .selected_index(self.choice)
                    .on_selection_changed(context.callback(FormMessage::Choice)),
                RadioButtons::new()
                    .header("Priority")
                    .items_source(["Low", "Normal", "High"])
                    .max_columns(3)
                    .selected_index(self.choice)
                    .on_selection_changed(context.callback(FormMessage::Choice)),
                CalendarDatePicker::new()
                    .header("Birthday")
                    .date(self.birthday)
                    .on_date_changed(context.callback(FormMessage::Birthday)),
                DatePicker::new()
                    .header("Start date")
                    .on_date_changed(context.callback(FormMessage::Birthday)),
                TimePicker::new()
                    .header("Reminder")
                    .minute_increment(15)
                    .on_time_changed(context.callback(FormMessage::Reminder)),
                RatingControl::new()
                    .caption("Rating")
                    .max_rating(5)
                    .value(self.rating)
                    .on_value_changed(context.callback(FormMessage::Rating)),
                ColorPicker::new()
                    .color(self.color)
                    .on_color_changed(context.callback(FormMessage::Color)),
                ProgressBar::new()
                    .minimum(0.0)
                    .maximum(100.0)
                    .value(self.volume),
                InfoBar::new()
                    .title("Saved")
                    .message("Your changes were stored.")
                    .severity(InfoBarSeverity::Success)
                    .is_open(self.info_open)
                    .is_closable(true)
                    .on_closed(context.message(FormMessage::CloseInfoBar)),
            ))
            .into()
    }
}
```

This form keeps every value in component state and rebuilds the controls on each update. Reactor
diffs the declarations, so a keystroke updates the text property rather than recreating the
`TextBox`.

## Buttons

| Control | Notes |
| --- | --- |
| `Button` | Standard button; `style(ButtonStyle::...)` |
| `RepeatButton` | Fires repeatedly while held; `delay`, `interval` in milliseconds |
| `ToggleButton` | Stays pressed; `is_checked` |
| `SplitButton` | Primary action plus a secondary area |
| `DropDownButton` | Opens an attached flyout |
| `HyperlinkButton` | `navigate_uri` opens a URI |
| `AppBarButton` | Command bar button; `label`, `icon`, `is_enabled` |
| `AppBarSeparator` | Divider for command bars |

`navigate_uri` returns a `Result` because the URI is validated when the view is built:

```rust
use windows_reactor::*;

pub fn button_family() -> View {
    StackPanel::new()
        .spacing(8.0)
        .children((
            Button::new().content("Primary"),
            RepeatButton::new()
                .content("Repeat")
                .delay(500)
                .interval(100),
            ToggleButton::new().content("Toggle"),
            SplitButton::new().content("Split"),
            DropDownButton::new().content("Drop down"),
            HyperlinkButton::new()
                .content("Open docs")
                .navigate_uri("https://learn.microsoft.com/windows/apps/")
                .unwrap(),
            AppBarButton::new().label("Add").icon(Symbol::Add),
            AppBarSeparator::new(),
            RadioButton::new().content("Option").group_name("group"),
        ))
        .into()
}
```

`HyperlinkButton`, `Image`, `BitmapIcon`, and the image-producing icon types return
`Result<Self>` when a path or URI is validated. Call `.unwrap()` for constants,
or handle the error when the value comes from user input.

## Accessibility

Every control supports the common automation methods from chapter 4. Label inputs with `header`
when possible; that also supplies the accessible name. For icon-only buttons, set
`automation_name` to a short action description. For status text that changes, use
`automation_live_setting(AutomationLiveSetting::Polite)` so screen readers announce it.

Next: [Collections](07-collections.md) covers list, grid, and tree controls.
