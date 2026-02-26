# Elasticsearch Version Changelog

Key features, breaking changes, and migration guidance across Elasticsearch versions.

---

## Elasticsearch 9.3 (Latest Stable)

### New Features
- **`pattern_text` field type (GA)**: Decomposes log messages into static templates and dynamic variables for ~50% storage reduction. Companion `<field>.template_id` field enables grouping similar messages
- **`exponential_histogram` field type**: Native support for OpenTelemetry exponential histograms
- **GPU-accelerated vector indexing (tech preview)**: NVIDIA cuVS integration for 12x indexing throughput improvement and 7x faster force merging
- **`bfloat16` dense vector element type**: New storage option for dense vectors
- **On-disk rescoring for vectors**: Direct I/O support for BBQ rescoring
- **Binary doc value compression (Zstd)**: Further storage reduction via Zstd compression of doc values
- **Doc values skippers**: Efficient querying without separate indexes
- **Jina AI models (GA)**: jina-embeddings-v3, jina-reranker-v2-base-multilingual, jina-reranker-v3 via Elastic Inference Service
- **Elastic Inference Service via Cloud Connect (GA)**: Self-managed clusters can offload inference to Elastic Cloud managed GPU infrastructure
- **`diversify` retriever (tech preview)**: Reduces redundancy in top-N results
- **Time series aggregations**: Sliding window support for TSDB
- **CEF processor**: Common Event Format parsing in ingest nodes

---

## Elasticsearch 9.2

### New Features
- **Failure Store enabled by default**: For new logs data streams — failed documents routed to a separate failure store instead of being rejected
- **LOOKUP JOIN on multiple fields**: Expanded join support in ES|QL
- **DiskBBQ (`bbq_disk`) index type**: On-disk binary quantization for dense_vector fields — lower memory usage than in-memory BBQ
- **INLINE STATS (tech preview)**: New ES|QL command for inline statistical computation
- **S3 conditional writes**: Repository-s3 uses conditional writes to prevent snapshot corruption
- **Elastic Agent Builder MCP endpoint**: Built-in MCP server replaces the deprecated standalone `@elastic/mcp-server-elasticsearch` package
- **Multi-field query format**: For linear and RRF retrievers (also backported to 8.19)

---

## Elasticsearch 9.1

### New Features
- **`pinned` retriever (GA)**: Places specified documents at the top of results
- **LogsDB/TSDB enhancements**: `_seq_no` field switched from BKD-tree to skip list — ~50% storage reduction for `_seq_no`, ~10% overall storage reduction for time-series data
- **Array field optimization**: Preserved leaf array ordering in doc_values eliminates duplicate storage in `_ignored_source`
- **Segment merging optimization**: Single-pass doc values merging — up to 40% faster, reduced CPU overhead

---

## Elasticsearch 9.0 (Released April 2025)

### Major Changes
- **Built on Lucene 10.1.0**: Better parallel processing, smarter indexing, hardware optimizations (SIMD)
- **Better Binary Quantization (BBQ) GA**: Updated algorithm with up to 20% higher recall and 8x-30x faster throughput vs. previous BBQ
- **ES|QL LOOKUP JOIN (tech preview)**: Cross-index and cross-dataset joins — not available in traditional Query DSL
- **KQL function in ES|QL (tech preview)**: KQL integration within ES|QL queries with scoring support
- **ColPali and ColBERT with MaxSim**: Multi-stage interaction models for sophisticated ranking
- **`linear` retriever (GA)**: Weighted score combination across multiple retrievers — alternative to RRF for hybrid search
- **`text_similarity_reranker` retriever (GA)**: ML-based re-ranking using inference endpoints
- **`rule` retriever (GA)**: Contextual query rules to pin or exclude documents
- **`rescorer` retriever (GA)**: Replaces traditional query rescoring functionality
- **JinaAI embeddings and reranking (GA)**: Via the Open Inference API
- **Default models**: ELSER, e5 (multilingual dense), and Elastic Rerank ship by default
- **`semantic_text` changes**: Now acts like a normal text field — `_source` contains the user-assigned value, not the embedding

### Breaking Changes
- **Requires upgrade to 8.19 first**: Cannot upgrade directly from 8.x < 8.19 to 9.0
- **Enterprise Search removed**: App Search, Workplace Search, and Elastic Web Crawler no longer available — must migrate before upgrading
- **AWS SDK v2**: repository-s3 plugin switched from AWS SDK v1 to v2 — behavior differences in auth and configuration
- **LDAP/AD bind password required**: Configuring a bind DN without a bind password prevents node startup
- **Deprecated tracing settings removed**: `tracing.apm.*` settings no longer work
- **Entitlements replace Java SecurityManager**: New protection mechanism for Java 24+ compatibility
- **LogsDB enabled by default**: For data streams matching `logs-*-*` pattern (unless upgrading from 8.18.2+ with existing streams)

### Deprecations
- **`elser` inference service**: Will be removed in a future version — use the `elasticsearch` service to access ELSER
- **Certificate-based remote cluster security**: Migrate to API key authentication
- **Standalone MCP server**: `@elastic/mcp-server-elasticsearch` deprecated in favor of Elastic Agent Builder MCP (9.2+)

---

## Elasticsearch 8.18 / 8.19 (Final 8.x)

### Key Features
- **8.18**: Released alongside 9.0 as the migration stepping stone
- **8.19**: Last 8.x release — required before upgrading to 9.0
- **Multi-field query format**: For linear and RRF retrievers (backported from 9.1)
- **Upgrade Assistant**: Built-in tool to identify and resolve issues before 9.0 upgrade

### Migration Path to 9.x
1. Upgrade to 8.19.x
2. Run the Upgrade Assistant to identify issues
3. Reindex any indices created before 8.0.0
4. Remove Enterprise Search if in use (migrate use cases first)
5. Review and update deprecated settings
6. Upgrade to 9.0+

---

## Elasticsearch 8.16 / 8.17

### Key Features
- **BBQ (tech preview in 8.16)**: Binary quantization for dense vectors
- **`semantic_text` field type (GA ~8.15-8.16)**: Auto-chunking and embedding without separate ingest pipeline
- **Retrievers API**: Introduced composable retriever framework (`standard`, `knn`, `rrf`)
- **LogsDB index mode**: Optimized log storage with synthetic `_source` and automatic sorting
- **`int4_hnsw` quantization**: 8x memory reduction for vectors

---

## Version Compatibility Quick Reference

| Feature | Minimum Version | Status in 9.3 |
|---------|----------------|---------------|
| Data streams | 7.9 | GA |
| Composable index templates | 7.8 | GA |
| kNN search | 8.0 | GA |
| ES\|QL | 8.11 | GA |
| `semantic_text` field | 8.15 | GA |
| Inference API | 8.15 | GA |
| Retrievers API | 8.14 | GA |
| `int8_hnsw` quantization | 8.14 | GA |
| RRF retriever | 8.14 | GA |
| `bbq_hnsw` quantization | 8.16 (preview) | GA (9.0) |
| LogsDB index mode | 8.17 | GA (default in 9.x) |
| `linear` retriever | 9.0 | GA |
| ES\|QL LOOKUP JOIN | 9.0 | Tech preview |
| `bbq_disk` (DiskBBQ) | 9.2 | GA |
| `pattern_text` field | 9.2 (preview) | GA (9.3) |
| `exponential_histogram` | 9.3 | GA |
| `bfloat16` vectors | 9.3 | GA |
| `diversify` retriever | 9.3 | Tech preview |
| GPU vector indexing | 9.3 | Tech preview |
