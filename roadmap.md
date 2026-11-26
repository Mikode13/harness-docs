# MiKode product roadmap

This roadmap describes delivery milestones, not a promise of dates. The [MVP requirements](mvp.md) remain the source of truth for product scope; [flows](flows/README.md) describe behavior and the linked issues track implementation. Status was checked against repository documentation and merged PRs on 2026-11-26.

## Status and evidence

- **Done**: merged code or documentation evidenced by a linked PR; this does not imply an end-to-end demonstration.
- **In progress**: some required implementation is merged, while important pieces remain.
- **Not started**: no end-to-end implementation is evidenced; design or issues may exist.
- **Demonstrated**: an end-to-end acceptance scenario has been run with a recorded revision and result. None of the MVP gates below are marked demonstrated yet.

A written design, a merged implementation and a demonstrated gate are distinct outcomes. When checking status, follow the evidence links rather than inferring implementation from the scope documents.

## Milestones

| Milestone | State | Evidence and next checkpoint |
| --- | --- | --- |
| 1. Harness foundation: run agents, use tools, compose workflows and recover code | Done (foundation only) | [Harness architecture](https://github.com/Mikode13/harness/blob/main/docs/architecture.md); existing capabilities described in [README](README.md). This is not the remote product. |
| 2. Durable harness records for the API | Done (library changes) | [Agent snapshots #181](https://github.com/Mikode13/harness/pull/181), [ordered run record #182](https://github.com/Mikode13/harness/pull/182), [composed session snapshots #183](https://github.com/Mikode13/harness/pull/183), all merged. Next: verify their integration in API runs. |
| 3. API product/system design | In progress | [Run lifecycle](design/run-lifecycle.md), [data model](design/data-model.md), [security](design/security.md), and [workspace/Git](design/workspace-and-git.md) are written; [notifications #8](https://github.com/Mikode13/harness-docs/issues/8) and [web reading #5](https://github.com/Mikode13/harness-docs/issues/5) remain open. Design completion must not be confused with deployment. |
| 4. Persisted API foundation | In progress | [API PR #26](https://github.com/Mikode13/harness-api/pull/26) merged PostgreSQL-backed sessions, prompts, events and API-key access. [Delivery tracker #1](https://github.com/Mikode13/harness-api/issues/1) contains remaining work, including supervised executors and workspaces. |
| 5. Standalone CLI on macOS/Linux | In progress | [Local CLI implementation](https://github.com/Mikode13/harness-cli), [SQLite-owned session and restart issue #21](https://github.com/Mikode13/harness-cli/issues/21). Local mode does not depend on MiKode API; Windows core execution remains excluded. |
| 6. Remote CLI on Windows/macOS/Linux | Not started | [CLI #18](https://github.com/Mikode13/harness-cli/issues/18), [API contract #12](https://github.com/Mikode13/harness-api/issues/12). The remote CLI is a client; Linux executes the harness. |
| 7. First CLI-to-mobile conversation on the Linux server | Not started | [First end-to-end slice #14](https://github.com/Mikode13/harness-api/issues/14), [mobile client #13](https://github.com/Mikode13/harness-api/issues/13). Demo: prompt on CLI, disconnect laptop, view and continue on mobile. |
| 8. Full personal MVP | Not started (end-to-end gate) | All five acceptance gates in [mvp.md](mvp.md#mvp-completion-gates), tracked in [API #1](https://github.com/Mikode13/harness-api/issues/1). Partial backend work does not demonstrate a gate. |
| 9. Additional users, per-user credentials and isolation | Not started (post-MVP) | [Security phases](design/security.md#phases). This does not add multi-user requirements to the personal MVP. |
| 10. Hybrid API-owned sessions with local runner | Not started (later) | [CLI #19](https://github.com/Mikode13/harness-cli/issues/19). Separate from the standalone local CLI and remote mode. |

The order among design, API foundation and local CLI work can overlap. The critical product validation remains the first cross-device slice, followed by all five MVP gates. Mobile is represented explicitly by both its implementation issue and the cross-device milestone.

## Research track: open models and provider portability

[Open-model research](research/open-models.md) runs as a **bounded, optional experiment** alongside MVP implementation. It evaluates a hosted open-model inference service (NodeClub, subject to access and current terms), direct integrations and self-hosted runtimes such as Ollama or vLLM. It may produce interface fixes and benchmarks, but passing it is **not** an MVP completion gate.

Later possibilities, **not committed MVP deliverables**:

1. Collect consented, validated task traces and evaluation datasets.
2. Experiment with LoRA/QLoRA fine-tuning on openly licensed models, and compare the adjusted model with its base model and with non-training agent improvements.
3. Register specialized model versions and select agents/models through a capability-aware router.
4. Allow others to create personal assistants, tools and memories after the multi-user security phase.

Do not conflate model inference, memory/retrieval, agent design changes and fine-tuning. The roadmap does not promise that training improves a model, or that third-party subscription output can be reused for training.

## Maintenance

Update this file when a milestone changes status, and link a merged PR, issue or recorded end-to-end evaluation showing why. Keep future ideas labelled as research or later work. When a gate is demonstrated, record the tested revision, scenario, and result in its tracking issue before marking it **Demonstrated** here. Changes in MVP requirements belong in [mvp.md](mvp.md), not silently in this roadmap.
