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

## Running at another time does not separate what was left behind

**What happened:** On 2026-11-16 the documents said that the risk on the git credential was
only between sessions, because a session's workspace process never runs beside that session's
executor. Reviewing it, the maintainer pointed out a way inside one session: a command the
agent starts installs a `pre-push` hook, and when the user later pushes, git runs that hook
from the process that holds the credential.

**Evidence:** In a scratch repository on Linux, a `pre-push` hook was written into
`.git/hooks` and a push was run with a variable in its environment standing for the
credential. The hook ran and read the variable. The same push with
`-c core.hooksPath=/dev/null` did not run it. Setting `remote.origin.pushurl` in the
repository's configuration changed where a push to `origin` goes.

**What it changed:** [security](../design/security.md#git-credentials) now has a rule of its
own: a git operation with the credential trusts nothing in the worktree. The entry "A model's
command can reach the git credential: accepted for now, to be changed" of
[`decisions.md`](decisions.md) says that this way is not part of the accepted risk, and that
changing the user would not close it.

**Lesson:** Two pieces of work that never run together can still reach each other through
what the first leaves for the second to run. A boundary in time, or between users, needs a
rule about the files both touch.

## Turning hooks off does not make a repository safe to push from

**What happened:** On 2026-11-17 the rule that a git operation with the credential trusts
nothing in the worktree had no mechanism. Turning hooks off and naming the registered
repository in the command looked like enough, and was tried before it was written down.

**Evidence:** In scratch repositories on Linux, with git 2.43, a repository standing for a
worktree's was given what a command could leave in it: a `pre-push` hook, a changed
`remote.origin.pushurl`, a `url.<other>.insteadOf` for the registered repository's address,
and a `credential.helper`. A push from it with `-c core.hooksPath=/dev/null` and the
registered repository's address in the command ended without an error, with the branch in the
other repository and not in the registered one: `insteadOf` had rewritten the address. Then an
empty repository was made, the branch was fetched into it from the first one as a local path,
and it was pushed from there to the same address. The branch reached the registered
repository and the hook did not run. The helper was not put to the test, since a local
address asks for no credential. An upstream written afterwards into the first repository,
with `git update-ref` and `git branch --set-upstream-to`, is what the harness's check reads.

**What it changed:** [security](../design/security.md#git-credentials) says that git runs
with the credential only in a clean repository, and
[changes-and-push](../flows/changes-and-push.md#pushing-and-opening-a-pull-request) that a
push runs from one. The entry "Git runs with the credential only in a clean repository" of
[`decisions.md`](decisions.md) records it.

**Lesson:** A repository's configuration is input to every git command that runs in it.
Where that input cannot be trusted, the command runs somewhere else; switching off the parts
known to be dangerous leaves the ones nobody listed.

## A clean repository is not enough while git reads the user's home

**What happened:** On 2026-11-18 the automated review pointed out that a transfer process runs
git as the writing user, so git reads that user's own configuration as well as the clean
repository's. A command of a model runs as the same user and can write there.

**Evidence:** In scratch repositories on Linux, with git 2.43, a home standing for the writing
user's was given a `.gitconfig` with a `core.hooksPath` to a `pre-push` hook and a
`url.<other>.insteadOf` for the registered repository's address. A push from a clean
repository, with that home, ended without an error, with the branch in the other repository,
and the hook ran and read the variable that stood for the credential. The same push started
with `env -i`, `PATH=/usr/bin:/bin`, an empty home, `GIT_CONFIG_GLOBAL=/dev/null`,
`GIT_CONFIG_NOSYSTEM=1` and `/usr/bin/git` reached the registered repository, and no hook
ran.

**What it changed:** [security](../design/security.md#git-credentials) says that a transfer
process reads no configuration but the clean repository's and runs in an environment the API
sets. The entry "A transfer process runs git with nothing of the user's" of
[`decisions.md`](decisions.md) records it.

**Lesson:** Git's configuration is not only the repository's. "Trusts nothing a command could
have changed" covers every file git reads, and the user's home is one a command can change.
