---
name: dotnet-testing
description: Create, improve, debug, and review automated tests for .NET and ASP.NET Core applications. Use when writing unit tests, integration tests, API tests, database tests, test fixtures, mocks, fakes, or test infrastructure, and when diagnosing flaky or ineffective .NET tests.
license: MIT
---

# Dotnet Testing

Use this skill when creating, debugging, or reviewing automated tests in .NET applications.

Inspect the repository's existing test projects before writing tests. Follow its established test frameworks, assertion libraries, mocking libraries, naming conventions, fixture patterns, helper utilities, and project structure. Do not introduce a new test stack when the repository already has one.

## Testing Philosophy

- Test behavior rather than implementation details.
- Prefer meaningful tests over chasing coverage percentages.
- Prefer tests that increase confidence in important application behavior and catch regressions.
- Avoid tests that merely reproduce implementation details.
- Avoid tests that only prove framework behavior.
- Keep tests deterministic.
- Ensure tests can run independently.
- Avoid test-order dependencies.

## Choosing The Test Level

- Start with the behavior you need to prove, then choose the smallest test level that can prove it reliably.
- Use unit tests for isolated business logic.
- Use integration tests when behavior depends on wiring, persistence, serialization, configuration, or framework integration.
- Use API tests when HTTP contracts, filters, middleware, routing, auth, or status codes matter.
- Use end-to-end tests only when the behavior cannot be proved with a smaller, cheaper, more reliable test.
- Do not force everything into unit tests if the important risk is integration behavior.

## Unit Tests

- Keep the unit under test clear.
- Do not mock the component the test exists to verify.
- Mock external boundaries when appropriate.
- Do not mock dependencies automatically; only isolate collaborators that would make the test slow, nondeterministic, or less meaningful.
- Focus unit tests on business rules, branching logic, invariants, calculations, and edge cases.
- Prefer simple object setup over elaborate mocking where possible.

## Integration Tests

- Use integration tests when the real composition matters.
- Cover persistence, dependency injection wiring, configuration, serialization, middleware, and application boundaries that unit tests cannot prove.
- Keep the scope intentional; integration tests should still target a concrete behavior, not the whole system by default.
- Reset or isolate state so tests remain independent.

## API Tests

- Test route behavior, model binding, validation, auth, status codes, and response contracts at the HTTP boundary.
- Keep API tests focused on externally observable behavior.
- Verify error payloads and problem responses when they are part of the contract.
- Cover success and failure cases that matter to consumers.

## End-To-End Tests

- Use end-to-end tests sparingly for cross-process or full-environment behavior that lower test levels cannot cover.
- Keep them small in number and high in value.
- Avoid relying on them for fast feedback on ordinary business logic.

## Arrange / Act / Assert

- Keep tests easy to scan.
- Make setup intentional, action singular, and assertions explicit.
- Avoid mixing multiple unrelated behaviors into one test.
- Prefer one logical act per test unless the scenario itself requires a sequence.

## Test Naming And Assertions

- Follow existing repository naming conventions.
- Use names that describe the behavior and relevant condition.
- Keep assertions specific and behavior-focused.
- Assert what matters; do not assert every property by default.
- Prefer clear failure messages or readable assertion styles when the repository already uses them.

## Business Rules And Parameterized Tests

- Test business rules directly and cover the meaningful edges.
- Use parameterized tests when the same behavior should hold across multiple inputs.
- Do not hide complex scenario meaning in oversized data tables.
- Keep the parameters readable enough that failures are easy to diagnose.

## Mocks, Stubs, Fakes, And Interaction Verification

- Use mocks, stubs, fakes, or in-memory substitutes according to the repository's established patterns.
- Verify dependency calls only when the interaction itself is meaningful behavior.
- Avoid interaction-heavy tests when observable outputs or state changes prove the behavior more directly.
- Favor fakes or lightweight test implementations over brittle mocks when that makes tests simpler and more realistic.

## Time, Randomness, And Determinism

- Avoid uncontrolled system time.
- Prefer `TimeProvider` or an equivalent abstraction when time affects behavior.
- Avoid uncontrolled randomness.
- Use deterministic seeds or fixed values when randomness matters.
- Do not let the current clock, timezone, locale, or machine state leak into test outcomes unless the test is explicitly about that behavior.

## Async Testing

- Await asynchronous operations correctly.
- Propagate cancellation when the behavior under test depends on it.
- Avoid `.Result`, `.Wait()`, and sync-over-async in tests.
- Never use arbitrary `Thread.Sleep` or `Task.Delay` as a workaround for flaky tests.

## EF Core And Database Testing

- Do not assume EF Core InMemory behaves like a relational database.
- Prefer the closest practical relational engine already used by the repository.
- Use the real provider or an appropriate disposable relational database when SQL translation, constraints, migrations, transactions, or concurrency semantics matter.
- Use EF Core InMemory only when its limitations are acceptable for the specific behavior.
- Test constraints, transactions, query behavior, concurrency, and relational semantics against a realistic store.
- Isolate database state per test or per fixture so tests remain independent.

## ASP.NET Core Integration Testing

- Use `WebApplicationFactory` where appropriate for ASP.NET Core API integration tests.
- Prefer exercising the application through its real HTTP pipeline when routing, filters, middleware, auth, JSON configuration, or DI wiring matter.
- Override configuration and dependencies carefully so the test still reflects production behavior where it matters.
- Keep test-only service replacements explicit and minimal.

## Authentication And Authorization Tests

- Test protected endpoints for `401`, `403`, and successful authorized access where relevant.
- Distinguish unauthenticated behavior from authenticated-but-forbidden behavior.
- Verify resource-level authorization when the application enforces ownership, scope, role, or policy checks.

## External HTTP Integration Testing

- Do not call real third-party APIs from normal automated tests.
- Mock or fake the external boundary at the HTTP layer or integration seam already used by the repository.
- Test timeout, retry-safe failure handling, error mapping, and malformed response handling where relevant.
- Keep external contract assumptions visible in test setup and assertions.

## Concurrency And Idempotency Tests

- Explicitly test concurrency-sensitive behavior concurrently where practical.
- Test repeated and concurrent requests when verifying idempotency.
- Verify the observable guarantees: single effective state change, safe duplicate handling, conflict behavior, or invariant protection.
- Use synchronization primitives or controlled orchestration when needed; do not rely on timing luck.

## Fixtures, Builders, And Shared Infrastructure

- Reuse shared test infrastructure only when it reduces duplication without hiding important behavior.
- Keep fixtures predictable and easy to understand.
- Use builders or factory helpers to highlight relevant data and hide irrelevant setup.
- Do not let helpers become a second application layer inside the test suite.
- Keep shared state tightly controlled.

## Flaky Tests And Regression Tests

- Treat flaky tests as defects.
- Remove nondeterminism instead of increasing sleeps or retries.
- If a test is flaky, identify whether the issue is timing, shared state, environment dependence, async misuse, or hidden ordering.
- Add regression tests when fixing bugs where practical.
- Make regression tests prove the bug scenario and the intended fixed behavior.

## What Not To Test

- Do not test private implementation details directly unless the repository explicitly does so and there is no better seam.
- Do not write tests that mirror the code line by line.
- Do not test framework behavior that the application does not customize.
- Do not assert incidental details that make refactoring harder without increasing confidence.

## Test Implementation Workflow

- Inspect the existing test projects and conventions first.
- Identify the behavior that actually needs proof.
- Reuse existing fixtures, helpers, and patterns where they help.
- Add only the setup needed for the scenario.
- Run the targeted tests after implementation.
- Run the broader affected test suite when practical.
- If the change fixes a bug, add a regression test when practical.

## Test Review Checklist

- Does the test follow the repository's existing test stack and conventions?
- Does it prove behavior rather than implementation details?
- Is the chosen test level the smallest one that reliably proves the behavior?
- Is the test deterministic and independent?
- Does it avoid test-order dependencies?
- Are assertions specific, meaningful, and not overly coupled to internals?
- Are mocks or fakes used intentionally rather than automatically?
- Is interaction verification used only when the interaction is meaningful behavior?
- Is time controlled with `TimeProvider` or an equivalent abstraction when needed?
- Is randomness controlled?
- Are async operations awaited correctly without sleeps as synchronization?
- If EF Core is involved, does the test use an appropriate database strategy for the behavior under test?
- If ASP.NET Core is involved, should this be covered with `WebApplicationFactory`?
- Are `401`, `403`, and success paths covered for protected endpoints where relevant?
- Are external HTTP dependencies isolated from real third-party services?
- Are concurrency and idempotency behaviors tested explicitly where they matter?
- Is shared test infrastructure helping clarity rather than hiding behavior?
- Would this test catch the regression it claims to prevent?
- Were the targeted tests run, and was the broader affected suite run when practical?
