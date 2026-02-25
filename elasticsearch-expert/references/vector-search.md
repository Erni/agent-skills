# Vector Search and AI Integration

## Table of Contents
1. [Dense Vector Configuration](#dense-vector-configuration)
2. [kNN Search Patterns](#knn-search-patterns)
3. [Hybrid Search](#hybrid-search)
4. [ELSER (Learned Sparse Encoder)](#elser)
5. [Inference API](#inference-api)
6. [Quantization and Performance](#quantization-and-performance)

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

## ELSER

### Overview

ELSER (Elastic Learned Sparse Encoder) generates sparse vector representations for semantic search without needing external embedding models.

### Setup

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

| Method | Memory Reduction | Recall Impact | Best For |
|--------|-----------------|---------------|----------|
| Full float32 | Baseline | Best | Small-medium scale |
| `int8_hnsw` | ~4x | Minimal | General production |
| `int4_hnsw` | ~8x | Small | Large-scale, cost-sensitive |
| `bbq_hnsw` | ~32x | Moderate (rescore helps) | Very large scale |

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
