# Index Mapping Design Guide

## Table of Contents
1. [Field Type Selection](#field-type-selection)
2. [Multi-Field Patterns](#multi-field-patterns)
3. [Dynamic Templates](#dynamic-templates)
4. [Nested vs Flattened vs Join](#nested-vs-flattened-vs-join)
5. [Mapping Evolution Strategies](#mapping-evolution-strategies)
6. [Common Pitfalls](#common-pitfalls)

---

## Field Type Selection

### Decision Matrix

| Use Case | Field Type | Notes |
|----------|-----------|-------|
| Exact match / aggregation | `keyword` | Max 32,766 bytes by default (ignore_above) |
| Full-text search | `text` | Analyzed, cannot aggregate without fielddata |
| Both exact + full-text | `text` with `keyword` sub-field | Standard multi-field pattern |
| Numeric range queries | `long`, `integer`, `double` | Use smallest type that fits |
| Numeric exact match only | `keyword` | Better performance if no range queries needed |
| Date/time | `date` or `date_nanos` | Always define `format` explicitly |
| Boolean | `boolean` | Avoid keyword for true/false values |
| IP addresses | `ip` | Supports CIDR notation in queries |
| Geo coordinates | `geo_point` | Lat/lon pair |
| Geo shapes | `geo_shape` | Polygons, lines, etc. |
| Embeddings / vectors | `dense_vector` | Specify `dims` and `similarity` |
| Sparse vectors (ELSER) | `sparse_vector` | For learned sparse models |
| Large unstructured JSON | `flattened` | Reduces mapping explosion risk |
| Histograms / pre-aggregated | `histogram` | For pre-computed percentiles |
| Bounded range values | `integer_range`, `date_range` | Meetings, reservations |
| Log messages (9.3+) | `pattern_text` | Decomposed log storage, ~50% reduction |
| OTel histograms (9.3+) | `exponential_histogram` | Pre-computed OpenTelemetry histograms |
| Pre-aggregated keywords | `counted_keyword` | Pre-aggregated keyword counts |

### Numeric Type Guidance

```json
{
  "mappings": {
    "properties": {
      "http_status": { "type": "keyword" },
      "price": { "type": "scaled_float", "scaling_factor": 100 },
      "quantity": { "type": "integer" },
      "timestamp_epoch_ms": { "type": "date", "format": "epoch_millis" }
    }
  }
}
```

Use `keyword` for numeric identifiers (status codes, zip codes, product IDs) that only need exact matching.
Use `scaled_float` for decimal values with known precision (prices, percentages) to save space.

---

## Multi-Field Patterns

### Standard Text + Keyword

```json
{
  "product_name": {
    "type": "text",
    "analyzer": "standard",
    "fields": {
      "keyword": {
        "type": "keyword",
        "ignore_above": 256
      },
      "autocomplete": {
        "type": "text",
        "analyzer": "autocomplete_analyzer"
      }
    }
  }
}
```

### Searchable Keyword with Normalization

```json
{
  "email": {
    "type": "keyword",
    "normalizer": "lowercase_normalizer"
  }
}
```

Define the normalizer in index settings:

```json
{
  "settings": {
    "analysis": {
      "normalizer": {
        "lowercase_normalizer": {
          "type": "custom",
          "filter": ["lowercase"]
        }
      }
    }
  }
}
```

---

## Dynamic Templates

### Prevent Mapping Explosion

```json
{
  "mappings": {
    "dynamic": "strict",
    "dynamic_templates": [
      {
        "strings_as_keywords": {
          "match_mapping_type": "string",
          "mapping": {
            "type": "keyword",
            "ignore_above": 512
          }
        }
      },
      {
        "labels_as_flattened": {
          "path_match": "metadata.labels.*",
          "mapping": {
            "type": "keyword"
          }
        }
      },
      {
        "disable_metrics_indexing": {
          "path_match": "metrics.*",
          "mapping": {
            "type": "float",
            "index": false
          }
        }
      }
    ],
    "properties": {
      "timestamp": { "type": "date" },
      "message": { "type": "text" }
    }
  }
}
```

### Setting `index.mapping.total_fields.limit`

Increase only when justified. Default is 1000. If you need to raise it frequently, consider `flattened` type or restructuring the data model.

---

## Nested vs Flattened vs Join

### When to Use Each

| Approach | Use When | Indexing Cost | Query Cost |
|----------|----------|--------------|------------|
| **Flattened objects** | Cross-field correlation not needed | Low | Low |
| **Nested** | Need to query across fields of same object (< 50 per doc) | Medium | Medium |
| **Join (parent-child)** | Parent updates independently of children, many children | High | High |
| **Denormalization** | Read-heavy, updates infrequent | Lowest query cost | Reindex on update |

### Nested Example

```json
{
  "mappings": {
    "properties": {
      "comments": {
        "type": "nested",
        "properties": {
          "author": { "type": "keyword" },
          "text": { "type": "text" },
          "created_at": { "type": "date" }
        }
      }
    }
  }
}
```

Querying nested:

```json
{
  "query": {
    "nested": {
      "path": "comments",
      "query": {
        "bool": {
          "must": [
            { "term": { "comments.author": "alice" } },
            { "match": { "comments.text": "elasticsearch" } }
          ]
        }
      }
    }
  }
}
```

### Limits to Enforce

- `index.mapping.nested_fields.limit`: 50 (default)
- `index.mapping.nested_objects.limit`: 10000 (default)
- Keep nested docs per parent < 100 for performance

---

## Mapping Evolution Strategies

### Adding New Fields
- Simply index documents with the new field (if dynamic mapping allows)
- Or use PUT mapping API to explicitly add the field

### Changing Field Types (requires reindex)

```json
POST _reindex
{
  "source": { "index": "my-index-v1" },
  "dest": { "index": "my-index-v2" }
}
```

### Using Aliases for Zero-Downtime Migration

```json
POST _aliases
{
  "actions": [
    { "remove": { "index": "my-index-v1", "alias": "my-index" } },
    { "add": { "index": "my-index-v2", "alias": "my-index" } }
  ]
}
```

### Field Aliases

```json
{
  "mappings": {
    "properties": {
      "user_id": { "type": "keyword" },
      "uid": {
        "type": "alias",
        "path": "user_id"
      }
    }
  }
}
```

---

## Common Pitfalls

1. **Using `text` for identifier fields** — Causes poor aggregation performance and unexpected matching
2. **Not setting `ignore_above` on keyword fields** — Leads to memory issues with long strings
3. **Dynamic mapping left as default** — Strings become both `text` and `keyword`, doubling storage
4. **Too many nested objects** — Each nested doc is a hidden Lucene document, multiplying index size
5. **Using `_all` or `copy_to` excessively** — Bloats index size with redundant data
6. **Not specifying date format** — Causes parsing failures when ingesting from multiple sources
7. **Mapping explosion** — Unbounded dynamic fields (e.g., user-provided keys) rapidly hit field limits
8. **Ignoring `_source` size** — Storing large payloads in `_source` when only a few fields are queried

---

## LogsDB Index Mode (8.17+, Default in 9.x)

LogsDB is a specialized index mode for log data that uses synthetic `_source` reconstruction and automatic index sorting for significant storage savings.

### Key Characteristics

- **Enabled by default** in 9.x for data streams matching `logs-*-*`
- Uses **synthetic `_source`** — `_source` is reconstructed from doc values and stored fields instead of being stored verbatim
- **Automatic index sorting** by `host.name`, `@timestamp` for better compression
- Up to **4x storage reduction** compared to standard index mode
- Requires Enterprise license for synthetic `_source`

### When to Use

| Index Mode | Use When |
|------------|----------|
| Standard | General-purpose indices, entity data, catalogs |
| LogsDB | Log data, event data, observability data streams |
| TSDB | Metrics with known dimensions (time_series_dimension fields) |

### Configuration

```json
PUT _index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "data_stream": {},
  "template": {
    "settings": {
      "index.mode": "logsdb"
    }
  }
}
```

### Limitations

- Synthetic `_source` may reorder fields and deduplicate array values
- Not all field types support synthetic `_source` — check compatibility
- `_source` filtering behaves differently (operates on reconstructed source)

---

## pattern_text Field Type (9.3+ GA)

The `pattern_text` type decomposes log messages into static template portions and dynamic variables, enabling much better compression than standard `text`.

### Mapping

```json
{
  "mappings": {
    "properties": {
      "message": {
        "type": "pattern_text"
      }
    }
  }
}
```

### How It Works

A log message like `User alice logged in from 192.168.1.1` is split into:
- **Template**: `User {} logged in from {}`
- **Variables**: `alice`, `192.168.1.1`

A companion `message.template_id` field stores a hash of the template, useful for grouping similar log messages.

### Configuration Options

| Parameter | Default | Description |
|-----------|---------|-------------|
| `analyzer` | Delimiter-based (splits on whitespace, `=?:[]{}\"\'`) | Controls tokenization |
| `index_options` | `docs` | `docs` (faster indexing) or `positions` (enables phrase queries) |

### Limitations

- Single value per document only
- No span queries (use interval queries instead)
- No sorting support, limited aggregation capabilities
- Best used with LogsDB index mode for maximum compression

### Compression Tip

Enable index sorting by `template_id` for significantly better compression:

```json
PUT my-index/_settings
{
  "index.logsdb.default_sort_on_message_template": true
}
```
