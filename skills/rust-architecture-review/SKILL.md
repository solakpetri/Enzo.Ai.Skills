---
name: rust-architecture-review
description: Review the architecture of Rust applications, libraries, and workspaces, covering crate and module boundaries, dependency direction, trait seams, generic propagation, error strategy, runtime and threading topology, shared state and task ownership, data ownership, persistence, consistency, messaging, integrations, resilience, observability, security boundaries, FFI and plugin boundaries, feature-flag design, dependency supply chain, build time, deployment, and architectural tradeoffs. Use when evaluating workspace structure, service boundaries, modularity, monoliths versus services, library design at scale, or proposed architectural changes in a Rust codebase.
license: MIT
---

# Rust Architecture Review

Use this skill when reviewing the architecture of a Rust application, library, workspace, or a proposed architectural change.

The goal is to identify structural risks and poor tradeoffs that are not visible from individual code changes. Evaluate whether the architecture fits the actual problem, not whether it matches a fashionable pattern.

Prefer the simplest architecture that meets current and reasonably foreseeable requirements. Do not recommend microservices, hexagonal architecture, CQRS, event sourcing, actor frameworks, or a crate per layer simply because they are considered best practice.

## Relationship With Other Skills

- Use `rust-core` for crate API design, error types, features, and Cargo details.
- Use `rust-async` for runtime, task, locking, and shutdown details.
- Use `rust-backend` for service implementation guidance.
- Use `rust-database` for persistence, transaction, and migration analysis.
- Use `rust-testing` for test architecture and strategy.
- Use `rust-performance` for performance and build-time analysis.
- Use `rust-unsafe-ffi` for unsafe and FFI boundaries.
- Use `rust-code-review` for line-level findings in a specific change.
- Keep this skill focused on system-level design, boundaries, dependencies, deployment, operational risks, and tradeoffs.

## Review Scope

Before reviewing architecture:

1. Inspect the workspace layout and every crate's purpose (`Cargo.toml`, `cargo metadata`, `cargo tree`).
2. Identify binaries, libraries, and deployment units.
3. Map the dependency graph between internal crates and key external dependencies.
4. Identify application, domain, and infrastructure boundaries.
5. Identify persistence boundaries and data ownership.
6. Identify external integrations and protocols.
7. Understand deployment topology where visible (Dockerfiles, manifests, CI, release tooling).
8. Understand the main workflows and their runtime execution model.
9. Identify cross-cutting concerns: configuration, errors, telemetry, auth.
10. Identify important non-functional requirements: latency, throughput, memory, startup, binary size, portability, and safety.

Do not infer requirements that the repository or stated problem does not support. Separate observed architecture from assumptions. If information is missing, state the uncertainty.

## Understand The System First

Identify what the system does, its main capabilities, important workflows, data ownership, external dependencies, deployment units, consistency and availability requirements, scaling characteristics, and security boundaries.

Treat code, crate manifests, feature definitions, build scripts, configuration, migrations, protocol definitions (`.proto`, OpenAPI), CI/CD files, and deployment manifests as architectural evidence.

## Crate And Workspace Boundaries

Review whether crate boundaries serve a purpose. Good reasons for a separate crate include:

- Independent publishing or versioning.
- Compile-time parallelism and incremental build speed on a large codebase.
- Enforcing dependency direction, since a crate cannot reach into another crate's private items.
- Isolating heavy, optional, platform-specific, or `unsafe` dependencies.
- Procedural macros, which must be separate crates.
- Separating `-sys` bindings from safe wrappers.

Look for:

- A crate per conceptual layer (`domain`, `application`, `infrastructure`, `api`) in a small codebase, adding manifest, visibility, and orphan-rule friction without real benefit.
- One giant crate whose compile time and coupling have become a bottleneck.
- `common`, `shared`, `utils`, or `core` crates that every crate depends on and that accumulate unrelated business logic.
- Workspace dependency versions that drift between crates when `[workspace.dependencies]` would unify them.

Modules within one crate are often the right boundary. Recommend crate splits only when they create a meaningful ownership, build, publishing, or dependency boundary.

## Dependency Direction

Cargo forbids cycles between crates, but modules inside a crate can still be tangled. Check:

- Whether domain logic depends on infrastructure crates (web frameworks, database drivers, cloud SDKs).
- Whether low-level crates depend on delivery mechanisms such as HTTP types, CLI parsing, or a specific runtime.
- Whether internal modules reach into each other's internals through broad `pub` or `pub(crate)` visibility.
- Whether external dependency types leak through internal crate APIs, coupling every crate to one dependency version.
- Whether the orphan rule is forcing trait implementations into awkward places, which signals that types or traits live in the wrong crate.

Prefer stable dependency directions. Do not enforce strict layering if the codebase intentionally uses a different coherent model.

## Trait Seams And Generic Propagation

- Evaluate whether traits mark real boundaries: external systems, alternate backends, and test substitution of I/O. Traits with one implementation and no boundary purpose add indirection.
- Watch for generic parameters spreading through the system (`Service<R: Repo, C: Clock, M: Mailer, ...>`). This inflates compile time and signatures, and makes every layer generic. Consider `Arc<dyn Trait>` at application boundaries where dynamic dispatch cost is irrelevant.
- Watch for `dyn` traits on hot paths where generics would be appropriate.
- Check object safety and `Send + Sync` requirements of boundary traits used in async services.
- Prefer concrete types inside a module and traits at its edges.

## Error Strategy

- Is there a coherent strategy, such as typed errors in libraries and domain code, and opaque errors with context at application edges?
- Do errors cross crate boundaries with meaningful variants, or does every crate wrap every other crate's errors?
- Is error translation to external contracts (HTTP, gRPC, CLI exit codes) centralized?
- Is panic policy defined: which components may panic, and whether panics abort or are caught at task or request boundaries?

## Runtime And Threading Topology

- Is there exactly one async runtime, created at the binary boundary, with libraries remaining runtime-agnostic or explicitly tied to it?
- Is CPU-bound work separated from the I/O runtime (blocking pool, `rayon`, dedicated threads)?
- Are long-lived tasks owned by a supervisor that handles failures, restarts, and shutdown?
- Is shutdown coordinated across the HTTP server, workers, consumers, and telemetry?
- Are concurrency limits and queue bounds defined at system entry points?

## Shared State And Ownership

- Identify who owns each piece of mutable runtime state.
- Look for `Arc<Mutex<...>>` shared across many components as an implicit global, creating contention and unclear ownership.
- Consider whether actors (a task owning state and receiving messages) or partitioned state would simplify ownership. Do not introduce an actor framework where a simple lock suffices.
- Check that in-memory state is not the only copy of data that must survive restarts.
- In horizontally scaled deployments, check that in-memory caches, rate limiters, and locks are not assumed to be global.

## Modularity

Evaluate whether areas can evolve without unrelated areas changing together. Look for feature leakage across modules, shared internal models, global singletons (`static` with `OnceLock`), overly broad traits, central utility crates, cross-module database access, and configuration that implicitly joins separate components.

Prefer explicit contracts between modules. Avoid creating abstractions solely to claim modularity.

## Monoliths And Services

Do not assume services are better than a well-structured Rust monolith. Rust binaries are small, fast to start, and efficient, which weakens many scaling arguments for splitting.

For a monolith, review modularity, ownership, deployment simplicity, shared database coupling, and whether a modular monolith solves the actual pressure.

For multiple services, review service boundaries, independent deployment, data ownership, network failure handling, contract versioning (protobuf, OpenAPI, shared types crates), observability, and operational cost.

Flag a distributed monolith: services that must deploy together, share a database, or share a types crate that forces lockstep releases.

## Shared Contract Crates

- A shared crate of request and response types between services couples their release cycles. Check whether wire compatibility is maintained independently of Rust type changes.
- Prefer schema-first contracts (protobuf, OpenAPI, JSON Schema) for services owned by different teams or deployed independently.
- Review `serde` attributes on shared types for compatibility: unknown-field handling, defaults, and enum tagging.

## Data Ownership, Persistence, And Consistency

- Identify which component owns each important piece of mutable data. Flag multiple services writing the same tables.
- Review database boundaries, transaction boundaries, schema ownership, and migration ownership.
- Distinguish local transactions from cross-service consistency. Consider outbox, inbox, idempotency, and compensation only where failure modes require them.
- Make the consistency model explicit, and check that downstream systems can tolerate it.
- Use `rust-database` for deeper analysis.

## Messaging And Integrations

- For messaging, review event ownership, contracts, idempotent consumers, duplicate delivery, ordering assumptions, retries, dead-lettering, schema evolution, and transactional publishing.
- For synchronous integrations, review latency and availability coupling, timeouts, retries, cascading failures, and rate limits.
- Identify critical request chains where one dependency failure takes down several components.

## Resilience And Reliability

Review realistic failure modes: process crashes, panics in tasks, restarts, dependency failures, partial writes, repeated delivery, interrupted jobs, queue growth, and backpressure.

Each resilience mechanism (timeouts, retries, circuit breakers, load shedding, checkpoints) should address a concrete failure mode. For restart-sensitive work, check for durable checkpoints and idempotent processing.

## FFI, Plugins, And Extension Points

- Treat FFI boundaries as architectural boundaries with their own safety, ownership, threading, and error contracts. Check that they are isolated in dedicated crates with safe wrappers.
- Rust has no stable ABI. Dynamic loading of Rust `dylib` plugins across compiler versions is unsound unless using the C ABI, `abi_stable`, or WebAssembly. Flag plugin architectures that ignore this.
- Prefer compile-time extension (features, traits, generic composition) or out-of-process extension (WASM, subprocesses, RPC) over in-process dynamic Rust plugins.

## Feature Flag Architecture

- Check that Cargo features are additive and that the dependency graph's feature unification cannot enable incompatible combinations.
- Flag mutually exclusive features (for example, choosing between two TLS backends or runtimes) that break when two dependents enable different ones. Prefer separate crates, runtime configuration, or a documented default with an override.
- Check that CI covers the feature combinations the project claims to support.
- Do not use Cargo features for runtime or tenant configuration.

## Dependency And Supply-Chain Risk

- Review the size and health of the dependency tree (`cargo tree`, `cargo deny`, `cargo audit`, `cargo vet`).
- Identify critical dependencies with a single maintainer, low activity, heavy `unsafe`, or native build requirements.
- Check for duplicated major versions of large crates, which increase build time and binary size.
- Check for a policy on MSRV, toolchain pinning, and dependency updates.
- Check that build scripts and proc macros from third parties are understood, since they execute at build time.

## Build Time And Developer Feedback

- Treat build time as an architectural concern for large Rust codebases. Slow builds reduce iteration speed and increase CI cost.
- Look for crate structures that serialize compilation, such as one crate depended on by everything and changing frequently.
- Look for heavy generic APIs and proc macros in widely used crates that multiply codegen.
- Check for caching (`sccache`, CI caches) and faster linkers where builds are a bottleneck.
- Use `rust-performance` for detailed build-time analysis.

## Observability

Check whether failures can be diagnosed: structured `tracing` with spans across tasks, propagated request and trace IDs, meaningful metrics, health and readiness endpoints, and visibility into panics and task failures. Do not require every observability technology. Focus on what the deployment actually needs.

## Security Boundaries

Review trust boundaries, authentication and authorization placement, service identities, secret ownership, network assumptions, and data sensitivity. Check where untrusted input is parsed and whether parsing is isolated, bounded, and fuzzed. Do not assume internal traffic is trusted.

## Configuration

Check that configuration is loaded and validated once at startup into typed structures, that secrets are separated from normal configuration, and that configuration does not become an undocumented runtime dependency graph between components.

## Deployment Architecture

Where deployment files are visible, inspect:

- Binary and container build strategy: static linking (musl), distroless or scratch images, multi-stage builds, and cross-compilation targets.
- Allocator choice with musl, since its default allocator performs poorly under contention.
- Release tooling and reproducibility: pinned toolchain, `Cargo.lock` committed, and deterministic builds.
- Health checks, readiness and liveness, startup dependencies, and graceful shutdown on `SIGTERM`.
- Database migration execution during deployment.
- Stateful versus stateless processes and scaling assumptions.

Do not invent Kubernetes concerns for applications that are not deployed on Kubernetes. If topology is not visible, state that the review is limited.

## Libraries At Scale

For crates consumed by other teams or published publicly, review:

- Public API surface size and stability.
- Semver discipline and tooling (`cargo semver-checks`).
- `#[non_exhaustive]` use on growing types.
- Feature design and default features.
- MSRV policy.
- Documentation and examples.
- Whether dependency types are exposed in the public API.

## Abstractions And Patterns

Flag abstractions without value: traits with one implementation and no seam, generic repository traits that only wrap a database crate, builder types for simple structs, wrapper crates around single dependencies, macro-heavy DSLs for ordinary code, and deep indirection.

Treat patterns (hexagonal, clean, onion, actors, ECS, CQRS, event sourcing, DDD) as tools. Evaluate whether each solves a concrete problem. A pragmatic partial implementation may be correct.

Do not remove abstractions that provide real testability, substitutability, or boundary protection.

## Architectural Change Review

When reviewing a proposed change, ask:

1. What problem does this change solve?
2. Is the problem real today?
3. Is there a simpler solution?
4. What complexity, build cost, and operational burden does it add?
5. What coupling does it remove or introduce?
6. What becomes easier, and what becomes harder?
7. How reversible is the decision?

Prefer reversible decisions when uncertainty is high. Do not prescribe a rewrite, in Rust or out of it, unless the risk clearly justifies it.

## Common Architectural Anti-Patterns

Actively look for:

- A crate per layer in a small codebase.
- A `common` crate containing business logic.
- Generic parameters threaded through every layer.
- `Arc<Mutex<_>>` global state shared by everything.
- Multiple async runtimes in one process.
- CPU-heavy work on the I/O runtime.
- Untracked background tasks with no shutdown path.
- In-memory state for durable workflows.
- Mutually exclusive Cargo features.
- Dynamic Rust plugins over an unstable ABI.
- Shared types crates forcing lockstep service releases.
- Distributed monoliths and shared databases across services.
- Retries without idempotency.
- Architecture chosen for pattern purity.

These are not automatic findings. Verify the concrete consequence before reporting.

## Review Workflow

1. Understand the system purpose.
2. Map crates, binaries, and deployment units.
3. Map the internal and external dependency graph.
4. Identify data ownership and consistency boundaries.
5. Identify critical workflows and their runtime execution.
6. Identify external dependencies and failure boundaries.
7. Assess crate boundaries, coupling, and cohesion.
8. Assess runtime topology, shared state, and shutdown.
9. Assess error and panic strategy.
10. Assess features, dependencies, and build time.
11. Assess deployment, scalability, and reliability risks.
12. Inspect security boundaries and observability.
13. Evaluate tradeoffs.
14. Verify each finding against actual system needs.
15. Produce prioritized recommendations.

## Review Checklist

- Crate and module boundaries.
- Ownership.
- Dependency direction.
- Trait seams and generic propagation.
- Error and panic strategy.
- Runtime and threading topology.
- Shared state.
- Modularity.
- Service boundaries and shared contracts.
- Data ownership and consistency.
- Messaging and integrations.
- Resilience and idempotency.
- FFI and plugin boundaries.
- Feature design.
- Dependency and supply-chain risk.
- Build time.
- Observability.
- Security boundaries.
- Configuration.
- Deployment.
- Abstraction complexity.
- Reversibility of decisions.

Do not force a finding for every checklist item.

## Review Output

Organize findings by severity.

### Critical

Architectural issues likely to cause severe data loss, security failure, unsoundness, or systemic production failure.

### High

Major risks that significantly affect reliability, scalability, maintainability, or correctness.

### Medium

Meaningful structural problems with moderate impact.

### Low

Smaller but concrete architectural concerns.

### Suggestions

Optional improvements or future considerations.

For every finding:

1. Identify the relevant crate, component, or boundary.
2. Describe the architectural problem.
3. Explain the concrete consequence.
4. Explain the tradeoff.
5. Recommend a practical direction.

If the architecture is reasonable for the system's requirements, say so. Do not manufacture complexity to produce findings. Simple architecture can be good architecture.

## Style

- Focus on concrete architectural consequences.
- Explain tradeoffs.
- Avoid pattern dogma.
- Avoid style criticism.
- Avoid speculative scale requirements.
- Prefer incremental improvements.
- Distinguish confirmed issues from assumptions.
- Avoid proposing large rewrites without strong justification.

Architecture review should help teams make better decisions, not score them against an idealized diagram.
