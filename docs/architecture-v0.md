# Chronora v0 Architecture

## Purpose and scope

Chronora v0 is a small, local-first temporal fact store exposed through an MCP server. Its job is to accept facts with explicit event-time semantics, retain their provenance, group ingestion activity into sessions, and answer straightforward temporal queries. The design favors deterministic behavior, inspectable storage, and safe retries over inference or distributed scale.

The v0 system has three logical layers:

1. **MCP boundary** — validates tool inputs and returns stable, structured results.
2. **Domain service** — applies timestamp, relation, session, and idempotency rules.
3. **SQLite repository** — performs transactional persistence and indexed queries.

MCP handlers do not issue ad hoc SQL. They call the domain service so the same invariants apply to every write path.

## TemporalFact data model

`TemporalFact` is the unit of stored knowledge. A fact represents an assertion, not an assertion that Chronora has independently verified.

| Field | Type | Required | Meaning |
| --- | --- | --- | --- |
| `fact_id` | UUID/ULID string | yes | Server-assigned stable identifier. |
| `subject` | string | yes | Entity the assertion is about. |
| `predicate` | string | yes | Names the asserted property or relationship. |
| `value` | JSON | yes | Scalar, object, or entity reference; JSON preserves typed values. |
| `create_time` | timestamp | yes | When Chronora durably accepted the fact. |
| `observed_time_start` | timestamp | no | Earliest time at which the assertion is said to hold or occur. |
| `observed_time_end` | timestamp | no | Latest time at which the assertion is said to hold or occur. |
| `source_id` | string | yes | Stable identifier for the producer or source namespace. |
| `source_ref` | string | no | Opaque reference back to the source record, message, or document. |
| `source_quality` | enum | yes | Caller-supplied assessment: `high`, `medium`, `low`, or `unknown`. |
| `session_id` | string | yes | Ingestion session assigned explicitly or by the segmenter. |
| `idempotency_key` | string | no | Retry key unique within `source_id`. |
| `metadata` | JSON object | no | Extensible source-specific attributes that do not affect core semantics. |

Both observed-time fields may be absent when the source gives no event time. If only `observed_time_start` is present, the fact is an instant at that time. If both are present, they form a closed interval and the start must not be later than the end. An end without a start is invalid in v0.

Facts are immutable in v0. A correction is recorded as another fact and may be connected to the original with a `supersedes` relation. This keeps ingestion history auditable and avoids silently changing temporal claims.

### `create_time` versus `observed_time`

Chronora uses two clocks deliberately:

- `create_time` is system time: when the row was first committed. The server assigns it in UTC and never accepts a client override. It provides a reliable ingestion order and is used for retry responses and session segmentation.
- `observed_time_start` and `observed_time_end` are domain time: when the source says the fact occurred or was true. They are supplied by the caller, can be missing, and may be earlier or later than `create_time`.

Queries must name the clock they use. Timeline and interval queries default to observed time and exclude undated facts unless the caller asks to include them. Audit and ingestion queries use `create_time`. Chronora never substitutes `create_time` for missing observed time because doing so would turn "learned now" into "happened now."

All API timestamps are RFC 3339 with an explicit offset. The service normalizes them to UTC and persists integer Unix microseconds for ordering and range comparisons. The response layer renders UTC timestamps with a `Z` suffix.

### `source_quality`

`source_quality` records an input's evidentiary quality, not a probability that the fact is true:

- `high` — direct or authoritative evidence for this assertion.
- `medium` — credible but indirect, incomplete, or not independently confirmed.
- `low` — speculative, noisy, or weakly supported.
- `unknown` — no defensible assessment was supplied.

The value is required, defaults to `unknown` at the MCP boundary, and is stored without automatic promotion. Query results expose it as a filter and as provenance; v0 does not combine competing sources into a synthetic truth score. `source_ref` and `metadata` can carry the explanation needed to audit the rating.

## Temporal relations

Relations connect two existing facts without copying their content. A relation has `relation_id`, `from_fact_id`, `to_fact_id`, `relation_type`, `create_time`, and optional `metadata`. The v0 relation types are:

- interval relations: `before`, `after`, `overlaps`, `contains`, `during`, and `simultaneous`;
- assertion lineage: `supersedes`;
- causal claim: `caused_by`.

Direction is significant: `A before B` is stored from A to B, while `A caused_by B` means B is the claimed cause of A. Self-relations are rejected. Duplicate edges are prevented by a unique key on `(from_fact_id, to_fact_id, relation_type)`.

Interval relations may be either explicit or derived. Explicit relations are source assertions and are persisted. Derived interval relations are computed only when both facts have sufficient observed-time bounds and are returned with `derived: true`; they are not stored. v0 validates the shape of an explicit relation but does not attempt global consistency checking or infer a transitive closure.

## Session segmentation

A session is an ingestion grouping, not an event-time bucket. It helps callers recover the context in which facts arrived without changing their observed-time meaning.

Each incoming fact belongs to a stream identified by `source_id` plus an optional caller-supplied actor or conversation key in metadata. Assignment follows these rules:

1. A valid explicit `session_id` always wins.
2. Otherwise, reuse the stream's open session when the gap from its most recent `create_time` is at most the configured inactivity threshold (30 minutes by default).
3. Start a new session when no open session exists, the threshold is exceeded, or the caller explicitly requests a boundary.

Sessions store `session_id`, stream key, `started_at`, `last_activity_at`, optional `ended_at`, and metadata. Segmenting on server-assigned `create_time` makes retry and late-arrival behavior deterministic. Idempotent retries return the original session and do not extend it. Closed sessions are not silently reopened; a late fact begins or joins the current ingestion session even if its observed time falls within an older session.

The inactivity threshold is configuration, recorded in server diagnostics, rather than baked into stored facts. v0 uses lazy closure: on the next write, an expired session is closed and a new one is created in the same transaction.

## SQLite persistence

One SQLite database is the source of truth. The initial schema consists of:

- `facts` for `TemporalFact` fields;
- `relations` with foreign keys to `facts`;
- `sessions` for stream segmentation state;
- `write_requests` for idempotency keys, request hashes, result identifiers, and commit times;
- `schema_migrations` for ordered schema versions.

JSON values and metadata are stored as canonical JSON text. Database constraints enforce required text, timestamp interval ordering, enum values, relation uniqueness, and referential integrity. Useful initial indexes cover `(subject, predicate)`, observed-time bounds, `create_time`, `session_id`, `source_id`, and both ends of a relation.

Every logical write—including fact insertion, session assignment, relation creation, and idempotency recording—runs in one transaction. Connections enable foreign keys, use WAL journal mode, and set a bounded busy timeout. The repository owns migrations and applies them once at startup under an exclusive migration transaction. Regular backups may use SQLite's online backup mechanism; copying a live database file directly is not a supported backup procedure.

The service is designed for one host with modest concurrent access. It may use multiple read connections, but writes are serialized by SQLite. Database paths are explicit configuration; MCP inputs cannot choose a path or submit SQL.

## Idempotent writes

Every mutating MCP tool accepts an optional `idempotency_key`. Reliable clients should provide one. Idempotency is scoped to `(source_id, tool_name, idempotency_key)` so independent sources cannot collide.

Within the write transaction, Chronora canonicalizes the validated request and hashes the fields that affect the result. It then:

1. inserts the idempotency record and performs the requested mutation when the scoped key is new;
2. returns the original result, including IDs, timestamps, and session assignment, when the key and request hash match an earlier committed request;
3. returns an `IDEMPOTENCY_CONFLICT` error when the key exists with a different request hash.

The unique database constraint, rather than an in-memory check, arbitrates concurrent duplicates. Failed or rolled-back transactions leave no idempotency record and can be retried. When no key is supplied, Chronora creates a new mutation; it does not deduplicate by fact content because two identical assertions may be independently meaningful.

## Candidate MCP tools

Tool names and schemas remain provisional, but the v0 surface should stay small:

| Tool | Purpose |
| --- | --- |
| `chronora_create_fact` | Store one fact and return its ID, normalized times, and session. |
| `chronora_create_facts` | Atomically store a bounded batch with per-item idempotency keys. |
| `chronora_create_relation` | Store an explicit relation between existing facts. |
| `chronora_get_fact` | Fetch one fact with provenance and optional direct relations. |
| `chronora_query_facts` | Filter by subject, predicate, value, source quality, source, session, and either time axis; paginate by stable cursor. |
| `chronora_get_timeline` | Return observed-time-ordered facts and optionally derived interval relations. |
| `chronora_list_sessions` | List sessions by stream and ingestion-time range. |
| `chronora_get_session` | Fetch session metadata and its facts in ingestion order. |
| `chronora_end_session` | Explicitly close the current session for a stream. |
| `chronora_health` | Report schema version and non-sensitive database/runtime status. |

Mutating tools return `created` or `replayed` so callers can distinguish a new commit from an idempotent response. List tools use opaque cursor pagination with a deterministic ID tie-breaker. Errors use stable codes for validation, missing records, idempotency conflicts, and transient database contention.

## v0 non-goals

Chronora v0 intentionally does not include:

- automatic fact extraction from free-form text or documents;
- truth resolution, confidence aggregation, contradiction adjudication, or source-quality inference;
- a general knowledge graph, ontology engine, or arbitrary graph traversal language;
- temporal reasoning beyond direct interval comparison, or validation of global relation consistency;
- in-place fact editing, deletion workflows, or automated retention policies;
- semantic/vector search, embeddings, or full-text ranking;
- multi-user authorization, row-level access control, encryption key management, or audit compliance claims;
- replication, distributed consensus, cloud synchronization, or multi-database tenancy;
- high-write-throughput operation beyond the practical limits of a single SQLite database;
- background session re-segmentation when thresholds change;
- subscriptions, event streaming, notifications, or a user interface.

These omissions keep v0 focused on a trustworthy persistence core. Later versions can add inference, richer search, and distributed operation without changing the distinction between ingestion time and observed time or weakening idempotent write semantics.
