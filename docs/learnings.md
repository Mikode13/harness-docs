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
