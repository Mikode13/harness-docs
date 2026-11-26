# Open-model research: provider portability and specialization

**Status:** Proposed experiment; not yet executed. **Scope:** Research alongside the personal MVP, not an MVP gate. **Updated:** 2026-11-26.

## Why

MiKode should not be tied to one model provider. The short-term hypothesis is that a hosted or self-hosted open-weight model can support at least some harness tasks at acceptable quality and predictable cost. The longer-term vision is a personal-agent platform where tools, context, memory, workflows and specialized models can be combined independently.

Research must not delay the [personal MVP](../mvp.md). Keep one primary delivery stream and a bounded secondary research stream. No production provider or hardware purchase is decided by this document.

## First experiment: hosted open-model inference

**Candidate:** [NodeClub](https://nodeclub.ai), subject to confirming that API access, model names, pricing, context limits, concurrency, fair-use rules, data retention and commercial terms are suitable. Do not assume its advertised subscription provides unlimited concurrent runs or fine-tuning.

**Starting point:** integrate a provider through the harness's direct LLM abstraction, not through `harness-api` business logic. Compare against an existing known-working provider. Prefer a real, already planned harness PR/task as the evaluation workload, and record its commit and environment.

### Questions to answer

1. Can the harness call the provider directly, and what API compatibility or adapter is needed?
2. Are tool calls, structured arguments, error handling, context length, cancellation and multi-turn state supported reliably?
3. What is the time to first token (where observable), total run duration, throughput, failure rate and cost under realistic workloads?
4. Does the model complete the same tasks as the existing provider? Record tests, review findings and human acceptance, not just plausible prose.
5. What happens under two or more concurrent runs, rate limiting, outages and retries? Avoid unbounded retry loops.
6. Can an agent change provider through configuration without changing tool/workflow domain code? Record supported capability differences rather than hiding them.
7. Which data and credentials leave the machine, and may traces or responses legally be stored, published or used for training?

### Reproducible test cases

| Scenario | What to measure |
| --- | --- |
| Minimal conversation + one tool call | Model/endpoint compatibility, exact tool arguments, latency |
| Small deterministic code change | Test pass rate, changes made, human review, total tokens and duration |
| Review an existing PR | Valid issues, missed issues, false positives, source attribution |
| Planner to executor to reviewer workflow | Delegation, context handoff, retries, tool-calling reliability |
| Concurrent runs and transient provider failure | Resource contention, errors, safe fallback and cancellation |
| Provider swap during a persisted conversation | Conversation continuity, cache/context changes and adapter leakages |

Use pinned model identifiers, software revisions, identical tasks and tool permissions, a reproducible environment, multiple runs when feasible, and a reserved evaluation set not used for tuning. Measure the full workflow, not just tokens/second.

### Decision record for an experiment

Record: date, task and commit, model identifier and quantization, provider/runtime version, prompts and tool versions, context length, hardware/region when known, number of repetitions, wall-clock time, successful completions, correctness evidence, total cost and limits, and any privacy/licensing constraint. Clearly separate measured results from provider claims.

**Go/no-go:** keep the provider available as an *experimental* option if it reliably completes the selected tasks and gives a meaningful cost/capability trade-off. Otherwise document why and stop; do not refactor the whole harness to accommodate a poor-performing endpoint.

## Second experiment: directly served open models

Compare at least one Qwen-family and one Gemma-family open-weight model (select actual revisions and verify licences at test time) using a local runtime such as Ollama or a remote vLLM server. Evaluate on the same harness scenarios. Test the model's tool-calling behavior, long-context needs, memory use, latency and Linux deployment dependencies.

An inference server can run separately from the API and the executor. Inference is not the same as running shell commands or worktrees: keep the latter inside the harness execution/security boundary. A self-hosted runtime is also not automatically a training service.

## Later: model specialization

Fine-tuning is a subsequent experiment only after evaluation and data collection work:

1. Collect verified, permissioned examples (accepted changes, tests, reviewer feedback and human corrections). Remove secrets and personal data.
2. Check model licenses and provider terms before using third-party traces or generated outputs as training data.
3. Establish a base-model baseline and hold-out tests. Distinguish improvements due to prompting, tools, memory/RAG or routing from those due to changed weights.
4. Trial LoRA/QLoRA on a small model and a narrowly scoped task (for example, code review). Train separate candidate adapters only when evidence supports specialization.
5. Re-run untouched evaluations; promote a version only if quality improves without unacceptable regressions. Retain rollback and provenance.

Avoid a continuous self-training loop without validation. Model versions and datasets should be reproducible and independently evaluated. Models used for sensitive advisory domains require extra domain-specific safeguards and should not be assumed competent merely because they can be fine-tuned.

## Boundaries and next actions

- **Now:** document benchmarks, verify NodeClub terms/access, and test one external open-model endpoint via the LLM interface; keep costs and time capped.
- **Next:** compare Qwen/Gemma in local or rented GPU inference after the first results.
- **Later:** training infrastructure, datasets, specialized adapters, model registry and router integration.
- **Not now:** requiring NodeClub, Ollama, any paid plan, a GPU purchase, multi-user hosting, or model fine-tuning for MVP acceptance.

Evidence should be linked from [the product roadmap](../roadmap.md) when experiments have results. Architectural changes to the harness or API must be recorded in their owning repository and, when they cross product boundaries, in this repository's [decisions](../docs/decisions.md).
