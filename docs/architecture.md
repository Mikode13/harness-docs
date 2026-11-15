# harness-docs architecture

This document describes the repository's current architecture. Historical reasoning belongs
in [`decisions.md`](decisions.md); future designs are identified explicitly.

## Purpose and scope

`harness-docs` owns the design of the MiKode product where it crosses projects: the scope,
the vocabulary, the flows, the design documents and the decisions behind them. It holds no
code and nothing depends on it at run time.

It does not own how any one project is built. Each code repository describes itself in its
own `docs/`. It does not own engineering policy either, which belongs to
[`Mikode13/engineering`](https://github.com/Mikode13/engineering).

## Architectural shape

A flat set of Markdown documents, grouped by what a reader looks for. The product's
documents are at the root and in two folders; the documents about this repository itself
are in `docs/`. That is proportional to a repository whose only artifact is text.

## Responsibilities and boundaries

| Path                   | Owns                                                                            |
| ---------------------- | ------------------------------------------------------------------------------- |
| `README.md`            | The map: what the product is, its projects and how they depend on each other.   |
| `mvp.md`               | The scope: requirements, acceptance, exclusions and the design agenda.          |
| `glossary.md`          | The product's terms, and where other documents use a word differently.          |
| `flows/`               | One document per flow, with its sequence diagram; `flows/README.md` lists them. |
| `design/`              | What is decided about one area of the product, such as a run's lifecycle.       |
| `docs/decisions.md`    | Why each product decision was taken, in chronological order.                    |
| `docs/architecture.md` | This document: how the repository is organized.                                 |
| `AGENTS.md`            | How an agent uses and updates the repository.                                   |

A flow says what happens, in order, between the parts of the product. A design document
says what holds for one area whatever the flow. A document of either kind separates what is
settled from its open points.

## Dependencies and contracts

- **Links point here, never out to a project's internals.** The code repositories link to
  these documents from their `AGENTS.md`. A document here may link to a project's issue or
  to its `docs/architecture.md`, but does not describe how that project is built.
- **The harness is not a reader.** The harness does not know the product, so its documents
  do not link here. What the product needs from it is written here, in
  `design/harness-needs.md`, as something any consumer could want.
- **A change in a project that departs from a document here updates that document**, with a
  decision. `AGENTS.md` here and in each code repository states the rule.

## Constraints and trade-offs

- The documents do not travel in the pull request that changes the code, so nothing checks
  that a project still matches its flow. The rule above is the only guard.
- `docs/decisions.md` records the product's decisions, which cross several repositories.
  The documentation standard assigns a decision that crosses projects to the ADR process of
  `Mikode13/engineering`. This repository reads that rule as covering engineering policy for
  every MiKode project, and treats the product as the project that owns these decisions.
- Validation covers formatting, Markdown structure and internal links. It cannot tell
  whether a document is still true.
