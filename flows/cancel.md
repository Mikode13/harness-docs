# Cancel

R05 of [mvp.md](../mvp.md): the user cancels a run that is working, and can then send a new
prompt. The rules were decided on 2026-11-14 and 2026-11-15, in the
[run lifecycle](../design/run-lifecycle.md); this document puts them in order. The terms are
in the [glossary](../glossary.md). None of it is built yet.

## The flow

```mermaid
sequenceDiagram
    participant C as Client
    participant A as API
    participant D as Database
    participant E as Executor
    participant S as Another device

    C->>A: cancel, as a command
    A->>D: the run is cancelling, a pending ask is rejected
    A-->>C: accepted
    A-->>S: state event: cancelling
    A->>E: cancel

    Note over E: stops the harness and everything it started
    E-->>A: stopped, and ends
    A->>A: confirms that nothing of the run is left
    A->>D: the run is cancelled
    A-->>C: state event: cancelled
    A-->>S: state event: cancelled

    C->>A: a new prompt
    Note over A: accepted, since the run is ended
```

1. **A client sends the cancellation as a command.** It gets back only that it was accepted.
   Disconnecting a client is never a cancellation.
2. **The run becomes `cancelling`.** If it was `waiting_for_input`, its ask is rejected, and
   an answer that arrives from now on does not count.
3. **The API tells the executor**, which stops the harness and everything the harness
   started, and says so. A run that is still `setting_up` has a workspace process instead,
   which is stopped the same way.
4. **The API confirms that nothing of the run is left**: the executor and every process it
   started. Only then is the run `cancelled`.
5. **Every subscribed device sees both changes of state**, the one that cancelled as the
   others do.
6. **A new prompt is accepted once the run is `cancelled`.** While it is `cancelling`, a
   prompt is rejected as busy, because what the run started may still be writing in the
   workspace.

## What is left

The user sees everything up to the last event, and at most one more message. Nothing is
removed. See [what is left after a run that does not complete](../design/run-lifecycle.md#what-is-left-after-a-run-that-does-not-complete).

## Open points

Not decided.

- **An executor that does not confirm.** The rule holds either way: the run is not ended
  until everything it started is confirmed gone. How the API ends those processes itself,
  and confirms it, is an open point of the
  [run lifecycle](../design/run-lifecycle.md#open-points).
- **A cancellation of a run that already ended**, such as one sent from a second device a
  moment after the first. What the client gets back was not discussed.
- **The snapshot of a cancelled run.** Whether the next run starts from the snapshot before
  it, and what it is told; see [prompt-and-reconnect](prompt-and-reconnect.md#open-points).
- **The code the run changed before it stopped.** It is an open point of the
  [run lifecycle](../design/run-lifecycle.md#open-points).
