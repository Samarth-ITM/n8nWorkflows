Analyze contract PDFs and score risk with Claude 3.5, Postgres, email and Slack alerts

https://n8nworkflows.xyz/workflows/analyze-contract-pdfs-and-score-risk-with-claude-3-5--postgres--email-and-slack-alerts-14823


# Analyze contract PDFs and score risk with Claude 3.5, Postgres, email and Slack alerts

# 1. Workflow Overview

This workflow receives a contract PDF through an HTTP webhook, extracts its text, analyzes it with Claude 3.5 through four parallel AI agent branches, computes a weighted legal risk score, stores the results in PostgreSQL, and sends email and Slack alerts when the risk exceeds a configured threshold.

Typical use cases include:

- Intake and triage of vendor/customer/legal agreements
- Automated contract review support
- Legal operations risk flagging
- Obligation extraction for downstream compliance tracking
- Centralized storage of contract analytics in a relational database

## 1.1 Input Reception and Runtime Configuration

The workflow starts from a webhook that accepts a PDF upload. Immediately after, a configuration node injects runtime values such as the risk threshold and alert destinations.

## 1.2 PDF Text Extraction

The uploaded file is parsed as a PDF and converted into plain text. That extracted text becomes the common input for all AI analysis branches.

## 1.3 Parallel AI Analysis

The extracted contract text is sent to four separate AI agents in parallel:

- Clause extraction
- Risk assessment
- Obligation extraction
- Executive summary generation

Each agent uses the same Claude 3.5 model and a dedicated structured output parser.

## 1.4 Result Aggregation and Risk Scoring

The outputs from the four AI branches are merged and passed to a custom JavaScript scoring node. This node calculates a weighted overall risk score, score breakdowns, red flags, and recommended actions.

## 1.5 Data Preparation and Persistence

The normalized contract data is prepared and then inserted into multiple PostgreSQL tables:

- `contracts`
- `contract_clauses`
- `contract_obligations`
- `risk_scoring`

## 1.6 Risk Evaluation and Alerting

After storage, the workflow compares the overall risk score to the configured threshold. If the threshold is met or exceeded, email and Slack alerts are sent.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Configuration

### Overview
This block receives the uploaded contract via HTTP and enriches the execution with configurable values used later for threshold checks and notifications. It defines the workflow’s runtime context before analysis begins.

### Nodes Involved
- Contract Upload Webhook
- Workflow Configuration

### Node Details

#### Contract Upload Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point that accepts HTTP POST requests containing the contract file.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `contract-upload`
  - Response mode: `responseNode`
- **Key expressions or variables used:** None directly in parameters.
- **Input and output connections:**
  - No input node
  - Outputs to `Workflow Configuration`
- **Version-specific requirements:**
  - Type version `2.1`
  - `responseNode` mode usually implies a response node should be used somewhere in the workflow, but none is present here.
- **Edge cases or potential failure types:**
  - Incoming request may not include a binary PDF file
  - Wrong content type or malformed multipart payload
  - Because there is no dedicated response node in the workflow, webhook response behavior may be incomplete or error-prone depending on n8n version/settings
  - Large PDF payloads may hit webhook body size limits
- **Sub-workflow reference:** None

#### Workflow Configuration
- **Type and technical role:** `n8n-nodes-base.set`  
  Adds static runtime settings used later in risk evaluation and alerts.
- **Configuration choices:**
  - `riskThreshold` = `0.7`
  - `alertRecipients` = placeholder string for high-risk email recipients
  - `slackChannel` = placeholder string for Slack channel ID
  - `includeOtherFields` = `true`, so upstream webhook data is preserved
- **Key expressions or variables used:**
  - Exposes fields later referenced as:
    - `$('Workflow Configuration').first().json.riskThreshold`
    - `$('Workflow Configuration').first().json.alertRecipients`
    - `$('Workflow Configuration').first().json.slackChannel`
- **Input and output connections:**
  - Input from `Contract Upload Webhook`
  - Output to `Extract PDF Text`
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases or potential failure types:**
  - Placeholder values must be replaced before production use
  - Invalid Slack channel ID or email list will not fail here, but will fail downstream in alert nodes
- **Sub-workflow reference:** None

---

## 2.2 PDF Text Extraction

### Overview
This block converts the uploaded PDF into machine-readable text. All downstream AI analysis relies on this extracted text being present and meaningful.

### Nodes Involved
- Extract PDF Text

### Node Details

#### Extract PDF Text
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Extracts plain text from the uploaded binary PDF.
- **Configuration choices:**
  - Operation: `pdf`
- **Key expressions or variables used:** None in the node itself; downstream nodes consume `{{$json.text}}`.
- **Input and output connections:**
  - Input from `Workflow Configuration`
  - Outputs in parallel to:
    - `Clause Extraction Agent`
    - `Risk Assessment Agent`
    - `Obligation Extraction Agent`
    - `Executive Summary Agent`
- **Version-specific requirements:**
  - Type version `1.1`
- **Edge cases or potential failure types:**
  - No binary file available from webhook payload
  - Password-protected, image-only, corrupted, or malformed PDFs
  - OCR is not configured, so scanned PDFs may produce little or no text
  - Very long contracts may produce text too large for efficient LLM handling
- **Sub-workflow reference:** None

---

## 2.3 Clause Analysis

### Overview
This branch asks the LLM to identify and categorize contractual clauses and emit a structured array of clause objects. It is the main source for clause presence checks during scoring.

### Nodes Involved
- Clause Extraction Agent
- Clause Parser
- Claude 3.5 Model

### Node Details

#### Clause Extraction Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain agent that performs clause extraction from the contract text.
- **Configuration choices:**
  - Prompt type: `define`
  - Input text: `={{ $json.text }}`
  - Uses a system message instructing the model to:
    - Extract all clauses
    - Categorize them into `payment_terms`, `termination`, `liability`, `ip`, `confidentiality`, `warranty`, `indemnification`
    - Return `clause_type`, `clause_text`, `ai_confidence`
  - Structured output enabled via output parser
- **Key expressions or variables used:**
  - `{{$json.text}}`
- **Input and output connections:**
  - Main input from `Extract PDF Text`
  - AI language model input from `Claude 3.5 Model`
  - AI output parser input from `Clause Parser`
  - Main output to `Merge Analysis Results` on input index `0`
- **Version-specific requirements:**
  - Type version `3`
  - Requires compatible LangChain nodes in the installed n8n version
- **Edge cases or potential failure types:**
  - Empty or truncated `text`
  - Model may return incomplete schema-conforming output
  - Hallucinated clause types outside the expected taxonomy
  - Token limits on large contracts
  - Anthropic credential/configuration issues
- **Sub-workflow reference:** None

#### Clause Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Forces structured parsing of the clause extraction response.
- **Configuration choices:**
  - JSON schema example expects:
    - root key `clauses`
    - array items with `clause_type`, `clause_text`, `ai_confidence`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `Clause Extraction Agent` via `ai_outputParser`
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases or potential failure types:**
  - Parser may fail if model output deviates significantly from schema
  - Schema example is illustrative rather than a strict JSON Schema, so exact enforcement depends on node implementation/version
- **Sub-workflow reference:** None

#### Claude 3.5 Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
  Shared LLM backend used by all four AI agents.
- **Configuration choices:**
  - Model: `claude-3-5-sonnet-20241022`
  - No special options set
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Provides `ai_languageModel` connection to:
    - `Clause Extraction Agent`
    - `Risk Assessment Agent`
    - `Obligation Extraction Agent`
    - `Executive Summary Agent`
- **Version-specific requirements:**
  - Type version `1.3`
  - Requires Anthropic credentials configured in n8n
  - Model availability depends on Anthropic account access and current supported model list
- **Edge cases or potential failure types:**
  - Invalid API key
  - Rate limits
  - Timeout on very large prompts
  - Model deprecation or renamed model IDs
- **Sub-workflow reference:** None

---

## 2.4 Risk Analysis

### Overview
This branch asks the LLM to evaluate contract clauses for legal/commercial risk and produce structured risk assessments with recommendations. The downstream score calculation depends heavily on this output.

### Nodes Involved
- Risk Assessment Agent
- Risk Parser
- Claude 3.5 Model

### Node Details

#### Risk Assessment Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI agent that assigns risk levels and recommendations based on the contract text.
- **Configuration choices:**
  - Prompt type: `define`
  - Input text: `={{ $json.text }}`
  - System message instructs the model to:
    - Analyze clauses for risks
    - Assign `critical`, `high`, `medium`, `low`, `none`
    - Identify category-specific risk factors
    - Return `clause_type`, `risk_level`, `risk_description`, `recommendation`, `ai_confidence`
  - Structured output enabled
- **Key expressions or variables used:**
  - `{{$json.text}}`
- **Input and output connections:**
  - Main input from `Extract PDF Text`
  - AI language model input from `Claude 3.5 Model`
  - AI output parser input from `Risk Parser`
  - Main output to `Merge Analysis Results` on input index `1`
- **Version-specific requirements:**
  - Type version `3`
- **Edge cases or potential failure types:**
  - Risk output may not align with extracted clauses because this branch runs independently
  - Unsupported risk labels could reduce scoring accuracy
  - Large contracts may exceed token or execution limits
- **Sub-workflow reference:** None

#### Risk Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Parses the risk analysis into structured JSON.
- **Configuration choices:**
  - JSON schema example expects:
    - root key `risks`
    - array items with `clause_type`, `risk_level`, `risk_description`, `recommendation`, `ai_confidence`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `Risk Assessment Agent` via `ai_outputParser`
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases or potential failure types:**
  - Nonconforming model output
  - Missing `risks` root key
- **Sub-workflow reference:** None

---

## 2.5 Obligation Tracking

### Overview
This branch extracts actionable obligations from the contract, including due dates, responsible parties, and an initial status. These results are later persisted to a dedicated obligations table.

### Nodes Involved
- Obligation Extraction Agent
- Obligation Parser
- Claude 3.5 Model

### Node Details

#### Obligation Extraction Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI agent focused on duty and deadline extraction.
- **Configuration choices:**
  - Input text: `={{ $json.text }}`
  - System message instructs the model to:
    - Find obligations
    - Look for language such as `must`, `shall`, `required`, `obligated`, `agrees to`
    - Categorize into `payment`, `delivery`, `reporting`, `compliance`
    - Extract due dates in ISO format or null
    - Identify responsible party
    - Set initial status to `pending`
  - Structured output enabled
- **Key expressions or variables used:**
  - `{{$json.text}}`
- **Input and output connections:**
  - Main input from `Extract PDF Text`
  - AI language model input from `Claude 3.5 Model`
  - AI output parser input from `Obligation Parser`
  - Main output to `Merge Analysis Results` on input index `2`
- **Version-specific requirements:**
  - Type version `3`
- **Edge cases or potential failure types:**
  - Due dates may be ambiguous or not ISO-normalized
  - Responsible party may be omitted in the contract
  - Model can miss implicit obligations
- **Sub-workflow reference:** None

#### Obligation Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`
- **Configuration choices:**
  - JSON schema example expects:
    - root key `obligations`
    - array items with `obligation_text`, `obligation_type`, `due_date`, `responsible_party`, `status`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `Obligation Extraction Agent` via `ai_outputParser`
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases or potential failure types:**
  - Parser failure on malformed AI output
  - Date formatting inconsistencies
- **Sub-workflow reference:** None

---

## 2.6 Executive Summary Generation

### Overview
This branch produces a structured summary of the contract, including parties, dates, value, key terms, and overall assessment. It also supplies core metadata later used for the primary contract record.

### Nodes Involved
- Executive Summary Agent
- Summary Parser
- Claude 3.5 Model

### Node Details

#### Executive Summary Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI agent that creates a high-level legal/commercial summary from the contract text.
- **Configuration choices:**
  - Input text: `={{ $json.text }}`
  - System message instructs the model to return:
    - `contract_name`
    - `parties`
    - `contract_type`
    - `key_terms`
    - `major_obligations`
    - `notable_provisions`
    - `overall_assessment`
  - The schema example also includes `execution_date`, `expiry_date`, and `value`
- **Key expressions or variables used:**
  - `{{$json.text}}`
- **Input and output connections:**
  - Main input from `Extract PDF Text`
  - AI language model input from `Claude 3.5 Model`
  - AI output parser input from `Summary Parser`
  - Main output to `Merge Analysis Results` on input index `3`
- **Version-specific requirements:**
  - Type version `3`
- **Edge cases or potential failure types:**
  - Prompt text and parser example are not perfectly aligned: prompt omits some fields present in parser example
  - Missing `execution_date`, `expiry_date`, or `value` can break downstream expectations or result in null/undefined values
- **Sub-workflow reference:** None

#### Summary Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`
- **Configuration choices:**
  - JSON schema example includes:
    - `contract_name`
    - `parties`
    - `contract_type`
    - `execution_date`
    - `expiry_date`
    - `value`
    - `key_terms`
    - `major_obligations`
    - `notable_provisions`
    - `overall_assessment`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Connected to `Executive Summary Agent` via `ai_outputParser`
- **Version-specific requirements:**
  - Type version `1.3`
- **Edge cases or potential failure types:**
  - Missing fields from AI output
  - Type mismatches, especially numeric `value`
- **Sub-workflow reference:** None

---

## 2.7 Aggregation and Weighted Risk Scoring

### Overview
This block combines all AI outputs into a single item and computes a weighted contract risk score, red flags, recommendations, and confidence-based review triggers.

### Nodes Involved
- Merge Analysis Results
- Calculate Risk Score

### Node Details

#### Merge Analysis Results
- **Type and technical role:** `n8n-nodes-base.merge`  
  Waits for four inputs and combines the AI outputs into one downstream dataset.
- **Configuration choices:**
  - `numberInputs` = `4`
- **Key expressions or variables used:** None in node configuration.
- **Input and output connections:**
  - Input 0 from `Clause Extraction Agent`
  - Input 1 from `Risk Assessment Agent`
  - Input 2 from `Obligation Extraction Agent`
  - Input 3 from `Executive Summary Agent`
  - Output to `Calculate Risk Score`
- **Version-specific requirements:**
  - Type version `3.2`
  - Behavior depends on merge mode defaults in that version
- **Edge cases or potential failure types:**
  - If one AI branch fails or returns no item, merge may block or produce unexpected results
  - Cardinality mismatches can affect downstream item indexing assumptions
- **Sub-workflow reference:** None

#### Calculate Risk Score
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript logic to derive weighted risk scoring and review signals.
- **Configuration choices:**
  - Reads merged inputs using:
    - `$input.first().json.clauses || []`
    - `$input.item(1).json.risks || []`
    - `$input.item(2).json.obligations || []`
    - `$input.item(3).json`
  - Risk weights:
    - `payment_terms: 0.25`
    - `liability: 0.30`
    - `termination: 0.15`
    - `ip: 0.15`
    - `confidentiality: 0.10`
    - `warranty: 0.03`
    - `indemnification: 0.02`
  - Risk scores:
    - `critical: 1.0`
    - `high: 0.75`
    - `medium: 0.5`
    - `low: 0.25`
    - `none: 0.0`
  - Adds red flags for:
    - High/critical risks
    - Missing critical clauses: `payment_terms`, `liability`, `termination`
    - Low-confidence AI items (`ai_confidence < 0.6`)
- **Key expressions or variables used:**
  - `$input.first()`
  - `$input.item(1)`
  - `$input.item(2)`
  - `$input.item(3)`
  - `new Date().toISOString()`
- **Input and output connections:**
  - Input from `Merge Analysis Results`
  - Output to `Prepare Contract Data`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Assumes merged item ordering is stable and exactly matches the four branch inputs
  - If summary output is nested differently than expected, `summaryData` may be malformed
  - Unsupported `risk_level` values default to zero
  - Duplicate risk entries of the same type overwrite `scoreBreakdown` keys while still adding to total score
  - The returned structure is a plain object, which is acceptable in Code node contexts only if n8n wraps it as expected in that version
- **Sub-workflow reference:** None

---

## 2.8 Data Preparation and Storage

### Overview
This block maps the aggregated contract analysis into database-ready fields and writes records into four PostgreSQL tables in sequence.

### Nodes Involved
- Prepare Contract Data
- Insert Contract
- Insert Clauses
- Insert Obligations
- Insert Risk Scoring

### Node Details

#### Prepare Contract Data
- **Type and technical role:** `n8n-nodes-base.set`  
  Normalizes and flattens data for database insertion.
- **Configuration choices:**
  - Creates:
    - `contract_id` from current ISO timestamp + slugified contract name
    - summary-derived metadata such as `contract_name`, `contract_type`, dates, value
    - static `status = draft`
    - placeholder `uploaded_by`
    - `upload_date = {{$now.toISO()}}`
    - `document_hash` = contract name length
    - serialized-looking fields for `clauses`, `risks`, `obligations`, `score_breakdown`, `red_flags`, `recommendations`
    - `overall_risk_score`, `calculated_at`
- **Key expressions or variables used:**
  - `{{ $now.toISO() }}`
  - `{{ $json.summary.contract_name.replace(/\s+/g, "-").toLowerCase() }}`
  - `{{ $json.summary.contract_name }}`
  - `{{ $json.summary.contract_type }}`
  - `{{ $json.summary.execution_date }}`
  - `{{ $json.summary.expiry_date }}`
  - `{{ $json.summary.value }}`
  - `{{ $json.clauses }}`
  - `{{ $json.risks }}`
  - `{{ $json.obligations }}`
  - `{{ $json.score_breakdown }}`
  - `{{ $json.red_flags }}`
  - `{{ $json.recommendations }}`
- **Input and output connections:**
  - Input from `Calculate Risk Score`
  - Output to `Insert Contract`
- **Version-specific requirements:**
  - Type version `3.4`
- **Edge cases or potential failure types:**
  - `contract_name` must exist or the `replace()` call will fail
  - `document_hash` is not a real file hash; it is only the contract name length
  - Arrays/objects assigned as `string` may be cast in inconsistent ways unless explicitly JSON-stringified
  - Placeholder `uploaded_by` must be replaced for real traceability
- **Sub-workflow reference:** None

#### Insert Contract
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Inserts the top-level contract record into `public.contracts`.
- **Configuration choices:**
  - Schema: `public`
  - Table: `contracts`
  - Defined columns:
    - `id`
    - `contract_name`
    - `contract_type`
    - `execution_date`
    - `expiry_date`
    - `value`
    - `status`
    - `uploaded_by`
    - `upload_date`
    - `document_hash`
    - `created_at`
- **Key expressions or variables used:**
  - Maps from prepared fields such as `{{$json.contract_id}}`, `{{$json.contract_name}}`, `{{$json.value}}`
  - `created_at = {{$now.toISO()}}`
- **Input and output connections:**
  - Input from `Prepare Contract Data`
  - Output to `Insert Clauses`
- **Version-specific requirements:**
  - Type version `2.6`
  - Requires PostgreSQL credentials
- **Edge cases or potential failure types:**
  - Table or column mismatch with actual schema
  - Type mismatch for numeric/date columns
  - Duplicate primary key on repeated executions with same generated ID pattern
- **Sub-workflow reference:** None

#### Insert Clauses
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Intended to insert clause-level rows into `public.contract_clauses`.
- **Configuration choices:**
  - Table: `contract_clauses`
  - Columns:
    - `contract_id`
    - `clause_type`
    - `clause_text`
    - `ai_confidence`
  - `queryBatching = single`
- **Key expressions or variables used:**
  - `{{$json.contract_id}}`
  - `{{$json.clause_type}}`
  - `{{$json.clause_text}}`
  - `{{$json.ai_confidence}}`
- **Input and output connections:**
  - Input from `Insert Contract`
  - Output to `Insert Obligations`
- **Version-specific requirements:**
  - Type version `2.6`
- **Edge cases or potential failure types:**
  - As currently wired, this node receives the output of `Insert Contract`, not exploded clause items
  - Unless a transformation step exists elsewhere, `clause_type`, `clause_text`, and `ai_confidence` may be missing, causing null inserts or failures
  - Batch behavior may not help without item splitting
- **Sub-workflow reference:** None

#### Insert Obligations
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Intended to insert obligation rows into `public.contract_obligations`.
- **Configuration choices:**
  - Table: `contract_obligations`
  - Columns:
    - `contract_id`
    - `obligation_text`
    - `obligation_type`
    - `due_date`
    - `responsible_party`
    - `status`
- **Key expressions or variables used:**
  - `{{$json.contract_id}}`
  - `{{$json.obligation_text}}`
  - `{{$json.obligation_type}}`
  - `{{$json.due_date}}`
  - `{{$json.responsible_party}}`
  - `{{$json.status}}`
- **Input and output connections:**
  - Input from `Insert Clauses`
  - Output to `Insert Risk Scoring`
- **Version-specific requirements:**
  - Type version `2.6`
- **Edge cases or potential failure types:**
  - Same structural issue as `Insert Clauses`: there is no item-splitting node between prepared aggregate data and this insert
  - Fields may not exist at row level
- **Sub-workflow reference:** None

#### Insert Risk Scoring
- **Type and technical role:** `n8n-nodes-base.postgres`  
  Inserts summarized risk scoring data into `public.risk_scoring`.
- **Configuration choices:**
  - Table: `risk_scoring`
  - Columns:
    - `contract_id`
    - `overall_risk_score`
    - `score_breakdown`
    - `red_flags`
    - `recommendations`
    - `calculated_at`
- **Key expressions or variables used:**
  - `{{$json.contract_id}}`
  - `{{$json.overall_risk_score}}`
  - `{{$json.score_breakdown}}`
  - `{{$json.red_flags}}`
  - `{{$json.recommendations}}`
  - `{{$json.calculated_at}}`
- **Input and output connections:**
  - Input from `Insert Obligations`
  - Output to `Check Risk Level`
- **Version-specific requirements:**
  - Type version `2.6`
- **Edge cases or potential failure types:**
  - Arrays/objects may need explicit JSON/JSONB casting depending on DB schema
  - If upstream insert nodes transform output unexpectedly, required fields may no longer be available
- **Sub-workflow reference:** None

---

## 2.9 Risk Evaluation and Notifications

### Overview
This block determines whether the contract is high risk and sends alert messages through Gmail and Slack when the threshold is reached.

### Nodes Involved
- Check Risk Level
- Send Email Alert
- Send Slack Alert

### Node Details

#### Check Risk Level
- **Type and technical role:** `n8n-nodes-base.if`  
  Compares the computed risk score to the configured threshold.
- **Configuration choices:**
  - Condition: `overall_risk_score >= riskThreshold`
  - Left value: `={{ $json.overall_risk_score }}`
  - Right value: `={{ $('Workflow Configuration').first().json.riskThreshold }}`
  - Strict type validation enabled
- **Key expressions or variables used:**
  - `{{$json.overall_risk_score}}`
  - `{{ $('Workflow Configuration').first().json.riskThreshold }}`
- **Input and output connections:**
  - Input from `Insert Risk Scoring`
  - True output goes to:
    - `Send Email Alert`
    - `Send Slack Alert`
  - No false branch connected
- **Version-specific requirements:**
  - Type version `2.3`
- **Edge cases or potential failure types:**
  - If `overall_risk_score` is missing or string-typed unexpectedly, strict validation may fail
  - False branch silently ends execution
- **Sub-workflow reference:** None

#### Send Email Alert
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends a high-risk contract notification email.
- **Configuration choices:**
  - Recipient(s): `={{ $('Workflow Configuration').first().json.alertRecipients }}`
  - Subject: `HIGH RISK CONTRACT ALERT: {{ $json.contract_name }}`
  - HTML body includes:
    - contract name
    - risk score and threshold
    - red flags rendered as list items
    - recommendations rendered as list items
- **Key expressions or variables used:**
  - `{{ $('Workflow Configuration').first().json.alertRecipients }}`
  - `{{ $json.contract_name }}`
  - `{{ $json.overall_risk_score }}`
  - `{{ $('Workflow Configuration').first().json.riskThreshold }}`
  - `{{ $json.red_flags.map(...) }}`
  - `{{ $json.recommendations.map(...) }}`
- **Input and output connections:**
  - Input from `Check Risk Level` true branch
- **Version-specific requirements:**
  - Type version `2.2`
  - Requires Gmail credentials and appropriate send permissions
- **Edge cases or potential failure types:**
  - Placeholder email addresses not replaced
  - If upstream node output no longer includes arrays for `red_flags` or `recommendations`, `.map()` will fail
  - Gmail OAuth scopes or quota issues
- **Sub-workflow reference:** None

#### Send Slack Alert
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a formatted warning message to a Slack channel.
- **Configuration choices:**
  - Send target type: `channel`
  - Channel ID: `={{ $('Workflow Configuration').first().json.slackChannel }}`
  - Message body includes:
    - contract name
    - risk score and threshold
    - numbered red flags
    - numbered recommendations
- **Key expressions or variables used:**
  - `{{ $('Workflow Configuration').first().json.slackChannel }}`
  - `{{ $json.contract_name }}`
  - `{{ $json.overall_risk_score }}`
  - `{{ $('Workflow Configuration').first().json.riskThreshold }}`
  - `{{ $json.red_flags.map((flag, i) => ...) }}`
  - `{{ $json.recommendations.map((rec, i) => ...) }}`
- **Input and output connections:**
  - Input from `Check Risk Level` true branch
- **Version-specific requirements:**
  - Type version `2.4`
  - Requires Slack credentials/app permissions for posting to the channel
- **Edge cases or potential failure types:**
  - Placeholder channel ID not replaced
  - `.map()` will fail if arrays are not preserved in upstream output
  - Bot may not be invited to channel
- **Sub-workflow reference:** None

---

## 2.10 Documentation / Visual Annotation Nodes

### Overview
These nodes do not affect execution. They provide visual documentation and operational context inside the n8n canvas.

### Nodes Involved
- Sticky Note1
- Sticky Note
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

### Node Details

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Content: `## Alert System\nemail for high-risk cases`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Content: `## Alert System\nSend Slack`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Content: `## Risk Evaluation\nCompare score with risk threshold`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Content: `## Data Storage\nStore contract, clauses, and risk data`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Content: `## Risk Scoring\nCalculate weighted contract risk score`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Content: `## Data Aggregation\nMerge outputs from all AI agents`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Content: `## Clause Analysis\nIdentify and categorize contract clauses`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note7
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Content: `## Risk Analysis\nEvaluate clause risks and recommendations`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note8
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Content: `## Obligation Tracking\nExtract duties, deadlines, and responsibilities`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note9
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Content: `## AI Model\nClaude 3.5 language model configuration`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note10
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Content: `## Executive Summary\nGenerate structured contract overview`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note11
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Content: `## Text Extraction\nExtract text from uploaded PDF file`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note12
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Content: `## Input Layer\nUpload contract via webhook trigger and \nconfigure risk`
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

#### Sticky Note13
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Content includes workflow explanation and setup steps.
- **Connections:** None
- **Edge cases:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Contract Upload Webhook | n8n-nodes-base.webhook | HTTP entry point for PDF upload |  | Workflow Configuration | ## Input Layer<br>Upload contract via webhook trigger and configure risk |
| Workflow Configuration | n8n-nodes-base.set | Injects risk threshold and alert destinations | Contract Upload Webhook | Extract PDF Text | ## Input Layer<br>Upload contract via webhook trigger and configure risk |
| Extract PDF Text | n8n-nodes-base.extractFromFile | Extracts plain text from uploaded PDF | Workflow Configuration | Clause Extraction Agent; Risk Assessment Agent; Obligation Extraction Agent; Executive Summary Agent | ## Text Extraction<br>Extract text from uploaded PDF file |
| Clause Extraction Agent | @n8n/n8n-nodes-langchain.agent | Extracts and categorizes contract clauses | Extract PDF Text; Claude 3.5 Model; Clause Parser | Merge Analysis Results | ## Clause Analysis<br>Identify and categorize contract clauses |
| Clause Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Parses structured clause output |  | Clause Extraction Agent | ## Clause Analysis<br>Identify and categorize contract clauses |
| Risk Assessment Agent | @n8n/n8n-nodes-langchain.agent | Produces clause-level risk analysis and recommendations | Extract PDF Text; Claude 3.5 Model; Risk Parser | Merge Analysis Results | ## Risk Analysis<br>Evaluate clause risks and recommendations |
| Risk Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Parses structured risk output |  | Risk Assessment Agent | ## Risk Analysis<br>Evaluate clause risks and recommendations |
| Obligation Extraction Agent | @n8n/n8n-nodes-langchain.agent | Extracts obligations, due dates, and responsible parties | Extract PDF Text; Claude 3.5 Model; Obligation Parser | Merge Analysis Results | ## Obligation Tracking<br>Extract duties, deadlines, and responsibilities |
| Obligation Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Parses structured obligations output |  | Obligation Extraction Agent | ## Obligation Tracking<br>Extract duties, deadlines, and responsibilities |
| Executive Summary Agent | @n8n/n8n-nodes-langchain.agent | Produces structured contract summary | Extract PDF Text; Claude 3.5 Model; Summary Parser | Merge Analysis Results | ## Executive Summary<br>Generate structured contract overview |
| Summary Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Parses structured summary output |  | Executive Summary Agent | ## Executive Summary<br>Generate structured contract overview |
| Claude 3.5 Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Shared Anthropic model for all AI agents |  | Clause Extraction Agent; Risk Assessment Agent; Obligation Extraction Agent; Executive Summary Agent | ## AI Model<br>Claude 3.5 language model configuration |
| Merge Analysis Results | n8n-nodes-base.merge | Combines outputs from four AI branches | Clause Extraction Agent; Risk Assessment Agent; Obligation Extraction Agent; Executive Summary Agent | Calculate Risk Score | ## Data Aggregation<br>Merge outputs from all AI agents |
| Calculate Risk Score | n8n-nodes-base.code | Computes weighted risk score, flags, and recommendations | Merge Analysis Results | Prepare Contract Data | ## Risk Scoring<br>Calculate weighted contract risk score |
| Prepare Contract Data | n8n-nodes-base.set | Maps analysis into database-ready fields | Calculate Risk Score | Insert Contract | ## Data Storage<br>Store contract, clauses, and risk data |
| Insert Contract | n8n-nodes-base.postgres | Inserts main contract record into Postgres | Prepare Contract Data | Insert Clauses | ## Data Storage<br>Store contract, clauses, and risk data |
| Insert Clauses | n8n-nodes-base.postgres | Intended to insert clause records into Postgres | Insert Contract | Insert Obligations | ## Data Storage<br>Store contract, clauses, and risk data |
| Insert Obligations | n8n-nodes-base.postgres | Intended to insert obligation records into Postgres | Insert Clauses | Insert Risk Scoring | ## Data Storage<br>Store contract, clauses, and risk data |
| Insert Risk Scoring | n8n-nodes-base.postgres | Inserts risk scoring record into Postgres | Insert Obligations | Check Risk Level | ## Data Storage<br>Store contract, clauses, and risk data |
| Check Risk Level | n8n-nodes-base.if | Compares computed score to configured threshold | Insert Risk Scoring | Send Email Alert; Send Slack Alert | ## Risk Evaluation<br>Compare score with risk threshold |
| Send Email Alert | n8n-nodes-base.gmail | Sends email for high-risk contracts | Check Risk Level |  | ## Alert System<br>email for high-risk cases |
| Send Slack Alert | n8n-nodes-base.slack | Sends Slack alert for high-risk contracts | Check Risk Level |  | ## Alert System<br>Send Slack |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note10 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note11 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note12 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Sticky Note13 | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and give it the title:  
   **Analyze contract PDFs and score risk with Claude 3.5, Postgres, email and Slack alerts**

2. **Add a Webhook node** named **Contract Upload Webhook**.
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Path: `contract-upload`
   - Response Mode: `Response Node`
   - Expect the incoming request to contain a PDF file as binary data.

3. **Add a Set node** named **Workflow Configuration** after the webhook.
   - Enable keeping other input fields.
   - Add:
     - `riskThreshold` as Number = `0.7`
     - `alertRecipients` as String = your recipient email(s)
     - `slackChannel` as String = Slack channel ID
   - Connect: `Contract Upload Webhook -> Workflow Configuration`

4. **Add an Extract From File node** named **Extract PDF Text**.
   - Operation: `PDF`
   - Connect: `Workflow Configuration -> Extract PDF Text`

5. **Add an Anthropic Chat Model node** named **Claude 3.5 Model**.
   - Type: `Anthropic Chat Model` from LangChain nodes
   - Model: `claude-3-5-sonnet-20241022`
   - Configure Anthropic credentials
   - This node will not be in the main flow; it connects through AI model ports.

6. **Add a Structured Output Parser node** named **Clause Parser**.
   - Use a schema example matching:
     - Root object with `clauses`
     - Array items containing `clause_type`, `clause_text`, `ai_confidence`

7. **Add an AI Agent node** named **Clause Extraction Agent**.
   - Prompt type: define manually
   - Input text: `={{ $json.text }}`
   - Enable structured output parser
   - System message should instruct the model to:
     - extract all clauses
     - categorize into `payment_terms`, `termination`, `liability`, `ip`, `confidentiality`, `warranty`, `indemnification`
     - return `clause_type`, `clause_text`, `ai_confidence`
   - Connect:
     - Main: `Extract PDF Text -> Clause Extraction Agent`
     - AI model: `Claude 3.5 Model -> Clause Extraction Agent`
     - AI parser: `Clause Parser -> Clause Extraction Agent`

8. **Add a Structured Output Parser node** named **Risk Parser**.
   - Schema example:
     - Root key `risks`
     - Each item with `clause_type`, `risk_level`, `risk_description`, `recommendation`, `ai_confidence`

9. **Add an AI Agent node** named **Risk Assessment Agent**.
   - Input text: `={{ $json.text }}`
   - System message should instruct the model to:
     - assess risk per clause
     - use `critical`, `high`, `medium`, `low`, `none`
     - provide descriptions and recommendations
     - return `clause_type`, `risk_level`, `risk_description`, `recommendation`, `ai_confidence`
   - Connect:
     - Main: `Extract PDF Text -> Risk Assessment Agent`
     - AI model: `Claude 3.5 Model -> Risk Assessment Agent`
     - AI parser: `Risk Parser -> Risk Assessment Agent`

10. **Add a Structured Output Parser node** named **Obligation Parser**.
    - Schema example:
      - Root key `obligations`
      - Each item with `obligation_text`, `obligation_type`, `due_date`, `responsible_party`, `status`

11. **Add an AI Agent node** named **Obligation Extraction Agent**.
    - Input text: `={{ $json.text }}`
    - System message should instruct the model to:
      - find obligations
      - use keywords such as `must`, `shall`, `required`, `obligated`, `agrees to`
      - categorize obligations into `payment`, `delivery`, `reporting`, `compliance`
      - extract due dates in ISO format or null
      - identify responsible party
      - set status to `pending`
    - Connect:
      - Main: `Extract PDF Text -> Obligation Extraction Agent`
      - AI model: `Claude 3.5 Model -> Obligation Extraction Agent`
      - AI parser: `Obligation Parser -> Obligation Extraction Agent`

12. **Add a Structured Output Parser node** named **Summary Parser**.
    - Schema example should include:
      - `contract_name`
      - `parties`
      - `contract_type`
      - `execution_date`
      - `expiry_date`
      - `value`
      - `key_terms`
      - `major_obligations`
      - `notable_provisions`
      - `overall_assessment`

13. **Add an AI Agent node** named **Executive Summary Agent**.
    - Input text: `={{ $json.text }}`
    - System message should instruct the model to generate a structured summary containing:
      - contract name
      - parties
      - contract type
      - dates
      - value
      - key terms
      - obligations
      - notable provisions
      - overall assessment
    - Connect:
      - Main: `Extract PDF Text -> Executive Summary Agent`
      - AI model: `Claude 3.5 Model -> Executive Summary Agent`
      - AI parser: `Summary Parser -> Executive Summary Agent`

14. **Add a Merge node** named **Merge Analysis Results**.
    - Set number of inputs to `4`
    - Connect:
      - `Clause Extraction Agent -> Merge Analysis Results` input 0
      - `Risk Assessment Agent -> Merge Analysis Results` input 1
      - `Obligation Extraction Agent -> Merge Analysis Results` input 2
      - `Executive Summary Agent -> Merge Analysis Results` input 3

15. **Add a Code node** named **Calculate Risk Score**.
    - Paste the scoring logic that:
      - reads clauses, risks, obligations, and summary from merged inputs
      - maps clause categories to weights
      - maps risk levels to numeric values
      - calculates a weighted score
      - adds red flags for high-risk issues
      - flags missing critical clauses
      - flags low-confidence AI outputs
      - outputs summary data plus:
        - `overall_risk_score`
        - `score_breakdown`
        - `red_flags`
        - `recommendations`
        - `calculated_at`
    - Connect: `Merge Analysis Results -> Calculate Risk Score`

16. **Add a Set node** named **Prepare Contract Data**.
    - Map fields as follows:
      - `contract_id` = `={{ $now.toISO() }}-{{ $json.summary.contract_name.replace(/\s+/g, "-").toLowerCase() }}`
      - `contract_name` = `={{ $json.summary.contract_name }}`
      - `contract_type` = `={{ $json.summary.contract_type }}`
      - `execution_date` = `={{ $json.summary.execution_date }}`
      - `expiry_date` = `={{ $json.summary.expiry_date }}`
      - `value` = `={{ $json.summary.value }}`
      - `status` = `draft`
      - `uploaded_by` = your user ID/email source
      - `upload_date` = `={{ $now.toISO() }}`
      - `document_hash` = `={{ $json.summary.contract_name.length }}`
      - `clauses` = `={{ $json.clauses }}`
      - `risks` = `={{ $json.risks }}`
      - `obligations` = `={{ $json.obligations }}`
      - `overall_risk_score` = `={{ $json.overall_risk_score }}`
      - `score_breakdown` = `={{ $json.score_breakdown }}`
      - `red_flags` = `={{ $json.red_flags }}`
      - `recommendations` = `={{ $json.recommendations }}`
      - `calculated_at` = `={{ $json.calculated_at }}`
    - Connect: `Calculate Risk Score -> Prepare Contract Data`

17. **Create PostgreSQL credentials** in n8n.
    - Provide host, port, database, user, password, SSL settings as needed.

18. **Add a Postgres node** named **Insert Contract**.
    - Schema: `public`
    - Table: `contracts`
    - Insert columns:
      - `id`
      - `contract_name`
      - `contract_type`
      - `execution_date`
      - `expiry_date`
      - `value`
      - `status`
      - `uploaded_by`
      - `upload_date`
      - `document_hash`
      - `created_at`
    - Map from the prepared fields.
    - Connect: `Prepare Contract Data -> Insert Contract`

19. **Add a Postgres node** named **Insert Clauses**.
    - Schema: `public`
    - Table: `contract_clauses`
    - Insert:
      - `contract_id`
      - `clause_type`
      - `clause_text`
      - `ai_confidence`
    - Set query batching to `single`
    - Connect: `Insert Contract -> Insert Clauses`

20. **Add a Postgres node** named **Insert Obligations**.
    - Schema: `public`
    - Table: `contract_obligations`
    - Insert:
      - `contract_id`
      - `obligation_text`
      - `obligation_type`
      - `due_date`
      - `responsible_party`
      - `status`
    - Connect: `Insert Clauses -> Insert Obligations`

21. **Add a Postgres node** named **Insert Risk Scoring**.
    - Schema: `public`
    - Table: `risk_scoring`
    - Insert:
      - `contract_id`
      - `overall_risk_score`
      - `score_breakdown`
      - `red_flags`
      - `recommendations`
      - `calculated_at`
    - Connect: `Insert Obligations -> Insert Risk Scoring`

22. **Important implementation note:**  
    As designed, the workflow does **not** include intermediate item-splitting nodes between aggregate analysis data and `Insert Clauses` / `Insert Obligations`. To make those inserts work reliably, you should add transformation steps such as:
    - a Code or Item Lists node to expand `clauses[]` into one item per clause before `Insert Clauses`
    - a similar expansion for `obligations[]` before `Insert Obligations`
    - optionally serialize arrays/objects explicitly using `JSON.stringify(...)` for JSON/JSONB columns

23. **Add an If node** named **Check Risk Level**.
    - Condition:
      - left value: `={{ $json.overall_risk_score }}`
      - operator: `>=`
      - right value: `={{ $('Workflow Configuration').first().json.riskThreshold }}`
    - Strict type validation enabled
    - Connect: `Insert Risk Scoring -> Check Risk Level`

24. **Create Gmail credentials** in n8n.
    - Use OAuth2
    - Ensure send permissions are granted for the mailbox

25. **Add a Gmail node** named **Send Email Alert**.
    - Connect it to the true branch of `Check Risk Level`
    - To: `={{ $('Workflow Configuration').first().json.alertRecipients }}`
    - Subject: `=HIGH RISK CONTRACT ALERT: {{ $json.contract_name }}`
    - HTML body should include:
      - contract name
      - risk score
      - threshold
      - red flags as HTML list
      - recommendations as HTML list

26. **Create Slack credentials** in n8n.
    - Ensure the bot/app can post to the target channel
    - Invite the bot to the channel if necessary

27. **Add a Slack node** named **Send Slack Alert**.
    - Connect it to the true branch of `Check Risk Level`
    - Target type: `channel`
    - Channel ID: `={{ $('Workflow Configuration').first().json.slackChannel }}`
    - Message text should include:
      - warning emoji/title
      - contract name
      - risk score and threshold
      - numbered red flags
      - numbered recommendations

28. **Optionally add sticky notes** to match the visual layout:
    - Input Layer
    - Text Extraction
    - Clause Analysis
    - Risk Analysis
    - Obligation Tracking
    - Executive Summary
    - AI Model
    - Data Aggregation
    - Risk Scoring
    - Data Storage
    - Risk Evaluation
    - Alert System
    - General “How it works” note

29. **Prepare the database schema** before activation. At minimum, create:
    - `public.contracts`
    - `public.contract_clauses`
    - `public.contract_obligations`
    - `public.risk_scoring`

30. **Recommended schema improvements before production use:**
    - Use a real document hash instead of contract name length
    - Store arrays/objects in `JSONB` columns where appropriate
    - Add a response node for the webhook because the webhook is configured in `responseNode` mode
    - Add OCR support for scanned PDFs if needed
    - Add retries or error handling for LLM and notification nodes
    - Align the summary prompt with the parser schema so `execution_date`, `expiry_date`, and `value` are consistently returned

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow analyzes uploaded contracts using AI to extract clauses, assess risks, track obligations, and generate a structured summary. | General workflow purpose |
| After a contract is uploaded, the PDF text is extracted and processed by multiple AI agents in parallel. These agents identify clauses, evaluate risks, extract obligations, and generate an executive summary. | Processing flow |
| The workflow calculates an overall risk score using weighted logic and flags high-risk contracts. All results are stored in a database, including clauses, obligations, and risk insights. | Scoring and storage |
| If the risk score exceeds the defined threshold, alerts are automatically sent via email and Slack for immediate review. | Alerting behavior |
| Configure webhook for contract upload. | Setup step |
| Add AI model credentials (Anthropic/OpenAI). | Setup step |
| Set Postgres database connection. | Setup step |
| Configure Slack and email alerts. | Setup step |
| Adjust risk threshold and alert settings. | Setup step |

## Additional Implementation Notes

- The workflow has **one execution entry point**: `Contract Upload Webhook`.
- It uses **no sub-workflows**.
- The four AI analysis branches are executed **in parallel** after PDF text extraction.
- The current design has a likely **data-shape gap** between aggregate analysis output and relational inserts for clauses and obligations.
- The webhook is configured with **response node mode**, but no explicit response node exists in the workflow. This should be corrected for reliable API behavior.