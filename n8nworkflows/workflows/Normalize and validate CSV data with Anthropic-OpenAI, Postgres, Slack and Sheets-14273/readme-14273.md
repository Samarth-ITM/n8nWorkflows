Normalize and validate CSV data with Anthropic/OpenAI, Postgres, Slack and Sheets

https://n8nworkflows.xyz/workflows/normalize-and-validate-csv-data-with-anthropic-openai--postgres--slack-and-sheets-14273


# Normalize and validate CSV data with Anthropic/OpenAI, Postgres, Slack and Sheets

# 1. Workflow Overview

This workflow receives a CSV file through an HTTP webhook, validates that the uploaded file is actually a CSV, extracts its rows, uses an AI model to infer schema and normalize headers, applies type coercion and data normalization, evaluates data quality, then splits downstream handling into two paths:

- **Clean / accepted data** → inserted into Postgres and followed by a Slack notification
- **Invalid / problematic data or unsupported files** → converted into an error report and logged into Google Sheets

Its main use cases are:

- Normalizing inconsistent CSV headers before database insertion
- Detecting type mismatches, missing values, and statistical outliers
- Logging import problems for later review
- Notifying stakeholders when processing succeeds

## 1.1 Input Reception and Configuration

The workflow begins with a webhook that accepts file uploads via `POST`, then sets reusable runtime values such as the target Postgres table, acceptable error threshold, and Slack channel.

## 1.2 File Validation and CSV Extraction

The uploaded binary file is checked to confirm it is a CSV based on MIME type or filename extension. Valid CSVs are parsed into structured rows; invalid files are routed into an error object.

## 1.3 AI Schema Detection

The extracted CSV rows are sent to an AI agent configured for schema inference and header normalization. A structured output parser constrains the AI output into a predictable JSON schema.

## 1.4 Normalization and Validation

Custom JavaScript applies inferred normalization logic, coerces values into detected types, optionally standardizes units, and adds metadata. Another code node then analyzes row quality, detects type inconsistencies, missing values, and numeric outliers, and produces a summary with clean rows and error rows.

## 1.5 Successful Data Persistence and Notification

The “clean” branch prepares a success payload, inserts data into Postgres, and sends a Slack message reporting processing completion.

## 1.6 Error Reporting and Logging

The error branch creates an error-report object and appends or updates a Google Sheets log for auditability.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Configuration

### Overview
This block exposes the ingestion endpoint and defines workflow-level configuration values used later for database insertion, error thresholding, and Slack delivery.

### Nodes Involved
- CSV Upload Webhook
- Workflow Configuration

### Node Details

#### CSV Upload Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point for file upload requests.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `csv-upload`
  - Response mode: `lastNode`, meaning the webhook response is returned only when the execution reaches the final node in the active branch.
- **Key expressions or variables used:**  
  No dynamic expressions in node configuration itself, but downstream nodes reference:
  - `$binary.data.fileName`
  - `$binary.data.mimeType`
- **Input and output connections:**  
  - No input (entry node)
  - Output → `Workflow Configuration`
- **Version-specific requirements:**  
  Uses webhook node type version `2.1`.
- **Edge cases or potential failure types:**  
  - If the incoming request does not contain a binary file in the expected property (`data`), downstream binary expressions may fail.
  - Since response mode is `lastNode`, a branch that ends unexpectedly can cause no meaningful response or execution failure.
  - If the webhook is not activated/published, external uploads will fail.
- **Sub-workflow reference:**  
  None.

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds static configuration values to the execution context.
- **Configuration choices:**  
  Defines:
  - `postgresTable`: placeholder value for the destination Postgres table
  - `errorThreshold`: `0.05`
  - `slackChannel`: placeholder value for the Slack channel ID
  - `includeOtherFields: true`, so original fields from the webhook item are preserved.
- **Key expressions or variables used:**  
  No computed expressions; values are static placeholders except the numeric threshold.
- **Input and output connections:**  
  - Input ← `CSV Upload Webhook`
  - Output → `Check File Type`
- **Version-specific requirements:**  
  Uses Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - Placeholder values must be replaced before production use.
  - `errorThreshold` is defined but never actually used by downstream logic, creating a configuration/implementation mismatch.
- **Sub-workflow reference:**  
  None.

---

## 2.2 File Validation and CSV Extraction

### Overview
This block determines whether the uploaded file is a CSV and either parses it into rows or generates an unsupported-file error payload.

### Nodes Involved
- Check File Type
- Extract CSV Data
- Error - Unsupported File Type

### Node Details

#### Check File Type
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional routing based on file metadata.
- **Configuration choices:**  
  Uses OR logic with two loose string checks:
  1. MIME type contains `csv`
  2. Filename ends with `.csv`
- **Key expressions or variables used:**  
  - `{{ $binary.data.mimeType }}`
  - `{{ $binary.data.fileName }}`
- **Input and output connections:**  
  - Input ← `Workflow Configuration`
  - True output → `Extract CSV Data`
  - False output → `Error - Unsupported File Type`
- **Version-specific requirements:**  
  Uses If node version `2.3`.
- **Edge cases or potential failure types:**  
  - If no binary property `data` exists, expressions may resolve to undefined and the file may be rejected or error.
  - MIME type values can vary by client; relying on `contains csv` may miss unusual but valid content types.
  - A `.csv` extension does not guarantee valid CSV content.
- **Sub-workflow reference:**  
  None.

#### Extract CSV Data
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Parses the uploaded CSV binary into itemized row data.
- **Configuration choices:**  
  - `includeEmptyCells: true`, preserving blank columns rather than collapsing them.
- **Key expressions or variables used:**  
  No explicit expressions configured.
- **Input and output connections:**  
  - Input ← `Check File Type` (true branch)
  - Output → `Schema Inference & Header Normalization`
- **Version-specific requirements:**  
  Uses Extract From File version `1.1`.
- **Edge cases or potential failure types:**  
  - Malformed CSV formatting may break parsing or produce inconsistent rows.
  - Very large files may create memory or execution-time pressure.
  - Empty cells are included, which is useful for validation but increases null-handling demands downstream.
- **Sub-workflow reference:**  
  None.

#### Error - Unsupported File Type
- **Type and technical role:** `n8n-nodes-base.set`  
  Produces a structured failure payload when the uploaded file is not recognized as CSV.
- **Configuration choices:**  
  Sets:
  - `error`: fixed message
  - `fileName`: from uploaded binary
  - `mimeType`: from uploaded binary
  - `status`: `failed`
- **Key expressions or variables used:**  
  - `{{ $binary.data.fileName }}`
  - `{{ $binary.data.mimeType }}`
- **Input and output connections:**  
  - Input ← `Check File Type` (false branch)
  - Output → `Generate Error Report`
- **Version-specific requirements:**  
  Uses Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - If no binary metadata exists, filename/mime type fields may be empty.
  - This node does not preserve a dedicated error code; downstream systems only get a free-text message.
- **Sub-workflow reference:**  
  None.

---

## 2.3 AI Schema Detection

### Overview
This block sends extracted CSV rows to an AI agent that infers schema, proposes canonical headers, and assesses quality. The model output is constrained using a structured parser.

### Nodes Involved
- Schema Inference & Header Normalization
- Anthropic Chat Model
- Structured Output Parser

### Node Details

#### Schema Inference & Header Normalization
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI agent node orchestrating prompt execution with an attached language model and structured parser.
- **Configuration choices:**  
  - Prompt type: `define`
  - User text input: serialized CSV row data  
    `CSV data to analyze: {{ JSON.stringify($json) }}`
  - System message instructs the model to:
    - infer data types
    - map original headers to canonical names
    - identify unit/currency inconsistencies
    - detect null patterns and data quality issues
    - return structured schema data
  - Output parser enabled
- **Key expressions or variables used:**  
  - `{{ JSON.stringify($json) }}`
- **Input and output connections:**  
  - Main input ← `Extract CSV Data`
  - AI language model input ← `Anthropic Chat Model`
  - AI output parser input ← `Structured Output Parser`
  - Main output → `Apply Normalization & Type Coercion`
- **Version-specific requirements:**  
  Uses LangChain agent version `3`.
- **Edge cases or potential failure types:**  
  - Large CSV payloads can exceed model token limits because the entire JSON is stringified into the prompt.
  - AI output may comply with parser schema but still not match downstream code expectations.
  - Hallucinated canonical fields or wrong type inference may degrade normalization.
  - Credential or API quota issues can stop the workflow.
- **Sub-workflow reference:**  
  None.

#### Anthropic Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
  Supplies the language model used by the agent.
- **Configuration choices:**  
  - Model: `claude-sonnet-4-5-20250929`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output (AI language model) → `Schema Inference & Header Normalization`
- **Version-specific requirements:**  
  Uses version `1.3`.
- **Edge cases or potential failure types:**  
  - Requires Anthropic credentials.
  - The chosen model identifier may depend on account availability and n8n node support.
  - Timeouts, rate limits, or invalid API keys are likely failure modes.
- **Sub-workflow reference:**  
  None.

#### Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a manual JSON schema for AI output.
- **Configuration choices:**  
  Manual schema expects:
  - `columns`: array of objects with:
    - `original_header`
    - `canonical_header`
    - `data_type`
    - `detected_issues`
    - `sample_values`
  - `row_count`
  - `quality_score`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output (AI output parser) → `Schema Inference & Header Normalization`
- **Version-specific requirements:**  
  Uses version `1.3`.
- **Edge cases or potential failure types:**  
  - Downstream code expects `schema.fields`, `originalName`, `canonicalName`, and `type`, but parser returns `columns`, `original_header`, `canonical_header`, and `data_type`. This is a major schema mismatch.
  - If the model returns invalid JSON or incomplete required structure, the node may fail.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Normalization and Validation

### Overview
This block applies the inferred schema to raw CSV rows, coerces values into normalized formats, optionally converts units, then runs validation logic to identify problematic rows and summarize data quality.

### Nodes Involved
- Apply Normalization & Type Coercion
- Validate Data Quality

### Node Details

#### Apply Normalization & Type Coercion
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node for schema-based normalization.
- **Configuration choices:**  
  The script:
  - reads AI schema from `Schema Inference & Header Normalization`
  - reads all parsed CSV rows from `Extract CSV Data`
  - defines helper functions:
    - `coerceValue()` for number/date/boolean/string conversion
    - `standardizeUnit()` for selected unit conversions
  - tries to apply schema mappings if `schema.fields` exists
  - otherwise falls back to copying original row data unchanged
  - appends `_metadata` with:
    - `rowNumber`
    - `normalizedAt`
    - `schemaApplied`
- **Key expressions or variables used inside code:**  
  - `$('Schema Inference & Header Normalization').first().json`
  - `$('Extract CSV Data').all()`
- **Input and output connections:**  
  - Main input ← `Schema Inference & Header Normalization`
  - Main output → `Validate Data Quality`
- **Version-specific requirements:**  
  Uses Code node version `2`.
- **Edge cases or potential failure types:**  
  - **Critical mismatch:** AI parser returns `columns`, but this code checks `schema.fields`. In the current form, normalization mapping will likely never apply, and raw data will simply pass through.
  - Dates are parsed with JavaScript `new Date(value)`, which is locale-sensitive and may produce inconsistent results.
  - Currency stripping only removes `[$,€£¥]`; other symbols or localized separators may fail.
  - Boolean parsing is narrow but reasonable; values outside expected tokens become `null`.
  - Unit conversion logic exists but no upstream node supplies `unit` or `targetUnit` in the parser schema.
- **Sub-workflow reference:**  
  None.

#### Validate Data Quality
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript validation and error classification node.
- **Configuration choices:**  
  The script:
  - reads all items from input
  - infers per-column dominant type
  - detects numeric outliers using IQR
  - flags:
    - `type_mismatch`
    - `outlier`
    - `missing_value`
  - separates rows into `cleanData` and `errorReport`
  - produces a summary object with total rows and error rate
- **Key expressions or variables used inside code:**  
  Uses `$input.all()`
- **Input and output connections:**  
  - Input ← `Apply Normalization & Type Coercion`
  - Output 1 → `Prepare Clean CSV Output`
  - Output 1 also → `Generate Error Report`
- **Version-specific requirements:**  
  Uses Code node version `2`.
- **Edge cases or potential failure types:**  
  - **Data-structure mismatch:** this code tries to read `items[0]?.json?.normalizedData`, but the previous code node outputs one item per row, not a wrapper object with `normalizedData`. It falls back to `items.map(item => item.json)`, which is fine, but indicates inconsistent design.
  - `_metadata` added in the previous node becomes a validated column too, which may distort type statistics and error detection.
  - Missing-value logic depends on `totalRows` during row iteration, but `totalRows` is incremented inside the same loop, making the threshold unstable per row.
  - Summary fields are nested under `summary`, but downstream nodes expect top-level fields like `rowsProcessed`, `errorRate`, and `totalErrors`.
  - Error rate is returned as a string with `%`, which later nodes sometimes treat as numeric.
- **Sub-workflow reference:**  
  None.

---

## 2.5 Successful Data Persistence and Notification

### Overview
This block prepares the result payload for successful processing, inserts accepted data into Postgres, and sends a Slack confirmation.

### Nodes Involved
- Prepare Clean CSV Output
- Insert into Postgres
- Send Notification

### Node Details

#### Prepare Clean CSV Output
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a simplified success payload for downstream storage and notification.
- **Configuration choices:**  
  Sets:
  - `cleanData` from `$json.cleanData`
  - `rowsProcessed` from `$json.rowsProcessed`
  - `errorRate` from `$json.errorRate`
  - `status` = `success`
- **Key expressions or variables used:**  
  - `{{ $json.cleanData }}`
  - `{{ $json.rowsProcessed }}`
  - `{{ $json.errorRate }}`
- **Input and output connections:**  
  - Input ← `Validate Data Quality`
  - Output → `Insert into Postgres`
- **Version-specific requirements:**  
  Uses Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - **Field mismatch:** `Validate Data Quality` returns `summary.totalRows`, not `rowsProcessed`, so `rowsProcessed` will likely be empty.
  - `cleanData` is assigned as a string type even though it is actually an array/object payload.
  - This node outputs a summary object, not individual rows, which does not match the expected shape for row-wise Postgres insertion.
- **Sub-workflow reference:**  
  None.

#### Insert into Postgres
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Inserts data into a Postgres table using auto-mapped input fields.
- **Configuration choices:**  
  - Schema: `public`
  - Table name from workflow config:
    `{{ $('Workflow Configuration').first().json.postgresTable }}`
  - Column mode: `autoMapInputData`
- **Key expressions or variables used:**  
  - `{{ $('Workflow Configuration').first().json.postgresTable }}`
- **Input and output connections:**  
  - Input ← `Prepare Clean CSV Output`
  - Output → `Send Notification`
- **Version-specific requirements:**  
  Uses Postgres node version `2.6`.
- **Edge cases or potential failure types:**  
  - Requires valid Postgres credentials.
  - Auto-mapping depends on table columns matching incoming JSON keys.
  - **Likely functional issue:** it receives a summary object containing `cleanData`, not one item per database row. This may insert only one record with aggregate fields or fail schema validation.
  - Table placeholder must be replaced.
- **Sub-workflow reference:**  
  None.

#### Send Notification
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a Slack message after database insertion.
- **Configuration choices:**  
  - Sends plain text to a selected channel
  - Channel ID taken from workflow configuration
  - Text includes rows processed, error rate, and status
- **Key expressions or variables used:**  
  - `{{ $json.rowsProcessed }}`
  - `{{ $json.errorRate }}`
  - `{{ $json.status }}`
  - `{{ $('Workflow Configuration').first().json.slackChannel }}`
- **Input and output connections:**  
  - Input ← `Insert into Postgres`
  - No downstream node
- **Version-specific requirements:**  
  Uses Slack node version `2.4`.
- **Edge cases or potential failure types:**  
  - Requires Slack credentials and channel access.
  - If upstream fields are missing, the message will contain blanks.
  - This branch only sends a success notification; no Slack alert exists for failures.
- **Sub-workflow reference:**  
  None.

---

## 2.6 Error Reporting and Logging

### Overview
This block converts validation or file-type failures into a log-friendly structure and writes the result to Google Sheets.

### Nodes Involved
- Generate Error Report
- Log to Google Sheets

### Node Details

#### Generate Error Report
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a compact error-log payload for Google Sheets.
- **Configuration choices:**  
  Sets:
  - `errorReport` from `$json.errorReport`
  - `totalErrors` from `$json.totalErrors`
  - `errorRate` from `$json.errorRate`
  - `timestamp` from current time
  - `fileName` from original webhook binary
- **Key expressions or variables used:**  
  - `{{ $json.errorReport }}`
  - `{{ $json.totalErrors }}`
  - `{{ $json.errorRate }}`
  - `{{ $now.toISO() }}`
  - `{{ $('CSV Upload Webhook').first().binary.data.fileName }}`
- **Input and output connections:**  
  - Input ← `Validate Data Quality`
  - Input ← `Error - Unsupported File Type`
  - Output → `Log to Google Sheets`
- **Version-specific requirements:**  
  Uses Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - **Field mismatch:** validation output uses `summary.errorRows`, not `totalErrors`; this value will likely be empty.
  - When triggered by unsupported-file branch, there is no `errorReport` array in the input, so logged content may be blank.
  - If binary metadata is unavailable, `fileName` may be missing.
- **Sub-workflow reference:**  
  None.

#### Log to Google Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends or updates rows in a Google Sheet for error tracking.
- **Configuration choices:**  
  - Operation: `appendOrUpdate`
  - Document ID: placeholder
  - Sheet name: `Error Log`
  - Matching column: `fileName`
  - Mapped columns:
    - `fileName`
    - `errorRate`
    - `timestamp`
    - `errorReport`
    - `totalErrors`
- **Key expressions or variables used:**  
  - `{{ $json.fileName }}`
  - `{{ $json.errorRate }}`
  - `{{ $json.timestamp }}`
  - `{{ $json.errorReport }}`
  - `{{ $json.totalErrors }}`
- **Input and output connections:**  
  - Input ← `Generate Error Report`
  - No downstream node
- **Version-specific requirements:**  
  Uses Google Sheets node version `4.7`.
- **Edge cases or potential failure types:**  
  - Requires Google Sheets credentials and document access.
  - Placeholder document ID must be replaced.
  - `errorReport` is likely a complex array/object; when mapped into a sheet cell, serialization may not be ideal unless converted to string first.
  - Matching by `fileName` can overwrite prior entries for repeated filenames.
- **Sub-workflow reference:**  
  None.

---

## 2.7 Documentation / Visual Annotation Nodes

### Overview
These sticky notes are visual documentation only. They do not affect execution, but they provide context, setup hints, and grouping guidance in the canvas.

### Nodes Involved
- Sticky Note1
- Sticky Note
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7

### Node Details

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual annotation for upload trigger area.
- **Configuration choices:**  
  Content: “Upload Trigger — Receives CSV file via webhook for processing.”
- **Input and output connections:** None
- **Version-specific requirements:** Version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: “Database Insert — Stores cleaned data into Postgres table and notify user”
- **Input and output connections:** None
- **Version-specific requirements:** Version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: “Error Handling — Generates error report for invalid or failed rows and Logs errors and reports into Google Sheets.”
- **Input and output connections:** None
- **Version-specific requirements:** Version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: “Data Normalization and Validation — Cleans data, converts types, and standardizes formats and Checks quality, detects errors, missing values, and outliers.”
- **Input and output connections:** None
- **Version-specific requirements:** Version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: “Schema Detection — AI infers schema, types, and normalizes column names.”
- **Input and output connections:** None
- **Version-specific requirements:** Version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: “CSV Extraction — Parses CSV file into structured rows for processing.”
- **Input and output connections:** None
- **Version-specific requirements:** Version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content: “Configuration — Defines Postgres table, error threshold, and Slack channel and Checks if uploaded file is a valid CSV format.”
- **Input and output connections:** None
- **Version-specific requirements:** Version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note7
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Content includes a workflow explanation and setup checklist.
- **Input and output connections:** None
- **Version-specific requirements:** Version `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| CSV Upload Webhook | webhook | Receives CSV uploads via HTTP POST |  | Workflow Configuration | ## Upload Trigger<br>Receives CSV file via webhook for processing.<br><br>## How it works<br>This workflow processes CSV files via webhook upload. It validates the file, extracts data, and uses AI to detect schema and standardize columns. The data is cleaned, normalized, and checked for errors like missing values or outliers. Clean data is stored in Postgres, while errors are logged and shared via Slack.<br><br>## Setup<br>1. Configure webhook endpoint for CSV upload<br>2. Set Postgres table name<br>3. Add Anthropic/OpenAI credentials<br>4. Connect Slack for notifications<br>5. Connect Google Sheets for error logs<br>6. Adjust error threshold settings<br>7. Test with sample CSV files<br>8. Activate the workflow |
| Workflow Configuration | set | Stores runtime configuration values | CSV Upload Webhook | Check File Type | ## Configuration<br>Defines Postgres table, error threshold, and Slack channel and Checks if uploaded file is a valid CSV format.<br><br>## How it works<br>This workflow processes CSV files via webhook upload. It validates the file, extracts data, and uses AI to detect schema and standardize columns. The data is cleaned, normalized, and checked for errors like missing values or outliers. Clean data is stored in Postgres, while errors are logged and shared via Slack.<br><br>## Setup<br>1. Configure webhook endpoint for CSV upload<br>2. Set Postgres table name<br>3. Add Anthropic/OpenAI credentials<br>4. Connect Slack for notifications<br>5. Connect Google Sheets for error logs<br>6. Adjust error threshold settings<br>7. Test with sample CSV files<br>8. Activate the workflow |
| Check File Type | if | Validates uploaded file metadata as CSV | Workflow Configuration | Extract CSV Data; Error - Unsupported File Type | ## Configuration<br>Defines Postgres table, error threshold, and Slack channel and Checks if uploaded file is a valid CSV format.<br><br>## How it works<br>This workflow processes CSV files via webhook upload. It validates the file, extracts data, and uses AI to detect schema and standardize columns. The data is cleaned, normalized, and checked for errors like missing values or outliers. Clean data is stored in Postgres, while errors are logged and shared via Slack.<br><br>## Setup<br>1. Configure webhook endpoint for CSV upload<br>2. Set Postgres table name<br>3. Add Anthropic/OpenAI credentials<br>4. Connect Slack for notifications<br>5. Connect Google Sheets for error logs<br>6. Adjust error threshold settings<br>7. Test with sample CSV files<br>8. Activate the workflow |
| Extract CSV Data | extractFromFile | Parses CSV binary into row items | Check File Type | Schema Inference & Header Normalization | ## CSV Extraction<br>Parses CSV file into structured rows for processing.<br><br>## How it works<br>This workflow processes CSV files via webhook upload. It validates the file, extracts data, and uses AI to detect schema and standardize columns. The data is cleaned, normalized, and checked for errors like missing values or outliers. Clean data is stored in Postgres, while errors are logged and shared via Slack.<br><br>## Setup<br>1. Configure webhook endpoint for CSV upload<br>2. Set Postgres table name<br>3. Add Anthropic/OpenAI credentials<br>4. Connect Slack for notifications<br>5. Connect Google Sheets for error logs<br>6. Adjust error threshold settings<br>7. Test with sample CSV files<br>8. Activate the workflow |
| Error - Unsupported File Type | set | Creates failure payload for non-CSV uploads | Check File Type | Generate Error Report | ## CSV Extraction<br>Parses CSV file into structured rows for processing.<br><br>## How it works<br>This workflow processes CSV files via webhook upload. It validates the file, extracts data, and uses AI to detect schema and standardize columns. The data is cleaned, normalized, and checked for errors like missing values or outliers. Clean data is stored in Postgres, while errors are logged and shared via Slack.<br><br>## Setup<br>1. Configure webhook endpoint for CSV upload<br>2. Set Postgres table name<br>3. Add Anthropic/OpenAI credentials<br>4. Connect Slack for notifications<br>5. Connect Google Sheets for error logs<br>6. Adjust error threshold settings<br>7. Test with sample CSV files<br>8. Activate the workflow |
| Schema Inference & Header Normalization | @n8n/n8n-nodes-langchain.agent | Uses AI to infer schema and canonical headers | Extract CSV Data; Anthropic Chat Model; Structured Output Parser | Apply Normalization & Type Coercion | ## Schema Detection<br>AI infers schema, types, and normalizes column names.<br><br>## How it works<br>This workflow processes CSV files via webhook upload. It validates the file, extracts data, and uses AI to detect schema and standardize columns. The data is cleaned, normalized, and checked for errors like missing values or outliers. Clean data is stored in Postgres, while errors are logged and shared via Slack.<br><br>## Setup<br>1. Configure webhook endpoint for CSV upload<br>2. Set Postgres table name<br>3. Add Anthropic/OpenAI credentials<br>4. Connect Slack for notifications<br>5. Connect Google Sheets for error logs<br>6. Adjust error threshold settings<br>7. Test with sample CSV files<br>8. Activate the workflow |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Provides Claude model for schema inference |  | Schema Inference & Header Normalization | ## Schema Detection<br>AI infers schema, types, and normalizes column names. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Constrains AI output into JSON schema |  | Schema Inference & Header Normalization | ## Schema Detection<br>AI infers schema, types, and normalizes column names. |
| Apply Normalization & Type Coercion | code | Normalizes rows and coerces values to inferred types | Schema Inference & Header Normalization | Validate Data Quality | ## Data Normalization and Validation<br>Cleans data, converts types, and standardizes formats and Checks quality, detects errors, missing values, and outliers.<br><br>## How it works<br>This workflow processes CSV files via webhook upload. It validates the file, extracts data, and uses AI to detect schema and standardize columns. The data is cleaned, normalized, and checked for errors like missing values or outliers. Clean data is stored in Postgres, while errors are logged and shared via Slack.<br><br>## Setup<br>1. Configure webhook endpoint for CSV upload<br>2. Set Postgres table name<br>3. Add Anthropic/OpenAI credentials<br>4. Connect Slack for notifications<br>5. Connect Google Sheets for error logs<br>6. Adjust error threshold settings<br>7. Test with sample CSV files<br>8. Activate the workflow |
| Validate Data Quality | code | Detects type issues, outliers, and missing values; separates clean vs error rows | Apply Normalization & Type Coercion | Prepare Clean CSV Output; Generate Error Report | ## Data Normalization and Validation<br>Cleans data, converts types, and standardizes formats and Checks quality, detects errors, missing values, and outliers.<br><br>## How it works<br>This workflow processes CSV files via webhook upload. It validates the file, extracts data, and uses AI to detect schema and standardize columns. The data is cleaned, normalized, and checked for errors like missing values or outliers. Clean data is stored in Postgres, while errors are logged and shared via Slack.<br><br>## Setup<br>1. Configure webhook endpoint for CSV upload<br>2. Set Postgres table name<br>3. Add Anthropic/OpenAI credentials<br>4. Connect Slack for notifications<br>5. Connect Google Sheets for error logs<br>6. Adjust error threshold settings<br>7. Test with sample CSV files<br>8. Activate the workflow |
| Prepare Clean CSV Output | set | Prepares success payload for DB insert and Slack | Validate Data Quality | Insert into Postgres | ## Database Insert<br>Stores cleaned data into Postgres table and notify user |
| Insert into Postgres | postgres | Writes processed data to Postgres | Prepare Clean CSV Output | Send Notification | ## Database Insert<br>Stores cleaned data into Postgres table and notify user |
| Send Notification | slack | Sends Slack success message | Insert into Postgres |  | ## Database Insert<br>Stores cleaned data into Postgres table and notify user |
| Generate Error Report | set | Builds error log payload for validation or unsupported-file failures | Validate Data Quality; Error - Unsupported File Type | Log to Google Sheets | ## Error Handling<br>Generates error report for invalid or failed rows and Logs errors and reports into Google Sheets. |
| Log to Google Sheets | googleSheets | Appends/updates error logs in Google Sheets | Generate Error Report |  | ## Error Handling<br>Generates error report for invalid or failed rows and Logs errors and reports into Google Sheets. |
| Sticky Note1 | stickyNote | Visual documentation for upload area |  |  |  |
| Sticky Note | stickyNote | Visual documentation for database area |  |  |  |
| Sticky Note2 | stickyNote | Visual documentation for error-handling area |  |  |  |
| Sticky Note3 | stickyNote | Visual documentation for normalization area |  |  |  |
| Sticky Note4 | stickyNote | Visual documentation for schema detection area |  |  |  |
| Sticky Note5 | stickyNote | Visual documentation for CSV extraction area |  |  |  |
| Sticky Note6 | stickyNote | Visual documentation for configuration area |  |  |  |
| Sticky Note7 | stickyNote | Visual documentation with workflow description and setup checklist |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is the exact rebuild sequence in n8n.

## 1. Create the webhook trigger
1. Add a **Webhook** node named **CSV Upload Webhook**.
2. Configure:
   - **HTTP Method:** `POST`
   - **Path:** `csv-upload`
   - **Response Mode:** `Last Node`
3. Ensure your incoming request sends the uploaded file as binary data, typically in property `data`.

## 2. Add workflow-level configuration
4. Add a **Set** node named **Workflow Configuration** after the webhook.
5. Enable **Include Other Input Fields**.
6. Add these fields:
   - `postgresTable` → string → your Postgres table name
   - `errorThreshold` → number → `0.05`
   - `slackChannel` → string → your Slack channel ID
7. Connect **CSV Upload Webhook → Workflow Configuration**.

## 3. Validate that the file is CSV
8. Add an **If** node named **Check File Type**.
9. Configure conditions with **OR** combinator:
   - Condition 1:
     - Left value: `{{ $binary.data.mimeType }}`
     - Operation: `contains`
     - Right value: `csv`
   - Condition 2:
     - Left value: `{{ $binary.data.fileName }}`
     - Operation: `endsWith`
     - Right value: `.csv`
10. Connect **Workflow Configuration → Check File Type**.

## 4. Parse valid CSV files
11. Add an **Extract From File** node named **Extract CSV Data**.
12. Configure:
   - Enable **Include Empty Cells**
13. Connect the **true** output of **Check File Type** to **Extract CSV Data**.

## 5. Build unsupported-file handling
14. Add a **Set** node named **Error - Unsupported File Type**.
15. Add fields:
   - `error` → string → `Unsupported file type. Only CSV files are accepted.`
   - `fileName` → string → `{{ $binary.data.fileName }}`
   - `mimeType` → string → `{{ $binary.data.mimeType }}`
   - `status` → string → `failed`
16. Connect the **false** output of **Check File Type** to this node.

## 6. Add AI schema inference
17. Add an **AI Agent** node named **Schema Inference & Header Normalization**.
18. Set **Prompt Type** to **Define**.
19. Set the main text input to:
   - `CSV data to analyze: {{ JSON.stringify($json) }}`
20. In **System Message**, paste the following logic in equivalent form:
   - The assistant specializes in CSV schema inference and normalization
   - Infer column data types
   - Map original headers to canonical names
   - Identify currency/unit inconsistencies
   - Detect null/data-quality issues
   - Return structured schema including original header, canonical header, data type, issues, sample values
21. Enable **Output Parser**.
22. Connect **Extract CSV Data → Schema Inference & Header Normalization**.

## 7. Add the Anthropic model
23. Add an **Anthropic Chat Model** node named **Anthropic Chat Model**.
24. Select model:
   - `Claude Sonnet 4.5` / `claude-sonnet-4-5-20250929` as available in your n8n version
25. Configure **Anthropic credentials**.
26. Connect this node to the AI Agent through the **AI Language Model** connection.

## 8. Add structured output enforcement
27. Add a **Structured Output Parser** node named **Structured Output Parser**.
28. Set **Schema Type** to **Manual**.
29. Define a JSON schema with:
   - top-level object
   - `columns` array of objects containing:
     - `original_header`
     - `canonical_header`
     - `data_type`
     - `detected_issues` as string array
     - `sample_values` as string array
   - `row_count` number
   - `quality_score` number
30. Connect this parser to the AI Agent through the **AI Output Parser** connection.

## 9. Add normalization code
31. Add a **Code** node named **Apply Normalization & Type Coercion**.
32. Connect **Schema Inference & Header Normalization → Apply Normalization & Type Coercion**.
33. Paste the provided JavaScript logic or equivalent functionality:
   - Read AI schema from the AI node
   - Read all CSV rows from **Extract CSV Data**
   - Coerce values into detected types
   - Convert booleans, dates, and numbers
   - Optionally standardize units
   - Rename fields using canonical headers
   - Add `_metadata`
34. Important implementation note: to make this workflow actually work correctly, adapt the code to use:
   - `schema.columns` instead of `schema.fields`
   - `original_header` instead of `originalName`
   - `canonical_header` instead of `canonicalName`
   - `data_type` instead of `type`

## 10. Add validation code
35. Add a **Code** node named **Validate Data Quality**.
36. Connect **Apply Normalization & Type Coercion → Validate Data Quality**.
37. Paste the JavaScript logic or equivalent:
   - determine dominant column types
   - detect outliers with IQR
   - flag missing values and type mismatches
   - produce:
     - summary
     - cleanData
     - errorReport
     - columnStats
38. Recommended correction for reliable downstream behavior:
   - output top-level fields such as:
     - `rowsProcessed`
     - `totalErrors`
     - `errorRate`
     - `cleanData`
     - `errorReport`
   instead of nesting everything only under `summary`

## 11. Prepare the success branch
39. Add a **Set** node named **Prepare Clean CSV Output**.
40. Connect **Validate Data Quality → Prepare Clean CSV Output**.
41. Add fields:
   - `cleanData` → `{{ $json.cleanData }}`
   - `rowsProcessed` → `{{ $json.rowsProcessed }}`
   - `errorRate` → `{{ $json.errorRate }}`
   - `status` → `success`
42. Recommended correction:
   - make `cleanData` a JSON/object field if supported, not string
   - ensure `rowsProcessed` comes from actual output, for example `{{ $json.summary.totalRows }}` if you keep the original validation code

## 12. Insert into Postgres
43. Add a **Postgres** node named **Insert into Postgres**.
44. Connect **Prepare Clean CSV Output → Insert into Postgres**.
45. Configure:
   - **Schema:** `public`
   - **Table:** `{{ $('Workflow Configuration').first().json.postgresTable }}`
   - **Column Mapping Mode:** `Auto-map input data`
46. Configure **Postgres credentials**.
47. Important correction:
   - If your goal is to insert each clean row, insert the rows themselves, not a wrapper object containing `cleanData`.
   - In practice, add an item-splitting step before Postgres or return one clean row per item from validation.

## 13. Add Slack notification
48. Add a **Slack** node named **Send Notification**.
49. Connect **Insert into Postgres → Send Notification**.
50. Configure:
   - Send message to **Channel**
   - **Channel ID:** `{{ $('Workflow Configuration').first().json.slackChannel }}`
   - Message text:
     - `CSV processing completed successfully!`
     - include rows processed, error rate, and status
51. Configure **Slack credentials**.
52. Ensure the Slack app has permission to post to the chosen channel.

## 14. Build the error-report branch
53. Add a **Set** node named **Generate Error Report**.
54. Connect:
   - **Validate Data Quality → Generate Error Report**
   - **Error - Unsupported File Type → Generate Error Report**
55. Add fields:
   - `errorReport` → `{{ $json.errorReport }}`
   - `totalErrors` → `{{ $json.totalErrors }}`
   - `errorRate` → `{{ $json.errorRate }}`
   - `timestamp` → `{{ $now.toISO() }}`
   - `fileName` → `{{ $('CSV Upload Webhook').first().binary.data.fileName }}`
56. Recommended correction:
   - if using the original validation output, map `totalErrors` from `{{ $json.summary.errorRows }}`
   - serialize error arrays with `JSON.stringify(...)` before storing in Sheets if needed

## 15. Log errors to Google Sheets
57. Add a **Google Sheets** node named **Log to Google Sheets**.
58. Connect **Generate Error Report → Log to Google Sheets**.
59. Configure:
   - **Operation:** `Append or Update`
   - **Document ID:** your Google Sheet document ID
   - **Sheet Name:** `Error Log`
   - **Matching Column:** `fileName`
60. Map columns:
   - `fileName` ← `{{ $json.fileName }}`
   - `timestamp` ← `{{ $json.timestamp }}`
   - `totalErrors` ← `{{ $json.totalErrors }}`
   - `errorRate` ← `{{ $json.errorRate }}`
   - `errorReport` ← `{{ $json.errorReport }}`
61. Configure **Google Sheets credentials**.

## 16. Add visual notes
62. Add **Sticky Note** nodes for documentation if you want the same visual grouping:
   - Upload Trigger
   - Configuration
   - CSV Extraction
   - Schema Detection
   - Data Normalization and Validation
   - Database Insert
   - Error Handling
   - Overall How it works / Setup

## 17. Final testing steps
63. Replace all placeholder values:
   - Postgres table
   - Slack channel ID
   - Google Sheets document ID
64. Activate all required credentials:
   - Anthropic
   - Postgres
   - Slack
   - Google Sheets
65. Test with:
   - a valid CSV
   - a non-CSV file
   - a CSV with missing fields
   - a CSV with mixed numeric/string columns
   - a CSV with outlier values
66. Activate the workflow only after confirming both success and error branches behave correctly.

### Sub-workflow setup
This workflow does **not** invoke any sub-workflow and does not require any child workflow configuration.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow processes CSV files via webhook upload. It validates the file, extracts data, and uses AI to detect schema and standardize columns. The data is cleaned, normalized, and checked for errors like missing values or outliers. Clean data is stored in Postgres, while errors are logged and shared via Slack. | General workflow description |
| Setup: 1. Configure webhook endpoint for CSV upload 2. Set Postgres table name 3. Add Anthropic/OpenAI credentials 4. Connect Slack for notifications 5. Connect Google Sheets for error logs 6. Adjust error threshold settings 7. Test with sample CSV files 8. Activate the workflow | General setup checklist |
| The title references Anthropic/OpenAI, but the actual workflow includes only an Anthropic chat model node and no OpenAI node. | Important implementation note |
| The configured `errorThreshold` is currently not consumed by the validation branch. If threshold-based branching is intended, additional logic is required. | Design gap |
| Several downstream mappings do not match upstream outputs: `schema.fields` vs `columns`, `rowsProcessed` vs `summary.totalRows`, `totalErrors` vs `summary.errorRows`. These should be corrected before production use. | Critical consistency note |
| Postgres insertion currently appears to receive an aggregate object rather than one item per clean row. A row-splitting transformation is likely required for practical database loading. | Data persistence note |