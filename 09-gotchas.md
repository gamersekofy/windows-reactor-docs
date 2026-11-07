# Pitfalls & Gotchas

A catalog of every trap we hit while building with `windows-reactor`, and how
to avoid each one.

## 1. `impl Into<Element>` vs `Element` (E0271)

**Symptom:**
```
expected `app` to return `Element`, but it returns `impl Into<Element>`
```

**Cause:** `App::render()` requires `Fn(&mut RenderCx) -> Element`. The
official intro post shows `impl Into<Element>` in example code, which is wrong.

**Fix:**
```rust
fn app(cx: &mut RenderCx) -> Element {
    vstack((...)).into()   // return Element + .into()
}
```

## 2. Empty ListView with no errors

**Symptom:** Other UI renders fine, but your ListView shows nothing — no
errors, no warnings.

**Cause:** Virtualized list never realizes its items because the layout signal
doesn't arrive inside a NavigationView.

**Fix:** `.eager_templated_realization(true)` on `App::new()`. The gallery
always sets this.

## 3. `ignoring Padding for StackPanel`

**Symptom:** Runtime warning `set_prop: ignoring Padding for StackPanel`.
Your `vstack(...).padding(...)` silently does nothing.

**Cause:** WinUI `StackPanel` has no Padding property.

**Fix:** Wrap in `border(...).padding(...)`:
```rust
border(hstack((...))).padding(Thickness { left: 12.0, top: 10.0, right: 12.0, bottom: 10.0 })
```

## 4. `no method named grid_column` on `TemplatedListBuilder`

**Symptom:** `.grid_column(0)` fails on a `list_view(...)` chain.

**Cause:** Placement methods live on `Element`, not on the builder.

**Fix:** Call `.build()` first:
```rust
list_view(items, row).build().grid_column(0)
```

## 5. `cannot compare Contact with Contact` — missing `PartialEq`

**Symptom:** `use_state` fails to compile on a custom struct.

**Cause:** `use_state` requires `T: PartialEq + Clone` (to detect no-op
updates).

**Fix:** `#[derive(Clone, PartialEq)]` on your structs (including nested
field types).

## 6. `use of moved value` on `SetState`

**Symptom:** "value moved into closure here" — a setter used in a second
closure after being captured by a first.

**Cause:** `SetState<T>` is not `Copy`; each `move` closure consumes it.

**Fix:** Clone the setter upfront once per closure:
```rust
let a = set.clone();
let b = set.clone();
let c = set;   // last use
```

## 7. Selection state doesn't propagate from inside `component()`

**Symptom:** `on_selection_changed` prints the right index, but the UI never
updates. Also: switching selection while in "edit mode" can render a broken
split view.

**Cause:** The `ComponentElement` boundary (`component(fn, props)`) doesn't
reliably propagate hook-driven state changes that originate from
TemplatedList selection callbacks to the UI.

**Fix:** Lift state to the root render function (`App::render`'s closure) and
pass values + setters down to plain functions as parameters. See
[Data & Models](08-data.md).

## 8. `DateTime` default is year 1601

**Symptom:** "Not set" dates display as `1601-01-01T00:00:00Z`.

**Cause:** `DateTime::default()` is a real date (the 1601 FILETIME epoch).

**Fix:** Use `Option<DateTime>` for optional dates and render "Not set" for
`None`.

## 9. `DateTime` epoch confusion in tests

**Symptom:** `date(1970, 1, 1).ticks() == 0` fails (actual: `116444736000000000`).

**Cause:** `ticks()` counts 100ns intervals since **1601**; Unix epoch is
1970.

**Fix:** Use `.unix_secs()` (seconds since 1970) when reasoning in Unix terms.

## 10. `initials()` takes an argument

**Symptom:** `PersonPicture::new().initials()` won't compile.

**Cause:** `.initials()` requires a `String` parameter.

**Fix:** `PersonPicture::new().display_name(&name).initials(get_initials(&name))`
with a helper that computes initials.

## 11. ContentDialog never closes

**Symptom:** Dialog stays open after clicking a button.

**Cause:** The `is_open` state isn't reset.

**Fix:** Always `set_open.call(false)` in `on_closed`, then handle the result.

## 12. ListView grows unbounded without `.height()`

**Symptom:** No scrolling — all items visible, window grows with list.

**Cause:** Without a height, the ListView measures to content.

**Fix:** Add `.height(f64)` for a fixed viewport + virtualization, or omit it
deliberately for small lists.

## 13. `DateTime` and string literals don't mix

**Symptom:** `birthday: Some("1988-11-22".into())` fails to compile.

**Cause:** `DateTime` has no `From<&str>` conversion.

**Fix:** Use `DateTime::from_unix_secs(...)` or a date helper.

## 14. Empty-string fields vs `Option` (design note)

Don't overuse `Option<String>`: `""` renders identically to "not set" and
keeps code simpler. Reserve `Option` for types whose defaults are visible
(`DateTime`, structs).

## 15. `.map_or` with function pointers

```rust
contacts.get(idx).map_or("(none)", String::as_str)
```

`String::as_str` is used as a function pointer (shorthand for
`|s| s.as_str()`). `.map_or(default, f)` returns `default` if `None`, else
`f(value)`.
