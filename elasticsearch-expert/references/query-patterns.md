# Query Optimization Patterns

## Table of Contents
1. [Query DSL Best Practices](#query-dsl-best-practices)
2. [Bool Query Composition](#bool-query-composition)
3. [Aggregation Patterns](#aggregation-patterns)
4. [Pagination Strategies](#pagination-strategies)
5. [ES|QL Patterns](#esql-patterns)
6. [Performance Anti-Patterns](#performance-anti-patterns)
7. [Query Profiling](#query-profiling)

---

## Query DSL Best Practices

### Filter vs Query Context

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "elasticsearch guide" } }
      ],
      "filter": [
        { "term": { "status": "published" } },
        { "range": { "publish_date": { "gte": "2024-01-01" } } }
      ]
    }
  }
}
```

**Rule**: Any clause that does not need relevance scoring belongs in `filter`. Filters are cached and skip scoring.

### Term-Level vs Full-Text Queries

| Query Type | Use For | Field Type |
|-----------|---------|-----------|
| `term` | Exact value match | `keyword`, `boolean`, `integer` |
| `terms` | Match any from a list | `keyword` |
| `range` | Numeric/date ranges | `date`, `long`, `double` |
| `match` | Full-text search | `text` |
| `match_phrase` | Exact phrase in text | `text` |
| `multi_match` | Search across multiple text fields | `text` |
| `query_string` | Complex query syntax | `text` (use with caution) |
| `wildcard` | Pattern matching | `keyword` (avoid on `text`) |
| `prefix` | Starts-with | `keyword` or `text` with edge_ngram |
| `exists` | Field exists | Any |

### Efficient Multi-Field Search

```json
{
  "query": {
    "multi_match": {
      "query": "search terms",
      "type": "best_fields",
      "fields": ["title^3", "description^2", "body"],
      "tie_breaker": 0.3
    }
  }
}
```

Use `cross_fields` type when multiple fields represent a single concept (e.g., first_name + last_name).

---

## Bool Query Composition

### Scoring vs Non-Scoring

```json
{
  "query": {
    "bool": {
      "must": [],
      "should": [],
      "filter": [],
      "must_not": []
    }
  }
}
```

| Clause | Scores? | Cached? | Use For |
|--------|---------|---------|---------|
| `must` | Yes | No | Primary relevance criteria |
| `should` | Yes | No | Boosting / optional criteria |
| `filter` | No | Yes | Mandatory constraints (status, date range) |
| `must_not` | No | Yes | Exclusions |

### Minimum Should Match

```json
{
  "query": {
    "bool": {
      "should": [
        { "term": { "tags": "elasticsearch" } },
        { "term": { "tags": "search" } },
        { "term": { "tags": "database" } }
      ],
      "minimum_should_match": 2
    }
  }
}
```

### Boosting Pattern (Prefer Over Negative `must_not`)

```json
{
  "query": {
    "boosting": {
      "positive": { "match": { "title": "elasticsearch" } },
      "negative": { "term": { "status": "deprecated" } },
      "negative_boost": 0.5
    }
  }
}
```

---

## Aggregation Patterns

### Composite Aggregation for Pagination

```json
{
  "size": 0,
  "aggs": {
    "my_buckets": {
      "composite": {
        "size": 100,
        "sources": [
          { "user": { "terms": { "field": "user_id" } } },
          { "date": { "date_histogram": { "field": "timestamp", "calendar_interval": "day" } } }
        ],
        "after": { "user": "user_123", "date": 1704067200000 }
      }
    }
  }
}
```

### Sampler for High-Cardinality Approximation

```json
{
  "aggs": {
    "sample": {
      "sampler": { "shard_size": 200 },
      "aggs": {
        "keywords": {
          "significant_terms": { "field": "tags" }
        }
      }
    }
  }
}
```

### Avoiding Expensive Aggregations

- Set `size` on `terms` aggs (default is 10, never set to MAX_INT)
- Use `execution_hint: "map"` for low-cardinality terms aggs
- Use `filter` agg to narrow data before expensive sub-aggs
- Prefer `cardinality` (HyperLogLog, approximate) over `value_count` for distinct counts at scale

### Multi-Level Aggregation Template

```json
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "range": { "timestamp": { "gte": "now-30d" } } }
      ]
    }
  },
  "aggs": {
    "by_category": {
      "terms": { "field": "category", "size": 20 },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } },
        "price_ranges": {
          "range": {
            "field": "price",
            "ranges": [
              { "to": 50 },
              { "from": 50, "to": 200 },
              { "from": 200 }
            ]
          }
        }
      }
    }
  }
}
```

---

## Pagination Strategies

### Comparison

| Method | Max Depth | Consistent? | Use Case |
|--------|----------|-------------|----------|
| `from`/`size` | 10,000 | No | Simple UI pagination |
| `search_after` + PIT | Unlimited | Yes | Deep pagination, scroll replacement |
| `scroll` | Unlimited | Yes (snapshot) | Bulk export (deprecated for search) |

### search_after with Point in Time

```json
POST my-index/_pit?keep_alive=5m

POST _search
{
  "size": 100,
  "query": { "match_all": {} },
  "pit": { "id": "<pit_id>", "keep_alive": "5m" },
  "sort": [
    { "timestamp": "desc" },
    { "_shard_doc": "asc" }
  ],
  "search_after": [1704067200000, 42]
}
```

Always include `_shard_doc` as tiebreaker sort for deterministic pagination.

---

## ES|QL Patterns

### Basic Analytics

```esql
FROM logs-*
| WHERE @timestamp >= NOW() - 1 hour AND log.level == "ERROR"
| STATS error_count = COUNT(*) BY service.name
| SORT error_count DESC
| LIMIT 10
```

### Enrichment and Transformation

```esql
FROM orders
| WHERE order_date >= "2024-01-01"
| EVAL revenue = quantity * unit_price,
       quarter = DATE_FORMAT(order_date, "yyyy-'Q'Q")
| STATS total_revenue = SUM(revenue),
         avg_order = AVG(revenue),
         order_count = COUNT(*)
  BY quarter, region
| SORT quarter, total_revenue DESC
```

### Pattern Matching

```esql
FROM access-logs
| WHERE url LIKE "/api/v*/users/*"
| GROK url "/api/v%{INT:api_version}/users/%{WORD:action}"
| STATS request_count = COUNT(*) BY api_version, action
```

### Joining with ENRICH

```esql
FROM network-logs
| ENRICH geo-policy ON source_ip WITH country, city, latitude, longitude
| STATS connection_count = COUNT(*) BY country
| SORT connection_count DESC
| LIMIT 20
```

### LOOKUP JOIN (9.0+ Tech Preview)

Cross-index joins not available in traditional Query DSL:

```esql
FROM employees
| LOOKUP JOIN departments ON department_id
| STATS avg_salary = AVG(salary) BY department_name
| SORT avg_salary DESC
```

Multi-field LOOKUP JOIN (9.2+):

```esql
FROM orders
| LOOKUP JOIN products ON product_id, region
| EVAL total = quantity * unit_price
| STATS revenue = SUM(total) BY category
```

### INLINE STATS (9.2+ Tech Preview)

Compute statistics inline without collapsing rows:

```esql
FROM logs-*
| WHERE @timestamp >= NOW() - 1 hour
| INLINE STATS avg_duration = AVG(event.duration) BY service.name
| EVAL deviation = event.duration - avg_duration
| WHERE deviation > avg_duration * 2
| SORT deviation DESC
| LIMIT 20
```

### KQL Function in ES|QL (9.0+ Tech Preview)

Use KQL syntax within ES|QL for familiar Kibana query patterns:

```esql
FROM logs-*
| WHERE KQL("service.name: api-gateway AND log.level: ERROR")
| STATS error_count = COUNT(*) BY url.path
| SORT error_count DESC
| LIMIT 10
```

---

## Geo Queries

### Geo Distance (Find within Radius)

```json
{
  "query": {
    "bool": {
      "filter": {
        "geo_distance": {
          "distance": "10km",
          "location": {
            "lat": 40.7128,
            "lon": -74.0060
          }
        }
      }
    }
  },
  "sort": [
    {
      "_geo_distance": {
        "location": { "lat": 40.7128, "lon": -74.0060 },
        "order": "asc",
        "unit": "km"
      }
    }
  ]
}
```

### Geo Bounding Box

```json
{
  "query": {
    "bool": {
      "filter": {
        "geo_bounding_box": {
          "location": {
            "top_left": { "lat": 41.0, "lon": -74.5 },
            "bottom_right": { "lat": 40.5, "lon": -73.5 }
          }
        }
      }
    }
  }
}
```

**Note**: Always use geo queries in `filter` context — geo calculations are expensive and don't contribute to relevance scoring.

---

## Function Score

### Custom Scoring with Decay

Boost results closer to a reference point (location, date, or number):

```json
{
  "query": {
    "function_score": {
      "query": { "match": { "title": "restaurant" } },
      "functions": [
        {
          "gauss": {
            "location": {
              "origin": { "lat": 40.7128, "lon": -74.0060 },
              "scale": "5km",
              "decay": 0.5
            }
          }
        },
        {
          "gauss": {
            "publish_date": {
              "origin": "now",
              "scale": "30d",
              "decay": 0.5
            }
          }
        }
      ],
      "score_mode": "multiply",
      "boost_mode": "multiply"
    }
  }
}
```

### Field Value Factor

Boost by a numeric field (e.g., popularity):

```json
{
  "query": {
    "function_score": {
      "query": { "match": { "title": "search term" } },
      "field_value_factor": {
        "field": "popularity",
        "factor": 1.2,
        "modifier": "log1p",
        "missing": 1
      }
    }
  }
}
```

| Modifier | Formula | Use When |
|----------|---------|----------|
| `none` | `field * factor` | Linear boost |
| `log1p` | `log(1 + field * factor)` | Diminishing returns (recommended for popularity) |
| `log2p` | `log(2 + field * factor)` | Stronger diminishing returns |
| `sqrt` | `sqrt(field * factor)` | Moderate diminishing returns |

---

## Performance Anti-Patterns

### 1. Wildcard Leading Queries
```json
{ "wildcard": { "title": "*search*" } }
```
**Problem**: Cannot use inverted index, scans all terms.
**Fix**: Use `ngram` tokenizer or `match` queries.

### 2. Script Scoring on Large Result Sets
```json
{
  "query": {
    "script_score": {
      "query": { "match_all": {} },
      "script": { "source": "doc['popularity'].value * 2" }
    }
  }
}
```
**Problem**: Runs script for every matching document.
**Fix**: Narrow with a `filter` first, or pre-compute and store the value.

### 3. High `from` Values
```json
{ "from": 9990, "size": 10 }
```
**Problem**: Elasticsearch must fetch and score 10,000 documents on each shard.
**Fix**: Use `search_after` with PIT.

### 4. Terms Aggregation Without Size Limit on High-Cardinality Fields
```json
{ "aggs": { "all_users": { "terms": { "field": "user_id" } } } }
```
**Problem**: Default size=10 may miss buckets; large sizes cause memory pressure.
**Fix**: Use `composite` aggregation for iteration, or `cardinality` for counts.

### 5. Deeply Nested Bool Queries
**Problem**: Many nested bool layers are hard to optimize and often indicate flawed query construction.
**Fix**: Flatten into a single `bool` with appropriate clauses.

---

## Query Profiling

### Enable Profile API

```json
POST my-index/_search
{
  "profile": true,
  "query": {
    "match": { "title": "elasticsearch" }
  }
}
```

### Key Metrics to Check
- `time_in_nanos` per query component — identify slowest clause
- `breakdown.build_scorer` — high values indicate expensive scoring
- `breakdown.advance` — high values indicate expensive iteration
- `collector` times — check for unnecessary collecting overhead

### Slow Query Log

```json
PUT my-index/_settings
{
  "index.search.slowlog.threshold.query.warn": "10s",
  "index.search.slowlog.threshold.query.info": "5s",
  "index.search.slowlog.threshold.fetch.warn": "1s",
  "index.search.slowlog.level": "info"
}
```
