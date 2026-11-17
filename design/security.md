# Security

Who may use the product, and as which user of the operating system each part of it runs. The
maintainer closed these two points on 2026-11-15. The rest of the product's security is not
designed; the open points say what is missing. None of it is built yet.

## A private network

In the first slice the server is reached only from a private network, not from the internet.
The maintainer decided it on 2026-11-17. It also leaves fewer ways for the accepted risk on
the git credential, below, to be used.

## One API key

The API accepts one key of the product's own, and only that key works. The maintainer's
devices share it. It exists so that nobody else can use the harness through the server.

The MVP is for personal use, with one user; see the constraints in [mvp.md](../mvp.md).

A key per device was proposed, so that a lost mobile could have its key revoked alone. The
maintainer declined it: losing a device and having that key taken is not a concern for him,
so one key is enough.

With one key, the API cannot tell one device from another by its key. It does not need to:
an ask is not sent to a device, and a device gets events because its subscription is open.
See [ask-and-answer](../flows/ask-and-answer.md).

## Users of the operating system

The server has four users. The maintainer has his own, with sudo. The product runs as three
others:

| User            | Runs                                                                                               | Can                                                 |
| --------------- | -------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| Service user    | The API                                                                                            | Use the database, and start the executors.          |
| Read-only agent | The executor of an agent with no tools that write                                                  | Read a workspace.                                   |
| Writing agent   | The executor of an agent with tools that write, every workspace process and every transfer process | Do more operations, such as writing in a workspace. |

- **The operating system enforces what the harness already does.** Whether an agent writes
  is known before its run starts: it has tools that write, or it has none. An executor of an
  agent with none runs as the read-only user, so a write is refused by the operating system
  as well as by the harness.
- **Neither agent user has sudo.**
- **Neither agent user can read the database credentials.** The executor does not connect to
  the database; see [where a run executes](run-lifecycle.md#where-a-run-executes).
- **A workflow runs as one user.** A workflow with a planner, an executor and a reviewer is
  one process, so all of it runs as the writing user, the roles that only read included.

## Git credentials

- **The API holds the credential and passes it only to a transfer process**, never to an
  executor nor to a workspace process; see [create-session](../flows/create-session.md). A
  workspace process prepares a workspace, restores code and pushes, and runs as the writing
  user. When it needs a clone, a fetch or a push, the API starts a transfer process for that
  one step, as the writing user too, with the credential.
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
  - **The name a branch is published under comes from the API**, which keeps it from the
    first push. It is not read from the worktree's configuration.
  - **A clean repository is made for one operation and removed when the operation ends.**
    One kept between operations would be on disk while the session's runs work, as the same
    user, and a command could leave a hook in it for a later push. While it exists, a
    command of another session could write in it; such a command can already read the
    credential from the process, which is the accepted risk below.
- **The credential is a fine-grained personal access token**, limited to the registered
  repositories, with what pushing and opening a pull request need.
- **No process that holds the credential runs git in a worktree's repository.** A workspace
  process runs git there, to add a worktree, to restore code through the harness's history
  or to record an upstream, and those commands run what a command of a model left: a hook,
  or a setting such as `core.fsmonitor`. So the workspace process never holds the
  credential, and a transfer process runs nothing but its step in the clean repository.
- **The API uses the same credential to open a pull request**, which is a call to the git
  host and touches no workspace. That is its only use outside a transfer process.

### An accepted risk, for now

A command a model starts could reach the credential in two ways. Only the first is accepted.

- **By reading it from a transfer process.** Only a transfer process holds the credential,
  and it works for a session only while that session has no executor. But the transfer
  process of one session and the commands a model starts in another run at the same time,
  as the same user, so such a command could read the credential from that process. Keeping
  the credential off disk does not prevent it. This is the accepted risk.
- **By leaving something for git to run.** A command could install a hook or change the
  configuration in its own session's worktree, and a later git command would run or obey it
  in a process that holds the credential. This is not accepted: the rules above close it,
  since git runs with the credential only in a clean repository and no process that holds
  it runs git in a worktree's repository.

The maintainer accepted the first on 2026-11-16 for the first slice, limited by a credential
that reaches only the registered repositories. It is a known gap, not the design: it will be
closed by running the transfer process as a user the model's commands do not run as. A
prompt injection could make a model start such a command. So could code that a model's
commands run without any injection: a project's tests and tooling, and their dependencies,
run as the writing user from the first slice.

## Open points

Not decided.

- **How the service user starts a process as another user.** The API runs as the service
  user and its executors run as the agent users.
- **How a workspace process and a transfer process hand over a clean repository**: how the
  API starts the transfer process while the workspace process waits, and how each learns
  that the other is done.
- **Whether a clean repository is kept between operations** once the processes that use it
  run as a user the model's commands do not run as. It is then a question of speed only: a
  clone or a fetch downloads the whole repository each time.
- **When the accepted risk on the git credential ends, and how.** Closing it before the
  product reads the web (R14 of [mvp.md](../mvp.md)) was proposed. That bound covers an
  injection from the web, and not the code that a model's commands run, such as the
  dependencies of a project's tests, which is there from the first slice. How the workspace
  process runs as another user, with the worktrees shared between both users, is not
  designed.
- **What exactly the writing agent may do** beyond writing in its workspace. The maintainer
  described it as the user "with more capability to do operations".
- **A threat model.** The server holds the keys of the model providers and git credentials,
  runs commands a model starts, which the harness trusts and does not sandbox, and, with
  R14 of [mvp.md](../mvp.md), reads content from the public web. The MVP names only
  authentication. A threat model and the isolation of what an executor runs, such as a
  container for each session and limits on what it can reach, were proposed and not
  discussed.
