# What the product needs from the harness

What the harness has to offer before the API can be built on it. A need is noted here when a
flow or a design document shows it, so the list is ready when that step of the
[order of work](../flows/README.md#order-of-work) comes.

A need is written as something any consumer of the harness could want. The harness learns
nothing about the product from this list: it does not know the API, a session, a device or
an executor, and nothing here asks it to.

## Agreed

### Save and restore an agent's conversation

[harness#111](https://github.com/Mikode13/harness/issues/111).

The API starts one executor for each run, as a process of its own; see
[where a run executes](run-lifecycle.md#where-a-run-executes). The harness keeps an agent's
conversation in memory, inside its process, so a new process for the second prompt starts
knowing nothing of the first.

The harness has to give the conversation of an agent as something a consumer can store, and
take it back to continue from it. The product calls it a snapshot and stores it as it is,
without reading it: its format is the harness's.

The flow [prompt-and-reconnect](../flows/prompt-and-reconnect.md) shows where it is used: a
run's executor restores the agent from the snapshot the run before left, and hands over its
own with the final answer, which the API stores before it ends the run.

The maintainer chose, on 2026-11-15, to do this before the API, once the design is finished.
Two other ways were considered:

- **An executor that lives as long as its session is in use**, keeping the conversation in
  memory between runs. It does not need this, but a restart loses what the model knew.
- **Passing only the text of the earlier messages** to each run, and replacing it later. It
  blocks nothing, but each run starts almost from nothing about the code.

### Save and restore a workflow's conversations

[harness#136](https://github.com/Mikode13/harness/issues/136).

A run of the product can be a workflow, with a planner, an executor and a reviewer; the
maintainer confirmed on 2026-11-21 that the MVP uses workflows. #111 covers one model-API
agent and leaves composed flows out, so the snapshot of a workflow needs #136, which builds on
#111. For the MVP, the snapshot taken when a run completes is enough: the product accepts that
a run cut short is interrupted, so continuing from a checkpoint inside a run can come later.

### The final answer and the ask as events

[harness#177](https://github.com/Mikode13/harness/issues/177).

Today the final answer of a run travels in its result, and what an agent asks its user
arrives through a callback: `ask` for a question and `approve` for an approval. Neither is
an event.

Both happen inside a run, so they belong with the harness's events; see
[events](events.md). The harness should offer them as events too, so that a consumer that
stores and forwards events has everything a run produced in one place.

The maintainer agreed to the two families of events on 2026-11-15, and this need was noted
then. On 2026-11-21 he chose the first of the options #177 gives: a complete record of the
run, apart from the optional narration of progress, with its own contract. The harness has a rule that a progress
event never carries what a consumer needs to be correct, so the issue asks for that rule to
be kept or changed on purpose.

## Candidates

Implied by the documents, and not agreed. Each one says why it might be needed.

- **The error that ends a run, as an event.** [events](events.md) puts an error inside a
  run, with the final answer and the ask, but only those two were named as needs. Today a
  run that fails throws.
- **A version for the schema of the events.** The product stores each event with the version
  of its schema, since a stored event outlives the harness that produced it. Whether that
  version is the harness package's own or one the harness states for its events is open.
- **Restoring a run that waits for an answer.** After a restart, a run that asked something
  is interrupted and its ask is closed, because the API has lost the executor that asked; see the
  [run lifecycle](run-lifecycle.md#rules). Going on from the ask instead depends on
  [harness#136](https://github.com/Mikode13/harness/issues/136). The MVP accepts the
  interruption, so this is not needed for it.
- **Following a run's commits when their branch is renamed.** The harness's history ties a
  run's commits to the name of their branch, and takes them back only while that branch is
  checked out. The product does not need this: it never renames a worktree's branch, and
  publishes it under another name instead; see
  [changes-and-push](../flows/changes-and-push.md#the-name-of-the-branch). It would be
  needed if a branch had to be renamed in its worktree.
- **Telling an agent that its earlier run did not complete.** The model of the next run is
  told; see the [run lifecycle](run-lifecycle.md#rules). Who says it is not decided: the
  harness may be the one that knows how, as it already tells an agent what a move through
  the history undid.
