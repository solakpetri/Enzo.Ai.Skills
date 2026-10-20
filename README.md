# Agent Skills for .NET and Rust

Reusable Agent Skills that provide AI coding agents with practical guidance for building, testing, reviewing, and maintaining .NET and Rust applications.

Agent Skills are self-contained instruction sets stored as `SKILL.md` files. They give compatible agents focused, reusable guidance for a specific task or domain without hard-coding project-specific assumptions.

## Available Skills

### .NET

| Skill | Description |
| --- | --- |
| [`dotnet-backend`](skills/dotnet-backend/) | Production-oriented guidance for designing, implementing, refactoring, and reviewing .NET backend code across ASP.NET Core APIs, dependency injection, async I/O, EF Core, transactions, external integrations, security, logging, and maintainable architecture. |
| [`dotnet-testing`](skills/dotnet-testing/) | Practical guidance for creating and reviewing automated .NET tests, including unit, integration, API, database, concurrency, idempotency, and ASP.NET Core `WebApplicationFactory` scenarios. |
| [`ef-core`](skills/ef-core/) | Practical guidance for designing, implementing, optimizing, troubleshooting, and reviewing Entity Framework Core persistence code. |
| [`code-review`](skills/code-review/) | Review guidance for .NET code changes, pull requests, patches, and proposed implementations. |
| [`architecture-review`](skills/architecture-review/) | Architecture review guidance for .NET application boundaries, dependencies, coupling, data flow, reliability, scalability, deployment, persistence, and integrations. |

### Rust

| Skill | Description |
| --- | --- |
| [`rust-core`](skills/rust-core/) | Idiomatic Rust guidance for ownership, borrowing, lifetimes, type-driven design, traits and generics, error handling, public API design, semver, Cargo workspaces, features, lints, and documentation. |
| [`rust-async`](skills/rust-async/) | Guidance for asynchronous and concurrent Rust, including Tokio, `Send` and `Sync` bounds, blocking work, cancellation safety, structured concurrency, shutdown, locks, channels, backpressure, and timeouts. |
| [`rust-backend`](skills/rust-backend/) | Production-oriented guidance for Rust backend services across Axum and similar frameworks, tower middleware, state, validation, error mapping, configuration, tracing, outbound HTTP, auth, and graceful shutdown. |
| [`rust-testing`](skills/rust-testing/) | Practical guidance for Rust tests, including unit, integration, doc, async, HTTP, and database tests, test doubles, property-based and snapshot testing, `loom`, Miri, fuzzing, and benchmarks. |
| [`rust-database`](skills/rust-database/) | Guidance for database access with sqlx, Diesel, or SeaORM, including pools, transactions, concurrency control, constraints, pagination, bulk operations, dynamic SQL safety, and migrations. |
| [`rust-performance`](skills/rust-performance/) | Guidance for measuring and improving Rust runtime performance, memory use, compile time, and binary size, from profiling and allocation discipline to release profiles, PGO, and allocators. |
| [`rust-unsafe-ffi`](skills/rust-unsafe-ffi/) | Guidance for writing and auditing unsafe Rust and FFI, including soundness, safety contracts, undefined behavior, raw pointers, panic safety, ownership across language boundaries, and Miri. |
| [`rust-code-review`](skills/rust-code-review/) | Review guidance for Rust code changes, pull requests, patches, and proposed implementations. |
| [`rust-architecture-review`](skills/rust-architecture-review/) | Architecture review guidance for Rust crate and workspace boundaries, dependency direction, runtime topology, shared state, features, supply chain, build time, deployment, and integrations. |

## Installation / Usage

The exact installation flow depends on the agent you use, but the general approach is the same:

1. Clone or download this repository.
2. Copy or reference the skill directory you want.
3. Place or configure it in the location your agent expects for Agent Skills.

```bash
git clone https://github.com/solakpetri/Enzo.Ai.Skills.git
```

Each skill is self-contained. You can use one skill without installing the others.

### OpenCode

OpenCode discovers skills from directories that contain `SKILL.md` files.

The simplest approach is to copy one or more skill directories into a supported OpenCode skills location, preserving the directory structure. Common locations are project-local `.opencode/skills/` and global `~/.config/opencode/skills/`:

```text
.opencode/skills/
├── dotnet-backend/
│   └── SKILL.md
├── dotnet-testing/
│   └── SKILL.md
├── ef-core/
│   └── SKILL.md
├── code-review/
│   └── SKILL.md
├── architecture-review/
│   └── SKILL.md
├── rust-core/
│   └── SKILL.md
├── rust-async/
│   └── SKILL.md
└── ...
```

OpenCode also supports registering an external skills directory through `skills.paths` in `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": {
    "paths": ["/path/to/Enzo.Ai.Skills/skills"]
  }
}
```

After adding or changing skills, restart OpenCode so it reloads configuration and skills.

### Other Agents

These skills use the Agent Skills `SKILL.md` structure. Other agents that support the same format, or a compatible one, may be able to use them with little or no modification.

Skills are not commands. Once a skill is available to an agent, developers can usually prompt normally and let the agent use the task context to decide what guidance applies. For example, a request like this may be enough for an agent that automatically discovers applicable skills:

```text
Add an endpoint for creating orders.
```

Loading and discovery behavior depends on the agent being used, so follow your agent's documentation for details.

## Usage Examples

Once a skill is available to the agent, developers can prompt normally. The skill provides specialized context and guidance for the task without requiring developers to duplicate the full skill instructions in every prompt.

### .NET Backend

[`dotnet-backend`](skills/dotnet-backend/)

```text
Add a POST /orders endpoint that creates a new order.

Inspect the existing solution first and follow its architecture and conventions.

Validate the request, persist the order, handle expected errors, use async APIs and CancellationToken, and add relevant tests.

Build the affected projects and run the relevant tests when finished.
```

### .NET Testing

[`dotnet-testing`](skills/dotnet-testing/)

```text
Add tests for order submission.

An order with no order lines must not be submitted.

Inspect the existing test projects and follow their framework and conventions.

Cover the successful case and important failure cases. Choose the smallest test level that reliably proves the behavior and avoid unnecessary mocking.

Run the relevant tests when finished.
```

### EF Core

[`ef-core`](skills/ef-core/)

```text
Add a query that returns the 20 most recent orders for a customer.

Return only:
- order id
- created date
- status
- total amount

Inspect the existing persistence code first. Use an efficient projection, read-only query behavior where appropriate, deterministic ordering, and CancellationToken.

Avoid loading unnecessary entities or navigation properties.

Add relevant tests and run them.
```

### Code Review

[`code-review`](skills/code-review/)

```text
Review the changes in the current branch against the default branch.

Do not modify any files.

Focus on defects introduced by the change, especially correctness, data integrity, concurrency, security, API behavior, EF Core usage, and missing meaningful tests.

For each finding, provide the severity, location, concrete failure scenario, consequence, and recommended direction.

Do not manufacture findings if the change is sound.
```

### Architecture Review

[`architecture-review`](skills/architecture-review/)

```text
Review the architecture of this .NET solution.

Do not modify any files.

Evaluate module and service boundaries, dependency direction, data ownership, persistence boundaries, integrations, reliability, scalability, and operational complexity.

Identify concrete architectural risks and explain their tradeoffs.

Do not recommend microservices or additional abstractions unless they solve a demonstrated problem.
```

### Rust Core

[`rust-core`](skills/rust-core/)

```text
Add a parser for the order import file format.

Inspect the crate's existing error types, module layout, and lints first.

Parse input into validated domain types, return a structured error type instead of panicking, and avoid unnecessary clones and allocations.

Run cargo clippy --all-targets and the relevant tests when finished.
```

### Rust Async

[`rust-async`](skills/rust-async/)

```text
Add a background worker that consumes order events from the queue.

Inspect the existing runtime setup and shutdown handling first.

Bound concurrency, keep blocking work off the runtime threads, make sure cancellation cannot lose messages, and wire the worker into graceful shutdown.

Add tests using paused Tokio time and run them.
```

### Rust Backend

[`rust-backend`](skills/rust-backend/)

```text
Add a POST /orders endpoint that creates a new order.

Inspect the existing service first and follow its framework, state, error, and module conventions.

Validate the request, persist the order, map errors to the existing error response format, enforce authorization, and add relevant tests through the router.

Run cargo check, cargo clippy, and the relevant tests when finished.
```

### Rust Testing

[`rust-testing`](skills/rust-testing/)

```text
Add tests for order submission.

An order with no order lines must not be submitted.

Inspect the existing test layout and dev-dependencies and follow their conventions.

Cover the successful case and important failure cases. Choose the smallest test level that reliably proves the behavior, and avoid adding traits only for mocking.

Run the relevant tests, including doctests, when finished.
```

### Rust Database

[`rust-database`](skills/rust-database/)

```text
Add a query that returns the 20 most recent orders for a customer.

Return only:
- order id
- created date
- status
- total amount

Inspect the existing data-access code first. Use explicit columns, bind parameters, deterministic ordering, and keyset pagination.

Add tests against the real database engine and run them.
```

### Rust Performance

[`rust-performance`](skills/rust-performance/)

```text
The order export endpoint is slow for large customers.

Profile a release build with realistic data before changing anything.

Fix the measured bottleneck, add a criterion benchmark that protects the improvement, and report before and after numbers.

Do not add unsafe code or change global build settings without explaining the tradeoff.
```

### Rust Unsafe And FFI

[`rust-unsafe-ffi`](skills/rust-unsafe-ffi/)

```text
Audit the unsafe code in the ring buffer module.

Do not modify any files.

For each unsafe block, verify the SAFETY reasoning, check whether safe code in the module can break the invariants, and check panic safety.

Report unsound APIs with a concrete triggering sequence of safe calls.
```

### Rust Code Review

[`rust-code-review`](skills/rust-code-review/)

```text
Review the changes in the current branch against the default branch.

Do not modify any files.

Focus on defects introduced by the change, especially correctness, reachable panics, unsafe soundness, async cancellation, locks held across await, data integrity, security, and semver breaks.

For each finding, provide the severity, location, concrete failure scenario, consequence, and recommended direction.

Do not manufacture findings if the change is sound.
```

### Rust Architecture Review

[`rust-architecture-review`](skills/rust-architecture-review/)

```text
Review the architecture of this Rust workspace.

Do not modify any files.

Evaluate crate boundaries, dependency direction, runtime topology, shared state, data ownership, feature design, dependency risk, build time, and deployment.

Identify concrete architectural risks and explain their tradeoffs.

Do not recommend splitting crates or services unless it solves a demonstrated problem.
```

## Repository Structure

```text
Enzo.Ai.Skills/
├── skills/
│   ├── architecture-review/
│   │   └── SKILL.md
│   ├── code-review/
│   │   └── SKILL.md
│   ├── dotnet-backend/
│   │   └── SKILL.md
│   ├── dotnet-testing/
│   │   └── SKILL.md
│   ├── ef-core/
│   │   └── SKILL.md
│   ├── rust-architecture-review/
│   │   └── SKILL.md
│   ├── rust-async/
│   │   └── SKILL.md
│   ├── rust-backend/
│   │   └── SKILL.md
│   ├── rust-code-review/
│   │   └── SKILL.md
│   ├── rust-core/
│   │   └── SKILL.md
│   ├── rust-database/
│   │   └── SKILL.md
│   ├── rust-performance/
│   │   └── SKILL.md
│   ├── rust-testing/
│   │   └── SKILL.md
│   └── rust-unsafe-ffi/
│       └── SKILL.md
├── LICENSE
└── README.md
```

Each skill lives in its own directory and is meant to be reusable on its own.

## Design Principles

- Practical guidance over framework documentation.
- Production-oriented recommendations.
- Respect existing repository conventions before introducing new patterns.
- Avoid unnecessary abstractions.
- Focus on decisions AI coding agents actually need to make.
- Stay framework- and library-neutral where that improves reuse.

## Contributing

Contributions are welcome for:

- improving existing skills
- correcting inaccurate guidance
- adding useful examples
- proposing new .NET- or Rust-focused skills

Please keep skills:

- focused
- actionable
- technically accurate
- broadly applicable
- concise enough to be useful as agent context

## License

This project is licensed under the [MIT License](LICENSE).
