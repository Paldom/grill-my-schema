# Grilling question bank

The full interrogation catalog, organized by axis. Each question names the
failure it exposes — ask the question, then push on the answer with the
failure. Evidence base: `serving-layer-facts.md`. Don't recite the bank;
select and sequence per `SKILL.md`.

**Contents:** [Ten fatal-flaw questions](#ten-fatal-flaw-questions) ·
[A Access patterns](#a-access-patterns) · [B Modeling and read models](#b-modeling-and-read-models) ·
[C Keys](#c-keys) · [D Indexing and pagination](#d-indexing-and-pagination) ·
[E Sync, freshness, consistency](#e-sync-freshness-consistency) ·
[F Read/write separation and writeback](#f-readwrite-separation-and-writeback) ·
[G Evolution, versioning, contract](#g-evolution-versioning-contract) ·
[H Tenancy, security, PII, compliance](#h-tenancy-security-pii-compliance) ·
[I Capacity and cost](#i-capacity-and-cost) · [J Failure modes](#j-failure-modes) ·
[K Observability](#k-observability) · [L Ownership and process](#l-ownership-and-process) ·
[M Commonly-missed aspects](#m-commonly-missed-aspects)

## Ten fatal-flaw questions

The ten that most often expose a design that cannot ship — a wrong answer
means redesign, not polish. Open with the ones the provided context leaves
most exposed.

1. **Show me the exact query each top screen fires — filters, sorts, joins,
   page size — and why it is one indexed read at real volume.** No inventory
   → the schema is designed from the data backward, and every later decision
   is a guess. Follow-up: what is the grain of each table (one row = what,
   as of when)?
2. **Where do the app's own writes live, and when the sync also supplies a
   field the app edits — who wins, per column, written where?** Writes
   touching replicated tables are lost-write bugs shipping on day one;
   unwritten merge rules are clobber bugs shipping on day two.
3. **Is every ID the app persists, bookmarks, or puts in a URL stable across
   a full gold rebuild, dedupe, or replay?** Regenerated surrogate keys
   silently orphan every app-owned row, favorite, and deep link. And what
   happens to rows with null or duplicate keys — managed syncs drop or fail
   on them, silently in the worst case.
4. **Can a user ever read a half-synced, cross-table-inconsistent state
   (order present, customer missing)?** What is the atomic unit of sync —
   row, table, or batch — and is the swap atomic across tables a screen
   joins?
5. **What is the freshness SLA per table, end to end — and what does the UI
   show when it is breached?** The slowest hop is usually the upstream gold
   refresh, not the sync; no breach behavior means the app confidently lies.
6. **How does a breaking change (rename, retype, regrain) reach production —
   and who finds out, contract CI or the pager?** No expand/contract plus
   stable-seam answer → the first warehouse refactor is an outage.
7. **One forgotten tenant predicate: what does the database itself prevent?**
   Replicated tables often can't carry row-level security — if upstream
   doesn't scope rows, nothing does. Is the tenant key in every unique
   constraint and leading every hot index?
8. **Which UI numbers are governed metrics, and can serving-layer SQL ever
   disagree with the semantic layer's definition?** Recomputed KPIs drift →
   the app and the BI dashboard show two truths, and users notice.
9. **Which PII lands in the serving copy, why, and how does erasure reach
   it — including refresh artifacts and backups?** "We delete in the
   warehouse" leaves a non-compliant replica.
10. **Who owns this schema, who is paged at 3am when the numbers are wrong,
    and what document would a new team read to learn what each table
    promises?** No owner + no document = the contract is tribal memory.

## A Access patterns

- Which screens/endpoints exist, and what exact query does each issue
  (filter columns, sort, joins, page size, expected result count)?
- Which five queries dominate by frequency? By latency sensitivity?
- What p95 latency does each class of screen need? What concurrency at peak?
- Any screen that aggregates at request time? Why isn't it precomputed?
- Any list-then-per-row-fetch (N+1) shape hiding in the UI design?
- Which queries are search-shaped (fuzzy, multi-facet)? Are they in scope for
  this store or a search engine's job?
- Which screen breaks first at 10× data volume, or on the largest tenant?
- Is there any query here that exists only because gold happens to be shaped
  that way?

## B Modeling and read models

- Is the serving model derived query-first (from the inventory) or copied
  model-first from gold? What would change if you started from the queries?
- Which tables are denormalized read models, which are canonical entities —
  and why is each the shape it is?
- What is the stated grain of each table? Who checked it against the UI
  (cards vs line items vs rollups)?
- Current state or history: per entity, which does the app actually need?
  Where does history live if a screen needs point-in-time?
- Which displayed numbers are governed metrics from the semantic layer, and
  what stops serving-layer SQL from recomputing them differently (two
  truths)?
- What is inside each JSON column, who enforces its shape, and why isn't it
  typed columns? (Blobs are contract evasion.)
- If a screen needs a join across more than two large tables, why isn't
  there a pre-joined projection upstream?
- Are you syncing whole gold tables when the app uses a handful of columns?

## C Keys

- What is the primary key of each serving table? Is it stable, non-null,
  unique in the source — and who guarantees that upstream?
- Do keys the app persists (bookmarks, URLs, app-owned foreign keys) survive
  a full gold rebuild, dedupe, replay, or re-mastering? What happens when
  the same real-world entity carries two source IDs for 48 hours (merge/
  split)?
- Are foreign keys enforced in the serving store? If yes, can sync ordering
  violate them mid-batch; if no, what detects orphans?
- What happens to rows with null keys? Duplicate keys? (Know your sync's
  behavior: drop, fail, or dedup-by-timeseries-key.)
- Do source/natural keys survive into the serving copy for traceability?
- Any key built on mutable attributes (email, phone)?
- How do app-owned rows reference replicated rows — and does that reference
  survive a full re-sync/rebuild?

## D Indexing and pagination

- For each hot query: which index serves it, and does the column order match
  (equality first, range/sort last)? Verified with a query plan, or assumed?
- How do long lists paginate? OFFSET at depth is linear cost and drifts under
  concurrent writes — where is keyset pagination with a composite cursor?
- Which JSON fields will be filtered on? Why are they not typed columns?
- What is the index budget — each index taxes every sync write; which query
  justifies each one?
- Total counts on lists: exact COUNT(*) per request (death at scale),
  estimated, precomputed, or forbidden — and is the UI honest about which?
- A user holds page 5 open while a refresh swaps the table underneath —
  duplicates, gaps, or error?

## E Sync, freshness, consistency

- Per table: what freshness SLA, in seconds/minutes, did the *product* agree
  to — and what does the full path (gold refresh + sync lag) deliver?
- Is the sync source incremental-capable (physical table with change feed),
  or a view/MV that silently degrades to snapshot full refreshes?
- Is the sync full-refresh or incremental, and what fraction of rows change
  per cycle? (High churn can make snapshot cheaper; low churn makes it
  wasteful.)
- How do deletes propagate? Late-arriving data? Out-of-order updates?
- What is the atomic unit of sync — row, table, batch? Can two tables a
  screen joins be visibly out of step with each other?
- How do historical restatements in gold reach users who already saw (or
  exported) the old number?
- Full rebuild from zero: how long does it take, and what does the app serve
  meanwhile?
- What consistency does the UI assume across tables synced independently —
  can a screen show an order whose customer hasn't landed yet?
- Read-your-writes: after the app writes its own state, does any screen mix
  that state with replicated data in a way users will read as inconsistent?
- Is there a `data_as_of` column (or equivalent) so freshness is visible to
  the API and UI?

## F Read/write separation and writeback

- Exactly which tables can the app write? Are they physically separate from
  replicated tables (schema/ownership), or just separate by convention?
- Could any process write to a replicated table? What breaks silently when
  the next refresh overwrites it?
- Does any app state need to flow back to the lakehouse (for analytics or
  ML)? Via which channel — CDC/export — and who owns it?
- Is any column owned by both sides (pipeline writes it, app updates it)?
  If users can edit a synced field, where is the per-column merge rule
  written — and who wins on the next refresh?
- App-owned writes: idempotency keys for retries/double-clicks, and
  optimistic locking (version/etag) against reads that were stale to begin
  with?

## G Evolution, versioning, contract

- Where is the contract written — tables, columns, types, grain, PK,
  freshness SLA, latency SLO, PII classification, owner, compatibility
  window? Who signs changes off?
- Does the app query stable views/aliases over replicated tables, or raw
  table names it can never escape?
- Walk me through your next breaking change (a rename, say): what are the
  expand, migrate, contract steps, and which deploys do they span?
- Which changes does your sync tolerate in place (usually additive only) and
  which break it? Does the team know the list?
- Who runs migrations for app-owned tables, with what tooling, and is that
  tooling firmly kept away from the pipeline-owned tables?

## H Tenancy, security, PII, compliance

- Which tenancy model — shared-schema+tenant key, schema-per-tenant,
  database-per-tenant — and what requirement (isolation, cost, tenant count,
  per-tenant export) drove it?
- What enforces isolation when application code forgets a filter? RLS on
  app-owned tables? Upstream row filtering for replicated ones?
- Which columns are PII? Which of them does the UI actually render? Why do
  the rest land in the serving copy at all?
- How does an erasure request reach the serving copy (and its refresh
  artifacts, and its backups)? Within what deadline — and has legal/privacy
  signed off the workflow, exemptions included?
- What stops a re-sync, CDC replay, or restore from resurrecting an erased
  subject — durable tombstones, suppression lists, or nothing?
- What privileges does the app's DB principal hold? Who can read the
  serving database besides the app?
- Any regulated-data overlay (health, financial, minors) changing the answers
  above?

## I Capacity and cost

- Rows and bytes per table today; growth over 12–24 months; hot working set
  vs the serving tier's memory?
- What does the sync itself cost per month at the chosen cadence, and what
  would the next-cheaper cadence break?
- Retention in the serving copy: current-state only, shallow window, or
  unbounded accretion (why)?
- What is the cost of the tail — the biggest table, the most frequent sync,
  the widest screen?

## J Failure modes

- Sync breaks at 3am: what do users see at 9am, and who got paged?
- Defined behavior for stale data: banner, degraded mode, or fail closed —
  chosen per screen or left to chance?
- Full-refresh window: double storage, deleted rows transiently present —
  does any compliance or capacity assumption break during it?
- Upstream emits garbage (nulls in keys, duplicate keys, truncated loads):
  what stops it from reaching users — propagate, quarantine, or block?
- Blast radius of one pathological app query: statement timeout, row limit,
  or nothing between a bad filter and a downed store?
- A restore from backup rewinds app-owned writes along with replicated data:
  reconciled how?
- Restore story: serving DB lost — rebuild from source, or restore from
  backup? How long? Who tested it?

## K Observability

- Which four dashboards exist on day one: sync lag per table, freshness age
  per table, query p95 per endpoint, connection/pool saturation?
- Is freshness measured as `now() - max(data_as_of)` (data-level), or
  inferred from pipeline success (a green pipeline can deliver stale data)?
- What detects silent divergence — row counts or checksums against gold —
  when nothing errored?
- Which alerts page, and are their thresholds the SLA numbers from the
  contract or round numbers someone liked?
- Can you trace a wrong value on a screen back through the serving table to
  the gold row that produced it?

## L Ownership and process

- Name the owning team for: gold sources, sync config, serving schema,
  app-owned schema, the contract document. Any "us, sort of" answers?
- What is the release choreography when a change spans gold + sync + app?
- Who reviews new tables/indexes in the serving layer, against what
  checklist?
- If two apps want this layer, does each get its own contract/schema, or
  will this one silently become shared infrastructure?

## M Commonly-missed aspects

- **Naming constraints of the target engine** (case sensitivity, reserved
  words, allowed characters) leaking warehouse names into the app.
- **Type-mapping surprises** at the OLAP→OLTP boundary: timestamps gaining/
  losing timezone semantics, decimals/floats, complex types landing as JSON.
- **Search vs lookup conflation** — faceted/fuzzy search forced into B-tree
  indexes.
- **Soft-delete semantics** — does the sync propagate source deletes, and do
  partial indexes exclude soft-deleted rows?
- **Idempotency/dedup keys for app-owned writes** (double-click, retry).
- **Connection pooling** for serverless/multi-instance app deployments.
- **Cold start / empty state** — first sync of a big table vs launch day.
- **Environments** — how dev/staging get realistic serving data without
  copying production PII.
- **The second consumer** — the moment BI or another app starts querying
  "the app's" serving tables, the contract silently gains consumers.
- **Exports and reports routed through the OLTP path** — accidental full
  scans on the serving store's connection pool.
- **Hot-key skew** — the celebrity tenant or entity that concentrates reads
  on one row range.
- **Compound staleness** — app-side caches stacked on top of sync lag.
- **Collation and search expectations** — case-insensitive matching or
  fuzzy search (`ILIKE`, trigram) no planned index supports.
