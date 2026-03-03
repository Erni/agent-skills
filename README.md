# Agent Skills

> Specialized skills for AI agents — works with Claude, Claude Code, Cursor, GitHub Copilot, VS Code, OpenAI Codex, and [20+ other platforms](https://agentskills.io).

A collection (single-skill so far) of **Agent Skills** following the [open Agent Skills standard](https://agentskills.io). Each skill is a self-contained folder that teaches an AI agent how to complete a specialized task reliably — no copy-pasting prompts, no repeated context setup. Write once, use everywhere.

## Skills

### 🔍 `elasticsearch-expert`

An expert-level Elasticsearch agent covering the full spectrum of search infrastructure — from first index to production cluster.

**Triggers when you ask about:**
- Index mapping design and field type selection
- Query DSL, aggregations, and search performance
- Cluster architecture, shard sizing, and ILM policies
- Data streams, index templates, and ingest pipelines
- Vector search, kNN, ELSER, and semantic search
- Solr → Elasticsearch migrations
- ES|QL and Kibana queries

This skill searches the web to stay current with the latest Elasticsearch releases, so answers reflect the current version — not a training data snapshot.

## How Skills Work

Skills use **progressive disclosure** to stay lightweight on your context window:

1. **Discovery** — At startup, the agent scans only skill names and descriptions (~30–50 tokens each).
2. **Activation** — When a task matches, the full `SKILL.md` instructions are loaded.
3. **On demand** — Supporting `scripts/`, `references/`, and `assets/` are fetched only as needed.

You can keep dozens of skills available without burning your context budget.

## Resources

- [Agent Skills specification](https://agentskills.io/specification)
- [What are skills?](https://support.claude.com/en/articles/12512176-what-are-skills)
- [Equipping agents for the real world with Agent Skills](https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

---

> Always test skills in your own environment before using them in critical workflows.
