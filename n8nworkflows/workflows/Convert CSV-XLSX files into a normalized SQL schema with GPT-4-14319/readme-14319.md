Convert CSV/XLSX files into a normalized SQL schema with GPT-4

https://n8nworkflows.xyz/workflows/convert-csv-xlsx-files-into-a-normalized-sql-schema-with-gpt-4-14319


# Convert CSV/XLSX files into a normalized SQL schema with GPT-4

# 1. Workflow Overview

This workflow accepts an uploaded CSV or XLSX file through a webhook, profiles its columns statistically, asks GPT-4o to infer a normalized relational schema, validates that schema with rule-based checks, generates SQL, and returns SQL + ERD + data dictionary as JSON.

Typical use cases:
- Turning raw tabular files into an initial SQL schema draft
- Assisting database normalization from spreadsheets
- Producing machine-consumable schema design artifacts for downstream engineering

## 1.1 Input Reception & Runtime Configuration
The workflow starts from a webhook that receives the uploaded file. It then injects configuration thresholds used later for primary key and foreign key heuristics.

## 1.2 File Type Detection & Structured Extraction
The workflow checks whether the uploaded file is CSV or not. CSV files go through CSV extraction; all other files effectively go through XLSX extraction.

## 1.3 Column Profiling & Dataset Aggregation
The extracted tabular data is analyzed in a Code node to compute column-level statistics such as null percentage, uniqueness, entropy, candidate keys, regex-like patterns, and reference hints. The resulting profiled datasets are then aggregated into one payload.

## 1.4 AI Schema Inference
A LangChain AI Agent sends the aggregated profile data to GPT-4o with a detailed system prompt instructing it to propose a normalized relational schema. A structured output parser enforces a JSON response shape.

## 1.5 Rule-Based Schema Validation
A Code node validates the AI-generated schema against several structural rules such as presence of primary keys, relationship integrity, type compatibility, and normalization warnings.

## 1.6 SQL Generation
A second Code node converts the validated schema into SQL DDL statements for tables, foreign keys, and indexes.

## 1.7 ERD & Data Dictionary Generation
A final Code node attempts to derive a data dictionary and Mermaid ERD from the generated schema payload.

## 1.8 Webhook Response
The workflow returns a JSON response containing the SQL schema, data dictionary, and ERD text.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception & Runtime Configuration

### Overview
This block receives the uploaded file over HTTP and defines numeric thresholds used later by schema inference and validation logic. It establishes the workflow’s runtime parameters early so downstream nodes can reference them.

### Nodes Involved
- Upload Files Webhook
- Workflow Configuration

### Node Details

#### Upload Files Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`; entry point for HTTP POST uploads.
- **Configuration choices:**
  - Path: `schema-builder`
  - Method: `POST`
  - Response mode: `lastNode`
  - Raw body enabled
- **Key expressions or variables used:** None in the node itself.
- **Input and output connections:**
  - Input: none, entry node
  - Output: `Workflow Configuration`
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**
  - Wrong HTTP method
  - Large uploads exceeding n8n/server limits
  - Raw body present but file metadata not shaped as expected for extractor nodes
  - If multipart/binary handling is not configured upstream, later nodes may not find file content
- **Sub-workflow reference:** None

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`; injects configuration values into the flow.
- **Configuration choices:**
  - Adds:
    - `fkOverlapThreshold = 0.7`
    - `maxNormalizationDepth = 3`
    - `minUniquePercentForPK = 0.95`
  - Keeps other incoming fields
- **Key expressions or variables used:** None; static values.
- **Input and output connections:**
  - Input: `Upload Files Webhook`
  - Output: `Check File Type`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If incoming payload structure is malformed, downstream nodes still run but config may be the only reliable fields
  - Note that `minUniquePercentForPK` is set to `0.95`, while some code/prompt logic appears to reason in percentages like `95`; unit consistency is important
- **Sub-workflow reference:** None

---

## 2.2 File Type Detection & Structured Extraction

### Overview
This block determines whether the upload is CSV and routes processing accordingly. It then converts the file into structured rows suitable for profiling.

### Nodes Involved
- Check File Type
- Extract CSV Data
- Extract Excel Data

### Node Details

#### Check File Type
- **Type and technical role:** `n8n-nodes-base.if`; branch logic based on MIME type.
- **Configuration choices:**
  - OR conditions:
    - `$json.data.mimeType` contains `csv`
    - `$json.data.mimeType` contains `text/csv`
- **Key expressions or variables used:**
  - `={{ $json.data.mimeType }}`
- **Input and output connections:**
  - Input: `Workflow Configuration`
  - True output: `Extract CSV Data`
  - False output: `Extract Excel Data`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - Assumes MIME type is available at `data.mimeType`
  - If webhook payload does not create `data.mimeType`, expression may resolve empty and route to Excel branch
  - XLSX detection is implicit by fallback, not explicit
  - Non-CSV non-XLSX files may still be sent to XLSX extraction and fail there
- **Sub-workflow reference:** None

#### Extract CSV Data
- **Type and technical role:** `n8n-nodes-base.extractFromFile`; parses CSV into JSON rows.
- **Configuration choices:**
  - Include empty cells enabled
- **Key expressions or variables used:** None shown.
- **Input and output connections:**
  - Input: `Check File Type` true branch
  - Output: `Column Profiling Engine`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Requires actual binary/file input in the shape expected by `Extract From File`
  - Encoding, delimiter, quoting, or malformed CSV issues may cause parse errors or bad row structures
  - Empty files produce no rows, which later profiling code silently skips
- **Sub-workflow reference:** None

#### Extract Excel Data
- **Type and technical role:** `n8n-nodes-base.extractFromFile`; parses Excel sheets.
- **Configuration choices:**
  - Operation: `xlsx`
- **Key expressions or variables used:** None shown.
- **Input and output connections:**
  - Input: `Check File Type` false branch
  - Output: `Column Profiling Engine`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Any non-CSV file is treated as Excel
  - Multi-sheet behavior depends on node defaults and uploaded workbook structure
  - Password-protected or corrupted workbooks will fail
  - If extraction output shape differs by version, downstream profiler may not find rows
- **Sub-workflow reference:** None

---

## 2.3 Column Profiling & Dataset Aggregation

### Overview
This block computes dataset and column profiling metrics that support later AI reasoning. It enriches raw rows into a compact analytical representation, then combines all profiled outputs into a single array.

### Nodes Involved
- Column Profiling Engine
- Aggregate All Datasets

### Node Details

#### Column Profiling Engine
- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript profiling engine.
- **Configuration choices:**
  - Uses Node.js `crypto` for MD5 hashing
  - Generates a dataset UUID
  - Profiles every detected column
- **Key expressions or variables used:**
  - Iterates over `$input.all()`
  - Reads item payload from `item.json`
  - Tries multiple field names:
    - `fileName` / `source_file`
    - `sheetName` / `sheet_name`
    - `data` / `rows`
  - Emits:
    - `dataset_id`
    - `source_file`
    - `sheet_name`
    - `row_count`
    - `file_hash`
    - `column_count`
    - `columns`
    - `composite_key_candidates`
    - `profiled_at`
- **What the code does technically:**
  - Detects data types (`null`, `number`, `boolean`, `date`, `email`, `url`, `string`)
  - Computes null and uniqueness percentages
  - Computes min/max values or string length min/max
  - Computes entropy
  - Extracts simplified regex-like patterns
  - Builds top-value histograms
  - Flags candidate IDs and likely reference columns
  - Flags low-cardinality repeated-entity attributes
  - Suggests composite key candidates
- **Input and output connections:**
  - Input: `Extract CSV Data`, `Extract Excel Data`
  - Output: `Aggregate All Datasets`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Uses `require('crypto')`; this depends on Code node runtime permissions/environment
  - If extractor output is not an array of rows or nested where expected, node may skip the dataset entirely
  - `uniquePercent` is calculated as a percentage 0–100, but one helper compares `> 95`; elsewhere config uses `0.95`, creating a unit mismatch risk
  - `isCandidateID` checks `uniquePercent > 95`, so near-unique columns below that are ignored
  - Date parsing can over-classify strings as dates
  - For mixed-type columns, `dataTypeDistribution` may be noisy
  - Empty datasets are silently dropped with `continue`
- **Sub-workflow reference:** None

#### Aggregate All Datasets
- **Type and technical role:** `n8n-nodes-base.aggregate`; consolidates all profiled items into one item.
- **Configuration choices:**
  - Aggregate mode: all item data
  - Destination field: `datasets`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Column Profiling Engine`
  - Output: `Schema Reasoning Agent`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - If profiling returned no items, downstream AI gets empty or missing dataset content
  - Large datasets can create a very large JSON prompt for the AI node
- **Sub-workflow reference:** None

---

## 2.4 AI Schema Inference

### Overview
This block sends the profiled dataset bundle to GPT-4o and asks it to infer a normalized SQL-oriented relational schema. A structured parser is attached to constrain the output shape.

### Nodes Involved
- Schema Reasoning Agent
- Schema JSON Parser
- OpenAI GPT

### Node Details

#### Schema Reasoning Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; LLM orchestration node.
- **Configuration choices:**
  - Prompt mode: defined directly in the node
  - Input text: `={{ JSON.stringify($json.datasets) }}`
  - System message instructs the model to:
    - detect entities
    - design normalized tables
    - infer relationships
    - add constraints and indexes
    - validate decisions with overlap threshold guidance
  - Output parser enabled
- **Key expressions or variables used:**
  - `{{ JSON.stringify($json.datasets) }}`
  - Prompt references config thresholds conceptually, but they are not explicitly injected into the text body except through instruction wording
- **Input and output connections:**
  - Main input: `Aggregate All Datasets`
  - AI language model input: `OpenAI GPT`
  - AI output parser input: `Schema JSON Parser`
  - Main output: `Rule Validator`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Prompt/schema mismatch: system prompt expects fields like `dataType`, `isPrimaryKey`, `sourceTable`, `tableName`; parser expects `type`, `primaryKey`, `fromTable`, `table`
  - Long prompt risk for large files
  - Model may still produce structurally valid but semantically weak schema
  - If parser rejects output, execution fails here
  - The config thresholds are not directly merged into the prompt payload, so “use configuration thresholds from input” is only partially enforced
- **Sub-workflow reference:** None

#### Schema JSON Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates/structures LLM JSON output.
- **Configuration choices:**
  - Manual JSON schema
  - Requires top-level:
    - `tables`
    - `relationships`
    - `indexes`
  - Table columns require:
    - `name`
    - `type`
  - Relationships require:
    - `fromTable`, `fromColumn`, `toTable`, `toColumn`
  - Indexes require:
    - `table`, `columns`, `type`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Connected as output parser to `Schema Reasoning Agent`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Strong schema mismatch with downstream code, which expects names like `dataType`, `isPrimaryKey`, `tableName`, `isUnique`
  - Optional `foreignKey` object exists on columns, but downstream validation primarily reads top-level relationships
  - If the model returns the field names requested in the agent prompt instead of parser schema, parsing likely fails
- **Sub-workflow reference:** None

#### OpenAI GPT
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; chat model backend.
- **Configuration choices:**
  - Model: `gpt-4o`
  - No built-in tools enabled
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Output to `Schema Reasoning Agent` as AI language model
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Missing/invalid OpenAI credentials
  - Rate limits or token quota exhaustion
  - Model availability changes
  - Output may vary enough to stress parser constraints
- **Sub-workflow reference:** None

---

## 2.5 Rule-Based Schema Validation

### Overview
This block validates the AI-produced schema against deterministic checks. It can either pass through enriched validation status or return a structured failure object.

### Nodes Involved
- Rule Validator

### Node Details

#### Rule Validator
- **Type and technical role:** `n8n-nodes-base.code`; custom schema validation logic.
- **Configuration choices:**
  - Loads current schema from first input item
  - Reads configuration from `$('Workflow Configuration').item.json`
  - Checks:
    - primary key presence/duplication
    - foreign key table/column existence
    - FK target being PK or not
    - type compatibility
    - orphan tables
    - over-normalization
    - under-normalization
    - repeated numbered column patterns
- **Key expressions or variables used:**
  - `$input.all()`
  - `$('Workflow Configuration').item.json`
  - Uses `schemaData.schema || schemaData`
- **Input and output connections:**
  - Input: `Schema Reasoning Agent`
  - Output: `SQL Schema Generator`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Severe field-name mismatch with parser output:
    - validator expects `isPrimaryKey`, `sourceTable`, `targetTable`, `sourceColumn`, `targetColumn`, `dataType`
    - parser defines `primaryKey`, `fromTable`, `toTable`, `type`
  - This means validation may produce false errors, false warnings, or runtime failures such as accessing `fkColumn.dataType` when only `type` exists
  - If schema validation fails, node returns an error-shaped object instead of tables/relationships/indexes, but downstream SQL generator assumes success-shaped input
- **Sub-workflow reference:** None

---

## 2.6 SQL Generation

### Overview
This block converts the validated schema into SQL DDL. It builds table definitions first, then foreign keys, then indexes.

### Nodes Involved
- SQL Schema Generator

### Node Details

#### SQL Schema Generator
- **Type and technical role:** `n8n-nodes-base.code`; emits SQL statements as text.
- **Configuration choices:**
  - Maps logical types to SQL:
    - string → `VARCHAR(n)` or `TEXT`
    - integer, bigint, decimal, float, double, boolean, date, datetime, timestamp, text, json
  - Generates:
    - `CREATE TABLE`
    - `ALTER TABLE ... ADD CONSTRAINT ... FOREIGN KEY`
    - `CREATE INDEX` / `CREATE UNIQUE INDEX`
- **Key expressions or variables used:**
  - `$input.all()`
  - Expects fields like:
    - `validatedSchema.tables`
    - `column.dataType`
    - `column.maxLength`
    - `column.isPrimaryKey`
    - `column.isUnique`
    - `relationship.fromTable`
    - `index.name`
    - `index.tableName`
    - `index.isUnique`
- **Input and output connections:**
  - Input: `Rule Validator`
  - Output: `Generate Data Dictionary & ERD`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If Rule Validator returned a failure object, `validatedSchema.tables` is undefined and this node will fail
  - Again, field-name mismatch:
    - parser uses `type`, not `dataType`
    - parser uses `table`, not `tableName`
  - Generated SQL is generic and may not match a specific SQL dialect
  - Reserved keywords in table/column names are not escaped
  - Composite primary keys are not explicitly handled
- **Sub-workflow reference:** None

---

## 2.7 ERD & Data Dictionary Generation

### Overview
This block attempts to create a structured data dictionary and Mermaid ERD from prior results. However, its implementation expects an object structure that does not match the actual SQL string emitted by the previous node.

### Nodes Involved
- Generate Data Dictionary & ERD

### Node Details

#### Generate Data Dictionary & ERD
- **Type and technical role:** `n8n-nodes-base.code`; intended post-processing node for documentation artifacts.
- **Configuration choices:**
  - Reads:
    - `sqlSchema`
    - `validationResults`
    - `schemaAnalysis`
  - Iterates over `sqlSchema` as if it were an array of table objects
  - Builds:
    - `dataDictionary`
    - `erdDiagram` in Mermaid `erDiagram` format
- **Key expressions or variables used:**
  - `$input.first().json.sqlSchema`
  - `$input.first().json.validationResults`
  - `$input.first().json.schemaAnalysis`
- **Input and output connections:**
  - Input: `SQL Schema Generator`
  - Output: `Return Results`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Major data-shape bug: `SQL Schema Generator` returns `sqlSchema` as a string, but this node treats it as iterable table objects
  - As written, it will iterate over characters of the SQL string, producing invalid or empty artifacts rather than a real dictionary/ERD
  - It also expects fields like `tableName`, `columns`, `column.type`, `column.primaryKey`, which are not present in the SQL string
- **Sub-workflow reference:** None

---

## 2.8 Webhook Response

### Overview
This block returns the final payload to the HTTP caller. It exposes the generated SQL schema, data dictionary, and Mermaid ERD as JSON.

### Nodes Involved
- Return Results

### Node Details

#### Return Results
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`; sends HTTP response back to caller.
- **Configuration choices:**
  - Response code: `200`
  - Respond with: JSON
  - Body includes:
    - `sqlSchema`
    - `dataDictionary`
    - `erdDiagram`
- **Key expressions or variables used:**
  - References `$json.sqlSchema`, `$json.dataDictionary`, `$json.erdDiagram`
- **Input and output connections:**
  - Input: `Generate Data Dictionary & ERD`
  - Output: terminal response
- **Version-specific requirements:** Type version `1.5`.
- **Edge cases or potential failure types:**
  - If upstream data dictionary / ERD generation is broken, response still returns malformed or empty values
  - Manual JSON body interpolation may break if the values are already serialized strings or contain unexpected formatting
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Upload Files Webhook | n8n-nodes-base.webhook | Receives uploaded file via POST webhook |  | Workflow Configuration | ## Input & Configuration |
| Workflow Configuration | n8n-nodes-base.set | Injects FK/PK/normalization thresholds into the flow | Upload Files Webhook | Check File Type | ## Input & Configuration |
| Check File Type | n8n-nodes-base.if | Routes file to CSV or XLSX extraction based on MIME type | Workflow Configuration | Extract CSV Data; Extract Excel Data | ## File Detection & Extraction |
| Extract CSV Data | n8n-nodes-base.extractFromFile | Parses CSV into structured rows | Check File Type | Column Profiling Engine | ## File Detection & Extraction |
| Extract Excel Data | n8n-nodes-base.extractFromFile | Parses XLSX into structured rows | Check File Type | Column Profiling Engine | ## File Detection & Extraction |
| Column Profiling Engine | n8n-nodes-base.code | Profiles columns and dataset characteristics | Extract CSV Data; Extract Excel Data | Aggregate All Datasets | ## Column Profiling Engine and Dataset Aggregation |
| Aggregate All Datasets | n8n-nodes-base.aggregate | Combines profiled datasets into one array payload | Column Profiling Engine | Schema Reasoning Agent | ## Column Profiling Engine and Dataset Aggregation |
| Schema Reasoning Agent | @n8n/n8n-nodes-langchain.agent | Uses GPT-4o to infer normalized relational schema | Aggregate All Datasets; OpenAI GPT; Schema JSON Parser | Rule Validator | ## AI Schema Reasoning |
| Schema JSON Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON output from AI agent |  | Schema Reasoning Agent | ## AI Schema Reasoning |
| OpenAI GPT | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supplies GPT-4o language model to the agent |  | Schema Reasoning Agent | ## AI Schema Reasoning |
| Rule Validator | n8n-nodes-base.code | Validates schema structure, keys, relationships, and normalization warnings | Schema Reasoning Agent | SQL Schema Generator | ## Schema Validation |
| SQL Schema Generator | n8n-nodes-base.code | Generates SQL DDL from schema model | Rule Validator | Generate Data Dictionary & ERD | ## SQL Schema Generation |
| Generate Data Dictionary & ERD | n8n-nodes-base.code | Attempts to create a data dictionary and Mermaid ERD | SQL Schema Generator | Return Results | ## ERD & Data Dictionary |
| Return Results | n8n-nodes-base.respondToWebhook | Returns SQL, data dictionary, and ERD as webhook response | Generate Data Dictionary & ERD |  | ## Final Output |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for input/config section |  |  | ## Input & Configuration |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note for schema validation section |  |  | ## Schema Validation |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation note for final output section |  |  | ## Final Output |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation note for AI reasoning section |  |  | ## AI Schema Reasoning |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation note for file detection and extraction |  |  | ## File Detection & Extraction |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation note for profiling and aggregation |  |  | ## Column Profiling Engine and Dataset Aggregation |
| Sticky Note7 | n8n-nodes-base.stickyNote | Global workflow description and setup guidance |  |  | ## AI Multi-Table Schema Builder with Automated Normalization and Relationship Detection |
| Sticky Note7 | n8n-nodes-base.stickyNote | Global workflow description and setup guidance |  |  | ## How it works |
| Sticky Note7 | n8n-nodes-base.stickyNote | Global workflow description and setup guidance |  |  | This workflow converts CSV/XLSX files into a fully validated database schema. It extracts and profiles data using statistical analysis, detects patterns and relationships, and uses an AI agent to design normalized tables. The schema is validated against strict rules before generating SQL scripts, ERD diagrams, and a data dictionary, which are returned via webhook. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Global workflow description and setup guidance |  |  | ## Setup steps |
| Sticky Note7 | n8n-nodes-base.stickyNote | Global workflow description and setup guidance |  |  | 1. Activate webhook and upload CSV/XLSX file |
| Sticky Note7 | n8n-nodes-base.stickyNote | Global workflow description and setup guidance |  |  | 2. Configure OpenAI (GPT-4) credentials |
| Sticky Note7 | n8n-nodes-base.stickyNote | Global workflow description and setup guidance |  |  | 3. Adjust thresholds (FK overlap, PK uniqueness) |
| Sticky Note7 | n8n-nodes-base.stickyNote | Global workflow description and setup guidance |  |  | 4. Execute workflow |
| Sticky Note7 | n8n-nodes-base.stickyNote | Global workflow description and setup guidance |  |  | 5. Use generated SQL, ERD, and dictionary |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for SQL generation section |  |  | ## SQL Schema Generation |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation note for ERD/data dictionary section |  |  | ## ERD & Data Dictionary |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node named `Upload Files Webhook`.**
   - Type: Webhook
   - HTTP Method: `POST`
   - Path: `schema-builder`
   - Response Mode: `Last Node`
   - In Options, enable `Raw Body`
   - This will serve as the workflow entry point for uploaded files.

2. **Create a Set node named `Workflow Configuration`.**
   - Connect it after `Upload Files Webhook`
   - Enable keeping incoming fields
   - Add numeric fields:
     - `fkOverlapThreshold = 0.7`
     - `maxNormalizationDepth = 3`
     - `minUniquePercentForPK = 0.95`

3. **Create an IF node named `Check File Type`.**
   - Connect it after `Workflow Configuration`
   - Configure OR logic with two string conditions against `{{$json.data.mimeType}}`
   - Condition 1: contains `csv`
   - Condition 2: contains `text/csv`
   - True branch should represent CSV.
   - False branch should represent Excel.

4. **Create an `Extract From File` node named `Extract CSV Data`.**
   - Connect the true branch of `Check File Type` to it
   - Leave operation as CSV/default parsing
   - In options, enable `Include Empty Cells`
   - This node should parse CSV file contents into row objects.

5. **Create another `Extract From File` node named `Extract Excel Data`.**
   - Connect the false branch of `Check File Type` to it
   - Set operation to `xlsx`
   - Use defaults unless your workbook structure requires sheet-specific tuning

6. **Create a Code node named `Column Profiling Engine`.**
   - Connect both `Extract CSV Data` and `Extract Excel Data` to it
   - Paste the profiling JavaScript logic that:
     - reads all input items
     - finds row arrays from fields like `data` or `rows`
     - computes dataset metadata
     - profiles each column
     - flags candidate IDs and reference columns
     - returns one profiled JSON object per dataset
   - Ensure your n8n environment supports `require('crypto')` in Code nodes.

7. **Create an Aggregate node named `Aggregate All Datasets`.**
   - Connect it after `Column Profiling Engine`
   - Aggregate mode: `Aggregate All Item Data`
   - Destination field name: `datasets`

8. **Create an OpenAI Chat Model node named `OpenAI GPT`.**
   - Type: LangChain OpenAI Chat Model
   - Model: `gpt-4o`
   - Configure OpenAI credentials
   - No extra tools required

9. **Create a Structured Output Parser node named `Schema JSON Parser`.**
   - Type: LangChain Structured Output Parser
   - Schema Type: `Manual`
   - Provide a JSON schema requiring:
     - `tables`
     - `relationships`
     - `indexes`
   - Important: if you want this workflow to work correctly end-to-end, align parser field names with downstream Code nodes.
   - Recommended normalized field names for compatibility:
     - columns: `name`, `dataType`, `nullable`, `isPrimaryKey`, `isUnique`, `sourceColumns`
     - relationships: `sourceTable`, `sourceColumn`, `targetTable`, `targetColumn`, `overlapPercent`
     - indexes: `name`, `tableName`, `columns`, `isUnique`

10. **Create an AI Agent node named `Schema Reasoning Agent`.**
    - Connect `Aggregate All Datasets` to its main input
    - Connect `OpenAI GPT` to its AI language model input
    - Connect `Schema JSON Parser` to its output parser input
    - Set prompt type to a manually defined prompt
    - Use input text:
      - `={{ JSON.stringify($json.datasets) }}`
    - Add a system prompt instructing the model to:
      - detect entities from profiled datasets
      - create normalized tables
      - infer relationships only when overlap and semantics support them
      - define constraints and indexes
      - return JSON only in the exact parser-defined structure

11. **Create a Code node named `Rule Validator`.**
    - Connect it after `Schema Reasoning Agent`
    - Paste validation code that:
      - reads the AI schema
      - fetches config from `Workflow Configuration`
      - validates primary keys, foreign keys, type compatibility, orphan tables, over/under-normalization
    - Important: update the code or parser schema so field names are fully aligned.
    - As shipped, there is a mismatch between:
      - parser fields like `type`, `primaryKey`, `fromTable`
      - validator fields like `dataType`, `isPrimaryKey`, `sourceTable`

12. **Create a Code node named `SQL Schema Generator`.**
    - Connect it after `Rule Validator`
    - Paste code that converts schema objects into SQL DDL
    - It should:
      - create tables first
      - add foreign keys second
      - create indexes third
    - Again, ensure expected field names match actual schema output.

13. **Create a Code node named `Generate Data Dictionary & ERD`.**
    - Connect it after `SQL Schema Generator`
    - If you want the current implementation to work, do not feed only a SQL string.
    - Recommended setup:
      - either modify `SQL Schema Generator` to also output the structured schema
      - or rewrite this node to parse the SQL string
    - The current code expects an array of table objects with nested columns, not a plain SQL script string.

14. **Create a `Respond to Webhook` node named `Return Results`.**
    - Connect it after `Generate Data Dictionary & ERD`
    - Set response code to `200`
    - Respond with JSON
    - Build a body containing:
      - `sqlSchema`
      - `dataDictionary`
      - `erdDiagram`

15. **Add optional Sticky Notes for documentation.**
    - Add notes for:
      - Input & Configuration
      - File Detection & Extraction
      - Column Profiling Engine and Dataset Aggregation
      - AI Schema Reasoning
      - Schema Validation
      - SQL Schema Generation
      - ERD & Data Dictionary
      - Final Output
      - Global workflow purpose and setup steps

16. **Configure credentials and environment.**
    - OpenAI credentials are required for `OpenAI GPT`
    - Verify Code nodes are allowed to run JS and import `crypto`
    - Ensure webhook/file upload handling produces binary or structured file data compatible with `Extract From File`

17. **Test with a sample CSV first.**
    - Send a POST request to the webhook
    - Confirm that:
      - MIME type exists at `data.mimeType`
      - extractor node emits row objects
      - profiler outputs at least one dataset
      - agent output matches parser schema exactly

18. **Resolve the current schema-field inconsistencies before production use.**
    - Best fix: standardize all schema object names across:
      - AI prompt
      - output parser
      - validator
      - SQL generator
      - ERD/data dictionary node
    - Recommended canonical names:
      - Column: `name`, `dataType`, `nullable`, `isPrimaryKey`, `isUnique`, `sourceColumns`
      - Relationship: `sourceTable`, `sourceColumn`, `targetTable`, `targetColumn`, `overlapPercent`
      - Index: `name`, `tableName`, `columns`, `isUnique`

19. **Fix the final documentation block.**
    - Either:
      - pass the structured validated schema into `Generate Data Dictionary & ERD`, or
      - update that node to use the structured schema instead of `sqlSchema` string
    - Without this change, the ERD and dictionary output will not be reliable.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI Multi-Table Schema Builder with Automated Normalization and Relationship Detection | Global workflow note |
| This workflow converts CSV/XLSX files into a fully validated database schema. It extracts and profiles data using statistical analysis, detects patterns and relationships, and uses an AI agent to design normalized tables. The schema is validated against strict rules before generating SQL scripts, ERD diagrams, and a data dictionary, which are returned via webhook. | Workflow purpose |
| Activate webhook and upload CSV/XLSX file | Setup guidance |
| Configure OpenAI (GPT-4) credentials | Setup guidance |
| Adjust thresholds (FK overlap, PK uniqueness) | Setup guidance |
| Execute workflow | Setup guidance |
| Use generated SQL, ERD, and dictionary | Setup guidance |

## Additional implementation notes
- The workflow is conceptually strong, but the current JSON contains several structural incompatibilities between AI output, validation code, SQL generation code, and ERD generation code.
- The biggest issues are:
  1. **Field-name mismatch** across parser, validator, and SQL generator
  2. **Data-shape mismatch** where the ERD/data dictionary node expects structured schema objects but receives a SQL string
  3. **Implicit XLSX fallback** for any non-CSV MIME type
  4. **Threshold unit inconsistency** between values such as `0.95` and checks expecting `95`
- Before relying on this in production, align the schema contract across all downstream nodes.