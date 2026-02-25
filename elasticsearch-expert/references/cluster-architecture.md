# Cluster Architecture Guide

## Table of Contents
1. [Shard Sizing Strategy](#shard-sizing-strategy)
2. [Node Roles and Topology](#node-roles-and-topology)
3. [Data Tiers and ILM](#data-tiers-and-ilm)
4. [Index Strategies](#index-strategies)
5. [Capacity Planning](#capacity-planning)
6. [High Availability](#high-availability)
7. [Ingest Pipeline Patterns](#ingest-pipeline-patterns)
8. [Production Checklist](#production-checklist)

---

## Shard Sizing Strategy

### Target Metrics

| Metric | Target | Why |
|--------|--------|-----|
| Shard size | 10-50 GB | Smaller shards waste overhead; larger slow recovery |
| Shards per node | < 1000 | Each shard consumes memory for metadata |
| Shards per GB heap | < 20 | Rule of thumb for heap pressure |
| Max docs per shard | ~2 billion | Lucene limit (2^31 - 1) |

### Calculating Primary Shards

```
primary_shards = ceil(expected_index_size_gb / target_shard_size_gb)
```

**Example**: 200 GB daily index, targeting 40 GB shards = 5 primary shards.

### Time-Based Index Shard Planning

For logs/metrics ingested at a known daily rate:

```
daily_volume_gb = events_per_second * avg_event_bytes * 86400 / 1e9
primary_shards = ceil(daily_volume_gb / 40)
```

### Oversharding Symptoms
- High heap usage relative to data volume
- Slow cluster state updates
- Long recovery times during node restarts
- Many small segments that don't merge efficiently

---

## Node Roles and Topology

### Role Definitions

| Role | Purpose | Heap | Disk | CPU |
|------|---------|------|------|-----|
| `master` | Cluster state management | 1-4 GB | Minimal | Low |
| `data_hot` | Ingest and recent queries | 30 GB max | Fast SSD/NVMe | High |
| `data_warm` | Older, less-queried data | 30 GB max | SSD | Medium |
| `data_cold` | Infrequent access | 30 GB max | HDD acceptable | Low |
| `data_frozen` | Searchable snapshots (rarely accessed) | 4-8 GB | Local cache only | Low |
| `ingest` | Pipeline processing | 4-8 GB | Minimal | Medium-High |
| `coordinating` | Request routing and reduce | 8-16 GB | Minimal | Medium |
| `ml` | Machine learning jobs | 8-30 GB | Minimal | High |
| `transform` | Continuous transforms | 4-8 GB | Minimal | Medium |

### Minimum Production Topology

```
3x dedicated master nodes (small instances)
N x data_hot nodes (sized to ingest throughput)
N x data_warm nodes (sized to warm data volume)
Optional: coordinating-only nodes for heavy search
Optional: ingest nodes for complex pipelines
```

### Node Configuration Example

```yaml
# elasticsearch.yml - hot data node
node.roles: [data_hot, ingest]
cluster.name: production
node.name: hot-data-01

# Heap: set in jvm.options (50% of RAM, max 31 GB)
# -Xms30g
# -Xmx30g
```

---

## Data Tiers and ILM

### ILM Policy Template

```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_primary_shard_size": "40gb",
            "max_age": "1d"
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 },
          "allocate": { "number_of_replicas": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": { "number_of_replicas": 0 },
          "set_priority": { "priority": 0 }
        }
      },
      "frozen": {
        "min_age": "90d",
        "actions": {
          "searchable_snapshot": {
            "snapshot_repository": "my-s3-repo"
          }
        }
      },
      "delete": {
        "min_age": "365d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

### Data Stream Setup

```json
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "data_stream": {},
  "composed_of": ["logs-mappings", "logs-settings"],
  "priority": 200,
  "template": {
    "settings": {
      "index.lifecycle.name": "logs-policy",
      "number_of_shards": 3,
      "number_of_replicas": 1
    }
  }
}

PUT _component_template/logs-settings
{
  "template": {
    "settings": {
      "index.codec": "best_compression",
      "index.refresh_interval": "30s"
    }
  }
}
```

---

## Index Strategies

### Time-Based Indices (Data Streams)

Best for: logs, metrics, events, audit trails.

```
logs-app-2024.01.15  (hot)
logs-app-2024.01.14  (warm)
logs-app-2023.12.15  (cold)
logs-app-2023.09.15  (frozen via searchable snapshot)
```

### Entity-Based Indices

Best for: catalogs, user profiles, configuration.

```
products-v2         (current version)
products-v1         (being migrated)
users               (single long-lived index)
```

Use aliases for zero-downtime reindexing between versions.

### Rollover-Based Strategy

Prefer rollover over calendar-based indices — shards sizes stay consistent.

```json
POST logs-app/_rollover
{
  "conditions": {
    "max_primary_shard_size": "40gb",
    "max_age": "1d",
    "max_docs": 100000000
  }
}
```

---

## Capacity Planning

### Sizing Formula

```
total_storage = source_data_gb * (1 + number_of_replicas) * (1 + indexing_overhead)
```

Where `indexing_overhead` is typically 0.1 to 0.3 (10-30%) depending on mappings and analyzers.

### Memory Planning

```
heap_per_node = min(50% of RAM, 31 GB)
remaining_ram = filesystem cache (critical for search performance)
```

**Important**: Never allocate more than 50% of RAM to heap. The OS filesystem cache is essential for Lucene segment performance.

### Disk Watermarks

| Watermark | Default | Effect |
|-----------|---------|--------|
| Low | 85% | No new shards allocated |
| High | 90% | Shards start relocating away |
| Flood stage | 95% | Index becomes read-only |

Set in cluster settings:

```json
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.disk.watermark.low": "85%",
    "cluster.routing.allocation.disk.watermark.high": "90%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "95%"
  }
}
```

---

## High Availability

### Cross-Cluster Replication (CCR)

```json
PUT _ccr/auto_follow/logs-replication
{
  "remote_cluster": "cluster-b",
  "leader_index_patterns": ["logs-*"],
  "follow_index_pattern": "{{leader_index}}-replicated"
}
```

### Snapshot Repository

```json
PUT _snapshot/s3-backup
{
  "type": "s3",
  "settings": {
    "bucket": "my-es-backups",
    "region": "eu-west-1",
    "base_path": "production",
    "max_snapshot_bytes_per_sec": "200mb"
  }
}
```

### Snapshot Lifecycle Management (SLM)

```json
PUT _slm/policy/daily-snapshots
{
  "schedule": "0 30 1 * * ?",
  "name": "<daily-snapshot-{now/d}>",
  "repository": "s3-backup",
  "config": {
    "indices": ["*"],
    "ignore_unavailable": true,
    "include_global_state": false
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}
```

---

## Ingest Pipeline Patterns

### Common Pipeline

```json
PUT _ingest/pipeline/logs-pipeline
{
  "description": "Standard log processing",
  "processors": [
    {
      "date": {
        "field": "log_timestamp",
        "formats": ["ISO8601", "yyyy-MM-dd HH:mm:ss"],
        "target_field": "@timestamp"
      }
    },
    {
      "grok": {
        "field": "message",
        "patterns": ["%{COMBINEDAPACHELOG}"]
      }
    },
    {
      "geoip": {
        "field": "clientip",
        "target_field": "geo"
      }
    },
    {
      "user_agent": {
        "field": "agent",
        "target_field": "user_agent"
      }
    },
    {
      "remove": {
        "field": ["log_timestamp", "agent"],
        "ignore_missing": true
      }
    }
  ],
  "on_failure": [
    {
      "set": {
        "field": "_index",
        "value": "failed-logs"
      }
    },
    {
      "set": {
        "field": "error.message",
        "value": "{{ _ingest.on_failure_message }}"
      }
    }
  ]
}
```

### Enrich Processor Pattern

```json
PUT _enrich/policy/user-enrich
{
  "match": {
    "indices": "users",
    "match_field": "user_id",
    "enrich_fields": ["full_name", "department", "role"]
  }
}

POST _enrich/policy/user-enrich/_execute
```

---

## Production Checklist

### Before Go-Live

- [ ] Dedicated master nodes (3 or 5, odd number)
- [ ] Heap set to 50% of RAM, max 31 GB
- [ ] `bootstrap.memory_lock: true` (no swapping)
- [ ] File descriptor limit >= 65536
- [ ] Max map count >= 262144 (`vm.max_map_count`)
- [ ] Disk watermarks configured
- [ ] Snapshot repository configured and tested
- [ ] SLM policy active
- [ ] ILM policies for all time-based indices
- [ ] Index templates (composable) for all index patterns
- [ ] Slow query and slow index logs configured
- [ ] Circuit breaker settings reviewed
- [ ] Thread pool queues sized (avoid unbounded queues)
- [ ] Security enabled (TLS, authentication, RBAC)
- [ ] Monitoring enabled (Stack Monitoring or Elastic Agent)
- [ ] `action.destructive_requires_name: true`
- [ ] Recovery throttling configured for large clusters
