---
name: sdk
description: "DataHub Python SDK for metadata management: search, lineage, data quality assertions, and enriching datasets with descriptions, tags, glossary terms, domains, and structured properties"
metadata:
  languages: "python"
  versions: "1.4.0.5"
  revision: 1
  updated-on: "2026-03-10"
  source: maintainer
  tags: "datahub,metadata,data-catalog,lineage,data-quality,search,governance"
---

# DataHub Python SDK

You are a DataHub API expert. Help me write code using the DataHub Python SDK (`acryl-datahub`) and the DataHub Cloud SDK (`acryl-datahub-cloud`).

Official documentation: https://docs.datahub.com/docs/python-sdk/sdk-v2/

## Installation

```bash
pip install acryl-datahub
```

For DataHub Cloud features (smart assertions, subscriptions):

```bash
pip install acryl-datahub-cloud
```

Requires Python >= 3.10.

## Client Initialization

```python
from datahub.sdk import DataHubClient

client = DataHubClient(server="<your_server>", token="<your_token>")
```

- **Hosted (DataHub Cloud):** `server="https://<your-instance>.acryl.io/gms"`
- **Local:** `server="http://localhost:8080"`

Generate a Personal Access Token from your DataHub instance settings.

### Initialize from environment

Set `DATAHUB_GMS_URL` and `DATAHUB_GMS_TOKEN` environment variables, or run `datahub init` to create `~/.datahubenv`:

```python
client = DataHubClient.from_env()
```

### Verify connectivity

```python
client.test_connection()
```

## Client Properties

The `DataHubClient` exposes specialized sub-clients as properties:

| Property | Type | Purpose |
|----------|------|---------|
| `client.entities` | `EntityClient` | CRUD operations on datasets, containers, etc. |
| `client.search` | `SearchClient` | Search and discover metadata |
| `client.lineage` | `LineageClient` | Add and retrieve lineage |
| `client.resolve` | `ResolverClient` | Resolve metadata references |
| `client.assertions` | (Cloud only) | Data quality assertions |
| `client.subscriptions` | (Cloud only) | Notification subscriptions |

## Search

Search across your entire data landscape using queries, filters, or both.

### Query-based search

```python
results = client.search.get_urns(query="sales")
for urn in results:
    print(urn)
```

### Filter-based search

```python
from datahub.sdk import FilterDsl as F

results = client.search.get_urns(filter=F.platform("snowflake"))
```

### Combine query and filters

```python
from datahub.sdk import FilterDsl as F

results = client.search.get_urns(
    query="forecast",
    filter=F.and_(F.platform("snowflake"), F.entity_type("dataset"))
)
```

### Filter reference

| Filter | Example |
|--------|---------|
| Platform | `F.platform("snowflake")` |
| Environment | `F.env("PROD")` |
| Entity type | `F.entity_type("dataset")` |
| Domain | `F.domain("urn:li:domain:marketing")` |
| Subtype | `F.entity_subtype("ML Experiment")` |
| Owner | `F.owner("urn:li:corpuser:jane")` |
| Tag | `F.tag("urn:li:tag:critical")` |
| Glossary term | `F.glossary_term("urn:li:glossaryTerm:PII")` |
| Container | `F.container("urn:li:container:my_db")` |
| Custom property | `F.has_custom_property("department", "sales")` |
| Deletion status | `F.soft_deleted("NOT_SOFT_DELETED")` |
| Custom field condition | `F.custom_filter(field="urn", condition="CONTAIN", values=["example"])` |

See `references/search.md` for detailed filter descriptions, custom_filter conditions, searchable fields, and common patterns.

### Logical operators

```python
# OR: charts or Snowflake datasets
results = client.search.get_urns(
    filter=F.or_(
        F.entity_type("chart"),
        F.and_(F.platform("snowflake"), F.entity_type("dataset")),
    )
)

# NOT: datasets not tagged "verified"
results = client.search.get_urns(
    filter=F.and_(F.entity_type("dataset"), F.not_(F.tag("urn:li:tag:verified")))
)
```

## Entity CRUD

### Create a dataset

```python
from datahub.sdk import Dataset

dataset = Dataset(
    platform="snowflake",
    name="prod.analytics.users",
    env="PROD",
    description="Core users table",
    tags=["critical", "pii"],
    terms=["urn:li:glossaryTerm:PII"],
    domain="urn:li:domain:analytics",
    owners=["urn:li:corpuser:jane"],
    custom_properties={"team": "data-eng", "refresh_cadence": "hourly"},
    schema=[
        ("user_id", "BIGINT"),
        ("email", "VARCHAR"),
        ("created_at", "TIMESTAMP", "Account creation timestamp"),
    ],
)

client.entities.create(dataset)
```

### Get an entity

```python
dataset = client.entities.get("urn:li:dataset:(urn:li:dataPlatform:snowflake,prod.analytics.users,PROD)")

print(dataset.description)
print(dataset.schema)
print(dataset.custom_properties)
```

### Update an entity

```python
dataset.set_description("Core users table with profile data")
dataset.set_custom_properties({"team": "data-eng", "refresh_cadence": "daily"})
client.entities.update(dataset)
```

### Upsert (create or update)

```python
client.entities.upsert(dataset)
```

### Delete an entity

```python
client.entities.delete(
    "urn:li:dataset:(urn:li:dataPlatform:snowflake,prod.analytics.users,PROD)",
    hard=False,   # soft delete (default); set True for permanent deletion
    cascade=False, # cascade to children (containers, dataflows)
)
```

## Enriching Metadata

### Descriptions

```python
dataset = client.entities.get(dataset_urn)
dataset.set_description("Updated description for this table")
client.entities.update(dataset)
```

### Tags

```python
dataset = client.entities.get(dataset_urn)

# On the dataset
dataset.set_tags(["critical", "pii"])

# On individual columns
for field in dataset.schema:
    if field.field_path == "email":
        field.add_tag("pii")
        field.set_description("User email address, PII-classified")

client.entities.update(dataset)
```

### Glossary terms

```python
dataset = client.entities.get(dataset_urn)
dataset.set_terms(["urn:li:glossaryTerm:PII", "urn:li:glossaryTerm:CustomerData"])

for field in dataset.schema:
    if field.field_path == "email":
        field.add_term("urn:li:glossaryTerm:EmailAddress")

client.entities.update(dataset)
```

### Domains

```python
dataset = client.entities.get(dataset_urn)
dataset.set_domain("urn:li:domain:analytics")
client.entities.update(dataset)
```

### Owners

```python
dataset = client.entities.get(dataset_urn)
dataset.set_owners([
    "urn:li:corpuser:alice",
    "urn:li:corpGroup:data-engineering",
])
client.entities.update(dataset)
```

### Structured properties

```python
dataset = client.entities.get(dataset_urn)
dataset.set_structured_properties({
    "urn:li:structuredProperty:data_tier": ["tier1"],
    "urn:li:structuredProperty:retention_days": [90],
})
client.entities.update(dataset)
```

### Custom properties

```python
dataset = client.entities.get(dataset_urn)
dataset.set_custom_properties({
    "team": "data-engineering",
    "slack_channel": "#data-eng",
    "refresh_cadence": "hourly",
})
client.entities.update(dataset)
```

## Containers

Organize datasets logically with containers (databases, schemas, folders).

```python
from datahub.sdk import Container
from datahub.emitter.mcp_builder import DatabaseKey, SchemaKey

database_key = DatabaseKey(
    platform="snowflake",
    instance="production",
    database="analytics_db",
)

database_container = Container(
    database_key,
    display_name="Analytics Database",
    description="Main analytics database",
    subtype="Database",
)
client.entities.upsert(database_container)

schema_key = SchemaKey(
    platform="snowflake",
    instance="production",
    database="analytics_db",
    schema="reporting",
)

schema_container = Container(
    schema_key,
    display_name="Reporting Schema",
    description="Schema for reporting tables and views",
    subtype="Schema",
    parent_container=database_key,
    domain="urn:li:domain:analytics",
)
client.entities.upsert(schema_container)
```

## Documents

Documents store knowledge content — tutorials, runbooks, FAQs, or references to external docs in systems like Notion or Confluence.

### Native document (stored in DataHub)

```python
from datahub.sdk import Document

doc = Document.create_document(
    id="getting-started-guide",
    title="Getting Started with DataHub",
    text="# Welcome\n\nThis guide will help you get started...",
    tags=["onboarding"],
    domain="urn:li:domain:data-governance",
)
client.entities.upsert(doc)
```

### External document (Notion, Confluence, etc.)

```python
doc = Document.create_external_document(
    id="notion-team-handbook",
    title="Engineering Handbook",
    platform="notion",
    external_url="https://notion.so/team/engineering-handbook",
    external_id="notion-page-abc123",
    text="Summary of the handbook for search indexing...",
)
client.entities.upsert(doc)
```

### AI-only context document

Documents hidden from global search/sidebar, accessible only through related assets:

```python
doc = Document.create_document(
    id="orders-dataset-context",
    title="Orders Dataset Context",
    text="This dataset contains daily order summaries...",
    show_in_global_context=False,
    related_assets=["urn:li:dataset:(urn:li:dataPlatform:snowflake,orders,PROD)"],
)
client.entities.upsert(doc)
```

### Document properties

- `doc.set_title("New Title")` / `doc.title`
- `doc.set_text("New content")` / `doc.text`
- `doc.publish()` / `doc.unpublish()` / `doc.status`
- `doc.set_parent_document("urn:li:document:parent-doc")` — hierarchical organization
- `doc.add_related_asset("urn:li:dataset:...")` / `doc.set_related_assets([...])`
- `doc.add_related_document("urn:li:document:...")` / `doc.set_related_documents([...])`
- `doc.hide_from_global_context()` / `doc.show_in_global_search()`

## Lineage

### Table-level lineage

```python
client.lineage.add_lineage(
    upstream="urn:li:dataset:(urn:li:dataPlatform:snowflake,raw.events,PROD)",
    downstream="urn:li:dataset:(urn:li:dataPlatform:snowflake,analytics.sessions,PROD)",
)
```

### Column-level lineage

```python
client.lineage.add_lineage(
    upstream="urn:li:dataset:(urn:li:dataPlatform:snowflake,raw.events,PROD)",
    downstream="urn:li:dataset:(urn:li:dataPlatform:snowflake,analytics.sessions,PROD)",
    column_lineage={
        "session_id": ["event_id"],
        "user_id": ["user_id"],
        "session_start": ["event_timestamp"],
    },
    transformation_text="SELECT event_id AS session_id, user_id, MIN(event_timestamp) AS session_start FROM raw.events GROUP BY event_id, user_id",
)
```

### Copy lineage with auto column matching

```python
client.lineage.add_lineage(
    upstream="urn:li:dataset:(urn:li:dataPlatform:postgres,users,PROD)",
    downstream="urn:li:dataset:(urn:li:dataPlatform:snowflake,users_replica,PROD)",
    column_lineage="auto_fuzzy",  # or "auto_strict", explicit mapping dict, or False
)
```

### Infer lineage from SQL

```python
client.lineage.infer_lineage_from_sql(
    query_text="INSERT INTO analytics.daily_revenue SELECT date, SUM(amount) FROM raw.orders GROUP BY date",
    platform="snowflake",
    env="PROD",
)
```

### Get lineage (impact analysis)

```python
upstream_results = client.lineage.get_lineage(
    source_urn="urn:li:dataset:(urn:li:dataPlatform:snowflake,analytics.sessions,PROD)",
    direction="upstream",
    max_hops=3,
)

for result in upstream_results:
    print(f"{result.urn} ({result.type}) - {result.hops} hop(s) away")

# Column-level lineage
column_results = client.lineage.get_lineage(
    source_urn="urn:li:dataset:(urn:li:dataPlatform:snowflake,analytics.sessions,PROD)",
    source_column="session_id",
    direction="upstream",
)
```

See `references/lineage.md` for datajob lineage, cross-entity lineage, and advanced patterns.

## Data Quality Assertions (DataHub Cloud)

Requires `acryl-datahub-cloud`. Smart assertions use anomaly detection to monitor data freshness, volume, and column metrics.

### Freshness assertion

```python
assertion = client.assertions.sync_smart_freshness_assertion(
    dataset_urn="urn:li:dataset:(urn:li:dataPlatform:snowflake,prod.analytics.users,PROD)",
    display_name="Freshness Anomaly Monitor",
    detection_mechanism="information_schema",
    sensitivity="medium",  # "low", "medium", "high"
    tags=["automated", "freshness"],
    enabled=True,
)
```

### Volume assertion

```python
assertion = client.assertions.sync_smart_volume_assertion(
    dataset_urn="urn:li:dataset:(urn:li:dataPlatform:snowflake,prod.analytics.users,PROD)",
    display_name="Volume Check",
    detection_mechanism="information_schema",
    sensitivity="medium",
    schedule="0 */6 * * *",
    tags=["automated", "volume"],
    enabled=True,
)
```

### Column metric assertion

```python
assertion = client.assertions.sync_smart_column_metric_assertion(
    dataset_urn="urn:li:dataset:(urn:li:dataPlatform:snowflake,prod.analytics.users,PROD)",
    column_name="email",
    metric_type="null_count",  # "null_count", "unique_count", "empty_count"
    display_name="Email Null Check",
    detection_mechanism="all_rows_query_datahub_dataset_profile",
    tags=["automated", "column_quality"],
    enabled=True,
)
```

### SQL assertion

```python
assertion = client.assertions.sync_smart_sql_assertion(
    dataset_urn="urn:li:dataset:(urn:li:dataPlatform:snowflake,prod.analytics.users,PROD)",
    display_name="Active Users Count",
    statement="SELECT COUNT(*) FROM prod.analytics.users WHERE status = 'active'",
    sensitivity="medium",
    schedule="0 */6 * * *",
    tags=["automated", "sql_check"],
    enabled=True,
)
```

See `references/assertions.md` for bulk assertion creation, updating, and advanced patterns.

## URN Format

DataHub identifies every entity with a URN (Uniform Resource Name) — a globally unique, typed identifier. URNs are the primary way to reference entities across search, lineage, and metadata operations.

**Do not guess or hand-craft URNs.** URNs should come from search results, entity creation responses, or known ingestion sources. Use `client.search.get_urns()` to discover existing entities before referencing them.

| Entity | URN Pattern | Description |
|--------|-------------|-------------|
| Dataset | `urn:li:dataset:(urn:li:dataPlatform:{platform},{name},{env})` | Tables, views, topics, and files in warehouses, lakes, OLTP databases, etc. |
| Schema field | `urn:li:schemaField:({dataset_urn},{field_path})` | A column within a dataset. Not a standalone entity — access via `dataset.schema`. |
| Container | `urn:li:container:{key}` | Databases, schemas, folders, S3 buckets — logical grouping of datasets |
| Data Flow | `urn:li:dataFlow:({orchestrator},{flow_id},{env})` | A pipeline or DAG (e.g. an Airflow DAG, a Spark application) |
| Data Job | `urn:li:dataJob:({flow_urn},{job_id})` | A task within a data flow (e.g. an Airflow task, a dbt model run) |
| Dashboard | `urn:li:dashboard:({platform},{id})` | BI dashboards (Looker, Tableau, PowerBI, etc.) |
| Chart | `urn:li:chart:({platform},{id})` | Individual visualizations or tiles within a dashboard |
| Domain | `urn:li:domain:{id}` | Business domain for organizing assets (e.g. Finance, Marketing) |
| Data Product | `urn:li:dataProduct:{id}` | A curated collection of datasets exposed as a product |
| Glossary term | `urn:li:glossaryTerm:{name}` | Standardized business vocabulary (e.g. PII, Revenue, Customer) |
| Tag | `urn:li:tag:{name}` | Lightweight labels for classification (e.g. critical, deprecated) |
| Document | `urn:li:document:{id}` | Knowledge content: tutorials, runbooks, FAQs, or external doc references |
| Assertion | `urn:li:assertion:{id}` | A data quality check bound to a dataset |
| Incident | `urn:li:incident:{id}` | A data incident or issue raised against an asset |
| User | `urn:li:corpuser:{id}` | An individual user identity |
| Group | `urn:li:corpGroup:{id}` | A team or group of users |

You can use string URNs directly or typed URN classes:

```python
from datahub.metadata.urns import DatasetUrn

urn = DatasetUrn("urn:li:dataset:(urn:li:dataPlatform:snowflake,db.schema.table,PROD)")
print(urn.platform)  # snowflake
print(urn.name)      # db.schema.table
print(urn.env)       # PROD
```

## Best Practices

- **Use `from_env()`** for production scripts to avoid hardcoding credentials.
- **Use `upsert()`** when you want idempotent create-or-update behavior.
- **Soft delete by default.** Only use `hard=True` when you need permanent removal.
- **Tag consistently.** Use plain tag names (e.g., `"critical"`) — the SDK auto-converts to URN format.
- **Process assertions single-threaded per dataset** to avoid race conditions.
- **Batch operations** with delays between batches to avoid overwhelming the API during large-scale updates.
