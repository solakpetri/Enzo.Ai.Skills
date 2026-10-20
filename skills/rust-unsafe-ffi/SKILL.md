---
name: rust-unsafe-ffi
description: Write, audit, and review unsafe Rust and foreign function interfaces, covering when unsafe is justified, soundness and safe abstractions, safety contracts and SAFETY comments, undefined behavior, aliasing and validity rules, raw pointers and provenance, uninitialized memory, transmute alternatives, Send and Sync implementations, panic safety, Drop and Pin, FFI with C and C++, ownership across language boundaries, callbacks, -sys crates, bindgen, cbindgen, cxx, language bindings, and verification with Miri, sanitizers, and fuzzing. Use when adding or reviewing unsafe blocks, unsafe traits or functions, raw pointer code, FFI bindings, or interop layers.
license: MIT
---

# Rust Unsafe And FFI

Use this skill when writing, auditing, or reviewing `unsafe` Rust or foreign function interfaces.

Inspect the repository first. Check whether crates use `#![forbid(unsafe_code)]` or `#![deny(unsafe_code)]`, which lints are enforced (`unsafe_op_in_unsafe_fn`, `clippy::undocumented_unsafe_blocks`), how existing unsafe code documents its invariants, whether Miri or sanitizers run in CI, and how bindings are generated. Follow those conventions. Never weaken a crate-level `forbid(unsafe_code)` to make a change compile without an explicit decision.

## When Unsafe Is Justified

- Use `unsafe` only when safe Rust cannot express the operation: FFI, hardware or OS interfaces, implementing low-level data structures or synchronization primitives, or a measured performance need that safe code cannot meet.
- Search for a safe alternative first: standard library APIs, `bytemuck`, `zerocopy`, `crossbeam`, `parking_lot`, `memchr`, `once_cell` and `std::sync::OnceLock`, or an existing audited crate.
- Do not use `unsafe` to work around the borrow checker in ordinary application code. That is almost always a design problem.
- Do not use `unsafe` to skip bounds checks without a profile showing they matter.

## Soundness

- A safe function is sound if no sequence of safe calls, with any inputs, can cause undefined behavior. Soundness is a property of the whole API, not of one block.
- The module privacy boundary is the unit of soundness. Any safe code inside the module that can break the invariant (for example, by writing a private `len` field) is part of the unsafe surface.
- Keep unsafe code in small, focused modules with private fields and a minimal safe public API.
- Mark a function `unsafe fn` when callers must uphold a precondition the function cannot check. Document the precondition in a `# Safety` section.
- Mark a trait `unsafe trait` when incorrect implementations can cause undefined behavior in code that relies on them.
- Never expose a safe function that is only sound "if used correctly".

## Safety Comments

- Write a `// SAFETY:` comment on every `unsafe` block explaining why each requirement of each unsafe operation is met at that point.
- Document every `unsafe fn` and `unsafe trait` with a `# Safety` section listing the caller's or implementer's obligations.
- Keep one unsafe operation per block where practical (`clippy::multiple_unsafe_ops_per_block`) so each justification is precise.
- Enable `unsafe_op_in_unsafe_fn` so unsafe operations inside `unsafe fn` still need explicit blocks and comments. It warns by default in the 2024 edition.

```rust
/// Returns the element at `index` without bounds checking.
///
/// # Safety
///
/// `index` must be less than `self.len()`.
pub unsafe fn get_unchecked(&self, index: usize) -> &T {
    // SAFETY: the caller guarantees `index < self.len`, and `self.ptr`
    // points to `self.len` initialized elements for the lifetime of `&self`.
    unsafe { &*self.ptr.as_ptr().add(index) }
}
```

## Undefined Behavior To Rule Out

Every unsafe block must rule out, at minimum:

- Dereferencing dangling, null, or misaligned pointers.
- Reading uninitialized memory as a typed value.
- Producing invalid values: a `bool` that is not 0 or 1, an invalid `enum` discriminant, an invalid `char`, a null or dangling reference, or a `str` that is not UTF-8.
- Violating aliasing rules: a `&mut T` must be unique while it is live, and data behind `&T` must not be mutated except through `UnsafeCell`.
- Data races. Concurrent unsynchronized access where at least one access is a write.
- Out-of-bounds pointer arithmetic, even without dereferencing (`ptr.add` must stay within the allocation, or one past the end).
- Calling a function through the wrong ABI or signature.
- Unwinding out of a function whose ABI does not permit it.

## Raw Pointers And Provenance

- Prefer `NonNull<T>` over `*mut T` for non-null owning pointers. It is covariant and enables niche optimization.
- Create raw pointers with `&raw const` and `&raw mut` (or `ptr::addr_of!`) when a reference to the place would be invalid, such as unaligned fields or uninitialized memory.
- Do not create intermediate references to memory you only intend to access through raw pointers. Creating the reference asserts validity and uniqueness.
- Follow strict provenance: use `ptr.addr()`, `ptr.with_addr()`, `ptr.map_addr()`, and `ptr::without_provenance()` instead of round-tripping pointers through `usize` casts.
- Use `ptr::read_unaligned` and `write_unaligned` for packed or byte-buffer data.
- Use `PhantomData` to express ownership, lifetimes, and variance for types that hold raw pointers.

## Uninitialized Memory

- Use `MaybeUninit<T>` for uninitialized memory. Never use `mem::uninitialized`, and never use `mem::zeroed` for types where all-zero bytes are not a valid value (references, `NonNull`, `Box`, most enums).
- Initialize fully before calling `assume_init`.
- Use `Vec::spare_capacity_mut` and then `set_len` only after the elements are initialized.

## Transmute And Type Punning

- Avoid `mem::transmute`. Use `from_ne_bytes`, `to_bits` or `from_bits`, `as` casts for numbers, `bytemuck::cast` or `zerocopy` for plain-old-data, and pointer casts with `cast()` for pointers.
- If `transmute` is unavoidable, annotate both types explicitly (`transmute::<A, B>`) and document why the layouts and validity invariants match.
- Never transmute between types whose layout Rust does not guarantee. Default `repr(Rust)` struct layout is unspecified.

## Send And Sync

- Implement `Send` and `Sync` manually only when you can prove the type's thread-safety. Write a `SAFETY` comment describing which synchronization makes it correct.
- Raw pointers make a type `!Send` and `!Sync` by default for good reason. Do not add `unsafe impl Send` just to satisfy `tokio::spawn`.
- Bound generic parameters correctly: `unsafe impl<T: Send> Send for MyBox<T> {}`, not an unconditional implementation.

## Panic Safety

- Assume any call into user code, trait methods, closures, or allocation can panic.
- Keep data structures in a valid state at every point where a panic can unwind. Commit length changes, such as `set_len`, only after elements are initialized, and use guard structs whose `Drop` restores invariants.
- Do not panic inside `Drop`, and do not let a panic during unwinding trigger another panic.

## Drop, ManuallyDrop, And Pin

- Use `ManuallyDrop` to control drop order or transfer ownership to foreign code. Make sure every value is eventually dropped or deliberately leaked.
- Remember that leaking is safe. Code must remain sound if a destructor never runs (`mem::forget`).
- Respect pinning guarantees for self-referential or address-sensitive types. Use `pin-project` or `pin-project-lite` instead of hand-written pin projections.

## FFI Fundamentals

- Use `#[repr(C)]` on every struct and enum shared with foreign code. Use `#[repr(transparent)]` for newtypes over FFI types.
- Use `std::ffi` and `core::ffi` types: `c_int`, `c_char`, `CStr`, `CString`, and `c_void`. Never assume `c_long` or `c_char` sizes and signedness.
- Declare foreign functions in `unsafe extern "C" { ... }` blocks (required in the 2024 edition). Mark functions as `safe fn` inside the block only when every possible call is sound.
- Use `#[unsafe(no_mangle)]` and `extern "C" fn` for functions exported to other languages.
- Validate everything received from foreign code: null pointers, lengths, UTF-8, and enum discriminants. Do not convert a C integer into a Rust `enum` without checking the value.
- Use `Option<NonNull<T>>`, `Option<&T>`, and `Option<extern "C" fn(...)>` for nullable pointers and callbacks. They have the same layout as nullable C pointers.

## Ownership Across The Boundary

- Make ownership explicit for every pointer crossing the boundary: who allocates, who frees, and for how long the pointer is valid.
- Free memory with the same allocator that allocated it. Export a `my_lib_free_*` function for memory Rust allocated. Never call C `free` on a Rust `Box` or the reverse.
- Transfer ownership to C with `Box::into_raw` and reclaim it with `Box::from_raw` exactly once.
- Keep `CString` values alive for as long as C uses their pointer. `CString::new(s)?.as_ptr()` on a temporary dangles immediately.
- Document whether returned borrowed pointers remain valid after further calls.

## Panics And Errors Across FFI

- Never let a panic unwind into foreign code. Since Rust 1.81, unwinding out of an `extern "C"` function aborts the process. Wrap exported functions in `std::panic::catch_unwind` and convert panics into error codes when an abort is unacceptable.
- Use `extern "C-unwind"` only when both sides are designed to propagate unwinding.
- Translate Rust errors into the foreign convention (error codes, out-parameters, or a thread-local last-error) and document it.
- Check foreign return codes and `errno` (`std::io::Error::last_os_error()`) immediately after the call.

## Callbacks And Threads

- Pass Rust closures to C as an `extern "C"` trampoline plus a `*mut c_void` user-data pointer. Make the lifetime and thread affinity of the user data explicit.
- Check whether the C library is thread-safe, and wrap non-thread-safe handles in a type that is `!Send` or synchronized.
- Do not call back into Rust from foreign threads unless the Rust side is prepared for it, for example by not assuming an async runtime context.

## Binding Generation And Crate Structure

- Put raw bindings in a `foo-sys` crate with `links = "foo"` and a `build.rs` that locates or builds the native library. Put the safe API in a separate `foo` crate.
- Use `bindgen` to generate Rust bindings from C headers. Commit the generated bindings or generate them at build time, following the repository's choice and the cost of requiring libclang.
- Use `cbindgen` to generate C headers for Rust libraries exposed to C.
- Use `cxx` for C++ interop instead of hand-written C shims when it fits. Use `pyo3`, `napi-rs`, `uniffi`, or `jni` for language bindings, following each tool's safety model.
- Keep the safe wrapper the only public API. Every safe method upholds the C library's preconditions.

## Static Mut And Globals

- Avoid `static mut`. References to it are denied by default in the 2024 edition. Use atomics, `Mutex`, `OnceLock`, or `UnsafeCell` wrappers with documented synchronization.
- Use `&raw mut STATIC` if raw access to a `static mut` is truly required, such as for FFI.

## Verification

- Run the test suite under Miri (`cargo +nightly miri test`) for crates with meaningful unsafe code. Consider `-Zmiri-strict-provenance`, and test with both Stacked Borrows and Tree Borrows when the aliasing model is in doubt.
- Run AddressSanitizer, ThreadSanitizer, or MemorySanitizer (`-Zsanitizer=...` on nightly) for FFI and concurrency code that Miri cannot execute.
- Fuzz unsafe parsers and decoders with `cargo fuzz` and keep the corpus.
- Use `loom` for custom synchronization primitives.
- Track unsafe usage across dependencies with `cargo geiger` when supply-chain risk matters.
- Write tests that target the edges: empty input, zero-sized types, maximum lengths, alignment boundaries, and panics in user callbacks.

## Audit Workflow

1. List every `unsafe` block, `unsafe fn`, `unsafe impl`, and `extern` declaration in scope.
2. For each, identify the unsafe operations and the exact requirements from the standard library or foreign documentation.
3. Verify that the `SAFETY` comment states why each requirement holds, and that the reasoning is true for all safe callers.
4. Inspect all safe code in the same module that can affect the invariants.
5. Check panic safety at every call that can unwind.
6. Check `Send`, `Sync`, and variance of types holding raw pointers.
7. For FFI, verify layouts, nullability, ownership, lifetimes, error handling, unwinding, and thread safety.
8. Run Miri, sanitizers, or fuzzing as applicable.
9. Report unsound APIs as critical, even when no current caller triggers the undefined behavior.

## Review Checklist

- Is unsafe necessary, or does a safe or audited alternative exist?
- Is the unsafe code isolated in a small module behind a sound safe API?
- Does every unsafe block have an accurate `SAFETY` comment?
- Do `unsafe fn` and `unsafe trait` items document their safety contracts?
- Are aliasing, validity, alignment, bounds, and initialization requirements met?
- Are raw pointers created without invalid intermediate references, and with provenance preserved?
- Are `Send` and `Sync` implementations justified and correctly bounded?
- Is the code panic-safe?
- Are FFI types `#[repr(C)]` and validated on entry?
- Is ownership across the boundary explicit, with matching allocators?
- Are panics prevented from unwinding across FFI?
- Is foreign thread safety respected?
- Is the code tested under Miri, sanitizers, or fuzzing where applicable?
