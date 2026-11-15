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

## Open points

Not decided.

- **Where the server can be reached from.** Whether it is exposed to the internet or only
  to a private network.
- **How the service user starts a process as another user.** The API runs as the service
  user and its executors run as the agent users.
- **Git credentials, and who prepares the workspace.** Cloning a repository and adding a
  worktree need credentials. Credentials limited to that repository, used by the executor,
  were proposed. It is open in the [run lifecycle](run-lifecycle.md#open-points).
- **What exactly the writing agent may do** beyond writing in its workspace. The maintainer
  described it as the user "with more capability to do operations".
- **A threat model.** The server holds the keys of the model providers and git credentials,
  runs commands a model starts, which the harness trusts and does not sandbox, and, with
  R14 of [mvp.md](../mvp.md), reads content from the public web. The MVP names only
  authentication. A threat model and the isolation of what an executor runs, such as a
  container for each session and limits on what it can reach, were proposed and not
  discussed.
