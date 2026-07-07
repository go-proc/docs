# supervisor — subreaper + process supervisor

`github.com/go-proc/supervisor` provides the two PID-1 responsibilities of an
init: **reaping** every orphaned child, and **supervising** a declared set of
processes through a pluggable runtime.

```sh
go get github.com/go-proc/supervisor
```

## The subreaper

```go
reaper := supervisor.NewReaper(log)   // nil log → discard
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
go reaper.Run(ctx)                     // run for the life of PID 1

for ev := range reaper.Events() {
    // ev.PID, ev.ExitStatus, ev.Signaled, ev.Signal
}
```

`Run` installs a `SIGCHLD` handler, then on every signal drains all reapable
children with a non-blocking `Wait4(-1, WNOHANG)`, publishing one `Reaped` per
collection. It does an initial drain before blocking so an exit that races
startup is never missed. On non-Linux platforms `Run` is a no-op stub that blocks
until `ctx` is cancelled — the API is identical, so cross-platform builds compile
and vet cleanly.

```go
type Reaped struct {
    PID        int
    ExitStatus int  // exit code; -1 when terminated by a signal
    Signaled   bool
    Signal     int  // terminating signal number when Signaled
}
```

No `syscall` types leak into the API: the Linux reaper translates a `WaitStatus`
into these portable fields.

!!! note "A full `Events` buffer drops correlation, not zombies"
    The events channel is buffered (64). If a receiver stalls and the buffer
    fills, the reaper **still reaps** the zombie — only the exit *notification* is
    dropped (and logged). Zombies never leak.

## The supervisor

```go
spec := &supervisor.Spec{Procs: []supervisor.Proc{
    {ID: "web", Target: "/bundles/web", Restart: supervisor.RestartAlways},
    {ID: "job", Target: "/bundles/job", Restart: supervisor.RestartOnFailure},
}}
sup := supervisor.New(log, myRuntime, reaper, spec)
if err := sup.Run(ctx); err != nil { // blocks until all done or ctx cancel
    log.Error("supervisor exited", "err", err)
}
```

`Run` starts each `Proc` via the `Runtime`, indexes its pid, then launches a
goroutine that consumes reaper events and applies each process's restart policy.
It blocks until every process has exited with no further restart, or `ctx` is
cancelled. A failed *initial* start is fatal (returned wrapped); a per-process
exit handled by the restart policy is not.

### Restart policies

```go
const (
    RestartNever     RestartPolicy = ""            // never restart (zero value)
    RestartOnFailure RestartPolicy = "on-failure"  // restart on non-zero exit or signal
    RestartAlways    RestartPolicy = "always"      // restart on any exit
)
```

### The Runtime interface

The supervisor never interprets what a `Proc` *is* — `Proc.Target` is an opaque
handle the `Runtime` understands (an OCI bundle dir, a command, a unit name).
Implement six methods and the supervisor drives anything: an OCI runtime
(crun/runc), a raw fork+exec launcher, or a test fake.

```go
type Runtime interface {
    Name() string
    Create(ctx context.Context, id, target string, stdio Stdio) error
    Start(ctx context.Context, id string) error
    State(ctx context.Context, id string) (status string, pid int, err error)
    Kill(ctx context.Context, id, signal string) error
    Delete(ctx context.Context, id string) error
}
```

### Clean shutdown

```go
sup.Stop(ctx, 5*time.Second) // SIGTERM every process, wait grace, then SIGKILL survivors
```

`Stop` disables restart for the rest of the supervisor's life, sends `SIGTERM` to
each still-running process (suppressing `ESRCH` for processes already gone), waits
up to `grace`, then `SIGKILL`s any survivors.

## Wiring it together

The reaper and supervisor share one `Reaper`: the reaper collects *all* zombies;
the supervisor correlates the ones whose pids it launched and ignores the rest.
Run the reaper for the whole life of PID 1; run (and optionally `Stop`) the
supervisor for the workload you manage.
