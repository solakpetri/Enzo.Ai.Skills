---
name: ef-core
description: Design, implement, optimize, troubleshoot, and review Entity Framework Core persistence code in .NET applications. Use when working with DbContext, entity configuration, relationships, queries, change tracking, transactions, concurrency, migrations, database constraints, or EF Core performance.
license: MIT
---

# EF Core

Use this skill when designing, implementing, optimizing, troubleshooting, or reviewing Entity Framework Core persistence code in production .NET applications.

This is not an EF Core tutorial. Use it to make practical engineering decisions around correctness, data integrity, performance, and maintainability.

Inspect the repository before changing persistence code. Respect the database provider, EF Core version, existing `DbContext` structure, entity configuration conventions, migration strategy, data-access patterns, transaction strategy, and testing approach. Do not impose architectural patterns only because they are common.

## Relationship With Dotnet Backend

- Use `dotnet-backend` for general .NET backend guidance.
- Use this skill for deeper persistence guidance involving EF Core, schema design, query behavior, transactions, concurrency, migrations, and database constraints.
- Do not rewrite backend architecture just to satisfy EF Core preferences.

## DbContext

- Treat `DbContext` as a scoped unit-of-work object by default in server applications.
- Register and consume contexts through dependency injection according to the repository's existing pattern.
- Avoid long-lived contexts; they accumulate tracked entities, stale state, and unexpected memory usage.
- A `DbContext` is not thread-safe. Concurrent EF operations must not execute against the same context instance.
- Do not share a context between threads, background tasks, parallel loops, or overlapping async operations.
- Await each EF operation before starting the next one on the same context.
- Use `IDbContextFactory<TContext>` or `AddDbContextFactory` when contexts must be created outside a normal request scope, such as background work, UI apps, parallel independent units of work, or explicit short-lived scopes.
- Do not solve context lifetime problems by making `DbContext` singleton, adding arbitrary locks, or disabling EF safeguards.

## Entity Modeling

- Model entities, keys, value generation, relationships, navigations, and requiredness intentionally.
- Prefer explicit modeling when ambiguity could produce the wrong database schema.
- Configure primary keys and alternate keys deliberately; avoid accidental key choices from naming alone when the domain is unclear.
- Choose value generation based on provider behavior and application needs, especially for identity columns, GUIDs, sequences, and computed values.
- Make required vs optional relationships explicit when nullability or delete behavior matters.
- Configure navigation properties and inverse navigations consistently with existing conventions.
- Use owned or complex types for values that have no independent identity and belong to an owning entity.
- Use value conversions for simple persistence transformations, but consider query translation, indexing, and provider support before converting important searchable values.
- Store enums in the form already used by the repository unless there is a clear reason to change. Consider migration and compatibility impact before switching between numeric and string storage.
- Use backing fields only when they protect meaningful domain behavior or encapsulation; do not add them for ceremony.

## Configuration

- Follow the repository's existing configuration style first.
- Use conventions when they are clear, stable, and already match the intended schema.
- Use data annotations for simple constraints when the repository already favors them and they do not obscure persistence details.
- Use Fluent API for relationships, indexes, constraints, conversions, precision, delete behavior, owned types, and provider-specific configuration.
- Use `IEntityTypeConfiguration<T>` when configurations are large, reused, or already organized that way.
- Keep entity configuration close enough to the model that future schema changes are easy to review.

## Queries

- Compose LINQ so filtering, sorting, projection, aggregation, and pagination execute in the database.
- Project directly into DTOs or read models when full entities are unnecessary.
- Use `Select` to load only required columns.
- Use `AsNoTracking()` for read-only queries unless tracked entities are needed later in the unit of work.
- Apply filters before materialization.
- Use deterministic `OrderBy` before pagination.
- Avoid unbounded queries unless the data size is inherently small and controlled.
- Use `Include` only when entity graphs are required; do not include navigations by default.
- Avoid N+1 queries by projecting needed data, using appropriate includes, or issuing intentional batched queries.
- Consider split queries for large include graphs where a single joined query causes cartesian explosion, while accounting for additional round trips and consistency implications.
- Avoid unnecessary materialization with `ToListAsync`, `AsEnumerable`, or similar calls before all database-side operations are composed.
- Inspect generated SQL when query shape, translation, correctness, or performance is uncertain.

```csharp
var orders = await dbContext.Orders
    .AsNoTracking()
    .Where(order => order.CustomerId == customerId)
    .OrderByDescending(order => order.CreatedUtc)
    .Select(order => new OrderSummary(order.Id, order.CreatedUtc, order.Total))
    .ToListAsync(cancellationToken);
```

## Change Tracking

- Use tracking when you intend to modify entities in the same context and persist those changes.
- Avoid tracking for read-only queries.
- Use identity resolution only when a no-tracking query needs consistent entity instances for repeated keys.
- Be explicit when attaching entities; set state only for properties that should actually be written.
- For disconnected updates, load the existing entity when correctness depends on current database state, concurrency tokens, relationships, or partial updates.
- Do not blindly call `Update()` on large disconnected object graphs; it can mark entire graphs modified and persist unintended changes.
- Inspect tracked state when debugging unexpected inserts, updates, deletes, or relationship changes.

## Saving Changes

- Use `SaveChangesAsync` for database I/O and pass `CancellationToken` where appropriate.
- Understand which tracked changes are being persisted before saving.
- Do not call `SaveChanges` or `SaveChangesAsync` repeatedly without a reason when changes belong to one logical unit of work.
- Account for EF Core batching behavior where relevant, but verify provider limits and generated commands before optimizing around it.
- Handle expected persistence failures, such as duplicate keys or concurrency conflicts, at the application boundary where they affect behavior.

## Transactions

- A single `SaveChanges` operation is transactional for supported relational providers.
- Use explicit transactions for multi-step workflows that must commit or roll back together, especially when multiple `SaveChanges` calls are required.
- Keep transaction boundaries intentional and as short as practical.
- Roll back explicitly or rely on disposal according to the repository's established transaction pattern.
- Consider isolation level when correctness depends on what concurrent transactions can read or write.
- Do not hold database transactions open while making slow external HTTP or network calls unless the design explicitly requires it and the consequences are understood.
- Coordinate carefully with external systems; database transactions do not make remote calls atomic.

## Concurrency

- Use optimistic concurrency tokens, such as row version columns or configured concurrency properties, when update conflicts must be detected.
- Treat application-level checks, optimistic concurrency, database constraints, and transaction isolation as different tools.
- Application-level checks can improve messages and flow, but they do not by themselves protect invariants under concurrent requests.
- Optimistic concurrency detects conflicting updates to the same data according to configured tokens.
- Database constraints enforce invariants at write time and are often required for correctness under race conditions.
- Transaction isolation controls visibility and interaction between concurrent transactions, but behavior depends on provider and isolation level.
- For invariants that must remain correct under concurrent requests, do not rely solely on `check then insert/update`. Use database-enforced guarantees where appropriate.
- Handle `DbUpdateConcurrencyException` where conflict behavior is part of the application contract.

## Database Constraints

- Use primary keys, foreign keys, required columns, unique constraints or indexes, and check constraints where they protect real invariants.
- Application validation should not be the only protection for critical data invariants.
- Prefer database-enforced uniqueness for values that must be unique under concurrent requests.
- Configure delete behavior deliberately; do not accept cascade behavior accidentally when deletes have business impact.
- Ensure constraints match the provider's capabilities and the repository's migration strategy.

## Indexes

- Add indexes for frequently filtered, joined, sorted, or uniqueness-enforced columns when the expected workload justifies them.
- Use composite indexes when query predicates and ordering benefit from the column order.
- Consider ordering, selectivity, and included columns only when supported by the provider and useful for actual queries.
- Avoid unnecessary indexes; they add storage cost and slow inserts, updates, and deletes.
- Do not recommend indexes blindly. Inspect query patterns and, when needed, execution plans.

## Migrations

- Create migrations using the repository's existing tooling, project layout, naming convention, and migration strategy.
- Review generated migrations and the model snapshot before committing them.
- Never assume a generated migration is safe merely because EF Core generated it.
- Inspect destructive or risky operations carefully: dropping columns, renaming columns, changing nullability, changing data types, adding unique constraints to existing data, and data transformations.
- For renames, prefer explicit rename operations when the database should preserve data, rather than drop-and-add operations.
- For non-null columns added to existing tables, plan defaults, backfills, or staged deployment as appropriate.
- For unique constraints on existing data, determine how duplicates are handled before deployment.
- Keep model snapshots consistent with migrations.
- Do not automatically apply migrations to production unless the repository's deployment strategy explicitly does so.

## Performance

- Optimize based on observed behavior, generated SQL, query counts, timings, or execution plans, not guesses.
- Look for N+1 queries, excessive round trips, unnecessary columns, excessive navigation loading, premature materialization, tracking read-only entities, unbounded queries, inefficient pagination, and excessive `SaveChanges` calls.
- Prefer server-side filtering, projection, aggregation, ordering, and pagination.
- Measure hot paths before introducing more complex patterns.
- Inspect generated SQL when behavior or performance is uncertain.
- Keep performance changes understandable and covered by tests or diagnostics when the risk is meaningful.

## Pagination

- Apply deterministic ordering before pagination.
- Offset pagination with `Skip` and `Take` is acceptable for many ordinary list views, but can become inefficient or unstable on large or frequently changing result sets.
- Consider keyset or seek pagination for high-volume, sequential navigation where stable performance matters.
- Do not introduce keyset pagination unless the use case warrants the additional complexity.
- Include a unique tie-breaker in ordering when stable page boundaries matter.

## Bulk Operations

- When many rows must change, first decide whether entities need to be loaded at all.
- Use supported set-based operations such as `ExecuteUpdateAsync` or `ExecuteDeleteAsync` when appropriate and compatible with the repository's EF Core version and provider.
- Account for the fact that set-based operations bypass normal change tracking, the `SaveChanges` pipeline for tracked entities, and tracked in-memory entity state.
- Verify that filters are correct before running destructive bulk operations.

## Raw SQL

- Use LINQ by default when it expresses the query clearly and translates well.
- Use raw SQL for queries EF cannot express efficiently, database-specific functionality, or carefully optimized hot paths.
- Always parameterize external values.
- Never construct SQL by concatenating untrusted input.
- Keep raw SQL isolated, reviewed, and covered by tests when it affects important behavior.

## Async Behavior

- Use async EF Core APIs for database I/O.
- Await async operations directly; do not block on them with `.Result` or `.Wait()`.
- Propagate `CancellationToken` from request or job boundaries through EF calls where appropriate.

```csharp
await dbContext.Orders
    .AsNoTracking()
    .Where(order => order.Status == OrderStatus.Open)
    .ToListAsync(cancellationToken);
```

Avoid:

```csharp
dbContext.Orders.ToListAsync().Result;
```

## Repository Patterns

- Do not automatically wrap EF Core in a generic repository.
- Recognize that `DbContext` and `DbSet<T>` already provide repository/unit-of-work-like capabilities.
- If the existing application uses repositories, respect that architecture.
- Repositories should provide meaningful domain or application boundaries, not simply duplicate every `DbSet` operation.
- Keep query specifications, repositories, or data services aligned with existing patterns and justified by actual complexity.

## Testing

- Follow the project's testing conventions, fixtures, database setup, and naming patterns.
- Do not treat EF Core InMemory as equivalent to a relational database.
- Use EF Core InMemory only when its differences from the production provider are irrelevant to the behavior under test.
- When behavior depends on SQL translation, relational constraints, transactions, provider-specific behavior, migrations, or concurrency, prefer the actual database engine or an appropriate disposable instance.
- Test important persistence behavior at the level that proves the risk without duplicating the full `dotnet-testing` guidance.

## Diagnostics And Troubleshooting

- Start with exception details, including inner exceptions and provider-specific error codes.
- Check generated SQL, query count, tracked entity state, context lifetime, concurrent usage, constraints, transaction boundaries, and provider behavior.
- Use database execution plans when performance warrants it.
- For `A second operation was started on this context instance before a previous operation completed`, investigate concurrent operations using the same `DbContext` instance before attempting workarounds.
- Do not solve `DbContext` concurrency problems by disabling EF safeguards or adding arbitrary locks without understanding the underlying lifetime or concurrency problem.
- For unexpected data changes, inspect tracking state, cascade behavior, relationship fixup, disconnected updates, and calls to `Update`, `Attach`, and `Remove`.

## Common Anti-Patterns

- Singleton `DbContext`.
- Sharing `DbContext` between threads.
- Parallel EF operations on one context.
- Blocking async EF calls with `.Result` or `.Wait()`.
- N+1 queries.
- Unnecessary `Include`.
- Loading entire tables.
- Missing pagination for unbounded result sets.
- Tracking read-only queries unnecessarily.
- Generic repositories that merely mirror `DbSet`.
- Calling `SaveChanges` repeatedly without reason.
- Check-then-insert logic without database protection.
- Blindly calling `Update()` on disconnected graphs.
- Trusting generated migrations without reviewing them.
- Concatenating user input into raw SQL.

## Implementation Workflow

1. Inspect the existing `DbContext` and persistence architecture.
2. Identify the database provider and EF Core version.
3. Inspect relevant entities and configurations.
4. Understand the required data invariant.
5. Determine whether schema changes are necessary.
6. Implement the smallest coherent change.
7. Consider query behavior and generated SQL.
8. Consider concurrency and transaction boundaries.
9. Create and review migrations when required.
10. Add or update relevant tests.
11. Build the affected projects.
12. Run relevant tests.

## Review Checklist

- Is `DbContext` lifetime scoped or otherwise correct for the execution model?
- Are EF operations free from unsafe concurrent use of the same context?
- Are async EF APIs awaited without `.Result`, `.Wait()`, or sync-over-async?
- Is `CancellationToken` propagated where appropriate?
- Does entity configuration match existing conventions and intended schema?
- Are keys, relationships, requiredness, navigations, and delete behavior explicit where needed?
- Are queries efficient, bounded, and projected when full entities are unnecessary?
- Is tracking used only when needed?
- Are N+1 queries and unnecessary includes avoided?
- Does pagination use deterministic ordering?
- Are critical invariants protected by database constraints where appropriate?
- Are indexes justified by query patterns or uniqueness requirements?
- Are transaction boundaries explicit and short enough?
- Are concurrency conflicts and race-prone invariants handled correctly?
- Were generated migrations reviewed for destructive or risky operations?
- Is raw SQL parameterized and isolated?
- Is data integrity protected in both application logic and the database where necessary?
- Does test coverage match the persistence risks of the change?
