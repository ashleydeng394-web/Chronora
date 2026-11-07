# Chronora v0 Architecture

## Purpose

Chronora is not a generic event store and not merely a timestamp display layer.

Its v0 purpose is narrower: **give the assistant reliable temporal facts about a conversation so it can reason about real elapsed time instead of treating message turns as if they were continuous lived time.**

Chronora should be able to answer questions such as:

- When did this message actually occur?
- How long has it been since the previous message?
- How long has it been since the previous user message?
- Did the conversation cross a calendar day boundary?
- Did the user return after a meaningful idle period?
- Is this still the same conversation session?
- How long has the current session lasted?

Core product principle:

> **The watch tells time; it is not a police officer.**

Chronora provides temporal facts and derived temporal relations. It does not turn those facts into behavioral rules such as sleep reminders, productivity pressure, or other prescriptive policies.

## v0 architecture

```text
Collectors -> Temporal facts -> Temporal relations -> Assistant-facing context
```

The four layers are intentionally separated:

1. **Collectors** acquire timestamps and message identifiers from a client or bridge.
2. **Temporal facts** preserve the best-known raw timing evidence.
3. **Temporal relations** derive gaps, day boundaries, and sessions from those facts.
4. **Assistant-facing context** exposes only the small amount of temporal context needed for the current interaction.

The core must remain independent from any one ChatGPT DOM implementation or client.

---

## 1. Collectors

Collectors are adapters. They acquire facts; they do not decide how the assistant should behave.

### Web collector

For ChatGPT web, the preferred source is the platform's real conversation/message timestamp metadata, especially the actual message `create_time`.

Principles:

- Prefer authoritative conversation data over DOM render time.
- DOM state may help identify or match a message, but it must not become the temporal source of truth.
- Re-rendering old messages must never make them appear to have happened "now".
- Stable platform message IDs should be preserved whenever available.

Research references that informed this direction include timegpt-style extraction of real historical `create_time`, timestamp extensions that compute message gaps, and MCP time tools for current-time access.

### Android collector

v0 does **not** assume that the ChatGPT Android app exposes internal message metadata.

A future Android path should preferably reuse an existing device bridge, such as 掌心窗 or another phone-side collector, and send facts into the same Chronora core.

Chronora should not require a separate dedicated Android app merely to provide time awareness.

### Collector metadata

Collector identity should be inspectable so facts can be audited later. A collector record may include:

- implementation name
- implementation version
- platform
- collection method

Collectors should remain replaceable without changing the storage or relation model.

---

## 2. TemporalFact

The v0 fact model is intentionally message-oriented and platform-independent.

```text
schema_version
source
platform
conversation_id
message_id
role
create_time
observed_time (optional)
source_quality
collector
```

A practical stored representation may add internal persistence fields such as `id`, `inserted_at`, or canonicalized collector metadata, but those internal fields must not redefine the meaning of the temporal fields above.

### Field semantics

#### `create_time`

`create_time` is the **best-known time when the message/event actually occurred according to the source**.

For ChatGPT, when the platform exposes its own historical message timestamp, that timestamp is the preferred `create_time`.

This is the main time axis used for:

- message ordering
- gap calculation
- cross-day calculation
- session segmentation
- assistant-facing temporal context

Chronora must never silently replace a missing historical `create_time` with "the time we happened to observe it".

#### `observed_time`

`observed_time` is the optional time when a collector observed or captured the fact.

It is diagnostic/provenance information, not a substitute for event time.

Example:

- a message actually occurred at 09:12 -> `create_time = 09:12`
- a browser collector loaded that old conversation at 15:40 -> `observed_time = 15:40`

The message must still be treated as a 09:12 message.

#### `source_quality`

v0 uses three semantic quality levels:

- `authoritative` — timestamp supplied by the platform or another authoritative source
- `observed` — timestamp obtained through a direct local observation that is useful but not platform-authoritative
- `inferred` — timestamp reconstructed or inferred when no better source exists

Quality is about **how the time was obtained**, not about whether the message content is truthful.

Ordering rule:

```text
authoritative > observed > inferred
```

A lower-quality fact must not overwrite a higher-quality timestamp for the same stable message identity.

### Core invariants

1. Historical messages must not become "now" merely because they were re-rendered.
2. `authoritative` timing must not be overwritten by `observed` or `inferred` timing.
3. Raw temporal facts and derived temporal relations are stored separately.
4. Stable platform message IDs should be used when possible.
5. Chat text is not required for Chronora v0 and should not be stored by default.

---

## 3. Idempotent ingestion

The primary idempotency identity for a message fact is:

```text
(source, conversation_id, message_id)
```

This allows collectors to safely send the same message multiple times.

Expected behavior:

- identical repeated collection is a no-op or replay
- a higher-quality version may enrich or replace lower-quality timing for the same fact
- a lower-quality version must not degrade an existing higher-quality fact
- a conflicting same-quality authoritative value should be surfaced as a conflict for inspection rather than silently overwritten

### Engineering detail

SQLite should arbitrate idempotency with a unique constraint rather than relying only on in-memory checks.

Writes should be transactional.

Useful implementation choices retained from the first Codex draft:

- WAL mode
- foreign keys enabled
- bounded busy timeout
- schema migrations
- stable error codes
- transactional upsert/enrichment behavior

These are implementation details; they do not change the temporal semantics above.

---

## 4. Temporal relations

Temporal relations are **derived from message facts**.

The v0 relation layer focuses only on what the assistant needs for conversation-time awareness:

```text
previous_message_id
previous_user_message_id
gap_from_previous_message_ms
gap_from_previous_user_message_ms
crossed_calendar_day
session_id
starts_new_session
session_started_at
session_elapsed_ms
returned_after_idle
```

These are derived values, not platform facts.

Chronora v0 does not need a general temporal knowledge graph with arbitrary `before`, `overlaps`, `caused_by`, or `supersedes` relations.

### Gap calculation

Gaps are calculated from `create_time`, because that is the event/message time axis.

If a reliable `create_time` is unavailable, Chronora should expose that uncertainty rather than silently substitute `observed_time`.

### Calendar-day crossing

`crossed_calendar_day` is derived in the configured local timezone relevant to the user/session, not by comparing UTC date strings blindly.

Timezone handling should therefore be explicit at the context/query boundary.

---

## 5. Session segmentation

A session is a Chronora-derived grouping of conversation activity.

It is **not** an ingestion session and should not be segmented by database write time.

The primary segmentation axis is message `create_time`.

### Configurable rules

v0 should treat session boundaries as configurable, not universally correct.

Example configuration:

```text
idle_gap = 30m | 60m | 90m
split_on_calendar_day = true | false
```

A new session may begin when:

- the time gap from the previous relevant message exceeds `idle_gap`
- a configured day boundary rule applies
- an explicit rebuild/configuration rule says so

The derived relation should preserve:

- `session_id`
- `starts_new_session`
- `session_started_at`
- `session_elapsed_ms`

### Rebuildability

Sessions should be treated as derived state.

They may be:

- computed on demand, or
- cached in a `sessions` table for convenience/performance

Either way, they should remain reconstructible from temporal facts and configuration.

Changing the idle threshold must not corrupt the raw facts.

---

## 6. SQLite persistence

SQLite is sufficient for v0.

At minimum, the database needs a `temporal_facts` table.

A `sessions` table is optional as a rebuildable cache.

A practical implementation may also use:

- `schema_migrations`
- a collector metadata table
- a configuration table

### Suggested `temporal_facts` responsibilities

Store:

- schema version
- source
- platform
- conversation ID
- message ID
- role
- create time
- observed time
- source quality
- collector metadata
- internal insert/update timestamps if needed

Do **not** require message text.

### Persistence rules

- Unique identity: `(source, conversation_id, message_id)`
- Repeated ingestion is safe.
- Higher-quality time may enrich lower-quality data.
- Lower-quality time cannot overwrite higher-quality data.
- Raw facts remain distinct from derived relations/session caches.
- Database writes should be transactional.

### Time representation

API boundaries should use offset-aware RFC 3339 / ISO 8601 timestamps.

Internally, SQLite may additionally store normalized integer epoch values for sorting and gap computation if useful, while preserving enough information to return correct offset-aware context.

---

## 7. Assistant-facing temporal context

Chronora should not dump a full timeline into the model on every turn.

It should return a small factual context.

Example:

```json
{
  "now": "2026-11-07T23:42:00+08:00",
  "conversation": {
    "current_session_started_at": "2026-11-07T23:04:12+08:00",
    "session_elapsed_minutes": 38
  },
  "user": {
    "last_message_at": "2026-11-07T23:31:04+08:00",
    "minutes_since_last_message": 11,
    "returned_after_idle": false,
    "crossed_calendar_day": false
  }
}
```

The payload describes reality. It does not prescribe behavior.

Do **not** return:

```json
{"recommendation":"It is late; tell the user to go to sleep."}
```

A factual payload such as:

```json
{"local_time":"02:07","minutes_since_last_message":3}
```

is appropriate. The assistant decides whether time matters in the current conversational context.

---

## 8. v0 MCP surface

Keep the first interface deliberately small.

### `get_now`

Return current local/UTC time information needed for temporal grounding.

### `ingest_temporal_facts`

Ingest one or more temporal facts using the idempotency and source-quality rules above.

### `get_message_time`

Return the best-known timing/provenance for a specific message identity.

### `get_conversation_temporal_context`

Return a compact assistant-facing context containing the latest message timing, gaps, session state, cross-day state, and current time.

Possible later tools:

- `list_sessions`
- `get_session`
- `rebuild_sessions`

Write/collector permissions and assistant read permissions should be separable in a later hardening pass.

Chronora v0 should not expose a broad generic fact CRUD API.

---

## 9. v0 acceptance criteria

v0 is successful when it can:

1. ingest a small hand-written set of message temporal facts
2. preserve genuine historical `create_time`
3. ingest the same message repeatedly without duplication
4. prevent low-quality timestamps from overwriting higher-quality ones
5. calculate the correct gap between consecutive messages
6. calculate the correct gap from the previous user message
7. identify a calendar-day boundary
8. segment sessions using configurable idle-gap rules
9. return a concise assistant-facing temporal context
10. do all of the above without reading or storing chat text

No UI or Android integration is required for this milestone.

---

## 10. Explicit v0 non-goals

Chronora v0 does not include:

- sleep reminders or other behavioral policy engines
- browser timestamp UI
- a dedicated Android app
- full chat archiving
- a memory system
- guessing missing historical times
- binding the core to the ChatGPT DOM
- semantic/vector search
- generalized knowledge-graph relationships
- arbitrary causal or temporal relation graphs
- distributed databases or replication
- automatic free-form text fact extraction

The first implementation is intentionally a **time core**, not a general personal data platform.

---

## 11. Implementation order

When implementation begins, do it in this order:

1. define the temporal fact schema and migrations
2. implement idempotent ingestion and source-quality precedence
3. implement gap and cross-day calculations
4. implement configurable session segmentation
5. expose the assistant-facing read interface
6. test with hand-written fixtures

Do not build the web collector first.

The core must remain stable even if the collector, client, or platform changes later.
