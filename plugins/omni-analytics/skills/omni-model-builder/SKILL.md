---
name: omni-model-builder
description: Create and edit Omni Analytics semantic model definitions — views, topics, dimensions, measures, relationships, and query views — using YAML through the Omni REST API. Use this skill whenever someone wants to add a field, create a new dimension or measure, define a topic, set up joins between tables, modify the data model, build a new view, add a calculated field, create a relationship, edit YAML, work on a branch, promote model changes, or any variant of "model this data", "add this metric", "create a view for", or "set up a join between". Also use for migrating modeling patterns since Omni's YAML is conceptually similar to other semantic layer definitions.
---

# Omni Model Builder

Create and modify Omni's semantic model through the YAML API — views, topics, dimensions, measures, relationships, and query views.

> **Tip**: Always use `omni-model-explorer` first to understand the existing model.

## Prerequisites

```bash
export OMNI_BASE_URL="https://yourorg.omniapp.co"
export OMNI_API_KEY="your-api-key"
```

You need **Modeler** or **Connection Admin** permissions.

## Omni's Layered Modeling Architecture

Omni uses a **layered approach** where each layer builds on top of the previous:

1. **Schema Layer** — Auto-generated from your database. Reflects tables, views, columns, and their types. Kept in sync via schema refresh.

2. **Shared Model Layer** — Your governed semantic model. Where you define dimensions, measures, joins, and topics that are reusable across the organization.

3. **Workbook Model Layer** — Ad hoc extensions within individual workbooks. Used for experimental fields before promotion to shared model.

4. **Branch Layer** — Intermediate development layer. Used when working in branches before merging changes to shared model.

**Key concept**: The schema layer is the foundation and source of truth for table/column structure. When your database schema changes (new tables, deleted columns, type changes), you refresh the schema to keep Omni in sync. All user-created content (dimensions, measures, relationships, topics) flows through the shared model layer.

**Development workflow**: When building or modifying the model, you work in **branches** (see "Safe Development Workflow" below). Branches are isolated copies where you can safely experiment before merging changes back to shared model. This skill covers creating and editing model definitions in both branches and shared models.

## Determine SQL Dialect

Before writing any SQL expressions, confirm the dialect from the connection — don't guess from the connection name:

```bash
# 1. Get the model's connectionId
curl -L "$OMNI_BASE_URL/api/v1/models/{modelId}" \
  -H "Authorization: Bearer $OMNI_API_KEY"

# 2. Look up the connection's dialect
curl -L "$OMNI_BASE_URL/api/v1/connections" \
  -H "Authorization: Bearer $OMNI_API_KEY"
# → find your connectionId and read the "dialect" field
# → e.g. "bigquery", "postgres", "snowflake", "databricks"
```

Use dialect-appropriate functions in your SQL (e.g. `SAFE_DIVIDE` for BigQuery, `NULLIF(a/b)` for Postgres/Snowflake).

## Schema Refresh: Syncing with Database Changes

The **schema layer** is auto-generated from your database and serves as the source of truth for table and column structure. When your database schema changes, you need to refresh Omni's schema layer to stay in sync.

### When to Trigger a Schema Refresh

- **New tables added** to your database that you want to model
- **Column added/renamed/deleted** in existing tables
- **Column types changed** in the database (e.g., VARCHAR → NUMERIC)
- **Creating a new view from scratch** and want Omni to auto-generate base dimensions
- **Troubleshooting type inference** — if Omni shows `type: UNKNOWN` and you've confirmed the column exists in the database

### What Schema Refresh Does

A schema refresh synchronizes your schema layer with the actual database structure:

1. Connects to your data warehouse
2. Reads table and column metadata
3. Updates the schema layer with new/changed/deleted objects
4. Auto-generates base dimensions for each column with correct types (DATE, VARCHAR, NUMERIC, etc.)
5. Sets up proper `timeframes` for temporal columns (date, week, month, quarter, year)

**Side effects to expect:**
- Refresh runs as a background job (may take several minutes for large databases)
- May auto-generate dimensions for columns you don't need to expose — suppress these with `hidden: true` in your extension layer
- If columns were deleted from the database, schema refresh will detect broken references in your model; use the Content Validator to identify and fix them

### How to Trigger a Schema Refresh

**Via API:**

```bash
# Standard refresh (adds/updates, doesn't remove deleted objects)
curl -L -X POST "$OMNI_BASE_URL/api/v1/models/{modelId}/refresh" \
  -H "Authorization: Bearer $OMNI_API_KEY"

# If branch-based schema refresh is enabled, specify the branch:
curl -L -X POST "$OMNI_BASE_URL/api/v1/models/{modelId}/refresh?branch_id={branchId}" \
  -H "Authorization: Bearer $OMNI_API_KEY"
```

Response: `{ "model_id": "...", "status": "running" }` — the refresh runs in the background.

**Check if refresh is complete:**

```bash
# Poll the model status
curl -L "$OMNI_BASE_URL/api/v1/models/{modelId}" \
  -H "Authorization: Bearer $OMNI_API_KEY" | jq '.model.schema_sync_status'
```

> **Note**: Schema refresh requires **Connection Admin** or higher permissions.

### Best Practice: Schema Refresh Workflow

When creating a new view from a table:

1. **Create a minimal view stub** or reference to the table
2. **Trigger schema refresh** to auto-generate base dimensions with correct types
3. **Review the generated dimensions** — ensure types look correct
4. **Edit extension layer** to add business logic:
   - Hide unwanted columns: `hidden: true`
   - Add measures, filters, custom dimensions
   - Add descriptions and `ai_context`
5. **Validate** and go live

This ensures your view has correct type information without manual casting.

## API Discovery

When unsure whether an endpoint or parameter exists, fetch the OpenAPI spec:

```bash
curl -L "$OMNI_BASE_URL/openapi.json" \
  -H "Authorization: Bearer $OMNI_API_KEY"
```

Use this to verify endpoints, available parameters, and request/response schemas before making calls.

## Safe Development Workflow

Always work in a branch. Never write directly to production.

### Step 0: Create a Branch

Omni branches are models with `modelKind: "BRANCH"`. There is no dedicated branch-creation endpoint — create one via `POST /api/v1/models`:

```bash
curl -L -X POST "$OMNI_BASE_URL/api/v1/models" \
  -H "Authorization: Bearer $OMNI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "modelKind": "BRANCH",
    "baseModelId": "{sharedModelId}",
    "connectionId": "{connectionId}",
    "modelName": "my-feature-branch"
  }'
```

The response `model.id` is your `branchId` — a UUID you'll pass to all subsequent API calls. To list existing branches at any time:

```bash
curl -L "$OMNI_BASE_URL/api/v1/models?include=activeBranches" \
  -H "Authorization: Bearer $OMNI_API_KEY"
```

> **Git-connected models**: If your model is connected to a git repo (`GET /api/v1/models/{modelId}/git` returns an `sshUrl`), merging an Omni branch will automatically commit the changes back to your git `baseBranch`. Choose one workflow and stick to it — either edit via the Omni branch API (then `git pull` to sync local files), or edit local files and push via git. Mixing both leads to conflicts.

### Step 1: Write YAML to a Branch

```bash
curl -L -X POST "$OMNI_BASE_URL/api/v1/models/{modelId}/yaml" \
  -H "Authorization: Bearer $OMNI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "fileName": "my_new_view.view",
    "yaml": "dimensions:\n  order_id:\n    primary_key: true\n  status:\n    label: Order Status\nmeasures:\n  count:\n    aggregate_type: count",
    "mode": "extension",
    "branchId": "{branchId}",
    "commitMessage": "Add my_new_view with status dimension and count measure"
  }'
```

> **Note**: The `branchId` parameter must be a UUID from the server (Step 0). Passing a string name instead will return `400 Bad Request: Unrecognized key: "branchName"`.

### Step 2: Validate

```bash
curl -L "$OMNI_BASE_URL/api/v1/models/{modelId}/validate?branchId={branchId}" \
  -H "Authorization: Bearer $OMNI_API_KEY"
```

Returns validation errors and warnings with `message`, `yaml_path`, and sometimes `auto_fix`.

### Step 3: Merge the Branch

> **Important**: Always ask the user for confirmation before merging. Merging applies changes to the production model and cannot be easily undone.

```bash
curl -L -X POST "$OMNI_BASE_URL/api/v1/models/{modelId}/branch/{branchName}/merge" \
  -H "Authorization: Bearer $OMNI_API_KEY"
```

If git with required PRs is configured, merge through your git workflow instead.

## YAML File Types

| Type | Extension | Purpose |
|------|-----------|---------|
| View | `.view` | Dimensions, measures, filters for a table |
| Topic | `.topic` | Joins views into a queryable unit |
| Relationships | (special) | Global join definitions |

Write with `mode: "extension"` (shared model layer). To delete a file, send empty `yaml`.

## Writing Views

### Basic View

```yaml
dimensions:
  order_id:
    primary_key: true
  status:
    label: Order Status
  created_at:
    label: Created Date
measures:
  count:
    aggregate_type: count
  total_revenue:
    sql: ${sale_price}
    aggregate_type: sum
    format: currency_2
```

### Understanding Schema Layer vs Extension Layer

When you create a view, Omni separates **schema** (database structure) from **model** (your business logic):

- **Schema layer**: Auto-generated base dimensions, one per column. Types come from the database. Read-only, synced via schema refresh.
- **Extension layer**: Your custom YAML. Can override base dimensions, add new dimensions/measures, hide columns, add business logic.

When both layers exist for a field with the same name, **your extension definition wins** but **type information comes from the schema layer**.

**Example**: Table has columns `created_at` (DATE) and `revenue` (NUMERIC).

```yaml
# Schema layer (auto-generated)
dimensions:
  created_at: {}  # type: DATE, auto-generates timeframes
  revenue: {}     # type: NUMERIC

# Extension layer (your YAML)
dimensions:
  created_at:
    label: "Order Created"
    description: "When the order was placed"

  revenue:
    hidden: true  # Hide the raw column

measures:
  total_revenue:
    sql: SUM(${revenue})
    aggregate_type: sum
    format: currency_2
```

Result: `created_at` inherits its type from schema layer (DATE with automatic week/month/year granularities) but gets your label. The raw `revenue` column is hidden, only exposed through the `total_revenue` measure.

**Key insight**: If your extension layer defines a dimension but there's no schema layer base dimension to provide type information, Omni can't infer granularities or types. Solution: trigger schema refresh to auto-generate the schema layer (see "Schema Refresh" section above).

### Dimension Parameters

See `references/modelParameters.md` for the complete list of 35+ dimension parameters, format values, and timeframes.

Most common parameters:
- `sql` — SQL expression using `${field_name}` references
- `label` — display name · `description` — help text (also used by Blobby)
- `primary_key: true` — unique key (critical for aggregations)
- `hidden: true` — hides from picker, still usable in SQL
- `format` — `number_2`, `currency_2`, `percent_2`, `id`
- `group_label` — groups fields in the picker
- `synonyms` — alternative names for AI matching (e.g., `[client, account, buyer]`)

### Measure Parameters

See `references/modelParameters.md` for the complete list of 24+ measure parameters and all 13 aggregate types.

Measure filters restrict rows before aggregation:

```yaml
measures:
  completed_orders:
    aggregate_type: count
    filters:
      status:
        is: complete
  california_revenue:
    sql: ${sale_price}
    aggregate_type: sum
    filters:
      state:
        is: California
```

Filter conditions: `is`, `is_not`, `greater_than`, `less_than`, `contains`, `starts_with`, `ends_with`

## Writing Topics

See [Topics setup](https://docs.omni.co/modeling/topics/setup.md) for complete YAML examples with joins, fields, and ai_context, and [Topic parameters](https://docs.omni.co/modeling/topics/parameters.md) for all available options.

Key topic elements:
- `base_view` — the primary view for this topic
- `joins` — nested structure for join chains (e.g., `users: {}` or `inventory_items: { products: {} }`)
- `ai_context` — guides Blobby's field mapping (e.g., "Map 'revenue' → total_revenue")
- `default_filters` — applied to all queries unless removed
- `always_where_sql` — non-removable filters
- `fields` — field curation: `[order_items.*, users.name, -users.internal_id]`

## Writing Relationships

```yaml
- join_from_view: order_items
  join_to_view: users
  on_sql: ${order_items.user_id} = ${users.id}
  relationship_type: many_to_one
  join_type: always_left
```

| Type | When to Use |
|------|-------------|
| `many_to_one` | Orders → Users |
| `one_to_many` | Users → Orders |
| `one_to_one` | Users → User Settings |
| `many_to_many` | Tags ↔ Products (rare) |

Getting `relationship_type` right prevents fanout and symmetric aggregate errors.

## Query Views

Virtual tables defined by a saved query:

```yaml
schema: PUBLIC
query:
  fields:
    order_items.user_id: user_id
    order_items.count: order_count
    order_items.total_revenue: lifetime_value
  base_view: order_items
  topic: order_items

dimensions:
  user_id:
    primary_key: true
  order_count: {}
  lifetime_value:
    format: currency_2
```

Or with raw SQL:

```yaml
schema: PUBLIC
sql: |
  SELECT user_id, COUNT(*) as order_count, SUM(sale_price) as lifetime_value
  FROM order_items GROUP BY 1
```

## Common Validation Errors

| Error | Fix |
|-------|-----|
| "No view X" | Check view name spelling |
| "No join path from X to Y" | Add a relationship |
| "Duplicate field name" | Remove duplicate or rename (or suppress with `hidden: true` if one is auto-generated) |
| "Invalid YAML syntax" | Check indentation (2 spaces, no tabs) |
| Column reference error (e.g., "Column `X` not found") | Check that the table exists and your Omni connection has access |

## Model Out of Sync with Database

If Omni's model doesn't reflect the current database structure (missing columns, wrong types, broken references, etc.), trigger a schema refresh to re-sync:

```bash
curl -L -X POST "$OMNI_BASE_URL/api/v1/models/{modelId}/refresh" \
  -H "Authorization: Bearer $OMNI_API_KEY"
```

**After the refresh completes**, validate to identify any issues:

```bash
curl -L "$OMNI_BASE_URL/api/v1/models/{modelId}/validate" \
  -H "Authorization: Bearer $OMNI_API_KEY"
```

Common issues after refresh and how to fix:

| Issue | What Happened | Fix |
|-------|---------------|-----|
| **Broken column references** | You reference a column that no longer exists in the database | Remove the dimension/measure from your extension layer or update the `sql` to reference a column that exists |
| **Field name collision** | You defined a measure with the same name as an auto-generated dimension | Suppress the auto-generated dimension with `hidden: true` or rename your measure |
| **Unknown field types** | Omni still shows `type: UNKNOWN` after refresh | Verify the table and column actually exist in the database and your connection has access to them |
| **Missing tables** | A table you reference doesn't appear after refresh | Verify the table exists in the database and check that your Omni connection includes the database/schema where it lives |

## Docs Reference

- [Model YAML API](https://docs.omni.co/api/models.md) · [Views](https://docs.omni.co/modeling/views.md) · [Topics](https://docs.omni.co/modeling/topics/parameters.md) · [Dimensions](https://docs.omni.co/modeling/dimensions.md) · [Measures](https://docs.omni.co/modeling/measures.md) · [Relationships](https://docs.omni.co/modeling/relationships.md) · [Query Views](https://docs.omni.co/modeling/query-views.md) · [Branch Mode](https://docs.omni.co/finding-content/drafting-publishing/branch-mode.md)

## Related Skills

- **omni-model-explorer** — understand the model before modifying
- **omni-ai-optimizer** — add AI context after building topics
- **omni-query** — test new fields
