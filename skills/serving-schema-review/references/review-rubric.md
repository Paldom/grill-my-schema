# Serving-schema review rubric

Checks for reviewing a concrete app-serving schema (DDL, ERD, table defs, or a
data contract) built over an analytics gold layer. Evidence for the checks:
`serving-layer-facts.md`. Severity bands: **Critical** = wrong results, lost
writes, leaked data, or broken sync in production; **High** = will not survive
production load or the first schema change; **Medium** = degrades latency,
cost, or operability; **Low** = hygiene.

**Contents:** [How to apply](#how-to-apply) · [1 Keys and grain](#1-keys-and-grain) ·
[2 Access-pattern fit](#2-access-pattern-fit) · [3 Types and time](#3-types-and-time) ·
[4 Read/write separation](#4-readwrite-separation) · [5 Sync and freshness](#5-sync-and-freshness) ·
[6 Evolution and contract](#6-evolution-and-contract) · [7 Tenancy, security, PII](#7-tenancy-security-pii) ·
[8 Capacity and cost](#8-capacity-and-cost) · [9 Failure modes and observability](#9-failure-modes-and-observability) ·
[10 Ownership](#10-ownership) · [Severity calibration examples](#severity-calibration-examples)

## How to apply

Run every section against the artifact. A check you cannot evaluate from the
artifact goes to the **separate evidence-gap ("unverifiable") list** — name
the missing evidence (access-pattern inventory, EXPLAIN plan, sync config,
contract doc), don't guess; evidence gaps are never mixed into the
severity-ranked findings. Report only what the artifact shows or omits;
never fabricate a column or index you haven't seen. Release gate: open
Critical and High findings block "ship"; an artifact cannot be called
production-ready while workload, sync-config, or security evidence is
absent.

## 1 Keys and grain

- Every table declares a primary key; PK columns are NOT NULL. Managed sync
  products silently drop null-PK rows and fail pipelines on duplicates —
  key hygiene is an upstream obligation, so ask where uniqueness is enforced.
- The grain of each table is stated (one row per what?) and consistent with
  its name; parent/child tables (e.g. `orders_read` vs `order_lines_read`)
  don't mix grains.
- Natural keys from source systems survive into the serving copy (traceability
  back to gold); surrogate keys added where source keys are messy.
- **Key stability across rebuilds:** any ID the app persists, bookmarks, or
  puts in URLs must survive a full re-sync/replay of the source. Surrogate
  keys regenerated on rebuild silently orphan app-owned references — ask how
  keys are minted and whether a replay preserves them.
- Foreign keys: enforced in the store, or consciously omitted (sync-ordering
  argument) with a named orphan-detection backstop — one or the other, stated.
- Unique constraints match business uniqueness *including* the tenant key and
  soft-delete state (a unique on the bare business key is a cross-tenant
  collision waiting to happen).
- No PK built on mutable business attributes (email, name).

## 2 Access-pattern fit

- An access-pattern inventory exists: every screen/endpoint with its filters,
  sorts, joins, page size. Without it, index review is guesswork — request it.
- Every hot filter/sort column is a typed column with a supporting index
  (composite order matching the query: equality columns first, range/sort
  last). One index per hot path; no "index everything".
- Hot-path request-time joins across >2 large tables are flagged as a risk
  to be proven, not assumed fatal: ask for `EXPLAIN`/load-test evidence at
  realistic volume; where composition is shown inadequate (or consistency/
  availability requirements justify it), recommend a denormalized read model
  built upstream (see facts: CQRS read models, denormalization evidence).
- List views use keyset pagination (`WHERE (sort_key, id) < (:cursor)` +
  matching index), not `OFFSET`, for anything deep or infinite-scrolling.
- No N+1 shape: list screens resolvable in one query.
- Aggregations the UI shows on every load are precomputed upstream, not
  computed per request.
- Total-count policy for lists is explicit: exact `COUNT(*)` per request
  (unaffordable at scale), estimated, precomputed, or forbidden — and the UI
  is honest about which.
- No unbounded `IN` lists or `SELECT *` in the query contract; row width of
  wide read models is known.
- Governed metrics shown in the UI trace to the semantic layer's definition;
  serving-layer SQL that re-derives a KPI differently is a finding ("two
  truths").

## 3 Types and time

- Timestamps: UTC, timezone-aware type (`timestamptz` in Postgres family),
  never a mix of with/without-timezone semantics in one table.
- Hot filters do not live inside JSON columns. JSON is for variable,
  rarely-filtered payloads; anything in a WHERE/ORDER BY/join is a typed
  column. (In Postgres, GIN indexes accelerate containment `@>`/`?`, not
  `->>` equality — that needs an expression index or a generated column.)
- Money in exact numeric types; no FLOAT for currency.
- Text columns that encode enums have a CHECK constraint or lookup table,
  matched to the semantic layer's allowed values.
- Nullability is deliberate: columns the UI always renders are NOT NULL, and
  each nullable column's null has one documented meaning (unknown / not
  applicable / not yet synced).
- JSON columns carry a schema-version field inside the payload if their shape
  evolves.

## 4 Read/write separation

- **Critical check:** no app write path targets a replicated/synced table.
  The failure mechanism depends on the sync product's documented semantics —
  silently overwritten on the next refresh (full-refresh targets), rejected,
  or left to diverge — but every variant is Critical; establish which one
  applies from the sync's docs, and mark it unverifiable if unknown.
  App-owned state (annotations, assignments, preferences, workflow status)
  lives in separate writable tables the app fully owns.
- The join between app-owned rows and replicated rows is by a stable key that
  survives re-syncs.
- No row is owned by both sides (a column the pipeline writes and the app
  also updates = conflict by design). Where a user-editable field shadows a
  synced field, a per-column merge rule is written down — who wins on the
  next refresh.
- App-owned writes carry optimistic-locking columns (`version`/`updated_at`
  etag) and idempotency keys where retries/double-clicks are possible.
- If app state must flow back to the lakehouse, there is an explicit writeback
  channel (CDC/export/outbox — transactional with the domain write), not
  writes into the replica.

## 5 Sync and freshness

- Each table has a stated freshness SLA, and the sync mode/cadence can
  actually meet it. End-to-end freshness = upstream gold refresh + sync lag;
  a seconds-level sync under an hourly gold job still serves hour-old data.
- Incremental freshness claims are backed by an incremental-capable source
  (physical table with change feed) — view/MV-backed syncs silently degrade
  to snapshot full refreshes on many platforms.
- Freshness is observable: a `data_as_of`/`_synced_at` column (or equivalent
  metadata) exists and the UI can surface staleness. Freshness is monitored
  as its own signal (`now() - max(data_as_of)`), not inferred from pipeline
  success — a green pipeline can deliver stale data.
- Only needed columns/rows are synced; no "sync the whole gold table because
  it was easier".
- **Torn-read exposure:** the atomic unit of sync is known (row, table,
  batch), and screens that join independently-synced tables either tolerate
  or are protected from cross-table skew (order visible, customer missing).
- Replay safety: reprocessing a window of source data cannot produce
  duplicate rows or PK collisions (watermark/tombstone/upsert-key columns
  where the sync needs them).
- Historical restatements upstream have a defined user-facing story (values
  can change under users who exported them).

## 6 Evolution and contract

- A written contract exists: tables, columns, types, nullability, PK, grain,
  freshness SLA, latency SLO, PII classification, owner, compatibility
  window. "The schema is the contract" without a document means no contract.
- The app queries stable views or versioned names (`_v1`) over replicated
  tables, so internals can be rebuilt/renamed without an app release.
- Planned changes are classified additive vs breaking; breaking changes
  follow expand/contract (expand → migrate consumers → contract; see
  martinfowler.com/bliki/ParallelChange.html). Renaming or retyping a column
  in the sync source is a breaking change even when the warehouse tolerates it.
- App-owned tables are under migration tooling (Flyway/Liquibase or
  equivalent); replicated tables are NOT owned by that tooling (the sync
  pipeline owns their DDL).

## 7 Tenancy, security, PII

- Multi-tenant: every tenant-scoped table carries the tenant key; it leads
  hot composite indexes; the enforcement point is named (row-level security
  on app-owned tables, upstream row filtering for replicated tables — many
  managed replicas can't take RLS directly because an internal role owns
  them).
- PII is minimized before it lands: columns the app doesn't render are
  excluded/masked upstream. Every PII column is named in the contract.
- Erasure is a documented, controller-approved workflow that reaches the
  copy: applicability determined (Art. 17 exemptions/legal holds), deletion
  or suppression propagated to live derivatives, **resurrection prevented**
  (re-sync, CDC replay, and snapshot restores cannot bring the subject
  back — durable tombstones/suppression), backups under an approved
  retention policy (ICO "put beyond use" is a backup posture, not a serving-
  copy substitute). Absence of legal/privacy sign-off on this path is itself
  a finding.
- The app's DB principal has least privilege: SELECT on replicated schema,
  scoped DML on app-owned schema, nothing else.

## 8 Capacity and cost

- Row counts and 12–24-month growth per table are stated; the hot working
  set plausibly fits the serving tier's memory.
- History policy: serving copy holds current state (or a shallow window);
  deep history stays in the lakehouse. A serving table that accretes
  unbounded history is a red flag.
- Sync mode/cadence chosen with cost in mind (full refreshes of barely-
  changing tables, or ultra-frequent incremental syncs, both waste money —
  check against the platform's current pricing docs, not from memory).
- Indexes are paid for on every sync write: each one must map to a hot query.

## 9 Failure modes and observability

- Documented behavior when the sync lags or breaks: stale-with-banner,
  degraded mode, or fail closed — chosen, not discovered.
- Duplicate/null source keys are handled by declared policy (dedup key,
  upstream constraint), not by hoping.
- Full-refresh window effects considered (double storage during refresh,
  transient persistence of deleted rows — a compliance window for PII).
- Monitoring covers: sync lag per table, replica query p95, connection count,
  pipeline success — with alert thresholds tied to the SLAs in the contract.
- Silent-divergence detection exists (row counts/checksums reconciled against
  the source) — divergence with no error is otherwise invisible.
- Query governance protects the store from one pathological query: statement
  timeouts, row limits, pool caps for the app principal.
- A restore-from-backup story covers app-owned writes (a PITR rewinds them
  along with replicated data — reconciled how?).
- Hot queries have been EXPLAINed on realistic volumes (ask for evidence).

## 10 Ownership

- Named owners (teams, not individuals) for: gold sources, sync config,
  serving schema, app-owned schema, the contract itself.
- A coordination process exists for changes that cross ownership boundaries
  (gold refactor → serving migration → app release).
- The contract states its compatibility guarantee period.

## ERD smell checks

Structural smells worth a finding even without full context:

- A table no named query pattern justifies (speculative modeling).
- Polymorphic references (`entity_type` + `entity_id`) with no integrity
  strategy.
- Status columns mixing workflow state with sync state.
- A god table serving as both operational ledger and analytics dump.
- Many-to-many junction with no natural key or temporal validity.
- Arguments to reject when offered as defenses: "we'll index later once we
  see slow queries", "it's just gold, but in Postgres", "offset is fine, we
  only have a few pages", "the source is always correct so the serving side
  needs no constraints", "only the app talks to this DB so PII is fine",
  "we'll figure out writeback after launch".

## Severity calibration examples

| Finding | Severity | Why |
|---|---|---|
| App UPDATEs a synced/replicated table | Critical | Writes lost, rejected, or diverging per the sync's documented semantics |
| No PK on a replicated table | Critical | Sync fails or silently drops rows |
| App persists keys that a source rebuild regenerates | Critical | Bookmarks, URLs, and app-owned FKs orphaned wholesale |
| Cross-tenant rows reachable (no tenant scoping on any layer) | Critical | Data leak |
| Column rename planned in sync source without expand/contract | High | Breaks sync/app on deploy |
| OFFSET pagination on unbounded list | High | Linear degradation, page drift |
| Hot filter in JSON column, no expression index | High | Full scans on the hot path |
| Mixed timestamp timezone semantics | High | Wrong times displayed; subtle bugs |
| Screens join independently-synced tables, no torn-read story | High | Users see half-synced, inconsistent state |
| Serving SQL re-derives a governed KPI differently | High | App and BI show two truths; trust collapse |
| No freshness column/monitoring | Medium | Stale data trusted as current |
| PII columns synced but never rendered | Medium | Needless exposure + erasure surface |
| Whole gold table synced for 4 used columns | Medium | Cost + blast radius |
| No documented sync-failure behavior | Medium | Incident improvisation |
| Enum-like TEXT without CHECK | Low | Data drift over time |
