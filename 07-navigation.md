# Navigation & Dialogs

## NavigationView

The app shell pattern: a pane of navigation items + a content area.

### Building a NavigationView

```rust
let nav_items = vec![
    NavViewItem::new("Contacts")
        .tag("contacts")
        .icon(Symbol::People),
    NavViewItem::new("Settings")
        .tag("settings")
        .icon(Symbol::Setting),
];

let content: Element = /* whatever page is currently selected */;

NavigationView::new(nav_items, content)
    .selected_tag("contacts")
    .on_selection_changed(move |tag: String| { /* update selected page */ })
    .pane_display_mode(NavigationViewPaneDisplayMode::Left)
    .pane_open(true)
    .pane_toggle_button_visible(true)
    .back_button_visible(false)
    .font_family("Segoe UI Variable")
    .into()
```

### `NavViewItem`

```rust
NavViewItem::new("Title")
    .tag("unique-tag")            // used by .selected_tag()
    .icon(Symbol::People)         // Symbol enum: Edit, Delete, Save, People, ...
```

Nested/hierarchical items:

```rust
let mut parent = NavViewItem::new("Category")
    .tag("category")
    .icon(Symbol::More);
parent = parent.child(NavViewItem::new("Child").tag("child"));
```

### Routing between pages

The gallery pattern: match on `selected_tag` and build the content element:

```rust
let content: Element = match selected_tag.as_str() {
    "home" => home_page(...),
    "settings" => component(settings_page, ()),
    _ => router::route(&selected_tag),
};
```

> ⚠️ Remember the `component()` state-propagation gotcha from
> [Architecture](02-architecture.md) — if a page owns interactive state
> (list selection, toggles), prefer lifting state to the root render fn and
> passing it as parameters.

### Fill the window

Wrap the NavView in a grid with star sizing so it fills available space:

```rust
grid((
    NavigationView::new(items, content)
        .grid_row(0),
))
.rows([GridLength::Star(1.0)])
.columns([GridLength::Star(1.0)])
.into()
```

## CommandBar

A toolbar of command buttons. From the gallery:

```rust
command_bar(vec![
    CommandBarCommandDef::Button {
        label: "Add".into(),
        icon: Some(Symbol::Add),
    },
    CommandBarCommandDef::Button {
        label: "Edit".into(),
        icon: Some(Symbol::Edit),
    },
    CommandBarCommandDef::Button {
        label: "Delete".into(),
        icon: Some(Symbol::Delete),
    },
])
.on_click({
    let set_status = set_status;
    move |label| set_status.call(format!("Last command: {label}"))
})
```

- `.on_click(closure)` receives the **label** of the clicked button as a
  `String`. Match on it to dispatch:

```rust
.on_click(move |label: String| match label.as_str() {
    "Edit" => on_edit(),
    "Delete" => on_delete(),
    _ => {}
})
```

- CommandBar works well at the top of a detail page with Edit/Delete (view
  mode) that switches to Save/Cancel (edit mode) — just conditionally render
  different command definitions based on your edit-mode state.

## ContentDialog (confirmation dialogs)

From the gallery:

```rust
let (open, set_open) = cx.use_state(false);

button("Show Dialog").on_click(move || set_open.call(true));

ContentDialog::new("Confirm Action")
    .content("Are you sure you want to proceed?")
    .primary_button_text("Yes")
    .close_button_text("No")
    .is_open(open)
    .on_closed({
        let set_open = set_open.clone();
        move |result| {
            set_open.call(false);
            match result {
                ContentDialogResult::Primary => { /* primary clicked */ }
                ContentDialogResult::Secondary => { /* secondary clicked */ }
                _ => { /* dismissed (close button / Esc) */ }
            }
        }
    })
```

- Three-button variant: `.primary_button_text("Save")` +
  `.secondary_button_text("Don't Save")` + `.close_button_text("Cancel")`.
- **You must reset `.is_open(false)` in `on_closed`** — the dialog doesn't
  auto-close its bound state.
- `ContentDialogResult` is `Primary`, `Secondary`, or none (closed).

### Delete-with-confirmation pattern (from our contacts app)

```rust
// open on Delete click
let delete_dialog: Element = ContentDialog::new("Delete Contact")
    .content("This action cannot be undone. Are you sure?")
    .primary_button_text("Delete")
    .close_button_text("Cancel")
    .is_open(delete_open)
    .on_closed({
        let set_delete_open = set_delete_open.clone();
        let set_contacts = set_contacts.clone();
        let set_selected = set_selected.clone();
        let contacts = contacts.clone();
        let idx = selected_index;
        move |result| {
            set_delete_open.call(false);
            if matches!(result, ContentDialogResult::Primary) {
                let mut new_contacts = contacts.clone();
                if idx >= 0 && (idx as usize) < new_contacts.len() {
                    new_contacts.remove(idx as usize);
                    set_contacts.call(new_contacts);
                    set_selected.call(-1);
                }
            }
        }
    })
    .into();
```

Include the dialog element in your layout tree (it renders as a modal
overlay).
