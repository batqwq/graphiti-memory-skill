# graphiti-memory-skill

An **[Agent Skill](https://www.anthropic.com/news/agent-skills)** that teaches an AI
agent to use the [Graphiti](https://github.com/getzep/graphiti) MCP server correctly —
a persistent **knowledge-graph memory** that survives across sessions.

Graphiti speaks the open [Model Context Protocol](https://modelcontextprotocol.io), so
it is **not Claude-only**: any MCP-capable agent — Claude Code, Claude Desktop, Cursor,
Cline, or your own custom agent — can connect to it, and this skill's guidance applies
to all of them. It ships as a Claude Code plugin/skill for one-command install, but
`SKILL.md` is plain, portable Markdown that any agent can load.

Graphiti isn't a flat key-value store. It ingests *source episodes*, then extracts
*entities* and *relationship facts* from them in the background. That graph shape makes
its tools easy to misuse — searching the wrong partition, treating a ranked guess as
truth, or reporting "saved!" when a write is still queued. This skill encodes the
correct workflows so the agent reads, writes, verifies, and deletes memory safely.

## What it covers

- **The mental model** — episodes → entities (nodes) → facts (edges), and which tool reads each layer.
- **Two non-negotiable rules** — always pin the `group_id`; never claim success without verifying.
- **Reading workflow** — facts-first retrieval, entity discovery, provenance, and why "empty" ≠ "absent".
- **Writing workflow** — coherent episodes, async queue polling, and post-write verification.
- **Correcting & deleting** — surgical, explicit deletion; superseding facts via new episodes; the destructive `clear_graph` guardrails.
- **A full tool reference** — every parameter, default, and alias for all 9 Graphiti tools, in [`references/tools.md`](skills/graphiti-memory/references/tools.md).

## Prerequisites

A running **Graphiti MCP server** connected to your agent / MCP client (Claude Code,
Claude Desktop, Cursor, Cline, or any MCP-capable agent), exposing the standard
Graphiti tools (`add_memory`, `search_memory_facts`, `search_nodes`, `get_episodes`,
`get_entity_edge`, `get_memory_queue_status`, `delete_entity_edge`, `delete_episode`,
`clear_graph`). See the
[Graphiti MCP server docs](https://github.com/getzep/graphiti/tree/main/mcp_server).

The skill itself is provider- and agent-agnostic — it teaches *how* to drive the tools
and does not hardcode any server URL, group IDs, or credentials.

## Install

Options A and B use **Claude Code**'s plugin/skill loader; Option C works for any other
agent.

### Option A — as a Claude Code plugin (recommended)

This repo is also a self-contained plugin marketplace:

```
/plugin marketplace add batqwq/graphiti-memory-skill
/plugin install graphiti-memory@graphiti-memory-skill
```

### Option B — as a personal skill, manual copy (Claude Code)

Copy the skill folder into your user skills directory:

```bash
git clone https://github.com/batqwq/graphiti-memory-skill.git
cp -r graphiti-memory-skill/skills/graphiti-memory ~/.claude/skills/
```

Then restart Claude Code (or reload skills). The skill triggers automatically when a
task involves remembering, recalling, or managing Graphiti memory — you don't invoke
it by hand.

### Option C — any other MCP-capable agent

The skill is just portable Markdown, so feed it to your agent however it loads
instructions — a system prompt, a rules file (e.g. a Cursor or Cline rule), a retrieved
doc, or your framework's own skill mechanism:

```bash
git clone https://github.com/batqwq/graphiti-memory-skill.git
# Point your agent at:
#   skills/graphiti-memory/SKILL.md            (workflows + the two rules)
#   skills/graphiti-memory/references/tools.md (full tool parameter reference)
```

The workflows and tool reference are agent-neutral; only the install glue differs.

## Repository layout

```
graphiti-memory-skill/
├── README.md
├── LICENSE
├── .claude-plugin/
│   ├── plugin.json            # plugin manifest (Claude Code)
│   └── marketplace.json       # self-referential marketplace (Claude Code)
└── skills/
    └── graphiti-memory/
        ├── SKILL.md           # the skill (workflows + rules) — agent-neutral
        ├── references/
        │   └── tools.md       # full per-tool parameter reference
        └── evals/
            └── evals.json     # sample test prompts for the skill
```

## How it works

In Claude Code, a skill loads in three stages (progressive disclosure): the `name` +
`description` are always in context; `SKILL.md` loads when the skill triggers; and
`references/tools.md` is pulled in only when exact parameters are needed. This keeps
the always-on footprint small while making deep detail available on demand. Other
agents can mirror this layering or just load `SKILL.md` directly — nothing in the
content depends on a Claude-specific feature.

## License

[MIT](LICENSE) © 2026 batqwq

Graphiti is a trademark of its respective authors; this is an independent,
community-authored skill and is not affiliated with Zep / Graphiti.
