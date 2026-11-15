# AGENTS.md

## Repository purpose

`harness-docs` holds the design of the MiKode product across its projects: scope, glossary,
flows, design documents, and the decisions and learnings that cross projects. It holds no
code. [`README.md`](README.md) maps the projects and the layout of this repository.

## Read before designing or implementing

Before designing or implementing anything in any of the projects, read
[`README.md`](README.md), [`glossary.md`](glossary.md), and the flow or design document for
the area you work on. Open only those; do not load every document by default.

- Use the glossary's terms. Add a term to the glossary before using a new one.
- A change in any project that departs from a flow or a design document updates that
  document in the same batch of work, and adds an entry to
  [`docs/decisions.md`](docs/decisions.md)
  that says why. An entry has **Decision**, **Context**, **Alternatives considered** when
  there were any, and **Consequences**.
- `docs/learnings.md` records what building or running the product taught, and links the
  decision it led to. It is created with its first lesson.
- [`docs/architecture.md`](docs/architecture.md) describes how this repository is organized.
  Update it when the layout or a rule about where a document lives changes.

## Writing

- Write as settled only what the maintainer confirmed. A proposal goes under "Open points",
  worded as one.
- Draw diagrams as Mermaid in the Markdown, so GitHub renders them and a diff shows their
  changes.
- Use English, plain declarative sentences, and relative links between documents here.
- Do not invent decisions, future artifacts or implementation details. Describe a design that
  is not built as not built.

## Consistency

A change is coherent when all of these hold. A review checks them.

- A flow or a design document does not contradict the glossary, the run lifecycle or another
  document.
- Nothing is written as settled in one document and listed as an open point in another.
- Every term a document uses for a part of the product is in the glossary.
- A change to what is settled has its entry in `docs/decisions.md`.
- [`flows/README.md`](flows/README.md) lists every flow with the status its document has.

## Where a document lives

A document lives with the owner of what it describes. The test: if the document would still
be true after deleting a repository, it does not belong to that repository. Product-level
documents are here. A code repository's `docs/architecture.md`, `docs/decisions.md` and
`docs/learnings.md` describe how that repository is built and link here, never the reverse.
The harness's documents never describe the product.

The repository follows the standards in
[`Mikode13/engineering`](https://github.com/Mikode13/engineering/blob/main/standards/README.md).
Do not duplicate their content here.

## Local validation

```sh
pnpm install --frozen-lockfile
pnpm run docs:check   # prettier --check, markdownlint, internal link check
```

A change is complete when the check passes and the documents it affects agree with each other.
