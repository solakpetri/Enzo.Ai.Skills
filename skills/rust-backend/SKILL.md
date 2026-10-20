---
name: rust-backend
description: Design, implement, refactor, and review production Rust backend services, covering HTTP APIs with Axum, Actix Web, or similar frameworks, tower middleware, application state, request validation, error-to-response mapping, configuration and secrets, structured logging and tracing, outbound HTTP clients, resilience, authentication and authorization, serialization, graceful shutdown, health checks, and service architecture. Use when creating or modifying endpoints, handlers, middleware, background workers, service wiring, or integrations in a Rust service.
license: MIT
---

# Rust Backend

Use this skill when working on production Rust backend services.

Inspect the repository before changing code. Identify the web framework and version, the runtime, how application state is built and shared, the error type used by handlers, the middleware stack, configuration loading, the tracing setup, persistence crates, and the test layout. Follow existing conventions. Do not introduce a second web framework, a new error-handling crate, a dependency-injection container, or a layered architecture the repository does not use, unless the change clearly requires it.

Examples use Axum and Tokio because they are the most common choice. Apply the same principles to Actix Web, Poem, Rocket, or tonic for gRPC.

Use `rust-core` for language-level guidance, `rust-async` for concurrency and cancellation, `rust-database` for persistence, and `rust-testing` for test strategy.

## General Principles

- Keep business logic separate from transport and infrastructure.
- Keep handlers thin. Extract and validate input, call application logic, and map the result to a response.
- Put domain rules in plain Rust types and functions that can be tested without HTTP or a database.
- Optimize based on evidence.
- Preserve working behavior when modifying an existing service unless the task explicitly changes it.

## Service Structure

- Organize code by feature or domain area. Each area owns its routes, handlers, application logic, and persistence access.
- Build the router in one place and compose feature routers with `Router::nest` or `merge`.
- Keep `main.rs` small. It loads configuration, initializes tracing, builds state, builds the router, binds the listener, and runs with graceful shutdown.
- Do not add a trait per service just to inject dependencies. Pass concrete types through state. Introduce traits at real seams, such as external systems that tests must replace.
- Avoid a global `lazy_static` or `OnceLock` for application dependencies. Pass them explicitly through state so tests can construct isolated instances.

## Application State

- Define one `AppState` struct that is cheap to clone. Hold shared resources as `Arc<T>` or types that are internally reference-counted, such as `sqlx::PgPool` and `reqwest::Client`.
- Use `State<AppState>` or substates via `FromRef` instead of `Extension` for application dependencies. `State` is checked at compile time.
- Keep mutable shared state out of `AppState` where possible. If it is needed, wrap it in the narrowest lock and follow `rust-async` rules about locks and `.await`.
- Do not put per-request data in shared state. Use extractors or request extensions.

```rust
#[derive(Clone)]
pub struct AppState {
    pub db: PgPool,
    pub http: reqwest::Client,
    pub config: Arc<AppConfig>,
}
```

## Handlers And Extractors

- Use typed extractors (`Path`, `Query`, `Json`, `State`, custom extractors) instead of parsing raw requests.
- Put the body-consuming extractor last in the handler arguments, as the framework requires.
- Write custom extractors (`FromRequestParts`) for cross-cutting request data such as the authenticated user, tenant, or request ID.
- Reject malformed input with consistent error responses. Customize the framework's default rejection responses if they do not match the API's error contract.
- Return a typed result such as `Result<Json<T>, ApiError>` instead of building responses manually.
- Be explicit about status codes: `201 Created` with a `Location` header for creates, `204 No Content` for empty successes, `404` versus `403` chosen deliberately for missing versus forbidden resources.

## Errors And Responses

- Define an application error type that implements `IntoResponse` (Axum) or `ResponseError` (Actix). Map domain errors to status codes in one place.
- Use a consistent error body. Prefer RFC 9457 problem details (`application/problem+json`) or the repository's established equivalent.
- Never return internal error messages, SQL errors, stack traces, or file paths to clients. Log the details with a request ID and return a generic message for `5xx` responses.
- Distinguish validation errors (`400` or `422`), authentication failures (`401`), authorization failures (`403`), missing resources (`404`), conflicts (`409`), and rate limits (`429`).
- Convert lower-level errors with `From` implementations so handlers can use `?`.
- Add `tower_http::catch_panic::CatchPanicLayer` or the framework equivalent so a panicking handler returns `500` instead of dropping the connection. Still treat panics as bugs.

```rust
pub enum ApiError {
    NotFound,
    Conflict(&'static str),
    Validation(Vec<FieldError>),
    Internal(anyhow::Error),
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        match self {
            ApiError::Internal(err) => {
                tracing::error!(error = ?err, "request failed");
                problem(StatusCode::INTERNAL_SERVER_ERROR, "internal error").into_response()
            }
            ApiError::NotFound => problem(StatusCode::NOT_FOUND, "not found").into_response(),
            // ...
        }
    }
}
```

## Validation

- Validate at the boundary before executing business logic. Use the repository's validation approach (`validator`, `garde`, or hand-written checks).
- Prefer parsing into domain types with private fields and validating constructors. A handler that receives `CreateOrder` with an `OrderQuantity` newtype cannot forget to validate.
- Also enforce invariants in domain or application logic, so they hold for background jobs, CLI commands, and message consumers, not only HTTP.
- Limit request body size. Configure `DefaultBodyLimit` or `RequestBodyLimitLayer` deliberately for endpoints that accept uploads.
- Bound collection sizes, string lengths, and pagination limits in request types.

## Serialization

- Keep API request and response types separate from persistence types and domain types when their lifecycles differ.
- Use `#[serde(rename_all = "camelCase")]` or the repository's naming convention consistently.
- Consider `#[serde(deny_unknown_fields)]` for configuration files and strict internal APIs. Avoid it on public request types that clients may extend, and know it does not work together with `#[serde(flatten)]`.
- Make optionality explicit. Distinguish missing fields from `null` when the API semantics require it, as in partial updates.
- Treat serialized types as contracts. Renaming a field or enum variant is a breaking change for clients and stored data.
- Serialize timestamps in an unambiguous format (RFC 3339 in UTC) and money as integer minor units or decimal strings, never floats.

## Middleware

- Use `tower` and `tower-http` layers for cross-cutting concerns: tracing (`TraceLayer`), request IDs (`SetRequestIdLayer`, `PropagateRequestIdLayer`), timeouts, body limits, compression, CORS, and panic catching.
- Understand layer ordering. Layers added later wrap earlier ones. With `ServiceBuilder`, layers run top to bottom on the request.
- Apply a request timeout to every route group. Long-running operations should become background jobs, not long requests.
- Use concurrency limits and load shedding (`ConcurrencyLimitLayer`, `LoadShedLayer`) for services that must degrade gracefully under overload.
- Configure CORS explicitly. Never combine wildcard origins with credentials.

## Configuration And Secrets

- Load configuration once at startup into a typed struct, using the repository's approach (`config`, `figment`, `envy`, or `std::env`).
- Fail fast with a clear error on missing or invalid required configuration.
- Wrap secrets in `secrecy::SecretString` or an equivalent so they are not printed by `Debug` or logged accidentally.
- Never commit secrets. Never log secrets, tokens, or full connection strings.
- Keep environment-specific values out of code.

## Logging, Tracing, And Metrics

- Use `tracing` with structured fields (`tracing::info!(order_id = %id, "order created")`), not string interpolation.
- Use `#[tracing::instrument(skip(state, body), fields(user_id = %user.id))]` on important handlers and functions. Skip large or sensitive arguments explicitly.
- Initialize the subscriber once in `main`. Use JSON output in production if the platform expects it.
- Propagate request IDs and trace context. Export to OpenTelemetry if the repository already does.
- Log errors once, at the boundary that handles them.
- Avoid logging personal data, secrets, and full request bodies.
- Expose metrics (request rate, latency, errors, saturation) through the repository's existing approach.

## Outbound HTTP And External APIs

- Build one `reqwest::Client` (or equivalent) at startup and share it. It pools connections internally and is cheap to clone.
- Set connect timeouts and request timeouts explicitly. The default has no overall timeout.
- Check status codes (`error_for_status()` or explicit matching) before deserializing.
- Treat remote responses as untrusted input. Validate them and handle missing or malformed fields.
- Wrap each external API in a small client type that owns URL construction, authentication, error mapping, and response types. Keep it out of handlers.
- Never build URLs to internal resources from user-controlled input without an allowlist (SSRF).

## Resilience

- Retry only operations that are safe to retry, with exponential backoff, jitter, and limits.
- Use idempotency keys for operations that clients or upstream systems may retry, such as payments and order creation.
- Distinguish transient failures (timeouts, `502`, `503`, connection resets) from permanent ones (`4xx` validation, auth).
- Use circuit breaking or load shedding for dependencies that fail slowly. Do not add these indiscriminately.
- Make background workers idempotent, and design them to resume after a restart.

## Authentication And Authorization

- Distinguish authentication from authorization.
- Validate tokens completely: signature, algorithm allowlist, issuer, audience, expiry, and not-before. Do not accept `alg: none` or let the token choose the algorithm.
- Implement the authenticated principal as an extractor so handlers cannot forget to require it.
- Enforce resource-level authorization in application logic: verify that the caller may access the specific record, not just the endpoint.
- Hash passwords with `argon2` or the repository's established algorithm, inside `spawn_blocking`.
- Compare secrets and tokens in constant time (`subtle` or `constant_time_eq`).

## Security

- Validate all external input, including headers, path segments, and query parameters.
- Use parameterized queries only. Never build SQL with `format!`.
- Guard file system access against path traversal. Canonicalize and check against an allowed root.
- Set security headers where the service serves browsers.
- Keep dependencies audited (`cargo audit`, `cargo deny`).
- Apply least privilege to database users, service accounts, and API keys.

## Graceful Shutdown And Health

- Run the server with graceful shutdown (`axum::serve(...).with_graceful_shutdown(signal)`), triggered by `SIGTERM` and Ctrl-C.
- Share one cancellation signal with background workers and wait for them to finish with a deadline.
- Provide separate liveness and readiness endpoints when the platform uses them. Readiness should reflect critical dependencies. Liveness should not fail just because a dependency is down.
- Fail readiness during shutdown so load balancers drain traffic before the process exits.

## Background Workers

- Run workers as tracked tasks that observe the shutdown signal.
- Persist work state for anything that must survive a restart. Do not keep durable work only in memory or in channels.
- Make processing idempotent and safe under concurrent or repeated execution.
- Bound concurrency and handle poison messages with retry limits and dead-lettering.

## Performance

- Measure first when performance is the goal. Use `rust-performance` for profiling guidance.
- Avoid obvious waste in hot paths: repeated queries, over-fetching, needless cloning of large bodies, and blocking calls on runtime threads.
- Stream large responses instead of buffering them fully in memory.
- Do not trade maintainability for speculative micro-optimizations.

## Refactoring Existing Services

- Trace the request flow end to end, from router to middleware to handler to persistence, before changing it.
- Match existing patterns unless they block the change or are unsafe.
- Make the smallest coherent change. Separate cleanup from functional changes.
- Keep API contracts stable unless the task explicitly changes them.

## Implementation Workflow

- Inspect the framework, state, error type, middleware, configuration, tracing, and test layout.
- Identify the real execution path before editing.
- Define request and response types, validation, and error mapping first.
- Implement application logic separately from the handler.
- Add or update tests: unit tests for logic, and HTTP-level tests through the router (`tower::ServiceExt::oneshot` or the framework's test client).
- Run `cargo check`, `cargo clippy --all-targets`, and the relevant tests.
- Review error mapping, auth, timeouts, logging, cancellation, and shutdown impact before finishing.

## Checklist

- Does the change follow the repository's framework, state, error, and module conventions?
- Are handlers thin, with business logic outside transport code?
- Are inputs validated at the boundary and invariants enforced in domain logic?
- Are errors mapped consistently without leaking internals?
- Are body sizes, collection sizes, and pagination bounded?
- Are serialized contracts stable and explicit?
- Is authentication enforced through extractors, with resource-level authorization in logic?
- Do outbound calls have timeouts, status checks, and safe retries?
- Is idempotency handled for retried operations?
- Are secrets protected from logs and `Debug` output?
- Is tracing structured, with request IDs and no sensitive data?
- Does new background work participate in graceful shutdown?
- Is blocking work kept off runtime worker threads?
- Do `cargo check`, `cargo clippy`, and the relevant tests pass?
