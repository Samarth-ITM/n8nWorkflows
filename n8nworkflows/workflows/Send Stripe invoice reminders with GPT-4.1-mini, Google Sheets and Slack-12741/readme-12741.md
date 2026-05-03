Send Stripe invoice reminders with GPT-4.1-mini, Google Sheets and Slack

https://n8nworkflows.xyz/workflows/send-stripe-invoice-reminders-with-gpt-4-1-mini--google-sheets-and-slack-12741


# Send Stripe invoice reminders with GPT-4.1-mini, Google Sheets and Slack

## Disclaimer (provided by user)
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---

# 1. Workflow Overview

**Workflow name (in JSON):** Automated Invoice Payment Reminder and Tracking System  
**User title:** Send Stripe invoice reminders with GPT-4.1-mini, Google Sheets and Slack

**Purpose:**  
Automatically reacts to Stripe invoice events (creation and payment failure), normalizes invoice data, generates a personalized payment reminder message using GPT-4.1-mini with a structured JSON output, routes behavior based on payment status, then executes operational actions: logging to Google Sheets, optional CRM posting, Slack alerting for overdue invoices, sending reminder emails, generating a PDF, and scheduling a follow-up event.

**Target use cases:**
- Accounts receivable automation (reminders, escalation, tracking)
- Lightweight invoice tracking in Sheets + optional CRM synchronization
- Automated internal notifications for overdue payments

## 1.1 Trigger & Configuration
Stripe event trigger → workflow-wide “constants” (company/support/CRM endpoint) → normalized invoice fields.

## 1.2 AI Message Generation (GPT) & Routing
Build an invoice summary prompt → generate structured {subject, body} → switch by payment status (Overdue/Pending/Paid).

## 1.3 Actions (Logging, Notification, Email, PDF, Calendar, CRM)
For routed invoices: append/update Sheets; then post to CRM; also (in parallel) Slack alert, email client, generate PDF, schedule follow-up.

---

# 2. Block-by-Block Analysis

## Block 2.1 — Trigger & Config

**Overview:**  
Receives invoice events from Stripe, injects workflow configuration placeholders, and reshapes Stripe’s invoice payload into consistent fields used by downstream nodes.

**Nodes involved:**
- Stripe Invoice Trigger
- Workflow Configuration
- Normalize Invoice Data

### Node: Stripe Invoice Trigger
- **Type / role:** `stripeTrigger` — entry point webhook trigger listening to Stripe events.
- **Configuration choices:**
  - Subscribed events: `invoice.created`, `invoice.payment_failed`.
  - Uses Stripe webhook infrastructure (requires Stripe credentials/configuration in n8n).
- **Key data used downstream:** Entire Stripe invoice object, including `customer`, `amount_due`, `currency`, `due_date`, `status`, `hosted_invoice_url`, `number`.
- **Connections:**
  - Output → Workflow Configuration
- **Edge cases / failures:**
  - Stripe webhook signing/verification issues.
  - Missing fields depending on Stripe settings (e.g., `customer.name` can be null, `due_date` may be absent).
  - If event payload differs between invoice events, expressions in later nodes may fail unless guarded.
- **Version notes:** typeVersion 1 (older Stripe Trigger); ensure your n8n instance supports this node version and Stripe API/webhook configuration.

### Node: Workflow Configuration
- **Type / role:** `set` — injects constants and placeholders used across the workflow.
- **Configuration choices (interpreted):**
  - Adds/overwrites fields:
    - `companyName`: placeholder
    - `supportEmail`: placeholder (used as email “from”)
    - `crmApiUrl`: placeholder (used by HTTP Request node)
  - `includeOtherFields: true` so Stripe data continues downstream.
- **Key expressions/variables:**
  - Static placeholder strings like `<__PLACEHOLDER_VALUE__Support Email Address__>`
- **Connections:**
  - Input: Stripe Invoice Trigger
  - Output: Normalize Invoice Data
- **Edge cases / failures:**
  - If placeholders aren’t replaced, downstream actions (email/CRM) will fail or behave incorrectly.

### Node: Normalize Invoice Data
- **Type / role:** `set` — maps Stripe fields into normalized names used later.
- **Configuration choices (interpreted mappings):**
  - `clientName` ← `{{$json.customer.name}}`
  - `clientEmail` ← `{{$json.customer.email}}`
  - `invoiceNumber` ← `{{$json.number}}`
  - `invoiceAmount` ← `{{$json.amount_due / 100}}` (Stripe amounts are cents)
  - `currency` ← `{{$json.currency.toUpperCase()}}`
  - `dueDate` ← `{{ new Date($json.due_date * 1000).toISOString() }}` (Stripe due_date is epoch seconds)
  - `paymentStatus` ← `{{$json.status}}`
  - `invoiceUrl` ← `{{$json.hosted_invoice_url}}`
  - Keeps other fields.
- **Connections:**
  - Input: Workflow Configuration
  - Output: Generate Payment Reminder
- **Edge cases / failures:**
  - `customer` may be an ID string in some Stripe configurations unless expanded; then `customer.name/email` won’t exist.
  - `due_date` can be null; the date expression will produce `Invalid Date`.
  - `currency` might be undefined; `.toUpperCase()` would throw.

---

## Block 2.2 — AI Logic & Routing

**Overview:**  
Creates a structured prompt from normalized invoice fields, uses GPT-4.1-mini to produce a JSON object (`subject`, `body`), then routes the invoice to action branches based on `paymentStatus`.

**Nodes involved:**
- Generate Payment Reminder (LangChain Agent)
- OpenAI Chat Model
- Structured Output Parser
- Route by Payment Status

### Node: Generate Payment Reminder
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates LLM call with system instructions + structured output parsing.
- **Configuration choices:**
  - Prompt (“text” field) contains a formatted invoice summary:
    - Client, Email, Invoice Number, Amount, Due Date, Payment Status, Invoice URL
  - System message instructs:
    - Return a *professional* subject and HTML body
    - Adjust tone based on `paymentStatus` (gentle vs urgent)
    - Output must be JSON with `subject` and `body`
  - `hasOutputParser: true` (wired to Structured Output Parser)
- **Key expressions/variables:**
  - Uses normalized fields like `{{$json.clientName}}`, `{{$json.invoiceAmount}}`, etc.
- **Connections:**
  - Main input: Normalize Invoice Data
  - AI Language Model input: OpenAI Chat Model → agent
  - AI Output Parser input: Structured Output Parser → agent
  - Main output: Route by Payment Status
- **Edge cases / failures:**
  - If `clientName`/`clientEmail` are missing, the email may be unusable; downstream nodes may fail (email send).
  - LLM may return invalid JSON or omit fields; parser will fail execution.
  - HTML in `body` may contain unsafe/unwanted markup if not constrained.

### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` — provides the chat model backing the agent.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Uses OpenAI credentials: “n8n free OpenAI API credits”
- **Connections:**
  - Output (AI language model) → Generate Payment Reminder
- **Edge cases / failures:**
  - Credential quota exhausted, invalid API key, model not available in region/account.
  - Latency/timeouts for LLM calls.

### Node: Structured Output Parser
- **Type / role:** `outputParserStructured` — enforces JSON schema on LLM output.
- **Configuration choices:**
  - Manual JSON schema with:
    - `subject`: string
    - `body`: string (described as HTML)
- **Connections:**
  - Output (AI output parser) → Generate Payment Reminder
- **Edge cases / failures:**
  - Schema is permissive (no required fields declared); however parser may still fail if output is not parseable JSON.
  - If you expect strict enforcement, consider adding `"required": ["subject","body"]` (not present here).

### Node: Route by Payment Status
- **Type / role:** `switch` — routes execution based on invoice status.
- **Configuration choices:**
  - Output branches (renamed):
    - **Overdue** if `paymentStatus` equals `overdue` OR `past_due`
    - **Pending** if equals `open` OR `pending`
    - **Paid** if equals `paid`
- **Connections:**
  - Input: Generate Payment Reminder
  - Output: (single switch output is connected to multiple action nodes in parallel)
    - Log to Google Sheets
    - Notify Accountant on Slack
    - Send Reminder Email
    - Generate Invoice PDF
    - Schedule Follow-up
- **Edge cases / failures:**
  - Stripe invoice statuses commonly include `draft`, `open`, `paid`, `uncollectible`, `void`; “overdue/past_due” may not match your Stripe status vocabulary depending on your setup. If unmatched, item may not be routed as expected.
  - All action nodes are connected directly; without per-branch connections, you must rely on the Switch node’s output mapping to ensure only the intended branch runs.

---

## Block 2.3 — Actions

**Overview:**  
Per routed invoice, the workflow logs or updates a record in Google Sheets, then posts to an external CRM endpoint. In parallel, it can alert Slack, email the client, generate a PDF file, and schedule a follow-up calendar event.

**Nodes involved:**
- Log to Google Sheets
- CRM Integration
- Notify Accountant on Slack
- Send Reminder Email
- Generate Invoice PDF
- Schedule Follow-up

### Node: Log to Google Sheets
- **Type / role:** `googleSheets` — persistent logging and idempotent update by invoice number.
- **Configuration choices:**
  - Operation: **appendOrUpdate**
  - Matching column: `invoiceNumber` (used to update existing rows instead of duplicating)
  - Mapping mode: auto-map input data
  - Sheet name: placeholder (e.g., “Invoice Log”)
  - Document ID: placeholder
- **Connections:**
  - Input: Route by Payment Status
  - Output: CRM Integration
- **Edge cases / failures:**
  - Spreadsheet schema mismatch: if `invoiceNumber` column does not exist, update/match fails.
  - OAuth token expired/insufficient permissions.
  - Auto-mapping may not map fields as expected; consider explicit mapping for reliability.

### Node: CRM Integration
- **Type / role:** `httpRequest` — posts invoice info to an external CRM endpoint.
- **Configuration choices:**
  - Method: POST
  - URL: from config `{{$('Workflow Configuration').first().json.crmApiUrl}}`
  - Body: JSON with `invoiceNumber`, `clientName`, `amount`, `currency`, `status`, `dueDate`
  - Authentication: “predefinedCredentialType” (requires an HTTP Request credential configured in n8n)
- **Connections:**
  - Input: Log to Google Sheets
  - Output: none (terminal node)
- **Edge cases / failures:**
  - Placeholder URL not replaced → request fails.
  - Auth credential missing/incorrect.
  - CRM API errors (4xx/5xx), schema expectations differ (field names/types).
  - No retry/backoff configured.

### Node: Notify Accountant on Slack
- **Type / role:** `slack` — sends internal notification for overdue invoices.
- **Configuration choices:**
  - Posts a formatted message containing client, invoice, amount, due date, status, and link.
  - Channel selection: by `channelId` placeholder.
- **Connections:**
  - Input: Route by Payment Status
  - Output: none
- **Edge cases / failures:**
  - If this runs for non-overdue statuses due to routing misconfiguration, it can spam a channel.
  - Slack credential missing or channel not found.
  - The message includes an emoji and Slack markdown; formatting may vary by Slack client.

### Node: Send Reminder Email
- **Type / role:** `emailSend` — sends the AI-generated reminder email to the client.
- **Configuration choices:**
  - `toEmail`: `{{$json.clientEmail}}`
  - `fromEmail`: `{{ $('Workflow Configuration').first().json.supportEmail }}`
  - `subject`: `{{$json.subject}}` (from structured LLM output)
  - `html`: `{{$json.body}}`
- **Connections:**
  - Input: Route by Payment Status
  - Output: none
- **Edge cases / failures:**
  - Email node requires SMTP or configured email transport in n8n; misconfiguration causes send failures.
  - Missing/invalid `clientEmail` will fail.
  - If LLM output parser fails upstream, `subject/body` won’t exist.

### Node: Generate Invoice PDF
- **Type / role:** `convertToFile` — converts HTML input into a PDF file output.
- **Configuration choices:**
  - Operation: HTML → file
  - File name: `Invoice_{{ $json.invoiceNumber }}.pdf`
- **Connections:**
  - Input: Route by Payment Status
  - Output: none
- **Edge cases / failures:**
  - This node expects HTML content in the incoming item (depending on n8n version/node behavior). In this workflow, no explicit HTML field is passed for conversion (unless you intend to use `body` or another HTML source). As wired, it may generate an empty/invalid PDF or fail.
  - If the intention is to convert the invoice page, you would typically fetch the invoice HTML first (HTTP Request) or use Stripe PDF/hosted invoice PDF URL if available.

### Node: Schedule Follow-up
- **Type / role:** `googleCalendar` — schedules a follow-up event one week later.
- **Configuration choices:**
  - `start`: now + 7 days
  - `end`: start + 30 minutes
  - Calendar ID: placeholder
  - Summary: `Follow-up: Invoice {invoiceNumber} - {clientName}`
  - Description: `Follow up on overdue invoice ...`
- **Connections:**
  - Input: Route by Payment Status
  - Output: none
- **Edge cases / failures:**
  - OAuth issues / calendar ID invalid.
  - This schedules follow-ups for all routed statuses unless routing is correctly isolated (intended likely only for overdue).
  - Timezone: uses ISO string from server time; may not match your business timezone expectations.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Stripe Invoice Trigger | Stripe Trigger | Entry point: listens for Stripe invoice events | — | Workflow Configuration | ## Trigger & Config |
| Workflow Configuration | Set | Defines constants (company/support/CRM URL) and passes through payload | Stripe Invoice Trigger | Normalize Invoice Data | ## Trigger & Config |
| Normalize Invoice Data | Set | Normalizes Stripe invoice payload into workflow fields | Workflow Configuration | Generate Payment Reminder | ## Trigger & Config |
| Generate Payment Reminder | LangChain Agent | Uses LLM to generate structured reminder content | Normalize Invoice Data | Route by Payment Status | ## AI Logic & Routing |
| OpenAI Chat Model | OpenAI Chat Model | Supplies GPT-4.1-mini to the agent | — | Generate Payment Reminder (ai_languageModel) | ## AI Logic & Routing |
| Structured Output Parser | Structured Output Parser | Enforces JSON output with subject/body | — | Generate Payment Reminder (ai_outputParser) | ## AI Logic & Routing |
| Route by Payment Status | Switch | Routes actions based on `paymentStatus` | Generate Payment Reminder | Log to Google Sheets; Notify Accountant on Slack; Send Reminder Email; Generate Invoice PDF; Schedule Follow-up | ## AI Logic & Routing |
| Log to Google Sheets | Google Sheets | Append/update invoice log keyed by invoiceNumber | Route by Payment Status | CRM Integration | ## Actions |
| CRM Integration | HTTP Request | Posts invoice info to external CRM endpoint | Log to Google Sheets | — | ## Actions |
| Notify Accountant on Slack | Slack | Alerts internal team about overdue invoices | Route by Payment Status | — | ## Actions |
| Send Reminder Email | Email Send | Emails client with GPT-generated subject + HTML body | Route by Payment Status | — | ## Actions |
| Generate Invoice PDF | Convert to File | Creates a PDF file from HTML | Route by Payment Status | — | ## Actions |
| Schedule Follow-up | Google Calendar | Creates a follow-up event for invoice | Route by Payment Status | — | ## Actions |
| Sticky Note | Sticky Note | Canvas annotation | — | — | ## Trigger & Config |
| Sticky Note1 | Sticky Note | Canvas annotation | — | — | ## AI Logic & Routing  |
| Sticky Note2 | Sticky Note | Canvas annotation | — | — | ## Actions |
| Sticky Note3 | Sticky Note | Canvas annotation | — | — | ## Main  Automates invoice generation and overdue payment reminders. AI personalizes emails, logs info, notifies accountants, generates PDFs, and schedules follow-ups.  Setup  1. Connect Stripe or accounting software trigger 2. Connect OpenAI for email generation 3. Connect Google Sheets for logging 4. Connect Slack for accountant notifications 5. Connect email for client reminders 6. Connect PDF node 7. Connect calendar for follow-ups 8. Optional CRM integration  **Author:** Hyrum Hurst, AI Automation Engineer **Company:** QuarterSmart **Contact:** hyrum@quartersmart.com |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: “Stripe Invoice Trigger” (Stripe Trigger)**
   - Events: select `invoice.created` and `invoice.payment_failed`.
   - Configure Stripe credentials / webhook connection as required by your n8n environment.
3. **Add node: “Workflow Configuration” (Set)**
   - Enable “Keep Only Set” = off / “Include Other Fields” = on (so the Stripe payload passes through).
   - Add fields:
     - `companyName` (string): your company name
     - `supportEmail` (string): a valid sender email you can send from
     - `crmApiUrl` (string): your CRM endpoint URL (optional, but node later uses it)
4. **Add node: “Normalize Invoice Data” (Set)**
   - Include other fields = on.
   - Add fields (expressions):
     - `clientName` = `{{$json.customer.name}}`
     - `clientEmail` = `{{$json.customer.email}}`
     - `invoiceNumber` = `{{$json.number}}`
     - `invoiceAmount` = `{{$json.amount_due / 100}}`
     - `currency` = `{{$json.currency.toUpperCase()}}`
     - `dueDate` = `{{ new Date($json.due_date * 1000).toISOString() }}`
     - `paymentStatus` = `{{$json.status}}`
     - `invoiceUrl` = `{{$json.hosted_invoice_url}}`
5. **Add node: “Generate Payment Reminder” (LangChain Agent)**
   - Prompt type: “Define” (custom prompt).
   - Text prompt: format invoice fields into a short record (client, email, invoice number, amount, due date, status, URL).
   - System message: instruct to produce a professional reminder and return JSON with `subject` and `body` (HTML).
   - Attach an output parser (next steps).
6. **Add node: “OpenAI Chat Model”**
   - Model: `gpt-4.1-mini`.
   - Configure OpenAI credentials (API key or n8n-provided credits).
   - Connect **OpenAI Chat Model** to **Generate Payment Reminder** via the **AI Language Model** connection type.
7. **Add node: “Structured Output Parser”**
   - Schema type: Manual.
   - Define schema with:
     - `subject`: string
     - `body`: string
   - Connect **Structured Output Parser** to **Generate Payment Reminder** via the **AI Output Parser** connection type.
8. **Add node: “Route by Payment Status” (Switch)**
   - Create rules on `{{$json.paymentStatus}}`:
     - Overdue: equals `overdue` OR `past_due`
     - Pending: equals `open` OR `pending`
     - Paid: equals `paid`
   - Connect **Generate Payment Reminder → Route by Payment Status** (main).
9. **Add node: “Log to Google Sheets” (Google Sheets)**
   - Operation: Append or Update.
   - Document: select the spreadsheet (by ID).
   - Sheet: choose sheet name (e.g., “Invoice Log”).
   - Matching column: `invoiceNumber` (ensure this column exists in the sheet).
   - Mapping: auto-map (or map explicitly if preferred).
   - Connect **Route by Payment Status → Log to Google Sheets**.
10. **Add node: “CRM Integration” (HTTP Request)**
   - Method: POST
   - URL: `{{$('Workflow Configuration').first().json.crmApiUrl}}`
   - Body type: JSON; send body enabled.
   - JSON body expression containing invoice fields (invoiceNumber, clientName, amount, currency, status, dueDate).
   - Configure HTTP credentials (the node expects “predefined credential type” authentication).
   - Connect **Log to Google Sheets → CRM Integration**.
11. **Add node: “Notify Accountant on Slack” (Slack)**
   - Operation: send message to a channel.
   - Channel: choose by ID/name.
   - Message text: include invoice details and link to `invoiceUrl`.
   - Configure Slack credentials (OAuth/bot token).
   - Connect **Route by Payment Status → Notify Accountant on Slack**.
12. **Add node: “Send Reminder Email” (Email Send)**
   - To: `{{$json.clientEmail}}`
   - From: `{{ $('Workflow Configuration').first().json.supportEmail }}`
   - Subject: `{{$json.subject}}`
   - HTML: `{{$json.body}}`
   - Configure email transport (SMTP) in n8n or node credentials as required.
   - Connect **Route by Payment Status → Send Reminder Email**.
13. **Add node: “Generate Invoice PDF” (Convert to File)**
   - Operation: HTML to File
   - Filename: `Invoice_{{ $json.invoiceNumber }}.pdf`
   - Connect **Route by Payment Status → Generate Invoice PDF**.
   - (If you actually want a meaningful PDF, ensure an HTML field is present on the incoming item—e.g., use `$json.body` or fetch invoice HTML first.)
14. **Add node: “Schedule Follow-up” (Google Calendar)**
   - Calendar: select calendar ID.
   - Start: now + 7 days (expression).
   - End: start + 30 minutes (expression).
   - Summary/Description: use invoice fields.
   - Configure Google Calendar OAuth2 credentials.
   - Connect **Route by Payment Status → Schedule Follow-up**.
15. **Add Sticky Notes** (optional, for canvas organization)
   - “Trigger & Config”, “AI Logic & Routing”, “Actions”, and the “Main/Setup/Author” note content if desired.
16. **Test with sample Stripe invoice events**
   - Verify `customer` expansion provides name/email.
   - Verify payment statuses match the Switch rules.
   - Validate Sheets column names and that append/update works as expected.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automates invoice generation and overdue payment reminders. AI personalizes emails, logs info, notifies accountants, generates PDFs, and schedules follow-ups. | Sticky note “Main” summary |
| Setup: 1) Connect Stripe trigger 2) Connect OpenAI 3) Connect Google Sheets 4) Connect Slack 5) Connect email 6) Connect PDF node 7) Connect calendar 8) Optional CRM integration | Sticky note “Main” setup list |
| Author: Hyrum Hurst, AI Automation Engineer; Company: QuarterSmart; Contact: hyrum@quartersmart.com | Sticky note “Main” credits/contact |