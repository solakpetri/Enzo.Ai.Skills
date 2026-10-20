---
name: rust-code-review
description: Review Rust code changes and pull requests for correctness, soundness, security, data integrity, concurrency, async cancellation, panics, error handling, API and semver breaks, dependency and feature changes, performance, testing, and maintainability. Use when reviewing diffs, commits, branches, pull requests, patches, or proposed changes in a Rust repository.
license: MIT
---

# Rust Code Review

Use this skill when reviewing Rust code changes, commits, branches, pull requests, patches, or proposed implementations.

The goal of the review is to find real defects and material risks before changes are merged. Prioritize correctness, soundness, security, data integrity, concurrency, reliability, breaking changes, performance, test coverage, and maintainability. The compiler and clippy already catch many mistakes. Spend review effort on what they cannot see: logic, invariants, panics on real inputs, cancellation, contracts, and unsafe reasoning. Do not turn the review into a style critique, and do not report personal preferences as defects.

## Relationship With Specialized Skills

- This skill controls the review process, scope, severity, and finding quality.
- Use `rust-core` for ownership, type design, error handling, API design, semver, features, and Cargo guidance.
- Use `rust-async` for runtime, cancellation, locking, channels, spawning, and shutdown guidance.
- Use `rust-backend` for HTTP services, middleware, auth, configuration, and integrations.
- Use `rust-database` for queries, transactions, concurrency control, constraints, and migrations.
- Use `rust-testing` for test quality and strategy.
- Use `rust-performance` for performance claims and hot-path changes.
- Use `rust-unsafe-ffi` for any `unsafe` code, raw pointers, or FFI.
- Do not duplicate the full guidance from specialized skills in the review. Apply it where it is relevant to a concrete risk in the changed code.

## Review Priorities

Evaluate issues in this order:

1. Correctness
2. Soundness (undefined behavior reachable from safe code)
3. Security
4. Data integrity
5. Concurrency and async behavior
6. Reliability, including panics and error handling
7. Breaking changes
8. Performance
9. Test coverage
10. Maintainability

Do not force a finding in every category. A review with zero findings is better than a review containing invented problems.

## Establish Scope First

- For a branch or pull request, compare against the intended base branch rather than treating the whole repository as new code.
- Inspect the workspace layout, affected crates, edition, MSRV, features, lints, and CI configuration.
- Understand what behavior is being added, removed, or changed.
- Inspect the complete diff before focusing on individual files, including `Cargo.toml`, `Cargo.lock`, `build.rs`, migrations, and query metadata.
- Inspect surrounding code when behavior depends on callers, trait implementations, feature flags, middleware, or configuration.
- Identify affected public APIs, downstream crates, serialized formats, database schemas, and test coverage.

Do not review changed lines in isolation when surrounding behavior determines correctness.

## Understand Intent

Before reporting findings, determine:

- What behavior is intended to change.
- What existing behavior should remain unchanged.
- Which execution paths, tasks, and threads are affected.
- Which external contracts may change.
- Which invariants must remain true.
- Which failure modes are realistically possible.

Distinguish intentional behavior changes from regressions. Do not report unrelated pre-existing problems as findings against the current change unless the change makes them materially worse.

## Finding Quality

Every reported finding must be actionable and evidence-based. A useful finding explains:

1. Where the problem is.
2. What is wrong.
3. Under what conditions it occurs.
4. Why it matters.
5. What direction a fix should take.

Avoid vague comments such as "consider using a more idiomatic approach" or "this might panic". Name the input or interleaving that triggers the problem. Do not speculate when the code, tests, or documentation can be inspected.

## Severity

Classify findings with the lowest severity that accurately reflects impact and likelihood.

### Critical

Severe consequences: undefined behavior reachable from safe code, memory-safety bugs, authorization bypass, injection, corruption of important data, or irreversible destructive behavior.

### High

Defects likely to cause incorrect behavior, panics on realistic input in request or worker paths, lost or duplicated data, deadlocks, significant concurrency bugs, broken public contracts, or serious reliability problems.

### Medium

Legitimate problems with meaningful impact but lower likelihood or severity: unhandled edge cases, cancellation hazards on uncommon paths, avoidable performance problems on real hot paths, incomplete error handling, or meaningful missing tests.

### Low

Small but concrete engineering problems. Use sparingly. Do not report formatting, naming, or clippy-level style as significant findings unless they materially affect understanding or correctness.

## Correctness Review

Trace important execution paths end to end. Look for:

- Incorrect conditions, inverted logic, and off-by-one errors in ranges (`..` versus `..=`).
- Missing match arms hidden by a wildcard `_` that swallows new enum variants.
- Incorrect `Option` and `Result` handling, such as `unwrap_or_default()` hiding a real failure, or `.ok()` discarding an error that matters.
- Silent truncation or sign changes in `as` casts. Prefer `try_from` where values can exceed the target range.
- Integer overflow on untrusted or accumulating values. Release builds wrap silently by default.
- Incorrect state transitions and partial updates.
- Unexpected side effects in `Drop`, `Clone`, `Deref`, or `From` implementations.
- Float comparisons and `NaN` handling where ordering or equality matters.

## Panic Review

Panics in services, workers, and libraries are reliability defects. Look for:

- `unwrap()` and `expect()` on values that depend on input, I/O, configuration, or other processes.
- Slice indexing and slicing (`v[i]`, `&s[a..b]`) with unchecked indices, including string slicing on non-UTF-8 character boundaries.
- Division or remainder by values that can be zero.
- `RefCell` borrow conflicts reachable through re-entrancy.
- Panics inside `Drop`, inside FFI exports, or while holding a lock (poisoning).
- Spawned tasks or threads whose panics are never observed.

Do not report `unwrap()` where the invariant is locally evident, such as on a regex literal or a value just checked. Focus on reachable panics.

## Error Handling Review

- Are errors propagated with their source preserved, rather than replaced with strings?
- Does the change add context where it helps diagnosis?
- Are errors logged and also returned, causing duplicate reports?
- Do library APIs avoid exposing `anyhow::Error` or `Box<dyn Error>` where callers need to match?
- Are new error variants added to exhaustive public enums (a breaking change)?
- Is `?` used on a path where failure should be handled or retried instead?

## Async And Concurrency Review

Do not assume sequential-looking code is safe under concurrent requests, cancellation, or retries. Look for:

- Blocking calls on async runtime threads: `std::fs`, `std::thread::sleep`, synchronous clients, heavy CPU work.
- Lock guards (`std::sync::MutexGuard`, `RefMut`, `DashMap` references) held across `.await`.
- Non-cancel-safe futures used in `select!`, `timeout`, or loops, and partial state mutation before an `.await` that can be cancelled.
- Spawned tasks that are detached, untracked, or excluded from shutdown.
- Unbounded channels, unbounded spawning, and missing concurrency limits.
- Inconsistent lock ordering that can deadlock.
- Atomics with orderings that do not establish the needed happens-before relationship.
- Check-then-act races across tasks, processes, or service instances.
- `unsafe impl Send` or `Sync` added to satisfy a compiler error.

## Unsafe And Soundness Review

Treat every added or changed `unsafe` block as high-risk until proven sound. Check:

- Whether `unsafe` is needed at all.
- Whether each unsafe block has an accurate `SAFETY` comment that covers every requirement.
- Whether safe code in the same module can break the invariants the unsafe code relies on.
- Aliasing, validity, alignment, bounds, initialization, and lifetime requirements.
- Panic safety during partially initialized states.
- FFI layout, ownership, nullability, and unwinding.

Report unsound safe APIs as critical even if no current caller triggers the undefined behavior. Use `rust-unsafe-ffi` for detailed auditing.

## Security Review

Report only concrete security problems with a plausible execution path. Look for:

- Missing authorization or incorrect authorization scope.
- SQL built with `format!`, command injection through `std::process::Command` with shell interpretation, and path traversal.
- Deserialization of untrusted input into types without size limits, or with recursion depth risks.
- Committed secrets, and secrets exposed through `Debug`, logs, or error messages.
- Non-constant-time comparison of secrets and tokens.
- Weak token validation, such as a missing audience, expiry, or algorithm allowlist.
- Fast non-DoS-resistant hashers on attacker-controlled keys.
- Unbounded allocation driven by input lengths, such as `Vec::with_capacity(untrusted_len)`.
- SSRF through user-controlled URLs.

Verify whether another layer already enforces the property before reporting.

## Data Integrity Review

When code modifies persistent state, identify the invariant that must remain true. Check:

- Transaction boundaries, and whether every success path commits.
- Uniqueness and other constraints.
- Check-then-write races and read-modify-write without version checks or locks.
- Partial persistence and duplicate execution under retries.
- Idempotency of handlers for retried requests and redelivered messages.

Use `rust-database` for deeper persistence analysis.

## API, Semver, And Contract Review

For library crates and shared contracts, look for unintended breaking changes:

- Removed, renamed, or re-typed public items.
- New variants on exhaustive public enums, or new fields on exhaustive public structs.
- New required trait methods, or tightened generic bounds.
- Lost auto traits (`Send`, `Sync`, `Unpin`) because of a new field.
- Leaked dependency types in public signatures, and major-version bumps of public dependencies.
- MSRV increases that violate the crate's policy.
- Changes to serialized formats (`serde` renames, tag representation, default values), HTTP contracts, database schemas, configuration keys, and environment variables.

If a breaking change appears intentional, verify the version bump, changelog, migration path, or consumer coordination.

## Dependencies, Features, And Build Review

- Is each new dependency justified, maintained, appropriately licensed, and not pulling excessive transitive weight?
- Do `Cargo.lock` changes include unexpected upgrades or duplicate major versions?
- Do new dependencies introduce `unsafe`, build scripts, or native code that needs scrutiny?
- Are new features additive? Does any code path break under `--no-default-features` or `--all-features`?
- Does `build.rs` remain deterministic, emit correct `rerun-if-changed` directives, and avoid network access?
- Would the change fail `cargo deny` or `cargo audit` policies the repository uses?

## Performance Review

Report performance findings only when there is a credible impact on a real path. Look for:

- N+1 queries, unbounded queries, and sequential independent I/O.
- Allocation or cloning of large data inside hot loops.
- Collecting intermediate collections unnecessarily on hot paths.
- Unbuffered I/O.
- Blocking or CPU-heavy work on async runtime threads.
- Lock contention introduced on shared hot state.
- Performance claims in the change description with no measurement.

Do not report speculative micro-optimizations.

## Test Review

Review whether tests meaningfully protect the changed behavior. Look for missing coverage of:

- Important success and failure paths.
- Boundary conditions and input that could panic.
- Authorization.
- Concurrency, cancellation, and idempotency.
- Persistence behavior against a realistic database.
- Feature-gated code paths.
- Regression scenarios for fixed bugs.
- Miri coverage for new unsafe code.

Do not request tests for trivial details solely to increase coverage. Use `rust-testing` for deeper guidance.

## Maintainability Review

Only report maintainability issues with practical consequences:

- Duplicated business rules likely to diverge.
- Traits or generics with no real purpose that obscure the code.
- Hidden side effects in trait implementations.
- Lifetimes or type complexity that make correctness hard to verify.
- Suppressed lints (`#[allow(...)]`) without justification on code where the lint signals a real problem.

Do not report naming or formatting issues that `rustfmt` and clippy should handle.

## Avoid False Positives

Before reporting a finding:

1. Verify the issue exists in the changed code.
2. Inspect surrounding implementation, trait implementations, and callers.
3. Check whether the type system, another layer, or a constraint already prevents it.
4. Determine whether the scenario is realistically reachable.
5. Verify the finding is not merely a preference.

Do not claim the code fails to compile without evidence. If uncertain, investigate further rather than presenting speculation as fact.

## Review Output

Prefer findings over a long summary. Order findings by severity, highest first.

Use this format for each finding:

```markdown
### [Severity] Short title

**Location:** `path/to/file.rs:line`

Explain the concrete problem and the scenario that triggers it. Explain the consequence. Provide a concise recommended direction for fixing it.
```

When useful, include a small code suggestion, but do not rewrite large sections of the implementation.

If no meaningful problems are found, say so clearly. You may mention residual verification risks separately, such as untested feature combinations or unrun Miri tests, but do not present them as confirmed defects.

## Review Workflow

1. Determine the review base and scope.
2. Inspect the complete diff, including manifests, lock files, build scripts, and migrations.
3. Understand the intended behavior.
4. Identify affected execution paths, tasks, and public APIs.
5. Inspect relevant surrounding code and tests.
6. Check correctness and reachable panics.
7. Audit any unsafe code for soundness.
8. Check security and authorization.
9. Check data integrity, concurrency, and cancellation.
10. Check error handling and external integrations.
11. Check API, semver, serialization, and dependency changes.
12. Check meaningful performance risks.
13. Verify each potential finding against the actual code.
14. Rank findings by severity.
15. Produce concise, actionable review feedback.

## Review Checklist

Before completing a review, consider:

- Correctness and regressions.
- Edge cases and reachable panics.
- Integer overflow and lossy casts.
- Error propagation and context.
- Unsafe soundness.
- Blocking in async code.
- Locks held across `.await`.
- Cancellation safety.
- Task lifecycle and shutdown.
- Concurrency and races.
- Data integrity and transactions.
- Idempotency.
- Security and authorization.
- Input validation and bounded allocation.
- Public API and semver.
- Serialized contracts.
- Feature additivity.
- Dependencies and `Cargo.lock`.
- Performance on real paths.
- Tests.
- Maintainability.

Do not force a comment for every checklist item.

## Style

- Prioritize signal over volume.
- Be specific, concise, and evidence-based.
- Avoid generic best-practice comments.
- Avoid stylistic nitpicks that tooling handles.
- Avoid praise that does not help the review.
- Do not exaggerate severity.
- Do not speculate when the code can be inspected.
- Focus primarily on defects introduced by the reviewed change.
