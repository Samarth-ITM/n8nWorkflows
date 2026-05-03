Escalate unresolved maintenance requests with GPT-4o, Google Sheets and Slack

https://n8nworkflows.xyz/workflows/escalate-unresolved-maintenance-requests-with-gpt-4o--google-sheets-and-slack-12687


# Escalate unresolved maintenance requests with GPT-4o, Google Sheets and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Escalate unresolved maintenance requests with GPT-4o, Google Sheets and Slack  
**Workflow name (in n8n):** Property Maintenance Request Tracking and Escalation System

**Purpose:**  
This workflow receives maintenance requests via an HTTP webhook, normalizes the incoming fields, uses OpenAI (GPT-4o-mini) to classify urgency (low/medium/high), logs the request into Google Sheets, waits for an SLA window, checks whether the request was resolved, and escalates unresolved items via Slack and email depending on urgency. A second entry point runs daily to email a summary of unresolved requests.

**Target use cases:**
- Property management teams tracking tenant maintenance requests
- SLA-based escalation of unresolved incidents
- Automated daily reporting of outstanding maintenance items

### 1.1 Input Reception & Configuration
Receives a maintenance request and injects workflow-wide configuration values (SLA thresholds, recipient emails, Slack channel).

### 1.2 Normalization & AI Triage
Normalizes request fields from potentially inconsistent inbound payloads, then uses GPT to classify urgency.

### 1.3 Persistence (Google Sheets) & SLA Wait
Appends the request to Google Sheets, then waits an SLA window before re-checking the request status in the sheet.

### 1.4 Resolution Check & Escalation Routing
If still unresolved, routes by urgency:
- High: Slack + manager email
- Medium: Slack only
- Low: no alert (logs a note only)

### 1.5 Daily Summary Reporting (Scheduled)
Every morning, retrieves requests from Google Sheets (intended unresolved set), aggregates them, and emails a daily summary.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Configuration
**Overview:** Accepts incoming maintenance requests via webhook and sets operational parameters (SLA hours and notification targets) used throughout the workflow.  
**Nodes involved:**  
- Maintenance Request Webhook  
- Workflow Configuration

#### Node: Maintenance Request Webhook
- **Type / role:** `Webhook` trigger; starts the “new request” flow.
- **Configuration (interpreted):**
  - HTTP Method: `POST`
  - Path: `/maintenance-request`
  - Response Mode: `Last Node` (the webhook response will be whatever the last executed node returns).
- **Key variables/expressions:** Provides inbound request data under `$json.body.*`.
- **Connections:** Outputs to **Workflow Configuration**.
- **Version-specific notes:** TypeVersion `2.1` (standard webhook node behavior).
- **Edge cases / failures:**
  - If inbound JSON body is missing/invalid, downstream normalization will default some fields but may produce “Unknown” values.
  - “Last Node” response can be delayed due to the **Wait** node (hours). In many real HTTP clients, this will timeout. Consider changing response mode to “On Received” or “Respond immediately” pattern to avoid client timeouts.

#### Node: Workflow Configuration
- **Type / role:** `Set` node; centralizes constants.
- **Configuration (interpreted):** Sets:
  - `slaHoursHigh` = 2  
  - `slaHoursMedium` = 24  
  - `slaHoursLow` = 72  
  - `managerEmail` = placeholder  
  - `slackChannel` = placeholder (Slack channel ID)  
  - `summaryEmail` = placeholder
  - “Include other fields” enabled (passes inbound data along).
- **Key variables/expressions:** None; static values.
- **Connections:** Inputs from **Maintenance Request Webhook**, outputs to **Normalize Request Fields**.
- **Edge cases / failures:**
  - Placeholder values must be replaced; otherwise Slack/email nodes will fail (invalid recipient/channel).

---

### Block 2 — Normalization & AI Triage
**Overview:** Normalizes incoming payload fields to a consistent schema and uses GPT to label urgency as low/medium/high using a constrained output prompt.  
**Nodes involved:**  
- Normalize Request Fields  
- Classify Urgency with AI

#### Node: Normalize Request Fields
- **Type / role:** `Set` node; maps multiple possible inbound field names into canonical fields.
- **Configuration (interpreted):**
  - `tenant` = `{{$json.body.tenant || $json.body.tenantName || 'Unknown'}}`
  - `unit` = `{{$json.body.unit || $json.body.unitNumber || 'Unknown'}}`
  - `issue` = `{{$json.body.issue || $json.body.description || 'No description provided'}}`
  - `submittedAt` = `{{$now.toISO()}}`
  - `requestId` = `{{$runIndex + '-' + Date.now()}}` (unique-ish identifier)
  - `status` = `"open"`
  - Includes other fields.
- **Connections:** Inputs from **Workflow Configuration**, outputs to **Classify Urgency with AI**.
- **Edge cases / failures:**
  - If webhook sends form-encoded data rather than JSON body, `$json.body.*` may be absent; results become “Unknown”.
  - `requestId` uniqueness: `Date.now()` is millisecond precision; collisions are unlikely but possible under heavy concurrency; consider UUID for robustness.

#### Node: Classify Urgency with AI
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` (OpenAI chat model); classifies urgency.
- **Configuration (interpreted):**
  - Model: `gpt-4o-mini`
  - Temperature: `0.3` (more deterministic)
  - System instructions: strict rubric and **must respond with only one word**: low/medium/high
  - User content: `Maintenance issue: {{ $json.issue }}`
  - Built-in tools: none
- **Outputs:** AI response appears in `$json.output` (as used downstream).
- **Connections:** Outputs to **Log Request to Sheets**.
- **Credentials:** OpenAI API credential (“n8n free OpenAI API credits”).
- **Edge cases / failures:**
  - Model may occasionally return extra whitespace or capitalization; workflow mitigates with `.toLowerCase().trim()` later.
  - If model returns unexpected text (e.g., “high priority”), the routing switch expects exact `high|medium|low` and may fall into fallback “none” (dropping the flow).
  - OpenAI auth/rate-limit/network errors will stop the execution unless error handling is added.

---

### Block 3 — Persistence (Google Sheets) & SLA Wait
**Overview:** Logs each request to Google Sheets as the system of record, waits for an SLA duration, then reloads the row by `requestId` to check current status.  
**Nodes involved:**  
- Log Request to Sheets  
- Wait for SLA Window  
- Check Request Status

#### Node: Log Request to Sheets
- **Type / role:** `Google Sheets` node; appends a new row.
- **Configuration (interpreted):**
  - Operation: `Append`
  - Document: placeholder Google Sheets document ID
  - Sheet name: placeholder (e.g., “Maintenance Requests”)
  - Columns appended:
    - `requestId` = `{{$json.requestId}}`
    - `tenant` = `{{$json.tenant}}`
    - `unit` = `{{$json.unit}}`
    - `issue` = `{{$json.issue}}`
    - `urgency` = `{{$json.output.toLowerCase().trim()}}`
    - `status` = `{{$json.status}}` (initially “open”)
    - `submittedAt` = `{{$json.submittedAt}}`
  - Mapping mode: auto-map input data (with provided schema)
- **Connections:** From **Classify Urgency with AI** to **Wait for SLA Window**.
- **Credentials:** Google Sheets OAuth2.
- **Edge cases / failures:**
  - Spreadsheet columns must match names; otherwise append/mapping can fail or write to wrong columns.
  - If the sheet is protected or has insufficient permissions, append fails.
  - If `urgency` ends up empty/unexpected, routing later may not match.

#### Node: Wait for SLA Window
- **Type / role:** `Wait` node; delays execution before escalation check.
- **Configuration (interpreted):**
  - Unit: `hours`
  - Amount: `{{ $('Workflow Configuration').first().json.slaHoursHigh }}`
- **Connections:** Outputs to **Check Request Status**.
- **Critical design note:** This wait always uses `slaHoursHigh` (2 hours), even for medium/low requests. The workflow defines `slaHoursMedium` and `slaHoursLow` but does not use them.
- **Edge cases / failures:**
  - Long waits keep executions pending; ensure n8n instance supports long-running waits (and that execution data pruning won’t remove needed context).
  - Combined with webhook “lastNode” response, this likely causes HTTP timeouts for callers.

#### Node: Check Request Status
- **Type / role:** `Google Sheets` node; looks up row(s) by requestId.
- **Configuration (interpreted):**
  - Filter: `requestId` equals `{{$json.requestId}}`
  - Document and sheet: placeholders (must match the logging sheet)
- **Connections:** Outputs to **Check if Resolved**.
- **Edge cases / failures:**
  - If multiple rows share the same `requestId` (shouldn’t happen), you may get multiple items downstream.
  - If no row is found, the switch node may not behave as expected (missing `status` field).

---

### Block 4 — Resolution Check & Escalation Routing
**Overview:** Determines whether the request is resolved; if not, routes by urgency and notifies via Slack and/or email.  
**Nodes involved:**  
- Check if Resolved  
- Route by Urgency  
- Send Slack Alert (High)  
- Email Manager (High)  
- Send Slack Alert (Medium)  
- Log Low Priority (No Alert)

#### Node: Check if Resolved
- **Type / role:** `Switch` node; filters unresolved requests.
- **Configuration (interpreted):**
  - Rule “unresolved”: if `{{$json.status}}` **not equals** `"resolved"`
  - Fallback output: `none` (items that do not match are dropped)
- **Connections:** “unresolved” output goes to **Route by Urgency**.
- **Edge cases / failures:**
  - Status must be updated in the sheet to exactly `resolved`. Variants like “Resolved”, “done”, or blanks will be treated as unresolved and escalated.
  - If `status` is missing, it will likely be treated as “not equals resolved” and escalated.

#### Node: Route by Urgency
- **Type / role:** `Switch` node; routes unresolved items to urgency branches.
- **Configuration (interpreted):**
  - Output `high` if `{{$json.urgency}}` equals `high`
  - Output `medium` if equals `medium`
  - Output `low` if equals `low`
  - Fallback output: `none`
- **Connections:**
  - `high` → **Send Slack Alert (High)**
  - `medium` → **Send Slack Alert (Medium)**
  - `low` → **Log Low Priority (No Alert)**
- **Edge cases / failures:**
  - If urgency was stored differently (e.g., “High”, “HIGH”, “high priority”), no branch matches and the item is dropped.

#### Node: Send Slack Alert (High)
- **Type / role:** `Slack` node; posts high-priority escalation message.
- **Configuration (interpreted):**
  - Target: Channel (by ID)
  - Channel ID: `{{ $('Workflow Configuration').first().json.slackChannel }}`
  - Message includes request fields and “UNRESOLVED after SLA window”.
- **Connections:** Outputs to **Email Manager (High)**.
- **Credentials:** Slack credential must be configured in n8n (not shown in JSON snippet as a credential object).
- **Edge cases / failures:**
  - Invalid channel ID, missing scopes (e.g., `chat:write`), or token issues will fail.
  - Message contains emoji; if your environment disallows, adjust text.

#### Node: Email Manager (High)
- **Type / role:** `Email Send` node; emails manager for high-priority unresolved requests.
- **Configuration (interpreted):**
  - To: `{{ $('Workflow Configuration').first().json.managerEmail }}`
  - From: placeholder sender email address
  - Subject: `URGENT: Unresolved High Priority Maintenance Request - Unit {{ $json.unit }}`
  - Format: text
- **Connections:** Terminal node on the high branch.
- **Credentials:** Depends on n8n email configuration (SMTP or configured email provider, depending on node setup).
- **Edge cases / failures:**
  - From address/domain may require verified SMTP settings.
  - Missing/placeholder email causes send failure.

#### Node: Send Slack Alert (Medium)
- **Type / role:** `Slack` node; posts medium-priority escalation message.
- **Configuration (interpreted):**
  - Channel ID: `{{ $('Workflow Configuration').first().json.slackChannel }}`
  - Message indicates unresolved after SLA window.
- **Connections:** Terminal node on the medium branch.
- **Edge cases / failures:** Same as high Slack node.

#### Node: Log Low Priority (No Alert)
- **Type / role:** `Set` node; annotates that no alert was sent.
- **Configuration (interpreted):**
  - Adds `escalationNote` = `"Low priority - logged only, no alert sent"`
  - Includes other fields.
- **Connections:** Terminal node on the low branch.
- **Edge cases / failures:**
  - This does not write back to Google Sheets; it only modifies execution data. If you want an audit trail in Sheets, add an update step.

---

### Block 5 — Daily Summary Reporting (Scheduled)
**Overview:** Runs every day at a set hour, fetches (intended) unresolved requests from Google Sheets, aggregates them into one item, and sends an email summary.  
**Nodes involved:**  
- Daily Summary Schedule  
- Get Unresolved Requests  
- Aggregate Unresolved Requests  
- Send Daily Summary Email

#### Node: Daily Summary Schedule
- **Type / role:** `Schedule Trigger`; starts the daily reporting flow.
- **Configuration (interpreted):**
  - Triggers daily at hour `8` (timezone depends on n8n instance settings).
- **Connections:** Outputs to **Get Unresolved Requests**.
- **Edge cases / failures:**
  - Timezone mismatch can send summaries at unexpected local time; confirm instance timezone.

#### Node: Get Unresolved Requests
- **Type / role:** `Google Sheets` node; retrieves rows based on status.
- **Configuration (interpreted):**
  - Filter UI configured as: lookupColumn `status`, lookupValue `resolved`
- **Important logic issue:** The node name indicates it should fetch *unresolved*, but the filter as configured matches `status = resolved`. For unresolved, it should either:
  - fetch all and filter out `resolved`, or
  - use a “not equals resolved” filter (if supported), or
  - use a separate column/flag.
- **Connections:** Outputs to **Aggregate Unresolved Requests**.
- **Edge cases / failures:** Same Google Sheets permission and schema concerns as above.

#### Node: Aggregate Unresolved Requests
- **Type / role:** `Aggregate` node; combines all incoming rows into one item.
- **Configuration (interpreted):**
  - Mode: `aggregateAllItemData` → outputs `{ data: [ ...all rows... ] }`
- **Connections:** Outputs to **Send Daily Summary Email**.
- **Edge cases / failures:**
  - If there are zero input items, behavior depends on node version; it may output an empty aggregation or no item. Ensure the email step can handle empty sets.

#### Node: Send Daily Summary Email
- **Type / role:** `Email Send` node; emails a daily list.
- **Configuration (interpreted):**
  - To: `{{ $('Workflow Configuration').first().json.summaryEmail }}`
  - From: placeholder sender email address
  - Subject includes date: `Daily Maintenance Request Summary - {{$now.toFormat('yyyy-MM-dd')}}`
  - Body:
    - Total Unresolved: `{{ $json.data.length }}`
    - Lists each entry via map/join with request fields
- **Connections:** Terminal node of scheduled branch.
- **Edge cases / failures:**
  - If aggregation didn’t produce `data`, the expression `$json.data.length` will fail.
  - Email sending constraints as described in the high-priority manager email node.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Maintenance Request Webhook | Webhook | Entry point for new maintenance requests | — | Workflow Configuration | ## Webook & Config |
| Workflow Configuration | Set | Defines SLA/recipients/channel constants | Maintenance Request Webhook | Normalize Request Fields | ## Webook & Config |
| Normalize Request Fields | Set | Canonicalizes inbound request fields | Workflow Configuration | Classify Urgency with AI | ## AI Logic and Urgency Level |
| Classify Urgency with AI | OpenAI (LangChain) | Classifies urgency low/medium/high | Normalize Request Fields | Log Request to Sheets | ## AI Logic and Urgency Level |
| Log Request to Sheets | Google Sheets | Appends request row to tracking sheet | Classify Urgency with AI | Wait for SLA Window | ## AI Logic and Urgency Level |
| Wait for SLA Window | Wait | Delays for SLA before status re-check | Log Request to Sheets | Check Request Status | ## AI Logic and Urgency Level |
| Check Request Status | Google Sheets | Looks up request row by requestId | Wait for SLA Window | Check if Resolved | ## AI Logic and Urgency Level |
| Check if Resolved | Switch | Filters unresolved requests | Check Request Status | Route by Urgency | ## AI Logic and Urgency Level |
| Route by Urgency | Switch | Routes unresolved by urgency | Check if Resolved | Send Slack Alert (High); Send Slack Alert (Medium); Log Low Priority (No Alert) | ## Notify |
| Send Slack Alert (High) | Slack | Posts high-priority escalation to Slack | Route by Urgency | Email Manager (High) | ## Notify |
| Email Manager (High) | Email Send | Emails manager for high-priority unresolved | Send Slack Alert (High) | — | ## Notify |
| Send Slack Alert (Medium) | Slack | Posts medium-priority escalation to Slack | Route by Urgency | — | ## Notify |
| Log Low Priority (No Alert) | Set | Adds note (no notification) | Route by Urgency | — | ## Notify |
| Daily Summary Schedule | Schedule Trigger | Daily entry point for summary | — | Get Unresolved Requests | ## Schedule Trigger |
| Get Unresolved Requests | Google Sheets | Fetches rows for summary (currently filters status=resolved) | Daily Summary Schedule | Aggregate Unresolved Requests | ## Schedule Trigger |
| Aggregate Unresolved Requests | Aggregate | Aggregates rows into a single list | Get Unresolved Requests | Send Daily Summary Email | ## Schedule Trigger |
| Send Daily Summary Email | Email Send | Emails daily summary list | Aggregate Unresolved Requests | — | ## Schedule Trigger |
| Sticky Note | Sticky Note | Documentation note block (no execution role) | — | — | ## Main  This workflow tracks maintenance requests over time and automatically escalates unresolved issues. It demonstrates SLA tracking, AI-based urgency classification, conditional escalation, and reporting.  Designed for property management teams that want visibility into delayed maintenance issues.  ## Setup  - Connect your form or tenant portal to the Webhook trigger - Set urgency classification prompt in the AI node - Configure SLA delay in the Wait node - Connect Google Sheets to track request status - Set Slack and Email recipients for escalations and summaries |
| Sticky Note1 | Sticky Note | Section label (no execution role) | — | — | ## Webook & Config |
| Sticky Note2 | Sticky Note | Section label (no execution role) | — | — | ## AI Logic and Urgency Level |
| Sticky Note3 | Sticky Note | Section label (no execution role) | — | — | ## Notify |
| Sticky Note4 | Sticky Note | Section label (no execution role) | — | — | ## Schedule Trigger |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: Webhook**
   - Name: `Maintenance Request Webhook`
   - Method: `POST`
   - Path: `maintenance-request`
   - Response mode: `Last Node` (note: may cause client timeouts due to Wait; consider changing later).
3. **Add node: Set**
   - Name: `Workflow Configuration`
   - Add fields:
     - `slaHoursHigh` (number) = `2`
     - `slaHoursMedium` (number) = `24`
     - `slaHoursLow` (number) = `72`
     - `managerEmail` (string) = your manager email
     - `slackChannel` (string) = Slack channel ID (e.g., `C01234567`)
     - `summaryEmail` (string) = team summary email
   - Enable “Include Other Fields”.
4. **Connect:** Webhook → Workflow Configuration.
5. **Add node: Set**
   - Name: `Normalize Request Fields`
   - Enable “Include Other Fields”.
   - Add fields with expressions:
     - `tenant` = `{{ $json.body.tenant || $json.body.tenantName || 'Unknown' }}`
     - `unit` = `{{ $json.body.unit || $json.body.unitNumber || 'Unknown' }}`
     - `issue` = `{{ $json.body.issue || $json.body.description || 'No description provided' }}`
     - `submittedAt` = `{{ $now.toISO() }}`
     - `requestId` = `{{ $runIndex + '-' + Date.now() }}`
     - `status` = `open`
6. **Connect:** Workflow Configuration → Normalize Request Fields.
7. **Add node: OpenAI (LangChain)**
   - Name: `Classify Urgency with AI`
   - Credentials: configure an OpenAI API key/credential.
   - Model: `gpt-4o-mini`
   - Temperature: `0.3`
   - Instructions (system): paste the rubric and enforce “ONLY one word: low, medium, or high”.
   - Input message/user content: `Maintenance issue: {{ $json.issue }}`
8. **Connect:** Normalize Request Fields → Classify Urgency with AI.
9. **Add node: Google Sheets**
   - Name: `Log Request to Sheets`
   - Credentials: Google Sheets OAuth2 account with access to the target spreadsheet.
   - Operation: `Append`
   - Select the Spreadsheet (Document ID) and Sheet name.
   - Map columns to:
     - requestId, tenant, unit, issue, urgency, status, submittedAt
   - For urgency use expression: `{{ $json.output.toLowerCase().trim() }}`
10. **Connect:** Classify Urgency with AI → Log Request to Sheets.
11. **Add node: Wait**
   - Name: `Wait for SLA Window`
   - Unit: `hours`
   - Amount: `{{ $('Workflow Configuration').first().json.slaHoursHigh }}`
   - (Optional improvement: use dynamic hours based on urgency.)
12. **Connect:** Log Request to Sheets → Wait for SLA Window.
13. **Add node: Google Sheets**
   - Name: `Check Request Status`
   - Same credentials, document, and sheet as logging.
   - Add filter: lookup column `requestId` equals `{{ $json.requestId }}`
14. **Connect:** Wait for SLA Window → Check Request Status.
15. **Add node: Switch**
   - Name: `Check if Resolved`
   - Rule: `{{ $json.status }}` **not equals** `resolved`
   - Output name: `unresolved`
   - Fallback output: `none`
16. **Connect:** Check Request Status → Check if Resolved (from main output).
17. **Add node: Switch**
   - Name: `Route by Urgency`
   - Rules:
     - if `{{ $json.urgency }}` equals `high` → output `high`
     - equals `medium` → output `medium`
     - equals `low` → output `low`
   - Fallback: `none`
18. **Connect:** Check if Resolved (unresolved output) → Route by Urgency.
19. **Add node: Slack**
   - Name: `Send Slack Alert (High)`
   - Credentials: Slack OAuth/token with `chat:write` scope.
   - Post to: Channel (by ID)
   - Channel ID: `{{ $('Workflow Configuration').first().json.slackChannel }}`
   - Message: include request fields (ID, unit, tenant, issue, submittedAt) and unresolved notice.
20. **Add node: Email Send**
   - Name: `Email Manager (High)`
   - Configure email credentials (SMTP or provider supported by your n8n).
   - To: `{{ $('Workflow Configuration').first().json.managerEmail }}`
   - From: your sender address
   - Subject: `URGENT: Unresolved High Priority Maintenance Request - Unit {{ $json.unit }}`
   - Text body: include request details.
21. **Connect:** Route by Urgency (high) → Send Slack Alert (High) → Email Manager (High).
22. **Add node: Slack**
   - Name: `Send Slack Alert (Medium)`
   - Same Slack setup/channel expression
   - Message tailored for medium priority.
23. **Connect:** Route by Urgency (medium) → Send Slack Alert (Medium).
24. **Add node: Set**
   - Name: `Log Low Priority (No Alert)`
   - Add field `escalationNote` = `Low priority - logged only, no alert sent`
   - Include other fields.
25. **Connect:** Route by Urgency (low) → Log Low Priority (No Alert).
26. **Add node: Schedule Trigger**
   - Name: `Daily Summary Schedule`
   - Set to run daily at 08:00.
27. **Add node: Google Sheets**
   - Name: `Get Unresolved Requests`
   - Same document/sheet/credentials.
   - Configure filter to retrieve unresolved requests.
   - (As built in JSON, it filters `status = resolved`; correct this to match your definition of unresolved.)
28. **Add node: Aggregate**
   - Name: `Aggregate Unresolved Requests`
   - Mode: “Aggregate All Item Data”.
29. **Add node: Email Send**
   - Name: `Send Daily Summary Email`
   - To: `{{ $('Workflow Configuration').first().json.summaryEmail }}`
   - Subject includes date: `Daily Maintenance Request Summary - {{ $now.toFormat('yyyy-MM-dd') }}`
   - Body: reference aggregated array `{{$json.data}}` and render each request.
30. **Connect scheduled branch:** Daily Summary Schedule → Get Unresolved Requests → Aggregate Unresolved Requests → Send Daily Summary Email.
31. **Activate credentials & replace placeholders**
   - Replace all placeholder sheet IDs/names, emails, Slack channel ID, and sender address.
32. **Test**
   - Send a POST request to `/maintenance-request` with JSON body containing at least `issue`, optionally `tenant`, `unit`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow tracks maintenance requests over time and automatically escalates unresolved issues. It demonstrates SLA tracking, AI-based urgency classification, conditional escalation, and reporting. Designed for property management teams that want visibility into delayed maintenance issues. | Sticky note “Main” |
| Setup: Connect your form or tenant portal to the Webhook trigger; set urgency classification prompt in the AI node; configure SLA delay in the Wait node; connect Google Sheets to track request status; set Slack and Email recipients for escalations and summaries. | Sticky note “Main” |
| Logic caveat: Wait node uses `slaHoursHigh` for all requests; medium/low SLA values are defined but unused. | Workflow design observation |
| Logic caveat: “Get Unresolved Requests” currently filters `status = resolved`, which contradicts its intent and the email text (“unresolved”). | Workflow design observation |
| Operational caveat: Webhook response mode is “last node” but the flow includes hour-long wait; most clients will timeout. | Integration concern |

