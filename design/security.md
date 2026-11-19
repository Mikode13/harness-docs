# Security

Who may use the product, as which user of the operating system each part of it runs, where
the server can be reached from, and how the git credential is kept from what a model runs.
The maintainer closed the users of the operating system on 2026-11-15 and replaced the agent
users with a sandbox for each session on 2026-11-19, an API key for each
user on 2026-11-19, the private network on
2026-11-17, and the git credential on 2026-11-16, 2026-11-17 and 2026-11-18; the sections say
which. The rest of the product's security is not designed; the open points say what is
missing. None of it is built yet.

## A private network

In the first slice the server is reached only from a private network, not from the internet.
The maintainer decided it on 2026-11-17. It leaves fewer ways to reach the server.

## An API key for each user

Each user has one key, and only the keys of users work. They exist so that nobody else, a bot
included, can use the product's infrastructure through the server. The maintainer decided it
on 2026-11-19, replacing the one key of 2026-11-15.

- **A few people the maintainer knows have a user each.** There is no sign-up and no
  password: the maintainer creates a user and its key.
- **A key is kept as a hash**, never as it was given.
- **A key can be disabled.** A key that is misused is disabled, and its requests are rejected.
  What each user does is recorded so that misuse can be seen; see
  [data-model](data-model.md#traceability).
- **A user's devices share that user's key.** A key per device was proposed, so that a lost
  mobile could have its key revoked alone, and declined: losing a device is not a concern for
  the maintainer.

The API cannot tell one device of a user from another by its key. It does not need to:
an ask is not sent to a device, and a device gets events because its subscription is open.
See [ask-and-answer](../flows/ask-and-answer.md).

## A sandbox for each session

Everything a model can run is run inside a sandbox: a container of its own for each session.
The maintainer decided it on 2026-11-19 for the MVP. It replaces the two agent users of the
operating system that every session shared, decided on 2026-11-15, under which the code of one
user's session could read another user's worktrees, the processes of their runs and the git
credential.

- **A session's executor and its workspace processes run in the session's sandbox.** So do
  the commands a model starts and everything they run, such as a project's tests and their
  dependencies.
- **A sandbox sees only its session's workspace.** Its worktrees with what they need of their
  repository, and, during a transfer, the clean repository of that
  transfer. Not the server's files, not another session's worktrees, and not the processes of
  the server or of other sandboxes.
- **An agent with no tools that write gets the worktrees mounted read-only.** The operating
  system refuses a write, as well as the harness, as the read-only user did before.
- **A transfer process runs outside every sandbox**, on the server, with the git credential.
  No code of a model runs there.
- **It is configured as a sandbox, not only as a container**: rootless, never privileged and
  with no added capabilities; the container engine's socket is never mounted into it; it mounts
  only the session's workspace and, during a transfer, its clean
  repository; and it has limits on processor, memory and processes.

What it does not protect, and is accepted: inside its session, a model can still damage its
own worktrees and read the keys of the model providers that the executor uses. A container
shares the server's kernel, so it isolates less than a virtual machine; for a few people the
maintainer knows that is enough, and something stronger is looked at if the product opens
to people he does not know.

### Users of the operating system

The server has the maintainer's own user, with sudo, and a service user that runs the API and
starts the sandboxes and the transfer processes. The service user can use the database; no
sandbox can read the database credentials, and the executor does not connect to the database;
see [where a run executes](run-lifecycle.md#where-a-run-executes).

## Git credentials

- **The API holds the credential and passes it only to a transfer process**, never to an
  executor nor to a workspace process; see [create-session](../flows/create-session.md). A
  workspace process prepares a workspace, restores code and pushes, in the session's sandbox.
  When it needs a clone, a fetch or a push, the API starts a transfer process for that one
  step, outside every sandbox, with the credential.
- **For now it is the maintainer's own account.** A separate git account only for agents is
  the aim, and waits because of its cost.
- **The credential is never written to disk** nor to the git configuration of a worktree,
  and a transfer process uses it only to clone, fetch and push. That narrows where it can be
  read.
- **A git operation with the credential trusts nothing in the worktree.** A command a model
  started could have left a hook there, or changed the repository's configuration: where it
  pushes to, a helper that is handed the credential. So such an operation runs no hook,
  obeys no configuration a command could have changed, and uses the repository registered
  for the project as its destination. Running it at another time, or as another user, does
  not replace this: git would still run what was left.
- **So git runs with the credential only in a clean repository.** The workspace process
  makes a repository of its own, which no command of a model has written in, and a transfer
  process uses the credential only there. Turning hooks off and naming the
  destination in the command is not enough; see
  [learnings](../docs/learnings.md#turning-hooks-off-does-not-make-a-repository-safe-to-push-from).
  - **To push**, the workspace process brings the session's branch into the clean
    repository from the worktree's repository, as a local path, and a transfer process
    pushes it from there to the registered repository.
  - **To clone or fetch**, a transfer process downloads the commits into the clean
    repository, and the workspace process passes them on to the worktree's repository as a
    local path.
  - **Every git command on a clean repository reads no configuration but that repository's.**
    The workspace processes that make, fill and empty it run in the session's sandbox, whose
    home a command of a model can write in, and the transfer process reads it afterwards. A
    `~/.gitconfig` with a hooks path or an `insteadOf` would reach a push from a clean
    repository, and an `init.templateDir` would copy a hook and a configuration into the clean
    repository when it is made. So each of those git commands runs in an environment the API
    sets whole, not inherited: no user or system git configuration
    (`GIT_CONFIG_GLOBAL=/dev/null`, `GIT_CONFIG_NOSYSTEM=1`), an empty home made for that
    operation, a `PATH` of system directories only, and the system's git by its full path, which
    no code in a sandbox can write. The git a workspace process runs in a worktree's repository
    needs none of this, since no process then holds the credential. See
    [learnings](../docs/learnings.md#a-clean-repository-is-not-enough-while-git-reads-the-users-home).
  - **The API runs the three steps one after another, each a process of its own.** A
    workspace process gets the clean repository ready and ends; a transfer process does
    the transfer and ends; a workspace process finishes and ends. No two of them run at
    the same time and none waits for another, so the git of a worktree's repository never
    runs while a process of that session holds the credential, and no code of another
    session can reach that process; see the sandbox above. They hand over nothing but the clean
    repository's path, which the API names for that operation, and each tells the API how
    its step ended. The API stores each step before it starts its process, and how it
    ended.
  - **The name a branch is published under comes from the API**, which keeps it from the
    first push. It is not read from the worktree's configuration.
  - **A clean repository is made for one operation and removed when the operation ends**,
    whether the transfer succeeded or not. It has a new path each time, outside every
    worktree, and is never used again: one that a failure or a restart left behind is
    removed by the next step that touches the workspace, never reused.
    It is mounted only into its session's sandbox, so no other session can write in it. One
    kept between operations would be there while the session's runs work, and a command
    could leave a hook in it for a later push.
- **The credential is a fine-grained personal access token**, limited to the registered
  repositories, with what pushing and opening a pull request need.
- **No process that holds the credential runs git in a worktree's repository.** A workspace
  process runs git there, to add a worktree, to restore code through the harness's history
  or to record an upstream, and those commands run what a command of a model left: a hook,
  or a setting such as `core.fsmonitor`. So the workspace process never holds the
  credential, and a transfer process runs nothing but its step in the clean repository.
- **The API uses the same credential to open a pull request**, which is a call to the git
  host and touches no workspace. That is its only use outside a transfer process.

### The accepted risk, closed by the sandbox

A command a model starts could reach the credential in two ways.

- **By reading it from a transfer process.** On 2026-11-16 the maintainer accepted this for
  the first slice: the transfer process of one session and the code a model left in another
  session ran at the same time, as the same user, so that code could read the credential from
  that process. It could be a command a model starts, a project's tests and their
  dependencies, or a hook that the session's own workspace process runs. The sandbox closes
  it: a transfer process runs outside every sandbox, and no code in a sandbox can see the
  server's processes.
- **By leaving something for git to run.** A command could install a hook or change the
  configuration in its own session's worktree, and a later git command would run or obey it
  in a process that holds the credential. This was never accepted: git runs with the
  credential only in a clean repository, and no process that holds it runs git in a
  worktree's repository.

The sandbox is decided and not built. Until it is, nothing that runs a model's commands on
the server ships without it.

## Open points

Not decided.

- **Proposed, to confirm: a model's commands cannot change the sandbox's own programs.** The
  image's filesystem would be read-only, and everything in the sandbox would run as a user
  that is not root inside it, so that only the mounted workspace, the clean repository of a
  transfer and a temporary home can be written. Otherwise a command could replace the git or
  the code of a workspace process to leave a hook or a configuration in a clean repository,
  which the transfer process would then run or obey with the credential. Raised by the
  automated review on 2026-11-19.
- **Proposed, to confirm: nothing runs in the session's sandbox while a transfer process
  works.** Before a transfer, every process the workspace step started would be ended and
  confirmed gone, as a run's are, and the clean repository would be mounted into the sandbox
  only during the workspace steps. Otherwise a process left behind, such as one a hook
  started, could write in the clean repository while the credential is in use. Raised by the
  automated review on 2026-11-19.

- **Docker or Podman**, and whether a sandbox lives for a session or for a run. The mounts
  differ between an agent that writes and one that does not.
- **What a sandbox mounts**, since a worktree needs its repository: with one clone for each
  repository shared by its sessions, mounting it would show other sessions' branches.
- **The network of a sandbox.** By default a container reaches the internet and could send
  code or keys out. Limiting it to the model providers and the package registries was
  proposed.
- **The image of a sandbox**: what it holds, such as Node, pnpm and git, and how a project
  that needs another runtime is served.
- **The user a transfer process runs as**, outside every sandbox.
- **Whether a clean repository is kept between operations.** A clone or a fetch downloads the
  whole repository each time, so keeping one would be faster. It is not a question of speed
  only: a clean repository is made and filled in its session's sandbox, so one that is kept
  could get a hook or a configuration from a command of a later run, which the next transfer
  would run or obey with the credential. Keeping one needs a place that no sandbox can write
  in.
- **A threat model.** The server holds the keys of the model providers and git credentials,
  runs commands a model starts, which the harness trusts and does not sandbox, and, with
  R14 of [mvp.md](../mvp.md), reads content from the public web. The sandbox of each session
  is decided; a threat model of the whole is not.
