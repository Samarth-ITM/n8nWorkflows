Analyze contract risk from PDFs with OpenAI, Supabase and Slack alerts

https://n8nworkflows.xyz/workflows/analyze-contract-risk-from-pdfs-with-openai--supabase-and-slack-alerts-13875


# Analyze contract risk from PDFs with OpenAI, Supabase and Slack alerts

# 1. Workflow Overview

This workflow analyzes uploaded contract documents, assesses legal risk with OpenAI, stores the results in Supabase, and sends notifications through Slack and Gmail.

Its main use cases are:
- Legal review intake for uploaded contracts
- Fast triage of vendor, procurement, employment, and partnership agreements
- Risk escalation for high-risk contracts
- Structured archiving of analysis results for later lookup

The workflow is organized into the following logical blocks.

## 1.1 Input Reception and Configuration
The workflow starts either from a public form upload or from a manual test trigger. A central configuration node defines shared parameters such as risk threshold, model name, Slack channels, and email behavior.

## 1.2 Duplicate Detection
Before spending AI tokens, the workflow checks Supabase for an existing record with the same filename/contract name. If a match is found, it stops further processing and returns a completion message.

## 1.3 Document Extraction and Normalization
If the contract is new, the workflow either extracts text from the uploaded PDF or injects a long sample contract for test mode. A code node then normalizes and truncates text, merges metadata, and prepares fields for AI prompts.

## 1.4 AI Two-Pass Analysis
The workflow calls OpenAI twice:
- Pass 1 performs structured classification
- Pass 2 performs deeper legal risk analysis

Both responses are parsed into structured JSON, with fallback defaults if parsing fails.

## 1.5 Report Generation, Persistence, and Notification Routing
The workflow builds a Slack Block Kit report and HTML email, inserts the result into Supabase, and routes execution depending on the computed risk score. High-risk items trigger Slack alerting and email; low-risk items trigger a Slack summary.

## 1.6 User Completion and Error Handling
At the end, the workflow returns a completion message to the submitter. Separately, an error trigger catches workflow failures and posts an alert to an admin Slack channel.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Configuration

### Overview
This block accepts either a real user submission through an n8n form or a developer-triggered manual run. It then injects global configuration values used throughout the workflow.

### Nodes Involved
- Upload Contract
- Manual Test
- Config

### Node Details

#### Upload Contract
- **Type and role:** `n8n-nodes-base.formTrigger`; public entry point for end users.
- **Configuration choices:**
  - Form title: `Contract AI Analysis — Upload & Analyze`
  - Accepts one file input labeled `Contract PDF`
  - Allowed file types: `.pdf,.docx`
  - Additional metadata fields:
    - Contract Type
    - Contract Name
    - Counterparty Name
    - Your Name
    - Your Email
    - Department
- **Key expressions or variables used:** none in the node itself.
- **Input and output connections:**
  - No upstream input; entry node
  - Output → `Config`
- **Version-specific requirements:** Form Trigger v2.2.
- **Edge cases / failures:**
  - File upload may be missing despite required form rules if malformed request hits webhook
  - `.docx` is accepted by the form, but downstream extraction is configured only for PDF, so DOCX uploads are likely to fail or produce no usable text
  - Invalid email formatting may be blocked by form validation
- **Sub-workflow reference:** none

#### Manual Test
- **Type and role:** `n8n-nodes-base.manualTrigger`; development/testing entry point.
- **Configuration choices:** default manual trigger.
- **Key expressions or variables used:** none
- **Input and output connections:**
  - No upstream input; entry node
  - Output → `Config`
- **Version-specific requirements:** v1
- **Edge cases / failures:**
  - Produces no data itself; later nodes rely on fallback/sample handling
- **Sub-workflow reference:** none

#### Config
- **Type and role:** `n8n-nodes-base.set`; central configuration store.
- **Configuration choices:**
  - `RISK_THRESHOLD` = `70`
  - `SLACK_CHANNEL` = `#legal-alerts`
  - `ADMIN_SLACK_CHANNEL` = `#n8n-errors`
  - `AI_MODEL` = `gpt-4o-mini`
  - `ALERT_EMAIL` = `user@example.com`
  - `ENABLE_EMAIL` = `true`
  - `CONTRACT_LANG` = `en`
- **Key expressions or variables used:** values are later read from downstream JSON or by cross-node lookup.
- **Input and output connections:**
  - Input ← `Upload Contract`, `Manual Test`
  - Output → `Check Duplicate`
- **Version-specific requirements:** Set node v3.4
- **Edge cases / failures:**
  - This node overwrites/defines only the listed fields, but preserves incoming data in normal Set-node behavior when assignments are configured this way in current n8n versions
  - Wrong threshold or wrong channels will alter routing/alerts
- **Sub-workflow reference:** none

---

## 2.2 Duplicate Detection

### Overview
This block queries Supabase REST API to see whether a contract with the same filename/name was already analyzed. If found, the workflow exits early with a completion message.

### Nodes Involved
- Check Duplicate
- Already Exists?
- Already Analyzed

### Node Details

#### Check Duplicate
- **Type and role:** `n8n-nodes-base.httpRequest`; direct REST lookup against Supabase.
- **Configuration choices:**
  - Method: GET-style request with query parameters
  - URL: `{{$env.SUPABASE_URL}}/rest/v1/contract_analyses`
  - Query parameters:
    - `filename = eq.{{ $json.filename ?? $json['Contract Name'] ?? 'unknown' }}`
    - `select = id,filename,overall_risk_score,status,analyzed_at`
    - `limit = 1`
  - Headers:
    - `apikey = {{$env.SUPABASE_SERVICE_KEY}}`
    - `Prefer = return=representation`
  - Authentication type is set to predefined credential type using an OpenAI credential slot, but in practice the request relies on custom header/API key values
  - `onError = continueRegularOutput`
- **Key expressions or variables used:**
  - `$env.SUPABASE_URL`
  - `$env.SUPABASE_SERVICE_KEY`
  - `$json.filename`
  - `$json['Contract Name']`
- **Input and output connections:**
  - Input ← `Config`
  - Output → `Already Exists?`
- **Version-specific requirements:** HTTP Request v4.2
- **Edge cases / failures:**
  - If environment variables are not set, request fails
  - If `filename` was lost upstream, lookup may default to `unknown`
  - Using exact filename match is weak deduplication; renamed copies bypass the check
  - `continueRegularOutput` means failed requests may still pass empty/error-like output downstream, causing the workflow to treat failures as “not duplicate”
  - The credential type is semantically mismatched; this can confuse maintainers
- **Sub-workflow reference:** none

#### Already Exists?
- **Type and role:** `n8n-nodes-base.if`; decides whether Supabase returned an existing record.
- **Configuration choices:**
  - Condition: `Array.isArray($json) ? $json.length : 0 > 0`
- **Key expressions or variables used:**
  - `{{ Array.isArray($json) ? $json.length : 0 }}`
- **Input and output connections:**
  - Input ← `Check Duplicate`
  - True → `Already Analyzed`
  - False → `Has File?`
- **Version-specific requirements:** IF node v2
- **Edge cases / failures:**
  - Depends on Supabase HTTP response being an array
  - If API errors produce a non-array payload, it resolves to 0 and continues as if no duplicate exists
- **Sub-workflow reference:** none

#### Already Analyzed
- **Type and role:** `n8n-nodes-base.form`; terminates the form flow with a completion screen.
- **Configuration choices:**
  - Operation: completion
  - Title: `Already Analyzed`
  - Message: tells user the contract already exists and to check Supabase
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input ← `Already Exists?` true branch
  - No downstream output
- **Version-specific requirements:** Form node v2.4
- **Edge cases / failures:**
  - User sees no direct existing analysis details, only a generic completion
  - For manual runs, this node is not especially useful as there is no form user waiting
- **Sub-workflow reference:** none

---

## 2.3 Document Extraction and Normalization

### Overview
This block decides whether to process a real uploaded file or use synthetic sample data. It then extracts or assembles contract text, truncates it, and maps metadata into a normalized structure for AI analysis.

### Nodes Involved
- Has File?
- Extract PDF Text
- Sample Data
- Prepare Text

### Node Details

#### Has File?
- **Type and role:** `n8n-nodes-base.if`; determines whether binary file data exists.
- **Configuration choices:**
  - Condition: `($binary || {}).data !== undefined`
- **Key expressions or variables used:**
  - `{{ ($binary || {}).data !== undefined }}`
- **Input and output connections:**
  - Input ← `Already Exists?` false branch
  - True → `Extract PDF Text`
  - False → `Sample Data`
- **Version-specific requirements:** IF node v2
- **Edge cases / failures:**
  - Assumes uploaded binary property is named `data`
  - If Form Trigger stores uploaded file under another binary property name, this check may fail
  - Manual test always falls through to sample data
- **Sub-workflow reference:** none

#### Extract PDF Text
- **Type and role:** `n8n-nodes-base.extractFromFile`; extracts text from uploaded PDF binary.
- **Configuration choices:**
  - Operation: `pdf`
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input ← `Has File?` true branch
  - Output → `Prepare Text`
- **Version-specific requirements:** Extract From File v1.1
- **Edge cases / failures:**
  - DOCX is accepted upstream but this node is not configured for DOCX extraction
  - Scanned/image-only PDFs may produce little or no text
  - Corrupted or encrypted PDFs may fail
- **Sub-workflow reference:** none

#### Sample Data
- **Type and role:** `n8n-nodes-base.set`; injects a full test contract and metadata.
- **Configuration choices:**
  - Adds:
    - `contractText`
    - `filename`
    - `Contract Type`
    - `Counterparty Name`
    - `Your Name`
    - `Your Email`
    - `Department`
  - Test sample is an MSA with indemnity, liability, confidentiality, GDPR, governing law, and force majeure clauses
- **Key expressions or variables used:** none
- **Input and output connections:**
  - Input ← `Has File?` false branch
  - Output → `Prepare Text`
- **Version-specific requirements:** Set node v3.4
- **Edge cases / failures:**
  - Good for deterministic testing
  - May hide real extraction issues because it bypasses file processing completely
- **Sub-workflow reference:** none

#### Prepare Text
- **Type and role:** `n8n-nodes-base.code`; normalizes extracted content and metadata into one JSON payload.
- **Configuration choices (interpreted):**
  - Reads first incoming item
  - Looks for text in:
    - `contractText`
    - `text`
    - `data`
    - `pages[]`
    - any long string field over 100 chars as fallback
  - Truncates main contract text to 12,000 characters
  - Also creates a 4,000-character short version for classification
  - Escapes both text variants with `JSON.stringify(...).slice(1, -1)` for safer prompt embedding
  - Pulls config from `$('Config').first().json`
  - Maps form fields into normalized keys:
    - `filename`
    - `contract_type`
    - `counterparty`
    - `submitter_name`
    - `submitter_email`
    - `department`
- **Key expressions or variables used:**
  - `$('Config').first()?.json`
  - `item.json?.contractText`
  - `item.json?.text`
  - `item.json?.data`
  - `item.json?.pages`
- **Input and output connections:**
  - Input ← `Extract PDF Text`, `Sample Data`
  - Output → `AI Pass 1 - Classify`
- **Version-specific requirements:** Code node v2
- **Edge cases / failures:**
  - Cross-node access to `Config` assumes the node always executed
  - Long contracts are truncated; risk analysis may miss important clauses beyond 12k chars
  - `filename` is populated from `Contract Name` first, not the uploaded file’s original name
  - If extraction yields almost no text, AI output quality will degrade sharply
- **Sub-workflow reference:** none

---

## 2.4 AI Two-Pass Analysis

### Overview
This block sends normalized contract text to OpenAI in two phases: classification first, then detailed risk analysis. The outputs are parsed into structured fields with safe fallback values if the model returns malformed JSON.

### Nodes Involved
- AI Pass 1 - Classify
- Parse Classification
- AI Pass 2 - Deep Risk
- Parse Risk Analysis

### Node Details

#### AI Pass 1 - Classify
- **Type and role:** `n8n-nodes-base.httpRequest`; OpenAI Chat Completions call for metadata extraction.
- **Configuration choices:**
  - POST to `https://api.openai.com/v1/chat/completions`
  - Model: `{{$json.AI_MODEL ?? 'gpt-4o-mini'}}`
  - Requests `response_format: { type: "json_object" }`
  - System prompt instructs the model to act as a legal document classifier and respond only with valid JSON
  - User prompt demands exact output fields:
    - `contract_type`
    - `parties`
    - `effective_date`
    - `expiration_date`
    - `auto_renewal`
    - `jurisdiction`
    - `governing_law`
    - `language`
    - `estimated_value`
  - Includes form-provided contract type and counterparty as context
  - Uses shortened escaped contract text
  - `temperature = 0.1`
  - `max_tokens = 500`
  - Retry enabled with 2 tries and 3-second wait
- **Key expressions or variables used:**
  - `$json.AI_MODEL`
  - `$json.CONTRACT_LANG`
  - `$json.contract_type`
  - `$json.counterparty`
  - `$json.contractTextShortSafe`
- **Input and output connections:**
  - Input ← `Prepare Text`
  - Output → `Parse Classification`
- **Version-specific requirements:** HTTP Request v4.2
- **Edge cases / failures:**
  - OpenAI auth failure, quota exhaustion, or rate limiting
  - Model may still return malformed JSON despite response_format request
  - Shortened text may omit party/jurisdiction data if located later in the contract
- **Sub-workflow reference:** none

#### Parse Classification
- **Type and role:** `n8n-nodes-base.code`; parses OpenAI classification result and merges it with prepared text data.
- **Configuration choices (interpreted):**
  - Reads `choices[0].message.content`
  - Strips code fences like ```json
  - `JSON.parse(...)`
  - On parse failure, falls back to:
    - `contract_type: 'Unknown'`
    - `parties: []`
    - `jurisdiction: 'Unknown'`
  - Merges parsed classification into the prior normalized payload from `Prepare Text`
- **Key expressions or variables used:**
  - `$input.first().json`
  - `$('Prepare Text').first()?.json`
- **Input and output connections:**
  - Input ← `AI Pass 1 - Classify`
  - Output → `AI Pass 2 - Deep Risk`
- **Version-specific requirements:** Code node v2
- **Edge cases / failures:**
  - If OpenAI response shape changes, parser breaks
  - Fallback object is intentionally minimal, so downstream prompts may lose context
- **Sub-workflow reference:** none

#### AI Pass 2 - Deep Risk
- **Type and role:** `n8n-nodes-base.httpRequest`; second OpenAI call for substantive risk analysis.
- **Configuration choices:**
  - POST to `https://api.openai.com/v1/chat/completions`
  - Same model source as pass 1
  - JSON response enforced via `response_format`
  - System prompt frames the model as a senior legal risk analyst
  - User prompt demands exact output fields:
    - `key_clauses`
    - `overall_risk_score`
    - `risk_level`
    - `executive_summary`
    - `top_risks`
    - `missing_clauses`
    - `obligations`
    - `negotiation_points`
    - `compliance_flags`
  - Injects classification context:
    - contract type
    - parties
    - jurisdiction
  - Focus areas explicitly specified:
    - indemnification
    - liability caps
    - termination penalties
    - IP ownership
    - data protection
    - warranty
    - non-compete
    - auto-renewal
    - governing law
    - force majeure
  - Uses full escaped contract text
  - `temperature = 0.15`
  - `max_tokens = 2500`
  - Retry enabled with 2 tries and 3-second wait
- **Key expressions or variables used:**
  - `$json.AI_MODEL`
  - `$json.CONTRACT_LANG`
  - `$json.classification.contract_type`
  - `($json.classification.parties ?? []).map(...)`
  - `$json.classification.jurisdiction`
  - `$json.contractTextSafe`
- **Input and output connections:**
  - Input ← `Parse Classification`
  - Output → `Parse Risk Analysis`
- **Version-specific requirements:** HTTP Request v4.2
- **Edge cases / failures:**
  - Large contracts truncated to 12k chars may lead to incomplete risk findings
  - Malformed JSON or partial output can still occur
  - Token limits may still truncate response if prompt is too large
- **Sub-workflow reference:** none

#### Parse Risk Analysis
- **Type and role:** `n8n-nodes-base.code`; normalizes and safeguards the AI risk output.
- **Configuration choices (interpreted):**
  - Parses `choices[0].message.content`
  - Removes code fences
  - On parse failure, sets fallback analysis:
    - `overall_risk_score = 50`
    - `risk_level = MEDIUM`
    - `executive_summary = AI parse error — manual review recommended`
    - `key_clauses = []`
    - `top_risks = ['Parse error']`
  - Clamps score into 0–100
  - Recomputes `risk_level` from numeric score:
    - `>= 85` → `CRITICAL`
    - `>= 70` → `HIGH`
    - `>= 40` → `MEDIUM`
    - otherwise `LOW`
  - Ensures arrays exist for:
    - `key_clauses`
    - `top_risks`
    - `missing_clauses`
    - `obligations`
    - `negotiation_points`
    - `compliance_flags`
  - Creates `raw_text_preview` from first 500 chars
  - Sets `status`:
    - `flagged` if score >= threshold
    - else `analyzed`
- **Key expressions or variables used:**
  - `$('Parse Classification').first()?.json`
  - `prev.RISK_THRESHOLD`
- **Input and output connections:**
  - Input ← `AI Pass 2 - Deep Risk`
  - Output → `Build Report`
- **Version-specific requirements:** Code node v2
- **Edge cases / failures:**
  - If classification payload is missing, merged context may be incomplete
  - Model-provided textual risk level is ignored in favor of score-based recalculation
- **Sub-workflow reference:** none

---

## 2.5 Report Generation, Persistence, and Notification Routing

### Overview
This block converts structured AI output into Slack and HTML report formats, stores the analysis in Supabase, and routes the item to the correct notification path based on the risk threshold.

### Nodes Involved
- Build Report
- Supabase Insert
- High Risk?
- Slack Alert - High Risk
- Gmail - Risk Report
- Slack Summary - Low Risk

### Node Details

#### Build Report
- **Type and role:** `n8n-nodes-base.code`; presentation-layer formatter for Slack and email.
- **Configuration choices (interpreted):**
  - Builds Slack Block Kit JSON including:
    - header
    - summary section
    - executive summary
    - optional top risks
    - optional key clauses
    - optional missing clauses
    - optional negotiation points
    - final context line
  - Builds styled HTML email report with:
    - color-coded risk banner
    - contract metadata table
    - executive summary
    - top risks
    - key clauses table
    - missing clauses
    - negotiation points
    - compliance flags
  - Limits display counts:
    - key clauses: top 5
    - top risks: top 5
    - missing clauses: top 3
    - negotiation points: top 3
    - compliance flags: top 3
  - Creates:
    - `slackBlocks` as a JSON string
    - `emailHtml`
    - `emailSubject`
- **Key expressions or variables used:** operates on current JSON only.
- **Input and output connections:**
  - Input ← `Parse Risk Analysis`
  - Output → `Supabase Insert`
- **Version-specific requirements:** Code node v2
- **Edge cases / failures:**
  - HTML is built from AI content without escaping, so malformed text could break rendering
  - Slack blocks are only created as a string; Slack node must be configured to actually use them, which is not visible here
- **Sub-workflow reference:** none

#### Supabase Insert
- **Type and role:** `n8n-nodes-base.supabase`; persists the final analysis.
- **Configuration choices:**
  - Table: `contract_analyses`
  - No explicit field mapping is shown in the JSON excerpt, so insertion likely depends on node defaults/current item fields
- **Key expressions or variables used:** none shown directly
- **Input and output connections:**
  - Input ← `Build Report`
  - Output → `High Risk?`
- **Version-specific requirements:** Supabase node v1
- **Edge cases / failures:**
  - Supabase credentials must be set
  - Table schema must exist first
  - If the node is not configured to map fields explicitly, behavior may vary by node version and may fail if unsupported fields are present
  - JSONB columns require serializable objects/arrays
- **Sub-workflow reference:** none

#### High Risk?
- **Type and role:** `n8n-nodes-base.if`; routes by threshold comparison.
- **Configuration choices:**
  - Condition: `overall_risk_score >= RISK_THRESHOLD`
- **Key expressions or variables used:**
  - `{{ $json.overall_risk_score }}`
  - `{{ $json.RISK_THRESHOLD ?? 70 }}`
- **Input and output connections:**
  - Input ← `Supabase Insert`
  - True → `Slack Alert - High Risk`, `Gmail - Risk Report`
  - False → `Slack Summary - Low Risk`
- **Version-specific requirements:** IF node v2
- **Edge cases / failures:**
  - If Supabase Insert changes the payload shape and removes original fields, condition may fail
- **Sub-workflow reference:** none

#### Slack Alert - High Risk
- **Type and role:** `n8n-nodes-base.slack`; intended to send a high-priority Slack alert.
- **Configuration choices:**
  - Operation: `create`
  - No message/channel/block configuration is visible in the provided JSON
- **Key expressions or variables used:** none visible
- **Input and output connections:**
  - Input ← `High Risk?` true branch
  - Output → `Form Ending`
- **Version-specific requirements:** Slack node v2.4
- **Edge cases / failures:**
  - Likely incomplete unless channel/text/blocks are configured in hidden defaults or omitted export fields
  - Slack auth/scope issues may prevent posting
- **Sub-workflow reference:** none

#### Gmail - Risk Report
- **Type and role:** `n8n-nodes-base.gmail`; sends HTML email report.
- **Configuration choices:**
  - To:
    - if `ENABLE_EMAIL` true, send to `submitter_email`
    - else fallback to `ALERT_EMAIL`
    - else fallback to `legal@company.com`
  - Subject: `{{$json.emailSubject}}`
  - Message body: `{{$json.emailHtml}}`
  - `onError = continueRegularOutput`
- **Key expressions or variables used:**
  - `$json.ENABLE_EMAIL`
  - `$json.submitter_email`
  - `$json.ALERT_EMAIL`
  - `$json.emailHtml`
  - `$json.emailSubject`
- **Input and output connections:**
  - Input ← `High Risk?` true branch
  - No downstream connection
- **Version-specific requirements:** Gmail node v2.1
- **Edge cases / failures:**
  - Empty sendTo may cause send failure when email is disabled
  - HTML rendering may vary by mail client
  - Because errors are continued, email failure does not stop form completion
- **Sub-workflow reference:** none

#### Slack Summary - Low Risk
- **Type and role:** `n8n-nodes-base.slack`; intended to send a lower-priority summary message.
- **Configuration choices:**
  - Operation: `create`
  - No visible message/channel configuration
- **Key expressions or variables used:** none visible
- **Input and output connections:**
  - Input ← `High Risk?` false branch
  - Output → `Form Ending`
- **Version-specific requirements:** Slack node v2.4
- **Edge cases / failures:**
  - Same as high-risk Slack node: appears underconfigured in the export shown
- **Sub-workflow reference:** none

---

## 2.6 User Completion and Error Handling

### Overview
This final block completes the user-facing form interaction and defines an out-of-band error notification path for operational monitoring.

### Nodes Involved
- Form Ending
- Error Trigger
- Slack Admin — Error

### Node Details

#### Form Ending
- **Type and role:** `n8n-nodes-base.form`; returns a completion screen to the form submitter.
- **Configuration choices:**
  - Dynamic title:
    - `Analysis Complete — {{ risk_level }}`
  - Dynamic message includes:
    - contract filename
    - risk score
    - risk level
    - contract type
    - executive summary
    - note that report is stored in Supabase
    - optional email confirmation if enabled
- **Key expressions or variables used:**
  - `$json.risk_level`
  - `$json.filename`
  - `$json.overall_risk_score`
  - `$json.classification?.contract_type`
  - `$json.contract_type`
  - `$json.executive_summary`
  - `$json.ENABLE_EMAIL`
  - `$json.submitter_email`
  - `$json.ALERT_EMAIL`
- **Input and output connections:**
  - Input ← `Slack Alert - High Risk`, `Slack Summary - Low Risk`
  - No downstream output
- **Version-specific requirements:** Form node v2.4
- **Edge cases / failures:**
  - Gmail branch is not connected to this node, so email completion does not influence user flow
  - If Slack node fails and does not pass data, completion might not occur depending on Slack node error settings
- **Sub-workflow reference:** none

#### Error Trigger
- **Type and role:** `n8n-nodes-base.errorTrigger`; starts a separate execution when the workflow errors.
- **Configuration choices:** default error trigger.
- **Key expressions or variables used:** none shown
- **Input and output connections:**
  - No upstream input; autonomous error entry point
  - Output → `Slack Admin — Error`
- **Version-specific requirements:** v1
- **Edge cases / failures:**
  - Fires only for workflow-level failures, not necessarily for nodes set to continue on error
- **Sub-workflow reference:** none

#### Slack Admin — Error
- **Type and role:** `n8n-nodes-base.slack`; intended admin error notification sender.
- **Configuration choices:**
  - Operation: `create`
  - `onError = continueRegularOutput`
  - No visible message/channel mapping in the export shown
- **Key expressions or variables used:** none visible
- **Input and output connections:**
  - Input ← `Error Trigger`
  - No downstream output
- **Version-specific requirements:** Slack node v2.4
- **Edge cases / failures:**
  - If not properly configured, errors may go unreported
  - Since it continues on error, secondary Slack failure will not create another exception loop
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Documentation note |  |  | ## Enterprise AI Contract Intelligence - Pro<br>**Who it's for:** Legal, Procurement, Compliance teams. ESN/agencies.<br>**What it does:**<br>1. Upload contract PDF via form (with metadata: type, department, counterparty)<br>2. Deduplication check against Supabase<br>3. AI Pass 1 - Classification (parties, dates, jurisdiction, contract type)<br>4. AI Pass 2 - Deep risk analysis (clauses, red flags, obligations, recommendations)<br>5. Build structured report (Slack Blocks + HTML email)<br>6. Store in Supabase with full metadata<br>7. Slack Block Kit alert (high risk) or summary (low risk)<br>8. Gmail HTML report to submitter (configurable)<br>9. Error handling with admin Slack channel<br>**Setup:** Run the SQL in the "Supabase Schema" sticky. Add credentials: OpenAI (Header Auth), Supabase, Slack, Gmail. Configure the Config node. |
| Supabase Schema | Sticky Note | Documentation note |  |  | **Supabase Schema** — Run in Supabase SQL Editor before use:<br>SQL shown for `contract_analyses` table and indexes. |
| Step 1 Ingest | Sticky Note | Documentation note |  |  | **Step 1 - Ingest**<br>Form upload with metadata (contract type, submitter, department, counterparty) or Manual test with sample data. Config node centralizes all settings. |
| Step 2 Deduplicate | Sticky Note | Documentation note |  |  | **Step 2 - Deduplicate**<br>Query Supabase for existing analysis with the same filename. If found, skip analysis and show previous result. Avoids wasting AI tokens on re-uploads. |
| Step 3 Extract | Sticky Note | Documentation note |  |  | **Step 3 - Extract & Prepare**<br>Extract text from PDF binary, normalize, truncate to 12k chars, merge with form metadata and config for downstream AI calls. |
| Step 4 AI Analysis | Sticky Note | Documentation note |  |  | **Step 4 - AI Two-Pass Analysis**<br>Pass 1: Quick classification (type, parties, dates, jurisdiction, ~300 tokens).<br>Pass 2: Deep risk analysis (clauses, obligations, red flags, recommendations, ~2000 tokens).<br>Split for cost efficiency and structured output. |
| Step 5 Report Alert | Sticky Note | Documentation note |  |  | **Step 5 - Report, Store & Alert**<br>Build Report generates Slack Block Kit JSON + HTML email body.<br>Supabase insert stores full analysis with metadata.<br>High risk → Slack Blocks alert + Gmail HTML report.<br>Low risk → Slack summary notification.<br>Form Ending shows completion to user. |
| Error Handling Note | Sticky Note | Documentation note |  |  | **Error Handling**<br>Error Trigger catches any workflow failure and sends a notification to the admin Slack channel (#n8n-errors). Includes workflow name, error message, and timestamp. |
| Upload Contract | Form Trigger | User file upload and metadata intake |  | Config | **Step 1 - Ingest**<br>Form upload with metadata (contract type, submitter, department, counterparty) or Manual test with sample data. Config node centralizes all settings. |
| Manual Test | Manual Trigger | Developer/manual entry point |  | Config | **Step 1 - Ingest**<br>Form upload with metadata (contract type, submitter, department, counterparty) or Manual test with sample data. Config node centralizes all settings. |
| Config | Set | Central workflow settings | Upload Contract, Manual Test | Check Duplicate | **Step 1 - Ingest**<br>Form upload with metadata (contract type, submitter, department, counterparty) or Manual test with sample data. Config node centralizes all settings. |
| Check Duplicate | HTTP Request | Supabase duplicate lookup | Config | Already Exists? | **Step 2 - Deduplicate**<br>Query Supabase for existing analysis with the same filename. If found, skip analysis and show previous result. Avoids wasting AI tokens on re-uploads. |
| Already Exists? | If | Branch on duplicate detection | Check Duplicate | Already Analyzed, Has File? | **Step 2 - Deduplicate**<br>Query Supabase for existing analysis with the same filename. If found, skip analysis and show previous result. Avoids wasting AI tokens on re-uploads. |
| Already Analyzed | Form | User completion for duplicate case | Already Exists? |  | **Step 2 - Deduplicate**<br>Query Supabase for existing analysis with the same filename. If found, skip analysis and show previous result. Avoids wasting AI tokens on re-uploads. |
| Has File? | If | Decide uploaded file vs sample path | Already Exists? | Extract PDF Text, Sample Data | **Step 3 - Extract & Prepare**<br>Extract text from PDF binary, normalize, truncate to 12k chars, merge with form metadata and config for downstream AI calls. |
| Extract PDF Text | Extract From File | PDF text extraction | Has File? | Prepare Text | **Step 3 - Extract & Prepare**<br>Extract text from PDF binary, normalize, truncate to 12k chars, merge with form metadata and config for downstream AI calls. |
| Sample Data | Set | Inject sample contract for testing | Has File? | Prepare Text | **Step 3 - Extract & Prepare**<br>Extract text from PDF binary, normalize, truncate to 12k chars, merge with form metadata and config for downstream AI calls. |
| Prepare Text | Code | Normalize text and metadata | Extract PDF Text, Sample Data | AI Pass 1 - Classify | **Step 3 - Extract & Prepare**<br>Extract text from PDF binary, normalize, truncate to 12k chars, merge with form metadata and config for downstream AI calls. |
| AI Pass 1 - Classify | HTTP Request | OpenAI classification call | Prepare Text | Parse Classification | **Step 4 - AI Two-Pass Analysis**<br>Pass 1: Quick classification (type, parties, dates, jurisdiction, ~300 tokens).<br>Pass 2: Deep risk analysis (clauses, obligations, red flags, recommendations, ~2000 tokens).<br>Split for cost efficiency and structured output. |
| Parse Classification | Code | Parse classification JSON | AI Pass 1 - Classify | AI Pass 2 - Deep Risk | **Step 4 - AI Two-Pass Analysis**<br>Pass 1: Quick classification (type, parties, dates, jurisdiction, ~300 tokens).<br>Pass 2: Deep risk analysis (clauses, obligations, red flags, recommendations, ~2000 tokens).<br>Split for cost efficiency and structured output. |
| AI Pass 2 - Deep Risk | HTTP Request | OpenAI risk analysis call | Parse Classification | Parse Risk Analysis | **Step 4 - AI Two-Pass Analysis**<br>Pass 1: Quick classification (type, parties, dates, jurisdiction, ~300 tokens).<br>Pass 2: Deep risk analysis (clauses, obligations, red flags, recommendations, ~2000 tokens).<br>Split for cost efficiency and structured output. |
| Parse Risk Analysis | Code | Parse and normalize risk JSON | AI Pass 2 - Deep Risk | Build Report | **Step 4 - AI Two-Pass Analysis**<br>Pass 1: Quick classification (type, parties, dates, jurisdiction, ~300 tokens).<br>Pass 2: Deep risk analysis (clauses, obligations, red flags, recommendations, ~2000 tokens).<br>Split for cost efficiency and structured output. |
| Build Report | Code | Generate Slack blocks and HTML email | Parse Risk Analysis | Supabase Insert | **Step 5 - Report, Store & Alert**<br>Build Report generates Slack Block Kit JSON + HTML email body.<br>Supabase insert stores full analysis with metadata.<br>High risk → Slack Blocks alert + Gmail HTML report.<br>Low risk → Slack summary notification.<br>Form Ending shows completion to user. |
| Supabase Insert | Supabase | Store full analysis record | Build Report | High Risk? | **Step 5 - Report, Store & Alert**<br>Build Report generates Slack Block Kit JSON + HTML email body.<br>Supabase insert stores full analysis with metadata.<br>High risk → Slack Blocks alert + Gmail HTML report.<br>Low risk → Slack summary notification.<br>Form Ending shows completion to user. |
| High Risk? | If | Route high vs low risk notifications | Supabase Insert | Slack Alert - High Risk, Gmail - Risk Report, Slack Summary - Low Risk | **Step 5 - Report, Store & Alert**<br>Build Report generates Slack Block Kit JSON + HTML email body.<br>Supabase insert stores full analysis with metadata.<br>High risk → Slack Blocks alert + Gmail HTML report.<br>Low risk → Slack summary notification.<br>Form Ending shows completion to user. |
| Slack Alert - High Risk | Slack | Send high-risk Slack alert | High Risk? | Form Ending | **Step 5 - Report, Store & Alert**<br>Build Report generates Slack Block Kit JSON + HTML email body.<br>Supabase insert stores full analysis with metadata.<br>High risk → Slack Blocks alert + Gmail HTML report.<br>Low risk → Slack summary notification.<br>Form Ending shows completion to user. |
| Gmail - Risk Report | Gmail | Email HTML report | High Risk? |  | **Step 5 - Report, Store & Alert**<br>Build Report generates Slack Block Kit JSON + HTML email body.<br>Supabase insert stores full analysis with metadata.<br>High risk → Slack Blocks alert + Gmail HTML report.<br>Low risk → Slack summary notification.<br>Form Ending shows completion to user. |
| Slack Summary - Low Risk | Slack | Send low-risk Slack summary | High Risk? | Form Ending | **Step 5 - Report, Store & Alert**<br>Build Report generates Slack Block Kit JSON + HTML email body.<br>Supabase insert stores full analysis with metadata.<br>High risk → Slack Blocks alert + Gmail HTML report.<br>Low risk → Slack summary notification.<br>Form Ending shows completion to user. |
| Form Ending | Form | Final user completion screen | Slack Alert - High Risk, Slack Summary - Low Risk |  | **Step 5 - Report, Store & Alert**<br>Build Report generates Slack Block Kit JSON + HTML email body.<br>Supabase insert stores full analysis with metadata.<br>High risk → Slack Blocks alert + Gmail HTML report.<br>Low risk → Slack summary notification.<br>Form Ending shows completion to user. |
| Error Trigger | Error Trigger | Start error-notification path |  | Slack Admin — Error | **Error Handling**<br>Error Trigger catches any workflow failure and sends a notification to the admin Slack channel (#n8n-errors). Includes workflow name, error message, and timestamp. |
| Slack Admin — Error | Slack | Send admin failure alert | Error Trigger |  | **Error Handling**<br>Error Trigger catches any workflow failure and sends a notification to the admin Slack channel (#n8n-errors). Includes workflow name, error message, and timestamp. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   `Analyze contract risk from PDFs with OpenAI, Supabase and Slack alerts`

2. **Create a Form Trigger node** named `Upload Contract`.
   - Type: `Form Trigger`
   - Title: `Contract AI Analysis — Upload & Analyze`
   - Description: `Upload a contract PDF for AI-powered analysis. The system will extract key clauses, assess risk, and provide actionable recommendations.`
   - Add these form fields:
     1. File field: `Contract PDF`, required, accept `.pdf,.docx`
     2. Dropdown: `Contract Type`, required, options:
        - NDA
        - MSA
        - SOW / Statement of Work
        - Employment
        - SaaS / License
        - Consulting
        - Partnership
        - Other
     3. Text: `Contract Name`, required
     4. Text: `Counterparty Name`, required
     5. Text: `Your Name`, required
     6. Email: `Your Email`, required
     7. Dropdown: `Department`, options:
        - Legal
        - Procurement
        - Finance
        - Sales
        - Engineering
        - HR
        - Operations
        - Other

3. **Create a Manual Trigger node** named `Manual Test`.

4. **Create a Set node** named `Config`.
   - Add fields:
     - `RISK_THRESHOLD` → Number → `70`
     - `SLACK_CHANNEL` → String → `#legal-alerts`
     - `ADMIN_SLACK_CHANNEL` → String → `#n8n-errors`
     - `AI_MODEL` → String → `gpt-4o-mini`
     - `ALERT_EMAIL` → String → `user@example.com`
     - `ENABLE_EMAIL` → Boolean → `true`
     - `CONTRACT_LANG` → String → `en`

5. **Connect entry nodes to Config.**
   - `Upload Contract` → `Config`
   - `Manual Test` → `Config`

6. **Create an HTTP Request node** named `Check Duplicate`.
   - Method: GET/query-based request
   - URL:
     - `={{ $env.SUPABASE_URL }}/rest/v1/contract_analyses`
   - Enable query parameters:
     - `filename` → `=eq.{{ $json.filename ?? $json['Contract Name'] ?? 'unknown' }}`
     - `select` → `id,filename,overall_risk_score,status,analyzed_at`
     - `limit` → `1`
   - Enable headers:
     - `apikey` → `={{ $env.SUPABASE_SERVICE_KEY }}`
     - `Prefer` → `return=representation`
   - Set response format to JSON/default object handling
   - Set node error behavior to continue output on error
   - Connect `Config` → `Check Duplicate`

7. **Set up environment values** in n8n for:
   - `SUPABASE_URL`
   - `SUPABASE_SERVICE_KEY`

8. **Create an IF node** named `Already Exists?`
   - Condition:
     - Left value: `={{ Array.isArray($json) ? $json.length : 0 }}`
     - Operator: greater than
     - Right value: `0`
   - Connect `Check Duplicate` → `Already Exists?`

9. **Create a Form node** named `Already Analyzed`.
   - Operation: `Completion`
   - Title: `Already Analyzed`
   - Message: `This contract has already been analyzed. Check Supabase for the existing report.`
   - Connect true branch of `Already Exists?` → `Already Analyzed`

10. **Create an IF node** named `Has File?`
    - Condition:
      - Left value: `={{ ($binary || {}).data !== undefined }}`
      - Operator: boolean equals
      - Right value: `true`
    - Connect false branch of `Already Exists?` → `Has File?`

11. **Create an Extract From File node** named `Extract PDF Text`.
    - Operation: `PDF`
    - Connect true branch of `Has File?` → `Extract PDF Text`

12. **Create a Set node** named `Sample Data`.
    - Add a long `contractText` test string containing a sample contract
    - Add:
      - `filename` = `sample-msa-techcorp.pdf`
      - `Contract Type` = `MSA`
      - `Counterparty Name` = `CloudVendor Solutions Inc.`
      - `Your Name` = `Ahmed Test`
      - `Your Email` = `user@example.com`
      - `Department` = `Legal`
    - Connect false branch of `Has File?` → `Sample Data`

13. **Create a Code node** named `Prepare Text`.
    - Paste this logic conceptually:
      - Read first input item
      - Try to locate contract text in `contractText`, `text`, `data`, or `pages`
      - If no text, search for any long string field
      - Truncate full text to 12,000 chars
      - Create short version at 4,000 chars
      - Escape both strings for prompt safety
      - Merge values from `Config`
      - Map incoming metadata into:
        - `filename`
        - `contract_type`
        - `counterparty`
        - `submitter_name`
        - `submitter_email`
        - `department`
    - Use the exact code from the provided workflow if you want identical behavior.
    - Connect:
      - `Extract PDF Text` → `Prepare Text`
      - `Sample Data` → `Prepare Text`

14. **Create an HTTP Request node** named `AI Pass 1 - Classify`.
    - Method: `POST`
    - URL: `https://api.openai.com/v1/chat/completions`
    - Authentication: OpenAI-compatible header auth / OpenAI credential
    - JSON body mode
    - Use model from expression:
      - `{{ $json.AI_MODEL ?? 'gpt-4o-mini' }}`
    - Include:
      - `response_format: { "type": "json_object" }`
      - system prompt saying it is a legal document classifier
      - user prompt requiring exact classification JSON schema
      - second user message containing `{{ $json.contractTextShortSafe }}`
      - `temperature: 0.1`
      - `max_tokens: 500`
    - Enable retry on failure:
      - max tries: 2
      - wait between tries: 3000 ms
    - Connect `Prepare Text` → `AI Pass 1 - Classify`

15. **Configure OpenAI credentials** in n8n.
    - Add API key credential usable by HTTP Request nodes
    - Ensure authorization header is sent properly

16. **Create a Code node** named `Parse Classification`.
    - Logic:
      - Read `choices[0].message.content`
      - Remove markdown code fences
      - Parse JSON
      - On failure, fallback to minimal object
      - Merge with `Prepare Text` output
    - Connect `AI Pass 1 - Classify` → `Parse Classification`

17. **Create an HTTP Request node** named `AI Pass 2 - Deep Risk`.
    - Method: `POST`
    - URL: `https://api.openai.com/v1/chat/completions`
    - Authentication: same OpenAI credential
    - JSON body mode
    - Include:
      - same model expression
      - `response_format: { "type": "json_object" }`
      - system prompt framing the model as a senior legal risk analyst
      - user prompt requiring exact deep-risk JSON schema
      - focus areas list
      - contract classification context from previous node
      - second user message containing `{{ $json.contractTextSafe }}`
      - `temperature: 0.15`
      - `max_tokens: 2500`
    - Enable retries exactly as above
    - Connect `Parse Classification` → `AI Pass 2 - Deep Risk`

18. **Create a Code node** named `Parse Risk Analysis`.
    - Logic:
      - Parse OpenAI JSON
      - Fallback to safe defaults on parse error
      - Clamp score to 0–100
      - Recompute risk level from score
      - Ensure arrays exist
      - Set:
        - `raw_text_preview`
        - `status`
    - Connect `AI Pass 2 - Deep Risk` → `Parse Risk Analysis`

19. **Create a Code node** named `Build Report`.
    - Logic:
      - Generate a Slack Block Kit array
      - Generate HTML email content
      - Generate email subject
      - Store them into:
        - `slackBlocks`
        - `emailHtml`
        - `emailSubject`
    - Connect `Parse Risk Analysis` → `Build Report`

20. **Create a Supabase node** named `Supabase Insert`.
    - Operation: insert/create row
    - Table: `contract_analyses`
    - Map relevant fields from the JSON payload into table columns:
      - `filename`
      - `contract_type`
      - `jurisdiction`
      - `submitter_name`
      - `submitter_email`
      - `department`
      - `counterparty`
      - `raw_text_preview`
      - `parties`
      - `key_clauses`
      - `overall_risk_score`
      - `risk_level`
      - `executive_summary`
      - `top_risks`
      - `recommendations` if you derive them, or leave null
      - `classification`
      - `status`
    - Connect `Build Report` → `Supabase Insert`

21. **Configure Supabase credentials** in n8n.
    - Use a service-role or properly scoped API key
    - Ensure the `contract_analyses` table exists before testing

22. **Run the SQL schema in Supabase SQL Editor**:
    - Create `contract_analyses`
    - Include all indexes from the sticky note
    - Ensure `gen_random_uuid()` is available

23. **Create an IF node** named `High Risk?`
    - Condition:
      - Left: `={{ $json.overall_risk_score }}`
      - Operator: greater than or equal
      - Right: `={{ $json.RISK_THRESHOLD ?? 70 }}`
    - Connect `Supabase Insert` → `High Risk?`

24. **Create a Slack node** named `Slack Alert - High Risk`.
    - Operation: create message
    - Configure it to post into the high-risk alert channel
    - Recommended channel expression:
      - `={{ $json.SLACK_CHANNEL || '#legal-alerts' }}`
    - Recommended text/blocks:
      - Use `{{$json.slackBlocks}}` as Block Kit payload if your Slack node supports blocks
      - Add a fallback plain text summary
    - Connect true branch of `High Risk?` → `Slack Alert - High Risk`

25. **Create a Gmail node** named `Gmail - Risk Report`.
    - To:
      - `={{ $json.ENABLE_EMAIL ? ($json.submitter_email || $json.ALERT_EMAIL || 'legal@company.com') : '' }}`
    - Subject:
      - `={{ $json.emailSubject }}`
    - Message body:
      - `={{ $json.emailHtml }}`
    - Enable continue on error
    - Connect true branch of `High Risk?` → `Gmail - Risk Report`

26. **Configure Gmail credentials** in n8n.
    - Use OAuth2
    - Make sure sending permissions are granted

27. **Create another Slack node** named `Slack Summary - Low Risk`.
    - Operation: create message
    - Configure it to post to the summary/legal channel
    - Use either a lighter Slack blocks payload or a plain text summary
    - Connect false branch of `High Risk?` → `Slack Summary - Low Risk`

28. **Create a Form node** named `Form Ending`.
    - Operation: completion
    - Title expression:
      - `={{ 'Analysis Complete — ' + ($json.risk_level ?? 'N/A') }}`
    - Message expression:
      - include filename, score, type, executive summary, Supabase note, and optional email confirmation
    - Connect:
      - `Slack Alert - High Risk` → `Form Ending`
      - `Slack Summary - Low Risk` → `Form Ending`

29. **Create an Error Trigger node** named `Error Trigger`.

30. **Create a Slack node** named `Slack Admin — Error`.
    - Operation: create message
    - Configure it to send to the admin channel such as `#n8n-errors`
    - Include workflow name, error message, execution URL if available, and timestamp
    - Set continue on error
    - Connect `Error Trigger` → `Slack Admin — Error`

31. **Configure Slack credentials** in n8n.
    - Use a bot token with permissions to post messages
    - Ensure the bot is a member of target channels

32. **Add optional sticky notes** for:
    - overview
    - Supabase schema
    - ingest
    - deduplication
    - extract
    - AI analysis
    - reporting/alerting
    - error handling

33. **Test the workflow in both modes.**
    - Manual run should go through `Sample Data`
    - Form submission should go through upload path
    - Confirm:
      - Supabase row insertion
      - risk routing
      - Slack behavior
      - email behavior
      - form completion screen

34. **Strongly recommended adjustments before production use:**
    - Restrict upload field to PDF only unless DOCX extraction is added
    - Make Slack nodes explicitly use `slackBlocks` and channel expressions
    - Verify Supabase Insert field mappings explicitly
    - Improve deduplication beyond filename alone, such as content hash
    - Add OCR for scanned PDFs if needed

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is labeled as **Enterprise AI Contract Intelligence - Pro** and targets Legal, Procurement, Compliance teams, plus ESN/agencies. | Overview sticky note |
| Required setup includes credentials for OpenAI, Supabase, Slack, and Gmail. | Overview sticky note |
| The database table required is `contract_analyses` with JSONB fields for parties, clauses, risks, and classification. | Supabase setup note |
| Deduplication is filename-based only. This is simple but not robust against renamed duplicate files. | Architectural note |
| The form allows `.docx`, but extraction is configured only for PDF. Add DOCX extraction or remove `.docx` from accepted file types. | Implementation note |
| The Slack nodes in the provided JSON do not show explicit message/block/channel parameters. Validate them manually in n8n before production use. | Implementation note |
| The error path is separate from the main flow and starts from `Error Trigger`. | Error-handling design |