Generate production database schemas from Excel and CSV with OpenAI and LangChain

https://n8nworkflows.xyz/workflows/generate-production-database-schemas-from-excel-and-csv-with-openai-and-langchain-14317


# Generate production database schemas from Excel and CSV with OpenAI and LangChain

# 1. Workflow Overview

This workflow accepts an uploaded Excel or CSV file through a webhook, profiles the dataset, asks an OpenAI-powered LangChain agent to infer a normalized production database schema, validates that schema against deterministic rules, and then generates implementation artifacts such as SQL DDL, ERD/data dictionary output, and a load plan.

Its primary use cases are:

- Turning flat spreadsheet data into a proposed relational database design
- Accelerating schema design for migration or analytics projects
- Producing machine-consumable schema artifacts from semi-structured tabular data
- Adding AI-generated explanations of design decisions

The workflow is logically organized into the following blocks.

## 1.1 Input Reception and Runtime Configuration
Receives the uploaded file via webhook and injects configurable thresholds used later for validation and revision logic.

## 1.2 File Type Detection and Extraction
Determines whether the file is Excel-like or CSV-like, extracts rows accordingly, then merges extraction output into a single stream.

## 1.3 Data Hashing and Profiling
Computes a file hash, normalizes and deduplicates rows, detects data quality issues, and profiles columns for type, uniqueness, IDs, and possible foreign keys.

## 1.4 AI Schema Inference
Sends profiling results to a LangChain agent backed by OpenAI GPT-4o, with a structured output parser enforcing a schema-like JSON response.

## 1.5 Validation and Revision Loop
Checks whether the inferred schema satisfies deterministic business and structural rules. If invalid, prepares feedback and loops back to the schema agent.

## 1.6 Output Artifact Generation
Once validation succeeds, generates SQL DDL, ERD/data dictionary output, and a data loading plan.

## 1.7 Output Combination, Explanation, and Webhook Response
Combines generated outputs, optionally asks another LLM to explain the result, and returns the final JSON payload to the webhook caller.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Runtime Configuration

**Overview:**  
This block starts the workflow through an HTTP POST webhook and defines configurable thresholds that are meant to influence downstream validation and retry behavior.

**Nodes Involved:**  
- File Upload Webhook
- Workflow Configuration

### File Upload Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for receiving the uploaded file.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `schema-generator`
  - Response mode: `lastNode`
- **Key expressions or variables used:**  
  None directly in node config.
- **Input and output connections:**  
  - No input; workflow entry node
  - Outputs to `Workflow Configuration`
- **Version-specific requirements:**  
  Uses typeVersion `2.1`; standard webhook behavior in recent n8n versions.
- **Edge cases or potential failure types:**  
  - Request may not include binary file data
  - Wrong content type or multipart formatting
  - If multiple binaries are uploaded, later logic only checks the first binary key
- **Sub-workflow reference:**  
  None

### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Injects configurable runtime settings into the item.
- **Configuration choices:**  
  Adds:
  - `maxRevisionAttempts = 3`
  - `minForeignKeyOverlap = 0.7`
  - `confidenceThreshold = 0.8`
  Preserves incoming fields using `includeOtherFields: true`.
- **Key expressions or variables used:**  
  Static values only.
- **Input and output connections:**  
  - Input from `File Upload Webhook`
  - Output to `Check File Type`
- **Version-specific requirements:**  
  Uses Set node typeVersion `3.4`.
- **Edge cases or potential failure types:**  
  Minimal; mainly depends on upstream item presence.
- **Sub-workflow reference:**  
  None

---

## 2.2 File Type Detection and Extraction

**Overview:**  
This block determines whether the uploaded file should be treated as Excel or CSV, extracts tabular content, and merges both branches into one downstream item.

**Nodes Involved:**  
- Check File Type
- Extract Excel Data
- Extract CSV Data
- Merge Extracted Data

### Check File Type
- **Type and technical role:** `n8n-nodes-base.if`  
  Routes the workflow based on whether the first binary property name contains `xls`.
- **Configuration choices:**  
  Condition evaluates:
  `Object.keys($binary)[0].toLowerCase().includes('xls')`
- **Key expressions or variables used:**  
  - `$binary`
- **Input and output connections:**  
  - Input from `Workflow Configuration`
  - True output to `Extract Excel Data`
  - False output to `Extract CSV Data`
- **Version-specific requirements:**  
  Uses IF node typeVersion `2.3`.
- **Edge cases or potential failure types:**  
  - If `$binary` is missing, expression may fail
  - Detection is based on binary key name, not MIME type or filename, so false classification is possible
  - `.xlsx`/`.xls` only works if binary key naming reflects extension
- **Sub-workflow reference:**  
  None

### Extract Excel Data
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Parses Excel workbook content into row objects.
- **Configuration choices:**  
  - Operation: `xlsx`
  - Header row enabled
  - Empty cells excluded
  - Reads typed values rather than forcing strings
  - Sheet name left blank, implying default sheet behavior
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input from true branch of `Check File Type`
  - Output to `Merge Extracted Data` input 0
- **Version-specific requirements:**  
  Uses typeVersion `1.1`.
- **Edge cases or potential failure types:**  
  - Unsupported workbook format
  - Empty workbook or missing sheet
  - Bad header row causing malformed JSON keys
- **Sub-workflow reference:**  
  None

### Extract CSV Data
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Parses CSV content into rows.
- **Configuration choices:**  
  - Includes empty cells
  - Other CSV parsing options remain default
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Input from false branch of `Check File Type`
  - Output to `Merge Extracted Data` input 1
- **Version-specific requirements:**  
  Uses typeVersion `1.1`.
- **Edge cases or potential failure types:**  
  - Unexpected delimiter/encoding/quotes
  - Missing header row if the file is raw CSV without headers
  - Empty cells may produce sparse or null-like values later
- **Sub-workflow reference:**  
  None

### Merge Extracted Data
- **Type and technical role:** `n8n-nodes-base.merge`  
  Consolidates either extraction branch into one item.
- **Configuration choices:**  
  - Mode: `combine`
  - Combine by: `combineAll`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Inputs from `Extract Excel Data` and `Extract CSV Data`
  - Output to `Compute File Hash & Profile Data`
- **Version-specific requirements:**  
  Uses Merge node typeVersion `3.2`.
- **Edge cases or potential failure types:**  
  - With only one branch active, merged output structure may differ from what downstream code expects
  - Combined data shape can be ambiguous if both branches ever produce data
- **Sub-workflow reference:**  
  None

---

## 2.3 Data Hashing and Profiling

**Overview:**  
This block computes a deterministic file hash, cleans and normalizes raw rows, removes duplicates, and derives descriptive statistics and candidate relational signals from the data.

**Nodes Involved:**  
- Compute File Hash & Profile Data
- Column Profiling Engine

### Compute File Hash & Profile Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Performs custom JavaScript data normalization and summary profiling.
- **Configuration choices:**  
  The code:
  - Requires Node.js `crypto`
  - Reads all input items
  - Extracts `items[0].json.data || items[0].json || []`
  - Computes SHA-256 hash of the raw dataset
  - Normalizes numeric strings and some date formats
  - Detects per-column primary type and mixed-type patterns
  - Detects duplicate rows via MD5 row hashes
  - Returns:
    - `fileHash`
    - `totalRows`
    - `cleanRows`
    - `duplicateRows`
    - `headerCandidates`
    - `columnStatistics`
    - `sampleRows`
    - `summary`
- **Key expressions or variables used:**  
  Internal JS variables only.
- **Input and output connections:**  
  - Input from `Merge Extracted Data`
  - Output to `Column Profiling Engine`
- **Version-specific requirements:**  
  Code node typeVersion `2`.
- **Edge cases or potential failure types:**  
  - If extracted data is not in `.json.data`, fallback may not match actual structure
  - Returns an error object instead of throwing; downstream nodes may continue with invalid shape
  - Date parsing may be locale-sensitive
  - Header normalization is calculated but not actually applied to sample rows
- **Sub-workflow reference:**  
  None

### Column Profiling Engine
- **Type and technical role:** `n8n-nodes-base.code`  
  Performs deeper per-column analytics and attempts foreign-key inference.
- **Configuration choices:**  
  The code:
  - Uses `$input.all()`
  - Treats each input item as one row and scans `item.json`
  - Computes:
    - null counts
    - type distributions
    - dominant type
    - uniqueness and cardinality
    - ID-likeliness based on name and uniqueness
    - monotonic numeric sequences
    - foreign key candidates by overlap across columns
  - Returns:
    - `columnProfiles`
    - summary object
    - `rawData`
- **Key expressions or variables used:**  
  Internal JS variables only.
- **Input and output connections:**  
  - Input from `Compute File Hash & Profile Data`
  - Output to `Schema Reasoning Agent`
- **Version-specific requirements:**  
  Code node typeVersion `2`.
- **Edge cases or potential failure types:**  
  - There is a structural mismatch: upstream returns one item with aggregated fields, but this node expects multiple row items
  - It may end up profiling metadata fields like `fileHash`, `summary`, or `sampleRows` instead of actual dataset columns
  - FK inference compares columns within the same item stream, which is weak for single-table spreadsheet inputs
- **Sub-workflow reference:**  
  None

---

## 2.4 AI Schema Inference

**Overview:**  
This block asks a schema-design LLM agent to infer a relational schema from profiling outputs and constrains the output through a structured JSON schema parser.

**Nodes Involved:**  
- Schema Reasoning Agent
- Schema Output Parser
- OpenAI GPT

### Schema Reasoning Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain agent that generates a schema proposal.
- **Configuration choices:**  
  - Prompt type: defined manually
  - User text:
    `{{ $json.cleanSampleRows }} with column statistics: {{ $json.columnStats }}`
  - System message instructs:
    - header normalization
    - SQL type inference
    - table grouping
    - PK/FK/index decisions
    - nullability and check constraints
    - strict no-invention rules
  - Has structured output parser enabled
- **Key expressions or variables used:**  
  - `$json.cleanSampleRows`
  - `$json.columnStats`
- **Input and output connections:**  
  - Main input from `Column Profiling Engine`
  - Retry-loop input from `Prepare Revision Feedback`
  - AI language model input from `OpenAI GPT`
  - AI output parser input from `Schema Output Parser`
  - Main output to `Rules Validation Layer`
- **Version-specific requirements:**  
  Uses LangChain agent typeVersion `3`.
- **Edge cases or potential failure types:**  
  - Expression mismatch: upstream data provides `sampleRows` and `columnStatistics`, not `cleanSampleRows` and `columnStats`
  - Agent may receive undefined prompt sections
  - LLM may still produce shape mismatches despite parser
  - Token limits if sample data becomes large
- **Sub-workflow reference:**  
  None

### Schema Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Forces the agent to return JSON conforming to a manually defined schema.
- **Configuration choices:**  
  Manual JSON Schema includes:
  - `tables[]`
  - column definitions with fields such as `name`, `sql_type`, `nullable`, PK/FK flags, references, checks, descriptions
  - `primary_keys`
  - `foreign_keys`
  - `indexes`
  - top-level `reasoning`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Connected as parser to `Schema Reasoning Agent`
- **Version-specific requirements:**  
  Uses typeVersion `1.3`.
- **Edge cases or potential failure types:**  
  - Parser may reject LLM output if required fields are absent
  - Output shape does not match downstream validator assumptions
- **Sub-workflow reference:**  
  None

### OpenAI GPT
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the chat model used by the schema agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - No built-in tools configured
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Connected via `ai_languageModel` to `Schema Reasoning Agent`
- **Version-specific requirements:**  
  Uses typeVersion `1.3`; requires compatible n8n LangChain/OpenAI package support.
- **Edge cases or potential failure types:**  
  - Missing or invalid OpenAI credentials
  - Model unavailability or rate limits
  - Long prompts causing truncation or cost spikes
- **Sub-workflow reference:**  
  None

---

## 2.5 Validation and Revision Loop

**Overview:**  
This block validates the AI schema against deterministic rules and attempts to send corrective feedback back into the agent if validation fails.

**Nodes Involved:**  
- Rules Validation Layer
- Check Validation Result
- Prepare Revision Feedback

### Rules Validation Layer
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom validator for schema correctness against observed data.
- **Configuration choices:**  
  The code validates:
  - columns must exist in input data
  - declared types must match actual values
  - FK overlap must exceed configured threshold
  - no circular FK relationships
  - PK columns must not contain duplicates
  - DECIMAL precision warnings
  - check constraints versus value ranges
  Returns original schema data with:
  - `validation.isValid`
  - `validation.errors`
  - `validation.warnings`
- **Key expressions or variables used:**  
  Internal JS variables; reads:
  - `schemaData.schema || schemaData`
  - `schemaData.config || {}`
  - `schemaData.profileData || {}`
  - `schemaData.rawData || []`
- **Input and output connections:**  
  - Input from `Schema Reasoning Agent`
  - Output to `Check Validation Result`
- **Version-specific requirements:**  
  Code node typeVersion `2`.
- **Edge cases or potential failure types:**  
  - Major schema-field mismatch with parser output:
    - validator expects `table.name`, `column.type`, `table.foreignKeys`, `column.primaryKey`
    - parser defines `table_name`, `sql_type`, `foreign_keys`, `is_primary_key`
  - Validation may incorrectly fail or pass due to missing expected fields
  - Config from `Workflow Configuration` is not explicitly merged into agent output
- **Sub-workflow reference:**  
  None

### Check Validation Result
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches on validation success/failure.
- **Configuration choices:**  
  Condition checks:
  `{{ $json.validationResult.isValid }}`
- **Key expressions or variables used:**  
  - `$json.validationResult.isValid`
- **Input and output connections:**  
  - Input from `Rules Validation Layer`
  - True branch to:
    - `Generate SQL DDL`
    - `Generate ERD & Data Dictionary`
    - `Generate Load Plan`
  - False branch to `Prepare Revision Feedback`
- **Version-specific requirements:**  
  IF node typeVersion `2.3`.
- **Edge cases or potential failure types:**  
  - Validator actually returns `validation.isValid`, not `validationResult.isValid`
  - This mismatch likely sends all runs down the failure path or evaluates undefined
- **Sub-workflow reference:**  
  None

### Prepare Revision Feedback
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates retry feedback for the schema agent.
- **Configuration choices:**  
  Sets:
  - `revisionFeedback` string from validator errors/warnings
  - `attemptNumber` incremented
  - `originalData` serialized object using profiling outputs
  Keeps other fields.
- **Key expressions or variables used:**  
  - `$('Rules Validation Layer').item.json.errors`
  - `$('Rules Validation Layer').item.json.warnings`
  - `$json.attemptNumber`
  - `$('Column Profiling Engine').item.json.columnStats`
  - `$('Column Profiling Engine').item.json.sampleRows`
- **Input and output connections:**  
  - Input from false branch of `Check Validation Result`
  - Output back to `Schema Reasoning Agent`
- **Version-specific requirements:**  
  Set node typeVersion `3.4`.
- **Edge cases or potential failure types:**  
  - Validator returns `validation.errors` and `validation.warnings`, not top-level `errors`/`warnings`
  - Profiling node returns `columnProfiles` and `rawData`, not `columnStats` / `sampleRows`
  - `maxRevisionAttempts` is configured earlier but never enforced here
  - Risk of infinite loop if invalid forever
- **Sub-workflow reference:**  
  None

---

## 2.6 Output Artifact Generation

**Overview:**  
When validation passes, this block transforms the schema into implementation-ready outputs: SQL DDL, ERD/data dictionary, and a load order plan.

**Nodes Involved:**  
- Generate SQL DDL
- Generate ERD & Data Dictionary
- Generate Load Plan

### Generate SQL DDL
- **Type and technical role:** `n8n-nodes-base.code`  
  Converts schema JSON into SQL DDL statements.
- **Configuration choices:**  
  The code:
  - Reads `validatedSchema || schema || item.json`
  - Builds header comments
  - Emits `CREATE TABLE` definitions
  - Adds PK/UNIQUE constraints
  - Creates indexes
  - Emits foreign key `ALTER TABLE` statements from `relationships`
  - Adds design notes
- **Key expressions or variables used:**  
  Internal JS variables only.
- **Input and output connections:**  
  - Input from true branch of `Check Validation Result`
  - Output to `Combine Final Outputs` input 0
- **Version-specific requirements:**  
  Code node typeVersion `2`.
- **Edge cases or potential failure types:**  
  - Expects `table.name`, `col.dataType`, `relationships`, etc., which do not match parser output names
  - May generate empty or invalid SQL if structure differs
  - SQL dialect is generic, not tuned for PostgreSQL/MySQL/SQL Server specifics
- **Sub-workflow reference:**  
  None

### Generate ERD & Data Dictionary
- **Type and technical role:** `n8n-nodes-base.code`  
  Produces ERD summary, Mermaid ERD text, and tabular data dictionary entries.
- **Configuration choices:**  
  The code:
  - Reads `schema.tables` and `schema.relationships`
  - Computes table and relationship summaries
  - Generates one data dictionary row per column
  - Produces Mermaid ER diagram text
- **Key expressions or variables used:**  
  Internal JS variables only.
- **Input and output connections:**  
  - Input from true branch of `Check Validation Result`
  - Output to `Combine Final Outputs` input 1
- **Version-specific requirements:**  
  Code node typeVersion `2`.
- **Edge cases or potential failure types:**  
  - Again expects `table.name`, `column.type`, `relationships`; parser defines different names
  - Sample-value enrichment likely empty because profiling data is not mapped through
- **Sub-workflow reference:**  
  None

### Generate Load Plan
- **Type and technical role:** `n8n-nodes-base.code`  
  Builds dependency-aware insertion order and loading recommendations.
- **Configuration choices:**  
  The code:
  - Reads `schema.tables` and `schema.relationships`
  - Builds a dependency graph using `foreign_key` type relationships
  - Topologically sorts tables
  - Detects circular dependencies
  - Produces batched load recommendations and Mermaid dependency graph
- **Key expressions or variables used:**  
  Internal JS variables only.
- **Input and output connections:**  
  - Input from true branch of `Check Validation Result`
  - Output directly to `Return Schema Results`
- **Version-specific requirements:**  
  Code node typeVersion `2`.
- **Edge cases or potential failure types:**  
  - Expects relationship keys like `from_table`, `to_table`; inconsistent with parser output
  - Not merged with DDL/ERD outputs before response
  - Response path bypasses `Combine Final Outputs`, so final payload may lack merged artifacts
- **Sub-workflow reference:**  
  None

---

## 2.7 Output Combination, Explanation, and Webhook Response

**Overview:**  
This block attempts to merge generated artifacts, optionally enrich them with an explanation from a second LLM, and return the final JSON response.

**Nodes Involved:**  
- Combine Final Outputs
- Explanation Agent (Optional)
- OpenAI GPT-(Explanation)
- Return Schema Results

### Combine Final Outputs
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines outputs from SQL and ERD generation.
- **Configuration choices:**  
  - Mode: `combine`
  - Combine by: `combineAll`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Inputs from `Generate SQL DDL` and `Generate ERD & Data Dictionary`
  - Output to `Explanation Agent (Optional)`
- **Version-specific requirements:**  
  Merge node typeVersion `3.2`.
- **Edge cases or potential failure types:**  
  - `Generate Load Plan` is not merged here
  - Downstream response may not contain all intended fields
- **Sub-workflow reference:**  
  None

### Explanation Agent (Optional)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Uses an LLM to explain schema decisions in natural language.
- **Configuration choices:**  
  - Prompt text defaults to:
    `Provide a summary of the generated schema and key design decisions.`
    unless `userQuestion` is supplied
  - System message instructs explanation of FK choices, normalization, issues, indexes, etc.
- **Key expressions or variables used:**  
  - `$json.userQuestion || "Provide a summary of the generated schema and key design decisions."`
- **Input and output connections:**  
  - Main input from `Combine Final Outputs`
  - AI language model from `OpenAI GPT-(Explanation)`
  - Main output to `Return Schema Results`
- **Version-specific requirements:**  
  LangChain agent typeVersion `3`.
- **Edge cases or potential failure types:**  
  - If combine output lacks full schema context, explanation quality drops
  - No structured parser, so output format may vary
- **Sub-workflow reference:**  
  None

### OpenAI GPT-(Explanation)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Model backend for the explanation agent.
- **Configuration choices:**  
  - Model: `gpt-4o`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Connected via `ai_languageModel` to `Explanation Agent (Optional)`
- **Version-specific requirements:**  
  Uses typeVersion `1.3`.
- **Edge cases or potential failure types:**  
  - Same OpenAI credential and rate-limit concerns as the first model node
- **Sub-workflow reference:**  
  None

### Return Schema Results
- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns final JSON to the original webhook caller.
- **Configuration choices:**  
  Responds with JSON body containing:
  - `success`
  - `schema.sqlDDL`
  - `schema.erdSummary`
  - `schema.dataDictionary`
  - `schema.loadPlan`
  - `schema.aiExplanation`
  - metadata timestamp and `fileHash`
- **Key expressions or variables used:**  
  - `$json.sqlDDL`
  - `$json.erdSummary`
  - `$json.dataDictionary`
  - `$json.loadPlan`
  - `$json.explanation || null`
  - `$json.fileHash || null`
- **Input and output connections:**  
  - Input from `Explanation Agent (Optional)`
  - Also separately from `Generate Load Plan`
- **Version-specific requirements:**  
  Respond to Webhook typeVersion `1.5`.
- **Edge cases or potential failure types:**  
  - Two incoming branches create inconsistent payload context
  - If triggered from `Generate Load Plan`, fields like `sqlDDL` and `erdSummary` may be missing
  - If triggered from `Explanation Agent`, `loadPlan` may be missing because it was never merged in
  - `responseMode` on webhook is `lastNode`, so branch timing can affect final output
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| File Upload Webhook | n8n-nodes-base.webhook | Receives uploaded CSV/XLSX file via HTTP POST |  | Workflow Configuration | ## AI Data Schema Generator |
| File Upload Webhook | n8n-nodes-base.webhook | Receives uploaded CSV/XLSX file via HTTP POST |  | Workflow Configuration | ## Input & Config |
| Workflow Configuration | n8n-nodes-base.set | Defines runtime thresholds and preserves input fields | File Upload Webhook | Check File Type | ## AI Data Schema Generator |
| Workflow Configuration | n8n-nodes-base.set | Defines runtime thresholds and preserves input fields | File Upload Webhook | Check File Type | ## Input & Config |
| Check File Type | n8n-nodes-base.if | Routes file to Excel or CSV extraction path | Workflow Configuration | Extract Excel Data, Extract CSV Data | ## AI Data Schema Generator |
| Check File Type | n8n-nodes-base.if | Routes file to Excel or CSV extraction path | Workflow Configuration | Extract Excel Data, Extract CSV Data | ## Input & Config |
| Extract Excel Data | n8n-nodes-base.extractFromFile | Parses Excel workbook rows into JSON | Check File Type | Merge Extracted Data | ## AI Data Schema Generator |
| Extract Excel Data | n8n-nodes-base.extractFromFile | Parses Excel workbook rows into JSON | Check File Type | Merge Extracted Data | ## File Extraction |
| Extract CSV Data | n8n-nodes-base.extractFromFile | Parses CSV rows into JSON | Check File Type | Merge Extracted Data | ## AI Data Schema Generator |
| Extract CSV Data | n8n-nodes-base.extractFromFile | Parses CSV rows into JSON | Check File Type | Merge Extracted Data | ## File Extraction |
| Merge Extracted Data | n8n-nodes-base.merge | Combines extracted file data into one stream | Extract Excel Data, Extract CSV Data | Compute File Hash & Profile Data | ## AI Data Schema Generator |
| Merge Extracted Data | n8n-nodes-base.merge | Combines extracted file data into one stream | Extract Excel Data, Extract CSV Data | Compute File Hash & Profile Data | ## merge |
| Merge Extracted Data | n8n-nodes-base.merge | Combines extracted file data into one stream | Extract Excel Data, Extract CSV Data | Compute File Hash & Profile Data | ## File Extraction |
| Compute File Hash & Profile Data | n8n-nodes-base.code | Normalizes data, hashes file content, computes initial stats | Merge Extracted Data | Column Profiling Engine | ## AI Data Schema Generator |
| Compute File Hash & Profile Data | n8n-nodes-base.code | Normalizes data, hashes file content, computes initial stats | Merge Extracted Data | Column Profiling Engine | ## Column Analysis Engine |
| Column Profiling Engine | n8n-nodes-base.code | Performs deep profiling and foreign-key candidate detection | Compute File Hash & Profile Data | Schema Reasoning Agent | ## AI Data Schema Generator |
| Column Profiling Engine | n8n-nodes-base.code | Performs deep profiling and foreign-key candidate detection | Compute File Hash & Profile Data | Schema Reasoning Agent | ## Column Analysis Engine |
| Schema Reasoning Agent | @n8n/n8n-nodes-langchain.agent | Generates relational schema proposal from profiling data | Column Profiling Engine, Prepare Revision Feedback, OpenAI GPT, Schema Output Parser | Rules Validation Layer | ## AI Schema Generation |
| OpenAI GPT | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supplies GPT-4o model to schema agent |  | Schema Reasoning Agent | ## AI Schema Generation |
| Schema Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured schema JSON format |  | Schema Reasoning Agent | ## AI Schema Generation |
| Rules Validation Layer | n8n-nodes-base.code | Validates AI schema against deterministic rules | Schema Reasoning Agent | Check Validation Result | ## Validation Layer |
| Check Validation Result | n8n-nodes-base.if | Branches on schema validity | Rules Validation Layer | Generate SQL DDL, Generate ERD & Data Dictionary, Generate Load Plan, Prepare Revision Feedback | ## Validation Layer |
| Prepare Revision Feedback | n8n-nodes-base.set | Builds retry feedback for schema regeneration | Check Validation Result | Schema Reasoning Agent | ## Revision Handling |
| Generate SQL DDL | n8n-nodes-base.code | Produces SQL CREATE TABLE and constraint statements | Check Validation Result | Combine Final Outputs | ## Schema Output Generation |
| Generate ERD & Data Dictionary | n8n-nodes-base.code | Produces ERD summary, Mermaid ERD, and data dictionary | Check Validation Result | Combine Final Outputs | ## Schema Output Generation |
| Generate Load Plan | n8n-nodes-base.code | Computes table load order and dependency plan | Check Validation Result | Return Schema Results | ## Schema Output Generation |
| Generate Load Plan | n8n-nodes-base.code | Computes table load order and dependency plan | Check Validation Result | Return Schema Results | ## Load Plan Engine |
| Combine Final Outputs | n8n-nodes-base.merge | Merges SQL and ERD outputs | Generate SQL DDL, Generate ERD & Data Dictionary | Explanation Agent (Optional) | ## Combine & Explain |
| Explanation Agent (Optional) | @n8n/n8n-nodes-langchain.agent | Produces natural-language explanation of schema decisions | Combine Final Outputs, OpenAI GPT-(Explanation) | Return Schema Results | ## Combine & Explain |
| OpenAI GPT-(Explanation) | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supplies GPT-4o model to explanation agent |  | Explanation Agent (Optional) | ## Combine & Explain |
| Return Schema Results | n8n-nodes-base.respondToWebhook | Returns final JSON payload to webhook caller | Explanation Agent (Optional), Generate Load Plan |  | ## Response Output |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation/comment node |  |  | ## AI Data Schema Generator |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation/comment node |  |  | ## Input & Config |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation/comment node |  |  | ## merge |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation/comment node |  |  | ## Combine & Explain |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation/comment node |  |  | ## File Extraction |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation/comment node |  |  | ## Column Analysis Engine |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation/comment node |  |  | ## Validation Layer |
| Sticky Note9 | n8n-nodes-base.stickyNote | Documentation/comment node |  |  | ## AI Schema Generation |
| Sticky Note11 | n8n-nodes-base.stickyNote | Documentation/comment node |  |  | ## Revision Handling |
| Sticky Note12 | n8n-nodes-base.stickyNote | Documentation/comment node |  |  | ## Schema Output Generation |
| Sticky Note13 | n8n-nodes-base.stickyNote | Documentation/comment node |  |  | ## Response Output |
| Sticky Note14 | n8n-nodes-base.stickyNote | Documentation/comment node |  |  | ## Load Plan Engine |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**
   - Name it `File Upload Webhook`.
   - Set method to `POST`.
   - Set path to `schema-generator`.
   - Set response mode to `Last Node`.
   - Expect multipart/form-data file upload.

2. **Add a Set node for configuration**
   - Name it `Workflow Configuration`.
   - Enable keeping incoming fields.
   - Add numeric fields:
     - `maxRevisionAttempts = 3`
     - `minForeignKeyOverlap = 0.7`
     - `confidenceThreshold = 0.8`
   - Connect `File Upload Webhook -> Workflow Configuration`.

3. **Add an IF node for file type detection**
   - Name it `Check File Type`.
   - Configure one boolean expression:
     - `={{ Object.keys($binary)[0].toLowerCase().includes('xls') }}`
   - Connect `Workflow Configuration -> Check File Type`.

4. **Add an Extract From File node for Excel**
   - Name it `Extract Excel Data`.
   - Operation: `xlsx`.
   - Use header row: enabled.
   - Leave sheet name blank.
   - Set:
     - `rawData = false`
     - `readAsString = false`
     - `includeEmptyCells = false`
   - Connect the **true** output of `Check File Type` to this node.

5. **Add an Extract From File node for CSV**
   - Name it `Extract CSV Data`.
   - Use default CSV extraction settings, but enable `includeEmptyCells = true`.
   - Connect the **false** output of `Check File Type` to this node.

6. **Add a Merge node**
   - Name it `Merge Extracted Data`.
   - Mode: `Combine`.
   - Combine by: `Combine All`.
   - Connect:
     - `Extract Excel Data -> Merge Extracted Data` input 1
     - `Extract CSV Data -> Merge Extracted Data` input 2

7. **Add a Code node for hashing and initial profiling**
   - Name it `Compute File Hash & Profile Data`.
   - Paste the provided JavaScript logic that:
     - loads extracted rows
     - computes SHA-256 hash
     - normalizes numeric/date values
     - detects duplicate rows
     - creates header and column stats
     - returns sample rows and summary
   - Connect `Merge Extracted Data -> Compute File Hash & Profile Data`.

8. **Add a Code node for deep column profiling**
   - Name it `Column Profiling Engine`.
   - Paste the provided JavaScript that:
     - inspects column nulls/types/cardinality
     - flags potential IDs
     - detects monotonic numeric fields
     - computes FK overlap candidates
   - Connect `Compute File Hash & Profile Data -> Column Profiling Engine`.

9. **Add an OpenAI Chat Model node**
   - Name it `OpenAI GPT`.
   - Choose `gpt-4o`.
   - Configure OpenAI credentials.
   - No tools required.

10. **Add a Structured Output Parser node**
    - Name it `Schema Output Parser`.
    - Choose manual schema mode.
    - Paste the JSON Schema that defines:
      - `tables`
      - `columns`
      - `primary_keys`
      - `foreign_keys`
      - `indexes`
      - `reasoning`

11. **Add a LangChain Agent node for schema generation**
    - Name it `Schema Reasoning Agent`.
    - Set prompt type to defined/manual.
    - Set the user text to:
      - `={{ $json.cleanSampleRows }} with column statistics: {{ $json.columnStats }}`
    - Add the long system instruction describing schema design constraints.
    - Enable structured output parser.
    - Connect:
      - `Column Profiling Engine -> Schema Reasoning Agent`
      - `OpenAI GPT -> Schema Reasoning Agent` as AI language model
      - `Schema Output Parser -> Schema Reasoning Agent` as AI output parser

12. **Add a Code node for validation**
    - Name it `Rules Validation Layer`.
    - Paste the validation JavaScript that checks:
      - column existence
      - type compatibility
      - FK overlap thresholds
      - circular dependencies
      - PK duplicate violations
      - decimal precision warnings
      - check constraints
    - Connect `Schema Reasoning Agent -> Rules Validation Layer`.

13. **Add an IF node for validation status**
    - Name it `Check Validation Result`.
    - Configure boolean condition:
      - `={{ $json.validationResult.isValid }}`
    - Connect `Rules Validation Layer -> Check Validation Result`.

14. **Add a Set node for revision feedback**
    - Name it `Prepare Revision Feedback`.
    - Enable keep other fields.
    - Add fields:
      - `revisionFeedback`
      - `attemptNumber`
      - `originalData`
    - Use expressions exactly as in the workflow JSON.
    - Connect the **false** branch of `Check Validation Result` to this node.
    - Connect `Prepare Revision Feedback -> Schema Reasoning Agent`.

15. **Add a Code node for SQL output**
    - Name it `Generate SQL DDL`.
    - Paste the JavaScript that renders:
      - CREATE TABLE statements
      - PK/UNIQUE constraints
      - indexes
      - ALTER TABLE foreign keys
    - Connect the **true** branch of `Check Validation Result` to this node.

16. **Add a Code node for ERD and data dictionary**
    - Name it `Generate ERD & Data Dictionary`.
    - Paste the JavaScript that creates:
      - ERD summary
      - Mermaid ERD code
      - one row per column in data dictionary
    - Connect the **true** branch of `Check Validation Result` to this node.

17. **Add a Code node for load planning**
    - Name it `Generate Load Plan`.
    - Paste the JavaScript that:
      - creates dependency graph
      - computes topological load order
      - detects circular dependencies
      - proposes batches and recommendations
    - Connect the **true** branch of `Check Validation Result` to this node.

18. **Add a Merge node for final artifacts**
    - Name it `Combine Final Outputs`.
    - Mode: `Combine`.
    - Combine by: `Combine All`.
    - Connect:
      - `Generate SQL DDL -> Combine Final Outputs`
      - `Generate ERD & Data Dictionary -> Combine Final Outputs`

19. **Add a second OpenAI Chat Model node**
    - Name it `OpenAI GPT-(Explanation)`.
    - Model: `gpt-4o`.
    - Reuse or create OpenAI credentials.

20. **Add a second LangChain Agent node**
    - Name it `Explanation Agent (Optional)`.
    - Prompt text:
      - `={{ $json.userQuestion || "Provide a summary of the generated schema and key design decisions." }}`
    - Use the explanation-focused system message from the workflow.
    - Connect:
      - `Combine Final Outputs -> Explanation Agent (Optional)`
      - `OpenAI GPT-(Explanation) -> Explanation Agent (Optional)` as AI language model

21. **Add a Respond to Webhook node**
    - Name it `Return Schema Results`.
    - Set response type to JSON.
    - Use the JSON body template from the workflow:
      - `success`
      - `schema.sqlDDL`
      - `schema.erdSummary`
      - `schema.dataDictionary`
      - `schema.loadPlan`
      - `schema.aiExplanation`
      - metadata timestamp and file hash
    - Connect:
      - `Explanation Agent (Optional) -> Return Schema Results`
      - `Generate Load Plan -> Return Schema Results`

22. **Add sticky notes for documentation**
    - Add notes matching the workflow comments if you want visual parity:
      - AI Data Schema Generator
      - Input & Config
      - File Extraction
      - merge
      - Column Analysis Engine
      - AI Schema Generation
      - Validation Layer
      - Revision Handling
      - Schema Output Generation
      - Load Plan Engine
      - Combine & Explain
      - Response Output

23. **Configure credentials**
    - For both OpenAI nodes:
      - Add valid OpenAI API credentials in n8n.
      - Ensure the account has access to `gpt-4o`.
    - No additional external credentials are required for the provided workflow.

24. **Activate and test the webhook**
    - Activate the workflow.
    - POST a CSV or XLSX file to the production webhook endpoint.
    - Confirm that the binary file field name is compatible with the `Check File Type` logic.

25. **Important implementation note**
    - To make this workflow fully functional, you should fix field-name mismatches across nodes:
      - `sampleRows` vs `cleanSampleRows`
      - `columnStatistics` vs `columnStats`
      - `validation` vs `validationResult`
      - `table_name` vs `name`
      - `sql_type` vs `type` or `dataType`
      - `foreign_keys` vs `foreignKeys`
      - `is_primary_key` vs `primaryKey` / `isPrimaryKey`
    - You should also add a retry limit check using `maxRevisionAttempts` to avoid infinite loops.
    - You should merge `Generate Load Plan` with the other outputs before responding, otherwise the final JSON will be incomplete depending on branch timing.

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows. No Execute Workflow node is present.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI Data Schema Generator: accepts CSV/XLSX via webhook, extracts and cleans data, profiles columns and relationships, asks AI for a normalized schema, validates it, then produces SQL DDL, ERD, data dictionary, and load plan outputs. | General workflow purpose |
| Setup steps note: Activate the webhook and send a CSV/XLSX file; configure AI (OpenAI / LangChain credentials); adjust thresholds (confidence, FK overlap) if needed; review generated schema, SQL, and ERD outputs. | General usage guidance |
| The workflow contains several internal field-name mismatches that likely prevent correct end-to-end execution without adjustment. | Implementation caution |
| The response stage currently has two inbound branches and no final merge with the load-plan branch, which can produce incomplete responses. | Architecture caution |
| The revision loop has no effective retry cap even though `maxRevisionAttempts` is configured. | Reliability caution |