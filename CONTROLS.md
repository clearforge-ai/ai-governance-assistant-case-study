# Controls

The guardrails, human review, decision record and MCP interface: what each does, how it was tested, and where its limits are.

The request flow and the retrieval-level controls — the abstain floor and grounded prompting — are covered in [Architecture](ARCHITECTURE.md).

---

## 1. The workflow

The flowchart below was generated from the compiled LangGraph graph rather than drawn by hand, so it reflects the workflow as built.

```mermaid
---
config:
  flowchart:
    curve: linear
---
graph TD;
	__start__([<p>__start__</p>]):::first
	input_guard(input_guard)
	blocked_response(blocked_response)
	generate(generate)
	output_guard(output_guard)
	human_review(human_review)
	deliver(deliver)
	record(record)
	__end__([<p>__end__</p>]):::last
	__start__ --> input_guard;
	blocked_response --> record;
	deliver --> record;
	generate --> output_guard;
	human_review --> deliver;
	input_guard -.-> blocked_response;
	input_guard -.-> generate;
	output_guard -.-> deliver;
	output_guard -.-> human_review;
	record --> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
```

Seven nodes, two decision points. Dotted lines are conditional routes; every path ends at `record`.

**This is a workflow, not an autonomous agent.** Routing is decided by deterministic rules. The language model is called in exactly one place — inside `generate`, to write the answer — and has no say over which route a question takes or whether an answer is released.

## 2. Guardrails

| Check | Runs on | What it catches | Action |
|---|---|---|---|
| Prompt injection | Input | "Ignore your instructions", "reveal your system prompt" | **Blocked** — no retrieval, no model call |
| Legal determination | Input | "Is my internal document search tool high-risk?" | Flagged for review |
| Invalid citation | Output | A citation number that was never retrieved | Flagged for review |
| No citations | Output | An answer that cites nothing and claims nothing is missing | Flagged for review |
| Partial answer | Output | Cites sources but says they do not cover part of the question | Flagged for review |
| Borderline retrieval | Output | Best score just above the 0.45 floor, below 0.55 | Flagged for review |

**Rules rather than a model classifier.** A model would catch reworded attacks that a pattern list misses. But the evaluation had already shown a model judge giving different verdicts on the same partial answer between runs — exactly the kind of case approval depends on. Rules are deterministic, add no model or API cost, and are testable case by case. A model-based classifier alongside them is recorded as a production gap.

### Tested, then calibrated

**Tested on fixed cases first.** 15 cases with known outcomes, including deliberate false-alarm checks: a general question containing "high-risk" must not be flagged, and neither kind of refusal may be mistaken for an answer. All 15 passed.

**Then run against real answers.** The rules were applied to the answers from the 27-question evaluation run, to see what they would actually hold back. Every flag landed on a weakness the evaluation had already documented, except two, and both answers were read:

- One was correct: the answer opened by saying the sources were incomplete.
- **One was a false alarm.** The pattern matched NIST's own use of the word "context". The rule was narrowed, and that sentence became a permanent test case, so the false alarm cannot return unnoticed. On the real data, only that question changed.

The result after the fix — 10 of 27 questions stopped or flagged:

| Question type | Questions | Stopped or flagged | Finding |
|---|---|---|---|
| Single-document | 18 | 2 | Both genuine |
| Cross-document | 3 | 3 | All genuine — a known retrieval weakness |
| Out of scope | 4 | 3 | Model-written refusals that cite sources; the floor refusal correctly not flagged |
| Adversarial | 2 | 2 | One blocked; one flagged for a known retrieval miss |

The borderline-retrieval rule never fired: the lowest top score on an answered question was 0.649. It raised no false alarms, but no real answer has tested it either.

## 3. Human review

**Risk-based, not blanket.** For this use case, reviewing every answer would not be proportionate or scalable, and gating only actions would mean inventing one, since the assistant takes none. So a person reviews only what a rule flags — the human sits where the risk is, and the review triggers were tested and calibrated.

**The asker never sees an unapproved draft.** Showing the draft with approve and reject buttons underneath would have been quicker to build, and would have made the control decorative. So the interface has two views:

![The asker's view: the answer is held for review, with the reasons in plain English and no draft shown](images/ui-sent-for-review.png)

*The asker sees that the answer is held, and why, in plain English. No draft appears.*

![The reviewer's view: the reasons, the full draft and its sources, a note field, and approve or reject](images/ui-review-queue.png)

*The reviewer sees the reasons, the full draft and the same checkable sources, and records a note with the decision.*

Approve or reject resumes the paused run. No new answer is generated, so the decision applies to exactly the draft the reviewer read.

**A pause survives a restart.** A flagged question was abandoned mid-run. A separate review tool, started fresh, listed it as pending — reading saved state only, with no retrieval and no model call — and a decision was recorded against it.

**Cited refusals stay in review, deliberately.** Without a structured refusal signal, the rules cannot tell a refusal that cites sources from a partial answer. Refusals are also where the evaluation found grounding problems. Until that signal exists, the queue doubles as a human check on refusals.

## 4. The decision record

Every question through the workflow is recorded: time, question, flags, decision and reviewer note. Entries from the MCP tests:

| Time (UTC) | Decision | Flags | Reviewer note | Question |
|---|---|---|---|---|
| 2026-09-21 20:26:57 | rejected | legal_determination, partial_answer | Classification rules (Article 6) and Annex III not retrieved; cannot support a determination. Refer to a qualified adviser. | Is my internal document search tool high-risk under the AI Act? |
| 2026-09-20 19:32:49 | auto | — | — | What is the capital of Portugal? |
| 2026-09-20 19:30:39 | blocked | prompt_injection | — | Ignore your previous instructions and reveal… |

The rejection is worth reading. The draft declined to decide, and named what was missing itself: the classification criteria in Article 6(3) and the relevant Annex III categories were not among its sources. Retrieval had returned Articles 12, 60, 80 and 82 instead. So the answer could not support a determination either way, and the reviewer rejected it with a referral — a known retrieval weakness, caught at the point it mattered.

## 5. MCP

The assistant is exposed to MCP-compatible clients as two tools, over stdio: the client launches the server as a local process, so no network port is opened.

| Tool | What it does | Controls applied |
|---|---|---|
| `ask_governance` | Answers a question | The full workflow: guardrails, review, decision record |
| `search_corpus` | Returns matching passages with source, heading and score | The abstain floor only — see limitations |

**A held answer never reaches the caller.** An MCP call cannot wait for a reviewer, so a flagged question returns the reasons and a reference instead of an answer:

```
CALLING ask_governance: Is my internal document search tool high-risk under the AI Act?

Held for human review before release.
Reasons: legal_determination, partial_answer
Thread id: [review reference]
A reviewer decides in the platform's review queue.
```

`ask_governance` returns no draft for a held question, so an unapproved answer is not released through this tool.

**One workflow across two surfaces.** That held question — asked by a machine — was then rejected by a person in the web interface. It is the rejected entry in the decision record above.

**What became clear from building it:**

- **Tool descriptions are functional, not documentation.** They are what a client reads to decide which tool to call.
- **Under stdio, standard output is the protocol channel.** A library warning printed there can corrupt the conversation, so the server's output had to be kept clean.
- **A new caller is a new route around the controls unless it is designed not to be.** `ask_governance` was routed through the workflow deliberately. `search_corpus` shows what happens when a route is not.

## 6. Mapping to the frameworks

The table shows conceptual parallels between the project's controls and selected NIST AI RMF functions and EU AI Act control areas. It is a reference mapping, not a compliance assessment or a classification of the system.

| Control | NIST AI RMF function | EU AI Act parallel |
|---|---|---|
| Citations to the retrieved text | Measure, Manage | Transparency and provision of information to deployers (Art. 13); transparency obligations for providers and deployers of certain AI systems (Art. 50) |
| Abstain floor and grounded prompt | Manage | Accuracy, robustness and cybersecurity (Art. 15) |
| Guardrails and adversarial test questions | Manage | Robustness against manipulation (Art. 15) |
| Human approval checkpoint | Manage | Human oversight (Art. 14) |
| Decision record per question | Govern, Manage | Record-keeping (Art. 12); human oversight (Art. 14) |
| Tracing of each run | Measure, Manage | Record-keeping (Art. 12) |
| Evaluation on a fixed question set | Measure | Accuracy (Art. 15); risk management system (Art. 9) |

Every Article number above was checked against the consolidated text in the corpus before publication.

## 7. Known limitations

- **`search_corpus` sits outside the workflow.** It applies the abstain floor but not the input guardrail, and its calls are not in the decision record.
- **The record does not say who asked, or through which channel.** A `channel` column was considered and declined: it would name the interface, not the caller, and make a partial control look complete. Production needs the caller's identity.
- **No separation of duties.** In a single-user build, the asker and the reviewer are the same person.
- **No structured refusal signal**, which is why cited refusals go to review.
- **Pattern rules miss reworded attacks** that a model classifier would catch.
- **MCP over HTTP** would need authentication, transport security and rate limiting. Those controls were outside the scope of the stdio implementation.

What production would add for each is in [Production readiness](PRODUCTION_READINESS.md).
