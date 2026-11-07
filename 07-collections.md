# 7. Collections

Reactor wraps the WinUI list controls and adds an explicit item model. The control you choose
determines the item type and the selection payload.

## Choosing a collection control

| Control | Layout | Item API | Selection payload |
| --- | --- | --- | --- |
| `ListView` | Vertical list | `items(Iterator<Item = DataItem>)` | `Option<usize>` |
| `GridView` | Wrapping grid | `items(IntoViews)` / `keyed_items` | `Option<usize>` |
| `ListBox` | Simple vertical list | `items(IntoViews)` / `keyed_items` | `Option<Rc<str>>` (tag) |
| `FlipView` | One page at a time | `items(IntoViews)` / `keyed_items` | `Option<usize>` |
| `ItemsRepeater` | Custom layout | `items(Iterator<Item = KeyedView>)` / `virtual_source` | none |
| `TreeView` | Hierarchical | `nodes(Iterator<Item = TreeNode>)` | `Rc<str>` (invoked key) |

`ListView` is the default for mail, contacts, and settings. `GridView` is the default for
galleries. `ListBox` suits short option lists where a string tag is more useful than an index.
`ItemsRepeater` is for custom virtualized layouts where you do not want item chrome.

## ListView

`ListView` takes `DataItem` values. A `DataItem` has a key, a default text representation, and an
optional rich `content`:

```rust
use windows_reactor::*;

pub struct ListDemo {
    selected: Option<usize>,
}

impl Component for ListDemo {
    type Input = ();
    type Message = Option<usize>;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self { selected: None }
    }

    fn update(&mut self, selected: Option<usize>, _context: &ComponentContext<Self>) {
        self.selected = selected;
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        const INBOX: [&str; 4] = [
            "Design review",
            "Flight confirmation",
            "Weekly status",
            "Welcome",
        ];
        ListView::new()
            .height(240.0)
            .selection_mode(ListViewSelectionMode::Single)
            .selected_index(self.selected)
            .on_selection_changed(context.forward())
            .items(INBOX.into_iter().enumerate().map(|(index, subject)| {
                DataItem::new(subject, subject).content(
                    StackPanel::new().spacing(2.0).children((
                        TextBlock::new()
                            .text(subject)
                            .font_weight(FontWeight::BOLD),
                        TextBlock::new()
                            .text(format!("Message #{}", index + 1))
                            .opacity(0.6),
                    )),
                )
            }))
            .into()
    }
}
```

`selection_mode` accepts `Single`, `Multiple`, `Extended`, and `None`. `selected_index` is
controlled: read it from state and write the new index back in `update`.

For reordering, add `can_reorder_items(true)` and handle `on_reordered`, whose payload is
`Vec<Key>` in the new order. `can_drag_items` enables dragging items out; `allow_drop` accepts
drops onto the list.

## A contact list

This example combines `ListView`, `PersonPicture`, and a details pane. Selection is an index
into a fixed contact table:

```rust
use windows_reactor::*;

pub struct ContactList {
    selected: Option<usize>,
}

struct Contact {
    name: &'static str,
    role: &'static str,
    initials: &'static str,
}

const CONTACTS: [Contact; 4] = [
    Contact {
        name: "Avery Chen",
        role: "Design lead",
        initials: "AC",
    },
    Contact {
        name: "Diego Alvarez",
        role: "Engineer",
        initials: "DA",
    },
    Contact {
        name: "Harper Singh",
        role: "Program manager",
        initials: "HS",
    },
    Contact {
        name: "Mina Okafor",
        role: "Researcher",
        initials: "MO",
    },
];

impl Component for ContactList {
    type Input = ();
    type Message = Option<usize>;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            selected: Some(0),
        }
    }

    fn update(&mut self, selected: Option<usize>, _context: &ComponentContext<Self>) {
        self.selected = selected;
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        let row = |contact: &Contact| {
            DataItem::new(contact.name, contact.name).content(
                StackPanel::new()
                    .orientation(Orientation::Horizontal)
                    .spacing(12.0)
                    .children((
                        PersonPicture::new()
                            .initials(contact.initials)
                            .width(40.0)
                            .height(40.0),
                        StackPanel::new().spacing(2.0).children((
                            TextBlock::new()
                                .text(contact.name)
                                .font_weight(FontWeight::BOLD),
                            TextBlock::new().text(contact.role).opacity(0.6),
                        )),
                    )),
            )
        };

        let details = self.selected.map_or_else(
            || "Select a contact".to_string(),
            |index| {
                let contact = &CONTACTS[index];
                format!("{} — {}", contact.name, contact.role)
            },
        );

        Grid::new()
            .rows([GridLength::Star(1.0)])
            .columns([GridLength::Star(2.0), GridLength::Star(1.0)])
            .column_spacing(16.0)
            .children((
                ListView::new()
                    .selection_mode(ListViewSelectionMode::Single)
                    .selected_index(self.selected)
                    .on_selection_changed(context.forward())
                    .items(CONTACTS.iter().map(row))
                    .grid_column(0),
                Border::new()
                    .padding(24.0)
                    .background(ThemeBrush::CardBackground)
                    .corner_radius(CornerRadius::uniform(8.0))
                    .content(TextBlock::new().text(details).font_size(20.0))
                    .grid_column(1),
            ))
            .into()
    }
}
```

The `DataItem` key is the contact name. If the list can be reordered, that key is what preserves
selection and item state.

## GridView

`GridView` takes any `IntoViews` children, or `KeyedView` values when you need stable identity.
`GridViewItem` adds the built-in item chrome and a `tag`:

```rust
use windows_reactor::*;

pub struct GridDemo {
    selected: Option<usize>,
}

impl Component for GridDemo {
    type Input = ();
    type Message = Option<usize>;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self { selected: None }
    }

    fn update(&mut self, selected: Option<usize>, _context: &ComponentContext<Self>) {
        self.selected = selected;
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        let items: Vec<String> = (1..=12).map(|i| format!("Item {i}")).collect();
        GridView::new()
            .height(300.0)
            .selected_index(self.selected)
            .on_selection_changed(context.forward())
            .keyed_items(items.iter().map(|item| {
                KeyedView::new(
                    item.clone(),
                    GridViewItem::new().tag(item).content(
                        Border::new()
                            .padding(16.0)
                            .corner_radius(CornerRadius::uniform(4.0))
                            .content(
                                TextBlock::new()
                                    .text(item)
                                    .font_weight(FontWeight::BOLD),
                            ),
                    ),
                )
            }))
            .into()
    }
}
```

Use `GridViewItem` when you want hover, selection, and focus visuals around each cell. Use a
plain `Border` (as in an `ItemsRepeater`) when you are drawing everything yourself.

## ListBox

`ListBox` is lighter than `ListView` and selects by tag rather than index:

```rust
use std::rc::Rc;

use windows_reactor::*;

pub struct ListBoxDemo {
    selected: Option<Rc<str>>,
}

impl Component for ListBoxDemo {
    type Input = ();
    type Message = Option<Rc<str>>;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            selected: Some("inbox".into()),
        }
    }

    fn update(&mut self, selected: Option<Rc<str>>, _context: &ComponentContext<Self>) {
        self.selected = selected;
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        ListBox::new()
            .on_selected_tag_changed(context.forward())
            .items((
                ListBoxItem::new()
                    .tag("inbox")
                    .is_selected(self.selected.as_deref() == Some("inbox"))
                    .content("Inbox"),
                ListBoxItem::new()
                    .tag("sent")
                    .is_selected(self.selected.as_deref() == Some("sent"))
                    .content("Sent"),
            ))
            .into()
    }
}
```

`ListBoxItem::tag` is the value delivered by `on_selected_tag_changed`. Because the payload is
`Option<Rc<str>>`, a `Message = Option<Rc<str>>` component can use `context.forward()`. A
`ListBox` with plain views uses `items(IntoViews)`; the item views do not carry tags, so selection
by tag requires `ListBoxItem`.

## FlipView

`FlipView` shows one child at a time with next/previous controls. It is useful for onboarding
pages, media, and carousels:

```rust
use windows_reactor::*;

pub struct FlipDemo {
    selected: Option<usize>,
}

impl Component for FlipDemo {
    type Input = ();
    type Message = Option<usize>;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self { selected: Some(0) }
    }

    fn update(&mut self, selected: Option<usize>, _context: &ComponentContext<Self>) {
        self.selected = selected;
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        FlipView::new()
            .height(200.0)
            .selected_index(self.selected)
            .on_selection_changed(context.forward())
            .keyed_items([
                KeyedView::new(
                    "slide-1",
                    TextBlock::new().text("Welcome").font_size(24.0),
                ),
                KeyedView::new(
                    "slide-2",
                    TextBlock::new().text("Features").font_size(24.0),
                ),
                KeyedView::new(
                    "slide-3",
                    TextBlock::new().text("Resources").font_size(24.0),
                ),
            ])
            .into()
    }
}
```

A `FlipView` child should size itself. Give the `FlipView` a fixed or star-sized height;
otherwise the control collapses to the height of the largest page.

## ItemsRepeater

`ItemsRepeater` is a layout primitive. It does not add selection, item chrome, or a scroll
surface. Wrap it in a `ScrollView` when the content can exceed the window:

```rust
use windows_reactor::*;

pub struct RepeaterDemo {
    rows: Vec<String>,
}

impl Component for RepeaterDemo {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            rows: vec!["One".into(), "Two".into(), "Three".into()],
        }
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        ScrollView::new()
            .content(
                ItemsRepeater::new()
                    .items(
                        self.rows
                            .iter()
                            .map(|row| keyed(row.clone(), TextBlock::new().text(row))),
                    ),
            )
            .into()
    }
}
```

`ItemsRepeater::items` requires `KeyedView`; use `keyed(key, view)` or `KeyedView::new`. A key
that changes forces that row to be recreated. Keys should identify the data, not the position.

For a single keyed item, use `.item(key, view)`.

## Virtualization

`VirtualSource` describes a large data set lazily. Reactor asks your closures for a key and a view
only when a row is about to be realized:

```rust
use windows_reactor::*;

pub struct VirtualDemo;

impl Component for VirtualDemo {
    type Input = ();
    type Message = ();

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self
    }

    fn view(&self, _input: &(), _context: &mut ViewContext<Self>) -> View {
        ScrollView::new()
            .content(
                ItemsRepeater::new()
                    .virtual_source(VirtualSource::new(
                        0,
                        10_000,
                        |index| index,
                        |index| TextBlock::new().text(format!("Row {index}")),
                    )),
            )
            .into()
    }
}
```

`VirtualSource::new(key_revision, len, key, view)` arguments:

| Argument | Meaning |
| --- | --- |
| `key_revision` | Bump this when the meaning of a key changes; forces a fresh key space |
| `len` | Number of logical rows |
| `key` | `Fn(usize) -> impl Into<Key>` |
| `view` | `Fn(usize) -> impl Into<View>` |

Keys must be stable for the lifetime of a `key_revision`. If the underlying collection is
replaced, increment the revision so stale rows are not reused. `VirtualSource::len` and
`is_empty` are available for computing scroll extents.

## TreeView

`TreeView` takes `TreeNode` values. Each node has a key, content, optional children, and an
`expanded` flag. `on_item_invoked` reports the invoked node's key as `Rc<str>`:

```rust
use std::rc::Rc;

use windows_reactor::*;

pub struct TreeDemo {
    last_invoked: Rc<str>,
}

impl Component for TreeDemo {
    type Input = ();
    type Message = Rc<str>;

    fn create(_input: &(), _context: &ComponentContext<Self>) -> Self {
        Self {
            last_invoked: "(none)".into(),
        }
    }

    fn update(&mut self, value: Rc<str>, _context: &ComponentContext<Self>) {
        self.last_invoked = value;
    }

    fn view(&self, _input: &(), context: &mut ViewContext<Self>) -> View {
        StackPanel::new()
            .spacing(8.0)
            .children((
                TreeView::new()
                    .selection_mode(TreeViewSelectionMode::Single)
                    .on_item_invoked(context.forward())
                    .nodes([
                        TreeNode::new("documents", "Documents")
                            .expanded(true)
                            .children([
                                TreeNode::new("work", "Work").children([
                                    TreeNode::new("report", "Report.docx"),
                                    TreeNode::new("budget", "Budget.xlsx"),
                                ]),
                                TreeNode::new("personal", "Personal"),
                            ]),
                        TreeNode::new("pictures", "Pictures").children([
                            TreeNode::new("vacation", "Vacation.jpg"),
                        ]),
                    ]),
                TextBlock::new().text(format!("Last invoked: {}", self.last_invoked)),
            ))
            .into()
    }
}
```

`TreeView` selection is reported by `on_item_invoked` with the node key. Use the key to look up
the node in your own model; the control does not keep a separate selected-index property.

## Keys, selection, and state

- Use `KeyedView` whenever rows contain components or local state. A row keyed by index loses
  its state on insertion or reorder.
- `ListView` and `GridView` report indices, not keys. Keep the backing `Vec` in the same order as
  the rendered items, or translate the index through a key list before storing it.
- `ListBox` and `TreeView` report keys, which are safer for reorderable data.
- The native controls virtualize automatically. Do not render ten thousand `DataItem`s if a
  `VirtualSource` describes the same data; the view cost is paid per realized row.

Next: [Navigation and Shell](08-navigation-and-shell.md) puts these controls into an app frame.
