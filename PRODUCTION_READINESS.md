# Production readiness

This is a working proof of concept, built and tested by one person for one user. This document sets out what I would change before production, in what order, and the evidence from this build behind each item. It is not a general checklist: everything here comes from what was built, tested or observed.

## What the proof of concept establishes — and what it doesn't

**Established:** the pipeline works end to end; the controls were tested on every path through the controlled workflow and calibrated against real answers; quality was measured against a fixed test set and a change was measured against it.

**Not established:** behaviour with real users, real load, a wider range of attacks, or a corpus that changes over time. Nothing here was run for longer than the build itself.

## Priority 1 — before any real user

The gaps that could release an answer nobody approved, or leave no record of who asked.

| Area | In the PoC today | Before production | Evidence from the build |
|---|---|---|---|
| Identity and access | One user; no public access | Single sign-on, HTTPS, rate limiting | Access was restricted to the owner, so no user authentication was built |
| Who asked | The decision record holds the question and outcome, not the person or channel | The caller's identity on every record | A channel column was considered and rejected: it names the interface, not the caller ([Controls](CONTROLS.md#7-known-limitations)) |
| Separation of duties | The asker and the reviewer are the same person | Distinct authenticated roles, a reviewer response target, escalation when the queue ages | Every review in the build was done by the person who asked |
| The search tool | `search_corpus` applies the relevance floor but bypasses the input guardrail and the decision record | Route it through the workflow, or restrict it | The tool calls retrieval directly rather than the workflow ([Controls](CONTROLS.md#5-mcp)) |
| MCP transport | stdio, so no network port | If remote clients are needed: HTTP with authentication, transport security and rate limiting | stdio was chosen so none of this was needed for a single user |
| Credentials | Held in local configuration | A secrets manager and scoped tokens | The evaluation service token used in the PoC was not scoped to least privilege |
| Data, if the corpus changes | Public regulation, so no personal or confidential data | If confidential or personal data is introduced, assess self-hosted tracing, provider data residency and a data protection impact assessment against the data and its risk | Traces contain the full prompt, including every retrieved passage |

## Priority 2 — before relying on the answers at scale

The weaknesses the evaluation found. Each fix would be measured against the test set before it was kept.

| Area | In the PoC today | Before production | Evidence from the build |
|---|---|---|---|
| Retrieval | Semantic search, top five chunks | Evaluate query rewriting, chunk enrichment and per-document retrieval for comparison questions | The penalties question missed Article 99 under four retrieval methods; comparison questions retrieved only NIST; an Article 50 question retrieved only NIST ([Architecture](ARCHITECTURE.md#47-what-was-learned-about-retrieval-quality)) |
| Refusal signal | Detected by rule for fixed refusals, by a judge where the model wrote one | A structured flag from the assistant: answered, partial or refused | The judge changed its verdict on the same partial answer between runs ([Evaluation](EVALUATION_AND_OBSERVABILITY.md#5-one-change-measured)) |
| Guardrail coverage | Pattern rules, tested on 15 cases and calibrated on 27 answers | Evaluate additional semantic or model-based guardrails alongside the rules, validated before adoption | Rules miss reworded attacks; one false alarm was found only by running the rules over real answers |
| Leak detection | LeakFree, validated on one positive case | More validation cases, then scoring on sampled live answers | A subtler leak — one outside fact in an otherwise clean refusal — is untested |
| Evaluation as a release gate | The test set runs on demand | Runs on every change, with a noise baseline for each metric | Noise was measured for answer correctness but not for relevancy, so a 4-point relevancy change could not be interpreted |

## Priority 3 — to run it as a service

Service and operational maturity. Some of it is needed before any production use; how far each goes depends on usage and criticality.

| Area | In the PoC today | Before production |
|---|---|---|
| Deployment | A single server, configured manually | A container platform, infrastructure as code, a deployment pipeline |
| Resilience | No redundancy | Backups of the vector index and the decision record; the review queue moved from a single-file database to a server database |
| Corpus management | One consolidated text, ingested once | Scheduled checks for amendments, re-ingestion, answers stating which version of the text they used, and the abstain floor re-measured after any corpus or model change |
| Cost and latency | Visible per question in the traces | Budgets and alerts. Generation varied from 2.55 to 43.67 seconds across eight test questions |
| Monitoring | A trace per question; a record per decision | Live quality scoring on sampled traffic, drift detection, user feedback feeding the test set |
| Operating model | One person | A named owner, a support model, an incident process, reviewer staffing, and user training |

## What I would not do first

- **Optimise the vector database.** In the traces measured, generation took most of the time. Nothing in the latency evidence pointed at retrieval speed.
- **Add step-by-step tracing of the workflow.** The decision record already holds each question's route and outcome; identity is the bigger gap.
- **Make the workflow more autonomous.** The controls work because routing is deterministic and the model only writes answers. More autonomy would need the foundational controls first, then its own risk assessment and evaluation.

## Why this order

Priority 1 closes the gaps where an answer could be released without approval, or with no record of who asked — the failures a governance tool can least afford. Priority 2 improves what the evaluation showed was weak, measured against the same test set, so nothing is kept because it merely looks better. Priority 3 scales with usage and can follow demand.
