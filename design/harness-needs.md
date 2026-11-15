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
take it back to continue from it. The product stores it as it is, without reading it: its
format is the harness's.

The maintainer chose, on 2026-11-15, to do this before the API, once the design is finished.
Two other ways were considered:

- **An executor that lives as long as its session is in use**, keeping the conversation in
  memory between runs. It does not need this, but a restart loses what the model knew.
- **Passing only the text of the earlier messages** to each run, and replacing it later. It
  blocks nothing, but each run starts almost from nothing about the code.

### The final answer and the ask as events

[harness#177](https://github.com/Mikode13/harness/issues/177).

Today the final answer of a run travels in its result, and what an agent asks its user
arrives through a callback: `ask` for a question and `approve` for an approval. Neither is
an event.

Both happen inside a run, so they belong with the harness's events; see
[events](events.md). The harness should offer them as events too, so that a consumer that
stores and forwards events has everything a run produced in one place.

The maintainer agreed to the two families of events on 2026-11-15, and this need was noted
then. How the harness offers them is not designed. The harness has a rule that a progress
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
  is interrupted and its ask is closed, because the run that asked no longer exists; see the
  [run lifecycle](run-lifecycle.md#rules). Going on from the ask instead depends on
  [harness#136](https://github.com/Mikode13/harness/issues/136). The MVP accepts the
  interruption, so this is not needed for it.
- **Telling an agent that its earlier run did not complete.** It is an open point of the
  [run lifecycle](run-lifecycle.md#open-points). If the model is to be told, the harness may
  be the one that knows how to say it, as it already tells an agent what a move through the
  history undid.
