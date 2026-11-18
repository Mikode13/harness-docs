# Flows

A flow document describes one thing the product does from end to end, across its projects:
who takes part, what each one sends and in which order, and what is left when it ends. Each
has a sequence diagram, the decisions it rests on and the points that are still open. The
terms are in the [glossary](../glossary.md).

A flow shows the path where nothing goes wrong. What happens when a run is cancelled, fails
or is interrupted is in the [run lifecycle](../design/run-lifecycle.md), and each such case
gets a flow of its own below.

How a session's workspace and git work across the flows, from preparing it to pushing, is put
together in [workspace-and-git](../design/workspace-and-git.md).

None of this is built yet.

## The flows of the first slice

| #   | Flow                                                     | Status                                                      |
| --- | -------------------------------------------------------- | ----------------------------------------------------------- |
| 1   | Send a prompt, execute and reconnect from another device | Written: [prompt-and-reconnect.md](prompt-and-reconnect.md) |
| 2   | Ask and answer from another device                       | Written: [ask-and-answer.md](ask-and-answer.md)             |
| 3   | Cancel                                                   | Written: [cancel.md](cancel.md)                             |
| 4   | Restart of the server                                    | Written: [restart.md](restart.md)                           |
| 5   | Create a session and set up its workspace                | Written: [create-session.md](create-session.md)             |
| 6   | Clear and delete                                         | Written: [clear-and-delete.md](clear-and-delete.md)         |
| 7   | See changes, restore code and push the branch            | Written: [changes-and-push.md](changes-and-push.md)         |
| 8   | Authenticate a device                                    | Written: [authenticate.md](authenticate.md)                 |

Flows beyond the first slice, such as changing the agent or model, delegation, memory and
plans, are designed after these.

## Order of work

The maintainer set the order on 2026-11-15:

1. Finish the flows of the first slice.
2. Design the database.
3. Finish what the harness needs; [harness-needs.md](../design/harness-needs.md) collects it
   as each flow shows it, so the list is ready when this step comes.
4. Build the API.
