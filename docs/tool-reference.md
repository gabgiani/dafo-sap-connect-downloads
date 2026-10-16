# Tool reference

[← Documentation index](../README.md#documentation)

The exact catalogue of an installation comes from `tools/list`. This page describes each tool family: how it is named, which arguments it accepts and what it returns. All examples use sample data.

General rules for every tool:

- Arguments not declared in the schema are rejected (`additionalProperties: false`), with a message listing the accepted ones.
- A blank or `null` value counts as *not given*.
- Every tool is read-only and returns its result in `structuredContent` (mirrored as JSON text).

## `search_sap_knowledge`

Finds short guidance cards. Does not read SAP.

| Argument | Type | Rules |
|---|---|---|
| `query` | string, required | 1–200 characters, keywords such as `production materials stock` |
| `top` | integer | 1–5, default 3 |

```json
{"package":"sap-manufacturing-risk","version":"1.5.0","matches":[
  {"id":"production-header","title":"Read production and its components",
   "question":"What is recorded for a production order?","evidence_status":"verified_sample"}]}
```

Cards of investigations that the administrator has not selected are not returned.

## `get_sap_knowledge`

Returns one card with its exact relationships. Does not read SAP.

| Argument | Type | Rules |
|---|---|---|
| `id` | string, required | A card id from `search_sap_knowledge` |

The result contains `card` (`title`, `question`, `tool`, example `arguments`, `steps`, `cannot_conclude`, `evidence_status`), `relationships` (each with `join`, `meaning`, `status` and `explanation`), the package `policy`, and `tool_available`: whether the card's read tool is published for this client.

## `describe_sap_tool_fields`

Explains the fields or columns a data tool returns: SAP type, documented description, allowed values, source table and join. Does not read SAP. See [the field dictionary](how-the-ai-understands-data.md#the-field-dictionary).

| Argument | Type | Rules |
|---|---|---|
| `tool` | string, required | A published `sap_read_*` or `sap_query_*` tool (an enum in the schema) |
| `fields` | array of strings | 1–200 returned field or column names, case-insensitive; omit to describe all |

## `sap_read_*` — API read tools

Each read tool reads one SAP Service Layer collection with the fields, lookup keys, fixed filters and customer scope saved by the administrator. Its description names the collection and the accepted lookups.

| Argument | Type | Rules |
|---|---|---|
| *lookup keys* | string (integers accepted) | Up to three, named after the SAP field: `DocEntry` → `doc_entry`, `CardCode` → `card_code`, `DocumentLines/BaseEntry` → `base_entry`. Exact match, ANDed with the fixed filters. |
| `top` | integer | 1–50, default 5 |
| `skip` | integer | Records to skip; use `next_skip` from the previous page |
| `customer_code` | string | Internal mode only, on tools with customer scope. Omitted = the configured default customer. |

```json
{"name":"sap_read_sales_orders","arguments":{"doc_entry":"1215","top":1}}
```

```json
{
  "tool": "sap_read_sales_orders",
  "entity": "Orders",
  "rows": [{"DocEntry": 1215, "DocNum": 1215, "CardCode": "C20000", "DocumentStatus": "bost_Open",
            "DocumentLines": [{"LineNum": 0, "ItemCode": "P20002", "Quantity": 5, "LineStatus": "bost_Open"}]}],
  "has_more": false,
  "next_skip": null,
  "coverage": "bounded_query",
  "nested_limit": 50,
  "nested_truncations": [],
  "note": "Administrator-selected scalar leaves only. …",
  "interpretation": {"card": "…", "steps": ["…"], "cannot_conclude": ["…"]},
  "column_dictionary": {"tool": "describe_sap_tool_fields", "id": "9b1e…"}
}
```

| Result field | Meaning |
|---|---|
| `rows` | Records with the selected fields; line fields grouped under their list (`DocumentLines`, `ProductionOrderLines`…) in their original order |
| `has_more`, `next_skip` | More records exist; continue with `skip` |
| `coverage` | `bounded_query`: a page, never proof of complete coverage |
| `nested_truncations` | Line lists cut at 50 entries |
| `empty_result` | Present when a lookup found nothing: explains that only this exact lookup is empty (an item code is not a resource code, a document number is not an internal entry) |
| `interpretation` | Rules for reading this result, from the knowledge package, when a card exists for this tool |
| `column_dictionary` | Pointer to the field dictionary |

A lookup on a line field (for example `base_entry` on `DocumentLines/BaseEntry`) returns, for each document, only the matching lines, newest first, bounded to 50 lines; `skip` is not available for it.

## `list_sap_queries`

Lists the published SAP queries (internal mode only). No arguments.

```json
{"queries":[
  {"tool":"sap_query_open_orders","sql_code":"DAFO_AI_OPEN_ORDERS","backend":"service_layer",
   "description":"Open sales order lines with remaining quantity…","execution_mode":"online",
   "parameters":[{"name":"customer_code","kind":"string","description":"…"}]},
  {"tool":"sap_query_sales_history","sql_code":"…","backend":"service_layer","execution_mode":"scheduled",
   "parameters":[{"name":"customer_code","kind":"string","column":"CardCode","operator":"eq","required":true}]}]}
```

For scheduled queries, `parameters` lists the filters over the stored rows. SQL text is never returned.

## `sap_query_*` — SAP queries

A saved SQL query published by the administrator (internal mode only). Its description contains the administrator's description, the SAP query code and how it runs.

### Online queries

| Argument | Type | Rules |
|---|---|---|
| `parameters` | object | Every declared SQL parameter is required; types `string`, `integer`, `number`, `boolean` or `date` (`YYYY-MM-DD`) |
| `top` | integer | 1–20 rows per page, default 5 |
| `skip` | integer | Service Layer only; continue with `next_skip` |
| `all_pages` | boolean | Service Layer only; returns every page in one result |

```json
{"name":"sap_query_open_orders","arguments":{"parameters":{"customer_code":"C20000"},"top":20}}
```

| Result field | Meaning |
|---|---|
| `rows` | Every returned column and value, as SAP returns them |
| `complete` | The traversal reached the end of the result |
| `has_more`, `next_skip` | Continue with `skip` and the same parameters |
| `truncation_reason` | Why `all_pages` stopped early (size, page count or time); resumable |
| `coverage` | `complete_query` when everything from the first row was returned, otherwise `bounded_query` |
| `column_dictionary` | Pointer to the field dictionary |

Queries run through the DI API bridge return the first page only (up to 20 rows) and accept neither `skip` nor `all_pages`.

### Scheduled queries

The tool reads the latest complete snapshot stored on the gateway computer. It never calls SAP.

| Argument | Type | Rules |
|---|---|---|
| `parameters` | object | The **filters** the administrator declared over the stored rows. Required filters must be given; optional ones can be omitted to return rows for every value. Text comparison is exact and case-sensitive. |
| `top` | integer | 1–20, default 5 |
| `skip` | integer | With `skip` > 0, `snapshot_version` is required |
| `snapshot_version` | string | From the first page, so all pages come from the same snapshot |
| `all_pages` | boolean | Every matching stored row |

```json
{"name":"sap_query_sales_history","arguments":{"parameters":{"customer_code":"C20000"},"top":20}}
```

| Result field | Meaning |
|---|---|
| `status` | `ready` |
| `rows`, `matching_rows` | The page and the number of stored rows that match the filters |
| `last_successful_extraction` | When the data was extracted from SAP; SAP may have changed since |
| `snapshot_rows`, `snapshot_complete` | Size of the stored snapshot; only complete extractions are stored |
| `snapshot_version` | Identifier of the snapshot; pass it back when paging |
| `extraction_parameters` | The fixed values used to extract the data |
| `has_more`, `next_skip`, `complete` | Paging over the stored rows |

Before the first successful extraction, or after the administrator changed the query, the call returns `isError: true` with *Query not available. No business conclusion can be drawn from this response.* See [Online or scheduled queries](online-vs-scheduled.md).

## `investigate_*` — investigations

Runs every read of a fixed path in order, starting from one identifier, and returns all the evidence in one result. The description starts with the investigation title and lists the kinds of identifier it accepts.

| Argument | Type | Rules |
|---|---|---|
| `start_from` | string, required | The kind of identifier, an enum of the path's entry points (for example `absolute_entry`, `item_no`, `sales_order_doc_entry`) |
| `identifier` | string, required | The value, copied from the question or an earlier result |

```json
{"name":"investigate_materials","arguments":{"start_from":"absolute_entry","identifier":"154"}}
```

```json
{
  "investigation": "materials",
  "question": "Which materials could block this production order?",
  "input": {"absolute_entry": "154"},
  "steps": [
    {"step": "production", "reader": "sap_read_production_overview", "arguments": {"absolute_entry": "154"}, "result": {"rows": ["…"]}},
    {"step": "stock", "reader": "sap_read_material_stock", "arguments": {"item_code": "C00011"}, "result": {"rows": ["…"]}},
    {"step": "stock", "reader": "sap_read_material_stock", "arguments": {"item_code": "C00012"}, "error": "SAP consultation failed. …"}
  ],
  "failed_steps": 1,
  "interpretation": {"sap_read_material_stock": {"steps": ["…"], "cannot_conclude": ["…"]}},
  "limitations": ["…"],
  "coverage": "The gateway ran every step of this path. …",
  "answer_with": "Use every step. Separate confirmed values, possible risks, what cannot be concluded and the next checks. …"
}
```

| Result field | Meaning |
|---|---|
| `steps` | One entry per read, in order. Each has `result`, `error` (the read failed) or `skipped` (nothing to look up, or more than 10 lines matched) |
| `failed_steps` | Number of failed reads |
| `interpretation` | Rules for each read tool used |
| `limitations` | What this investigation cannot establish |
| `answer_with` | How the answer must be structured |

Identical lookups run once. A failed or skipped step is unavailable evidence, never absence. Investigations included in the knowledge package:

| Tool | Starts from | Reads |
|---|---|---|
| `investigate_materials` | production order, finished item, sales order | Production order and lines → stock of every material line |
| `investigate_traceability` | production order, finished item, sales order | Production order → goods issues and receipts based on it |
| `investigate_capacity` | production order, finished item, sales order | Production order → capacity commitments → master record of every resource line |
| `investigate_procurement` | item, purchase order | Purchase order lines → stock of each item → purchase receipts |

Investigations created in setup appear with their own names; they are published only when all their read tools are published.

## Interactive exploration (setup only)

The knowledge map in setup can also be explored with live SAP data from any box. That exploration is an administrator tool for checking relationships; it is not an MCP tool.
