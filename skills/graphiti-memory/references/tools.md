# Graphiti MCP — Tool Reference

Exact parameters, defaults, and behavior for every Graphiti MCP tool. The actual
tool names are namespaced by the MCP server (commonly
`mcp__<server>__<tool>`, e.g. `mcp__Graphiti__add_memory`); the logical names
below match the ones the server documents.

Read [`../SKILL.md`](../SKILL.md) first for the workflows and the two
non-negotiable rules. This file is the lookup table.

## Contents

- [Writing](#writing) — `add_memory`
- [Reading](#reading) — `search_memory_facts`, `search_nodes`, `get_entity_edge`, `get_episodes`
- [Status](#status) — `get_status`, `get_memory_queue_status`
- [Deleting](#deleting) — `delete_entity_edge`, `delete_episode`, `clear_graph`

---

## Writing

### `add_memory`

Queue **one** source episode for asynchronous extraction into the graph. Returns
only confirmation that the episode was *queued* — not that processing finished.

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `name` | string | — (required) | Short, stable, descriptive title for the episode. |
| `episode_body` | string | — (required) | The full source content. For `source="json"`, pass a JSON-**encoded string**, not an object. |
| `group_id` | string \| null | `null` | The exact confirmed group. Mandatory in practice — ask the user if unknown; do not omit or guess. |
| `source` | string | `"text"` | `"text"` (prose), `"json"` (JSON-encoded string), or `"message"` (conversation-style). |
| `source_description` | string | `""` | Brief provenance, e.g. document type or conversation context. |
| `uuid` | string \| null | `null` | Optional caller-supplied stable episode UUID. Normally omit. |

**After calling:** poll [`get_memory_queue_status`](#get_memory_queue_status) with the
same `group_id` until `pending=0` and `processing=0`, then verify with
[`search_memory_facts`](#search_memory_facts). Write explicit names, relationships,
dates, provenance, and uncertainty; one event per episode; no pronouns or
keyword fragments.

---

## Reading

### `search_memory_facts`

Primary retrieval tool. Searches derived relationship **facts (edges)** by semantic
meaning. Use for factual, relational, and temporal questions.

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `query` | string | — (required) | One focused natural-language question naming the entities, relationship, and any time constraint. Not a keyword list. |
| `group_ids` | string[] \| null | `null` | Exact confirmed groups. Mandatory in practice — never broaden or search across unconfirmed groups. |
| `center_node_uuid` | string \| null | `null` | Optional entity-node UUID (from `search_nodes`) to focus retrieval around that entity. Pass a real UUID, never a name. |
| `max_facts` | integer | `10` | Max ranked facts to return; positive, server-capped. |

Results are relevance-ranked **candidates**, not guaranteed truth. Inspect important
ones with `get_entity_edge`.

### `search_nodes`

Discover entity **nodes** by name, alias, role, summary, or meaning. Use for entity
discovery or to obtain a UUID — not for relationship questions (use
`search_memory_facts` for those).

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `query` | string | — (required) | Focused entity lookup: a canonical name, alias, role, or identifying context. |
| `group_ids` | string[] \| null | `null` | Exact confirmed groups. Mandatory in practice. |
| `entity_types` | string[] \| null | `null` | Optional exact configured entity labels to filter by. Omit if unknown. |
| `max_nodes` | integer | `10` | Max ranked nodes; positive, server-capped. |

A node's `summary` is generated context and may be stale — verify critical claims
against facts or episodes. Empty results are not proof of absence.

### `get_entity_edge`

Retrieve the full stored representation of **one** fact (edge) by UUID. Read-only.

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `uuid` | string | — (required) | Exact fact UUID from `search_memory_facts` in the confirmed group. Never guess. |

Use it to inspect provenance, temporal fields (valid-from / valid-to), and attributes
before relying on or deleting a fact.

### `get_episodes`

List raw source **episodes** from one or more groups in deterministic `created_at`
order. Provenance / recent-history inspection — **not** semantic search.

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `group_ids` | string[] \| null | `null` | Exact confirmed groups. Preferred over the legacy `group_id`. |
| `max_episodes` | integer \| null | `null` | Preferred page size; positive, server-capped. |
| `offset` | integer | `0` | Episodes to skip, for pagination. |
| `sort_order` | string | `"desc"` | `"desc"` (newest first) or `"asc"` (oldest first). |
| `content_max_chars` | integer | `1200` | Per-episode preview length; `0` omits content previews. |
| `group_id` | string \| null | `null` | Legacy single-group alias. Prefer `group_ids`. |
| `limit` / `last_n` | integer \| null | `null` | Legacy page-size aliases. Prefer `max_episodes`; don't pass disagreeing values. |

Use it to locate and verify an episode UUID before `delete_episode`.

---

## Status

### `get_status`

Connectivity diagnostic only — confirms the MCP service is initialized and can reach
its graph database. No parameters.

A healthy result does **not** imply queues are empty, memories exist, or that LLM /
embedding providers are working for new writes. Use `get_memory_queue_status` for
ingestion health.

### `get_memory_queue_status`

Inspect asynchronous ingestion progress without modifying the graph.

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `group_id` | string \| null | `null` | The group whose queue you're checking. After `add_memory`, pass the **same** group_id. Omit only for an explicit admin-wide overview. |

A group is caught up only when `pending` **and** `processing` are both `0`. Report any
`failed` count or `last_error`; a non-zero `retried` count indicates transient
JSON-output failures that auto-recovered (useful when diagnosing reliability).

---

## Deleting

> Deletion is surgical and explicit. Only remove the exact item the user named, on
> their explicit request. Never delete because results look duplicated, stale, noisy,
> or wrong. See [SKILL.md → Correcting and deleting](../SKILL.md#correcting-and-deleting).

### `delete_entity_edge`

Permanently delete exactly one derived **fact (edge)** by UUID.

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `uuid` | string | — (required) | Exact fact UUID, located via `search_memory_facts` and verified with `get_entity_edge`. Never guess. |

Does not delete the source episode or connected nodes; a surviving episode may cause a
similar fact to be re-derived. Re-search the same group afterward to confirm.

### `delete_episode`

Permanently delete exactly one source **episode** by UUID.

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `uuid` | string | — (required) | Exact episode UUID, verified via `get_episodes`. Never guess. |

Removes the source record and its direct links, but does not guarantee removal of
every node, derived fact, or summary it influenced. Verify in the same group after.

### `clear_graph`

**Irreversible.** Deletes all episodes, nodes, and facts in the named group
partition(s). No backup is created.

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `group_ids` | string[] \| null | `null` | Explicit list of every group the user named and authorized for permanent erasure. Never omit, guess, or broaden. |

Only call when the user has explicitly asked to reset, named every target group, and
confirmed permanent deletion is acceptable. Not a cleanup, dedupe, migration, or
troubleshooting tool. Verify only the authorized groups afterward and report the
result.
