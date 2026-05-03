No AI suggestion available

https://n8nworkflows.xyz/workflows/no-ai-suggestion-available-14276


# No AI suggestion available

# 1. Workflow Overview

This workflow automates financial reconciliation for records arriving from multiple sources: bank statements, invoices, ERP exports, and CSV uploads. It receives incoming data through a webhook, standardizes records into a common schema, performs deterministic matching, optionally escalates weaker matches to an AI-based fuzzy matching stage, calculates a final confidence score, routes the result either to automatic reconciliation or human review, then logs and reports the outcome.

The workflow is structured into the following logical blocks:

## 1.1 Input Reception and Runtime Configuration
The workflow starts from a webhook and immediately injects runtime configuration values such as confidence thresholds and matching keys.

## 1.2 Source Detection and Routing
An IF node checks the incoming source type and routes data to one of several normalization branches.

## 1.3 Source-Specific Normalization
Each supported source format is mapped into a unified schema with fields such as `recordId`, `amount`, `date`, `description`, `sourceType`, and `normalizedAt`.

## 1.4 Consolidation of Normalized Records
The normalized outputs are merged through chained Merge nodes to create a combined dataset for downstream matching.

## 1.5 Deterministic Matching
A Code node performs exact and semi-exact record matching using combinations of ID, amount, and date, and assigns initial confidence scores.

## 1.6 Match Quality Evaluation and AI Escalation
A quality check is intended to identify low-confidence matches and send them to an AI agent for fuzzy matching, while stronger matches proceed directly.

## 1.7 Confidence Scoring and Audit Enrichment
Deterministic and AI-derived scores are combined into a final confidence score, with a generated audit trail.

## 1.8 Decision Routing
Records above the configured confidence threshold are auto-reconciled; lower-confidence results are flagged for human review.

## 1.9 Reporting and Notification
The final reconciliation outcomes are written into Google Sheets and then summarized to Slack.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Runtime Configuration

**Overview:**  
This block receives financial data via HTTP POST and appends core workflow configuration values that are referenced later during matching and routing. These parameters define the acceptance threshold for automatic reconciliation and the threshold for AI escalation.

**Nodes Involved:**  
- Webhook - Receive Financial Data  
- Workflow Configuration

### Node Details

#### Webhook - Receive Financial Data
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point of the workflow.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `financial-data-reconciliation`
  - Response mode: `lastNode`
- **Key expressions or variables used:**  
  None in the node itself.
- **Input and output connections:**  
  - Input: none
  - Output: `Workflow Configuration`
- **Version-specific requirements:**  
  Uses webhook node version `2.1`.
- **Edge cases or potential failure types:**  
  - Incorrect webhook URL or method
  - Payload missing expected fields such as `sourceType`
  - Large file or binary payload handling issues if CSV uploads are expected
  - Because response mode is `lastNode`, any downstream failure can make the webhook request appear failed
- **Sub-workflow reference:**  
  None

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Injects workflow-wide parameters.
- **Configuration choices:**  
  Adds:
  - `confidenceThreshold = 0.85`
  - `matchingKeys = ["transactionId", "invoiceNumber", "amount", "date"]`
  - `fuzzyMatchThreshold = 0.7`
  `includeOtherFields` is enabled, so the original incoming payload is preserved.
- **Key expressions or variables used:**  
  Static values only.
- **Input and output connections:**  
  - Input: `Webhook - Receive Financial Data`
  - Output: `Check Data Source Type`
- **Version-specific requirements:**  
  Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - None likely internally
  - Later nodes may rely on these values even if the source payload is malformed
- **Sub-workflow reference:**  
  None

---

## 2.2 Source Detection and Routing

**Overview:**  
This block determines whether the incoming payload is a bank statement. If true, it routes to bank normalization; otherwise it sends the payload to invoice, ERP, and CSV-related branches simultaneously.

**Nodes Involved:**  
- Check Data Source Type

### Node Details

#### Check Data Source Type
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional router based on incoming `sourceType`.
- **Configuration choices:**  
  Checks whether:
  - `{{$json.sourceType}} == "bank_statement"`
- **Key expressions or variables used:**  
  - `={{ $json.sourceType }}`
- **Input and output connections:**  
  - Input: `Workflow Configuration`
  - True output: `Normalize Bank Statement Schema`
  - False output: `Normalize Invoice Schema`, `Normalize ERP Schema`, `Extract CSV Data`
- **Version-specific requirements:**  
  IF node version `2.3`.
- **Edge cases or potential failure types:**  
  - If `sourceType` is missing, the node routes to the false branch
  - Current logic does not distinguish between `invoice`, `erp`, and `csv_upload`; all three false-branch nodes are triggered together
  - This may create invalid normalization attempts and downstream noise if payloads are not shaped for all three branches
- **Sub-workflow reference:**  
  None

---

## 2.3 Source-Specific Normalization

**Overview:**  
This block transforms each supported source format into a common schema for matching. Each branch creates standardized fields and preserves original fields for traceability.

**Nodes Involved:**  
- Extract CSV Data  
- Normalize Bank Statement Schema  
- Normalize Invoice Schema  
- Normalize ERP Schema  
- Normalize CSV Schema

### Node Details

#### Extract CSV Data
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Extracts structured data from an uploaded file, presumably for CSV input.
- **Configuration choices:**  
  No explicit extraction options are configured.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Check Data Source Type` false branch
  - Output: `Normalize CSV Schema`
- **Version-specific requirements:**  
  Version `1.1`.
- **Edge cases or potential failure types:**  
  - Requires binary file input in the expected property; this workflow does not explicitly define binary property mapping
  - If the webhook sends plain JSON instead of binary CSV content, extraction will fail
  - Malformed CSV, unsupported delimiter, encoding issues
- **Sub-workflow reference:**  
  None

#### Normalize Bank Statement Schema
- **Type and technical role:** `n8n-nodes-base.set`  
  Maps bank statement fields to the unified schema.
- **Configuration choices:**  
  Creates:
  - `recordId = {{$json.transaction_id}}`
  - `amount = {{$json.transaction_amount}}`
  - `date = {{$json.transaction_date}}`
  - `description = {{$json.transaction_description}}`
  - `sourceType = "bank_statement"`
  - `normalizedAt = {{$now.toISO()}}`
  Keeps original fields.
- **Key expressions or variables used:**  
  - `={{ $json.transaction_id }}`
  - `={{ $json.transaction_amount }}`
  - `={{ $json.transaction_date }}`
  - `={{ $json.transaction_description }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**  
  - Input: `Check Data Source Type` true branch
  - Output: `Merge All Normalized Data`
- **Version-specific requirements:**  
  Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - Missing or differently named transaction fields
  - Amount type conversion issues
  - Invalid date formats can still pass as strings but later affect matching
- **Sub-workflow reference:**  
  None

#### Normalize Invoice Schema
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**  
  Creates:
  - `recordId = {{$json.invoice_number}}`
  - `amount = {{$json.invoice_total}}`
  - `date = {{$json.invoice_date}}`
  - `description = {{$json.invoice_description}}`
  - `sourceType = "invoice"`
  - `normalizedAt = {{$now.toISO()}}`
- **Key expressions or variables used:**  
  - `={{ $json.invoice_number }}`
  - `={{ $json.invoice_total }}`
  - `={{ $json.invoice_date }}`
  - `={{ $json.invoice_description }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**  
  - Input: `Check Data Source Type` false branch
  - Output: `Merge All Normalized Data1`
- **Version-specific requirements:**  
  Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - If the payload is actually ERP or CSV data, these invoice fields may be empty
  - Empty `recordId` or `amount` will degrade deterministic matching
- **Sub-workflow reference:**  
  None

#### Normalize ERP Schema
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**  
  Creates:
  - `recordId = {{$json.erp_transaction_id}}`
  - `amount = {{$json.erp_amount}}`
  - `date = {{$json.erp_date}}`
  - `description = {{$json.erp_description}}`
  - `sourceType = "erp"`
  - `normalizedAt = {{$now.toISO()}}`
- **Key expressions or variables used:**  
  - `={{ $json.erp_transaction_id }}`
  - `={{ $json.erp_amount }}`
  - `={{ $json.erp_date }}`
  - `={{ $json.erp_description }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**  
  - Input: `Check Data Source Type` false branch
  - Output: `Merge All Normalized Data1`
- **Version-specific requirements:**  
  Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - Same routing issue as above: this node runs for any non-bank payload
  - Missing ERP fields produce partially empty normalized records
- **Sub-workflow reference:**  
  None

#### Normalize CSV Schema
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**  
  Creates:
  - `recordId = {{$json.id}}`
  - `amount = {{$json.amount}}`
  - `date = {{$json.date}}`
  - `description = {{$json.description}}`
  - `sourceType = "csv_upload"`
  - `normalizedAt = {{$now.toISO()}}`
- **Key expressions or variables used:**  
  - `={{ $json.id }}`
  - `={{ $json.amount }}`
  - `={{ $json.date }}`
  - `={{ $json.description }}`
  - `={{ $now.toISO() }}`
- **Input and output connections:**  
  - Input: `Extract CSV Data`
  - Output: `Merge All Normalized Data`
- **Version-specific requirements:**  
  Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - Depends on successful CSV extraction
  - CSV column names must match the expected names exactly
- **Sub-workflow reference:**  
  None

---

## 2.4 Consolidation of Normalized Records

**Overview:**  
This block combines normalized outputs into a single stream for matching. The workflow uses three Merge nodes in sequence to consolidate multiple branches.

**Nodes Involved:**  
- Merge All Normalized Data  
- Merge All Normalized Data1  
- Merge All Normalized Data2

### Node Details

#### Merge All Normalized Data
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines bank and CSV normalized outputs.
- **Configuration choices:**  
  - Mode: `combine`
  - Combine by: `combineAll`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Inputs: `Normalize CSV Schema`, `Normalize Bank Statement Schema`
  - Output: `Merge All Normalized Data2`
- **Version-specific requirements:**  
  Merge version `3.2`.
- **Edge cases or potential failure types:**  
  - `combineAll` can produce cross-combined outputs rather than simple append behavior depending on item counts
  - If one branch produces no items, behavior must be verified in the target n8n version
- **Sub-workflow reference:**  
  None

#### Merge All Normalized Data1
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines invoice and ERP normalized outputs.
- **Configuration choices:**  
  - Mode: `combine`
  - Combine by: `combineAll`
- **Input and output connections:**  
  - Inputs: `Normalize Invoice Schema`, `Normalize ERP Schema`
  - Output: `Merge All Normalized Data2`
- **Version-specific requirements:**  
  Merge version `3.2`.
- **Edge cases or potential failure types:**  
  - Same `combineAll` caution as above
  - If both invoice and ERP run for the same non-bank payload, this may merge unrelated records
- **Sub-workflow reference:**  
  None

#### Merge All Normalized Data2
- **Type and technical role:** `n8n-nodes-base.merge`  
  Final consolidation before deterministic matching.
- **Configuration choices:**  
  - Mode: `combine`
  - Combine by: `combineAll`
- **Input and output connections:**  
  - Inputs: `Merge All Normalized Data`, `Merge All Normalized Data1`
  - Output: `Deterministic Matching Logic`
- **Version-specific requirements:**  
  Merge version `3.2`.
- **Edge cases or potential failure types:**  
  - Final dataset shape may differ from expectations if upstream Merge semantics are not append-like
- **Sub-workflow reference:**  
  None

---

## 2.5 Deterministic Matching

**Overview:**  
This block applies rule-based reconciliation using normalized `recordId`, `amount`, and `date` values. It generates grouped match candidates, assigns confidence levels, and labels unmatched items.

**Nodes Involved:**  
- Deterministic Matching Logic  
- Check Match Quality

### Node Details

#### Deterministic Matching Logic
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript for exact and partial matching.
- **Configuration choices:**  
  The script:
  - Reads all incoming items with `$input.all()`
  - Normalizes values:
    - lowercase trimmed strings for IDs
    - numeric formatting for amounts
    - ISO `YYYY-MM-DD` dates
  - Builds matching keys with descending confidence:
    - `recordId|amount|date` → `1.0`
    - `recordId|amount` → `0.85`
    - `recordId|date` → `0.85`
    - `amount|date` → `0.7`
    - `recordId` → `0.6`
  - Groups records by key
  - Emits result objects with:
    - `matchKey`
    - `matchedFields`
    - `matchCount`
    - `confidence`
    - `matchType`
    - `records`
    - `status`
- **Key expressions or variables used:**  
  Internal JS only; references include:
  - `json.recordId || json.id || json.transactionId`
  - `json.amount || json.total || json.value`
  - `json.date || json.transactionDate || json.timestamp`
- **Input and output connections:**  
  - Input: `Merge All Normalized Data2`
  - Output: `Check Match Quality`
- **Version-specific requirements:**  
  Code node version `2`.
- **Edge cases or potential failure types:**  
  - Invalid dates may normalize unexpectedly
  - Currency-formatted strings may parse incorrectly in unusual locales
  - The same record can participate in multiple matching groups, generating duplicate candidate outputs
  - Single unmatched items are emitted as separate `no_match` entries
- **Sub-workflow reference:**  
  None

#### Check Match Quality
- **Type and technical role:** `n8n-nodes-base.if`  
  Intended to route low-confidence deterministic matches toward AI fuzzy matching.
- **Configuration choices:**  
  Compares:
  - left value: `$('Deterministic Matching Logic').item.json.matchConfidence`
  - right value: `$('Workflow Configuration').first().json.fuzzyMatchThreshold`
  using `<`
- **Key expressions or variables used:**  
  - `={{ $('Deterministic Matching Logic').item.json.matchConfidence }}`
  - `={{ $('Workflow Configuration').first().json.fuzzyMatchThreshold }}`
- **Input and output connections:**  
  - Input: `Deterministic Matching Logic`
  - True output: `Merge Matched Results`
  - False output: `AI Fuzzy Matching Agent`
- **Version-specific requirements:**  
  IF node version `2.3`.
- **Edge cases or potential failure types:**  
  - **Important implementation issue:** the code node outputs `confidence`, not `matchConfidence`. This expression likely resolves to `undefined`.
  - Because of this mismatch, routing behavior may be incorrect or fail condition evaluation.
  - The branch naming is logically inverted relative to the intended business flow:
    - low confidence should go to AI
    - high confidence should go directly forward
    but the current connection sends one branch to merge and the other to AI without clear confirmation of which is true/false in documentation
  - AI node expects `unmatchedRecords`, but no previous node creates that field
- **Sub-workflow reference:**  
  None

---

## 2.6 Match Quality Evaluation and AI Escalation

**Overview:**  
This block is designed to send unresolved or weak deterministic matches to an AI agent backed by OpenAI, and to parse structured AI output. It then merges AI-assisted results with deterministic results.

**Nodes Involved:**  
- AI Fuzzy Matching Agent  
- OpenAI Chat Model  
- Structured Output Parser  
- Merge Matched Results

### Node Details

#### AI Fuzzy Matching Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI agent for fuzzy financial reconciliation.
- **Configuration choices:**  
  - Prompt text: `Records to match: {{ $json.unmatchedRecords }}`
  - Prompt type: defined directly in node
  - Output parser enabled
  - System message instructs the AI to:
    - compare descriptions semantically
    - allow amount tolerance within 1%
    - allow date proximity within 3 days
    - consider vendor name variations
    - return matched pairs with confidence scores
- **Key expressions or variables used:**  
  - `=Records to match: {{ $json.unmatchedRecords }}`
- **Input and output connections:**  
  - Main input: `Check Match Quality`
  - AI language model input: `OpenAI Chat Model`
  - AI output parser input: `Structured Output Parser`
  - Main output: `Merge Matched Results`
- **Version-specific requirements:**  
  Agent version `3`; requires compatible LangChain package support in n8n.
- **Edge cases or potential failure types:**  
  - Missing OpenAI credentials
  - `unmatchedRecords` field is not produced upstream, so the prompt may receive `undefined`
  - Token limits if large unmatched datasets are passed
  - Non-deterministic AI output despite structured parser constraints
  - Latency or provider rate limits
- **Sub-workflow reference:**  
  None

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the underlying LLM for the AI agent.
- **Configuration choices:**  
  - Model: `gpt-4.1-mini`
  - No additional options configured
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output to AI language model port of `AI Fuzzy Matching Agent`
- **Version-specific requirements:**  
  Version `1.3`; requires OpenAI credentials configured in n8n.
- **Edge cases or potential failure types:**  
  - Invalid API key
  - Model availability restrictions
  - Cost and token usage considerations
- **Sub-workflow reference:**  
  None

#### Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Forces AI output into a target JSON structure.
- **Configuration choices:**  
  Example schema expects:
  - `matchedPairs` array with `sourceRecord`, `targetRecord`, `confidenceScore`, `matchReason`
  - `unmatchedRecords` array
- **Key expressions or variables used:**  
  Static schema example only.
- **Input and output connections:**  
  - Output to AI output parser port of `AI Fuzzy Matching Agent`
- **Version-specific requirements:**  
  Version `1.3`.
- **Edge cases or potential failure types:**  
  - AI may still fail to conform perfectly
  - Example schema is not a strict JSON Schema definition, only an example structure
- **Sub-workflow reference:**  
  None

#### Merge Matched Results
- **Type and technical role:** `n8n-nodes-base.merge`  
  Combines deterministic and AI-derived results for confidence scoring.
- **Configuration choices:**  
  - Mode: `combineAll`
- **Input and output connections:**  
  - Input 1: `Check Match Quality`
  - Input 2: `AI Fuzzy Matching Agent`
  - Output: `Calculate Confidence Scores`
- **Version-specific requirements:**  
  Merge version `3.2`.
- **Edge cases or potential failure types:**  
  - If only one branch receives data, result behavior depends on Merge node semantics
  - Deterministic and AI outputs have different schemas and may not combine as intended
- **Sub-workflow reference:**  
  None

---

## 2.7 Confidence Scoring and Audit Enrichment

**Overview:**  
This block calculates a final confidence score and enriches each item with an audit trail, confidence label, and percentage value.

**Nodes Involved:**  
- Calculate Confidence Scores

### Node Details

#### Calculate Confidence Scores
- **Type and technical role:** `n8n-nodes-base.code`  
  Computes weighted confidence and audit metadata.
- **Configuration choices:**  
  For each item:
  - Reads `deterministicMatchScore` and `aiMatchScore`
  - Uses weights:
    - deterministic: `0.6`
    - AI: `0.4`
  - If AI score exists, computes weighted hybrid score
  - Otherwise uses deterministic score alone
  - Produces:
    - `finalConfidence`
    - `confidenceLevel`
    - `confidencePercentage`
    - `auditTrail`
- **Key expressions or variables used:**  
  Internal JS references:
  - `item.deterministicMatchScore || 0`
  - `item.aiMatchScore || 0`
- **Input and output connections:**  
  - Input: `Merge Matched Results`
  - Output: `Route by Confidence Threshold`
- **Version-specific requirements:**  
  Code node version `2`, mode `runOnceForEachItem`.
- **Edge cases or potential failure types:**  
  - **Important schema mismatch:** upstream nodes do not provide `deterministicMatchScore` or `aiMatchScore`
  - Deterministic block outputs `confidence`, not `deterministicMatchScore`
  - AI parser outputs `confidenceScore` inside `matchedPairs`, not `aiMatchScore`
  - As a result, `finalConfidence` may default to `0` for many or all items
- **Sub-workflow reference:**  
  None

---

## 2.8 Decision Routing

**Overview:**  
This block routes results according to the final confidence threshold. Confident matches are marked as auto-reconciled; others are flagged for manual review.

**Nodes Involved:**  
- Route by Confidence Threshold  
- Mark as Auto-Reconciled  
- Flag for Human Review  
- Merge All Results

### Node Details

#### Route by Confidence Threshold
- **Type and technical role:** `n8n-nodes-base.if`
- **Configuration choices:**  
  Checks whether:
  - `{{$json.finalConfidence}} >= {{$('Workflow Configuration').first().json.confidenceThreshold}}`
- **Key expressions or variables used:**  
  - `={{ $json.finalConfidence }}`
  - `={{ $('Workflow Configuration').first().json.confidenceThreshold }}`
- **Input and output connections:**  
  - Input: `Calculate Confidence Scores`
  - True output: `Mark as Auto-Reconciled`
  - False output: `Flag for Human Review`
- **Version-specific requirements:**  
  IF version `2.3`.
- **Edge cases or potential failure types:**  
  - If `finalConfidence` is undefined or 0 due to prior mapping issues, nearly everything will go to manual review
- **Sub-workflow reference:**  
  None

#### Mark as Auto-Reconciled
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**  
  Adds:
  - `reviewStatus = "auto_reconciled"`
  - `reconciledAt = {{$now.toISO()}}`
  Preserves all input fields.
- **Key expressions or variables used:**  
  - `={{ $now.toISO() }}`
- **Input and output connections:**  
  - Input: `Route by Confidence Threshold` true branch
  - Output: `Merge All Results`
- **Version-specific requirements:**  
  Set version `3.4`.
- **Edge cases or potential failure types:**  
  Minimal; only dependent on upstream data quality.
- **Sub-workflow reference:**  
  None

#### Flag for Human Review
- **Type and technical role:** `n8n-nodes-base.set`
- **Configuration choices:**  
  Adds:
  - `reviewStatus = "pending_human_review"`
  - `reviewReason = "Low confidence match - requires manual verification"`
  - `flaggedAt = {{$now.toISO()}}`
  Preserves all input fields.
- **Key expressions or variables used:**  
  - `={{ $now.toISO() }}`
- **Input and output connections:**  
  - Input: `Route by Confidence Threshold` false branch
  - Output: `Merge All Results`
- **Version-specific requirements:**  
  Set version `3.4`.
- **Edge cases or potential failure types:**  
  Minimal; likely to receive most items if prior confidence mapping is not corrected.
- **Sub-workflow reference:**  
  None

#### Merge All Results
- **Type and technical role:** `n8n-nodes-base.merge`
- **Configuration choices:**  
  Default merge settings are used.
- **Input and output connections:**  
  - Inputs: `Flag for Human Review`, `Mark as Auto-Reconciled`
  - Output: `Log to Reconciliation Report`
- **Version-specific requirements:**  
  Merge version `3.2`.
- **Edge cases or potential failure types:**  
  - Since parameters are empty, behavior depends on n8n defaults
  - Verify whether items are appended, paired, or combined in the installed version
- **Sub-workflow reference:**  
  None

---

## 2.9 Reporting and Notification

**Overview:**  
This block persists reconciliation outcomes in Google Sheets and sends a summary notification to Slack.

**Nodes Involved:**  
- Log to Reconciliation Report  
- Notify Finance Team

### Node Details

#### Log to Reconciliation Report
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Writes reconciliation output to Google Sheets.
- **Configuration choices:**  
  - Operation: `appendOrUpdate`
  - Sheet: `Reconciliation Report`
  - Mapping mode: auto-map input data
  - Document ID: placeholder value to be replaced
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Merge All Results`
  - Output: `Notify Finance Team`
- **Version-specific requirements:**  
  Google Sheets node version `4.7`.
- **Edge cases or potential failure types:**  
  - Missing Google credentials
  - Invalid spreadsheet ID
  - `appendOrUpdate` may require matching column behavior depending on data shape
  - Auto-mapping may create inconsistent writes if item fields vary
- **Sub-workflow reference:**  
  None

#### Notify Finance Team
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a reconciliation summary message.
- **Configuration choices:**  
  - Sends to selected channel
  - Channel ID/name placeholder must be replaced
  - Message body includes:
    - `totalRecords`
    - `autoReconciledCount`
    - `pendingReviewCount`
    - `reportUrl`
- **Key expressions or variables used:**  
  - `{{ $json.totalRecords }}`
  - `{{ $json.autoReconciledCount }}`
  - `{{ $json.pendingReviewCount }}`
  - `{{ $json.reportUrl }}`
- **Input and output connections:**  
  - Input: `Log to Reconciliation Report`
  - Output: none
- **Version-specific requirements:**  
  Slack node version `2.4`.
- **Edge cases or potential failure types:**  
  - Missing Slack credentials
  - Invalid channel
  - **Important schema mismatch:** upstream nodes do not generate `totalRecords`, `autoReconciledCount`, `pendingReviewCount`, or `reportUrl`
  - Notification may therefore send blank or undefined values unless an aggregation step is added
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook - Receive Financial Data | Webhook | Receives incoming financial reconciliation payloads via HTTP POST |  | Workflow Configuration | ## Data Input\nReceives financial data via webhook from multiple sources and Defines matching keys, thresholds, and reconciliation rules. |
| Workflow Configuration | Set | Injects thresholds and matching configuration into the payload | Webhook - Receive Financial Data | Check Data Source Type | ## Data Input\nReceives financial data via webhook from multiple sources and Defines matching keys, thresholds, and reconciliation rules. |
| Check Data Source Type | If | Routes bank statement data to a dedicated branch and all other data to invoice/ERP/CSV branches | Workflow Configuration | Normalize Bank Statement Schema; Normalize Invoice Schema; Normalize ERP Schema; Extract CSV Data | ## Source Routing\nDetects data type and routes to correct normalization flow.) |
| Extract CSV Data | Extract From File | Parses uploaded CSV content into records | Check Data Source Type | Normalize CSV Schema | ## Data Normalization\nConverts bank, invoice, ERP, and CSV data into a unified schema. |
| Normalize Bank Statement Schema | Set | Maps bank statement fields to unified schema | Check Data Source Type | Merge All Normalized Data | ## Data Normalization\nConverts bank, invoice, ERP, and CSV data into a unified schema. |
| Normalize Invoice Schema | Set | Maps invoice fields to unified schema | Check Data Source Type | Merge All Normalized Data1 | ## Data Normalization\nConverts bank, invoice, ERP, and CSV data into a unified schema. |
| Normalize ERP Schema | Set | Maps ERP fields to unified schema | Check Data Source Type | Merge All Normalized Data1 | ## Data Normalization\nConverts bank, invoice, ERP, and CSV data into a unified schema. |
| Normalize CSV Schema | Set | Maps CSV columns to unified schema | Extract CSV Data | Merge All Normalized Data | ## Data Normalization\nConverts bank, invoice, ERP, and CSV data into a unified schema. |
| Merge All Normalized Data | Merge | Combines bank and CSV normalized outputs | Normalize CSV Schema; Normalize Bank Statement Schema | Merge All Normalized Data2 | ## Data Merge\nCombines all normalized records for comparison. |
| Merge All Normalized Data1 | Merge | Combines invoice and ERP normalized outputs | Normalize Invoice Schema; Normalize ERP Schema | Merge All Normalized Data2 | ## Data Merge\nCombines all normalized records for comparison. |
| Merge All Normalized Data2 | Merge | Final consolidation of normalized records before deterministic matching | Merge All Normalized Data; Merge All Normalized Data1 | Deterministic Matching Logic | ## Data Merge\nCombines all normalized records for comparison. |
| Deterministic Matching Logic | Code | Performs exact and semi-exact record matching and assigns initial confidence | Merge All Normalized Data2 | Check Match Quality | ## Deterministic Matching\nMatches records using exact fields like ID, amount, and date. |
| Check Match Quality | If | Intended to decide whether AI fuzzy matching is needed | Deterministic Matching Logic | Merge Matched Results; AI Fuzzy Matching Agent | ## Deterministic Matching\nMatches records using exact fields like ID, amount, and date. |
| AI Fuzzy Matching Agent | LangChain Agent | Uses AI to evaluate low-confidence or unmatched records | Check Match Quality; OpenAI Chat Model; Structured Output Parser | Merge Matched Results | ## AI Fuzzy Matching\nUses AI to find near matches based on text, amount, and date. |
| OpenAI Chat Model | OpenAI Chat Model | Supplies GPT model to the AI agent |  | AI Fuzzy Matching Agent | ## AI Fuzzy Matching\nUses AI to find near matches based on text, amount, and date. |
| Structured Output Parser | Structured Output Parser | Forces AI output into a structured JSON-like result format |  | AI Fuzzy Matching Agent | ## AI Fuzzy Matching\nUses AI to find near matches based on text, amount, and date. |
| Merge Matched Results | Merge | Combines deterministic and AI-assisted matching outputs | Check Match Quality; AI Fuzzy Matching Agent | Calculate Confidence Scores | ## Confidence Scoring\nCombines matching results into final confidence score with audit trail. |
| Calculate Confidence Scores | Code | Calculates final confidence score and audit trail | Merge Matched Results | Route by Confidence Threshold | ## Confidence Scoring\nCombines matching results into final confidence score with audit trail. |
| Route by Confidence Threshold | If | Routes results based on final confidence threshold | Calculate Confidence Scores | Mark as Auto-Reconciled; Flag for Human Review | ## Decision Routing\nRoutes high-confidence matches or flags low-confidence for review. |
| Mark as Auto-Reconciled | Set | Marks confident results as automatically reconciled | Route by Confidence Threshold | Merge All Results | ## Decision Routing\nRoutes high-confidence matches or flags low-confidence for review. |
| Flag for Human Review | Set | Marks lower-confidence results for manual verification | Route by Confidence Threshold | Merge All Results | ## Decision Routing\nRoutes high-confidence matches or flags low-confidence for review. |
| Merge All Results | Merge | Recombines auto-reconciled and manual-review items | Flag for Human Review; Mark as Auto-Reconciled | Log to Reconciliation Report | ## Decision Routing\nRoutes high-confidence matches or flags low-confidence for review. |
| Log to Reconciliation Report | Google Sheets | Writes final reconciliation data to Google Sheets | Merge All Results | Notify Finance Team | ## Reporting\nLogs reconciliation results into Google Sheets and Sends summary of reconciliation results via Slack. |
| Notify Finance Team | Slack | Sends a reconciliation summary to Slack | Log to Reconciliation Report |  | ## Reporting\nLogs reconciliation results into Google Sheets and Sends summary of reconciliation results via Slack. |
| Sticky Note | Sticky Note | Canvas documentation note |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation note |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation note |  |  |  |
| Sticky Note3 | Sticky Note | Canvas documentation note |  |  |  |
| Sticky Note4 | Sticky Note | Canvas documentation note |  |  |  |
| Sticky Note5 | Sticky Note | Canvas documentation note |  |  |  |
| Sticky Note6 | Sticky Note | Canvas documentation note |  |  |  |
| Sticky Note7 | Sticky Note | Canvas documentation note |  |  |  |
| Sticky Note8 | Sticky Note | Canvas documentation note |  |  |  |
| Sticky Note9 | Sticky Note | Canvas documentation note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence for n8n.

## 1. Create the trigger node
1. Add a **Webhook** node.
2. Name it **Webhook - Receive Financial Data**.
3. Set:
   - **HTTP Method**: `POST`
   - **Path**: `financial-data-reconciliation`
   - **Response Mode**: `Last Node`
4. Keep other options default unless your deployment requires authentication or custom response handling.

## 2. Add runtime configuration
5. Add a **Set** node connected from the webhook.
6. Name it **Workflow Configuration**.
7. Enable **Include Other Input Fields**.
8. Add fields:
   - `confidenceThreshold` as Number = `0.85`
   - `matchingKeys` as Array = `["transactionId", "invoiceNumber", "amount", "date"]`
   - `fuzzyMatchThreshold` as Number = `0.7`

## 3. Add source routing
9. Add an **If** node after the configuration node.
10. Name it **Check Data Source Type**.
11. Configure the condition:
   - Left value: `{{$json.sourceType}}`
   - Operation: `equals`
   - Right value: `bank_statement`
12. Connect:
   - **True** output to bank normalization
   - **False** output to invoice normalization, ERP normalization, and CSV extraction

## 4. Build normalization nodes
13. Add a **Set** node named **Normalize Bank Statement Schema**.
14. Enable **Include Other Input Fields**.
15. Add:
   - `recordId` = `{{$json.transaction_id}}`
   - `amount` = `{{$json.transaction_amount}}`
   - `date` = `{{$json.transaction_date}}`
   - `description` = `{{$json.transaction_description}}`
   - `sourceType` = `bank_statement`
   - `normalizedAt` = `{{$now.toISO()}}`

16. Add a **Set** node named **Normalize Invoice Schema**.
17. Enable **Include Other Input Fields**.
18. Add:
   - `recordId` = `{{$json.invoice_number}}`
   - `amount` = `{{$json.invoice_total}}`
   - `date` = `{{$json.invoice_date}}`
   - `description` = `{{$json.invoice_description}}`
   - `sourceType` = `invoice`
   - `normalizedAt` = `{{$now.toISO()}}`

19. Add a **Set** node named **Normalize ERP Schema**.
20. Enable **Include Other Input Fields**.
21. Add:
   - `recordId` = `{{$json.erp_transaction_id}}`
   - `amount` = `{{$json.erp_amount}}`
   - `date` = `{{$json.erp_date}}`
   - `description` = `{{$json.erp_description}}`
   - `sourceType` = `erp`
   - `normalizedAt` = `{{$now.toISO()}}`

22. Add an **Extract From File** node named **Extract CSV Data**.
23. Leave options default unless you need to specify delimiter, encoding, or binary property.
24. Connect the false branch of **Check Data Source Type** to **Extract CSV Data**.

25. Add a **Set** node named **Normalize CSV Schema**.
26. Enable **Include Other Input Fields**.
27. Add:
   - `recordId` = `{{$json.id}}`
   - `amount` = `{{$json.amount}}`
   - `date` = `{{$json.date}}`
   - `description` = `{{$json.description}}`
   - `sourceType` = `csv_upload`
   - `normalizedAt` = `{{$now.toISO()}}`

## 5. Merge normalized data
28. Add a **Merge** node named **Merge All Normalized Data**.
29. Set:
   - **Mode**: `Combine`
   - **Combine By**: `Combine All`
30. Connect:
   - **Normalize CSV Schema** to input 1
   - **Normalize Bank Statement Schema** to input 2

31. Add a second **Merge** node named **Merge All Normalized Data1**.
32. Set:
   - **Mode**: `Combine`
   - **Combine By**: `Combine All`
33. Connect:
   - **Normalize Invoice Schema** to input 1
   - **Normalize ERP Schema** to input 2

34. Add a third **Merge** node named **Merge All Normalized Data2**.
35. Set:
   - **Mode**: `Combine`
   - **Combine By**: `Combine All`
36. Connect:
   - **Merge All Normalized Data** to input 1
   - **Merge All Normalized Data1** to input 2

## 6. Add deterministic matching
37. Add a **Code** node named **Deterministic Matching Logic**.
38. Connect it after **Merge All Normalized Data2**.
39. Paste the deterministic matching logic that:
   - reads all input items
   - normalizes record ID, amount, and date
   - builds multiple matching keys
   - groups records
   - emits matched and unmatched results
40. Make sure it returns one JSON item per match group.

## 7. Add match quality routing
41. Add an **If** node named **Check Match Quality** after the deterministic code node.
42. Configure the intended comparison between deterministic confidence and the fuzzy threshold.
43. For a correct implementation, use:
   - Left value: `{{$json.confidence}}`
   - Operation: `smaller than`
   - Right value: `{{$('Workflow Configuration').first().json.fuzzyMatchThreshold}}`
44. If you want low-confidence records to go to AI:
   - Connect the **true** output to **AI Fuzzy Matching Agent**
   - Connect the **false** output to **Merge Matched Results**
45. Note: the provided JSON currently references `matchConfidence`, which does not exist. Replace it with `confidence`.

## 8. Add AI fuzzy matching
46. Add an **AI Agent** node named **AI Fuzzy Matching Agent**.
47. Set prompt mode to define the prompt directly.
48. In the user prompt, pass the records you want analyzed. For a working design, ensure an upstream node creates a field like `unmatchedRecords`.
49. Use a prompt such as:
   - `Records to match: {{$json.unmatchedRecords}}`
50. Add a system message instructing the model to:
   - compare descriptions semantically
   - allow amount tolerance of 1%
   - allow date window of 3 days
   - consider vendor name variations
   - return structured results

51. Add an **OpenAI Chat Model** node.
52. Name it **OpenAI Chat Model**.
53. Choose model `gpt-4.1-mini`.
54. Attach valid **OpenAI credentials**.

55. Add a **Structured Output Parser** node.
56. Name it **Structured Output Parser**.
57. Define a structure with:
   - `matchedPairs`
   - `unmatchedRecords`
58. Connect:
   - **OpenAI Chat Model** to the AI language model input of the agent
   - **Structured Output Parser** to the parser input of the agent

## 9. Merge deterministic and AI results
59. Add a **Merge** node named **Merge Matched Results**.
60. Set mode to `Combine All`.
61. Connect:
   - direct deterministic branch from **Check Match Quality**
   - AI branch from **AI Fuzzy Matching Agent**

## 10. Add final confidence scoring
62. Add a **Code** node named **Calculate Confidence Scores**.
63. Set it to **Run Once for Each Item**.
64. Use logic that combines deterministic and AI scores with weights:
   - deterministic: `0.6`
   - AI: `0.4`
65. The provided JSON expects:
   - `deterministicMatchScore`
   - `aiMatchScore`
66. To make the workflow work as intended, map upstream data accordingly, for example:
   - deterministic branch should expose `deterministicMatchScore = confidence`
   - AI branch should expose `aiMatchScore = confidenceScore`
67. Ensure the node outputs:
   - `finalConfidence`
   - `confidenceLevel`
   - `confidencePercentage`
   - `auditTrail`

## 11. Route by reconciliation confidence
68. Add an **If** node named **Route by Confidence Threshold**.
69. Configure:
   - Left value: `{{$json.finalConfidence}}`
   - Operation: `greater than or equal`
   - Right value: `{{$('Workflow Configuration').first().json.confidenceThreshold}}`
70. Connect:
   - true branch to **Mark as Auto-Reconciled**
   - false branch to **Flag for Human Review**

## 12. Mark auto-reconciled records
71. Add a **Set** node named **Mark as Auto-Reconciled**.
72. Enable **Include Other Input Fields**.
73. Add:
   - `reviewStatus` = `auto_reconciled`
   - `reconciledAt` = `{{$now.toISO()}}`

## 13. Mark records for manual review
74. Add a **Set** node named **Flag for Human Review**.
75. Enable **Include Other Input Fields**.
76. Add:
   - `reviewStatus` = `pending_human_review`
   - `reviewReason` = `Low confidence match - requires manual verification`
   - `flaggedAt` = `{{$now.toISO()}}`

## 14. Merge final result branches
77. Add a **Merge** node named **Merge All Results**.
78. Connect:
   - **Flag for Human Review**
   - **Mark as Auto-Reconciled**
79. Verify the merge mode in your n8n version so that it appends or combines results the way you expect.

## 15. Add Google Sheets logging
80. Add a **Google Sheets** node named **Log to Reconciliation Report**.
81. Set:
   - Operation: `Append or Update`
   - Document ID: your Google Sheets document ID
   - Sheet Name: `Reconciliation Report`
   - Column mapping: auto-map input data, or define explicit mapping if you want stable schema
82. Attach valid **Google Sheets credentials**.

## 16. Add Slack notification
83. Add a **Slack** node named **Notify Finance Team**.
84. Set it to send a message to a target channel.
85. Attach valid **Slack credentials**.
86. Select the target channel ID or channel name.
87. Configure the message body.
88. For a working setup, add a summary/aggregation node before Slack, because the provided workflow does not compute:
   - `totalRecords`
   - `autoReconciledCount`
   - `pendingReviewCount`
   - `reportUrl`
89. Without that aggregation step, the Slack message will contain undefined values.

## 17. Connect the reporting chain
90. Connect **Merge All Results** to **Log to Reconciliation Report**.
91. Connect **Log to Reconciliation Report** to **Notify Finance Team**.

## 18. Configure credentials
92. Configure **OpenAI** credentials for the chat model node.
93. Configure **Google Sheets OAuth2** or service-account-style access supported by your n8n environment.
94. Configure **Slack** credentials for the Slack node.
95. Test webhook ingestion with representative payloads for:
   - bank statement JSON
   - invoice JSON
   - ERP JSON
   - CSV file upload

## 19. Recommended corrections before production use
96. Replace `matchConfidence` with `confidence` in **Check Match Quality**.
97. Ensure low-confidence items route to AI, not the reverse.
98. Add preprocessing to generate `unmatchedRecords` for the AI node.
99. Normalize AI output into top-level fields expected by **Calculate Confidence Scores**.
100. Add a summary aggregation node before Slack.
101. Consider replacing current source routing with multi-branch explicit checks:
   - `bank_statement`
   - `invoice`
   - `erp`
   - `csv_upload`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automates reconciliation across bank statements, invoices, ERP systems, and CSV uploads. It standardizes all data into a unified format and performs deterministic matching using fields like ID, amount, and date. For unmatched records, AI fuzzy matching identifies potential matches using descriptions, amount tolerance, and date proximity. A final confidence score is calculated, auto-reconciling high-confidence matches while flagging others for review. | Canvas overview note |
| Configure webhook to receive financial data; set matching keys and confidence thresholds; connect OpenAI for fuzzy matching; connect Google Sheets for reporting; connect Slack for notifications; ensure input data follows expected formats; test with sample data before activating. | Canvas setup note |
| The workflow contains several implementation mismatches that should be corrected before production use: `matchConfidence` vs `confidence`, missing `unmatchedRecords`, missing `deterministicMatchScore` and `aiMatchScore`, and missing Slack summary fields. | Operational warning |
| The title is `No AI suggestion available`, which is non-descriptive compared to the actual business function. Renaming it to something like “Financial Data Reconciliation with AI Fuzzy Matching” would improve maintainability. | Naming recommendation |

