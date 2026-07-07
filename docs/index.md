# go-proc documentation

**Pure-Go (no cgo) process-lifecycle primitives** — the small, reusable pieces an
init or a workload manager needs: a PID-1 subreaper, a process supervisor, and a
restart/reconcile state machine. Extracted from a microVM init and **generalized**
to depend on no particular workload model, transport, or scheduler.

Two libraries:

- [`github.com/go-proc/supervisor`](https://github.com/go-proc/supervisor) — a
  **PID-1 subreaper** (`SIGCHLD` + `Wait4`, Linux, with a portable no-op stub
  elsewhere) **and** a **process supervisor** that drives a pluggable `Runtime`
  over a `Spec` of `Proc`s and restarts them per `RestartPolicy`.
- [`github.com/go-proc/respawn`](https://github.com/go-proc/respawn) — a
  **restart / reconcile state machine**: grace-period debounce, sliding-window
  anti-thrash, and constant/exponential backoff, driven by
  `down`/`up`/`unhealthy`/`healthy` signals.

!!! success "Status: both primitives shipped"
    `supervisor` and `respawn` are released at **100% test coverage** (error
    branches included), race-tested, `gofmt` + `go vet` clean, CI green across the
    six 64-bit Go targets — amd64, arm64, riscv64, loong64, ppc64le, s390x. The
    Linux-only subreaper sits behind a build tag with a portable stub, so
    `go build` / `go vet` stay green on macOS and Windows too.

## Reap, supervise, respawn

Each library owns a distinct slice of process lifecycle:

| Concern | Library | Shape |
| --- | --- | --- |
| **Reap** orphaned children (PID 1's job) | `supervisor` | `Reaper` — `SIGCHLD` + `Wait4(-1, WNOHANG)` |
| **Supervise** declared processes | `supervisor` | `Supervisor` over a `Runtime` + `Spec` |
| **Respawn** on failure, without thrashing | `respawn` | `Decide` state machine + `Reconciler` |

The subreaper and supervisor are two responsibilities split on purpose: the
reaper runs **unconditionally** (PID 1 is the kernel's reaper of last resort for
the whole orphan tree, including helpers a runtime forks that you never tracked),
while the supervisor only cares about the PIDs it launched. `respawn` sits a layer
up: given a stream of health/liveness signals, it decides *whether and when* to
bring a unit back, honouring anti-flap and anti-thrash limits.

## Install

```sh
go get github.com/go-proc/supervisor
go get github.com/go-proc/respawn
```

## Principles

- **Pure Go, `CGO_ENABLED=0`** — trivial cross-compilation, a single static
  binary, no C toolchain. Platform-specific syscalls live behind build tags.
- **Generic seams.** A six-method `Runtime` interface, a two-method `Actions`
  interface, and local value types are the only coupling points — no weft, gRPC,
  or health-checker imports.
- **A pure decision core.** Scheduling is a pure function of
  `(policy, history, signal, now)`; concurrency is a thin shell around it.
- **100% test coverage**, including every error branch, enforced as a CI gate.

## Where to go next

- [Design — reap, supervise, respawn](why.md) — why the three concerns are split,
  and how portability is preserved.
- [supervisor](supervisor.md) — the subreaper, the `Runtime` interface, restart
  policies, and clean shutdown.
- [respawn](respawn.md) — the signal model, the `Decide` core, and the
  `Reconciler`.
- [Roadmap](roadmap.md) — what is done and what could come next.

Source lives at [github.com/go-proc](https://github.com/go-proc).
