Coordinate move-out cleaning and repair tasks with Google Sheets, Slack, email and Claude

https://n8nworkflows.xyz/workflows/coordinate-move-out-cleaning-and-repair-tasks-with-google-sheets--slack--email-and-claude-12429


# Coordinate move-out cleaning and repair tasks with Google Sheets, Slack, email and Claude

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow coordinates move-out cleaning and repair tasks by pulling tenant/property data from Google Sheets, generating tailored move-out instructions and vendor checklists using Claude (Anthropic), notifying vendors via email, alerting the property management team via Slack, and logging vendor confirmations back into Google Sheets. If a vendor reports a delay, it generates escalation recommendations and posts a delay alert to Slack.

**Target use cases:**
- Property managers coordinating move-out turnovers (cleaning + repairs).
- Standardizing tenant instructions and vendor checklists with AI.
- Centralizing task status updates and delays with lightweight vendor reporting.

### 1.1 Scheduled Task Initiation (daily)
Runs on a schedule, loads configuration, and retrieves tenant/property rows from Google Sheets.

### 1.2 AI Generation (instructions + checklist)
Uses an Anthropic chat model (Claude) to produce structured tenant instructions and vendor task checklist.

### 1.3 Notifications (vendor email + team Slack)
Formats outputs and sends:
- Email to vendor(s)
- Slack notification to the property management team

### 1.4 Vendor Confirmation Intake + Logging
Receives vendor POST confirmations via webhook and appends/updates task completion status in Google Sheets.

### 1.5 Delay Detection + Escalation
If status is “delayed”, Claude suggests follow-up actions and the workflow posts a Slack delay alert.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Task Initiation (daily)

**Overview:**  
Triggers every day at a defined hour, sets key configuration variables, then pulls tenant/property data from Google Sheets for downstream AI generation.

**Nodes involved:**  
- Schedule Trigger  
- Workflow Configuration  
- Get Tenant & Property Info

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based workflow entry point.
- **Configuration (interpreted):** Runs daily at **09:00** (based on `triggerAtHour: 9`).
- **Connections:**
  - Output → **Workflow Configuration**
- **Edge cases / failures:**
  - n8n timezone settings can shift “9:00” relative to business locale.
  - If downstream credentials fail, the trigger still fires but executions error.

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — centralizes reusable constants.
- **Configuration (interpreted):** Sets variables (placeholders in your JSON):
  - `daysBeforeMoveOut` (number) — *currently placeholder; not used elsewhere in this workflow as provided.*
  - `googleSheetId` (string) — spreadsheet ID containing tenant data.
  - `vendorEmailAddress` (string) — vendor destination email.
  - `slackChannel` (string) — Slack channel ID for notifications.
  - Includes other incoming fields (`includeOtherFields: true`).
- **Key expressions/variables used:**
  - Variables referenced later as: `$('Workflow Configuration').first().json.<field>`
- **Connections:**
  - Input ← Schedule Trigger
  - Output → Get Tenant & Property Info
- **Edge cases / failures:**
  - Placeholder values must be replaced or expressions that depend on them will break (Sheets/Slack/Gmail).
  - If multiple items flow in, `.first()` may unintentionally ignore other configuration items (usually fine for config).

#### Node: Get Tenant & Property Info
- **Type / role:** `n8n-nodes-base.googleSheets` — reads tenant/unit rows from a Google Sheet.
- **Configuration (interpreted):**
  - **Document ID:** from config: `={{ $('Workflow Configuration').first().json.googleSheetId }}`
  - **Sheet name:** placeholder string; must be set to your tenant data tab.
  - **Range:** autodetected (`detectAutomatically`).
  - **Credentials:** “Google Sheets account 3” (OAuth2).
- **Connections:**
  - Input ← Workflow Configuration
  - Output → Generate Move-Out Instructions
- **Edge cases / failures:**
  - OAuth token expiration / permission errors.
  - Missing or mismatched column names: downstream nodes expect fields like `unit`, `tenantName`, `leaseEndDate`, `moveOutDate`.
  - Large sheets: performance/timeouts depending on n8n limits and row count.

---

### Block 2 — AI Generation (instructions + checklist)

**Overview:**  
For each tenant/unit item, the workflow prompts Claude to produce structured move-out instructions and a vendor checklist.

**Nodes involved:**  
- Generate Move-Out Instructions  
- Anthropic Chat Model

#### Node: Anthropic Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` — provides the language model backend for the agent node.
- **Configuration (interpreted):**
  - Model: `claude-sonnet-4-5-20250929` (shown as “Claude Sonnet 4.5”)
- **Connections:**
  - Output (AI language model) → Generate Move-Out Instructions (as `ai_languageModel`)
- **Version-specific notes:**
  - Requires n8n’s LangChain-based AI nodes and valid Anthropic credentials configured in n8n.
- **Edge cases / failures:**
  - Auth / billing limits / rate limits from Anthropic.
  - Model name availability may vary across Anthropic/n8n versions.

#### Node: Generate Move-Out Instructions
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — agent that formats and generates content using the provided LLM.
- **Configuration (interpreted):**
  - **Prompt text** (built from the Google Sheets row):
    - `Unit: {{$json.unit}}, Tenant: {{$json.tenantName}}, Lease End Date: {{$json.leaseEndDate}}, Move-Out Date: {{$json.moveOutDate}}`
  - **System message:** instructs the model to:
    1. Generate tenant instructions
    2. Create vendor repair/cleaning checklist
    3. Cover common areas (kitchen/bathroom/floors/walls/appliances)
    4. Prioritize by urgency and estimated completion time
    5. Return structured sections: tenant instructions + vendor checklist
  - **Model dependency:** receives `ai_languageModel` from Anthropic Chat Model node.
- **Outputs:**
  - Produces an `output` field (used later as email body).
- **Connections:**
  - Input ← Get Tenant & Property Info
  - AI Model input ← Anthropic Chat Model
  - Output → Structure Tasks for Vendors
- **Edge cases / failures:**
  - If sheet row fields are empty/misnamed, prompt becomes incomplete.
  - Output formatting is AI-generated; it may vary unless you enforce a strict schema (e.g., JSON).
  - Long outputs may exceed Gmail/Slack practical limits if not constrained.

---

### Block 3 — Notifications (vendor email + team Slack)

**Overview:**  
Takes the AI output, packages it into an email and Slack message, then notifies the vendor and the internal property management channel.

**Nodes involved:**  
- Structure Tasks for Vendors  
- Send Email to Vendors  
- Notify Property Management Team

#### Node: Structure Tasks for Vendors
- **Type / role:** `n8n-nodes-base.set` — maps AI output + config into email and Slack fields.
- **Configuration (interpreted):**
  - `vendorEmail` = `={{ $('Workflow Configuration').first().json.vendorEmailAddress }}`
  - `emailSubject` = `Move-Out Cleaning & Repair Tasks - Unit <unit>`
  - `emailBody` = `={{ $json.output }}` (AI agent output)
  - `slackMessage` = `Move-out tasks initiated for Unit <unit> - Lease ends on <leaseEndDate>`
- **Key expressions:**
  - Unit and lease end date pulled using `.first()` from **Get Tenant & Property Info**:
    - `$('Get Tenant & Property Info').first().json.unit`
    - `$('Get Tenant & Property Info').first().json.leaseEndDate`
- **Connections:**
  - Input ← Generate Move-Out Instructions
  - Output → Send Email to Vendors
  - Output → Notify Property Management Team
- **Edge cases / failures:**
  - Use of `.first()` can mismatch when multiple tenant rows are processed: Slack/email subject may always reference the first row rather than the current item. Prefer `$json.unit` within the per-item context if processing multiple rows.
  - If AI output is empty, emailBody will be blank.

#### Node: Send Email to Vendors
- **Type / role:** `n8n-nodes-base.gmail` — sends vendor email.
- **Configuration (interpreted):**
  - To: `={{ $json.vendorEmail }}`
  - Subject: `={{ $json.emailSubject }}`
  - Body: `={{ $json.emailBody }}`
- **Credentials:** Gmail OAuth2 credentials must be configured in n8n (not shown in node’s credential block in the JSON snippet, but required to run).
- **Connections:**
  - Input ← Structure Tasks for Vendors
- **Edge cases / failures:**
  - Gmail OAuth expiration or insufficient scopes.
  - Sending limits/quota.
  - If `vendorEmailAddress` placeholder not replaced, sendTo will be invalid.

#### Node: Notify Property Management Team
- **Type / role:** `n8n-nodes-base.slack` — posts Slack notification to a channel.
- **Configuration (interpreted):**
  - Sends `text`: `={{ $json.slackMessage }}`
  - Target: channel ID from config: `={{ $('Workflow Configuration').first().json.slackChannel }}`
- **Credentials:** Slack OAuth credentials required (configured in n8n).
- **Connections:**
  - Input ← Structure Tasks for Vendors
- **Edge cases / failures:**
  - Slack channel ID invalid or bot not invited to channel.
  - Rate limits if many rows/items.

---

### Block 4 — Vendor Confirmation Intake + Logging

**Overview:**  
Provides a webhook endpoint for vendors (or an external system) to report task completion status, then logs/updates that status in Google Sheets.

**Nodes involved:**  
- Vendor Confirmation Webhook  
- Log Task Completion

#### Node: Vendor Confirmation Webhook
- **Type / role:** `n8n-nodes-base.webhook` — second workflow entry point (HTTP POST).
- **Configuration (interpreted):**
  - Method: **POST**
  - Path: `/vendor-confirmation`
  - Response mode: **lastNode** (returns data from the last executed node)
- **Expected inbound payload (implied by expressions):**
  - `body.unit`
  - `body.status` (e.g., `completed`, `delayed`)
  - `body.notes`
- **Connections:**
  - Output → Log Task Completion
- **Edge cases / failures:**
  - If vendor payload structure differs, downstream expressions like `$json.body.unit` will be undefined.
  - No authentication/verification is configured here; endpoint could be abused unless protected (e.g., secret token, basic auth, reverse proxy).

#### Node: Log Task Completion
- **Type / role:** `n8n-nodes-base.googleSheets` — appends or updates a log row in Sheets.
- **Configuration (interpreted):**
  - Operation: **appendOrUpdate**
  - Document ID: from config: `={{ $('Workflow Configuration').first().json.googleSheetId }}`
  - Sheet name: placeholder; must be set to a “task completions” logging tab.
  - Mapped columns:
    - `unit` = `={{ $json.body.unit }}`
    - `taskStatus` = `={{ $json.body.status }}`
    - `vendorNotes` = `={{ $json.body.notes }}`
    - `completionDate` = `={{ $now() }}`
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account 3”).
- **Connections:**
  - Input ← Vendor Confirmation Webhook
  - Output → Check Task Delays
- **Edge cases / failures:**
  - appendOrUpdate requires a matching key strategy (depends on sheet structure); if not configured properly in the sheet, you may get duplicates or failed updates.
  - If the logging sheet lacks these columns, mapping may fail or write to wrong headers.

---

### Block 5 — Delay Detection + Escalation

**Overview:**  
If the vendor reports status “delayed”, the workflow asks Claude for escalation steps and posts a prioritized alert to Slack.

**Nodes involved:**  
- Check Task Delays  
- Suggest Follow-Up Actions  
- Anthropic Chat Model1  
- Send Follow-Up Alert

#### Node: Check Task Delays
- **Type / role:** `n8n-nodes-base.if` — conditional branching.
- **Configuration (interpreted):**
  - Condition: `$json.body.status` **equals** `"delayed"`
- **Connections:**
  - Input ← Log Task Completion
  - **True output** → Suggest Follow-Up Actions
  - (No false-path node configured; non-delayed events simply end after this node.)
- **Edge cases / failures:**
  - Case sensitivity: `"Delayed"` or `"DELAYED"` will not match.
  - If `body.status` missing, condition evaluates false and no follow-up happens.

#### Node: Anthropic Chat Model1
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` — second LLM provider instance for escalation agent.
- **Configuration (interpreted):**
  - Same model: `claude-sonnet-4-5-20250929`
- **Connections:**
  - Output (AI language model) → Suggest Follow-Up Actions (as `ai_languageModel`)
- **Edge cases / failures:**
  - Same Anthropic auth/rate-limit concerns as earlier.

#### Node: Suggest Follow-Up Actions
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — generates escalation plan.
- **Configuration (interpreted):**
  - Prompt text includes:
    - Unit: `{{ $json.body.unit }}`
    - Reason: `{{ $json.body.notes }}`
    - Original completion date: `{{ $json.completionDate }}`
  - System message requests:
    - follow-up actions
    - alternative vendors
    - revised timeline
    - risks/impacts
    - prioritized recommendations
- **Connections:**
  - Input ← Check Task Delays (true branch)
  - AI Model input ← Anthropic Chat Model1
  - Output → Send Follow-Up Alert
- **Edge cases / failures:**
  - `completionDate` may not exist in the incoming item at this point (depends on Google Sheets node output). If missing, the prompt may include an empty value.
  - Free-form output again can vary; consider requiring a strict structure if automation relies on it.

#### Node: Send Follow-Up Alert
- **Type / role:** `n8n-nodes-base.slack` — posts delay alert to Slack.
- **Configuration (interpreted):**
  - Text expression:
    - `⚠️ TASK DELAY ALERT - Unit <unit>\n\n<AI output>`
  - Unit is referenced as:
    - `$('Log Task Completion').first().json.body.unit`
  - Channel ID from config:
    - `={{ $('Workflow Configuration').first().json.slackChannel }}`
- **Connections:**
  - Input ← Suggest Follow-Up Actions
- **Edge cases / failures:**
  - The unit reference uses `.first()` and also assumes `Log Task Completion` output contains `body.unit`. Depending on the Google Sheets node output structure, `body` might not exist there. Safer: use `{{$json.body.unit}}` from the current item if still present, or pass unit explicitly in a Set node.
  - Slack message length limits if AI output is long.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Daily scheduled entry point | — | Workflow Configuration | ## Trigger, Configuration, Log Data |
| Workflow Configuration | n8n-nodes-base.set | Stores config (sheet ID, vendor email, Slack channel, etc.) | Schedule Trigger | Get Tenant & Property Info | ## Trigger, Configuration, Log Data |
| Get Tenant & Property Info | n8n-nodes-base.googleSheets | Reads tenant/unit data from Google Sheets | Workflow Configuration | Generate Move-Out Instructions | ## Trigger, Configuration, Log Data |
| Generate Move-Out Instructions | @n8n/n8n-nodes-langchain.agent | Uses Claude to generate tenant instructions + vendor checklist | Get Tenant & Property Info | Structure Tasks for Vendors | ## AI Logic |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude model provider for instruction generation | — | Generate Move-Out Instructions (ai_languageModel) | ## AI Logic |
| Structure Tasks for Vendors | n8n-nodes-base.set | Builds email/slack fields from AI output + config | Generate Move-Out Instructions | Send Email to Vendors; Notify Property Management Team | ## Notify |
| Send Email to Vendors | n8n-nodes-base.gmail | Emails vendor checklist/instructions | Structure Tasks for Vendors | — | ## Notify |
| Notify Property Management Team | n8n-nodes-base.slack | Posts Slack message that move-out tasks started | Structure Tasks for Vendors | — | ## Notify |
| Vendor Confirmation Webhook | n8n-nodes-base.webhook | HTTP entry point for vendor completion/delay updates | — | Log Task Completion | ## Trigger, Configuration, Log Data |
| Log Task Completion | n8n-nodes-base.googleSheets | Appends/updates completion status in Google Sheets | Vendor Confirmation Webhook | Check Task Delays | ## Trigger, Configuration, Log Data |
| Check Task Delays | n8n-nodes-base.if | Branches only when status == delayed | Log Task Completion | Suggest Follow-Up Actions (true branch) | ## Trigger, Configuration, Log Data |
| Suggest Follow-Up Actions | @n8n/n8n-nodes-langchain.agent | Uses Claude to propose escalation actions for delays | Check Task Delays (true) | Send Follow-Up Alert | ## AI Logic |
| Anthropic Chat Model1 | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude model provider for delay escalation | — | Suggest Follow-Up Actions (ai_languageModel) | ## AI Logic |
| Send Follow-Up Alert | n8n-nodes-base.slack | Posts delay alert + AI recommendations to Slack | Suggest Follow-Up Actions | — | ## Notify |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note (not executed) | — | — | ## Main  This workflow automatically triages tenant complaints using AI and routes high-priority issues to property managers immediately while logging all complaints for reporting. Medium and low-priority issues are acknowledged and scheduled for follow-up. This helps property management teams respond quickly, maintain tenant satisfaction, and prevent missed complaints.  ## Setup  1. Connect your form or tenant portal to the Webhook trigger. 2. Add credentials for Slack, Email, Google Sheets, and AI. 3. Customize AI classification prompts to match your complaint categories. 4. Test each routing path (High / Medium / Low) before going live. 5. Adjust task management or follow-up logic according to team workflow. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Section header note (not executed) | — | — | ## Trigger, Configuration, Log Data |
| Sticky Note2 | n8n-nodes-base.stickyNote | Section header note (not executed) | — | — | ## AI Logic |
| Sticky Note3 | n8n-nodes-base.stickyNote | Section header note (not executed) | — | — | ## Notify |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: *Automated Move-Out Cleaning and Repair Task Management* (or your preferred name).
- Ensure n8n has access to:
  - Google Sheets OAuth2 credentials
  - Slack OAuth credentials
  - Gmail OAuth2 credentials
  - Anthropic credentials (for LangChain Anthropic node)

2) **Add “Schedule Trigger”**
- Node type: **Schedule Trigger**
- Set it to run **daily at 09:00** (match your timezone).

3) **Add “Workflow Configuration” (Set node)**
- Node type: **Set**
- Add fields (as workflow constants):
  - `daysBeforeMoveOut` (Number) — optional (present but unused in this JSON)
  - `googleSheetId` (String) — your spreadsheet ID
  - `vendorEmailAddress` (String)
  - `slackChannel` (String) — Slack channel ID
- Turn on “Include Other Fields” if available in your n8n version.
- Connect: **Schedule Trigger → Workflow Configuration**

4) **Add “Get Tenant & Property Info” (Google Sheets)**
- Node type: **Google Sheets**
- Credential: select your Google Sheets OAuth2 credential
- Configure:
  - Document: use expression `{{$('Workflow Configuration').first().json.googleSheetId}}`
  - Sheet name: your tenant data tab (e.g., `Tenants`)
  - Range: auto-detect (or specify)
- Connect: **Workflow Configuration → Get Tenant & Property Info**
- Ensure your sheet has columns like: `unit`, `tenantName`, `leaseEndDate`, `moveOutDate`.

5) **Add “Anthropic Chat Model”**
- Node type: **Anthropic Chat Model** (LangChain)
- Choose model: `claude-sonnet-4-5-20250929` (or closest available)
- Configure Anthropic credentials in n8n settings for this node.

6) **Add “Generate Move-Out Instructions” (Agent)**
- Node type: **AI Agent** (LangChain Agent)
- Set **Prompt Type** to “Define” (or equivalent)
- Prompt text expression (using current item):
  - `Unit: {{$json.unit}}, Tenant: {{$json.tenantName}}, Lease End Date: {{$json.leaseEndDate}}, Move-Out Date: {{$json.moveOutDate}}`
- System message: paste the workflow’s system instructions (tenant instructions + vendor checklist + prioritization).
- Connect data: **Get Tenant & Property Info → Generate Move-Out Instructions**
- Connect model: **Anthropic Chat Model (ai_languageModel) → Generate Move-Out Instructions**

7) **Add “Structure Tasks for Vendors” (Set)**
- Node type: **Set**
- Add fields:
  - `vendorEmail` = `{{$('Workflow Configuration').first().json.vendorEmailAddress}}`
  - `emailSubject` = `Move-Out Cleaning & Repair Tasks - Unit {{$('Get Tenant & Property Info').first().json.unit}}`
  - `emailBody` = `{{$json.output}}`
  - `slackMessage` = `Move-out tasks initiated for Unit {{$('Get Tenant & Property Info').first().json.unit}} - Lease ends on {{$('Get Tenant & Property Info').first().json.leaseEndDate}}`
- Connect: **Generate Move-Out Instructions → Structure Tasks for Vendors**
  - (If processing multiple rows, consider using `$json.unit` instead of `.first()`.)

8) **Add “Send Email to Vendors” (Gmail)**
- Node type: **Gmail**
- Credential: your Gmail OAuth2 credential
- Configure:
  - To: `{{$json.vendorEmail}}`
  - Subject: `{{$json.emailSubject}}`
  - Message/body: `{{$json.emailBody}}`
- Connect: **Structure Tasks for Vendors → Send Email to Vendors**

9) **Add “Notify Property Management Team” (Slack)**
- Node type: **Slack**
- Credential: your Slack OAuth credential
- Operation: post message to channel (as supported by your node version)
- Configure:
  - Channel: expression `{{$('Workflow Configuration').first().json.slackChannel}}`
  - Text: `{{$json.slackMessage}}`
- Connect: **Structure Tasks for Vendors → Notify Property Management Team**

10) **Add “Vendor Confirmation Webhook”**
- Node type: **Webhook**
- Method: **POST**
- Path: `vendor-confirmation`
- Response: **Last Node**
- This becomes a second entry point to the workflow.
- Expected POST JSON body example (vendor/system must send):
  - `{"unit":"A-101","status":"completed","notes":"All done"}`

11) **Add “Log Task Completion” (Google Sheets)**
- Node type: **Google Sheets**
- Credential: Google Sheets OAuth2
- Operation: **Append or Update**
- Document ID expression: `{{$('Workflow Configuration').first().json.googleSheetId}}`
- Sheet name: your logging tab (e.g., `TaskLog`)
- Map columns:
  - `unit` = `{{$json.body.unit}}`
  - `taskStatus` = `{{$json.body.status}}`
  - `vendorNotes` = `{{$json.body.notes}}`
  - `completionDate` = `{{$now()}}`
- Connect: **Vendor Confirmation Webhook → Log Task Completion**

12) **Add “Check Task Delays” (IF)**
- Node type: **IF**
- Condition: `{{$json.body.status}}` equals `delayed`
- Connect: **Log Task Completion → Check Task Delays**

13) **Add “Anthropic Chat Model1”**
- Node type: **Anthropic Chat Model** (second instance is fine)
- Same model selection and credentials.

14) **Add “Suggest Follow-Up Actions” (Agent)**
- Node type: **AI Agent**
- Prompt text (as in JSON):
  - `Task delayed for Unit: {{ $json.body.unit }}, Reason: {{ $json.body.notes }}, Original completion date: {{ $json.completionDate }}`
- System message: paste the escalation instruction block (follow-up actions, alternative vendors, revised timeline, risks).
- Connect: **Check Task Delays (true output) → Suggest Follow-Up Actions**
- Connect model: **Anthropic Chat Model1 (ai_languageModel) → Suggest Follow-Up Actions**

15) **Add “Send Follow-Up Alert” (Slack)**
- Node type: **Slack**
- Channel: `{{$('Workflow Configuration').first().json.slackChannel}}`
- Text:
  - `⚠️ TASK DELAY ALERT - Unit {{$('Log Task Completion').first().json.body.unit}}\n\n{{$json.output}}`
- Connect: **Suggest Follow-Up Actions → Send Follow-Up Alert**

16) **Activate and test**
- Test scheduled path by running manually with sample sheet rows.
- Test webhook path by sending a POST to `/webhook/vendor-confirmation` (test URL) then to production URL after activation.
- Confirm Sheets logging works and Slack alerts appear for `status="delayed"`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The “Main” sticky note content describes *tenant complaint triage* (High/Medium/Low), which does not match this move-out coordination workflow. Consider updating it to avoid operator confusion. | Sticky note labeled “Main” |
| `daysBeforeMoveOut` is defined but not used in any condition/filter. If the intention is to only trigger for leases ending soon, add a date filter step after reading the sheet. | Workflow Configuration variable |
| Multiple `.first()` references can cause incorrect unit/lease date when multiple tenant rows are processed in one run. Prefer per-item fields (`$json.unit`, `$json.leaseEndDate`) or split executions by row. | Structure Tasks for Vendors; Send Follow-Up Alert |
| The vendor webhook has no authentication in the provided configuration. Add a shared secret token, header check, or other verification to prevent spoofing. | Vendor Confirmation Webhook |