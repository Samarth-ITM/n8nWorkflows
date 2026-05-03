Screen and score investment deals with AI using OpenAI, Gmail, and Telegram

https://n8nworkflows.xyz/workflows/screen-and-score-investment-deals-with-ai-using-openai--gmail--and-telegram-14239


# Screen and score investment deals with AI using OpenAI, Gmail, and Telegram

# 1. Workflow Overview

This workflow screens incoming investment deal submissions from two intake channels—Gmail and an authenticated webhook—then uses OpenAI twice: first to extract structured deal information, and second to score the opportunity against predefined investment criteria. Based on the AI verdict, it sends a Telegram notification and logs the deal to Google Sheets.

Typical use cases:
- Screening inbound startup pitches sent by email
- Accepting deal submissions from forms, partner portals, or APIs
- Standardizing deal triage before human review
- Creating a lightweight investment pipeline with alerts and scoring

## 1.1 Intake Reception

The workflow has **two entry points**:
- **New Email Received**: polls Gmail for unread messages and downloads attachments
- **Deal Submission Webhook**: accepts authenticated HTTP POST submissions

The webhook path responds immediately to the sender with a JSON acknowledgment, then continues internal processing.

## 1.2 Normalization and Content Assembly

Both intake paths are normalized into a common schema:
- sender identity
- subject
- content body
- attachments/document indicator
- received timestamp
- source channel

The workflow then builds a single `deal_text` string combining metadata and message content for AI analysis.

## 1.3 Validation

A gate checks whether the assembled `deal_text` is long enough to be meaningful. If not, the workflow stops with an error.

## 1.4 AI Extraction

The first OpenAI call extracts structured deal fields such as:
- company name
- industry
- funding stage
- business model
- revenue
- growth
- funding ask
- valuation
- location
- team size
- highlights
- red flags

## 1.5 AI Scoring

The second OpenAI call scores the deal on five weighted criteria:
- industry fit
- revenue stage
- growth trajectory
- team strength
- deal clarity

It also returns:
- overall score
- verdict (`PASS`, `REVIEW`, `REJECT`)
- one-line summary
- recommendation

## 1.6 Routing, Alerting, and Logging

The workflow routes by verdict:
- `PASS` → Telegram pass alert
- `REVIEW` → Telegram review notice
- everything else → Telegram reject log

All three routes converge into a Google Sheets append step for pipeline logging.

---

# 2. Block-by-Block Analysis

## 2.1 Intake & Validation

**Overview:**  
This block receives deal submissions from Gmail or an authenticated webhook, standardizes the data shape, builds a unified text representation, and rejects empty or near-empty submissions before AI usage.

**Nodes Involved:**  
- New Email Received
- Deal Submission Webhook
- Respond - Deal Received
- Normalize Email Data
- Normalize Webhook Data
- Build Deal Text
- Has Deal Content?
- Stop - No Deal Content

### New Email Received

- **Type and technical role:** `n8n-nodes-base.gmailTrigger`  
  Polling trigger that monitors Gmail for unread emails.
- **Configuration choices:**
  - Polls **every minute**
  - Only unread emails
  - Excludes spam/trash
  - Downloads attachments
  - `simple: false`, so it keeps richer Gmail payload structure
- **Key expressions or variables used:**  
  None directly in this node.
- **Input and output connections:**
  - Entry point node
  - Outputs to **Normalize Email Data**
- **Version-specific requirements:**  
  Uses `typeVersion: 1.2`; Gmail Trigger behavior depends on valid Gmail OAuth credentials and polling support in the n8n version.
- **Edge cases or potential failure types:**
  - Gmail OAuth failure or expired token
  - Trigger not firing if workflow is inactive
  - Message structure differences if Gmail API returns unusual sender data
  - Large attachments may affect performance/storage
- **Sub-workflow reference:**  
  None

### Deal Submission Webhook

- **Type and technical role:** `n8n-nodes-base.webhook`  
  HTTP entry point for external systems to submit deals.
- **Configuration choices:**
  - HTTP method: `POST`
  - Path: `deal-intake`
  - Authentication: `headerAuth`
  - Response mode: handled by a separate response node
- **Key expressions or variables used:**  
  Downstream logic references `{{$json.body...}}`
- **Input and output connections:**
  - Entry point node
  - Outputs to **Respond - Deal Received**
- **Version-specific requirements:**  
  Uses `typeVersion: 2`; header auth must be configured in workflow/webhook credentials.
- **Edge cases or potential failure types:**
  - Missing or invalid auth header
  - Wrong payload structure if body fields are absent
  - External callers using GET or wrong content type
  - Webhook URL differences between test and production mode
- **Sub-workflow reference:**  
  None

### Respond - Deal Received

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Sends the HTTP response back to the webhook caller.
- **Configuration choices:**
  - Responds with JSON
  - Body:
    ```json
    {"status": "received", "message": "Deal submitted for screening"}
    ```
- **Key expressions or variables used:**  
  Static JSON response body
- **Input and output connections:**
  - Input from **Deal Submission Webhook**
  - Output to **Normalize Webhook Data**
- **Version-specific requirements:**  
  Uses `typeVersion: 1.1`
- **Edge cases or potential failure types:**
  - Must be paired correctly with webhook response mode
  - If removed or misconfigured, webhook callers may hang or get wrong response behavior
- **Sub-workflow reference:**  
  None

### Normalize Email Data

- **Type and technical role:** `n8n-nodes-base.set`  
  Maps Gmail payload fields into a common intake schema.
- **Configuration choices:**
  Creates:
  - `source = "email"`
  - `sender_name = {{$json.from.value[0].name || $json.from.text || 'Unknown'}}`
  - `sender_email = {{$json.from.value[0].address || ''}}`
  - `subject = {{$json.subject || 'No Subject'}}`
  - `email_body = {{$json.textPlain || $json.text || $json.snippet || ''}}`
  - `has_attachments = {{$json.attachments ? $json.attachments.length > 0 : false}}`
  - `received_at = {{$now.toISO()}}`
  - `message_id = {{$json.id || ''}}`
- **Key expressions or variables used:**  
  `from.value[0]`, `textPlain`, `text`, `snippet`, `$now.toISO()`
- **Input and output connections:**
  - Input from **New Email Received**
  - Output to **Build Deal Text**
- **Version-specific requirements:**  
  Uses `Set` node `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - `from.value[0]` may be undefined for malformed sender data
  - Plain text body may be empty if email is HTML-only
  - Attachment metadata may differ across Gmail payloads
- **Sub-workflow reference:**  
  None

### Normalize Webhook Data

- **Type and technical role:** `n8n-nodes-base.set`  
  Maps webhook payload fields into the same internal schema used for email submissions.
- **Configuration choices:**
  Creates:
  - `source = "webhook"`
  - `sender_name = {{$json.body.sender_name || $json.body.name || 'Unknown'}}`
  - `sender_email = {{$json.body.sender_email || $json.body.email || ''}}`
  - `subject = {{$json.body.company_name || $json.body.deal_name || 'Deal Submission'}}`
  - `email_body = {{$json.body.description || $json.body.summary || $json.body.pitch || ''}}`
  - `has_attachments = {{$json.body.document_url ? true : false}}`
  - `received_at = {{$now.toISO()}}`
  - `document_url = {{$json.body.document_url || ''}}`
- **Key expressions or variables used:**  
  `body.sender_name`, `body.email`, `body.description`, `body.document_url`
- **Input and output connections:**
  - Input from **Respond - Deal Received**
  - Output to **Build Deal Text**
- **Version-specific requirements:**  
  Uses `Set` node `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - If caller sends fields at top level instead of under `body`, expressions return empty values
  - Missing description/summary/pitch can lead to short content and stop path
  - `document_url` is only flagged, not downloaded or parsed
- **Sub-workflow reference:**  
  None

### Build Deal Text

- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript node that assembles a single AI-ready text blob from normalized fields and available binary attachment metadata.
- **Configuration choices:**
  - Reads the first input item
  - Uses `subject`, sender info, and `email_body`
  - Iterates through `item.binary`
  - If a binary file looks like a PDF or document, appends an attachment marker such as `[Attachment: filename.pdf]`
  - Outputs:
    - merged original JSON
    - `deal_text`
    - `deal_text_length`
- **Key expressions or variables used:**
  - `const item = $input.first();`
  - `Object.keys(item.binary || {})`
  - `binary.mimeType`
  - `binary.fileName`
- **Input and output connections:**
  - Input from **Normalize Email Data** or **Normalize Webhook Data**
  - Output to **Has Deal Content?**
- **Version-specific requirements:**  
  Uses Code node `typeVersion: 2`
- **Edge cases or potential failure types:**
  - Only attachment names/types are included, not extracted document text
  - If multiple items somehow arrive, only the first is processed
  - Binary structure may vary depending on source node
- **Sub-workflow reference:**  
  None

### Has Deal Content?

- **Type and technical role:** `n8n-nodes-base.if`  
  Validation gate to determine whether `deal_text` is substantive enough.
- **Configuration choices:**
  - Checks `{{$json.deal_text_length}} > 20`
  - True path continues
  - False path stops
- **Key expressions or variables used:**  
  `{{$json.deal_text_length}}`
- **Input and output connections:**
  - Input from **Build Deal Text**
  - True → **Extract Deal Info - OpenAI**
  - False → **Stop - No Deal Content**
- **Version-specific requirements:**  
  Uses `If` node `typeVersion: 2.2`
- **Edge cases or potential failure types:**
  - Threshold is simplistic; very low-quality but long submissions will still pass
  - Type validation is strict; malformed numeric values could fail condition handling
- **Sub-workflow reference:**  
  None

### Stop - No Deal Content

- **Type and technical role:** `n8n-nodes-base.stopAndError`  
  Explicitly halts execution for empty submissions.
- **Configuration choices:**  
  Default stop/error behavior; no custom message configured.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**
  - Input from false branch of **Has Deal Content?**
  - No downstream output
- **Version-specific requirements:**  
  Uses `typeVersion: 1`
- **Edge cases or potential failure types:**
  - Will mark the execution as errored/stopped
  - Could be noisy in execution logs if many poor submissions are expected
- **Sub-workflow reference:**  
  None

---

## 2.2 AI Extraction & Scoring

**Overview:**  
This block sends the assembled deal text to OpenAI for structured extraction, merges the AI output with original metadata, then runs a second OpenAI scoring pass to produce a verdict and recommendation.

**Nodes Involved:**  
- Extract Deal Info - OpenAI
- Merge Deal Fields
- Score Deal - OpenAI
- Parse Score Results

### Extract Deal Info - OpenAI

- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  Chat-model call for structured information extraction.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - `jsonOutput: true`
  - `onError: continueRegularOutput`
  - Uses a system prompt instructing the model to return only JSON with:
    - company_name
    - industry
    - stage
    - business_model
    - revenue
    - revenue_growth
    - ask_amount
    - valuation
    - team_size
    - location
    - key_highlights
    - red_flags
  - User content is `{{$json.deal_text}}`
- **Key expressions or variables used:**  
  `{{$json.deal_text}}`
- **Input and output connections:**
  - Input from true branch of **Has Deal Content?**
  - Output to **Merge Deal Fields**
- **Version-specific requirements:**  
  Uses LangChain OpenAI node `typeVersion: 1.8`; requires OpenAI credentials and a version supporting `jsonOutput`.
- **Edge cases or potential failure types:**
  - Invalid JSON despite prompt request
  - API auth/rate-limit/model access errors
  - Long content may exceed token constraints
  - Since `onError` continues, downstream nodes may receive incomplete structure
- **Sub-workflow reference:**  
  None

### Merge Deal Fields

- **Type and technical role:** `n8n-nodes-base.set`  
  Consolidates extracted AI fields with upstream metadata from **Build Deal Text**.
- **Configuration choices:**
  - Reads extracted values from `$json.message.content.*`
  - Applies defaults like `Unknown Company`, `Not disclosed`, `None noted`
  - Rehydrates original data from `$('Build Deal Text').item.json.*`
  - Preserves:
    - source
    - sender_name
    - sender_email
    - received_at
    - deal_text
- **Key expressions or variables used:**
  - `$json.message.content.company_name`
  - `$('Build Deal Text').item.json.source`
  - `$('Build Deal Text').item.json.deal_text`
- **Input and output connections:**
  - Input from **Extract Deal Info - OpenAI**
  - Output to **Score Deal - OpenAI**
- **Version-specific requirements:**  
  Uses `Set` node `typeVersion: 3.4`; cross-node item references assume expected execution structure.
- **Edge cases or potential failure types:**
  - If OpenAI returns malformed/non-JSON content, `message.content.*` access may fail or return undefined
  - Cross-node reference may be brittle if future workflow branching/item cardinality changes
- **Sub-workflow reference:**  
  None

### Score Deal - OpenAI

- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  Second chat-model call that scores the deal and assigns a routing verdict.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - `jsonOutput: true`
  - `onError: continueRegularOutput`
  - System prompt defines five scoring dimensions and explicit verdict thresholds
  - Weighted formula:
    - industry_fit × 0.2
    - revenue_stage × 0.25
    - growth_trajectory × 0.25
    - team_strength × 0.2
    - deal_clarity × 0.1
  - User prompt passes extracted fields plus truncated raw text:
    - `{{ $json.deal_text.substring(0, 3000) }}`
- **Key expressions or variables used:**
  - `{{ $json.company_name }}`
  - `{{ $json.industry }}`
  - `{{ $json.deal_text.substring(0, 3000) }}`
- **Input and output connections:**
  - Input from **Merge Deal Fields**
  - Output to **Parse Score Results**
- **Version-specific requirements:**  
  Uses LangChain OpenAI `typeVersion: 1.8`
- **Edge cases or potential failure types:**
  - LLM may not exactly follow scoring thresholds/formula
  - Invalid JSON or missing fields despite `jsonOutput`
  - API rate limits or model access issues
  - Truncation to 3000 chars may omit useful context
- **Sub-workflow reference:**  
  None

### Parse Score Results

- **Type and technical role:** `n8n-nodes-base.set`  
  Converts the scoring result into explicit numeric/string fields and merges back key deal metadata for downstream routing and logging.
- **Configuration choices:**
  - Pulls numeric fields from `$json.message.content.*`
  - Provides defaults of `5` or `REVIEW` when absent
  - Restores deal information from `$('Merge Deal Fields').item.json.*`
- **Key expressions or variables used:**
  - `$json.message.content.industry_fit`
  - `$('Merge Deal Fields').item.json.company_name`
  - `$('Merge Deal Fields').item.json.sender_email`
- **Input and output connections:**
  - Input from **Score Deal - OpenAI**
  - Output to **Is PASS?**
- **Version-specific requirements:**  
  Uses `Set` node `typeVersion: 3.4`
- **Edge cases or potential failure types:**
  - If model output is malformed, defaults may hide upstream failure
  - Numeric parsing may be inconsistent if model returns strings
  - Cross-node references depend on single-item execution alignment
- **Sub-workflow reference:**  
  None

---

## 2.3 Route & Log

**Overview:**  
This block routes deals by verdict, sends a Telegram notification corresponding to the outcome, and appends every processed deal to a Google Sheets pipeline tab.

**Nodes Involved:**  
- Is PASS?
- Is REVIEW?
- Telegram - PASS Alert
- Telegram - REVIEW Notice
- Telegram - REJECT Log
- Log Deal to Pipeline Sheet

### Is PASS?

- **Type and technical role:** `n8n-nodes-base.if`  
  First verdict router.
- **Configuration choices:**
  - Checks `{{$json.verdict}} === "PASS"`
  - True branch sends pass alert
  - False branch proceeds to review check
- **Key expressions or variables used:**  
  `{{$json.verdict}}`
- **Input and output connections:**
  - Input from **Parse Score Results**
  - True → **Telegram - PASS Alert**
  - False → **Is REVIEW?**
- **Version-specific requirements:**  
  Uses `If` node `typeVersion: 2.2`
- **Edge cases or potential failure types:**
  - Case-sensitive exact string comparison
  - Unexpected verdict values fall through to later branches
- **Sub-workflow reference:**  
  None

### Is REVIEW?

- **Type and technical role:** `n8n-nodes-base.if`  
  Second verdict router for non-PASS deals.
- **Configuration choices:**
  - Checks `{{$json.verdict}} === "REVIEW"`
  - True branch sends review notice
  - False branch treats remaining cases as reject
- **Key expressions or variables used:**  
  `{{$json.verdict}}`
- **Input and output connections:**
  - Input from false branch of **Is PASS?**
  - True → **Telegram - REVIEW Notice**
  - False → **Telegram - REJECT Log**
- **Version-specific requirements:**  
  Uses `If` node `typeVersion: 2.2`
- **Edge cases or potential failure types:**
  - Any non-`PASS`, non-`REVIEW` verdict becomes reject, including malformed values
- **Sub-workflow reference:**  
  None

### Telegram - PASS Alert

- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends a Telegram message for PASS outcomes.
- **Configuration choices:**
  - `chatId = YOUR_TELEGRAM_CHAT_ID`
  - Markdown parse mode
  - No explicit text/body is present in the provided JSON
- **Key expressions or variables used:**  
  None shown in parameters
- **Input and output connections:**
  - Input from true branch of **Is PASS?**
  - Output to **Log Deal to Pipeline Sheet**
- **Version-specific requirements:**  
  Uses `typeVersion: 1.2`; requires Telegram bot credentials.
- **Edge cases or potential failure types:**
  - Placeholder chat ID must be replaced
  - Missing message text may cause runtime issues depending on node defaults/UI state
  - Markdown formatting can break on unescaped characters if text is later added
- **Sub-workflow reference:**  
  None

### Telegram - REVIEW Notice

- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends a Telegram message for REVIEW outcomes.
- **Configuration choices:**
  - `chatId = YOUR_TELEGRAM_CHAT_ID`
  - Markdown parse mode
  - No explicit text/body is shown
- **Key expressions or variables used:**  
  None shown in parameters
- **Input and output connections:**
  - Input from true branch of **Is REVIEW?**
  - Output to **Log Deal to Pipeline Sheet**
- **Version-specific requirements:**  
  Uses `typeVersion: 1.2`
- **Edge cases or potential failure types:**
  - Placeholder chat ID must be replaced
  - May fail if operation/message text is incomplete
- **Sub-workflow reference:**  
  None

### Telegram - REJECT Log

- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends a Telegram message for rejected deals.
- **Configuration choices:**
  - `chatId = YOUR_TELEGRAM_CHAT_ID`
  - Markdown parse mode
  - No explicit text/body is shown
- **Key expressions or variables used:**  
  None shown in parameters
- **Input and output connections:**
  - Input from false branch of **Is REVIEW?**
  - Output to **Log Deal to Pipeline Sheet**
- **Version-specific requirements:**  
  Uses `typeVersion: 1.2`
- **Edge cases or potential failure types:**
  - Placeholder chat ID must be replaced
  - Same note as other Telegram nodes regarding missing explicit content
- **Sub-workflow reference:**  
  None

### Log Deal to Pipeline Sheet

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends the processed deal as a row in a Google Sheet.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet ID: `YOUR_GOOGLE_SHEET_ID`
  - Sheet name: `Deal Pipeline`
  - Uses define-below column mapping
  - Appends fields including:
    - company_name
    - sender_name
    - sender_email
    - source
    - received_at
    - industry
    - stage
    - revenue
    - ask_amount
    - key_highlights
    - red_flags
    - scoring fields
    - verdict
    - recommendation
    - one_line_summary
- **Key expressions or variables used:**  
  Many `{{$json.field}}` mappings, e.g. `{{$json.overall_score}}`, `{{$json.verdict}}`
- **Input and output connections:**
  - Input from all three Telegram nodes
  - No downstream output
- **Version-specific requirements:**  
  Uses Google Sheets node `typeVersion: 4.5`
- **Edge cases or potential failure types:**
  - Spreadsheet ID placeholder must be replaced
  - Sheet tab `Deal Pipeline` must exist
  - OAuth permissions must allow write access
  - Column headers in sheet should align with mapped field names for clean append behavior
- **Sub-workflow reference:**  
  None

---

## 2.4 Documentation / Sticky Note Layer

**Overview:**  
These nodes do not execute business logic but provide important operational guidance, setup requirements, and visual grouping in the canvas.

**Nodes Involved:**  
- Section - Intake
- Section - Screen
- Section - Route
- Warning
- Main Description

### Section - Intake

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for the intake/validation area.
- **Configuration choices:**  
  Describes intake normalization, text building, and empty-submission stop behavior.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Sticky note `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

### Section - Screen

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for AI extraction/scoring.
- **Configuration choices:**  
  Explains the two OpenAI steps and five weighted criteria.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Sticky note `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

### Section - Route

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation for routing and logging.
- **Configuration choices:**  
  Explains PASS/REVIEW/REJECT thresholds and logging behavior.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Sticky note `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

### Warning

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Setup warning note.
- **Configuration choices:**  
  Reminds operator to replace Telegram and Google Sheet placeholders and connect credentials.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Sticky note `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

### Main Description

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  High-level workflow description and setup/customization guidance.
- **Configuration choices:**  
  Includes process summary, setup checklist, and customization ideas.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Sticky note `typeVersion: 1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New Email Received | gmailTrigger | Poll unread Gmail deal submissions |  | Normalize Email Data | ### Intake & validate<br>Deals enter via Gmail or webhook. Both paths normalize into the same fields. Build Deal Text combines everything into a single string for AI processing. Empty submissions are stopped. |
| Deal Submission Webhook | webhook | Receive authenticated POST deal submissions |  | Respond - Deal Received | ### Intake & validate<br>Deals enter via Gmail or webhook. Both paths normalize into the same fields. Build Deal Text combines everything into a single string for AI processing. Empty submissions are stopped. |
| Respond - Deal Received | respondToWebhook | Return immediate JSON acknowledgment to webhook caller | Deal Submission Webhook | Normalize Webhook Data | ### Intake & validate<br>Deals enter via Gmail or webhook. Both paths normalize into the same fields. Build Deal Text combines everything into a single string for AI processing. Empty submissions are stopped. |
| Normalize Email Data | set | Map Gmail payload into shared schema | New Email Received | Build Deal Text | ### Intake & validate<br>Deals enter via Gmail or webhook. Both paths normalize into the same fields. Build Deal Text combines everything into a single string for AI processing. Empty submissions are stopped. |
| Normalize Webhook Data | set | Map webhook payload into shared schema | Respond - Deal Received | Build Deal Text | ### Intake & validate<br>Deals enter via Gmail or webhook. Both paths normalize into the same fields. Build Deal Text combines everything into a single string for AI processing. Empty submissions are stopped. |
| Build Deal Text | code | Build unified text prompt payload | Normalize Email Data, Normalize Webhook Data | Has Deal Content? | ### Intake & validate<br>Deals enter via Gmail or webhook. Both paths normalize into the same fields. Build Deal Text combines everything into a single string for AI processing. Empty submissions are stopped. |
| Has Deal Content? | if | Validate minimum deal text length | Build Deal Text | Extract Deal Info - OpenAI, Stop - No Deal Content | ### Intake & validate<br>Deals enter via Gmail or webhook. Both paths normalize into the same fields. Build Deal Text combines everything into a single string for AI processing. Empty submissions are stopped. |
| Stop - No Deal Content | stopAndError | Halt execution for empty submissions | Has Deal Content? |  | ### Intake & validate<br>Deals enter via Gmail or webhook. Both paths normalize into the same fields. Build Deal Text combines everything into a single string for AI processing. Empty submissions are stopped. |
| Extract Deal Info - OpenAI | @n8n/n8n-nodes-langchain.openAi | Extract structured investment deal fields | Has Deal Content? | Merge Deal Fields | ### AI extraction & scoring<br>First OpenAI call extracts structured deal info. Second call scores on 5 weighted criteria (industry fit, revenue, growth, team, clarity) and assigns a verdict. |
| Merge Deal Fields | set | Merge extracted deal fields with original metadata | Extract Deal Info - OpenAI | Score Deal - OpenAI | ### AI extraction & scoring<br>First OpenAI call extracts structured deal info. Second call scores on 5 weighted criteria (industry fit, revenue, growth, team, clarity) and assigns a verdict. |
| Score Deal - OpenAI | @n8n/n8n-nodes-langchain.openAi | Score the deal and assign PASS/REVIEW/REJECT | Merge Deal Fields | Parse Score Results | ### AI extraction & scoring<br>First OpenAI call extracts structured deal info. Second call scores on 5 weighted criteria (industry fit, revenue, growth, team, clarity) and assigns a verdict. |
| Parse Score Results | set | Normalize scoring output for routing and logging | Score Deal - OpenAI | Is PASS? | ### AI extraction & scoring<br>First OpenAI call extracts structured deal info. Second call scores on 5 weighted criteria (industry fit, revenue, growth, team, clarity) and assigns a verdict. |
| Is PASS? | if | Route PASS deals | Parse Score Results | Telegram - PASS Alert, Is REVIEW? | ### Route & log<br>Routes by verdict: PASS (7.5+), REVIEW (5-7.4), REJECT (<5). Each tier gets a Telegram notification. All deals are appended to the Google Sheets pipeline. |
| Is REVIEW? | if | Route REVIEW vs REJECT | Is PASS? | Telegram - REVIEW Notice, Telegram - REJECT Log | ### Route & log<br>Routes by verdict: PASS (7.5+), REVIEW (5-7.4), REJECT (<5). Each tier gets a Telegram notification. All deals are appended to the Google Sheets pipeline. |
| Telegram - PASS Alert | telegram | Send Telegram alert for PASS deals | Is PASS? | Log Deal to Pipeline Sheet | ### Route & log<br>Routes by verdict: PASS (7.5+), REVIEW (5-7.4), REJECT (<5). Each tier gets a Telegram notification. All deals are appended to the Google Sheets pipeline. |
| Telegram - REVIEW Notice | telegram | Send Telegram notification for REVIEW deals | Is REVIEW? | Log Deal to Pipeline Sheet | ### Route & log<br>Routes by verdict: PASS (7.5+), REVIEW (5-7.4), REJECT (<5). Each tier gets a Telegram notification. All deals are appended to the Google Sheets pipeline. |
| Telegram - REJECT Log | telegram | Send Telegram notification for rejected deals | Is REVIEW? | Log Deal to Pipeline Sheet | ### Route & log<br>Routes by verdict: PASS (7.5+), REVIEW (5-7.4), REJECT (<5). Each tier gets a Telegram notification. All deals are appended to the Google Sheets pipeline. |
| Log Deal to Pipeline Sheet | googleSheets | Append processed deal row to Google Sheets | Telegram - PASS Alert, Telegram - REVIEW Notice, Telegram - REJECT Log |  | ### Route & log<br>Routes by verdict: PASS (7.5+), REVIEW (5-7.4), REJECT (<5). Each tier gets a Telegram notification. All deals are appended to the Google Sheets pipeline. |
| Section - Intake | stickyNote | Canvas documentation for intake block |  |  |  |
| Section - Screen | stickyNote | Canvas documentation for AI block |  |  |  |
| Section - Route | stickyNote | Canvas documentation for routing block |  |  |  |
| Warning | stickyNote | Setup warning and credential reminder |  |  |  |
| Main Description | stickyNote | Global workflow description and setup notes |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   **Screen and Score Investment Deals with AI using OpenAI, Gmail, and Telegram**

2. **Add a Gmail Trigger node** named **New Email Received**.
   - Type: Gmail Trigger
   - Polling: every minute
   - Read status filter: unread
   - Include spam/trash: disabled
   - Download attachments: enabled
   - Use Gmail OAuth2 credentials with mailbox access to the inbox you want to monitor

3. **Add a Webhook node** named **Deal Submission Webhook**.
   - Type: Webhook
   - HTTP method: POST
   - Path: `deal-intake`
   - Authentication: Header Auth
   - Response mode: Response Node
   - Configure Header Auth credentials in n8n
   - Expected request body fields:
     - `sender_name` or `name`
     - `sender_email` or `email`
     - `company_name` or `deal_name`
     - `description` or `summary` or `pitch`
     - optional `document_url`

4. **Add a Respond to Webhook node** named **Respond - Deal Received**.
   - Connect it after **Deal Submission Webhook**
   - Respond with JSON
   - Response body:
     ```json
     {"status":"received","message":"Deal submitted for screening"}
     ```

5. **Add a Set node** named **Normalize Email Data**.
   - Connect **New Email Received** → **Normalize Email Data**
   - Create these fields:
     - `source` = `email`
     - `sender_name` = `{{$json.from.value[0].name || $json.from.text || 'Unknown'}}`
     - `sender_email` = `{{$json.from.value[0].address || ''}}`
     - `subject` = `{{$json.subject || 'No Subject'}}`
     - `email_body` = `{{$json.textPlain || $json.text || $json.snippet || ''}}`
     - `has_attachments` = `{{$json.attachments ? $json.attachments.length > 0 : false}}`
     - `received_at` = `{{$now.toISO()}}`
     - `message_id` = `{{$json.id || ''}}`

6. **Add another Set node** named **Normalize Webhook Data**.
   - Connect **Respond - Deal Received** → **Normalize Webhook Data**
   - Create these fields:
     - `source` = `webhook`
     - `sender_name` = `{{$json.body.sender_name || $json.body.name || 'Unknown'}}`
     - `sender_email` = `{{$json.body.sender_email || $json.body.email || ''}}`
     - `subject` = `{{$json.body.company_name || $json.body.deal_name || 'Deal Submission'}}`
     - `email_body` = `{{$json.body.description || $json.body.summary || $json.body.pitch || ''}}`
     - `has_attachments` = `{{$json.body.document_url ? true : false}}`
     - `received_at` = `{{$now.toISO()}}`
     - `document_url` = `{{$json.body.document_url || ''}}`

7. **Add a Code node** named **Build Deal Text**.
   - Connect both normalization nodes into it
   - Use this logic:
     - Read the first item
     - Build text from subject, sender, and email body
     - Inspect binary attachments
     - If PDF/document-like attachments exist, append labels such as `[Attachment: file.pdf]`
     - Return original JSON plus:
       - `deal_text`
       - `deal_text_length`
   - Implement the same behavior as:
     - combine metadata and body into one string
     - count string length
     - include attachment names, not extracted file contents

8. **Add an If node** named **Has Deal Content?**
   - Connect **Build Deal Text** → **Has Deal Content?**
   - Condition:
     - numeric
     - `{{$json.deal_text_length}}`
     - greater than `20`

9. **Add a Stop and Error node** named **Stop - No Deal Content**.
   - Connect the false output of **Has Deal Content?** to this node

10. **Add an OpenAI node** named **Extract Deal Info - OpenAI**.
    - Connect the true output of **Has Deal Content?** to this node
    - Use the LangChain OpenAI chat node
    - Credentials: OpenAI API
    - Model: `gpt-4o-mini`
    - JSON output: enabled
    - On error: continue regular output
    - System prompt should instruct the model to return only JSON with:
      - `company_name`
      - `industry`
      - `stage`
      - `business_model`
      - `revenue`
      - `revenue_growth`
      - `ask_amount`
      - `valuation`
      - `team_size`
      - `location`
      - `key_highlights`
      - `red_flags`
    - User message/content:
      - `{{$json.deal_text}}`

11. **Add a Set node** named **Merge Deal Fields**.
    - Connect **Extract Deal Info - OpenAI** → **Merge Deal Fields**
    - Map extracted fields from `$json.message.content.*`
    - Add defaults:
      - `company_name`: `Unknown Company`
      - text fields mostly `Not disclosed`
      - `red_flags`: `None noted`
    - Also bring back original values from **Build Deal Text** using node references:
      - `source`
      - `sender_name`
      - `sender_email`
      - `received_at`
      - `deal_text`

12. **Add a second OpenAI node** named **Score Deal - OpenAI**.
    - Connect **Merge Deal Fields** → **Score Deal - OpenAI**
    - Credentials: same OpenAI account
    - Model: `gpt-4o-mini`
    - JSON output: enabled
    - On error: continue regular output
    - System prompt should define scoring:
      - `industry_fit`
      - `revenue_stage`
      - `growth_trajectory`
      - `team_strength`
      - `deal_clarity`
      - `overall_score`
      - `verdict`
      - `one_line_summary`
      - `recommendation`
    - Include verdict thresholds:
      - PASS: `>= 7.5`
      - REVIEW: `>= 5.0 and < 7.5`
      - REJECT: `< 5.0`
    - Include weighted formula:
      - `industry_fit*0.2 + revenue_stage*0.25 + growth_trajectory*0.25 + team_strength*0.2 + deal_clarity*0.1`
    - User message should pass extracted fields and:
      - `{{$json.deal_text.substring(0, 3000)}}`

13. **Add a Set node** named **Parse Score Results**.
    - Connect **Score Deal - OpenAI** → **Parse Score Results**
    - Parse from `$json.message.content.*`
    - Create:
      - `industry_fit`
      - `revenue_stage`
      - `growth_trajectory`
      - `team_strength`
      - `deal_clarity`
      - `overall_score`
      - `verdict`
      - `one_line_summary`
      - `recommendation`
    - Set defaults:
      - score fields default to `5`
      - verdict defaults to `REVIEW`
    - Re-add metadata from **Merge Deal Fields**:
      - `company_name`
      - `industry`
      - `stage`
      - `ask_amount`
      - `revenue`
      - `key_highlights`
      - `red_flags`
      - `sender_name`
      - `sender_email`
      - `received_at`
      - `source`

14. **Add an If node** named **Is PASS?**
    - Connect **Parse Score Results** → **Is PASS?**
    - Condition:
      - string equals
      - `{{$json.verdict}}`
      - `PASS`

15. **Add another If node** named **Is REVIEW?**
    - Connect the false output of **Is PASS?** → **Is REVIEW?**
    - Condition:
      - string equals
      - `{{$json.verdict}}`
      - `REVIEW`

16. **Add a Telegram node** named **Telegram - PASS Alert**.
    - Connect true output of **Is PASS?** to it
    - Configure Telegram Bot credentials
    - Set chat ID to your real target value instead of `YOUR_TELEGRAM_CHAT_ID`
    - Parse mode: Markdown
    - Add a send-message operation and define a message body, for example including:
      - company name
      - overall score
      - verdict
      - one-line summary
      - recommendation

17. **Add a Telegram node** named **Telegram - REVIEW Notice**.
    - Connect true output of **Is REVIEW?** to it
    - Same Telegram credential
    - Same chat ID replacement
    - Parse mode: Markdown
    - Add review-oriented message content

18. **Add a Telegram node** named **Telegram - REJECT Log**.
    - Connect false output of **Is REVIEW?** to it
    - Same credential
    - Same chat ID replacement
    - Parse mode: Markdown
    - Add reject-oriented message content

19. **Add a Google Sheets node** named **Log Deal to Pipeline Sheet**.
    - Connect all three Telegram nodes into this single node
    - Credentials: Google Sheets OAuth2
    - Operation: Append
    - Document ID: replace `YOUR_GOOGLE_SHEET_ID`
    - Sheet name: `Deal Pipeline`
    - Use manual field mapping
    - Map these columns:
      - `company_name`
      - `sender_name`
      - `sender_email`
      - `source`
      - `received_at`
      - `industry`
      - `stage`
      - `revenue`
      - `ask_amount`
      - `key_highlights`
      - `red_flags`
      - `industry_fit`
      - `revenue_stage`
      - `growth_trajectory`
      - `team_strength`
      - `deal_clarity`
      - `overall_score`
      - `verdict`
      - `one_line_summary`
      - `recommendation`

20. **Create the Google Sheet structure**.
    - Create a spreadsheet
    - Add a tab named **Deal Pipeline**
    - Create headers matching the mapped column names above

21. **Configure credentials required by the workflow**:
    - Gmail OAuth2 for the Gmail trigger
    - OpenAI API credentials for both AI nodes
    - Telegram Bot credentials for the Telegram nodes
    - Google Sheets OAuth2 for the logging node
    - Header Auth credentials for the webhook node

22. **Replace all placeholders before activation**:
    - `YOUR_TELEGRAM_CHAT_ID` in all three Telegram nodes
    - `YOUR_GOOGLE_SHEET_ID` in the Google Sheets node

23. **Test the Gmail path**.
    - Send an unread email to the connected inbox
    - Confirm normalization, AI extraction, routing, and Google Sheets logging

24. **Test the webhook path**.
    - Send an authenticated POST request to `/deal-intake`
    - Example expected payload shape:
      ```json
      {
        "sender_name": "Jane Doe",
        "sender_email": "jane@example.com",
        "company_name": "Acme AI",
        "description": "AI platform for industrial operations",
        "document_url": "https://example.com/deck.pdf"
      }
      ```
    - Verify immediate JSON acknowledgment and downstream processing

25. **Activate the workflow** only after both intake paths and all credentials are confirmed.

### Sub-workflow setup
This workflow does **not** invoke any sub-workflow and does **not** contain an Execute Workflow node. There are **two entry points**, both documented above:
- Gmail trigger
- Authenticated webhook

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Dont forget to** Replace YOUR_TELEGRAM_CHAT_ID in all three Telegram nodes and YOUR_GOOGLE_SHEET_ID in the Log Deal node before activating. Connect all five credentials (Gmail, OpenAI, Telegram, Sheets, Header Auth). | Setup warning |
| ## Screen and Score Investment Deals with AI. Screens incoming deal submissions via email or webhook, extracts key data with OpenAI, scores against 5 weighted investment criteria, and sends Telegram alerts by verdict. | Workflow purpose |
| Setup checklist includes connecting Gmail, OpenAI, Telegram Bot, and Google Sheets credentials; replacing placeholders; creating a `Deal Pipeline` tab; and setting webhook auth header. | Operational setup |
| Customization ideas: edit scoring criteria and weights in the scoring prompt, swap Telegram for Slack or email, or connect a CRM instead of Google Sheets. | Extension ideas |

## Additional implementation notes

- The workflow is currently **inactive**.
- Workflow settings show:
  - `binaryMode: separate`
  - `executionOrder: v1`
- The code step only detects document attachments by metadata; it does **not** extract PDF or DOC text.
- Both OpenAI nodes use `onError: continueRegularOutput`, so downstream defaults may mask model/API failures unless you add explicit validation.
- The Telegram nodes in the provided JSON include `chatId` and parse mode but do not show explicit message text; in practice, you should verify or complete their message configuration before production use.