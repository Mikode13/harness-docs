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

_Where it runs is superseded by the entry "A sandbox for each session": the executor runs in the
session's sandbox. One executor per run, talking only to the API, stands._

**Decision:** The API starts one executor per run, as a child process. The executor runs the
harness for that run, talks only to the API over a local channel, and does not connect to the
database. The API is the only writer of the database and of a run's state.
[`run-lifecycle.md`](../design/run-lifecycle.md) holds the rules.

**Context:** Agenda item 1 of [`mvp.md`](../mvp.md). Inside the API's process, a run that hangs
or runs out of memory takes the API down, and the commands a model starts run as the user that
holds the database credentials. An executor as a service of its own would also survive a
restart of the API, but needs a channel that survives either side going away, a heartbeat and
a reconciliation. The maintainer chose the child process on 2026-11-14.

**Consequences:** A failing run leaves the API up. While the API runs, it sees an executor end
because its process exits, with no heartbeat. Being a child does not make an executor end with
the API, nor a command with its executor: on Linux both can outlive their parent. So ending a
run means ending its executor and everything it started, and confirming that none is left,
before its conversation or its workspace takes another run. A restart of the API does the same
for the runs it finds not ended, and only then are they `interrupted`. How is not decided;
[`learnings.md`](learnings.md) has what showed it. Issue
[#6](https://github.com/Mikode13/harness-api/issues/6) describes a separately runnable
executor that claims tasks from PostgreSQL; it keeps its text until it is refined. Moving the
executor out later does not change a run's states.

## A run has eight states, and a restart interrupts it

**Decision:** A run is `setting_up`, `running`, `waiting_for_input`, `cancelling`, `cancelled`,
`completed`, `failed` or `interrupted`, and the last four are final. A cancellation passes
through `cancelling` until the executor confirms that everything stopped. Cancelling rejects a
pending ask, and an answer that arrives later does not count. After a restart of the API, a
run that was not ended becomes `interrupted`, once everything it started is confirmed gone,
and its pending ask is closed. A run is `completed` once its snapshot is stored. A session cannot
be cleared or deleted while a run of its conversation is not ended; the API enforces it and
the clients mirror it.

**Context:** Agenda item 2 of [`mvp.md`](../mvp.md). Issue
[#4](https://github.com/Mikode13/harness-api/issues/4) listed the states for a task with
`queued`; nothing is queued (R05), and what happens first is preparing the workspace, so it
is `setting_up`. Stopping a command's processes takes seconds, and a prompt accepted in that
time would write in the same workspace. An ask is a stored event, which a reconnecting client
gets again, but a restarted API has lost its channel to the executor that asked, so an answer
cannot reach it.

**Consequences:** One run is active per conversation, and one writing run per workspace,
which on the server is the same rule. A question asked before a restart has to be asked again
by a new run; resuming it depends on harness
[#136](https://github.com/Mikode13/harness/issues/136). After a run that does not complete,
the user sees everything up to the last event and at most one message with the error, and the
next run recovers the conversation, compacting it when it is very large. The code such a run
changed stays in the workspace, and the user decides whether to restore it; the maintainer
confirmed this on 2026-11-16, after it had only been assumed.

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

_The one key is superseded by the entry "An API key for each user, kept as a hash and able to
be disabled", and the two agent users by the entry "A sandbox for each session"._

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
keep. The product stores what the harness gives it, a snapshot, without reading it, so its
format stays the harness's. A run's executor restores the agent from the snapshot the run
before left and hands over its own with the final answer, and a run is `completed` only once
that snapshot is stored; [`flows/prompt-and-reconnect.md`](../flows/prompt-and-reconnect.md)
shows the steps. The order of work is the flows, then the database, then what the
harness needs, then the API.

## The client names a session and a prompt, and the API creates what it does not find

**Decision:** The client generates the id of a session and the id of each prompt. A session is
created by the first prompt sent to it: the API creates a session it does not find, takes the
prompt as one more of a session that exists, and rejects a prompt to a session that is deleted.
A prompt whose id the API already has is the same prompt sent again and starts no second run:
the API checks the id first after authenticating the request, and answers with the first
acceptance and run.
The API gives a new session a default title, the first prompt's text cut short, and a title
the user set is never replaced. [`flows/create-session.md`](../flows/create-session.md) holds
the flow.

**Context:** Decided by the maintainer on 2026-11-15. In his words, "new" makes an empty chat
that exists only in the client, and the id should come from the client, not from the API. He
saw two cases, a session that exists and one that does not. The third came from the review of
that idea: a device still showing a session another device deleted. So did the prompt's id: a
client that misses the acknowledgement of a first prompt sends it again, and the session now
exists, so without an id the retry would be a second prompt.

**Consequences:** No request creates a session by itself, and a session never exists without a
prompt. A retry is safe for the first prompt and for every other, which closes the open point
of the first flow about sending a prompt twice. The session keeps a mark that its title is the
user's.

## A session works in the registered projects its user chooses

**Decision:** A project is a name and its remote repository, registered in the API, and only
registered projects are cloned. The user chooses the projects of a session before the first
prompt, in the client, where the choice is visible and can be changed; the model never chooses.
The CLI proposes the project whose repository is the remote of the folder it runs in, a mobile
proposes the one used last, and "none" is an explicit choice. More than one project gives a
workspace with several roots. The workspace is fixed when the session is created. A worktree
starts from the repository's default branch, or from the branch of the user's folder when the
CLI offers it and the user takes it.

**Context:** Decided by the maintainer on 2026-11-15. With an agent on the user's machine the
project is the folder it runs in. On a server the CLI would have to say which folder, and a
folder's name is not reliable; its git remote is. He also wanted the door open: in the folder
of harness-cli, asking for a change in the harness. Stopping the run to ask which repository
to clone was considered and left for a session that starts with none.

**Alternatives considered:**

- **Any repository URL, sent with the first prompt.** More flexible, but the credential would
  have to reach every repository.
- **Asking during the run.** It needs an ask before the run has anything to work on, and a
  failure shows later.

**Consequences:** Registering a project is a deliberate step; how is not decided. A session
that needs another project is another session. Whether a session created with "none" can get
a project later is not decided. Changes in the user's folder that were not
pushed do not reach the server, and the CLI has to say so.

## A workspace process prepares the workspace before the executor, with the git credential

_Where it runs is superseded by the entry "A sandbox for each session": a workspace process runs
in the session's sandbox, not as the writing user, and the entry "Only a transfer process holds
the git credential" takes the credential from it._

**Decision:** A workspace process prepares a session's workspace inside the run, while it is
`setting_up`, before the run's executor starts: it clones each project and adds a worktree for
the session, and reports it with events of state. The API starts it as the writing user,
whatever the run's agent. Every run checks that the workspace is ready and has it prepared when
it is not. The same kind of process restores code and pushes a branch. The API holds the git
credential and passes it to the workspace process, never to an executor; it is used only to
clone, fetch and push, and is never written to disk nor to a worktree's git configuration. For
now the credential is the maintainer's own account.

**Context:** Decided by the maintainer on 2026-11-15 and 2026-11-16. He first gave the
preparation to the executor: it has the disk, so the API keeps to the database. The automated
review of that flow found that it could not hold: the executor of an agent with no tools that
write runs as the read-only user, which cannot clone and should not hold the credential. A
restore and a push also needed something to carry them out that is not a run. He would prepare
a workspace before the first prompt, but nothing says when a session will be used, or whether
it will, so it is prepared when the first run needs it. A separate git account for agents is
the aim and waits because of its cost.

**Alternatives considered:**

- **The run's executor prepares the workspace.** One process fewer, but an agent that only
  reads would need to write and to hold the credential.

**Consequences:** The first run of a session is slower, and the user sees why. A run that
fails while cloning ends as `failed` and leaves the session without a workspace, which the next
prompt has prepared again. No executor ever holds the credential. A run has two processes one
after the other, and ending a run means ending whichever is there. The next entry says what
was decided about who can still reach the credential, and a later one which credential it is.
The entry "Only a transfer process holds the git credential" takes the credential away from
the workspace process.

## A model's command can reach the git credential: accepted for now, to be changed

_Closed by the entry "A sandbox for each session": no code of a model runs outside a sandbox,
and a transfer process runs outside every sandbox._

**Decision:** For the first slice, the risk that a command a model starts reads the git
credential is accepted. It is limited by a credential that reaches only the registered
repositories. This is a known gap and not the intended design: it will be closed by running
the workspace process, which is what holds the credential, as a user of the operating system
that the model's commands do not run as.
[`design/security.md`](../design/security.md#git-credentials) states it.

**Context:** Decided by the maintainer on 2026-11-16, after the automated review of the flow
that creates a session. The flow kept the credential off disk and gave that as the protection.
The review pointed out that a command a model starts runs as the same user as the process that
holds the credential and could read it from that process, for example after a prompt injection
from web content. With the workspace process, no executor holds the credential and a session's
workspace process never runs beside that session's executor. The gap that is accepted is
between sessions: one session's workspace process and the commands of another session's run,
at the same time and as the same user. A second way was found in the maintainer's review: a
command can leave a hook or a changed configuration in its own worktree, which a later push
would run or obey with the credential. That one is not accepted. A git operation with the
credential trusts nothing in the worktree: it runs no hook, obeys no configuration a command
could have changed, and uses the registered repository as its destination.

**Alternatives considered:**

- **Running the workspace process as another user from the start.** Safer, but the worktrees
  have to be shared between that user and the one that runs the model's commands, which is not
  designed, and it would hold up the first slice.

**Consequences:** Running the workspace process at another time, or later as another user,
does not replace that rule, since git would still run what was left. A later entry says how
git is run so that it holds. Until the accepted gap changes, a leaked credential gives what that
credential can do in the registered repositories, with the maintainer's account while there is
no account for agents. The entry "Only a transfer process holds the git credential" moves the
credential, and so the process this risk is about, to a transfer process. Nothing that makes
an injection likelier should ship before the change;
closing the gap before the product reads the web, R14 of [`mvp.md`](../mvp.md), was proposed,
and when is not decided. The automated review pointed out later that an injection is not the
only way: the code that a model's commands run, such as the dependencies of a project's tests,
runs as the same user from the first slice, and that bound would not cover it.

## Pushing and opening a pull request are the user's commands, and the branch is named then

**Decision:** The product reuses how the harness commits, shows a run's changes and restores
code, and adds only what the harness never does. A branch is pushed and a pull request is
opened only when the user asks from a client; neither is ever automatic, and neither is a
command a model runs. A workspace process pushes, and only while no run of the session is
active. A worktree's branch has a provisional name with the session's id, and keeps it. At the
first push the client proposes a name, or the model does if the user asks, the user confirms
it, and the branch is published under that name, which becomes its upstream; it is never
renamed in the worktree. A session with several projects uses the same name in each. At the
first push a name the remote already has is an error the client shows; every later push goes
to the branch's upstream.
[`flows/changes-and-push.md`](../flows/changes-and-push.md) holds the flow.

**Context:** Decided by the maintainer on 2026-11-16. The harness already asks before it
commits, lists and shows a run's changes, moves a workspace through its history and refuses to
reset a branch whose upstream holds the commits; R09 of [`mvp.md`](../mvp.md) says to reuse
that. He was clear that pushing and opening a pull request will not be automatic. A command of
the client was chosen over a configured command the agent runs: it does not depend on the
model deciding to call it, and a configured command would need the credential where a model's
commands can read it. A worktree needs a branch when it is created, and then nobody knows
whether the work is a fix or a feature, which the projects' branch names say.

**Alternatives considered:**

- **Pushing after each commit.** The work would survive the server without asking, which he
  ruled out.
- **Naming the branch when the session is created.** The name is final from the start, but the
  user has to choose the kind of change before writing the first prompt.
- **Renaming the branch in the worktree at the first push.** It was the first choice. The
  harness's history ties a run's commits to the name of their branch and takes them back only
  while that branch is checked out, so after a rename a revert leaves them;
  [`learnings.md`](learnings.md) has the evidence.

**Consequences:** Work that was committed and not pushed exists only on the server. The commit
question is an ask, answered from any device. The branch has one name in the worktree and
another on the remote. A restore and a push are carried out by a workspace process, as the
entry about preparing a workspace says, and hold the workspace as a run does: after a restart
of the API one that was in progress has to be recognized, ended or reconciled before the
workspace takes any work. A later entry settles who writes a pull request's title and
description and which credential opens it.

## What happens at the edges of a run, a push and a clear

**Decision:** Each of these follows the simplest rule that keeps one account of what happened.

- A rejected approval does what the harness does: the model is told, and the run goes on.
- A cancellation of a run that already ended is answered with the run's final state, not with
  an error.
- A run that does not complete leaves no snapshot of its own: the next run starts from the
  last one stored, and its model is told that the earlier run did not complete.
- A run whose snapshot cannot be stored ends as `failed`, and the event with its answer is
  kept.
- A push is reported project by project; nothing is undone, and pushing again pushes what is
  missing.
- A restore and a push cannot be cancelled.
- The model writes a pull request's title and description, and nobody confirms them first.
- A clear changes nothing in git: each worktree stays on its branch, and the client says so.

**Context:** Decided by the maintainer on 2026-11-17, answering a list of the open points of
the flows of the first slice. For the pull request he first agreed to confirming the text and
then preferred it automatic, since he asks for a change when he sees it. For a restore and a
push, the harness and git carry them out themselves, so there is nothing to offer. For a
clear he first spoke of the client offering to update the branch, and then decided that git
stays as it is.

**Alternatives considered:**

- **A clear moves each worktree to the default branch.** The maintainer left it to
  convenience, and it was not taken. Git lets a branch be checked out in one worktree only,
  so two sessions of a project could not both be on the default branch, and commits that were
  not pushed would stay on a branch no worktree is on.

**Consequences:** The flows no longer list these as open. A session keeps one branch across
its conversations. How a project is registered is not decided.

## A fine-grained token on a private network, for the first slice

_Whose token it is is superseded by the entry "Each user brings their own credentials": each
user's own fine-grained token, not the maintainer's for everyone. The private network is
superseded by the entry "Clients reach the server over HTTPS, and multi-user security waits for
a second phase"._

**Decision:** The git credential is a fine-grained personal access token of the maintainer's
account, limited to the registered repositories, with what pushing and opening a pull request
need. The workspace process uses it to clone, fetch and push, and the API uses it to open a
pull request, which is its only use outside the workspace process. In the first slice the
server is reached only from a private network.

**Context:** Decided by the maintainer on 2026-11-17. A separate git account for agents waits
because of its cost. The API opening the pull request was in the push flow while security
named only the workspace process's uses of the credential, which the maintainer's review
pointed out.

**Alternatives considered:**

- **A GitHub App**, whose tokens last an hour and can be limited to repositories. It limits a
  leak more, and needs an app to be created and its tokens renewed.

**Consequences:** A leaked token gives what it can do in the registered repositories until it
is revoked, since it does not expire by itself as an app's would. A server that is not on the
internet leaves fewer ways to reach it. Registering a project includes giving the token access
to its repository. The entry "Only a transfer process holds the git credential" moves the
clone, the fetch and the push to a transfer process, so the workspace process no longer holds
the token.

## Git runs with the credential only in a clean repository

_The entry "A sandbox for each session" closed the risk this entry expected another user to
close: the clean repository is made in the session's sandbox, and keeping one between operations
is not a question of speed only; see the open points of `design/security.md`._

**Decision:** The workspace process gives the credential only to git commands that run in a
repository it made itself, which no command of a model has written in. To push, it brings the
session's branch into that repository from the worktree's repository, as a local path and
without the credential, and pushes from there to the registered repository. To clone or fetch,
it downloads into the clean repository, and the worktree's repository takes the commits from
it without the credential. The name a branch is published under is kept by the API and is not
read from the worktree's configuration. After a push, the workspace process records the
upstream in the worktree's repository, without the credential. A clean repository is made for
one operation and removed when the operation ends.
[`design/security.md`](../design/security.md#git-credentials) states it.

**Context:** Decided by the maintainer on 2026-11-17. The rule that a git operation with the
credential trusts nothing in the worktree had no mechanism. He took the extra step because
preventing is better than repairing and it costs little time, and will look for another way if
that time becomes a problem. The name and the upstream follow from rules already settled: a
push obeys no configuration a command could have changed, and the harness reads the upstream
to refuse a reset of commits that were pushed; the maintainer confirmed both on 2026-11-18.
The next entry moves the credential from the workspace process to a transfer process.

**Alternatives considered:**

- **Pushing from the worktree with hooks turned off and the destination named in the
  command.** It was tried and it is not enough: a setting in the repository's configuration
  still sent the push to another repository. Git has no setting that ignores a repository's
  own configuration, and overriding each dangerous setting in the command needs the list of
  all of them. The entry "Turning hooks off does not make a repository safe to push from" of
  [`learnings.md`](learnings.md) has the evidence.
- **A clean repository kept between operations**, for example one remade each day, which the
  maintainer first thought of. The workspace process runs as the writing user, so a kept
  repository could be written in by a command of the session's own run, and a later push
  would run what was left: the way this decision closes, in another folder.

**Consequences:** Each operation with the credential makes one more local copy. Git still
reads the worktree's repository when the branch is brought from it, without the credential in
that command. A clone or a fetch downloads the whole repository each time. Keeping a clean
repository between operations is looked at again when it is made and owned by a user the
model's commands do not run as, when it is a question of speed only.

## Only a transfer process holds the git credential

_Where it runs is superseded by the entry "A sandbox for each session": a transfer process runs
outside every sandbox, not as the writing user._

**Decision:** The API gives the git credential only to a transfer process: one it starts for a
single clone, fetch or push that a workspace process needs, as the writing user. A transfer
process runs git only in a clean repository and does nothing else. The workspace process is
never given the credential: it prepares a workspace, restores code, brings a branch into a
clean repository, passes on what was downloaded and records an upstream, all without it. No
process that holds the credential runs git in a worktree's repository.
[`design/security.md`](../design/security.md#git-credentials) states it, and the glossary has
the term.

**Context:** Decided by the maintainer on 2026-11-17, after the automated review of the clean
repository, and the name "transfer process" confirmed on 2026-11-18. The rule covered the git
commands that use the credential, but the workspace process held it and also ran git in the
worktree's repository: to add a worktree, to restore through the harness's history, to record an
upstream. Those commands run a hook or obey a setting such as `core.fsmonitor` that a command of
an earlier run of the same session left, and what they run is a child of a process that holds
the credential. That is the way inside one session that was not accepted, and it would still be
open after the accepted risk between sessions is closed.

**Alternatives considered:**

- **Applying the rule to every git command of the process that holds the credential.** A
  restore runs the harness's history in the worktree's repository, which the product does not
  control, so the rule could not be kept there.

**Consequences:** A clone, a fetch and a push each take one process more. The accepted risk is
about the transfer process now, which runs only for the length of a transfer. A cancellation
or a restart ends a transfer process as it ends a workspace process. The next entry says how a
workspace process and a transfer process hand over the clean repository.

## A transfer is three processes, one after another

_The entries "A sandbox for each session" and "The sandbox keeps a transfer out of a model's
reach" supersede its view of how the risk closes: the workspace steps run in the session's
sandbox, the transfer outside it, with nothing running in the sandbox meanwhile._

**Decision:** The API runs a clone, a fetch or a push as three steps, each a process it starts
and that ends before the next starts. A workspace process gets a clean repository ready: an
empty one to download into, or one with the session's branch brought in from the worktree's
repository. A transfer process, with the credential, downloads into it or pushes from it. A
workspace process finishes: it clones or fetches from the clean repository and adds a worktree,
or records the upstream of what was pushed, and removes the clean repository. They hand over
nothing but the clean repository's path, which the API names for that operation, and each tells
the API how its step ended. The API stores each step before it starts its process, and how it
ended. A clean repository has a new path each time, outside every worktree; one that a failure
or a restart left behind is removed, never reused.

**Context:** The maintainer left the design to Claude on 2026-11-17, after the entry "Only a
transfer process holds the git credential" left open how the two processes hand over the clean
repository. Every process of the product already talks only to the API, and the API already
starts each process and stores what it reports. The maintainer confirmed the design on
2026-11-18.

**Alternatives considered:**

- **One workspace process that waits while the API runs the transfer process.** It needs the
  workspace process to learn from the API when the transfer is done, and it is alive, as the
  same user, while the credential is in the transfer process.
- **The workspace process starting the transfer process itself.** The credential would pass
  through the workspace process, which runs git in the worktree's repository.

**Consequences:** No process waits for another and no two talk to each other. While the risk on
the credential is accepted, the workspace process makes the clean repository as the writing
user. Closing that risk takes more than running the transfer process as another user: that user
has to make and own the clean repository, or another session's command could plant a hook in it,
so the first step moves to that user too, and how it brings a branch in from a worktree's
repository is not designed. Within one session, the git of a worktree's repository never runs
while a process holds the credential; another session's workspace process can, which is part of
the accepted risk. After a restart the API knows which of the three steps was cut short. A
transfer starts three processes instead of one.

## Git runs on a clean repository with nothing of the user's

_The agent users it names were replaced by the entry "A sandbox for each session"; the
controlled environment stands, and the system's git is the sandbox image's, which the entry "The
sandbox keeps a transfer out of a model's reach" makes read-only._

**Decision:** Every git command on a clean repository, the transfer process's and those of the
workspace processes that make, fill and empty it, runs in an environment the API sets whole, not
inherited: no user or system git configuration (`GIT_CONFIG_GLOBAL=/dev/null`,
`GIT_CONFIG_NOSYSTEM=1`), an empty home made for that operation, a `PATH` of system
directories only, and the system's git by its full path, which the agent users cannot write.
[`design/security.md`](../design/security.md#git-credentials) states it.

**Context:** Confirmed by the maintainer on 2026-11-18, after the automated review pointed it out. A
transfer process runs as the writing user, and git reads that user's `~/.gitconfig`, where a command
of a model can leave a hooks path or an `insteadOf`. That is the way the maintainer did not accept
on 2026-11-16, and the clean repository did not close it. It follows from his rule that a git
operation with the credential obeys no configuration a command could have changed. The experiment is
in [`learnings.md`](learnings.md), under "A clean repository is not enough while git reads the
user's home". A second review the same day pointed out that the workspace process that makes the
clean repository read that home too, and an `init.templateDir` there copied a hook and a
configuration into the clean repository, which the transfer process then ran and obeyed.

**Alternatives considered:**

- **Turning hooks off in the transfer process.** Tried: a hook copied from a template no
  longer ran, but a configuration copied with it still sent the push to another repository.
- **Listing it under the accepted risk until the transfer process runs as another user.** The
  accepted risk is about another session reading the credential while a transfer runs. This
  way reaches every later push of any session, and closing it takes a few variables.

**Consequences:** Nothing in the writing user's home reaches a clean repository or a
transfer: neither its git
configuration nor a program earlier in its `PATH`. The git a transfer runs is the system's,
updated with the system. Other files git reads from a home, such as attributes, are empty in
a new home.

## The harness's local store belongs to the CLI

**Decision:** The harness is used in two ways that are not related for now: as a local library,
from the CLI, and through the API on the server. Used locally, the CLI keeps the conversations
and the configuration of agents and workflows in a store of its own, a SQLite database or a JSON
file, in harness-cli or in a package of its own that the CLI uses, never in the harness. The CLI
tells local use from use through the server. Local use runs on Linux and macOS, as the harness
does; the CLI used through the API also runs on Windows.
[harness-cli#21](https://github.com/Mikode13/harness-cli/issues/21) tracks it.

**Context:** Decided by the maintainer on 2026-11-19. The harness gives events and a snapshot,
and its consumers do with them what they need: send them through an API, store them in a
database or in a local file. The harness's own decision of 2026-11-15, that a consumer stores
what it saves ([harness#178](https://github.com/Mikode13/harness/pull/178)), keeps the store
out of it.

**Alternatives considered:**

- **A store inside the harness.** It would reverse that decision, and make the harness know
  where its consumers keep things.

**Consequences:** The API and the CLI each keep their own store, with no relation between them
for now. The harness needs the events of
[harness#177](https://github.com/Mikode13/harness/issues/177) and the snapshot of
[harness#111](https://github.com/Mikode13/harness/issues/111) for both.

## An API key for each user, kept as a hash and able to be disabled

**Decision:** A few people the maintainer knows each have a user and one key. There is no
sign-up and no password: the maintainer creates a user and its key, and disables a key that is
misused. A key is kept as a hash. What each user does and what it costs is recorded, to see
misuse and to find where the time goes. Every id a request names must belong to a session the
user owns, or is a member of once sessions have members, and the ids a client generates are
unique for each user, so the same id from two users names two things. A session has one owner;
what a user sends is an event that names its author, so that chats with several people can be
added later. [`design/security.md`](../design/security.md#an-api-key-for-each-user) and
[`design/data-model.md`](../design/data-model.md) state it.

**Context:** Decided by the maintainer on 2026-11-19, while setting the shape of the data model.
He wants to give access to a few people without letting bots or anyone else use the
infrastructure, and to switch off a key that gets out of hand. He confirmed the rule on ids the
same day, after the automated review pointed out that with several users nothing kept one from
naming another's session. He wanted a group chat with a colleague that day and could not have
one; it will be rare, so the model only keeps the door open.

**Alternatives considered:**

- **One key for the product**, decided on 2026-11-15. It cannot tell people apart, or switch
  one of them off.
- **Users with passwords**, more than a few known people need.

**Consequences:** The MVP is no longer for one person: its constraint now names a few people.
The git credential was the maintainer's account, shared by every user, until the entry "Each
user brings their own credentials"; which projects each user may work in is an open point of
the data model. The agent users of the
operating system were shared too, until the entry "A sandbox for each session" gave each
session a sandbox of its own and closed the accepted risk on the credential.

## The shape of the data model

**Decision:** Every entity has an id of its own. A user has one key, a configuration of agents
and workflows, and sessions. A session works in roots, one per project, and holds its
conversations. Events belong to the conversation, in one numbered order, and an event a run
reported names its run. A conversation has one run per prompt, and each run records the agent,
or workflow, and the model it used; how they are chosen belongs to the configuration. Ids a
client generates are unique within their user. What a run changed, its snapshot and its usage
belong to the run. The files a run changed and their diff are kept in the database, so a client
reads a run's changes from there, while undoing them stays with the harness's history next to
the worktree, as the maintainer accepted on 2026-11-14. A restore or a push is a workspace
operation with its steps. The tables belong to harness-api.
[`design/data-model.md`](../design/data-model.md) holds it.

**Context:** The maintainer described the entities on 2026-11-19 from the tables he is used to:
a user with an id, a key and an optional name, and depending on it configuration, sessions,
runs, events, plans and memories; a run with its workflow and the files it changed. Claude
fitted it to what the flows had settled: events in one order per conversation, the user's
message as an event, and a workflow as configuration that a run carries out. He answered without
objecting, which he has said means agreement, and added the user's key, one owner per session
and traceability.

**Consequences:** The conceptual model is product-level and lives here; the schema, the indexes
and the migrations live in harness-api. The engine and the format of the ids are not decided.

## A sandbox for each session

**Decision:** Everything a model can run is run in a sandbox: a container of its own for each
session. A session's executor and its workspace processes run in it, with the commands a model
starts and everything they run. A sandbox sees only its session's workspace, the worktrees
mounted read-only for an agent with no tools that write. A transfer process runs outside every
sandbox. A sandbox is rootless, never privileged and with no added capabilities, never has the
container engine's socket mounted, and has limits on processor, memory and processes. The server
keeps the maintainer's user and a service user that runs the API and starts the sandboxes and
the transfer processes. [`design/security.md`](../design/security.md#a-sandbox-for-each-session)
states it.

**Context:** Decided by the maintainer on 2026-11-19, for the MVP. With several users, the code
of one user's session ran as the same agent user as everyone else's, and could read another
user's worktrees, the processes of their runs, the keys of the model providers in them, and
the git credential during a transfer. A pair of operating-system users for each product user
was proposed first, and cost less; the sandbox was chosen because it also closes the accepted
risk on the git credential and gives a way to end everything a run started.

**Alternatives considered:**

- **A read-only and a writing user of the operating system for each user of the product.**
  Cheaper today, and it separates users, but it leaves the sessions of one user able to reach
  each other, the risk on the credential open, and the ending of a process tree to be solved.
- **A virtual machine for each session**, such as with gVisor or microVMs. It isolates more,
  since a container shares the server's kernel, and costs more. It is looked at if the product
  opens to people the maintainer does not know.

**Consequences:** It replaces the two agent users decided on 2026-11-15, the executor as a
plain child process of the API decided on 2026-11-14, and the risk on the git credential
accepted on 2026-11-16. Inside its session, a model can still damage its worktrees and read
the keys of the model providers the executor uses. The server needs Docker or Podman and an
image with what the projects need. Which of the two, whether a sandbox lives for a session or
for a run, what it mounts, its network and its image are not decided.

## The sandbox keeps a transfer out of a model's reach

**Decision:** A sandbox's image is read-only, and everything in it runs as a user that is not
root, so that only the mounted workspace, the clean repository of a transfer and a temporary
home can be written. Nothing runs in a session's sandbox while a transfer process works: before
a transfer, every process the workspace step started is ended and confirmed gone, as a run's
are, and the clean repository is mounted into the sandbox only during the workspace steps.
[`design/security.md`](../design/security.md#a-sandbox-for-each-session) states it.

**Context:** Confirmed by the maintainer on 2026-11-19, after the automated review raised both
once the workspace steps of a transfer ran in a sandbox. Without the first, a model's command
could replace the git or the code of a workspace process and leave a hook or a configuration in
a clean repository. Without the second, a process left behind, such as one a hook started,
could write in the clean repository while the transfer process uses it with the credential.
Either would reach the credential through the clean repository, which the entries "A transfer
is three processes, one after another" and "A sandbox for each session" are meant to keep out
of a model's reach.

**Consequences:** The image has to hold everything the workspace processes and the projects'
commands need, since nothing can be installed into it at run time. A transfer waits until the
sandbox is empty, which the same mechanism that ends a run's processes confirms.

## Each user brings their own credentials

**Decision:** Each user brings their own credentials: a git credential, a fine-grained personal
access token of their own account, and the keys of the model providers, such as OpenAI and
Anthropic. The API keeps them in the database, encrypted. A run uses the credentials of the user
whose session it is: the keys reach that run's executor, and the git credential only a transfer
process. [`design/security.md`](../design/security.md#each-users-credentials) states it.

**Context:** Decided by the maintainer on 2026-11-19. The automated review pointed out that,
inside its session, a model can read the keys of the model providers that the executor uses,
and that with several users those keys were the maintainer's for everyone. With each user's
own, a model reaches at most the keys of the user it works for, and a push is made as that
user. In the same message he did not object to the shape of the data model, which he has said
means agreement.

**Alternatives considered:**

- **The maintainer's keys for everyone, with the sandbox's network limited.** It narrows where
  a leaked key can be sent, but every user's session could still read the maintainer's keys.

**Consequences:** The credentials are now kept in the database rather than only held by the
API, so how they are encrypted, and where the key that decrypts them lives, has to be designed.
A user cannot work until they have given their credentials. Which repositories a user reaches
follows from their own token.

## The users' credentials are encrypted in the database, with the key outside it

**Decision:** The API keeps each user's credentials in the database, encrypted and never in the
clear. The key that decrypts them is kept outside the database, in a file on the server that
only the service user can read or in its environment. They are decrypted only when they are
used, and passed to an executor or a transfer process without being written anywhere.
[`design/security.md`](../design/security.md#each-users-credentials) states it.

**Context:** Decided by the maintainer on 2026-11-20, from Claude's proposal for the MVP, after
the entry "Each user brings their own credentials" left open how they are stored.

**Alternatives considered:**

- **A dedicated secrets manager**, such as HashiCorp Vault or a cloud provider's. It rotates keys
  and records who reads each secret, and is one more service to run; it is left for when the
  product grows.

**Consequences:** A copy of the database, or of its backups, does not give the credentials
without the key. Someone who controls the server does get them; what limits that is an open
point of security. The cipher and how its key is rotated are not decided.

## What has to survive the server's disk

**Decision:** The API runs on a VPS whose disk is not guaranteed to stay the same. The database
and the workspaces live on a persistent volume, and the database is also copied regularly
outside the server. The key that decrypts the users' credentials is kept outside the server as
well, never next to the copies of the database. Clones and worktrees are rebuilt by the next
run when they are lost. Work that was not pushed is lost with the disk, and that is accepted.
The provider is chosen before the API is deployed, and must offer persistent volumes, object
storage, full virtualization such as KVM, and a firewall.
[`design/data-model.md`](../design/data-model.md#where-it-is-kept) states it.

**Context:** Decided by the maintainer on 2026-11-20. He raised that a VPS does not guarantee
that the same disk is always used. A provider has not been chosen, and does not need to be
before the database is designed.

**Alternatives considered:**

- **A managed database outside the server.** It takes the database off the VPS's disk, and is
  one more service to pay for; the copies outside the server were preferred for now.
- **Pushing the session's branch on its own, so that work is never lost.** It goes against the
  decision that a push is only ever the user's command.

**Consequences:** Losing the whole server costs at most the work that was not pushed and the
time to restore the last copy of the database. A virtualization that is a container, as some
cheap VPS offer, is ruled out by the sandbox. How often the database is copied is not decided.

## Clients reach the server over HTTPS, and multi-user security waits for a second phase

**Decision:** Clients reach the API over HTTPS with an API key; the server opens only SSH, with
keys only, and HTTPS, and the API limits the requests of each address and key. Security is built
in two phases: at first the maintainer is the only user, and what only matters once other people
use the product waits until they do, such as each user's own credentials and their encryption,
the keys of other users, and keeping users apart. It stays in the documents.
[`design/security.md`](../design/security.md#how-the-server-is-reached) states it.

**Context:** Decided by the maintainer on 2026-11-20. He had first kept a virtual private
network, against a man in the middle taking a key, and then saw that HTTPS covers that as any
login does, and that covering every case is not the product's job. He found that the design's
complexity was growing faster than the product, with focus on unlikely risks; the risks he
keeps first are a model running commands on the server, covered by the sandbox, and reading
the web, designed in [harness-docs#5](https://github.com/Mikode13/harness-docs/issues/5).

**Alternatives considered:**

- **A virtual private network**, such as Tailscale or WireGuard, that every device joins. It
  keeps the API unreachable before authentication, and needs an app on every device.
- **Building every measure from the start.** It delays the product for users who are not there
  yet.

**Consequences:** The server needs a domain and a certificate. The private network of
2026-11-17 is superseded. The push in three processes is reviewed with the code when it is
implemented, in case it can be simpler. The roadmap places the second phase.
