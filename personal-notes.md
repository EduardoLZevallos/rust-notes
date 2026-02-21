# Personal Rust Notes

Notes from working on the Lemmy codebase. Written for someone coming from Python.

---

## Crates

A **crate** is Rust's unit of compilation — roughly like a Python package/library.

- A crate is either a **library** (code others use) or a **binary** (an executable program).
- A large project like Lemmy is split into many crates (e.g. `lemmy_db_schema`, `lemmy_api`, `lemmy_utils`).
- Crates are managed by **Cargo** (`Cargo.toml`), similar to how Python uses `pip` + `pyproject.toml`.
- Published to [crates.io](https://crates.io), similar to PyPI.

---

## Visibility: `pub`, `pub(crate)`, private

Controls who can access a field, function, or struct.

| Keyword       | Who can access it                        | Python analogy         |
|---------------|------------------------------------------|------------------------|
| (nothing)     | Only the current module                  | `__name` (name mangled)|
| `pub(crate)`  | Any code in the same crate               | `_name` (convention)   |
| `pub`         | Anyone, including other crates           | No restriction         |

Example from Lemmy:
```rust
pub struct ModlogInsertForm<'a> {
  pub(crate) kind: ModlogKind,  // only accessible within lemmy_db_schema crate
  pub bulk: bool,               // accessible from any crate
}
```

**Why it matters:** If a field is `pub(crate)` in crate A, code in crate B cannot read or write it — the compiler will refuse to compile with error `field is private`.

---

## Structs

Rust structs are like Python dataclasses but with strict typing and no runtime flexibility.

```rust
pub struct Modlog {
  pub id: ModlogId,
  pub kind: ModlogKind,
  pub bulk: bool,
}
```

Python equivalent:
```python
@dataclass
class Modlog:
    id: ModlogId
    kind: ModlogKind
    bulk: bool
```

---

## `derive` macros

Rust doesn't automatically give your types common behaviour — you opt in with `#[derive(...)]`.

```rust
#[derive(Clone, PartialEq, Eq, Debug, Serialize, Deserialize)]
pub struct Modlog { ... }
```

| Derive          | What it gives you                              | Python equivalent           |
|-----------------|------------------------------------------------|-----------------------------|
| `Clone`         | `.clone()` method to copy a value              | `copy.copy()`               |
| `Debug`         | `{:?}` formatting for printing                 | `__repr__`                  |
| `PartialEq`/`Eq`| `==` comparison                               | `__eq__`                    |
| `Serialize`     | Convert to JSON/etc (via `serde`)              | `json.dumps()`              |
| `Deserialize`   | Parse from JSON/etc (via `serde`)              | `json.loads()`              |

---

## `derive_new` and `#[new(default)]`

The `derive_new::new` macro auto-generates a constructor function for a struct.

`#[new(default)]` tells it: "don't require this field in the constructor — fill it with the default value instead."

For a `bool`, the default is `false`. For `Option<T>`, the default is `None`.

```rust
#[derive(derive_new::new)]
pub struct ModlogInsertForm<'a> {
  pub(crate) kind: ModlogKind,   // required in constructor
  pub(crate) is_revert: bool,    // required in constructor
  #[new(default)]
  pub(crate) bulk: bool,         // NOT required — defaults to false
}

// Generated constructor looks like:
// ModlogInsertForm::new(kind, is_revert, mod_id) -> ModlogInsertForm
```

---

## `Option<T>`

Rust has no `None`/`null` unless you explicitly use `Option`.

```rust
pub reason: Option<String>  // can be Some("text") or None
```

Python equivalent: `reason: str | None = None`

---

## `cfg_attr` — conditional compilation

Attributes like `#[cfg_attr(feature = "full", derive(Queryable))]` mean:
"only apply this if the `full` feature flag is enabled at compile time."

This is how Lemmy keeps the database layer (`diesel`) out of the API-only builds.

---

## Diesel ORM

Diesel is Rust's main ORM (like SQLAlchemy for Python).

Key difference from Python ORMs:
- **Migration-first**: you write raw SQL migrations, then run them, and the schema is generated.
- Lemmy uses `lemmy_server migration --all run` (NOT `diesel CLI`) to run migrations.
- The `schema.rs` file is auto-generated from the database and must match the DB exactly.

---

## Lifetimes `'a`

```rust
pub struct ModlogInsertForm<'a> {
  pub(crate) reason: Option<&'a str>,
}
```

The `'a` is a **lifetime annotation** — it tells the compiler that the borrowed string reference (`&str`) must live at least as long as the struct. This prevents use-after-free bugs at compile time.

Python doesn't have this concept because it uses a garbage collector.

---

## Error: `field is private`

```
error[E0616]: field `bulk` of struct `ModlogInsertForm` is private
```

This means code in crate B is trying to access a `pub(crate)` (or private) field defined in crate A. Fix: either make the field `pub`, or move the accessing code into the same crate.

---

## Cargo commands

| Command                                  | What it does                                 |
|------------------------------------------|----------------------------------------------|
| `cargo build`                            | Compile the project                          |
| `cargo test --workspace`                 | Run all tests across all crates              |
| `cargo test --workspace --no-fail-fast`  | Keep running even if a test fails            |
| `cargo clippy`                           | Linter (like `ruff` for Python)              |
| `cargo fmt`                              | Auto-formatter (like `black` for Python)     |
| `cargo install <name>`                   | Install a binary crate (like `pip install`)  |
