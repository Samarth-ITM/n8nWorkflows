Flag duplicate and risky AP invoices with Gmail, OpenAI and Supabase

https://n8nworkflows.xyz/workflows/flag-duplicate-and-risky-ap-invoices-with-gmail--openai-and-supabase-14146


# Flag duplicate and risky AP invoices with Gmail, OpenAI and Supabase

# 1. Workflow Overview

This workflow monitors a Gmail inbox for incoming invoice emails, uses OpenAI to extract structured invoice data, checks Supabase for duplicate invoices and vendor history, performs an AI-based fraud/risk assessment, and then routes risky invoices to Slack and email notifications before logging the outcome.

Its primary use cases are:

- Accounts payable invoice intake
- Duplicate invoice detection
- Vendor anomaly screening
- AI-assisted fraud/risk triage
- Audit logging into Supabase

## 1.1 Input Reception

The workflow starts from a Gmail trigger that polls for newly received emails. These emails are assumed to contain invoice details either in the message body or attachments.

## 1.2 Invoice Data Extraction with AI

The email content is sent to an AI Agent backed by OpenAI GPT-4o and a structured output schema. The agent extracts normalized invoice fields such as invoice number, vendor name, amount, dates, currency, line items, and whether the email appears to be an invoice at all.

## 1.3 Duplicate Detection

The extracted invoice number is used to query the `invoices` table in Supabase. A Code node then collapses the returned rows into a single item and computes a `duplicate_count`.

## 1.4 Vendor History Enrichment

The workflow queries the `vendors` table in Supabase using the extracted vendor name. A second Code node merges vendor history with the invoice context and computes features such as whether the vendor is known, whether it has been flagged before, and the deviation from average invoice amount.

## 1.5 Fraud Risk Assessment with AI

A second AI Agent evaluates the enriched invoice context and produces a structured fraud/risk assessment, including a risk level, score, indicators, reasoning, recommended action, and whether the invoice should be held.

## 1.6 Conditional Routing and Notifications

An IF node checks whether the AI marked the invoice as requiring a hold. If yes, the workflow posts a Slack alert and emails a hold notice to the AP team. If not, it skips directly to logging.

## 1.7 Logging

All processed invoices eventually reach the final Supabase logging node. High-risk invoices reach it after alerting; lower-risk invoices reach it directly.

---

# 2. Block-by-Block Analysis

## Block 1: Intake and Triggering

### Overview
This block detects new invoice emails in Gmail and starts the workflow. It is the only execution entry point in this workflow.

### Nodes Involved
- Gmail — Invoice Inbox

### Node Details

#### Gmail — Invoice Inbox
- **Type and technical role:** `n8n-nodes-base.gmailTrigger`  
  Polling trigger that starts the workflow when new emails arrive.
- **Configuration choices:**
  - Polling mode set to every minute
  - `includeSpamTrash` is disabled, so spam and trash are excluded
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input: none, this is the entry node
  - Output: `Extract Invoice Data`
- **Version-specific requirements:**
  - Type version `1.2`
  - Requires a Gmail OAuth2 credential in n8n
- **Edge cases or potential failure types:**
  - Gmail OAuth expiration or invalid credential
  - Gmail quota/rate limits
  - Email body may not contain enough information for extraction
  - Attachments are not explicitly parsed by downstream nodes; only email fields like `text` or `snippet` are referenced in prompts
- **Sub-workflow reference:** None

---

## Block 2: AI-Based Invoice Extraction

### Overview
This block sends the incoming email content to an AI Agent that extracts structured invoice information. It uses GPT-4o and a structured schema to normalize the output.

### Nodes Involved
- Extract Invoice Data
- OpenAI — Extract
- Invoice Data Schema

### Node Details

#### Extract Invoice Data
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI Agent node that prompts the language model to extract structured invoice fields.
- **Configuration choices:**
  - Prompt includes:
    - sender: `{{ $json.from }}`
    - subject: `{{ $json.subject }}`
    - body: `{{ $json.text || $json.snippet }}`
  - System message instructs the model to:
    - behave as an AP assistant
    - extract invoice numbers, vendor names, numeric amounts, ISO dates
    - return `is_invoice = false` if the message does not look like an invoice
  - Structured output is enabled through an output parser
  - Intermediate steps are disabled
- **Key expressions or variables used:**
  - `{{ $json.from }}`
  - `{{ $json.subject }}`
  - `{{ $json.text || $json.snippet }}`
- **Input and output connections:**
  - Main input: `Gmail — Invoice Inbox`
  - AI language model input: `OpenAI — Extract`
  - AI output parser input: `Invoice Data Schema`
  - Main output: `Check Duplicates`
- **Version-specific requirements:**
  - Type version `1.8`
  - Requires LangChain-compatible AI nodes available in the n8n instance
- **Edge cases or potential failure types:**
  - Missing or empty message body
  - Hallucinated fields if invoice content is ambiguous
  - Attachments are mentioned in notes but not explicitly read in the prompt
  - If schema validation fails, execution may error depending on parser behavior
  - `invoice_number`, `vendor_name`, `amount`, `currency`, and `is_invoice` are required by schema, so extraction may fail on very poor inputs
- **Sub-workflow reference:** None

#### OpenAI — Extract
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Language model provider for the extraction agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0` for deterministic extraction
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output via `ai_languageModel` to `Extract Invoice Data`
- **Version-specific requirements:**
  - Type version `1.2`
  - Requires OpenAI credentials/API key
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model access restrictions
  - OpenAI rate limiting or timeout
- **Sub-workflow reference:** None

#### Invoice Data Schema
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Validates and structures the extraction output.
- **Configuration choices:**
  - Manual JSON schema with fields:
    - `invoice_number` string
    - `vendor_name` string
    - `amount` number
    - `currency` string
    - `invoice_date` string
    - `due_date` string
    - `line_items` array of objects
    - `is_invoice` boolean
  - Required:
    - `invoice_number`
    - `vendor_name`
    - `amount`
    - `currency`
    - `is_invoice`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output via `ai_outputParser` to `Extract Invoice Data`
- **Version-specific requirements:**
  - Type version `1.2`
- **Edge cases or potential failure types:**
  - Required fields missing
  - Type mismatches, especially if amount is returned as text instead of number
  - Non-ISO dates despite system instruction
- **Sub-workflow reference:** None

---

## Block 3: Duplicate Invoice Detection

### Overview
This block checks whether the extracted invoice number already exists in Supabase. It then converts potentially multiple database rows into a single item containing `duplicate_count`.

### Nodes Involved
- Check Duplicates
- Count Duplicates

### Node Details

#### Check Duplicates
- **Type and technical role:** `n8n-nodes-base.supabase`  
  Queries the `invoices` table for records matching the extracted invoice number.
- **Configuration choices:**
  - Operation: `getAll`
  - Table: `invoices`
  - Return all matches
  - Filter:
    - `invoice_number = {{ $json.output.invoice_number }}`
  - Retries enabled:
    - max tries: 3
    - wait between tries: 2000 ms
  - `onError: continueErrorOutput`
- **Key expressions or variables used:**
  - `{{ $json.output.invoice_number }}`
- **Input and output connections:**
  - Input: `Extract Invoice Data`
  - Output: `Count Duplicates`
- **Version-specific requirements:**
  - Type version `1`
  - Requires Supabase credentials/API configuration in n8n
- **Edge cases or potential failure types:**
  - Missing `output.invoice_number`
  - Wrong table name or missing column
  - Supabase auth or network issues
  - Because `continueErrorOutput` is enabled, downstream logic must tolerate error items
- **Sub-workflow reference:** None

#### Count Duplicates
- **Type and technical role:** `n8n-nodes-base.code`  
  Aggregates Supabase query results into one item and computes the duplicate count.
- **Configuration choices:**
  - JavaScript logic:
    - Reads all incoming rows from `Check Duplicates`
    - Reads original invoice extraction from `Extract Invoice Data`
    - Filters out rows where `r.json.error` exists
    - Returns one merged object with `duplicate_count`
- **Key expressions or variables used:**
  - `$('Extract Invoice Data').first().json.output || {}`
  - `$input.all()`
- **Input and output connections:**
  - Input: `Check Duplicates`
  - Output: `Check Vendor History`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - If `Extract Invoice Data` produced no `output`, the fallback `{}` prevents total failure but may propagate incomplete data
  - If Supabase returns an error item only, duplicate count becomes `0`
  - If multiple invoice items are processed concurrently, cross-item references using `.first()` may not behave as intended for batching
- **Sub-workflow reference:** None

---

## Block 4: Vendor History Enrichment

### Overview
This block looks up the vendor in Supabase and enriches the invoice with vendor profile information and amount deviation calculations. The result becomes the full context used for risk scoring.

### Nodes Involved
- Check Vendor History
- Merge All Context

### Node Details

#### Check Vendor History
- **Type and technical role:** `n8n-nodes-base.supabase`  
  Queries the `vendors` table for history associated with the extracted vendor name.
- **Configuration choices:**
  - Operation: `getAll`
  - Table: `vendors`
  - Return all matches
  - Filter:
    - `vendor_name = {{ $json.vendor_name }}`
  - Retries enabled:
    - max tries: 3
    - wait between tries: 2000 ms
  - `onError: continueErrorOutput`
- **Key expressions or variables used:**
  - `{{ $json.vendor_name }}`
- **Input and output connections:**
  - Input: `Count Duplicates`
  - Output: `Merge All Context`
- **Version-specific requirements:**
  - Type version `1`
  - Requires Supabase credentials/API configuration
- **Edge cases or potential failure types:**
  - Exact match lookup may fail due to vendor naming variations
  - Missing `vendor_name`
  - Table/column mismatch in Supabase
  - Error items may still reach downstream nodes
- **Sub-workflow reference:** None

#### Merge All Context
- **Type and technical role:** `n8n-nodes-base.code`  
  Combines invoice extraction, duplicate count, and vendor profile into one consolidated record.
- **Configuration choices:**
  - Reads all vendor rows
  - Reads invoice context from `Count Duplicates`
  - Uses the first vendor match, or `null` if none
  - Outputs:
    - `vendor_known`
    - `vendor_avg_amount`
    - `vendor_flagged`
    - `vendor_total_invoices`
    - `vendor_last_invoice_date`
    - `amount_deviation_pct`
  - Deviation formula:
    - `((invoice amount - avg_amount) / avg_amount) * 100`
    - rounded to nearest integer
- **Key expressions or variables used:**
  - `$('Count Duplicates').first().json`
  - `$input.all()`
- **Input and output connections:**
  - Input: `Check Vendor History`
  - Output: `Assess Fraud Risk`
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - If `vendor.avg_amount` is null or `<= 0`, deviation is set to `null`
  - If multiple vendor rows exist, only the first is used
  - If amount is missing or non-numeric, deviation may become `NaN`
  - Cross-item `.first()` behavior can be problematic in multi-item runs
- **Sub-workflow reference:** None

---

## Block 5: AI Fraud Risk Assessment

### Overview
This block evaluates the merged invoice context and outputs a structured fraud/risk decision. It uses GPT-4o and a structured schema designed for AP fraud analysis.

### Nodes Involved
- Assess Fraud Risk
- OpenAI — Risk
- Fraud Risk Schema

### Node Details

#### Assess Fraud Risk
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI Agent that scores the invoice’s fraud/payment-error risk.
- **Configuration choices:**
  - Prompt includes:
    - invoice number
    - vendor
    - amount/currency
    - invoice and due dates
    - duplicate count
    - vendor known / flagged state
    - average amount
    - deviation percent
    - historical invoice volume
  - System instructions define heuristics:
    - >50% deviation = high risk
    - >100% deviation = critical
    - unknown vendors and flagged vendors increase concern
  - Structured output parser enabled
  - Intermediate steps disabled
- **Key expressions or variables used:**
  - `{{ $json.invoice_number }}`
  - `{{ $json.vendor_name }}`
  - `{{ $json.amount }}`
  - `{{ $json.currency }}`
  - `{{ $json.invoice_date }}`
  - `{{ $json.due_date }}`
  - `{{ $json.duplicate_count }}`
  - `{{ $json.vendor_known }}`
  - `{{ $json.vendor_flagged }}`
  - `{{ $json.vendor_avg_amount }}`
  - `{{ $json.amount_deviation_pct }}`
  - `{{ $json.vendor_total_invoices }}`
- **Input and output connections:**
  - Main input: `Merge All Context`
  - AI language model input: `OpenAI — Risk`
  - AI output parser input: `Fraud Risk Schema`
  - Main output: `High Risk?`
- **Version-specific requirements:**
  - Type version `1.8`
  - Requires LangChain/OpenAI support
- **Edge cases or potential failure types:**
  - Model may interpret incomplete context inconsistently
  - If required schema fields are absent, parser can fail
  - Prompt references suspicious timing patterns, but the workflow does not explicitly compute them
- **Sub-workflow reference:** None

#### OpenAI — Risk
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Language model provider for risk scoring.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output via `ai_languageModel` to `Assess Fraud Risk`
- **Version-specific requirements:**
  - Type version `1.2`
  - Requires OpenAI credentials/API key
- **Edge cases or potential failure types:**
  - API auth issues
  - Rate limit or timeout
  - Model unavailability
- **Sub-workflow reference:** None

#### Fraud Risk Schema
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a predictable risk-assessment structure.
- **Configuration choices:**
  - Manual schema with:
    - `risk_level`: enum `low|medium|high|critical`
    - `risk_score`: integer 0–100
    - `fraud_indicators`: array of strings
    - `reasoning`: string
    - `recommended_action`: string
    - `requires_hold`: boolean
  - All fields required
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Output via `ai_outputParser` to `Assess Fraud Risk`
- **Version-specific requirements:**
  - Type version `1.2`
- **Edge cases or potential failure types:**
  - Parser failure if the model emits invalid enum values
  - Risk score outside allowed bounds
  - Missing arrays or booleans
- **Sub-workflow reference:** None

---

## Block 6: Conditional Routing and Escalation

### Overview
This block decides whether the assessed invoice needs to be held. Held invoices trigger Slack and Gmail notifications before being logged.

### Nodes Involved
- High Risk?
- Alert Slack
- Send Hold Notice

### Node Details

#### High Risk?
- **Type and technical role:** `n8n-nodes-base.if`  
  Routes items depending on whether the AI says the invoice requires a hold.
- **Configuration choices:**
  - Loose type validation enabled
  - Condition:
    - `{{ $json.output.requires_hold }} == true`
- **Key expressions or variables used:**
  - `{{ $json.output.requires_hold }}`
- **Input and output connections:**
  - Input: `Assess Fraud Risk`
  - True output: `Alert Slack`
  - False output: `Log Invoice to Supabase`
- **Version-specific requirements:**
  - Type version `2.2`
- **Edge cases or potential failure types:**
  - If `output.requires_hold` is missing, false branch may be taken unintentionally
  - Loose validation may coerce string values like `"true"`
- **Sub-workflow reference:** None

#### Alert Slack
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a fraud alert to Slack when an invoice requires a hold.
- **Configuration choices:**
  - Message text includes:
    - risk level
    - invoice number
    - vendor
    - amount and currency
    - risk score
    - fraud indicators
    - reasoning
    - recommended action
  - Retries enabled:
    - max tries: 3
    - 2000 ms delay
  - `onError: continueErrorOutput`
- **Key expressions or variables used:**
  - `{{ $('Merge All Context').first().json.risk_level.toUpperCase() }}`
  - `{{ $('Merge All Context').first().json.invoice_number }}`
  - `{{ $('Merge All Context').first().json.vendor_name }}`
  - `{{ $('Merge All Context').first().json.amount }}`
  - `{{ $('Merge All Context').first().json.currency }}`
  - `{{ $json.output.risk_score }}`
  - `{{ $json.output.fraud_indicators.join('\n') }}`
  - `{{ $json.output.reasoning }}`
  - `{{ $json.output.recommended_action }}`
- **Input and output connections:**
  - Input: `High Risk?` true branch
  - Output: `Send Hold Notice`
- **Version-specific requirements:**
  - Type version `2.2`
  - Requires Slack credential
- **Edge cases or potential failure types:**
  - There is a likely expression bug:
    - `$('Merge All Context').first().json.risk_level` does not exist there
    - `risk_level` is inside `Assess Fraud Risk` output, not `Merge All Context`
  - If `fraud_indicators` is missing, `.join()` fails
  - Slack permission or channel issues
- **Sub-workflow reference:** None

#### Send Hold Notice
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an HTML email to the AP manager or review mailbox for held invoices.
- **Configuration choices:**
  - Recipient: `user@example.com`
  - Subject and HTML body include invoice and risk details
  - Retries enabled:
    - max tries: 3
    - 2000 ms delay
  - `onError: continueErrorOutput`
- **Key expressions or variables used:**
  - Subject:
    - `{{ $('Merge All Context').first().json.invoice_number }}`
    - `{{ $('Merge All Context').first().json.vendor_name }}`
    - `{{ $('Merge All Context').first().json.risk_level.toUpperCase() }}`
  - Body:
    - invoice details from `Merge All Context`
    - risk data from `Assess Fraud Risk`
    - fraud indicators with HTML `<br>`
- **Input and output connections:**
  - Input: `Alert Slack`
  - Output: `Log Invoice to Supabase`
- **Version-specific requirements:**
  - Type version `2.1`
  - Requires Gmail OAuth2 credential
- **Edge cases or potential failure types:**
  - Another likely expression bug:
    - `$('Merge All Context').first().json.risk_level` does not exist there
  - Hard-coded recipient should be replaced for production
  - HTML formatting may break if fields are null
  - Gmail sending permissions/quota issues
- **Sub-workflow reference:** None

---

## Block 7: Logging

### Overview
This block stores the final invoice processing result in Supabase. Both the high-risk and non-high-risk paths converge here.

### Nodes Involved
- Log Invoice to Supabase

### Node Details

#### Log Invoice to Supabase
- **Type and technical role:** `n8n-nodes-base.supabase`  
  Inserts a record into Supabase as the final audit/logging step.
- **Configuration choices:**
  - Operation: `insert`
  - No table or field mappings are defined in the provided JSON
- **Key expressions or variables used:** None explicitly configured
- **Input and output connections:**
  - Inputs:
    - `High Risk?` false branch
    - `Send Hold Notice`
  - Outputs: none
- **Version-specific requirements:**
  - Type version `1`
  - Requires Supabase credentials
- **Edge cases or potential failure types:**
  - This node is incomplete as shown:
    - no `tableId`
    - no mapped fields/body
  - It will likely fail unless configured manually after import
  - With `continueErrorOutput`, failures may not stop overall execution
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail — Invoice Inbox | n8n-nodes-base.gmailTrigger | Poll Gmail for new invoice emails and start the workflow |  | Extract Invoice Data | ## How It Works<br>1. Gmail Trigger detects new invoice emails (attach PDF or include invoice details in body)<br>2. AI Agent extracts: invoice number, vendor, amount, invoice date, due date, line items<br>3. Supabase checked for duplicate invoice numbers — Count Duplicates Code node prevents item multiplication<br>4. Supabase checked for vendor payment history — Merge All Context Code node consolidates everything<br>5. Second AI Agent scores fraud risk (low/medium/high/critical) with reasoning<br>6. IF risk >= high: post Slack alert + send hold notice to AP team via email<br>7. Invoice logged to Supabase invoices table regardless of risk outcome |
| Extract Invoice Data | @n8n/n8n-nodes-langchain.agent | Extract structured invoice data from email content | Gmail — Invoice Inbox | Check Duplicates | AI extracts structured invoice data from email subject, body, and attachments |
| OpenAI — Extract | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide GPT-4o model for invoice extraction |  | Extract Invoice Data |  |
| Invoice Data Schema | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured extraction output |  | Extract Invoice Data |  |
| Check Duplicates | n8n-nodes-base.supabase | Query invoices table for matching invoice number | Extract Invoice Data | Count Duplicates | Queries invoices table for existing records with same invoice number |
| Count Duplicates | n8n-nodes-base.code | Count duplicate invoice matches and collapse rows into one item | Check Duplicates | Check Vendor History | Consolidates duplicate rows into a single item, counts matches |
| Check Vendor History | n8n-nodes-base.supabase | Query vendor history from vendors table | Count Duplicates | Merge All Context | Queries vendor history to detect unusual amounts or flagged suppliers |
| Merge All Context | n8n-nodes-base.code | Merge invoice, duplicate, and vendor context into one record | Check Vendor History | Assess Fraud Risk | Merges invoice data, duplicate count, and vendor history into one item for risk assessment |
| OpenAI — Risk | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide GPT-4o model for fraud/risk scoring |  | Assess Fraud Risk |  |
| Fraud Risk Schema | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured fraud risk output |  | Assess Fraud Risk |  |
| Assess Fraud Risk | @n8n/n8n-nodes-langchain.agent | Score fraud risk and recommend action | Merge All Context | High Risk? | AI Agent scores fraud risk using duplicate count, vendor history, and amount deviation |
| High Risk? | n8n-nodes-base.if | Route invoices requiring hold vs normal logging | Assess Fraud Risk | Alert Slack; Log Invoice to Supabase | Routes to Slack alert path if risk requires hold, otherwise logs directly |
| Alert Slack | n8n-nodes-base.slack | Post a fraud alert into Slack | High Risk? | Send Hold Notice | Posts fraud alert to #invoice-alerts Slack channel with full details |
| Send Hold Notice | n8n-nodes-base.gmail | Email AP/reviewer that invoice is on hold | Alert Slack | Log Invoice to Supabase |  |
| Log Invoice to Supabase | n8n-nodes-base.supabase | Store processed invoice outcome in Supabase | High Risk?; Send Hold Notice |  | Logs every processed invoice to Supabase for record keeping |
| Overview | n8n-nodes-base.stickyNote | Documentation note |  |  | ## AP Invoice Fraud Detection<br>Version 1.0.0 — Finance<br><br>Processes incoming invoices from Gmail, extracts structured data with AI, checks Supabase for duplicate invoice numbers, validates vendor payment history, runs a risk assessment, and routes high-risk invoices to a Slack alert before logging all decisions.<br><br>Flow: Gmail Trigger => Extract Invoice Data (AI Agent) => Check Duplicates (Supabase) => Count Duplicates (Code) => Check Vendor History (Supabase) => Merge All Context (Code) => Assess Fraud Risk (AI Agent) => High Risk? (IF) => Alert Slack + Log => Log to Supabase |
| Prerequisites | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Prerequisites<br>- Gmail account receiving invoices (Gmail OAuth2 credential)<br>- OpenAI API key (GPT-4o)<br>- Supabase project with two tables:<br>  - invoices: invoice_number, vendor_name, amount, status, created_at<br>  - vendors: vendor_name, total_invoices, avg_amount, last_invoice_date, flagged<br>- Slack workspace and channel for fraud alerts (Slack OAuth2 or Bot Token)<br>- Gmail for sending hold notifications (reuse same credential) |
| Setup Required | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Setup Required<br>1. Gmail Trigger — connect Gmail OAuth2 credential<br>2. Extract Invoice Data — connect OpenAI credential<br>3. Check Duplicates + Check Vendor History — connect Supabase API credential, update table names<br>4. Assess Fraud Risk — connect OpenAI credential<br>5. Alert Slack — connect Slack credential, update #invoice-alerts channel name<br>6. Send Hold Notice — connect Gmail OAuth2 credential, update your AP manager email<br>7. Log Invoice + Log Fraud Flag — connect Supabase API credential |
| How It Works | n8n-nodes-base.stickyNote | Documentation note |  |  | ## How It Works<br>1. Gmail Trigger detects new invoice emails (attach PDF or include invoice details in body)<br>2. AI Agent extracts: invoice number, vendor, amount, invoice date, due date, line items<br>3. Supabase checked for duplicate invoice numbers — Count Duplicates Code node prevents item multiplication<br>4. Supabase checked for vendor payment history — Merge All Context Code node consolidates everything<br>5. Second AI Agent scores fraud risk (low/medium/high/critical) with reasoning<br>6. IF risk >= high: post Slack alert + send hold notice to AP team via email<br>7. Invoice logged to Supabase invoices table regardless of risk outcome |
| Extract note | n8n-nodes-base.stickyNote | Documentation note |  |  | AI extracts structured invoice data from email subject, body, and attachments |
| Duplicates note | n8n-nodes-base.stickyNote | Documentation note |  |  | Queries invoices table for existing records with same invoice number |
| Count Dup note | n8n-nodes-base.stickyNote | Documentation note |  |  | Consolidates duplicate rows into a single item, counts matches |
| Vendor note | n8n-nodes-base.stickyNote | Documentation note |  |  | Queries vendor history to detect unusual amounts or flagged suppliers |
| Merge Context note | n8n-nodes-base.stickyNote | Documentation note |  |  | Merges invoice data, duplicate count, and vendor history into one item for risk assessment |
| Risk note | n8n-nodes-base.stickyNote | Documentation note |  |  | AI Agent scores fraud risk using duplicate count, vendor history, and amount deviation |
| High Risk note | n8n-nodes-base.stickyNote | Documentation note |  |  | Routes to Slack alert path if risk requires hold, otherwise logs directly |
| Slack note | n8n-nodes-base.stickyNote | Documentation note |  |  | Posts fraud alert to #invoice-alerts Slack channel with full details |
| Log Invoice note | n8n-nodes-base.stickyNote | Documentation note |  |  | Logs every processed invoice to Supabase for record keeping |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Flag duplicate and risky AP invoices with Gmail, OpenAI and Supabase`.

2. **Add a Gmail Trigger node** named `Gmail — Invoice Inbox`.
   - Node type: `Gmail Trigger`
   - Connect a Gmail OAuth2 credential
   - Configure polling to run every minute
   - Disable spam/trash inclusion
   - This is the workflow entry point

3. **Add an AI Agent node** named `Extract Invoice Data`.
   - Node type: `AI Agent` / LangChain Agent
   - Set prompt type to define manually
   - Use this extraction prompt logic:
     - include sender
     - include subject
     - include body from `text`, fallback to `snippet`
     - ask for invoice number, vendor name, amount, currency, invoice date, due date, line items
     - instruct null for missing fields
   - Add a system message instructing:
     - precise AP extraction
     - numeric amounts only
     - ISO 8601 dates
     - `is_invoice = false` if the email is not an invoice
   - Disable intermediate steps if that option is available
   - Enable structured output parsing

4. **Add an OpenAI Chat Model node** named `OpenAI — Extract`.
   - Node type: `OpenAI Chat Model`
   - Connect OpenAI credentials
   - Select model `gpt-4o`
   - Set temperature to `0`
   - Connect this node to the AI Agent’s `ai_languageModel` input

5. **Add a Structured Output Parser node** named `Invoice Data Schema`.
   - Node type: `Structured Output Parser`
   - Use a manual schema with:
     - `invoice_number` string
     - `vendor_name` string
     - `amount` number
     - `currency` string
     - `invoice_date` string
     - `due_date` string
     - `line_items` array of objects
     - `is_invoice` boolean
   - Mark at least these as required:
     - `invoice_number`
     - `vendor_name`
     - `amount`
     - `currency`
     - `is_invoice`
   - Connect this node to the AI Agent’s `ai_outputParser` input

6. **Connect `Gmail — Invoice Inbox` to `Extract Invoice Data`** on the main flow.

7. **Add a Supabase node** named `Check Duplicates`.
   - Node type: `Supabase`
   - Connect Supabase credentials
   - Operation: `Get Many` / `Get All`
   - Table: `invoices`
   - Add filter:
     - `invoice_number` equals `{{ $json.output.invoice_number }}`
   - Enable return all matches
   - Enable retry on failure:
     - 3 tries
     - 2000 ms delay
   - Set error handling to continue to error output if available
   - Connect `Extract Invoice Data` to `Check Duplicates`

8. **Add a Code node** named `Count Duplicates`.
   - Node type: `Code`
   - Use JavaScript similar to:
     ```js
     const dupRows = $input.all();
     const invoice = $('Extract Invoice Data').first().json.output || {};
     return [{
       json: {
         ...invoice,
         duplicate_count: dupRows.filter(r => !r.json.error).length
       }
     }];
     ```
   - Connect `Check Duplicates` to `Count Duplicates`

9. **Add another Supabase node** named `Check Vendor History`.
   - Node type: `Supabase`
   - Operation: `Get Many` / `Get All`
   - Table: `vendors`
   - Add filter:
     - `vendor_name` equals `{{ $json.vendor_name }}`
   - Enable return all matches
   - Enable retry on failure:
     - 3 tries
     - 2000 ms delay
   - Set continue-on-error behavior
   - Connect `Count Duplicates` to `Check Vendor History`

10. **Add a second Code node** named `Merge All Context`.
    - Node type: `Code`
    - Use JavaScript similar to:
      ```js
      const vendorRows = $input.all();
      const invoiceCtx = $('Count Duplicates').first().json;
      const vendors = vendorRows.filter(r => !r.json.error).map(r => r.json);
      const vendor = vendors[0] || null;

      return [{
        json: {
          ...invoiceCtx,
          vendor_known: !!vendor,
          vendor_avg_amount: vendor ? vendor.avg_amount : null,
          vendor_flagged: vendor ? (vendor.flagged || false) : false,
          vendor_total_invoices: vendor ? vendor.total_invoices : 0,
          vendor_last_invoice_date: vendor ? vendor.last_invoice_date : null,
          amount_deviation_pct: vendor && vendor.avg_amount > 0
            ? Math.round(((invoiceCtx.amount - vendor.avg_amount) / vendor.avg_amount) * 100)
            : null
        }
      }];
      ```
    - Connect `Check Vendor History` to `Merge All Context`

11. **Add a second AI Agent node** named `Assess Fraud Risk`.
    - Node type: `AI Agent` / LangChain Agent
    - Prompt should include:
      - invoice number
      - vendor
      - amount and currency
      - invoice date and due date
      - duplicate count
      - vendor known
      - vendor flagged
      - vendor average amount
      - amount deviation percent
      - vendor total historical invoices
    - Ask for a fraud or payment-error risk evaluation with score and recommended action
    - Add a system message defining the policy:
      - duplicate submissions are risky
      - >50% amount deviation = high risk
      - >100% deviation = critical
      - unknown or flagged vendors increase risk
      - risk levels: low, medium, high, critical
      - include clear reasoning
    - Enable structured output parsing

12. **Add an OpenAI Chat Model node** named `OpenAI — Risk`.
    - Model: `gpt-4o`
    - Temperature: `0`
    - Connect it to `Assess Fraud Risk` through the model input

13. **Add another Structured Output Parser node** named `Fraud Risk Schema`.
    - Manual schema:
      - `risk_level`: string enum `low`, `medium`, `high`, `critical`
      - `risk_score`: integer 0–100
      - `fraud_indicators`: array of strings
      - `reasoning`: string
      - `recommended_action`: string
      - `requires_hold`: boolean
    - Make all fields required
    - Connect it to `Assess Fraud Risk` through the parser input

14. **Connect `Merge All Context` to `Assess Fraud Risk`** in the main flow.

15. **Add an IF node** named `High Risk?`.
    - Condition:
      - left value: `{{ $json.output.requires_hold }}`
      - operator: equals
      - right value: `true`
    - Loose type validation can remain enabled
    - Connect `Assess Fraud Risk` to `High Risk?`

16. **Add a Slack node** named `Alert Slack`.
    - Node type: `Slack`
    - Connect Slack OAuth2 or bot token
    - Configure it to post a message to your fraud alert channel, for example `#invoice-alerts`
    - Use a message containing:
      - risk level
      - invoice number
      - vendor
      - amount
      - risk score
      - fraud indicators
      - reasoning
      - recommended action
    - Important: correct the expression bug from the source workflow.
      - Use risk level from `Assess Fraud Risk`, not from `Merge All Context`
      - Safer example:
        - `{{ $('Assess Fraud Risk').first().json.output.risk_level.toUpperCase() }}`
    - Enable retries if desired
    - Connect the **true** branch of `High Risk?` to `Alert Slack`

17. **Add a Gmail node** named `Send Hold Notice`.
    - Node type: `Gmail`
    - Connect Gmail OAuth2 credentials
    - Operation: send email
    - Set recipient to the AP manager or shared mailbox
    - Replace the placeholder `user@example.com`
    - Configure an HTML message with:
      - invoice number
      - vendor
      - amount
      - risk level
      - risk score
      - fraud indicators
      - reasoning
      - recommended action
    - Important: correct the subject expression bug from the source workflow.
      - Use `{{ $('Assess Fraud Risk').first().json.output.risk_level.toUpperCase() }}`
      - not `{{ $('Merge All Context').first().json.risk_level.toUpperCase() }}`
    - Connect `Alert Slack` to `Send Hold Notice`

18. **Add a final Supabase node** named `Log Invoice to Supabase`.
    - Node type: `Supabase`
    - Connect Supabase credentials
    - Operation: `Insert`
    - Choose the target table, likely `invoices` or a dedicated audit/log table
    - Map fields explicitly, for example:
      - `invoice_number` ← extracted invoice number
      - `vendor_name` ← vendor name
      - `amount` ← amount
      - `status` ← risk level or approval status
      - `created_at` ← current timestamp
      - optionally add:
        - `risk_score`
        - `reasoning`
        - `requires_hold`
        - `duplicate_count`
    - Note: the provided workflow JSON does not include these mappings, so you must define them manually
    - Connect:
      - the **false** branch of `High Risk?` to `Log Invoice to Supabase`
      - `Send Hold Notice` to `Log Invoice to Supabase`

19. **Optionally add sticky notes** for documentation.
    - Include overview, prerequisites, setup notes, and block descriptions
    - These are non-executable but useful for future maintainers

20. **Create and configure credentials** before activation:
    - **Gmail OAuth2**
      - for trigger and hold notice email
    - **OpenAI**
      - with access to `gpt-4o`
    - **Supabase**
      - API URL and key
    - **Slack**
      - OAuth2 or bot token with permission to post to the selected channel

21. **Prepare Supabase tables** before testing.
    - Minimum expected tables from the notes:
      - `invoices`
        - `invoice_number`
        - `vendor_name`
        - `amount`
        - `status`
        - `created_at`
      - `vendors`
        - `vendor_name`
        - `total_invoices`
        - `avg_amount`
        - `last_invoice_date`
        - `flagged`
    - Consider adding indexes on:
      - `invoice_number`
      - `vendor_name`

22. **Test with sample emails**:
    - a normal invoice from a known vendor
    - a duplicate invoice number
    - an unknown vendor
    - a vendor with a large amount deviation
    - a non-invoice email to verify `is_invoice` handling

23. **Add production hardening**, because the imported workflow is not complete in several places:
    - Add an IF node after extraction to stop or ignore messages where `is_invoice = false`
    - Normalize vendor names before lookup
    - Explicitly parse PDF attachments if required
    - Fix Slack and email risk-level expressions
    - Complete Supabase insert mappings
    - Consider logging both the extracted invoice and the AI decision separately

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AP Invoice Fraud Detection, Version 1.0.0 — Finance. Processes incoming invoices from Gmail, extracts structured data with AI, checks Supabase for duplicate invoice numbers, validates vendor payment history, runs a risk assessment, and routes high-risk invoices to a Slack alert before logging all decisions. | Workflow overview note |
| Prerequisites: Gmail account receiving invoices, OpenAI API key, Supabase project with `invoices` and `vendors` tables, Slack workspace/channel, and Gmail for hold notifications. | Environment/setup context |
| Setup sequence noted in workflow: connect Gmail, OpenAI, Supabase, Slack, and Gmail send credentials; update table names, channel name, and AP manager email. | Configuration context |
| The workflow notes say attachments may contain invoice data, but the implemented extraction prompt only references email fields (`from`, `subject`, `text`, `snippet`). Attachment parsing is not explicitly implemented. | Important implementation gap |
| The final Supabase logging node is not fully configured in the provided JSON. You must define the destination table and field mappings manually. | Important implementation gap |
| Slack and Gmail notification expressions incorrectly reference `risk_level` from `Merge All Context`; that field exists in `Assess Fraud Risk` output instead. | Important implementation gap |