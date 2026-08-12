---
name: dotnet-backend
description: Design, implement, refactor, and review production .NET backend applications using C#, ASP.NET Core, dependency injection, asynchronous programming, APIs, persistence, and layered architecture. Use when creating or modifying backend services, endpoints, application services, repositories, domain logic, or infrastructure in a .NET project.
license: MIT
---

# Dotnet Backend

Use this skill when working on production .NET backend systems.

Inspect the repository before changing code. Follow existing conventions, naming, layering, libraries, and testing patterns before introducing new ones. Do not impose Clean Architecture, CQRS, MediatR, repository layers, or other patterns unless the repository already uses them or the change clearly benefits from them.

Prefer simple solutions over unnecessary abstractions. Make the smallest coherent change that solves the problem.

## General Principles

- Keep business logic separate from transport and infrastructure concerns.
- Keep controllers and endpoints thin. Parse the request, delegate work, translate the result.
- Put domain and application rules where they can be reused and tested without HTTP concerns.
- Optimize based on evidence, not premature optimization.
- When modifying an existing system, preserve working behavior unless the task explicitly changes it.

## Modern C#

- Use current C# features already accepted in the repository when they improve clarity.
- Prefer clear types and names over cleverness.
- Favor immutability for request and response models when practical.
- Use nullable reference types correctly. Do not silence warnings without understanding the nullability contract.
- Use `required`, pattern matching, switch expressions, and collection expressions when they improve readability and match repo style.

## ASP.NET Core APIs

- Keep API handlers focused on HTTP concerns: binding, status codes, headers, auth context, and result translation.
- Put business rules in application or domain services, not controllers.
- Return consistent error responses. Prefer `ProblemDetails` for server-side API errors.
- Validate requests at the boundary before executing business logic.
- Distinguish between create, update, delete, and query operations clearly.
- Be explicit about route shape, response codes, and contracts.

## Architecture And Boundaries

- Respect the existing architecture first.
- Separate transport, application logic, domain logic, and infrastructure concerns even in smaller projects.
- Do not let controllers, endpoints, or handlers accumulate persistence and orchestration logic.
- Avoid generic repository abstractions that merely duplicate `DbContext`.
- Introduce abstractions when they isolate a real boundary, external dependency, or complex policy.

## Dependency Injection

- Register services with the narrowest lifetime that matches their behavior.
- Prefer stateless services.
- Use singleton only for services that are thread-safe and do not depend on scoped state.
- Treat `DbContext` as scoped.
- Do not capture scoped services in singletons.
- Prefer constructor injection. Avoid service locator patterns.

## Async And Cancellation

- Use async APIs for I/O.
- Avoid `.Result`, `.Wait()`, and sync-over-async.
- Propagate `CancellationToken` where appropriate from the request boundary through application, infrastructure, database, and external calls.
- Accept `CancellationToken` in async methods that perform cancellable work.

```csharp
public Task<OrderDto> GetOrderAsync(Guid id, CancellationToken cancellationToken)
```

- Do not fake async with `Task.Run` in server code unless there is a specific CPU-bound reason and it is understood.

## EF Core

- Use one `DbContext` per logical unit of work.
- Never execute concurrent operations against the same EF Core `DbContext`.
- Treat `DbContext` as not thread-safe.
- Use `AsNoTracking()` for appropriate read-only queries.
- Avoid N+1 queries.
- Project only required database fields where practical.
- Prefer server-side filtering, sorting, aggregation, and pagination.
- Be deliberate about `Include`; use it when needed, not by default.
- Watch for client-side evaluation and accidental materialization too early.
- Persist changes intentionally; do not scatter `SaveChangesAsync` calls across one workflow without understanding transaction boundaries.

## Transactions, Concurrency, And Integrity

- Think explicitly about transaction boundaries.
- Keep transactions as short as practical.
- Treat database constraints as part of protecting concurrency-sensitive invariants.
- Use unique indexes, foreign keys, check constraints, and concurrency tokens where they enforce correctness.
- Consider optimistic concurrency for update races.
- Handle duplicate key and concurrency failures explicitly where they affect behavior.
- Make state transitions explicit and protect invariants in code and in the database.

## Repository Usage

- If the project already uses repositories, follow the existing pattern.
- If it does not, prefer direct `DbContext` usage over adding a generic repository layer that adds no value.
- Create repository abstractions only when they encapsulate real query logic, aggregate persistence rules, or boundary concerns.

## HTTP Clients And External APIs

- Use `IHttpClientFactory` rather than creating an `HttpClient` per request.
- Configure named or typed clients when the integration has distinct settings or policies.
- Set explicit timeout behavior and understand where it is enforced.
- Keep external API mapping and error translation out of controllers.
- Treat remote responses as unreliable input. Validate and handle missing or malformed data.
- Consider idempotency for operations that may be retried or replayed.

## Resilience

- Apply timeout, retry, circuit breaker, and rate limiting policies where they fit the dependency and failure mode.
- Retry only operations that are safe to retry.
- Do not retry non-idempotent operations blindly.
- Use backoff and limits. Avoid infinite or overly aggressive retries.
- Distinguish transient failures from validation, auth, and permanent business failures.

## Validation, Errors, And Authorization

- Validate requests before mutating state.
- Return domain-appropriate failures, not raw exception text.
- Use `ProblemDetails` or the repository's established equivalent for API error payloads.
- Reserve exceptions for exceptional cases; do not use them for ordinary control flow.
- Distinguish authentication from authorization.
- Perform authorization server-side.
- Enforce resource-level authorization where needed, not only endpoint-level checks.

## Configuration, Secrets, And Logging

- Use configuration binding patterns already present in the repository.
- Fail clearly on missing required configuration.
- Never commit secrets.
- Never log secrets or credentials.
- Prefer secret managers and environment-specific configuration over checked-in sensitive values.
- Use structured logging.
- Log enough context to diagnose failures, but avoid noisy logs and sensitive data.
- Include correlation identifiers or request identifiers when the system already supports them.

## Security

- Validate all external input.
- Parameterize database access; do not build SQL unsafely.
- Apply least privilege to service accounts, API keys, and database access.
- Sanitize data crossing trust boundaries.
- Be careful with file access, deserialization, redirects, and user-controlled URLs.
- Do not expose internal exception details to clients.

## Performance

- Measure first when performance is the goal.
- Eliminate obvious waste in hot paths: unnecessary allocations, repeated queries, over-fetching, excessive serialization, and chatty network calls.
- Use batching, projection, and pagination where appropriate.
- Do not trade maintainability for speculative micro-optimizations.

## Refactoring Existing Systems

- Start by reading the existing flow end to end.
- Match the repository's patterns unless they block the requested change or are clearly unsafe.
- Make incremental changes that preserve behavior.
- Separate cleanup from functional changes when possible.
- Avoid broad rewrites when a targeted fix will do.
- If the current design has issues, improve the local area without forcing a full architecture migration.

## Implementation Workflow

- Inspect the affected project structure, startup wiring, tests, and local conventions first.
- Identify the real execution path before editing.
- Make the smallest coherent change.
- Update or add tests when the repository has test coverage for the affected area.
- Build the affected projects before considering implementation complete.
- If feasible, run the relevant tests for the changed area.
- Review logs, nullability, cancellation, query shape, transaction boundaries, and auth impact before finishing.

## Code Review Checklist

- Does the change follow existing repository conventions?
- Are controllers or endpoints still thin?
- Is business logic separated from transport and infrastructure concerns?
- Are async APIs used for I/O without `.Result`, `.Wait()`, or sync-over-async?
- Is `CancellationToken` propagated where appropriate?
- Is EF Core usage safe, including `DbContext` lifetime and no concurrent operations on the same context?
- Are read-only queries using `AsNoTracking()` where appropriate?
- Do queries avoid N+1 behavior and over-fetching?
- Are transaction boundaries explicit and correct?
- Are concurrency-sensitive invariants protected in code and by database constraints where needed?
- Are retries applied only to safe operations?
- Is idempotency considered for retried or externally repeated operations?
- Is validation performed at the boundary?
- Are errors translated to the API contract appropriately, including `ProblemDetails` where used?
- Is authorization enforced server-side and at the right resource boundary?
- Are configuration and secrets handled safely?
- Is logging structured and free of secrets?
- Are security and performance risks considered without unnecessary abstraction?
- Was the affected project built, and were relevant tests run if available?
