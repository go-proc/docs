# Contributing

Contributions are welcome on either library
([supervisor](https://github.com/go-proc/supervisor),
[respawn](https://github.com/go-proc/respawn)).

## Ground rules

- **Pure Go, `CGO_ENABLED=0`.** No cgo, no third-party dependencies in the core.
  Platform-specific code goes behind a build tag with a portable stub so
  `go build` / `go vet` stay green on every OS.
- **100% test coverage**, error branches included, is a **CI gate**. New code
  ships with tests that cover it — the pipeline fails below 100%.
- **`gofmt` + `go vet` clean**, and `-race` clean.
- **BSD-3-Clause**, English-only in all public content (issues, PRs, commits).

## Running the checks locally

```sh
go vet ./...
go build ./...
COVERPKG=$(go list ./... | paste -sd, -)
go test -race -coverpkg="$COVERPKG" -coverprofile=cover.out ./...
go tool cover -func=cover.out | tail -1   # must read 100.0%
```

!!! note "Coverage of the Linux subreaper"
    In `supervisor`, the subreaper's `SIGCHLD` + `Wait4` path is Linux-only, so
    the coverage gate is measured on the native Linux lane — that is where the
    reaper is both compiled and exercised. Its `drain` loop is driven
    deterministically through a `wait4` seam (exit / signal / error / buffer-full
    cases) plus a real `SIGCHLD` delivery test that skips itself under qemu-user.
    macOS/Windows lanes cover the portable supervisor logic.

## Cross-arch

CI runs the suite natively on amd64/arm64 and under qemu-user on riscv64,
loong64, ppc64le, and s390x. Keep new code arch-neutral; if a test needs native
syscalls, gate it so the qemu lanes stay green.

## Docs

These docs are MkDocs Material, versioned with [mike](https://github.com/jimporter/mike).

```sh
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
mkdocs serve   # http://localhost:8000
```
