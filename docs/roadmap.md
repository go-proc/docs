# Roadmap

Both primitives are shipped and released at 100% coverage across all six 64-bit
Go arches. This page tracks what is done and what could come next.

## Done

- [x] **supervisor · subreaper** — `SIGCHLD` + `Wait4(-1, WNOHANG)` drain,
  portable `Reaped` events, Linux build tag with a no-op stub elsewhere.
- [x] **supervisor · process supervision** — pluggable `Runtime`, `Spec`/`Proc`
  model, pid correlation, `RestartNever`/`RestartOnFailure`/`RestartAlways`,
  `Stop` with SIGTERM → grace → SIGKILL.
- [x] **respawn · decision core** — pure `Decide` / `DecideAfterGrace`, grace
  debounce, sliding-window `max_restarts`, constant/exponential backoff, cooldown.
- [x] **respawn · Reconciler** — per-unit goroutines, buffered signal fan-out,
  policy hot-swap, race-safe watch/unwatch.
- [x] **100% coverage** (error branches included), `-race`, `gofmt` + `go vet`
  clean, CI green on amd64, arm64, riscv64, loong64, ppc64le, s390x.

## Possible next steps

- [ ] A ready-made `Runtime` adapter for OCI runtimes (crun/runc) as a separate,
  optional sub-package (kept out of the core to preserve zero dependencies).
- [ ] A ready-made fork+exec `Runtime` for plain processes.
- [ ] Metrics/observability hooks (restart counts, cooldown entries) surfaced as
  a small callback interface.
- [ ] A helper that bridges `supervisor` exit events into `respawn` signals, so
  the two compose out of the box.

Have a use case that needs one of these sooner? Open an issue on the relevant
repository.
