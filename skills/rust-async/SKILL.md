---
name: rust-async
description: Design, implement, debug, and review asynchronous and concurrent Rust code, covering Tokio and other runtimes, futures, Send and Sync bounds, task spawning, blocking work, cancellation safety, select!, structured concurrency, shutdown, locks and shared state, channels, backpressure, timeouts, async traits, Pin, and threads. Use when writing async functions, spawning tasks, sharing state across tasks or threads, using channels or locks, handling shutdown, or diagnosing hangs, deadlocks, starvation, or Send errors.
license: MIT
---

# Rust Async

Use this skill when writing or reviewing asynchronous or concurrent Rust code.

Inspect the repository first. Identify the runtime (usually Tokio, sometimes async-std, smol, or embassy), its flavor and configuration, existing task-spawning and shutdown patterns, the channel and lock types in use, and whether `tokio-util`, `futures`, or `tower` are already dependencies. Follow those choices. Do not mix runtimes or introduce a second async ecosystem.

The examples below use Tokio because it is the most common runtime. Apply the same principles to other runtimes.

## Mental Model

- A future does nothing until polled. Calling an `async fn` without `.await` or spawning it does nothing.
- Every `.await` is a point where the task can be suspended and where it can be cancelled by being dropped.
- Async tasks are cooperatively scheduled. A task that never yields blocks its worker thread and every task queued on it.
- Async adds value for many concurrent I/O-bound operations. It does not make CPU-bound work faster.
- Decide whether async is warranted. A CLI tool or batch job that does a few sequential requests is often simpler with blocking code.

## Runtime

- Create the runtime once at the binary boundary (`#[tokio::main]` or an explicit `Builder`). Libraries must not create runtimes or assume one exists unless that is their documented contract.
- Choose the flavor deliberately. Use `multi_thread` for servers. Use `current_thread` for single-threaded or embedded use, tests, or when `!Send` futures are required.
- Never create or `block_on` a runtime from inside async code. It panics or deadlocks.
- Bridge sync code into async with `Handle::block_on` from a non-runtime thread, or `Runtime::block_on` at the top level.
- Libraries should be runtime-agnostic where practical. Gate runtime-specific code behind features if the crate supports several runtimes.

## Send, Sync, And 'static

- `tokio::spawn` requires the future to be `Send + 'static`. Everything held across an `.await` becomes part of the future's state and must be `Send`.
- Common causes of `!Send` futures: holding `std::sync::MutexGuard`, `Rc`, `RefCell` borrows, or raw pointers across `.await`.
- Fix `!Send` errors by narrowing scopes so the value is dropped before the `.await`, not by switching to `spawn_local` or adding `unsafe impl Send`.
- Satisfy `'static` by moving owned data or `Arc` clones into the task. Do not borrow from the spawning scope.
- Use `tokio::task::LocalSet` or `spawn_local` only when `!Send` data is a genuine requirement.

```rust
// Guard dropped before the await, so the future stays Send.
let snapshot = {
    let guard = state.lock().unwrap();
    guard.clone()
};
client.publish(snapshot).await?;
```

## Blocking And CPU-Bound Work

- Never block a runtime worker thread. Blocking calls include `std::fs`, `std::thread::sleep`, synchronous HTTP or database clients, heavy compression, hashing passwords, large JSON parsing, and long CPU loops.
- Move blocking calls to `tokio::task::spawn_blocking`. The blocking pool is large but bounded. Do not use it for unbounded fan-out.
- Use a dedicated thread pool such as `rayon` for parallel CPU-bound work. Bridge results back with a `oneshot` channel.
- Use `tokio::fs` for occasional file operations. For heavy file I/O, batch the work inside one `spawn_blocking` call.
- Insert `tokio::task::yield_now().await` in long-running async loops that do not otherwise await.
- Do not use `block_in_place` as a general fix. It only works on the multi-threaded runtime and still reduces worker capacity.

## Cancellation And Cancellation Safety

- Dropping a future cancels it at its current `.await` point. Code after that point never runs.
- Cancellation happens through `select!`, `timeout`, `JoinHandle::abort`, dropping a `JoinSet`, request disconnection in web frameworks, and shutdown.
- A future is cancel-safe if dropping it after any `.await` loses no data and leaves no broken state.
- Check the documentation of every future used in a `select!` loop. Generally cancel-safe: `mpsc::Receiver::recv`, `broadcast::Receiver::recv`, `TcpListener::accept`, `sleep`, and `CancellationToken::cancelled`. Not cancel-safe: `AsyncReadExt::read_exact`, `read_to_end`, `read_line`, `AsyncWriteExt::write_all`, and most multi-step operations.
- Do not perform partial state mutations across `.await` points when the future can be cancelled. Commit state only after the last await, or design an explicit recovery path.
- Use `tokio::pin!` or `std::pin::pin!` with `&mut fut` in `select!` loops to resume a non-cancel-safe operation instead of restarting it each iteration.
- Use `biased;` in `select!` when branch priority matters, such as checking shutdown first.
- Remember that `Drop` cannot be async. If cleanup requires I/O, design an explicit `shutdown().await` method and call it.

```rust
loop {
    tokio::select! {
        biased;
        _ = shutdown.cancelled() => break,
        msg = rx.recv() => match msg {
            Some(msg) => handle(msg).await?,
            None => break,
        },
    }
}
```

## Spawning And Structured Concurrency

- Do not fire and forget. Keep the `JoinHandle`, use a `JoinSet`, or use `tokio_util::task::TaskTracker` so failures and panics are observed and shutdown can wait.
- Dropping a `JoinHandle` detaches the task. It does not cancel it. Dropping a `JoinSet` aborts all of its tasks.
- A panicking task returns `JoinError` from its handle. Check `is_panic()` and decide whether to propagate, restart, or abort the process.
- Prefer concurrency without spawning when tasks do not need to run independently. Use `join!`, `try_join!`, `FuturesUnordered`, or `StreamExt::buffer_unordered` within the same task.
- Bound concurrency explicitly with a `Semaphore`, `buffer_unordered(n)`, or a fixed worker pool. Unbounded `spawn` in a loop over input is a resource-exhaustion bug.
- Propagate tracing spans into spawned tasks with `.instrument(span)` so logs keep their context.

## Shutdown

- Handle `SIGTERM` and Ctrl-C (`tokio::signal`) at the binary boundary and turn them into a single cancellation signal, usually `CancellationToken` or a `watch` channel.
- Stop accepting new work first, then let in-flight work drain with a deadline, then force-cancel what remains.
- Make background loops observe the shutdown signal at every iteration.
- Flush telemetry, close connections, and commit checkpoints before exit.
- Know that returning from `main` drops the runtime and cancels all remaining tasks at their next `.await`.

## Shared State And Locks

- Prefer message passing or owned state per task when the design allows. Share state only when needed.
- Use `std::sync::Mutex` (or `parking_lot`) for short critical sections that do not contain `.await`. It is faster than an async mutex.
- Use `tokio::sync::Mutex` only when the guard must be held across `.await`, such as serializing access to an I/O resource. Consider an actor task instead.
- Never hold a lock guard across `.await` with a synchronous mutex. It makes the future `!Send` and can deadlock under contention.
- Be careful with concurrent maps such as `DashMap`. Holding a reference or entry across `.await`, or locking two keys in one task, can deadlock.
- Establish a consistent lock order when multiple locks are acquired.
- Use `RwLock` only when reads vastly outnumber writes and critical sections are non-trivial. Otherwise a `Mutex` is simpler and often faster.
- Use atomics for counters and flags. Use `Ordering::SeqCst` unless you can explain why a weaker ordering is correct. `Relaxed` is correct for independent counters, not for publishing data.
- Use `arc-swap` or a `watch` channel for read-mostly configuration that is replaced wholesale.
- Do not use lock poisoning as a recovery mechanism. Decide whether to propagate the panic or reset state, and do it consistently.

## Channels And Backpressure

- Prefer bounded channels. Unbounded channels hide overload until memory runs out.
- Choose the channel by semantics: `mpsc` for work queues, `oneshot` for a single reply, `broadcast` for fan-out where every subscriber sees every message, and `watch` for latest-value state.
- Handle `broadcast::error::RecvError::Lagged`. Slow receivers lose messages.
- Treat a closed channel (`recv` returning `None`, `send` returning `Err`) as a normal shutdown signal, not a panic.
- Decide what happens when a bounded channel is full: await (backpressure), `try_send` and drop or reject (load shedding), or reply with a busy error. Make the choice explicit.
- Use the actor pattern (a task owning state, receiving commands with `oneshot` reply channels) for complex shared mutable state.

## Timeouts And Retries

- Put a timeout on every network call and every wait on external input. Use `tokio::time::timeout` or the client's own timeout settings.
- Apply timeouts at the right granularity: per attempt and for the overall operation.
- Retry only idempotent operations, with exponential backoff, jitter, and a maximum attempt count or deadline.
- Remember that a timeout cancels the inner future. The cancellation-safety rules apply.

## Async Traits And Pin

- Native `async fn` in traits is stable but not object-safe, and callers cannot require the returned future to be `Send` without extra design. Use `trait_variant`, explicit `-> impl Future<Output = T> + Send` signatures, or the `async-trait` crate when `dyn` dispatch is required. Follow the repository's existing choice.
- Use `Pin<Box<dyn Future<Output = T> + Send + '_>>` (`BoxFuture`) for dynamic dispatch and recursion.
- Do not implement `Future` or `Stream` manually unless combinators and `async` blocks cannot express the behavior. Manual poll implementations are subtle and need careful waker handling.
- Use `async-stream` or `futures::stream::unfold` to build streams instead of a manual `Stream` implementation.

## Threads Without Async

- Use `std::thread::scope` for short-lived parallelism that borrows from the caller.
- Use `rayon` for data parallelism over collections.
- Join every thread you spawn, or document why it is a daemon thread.
- Use `std::sync::mpsc` or `crossbeam-channel` for thread communication when no async runtime is involved.

## Diagnosing Problems

- **Hang or no progress:** look for a blocked worker thread (blocking call in async), a lock held across `.await`, a channel with no sender closing, or a `select!` loop that never yields.
- **Deadlock:** look for inconsistent lock order, re-entrant locking of a non-reentrant mutex, or concurrent map entries held across `.await`.
- **Lost work:** look for cancelled non-cancel-safe futures, dropped `JoinHandle`s, ignored `JoinError`s, or lagged broadcast receivers.
- **Latency spikes:** look for CPU-heavy work on worker threads, unbounded concurrency, or lock contention.
- Use `tokio-console` to inspect task states and poll times, and `tracing` spans to correlate.

## Implementation Workflow

- Identify the runtime, existing spawn, shutdown, lock, and channel patterns.
- Decide which work is I/O-bound, CPU-bound, and blocking.
- Design ownership of state per task before writing code.
- Check every `.await` for cancellation impact and every held value for `Send`.
- Bound all concurrency and queues.
- Wire the new tasks into existing shutdown handling.
- Test with `#[tokio::test]`, controlled time, and concurrent scenarios. Use `rust-testing` for details.

## Checklist

- Is async justified for this code?
- Is there any blocking or CPU-heavy work on runtime worker threads?
- Are lock guards dropped before every `.await`?
- Are futures used in `select!` and `timeout` cancel-safe, or protected with pinning?
- Can cancellation leave partially mutated state?
- Are spawned tasks tracked, joined, and included in shutdown?
- Are task panics observed?
- Are channels bounded, with explicit full-queue behavior?
- Is concurrency bounded?
- Do network calls and external waits have timeouts?
- Are retries limited to idempotent operations with backoff?
- Are atomics using orderings that can be justified?
- Do spawned tasks carry tracing context?
