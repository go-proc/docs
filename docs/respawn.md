# respawn — restart / reconcile state machine

`github.com/go-proc/respawn` decides *whether and when* to bring a unit back after
it goes down — honouring anti-flap and anti-thrash limits — and effects the
respawn through a caller-supplied interface. It is pure Go with **zero**
dependencies.

```sh
go get github.com/go-proc/respawn
```

## The signal model

You feed the machine a stream of signals per named unit:

| Signal | Meaning |
| --- | --- |
| `SignalDown` | the unit stopped (crash, eviction, operator action, host reboot) |
| `SignalUp` | the unit is running again — resets transient counters |
| `SignalUnhealthy` | a liveness probe failed past its threshold — a *soft* down |
| `SignalHealthy` | the rolling probe verdict crossed back to healthy |

Health is a **local `SignalKind`**, not an imported probe type — you decide what
"unhealthy" means and emit the signal.

## The pure decision core

```go
type RespawnPolicy struct {
    Enabled        bool
    GracePeriodMs  int64   // debounce a flap before reacting
    MaxRestarts    int32   // cap within...
    WindowMs       int64   // ...this sliding window
    Backoff        string  // "constant" | "exponential"
    InitialDelayMs int64   // doubles each retry when exponential (capped 5m)
}

func Decide(policy *RespawnPolicy, h *History, attempt int, sig Signal, now time.Time) Plan
```

`Decide` is an allocation-light pure function: given the policy, the per-unit
restart `History`, the incoming signal, and the current time, it returns a `Plan`
value describing what to do — no goroutines, no real sleeps. That makes every
decision assertable against a fixed clock.

```go
type Plan struct {
    State    State         // running | grace_wait | backoff | respawning | cooldown | stopped
    Action   Action        // none | start | stop | wait
    DelayFor time.Duration  // how long to wait (backoff / grace / cooldown)
    Reason   string
}
```

### The flow

1. A `down`/`unhealthy` signal first parks in **`grace_wait`** for
   `GracePeriodMs`, so a flap that self-heals never triggers a restart.
2. After grace, if `MaxRestarts` within the rolling `WindowMs` is **not**
   exhausted, the plan is a **`start`** (or **`stop`**-then-start for an
   `unhealthy` unit), preceded by **backoff**.
3. If the restart budget **is** exhausted, the unit moves to **`cooldown`** until
   the window rolls.

`unhealthy` is recovered as stop-then-start because the guest is nominally still
running — the policy does not trust it to recover on its own.

## The concurrent Reconciler

```go
r := respawn.New(myActions, log)      // log may be nil
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

r.Watch(ctx, "web-1", policy)          // one goroutine per unit, lazily started
r.Send(respawn.Signal{VMName: "web-1", Kind: respawn.SignalDown, When: time.Now()})

cancel(); r.Wait()                     // drain on shutdown
```

The `Reconciler` is the thin concurrent shell around `Decide`: one goroutine per
watched unit draining a buffered signal channel, fanning out across units by name
so distinct units never serialise. It effects plans through your `Actions`:

```go
type VMActions interface {
    StartVM(ctx context.Context, name string) error
    StopVM(ctx context.Context, name string) error
}
```

!!! note "Naming"
    The `Actions` methods and `Signal.VMName` keep the `VM` names of the origin
    codebase, but the package is **unit-agnostic** — a "VM" here is any named
    thing you supervise (a process, a container, a service).

## Composing with supervisor

`supervisor` answers "a process I launched exited — apply its `RestartPolicy`" at
the OS level. `respawn` answers the higher-level "given health/liveness signals
over time, *should* I respawn this unit, and how fast" — with debounce,
thrash limits, and backoff. Use `supervisor` for the local process tree, `respawn`
when restart decisions need memory and rate-limiting.
