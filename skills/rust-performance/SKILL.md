---
name: rust-performance
description: Measure, diagnose, and improve the runtime performance, memory use, compile time, and binary size of Rust code, covering benchmarking, CPU and heap profiling, allocation discipline, data layout, collections and hashing, strings, iterators and bounds checks, inlining, I/O buffering, serialization, async overhead, lock contention and false sharing, release profiles, LTO, PGO, allocators, SIMD, and build-time costs of generics and dependencies. Use when optimizing hot paths, investigating latency, throughput, memory growth, slow builds, or large binaries, or reviewing performance-sensitive Rust changes.
license: MIT
---

# Rust Performance

Use this skill when performance, memory use, compile time, or binary size is the goal, or when reviewing code on a known hot path.

Inspect the repository first. Check `[profile.*]` settings in `Cargo.toml`, `.cargo/config.toml` (linker, `rustflags`, `target-cpu`), existing benchmarks, the global allocator, profiling and tracing setup, and any documented performance budgets. Follow existing tooling. Do not change global build settings as part of a local optimization without stating the tradeoff.

## Principles

- Measure first. Intuition about Rust performance is often wrong because the optimizer is aggressive.
- Define the target before optimizing: latency percentile, throughput, peak memory, startup time, build time, or binary size.
- Optimize the bottleneck the profiler shows, not the code that looks slow.
- Prefer algorithmic and data-structure improvements over micro-optimizations.
- Keep the optimized code readable, tested, and benchmarked so the gain survives future changes.
- Stop when the target is met. Every optimization is a maintenance cost.

## Measurement

- Always measure release builds. Debug builds are often 10–100 times slower and are meaningless for performance work.
- Use `criterion` or `divan` for microbenchmarks. Use `std::hint::black_box` on inputs and outputs so work is not optimized away.
- Benchmark realistic input sizes and distributions, not only tiny or ideal cases.
- Compare against a saved baseline, run several times, and watch for noise from CPU frequency scaling, thermal throttling, and other processes.
- Use load tests with realistic concurrency for services. Report percentiles (p50, p99, p99.9), not just averages.
- Keep `debug = "line-tables-only"` or `debug = true` in a profiling profile so profiles have symbols.

## Profiling

- **CPU:** `cargo flamegraph`, `perf record` plus `perf report`, `samply`, Instruments on macOS, VTune, or Superluminal. Look for wide frames, not tall ones.
- **Heap:** `dhat` (`dhat-rs`), `heaptrack`, or `bytehound` to find allocation counts, peak usage, and growth.
- **Async:** `tokio-console` for task poll times and scheduling delays. Use `tracing` spans with timing for request-level breakdowns.
- **Compile time:** `cargo build --timings` for crate-level timings, and `cargo llvm-lines` for monomorphization bloat.
- **Binary size:** `cargo bloat` to see which crates and functions take space.
- Profile with production-like data and configuration.

## Allocation Discipline

- Allocation is often the dominant cost in hot paths. Count allocations before and after a change.
- Preallocate with `Vec::with_capacity`, `String::with_capacity`, and `HashMap::with_capacity` when the size is known or estimable.
- Reuse buffers across iterations (`buf.clear()` keeps capacity) instead of allocating new ones.
- Borrow instead of owning: `&str` over `String`, slices over `Vec`, and `Cow` when ownership is conditional.
- Avoid `format!`, `to_string()`, and `clone()` in tight loops. Write into a reused buffer with `write!`.
- Use `SmallVec`, `ArrayVec`, or `tinyvec` for collections that are usually small, only when profiling shows allocation cost.
- Use `Box<[T]>` or `Arc<str>` for immutable data that does not need growth capacity.
- Use arenas (`bumpalo`, `typed-arena`) for many short-lived objects with the same lifetime.

## Data Layout

- Prefer contiguous data (`Vec`, slices) over pointer-heavy structures (`LinkedList`, `BTreeMap` of boxes, trees of `Rc`). Cache locality dominates.
- Consider struct-of-arrays for hot loops that touch only some fields of many records.
- Keep enums small. Box large, rarely used variants (`clippy::large_enum_variant`).
- Check type sizes with `std::mem::size_of` when types are stored in bulk. Use niche-optimized types, such as `Option<NonZeroU32>` and `Option<Box<T>>`, that cost no extra space.
- Use indices into a `Vec` instead of references or `Rc` for graph-like data.

## Collections And Hashing

- The default `HashMap` hasher (SipHash) resists HashDoS but is slower. Use a faster hasher (`ahash`, `rustc-hash`/`FxHash`, `foldhash`) only for keys that attackers cannot control, or when the faster hasher is DoS-resistant.
- Use `BTreeMap` when ordered iteration or range queries are needed, not as a default.
- Use the `entry` API to avoid double lookups.
- Use `Vec` with binary search, or a sorted `Vec`, for small, read-mostly maps.
- Use `IndexMap` when insertion order matters.

## Strings And Text

- Avoid repeated `String` concatenation in loops. Build with one buffer.
- Use byte-level processing (`as_bytes`, `memchr`) for ASCII-heavy parsing when UTF-8 semantics are not required.
- Use `compact_str`, `smol_str`, or interning for many small, repeated strings, only when measured.
- Avoid UTF-8 validation more than once. Keep validated data as `&str` or `String`.

## Iterators, Bounds Checks, And Loops

- Iterator chains usually compile to code as fast as hand-written loops, and often faster because they avoid bounds checks.
- When indexing in hot loops, help the optimizer remove bounds checks: iterate with `zip`, slice to a known length first (`let a = &a[..n];`), or use `chunks_exact`.
- Avoid `collect()` into intermediate collections that are only iterated again.
- Do not use `unsafe { get_unchecked }` to remove bounds checks unless a profile shows they matter and the invariant is proven. Use `rust-unsafe-ffi`.

## Inlining And Code Generation

- Generic functions and functions in the same crate are inlinable already.
- Add `#[inline]` to small, non-generic public functions called across crate boundaries in hot paths when LTO is not enabled.
- Avoid `#[inline(always)]` except where measurement proves it helps. It can increase code size and hurt instruction cache performance.
- Use `#[cold]` and `#[inline(never)]` on error and slow paths to keep hot code compact.
- Generics are monomorphized per type, which is fast at runtime but costs compile time and binary size. Use an inner non-generic function or `dyn` when a generic function body is large and performance-insensitive.

```rust
pub fn load<P: AsRef<Path>>(path: P) -> io::Result<Config> {
    fn inner(path: &Path) -> io::Result<Config> {
        // large body compiled once
    }
    inner(path.as_ref())
}
```

## I/O

- Wrap files and sockets in `BufReader` or `BufWriter`. Unbuffered small reads and writes are syscall-bound.
- Lock `stdout` once and wrap it in `BufWriter` for heavy output. `println!` locks on every call, and stdout is line-buffered.
- Flush `BufWriter` explicitly and handle the error. Errors on drop are silently ignored.
- Use memory-mapped files (`memmap2`) only for large, random-access, read-mostly files, and understand the safety caveats when files change underneath.
- Batch network writes and use vectored I/O where the protocol allows.

## Serialization

- Use zero-copy deserialization (`#[serde(borrow)]` with `&str` or `Cow<str>`) for large inputs that are processed immediately.
- Deserialize into typed structs, not `serde_json::Value`, on hot paths.
- Consider binary formats (`bincode`, `postcard`, `rkyv`, protobuf) for internal high-volume data, weighing the schema-evolution cost.
- Stream large payloads instead of buffering whole documents.

## Async Overhead

- Do not spawn a task per tiny unit of work. Spawning has a cost. Use `join!` or `buffer_unordered` within a task.
- Watch future sizes. Large futures (big arrays or deeply nested state machines) are expensive to move. Box them (`Box::pin`) where `clippy::large_futures` flags them.
- Keep CPU-heavy work off runtime threads. It inflates latency for all tasks.
- Use `rust-async` for runtime and scheduling guidance.

## Concurrency And Contention

- Profile lock contention before redesigning. A `Mutex` with short critical sections is often fine.
- Reduce contention by sharding state, using per-thread or per-core accumulators, or replacing shared state with message passing.
- Avoid cloning and dropping `Arc` in hot loops across threads. The atomic reference count causes cache-line contention. Pass `&T` where the lifetime allows.
- Prevent false sharing of frequently written per-thread data by padding to cache lines (`crossbeam_utils::CachePadded`).
- Use `rayon` for data parallelism. Check that the work per item outweighs the scheduling overhead.

## Release Profile

- Tune `[profile.release]` deliberately and document why:
  - `lto = "thin"` or `"fat"` enables cross-crate optimization at the cost of build time.
  - `codegen-units = 1` improves optimization at the cost of build parallelism.
  - `panic = "abort"` reduces size and slightly improves speed, but disables unwinding. `catch_unwind` stops working, and panics in threads abort the process.
  - `opt-level = 3` is the default. `"s"` or `"z"` optimizes for size.
  - `strip = true` removes symbols from shipped binaries.
- Consider a separate `[profile.profiling]` that inherits from release with debug info.
- Use `-C target-cpu=native` only for binaries that run on the build machine's CPU class. It produces non-portable binaries.
- Consider profile-guided optimization (`cargo pgo`) and BOLT for CPU-bound services where a few percent matters and the build pipeline can support it.

## Allocators

- Consider `mimalloc` or `jemalloc` (`tikv-jemallocator`) for allocation-heavy, multi-threaded services, especially with musl, whose default allocator is slow.
- Measure throughput, latency, and memory footprint before and after. Allocator changes have workload-dependent results.

## SIMD

- Rely on auto-vectorization first. Write simple loops over slices and check the assembly (`cargo asm`, Compiler Explorer).
- Use crates such as `wide` or established SIMD-accelerated libraries (`memchr`, `simdutf8`) before writing intrinsics.
- If writing `std::arch` intrinsics, use runtime feature detection (`is_x86_feature_detected!`) with `#[target_feature]` functions, and provide a scalar fallback. Follow `rust-unsafe-ffi` for the safety requirements.

## Compile Time

- Measure with `cargo build --timings` before restructuring.
- Split large crates so codegen can parallelize, but only along meaningful boundaries.
- Reduce heavy generic instantiation with the inner-function pattern or `dyn`.
- Trim dependency features (`default-features = false`) and remove unused dependencies (`cargo machete`, `cargo udeps`).
- Be wary of heavy procedural macros in widely used types.
- Use a faster linker (`mold`, `lld`) and a build cache (`sccache`) in development and CI where supported.

## Binary Size

- Use `cargo bloat --release --crates` to find what contributes.
- Apply `opt-level = "z"`, `lto = true`, `codegen-units = 1`, `panic = "abort"`, and `strip = true` when size is the goal, accepting the tradeoffs.
- Avoid pulling in large dependencies for small features.

## Common Anti-Patterns

- Optimizing without a profile.
- Benchmarking debug builds.
- `clone()` to satisfy the borrow checker in hot loops.
- Collecting into `Vec` between iterator stages.
- Unbuffered I/O.
- `serde_json::Value` on hot paths.
- Fast non-DoS-resistant hashers on attacker-controlled keys.
- `#[inline(always)]` everywhere.
- `unsafe` to skip bounds checks without measured benefit.
- Global build-setting changes without documented tradeoffs.

## Optimization Workflow

1. Define the performance target and how it is measured.
2. Reproduce the problem with realistic data on a release build.
3. Profile to find the bottleneck.
4. Form a hypothesis and make one change at a time.
5. Measure against the baseline.
6. Keep the change only if it meets the target or gives a meaningful gain.
7. Add or update a benchmark that protects the gain.
8. Run the full test suite. Optimizations must not change behavior.

## Review Checklist

- Is there evidence (profile or benchmark) that this code is a bottleneck?
- Was the improvement measured on a release build with realistic input?
- Are allocations in hot paths minimized and buffers reused?
- Are data structures and hashers appropriate for the access pattern and threat model?
- Are iterator chains free of needless intermediate collections?
- Is I/O buffered and flushed with error handling?
- Is CPU-heavy work kept off async runtime threads?
- Is lock contention or `Arc` churn avoided on hot paths?
- Are any `unsafe` optimizations justified by measurement and proven sound?
- Are profile, allocator, and target-CPU changes deliberate and documented?
- Is there a benchmark protecting the optimization?
- Does the optimized code remain readable and tested?
