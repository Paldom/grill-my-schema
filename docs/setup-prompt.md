# Setup prompt: the grill → design → review loop

Paste the block below as a `/goal` in a Claude Code session inside **your
application's repo** (with this skills repo installed). Fill the `<...>`
placeholders first. It drives the two skills in their intended order —
interrogation until the fatal questions are answered, then artifact review
until nothing Critical/High remains — and never touches git.

The loop in one line: `/grill-my-schema` (questions → decisions/gaps/risks)
→ *you/the team design the schema* → `/serving-schema-review` (findings →
fixes) → re-review until the gate passes.

```
/goal Drive the application-serving schema for <APP> to review-clean. Work in docs/serving-schema/ of this repo. NEVER run git commit or git push - leave every change for me to review.

Context: UI/screens at <PATH-OR-LINK>; upstream gold/semantic layer described at <PATH-OR-LINK>; target serving store and sync mechanism: <STORE/SYNC, e.g. "managed Postgres, vendor sync" - or "undecided">.

Phase 1 - GRILL (gate A). Invoke /grill-my-schema with the context above. Answer what you can from the linked material; put every question you cannot answer to me in batches. Persist the outputs as docs/serving-schema/decision-log.md, gap-list.md, risk-register.md, updating them each round. Gate A passes when: every one of the skill's ten fatal-flaw questions has a written decision or an owner+date in the gap list, and the access-pattern inventory (every screen/endpoint with filters, sorts, page sizes, freshness expectation) exists as docs/serving-schema/access-patterns.md. Do not start Phase 2 while gate A fails.

Phase 2 - DESIGN (mine, not yours). I will produce the schema artifact (DDL/ERD/contract) at docs/serving-schema/schema.sql (or schema.md). You may restate decisions from the log when I ask, but do not generate the schema yourself - schema generation is out of scope for these skills by design. When the artifact lands, proceed.

Phase 3 - REVIEW (gate B). Invoke /serving-schema-review on the artifact plus access-patterns.md and the decision log. Persist the report as docs/serving-schema/review-<n>.md: severity-ranked findings (Critical/High/Medium/Low, each with failure scenario + fix), a separate evidence-gap list, kept-right notes, and a verdict. If the review exposes an undecided design axis (no freshness SLA, no write-path plan), reopen Phase 1 for that axis only. I will fix the artifact; then re-run the review as review-<n+1>.md. If the schema has independent domains, review disjoint domains in parallel subagents only if their findings write to separate review files.

Gate B / Definition of Done: latest review has zero open Critical or High findings (each earlier one fixed, or rejected with written evidence in the review file); every evidence gap is either supplied or accepted with a named owner in risk-register.md; decision-log/gap-list/risk-register are current; zero git commits were made. Finish with a summary listing every file you changed and the top 3 residual risks.
```

## Why this shape

- **Ordering constraint:** gate A before Phase 2 — reviewing an artifact
  whose design questions were never answered produces findings about the
  wrong thing; the review skill itself hands such cases back to the grill.
- **Verifier gates, bracketed:** each phase ends with an explicit,
  checkable condition (gate A: fatal-flaw questions decided + inventory
  exists; gate B: no open Critical/High), verified against files on disk,
  not memory.
- **Parallelism only on disjoint surfaces:** reviews of independent schema
  domains may fan out because each writes its own `review-*.md`; everything
  else is sequential by design.
- **No git actions:** the owner commits. The prompt states it twice because
  goal-driven sessions drift.

## Command fact-check

| Command in the prompt | Ships in this repo as |
| --- | --- |
| `/grill-my-schema` | [`skills/grill-my-schema/`](../skills/grill-my-schema/) — outputs prioritized questions, decision log, gap list, risk register; ten fatal-flaw questions live in its `references/question-bank.md` |
| `/serving-schema-review` | [`skills/serving-schema-review/`](../skills/serving-schema-review/) — outputs severity-ranked findings, separate evidence-gap list, kept-right notes, verdict |

Both skills also activate on plain descriptions of the task; the explicit
slash forms above make the phases deterministic (and are required in
headless `claude -p` runs, which don't auto-trigger skills).
