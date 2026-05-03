Generate and send AI-powered sales quotes with Gmail, OpenAI and Supabase

https://n8nworkflows.xyz/workflows/generate-and-send-ai-powered-sales-quotes-with-gmail--openai-and-supabase-12994


# Generate and send AI-powered sales quotes with Gmail, OpenAI and Supabase

## 1. Workflow Overview

**Purpose:** This workflow automates intake, analysis, pricing, and drafting of sales quotes using **Gmail + OpenAI + Supabase**, with an optional **Slack** alert and a separate **“Approve & Send”** webhook for a human-review dashboard (e.g., Replit).

**Primary use cases:**
- Automatically detect genuine quote requests from inbound emails (reducing OpenAI spend on spam/newsletters).
- Normalize quote requests coming from either **Gmail** or a **website form**.
- Extract structured deal data, generate tiered pricing, draft a personalized quote email, and store everything in **Supabase** for human review.
- After approval, send the final quote via **Gmail** and mark the record as **SENT**.

### 1.1 Input Reception (Two entry points)
- **Email polling** via Gmail Trigger (every minute).
- **Form submission** via Webhook (`/webhook/quote-request`) with immediate HTTP response.

### 1.2 AI Email Filtering (Email path only)
- OpenAI classifies whether the email is a quote request with confidence threshold **>= 60**.

### 1.3 Source Merge & Normalization
- Email and form payloads are merged and transformed into a consistent normalized schema.

### 1.4 AI Extraction & Qualification
- OpenAI extracts contact/company details, service category, budget/timeline, and computes multiple scores (complexity, urgency, qualification, value estimate).

### 1.5 Persistence (Supabase) + Historical Context
- Insert initial record (status **PROCESSING**).
- Fetch past won quotes (last 10) for pricing context.

### 1.6 AI Pricing + Draft Quote Creation
- OpenAI calculates 3-tier pricing and rationale.
- OpenAI drafts a quote email body (under 300 words).

### 1.7 Final Database Update + Slack Alert
- Update record with pricing + draft email and set status **PENDING_REVIEW**.
- If estimated value >= **$10,000**, send a Slack alert.

### 1.8 Human Approval → Send Quote Webhook
- Dashboard triggers `/webhook/send-quote` with final email body.
- Workflow sends via Gmail and updates Supabase status to **SENT**, returning JSON success.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Email Intake (Gmail polling)
**Overview:** Polls the Gmail inbox every minute and forwards full email content to the AI filter.  
**Nodes Involved:** `Gmail Trigger`, `📧 Email Input`

#### Node: Gmail Trigger
- **Type / Role:** `n8n-nodes-base.gmailTrigger` — scheduled polling trigger for new Gmail messages.
- **Configuration (interpreted):**
  - Polling mode: **everyMinute**
  - Filter: label **INBOX** only
  - `simple: false` → returns richer/resolved message structure (headers, html, text, etc.)
- **Key fields used downstream:**  
  - `headers.subject`, `headers.from`, `text`, `id`, `from.value[0].address`
- **Connections:**
  - Output → `OpenAI - Email Filter`
- **Version notes:** typeVersion **1**
- **Edge cases / failures:**
  - OAuth token expiration / revoked consent
  - Gmail API quota limits
  - Unexpected message structure (missing `from.value[0]` or `text`)
  - Duplicate processing if Gmail label/state is not managed (this workflow does not mark as read/archive)

#### Sticky note: 📧 Email Input
- Documents Gmail polling, INBOX label filter, “Resolved” content expectation.

---

### Block 2.2 — Form Intake (Website webhook)
**Overview:** Receives website quote requests via webhook and responds immediately with success JSON; form submissions bypass the AI email filter.  
**Nodes Involved:** `Webhook - Form Submission`, `Respond - Form OK`, `🌐 Form Input`

#### Node: Webhook - Form Submission
- **Type / Role:** `n8n-nodes-base.webhook` — public HTTP entrypoint.
- **Configuration (interpreted):**
  - Path: `quote-request` → endpoint `/webhook/quote-request`
  - Method: **POST**
  - Response mode: `responseNode` (response handled by `Respond - Form OK`)
- **Connections:**
  - Output (main) → `Merge - Both Paths` (index **1** input)
  - Output (main) → `Respond - Form OK`
- **Version notes:** typeVersion **1**
- **Edge cases / failures:**
  - Missing expected fields; normalization tries both `item.body` and top-level fields, but mismatched names (e.g., `budgetRange` vs `budget`) may lead to empty normalized values.
  - No authentication/signature check (relies on `x-is-trusted` header in pinned example; not enforced in node config).
  - Large payload sizes could exceed platform limits.

#### Node: Respond - Form OK
- **Type / Role:** `n8n-nodes-base.respondToWebhook` — sends HTTP response.
- **Configuration (interpreted):**
  - Respond with JSON: `{ "success": true, "message": "Form received" }`
- **Connections:** none (terminal for the HTTP response)
- **Version notes:** typeVersion **1.1**
- **Edge cases:** If upstream errors before this node executes, webhook may time out.

#### Sticky note: 🌐 Form Input
- Includes expected fields and example webhook URL:
  - `https://your-n8n.com/webhook/quote-request`
- Notes: “Forms skip AI filter”.

---

### Block 2.3 — AI Email Filter + Routing (Email path)
**Overview:** Uses OpenAI to decide if an email is a real quote request; only confident matches continue to the main pipeline, others are effectively dropped (note says “archive email”, but no archiving node exists).  
**Nodes Involved:** `OpenAI - Email Filter`, `Parse Filter Response`, `IF - Is Quote Request?`, `🤖 AI Filter`, `⚡ Routing`

#### Node: OpenAI - Email Filter
- **Type / Role:** `@n8n/n8n-nodes-langchain.openAi` — LLM call for classification.
- **Configuration (interpreted):**
  - Model: `gpt-4o`
  - Temperature: `0.2` (more deterministic)
  - Prompt: asks for **ONLY valid JSON** with:
    - `is_quote_request` boolean
    - `confidence` 0–100
    - `detected_signals` array
    - `reasoning` brief
  - Uses Gmail fields: `{{$json.headers.subject}}`, `{{$json.text}}`, `{{$json.headers.from}}`
- **Connections:**
  - Output → `Parse Filter Response`
- **Version notes:** node typeVersion **1.8**
- **Edge cases / failures:**
  - Model returns non-JSON (markdown fences, extra text)
  - Token limits if email bodies are very large
  - OpenAI auth/quota issues

#### Node: Parse Filter Response
- **Type / Role:** `n8n-nodes-base.code` — robust parsing of LLM output + attaches original email.
- **Configuration (interpreted):**
  - Extracts content from common OpenAI response shapes: `text`, `output`, `message.content`
  - Strips ```json fences
  - On success: merges parsed JSON + `original_email` from `$('Gmail Trigger').first().json`
  - On failure: returns `{ is_quote_request:false, confidence:0, ... reasoning:'Failed to parse...' }` and still attaches `original_email`.
- **Connections:**
  - Output → `IF - Is Quote Request?`
- **Version notes:** typeVersion **2**
- **Edge cases:**
  - Hard dependency on node name **`Gmail Trigger`** via `$('Gmail Trigger')` selector; renaming breaks it.
  - If executed without Gmail Trigger context (shouldn’t happen in current wiring), `original_email` could be undefined.

#### Node: IF - Is Quote Request?
- **Type / Role:** `n8n-nodes-base.if` — branching gate.
- **Configuration (interpreted):**
  - Condition AND:
    - `is_quote_request == true`
    - `confidence >= 60`
- **Connections:**
  - True → `Merge - Both Paths`
  - False → no node connected (despite sticky note saying “Archive email”)
- **Version notes:** typeVersion **2**
- **Edge cases:**
  - If OpenAI output parsing fails, email is dropped (confidence 0).
  - No action on false branch → emails remain in INBOX and may be reprocessed depending on trigger behavior.

---

### Block 2.4 — Merge Sources & Normalize
**Overview:** Combines the accepted email path and the form path into one pipeline and normalizes fields into a consistent structure for extraction.  
**Nodes Involved:** `Merge - Both Paths`, `Normalize Data`, `🔀 Merge Sources`

#### Node: Merge - Both Paths
- **Type / Role:** `n8n-nodes-base.merge` — merges two inbound streams.
- **Configuration (interpreted):**
  - Default merge behavior (no explicit mode set in parameters)
  - It is used primarily as a “join point” so either source continues.
- **Inputs:**
  - Input 0: from `IF - Is Quote Request?` (email accepted)
  - Input 1: from `Webhook - Form Submission`
- **Connections:**
  - Output → `Normalize Data`
- **Version notes:** typeVersion **3**
- **Edge cases:**
  - Depending on merge mode defaults, it may behave as “append” or require paired items; with triggers, typical expectation is “pass-through/append”, but if the default changes, you could get blocked executions waiting on both inputs. (Safer would be explicitly setting mode to “append”.)

#### Node: Normalize Data
- **Type / Role:** `n8n-nodes-base.code` — standardizes payload fields.
- **Configuration (interpreted):**
  - Creates baseline:
    - `source`, `raw_content`, `contact_email`, `timestamp`
  - Email path detection: `if (item.original_email)`:
    - `source='email'`
    - `raw_content` from `text | textPlain | snippet`
    - `contact_email` from `original_email.from.value[0].address`
    - `subject`, `email_id`
  - Form path detection: `else if (item.body || item.name)`:
    - `source='form'`
    - `raw_content = JSON.stringify(item.body || item)`
    - Maps fields from either top-level or `body.*`
- **Connections:**
  - Output → `OpenAI - Extract & Classify`
- **Version notes:** typeVersion **2**
- **Edge cases:**
  - The pinned form example uses `budgetRange` and `serviceType`, but code expects `budget` / `service_type`; those will normalize as empty unless the form matches expected naming.
  - If Gmail `from.value` array is absent, `contact_email` becomes empty, affecting extraction quality.

---

### Block 2.5 — AI Extraction & Classification
**Overview:** Converts normalized free-text requests into a structured deal record with enrichment scores used for prioritization and later routing.  
**Nodes Involved:** `OpenAI - Extract & Classify`, `Parse Extraction`, `🧠 Extraction`

#### Node: OpenAI - Extract & Classify
- **Type / Role:** `@n8n/n8n-nodes-langchain.openAi` — structured data extraction.
- **Configuration (interpreted):**
  - Model: `gpt-4o`
  - Temperature: `0.2`
  - Prompt includes normalized fields: source, raw content, contact email, subject
  - Required JSON keys:
    - Contact/company info
    - `service_type` category (freight_logistics, warehousing, last_mile, customs, consulting, other)
    - `budget_range`, `timeline`, `special_requirements`, `decision_maker`
    - `complexity_score` 1–10, `urgency_level`, `value_estimate`, `qualification_score`
- **Connections:**
  - Output → `Parse Extraction`
- **Version notes:** typeVersion **1.8**
- **Edge cases:**
  - Model returns invalid JSON or missing fields
  - Long content may be truncated

#### Node: Parse Extraction
- **Type / Role:** `n8n-nodes-base.code` — parses extraction JSON and re-attaches normalized metadata.
- **Configuration (interpreted):**
  - Reads LLM response content from `text|output|message.content`
  - Parses cleaned JSON
  - Adds:
    - `source`, `raw_content` from `$('Normalize Data')`
    - `email_id`, `received_at`
  - On parse failure: returns fallback defaults (unknown contact/company, medium urgency, etc.) and includes `parse_error`.
- **Connections:**
  - Output → `Prepare Insert`
- **Version notes:** typeVersion **2**
- **Edge cases:**
  - Depends on node name **`Normalize Data`** via selector; renaming breaks.
  - Fallback email uses `normalized.contact_email || 'user@example.com'` which can corrupt data if contact email is missing.

---

### Block 2.6 — Save Initial Record (Supabase)
**Overview:** Inserts a quote request record in Supabase with status **PROCESSING** so downstream updates can reference a persistent ID.  
**Nodes Involved:** `Prepare Insert`, `Supabase - Save Initial`, `💾 Save Initial`

#### Node: Prepare Insert
- **Type / Role:** `n8n-nodes-base.code` — maps extracted fields to DB insert shape.
- **Configuration (interpreted):**
  - Creates insert JSON fields: contact_name, company_name, email, phone, service_type, etc.
  - Sets `status: 'PROCESSING'`
- **Connections:**
  - Output → `Supabase - Save Initial`
- **Version notes:** typeVersion **2**
- **Edge cases:** missing `email` from extraction could cause DB constraints failure if column is required.

#### Node: Supabase - Save Initial
- **Type / Role:** `n8n-nodes-base.supabase` — inserts into Supabase table.
- **Configuration (interpreted):**
  - Table: `quote_requests`
  - Data mapping: `autoMapInputData`
  - Operation: (implicit insert; node is configured for creating a record)
- **Connections:**
  - Output → `Supabase - Search Similar`
- **Credentials:** Supabase API credential “YouTube Demo”
- **Version notes:** typeVersion **1**
- **Edge cases / failures:**
  - Table schema mismatch (column names/types)
  - RLS policies blocking insert
  - Missing required columns (e.g., created_at default not set)

---

### Block 2.7 — Historical Similar Quotes Lookup (Supabase)
**Overview:** Pulls recent won deals to improve pricing accuracy.  
**Nodes Involved:** `Supabase - Search Similar`, `📊 Historical`

#### Node: Supabase - Search Similar
- **Type / Role:** `n8n-nodes-base.supabase` — queries records.
- **Configuration (interpreted):**
  - Operation: `getAll`
  - Table: `quote_requests`
  - Limit: 10
  - Filter string: `=status=eq.WON&order=created_at.desc`
  - `alwaysOutputData: true` (continues even if no rows returned)
- **Connections:**
  - Output → `OpenAI - Calculate Pricing`
- **Version notes:** typeVersion **1**
- **Important behavior note:** Despite the sticky note claiming “same service_type”, the actual filter only enforces `status=WON` and ordering. No service_type filter is applied.
- **Edge cases:**
  - If `created_at` column doesn’t exist, query fails.
  - If returns an empty array, pricing prompt still works (but has less context).

---

### Block 2.8 — AI Pricing Calculation
**Overview:** Generates Good/Better/Best pricing based on extracted scores and (optionally) historical quotes.  
**Nodes Involved:** `OpenAI - Calculate Pricing`, `Parse Pricing`, `💰 Pricing`

#### Node: OpenAI - Calculate Pricing
- **Type / Role:** `@n8n/n8n-nodes-langchain.openAi` — pricing logic via LLM.
- **Configuration (interpreted):**
  - Model: `gpt-4o`
  - Temperature: `0.3`
  - Prompt references extraction values via:
    - `$('Parse Extraction').first().json.*`
  - Supplies “Similar Past Quotes” as `{{ JSON.stringify($json) }}` where `$json` is output from Supabase search (likely an array/object of records)
  - Pricing rules embedded (base rate, multipliers, minimum, tiers)
  - Expected strict JSON output with prices and rationale
- **Connections:**
  - Output → `Parse Pricing`
- **Version notes:** typeVersion **1.8**
- **Edge cases:**
  - Hard dependency on node name `Parse Extraction`
  - LLM can output non-numeric values; parser will fail and fallback will be used

#### Node: Parse Pricing
- **Type / Role:** `n8n-nodes-base.code` — parse pricing JSON and attach DB record ID.
- **Configuration (interpreted):**
  - Parses LLM output as JSON after stripping markdown fences
  - Retrieves inserted record ID from `$('Supabase - Save Initial')`:
    - Handles array or object response shapes
  - Outputs: pricing fields + `extracted` + `record_id`
  - On parse failure: creates fallback prices based on `value_estimate`
- **Connections:**
  - Output → `OpenAI - Generate Draft`
- **Version notes:** typeVersion **2**
- **Edge cases:**
  - Depends on node names `Supabase - Save Initial` and `Parse Extraction`
  - If Supabase insert response doesn’t include `id`, later updates will fail

---

### Block 2.9 — Draft Quote Email Generation
**Overview:** Writes a customer-ready email body using the extracted data and calculated pricing tiers.  
**Nodes Involved:** `OpenAI - Generate Draft`, `Prepare Update`, `✉️ Draft Email`

#### Node: OpenAI - Generate Draft
- **Type / Role:** `@n8n/n8n-nodes-langchain.openAi` — content generation.
- **Configuration (interpreted):**
  - Model: `gpt-4o`
  - Temperature: `0.7` (more creative tone)
  - Prompt includes contact/company, service, timeline, requirements, 3-tier prices, rationale
  - Output constraint: “ONLY the email body text. No subject line, no JSON, no markdown”
  - Stylistic rule: “Don’t use em dashes”
- **Connections:**
  - Output → `Prepare Update`
- **Version notes:** typeVersion **1.8**
- **Edge cases:**
  - LLM might include subject line anyway; workflow doesn’t sanitize
  - Very long output could exceed DB field limits if `draft_email` column is small

#### Node: Prepare Update
- **Type / Role:** `n8n-nodes-base.code` — packages DB update payload.
- **Configuration (interpreted):**
  - Extracts draft email from OpenAI response fields (`text|output|message.content`)
  - Uses `$('Parse Pricing')` to fetch:
    - record id, prices, extracted qualification score
  - Sets:
    - `priority_score = qualification_score`
    - `status = 'PENDING_REVIEW'`
- **Connections:**
  - Output → `Supabase - Update Complete`
- **Version notes:** typeVersion **2**
- **Edge cases:** depends on node name `Parse Pricing`; if renamed, update breaks.

---

### Block 2.10 — Update Supabase + Slack High-Value Alert
**Overview:** Persists final draft + prices, marks record ready for review, and alerts Slack for high-value requests.  
**Nodes Involved:** `Supabase - Update Complete`, `IF - High Value?`, `Slack - High Value Alert`, `🔄 Update DB`, `🔔 Slack Alert`

#### Node: Supabase - Update Complete
- **Type / Role:** `n8n-nodes-base.supabase` — updates the record created earlier.
- **Configuration (interpreted):**
  - Operation: `update`
  - Table: `quote_requests`
  - Filter: `id == {{$json.id}}`
  - Updates fields: `good_price`, `better_price`, `best_price`, `draft_email`, `priority_score`, and `status` set to `PENDING_REVIEW`
- **Connections:**
  - Output → `IF - High Value?`
- **Credentials:** Supabase API credential “YouTube Demo”
- **Version notes:** typeVersion **1**
- **Edge cases:**
  - If `id` is null/empty, update affects 0 rows
  - RLS policy may block update

#### Node: IF - High Value?
- **Type / Role:** `n8n-nodes-base.if` — optional notification gate.
- **Configuration (interpreted):**
  - Condition: `value_estimate >= 10000`
  - Value pulled from `$('Prepare Update').first().json.extracted.value_estimate`
- **Connections:**
  - True → `Slack - High Value Alert`
  - False → no node (silent)
- **Version notes:** typeVersion **2**
- **Edge cases:** depends on node name `Prepare Update`.

#### Node: Slack - High Value Alert
- **Type / Role:** `n8n-nodes-base.slack` — posts Slack message.
- **Configuration (interpreted):**
  - Auth: OAuth2
  - Posts to channel ID `C08MU2RNGTU` (cached name “ai-assistant”)
  - Message includes company/contact/email/value/service/urgency and a dashboard link:
    - `https://your-replit-dashboard.com` (as Slack formatted link)
- **Connections:** none (terminal)
- **Version notes:** typeVersion **2.2**
- **Edge cases / failures:**
  - Slack OAuth scopes missing (chat:write)
  - Channel not found / bot not in channel
  - Dashboard link is placeholder

---

### Block 2.11 — Approve & Send (Webhook → Gmail → Supabase)
**Overview:** A separate webhook endpoint is called by a dashboard after human review; it sends the final email via Gmail and marks the Supabase record as SENT.  
**Nodes Involved:** `Webhook - Send Quote`, `Gmail - Send Quote`, `Set Sent Status`, `Supabase - Mark Sent`, `Respond - Success`, `📤 Send Webhook`

#### Node: Webhook - Send Quote
- **Type / Role:** `n8n-nodes-base.webhook` — inbound approval action endpoint.
- **Configuration (interpreted):**
  - Path: `send-quote` → endpoint `/webhook/send-quote`
  - Method: POST
  - Response mode: `responseNode` (response handled by `Respond - Success`)
- **Expected body fields (from sticky note + pinned data):**
  - `quote_id`, `recipient_email`, `company_name`, `service_type`, `final_email_body`
- **Connections:**
  - Output → `Gmail - Send Quote`
- **Version notes:** typeVersion **1**
- **Edge cases:**
  - No auth/signature validation; anyone who can reach the endpoint could send emails unless protected at infrastructure level.
  - Missing/invalid email fields cause Gmail send failure.

#### Node: Gmail - Send Quote
- **Type / Role:** `n8n-nodes-base.gmail` — sends outgoing email.
- **Configuration (interpreted):**
  - To: `{{$json.body.recipient_email}}`
  - Subject: `Quote for {{company_name}} - {{service_type}}`
  - Body: `{{$json.body.final_email_body}}`
  - Email type: text
- **Connections:**
  - Output → `Set Sent Status`
- **Credentials:** Gmail OAuth2 “nick@reprisesai.com”
- **Version notes:** typeVersion **2.1**
- **Edge cases:**
  - Gmail sending limits / rate limits
  - Invalid recipient address
  - The body might include unsafe/unintended content if dashboard doesn’t sanitize edits

#### Node: Set Sent Status
- **Type / Role:** `n8n-nodes-base.code` — prepares Supabase update payload.
- **Configuration (interpreted):**
  - `id` from `$('Webhook - Send Quote').first().json.body.quote_id`
  - Sets `status: 'SENT'` and `sent_at: ISO timestamp`
- **Connections:**
  - Output → `Supabase - Mark Sent`
- **Version notes:** typeVersion **2**
- **Edge cases:** depends on node name `Webhook - Send Quote`.

#### Node: Supabase - Mark Sent
- **Type / Role:** `n8n-nodes-base.supabase` — updates record status and sent_at.
- **Configuration (interpreted):**
  - Operation: update
  - Filter: `id == {{$json.id}}`
  - Updates: `status`, `sent_at`
- **Connections:**
  - Output → `Respond - Success`
- **Version notes:** typeVersion **1**
- **Edge cases:** RLS/permissions; id not found.

#### Node: Respond - Success
- **Type / Role:** `n8n-nodes-base.respondToWebhook` — returns HTTP response to dashboard.
- **Configuration (interpreted):**
  - JSON response includes quote_id, status, and recipient email (from `Webhook - Send Quote`)
- **Connections:** none (terminal)
- **Version notes:** typeVersion **1.1**
- **Edge cases:** if earlier nodes fail, dashboard may not receive structured response.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Workflow Overview | Sticky Note | Documentation / overview |  |  | ## 🎯 AI QUOTE SYSTEM - COMPLETE WORKFLOW … Credentials Needed: Gmail OAuth2, OpenAI API, Supabase API, Slack API (optional) |
| 📧 Email Input | Sticky Note | Documentation (email path) |  |  | ## 📧 EMAIL INPUT PATH … |
| Gmail Trigger | gmailTrigger | Poll Gmail INBOX every minute |  | OpenAI - Email Filter | ## 📧 EMAIL INPUT PATH … |
| Sticky Note | Sticky Note | External reference |  |  | @[youtube](sAGuFV60Big) |
| 🌐 Form Input | Sticky Note | Documentation (form path) |  |  | ## 🌐 WEBSITE FORM INPUT … `https://your-n8n.com/webhook/quote-request` … |
| Webhook - Form Submission | webhook | Receive website form submissions |  | Merge - Both Paths; Respond - Form OK | ## 🌐 WEBSITE FORM INPUT … |
| Respond - Form OK | respondToWebhook | Respond immediately to form POST | Webhook - Form Submission |  | ## 🌐 WEBSITE FORM INPUT … |
| 🤖 AI Filter | Sticky Note | Documentation (email filter) |  |  | ## 🤖 AI EMAIL FILTER … Threshold: confidence >= 60 |
| OpenAI - Email Filter | @n8n/n8n-nodes-langchain.openAi | Classify email as quote request | Gmail Trigger | Parse Filter Response | ## 🤖 AI EMAIL FILTER … |
| Parse Filter Response | code | Parse LLM JSON; attach original email | OpenAI - Email Filter | IF - Is Quote Request? | ## 🤖 AI EMAIL FILTER … |
| ⚡ Routing | Sticky Note | Documentation (routing) |  |  | ## ⚡ ROUTING DECISION … (Note: “Archive email” not implemented) |
| IF - Is Quote Request? | if | Gate by is_quote_request & confidence | Parse Filter Response | Merge - Both Paths (true) | ## ⚡ ROUTING DECISION … |
| 🔀 Merge Sources | Sticky Note | Documentation (merge) |  |  | ## 🔀 MERGE DATA SOURCES … |
| Merge - Both Paths | merge | Join email-accepted + form inputs | IF - Is Quote Request?; Webhook - Form Submission | Normalize Data | ## 🔀 MERGE DATA SOURCES … |
| Normalize Data | code | Normalize email/form payload into common schema | Merge - Both Paths | OpenAI - Extract & Classify | ## 🔀 MERGE DATA SOURCES … |
| 🧠 Extraction | Sticky Note | Documentation (extraction/classification) |  |  | ## 🧠 AI EXTRACTION & CLASSIFICATION … |
| OpenAI - Extract & Classify | @n8n/n8n-nodes-langchain.openAi | Extract structured fields & scores | Normalize Data | Parse Extraction | ## 🧠 AI EXTRACTION & CLASSIFICATION … |
| Parse Extraction | code | Parse extraction JSON + add metadata | OpenAI - Extract & Classify | Prepare Insert | ## 🧠 AI EXTRACTION & CLASSIFICATION … |
| 💾 Save Initial | Sticky Note | Documentation (Supabase insert) |  |  | ## 💾 SAVE INITIAL RECORD … |
| Prepare Insert | code | Map extracted data for DB insert | Parse Extraction | Supabase - Save Initial | ## 💾 SAVE INITIAL RECORD … |
| Supabase - Save Initial | supabase | Insert quote request record | Prepare Insert | Supabase - Search Similar | ## 💾 SAVE INITIAL RECORD … |
| 📊 Historical | Sticky Note | Documentation (historical lookup) |  |  | ## 📊 SEARCH SIMILAR QUOTES … (Implementation only filters status=WON) |
| Supabase - Search Similar | supabase | Fetch last 10 WON quotes | Supabase - Save Initial | OpenAI - Calculate Pricing | ## 📊 SEARCH SIMILAR QUOTES … |
| 💰 Pricing | Sticky Note | Documentation (pricing) |  |  | ## 💰 AI PRICING CALCULATION … |
| OpenAI - Calculate Pricing | @n8n/n8n-nodes-langchain.openAi | Compute Good/Better/Best pricing | Supabase - Search Similar | Parse Pricing | ## 💰 AI PRICING CALCULATION … |
| Parse Pricing | code | Parse pricing JSON; attach record_id | OpenAI - Calculate Pricing | OpenAI - Generate Draft | ## 💰 AI PRICING CALCULATION … |
| ✉️ Draft Email | Sticky Note | Documentation (draft email) |  |  | ## ✉️ GENERATE DRAFT EMAIL … |
| OpenAI - Generate Draft | @n8n/n8n-nodes-langchain.openAi | Draft quote email body | Parse Pricing | Prepare Update | ## ✉️ GENERATE DRAFT EMAIL … |
| Prepare Update | code | Prepare DB update incl. draft/prices | OpenAI - Generate Draft | Supabase - Update Complete | ## 🔄 UPDATE DATABASE … |
| 🔄 Update DB | Sticky Note | Documentation (Supabase update) |  |  | ## 🔄 UPDATE DATABASE … |
| Supabase - Update Complete | supabase | Update record: prices, draft, status | Prepare Update | IF - High Value? | ## 🔄 UPDATE DATABASE … |
| 🔔 Slack Alert | Sticky Note | Documentation (Slack) |  |  | ## 🔔 SLACK NOTIFICATION … |
| IF - High Value? | if | Gate Slack alert for value_estimate >= 10k | Supabase - Update Complete | Slack - High Value Alert (true) | ## 🔔 SLACK NOTIFICATION … |
| Slack - High Value Alert | slack | Post high-value alert to Slack | IF - High Value? |  | ## 🔔 SLACK NOTIFICATION … |
| 📤 Send Webhook | Sticky Note | Documentation (approve & send) |  |  | ## 📤 SEND QUOTE WEBHOOK … |
| Webhook - Send Quote | webhook | Receive approved final email payload |  | Gmail - Send Quote | ## 📤 SEND QUOTE WEBHOOK … |
| Gmail - Send Quote | gmail | Send final quote email | Webhook - Send Quote | Set Sent Status | ## 📤 SEND QUOTE WEBHOOK … |
| Set Sent Status | code | Prepare SENT status update payload | Gmail - Send Quote | Supabase - Mark Sent | ## 📤 SEND QUOTE WEBHOOK … |
| Supabase - Mark Sent | supabase | Update record: status=SENT, sent_at | Set Sent Status | Respond - Success | ## 📤 SEND QUOTE WEBHOOK … |
| Respond - Success | respondToWebhook | Return success JSON to dashboard | Supabase - Mark Sent |  | ## 📤 SEND QUOTE WEBHOOK … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials**
   1. Gmail OAuth2 credential with an account allowed to read inbox and send email.
   2. OpenAI API credential (for the LangChain OpenAI node).
   3. Supabase API credential (URL + service role key or a key permitted by your RLS policies).
   4. (Optional) Slack OAuth2 credential with `chat:write` and access to the target channel.

2. **Create the Email intake path**
   1. Add **Gmail Trigger**:
      - Poll times: **Every minute**
      - Filters: `labelIds = ["INBOX"]`
      - Set `simple = false`
      - Connect Gmail OAuth2 credential
   2. Add **OpenAI (LangChain) node** named `OpenAI - Email Filter`:
      - Model: `gpt-4o`
      - Temperature: `0.2`
      - Message prompt: ask for JSON with `is_quote_request`, `confidence`, `detected_signals`, `reasoning`, using email subject/body/from.
   3. Add **Code** node `Parse Filter Response`:
      - Parse the OpenAI output as JSON (strip markdown fences)
      - Attach `original_email` from `$('Gmail Trigger').first().json`
   4. Add **IF** node `IF - Is Quote Request?`:
      - Conditions (AND):
        - Boolean equals: `{{$json.is_quote_request}} == true`
        - Number gte: `{{$json.confidence}} >= 60`
      - Keep only the **true** path connected forward (false path optional).

3. **Create the Form intake path**
   1. Add **Webhook** node `Webhook - Form Submission`:
      - HTTP Method: POST
      - Path: `quote-request`
      - Response mode: `Respond to Webhook` node
   2. Add **Respond to Webhook** node `Respond - Form OK`:
      - Respond with JSON: `{ "success": true, "message": "Form received" }`
   3. Ensure the Webhook node output also continues to the main pipeline (next steps).

4. **Merge both sources**
   1. Add **Merge** node `Merge - Both Paths`:
      - Connect:
        - From `IF - Is Quote Request?` **true** output → Merge input 0
        - From `Webhook - Form Submission` → Merge input 1
      - (Recommended) Set merge mode explicitly to **Append** (to avoid waiting for both inputs).
   2. Add **Code** node `Normalize Data`:
      - For email items with `original_email`: set `source=email`, extract `raw_content`, `contact_email`, `subject`, `email_id`.
      - For form items: set `source=form`, store raw JSON string, map key fields (name/email/phone/company/service_type/budget/timeline/message).
      - Output a single normalized object.

5. **AI extraction & scoring**
   1. Add **OpenAI** node `OpenAI - Extract & Classify`:
      - Model `gpt-4o`, temp `0.2`
      - Prompt: request strict JSON with extracted fields and classification scores.
   2. Add **Code** node `Parse Extraction`:
      - Parse JSON
      - Merge in `source/raw_content/email_id/received_at` from `$('Normalize Data')`
      - Provide fallback defaults on parse errors.

6. **Insert initial record into Supabase**
   1. Add **Code** node `Prepare Insert`:
      - Map extracted fields to the DB schema
      - Set `status = "PROCESSING"`
   2. Add **Supabase** node `Supabase - Save Initial`:
      - Table: `quote_requests`
      - Operation: Insert (create)
      - Data: auto-map from input
      - Connect Supabase credential

7. **Fetch historical won quotes**
   1. Add **Supabase** node `Supabase - Search Similar`:
      - Operation: Get All
      - Table: `quote_requests`
      - Limit: 10
      - Filter string: `=status=eq.WON&order=created_at.desc`
      - Enable “Always output data”
      - (Optional improvement) add `service_type=eq.{{...}}` to truly match same service.

8. **AI pricing**
   1. Add **OpenAI** node `OpenAI - Calculate Pricing`:
      - Model `gpt-4o`, temp `0.3`
      - Prompt includes extracted complexity/urgency/value/timeline plus historical quotes JSON.
      - Require strict JSON: `good_price`, `better_price`, `best_price`, `recommended_tier`, `pricing_rationale`.
   2. Add **Code** node `Parse Pricing`:
      - Parse pricing JSON
      - Pull `record_id` from `$('Supabase - Save Initial')`
      - Attach `extracted` from `$('Parse Extraction')`
      - Provide fallback pricing if parsing fails.

9. **Draft the quote email**
   1. Add **OpenAI** node `OpenAI - Generate Draft`:
      - Model `gpt-4o`, temp `0.7`
      - Prompt: write only the email body (no subject/markdown/JSON), include 3 tiers and CTA, under 300 words.
   2. Add **Code** node `Prepare Update`:
      - Extract draft text
      - Set `status = "PENDING_REVIEW"`
      - Set `priority_score = qualification_score`
      - Output `{ id, good_price, better_price, best_price, draft_email, priority_score, status, extracted }`

10. **Update Supabase record**
   1. Add **Supabase** node `Supabase - Update Complete`:
      - Operation: Update
      - Filter: `id eq {{$json.id}}`
      - Fields: `good_price`, `better_price`, `best_price`, `draft_email`, `priority_score`, `status="PENDING_REVIEW"`

11. **Slack alert for high-value deals (optional)**
   1. Add **IF** node `IF - High Value?`:
      - Condition: `{{$('Prepare Update').first().json.extracted.value_estimate}} >= 10000`
   2. Add **Slack** node `Slack - High Value Alert`:
      - Auth: OAuth2
      - Channel: select desired channel
      - Message: include extracted details and a link to your dashboard.

12. **Create “Approve & Send” webhook path**
   1. Add **Webhook** node `Webhook - Send Quote`:
      - Method: POST
      - Path: `send-quote`
      - Response mode: response node
      - Expected body: `quote_id`, `recipient_email`, `company_name`, `service_type`, `final_email_body`
   2. Add **Gmail** node `Gmail - Send Quote`:
      - To: `{{$json.body.recipient_email}}`
      - Subject: `Quote for {{$json.body.company_name}} - {{$json.body.service_type}}`
      - Message: `{{$json.body.final_email_body}}`
      - Type: text
   3. Add **Code** node `Set Sent Status`:
      - Output `{ id: quote_id, status:'SENT', sent_at: nowISO }`
   4. Add **Supabase** node `Supabase - Mark Sent`:
      - Update `quote_requests` where `id eq {{$json.id}}`
      - Set `status`, `sent_at`
   5. Add **Respond to Webhook** node `Respond - Success`:
      - Return JSON confirming sent status and recipient.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| @[youtube](sAGuFV60Big) | Embedded sticky note link in the workflow |
| Webhook (form intake): `https://your-n8n.com/webhook/quote-request` | Documented in the form input sticky note (replace domain with your instance) |
| Dashboard link placeholder: `https://your-replit-dashboard.com` | Used in Slack message; replace with your actual dashboard URL |
| “Archive email” mentioned but not implemented | Routing sticky note says to archive non-quotes; there is no Gmail action node connected on the false branch |
| Historical search note vs implementation mismatch | Sticky says “same service_type”; query currently filters only `status=WON` and orders by `created_at` |