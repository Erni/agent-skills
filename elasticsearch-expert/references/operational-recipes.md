# Operational Troubleshooting Recipes

Runbook-style patterns for diagnosing and resolving common Elasticsearch cluster issues.

---

## Table of Contents
1. [Unassigned Shards](#unassigned-shards)
2. [Disk Watermark Issues](#disk-watermark-issues)
3. [Circuit Breaker Trips](#circuit-breaker-trips)
4. [Slow Query Investigation](#slow-query-investigation)
5. [Mapping Explosion](#mapping-explosion)
6. [High JVM Heap Pressure](#high-jvm-heap-pressure)
7. [SRE Aggregation Recipes](#sre-aggregation-recipes)

---

## Unassigned Shards

### Step 1: Identify Unassigned Shards

```json
GET _cat/shards?v&h=index,shard,prirep,state,unassigned.reason&s=state
```

### Step 2: Get Allocation Explanation

```json
GET _cluster/allocation/explain
{
  "index": "my-index",
  "shard": 0,
  "primary": true
}
```

### Step 3: Common Causes and Fixes

| Cause | Indicator | Fix |
|-------|-----------|-----|
| **Disk watermark exceeded** | `NODE_WITH_ENOUGH_DISK_SPACE` | Free disk or adjust watermarks |
| **Shard allocation filtered** | `DECIDERS_NO` with allocation filter | Check `index.routing.allocation.*` settings |
| **Too many shards per node** | `MAX_SHARDS_PER_NODE` | Increase `cluster.max_shards_per_node` or reduce shards |
| **Awareness attribute missing** | `AWARENESS` | Check `cluster.routing.allocation.awareness.*` |
| **Node left cluster** | `NODE_LEFT` | Restart node or wait for timeout |
| **Corrupt shard** | `ALLOCATION_FAILED` | Use `_cluster/reroute` with `allocate_stale_primary` (data loss risk) |

### Step 4: Force Allocation (Last Resort)

```json
POST _cluster/reroute
{
  "commands": [
    {
      "allocate_stale_primary": {
        "index": "my-index",
        "shard": 0,
        "node": "node-1",
        "accept_data_loss": true
      }
    }
  ]
}
```

---

## Disk Watermark Issues

### Check Current Disk Usage

```json
GET _cat/allocation?v&h=node,disk.used,disk.avail,disk.total,disk.percent
```

### Check Current Watermark Settings

```json
GET _cluster/settings?include_defaults&filter_path=*.cluster.routing.allocation.disk*
```

### Watermark Thresholds

| Watermark | Default | Effect |
|-----------|---------|--------|
| Low | 85% | No new shards allocated to this node |
| High | 90% | Shards start relocating away from this node |
| Flood stage | 95% | Affected indices become read-only (`index.blocks.read_only_allow_delete`) |

### Resolution Steps

1. **Immediate relief** — delete old indices or snapshots:
```json
DELETE old-logs-2024.01.*
```

2. **Unblock flood-stage indices**:
```json
PUT my-index/_settings
{
  "index.blocks.read_only_allow_delete": null
}
```

3. **Adjust watermarks** (temporary — address root cause):
```json
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "90%",
    "cluster.routing.allocation.disk.watermark.high": "95%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "97%"
  }
}
```

4. **Long-term** — implement ILM with delete phase, add storage, or enable searchable snapshots for cold/frozen tiers.

---

## Circuit Breaker Trips

### Check Circuit Breaker Stats

```json
GET _nodes/stats/breaker
```

### Common Breakers

| Breaker | Default Limit | Typical Cause |
|---------|--------------|---------------|
| `parent` | 95% of JVM heap | Overall memory pressure |
| `fielddata` | 40% of JVM heap | High-cardinality text field aggregations |
| `request` | 60% of JVM heap | Large aggregation responses |
| `in_flight_requests` | 100% of JVM heap | Too many concurrent large requests |

### Resolution

1. **Fielddata breaker** — avoid `fielddata: true` on text fields. Use `keyword` sub-fields for aggregations:
```json
GET _cat/fielddata?v&h=node,field,size&s=size:desc
```

2. **Request breaker** — reduce aggregation cardinality, add `size` limits to `terms` aggs, use `composite` for pagination.

3. **Parent breaker** — increase heap (up to 31 GB max), reduce concurrent queries, or add coordinating-only nodes.

### Adjust Breaker Limits (Temporary)

```json
PUT _cluster/settings
{
  "transient": {
    "indices.breaker.fielddata.limit": "50%",
    "indices.breaker.request.limit": "70%"
  }
}
```

---

## Slow Query Investigation

### Step 1: Enable Slow Query Logs

```json
PUT my-index/_settings
{
  "index.search.slowlog.threshold.query.warn": "10s",
  "index.search.slowlog.threshold.query.info": "5s",
  "index.search.slowlog.threshold.query.debug": "2s",
  "index.search.slowlog.threshold.fetch.warn": "1s",
  "index.search.slowlog.threshold.fetch.info": "800ms",
  "index.search.slowlog.level": "info"
}
```

### Step 2: Profile a Specific Query

```json
POST my-index/_search
{
  "profile": true,
  "query": {
    "match": { "title": "slow query example" }
  }
}
```

### Step 3: Interpret Profile Output

| Metric | High Value Indicates |
|--------|---------------------|
| `build_scorer` | Expensive scoring (complex scripts, many terms) |
| `advance` | Expensive iteration (wildcard, regex, leading wildcards) |
| `match` | Expensive matching (phrase queries on large fields) |
| `next_doc` | Many documents evaluated (consider adding filters) |

### Step 4: Check Hot Threads

```json
GET _nodes/hot_threads?threads=5
```

### Common Fixes

1. Move non-scoring clauses to `filter` context
2. Replace leading wildcard queries with `ngram` tokenizer
3. Add `size` limits to aggregations
4. Use `search_after` instead of deep `from`/`size`
5. Pre-compute expensive script values as indexed fields
6. Increase `refresh_interval` for write-heavy indices

---

## Mapping Explosion

### Detect

```json
GET _cluster/stats?filter_path=indices.mappings.field_types
```

Per-index field count:
```json
GET my-index/_mapping?filter_path=*.mappings._meta
GET _cat/indices?v&h=index,docs.count,store.size&s=store.size:desc
```

### Check Current Limit

```json
GET my-index/_settings?filter_path=*.settings.index.mapping.total_fields.limit
```

Default is 1000. If you need to raise it frequently, the data model needs restructuring.

### Prevention Strategies

1. **Use `dynamic: "strict"`** — reject unknown fields
2. **Use `flattened` type** — for arbitrary key-value data
3. **Use `dynamic_templates`** — route unknown fields to appropriate types
4. **Restructure data** — move variable fields into a key-value array pattern:

```json
{
  "attributes": [
    { "key": "color", "value": "red" },
    { "key": "size", "value": "large" }
  ]
}
```

---

## High JVM Heap Pressure

### Check Heap Usage

```json
GET _nodes/stats/jvm?filter_path=nodes.*.jvm.mem
```

```json
GET _cat/nodes?v&h=name,heap.percent,heap.max,ram.percent,cpu
```

### Common Causes

| Cause | Diagnostic | Fix |
|-------|-----------|-----|
| Fielddata cache | `GET _cat/fielddata?v` | Use keyword fields for aggs |
| Too many shards | `GET _cat/shards?v \| wc -l` | Merge indices, increase shard size |
| Large aggregations | Check slow query log | Add filters, reduce cardinality |
| Bulk queue pressure | `GET _cat/thread_pool/write?v` | Reduce bulk size, add data nodes |
| Too many concurrent queries | `GET _cat/thread_pool/search?v` | Add coordinating nodes |

### GC Troubleshooting

Watch for GC pauses in logs:
```
[gc][123] overhead, spent [5.2s] collecting in the last [6s]
```

If frequent: increase heap (max 31 GB), reduce fielddata, close unused indices, reduce shard count.

---

## SRE Aggregation Recipes

### Error Rate Dashboard (per 15-minute window)

```json
POST logs-*/_search?size=0
{
  "query": {
    "bool": {
      "filter": [
        { "range": { "@timestamp": { "gte": "now-24h" } } }
      ]
    }
  },
  "aggs": {
    "over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "15m"
      },
      "aggs": {
        "total": { "value_count": { "field": "_index" } },
        "errors": {
          "filter": { "term": { "log.level": "ERROR" } }
        },
        "error_rate": {
          "bucket_script": {
            "buckets_path": {
              "errors": "errors._count",
              "total": "total"
            },
            "script": "params.total > 0 ? params.errors / params.total * 100 : 0"
          }
        }
      }
    }
  }
}
```

### Top-N Endpoints Leaderboard

```json
POST logs-*/_search?size=0
{
  "query": {
    "bool": {
      "filter": [
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "aggs": {
    "top_endpoints": {
      "terms": {
        "field": "url.path",
        "size": 20,
        "order": { "request_count": "desc" }
      },
      "aggs": {
        "request_count": { "value_count": { "field": "_index" } },
        "avg_latency": { "avg": { "field": "event.duration" } },
        "p95_latency": { "percentiles": { "field": "event.duration", "percents": [95] } },
        "error_count": {
          "filter": {
            "range": { "http.response.status_code": { "gte": 500 } }
          }
        }
      }
    }
  }
}
```

### Service Health Summary

```json
POST logs-*/_search?size=0
{
  "query": {
    "bool": {
      "filter": [
        { "range": { "@timestamp": { "gte": "now-5m" } } }
      ]
    }
  },
  "aggs": {
    "by_service": {
      "terms": { "field": "service.name", "size": 50 },
      "aggs": {
        "log_levels": {
          "terms": { "field": "log.level" }
        },
        "error_rate": {
          "filters": {
            "filters": {
              "errors": { "terms": { "log.level": ["ERROR", "FATAL"] } },
              "all": { "match_all": {} }
            }
          }
        }
      }
    }
  }
}
```
