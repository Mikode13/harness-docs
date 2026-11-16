# Security

Who may use the product, and as which user of the operating system each part of it runs. The
maintainer closed these two points on 2026-11-15. The rest of the product's security is not
designed; the open points say what is missing. None of it is built yet.

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

| User            | Runs                                              | Can                                                 |
| --------------- | ------------------------------------------------- | --------------------------------------------------- |
| Service user    | The API                                           | Use the database, and start the executors.          |
| Read-only agent | The executor of an agent with no tools that write | Read a workspace.                                   |
| Writing agent   | The executor of an agent with tools that write    | Do more operations, such as writing in a workspace. |

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

- **The API holds the credential and passes it to the executor**, which prepares the
  workspace; see [create-session](../flows/create-session.md).
- **For now it is the maintainer's own account.** A separate git account only for agents is
  the aim, and waits because of its cost.
- **The credential is never written to disk** nor to the git configuration of a worktree,
  and the executor uses it only to clone, fetch and push. That narrows where it can be read.
- **The credential reaches only the registered repositories.**

### An accepted risk, for now

A command a model starts runs as the same user as the executor that holds the credential.
Keeping the credential off disk does not keep it from such a command: it could read it from
the executor's process, or through what it inherits. A prompt injection could make a model
start one.

The maintainer accepted this on 2026-11-16 for the first slice, limited by a credential that
reaches only the registered repositories. It is a known gap, not the design: it will be
closed by running the git operations that need the credential as a user the model's commands
do not run as.

## Open points

Not decided.

- **Where the server can be reached from.** Whether it is exposed to the internet or only
  to a private network.
- **How the service user starts a process as another user.** The API runs as the service
  user and its executors run as the agent users.
- **When the accepted risk on the git credential ends, and how.** Closing it before the
  product reads the web (R14 of [mvp.md](../mvp.md)) was proposed. How the git operations
  run as another user, with the worktrees shared between both users, is not designed.
- **Which git credential, while it is the maintainer's account.** A fine-grained personal
  access token limited to the registered repositories, or a GitHub App whose tokens last an
  hour, were proposed to limit what a leaked credential can reach.
- **What exactly the writing agent may do** beyond writing in its workspace. The maintainer
  described it as the user "with more capability to do operations".
- **A threat model.** The server holds the keys of the model providers and git credentials,
  runs commands a model starts, which the harness trusts and does not sandbox, and, with
  R14 of [mvp.md](../mvp.md), reads content from the public web. The MVP names only
  authentication. A threat model and the isolation of what an executor runs, such as a
  container for each session and limits on what it can reach, were proposed and not
  discussed.
