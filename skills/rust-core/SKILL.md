---
name: rust-core
description: Design, implement, refactor, and review idiomatic Rust code, covering ownership, borrowing, lifetimes, type-driven design, traits and generics, error handling, iterators, modules and visibility, public API design, semver, Cargo workspaces, features, MSRV, editions, lints, and documentation. Use when writing or changing any Rust code, designing a crate or module API, choosing between generics and trait objects, defining error types, or organizing a Cargo workspace.
license: MIT
---

# Rust Core

Use this skill when writing, refactoring, or reviewing Rust code in any domain. It covers language-level and crate-level decisions. Use the specialized Rust skills for async, backend services, persistence, performance, unsafe code, testing, and reviews.

Inspect the repository before changing code. Check `Cargo.toml`, the workspace layout, `rust-toolchain.toml`, the edition, the declared `rust-version`, `clippy.toml`, `rustfmt.toml`, CI lint settings, and existing error and module conventions. Follow what is there. Do not introduce new crates, macros, or architectural patterns when the existing toolset already solves the problem.

## General Principles

- Let the type system carry invariants. A rule enforced by a type cannot be forgotten at a call site.
- Prefer simple, concrete code. Generics, traits, macros, and lifetimes are tools with real costs in compile time, readability, and error messages.
- Make the common path obvious and the dangerous path explicit.
- Optimize for the reader who has to change this code in two years.
- Preserve working behavior when modifying existing code unless the task explicitly changes it.

## Ownership And Borrowing

- Design function signatures around ownership intent: take `&T` to read, `&mut T` to modify, and `T` to consume or store.
- Accept the most general borrowed form: `&str` over `&String`, `&[T]` over `&Vec<T>`, `&Path` over `&PathBuf`.
- Accept `impl Into<String>` or `impl AsRef<Path>` only when it meaningfully improves call-site ergonomics. It also adds monomorphization.
- Do not sprinkle `.clone()` to silence the borrow checker. First check whether the data flow is wrong. Cloning is fine when it is cheap, such as `Arc`, small `Copy`-like data, or a deliberate snapshot, and the intent is clear.
- Restructure code before reaching for interior mutability. Split borrows by destructuring structs, narrow scopes, or move the mutation to the owner.
- Treat `Rc<RefCell<T>>` or `Arc<Mutex<T>>` graphs as a design smell outside of genuinely shared, mutable state. Prefer ownership trees, indices into arenas, or message passing.
- Use `Cow<'_, T>` when a function usually borrows but sometimes needs to own.
- Use `std::mem::take`, `replace`, and `Option::take` to move out of `&mut` without cloning.

## Lifetimes

- Rely on elision. Add explicit lifetimes only when the compiler requires them or they document a real relationship.
- Avoid structs that borrow (`struct Parser<'a>`) unless they are short-lived views or the zero-copy benefit is measured. Borrowing structs propagate lifetimes through every user.
- Do not use `'static` bounds to make errors go away. Understand whether the value must outlive a spawned task or thread, or just needs to be owned.
- Never use `Box::leak` or `unsafe` to fake a lifetime outside of deliberate, process-lifetime singletons.

## Type-Driven Design

- Use newtypes for domain identifiers and validated values (`struct UserId(Uuid)`, `struct Email(String)`). Validate in the constructor and keep the field private.
- Make invalid states unrepresentable with enums instead of combinations of booleans and `Option` fields.
- Use the typestate pattern for multi-step protocols when misuse is costly. Do not use it for simple flows where it only adds generic noise.
- Parse, don't validate. Convert untrusted input into typed values once at the boundary, then pass typed values inward.
- Prefer `enum` over stringly typed values. Implement `FromStr` and `Display` for round-tripping.
- Use `#[must_use]` on functions and types whose results must not be ignored, such as builders, guards, and computed results.

```rust
pub struct Email(String);

impl Email {
    pub fn parse(raw: &str) -> Result<Self, InvalidEmail> {
        let trimmed = raw.trim();
        if trimmed.contains('@') && !trimmed.starts_with('@') {
            Ok(Self(trimmed.to_owned()))
        } else {
            Err(InvalidEmail)
        }
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}
```

## Traits And Generics

- Use generics (`fn f<T: Trait>` or `impl Trait`) when performance matters, the set of types is open, and the call sites are few.
- Use trait objects (`&dyn Trait`, `Box<dyn Trait>`) for heterogeneous collections, plugin boundaries, reducing compile time and binary size, or when the type is chosen at runtime.
- Do not create a trait with a single implementation just for mocking. Introduce one when a real seam exists, such as an external system, an alternate backend, or a test double that replaces I/O.
- Keep traits small and cohesive. Large traits are hard to implement and hard to fake.
- Put bounds on `impl` blocks and functions, not on struct definitions, unless the struct cannot exist without them.
- Implement standard traits deliberately: `Debug` almost always, `Clone`, `PartialEq`, `Eq`, `Hash`, `Default` when meaningful, and `Display` for user-facing values. Implement `From` for infallible conversions and `TryFrom` for fallible ones.
- Do not implement `Deref` to emulate inheritance. Reserve it for smart-pointer-like types.
- Use associated types when there is one natural output per implementation. Use generic parameters when a type can implement the trait several ways.
- Be aware of object safety. Generic methods and `Self` returns prevent `dyn` use unless bounded with `where Self: Sized`.

## Error Handling

- Use `Result` for recoverable failures and panics for broken invariants and programmer errors.
- Libraries define structured error types, commonly with `thiserror`. Callers must be able to match on meaningful variants.
- Applications and binaries may use `anyhow` or `eyre` at the top level, adding context with `.context(...)` and `.with_context(...)`.
- Do not expose `anyhow::Error` from a library's public API.
- Do not use `unwrap()` or `expect()` on fallible operations in library or request-path code. Use `expect("reason")` only when failure means a violated invariant, and state the invariant in the message.
- Propagate with `?` and convert with `From` implementations or `map_err`. Do not lose the source error. Chain it through `#[source]` or `#[from]`.
- Keep error variants meaningful to the caller. Do not create one variant per internal call site, and do not collapse everything into `Other(String)`.
- Do not log and return the same error. Choose one place, usually the boundary, to report it.
- Mark public error enums `#[non_exhaustive]` when new variants are likely.
- Error messages are lowercase without trailing punctuation, per Rust convention, and describe what failed, not what to do.

```rust
#[derive(Debug, thiserror::Error)]
#[non_exhaustive]
pub enum ConfigError {
    #[error("failed to read config file {path}")]
    Read {
        path: PathBuf,
        #[source]
        source: std::io::Error,
    },
    #[error("invalid value for {key}: {reason}")]
    Invalid { key: &'static str, reason: String },
}
```

## Panics

- Indexing (`v[i]`), slicing, `unwrap`, integer division by zero, `RefCell` borrow conflicts, and arithmetic overflow in debug builds can all panic. Use `get`, `checked_*`, `saturating_*`, or `wrapping_*` where input is not trusted.
- Choose overflow semantics explicitly for arithmetic on untrusted or accumulating values. Release builds wrap silently unless `overflow-checks = true` is set.
- Do not panic across FFI boundaries or inside `Drop` implementations.
- A panic in a spawned thread or task does not crash the process by default. Make sure join handles are inspected or failures are surfaced.

## Iterators And Closures

- Prefer iterator chains when they read clearly. Prefer a `for` loop when the chain needs complex control flow, early returns with side effects, or nested `?`.
- Avoid collecting intermediate `Vec`s that are immediately iterated again.
- Use `collect::<Result<Vec<_>, _>>()` to short-circuit on the first error.
- Return `impl Iterator<Item = T>` from functions instead of allocating, when the borrow structure allows.
- Keep closures small. Extract named functions when logic grows or needs testing.

## Modules And Visibility

- Organize modules by domain feature, not by kind (`models`, `utils`, `helpers`).
- Default to private. Use `pub(crate)` for crate-internal sharing and `pub` only for the intended API.
- Re-export the public API from the crate root or a curated `prelude` only if the crate already uses that convention.
- Keep `mod.rs` versus `foo.rs` plus `foo/` consistent with the repository.
- Avoid glob imports outside of preludes and test modules.

## Public API Design

- Follow the Rust API Guidelines: conventional naming (`as_`, `to_`, `into_`, `try_`, `iter`, `iter_mut`, `into_iter`), common trait implementations, and `Send + Sync` where possible.
- Every public item is a semver commitment. Keep the surface small.
- Use `#[non_exhaustive]` on public structs and enums that will grow. Know that it forces wildcard arms and prevents struct literal construction downstream.
- Use sealed traits when downstream crates may call but not implement a trait.
- Use builders for types with many optional parameters. Do not use them for types with two required fields.
- Do not leak dependency types through the public API unless the dependency is a deliberate part of the contract. Leaked types couple your semver to theirs.
- Prefer taking closures or generic parameters over callback traits for one-off hooks.
- Document panics, errors, and safety requirements under `# Panics`, `# Errors`, and `# Safety`.

## Semver Awareness

Treat these as breaking changes in a published crate:

- Removing or renaming public items.
- Adding a variant to an exhaustive public enum.
- Adding a public field to a struct without `#[non_exhaustive]` when all fields were public.
- Adding a required method to a public trait without a default implementation.
- Tightening generic bounds.
- Changing auto-trait behavior (`Send`, `Sync`, `Unpin`) by adding a non-`Send` field.
- Raising MSRV, unless the crate's policy says otherwise.
- Bumping a public dependency's major version.

Use `cargo semver-checks` in CI for published libraries where practical.

## Cargo, Workspaces, And Features

- Use a workspace for multi-crate repositories with shared `[workspace.dependencies]`, `[workspace.package]`, and `[workspace.lints]` when the repository already uses them or the duplication is real.
- Split crates for real reasons: compile-time parallelism, separate publishing, clear dependency direction, or isolating heavy dependencies. Do not split one crate per layer by default.
- Features must be additive. Enabling a feature must never remove API or change behavior incompatibly, because Cargo unifies features across the dependency graph.
- Do not create mutually exclusive features. Use separate crates or runtime configuration instead.
- Gate optional heavy dependencies behind features with `dep:` syntax to avoid implicit feature names.
- Keep `default-features` minimal for libraries.
- Commit `Cargo.lock` for binaries and applications. Follow the repository's policy for libraries.
- Pin a toolchain with `rust-toolchain.toml` when reproducible builds matter.
- Declare `rust-version` and test it in CI when publishing a library with an MSRV policy.
- Review new dependencies for maintenance, transitive weight, `unsafe` usage, licenses, and supply-chain risk. Prefer the standard library when it is sufficient.

## Editions And Language Features

- Use the edition already set by the repository. Edition migrations are separate changes, done with `cargo fix --edition`.
- Use stable language features already accepted by the repository: let-else, if-let chains where the edition supports them, GATs, `impl Trait` in return position, and async functions in traits.
- Do not introduce nightly features into a stable codebase.

## Macros

- Prefer functions and generics. Use `macro_rules!` when it removes real repetition that functions cannot express.
- Use procedural macros only when the benefit is large. They add compile time, tooling friction, and debugging difficulty.
- Keep macro-generated code hygienic and refer to paths with `$crate::`.

## Lints And Formatting

- Run `cargo fmt` and `cargo clippy --all-targets --all-features` and keep them clean in the configuration the repository uses.
- Fix clippy findings instead of suppressing them. Add `#[allow(...)]` or `#[expect(...)]` only at the narrowest scope, with a comment explaining why.
- Consider `#![deny(unsafe_code)]` or `#![forbid(unsafe_code)]` in crates that should never contain `unsafe`.
- Do not enable `clippy::pedantic` or `clippy::nursery` wholesale in an existing codebase as part of an unrelated change.

## Documentation

- Document every public item with what it does, not how it does it.
- Include runnable examples in doc comments for important APIs. Doctests are tests.
- Use intra-doc links (`[`Type`]`, `[`method`](Type::method)`).
- Add crate-level docs (`//!`) that explain purpose, key types, and feature flags.
- Write comments that explain why. Do not narrate what the code obviously does.

## Refactoring Existing Code

- Read the module and its callers before changing a type or signature.
- Make the smallest coherent change. Separate mechanical renames and moves from behavior changes.
- Lean on the compiler. Change a type, then fix the errors it reports instead of grepping and guessing.
- Avoid changing public signatures in published crates unless the task requires a breaking change.

## Implementation Workflow

- Inspect `Cargo.toml`, workspace layout, toolchain, edition, lints, and existing conventions.
- Identify the ownership and data flow before writing code.
- Design types and signatures first, then implement.
- Run `cargo check`, `cargo clippy --all-targets`, and `cargo fmt --check`.
- Run `cargo test` for the affected crates, including doctests.
- Review `unwrap`, `expect`, indexing, arithmetic, error propagation, and public API impact before finishing.

## Checklist

- Does the change follow the repository's edition, MSRV, lints, and conventions?
- Do signatures express ownership intent with the most general borrowed types?
- Are clones deliberate rather than borrow-checker workarounds?
- Are invariants enforced by types where practical?
- Are errors structured for libraries, with sources preserved and context added?
- Are `unwrap`, `expect`, indexing, and arithmetic safe for the inputs they receive?
- Are generics versus trait objects chosen deliberately?
- Is visibility as narrow as possible?
- Does the change avoid unintended semver breaks?
- Are features additive and new dependencies justified?
- Are public items documented, with errors, panics, and safety sections where relevant?
- Do `cargo check`, `cargo clippy`, `cargo fmt --check`, and `cargo test` pass?
