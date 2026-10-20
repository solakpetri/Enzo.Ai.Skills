---
name: rust-testing
description: Create, improve, debug, and review automated tests for Rust code, covering unit tests, integration tests, doctests, async tests, HTTP and database tests, test doubles, property-based testing, snapshot testing, concurrency model checking with loom, undefined-behavior detection with Miri, fuzzing, benchmarks, compile-fail tests, feature matrices, nextest, and coverage. Use when writing or reviewing tests, test fixtures, mocks, fakes, or test infrastructure in a Rust project, or when diagnosing flaky or ineffective Rust tests.
license: MIT
---

# Rust Testing

Use this skill when creating, debugging, or reviewing automated tests in Rust projects.

Inspect the existing tests before writing new ones. Check where unit and integration tests live, which test helpers and fixtures exist, which crates are in `[dev-dependencies]` (`mockall`, `proptest`, `insta`, `rstest`, `wiremock`, `testcontainers`, `sqlx`), whether CI uses `cargo nextest`, and which feature combinations CI tests. Follow those conventions. Do not introduce a new test framework or mocking library when the repository already has one.

## Testing Philosophy

- Test behavior, not implementation details.
- Prefer meaningful tests over coverage percentages.
- Let the type system remove whole classes of tests. Do not test what the compiler already guarantees. Spend the effort on logic, boundaries, and integration.
- Keep tests deterministic and independent of execution order.
- Remember that tests in one binary run in parallel threads by default. Shared global state (environment variables, current directory, files, ports, static caches) causes flakiness.

## Choosing The Test Level

- Start with the behavior to prove, then choose the smallest test level that proves it reliably.
- Use unit tests for pure logic, parsing, state machines, and invariants.
- Use integration tests in `tests/` for the public API of a crate, or when behavior depends on wiring, persistence, serialization, or configuration.
- Use HTTP-level tests when routing, extractors, middleware, auth, status codes, or error bodies matter.
- Use end-to-end tests sparingly for cross-process behavior.
- Do not force everything into unit tests if the important risk is integration behavior.

## Unit Tests

- Place unit tests in a `#[cfg(test)] mod tests` block in the same file. They can access private items, which is appropriate for testing internal logic with a stable contract.
- Name tests after the behavior: `rejects_order_without_lines`, not `test_validate_2`.
- Return `Result<(), E>` from tests to use `?` instead of chains of `unwrap()`.
- Use `assert_eq!` and `assert!` with messages that explain the expectation when failures would otherwise be ambiguous.
- Use `#[should_panic(expected = "...")]` only for functions whose contract is to panic. Always include `expected`.
- Use `rstest` for parameterized cases if the repository uses it. Otherwise a table of cases in a loop works, as long as each case is identifiable in the failure message.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn rejects_quantity_above_limit() {
        let err = OrderQuantity::try_from(1_001).unwrap_err();
        assert_eq!(err, QuantityError::TooLarge { max: 1_000 });
    }
}
```

## Integration Tests

- Each file in `tests/` compiles as a separate crate that can only use the public API. Many files mean many link steps. Group related tests into modules of one test binary when compile time matters.
- Put shared helpers in `tests/common/mod.rs` (not `tests/common.rs`, which would be treated as a test crate).
- Binary crates cannot be imported by integration tests. Keep logic in `lib.rs` with a thin `main.rs`.
- Isolate state per test: unique temporary directories (`tempfile`), separate databases or schemas, and ephemeral ports (bind to `127.0.0.1:0`).

## Doctests

- Treat examples in public documentation as tests. They prove the documented API compiles and works.
- Use `no_run` for examples that need network or external resources, and `compile_fail` to document misuse the type system prevents.
- Hide setup lines with `#` to keep examples readable.
- Remember that `cargo nextest` does not run doctests. Run `cargo test --doc` separately in CI.

## Test Doubles

- Prefer real implementations when they are fast and deterministic: in-memory data structures, temporary files, and local servers.
- Introduce a trait seam only at real boundaries such as external services, clocks, randomness, and I/O. Do not add traits to every struct for mocking.
- Prefer hand-written fakes for boundaries with stateful behavior. They are easier to read and refactor than expectation-heavy mocks.
- Use `mockall` when interaction verification is the behavior under test, or when the repository already relies on it.
- Do not mock the component the test exists to verify.
- For outbound HTTP, use a local mock server (`wiremock`, `httpmock`) instead of mocking the HTTP client type.

## Time And Randomness

- Inject time through a clock trait or parameter for logic that depends on wall-clock time (`SystemTime`, `chrono::Utc::now`).
- Use `#[tokio::test(start_paused = true)]` or `tokio::time::pause()` with `advance()` for Tokio timers, timeouts, and intervals. This requires Tokio's `test-util` feature and affects only Tokio time, not `SystemTime`.
- Seed random number generators in tests, and print the seed on failure if it varies.
- Never use `std::thread::sleep` or `tokio::time::sleep` with real time to wait for a condition.

## Async Tests

- Use `#[tokio::test]` (or the runtime's equivalent). Choose `flavor = "multi_thread"` when the behavior depends on parallel execution or `Send` requirements.
- Test cancellation and timeout paths explicitly where the code claims to handle them.
- Drive concurrent scenarios with `join!`, `JoinSet`, `Barrier`, or `Notify` to create the interleaving under test. Do not rely on timing.
- Assert that spawned tasks complete and do not panic. Check their `JoinHandle` results.

## HTTP Tests

- Test the real router with its middleware stack. With Axum, call the router directly using `tower::ServiceExt::oneshot`, or use the test client the repository already uses.
- Build the application through the same function production uses, overriding only configuration and external dependencies.
- Assert status codes, important headers, and error bodies when they are part of the contract.
- Test `401`, `403`, and authorized access separately for protected endpoints.

## Database Tests

- Test persistence against the real database engine when SQL behavior, constraints, transactions, or migrations matter. SQLite is not a substitute for PostgreSQL semantics.
- Use `#[sqlx::test]` for per-test databases with migrations applied, or `testcontainers` for disposable containers, matching the repository's approach.
- Isolate data per test using separate databases, schemas, or transactions that roll back.
- Test constraint violations and concurrency conflicts as behavior, not only the happy path.
- Mark tests that need external infrastructure with `#[ignore = "requires docker"]` or a feature flag if the repository runs them separately.

## Property-Based Testing

- Use `proptest` or `quickcheck` for parsers, serializers, encoders, data structures, and any function with algebraic properties.
- Test properties, not examples: round trips (`decode(encode(x)) == x`), invariants preserved after operations, equivalence with a simple reference implementation, and idempotence.
- Constrain generators to valid input domains, and test invalid domains separately.
- Commit the `proptest-regressions` directory so found failures are replayed.

## Snapshot Testing

- Use `insta` for large or structured outputs such as serialized responses, error messages, generated code, and CLI output.
- Review snapshots like code (`cargo insta review`). Do not accept them blindly.
- Redact unstable values such as timestamps, UUIDs, and paths.
- Do not use snapshots for small values where `assert_eq!` is clearer.

## Concurrency Testing

- Use `loom` to exhaustively check interleavings for lock-free code, custom synchronization primitives, and subtle atomic orderings. Gate it with `#[cfg(loom)]` and keep the models small.
- For higher-level concurrency (duplicate requests, idempotency, optimistic locking), run real concurrent operations and assert the invariant: one effective state change, correct conflict handling, or no lost updates.
- Use `Barrier` or channels to force the racy interleaving deterministically.

## Unsafe Code And Miri

- Run tests that exercise `unsafe` code under Miri (`cargo +nightly miri test`) to detect undefined behavior, invalid aliasing, out-of-bounds access, and use-after-free.
- Keep Miri-compatible tests free of FFI and heavy I/O, or skip them under `cfg(miri)`.
- Add Miri to CI for crates with meaningful `unsafe`. Use `rust-unsafe-ffi` for deeper guidance.

## Fuzzing

- Fuzz parsers, decoders, protocol handlers, and anything that consumes untrusted bytes, using `cargo fuzz` (libFuzzer) or `afl`.
- Use `arbitrary` to generate structured input.
- Assert invariants in fuzz targets, not just "does not crash".
- Turn every fuzz finding into a regular regression test.

## Compile-Fail And API Tests

- Use `trybuild` for macros and type-level APIs to prove that misuse fails to compile with a useful error.
- Use `compile_fail` doctests for small cases.

## Benchmarks

- Use `criterion` or `divan` for statistically sound benchmarks. Use `std::hint::black_box` to prevent the optimizer from removing the work.
- Benchmark realistic inputs and sizes.
- Keep benchmarks out of the normal test run. Compare against a saved baseline when evaluating a change.
- Use `rust-performance` for profiling guidance.

## Features And Platforms

- Test the feature combinations the crate supports: default, `--no-default-features`, and `--all-features`. Use `cargo hack --feature-powerset` for libraries with several independent features.
- Test the declared MSRV in CI for libraries that promise one.
- Use `#[cfg(target_os = ...)]` tests only for genuinely platform-specific behavior.

## Global State And Flakiness

- Treat flaky tests as defects. Find the nondeterminism instead of adding retries or sleeps.
- Avoid mutating environment variables in tests. `std::env::set_var` is `unsafe` in the 2024 edition because it races with other threads. Pass configuration explicitly instead.
- Avoid fixed ports, fixed file paths, and shared static state.
- Use `serial_test` only as a last resort for unavoidable global state, and document why.
- Prefer `cargo nextest`, which runs each test in its own process, when global state is hard to eliminate.

## Coverage

- Use `cargo llvm-cov` for coverage when the repository tracks it.
- Use coverage to find untested important behavior, not as a target to optimize.

## What Not To Test

- Do not test what the type system already guarantees.
- Do not write tests that mirror the implementation line by line.
- Do not test third-party crate behavior that the code does not customize.
- Do not assert incidental details, such as `Debug` output or exact log lines, unless they are part of the contract.

## Test Implementation Workflow

- Inspect the existing test layout, helpers, dev-dependencies, and CI test commands.
- Identify the behavior that needs proof and choose the test level.
- Reuse existing fixtures and helpers where they help.
- Write the test, see it fail for the right reason when fixing a bug, then make it pass.
- Run the targeted tests, then the affected crate's full suite, including doctests.
- Run the feature combinations CI uses when the change touches feature-gated code.

## Test Review Checklist

- Does the test follow the repository's test stack and layout?
- Does it prove behavior rather than implementation details?
- Is the test level the smallest one that reliably proves the behavior?
- Is it deterministic and isolated from global state, ports, files, and environment variables?
- Is time controlled with a clock seam or paused Tokio time?
- Are concurrent scenarios driven deterministically rather than by timing?
- Are test doubles used only at real boundaries?
- Does database testing use a realistic engine when SQL semantics matter?
- Are spawned tasks and their failures observed?
- Are property, snapshot, fuzz, or Miri tests used where they add real confidence?
- Do tests return `Result` and use `?` instead of unexplained `unwrap()`?
- Do doctests and feature combinations still pass?
- If a bug was fixed, is there a regression test that fails without the fix?
