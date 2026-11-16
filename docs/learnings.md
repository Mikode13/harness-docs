# Learnings

What designing, building or running the product taught: a failure, a surprise, or an
assumption that proved wrong. Each entry says what happened, with its evidence, and links
the decision or the document it changed. A lesson about one repository goes in that
repository's `docs/learnings.md`.

## A child process does not end with its parent

**What happened:** On 2026-11-15 the first version of the run lifecycle said that an executor
does not outlive the API, since it is its child, and that killing an executor ends the run's
work. Reviewing it, the maintainer reproduced on Linux that both are false: an executor can
survive the API that started it, and a command can survive its executor.

**Evidence:** The same day, on Linux 6.18, a Node.js process started a second one, which
started `sleep 300`. The first process was ended with `SIGKILL`, and in a second try with
`SIGTERM`. Both times the second process and `sleep` kept running, with PID 1 as their new
parent. The second process was then ended with `SIGKILL`, and `sleep` kept running.

**What it changed:** A run marked `interrupted` while its processes still ran would have let
the next prompt start a second run writing in the same workspace. The
[run lifecycle](../design/run-lifecycle.md#rules) now requires that everything a run started
is ended, and confirmed gone, before its conversation or its workspace takes another run,
after a restart of the API as well. The entry "A run executes in a child process of the API,
one per run" of [`decisions.md`](decisions.md) says the same. How it is done is an open
point there.

**Lesson:** Starting a process as a child gives a way to watch it while the parent lives, not
a guarantee about its end. A design that depends on a process being gone has to say who ends
it and how that is confirmed.

## Renaming a branch leaves a run's commits behind

**What happened:** On 2026-11-16 the flow for pushing renamed a worktree's branch at the first
push, from its provisional name to the one the user chose, and said that the product reuses
the harness's history as it is. Reviewing it, the maintainer found that the two do not hold
together.

**Evidence:** The harness's history records the commits of a run under the name of the branch
they were made on. Before it takes them back, it checks that this branch is still the one
checked out (`readiness`, in `src/recovery/infrastructure/gitCommits.ts` of the harness at
`7f7c6ea`). The maintainer reproduced it with the harness's own functions: after renaming the
branch, `undoCommits` with `revert` answered `left`, because the original branch was no longer
checked out. Separately, pushing a branch under another name was tried in a scratch
repository: `git push -u origin session-abc123:feat/export-button` left the local branch with
its name and set `origin/feat/export-button` as its upstream.

**What it changed:** The branch of a worktree keeps its provisional name for as long as the
worktree exists, and is published under the chosen name;
[changes-and-push](../flows/changes-and-push.md#the-name-of-the-branch) and the entry
"Pushing and opening a pull request are the user's commands, and the branch is named then" of
[`decisions.md`](decisions.md) say so. Following a rename is a candidate in
[harness-needs](../design/harness-needs.md) that the product does not need.

**Lesson:** "Reused as it is" is a claim about every step around the reused part. A change to
something the reused part keys on, here a branch's name, has to be checked against it.
