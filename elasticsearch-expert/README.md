# Elasticsearch Expert Skill

An agent skill for [skills.sh](https://skills.sh) that provides deep Elasticsearch expertise covering index mapping design, query optimization, cluster architecture, vector search, and more.

## Installation

```bash
npx skills add <your-username>/elasticsearch-skill
```

## What This Skill Does

When installed, this skill enhances an AI agent with expert-level Elasticsearch knowledge:

- **Index Mapping Design** — Field type selection, multi-field patterns, dynamic templates, nested vs flattened strategies, mapping evolution
- **Query Optimization** — Query DSL best practices, bool query composition, aggregation patterns, ES|QL, pagination strategies, profiling
- **Cluster Architecture** — Shard sizing, node roles, data tiers (hot/warm/cold/frozen), ILM policies, capacity planning, ingest pipelines
- **Vector Search & AI** — Dense vector configuration, kNN search, hybrid search (RRF), ELSER, inference API, quantization strategies
- **Analysis & Text Processing** — Custom analyzers, language-specific stemming, autocomplete patterns
- **Security & Observability** — Field/document-level security, cluster health diagnostics, monitoring

## Project Structure

```
elasticsearch-skill/
├── SKILL.md                          # Main skill file (loaded when triggered)
├── references/
│   ├── mapping-guide.md              # Detailed mapping patterns and pitfalls
│   ├── query-patterns.md             # Query templates and anti-patterns
│   ├── cluster-architecture.md       # Sizing, tiers, HA, production checklist
│   └── vector-search.md             # kNN, hybrid search, ELSER, quantization
├── evals/
│   └── evals.json                    # Test cases for skill validation
└── README.md
```

## Keeping Up to Date

This skill is designed to be updated as Elasticsearch evolves. Key areas to maintain:

- **Reference files** — Update with new features from each Elasticsearch release
- **Version-specific notes** — Add guidance for breaking changes between major versions
- **New capabilities** — Add reference files for new domains (e.g., Elasticsearch Serverless, new query types)

## Compatibility

Works with any agent platform that supports the [Agent Skills specification](https://agentskills.io):

- Claude Code
- GitHub Copilot
- Cursor
- Cline
- And others

## License

MIT
