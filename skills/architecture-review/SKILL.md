---
name: architecture-review
description: Review .NET application architecture for boundaries, dependencies, coupling, data flow, scalability, reliability, maintainability, deployment, persistence, integrations, and architectural tradeoffs. Use when evaluating solution structure, service boundaries, modularity, microservices, layered architectures, distributed systems, or proposed architectural changes.
license: MIT
---

# Architecture Review

Use this skill when reviewing the architecture of a .NET application or a proposed architectural change.

The goal is to identify structural risks and poor tradeoffs that may not be visible from individual code changes. Evaluate whether the architecture is appropriate for the actual problem, not whether it matches a fashionable pattern.

Prefer the simplest architecture that satisfies current and reasonably foreseeable requirements. Do not recommend microservices, CQRS, event sourcing, Clean Architecture, Hexagonal Architecture, or any other pattern simply because it is considered best practice.

## Relationship With Other Skills

- Use `dotnet-backend` for implementation-level .NET backend, ASP.NET Core, dependency injection, async, security, logging, and integration guidance.
- Use `ef-core` for deeper Entity Framework Core persistence, schema, query, migration, transaction, and database constraint analysis.
- Use `dotnet-testing` for testing architecture, test strategy, integration testing, concurrency testing, and regression coverage guidance.
- Use `code-review` for line-level pull request findings, changed-code defects, severity discipline, and review output style.
- Keep this skill focused on system-level design, boundaries, dependencies, deployment, operational risks, and architectural tradeoffs.

## Review Scope

Before reviewing architecture:

1. Inspect the repository structure.
2. Identify projects, modules, services, and deployment units.
3. Inspect dependency direction.
4. Identify application boundaries.
5. Identify persistence boundaries.
6. Identify external integrations.
7. Understand deployment topology where visible.
8. Understand the main business workflows.
9. Identify cross-cutting concerns.
10. Identify important non-functional requirements.

Do not infer architectural requirements that are not supported by the repository or stated problem. Separate observed architecture from assumptions. If information is missing, state the uncertainty instead of inventing infrastructure.

## Understand The System First

Identify:

- What the system does.
- Main business capabilities.
- Important workflows.
- Data ownership.
- External dependencies.
- Deployment units.
- Consistency requirements.
- Availability requirements.
- Likely scaling characteristics.
- Security boundaries.

Treat code, project references, configuration, infrastructure files, database migrations, message contracts, API contracts, CI/CD files, Docker files, and deployment manifests as architectural evidence. Do not assume scale, availability, or deployment requirements that the repository does not show.

## Boundaries

Review whether responsibilities are separated at useful boundaries:

- Modules.
- Projects.
- Services.
- Bounded contexts.
- Domain areas.
- Infrastructure boundaries.
- API boundaries.

Look for:

- Unclear ownership.
- Duplicated responsibilities.
- Inappropriate shared state.
- Accidental coupling.
- Circular dependencies.
- Unrelated functionality grouped together.
- Artificial fragmentation.

Do not recommend splitting components unless the split creates a meaningful ownership, deployment, scaling, operational, or domain boundary. Do not punish a simple monolith when it matches the system's needs.

## Dependency Direction

Inspect how components depend on each other. Check project references, package references, namespace usage, startup wiring, dependency injection registration, shared libraries, message contracts, and generated clients.

Look for:

- Circular project references.
- Domain logic depending on infrastructure.
- Lower-level components depending on higher-level delivery mechanisms.
- Shared libraries that become dependency dumping grounds.
- Direct coupling between unrelated modules.
- APIs or workers reaching into another module's internals.

Prefer stable dependency directions. Do not enforce a strict layered architecture if the repository intentionally uses a different coherent model.

## Coupling And Cohesion

Evaluate whether components have high internal cohesion, understandable responsibilities, and minimal unnecessary coupling.

Identify coupling through:

- Direct references.
- Shared databases.
- Shared mutable state.
- Message contracts.
- Synchronous HTTP calls.
- Common libraries.
- Configuration.
- Deployment assumptions.

Distinguish unavoidable business coupling from accidental technical coupling. A useful architecture review explains what changes together today, what is likely to change independently, and where the architecture makes those changes harder than necessary.

## Project And Solution Structure

For .NET solutions, inspect whether project boundaries have clear purposes. Possible project types include:

- API.
- Application.
- Domain.
- Infrastructure.
- Worker.
- Contracts.
- Shared libraries.
- Tests.

Do not require a separate project for every conceptual layer. Additional projects increase maintenance, navigation, package management, build time, dependency management, and test setup complexity. Recommend project separation only when it creates a meaningful boundary.

## Modularity

Evaluate whether the application can evolve without unrelated areas changing together.

Look for:

- Feature leakage across modules.
- Shared internal models.
- Global services.
- Overly broad interfaces.
- Central utility projects.
- Cross-module database access.
- Configuration that implicitly joins otherwise separate components.

Prefer explicit contracts between modules. Avoid creating abstractions solely to claim modularity.

## Monoliths And Microservices

Do not assume microservices are inherently better.

For a monolith, review:

- Modularity.
- Ownership boundaries.
- Deployment simplicity.
- Shared database coupling.
- Ability to evolve components independently.
- Whether a modular monolith would solve the actual pressure without distributed complexity.

For microservices, review:

- Service boundaries.
- Independent deployment.
- Data ownership.
- Network failure handling.
- Operational complexity.
- Observability.
- Consistency.
- Versioned contracts.

Flag distributed architectures that provide little benefit over a simpler modular monolith. A distributed monolith often has the cost of microservices without independent ownership, independent deployment, or clear data boundaries.

## Service Boundaries

When reviewing microservices or distributed systems, check whether services are separated by meaningful business capabilities.

Avoid boundaries based only on:

- Database tables.
- CRUD entities.
- Technical layers.
- Arbitrary size limits.

A service boundary should usually imply meaningful ownership and lifecycle independence. Services that must be changed, deployed, scaled, and queried together may not be meaningfully separate.

## Data Ownership

Identify which component owns each important piece of mutable business data.

Look for:

- Multiple services writing the same tables.
- Shared databases creating hidden coupling.
- Unclear authoritative sources.
- Duplicated data without a synchronization strategy.
- Direct cross-service database access.
- Read models that are treated as authoritative write models.

Prefer one clear owner for mutable business data. Other components should use explicit contracts or replicated read models where appropriate.

## Persistence

Evaluate whether persistence choices align with the domain and consistency needs.

Review:

- Database boundaries.
- Transaction boundaries.
- Schema ownership.
- Query patterns.
- Consistency expectations.
- Migrations.
- Persistence coupling.

Use `ef-core` for deeper Entity Framework Core-specific analysis. Do not treat repositories or Unit of Work abstractions as mandatory architectural requirements.

## Transactions And Consistency

Identify operations that require atomic behavior. Distinguish:

- Local database transactions.
- Cross-service consistency.
- Eventual consistency.
- Distributed workflows.

Avoid assuming a distributed transaction is required. For distributed operations, consider patterns such as outbox, inbox, idempotency, retries, compensating actions, and workflow orchestration only when the failure mode requires them.

Make the consistency model explicit. If the architecture permits temporary inconsistency, check whether users, downstream systems, jobs, and support workflows can tolerate and diagnose it.

## Messaging And Events

If the system uses messaging, review:

- Ownership of events.
- Message contracts.
- Idempotency.
- Duplicate delivery.
- Ordering assumptions.
- Retries.
- Dead-letter handling.
- Schema evolution.
- Transactional publishing.

Distinguish domain events from integration events where that distinction is useful. Do not recommend event-driven architecture simply to reduce direct method calls.

## Synchronous Integrations

Review synchronous service-to-service or external HTTP dependencies for:

- Latency coupling.
- Availability coupling.
- Timeout behavior.
- Retry behavior.
- Cascading failures.
- Authentication.
- Rate limiting.

Identify critical request chains where one dependency failure can take down multiple upstream components. Prefer changes that reduce concrete failure impact without adding unnecessary asynchronous complexity.

## Resilience And Reliability

Review architecture for realistic failure modes:

- Process crashes.
- Pod, container, or host restarts.
- Dependency failures.
- Partial database writes.
- Repeated message delivery.
- Long-running workflows.
- Interrupted background jobs.
- Queue growth.
- Backpressure.

Consider timeouts, retries, circuit breakers, idempotency, rate limiting, backpressure, durable checkpoints, and graceful degradation. Do not add resilience mechanisms indiscriminately. Each mechanism should address a concrete failure mode.

For long-running or restart-sensitive work, review whether progress can be resumed safely. Look for idempotent processing and durable checkpoints where required.

## Background Processing

When workers or scheduled jobs exist, review:

- Ownership of work.
- Persistence of work state.
- Retries.
- Duplicate execution.
- Concurrency.
- Cancellation.
- Shutdown behavior.
- Restart recovery.

Do not rely on in-memory state for work that must survive process termination. Check whether scheduled work can overlap accidentally and whether repeated execution is safe.

## Scalability

Evaluate likely scaling bottlenecks based on actual workloads where known.

Check:

- Stateful components.
- Database bottlenecks.
- Hot partitions.
- Shared resources.
- Synchronous chains.
- Long-running operations.
- Unbounded queues.
- Cache contention.

Do not recommend horizontal scaling by default when a simpler optimization would solve the problem. Distinguish throughput, latency, storage, and concurrency requirements.

## Performance

Review architectural performance risks such as:

- Excessive network hops.
- Chatty service communication.
- Repeated database access.
- Unnecessary serialization.
- Synchronous processing of expensive work.
- Oversized shared payloads.

Do not focus on low-level micro-optimizations. Use `code-review` for line-level performance findings.

## API Boundaries

Review whether public and internal APIs have clear contracts.

Check:

- Versioning.
- Backward compatibility.
- Ownership.
- Request and response boundaries.
- Internal model leakage.
- Error contracts.

Avoid exposing persistence entities directly as long-lived external contracts when that creates coupling. Review whether consumers can evolve independently and whether breaking changes have a migration path.

## Security Boundaries

Review:

- Trust boundaries.
- Authentication.
- Authorization.
- Service identities.
- Secret ownership.
- Network assumptions.
- Data sensitivity.

Do not assume traffic is trusted simply because it is internal. Identify where authorization decisions belong and whether data crosses boundaries that require different controls.

## Configuration

Evaluate whether configuration ownership is clear.

Look for:

- Shared configuration files.
- Duplicated settings.
- Environment coupling.
- Secrets mixed with normal configuration.
- Configuration that implicitly couples services.

Configuration should not become an undocumented runtime dependency graph.

## Observability

Check whether the architecture supports diagnosing distributed or asynchronous behavior.

Look for:

- Structured logs.
- Correlation identifiers.
- Tracing.
- Meaningful metrics.
- Health checks.
- Dependency visibility.

Do not require every observability technology. Focus on whether failures can realistically be diagnosed in the architecture the repository actually has.

## Deployment Architecture

Where repository or deployment files provide enough information, inspect:

- Deployment units.
- Containers.
- Pods.
- Process boundaries.
- Stateful versus stateless services.
- Scaling assumptions.
- Health checks.
- Readiness and liveness.
- Startup dependencies.
- Graceful shutdown.

Do not invent Kubernetes-specific concerns if the application is not deployed on Kubernetes. If deployment topology is not visible, say that the review is limited by missing deployment evidence.

## Shared Libraries

Review shared or common projects carefully. Watch for `Common`, `Shared`, or `Utils` projects becoming repositories for unrelated behavior.

Shared libraries should usually contain genuinely stable cross-cutting contracts or utilities. Avoid using shared libraries to bypass proper service or module boundaries. Flag shared business logic when it creates unclear ownership or forces unrelated components to change together.

## Abstractions

Assess whether abstractions create actual value.

Flag:

- Interfaces with only one implementation and no boundary purpose.
- Generic repositories that hide EF Core without adding semantics.
- Excessive factories.
- Wrappers around simple framework APIs.
- Unnecessary mediator layers.
- Deep indirection.

Do not remove abstractions that provide meaningful testability, ownership, substitutability, or integration boundaries.

## Architectural Patterns

Treat patterns as tools. When encountering Clean Architecture, Hexagonal Architecture, Onion Architecture, CQRS, Event Sourcing, DDD, Microservices, Modular Monolith, Repository, or Unit of Work, evaluate whether the pattern solves a concrete problem.

Do not judge architecture based on pattern purity. A partial or pragmatic implementation may be correct if it preserves useful boundaries and avoids unnecessary complexity.

## Architectural Change Review

When reviewing a proposed architectural change, ask:

1. What problem does this change solve?
2. Is the problem real today?
3. Is there a simpler solution?
4. What complexity does the change add?
5. What new operational burden does it introduce?
6. What coupling does it remove or introduce?
7. What becomes easier?
8. What becomes harder?
9. How reversible is the decision?

Prefer reversible decisions when uncertainty is high. Do not prescribe a large rewrite unless the risk clearly justifies it.

## Common Architectural Anti-Patterns

Actively look for:

- Distributed monolith.
- Shared database across independently deployed services.
- Circular project dependencies.
- `Common` projects containing business logic.
- Microservices split by CRUD entity.
- Synchronous chains across many services.
- Multiple services owning the same mutable data.
- Excessive abstraction.
- Infrastructure concerns leaking into domain logic.
- In-memory state for durable workflows.
- Queues without idempotent consumers.
- Retries without idempotency.
- Event-driven design without ownership clarity.
- Architecture based primarily on fashionable patterns.

These are not automatic findings. Verify the concrete consequence against actual system needs before reporting them.

## Review Workflow

When performing an architecture review:

1. Understand the system purpose.
2. Map major components.
3. Identify dependencies.
4. Identify data ownership.
5. Identify critical workflows.
6. Identify external dependencies.
7. Identify consistency boundaries.
8. Identify failure boundaries.
9. Assess coupling and cohesion.
10. Assess deployment topology.
11. Assess scalability and reliability risks.
12. Inspect security boundaries.
13. Inspect observability.
14. Evaluate architectural tradeoffs.
15. Verify each finding against actual system needs.
16. Produce prioritized recommendations.

## Review Checklist

Before completing an architecture review, consider:

- Boundaries.
- Ownership.
- Dependency direction.
- Coupling.
- Cohesion.
- Modularity.
- Service boundaries.
- Data ownership.
- Persistence.
- Transaction boundaries.
- Consistency.
- Messaging.
- Synchronous dependencies.
- Resilience.
- Idempotency.
- Scalability.
- Reliability.
- Background processing.
- API contracts.
- Security boundaries.
- Configuration.
- Deployment.
- Observability.
- Shared libraries.
- Abstraction complexity.
- Reversibility of decisions.

Do not force a finding for every checklist item.

## Review Output

Organize findings by severity.

### Critical

Architectural issues likely to cause severe data loss, security failure, or systemic production failure.

### High

Major architectural risks that significantly affect reliability, scalability, maintainability, or correctness.

### Medium

Meaningful structural problems with moderate impact.

### Low

Smaller but concrete architectural concerns.

### Suggestions

Optional improvements or future considerations.

For every finding:

1. Identify the relevant component or boundary.
2. Describe the architectural problem.
3. Explain the concrete consequence.
4. Explain the tradeoff.
5. Recommend a practical direction.

If the architecture is reasonable for the system's requirements, say so. Do not manufacture architectural complexity merely to produce findings. Simple architecture can be good architecture.

## Style

- Focus on concrete architectural consequences.
- Explain tradeoffs.
- Avoid pattern dogma.
- Avoid style criticism.
- Avoid speculative scale requirements.
- Prefer incremental improvements.
- Prefer simple architectures when they meet requirements.
- Distinguish confirmed issues from assumptions.
- Avoid proposing large rewrites without strong justification.

Architecture review should help teams make better decisions, not score them against an idealized architecture diagram.

## Validation

Before finishing:

1. Verify the frontmatter is valid.
2. Verify the skill name matches its directory.
3. Inspect existing skills for overlap.
4. Remove unnecessary duplication.
5. Verify recommendations are not tied to one architecture style.
6. Verify no unrelated files were changed.
7. Review the final git diff.
