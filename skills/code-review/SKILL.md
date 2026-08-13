---
name: code-review
description: Review .NET code changes and pull requests for correctness, security, data integrity, concurrency, reliability, performance, maintainability, testing, and architectural issues. Use when reviewing diffs, commits, branches, pull requests, patches, or proposed changes in a .NET repository.
license: MIT
---

# Code Review

Use this skill when reviewing .NET code changes, commits, branches, pull requests, patches, or proposed implementations.

The goal of the review is to identify real defects and material risks before changes are merged. Prioritize correctness, security, data integrity, concurrency, reliability, breaking changes, performance, test coverage, and maintainability. Do not turn the review into a style critique, and do not report personal preferences as defects.

## Relationship With Specialized Skills

- This skill controls the review process, scope, severity, and finding quality.
- Use `dotnet-backend` for deeper ASP.NET Core, architecture, dependency injection, async, backend reliability, security, and integration guidance.
- Use `dotnet-testing` for deeper test quality, test strategy, fixtures, integration testing, concurrency testing, and regression coverage guidance.
- Use `ef-core` for deeper Entity Framework Core guidance involving `DbContext`, queries, transactions, migrations, database constraints, concurrency, and persistence performance.
- Do not duplicate the full guidance from specialized skills in the review. Apply it where it is relevant to a concrete risk in the changed code.

## Review Priorities

Evaluate issues in this order:

1. Correctness
2. Security
3. Data integrity
4. Concurrency
5. Reliability
6. Breaking changes
7. Performance
8. Test coverage
9. Maintainability

Do not force a finding in every category. A review with zero findings is better than a review containing invented problems.

## Establish Scope First

Before reviewing findings, determine the appropriate review base and scope.

- For a branch or pull request, compare against the intended base branch rather than treating the whole repository as new code.
- Inspect the repository structure, projects, test layout, configuration, and relevant conventions.
- Understand what behavior is being added, removed, or changed.
- Inspect the complete diff before focusing on individual files.
- Inspect surrounding code when behavior depends on callers, dependencies, configuration, middleware, persistence mappings, or tests.
- Identify affected callers, background jobs, API consumers, database schemas, messages, external integrations, and test coverage.

Do not review changed lines in isolation when surrounding behavior determines correctness.

## Understand Intent

Before reporting findings, determine:

- What behavior is intended to change.
- What existing behavior should remain unchanged.
- Which components and execution paths are affected.
- Which external contracts may change.
- Which data invariants must remain true.
- Which failure modes are realistically possible.

Distinguish intentional behavior changes from regressions. Do not report unrelated pre-existing problems as findings against the current change unless the change makes them materially worse.

## Finding Quality

Every reported finding must be actionable and evidence-based. A useful finding explains:

1. Where the problem is.
2. What is wrong.
3. Under what conditions it occurs.
4. Why it matters.
5. What direction a fix should take.

Avoid vague comments such as:

- "This could be improved."
- "Consider refactoring."
- "This might cause issues."
- "This is not best practice."

Explain the concrete failure or risk. Do not speculate when the code, tests, configuration, or surrounding implementation can be inspected.

## Severity

Classify findings with the lowest severity that accurately reflects impact and likelihood.

### Critical

Use for issues that can cause severe consequences, such as major security vulnerabilities, authorization bypass, corruption of important data, catastrophic production failures, or irreversible destructive behavior.

### High

Use for defects likely to cause incorrect application behavior, lost or duplicated data, significant concurrency bugs, broken public contracts, or serious reliability problems.

### Medium

Use for legitimate problems with meaningful impact but lower likelihood or severity, such as important edge cases not handled, avoidable performance problems, incomplete error handling, or meaningful missing tests.

### Low

Use sparingly for small but concrete engineering problems. Do not classify formatting, naming, or stylistic preferences as significant findings unless they materially affect understanding or correctness.

## Correctness Review

Trace important execution paths end to end rather than only reading methods independently. Look for:

- Incorrect conditions or inverted logic.
- Off-by-one errors.
- Missing cases or invalid assumptions.
- Null handling problems.
- Incorrect state transitions.
- Partial updates.
- Incorrect exception behavior.
- Incorrect HTTP responses.
- Unexpected side effects.

Verify that the implementation actually satisfies the intended behavior and preserves behavior that should remain unchanged.

## Async And Concurrency Review

Do not assume sequential-looking application logic is safe under concurrent requests. Pay particular attention to:

- Missing `await`.
- Fire-and-forget tasks.
- Sync-over-async, `.Result`, and `.Wait()`.
- Shared mutable state.
- Race conditions.
- Incorrect locking.
- Parallel access to non-thread-safe dependencies.
- Duplicate processing.
- Concurrent operations using the same EF Core `DbContext`.

When concurrency behavior matters, inspect whether the code protects the invariant under simultaneous requests, retries, background processing, and repeated delivery.

## Data Integrity Review

When code modifies persistent state, identify the invariant that must remain true. Check:

- Transaction boundaries.
- Uniqueness and database constraints.
- Duplicate execution.
- Check-then-write races.
- Partial persistence.
- Optimistic concurrency.
- Retry behavior.
- Idempotency.

Application-level validation alone may not protect invariants from concurrent requests. Use `ef-core` when deeper EF Core persistence analysis is relevant.

## ASP.NET Core And API Review

For API changes, inspect:

- Routing.
- Request binding.
- Validation.
- Status codes.
- Authorization.
- Cancellation.
- Error handling.
- Response contracts.
- Backwards compatibility.

Controllers and endpoint handlers should generally coordinate transport concerns rather than contain substantial business logic. Use `dotnet-backend` for deeper backend guidance where relevant.

## Security Review

Report only concrete security problems with a plausible execution path. Look for:

- Missing authorization or incorrect authorization scope.
- Injection.
- Unsafe deserialization.
- Committed secrets.
- Sensitive logging.
- Insecure configuration.
- Path traversal.
- Mass assignment.
- Insecure direct object references.
- Untrusted external input.
- Credential exposure.

Distinguish authentication from authorization. Verify whether another layer already enforces the security property before reporting a finding.

## External Integration Review

When reviewing HTTP clients, queues, webhooks, messaging, or external-service integrations, inspect:

- Timeouts.
- Cancellation.
- Error handling.
- Retry behavior.
- Idempotency.
- Response validation.
- Authentication.
- Rate limiting.
- Partial failures.

Be particularly careful when retries can duplicate non-idempotent operations.

## Performance Review

Report performance findings only when there is a credible impact. Look for:

- N+1 database queries.
- Unnecessary database round trips.
- Unbounded queries.
- Excessive allocations in meaningful hot paths.
- Loading excessive data.
- Sequential independent I/O.
- Repeated expensive operations.
- Blocking asynchronous code.

Do not report speculative micro-optimizations.

## Test Review

Review whether tests meaningfully protect the changed behavior. Look for missing coverage of:

- Important success paths.
- Failure paths.
- Boundary conditions.
- Authorization.
- Concurrency.
- Idempotency.
- Persistence behavior.
- Regression scenarios.

Do not request tests for trivial implementation details solely to increase coverage. Use `dotnet-testing` for deeper testing guidance when relevant.

## Breaking Change Review

Look for unintended changes to:

- Public APIs.
- Request and response contracts.
- Serialized models.
- Database schemas.
- Configuration keys.
- Environment variables.
- Events and messages.
- Externally consumed interfaces.

If a breaking change appears intentional, verify that the surrounding change accounts for migration, compatibility, rollout, or consumer requirements.

## EF Core Migrations

When EF Core migrations are part of the change, inspect them explicitly. Look for:

- Destructive operations.
- Accidental column drops.
- Incorrect renames.
- Nullability changes.
- Data loss.
- Unsafe unique constraints.
- Missing data migration.
- Model snapshot inconsistencies.

Do not assume a migration is correct because EF Core generated it. Use `ef-core` for detailed persistence review.

## Error Handling And Logging

Check whether failures are handled at the correct boundary, preserve useful diagnostic context, expose appropriate responses, avoid leaking internal information, and leave the system in a consistent state.

Do not recommend broad `try`/`catch` blocks merely to suppress exceptions.

For logging, check for missing context on important failures, incorrect log levels, sensitive information, credentials, tokens, personal data, and misleading messages. Prefer structured logging, but do not demand logging for every method or code path.

## Maintainability Review

Only report maintainability issues when they have practical consequences. Examples include:

- Duplicated business rules likely to diverge.
- Misleading abstractions.
- Hidden side effects.
- Excessive coupling.
- Unclear ownership of important behavior.
- Unnecessary complexity that makes correctness difficult to verify.

Do not report naming or formatting issues that automated tooling should handle unless they materially affect understanding.

## Avoid False Positives

Before reporting a finding:

1. Verify the issue exists in the changed code.
2. Inspect surrounding implementation.
3. Check whether another layer already handles it.
4. Determine whether the scenario is realistically reachable.
5. Verify the finding is not merely a preference.

Do not invent runtime behavior. If uncertain, investigate further rather than presenting speculation as fact.

## Review Output

Prefer findings over a long summary. Order findings by severity, highest first.

Use this format for each finding:

```markdown
### [Severity] Short title

**Location:** `path/to/file.cs:line`

Explain the concrete problem and the scenario that triggers it. Explain the consequence. Provide a concise recommended direction for fixing it.
```

When useful, include a small code suggestion, but do not rewrite large sections of the implementation.

If no meaningful problems are found, say so clearly. You may mention residual testing or verification risks separately, but do not present them as confirmed defects.

## Review Workflow

1. Determine the review base and scope.
2. Inspect the complete diff.
3. Understand the intended behavior.
4. Identify affected execution paths.
5. Inspect relevant surrounding code.
6. Inspect relevant tests.
7. Check correctness and edge cases.
8. Check security and authorization.
9. Check data integrity and concurrency.
10. Check external integrations and failure behavior.
11. Check persistence and migrations.
12. Check meaningful performance risks.
13. Verify each potential finding against the actual code.
14. Rank findings by severity.
15. Produce concise, actionable review feedback.

## Review Checklist

Before completing a review, consider:

- Correctness.
- Regressions.
- Edge cases.
- Null handling.
- Async behavior.
- Concurrency.
- Data integrity.
- Transactions.
- Idempotency.
- Security.
- Authorization.
- Validation.
- Error handling.
- API contracts.
- Backwards compatibility.
- EF Core behavior.
- Migrations.
- External integrations.
- Retry behavior.
- Performance.
- Logging.
- Tests.
- Maintainability.

Do not force a comment for every checklist item.

## Style

- Prioritize signal over volume.
- Be specific, concise, and evidence-based.
- Avoid generic best-practice comments.
- Avoid stylistic nitpicks.
- Avoid praise that does not help the review.
- Do not exaggerate severity.
- Do not speculate when the code can be inspected.
- Focus primarily on defects introduced by the reviewed change.
