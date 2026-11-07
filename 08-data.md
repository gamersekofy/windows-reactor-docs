# Data & Models

## Structs used with `use_state`

State stored via hooks requires `Clone + PartialEq`:

```rust
#[derive(Clone, PartialEq)]
pub struct Contact {
    pub id: u64,
    pub first_name: String,
    pub last_name: String,
    pub email: String,
    pub phone: String,
    pub address: Option<Address>,
    pub birthday: Option<DateTime>,
    pub notes: String,
}

#[derive(Clone, PartialEq)]
pub struct Address {
    pub street: String,
    pub city: String,
    pub state: String,
    pub zip: String,
    pub country: String,
}
```

## Optional Fields: `Option<T>` vs empty values

| Type | Use `Option`? | Why |
|------|---------------|-----|
| `String` | No | `""` renders as nothing — same as "not set", no visual difference |
| `DateTime` | **Yes** | `DateTime::default()` is a *real* date (1601-01-01), which displays as garbage |
| `Address` (struct) | Yes | A whole-address "not set" state is meaningful; all-empty vs none differ |
| `bool` / numbers | Usually no | Defaults are fine |

Rule of thumb: use `Option` when the type's default is **visible and
meaningful** (DateTime, structs, custom types). Use bare values when the
default is invisible or semantically fine (String, numbers, bools).

## `DateTime`

`windows_reactor` re-exports `DateTime` from the `windows_time` crate.

### Key facts

- **Internal epoch is 1601-01-01 UTC** (Windows FILETIME), NOT the Unix epoch.
- Backing value: `i64` of 100-nanosecond ticks since 1601.

### Constructors

```rust
DateTime::from_unix_secs(secs: i64)      // seconds since 1970-01-01
DateTime::from_unix_millis(millis: i64)
DateTime::from_ticks(ticks: i64)         // raw 1601-based ticks
DateTime::now()                          // current time (std feature)
DateTime::UNIX_EPOCH                     // = 1970-01-01 00:00:00 UTC
```

### Accessors

```rust
d.year()        // i64
d.month()       // u32 (1 = January)
d.day()         // u32 (1-31)
d.hour()        // u32
d.minute()      // u32
d.second()      // u32
d.milliseconds()
d.day_of_week() // 0 = Sunday
d.unix_secs()   // seconds since 1970
d.ticks()       // raw 1601-based ticks
d.to_local()    // apply timezone offset (windows)
```

### Display

`DateTime` implements `Display` (ISO-8601 style):
`1992-03-15T00:00:00Z`. Use `d.to_string()` in text blocks.

### Handy date helper (for fixed dates in sample data)

```rust
fn date(year: i64, month: u32, day: u32) -> DateTime {
    let leap_days = |y: i64| y / 4 - y / 100 + y / 400;
    let days_before = |y: i64| (y - 1) * 365 + leap_days(y - 1);
    let month_offsets: [i64; 12] = [0, 31, 59, 90, 120, 151, 181, 212, 243, 273, 304, 334];
    let is_leap = year % 4 == 0 && (year % 100 != 0 || year % 400 == 0);
    let extra: i64 = if is_leap && month > 2 { 1 } else { 0 };
    let total_days = days_before(year) + month_offsets[month as usize - 1] + extra + day as i64 - 1;
    DateTime::from_unix_secs((total_days - days_before(1970)) * 86400)
}
```

> Unit-test the helper: `date(1970, 1, 1).unix_secs() == 0` (the `ticks()`
> value at the Unix epoch is `116444736000000000`, not 0!).

## State Management Patterns

### The "lift state to root" pattern (our app)

Because of the `component()` boundary issue (see
[Architecture](02-architecture.md)), our contacts app keeps **all state on the
root render function** and passes values + setters down to layout functions:

```rust
// main.rs — root render fn owns state
fn contacts_shell(cx: &mut RenderCx) -> Element {
    let (selected_contact, set_selected) = cx.use_state(-1_i32);
    let (contacts, set_contacts) = cx.use_state(initial_contacts());
    let (edit_mode, set_edit_mode) = cx.use_state(false);
    let (edit_draft, set_edit_draft) = cx.use_state(None::<Contact>);
    let (delete_open, set_delete_open) = cx.use_state(false);

    let content = pages::contacts::contacts_split_layout(
        contacts.clone(),
        selected_contact,
        set_selected,
        set_contacts,
        edit_mode,
        set_edit_mode,
        edit_draft,
        set_edit_draft,
        delete_open,
        set_delete_open,
    );
    // ... NavigationView wrapper
}

// contacts.rs — layout fn takes state as parameters
pub fn contacts_split_layout(
    contacts: Vec<Contact>,
    selected_contact: i32,
    set_selected_contact: SetState<i32>,
    set_contacts: SetState<Vec<Contact>>,
    edit_mode: bool,
    set_edit_mode: SetState<bool>,
    edit_draft: Option<Contact>,
    set_edit_draft: SetState<Option<Contact>>,
    delete_open: bool,
    set_delete_open: SetState<bool>,
) -> Element { /* ... */ }
```

### Cloning `SetState` for multiple closures

`SetState<T>` is not `Copy`. Every `move` closure that needs a setter must get
its own clone. **Clone them all upfront** to avoid borrow-move errors:

```rust
let ss = set_selected.clone();     // for dialog
let ss2 = set_selected;            // last use — no clone needed
let sc = set_contacts.clone();     // for save closure
let sc2 = set_contacts.clone();    // for delete dialog
let sc3 = set_contacts;            // for list_view (last use)
// ... etc
```

### Derived values (not hooks)

```rust
let contact_count = contacts.len() as i32;
let has_selection = selected_contact >= 0 && selected_contact < contact_count;
```

Computed inline each render — cheap, no hook needed.

### Safe indexing

Always bounds-check before indexing state arrays:

```rust
contacts.get(selected_contact.max(0) as usize)
    .map_or("(none)".to_string(), |c| display_name(c))
```

or guard with a `has_selection` flag before `contacts[idx as usize]`.
