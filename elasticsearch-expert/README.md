# Elasticsearch Expert Skill

An agent skill for [skills.sh](https://skills.sh) that provides deep Elasticsearch expertise covering index mapping design, query optimization, cluster architecture, vector search, and more.

## Installation

```bash
npx skills add https://github.com/Erni/agent-skills --skill elasticsearch-expert
```

## What This Skill Does

When installed, this skill enhances an AI agent with expert-level Elasticsearch knowledge:

- **Index Mapping Design** — Field type selection (including `pattern_text`, `exponential_histogram`), multi-field patterns, dynamic templates, nested vs flattened strategies, LogsDB/TSDB index modes, mapping evolution
- **Query Optimization** — Query DSL best practices, bool query composition, geo queries, function_score, aggregation patterns, ES|QL (including LOOKUP JOIN, INLINE STATS), pagination strategies, profiling
- **Cluster Architecture** — Shard sizing, node roles, data tiers (hot/warm/cold/frozen), ILM policies, LogsDB/TSDB, Failure Store, capacity planning, ingest pipelines
- **Vector Search & AI** — Dense vector configuration, kNN search, Retrievers API (standard, knn, rrf, linear, text_similarity_reranker, rule, pinned, rescorer, diversify), ELSER, inference API, quantization (int8/int4/BBQ/DiskBBQ/bfloat16), ColPali/ColBERT
- **Analysis & Text Processing** — Custom analyzers, language-specific stemming, autocomplete patterns
- **Serverless Elasticsearch** — API limitations, alternative approaches, data retention policies
- **Operational Troubleshooting** — Unassigned shards, circuit breakers, disk watermarks, slow queries, SRE aggregation recipes
- **Security & Observability** — Field/document-level security, cluster health diagnostics, monitoring

### Example prompt that will trigger the skill and showcase its capabilities
You are an Elasticsearch expert assistant. I need you to start by reading your Elasticsearch skill documentation, then demonstrate your expertise by doing the following:

Design an index mapping for an e-commerce product catalog with the following requirements:

- Full-text search on product name and description
- Faceted filtering by category, brand, price range, and tags
- Support for semantic/vector search on product descriptions
- Optimized for high read throughput with 10M+ documents

Write the optimal search query that:

- Combines BM25 keyword search with vector similarity (hybrid search)
- Applies category and price range filters
- Returns aggregations for faceted navigation
- Includes highlighted snippets

Recommend an ILM policy and shard strategy for this scale.

Explain your reasoning at each step, citing current Elasticsearch best practices.

**What the previous prompt does**

It hits several impressive talking points:

- The skill will demonstrate knowledge of modern hybrid search (BM25 + kNN vectors), production-grade index design, and cluster architecture.
- It shows concrete, real-world output rather than abstract capability claims.
- You can swap "e-commerce product catalog" for your actual domain (logs, documents, customers) to make it even more relevant to your org.

## Project Structure

```
elasticsearch-skill/
├── SKILL.md                          # Main skill file (loaded when triggered)
├── references/
│   ├── mapping-guide.md              # Mapping patterns, LogsDB, pattern_text, pitfalls
│   ├── query-patterns.md             # Query templates, geo, function_score, ES|QL JOIN
│   ├── cluster-architecture.md       # Sizing, tiers, LogsDB/TSDB, HA, production checklist
│   ├── vector-search.md              # Retrievers, kNN, hybrid search, ELSER, quantization
│   ├── version-changelog.md          # ES 9.0-9.3 features, breaking changes, migration
│   └── operational-recipes.md        # Troubleshooting runbooks, SRE patterns, diagnostics
├── evals/
│   └── evals.json                    # Test cases for skill validation (10 scenarios)
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
