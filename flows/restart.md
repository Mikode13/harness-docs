# Restart of the server

R08 of [mvp.md](../mvp.md): what happens to the runs that were working when the API stops,
by a deployment or by a failure, and starts again, and to a restore or a push that was in
progress. The rules were decided on 2026-11-14 and
2026-11-15, in the [run lifecycle](../design/run-lifecycle.md); this document puts them in
order. The terms are in the [glossary](../glossary.md). None of it is built yet.

A run that was working is not resumed and is not run again. It becomes `interrupted`, and
the user decides how to go on.

## The flow

```mermaid
sequenceDiagram
    participant C as Client
    participant A as API
    participant D as Database
    participant P as Executors and workspace processes

    Note over A: stops
    Note over C: its subscription drops
    Note over P: may go on running, with no API to talk to

    Note over A: starts again
    A->>D: reads the runs that are not ended
    A->>D: reads the restores and pushes that were in progress
    A->>P: ends what is left of each of them
    A->>A: confirms that nothing of each is left
    A->>D: each run is interrupted, its pending ask is closed
    A->>A: reconciles each restore or push, the workspace can be used again

    C->>A: subscribes, after the last number it saw
    A-->>C: the events it missed, the interruption among them
    Note over C: the user decides how to go on
```

1. **The API stops.** Its clients lose their subscriptions. Its executors and its workspace
   processes do not end with it: each can go on running, and so can a command an executor
   started.
2. **The API starts again and reads the runs that are not ended**: those that were
   `setting_up`, `running`, `waiting_for_input` or `cancelling`.
3. **It also recognizes the restores and the pushes that were in progress.** A workspace
   process restores and pushes outside a run, so it is not among those runs, and it may
   still be writing in a workspace.
4. **For each, it ends what is left and confirms that nothing is.** The executor or the
   workspace process, and every process it started. Until that is confirmed the run is not
   ended and the restore or the push is not over, and the workspace takes no work: a prompt
   is rejected as busy.
5. **Each run becomes `interrupted`, and its pending ask is closed.** The API lost its
   channel to the executor that asked, so an answer could not reach it.
6. **Each restore or push is reconciled**, so that the workspace is in a state the product
   knows and can be used again. The harness finishes a move through its history that its
   process did not finish, with the next move or run; what a push needs is not decided.
7. **A client that reconnects gets what it missed**, as in
   [prompt-and-reconnect](prompt-and-reconnect.md), and sees that the run was interrupted.
8. **The user decides how to go on.** Nothing is run again by itself: a prompt that had
   started is never replayed, since its tools may already have had effects.

## An executor that dies while the API runs

The API sees the executor's process exit. The rest is the same from step 4: it confirms that
nothing the run started is left, and the run becomes `interrupted`. A workspace process that
dies during a restore or a push is treated the same way, and its work is reconciled.

## What is left

The messages and the events stored before the restart are kept, and so is the code that
earlier runs completed. The user sees everything up to the last stored event, and at most
one more message that says what happened.

Whether what the interrupted run itself changed stays in the workspace is not decided; it is
an open point of the [run lifecycle](../design/run-lifecycle.md#open-points).

## Open points

Not decided.

- **How a restore or a push in progress is recognized after a restart**, and what
  reconciling a push takes: it may have reached one project and not another. It is an open
  point of the [run lifecycle](../design/run-lifecycle.md#open-points).
- **How the processes of a run are ended and confirmed gone** after a restart, when the API
  no longer holds the executors it started. It is an open point of the
  [run lifecycle](../design/run-lifecycle.md#open-points);
  [learnings](../docs/learnings.md) has what showed the need.
- **What an executor did after it lost the API.** It may have gone on changing files, with
  nobody to report to, so no event tells it. Whether the user is told, and from what, was
  not discussed.
- **The snapshot of an interrupted run.** It left none. Whether the next run starts from
  the snapshot before it, and what it is told; see
  [prompt-and-reconnect](prompt-and-reconnect.md#open-points).
- **Going on from a checkpoint.** R08 allows restoring execution from supported safe
  checkpoints. None is supported yet: resuming a run that waits for an answer depends on
  [harness#136](https://github.com/Mikode13/harness/issues/136).
