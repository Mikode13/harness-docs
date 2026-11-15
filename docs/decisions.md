# Decisions

This file records the product's decisions, which cross its projects: scope, vocabulary, flows
and design. A decision about how one repository is built goes in that repository's
`docs/decisions.md`, and a rule for all MiKode projects goes in
[`Mikode13/engineering`](https://github.com/Mikode13/engineering). Entries are in
chronological order, and each has a **Decision**, a **Context** and its **Consequences**.

## Sessions hold conversations, and clearing archives

**Decision:** The user lists, opens, renames and deletes sessions. A session holds one active
conversation at a time, and clearing it archives that conversation and starts a new, empty one
in the same session. Deletion is logical first, and a real purge may come later. An archived
conversation can still be found; a deleted one is no longer returned by listing, opening or
retrieval. [`glossary.md`](../glossary.md) defines the terms.

**Context:** The maintainer's example is the session for translations, whose workspace is a
worktree for translations: the session is cleared to start a different translations task and
keeps its title and workspace. A conversation per work item would tie the workspace to
something the user discards. R13 of [`mvp.md`](../mvp.md) retrieves messages from other
conversations, so a clear cannot delete the conversation it replaces. Deleting is for what is
not worth remembering, such as a throwaway question.

**Consequences:** R01 is reworded around sessions and gains clearing, and the other
requirements change a word only where it meant something else. Issues that say "conversation"
or "task" keep their text until they are refined; the glossary lists where they differ. The
states of a run, the order and storage of events, and what a deletion does to archived
conversations are not decided; the glossary's open points hold the proposals.

## A session works in a workspace, whose roots are worktrees on the server

**Decision:** A session has one workspace, or none when it is not about code. A workspace is
where the harness works: one root or several, each a worktree or a project's main folder. A
worktree is a copy of a repository and nothing more. On the server of the MVP every root is a
worktree. [`glossary.md`](../glossary.md) defines the terms.

**Context:** The harness can work in a worktree, in a project's main folder, or in several
projects at once, such as the API and the harness together, and its `workspace` option is
already a list of `roots`. Calling all of that a worktree named one case for the whole. The
session has the workspace, not the reverse, because the session is what the user lists and a
session without code has no workspace.

**Consequences:** R03 of [`mvp.md`](../mvp.md) stays as it is: sessions on the same project use
separate worktrees. Two sessions on a project's main folder would write in the same place,
where the harness lets one run write at a time, so that form is left for local use. A
conversation has many runs, one per prompt, and each run has a state; which states is not
decided.

## A run executes in a child process of the API, one per run

**Decision:** The API starts one executor per run, as a child process. The executor runs the
harness for that run, talks only to the API over a local channel, and does not connect to the
database. The API is the only writer of the database and of a run's state.
[`run-lifecycle.md`](../design/run-lifecycle.md) holds the rules.

**Context:** Agenda item 1 of [`mvp.md`](../mvp.md). Inside the API's process, a run that hangs
or runs out of memory takes the API down, and the commands a model starts run as the user that
holds the database credentials. An executor as a service of its own would also survive a
restart of the API, but needs a channel that survives either side going away, a heartbeat and
a reconciliation. The maintainer chose the child process on 2026-11-14.

**Consequences:** A failing run leaves the API up, and killing its process always ends it. The
API sees an executor die because its process exits, with no heartbeat. A restart of the API
ends every executor, and their runs become `interrupted`. Issue
[#6](https://github.com/Mikode13/harness-api/issues/6) describes a separately runnable
executor that claims tasks from PostgreSQL; it keeps its text until it is refined. Moving the
executor out later does not change a run's states.

## A run has eight states, and a restart interrupts it

**Decision:** A run is `setting_up`, `running`, `waiting_for_input`, `cancelling`, `cancelled`,
`completed`, `failed` or `interrupted`, and the last four are final. A cancellation passes
through `cancelling` until the executor confirms that everything stopped. Cancelling rejects a
pending ask, and an answer that arrives later does not count. After a restart of the API, a
run that was not ended becomes `interrupted` and its pending ask is closed. A session cannot
be cleared or deleted while a run of its conversation is not ended; the API enforces it and
the clients mirror it.

**Context:** Agenda item 2 of [`mvp.md`](../mvp.md). Issue
[#4](https://github.com/Mikode13/harness-api/issues/4) listed the states for a task with
`queued`; nothing is queued (R05), and what happens first is preparing the workspace, so it
is `setting_up`. Stopping a command's processes takes seconds, and a prompt accepted in that
time would write in the same workspace. An ask is a stored event, which a reconnecting client
gets again, but the run that asked lives in the executor and does not survive a restart.

**Consequences:** One run is active per conversation, and one writing run per workspace,
which on the server is the same rule. A question asked before a restart has to be asked again
by a new run; resuming it depends on harness
[#136](https://github.com/Mikode13/harness/issues/136). After a run that does not complete,
the user sees everything up to the last event and at most one message with the error, and the
next run recovers the conversation, compacting it when it is very large.

## The product's design lives in a repository of its own

**Decision:** `harness-docs` holds the scope, the glossary, the flows, the design documents, and
the decisions and learnings that cross projects. Each code repository keeps only the documents
about how it is built, and links here.

**Context:** The maintainer decided this on 2026-11-15. The documents were in harness-api, but
they also describe the CLI, the mobile app and the contract package. The harness cannot hold
them, because it does not know the product. A GitHub wiki has no pull requests, no review and
no link checks.

**Consequences:** The documents no longer travel in the pull request that changes the code, so
each code repository's `AGENTS.md` points here and requires the flow to be updated with the
change. harness-api's pull requests #21, #22 and #23, which added these documents there, are
replaced by this repository.

## Events come in two families, and a contract package defines the product's

**Decision:** An event is one of two kinds. What happens inside a run, such as a tool's
activity, text, an ask, the final answer or an error, is defined by the harness. The state
around a run, such as its workspace being set up, an interruption by a restart or a session
being cleared, is defined by the product, in a contract package of types only that depends on
the harness. The API, the CLI and the mobile app depend on that package. The API stores and
delivers the harness's events as they are and adds only what it needs, and each stored event
carries the version of its schema. [`design/events.md`](../design/events.md) holds the rest.

**Context:** Decided by the maintainer on 2026-11-15. He first asked to reuse the harness's
events, with at most the ones the API needs for its own infrastructure, and then set the two
apart: one is what occurs inside the conversation, the other is the state of the
conversation. His rule for the dependencies is that they go from the harness outwards: the
harness knows nothing of the API or of the CLI, and the others build on it. The harness
therefore cannot define the second family, since "interrupted because the API restarted" is
the API under another name. Defining it in the API would make every client depend on the API.

**Consequences:** There is one schema for what a run produces and nothing to translate. A
package that does not exist yet has to be created and named. The final answer and the ask are
not events in the harness today, so the API builds them until the harness offers them;
[`design/harness-needs.md`](../design/harness-needs.md) lists it. A client holds two
subscriptions, one for a conversation and one for the user's changes of state, and the second
is part of the contract between the API and the clients, unknown to the harness.

## An ask is not sent to a device, and the first answer wins

**Decision:** The API does not route an ask to a device. It adds the ask to the
conversation's events, and the devices subscribed to that conversation get it. A device is
active when its subscription is open; the device that sent the prompt is not special. A
device that was away asks for the events after the last number it saw and gets the ask, or
the ask and its answer. The first answer wins and a later one is rejected. The device that
answers gets only accepted or rejected, and updates its screen from the answer event.
[`flows/ask-and-answer.md`](../flows/ask-and-answer.md) holds the flow.

**Context:** Decided by the maintainer on 2026-11-15. His concern was telling which devices
are in use: with a desktop, a laptop and a mobile, an ask must not go to the desktop when it
is off or has not loaded that conversation, and the server cannot know that a laptop closed.
Keeping track of devices to send each ask to the right one would need a list of devices and
of what each has open. An open subscription already says which devices are there, and
R06 of [`mvp.md`](../mvp.md) says a question does not depend on the terminal that started the
run.

**Consequences:** The API keeps no list of devices and never needs to know whether one
received an ask: the record of events is the guarantee, not the delivery. A device that
answers and one that only watches update their screens the same way. Notifications are the
one place where a device has to be known, since they are for a device that is not connected;
they are out for now.

## One API key, and three users of the operating system

**Decision:** The API accepts one key of the product's own, shared by the maintainer's
devices, and only that key works. On the server the product runs as three users besides the
maintainer's own, which has sudo: a service user that runs the API and starts the executors,
an agent user that can only read, and an agent user that can do more. The executor of an
agent with no tools that write runs as the read-only user. Neither agent user has sudo or can
read the database credentials. [`design/security.md`](../design/security.md) holds the rest.

**Context:** Decided by the maintainer on 2026-11-15. The key is so that nobody else can use
the harness through the server. A key per device was proposed, to revoke a lost device's
alone, and declined: losing a device is not a concern for him. The two agent users are his:
one only for reading and one with more capability. Whether an agent writes is known before
its run starts, from its tools, which is what lets the API choose the user.

**Consequences:** The API cannot tell devices apart by their key, and does not need to, since
an ask is not sent to a device. The operating system refuses a write from an agent that only
reads, as well as the harness. A workflow with a planner, an executor and a reviewer is one
process, so its roles that only read run as the writing user too. How the service user starts
a process as another user is not decided, and neither is where the server can be reached
from.

## The harness saves and restores a conversation before the API is built

**Decision:** Saving and restoring an agent's conversation,
[harness#111](https://github.com/Mikode13/harness/issues/111), is done before the API, once
the design is finished. [`design/harness-needs.md`](../design/harness-needs.md) lists what else
the product needs from the harness.

**Context:** Decided by the maintainer on 2026-11-15. The API starts one executor for each
run, as a process of its own, and the harness keeps an agent's conversation in memory, so the
second prompt of a conversation would start knowing nothing of the first. Two other ways were
considered: an executor that lives as long as its session is in use, which loses what the
model knew on a restart, and passing only the text of the earlier messages until harness#111
exists, which blocks nothing but makes each run start almost from nothing about the code.

**Consequences:** The API starts later, after the harness work, and is built on what it will
keep. The product stores the conversation the harness gives it without reading it, so its
format stays the harness's. The order of work is the flows, then the database, then what the
harness needs, then the API.
