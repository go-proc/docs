# Design — reap, supervise, respawn

go-proc splits process lifecycle into three concerns that are usually tangled
together in an init or an agent. Keeping them separate is what makes each piece
small, testable, and reusable.

## 1. Reaping is unconditional

When a process runs as **PID 1** (a container/microVM init, or any process the
kernel has designated a *subreaper*), the kernel reparents every orphan to it.
Those orphans must be `wait()`ed or they become permanent zombies that leak
kernel process-table slots.

Crucially, PID 1 receives orphans it **never started** — a container runtime
forks intermediate helpers, a shell backgrounds a job, a library spawns a worker.
So the reaper cannot be "the thing that watches my children"; it must
unconditionally `Wait4(-1, …)` every reapable child on every `SIGCHLD`.

That is why `supervisor.Reaper` is independent of the supervisor: it collects
*all* zombies and publishes a `Reaped` event per collection. Consumers correlate
the ones they care about and ignore the rest — the zombie is already gone either
way.

## 2. Supervising is selective

The `Supervisor` is the opposite: it only tracks the PIDs it launched through the
`Runtime`. It indexes each started process by its runtime-reported pid, and when
a `Reaped` event arrives for a pid it owns, it applies that process's
`RestartPolicy`. Events for unknown pids are dropped.

This separation means the supervisor logic is **completely platform-independent**
— it never touches a syscall. Only the reaper is Linux-specific, and it sits
behind a build tag:

```go
//go:build linux    // reaper.go:  SIGCHLD + Wait4
//go:build !linux   // reaper_stub.go:  Run blocks until ctx is cancelled
```

So `go build ./...`, `go vet ./...`, and the portable tests stay green on macOS
and Windows, while the real subreaper is compiled and exercised on Linux (where
100% coverage is measured).

## 3. Respawning is a decision, not a reflex

"Process exited → start it again" is naïve: a process that crashes on boot will
thrash your machine, and a liveness blip that self-heals should not trigger a
restart at all. `respawn` turns the reflex into a **decision**:

- a **grace period** debounces transient flaps before reacting;
- **`max_restarts` within a sliding `window`** caps thrash and forces a
  `cooldown` when exceeded;
- **backoff** (constant or exponential, capped at 5 minutes) paces retries;
- an **`unhealthy`** liveness failure on an otherwise-running unit is recovered
  as *stop-then-start*, not a bare start.

The heart of it is a pure function:

```go
func Decide(policy *RespawnPolicy, h *History, attempt int, sig Signal, now time.Time) Plan
```

Because it takes `now` as an argument and returns a plain `Plan` value, every
scheduling decision is unit-testable against a fixed clock — no goroutines, no
real sleeps. The concurrent `Reconciler` is a thin shell that feeds signals into
this core and effects the resulting plan through a caller-supplied `Actions`
interface.

## Why no weft, gRPC, or health-checker dependency

These primitives were lifted from a microVM agent, where the restart policy came
from a protobuf message and health came from a probe runner. Both couplings were
**cut**: `RespawnPolicy` is now a local value type, and health is expressed as a
local `SignalKind` (`down`/`up`/`unhealthy`/`healthy`) — you feed those signals
from whatever source you like. The result is a pair of libraries with **zero**
third-party dependencies that drop into any init, agent, or workload manager.
