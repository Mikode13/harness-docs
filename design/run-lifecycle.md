# Run lifecycle

How a run starts, where it executes, which states it goes through and what is left when it
ends. The maintainer decided this on 2026-11-14, in the system-design discussion of agenda
items 1 and 2 of [mvp.md](../mvp.md). The terms are in the [glossary](../glossary.md). None of it
is built yet.

## Where a run executes

The API starts one executor for each run, as a child process of its own. The executor runs
the harness for that run and nothing else.

- The executor talks only to the API, over a local channel. It does not connect to the
  database.
- Before the executor, the API starts a workspace process when the session's workspace has
  to be prepared. It is a child process too, talks only to the API, and ends before the
  executor starts. See the [rules](#rules).
- The API is the only writer of the database, and so the only writer of a run's state. The
  executor reports what happens; the API stores it and decides the state.
- While the API runs, it knows an executor ended because its process exited. No heartbeat
  is needed for that.
- Being a child does not make an executor end with the API. On Linux an executor can outlive
  the API that started it, and a command can outlive the executor that started it. The first
  of the [rules](#rules) says what that requires.

## States

| State               | Meaning                                                                 |
| ------------------- | ----------------------------------------------------------------------- |
| `setting_up`        | The prompt is accepted and stored, and the workspace is being prepared. |
| `running`           | The harness is working.                                                 |
| `waiting_for_input` | The run asked the user something and waits. Its last event is an ask.   |
| `cancelling`        | The user cancelled, and the executor is stopping what it started.       |
| `cancelled`         | The executor confirmed that everything stopped.                         |
| `completed`         | The run ended with its answer, and its snapshot is stored.              |
| `failed`            | The run ended with an error.                                            |
| `interrupted`       | The API restarted, or the executor died, while the run was not ended.   |

`cancelled`, `completed`, `failed` and `interrupted` are final: a run in one of them never
changes again.

```mermaid
stateDiagram-v2
    [*] --> setting_up
    setting_up --> running
    setting_up --> failed
    running --> waiting_for_input: ask
    waiting_for_input --> running: answer
    running --> completed
    running --> failed
    setting_up --> cancelling: cancel
    running --> cancelling: cancel
    waiting_for_input --> cancelling: cancel
    cancelling --> cancelled
    setting_up --> interrupted: restart
    running --> interrupted: restart
    waiting_for_input --> interrupted: restart
    cancelling --> interrupted: restart
    completed --> [*]
    failed --> [*]
    cancelled --> [*]
    interrupted --> [*]
```

In the diagram, "restart" stands for a restart of the API and for the death of an executor.

## Rules

- **A workspace process prepares the workspace, while the run is `setting_up`.** The API
  starts it before the run's executor, as the writing user whatever the run's agent. Every
  run checks that its session's workspace is ready and has it prepared when it is not, so
  the first run clones and a later one rebuilds what was lost. A failure while preparing
  ends the run as `failed`. See [create-session](../flows/create-session.md).
- **Everything a run started is gone before another run starts.** Ending a run means ending
  its executor, its workspace process if it was still preparing, and every process they
  started, and confirming that none is left.
  Until that is confirmed the run is not ended, and neither its conversation nor its
  workspace takes another run. It holds for a cancellation, for an executor that died and
  for a restart of the API: a process that lost its parent keeps running, and marking its
  run `interrupted` while it runs would let two runs write in the same workspace. How the
  processes are ended and how their end is confirmed is an [open point](#open-points).
- **One active run per conversation.** A prompt sent while a run of the conversation is not
  ended is rejected as busy, as R05 requires. Nothing is queued. A prompt whose id the API
  already has is not a new prompt: it gets the acceptance and the run of the first time,
  whatever state that run is in.
- **A restore or a push holds the workspace as a run does.** A workspace process that
  restores code or pushes works outside a run. While it works, its session takes no prompt,
  which is rejected as busy, and neither a clear nor a deletion. It is ended and confirmed
  gone as a run's processes are. After a restart, the API has to recognize a restore or a
  push that was in progress, end or reconcile it, and confirm that the workspace can be
  used again, before the workspace takes any work. See [restart](../flows/restart.md).
- **One writing run per workspace.** Whether a run writes is known before it starts: its
  agent has tools that write, or it has none. Two runs that write in the same workspace
  would work against each other. Inside one run, several subagents may work in the
  workspace at once. On the server each session has worktrees of its own, so this follows
  from the rule above.
- **A new prompt waits for `cancelled`.** While a run is `cancelling`, what it started may
  still be writing, so the conversation stays busy until the executor confirms the stop.
- **No clear and no delete while a run is active.** The API rejects clearing or deleting a
  session whose conversation has a run that is not ended. The clients show a message or do
  not offer the action, and the API enforces it whatever the client does.
- **Cancelling rejects a pending ask.** A run cancelled while it is `waiting_for_input` has
  its ask rejected.
- **A late answer does not count.** An answer that arrives after the run was cancelled, or
  after the ask was answered, is not considered and does not resume anything.
- **A restart closes a pending ask.** The ask is a stored event, so a client that reconnects
  gets it again as the last event and can answer it. After a restart, the API has lost its
  channel to the executor that asked, whether or not that process is still there, so an
  answer cannot reach it. Once everything the run started is confirmed gone, the run becomes
  `interrupted` and its ask is closed, so no client shows a question that leads nowhere.
- **A run is `completed` once its snapshot is stored.** The executor hands the snapshot over
  with the final answer, and the API stores both before it ends the run. A conversation
  whose messages are stored and whose snapshot is lost shows the chat, and has lost what the
  next prompt needs. See [prompt-and-reconnect](../flows/prompt-and-reconnect.md).

## What is left after a run that does not complete

After `failed`, `cancelled` or `interrupted`, the user sees everything up to the last event,
and at most one more message that says what went wrong. Nothing is removed.

The next run can recover the conversation, and compacts it when it is very large.

## Alternatives considered

- **Running the harness inside the API's process.** The simplest: one process, and the
  harness's callbacks are plain calls. Rejected because a run that hangs or runs out of
  memory takes the API down with it, and because the commands a model starts would run as
  the user that holds the database credentials.
- **An executor as a service of its own.** It would also let the API restart without ending
  the runs. It needs a channel that survives either side going away, a heartbeat to tell a
  dead executor, and a way to reconcile both states when they meet again. Left for later:
  the states above do not change if the executor moves out.

## Open points

Not decided. Each is an input to the next design steps.

- **How a run's processes are ended and confirmed gone.** It has to reach a process that
  lost its parent, and work after a restart of the API, when the API no longer holds the
  executors it started.
- **How a restore or a push in progress is recognized after a restart**, and how it is
  ended or reconciled. It is not a run, so the list of runs that are not ended does not
  hold it.
- **The channel between the API and the executor.** What carries the events one way and the
  answers and the cancellation the other.
- **What the model is told.** Whether the next prompt tells the model that the earlier run
  did not complete.
- **Code a run changed before it stopped.** It might stay in the workspace until the user
  restores it (R09); this was assumed in the discussion, not confirmed.
- **Reasons within `failed`.** Whether a failed run names a kind of failure, such as the
  provider or the workspace, or only carries a message.

## Words used elsewhere

- **[#4](https://github.com/Mikode13/harness-api/issues/4)** names the same states for a
  task, with `queued` where this document says `setting_up`, and without `cancelling`.
- **[#6](https://github.com/Mikode13/harness-api/issues/6)** describes "a separately
  runnable backend executor" that claims queued tasks from PostgreSQL. Here the API starts
  an executor per run as its child, and the executor does not reach the database. The issue
  keeps its text until it is refined. Its other rules stand: a cancellation waits for the
  commands to end before it reports `cancelled`, a dead executor's work becomes
  `interrupted`, and a started prompt is never replayed by itself.
