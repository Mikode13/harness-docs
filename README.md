# harness-docs

The design of the MiKode harness product, across its projects.

## The product

MiKode is one personal place for conversations with agents. The user starts work from a CLI,
turns off the laptop while execution continues on a server, and continues the same
conversation from a mobile app. The user can change agents or supported models without
copying the conversation history between products. [`mvp.md`](mvp.md) holds the scope and the
requirements.

The end-to-end product has not been built yet. These documents define the target; each
project's own `docs/architecture.md` describes what that project is today.

| Project                         | Available today                                                        | Still planned                                                                  |
| ------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `harness`                       | Agent SDK adapters, a model API tool loop, workflows and code recovery | Durable conversation snapshots and the product's required observation contract |
| `harness-cli`                   | Local interactive and single-turn execution through Agent SDKs         | Server mode and an independent local conversation store                        |
| `harness-api`                   | An HTTP scaffold with `GET /health`                                    | Agent execution, authentication, persistence and client events                 |
| Contract package and mobile app | Not created                                                            | Shared event types and a mobile client                                         |

The MVP is for the maintainer's personal use. The
[security design](design/security.md) separates its requirements from a later phase for
onboarding other users and managing their credentials. The presence of those entities in the
[data model](design/data-model.md) does not make that later phase an MVP prerequisite.

## The projects

This is the target dependency graph, not the dependencies installed today. An arrow means
"depends on".

```mermaid
flowchart TD
    cli[harness-cli] --> contract[Contract package]
    mobile[Mobile app] --> contract
    api[harness-api] --> contract
    contract --> harness[harness]
    api --> harness
    cli -.-> harness
```

The dotted arrow is the CLI used locally: it runs agents on the user's machine through the
harness without the API. Local execution exists; its own conversation store is planned.
It will stay alongside the CLI's use through the API, with independent histories; see [harness-cli#21](https://github.com/Mikode13/harness-cli/issues/21).

- `harness` (`@mikode13/harness`, [github.com/Mikode13/harness](https://github.com/Mikode13/harness)):
  the library that runs agents. It knows nothing about any other project.
- The contract package (not created yet, name not decided): only types, for the product's
  state events. It depends on the harness. The API and the clients depend on it.
- `harness-api` ([github.com/Mikode13/harness-api](https://github.com/Mikode13/harness-api)):
  the server. It will depend on the harness and on the contract package. Today it is a
  health-only scaffold with neither dependency.
- `harness-cli` ([github.com/Mikode13/harness-cli](https://github.com/Mikode13/harness-cli)):
  the terminal client. It currently depends on the harness and runs agents locally.
  Its own conversation store is planned
  ([harness-cli#21](https://github.com/Mikode13/harness-cli/issues/21)). Through the server it
  will depend on the contract package and talk to the API
  ([harness-cli#18](https://github.com/Mikode13/harness-cli/issues/18)).
- The mobile app (not started; [harness-api#13](https://github.com/Mikode13/harness-api/issues/13)):
  a client, like the CLI.
- `harness-docs`: this repository. Nothing depends on it; the other repositories link to it.
- `engineering` ([github.com/Mikode13/engineering](https://github.com/Mikode13/engineering)):
  cross-project engineering standards. It is how the projects are built, not what the product
  is.

Every dependency points towards the harness, and the harness depends on none of them.

## Reading the design

Start with [the MVP scope](mvp.md), [product roadmap](roadmap.md) and
[glossary](glossary.md), then use the
[flow index](flows/README.md) to follow one user action. The [data model](design/data-model.md)
shows the conceptual entities; physical tables, indexes and migrations belong to the API.
[Harness needs](design/harness-needs.md) identifies library capabilities the product still
requires.

[Security](design/security.md) and [workspace and Git operations](design/workspace-and-git.md)
describe the execution boundaries and their remaining open points. Web search and reading are
still being designed in [issue #5](https://github.com/Mikode13/harness-docs/issues/5); the proposed
fetching and summarization components are not implemented or a settled security guarantee.

## Where a document lives

A document lives with the owner of what it describes. The test: if the document would still
be true after deleting a repository, it does not belong to that repository.

Product-level documents are here: scope, glossary, flows, design, and the decisions and
learnings that cross projects. Each code repository keeps its own `docs/architecture.md`,
`docs/decisions.md` and `docs/learnings.md`, about how that repository is built, and links
here, never the reverse. The harness's documents never describe the product.

## Layout

- [`mvp.md`](mvp.md): the scope and the requirements of the MVP.
- [`roadmap.md`](roadmap.md): evidenced milestones, delivery states and future research.
- [`research/open-models.md`](research/open-models.md): bounded open-model and provider evaluation plan.
- [`glossary.md`](glossary.md): the words the product uses for its parts.
- `flows/`: one file per flow, with its diagram; [`flows/README.md`](flows/README.md) lists
  them.
- `design/`: the design documents: [`run-lifecycle.md`](design/run-lifecycle.md),
  [`events.md`](design/events.md), [`security.md`](design/security.md),
  [`harness-needs.md`](design/harness-needs.md), [`data-model.md`](design/data-model.md) and
  [`workspace-and-git.md`](design/workspace-and-git.md), which puts together how a session's
  workspace and git work.
- [`docs/decisions.md`](docs/decisions.md): the product's decisions, which cross its
  projects.
- [`docs/learnings.md`](docs/learnings.md): what designing, building or running the product
  taught, with its evidence.
- [`docs/architecture.md`](docs/architecture.md): how this repository itself is organized.
- [`AGENTS.md`](AGENTS.md): how an agent uses and updates this repository.

## Checks

```sh
pnpm install --frozen-lockfile
pnpm run docs:check
```

The checks cover formatting, Markdown structure and internal links. External URLs are not
checked.

## License

The documentation in this repository is licensed under the
[Creative Commons Attribution 4.0 International License](LICENSE) (CC BY 4.0). The software
projects it describes have their own licenses.
