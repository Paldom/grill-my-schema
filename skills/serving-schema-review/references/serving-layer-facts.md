# Verified facts behind the grilling questions

Primary-source-verified claims this skill's questions rest on. Platform-specific
rows are **worked examples**, not requirements — the discipline is platform-independent.

**Contents:** [Why a serving layer exists](#why-a-serving-layer-exists) ·
[Read models and CQRS](#read-models-and-cqrs) · [Denormalization evidence](#denormalization-evidence) ·
[Pagination](#pagination) · [Sync and replication constraints](#sync-and-replication-constraints) ·
[Schema evolution](#schema-evolution) · [PII and erasure](#pii-and-erasure) ·
[Platform worked examples](#platform-worked-examples) · [Volatile facts policy](#volatile-facts-policy)

## Why a serving layer exists

An analytics gold layer is columnar, scan-optimized, and evolves for business
questions; an application needs indexed point lookups, bounded p95 latency at
high concurrency, ACID writes for its own state, and a schema that does not
change when the warehouse refactors. Pointing a user-facing app directly at
gold couples it to all four mismatches at once. Vendors that ship a managed
bridge state the same division of labor — e.g. Databricks: "The lakehouse is
optimized for analytics and enrichment, while Lakebase is designed for
operational workloads that require fast lookup-style queries and transactional
consistency" ([synced tables docs](https://docs.databricks.com/aws/en/oltp/projects/sync-tables));
Microsoft's reverse-ETL guidance gives the same rationale for Fabric
([Microsoft Learn](https://learn.microsoft.com/en-us/fabric/database/sql/use-case-reverse-etl)).
Serving gold directly is acceptable only for internal analytical dashboards
that tolerate warehouse latency and concurrency limits.

## Read models and CQRS

The serving layer is a CQRS read model materialized across a platform
boundary: the analytics pipeline is the command side; the app queries
purpose-built projections. Canonical statement, Azure Architecture Center
CQRS pattern: "The read data store can use its own data schema that's
optimized for queries. For example, it can store a materialized view of the
data to avoid complex joins or O/RM mappings"
([learn.microsoft.com](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)).
Corollaries: multiple read models from the same source are normal (one per
query shape), and projections must be rebuildable from the source.

## Denormalization evidence

Fivetran's public benchmark found a single denormalized table (OBT) beat
star-schema joins by 25–50% on average across Redshift, Snowflake, and
BigQuery (~49% on BigQuery)
([fivetran.com/blog/star-schema-vs-obt](https://fivetran.com/blog/star-schema-vs-obt)).
**Caveat:** that benchmark tested columnar analytical engines, not row-store
OLTP — it is context, not normative evidence for a serving store. Row stores
execute indexed joins well; denormalization buys latency but costs stale
duplicates, write amplification, wider rows, and policy duplication. The
defensible order: start from the access-pattern inventory and SLOs, measure
the normalized/composed shape (`EXPLAIN`, load test), and introduce a
denormalized read model when the evidence — or an availability/consistency
requirement — justifies it. Never transfer the 25–50% number to OLTP.

## Pagination

`OFFSET` pagination scans and discards all preceding rows, so cost grows
linearly with page depth, and pages shift when rows are inserted/deleted
between requests. Keyset (seek) pagination — `WHERE (sort_key, id) <
(:last_sort, :last_id)` with a matching composite index — is constant-cost at
any depth. Canonical: Markus Winand's
[no-offset](https://use-the-index-luke.com/no-offset); production
corroboration: [GitLab's pagination guidance](https://docs.gitlab.com/ee/development/database/offset_pagination_optimization.html)
(offset queries "scale linearly" and time out at deep pages).

Keyset's preconditions, not optional: a stable total order with a unique
tie-breaker (hence the `id` in the cursor), an index matching the full sort,
and cursors bound to the same filters. It does not give arbitrary
page-number jumps or snapshot consistency across pages. `OFFSET` remains
fine for shallow, bounded lists, admin screens, and genuine page-number
navigation at modest depth.

## Sync and replication constraints

The **portable principle**: a pipeline-owned table must not receive
application writes unless the synchronization product's documented contract
explicitly supports them — and every sync product punishes key defects,
schema changes, and source types in its own specific way. The behaviors
below are **verified for one product** (Databricks Lakebase synced tables,
[docs](https://docs.databricks.com/aws/en/oltp/projects/sync-tables),
accessed 2026-07-27) and are cited as a worked example of what to establish
from *your* sync's current documentation — connectors differ on every one of
these points (some reject bad rows, halt, or synthesize keys instead):

- **Replicated tables are read-only** (Lakebase): "The pipeline owns the
  table's data, so direct writes are overwritten on the next refresh." App
  state gets separate app-owned writable tables; how the two are composed —
  database join, app-side composition, or an app-owned projection — is a
  workload decision (latency, consistency, orphan handling), not a rule.
- **A primary key is required** (Lakebase); PK columns are non-nullable and
  "rows with nulls in primary key columns are excluded from the sync"
  (silently). Duplicate PKs fail the pipeline unless a timeseries key
  configures last-writer-wins dedup (with a performance penalty). Whatever
  your product does, key hygiene is an upstream obligation — find out
  whether yours drops, halts, or dedups, and design for that answer.
- **Freshness mode is constrained by the source** (Lakebase): incremental
  sync requires change-data-feed on a physical table; views, materialized
  views, and other formats fall back to snapshot-only full refreshes.
  Promising near-real-time freshness over a view-backed sync is a classic
  silent failure of the category.
- **Only additive schema changes propagate** in Lakebase's incremental
  modes; renames, drops, and type changes break the sync. Ask your product
  which change classes it tolerates in place.
- **Complex types land as JSON** (JSONB in Postgres targets), which invites
  hot filters on un-indexable expressions.

Equivalent non-Databricks example: Microsoft Fabric's reverse-ETL reference
architecture targets a SQL database with explicit landing / optional history /
quarantine / serving schemas and the same "operational store, not the
warehouse" rationale
([Microsoft Learn](https://learn.microsoft.com/en-us/fabric/database/sql/use-case-reverse-etl),
accessed 2026-07-27).

## Schema evolution

The safe pattern for breaking changes to a schema with consumers you don't
control is parallel change (expand/contract): expand (add the new structure,
backward-compatibly) → migrate consumers → contract (remove the old last).
Canonical: [martinfowler.com/bliki/ParallelChange.html](https://www.martinfowler.com/bliki/ParallelChange.html).
Two serving-layer applications: (1) never rename/retype a column in the sync
source without it; (2) have the app query stable views (or versioned table
names like `_v1`/`_v2`) over replicated tables so internals can be rebuilt
without an app release.

## PII and erasure

GDPR right to erasure (Art. 17) extends to copies and replicas — the serving
database is a copy, so an erasure that reaches only the warehouse leaves a
non-compliant replica. The workable shape is a **controller-approved erasure
workflow**, not a one-line recipe: determine applicability first (Art. 17
has exemptions — legal holds, legal obligations); propagate deletion or
suppression to every live derivative and cache; **prevent resurrection** —
a re-sync, CDC replay, or restore of an older snapshot can bring an erased
subject back unless durable tombstones or suppression lists survive those
paths; handle backups under an approved retention policy. For backups where
immediate erasure is infeasible, the UK ICO accepts data being "put beyond
use" with a commitment to eventual deletion
([ICO right-to-erasure guidance](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/individual-rights/right-to-erasure/),
accessed 2026-07-27) — that is a backup posture, not a substitute for
deleting from the active serving copy. Erasure design is a legal/privacy
review item, not an engineering-only decision.

The cheaper control is minimization: if a PII column is not needed by a
screen, never sync it — data that never lands in the serving copy never needs
erasing there.

## Platform worked examples

| Concern | Databricks Lakebase | Microsoft Fabric SQL DB | Self-managed (Postgres + CDC/ETL) |
|---|---|---|---|
| Bridge | Synced tables (Lakeflow) | Reverse-ETL pipelines | Debezium/dbt/custom jobs |
| Read-only replica enforced by | Pipeline-owned table | Convention (serving schema) | Convention/grants |
| App-owned writes | Separate Postgres tables, same instance | Same SQL DB, separate schema | Separate schema/database |
| Freshness modes | Snapshot / Triggered / Continuous | Scheduled pipelines | Job cadence / streaming |

## Volatile facts policy

Capacity limits, throughput rates, connection ceilings, and pricing of managed
serving products change frequently and are deliberately **excluded** from this
skill. When a question needs a current number (quota, sync throughput, price),
answer it from the vendor's current documentation, dated.
