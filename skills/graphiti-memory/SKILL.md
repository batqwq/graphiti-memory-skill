---
name: graphiti-memory
description: >-
  Persistent knowledge-graph memory backed by the Graphiti MCP server. Use this
  skill whenever the user wants to remember, store, recall, look up, verify, correct,
  or delete durable facts, relationships, preferences, decisions, people, projects,
  or history that should survive across sessions — including any use of the Graphiti
  tools add_memory, search_memory_facts, search_nodes, get_episodes, get_entity_edge,
  get_memory_queue_status, delete_entity_edge, delete_episode, or clear_graph. Trigger
  it for "save this to memory", "remember that…", "what do you know about X", "recall
  our earlier decision", "add this to the knowledge graph", checking what was stored,
  or any group-scoped Graphiti operation — even when the user does not say "Graphiti"
  by name. Graphiti is a graph (episodes → entities → facts), so its tools behave very
  differently from a flat key-value store; consult this skill before calling them.
---

# Graphiti Memory

Graphiti is a **persistent knowledge-graph memory service**. It does not store
documents you can read back verbatim — it ingests *source episodes*, then an LLM
extracts *entities* and *relationship facts* from them in the background. You
retrieve meaning, not files. Getting good results depends on respecting that
shape, so read this before reaching for the tools.

## The mental model: three layers

| Layer | What it is | Tool that reads it |
|-------|-----------|--------------------|
| **Episode** | The raw source you wrote — a note, a message, a JSON record. Ground truth & provenance. | `get_episodes` |
| **Node (entity)** | A person, project, place, or thing extracted from episodes. Has a name, a generated summary, and a UUID. | `search_nodes` |
| **Edge (fact)** | A relationship between entities, with temporal validity (e.g. "User prefers dark mode", valid-from a date). | `search_memory_facts` → `get_entity_edge` |

Writes flow **down** (you add an episode; entities and facts are derived from it
asynchronously). Reads usually start at the **fact** layer and drill toward
provenance only when a claim matters.

## Two rules that are not optional

These come straight from the server's operating contract. Violating them produces
confidently-wrong answers or cross-contaminated memory, so treat them as load-bearing.

**1. Pin the `group_id` every time.** Every group-scoped call must carry the exact
`group_id` (or `group_ids`) the user or trusted application context has confirmed.
Groups are hard partitions — think separate notebooks. Never guess, invent, rename,
broaden, or silently omit one, and never search across groups, because a plausible
result from the wrong group looks identical to a right one. **If you don't know the
group, ask the user before calling the tool** — don't pick a default.

**2. Don't claim success until you've verified it.** Search results are
relevance-ranked *candidates*, not proof. Writes are *queued*, not done. A deletion
can leave derived facts behind. So after any write, correction, or deletion, run the
follow-up check (below) and report what you actually observed — not what you intended.

## Pick the right tool

```
Question about a fact / relationship / "when did…"  → search_memory_facts
Need to find an entity or get its UUID              → search_nodes
Want the full detail of one fact you already found  → get_entity_edge
"Where did this come from?" / recent raw history    → get_episodes
User wants something remembered                     → add_memory
Did my write finish?                                → get_memory_queue_status
Is the service even up?                             → get_status
User explicitly asks to remove one fact/episode     → delete_entity_edge / delete_episode
User explicitly asks to wipe a whole group          → clear_graph  (see "Destructive")
```

Full parameter tables, defaults, and aliases live in
[`references/tools.md`](references/tools.md) — read it when you need exact argument
names or are using a tool for the first time in a session.

## Reading workflow

1. **Start with `search_memory_facts`** for anything factual, relational, or
   temporal. Phrase one focused natural-language *question* that names the entity,
   the relationship, and any time constraint — e.g. *"What database does the
   AI-Memory project use as of 2026?"* A bag of keywords retrieves poorly because
   the query is embedded as a whole.
2. **Use `search_nodes` only to discover an entity or fetch a UUID.** A node's
   summary is generated context, not authoritative evidence — don't quote it as
   fact. Its main job is to hand you a `center_node_uuid` you can pass back into
   `search_memory_facts` to focus the search around that entity.
3. **Use `get_entity_edge`** to pull the complete stored fact (provenance, valid-from /
   valid-to dates, attributes) before you rely on it for anything important.
4. **Use `get_episodes`** when the user asks where a memory came from, or to review
   recent raw history. This is provenance inspection — not semantic search.
5. **An empty result is not proof of absence.** Before concluding "there's nothing
   stored," try a more specific query, double-check the `group_id`, and check
   `get_memory_queue_status` in case the relevant write is still being processed.

## Writing workflow

1. **Write one coherent episode per `add_memory` call.** Include explicit entity
   names (not pronouns), the relationships between them, dates, where the
   information came from, and any uncertainty. Resolve "he/it/that" to real names —
   the background extractor has no other context, so ambiguous text yields garbage
   entities. Don't pack several unrelated events into one episode; split them.
2. **Only store what the user actually wants remembered.** Don't quietly persist
   guesses, inferences, or working notes. Memory is durable and will resurface.
3. **`add_memory` only *queues* the episode.** The response confirms queueing, nothing
   more. Poll `get_memory_queue_status` with the **same `group_id`** until both
   `pending` and `processing` are `0`. Any non-zero `failed` count or `last_error`
   must be surfaced and investigated — never report a successful import while
   failures remain.
4. **Verify before claiming it's saved.** Once the queue is clear, run a
   `search_memory_facts` (or `search_nodes`) in that group to confirm the fact is
   actually retrievable, then tell the user what you found.

**Example episode (good):**
> Name: `AI-Memory infra`
> Body: *"As of 2026-05-31, the AI-Memory project (owned by batqwq) uses Neo4j as its
> graph database for Graphiti memory. The embedding model was migrated from Gemini to
> Qwen3-embedding-8b via OpenRouter."*

It names the entities, states the relationship, anchors a date, and notes provenance —
so extraction produces clean nodes and facts.

## Correcting and deleting

Deletion is surgical and explicit — it is **not** a cleanup, dedupe, or "this looks
wrong" tool. Only delete the exact item the user named, and only on their explicit
request.

1. **Locate and confirm the UUID.** Find the fact with `search_memory_facts` and
   verify it with `get_entity_edge` (or find the episode with `get_episodes`). Never
   pass a guessed UUID.
2. **Delete the specific item** with `delete_entity_edge` (one fact) or
   `delete_episode` (one source episode). Note that deleting an episode does not
   guarantee removal of every node or fact derived from it, and a surviving episode
   can cause a similar fact to be re-derived.
3. **Re-search the same group and report the verified result** — don't assume the
   delete propagated.

To correct a wrong fact, usually **add a new, accurate episode** rather than deleting
— Graphiti is bitemporal and supersedes old facts with newer ones. Reserve deletion
for information that must genuinely be erased.

### Destructive: `clear_graph`

`clear_graph` irreversibly wipes **every** episode, node, and fact in the named
group(s), with no backup. Only call it when the user has explicitly asked to reset,
named every target group, and confirmed permanent deletion is acceptable. If any of
those three is missing, stop and ask. Never use it to "fix" empty, noisy, or
duplicated results.

## Common pitfalls

- **Omitting `group_id`** because a default seems obvious — ask instead.
- **Treating the top search hit as truth** — verify with `get_entity_edge` before
  acting on anything that matters.
- **Reporting "saved!" right after `add_memory`** — it was only queued; poll the
  queue and verify first.
- **Keyword-soup queries** — Graphiti embeds the whole query string; write a real
  question.
- **Quoting a node summary as a fact** — summaries are generated and can be stale;
  the fact edges are the evidence.
- **Deleting to "clean up"** — deletion is for explicit, named removals only.
