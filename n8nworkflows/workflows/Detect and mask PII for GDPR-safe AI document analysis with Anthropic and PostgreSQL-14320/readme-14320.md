Detect and mask PII for GDPR-safe AI document analysis with Anthropic and PostgreSQL

https://n8nworkflows.xyz/workflows/detect-and-mask-pii-for-gdpr-safe-ai-document-analysis-with-anthropic-and-postgresql-14320


# Detect and mask PII for GDPR-safe AI document analysis with Anthropic and PostgreSQL

# 1. Workflow Overview

This workflow implements a GDPR-oriented document-processing pipeline in n8n. It receives an uploaded PDF through a webhook, extracts text, detects personally identifiable information (PII) using a mix of regex-based code nodes and an AI agent, replaces detected PII with tokens, stores the originals in PostgreSQL, processes only the masked text with Anthropic, optionally restores approved values, and writes an audit log.

Its primary use cases are:
- Privacy-safe AI analysis of uploaded business documents
- Controlled handling of PII before sending content to an LLM
- Vault-based tokenization and later selective re-injection
- Compliance traceability through audit storage

## 1.1 Document Intake & Configuration
The workflow starts with a webhook that accepts a POST request containing a raw file payload. A Set node initializes runtime metadata such as a generated document ID, confidence threshold, and PostgreSQL table names.

## 1.2 Text Extraction
The uploaded file is passed to a PDF text extraction node, which outputs extracted text while preserving source content.

## 1.3 Multi-Detector PII Detection
The extracted text is analyzed by four parallel detectors:
- Email regex detector
- Phone regex detector
- ID number regex detector
- AI-based address detector using Anthropic with structured output parsing

## 1.4 PII Aggregation & Conflict Resolution
All detector outputs are merged, consolidated, deduplicated, and overlap-resolved into a single PII map.

## 1.5 Tokenization & Vault Storage
Detected values are tokenized, original values are prepared for secure storage in PostgreSQL, and masked text is generated.

## 1.6 Masking Validation & AI Processing
A masking-validation branch decides whether the masked text is safe to send to the LLM. If masking succeeds, Anthropic processes the masked text and returns structured output.

## 1.7 Re-Injection & Retrieval
The workflow identifies fields where original values may be restored, queries the vault, and attempts token replacement according to policy.

## 1.8 Audit Logging
Processing metadata is stored in PostgreSQL for compliance and traceability.

## 1.9 Error Handling & Alerts
If masking fails, AI processing is blocked and an HTTP alert is sent to an external notification endpoint.

---

# 2. Block-by-Block Analysis

## 2.1 Document Intake & Configuration

### Overview
This block receives the uploaded document and creates baseline workflow settings used later across detection, storage, and audit phases. It is the workflow entry point.

### Nodes Involved
- Document Upload Webhook
- Workflow Configuration

### Node Details

#### Document Upload Webhook
- **Type and role:** `n8n-nodes-base.webhook`; workflow entry point for HTTP uploads.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `gdpr-document-upload`
  - Response mode: `lastNode`
  - Raw body enabled, which is important for binary file intake
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: none
  - Output: `Workflow Configuration`
- **Version-specific requirements:** Type version `2.1`
- **Edge cases / failures:**
  - Webhook path conflict with another workflow
  - Incorrect request format for binary/PDF upload
  - Timeout if downstream execution is long, since response mode waits for the last node
- **Sub-workflow reference:** None

#### Workflow Configuration
- **Type and role:** `n8n-nodes-base.set`; injects workflow runtime configuration values.
- **Configuration choices:**
  - Adds:
    - `documentId = {{$now.toISO()}}`
    - `confidenceThreshold = 0.8`
    - `vaultTable = pii_vault`
    - `auditTable = pii_audit_log`
  - Keeps existing fields with `includeOtherFields: true`
- **Key expressions or variables used:**
  - `$now.toISO()`
- **Input and output connections:**
  - Input: `Document Upload Webhook`
  - Output: `Extract Text`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases / failures:**
  - ISO timestamp as document ID may not be unique enough under concurrent execution
  - Later nodes must consistently reference `documentId`
- **Sub-workflow reference:** None

---

## 2.2 Text Extraction

### Overview
This block extracts text from the uploaded PDF so the PII detectors and later AI processing can work on plain text.

### Nodes Involved
- Extract Text

### Node Details

#### Extract Text
- **Type and role:** `n8n-nodes-base.extractFromFile`; converts PDF binary input into text.
- **Configuration choices:**
  - Operation: `pdf`
  - `keepSource: both`, so extracted content and source are retained
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: `Workflow Configuration`
  - Outputs in parallel to:
    - `Email Detector`
    - `Phone Detector`
    - `ID Number Detector`
    - `Address Detector AI`
- **Version-specific requirements:** Type version `1.1`
- **Edge cases / failures:**
  - Non-PDF uploads may fail
  - Scanned PDFs without embedded text may yield poor or empty extraction
  - Very large PDFs may cause memory/time issues
- **Sub-workflow reference:** None

---

## 2.3 Multi-Detector PII Detection

### Overview
This block detects different PII classes in parallel. Three detectors are regex-based code nodes, and one uses an Anthropic-powered AI agent with a structured output parser for addresses.

### Nodes Involved
- Email Detector
- Phone Detector
- ID Number Detector
- Address Detector AI
- Anthropic Chat Model
- Address Output Parser

### Node Details

#### Email Detector
- **Type and role:** `n8n-nodes-base.code`; regex-based email extraction.
- **Configuration choices:**
  - Reads `item.json.text`
  - Uses a standard email regex
  - Emits `detections`, `detector`, `original_text`
- **Key expressions or variables used:**
  - Accesses `$input.all()`
- **Input and output connections:**
  - Input: `Extract Text`
  - Output: `Merge PII Detections` input 0
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**
  - False positives for malformed emails
  - Missed internationalized email formats
  - Output field naming differs from some other detectors
- **Sub-workflow reference:** None

#### Phone Detector
- **Type and role:** `n8n-nodes-base.code`; regex-based phone detection with several patterns.
- **Configuration choices:**
  - Uses international, North American, and general separated-number patterns
  - Deduplicates using a `Set`
  - Outputs `detected_pii`, `original_text`, `detection_type`, `count`
- **Key expressions or variables used:**
  - Accesses `$input.all()`
- **Input and output connections:**
  - Input: `Extract Text`
  - Output: `Merge PII Detections` input 1
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**
  - False positives on invoice numbers or numeric codes
  - Output schema is inconsistent with `detections` used elsewhere
  - If no phone is found, returns one item with empty array
- **Sub-workflow reference:** None

#### ID Number Detector
- **Type and role:** `n8n-nodes-base.code`; regex-based detection of SSN, PAN, driver's license, bank account numbers, and IBAN.
- **Configuration choices:**
  - Reads `extractedText` or `text`
  - Outputs `detections`, `originalText`, `detectionCount`
  - Includes subtype-specific metadata
- **Key expressions or variables used:**
  - Accesses `$input.all()`
- **Input and output connections:**
  - Input: `Extract Text`
  - Output: `Merge PII Detections` input 2
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**
  - Bank-account regex can easily overmatch ordinary long numbers
  - Driver's license formats are region-specific and incomplete
  - One SSN validation check appears incorrect: it compares against `+1234567890`
- **Sub-workflow reference:** None

#### Address Detector AI
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; AI-based address extraction.
- **Configuration choices:**
  - Prompt text: `{{$json.text}}`
  - System message instructs model to detect physical addresses and return positions plus confidence
  - `promptType: define`
  - Structured output enabled
- **Key expressions or variables used:**
  - `{{$json.text}}`
- **Input and output connections:**
  - Main input: `Extract Text`
  - AI language model input: `Anthropic Chat Model`
  - AI output parser input: `Address Output Parser`
  - Main output: `Merge PII Detections` input 3
- **Version-specific requirements:** Type version `3`
- **Edge cases / failures:**
  - Model may infer addresses not explicitly present
  - Position offsets may be unreliable unless carefully prompted
  - Empty text leads to poor/empty output
- **Sub-workflow reference:** None

#### Anthropic Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; provides the language model for address detection.
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output to `Address Detector AI` as AI language model
- **Version-specific requirements:** Type version `1.3`
- **Edge cases / failures:**
  - Missing Anthropic credentials
  - Model availability by n8n version/region/account
  - API quota or rate limiting
- **Sub-workflow reference:** None

#### Address Output Parser
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces a JSON schema for address output.
- **Configuration choices:**
  - Manual schema requiring:
    - `addresses` array
    - Each address includes `value`, `type`, `start_pos`, `end_pos`, `confidence`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output parser connected to `Address Detector AI`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases / failures:**
  - Parser failure if model response does not match schema
  - The agent output must later be normalized to match the merge/consolidation expectations
- **Sub-workflow reference:** None

---

## 2.4 PII Aggregation & Conflict Resolution

### Overview
This block merges all detector outputs, then resolves duplication and overlapping matches to create a final PII map for tokenization.

### Nodes Involved
- Merge PII Detections
- PII Consolidation & Conflict Resolver

### Node Details

#### Merge PII Detections
- **Type and role:** `n8n-nodes-base.merge`; collects four detector streams.
- **Configuration choices:**
  - `numberInputs = 4`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Inputs:
    - `Email Detector`
    - `Phone Detector`
    - `ID Number Detector`
    - `Address Detector AI`
  - Output: `PII Consolidation & Conflict Resolver`
- **Version-specific requirements:** Type version `3.2`
- **Edge cases / failures:**
  - Different node output schemas may reduce usable merged data
  - Missing branch output can affect merge behavior depending on runtime data
- **Sub-workflow reference:** None

#### PII Consolidation & Conflict Resolver
- **Type and role:** `n8n-nodes-base.code`; consolidates detections, resolves overlaps, deduplicates.
- **Configuration choices:**
  - Collects `item.json.detections`
  - Sorts detections
  - Resolves overlaps by confidence, then span length
  - Deduplicates by `type|value|start|end`
  - Produces `piiMap`, `totalDetections`, `originalText`, `timestamp`
- **Key expressions or variables used:**
  - `$input.all()`
  - `$input.first().json.extractedText || $input.first().json.text || ''`
- **Input and output connections:**
  - Input: `Merge PII Detections`
  - Output: `Tokenization & Vault Storage`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**
  - Code expects `start` and `end`, but earlier detectors emit `start_pos` and `end_pos`
  - Code expects `detections`, but `Phone Detector` emits `detected_pii`
  - Address AI parser schema uses `addresses`, not `detections`
  - As written, this mismatch can lead to empty or broken consolidation
- **Sub-workflow reference:** None

---

## 2.5 Tokenization & Vault Storage

### Overview
This block converts detected PII into tokens, prepares vault records, writes them to PostgreSQL, and attempts to create masked text.

### Nodes Involved
- Tokenization & Vault Storage
- Store Tokens in Vault
- Generate Masked Text

### Node Details

#### Tokenization & Vault Storage
- **Type and role:** `n8n-nodes-base.code`; creates token mappings and vault records.
- **Configuration choices:**
  - Reads `item.json.consolidatedPII`
  - Generates tokens like `<<TYPE_HASH>>`
  - Builds `vaultRecords`, `tokenMap`, `maskedText`, `documentId`
- **Key expressions or variables used:**
  - `$input.all()`
- **Input and output connections:**
  - Input: `PII Consolidation & Conflict Resolver`
  - Output: `Store Tokens in Vault`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**
  - Expects `consolidatedPII`, but previous node outputs `piiMap`
  - If no data matches expected field names, vault records remain empty
  - Random 4-character token suffix is collision-prone at scale
  - Token replacement is value-based, so repeated identical values in different contexts all get same token
- **Sub-workflow reference:** None

#### Store Tokens in Vault
- **Type and role:** `n8n-nodes-base.postgres`; inserts tokenized PII records into PostgreSQL.
- **Configuration choices:**
  - Table name from `{{$('Workflow Configuration').first().json.vaultTable}}`
  - Schema: `public`
  - Defined column mapping:
    - `type`
    - `token`
    - `created_at`
    - `document_id`
    - `original_value`
- **Key expressions or variables used:**
  - `{{$('Workflow Configuration').first().json.vaultTable}}`
  - Column expressions from current item JSON
- **Input and output connections:**
  - Input: `Tokenization & Vault Storage`
  - Output: `Generate Masked Text`
- **Version-specific requirements:** Type version `2.6`
- **Edge cases / failures:**
  - The `token` column is set to literal `YOUR_CREDENTIAL_HERE`, which is clearly incorrect
  - Input is a single item containing `vaultRecords` array, but Postgres insert typically expects itemized rows unless transformed
  - Missing database credentials or table schema mismatch
- **Sub-workflow reference:** None

#### Generate Masked Text
- **Type and role:** `n8n-nodes-base.code`; attempts final masking validation/reconstruction using vault rows.
- **Configuration choices:**
  - Reads original text from `$('Extract Text').first().json`
  - Reads token data from `$('Store Tokens in Vault').all()`
  - Replaces `original_value` with `token`
  - Emits `masked_text`, `token_count`, `masking_success`, `replacements`
- **Key expressions or variables used:**
  - `$('Extract Text').first().json`
  - `$('Store Tokens in Vault').all()`
- **Input and output connections:**
  - Input: `Store Tokens in Vault`
  - Output: `Masking Success Check`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**
  - Depends on `Store Tokens in Vault` returning inserted token rows with the same fields
  - If Postgres node does not return inserted values as expected, masking fails
  - Uses `pii_type`, but vault insert uses `type`
- **Sub-workflow reference:** None

---

## 2.6 Masking Validation & AI Processing

### Overview
This block ensures AI only receives masked content. If validation passes, the masked text is sent to Anthropic for structured document analysis.

### Nodes Involved
- Masking Success Check
- AI Processing (Masked Data)
- AI Processing Model
- AI Output Parser

### Node Details

#### Masking Success Check
- **Type and role:** `n8n-nodes-base.if`; gateway that blocks or allows AI processing.
- **Configuration choices:**
  - Checks whether `{{$('Generate Masked Text').item.json.masking_success}}` equals `true`
- **Key expressions or variables used:**
  - `{{$('Generate Masked Text').item.json.masking_success}}`
- **Input and output connections:**
  - Input: `Generate Masked Text`
  - True output: `AI Processing (Masked Data)`
  - False output: `Block AI Processing`
- **Version-specific requirements:** Type version `2.3`
- **Edge cases / failures:**
  - Boolean/string comparison can behave unexpectedly if output is not a true boolean
- **Sub-workflow reference:** None

#### AI Processing (Masked Data)
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`; processes masked document text safely.
- **Configuration choices:**
  - Text input: `{{$json.masked_text}}`
  - System message instructs model to preserve tokens exactly
  - Structured output enabled
- **Key expressions or variables used:**
  - `{{$json.masked_text}}`
- **Input and output connections:**
  - Main input: `Masking Success Check`
  - AI language model input: `AI Processing Model`
  - AI output parser input: `AI Output Parser`
  - Main output: `Re-Injection Controller`
- **Version-specific requirements:** Type version `3`
- **Edge cases / failures:**
  - If masked text is empty, output quality is poor
  - Token format expected here is `<<EMAIL_7F3A>>`, but later re-injection logic expects different token patterns
- **Sub-workflow reference:** None

#### AI Processing Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; LLM backend for masked document analysis.
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output to `AI Processing (Masked Data)` as AI language model
- **Version-specific requirements:** Type version `1.3`
- **Edge cases / failures:**
  - Missing credentials
  - Unsupported model in current environment
  - Rate limiting or token quota exhaustion
- **Sub-workflow reference:** None

#### AI Output Parser
- **Type and role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces structured AI output.
- **Configuration choices:**
  - Manual schema includes:
    - `documentType`
    - `summary`
    - optional arrays for `keyEntities`, `dates`, `amounts`
    - `processedData`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output parser connected to `AI Processing (Masked Data)`
- **Version-specific requirements:** Type version `1.3`
- **Edge cases / failures:**
  - Parser mismatch if AI returns invalid schema
  - Numeric coercion issues for amount fields
- **Sub-workflow reference:** None

---

## 2.7 Re-Injection & Retrieval

### Overview
This block decides which output fields are eligible for PII restoration, queries the vault, and performs conditional replacement.

### Nodes Involved
- Re-Injection Controller
- Retrieve Original Values
- Restore Original PII

### Node Details

#### Re-Injection Controller
- **Type and role:** `n8n-nodes-base.code`; scans AI output for tokens and prepares a retrieval map.
- **Configuration choices:**
  - Reads `aiOutput` or current JSON
  - Looks at `fieldPermissions`
  - Searches for tokens matching `TOKEN_([A-Z_]+)_([a-f0-9-]+)`
  - Builds `tokensToRetrieve`, `reinjectionMap`, `fieldsToRestore`
- **Key expressions or variables used:**
  - `$input.all()`
  - `$execution.id`
- **Input and output connections:**
  - Input: `AI Processing (Masked Data)`
  - Output: `Retrieve Original Values`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**
  - Token regex does not match the actual tokenization format `<<TYPE_HASH>>`
  - If no `fieldPermissions` are supplied, nothing is restored
  - Complex nested structures may still work, but only string fields are checked
- **Sub-workflow reference:** None

#### Retrieve Original Values
- **Type and role:** `n8n-nodes-base.postgres`; fetches original vault records for requested tokens.
- **Configuration choices:**
  - Operation: `select`
  - Table from `{{$('Workflow Configuration').first().json.vaultTable}}`
  - `where` clause on column `token`
  - `returnAll: true`
- **Key expressions or variables used:**
  - `{{$('Workflow Configuration').first().json.vaultTable}}`
  - `{{$('Re-Injection Controller').item.json.token}}`
- **Input and output connections:**
  - Input: `Re-Injection Controller`
  - Output: `Restore Original PII`
- **Version-specific requirements:** Type version `2.6`
- **Edge cases / failures:**
  - Expression references `item.json.token`, but controller outputs `tokensToRetrieve` array, not `token`
  - Select query likely returns nothing without item splitting
  - DB auth/schema issues
- **Sub-workflow reference:** None

#### Restore Original PII
- **Type and role:** `n8n-nodes-base.code`; recursively replaces permitted tokens with original values.
- **Configuration choices:**
  - Reads current item as `aiOutput`
  - Loads vault rows via `$('Retrieve Original Values').all()`
  - Restores only if `allowed_for_reinjection` is truthy
  - Emits `finalOutput`, `restoredFields`, `restorationTimestamp`
- **Key expressions or variables used:**
  - `$input.first().json`
  - `$('Retrieve Original Values').all()`
- **Input and output connections:**
  - Input: `Retrieve Original Values`
  - Output: `Store Audit Log`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**
  - Token regex expects `[TOKEN_XXXX]`, which does not match earlier token format
  - Uses `pii_type` and `allowed_for_reinjection`, fields not inserted by vault storage node
  - Likely preserves tokens unchanged in current form
- **Sub-workflow reference:** None

---

## 2.8 Audit Logging

### Overview
This block writes compliance-related metadata to PostgreSQL after restoration.

### Nodes Involved
- Store Audit Log

### Node Details

#### Store Audit Log
- **Type and role:** `n8n-nodes-base.postgres`; stores traceability metadata.
- **Configuration choices:**
  - Table from `{{$('Workflow Configuration').first().json.auditTable}}`
  - Schema: `public`
  - Columns mapped:
    - `actor = system`
    - `timestamp`
    - `document_id`
    - `token_count`
    - `pii_types_detected`
    - `ai_access_confirmed = true`
    - `re_injection_events`
- **Key expressions or variables used:**
  - `{{$('Workflow Configuration').first().json.auditTable}}`
  - Several expressions from current JSON
- **Input and output connections:**
  - Input: `Restore Original PII`
  - Output: none
- **Version-specific requirements:** Type version `2.6`
- **Edge cases / failures:**
  - Upstream node does not output several mapped fields, so inserts may be null
  - Missing audit table or incompatible schema
- **Sub-workflow reference:** None

---

## 2.9 Error Handling & Alerts

### Overview
If masking fails, the workflow blocks LLM processing and sends an external alert so the document can be manually reviewed.

### Nodes Involved
- Block AI Processing
- Send Alert Notification

### Node Details

#### Block AI Processing
- **Type and role:** `n8n-nodes-base.set`; creates a blocked/compliance-failure status payload.
- **Configuration choices:**
  - Sets:
    - `error = "Masking failed - AI processing blocked"`
    - `status = "BLOCKED"`
    - `requires_manual_review = true`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: false branch of `Masking Success Check`
  - Output: `Send Alert Notification`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases / failures:**
  - Does not include all fields expected by the alert node
- **Sub-workflow reference:** None

#### Send Alert Notification
- **Type and role:** `n8n-nodes-base.httpRequest`; sends compliance failure notification externally.
- **Configuration choices:**
  - Method: `POST`
  - URL placeholder must be replaced
  - Sends body with:
    - `error_details = {{$json.error_details}}`
    - `document_id = {{$json.document_id}}`
    - `timestamp = {{$now.toISO()}}`
- **Key expressions or variables used:**
  - `{{$json.error_details}}`
  - `{{$json.document_id}}`
  - `{{$now.toISO()}}`
- **Input and output connections:**
  - Input: `Block AI Processing`
  - Output: none
- **Version-specific requirements:** Type version `4.3`
- **Edge cases / failures:**
  - URL is still a placeholder
  - Upstream node emits `error`, not `error_details`
  - `document_id` may be missing unless explicitly propagated
- **Sub-workflow reference:** None

---

## 2.10 Sticky Notes / Documentation Nodes

These are non-executable annotation nodes but must still be documented.

### Nodes Involved
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7
- Sticky Note8
- Sticky Note9
- Sticky Note10
- Sticky Note11
- Sticky Note12
- Sticky Note13

### Overview
These nodes visually document the functional zones of the workflow and provide setup guidance.

### Node Details
Each sticky note is of type `n8n-nodes-base.stickyNote`, version `1`, with no runtime effect. Their content is reflected in the summary table below and should be preserved when rebuilding the workflow for maintainability.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Document Upload Webhook | n8n-nodes-base.webhook | Receives uploaded document via HTTP POST |  | Workflow Configuration | ## Document Intake & Configuration\nReceives documents via webhook and initializes workflow settings such as document ID, confidence thresholds, and database table configuration. |
| Workflow Configuration | n8n-nodes-base.set | Initializes workflow variables and table names | Document Upload Webhook | Extract Text | ## Document Intake & Configuration\nReceives documents via webhook and initializes workflow settings such as document ID, confidence thresholds, and database table configuration. |
| Extract Text | n8n-nodes-base.extractFromFile | Extracts text from uploaded PDF | Workflow Configuration | Email Detector; Phone Detector; ID Number Detector; Address Detector AI | ## Text Extraction\nExtracts raw text from uploaded PDF or document files while preserving original content for downstream processing. |
| Email Detector | n8n-nodes-base.code | Detects email addresses with regex | Extract Text | Merge PII Detections | ## Multi-Detector PII Detection\nIdentifies sensitive data including emails, phone numbers, IDs, and addresses using regex-based logic and AI-powered detection. |
| Phone Detector | n8n-nodes-base.code | Detects phone numbers with regex patterns | Extract Text | Merge PII Detections | ## Multi-Detector PII Detection\nIdentifies sensitive data including emails, phone numbers, IDs, and addresses using regex-based logic and AI-powered detection. |
| ID Number Detector | n8n-nodes-base.code | Detects ID-like numbers and banking identifiers | Extract Text | Merge PII Detections | ## Multi-Detector PII Detection\nIdentifies sensitive data including emails, phone numbers, IDs, and addresses using regex-based logic and AI-powered detection. |
| Address Detector AI | @n8n/n8n-nodes-langchain.agent | Detects addresses with LLM assistance | Extract Text; Anthropic Chat Model; Address Output Parser | Merge PII Detections | ## Address Detection (AI-Powered)\nUses an AI model to identify physical addresses in text and returns structured results with location, position, and confidence for each detected address. |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM backend for address detection |  | Address Detector AI | ## Address Detection (AI-Powered)\nUses an AI model to identify physical addresses in text and returns structured results with location, position, and confidence for each detected address. |
| Address Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parser for address detector output |  | Address Detector AI | ## Address Detection (AI-Powered)\nUses an AI model to identify physical addresses in text and returns structured results with location, position, and confidence for each detected address. |
| Merge PII Detections | n8n-nodes-base.merge | Combines detector outputs | Email Detector; Phone Detector; ID Number Detector; Address Detector AI | PII Consolidation & Conflict Resolver | ## PII Aggregation\nCombines outputs from multiple detectors into a unified dataset for further processing. |
| PII Consolidation & Conflict Resolver | n8n-nodes-base.code | Resolves overlaps and builds unified PII map | Merge PII Detections | Tokenization & Vault Storage | ## Conflict Resolution Engine\nResolves overlapping detections, prioritizes higher confidence matches, and removes duplicate PII entries. |
| Tokenization & Vault Storage | n8n-nodes-base.code | Generates PII tokens and vault records | PII Consolidation & Conflict Resolver | Store Tokens in Vault | ## Tokenization & Vault Storage\nReplaces detected PII with secure tokens and stores original values in a protected database vault. |
| Store Tokens in Vault | n8n-nodes-base.postgres | Stores original PII/token mappings in PostgreSQL | Tokenization & Vault Storage | Generate Masked Text | ## Tokenization & Vault Storage\nReplaces detected PII with secure tokens and stores original values in a protected database vault. |
| Generate Masked Text | n8n-nodes-base.code | Replaces original PII with stored tokens | Store Tokens in Vault | Masking Success Check | ## Masking Validation\nEnsures all PII has been successfully masked and blocks further processing if any sensitive data remains exposed. |
| Masking Success Check | n8n-nodes-base.if | Allows or blocks AI processing based on masking status | Generate Masked Text | AI Processing (Masked Data); Block AI Processing | ## Masking Validation\nEnsures all PII has been successfully masked and blocks further processing if any sensitive data remains exposed. |
| AI Processing (Masked Data) | @n8n/n8n-nodes-langchain.agent | Analyzes masked document with LLM | Masking Success Check; AI Processing Model; AI Output Parser | Re-Injection Controller | ## AI Processing (Safe Data)\nProcesses the masked document using AI while preserving tokens to prevent exposure of sensitive information. |
| AI Processing Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM backend for masked document processing |  | AI Processing (Masked Data) | ## AI Processing (Safe Data)\nProcesses the masked document using AI while preserving tokens to prevent exposure of sensitive information. |
| AI Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parser for AI document analysis output |  | AI Processing (Masked Data) | ## AI Processing (Safe Data)\nProcesses the masked document using AI while preserving tokens to prevent exposure of sensitive information. |
| Re-Injection Controller | n8n-nodes-base.code | Identifies restorable fields and requested tokens | AI Processing (Masked Data) | Retrieve Original Values | ## Re-Injection Controller\nDetermines which fields are allowed to restore original PII based on permissions and prepares retrieval requests and Fetches original sensitive values and Replaces tokens with original values only |
| Retrieve Original Values | n8n-nodes-base.postgres | Fetches original values from vault by token | Re-Injection Controller | Restore Original PII | ## Re-Injection Controller\nDetermines which fields are allowed to restore original PII based on permissions and prepares retrieval requests and Fetches original sensitive values and Replaces tokens with original values only |
| Restore Original PII | n8n-nodes-base.code | Restores approved original values into AI output | Retrieve Original Values | Store Audit Log | ## Re-Injection Controller\nDetermines which fields are allowed to restore original PII based on permissions and prepares retrieval requests and Fetches original sensitive values and Replaces tokens with original values only |
| Store Audit Log | n8n-nodes-base.postgres | Stores compliance and processing metadata | Restore Original PII |  | ## Audit Logging\nStores processing metadata, detected PII types, and re-injection events for compliance and traceability. |
| Block AI Processing | n8n-nodes-base.set | Creates blocked/manual-review payload | Masking Success Check | Send Alert Notification | ## Error Handling & Alerts\nBlocks AI processing and triggers alerts when masking fails or compliance rules are violated. |
| Send Alert Notification | n8n-nodes-base.httpRequest | Sends external alert on masking/compliance failure | Block AI Processing |  | ## Error Handling & Alerts\nBlocks AI processing and triggers alerts when masking fails or compliance rules are violated. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation for intake/config block |  |  | ## Document Intake & Configuration\nReceives documents via webhook and initializes workflow settings such as document ID, confidence thresholds, and database table configuration. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation for text extraction block |  |  | ## Text Extraction\nExtracts raw text from uploaded PDF or document files while preserving original content for downstream processing. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation for regex detector block |  |  | ## Multi-Detector PII Detection\nIdentifies sensitive data including emails, phone numbers, IDs, and addresses using regex-based logic and AI-powered detection. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual documentation for address AI block |  |  | ## Address Detection (AI-Powered)\nUses an AI model to identify physical addresses in text and returns structured results with location, position, and confidence for each detected address. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual documentation for merge block |  |  | ## PII Aggregation\nCombines outputs from multiple detectors into a unified dataset for further processing. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual documentation for conflict-resolution block |  |  | ## Conflict Resolution Engine\nResolves overlapping detections, prioritizes higher confidence matches, and removes duplicate PII entries. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual documentation for re-injection block |  |  | ## Re-Injection Controller\nDetermines which fields are allowed to restore original PII based on permissions and prepares retrieval requests and Fetches original sensitive values and Replaces tokens with original values only |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual documentation for safe AI processing block |  |  | ## AI Processing (Safe Data)\nProcesses the masked document using AI while preserving tokens to prevent exposure of sensitive information. |
| Sticky Note9 | n8n-nodes-base.stickyNote | Visual documentation for audit logging block |  |  | ## Audit Logging\nStores processing metadata, detected PII types, and re-injection events for compliance and traceability. |
| Sticky Note10 | n8n-nodes-base.stickyNote | Visual documentation for error-handling block |  |  | ## Error Handling & Alerts\nBlocks AI processing and triggers alerts when masking fails or compliance rules are violated. |
| Sticky Note11 | n8n-nodes-base.stickyNote | Visual documentation for tokenization/vault block |  |  | ## Tokenization & Vault Storage\nReplaces detected PII with secure tokens and stores original values in a protected database vault. |
| Sticky Note12 | n8n-nodes-base.stickyNote | Visual documentation for masking validation block |  |  | ## Masking Validation\nEnsures all PII has been successfully masked and blocks further processing if any sensitive data remains exposed. |
| Sticky Note13 | n8n-nodes-base.stickyNote | High-level project note and setup guidance |  |  | ## GDPR-Compliant AI Document Processing Pipeline\nThis workflow securely processes documents by detecting and tokenizing PII, masking sensitive data before AI analysis, and selectively restoring original values with full audit logging to ensure privacy, security, and regulatory compliance.\n\n## Setup steps\n1. Activate the webhook and upload a document (PDF or supported file)\n2. Configure AI credentials (Anthropic / OpenAI for detection and processing)\n3. Set database credentials for PII vault and audit log storage\n4. Adjust detection thresholds and compliance settings if needed\n5. Execute workflow and review masked output, AI results, and audit logs |

---

# 4. Reproducing the Workflow from Scratch

Below is the practical rebuild sequence in n8n.

## Important implementation note before rebuilding
The workflow concept is clear, but the exported JSON contains several schema mismatches between nodes. If you want a working rebuild, standardize these fields across all detection and masking steps:

- Use one detection array key everywhere, preferably: `detections`
- Use one coordinate naming convention everywhere: `start`, `end`
- Use one token format everywhere, preferably either:
  - `<<EMAIL_AB12>>`, or
  - `[TOKEN_EMAIL_AB12]`
- Use one PII type field everywhere, preferably: `type`
- Ensure the Postgres vault table stores and returns:
  - `token`
  - `original_value`
  - `type`
  - `document_id`
  - `created_at`
  - optionally `allowed_for_reinjection`

The steps below describe both the current node setup and the corrections needed for a reliable recreation.

## 4.1 Create the entry webhook
1. Add a **Webhook** node named **Document Upload Webhook**.
2. Set:
   - **HTTP Method:** `POST`
   - **Path:** `gdpr-document-upload`
   - **Response Mode:** `Last Node`
   - Enable **Raw Body**
3. Save the node.
4. If you expect binary PDF upload, confirm your client sends a proper file payload compatible with downstream extraction.

## 4.2 Add workflow configuration
5. Add a **Set** node named **Workflow Configuration**.
6. Connect **Document Upload Webhook → Workflow Configuration**.
7. Add these fields:
   - `documentId` as String: `={{ $now.toISO() }}`
   - `confidenceThreshold` as Number: `0.8`
   - `vaultTable` as String: `pii_vault`
   - `auditTable` as String: `pii_audit_log`
8. Enable keeping other incoming fields.

## 4.3 Add PDF text extraction
9. Add an **Extract From File** node named **Extract Text**.
10. Connect **Workflow Configuration → Extract Text**.
11. Configure:
   - **Operation:** `PDF`
   - **Keep Source:** `Both`
12. Ensure the webhook request provides a usable binary PDF file.

## 4.4 Add regex-based email detection
13. Add a **Code** node named **Email Detector**.
14. Connect **Extract Text → Email Detector**.
15. Paste code that:
   - reads `item.json.text`
   - detects email addresses via regex
   - returns a `detections` array
16. Prefer this normalized output shape for each detection:
   - `value`
   - `type: "email"`
   - `start`
   - `end`
   - `confidence`
   - `source: "email_detector"`

## 4.5 Add regex-based phone detection
17. Add a **Code** node named **Phone Detector**.
18. Connect **Extract Text → Phone Detector**.
19. Paste phone detection code.
20. Normalize output to:
   - top-level key `detections`
   - detection coordinates `start` and `end`
21. Do not use `detected_pii` if you want consolidation to work cleanly.

## 4.6 Add regex-based ID detection
22. Add a **Code** node named **ID Number Detector**.
23. Connect **Extract Text → ID Number Detector**.
24. Paste the ID detection logic.
25. Normalize output to:
   - top-level key `detections`
   - each item includes `start` and `end`
   - optional `subtype`
   - `source: "id_detector"`

## 4.7 Add AI-based address detection
26. Add an **AI Agent** node named **Address Detector AI**.
27. Connect **Extract Text → Address Detector AI**.
28. Set **Prompt Type** to `Define`.
29. Set **Text** to:
   - `={{ $json.text }}`
30. In **System Message**, use:
   - “You are a PII detection specialist. Analyze the provided text and identify all physical addresses (street, city, state, postal code, country). Return each address found with its position in the text and a confidence score between 0 and 1.”
31. Enable **Structured Output**.

### Add model for address detection
32. Add an **Anthropic Chat Model** node named **Anthropic Chat Model**.
33. Select Anthropic credentials.
34. Choose model:
   - `claude-sonnet-4-5-20250929`
35. Connect the model node to **Address Detector AI** via the AI language-model connection.

### Add parser for address detection
36. Add a **Structured Output Parser** node named **Address Output Parser**.
37. Connect it to **Address Detector AI** via the AI output-parser connection.
38. Use a schema that returns an object with a `detections` array rather than `addresses` if you want downstream compatibility.
39. Recommended schema item fields:
   - `value`
   - `type` enum containing `address`
   - `start`
   - `end`
   - `confidence`
   - `source`

## 4.8 Merge all detection streams
40. Add a **Merge** node named **Merge PII Detections**.
41. Set **Number of Inputs** to `4`.
42. Connect:
   - **Email Detector → Merge PII Detections** input 1
   - **Phone Detector → Merge PII Detections** input 2
   - **ID Number Detector → Merge PII Detections** input 3
   - **Address Detector AI → Merge PII Detections** input 4

## 4.9 Add consolidation and overlap resolution
43. Add a **Code** node named **PII Consolidation & Conflict Resolver**.
44. Connect **Merge PII Detections → PII Consolidation & Conflict Resolver**.
45. Implement logic to:
   - gather all `detections`
   - sort by `start`
   - resolve overlaps using confidence, then span length
   - deduplicate exact matches
   - output:
     - `consolidatedPII`
     - `originalText`
     - `totalDetections`
     - `documentId`
46. If you keep the original JSON code, you must fix references from `start_pos/end_pos` to `start/end`, or vice versa consistently.

## 4.10 Create tokenization node
47. Add a **Code** node named **Tokenization & Vault Storage**.
48. Connect **PII Consolidation & Conflict Resolver → Tokenization & Vault Storage**.
49. Configure code to:
   - read `consolidatedPII`
   - generate token per detection, such as `<<EMAIL_AB12>>`
   - create one vault record per token
   - generate masked text by replacing original values
50. Output at least:
   - `vaultRecords` array
   - `masked_text`
   - `original_text`
   - `documentId`
   - `token_count`
51. Prefer a stronger unique token suffix than 4 hex chars if collisions matter.

## 4.11 Prepare PostgreSQL vault storage
52. Create a PostgreSQL credential in n8n with access to your compliance database.
53. Create the vault table, for example:
   - `token`
   - `original_value`
   - `type`
   - `document_id`
   - `created_at`
   - `allowed_for_reinjection` optional boolean
54. Add a **Postgres** node named **Store Tokens in Vault**.
55. Connect **Tokenization & Vault Storage → Store Tokens in Vault**.
56. Set:
   - **Schema:** `public`
   - **Table:** `={{ $('Workflow Configuration').first().json.vaultTable }}`
57. Important: itemize `vaultRecords` before insert if needed. In practice, you may need an extra split node or return one item per vault record from the code node.
58. Map columns:
   - `token` → current item token
   - `original_value` → current item original value
   - `type` → current item type
   - `document_id` → current item document ID
   - `created_at` → current item timestamp
59. Remove the placeholder literal `YOUR_CREDENTIAL_HERE`; that value is not valid.

## 4.12 Generate and validate masked text
60. Add a **Code** node named **Generate Masked Text**.
61. Connect **Store Tokens in Vault → Generate Masked Text**.
62. Make it return:
   - `masked_text`
   - `original_text`
   - `token_count`
   - `masking_success`
   - `replacements`
63. If your tokenization node already generated `masked_text`, you can simplify this node to validation only.
64. Ensure this node uses the same token format stored in PostgreSQL.

## 4.13 Add masking gate
65. Add an **If** node named **Masking Success Check**.
66. Connect **Generate Masked Text → Masking Success Check**.
67. Condition:
   - Boolean equals `true`
   - Left value: `={{ $json.masking_success }}`
68. Connect the **true** output to AI processing.
69. Connect the **false** output to the blocking branch.

## 4.14 Add safe AI document processing
70. Add an **AI Agent** node named **AI Processing (Masked Data)**.
71. Connect true branch of **Masking Success Check → AI Processing (Masked Data)**.
72. Set **Text** to:
   - `={{ $json.masked_text }}`
73. Use this system instruction:
   - “You are a document processing AI. Extract structured information from the provided document. IMPORTANT: You are working with masked data where PII has been replaced with tokens like <<EMAIL_7F3A>>. Process the document normally and preserve these tokens in your output exactly as they appear.”
74. Enable structured output.

### Add model for main AI processing
75. Add another **Anthropic Chat Model** node named **AI Processing Model**.
76. Configure Anthropic credentials and choose:
   - `claude-sonnet-4-5-20250929`
77. Connect it to **AI Processing (Masked Data)** as AI language model.

### Add parser for main AI processing
78. Add a **Structured Output Parser** node named **AI Output Parser**.
79. Connect it to **AI Processing (Masked Data)** as AI output parser.
80. Define schema with:
   - `documentType` required string
   - `summary` required string
   - optional `keyEntities`
   - optional `dates`
   - optional `amounts`
   - optional `processedData`

## 4.15 Add re-injection controller
81. Add a **Code** node named **Re-Injection Controller**.
82. Connect **AI Processing (Masked Data) → Re-Injection Controller**.
83. Implement logic to:
   - inspect structured AI output
   - find tokens in allowed fields
   - build one item per token to retrieve, or explicitly split them later
84. Standardize token detection regex to match your chosen token format.
   - If using `<<EMAIL_AB12>>`, search for `<<[A-Z_]+_[A-F0-9]+>>`
85. Provide or compute `fieldPermissions` if you want selective restoration.

## 4.16 Add vault lookup for original values
86. Add a **Postgres** node named **Retrieve Original Values**.
87. Connect **Re-Injection Controller → Retrieve Original Values**.
88. Set:
   - **Operation:** `Select`
   - **Schema:** `public`
   - **Table:** `={{ $('Workflow Configuration').first().json.vaultTable }}`
89. Filter by token column using the current item token.
90. If the controller produces an array of tokens, split it into items first so each query has one token value.

## 4.17 Add restoration node
91. Add a **Code** node named **Restore Original PII**.
92. Connect **Retrieve Original Values → Restore Original PII**.
93. Implement recursive replacement using the same token format used upstream.
94. If selective restoration is required, only restore rows where `allowed_for_reinjection = true`.
95. Output:
   - `finalOutput`
   - `restoredFields`
   - `restorationTimestamp`
   - `totalTokensRestored`

## 4.18 Add audit log storage
96. Create the audit table in PostgreSQL, for example with columns:
   - `actor`
   - `timestamp`
   - `document_id`
   - `token_count`
   - `pii_types_detected`
   - `ai_access_confirmed`
   - `re_injection_events`
97. Add a **Postgres** node named **Store Audit Log**.
98. Connect **Restore Original PII → Store Audit Log**.
99. Set:
   - **Schema:** `public`
   - **Table:** `={{ $('Workflow Configuration').first().json.auditTable }}`
100. Map audit fields from the current item or precompute them in the previous node.

## 4.19 Add blocking branch
101. Add a **Set** node named **Block AI Processing**.
102. Connect false branch of **Masking Success Check → Block AI Processing**.
103. Set:
   - `error = Masking failed - AI processing blocked`
   - `status = BLOCKED`
   - `requires_manual_review = true`

## 4.20 Add alert notification
104. Add an **HTTP Request** node named **Send Alert Notification**.
105. Connect **Block AI Processing → Send Alert Notification**.
106. Set:
   - **Method:** `POST`
   - **URL:** your alerting webhook endpoint
107. Send body fields such as:
   - `error_details`
   - `document_id`
   - `timestamp`
108. Make sure `Block AI Processing` actually provides the fields referenced by this node.

## 4.21 Add visual notes
109. Add the 13 sticky notes if you want parity with the original layout.
110. Copy the section titles and text from the summary table into each note.

## 4.22 Credential configuration summary
111. **Anthropic credentials** are required for:
   - `Anthropic Chat Model`
   - `AI Processing Model`
112. **PostgreSQL credentials** are required for:
   - `Store Tokens in Vault`
   - `Retrieve Original Values`
   - `Store Audit Log`
113. **HTTP endpoint configuration** is required for:
   - `Send Alert Notification`

## 4.23 Recommended fixes for a fully working rebuild
114. Normalize all detector outputs to `detections`.
115. Normalize coordinates to `start` and `end`.
116. Change the address parser schema from `addresses` to `detections`.
117. Make `PII Consolidation & Conflict Resolver` output `consolidatedPII` if the tokenization node expects that name, or update tokenization to use `piiMap`.
118. Make `Tokenization & Vault Storage` emit one item per vault record if you want direct Postgres inserts.
119. Use the same token format in:
   - tokenization
   - AI instructions
   - re-injection controller
   - restoration code
120. Add `allowed_for_reinjection` to the vault schema if you want policy-based restoration.
121. Propagate `documentId` through all branches used for alerting and audit.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| GDPR-Compliant AI Document Processing Pipeline: This workflow securely processes documents by detecting and tokenizing PII, masking sensitive data before AI analysis, and selectively restoring original values with full audit logging to ensure privacy, security, and regulatory compliance. | Overall workflow concept |
| Setup steps: 1. Activate the webhook and upload a document (PDF or supported file). 2. Configure AI credentials (Anthropic / OpenAI for detection and processing). 3. Set database credentials for PII vault and audit log storage. 4. Adjust detection thresholds and compliance settings if needed. 5. Execute workflow and review masked output, AI results, and audit logs. | Project setup guidance |
| The exported workflow is conceptually strong but contains multiple field-name mismatches that will likely prevent correct end-to-end execution without normalization. | Implementation note |
| The description mentions Anthropic and PostgreSQL only; no external blog, video, or branding links are embedded in the workflow. | Resource availability |

## Final technical assessment
This workflow is best understood as a strong architectural prototype rather than a guaranteed plug-and-run build. The main corrections needed are:
- unify detection output schema
- unify token format
- fix Postgres insert/select mappings
- propagate document ID and audit fields consistently
- align re-injection logic with actual stored token format

Once those are corrected, the design provides a sound pattern for privacy-preserving AI document analysis in n8n.