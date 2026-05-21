---
name:        profile-hotspot
description: Find and confirm a performance hotspot by measurement — define target, capture a profile, read the flamegraph, verify the fix.
version:     1
trigger_keywords: [profiling, performance, slow, latency, flamegraph, perf, hotspot, optimization]
tools:       [bash]
requires:    [PROFILING]
source:      project_shared
created_at:  2026-05-21
updated_at:  2026-05-21
expires:     null
---

## When to use

Goal-shape: "this is slow, find out why", "profile this service", "where is the time going". Use when you need to **locate** a bottleneck before changing code.

Not a use case: optimizing on a hunch with no measurement (that's premature optimization — this skill exists to prevent it); a slowness already known to be in the database (use `pg-heavy-queries`) or the network (use `diagnose-network`).

## Prerequisites

- A reproducible problematic scenario — a load test, a request, or a representative workload.
- A profiler matched to the suspected resource and runtime (see step 3).
- A baseline measurement to compare against.

## Procedure

1. **Define the goal.** Which metric (p99 latency, throughput, memory, cloud cost) and what target — "<200ms" is measurable, "fast" is not. Without this, profiling is tourism.
2. **Reproduce reliably.** Get the problematic scenario to happen on demand; intermittent-only bugs can't be profiled cleanly.
3. **Capture a profile** with the right tool for the suspected resource:
   - **CPU** (sampling, low overhead, prod-safe): `perf record -F 99 -p <pid> -g -- sleep 30`; `py-spy record --pid <pid>`; `go test -cpuprofile`; async-profiler `-e cpu`.
   - **Allocation/heap**: `tracemalloc`/`memray` (Python), `-memprofile` (Go), `async-profiler -e alloc` (JVM), `heaptrack` (Rust). Distinguish *total allocation* (GC pressure) from *retention* (leaks/footprint).
   - **Off-CPU / waiting**: `async-profiler -e wall`, `perf sched`, bcc `offcputime` — for time spent in I/O, locks, GC.
   - **Syscalls/I/O**: `strace -c`, `iostat`, eBPF (`biosnoop`, `tcptop`).
   Prefer sampling over instrumenting profilers under real load — instrumenting distorts the run.
4. **Generate a flamegraph** and find the hotspot: look for **wide plateaus at the top** (leaves where time is actually spent). Tall stacks without a wide top are deep-but-fast calls, not the hotspot.
5. **Form one hypothesis** about the cause and make **one small change**.
6. **Re-measure** against the baseline; a differential flamegraph makes the delta obvious.
7. **Keep or revert.** Gain confirmed → keep. No gain → revert and try another angle. "Looks better" is not a measurement.

## Verification

- The post-change measurement beats the baseline on the metric from step 1, in a prod-like environment.
- The targeted function shrank or left the top of the flamegraph.
- Remember Amdahl: a function that was 20% of runtime, fully eliminated, yields ~20% — confirm the gain is worth it.

## Anti-cases

- Profiling local code when the bottleneck is the DB or network → look end-to-end first.
- "100% CPU" assumed to be the bottleneck → could be a spinning thread; cross-check with an off-CPU profile.
- Micro-benchmark gain that vanishes end-to-end → cache/branch-prediction/JIT effects; trust the end-to-end number.
- Measuring JVM/V8 before warmup → early samples are pessimistic; warm up first.
