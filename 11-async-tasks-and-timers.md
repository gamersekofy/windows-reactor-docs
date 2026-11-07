# 11. Async, Tasks, and Effects

`update` and `view` run on the UI thread. Anything that blocks — file I/O, network calls,
sleeps — belongs off that thread, or in an effect that runs after the frame is committed.

## Background work

`spawn_background` runs a closure on a worker thread and sends its return value back as a
message:

```rust
use std::time::Duration;

use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    Start,
    Finished(i32),
    Cancelled,
    CancelTask,
}

pub struct AsyncDemo {
    busy: bool,
    count: i32,
    task: Option<ComponentTask>,
}

impl Component for AsyncDemo {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            busy: false,
            count: 0,
            task: None,
        }
    }

    fn update(&mut self, message: Message, context: &ComponentContext<Self>) {
        match message {
            Message::Start if !self.busy => {
                self.busy = true;
                let result = self.count + 1;
                self.task = Some(context.spawn_background(move |cancel| {
                    std::thread::sleep(Duration::from_millis(500));
                    if cancel.is_cancelled() {
                        Message::Cancelled
                    } else {
                        Message::Finished(result)
                    }
                }));
            }
            Message::Finished(value) => {
                self.count = value;
                self.busy = false;
            }
            Message::Cancelled => self.busy = false,
            Message::CancelTask => {
                if let Some(task) = &self.task {
                    task.cancel();
                }
                self.busy = false;
            }
            Message::Start => {}
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(8.0)
            .children((
                TextBlock::new()
                    .text(if self.busy {
                        "Working...".to_string()
                    } else {
                        format!("Count: {}", self.count)
                    })
                    .font_size(20.0),
                Button::new()
                    .on_click(context.message(Message::Start))
                    .content("Start background work"),
                Button::new()
                    .on_click(context.message(Message::CancelTask))
                    .content("Cancel"),
            ))
            .into()
    }
}
```

Key rules:

- The closure must be `Send + 'static`, and `Message` must be `Send`. This is why messages carry
  ids and `String`/`Rc`-free data when used with background work.
- `spawn_background` returns a `ComponentTask`. Store it if you need to cancel.
- The `CancellationToken` is only a flag. Check `cancel.is_cancelled()` at natural pause points;
  Reactor cannot interrupt a blocking call.
- Cancelling after the worker has finished, or dropping the task, does not stop a closure that is
  already running.
- `ComponentTask::is_rejected()` reports whether the result message was rejected because the
  component queue was full or closed. Rejection is terminal; the result is not retried.
- A task result is dropped if the component has been unmounted.

If the work needs to return many intermediate values, send them through a channel you own and
wake the component with `context.sender()`, or split the work into messages.

## Timers

`set_timeout` delivers a message after a delay. `set_local_timeout` builds the message on the UI
thread when the timer fires:

```rust
use std::time::Duration;

use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    Start,
    Tick,
    Reset,
}

pub struct Timers {
    ticks: u32,
    timer: Option<ComponentTimer>,
}

impl Component for Timers {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            ticks: 0,
            timer: None,
        }
    }

    fn update(&mut self, message: Message, context: &ComponentContext<Self>) {
        match message {
            Message::Start => {
                self.timer = Some(context.set_timeout(Duration::from_secs(1), Message::Tick));
            }
            Message::Tick => {
                self.ticks += 1;
                self.timer = Some(context.set_timeout(Duration::from_secs(1), Message::Tick));
            }
            Message::Reset => self.ticks = 0,
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(8.0)
            .children((
                format!("Ticks: {}", self.ticks),
                Button::new()
                    .on_click(context.message(Message::Start))
                    .content("Start timer"),
            ))
            .into()
    }
}
```

`ComponentTimer` is `#[must_use]` and **dropping it cancels delivery**. A local binding such as
`let _timer = context.set_timeout(...)` is dropped at the end of `update`, which cancels the timer
before it can fire. Store the handle in a field until it fires. Once the timer has fired, dropping
the handle is harmless because delivery has already completed.

A toggleable repeating timer uses the same storage pattern:

```rust
use std::time::Duration;

use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    Toggle,
    Tick,
}

pub struct RepeatingTimer {
    timer: Option<ComponentTimer>,
    ticks: u32,
}

impl Component for RepeatingTimer {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            timer: None,
            ticks: 0,
        }
    }

    fn update(&mut self, message: Message, context: &ComponentContext<Self>) {
        match message {
            Message::Toggle => {
                if let Some(timer) = self.timer.take() {
                    timer.cancel();
                } else {
                    self.timer = Some(context.set_timeout(
                        Duration::from_secs(1),
                        Message::Tick,
                    ));
                }
            }
            Message::Tick => {
                self.ticks += 1;
                self.timer =
                    Some(context.set_timeout(Duration::from_secs(1), Message::Tick));
            }
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(8.0)
            .children((
                format!("Ticks: {}", self.ticks),
                Button::new()
                    .on_click(context.message(Message::Toggle))
                    .content("Toggle timer"),
            ))
            .into()
    }
}
```

`set_local_timeout` takes `FnOnce() -> Message` and does not require `Message: Send`:

```rust
use std::time::Duration;

use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    Start,
    Tick,
}

pub struct LocalTimer {
    timer: Option<ComponentTimer>,
}

impl Component for LocalTimer {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self { timer: None }
    }

    fn update(&mut self, message: Message, context: &ComponentContext<Self>) {
        match message {
            Message::Start => {
                self.timer = Some(context.set_local_timeout(
                    Duration::from_millis(250),
                    || Message::Tick,
                ));
            }
            Message::Tick => {}
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        Button::new()
            .on_click(context.message(Message::Start))
            .content("Start")
            .into()
    }
}
```

Use `set_local_timeout` when the message contains non-`Send` values such as `Rc<str>` or a
`Callback`.

## Effects

An effect runs after the new view has been committed. It is the place for imperative work that
should not happen during rendering: focusing a control, scrolling, starting an animation, or
syncing with a non-Reactor API.

```rust
use std::rc::Rc;

use windows_reactor::*;

pub struct FocusEffect {
    input: ElementRef<TextBox>,
    result: Option<Rc<str>>,
}

impl Component for FocusEffect {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            input: ElementRef::new(),
            result: None,
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        let input = self.input.clone();
        context.use_effect("focus-on-mount", (), move || {
            let _ = input.request_focus();
            None
        });

        StackPanel::new()
            .spacing(8.0)
            .children((
                TextBox::new()
                    .text(self.result.as_deref().unwrap_or(""))
                    .element_ref(&self.input),
                format!("result = {:?}", self.result),
            ))
            .into()
    }
}
```

`use_effect(key, dependency, setup)` semantics:

| Aspect | Behavior |
| --- | --- |
| When it runs | After the view is applied, never during rendering |
| When it reruns | When `dependency` compares unequal to the previous value |
| First run | On the first render of the component |
| Cleanup | The optional cleanup closure returned by `setup` runs before the next setup and on unmount |
| Key | Must be unique within one render; declaring the same key twice panics |

Dependencies can be any `PartialEq + 'static` value. Tuples work well for several values:

```rust
use windows_reactor::*;

pub struct MultiDep {
    count: u32,
    label: String,
}

impl Component for MultiDep {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            count: 0,
            label: "initial".into(),
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        let count = self.count;
        let label = self.label.clone();
        context.use_effect("sync", (count, label.clone()), move || {
            let _ = (count, label);
            None
        });

        TextBlock::new().text(format!("{} / {}", self.count, self.label)).into()
    }
}
```

The key must be unique in a single `view` call. Two `use_effect("focus", ...)` calls panic with
`effect "focus" declared more than once`. Derive `EffectKey` from a value with
`impl Into<Key>`; strings and integers are accepted.

### Guards

`use_effect_guard(key, dependency, setup)` is the ergonomic form when the effect owns a value
that should be dropped on rerun or unmount. The returned guard is dropped automatically:

```rust
use windows_reactor::*;

pub struct EffectGuardDemo {
    value: u32,
}

struct LogOnDrop(&'static str);

impl Drop for LogOnDrop {
    fn drop(&mut self) {
        // Runs when the dependency changes or the component unmounts.
        let _ = self.0;
    }
}

impl Component for EffectGuardDemo {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self { value: 0 }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        context.use_effect_guard("log-value", self.value, || LogOnDrop("value"));
        TextBlock::new().text(format!("Value: {}", self.value)).into()
    }
}
```

Use a guard for subscriptions, native handles, and cancellation tokens that must be released
when the dependency changes. Use the plain `use_effect` form when you need custom cleanup logic
or an effect that returns nothing.

### Effects versus `update`

| Work | Where |
| --- | --- |
| Change component state | `update` |
| Focus a control after it appears | effect |
| Start or stop a native animation | effect with a guard |
| Read a value and synchronously update state | `update` (no side effects in `view`) |
| Persist window geometry | `on_window_placement` handler |
| Long-running work | `spawn_background` |

Do not call `context.use_effect` conditionally with varying keys across renders if the
condition changes which key is present. Reactor compares effects by key: a key that disappears
has its cleanup run, and a new key starts a new effect. That is correct, but it is easy to create
a loop if the effect itself changes the dependency.

## Completion handles

`context.completion()` returns a completion handle: a cloneable, typed sender that is consumed
when it completes. Store the handle while the operation is in flight, then call `complete` from
the message handler that finishes the work. Because `complete(self)` consumes the handle, one
handle can send one message; clone it if several continuations need to report.

```rust
use windows_reactor::*;

#[derive(Clone)]
pub enum Message {
    Start,
    Finish,
    Done(u32),
}

pub struct CompletionDemo {
    pending: Option<ComponentCompletion<Message>>,
    completed: bool,
}

impl Component for CompletionDemo {
    type Input = ();
    type Message = Message;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            pending: None,
            completed: false,
        }
    }

    fn update(&mut self, message: Message, context: &ComponentContext<Self>) {
        match message {
            Message::Start => self.pending = Some(context.completion()),
            Message::Finish => {
                if let Some(completion) = self.pending.take() {
                    let _ = completion.complete(Message::Done(1));
                }
            }
            Message::Done(_) => self.completed = true,
        }
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(8.0)
            .children((
                format!("completed = {}", self.completed),
                Button::new()
                    .on_click(context.message(Message::Start))
                    .content("Begin"),
                Button::new()
                    .on_click(context.message(Message::Finish))
                    .content("Finish"),
            ))
            .into()
    }
}
```

`complete` and `complete_with` return `bool`: `true` if the message was queued, `false` if the
host had closed or the message queue was full. `complete` requires `Message: Send`;
`complete_with` takes a `FnOnce() -> Message + Send` and builds the value on the component's
thread, so the message type itself does not have to be `Send`.

## Threading summary

| API | Thread | Message bound |
| --- | --- | --- |
| `update`, `view`, effects | UI thread | none beyond `'static` |
| `spawn_background` | worker thread | `Send` |
| `set_timeout` | message sent from a timer | `Send` |
| `set_local_timeout` | message built on the UI thread | none |
| `spawn_background` cancellation | `CancellationToken` is `Send` | — |

Next: [Media, Icons, and Data](12-media-icons-and-data.md) covers images, dates, and drag-and-drop.
