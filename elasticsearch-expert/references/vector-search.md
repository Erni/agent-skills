# Vector Search and AI Integration

## Table of Contents
1. [Dense Vector Configuration](#dense-vector-configuration)
2. [kNN Search Patterns](#knn-search-patterns)
3. [Hybrid Search](#hybrid-search)
4. [Retrievers API](#retrievers-api)
5. [ELSER (Learned Sparse Encoder)](#elser)
6. [Inference API](#inference-api)
7. [Quantization and Performance](#quantization-and-performance)
8. [ColPali and ColBERT](#colpali-and-colbert)

---

## Dense Vector Configuration

### Basic Mapping

```json
{
  "mappings": {
    "properties": {
      "title": { "type": "text" },
      "content": { "type": "text" },
      "content_embedding": {
        "type": "dense_vector",
        "dims": 768,
        "index": true,
        "similarity": "cosine"
      }
    }
  }
}
```

### Similarity Functions

| Function | Range | Use When |
|----------|-------|----------|
| `cosine` | [-1, 1] | Embeddings are normalized or direction matters most |
| `dot_product` | Unbounded | Pre-normalized vectors, slightly faster than cosine |
| `l2_norm` | [0, +inf) | Magnitude matters (image features, spatial data) |
| `max_inner_product` | Unbounded | Non-normalized vectors where inner product is the metric |

### HNSW Tuning

```json
{
  "mappings": {
    "properties": {
      "embedding": {
        "type": "dense_vector",
        "dims": 384,
        "index": true,
        "similarity": "dot_product",
        "index_options": {
          "type": "hnsw",
          "m": 16,
          "ef_construction": 100
        }
      }
    }
  }
}
```

| Parameter | Default | Effect |
|-----------|---------|--------|
| `m` | 16 | Connections per node. Higher = better recall, more memory |
| `ef_construction` | 100 | Build-time candidates. Higher = better graph, slower indexing |

For search-time tuning, use `num_candidates` in the kNN query.

---

## kNN Search Patterns

### Basic kNN Query

```json
POST my-index/_search
{
  "knn": {
    "field": "content_embedding",
    "query_vector": [0.1, 0.2, 0.3, ...],
    "k": 10,
    "num_candidates": 100
  },
  "_source": ["title", "content"]
}
```

### kNN with Pre-Filter

```json
POST my-index/_search
{
  "knn": {
    "field": "content_embedding",
    "query_vector": [0.1, 0.2, 0.3, ...],
    "k": 10,
    "num_candidates": 100,
    "filter": {
      "bool": {
        "must": [
          { "term": { "category": "technology" } },
          { "range": { "publish_date": { "gte": "2024-01-01" } } }
        ]
      }
    }
  }
}
```

**Important**: Filters in kNN are applied during the ANN search, not after — this is more efficient and accurate than post-filtering.

### `num_candidates` Guidance

| `num_candidates` | Recall | Speed | Use Case |
|-------------------|--------|-------|----------|
| k * 1.5 | Low | Fast | Quick similarity check |
| k * 10 (default-ish) | Good | Balanced | General search |
| k * 50+ | High | Slow | Precision-critical |

---

## Hybrid Search

### Reciprocal Rank Fusion (RRF) — Recommended

```json
POST my-index/_search
{
  "retriever": {
    "rrf": {
      "retrievers": [
        {
          "standard": {
            "query": {
              "match": { "content": "machine learning fundamentals" }
            }
          }
        },
        {
          "knn": {
            "field": "content_embedding",
            "query_vector": [0.1, 0.2, ...],
            "k": 10,
            "num_candidates": 100
          }
        }
      ],
      "rank_constant": 60,
      "rank_window_size": 100
    }
  }
}
```

RRF merges ranked lists without needing score normalization. Available from 8.14+.

### Linear Combination (Manual Hybrid)

```json
POST my-index/_search
{
  "query": {
    "script_score": {
      "query": {
        "bool": {
          "should": [
            { "match": { "content": "machine learning" } }
          ],
          "filter": [
            { "exists": { "field": "content_embedding" } }
          ]
        }
      },
      "script": {
        "source": """
          double bm25 = _score;
          double vector = cosineSimilarity(params.query_vector, 'content_embedding') + 1.0;
          return params.bm25_weight * bm25 + params.vector_weight * vector;
        """,
        "params": {
          "query_vector": [0.1, 0.2, ...],
          "bm25_weight": 0.3,
          "vector_weight": 0.7
        }
      }
    }
  }
}
```

---

## Retrievers API

The Retrievers API (8.14+) provides a composable framework for building search pipelines. Prefer retrievers over manually combining queries — they handle score normalization and result merging automatically.

### Available Retrievers

| Retriever | Purpose | Availability |
|-----------|---------|-------------|
| `standard` | Traditional query (replaces top-level `query`) | GA (8.14+) |
| `knn` | Vector similarity search | GA (8.14+) |
| `rrf` | Reciprocal Rank Fusion — merges ranked lists without score normalization | GA (8.14+) |
| `linear` | Weighted score combination across retrievers | GA (9.0+) |
| `text_similarity_reranker` | ML-based semantic re-ranking via inference endpoint | GA (9.0+) |
| `rule` | Contextual query rules to pin or exclude documents | GA (9.0+) |
| `pinned` | Places specified documents at the top of results | GA (9.1+) |
| `rescorer` | Replaces traditional query rescoring | GA (9.0+) |
| `diversify` | Reduces redundancy in top-N results | Tech preview (9.3+) |

### Linear Retriever — Weighted Hybrid Search

The `linear` retriever calculates a weighted sum across sub-retrievers, preserving relative importance. Use this when you need explicit control over score weighting (vs. RRF which is rank-based).

```json
POST my-index/_search
{
  "retriever": {
    "linear": {
      "retrievers": [
        {
          "retriever": {
            "standard": {
              "query": {
                "match": { "content": "machine learning fundamentals" }
              }
            }
          },
          "weight": 0.3
        },
        {
          "retriever": {
            "knn": {
              "field": "content_embedding",
              "query_vector": [0.1, 0.2, ...],
              "k": 10,
              "num_candidates": 100
            }
          },
          "weight": 0.7
        }
      ]
    }
  }
}
```

### Text Similarity Reranker

Re-rank results using an inference model for semantic relevance:

```json
POST my-index/_search
{
  "retriever": {
    "text_similarity_reranker": {
      "retriever": {
        "rrf": {
          "retrievers": [
            { "standard": { "query": { "match": { "content": "search query" } } } },
            { "knn": { "field": "embedding", "query_vector": [...], "k": 50, "num_candidates": 200 } }
          ]
        }
      },
      "field": "content",
      "inference_id": "my-reranker",
      "inference_text": "search query",
      "rank_window_size": 100
    }
  }
}
```

### Rule Retriever — Pinning and Exclusion

Apply business rules to search results:

```json
POST my-index/_search
{
  "retriever": {
    "rule": {
      "retriever": {
        "standard": {
          "query": { "match": { "content": "running shoes" } }
        }
      },
      "ruleset_ids": ["my-ruleset"],
      "match_criteria": {
        "query_string": "running shoes"
      }
    }
  }
}
```

### Composing Retrievers

Retrievers can be nested. A common production pattern: RRF or linear fusion → rerank → pin business results:

```json
POST my-index/_search
{
  "retriever": {
    "text_similarity_reranker": {
      "retriever": {
        "rrf": {
          "retrievers": [
            { "standard": { "query": { "match": { "title": "elasticsearch guide" } } } },
            { "knn": { "field": "embedding", "query_vector": [...], "k": 20, "num_candidates": 100 } }
          ],
          "rank_window_size": 100
        }
      },
      "field": "content",
      "inference_id": "my-reranker",
      "inference_text": "elasticsearch guide",
      "rank_window_size": 50
    }
  }
}
```

---

## ELSER

### Overview

ELSER (Elastic Learned Sparse Encoder) generates sparse vector representations for semantic search without needing external embedding models. ELSER ships by default in 9.0+ alongside the e5 multilingual dense model and Elastic Rerank.

> **9.x Deprecation**: The `elser` inference service is deprecated. Use the `elasticsearch` service to access ELSER models through the inference API instead.

### Setup (8.x)

```json
PUT _ml/trained_models/.elser_model_2
{
  "input": { "field_names": ["text_field"] }
}

POST _ml/trained_models/.elser_model_2/deployment/_start
{
  "number_of_allocations": 1,
  "threads_per_allocation": 1
}
```

### Setup (9.x — via Inference API)

```json
PUT _inference/sparse_embedding/my-elser
{
  "service": "elasticsearch",
  "service_settings": {
    "model_id": ".elser_model_2",
    "num_allocations": 1,
    "num_threads": 1
  }
}
```

### Mapping with ELSER

```json
{
  "mappings": {
    "properties": {
      "content": { "type": "text" },
      "content_embedding": { "type": "sparse_vector" }
    }
  }
}
```

### Ingest Pipeline for ELSER

```json
PUT _ingest/pipeline/elser-pipeline
{
  "processors": [
    {
      "inference": {
        "model_id": ".elser_model_2",
        "input_output": [
          {
            "input_field": "content",
            "output_field": "content_embedding"
          }
        ]
      }
    }
  ]
}
```

### Querying with ELSER

```json
POST my-index/_search
{
  "query": {
    "sparse_vector": {
      "field": "content_embedding",
      "inference_id": ".elser_model_2",
      "query": "how does machine learning work"
    }
  }
}
```

---

## Inference API

### Configuring an Inference Endpoint (8.15+)

```json
PUT _inference/text_embedding/my-embeddings
{
  "service": "elasticsearch",
  "service_settings": {
    "model_id": ".multilingual-e5-small",
    "num_allocations": 1,
    "num_threads": 1
  }
}
```

### Using with Semantic Text Field (8.15+)

```json
{
  "mappings": {
    "properties": {
      "content": {
        "type": "semantic_text",
        "inference_id": "my-embeddings"
      }
    }
  }
}
```

Semantic text fields automatically chunk, embed, and enable search — no separate pipeline needed.

### Querying Semantic Text

```json
POST my-index/_search
{
  "query": {
    "semantic": {
      "field": "content",
      "query": "how does elasticsearch handle vector search"
    }
  }
}
```

---

## Quantization and Performance

### Scalar Quantization (8.14+)

Reduces memory by ~4x with minimal recall loss.

```json
{
  "mappings": {
    "properties": {
      "embedding": {
        "type": "dense_vector",
        "dims": 768,
        "index": true,
        "similarity": "cosine",
        "index_options": {
          "type": "int8_hnsw"
        }
      }
    }
  }
}
```

### Binary Quantization (8.16+)

Reduces memory by ~32x, best for high-dimensional embeddings with rescoring.

```json
{
  "index_options": {
    "type": "bbq_hnsw"
  }
}
```

### Quantization Comparison

| Method | Memory Reduction | Recall Impact | Best For | Availability |
|--------|-----------------|---------------|----------|-------------|
| Full `float32` | Baseline | Best | Small-medium scale | All versions |
| `bfloat16` | ~2x | Negligible | Medium scale, balanced | 9.3+ |
| `int8_hnsw` | ~4x | Minimal | General production | 8.14+ |
| `int4_hnsw` | ~8x | Small | Large-scale, cost-sensitive | 8.16+ |
| `bbq_hnsw` | ~32x | Moderate (rescore helps) | Very large scale, in-memory | 9.0+ (GA) |
| `bbq_disk` | ~32x | Moderate (rescore helps) | Very large scale, disk-based | 9.2+ |

### bfloat16 Element Type (9.3+)

```json
{
  "mappings": {
    "properties": {
      "embedding": {
        "type": "dense_vector",
        "dims": 768,
        "index": true,
        "similarity": "cosine",
        "element_type": "bfloat16"
      }
    }
  }
}
```

### DiskBBQ — On-Disk Binary Quantization (9.2+)

Lower memory usage than in-memory BBQ by storing quantized vectors on disk:

```json
{
  "index_options": {
    "type": "bbq_disk"
  }
}
```

Best for very large vector datasets where memory cost of `bbq_hnsw` is still too high. Slightly higher latency than in-memory BBQ.

### Rescoring Pattern for Quantized Vectors

Use oversampling with rescoring to recover recall lost from quantization:

```json
POST my-index/_search
{
  "knn": {
    "field": "embedding",
    "query_vector": [0.1, 0.2, ...],
    "k": 10,
    "num_candidates": 200,
    "rescore_vector": {
      "oversample": 2.0
    }
  }
}
```

---

## ColPali and ColBERT

### Overview (9.0+)

ColPali and ColBERT are multi-stage interaction models that work with MaxSim (Maximum Similarity) scoring. Unlike single-vector models that compress an entire document into one embedding, these models produce token-level embeddings and compute fine-grained similarity.

- **ColBERT**: Token-level text embeddings — each token in query and document gets its own vector. MaxSim computes the maximum similarity between each query token and all document tokens, then sums the scores.
- **ColPali**: Extends the concept to visual document understanding — produces embeddings from document images (PDFs, slides) without OCR.

### When to Use

| Approach | Best For | Trade-off |
|----------|----------|-----------|
| Single dense vector | General semantic search | Fast, low memory |
| ColBERT/MaxSim | High-precision text retrieval | Higher memory, better recall |
| ColPali/MaxSim | Visual document search (PDFs, images) | Highest memory, no OCR needed |

These models are configured through the Inference API and can be combined with other retrievers for multi-stage retrieval pipelines.
