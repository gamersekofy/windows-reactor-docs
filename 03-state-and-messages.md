# 3. State and Messages

State changes in Reactor are explicit. A control raises an event, the event is mapped to a
message, `update` changes a field, and Reactor re-renders the component.

## Define a message type

A message can be a unit type, a single payload, or an enum. Enums scale best because one
component can handle several kinds of interaction:

```rust
#[derive(Clone)]
pub enum CounterMessage {
    Increment,
    Reset,
    Set(i32),
}
```

Derive `Clone` when you want to reuse a message value with `context.message(...)`. `Clone` is not
required by the `Component` trait itself.

## Three ways to create a callback

| Call | Result | Use for |
| --- | --- | --- |
| `context.forward()` | `Callback<C::Message>` | An event whose payload is exactly the message type |
| `context.callback(map)` | `Callback<T>` | An event with a payload that must be wrapped in a message |
| `context.message(msg)` | `Callback<()>` | A unit event, or when the payload can be ignored |

`forward` is the shortest option when `Message` is `()`, because a click payload is also `()`.
For an enum message, `on_click` cannot accept `forward` — `on_click` requires a `Callback<()>`, and
`Callback<CounterMessage>` is not one. Use `context.message(...)`:

```rust
use windows_reactor::*;

#[derive(Clone)]
pub enum CounterMessage {
    Increment,
    Reset,
    Set(i32),
}

pub struct EnumeratedCounter {
    count: i32,
}

impl Component for EnumeratedCounter {
    type Input = ();
    type Message = CounterMessage;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self { count: 0 }
    }

    fn update(&mut self, message: CounterMessage, _context: &ComponentContext<Self>) {
        match message {
            CounterMessage::Increment => self.count += 1,
            CounterMessage::Reset => self.count = 0,
            CounterMessage::Set(value) => self.count = value,
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(8.0)
            .children((
                TextBlock::new().text(format!("Count: {}", self.count)),
                StackPanel::new()
                    .orientation(Orientation::Horizontal)
                    .spacing(8.0)
                    .children((
                        Button::new()
                            .on_click(context.message(CounterMessage::Increment))
                            .content("Increment"),
                        Button::new()
                            .on_click(context.message(CounterMessage::Reset))
                            .content("Reset"),
                        Button::new()
                            .on_click(context.message(CounterMessage::Set(42)))
                            .content("Set 42"),
                    )),
            ))
            .into()
    }
}
```

The compiler error for the wrong shape is clear:

```text
the trait bound `Callback<CounterMessage>: IntoUnitCallback` is not satisfied
```

When you see it, replace `context.forward()` with `context.message(Msg::X)`.

## Controlled inputs

A controlled input gets its value from state and reports changes back. The view never owns the
value, so the component is always the source of truth:

```rust
use std::rc::Rc;

use windows_reactor::*;

#[derive(Clone)]
pub enum FormMessage {
    NameChanged(Rc<str>),
    AmountChanged(Option<f64>),
}

pub struct ControlledForm {
    name: Rc<str>,
    amount: Option<f64>,
}

impl Component for ControlledForm {
    type Input = ();
    type Message = FormMessage;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            name: Rc::from(""),
            amount: Some(5.0),
        }
    }

    fn update(&mut self, message: FormMessage, _context: &ComponentContext<Self>) {
        match message {
            FormMessage::NameChanged(name) => self.name = name,
            FormMessage::AmountChanged(amount) => self.amount = amount,
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(8.0)
            .children((
                TextBox::new()
                    .text(self.name.clone())
                    .placeholder_text("Name")
                    .on_text_changed(context.callback(FormMessage::NameChanged)),
                NumberBox::new()
                    .minimum(0.0)
                    .maximum(10.0)
                    .value(self.amount)
                    .on_value_changed(context.callback(FormMessage::AmountChanged)),
                format!("Hello, {}", self.name),
            ))
            .into()
    }
}
```

Payload types are chosen by the control, not by the app. `TextBox::on_text_changed` sends
`Rc<str>`, `NumberBox::on_value_changed` sends `Option<f64>`, and a list selection sends
`Option<usize>` or an `Option<Rc<str>>` depending on the control. Chapter 6 and chapter 7 list the
exact payload for every event used in these docs.

Because `Rc<str>` is cheap to clone, storing text as `Rc<str>` avoids copying on every keystroke.
If you prefer `String` in your own message, adapt the payload with `context.callback`, for
example `context.callback(|value: Rc<str>| Message::Name(value.to_string()))`.

## Share state with context

`Context<T>` is a statically identified slot. A provider supplies a value for a subtree;
descendants read it with `use_context`. Providers re-render only the consumers whose value
actually changed.

Declare the context once, usually as a `static`:

```rust
use std::sync::LazyLock;

use windows_reactor::*;

pub static THEME: LazyLock<Context<String>> =
    LazyLock::new(|| Context::new("light".to_string()));
```

`Context::new` takes the default used when no provider is above the consumer.

```rust
use std::sync::LazyLock;

use windows_reactor::*;

pub static THEME: LazyLock<Context<String>> =
    LazyLock::new(|| Context::new("light".to_string()));

pub struct ThemedLeaf;

impl Component for ThemedLeaf {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        let theme = context.use_context(&THEME);
        TextBlock::new().text(format!("theme = {theme}")).into()
    }
}

pub struct ThemeRoot {
    theme: String,
}

impl Component for ThemeRoot {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            theme: "dark".into(),
        }
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        provide(
            &THEME,
            self.theme.clone(),
            StackPanel::new().children((
                TextBlock::new().text("provided context"),
                View::component::<ThemedLeaf>(()),
            )),
        )
    }
}
```

`provide` takes `&Context<T>`, the value, and any `Into<View>` child. The value type must be
`Clone + PartialEq + 'static`.

Context is not a global. Only components rendered inside the provider see the value; everyone
else sees the default. This makes previews and tests straightforward: provide a different value
at the root.

## Element references

`ElementRef<T>` is a typed handle to a control that has been mounted. Attach it with
`.element_ref(&reference)` and call methods on the reference later:

```rust
use windows_reactor::*;

pub struct FocusDemo {
    input: ElementRef<TextBox>,
    status: &'static str,
}

#[derive(Clone)]
pub enum FocusMessage {
    Focus,
}

impl Component for FocusDemo {
    type Input = ();
    type Message = FocusMessage;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            input: ElementRef::new(),
            status: "Not focused",
        }
    }

    fn update(&mut self, message: FocusMessage, _context: &ComponentContext<Self>) {
        match message {
            FocusMessage::Focus => {
                self.status = if self.input.request_focus() {
                    "Focused"
                } else {
                    "Focus failed"
                };
            }
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(8.0)
            .children((
                TextBox::new()
                    .text("Focus target")
                    .element_ref(&self.input),
                Button::new()
                    .on_click(context.message(FocusMessage::Focus))
                    .content("Focus"),
                self.status,
            ))
            .into()
    }
}
```

`request_focus()` returns `bool`: `true` means the request was enqueued, not that focus
succeeded. For the native result, use `request_focus_result`:

```rust
use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    Focus,
    Focused(Result<bool, FocusError>),
}

pub struct FocusWithResult {
    input: ElementRef<TextBox>,
}

impl Component for FocusWithResult {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            input: ElementRef::new(),
        }
    }

    fn update(&mut self, message: Message, _context: &ComponentContext<Self>) {
        if let Message::Focus = message {
            let _ = self.input.request_focus_result(|_| {});
        }
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        TextBox::new()
            .text("Target")
            .element_ref(&self.input)
            .into()
    }
}
```

Inside `update` you usually want the result delivered as a message. Clone the sender before the
closure:

```rust
use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    Focus,
    Focused(Result<bool, FocusError>),
}

pub struct FocusReporter {
    input: ElementRef<TextBox>,
}

impl Component for FocusReporter {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            input: ElementRef::new(),
        }
    }

    fn update(&mut self, message: Message, context: &ComponentContext<Self>) {
        match message {
            Message::Focus => {
                let sender = context.sender();
                let _ = self.input.request_focus_result(move |result| {
                    let _ = sender.send(Message::Focused(result));
                });
            }
            Message::Focused(_) => {}
        }
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        TextBox::new()
            .text("Target")
            .element_ref(&self.input)
            .into()
    }
}
```

The type parameter is checked. `ElementRef<TextBox>` can only be attached to a `TextBox`; passing
it to another control is a compile error. Use `ElementRef` (the default `ElementRef<AnyElement>`)
when a helper needs to accept any element.

Controls that support extra integration APIs have typed references with additional methods:
`ElementRef<Image>` can replace the native image source, `ElementRef<WebView2>` can expose the core
web view, and `ElementRef<SwapChainPanel>` can bind a swap chain. These are advanced interop
points; most apps only need focus and automation notifications.

## Sending messages from outside an event

`context.sender()` returns a `LocalSender<C::Message>` that can be cloned into closures and
timers. `send` returns `false` when the component has been dropped, so late callbacks are safe:

```rust
use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    Finished(u32),
}

pub struct SenderDemo {
    result: Option<u32>,
}

impl Component for SenderDemo {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self { result: None }
    }

    fn update(&mut self, message: Message, _context: &ComponentContext<Self>) {
        match message {
            Message::Finished(value) => self.result = Some(value),
        }
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        TextBlock::new()
            .text(format!("result = {:?}", self.result))
            .into()
    }
}
```

For work that must return a value to the component, prefer `spawn_background` or
`context.completion()` (chapter 11) over keeping a raw sender in a global.

## Message hygiene

- Keep messages small. Move large payloads in an `Rc` or an id.
- Prefer `Rc<str>` for text payloads to avoid allocation on every keystroke.
- Do not put a `Callback` inside a message unless the component's input genuinely needs to invoke
  a parent callback; callbacks are cheap to clone and compare by identity, which makes them valid
  input fields.
- `#[non_exhaustive]` enums such as `ContentDialogResult` and `InfoBarSeverity` require a wildcard
  arm when matched.

Next: [Layout](04-layout.md) shows how controls compose into a window.
