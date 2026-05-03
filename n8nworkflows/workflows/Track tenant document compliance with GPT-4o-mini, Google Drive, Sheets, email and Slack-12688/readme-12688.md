Track tenant document compliance with GPT-4o-mini, Google Drive, Sheets, email and Slack

https://n8nworkflows.xyz/workflows/track-tenant-document-compliance-with-gpt-4o-mini--google-drive--sheets--email-and-slack-12688


# Track tenant document compliance with GPT-4o-mini, Google Drive, Sheets, email and Slack

## 1. Workflow Overview

**Purpose:** This workflow tracks tenant document compliance for property management. It ingests document-upload events, stores uploaded documents in Google Drive, uses **GPT-4o-mini** to classify whether the document is **COMPLETE** or **INCOMPLETE**, logs the result to Google Sheets, sends reminder emails for incomplete documents, and escalates unresolved cases to Slack after a waiting period. Separately, it runs a weekly scheduled report email summarizing tracked data.

**Target use cases:**
- Automating compliance checks for tenant-provided documents (IDs, leases, insurance, income proof, etc.)
- Maintaining an auditable compliance log in Google Sheets
- Automated tenant follow-ups and escalation to a team Slack channel
- Weekly reporting to a compliance mailbox/stakeholder

### 1.1 Webhook Intake & Configuration
Receives upload metadata via a webhook and loads workflow-wide configuration values (Drive folder, Sheet ID, Slack channel, report email).

### 1.2 Tenant/Data Normalization
Maps incoming webhook payload into consistent internal fields (tenantId, tenantName, etc.) and stamps an upload timestamp.

### 1.3 Storage + AI Compliance Classification
Uploads a file (or at least sets a Drive file name) to Google Drive, then prompts GPT-4o-mini to return only **COMPLETE** or **INCOMPLETE**.

### 1.4 Logging + Notification/Escalation Path
Writes/updates a row in Google Sheets, switches on AI output, sends reminder email for incomplete documents, waits 7 days, then escalates to Slack.

### 1.5 Weekly Compliance Reporting
On a weekly schedule, reads the “Document Log” sheet, aggregates all rows, and emails a weekly report.

---

## 2. Block-by-Block Analysis

### Block 1 — Webhook Intake & Workflow Configuration
**Overview:** Accepts a POST request from a document upload form/system and initializes key configuration constants used across the workflow.

**Nodes involved:**
- Document Upload Webhook
- Workflow Configuration

#### Node: Document Upload Webhook
- **Type / role:** `Webhook` (entry point). Receives tenant/document metadata via HTTP POST.
- **Key configuration:**
  - **Path:** `tenant-document-upload`
  - **Method:** `POST`
  - **Response mode:** `Last node` (the workflow response will come from the last executed node in this execution path).
- **Inputs/Outputs:**
  - **No input** (trigger node)
  - Output goes to **Workflow Configuration**
- **Key data expectations (from expressions downstream):**
  - `body.tenantId`, `body.tenantName`, `body.tenantEmail`, `body.documentType`, `body.documentName`
- **Edge cases / failures:**
  - Missing `body.*` fields will cause later expressions to evaluate to `undefined`, leading to bad filenames, blank emails, or logging issues.
  - No file binary is shown being received/handled; if you expect actual file content, the webhook must be configured to accept binary and subsequent Drive node must upload binary data.
  - Because response is “lastNode”, long-running paths (e.g., **Wait 7 days**) are not suitable for synchronous webhook response in typical client expectations.

#### Node: Workflow Configuration
- **Type / role:** `Set` node. Stores central configuration values for reuse.
- **Configuration choices:**
  - Sets:
    - `driveFolderId` (placeholder)
    - `sheetsDocumentId` (placeholder)
    - `slackChannel` (placeholder)
    - `complianceReportEmail` (placeholder)
  - **Include other fields:** enabled (keeps webhook payload available).
- **Inputs/Outputs:**
  - Input from **Document Upload Webhook**
  - Output to **Normalize Tenant Data**
- **Key expressions/variables used downstream:**
  - Referenced via: `$('Workflow Configuration').first().json.<field>`
- **Edge cases / failures:**
  - Placeholders must be replaced with real IDs; otherwise Drive/Sheets/Slack operations will fail (invalid ID / not found).
  - If multiple items ever pass through, `.first()` may not reflect the intended config item; here it’s safe because the trigger produces one item.

**Sticky note context:** “## Webhook & Workflow Config” (covers this block).

---

### Block 2 — Tenant/Data Normalization
**Overview:** Converts incoming webhook body fields into normalized top-level fields and adds a timestamp for tracking and escalation context.

**Nodes involved:**
- Normalize Tenant Data

#### Node: Normalize Tenant Data
- **Type / role:** `Set` node. Maps and standardizes fields.
- **Configuration choices:**
  - Sets:
    - `tenantId` = `{{$json.body.tenantId}}`
    - `tenantName` = `{{$json.body.tenantName}}`
    - `tenantEmail` = `{{$json.body.tenantEmail}}`
    - `documentType` = `{{$json.body.documentType}}`
    - `documentName` = `{{$json.body.documentName}}`
    - `uploadTimestamp` = `{{$now.toISO()}}`
  - **Include other fields:** enabled.
- **Inputs/Outputs:**
  - Input from **Workflow Configuration**
  - Output to **Upload to Google Drive**
- **Edge cases / failures:**
  - If webhook sends different field names (e.g., `tenant_id`), these will be empty.
  - `documentName` should be sanitized (illegal characters) for Drive file naming; otherwise Drive may reject or modify names.

---

### Block 3 — Storage + AI Compliance Classification
**Overview:** Stores the uploaded document reference in Google Drive and asks GPT-4o-mini to classify completeness with a strict one-word output.

**Nodes involved:**
- Upload to Google Drive
- Check Document Completeness

#### Node: Upload to Google Drive
- **Type / role:** `Google Drive` node. Intended to upload/store the document.
- **Configuration choices:**
  - **Name:** `{{$json.documentName}}`
  - **Drive:** “My Drive”
  - **Folder ID:** from config `driveFolderId`
- **Credentials:** Google Drive OAuth2 (“Google Drive account 2”)
- **Inputs/Outputs:**
  - Input from **Normalize Tenant Data**
  - Output to **Check Document Completeness**
- **Important implementation note:**
  - As configured, this node sets a file name and folder, but the JSON does **not** show binary file content mapping. In n8n, actual file upload typically requires binary data (from webhook binary, HTTP download, etc.) and an “upload” operation. Verify the node operation is truly “Upload” and that binary property is mapped.
- **Edge cases / failures:**
  - Auth errors / expired OAuth token.
  - Folder not found or lacking permission.
  - Missing binary file data (common cause: Drive node creates metadata only or fails).

#### Node: Check Document Completeness
- **Type / role:** `OpenAI (LangChain)` node (`@n8n/n8n-nodes-langchain.openAi`). Performs AI classification.
- **Model:** `gpt-4o-mini` (by ID)
- **Configuration choices:**
  - System-style instruction: “You are a document compliance checker… Respond with only COMPLETE or INCOMPLETE.”
  - Prompt content includes only metadata:
    - Document Type
    - Document Name
    - Request to return only one word
- **Credentials:** OpenAI API (“n8n free OpenAI API credits”)
- **Inputs/Outputs:**
  - Input from **Upload to Google Drive**
  - Output to **Log Document Status**
- **Key output referenced later:**
  - Switch checks `{{$json.message.content}}` for “COMPLETE/INCOMPLETE”
- **Edge cases / failures:**
  - Because the prompt includes **no actual document text or file**, the AI can only guess based on the name/type; classification will be unreliable. For real compliance, pass extracted text (OCR/PDF parsing) or structured fields.
  - Model may return extra text despite instructions; switch uses `contains`, so “This is COMPLETE.” still matches, but ambiguity like “INCOMPLETE (missing signature)” matches both “INCOMPLETE” and “COMPLETE” (contains “COMPLETE” inside “INCOMPLETE”)—see next block.
  - Rate limits, invalid credentials, or transient OpenAI errors.

**Sticky note context:** “## AI Status” (covers this block).

---

### Block 4 — Logging + Classification + Notify/Escalate
**Overview:** Logs or updates the document status in a Google Sheet, routes based on AI result, sends email reminders for incomplete documents, waits 7 days, and escalates persistent incompleteness to Slack.

**Nodes involved:**
- Log Document Status
- Check Completeness Status
- Send Reminder Email
- Wait for Recheck
- Escalate to Slack

#### Node: Log Document Status
- **Type / role:** `Google Sheets` node. Appends or updates a row in a log.
- **Operation:** `appendOrUpdate`
- **Sheet:** “Document Log”
- **Document ID:** from config `sheetsDocumentId`
- **Mapping mode:** Auto-map input data
- **Matching columns:** `Tenant ID`
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account 3”)
- **Inputs/Outputs:**
  - Input from **Check Document Completeness**
  - Output to **Check Completeness Status**
- **Key considerations:**
  - Auto-map depends on column names matching incoming JSON keys. Your incoming keys are `tenantId`, `tenantName`, etc., but the matching column is **“Tenant ID”** (space/case differs). If the sheet columns are “Tenant ID” but incoming JSON key is `tenantId`, the match/update may not work as intended unless n8n’s automap handles header mapping flexibly.
- **Edge cases / failures:**
  - Sheet tab name mismatch (“Document Log” must exist).
  - Missing permissions to the spreadsheet.
  - appendOrUpdate needs a reliable unique key; if “Tenant ID” is not unique per document type, rows may overwrite unexpectedly.

#### Node: Check Completeness Status
- **Type / role:** `Switch` node. Branches execution based on AI output content.
- **Rules configured:**
  - Output `incomplete` if `{{$json.message.content}}` contains `INCOMPLETE`
  - Output `complete` if `{{$json.message.content}}` contains `COMPLETE`
- **Inputs/Outputs:**
  - Input from **Log Document Status**
  - Output currently connected only from **(default index 0)** to **Send Reminder Email**
- **Critical edge case (string containment bug):**
  - The word **“INCOMPLETE” contains “COMPLETE”** as a substring. If the AI returns “INCOMPLETE”, the “contains COMPLETE” rule may also match depending on evaluation order. This can misroute items.
  - Safer conditions: exact equals, regex with word boundaries, or check INCOMPLETE first and stop evaluation, or map AI output to a normalized field first (trim/uppercase) then strict compare.
- **Connection caveat:**
  - The workflow connections show only one outgoing connection (to “Send Reminder Email”) without specifying which named output is used. In n8n, Switch provides multiple outputs; ensure the “incomplete” output is what’s actually connected to the email reminder.

#### Node: Send Reminder Email
- **Type / role:** `Email Send` node. Sends a reminder to the tenant.
- **Key configuration:**
  - **To:** `{{$json.tenantEmail}}`
  - **Subject:** `Reminder: Incomplete Document - {{$json.documentType}}`
  - **HTML body:** templated using tenantName/documentType/documentName
  - **From:** placeholder sender email address
- **Inputs/Outputs:**
  - Input from **Check Completeness Status**
  - Output to **Wait for Recheck**
- **Edge cases / failures:**
  - Missing SMTP/Email credentials (not shown in JSON; must be configured in n8n).
  - Invalid tenant email.
  - If this should only send on incomplete, ensure switch routing is correct.

#### Node: Wait for Recheck
- **Type / role:** `Wait` node. Delays escalation.
- **Configuration:** Wait **7 days**
- **Inputs/Outputs:**
  - Input from **Send Reminder Email**
  - Output to **Escalate to Slack**
- **Edge cases / failures:**
  - Wait nodes require n8n to persist executions; if execution data pruning is aggressive or instance restarts without persistence, waits may not resume.
  - No “recheck” actually happens; it always escalates after 7 days regardless of whether tenant re-uploaded a complete document in the meantime. A proper recheck would query the sheet for updated status or listen for a new upload event and cancel pending escalation.

#### Node: Escalate to Slack
- **Type / role:** `Slack` node. Posts an escalation message to a channel.
- **Configuration choices:**
  - **Channel:** from config `slackChannel`
  - **Message text:** formatted escalation including tenant/document metadata and uploadTimestamp
- **Credentials:** Not shown in JSON; must be configured (Slack OAuth/token) in n8n.
- **Inputs/Outputs:**
  - Input from **Wait for Recheck**
  - No downstream nodes
- **Edge cases / failures:**
  - Invalid channel ID, missing scopes (`chat:write`, etc.), or token revocation.
  - Message contains a leading “=⚠️ …” (an equals sign). This is harmless for Slack but looks like an expression artifact; remove the leading `=` unless intentionally used.

**Sticky note context:** “## Classify and Notify” (covers this block).

---

### Block 5 — Weekly Compliance Reporting
**Overview:** Every week, reads the compliance log sheet, aggregates the dataset, and emails a high-level weekly report.

**Nodes involved:**
- Weekly Schedule
- Read Compliance Data
- Aggregate Compliance Status
- Send Compliance Report
- (Also depends on) Workflow Configuration for the report recipient and spreadsheet ID

#### Node: Weekly Schedule
- **Type / role:** `Schedule Trigger` (entry point).
- **Configuration:** Weekly, **day 1** at **09:00** (n8n “triggerAtDay” is locale-dependent; commonly Monday=1).
- **Inputs/Outputs:**
  - No input (trigger)
  - Output to **Read Compliance Data**
- **Edge cases / failures:**
  - Time zone depends on n8n instance settings.
  - If day indexing is misinterpreted, it may run on the wrong weekday.

#### Node: Read Compliance Data
- **Type / role:** `Google Sheets` node. Reads rows from the log.
- **Configuration:**
  - Spreadsheet ID from config `sheetsDocumentId`
  - Sheet name: “Document Log”
  - Operation not explicitly shown, but implied “read/get all”
- **Inputs/Outputs:**
  - Input from **Weekly Schedule**
  - Output to **Aggregate Compliance Status**
- **Edge cases / failures:**
  - If **Workflow Configuration** is not executed in this scheduled path, the expression `$('Workflow Configuration').first().json.sheetsDocumentId` may fail because that node isn’t in this branch’s execution context.
  - Typical fix: create a separate config node in the scheduled branch, or store config in environment variables, or use static data.

#### Node: Aggregate Compliance Status
- **Type / role:** `Aggregate` node. Aggregates all items into one.
- **Configuration:** `aggregateAllItemData` (collects the entire set into a single JSON structure, typically under `data`).
- **Inputs/Outputs:**
  - Input from **Read Compliance Data**
  - Output to **Send Compliance Report**
- **Edge cases / failures:**
  - If sheet is large, aggregating all rows can create large payloads and memory usage.

#### Node: Send Compliance Report
- **Type / role:** `Email Send` node. Sends weekly summary email.
- **Key configuration:**
  - **To:** `$('Workflow Configuration').first().json.complianceReportEmail` (same context issue as above)
  - **Subject:** `Weekly Tenant Document Compliance Report - YYYY-MM-DD`
  - **HTML body:** includes generation timestamp and `Total documents tracked: {{$json.data.length}}`
  - **From:** placeholder sender
- **Inputs/Outputs:**
  - Input from **Aggregate Compliance Status**
  - End of branch
- **Edge cases / failures:**
  - Same credential needs as the reminder email node.
  - Context issue if config node not executed in this branch.

**Sticky note context:** “## Schedule Trigger” (covers this block).

---

### Sticky Note (Global/Context)
#### Node: Sticky Note4
- **Type / role:** Sticky note describing purpose and setup.
- **Content preserved:**
  - **Main:** tracks tenant document compliance, validates with AI, sends reminders, escalates missing items, generates reports.
  - **Setup:** connect upload form, configure Drive folder, set AI prompt, connect Sheets/Email/Slack.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Document Upload Webhook | Webhook | Receives upload event payload (tenant/document metadata) | — | Workflow Configuration | ## Webhook & Workflow Config |
| Workflow Configuration | Set | Stores Drive/Sheets/Slack/email configuration constants | Document Upload Webhook | Normalize Tenant Data | ## Webhook & Workflow Config |
| Normalize Tenant Data | Set | Normalizes webhook body fields; adds timestamp | Workflow Configuration | Upload to Google Drive | ## Webhook & Workflow Config |
| Upload to Google Drive | Google Drive | Stores/creates document in configured Drive folder | Normalize Tenant Data | Check Document Completeness | ## AI Status |
| Check Document Completeness | OpenAI (LangChain) | Classifies document completeness as COMPLETE/INCOMPLETE | Upload to Google Drive | Log Document Status | ## AI Status |
| Log Document Status | Google Sheets | Appends/updates compliance log row | Check Document Completeness | Check Completeness Status | ## AI Status |
| Check Completeness Status | Switch | Routes based on AI output | Log Document Status | Send Reminder Email | ## Classify and Notify |
| Send Reminder Email | Email Send | Emails tenant when document is incomplete | Check Completeness Status | Wait for Recheck | ## Classify and Notify |
| Wait for Recheck | Wait | Delays escalation by 7 days | Send Reminder Email | Escalate to Slack | ## Classify and Notify |
| Escalate to Slack | Slack | Posts escalation after delay | Wait for Recheck | — | ## Classify and Notify |
| Weekly Schedule | Schedule Trigger | Starts weekly reporting run | — | Read Compliance Data | ## Schedule Trigger |
| Read Compliance Data | Google Sheets | Reads all compliance log rows | Weekly Schedule | Aggregate Compliance Status | ## Schedule Trigger |
| Aggregate Compliance Status | Aggregate | Combines all rows into one item for reporting | Read Compliance Data | Send Compliance Report | ## Schedule Trigger |
| Send Compliance Report | Email Send | Emails weekly summary to compliance mailbox | Aggregate Compliance Status | — | ## Schedule Trigger |
| Sticky Note | Sticky Note | Comment block label | — | — |  |
| Sticky Note1 | Sticky Note | Comment block label | — | — |  |
| Sticky Note2 | Sticky Note | Comment block label | — | — |  |
| Sticky Note3 | Sticky Note | Comment block label | — | — |  |
| Sticky Note4 | Sticky Note | High-level description & setup instructions | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named “Tenant Document Compliance Tracking and Reminder System”.
2. **Add Webhook node**:  
   - Type: **Webhook**  
   - Method: **POST**  
   - Path: `tenant-document-upload`  
   - Response: **Last Node**  
   - Ensure your uploader sends JSON body with: `tenantId, tenantName, tenantEmail, documentType, documentName` (and file if applicable).
3. **Add Set node** named **Workflow Configuration** after the webhook:  
   - Add fields:
     - `driveFolderId` = (your Google Drive folder ID)
     - `sheetsDocumentId` = (your Google Sheets spreadsheet ID)
     - `slackChannel` = (Slack channel ID)
     - `complianceReportEmail` = (recipient for weekly report)
   - Enable **Include Other Fields**.
4. **Add Set node** named **Normalize Tenant Data**:  
   - Map:
     - `tenantId` = `{{$json.body.tenantId}}`
     - `tenantName` = `{{$json.body.tenantName}}`
     - `tenantEmail` = `{{$json.body.tenantEmail}}`
     - `documentType` = `{{$json.body.documentType}}`
     - `documentName` = `{{$json.body.documentName}}`
     - `uploadTimestamp` = `{{$now.toISO()}}`
   - Enable **Include Other Fields**.
5. **Add Google Drive node** named **Upload to Google Drive**:  
   - Authenticate with **Google Drive OAuth2** credentials.  
   - Set **Drive** to *My Drive*.  
   - Set **Folder ID** to `{{$('Workflow Configuration').first().json.driveFolderId}}`.  
   - Set **Name** to `{{$json.documentName}}`.  
   - If you are uploading an actual file: configure the node’s upload operation and map the incoming **binary** property (requires webhook to receive binary).
6. **Add OpenAI (LangChain) node** named **Check Document Completeness**:  
   - Credentials: **OpenAI API**  
   - Model: **gpt-4o-mini**  
   - Instructions: compliance checker, return only COMPLETE/INCOMPLETE  
   - User content: include your metadata (and ideally extracted text) and demand single-word output.
7. **Add Google Sheets node** named **Log Document Status**:  
   - Credentials: **Google Sheets OAuth2**  
   - Spreadsheet ID: `{{$('Workflow Configuration').first().json.sheetsDocumentId}}`  
   - Sheet/tab name: `Document Log`  
   - Operation: **Append or Update**  
   - Matching column: `Tenant ID` (ensure the sheet has this exact header, or adjust to match your JSON key).
8. **Add Switch node** named **Check Completeness Status**:  
   - Condition uses AI output field (e.g., `{{$json.message.content}}`).  
   - Create outputs for incomplete vs complete.  
   - Recommendation when rebuilding: normalize AI output (trim/uppercase) and use **equals** instead of **contains** to avoid “INCOMPLETE” matching “COMPLETE”.
9. **Add Email Send node** named **Send Reminder Email** on the **incomplete** route:  
   - Configure email credentials (SMTP / provider integration).  
   - From: your sender address  
   - To: `{{$json.tenantEmail}}`  
   - Subject/HTML: use the provided templating.
10. **Add Wait node** named **Wait for Recheck** after the reminder:  
   - Wait: **7 days**
11. **Add Slack node** named **Escalate to Slack** after the wait:  
   - Configure Slack credentials with `chat:write` scope.  
   - Channel ID: `{{$('Workflow Configuration').first().json.slackChannel}}`  
   - Message text: include tenant/document details.
12. **Add Schedule Trigger** named **Weekly Schedule** (second entry point):  
   - Weekly at desired day/time (configured here as day 1, 09:00).
13. **Add Google Sheets node** named **Read Compliance Data** after the schedule:  
   - Spreadsheet ID: you must provide it reliably for this branch.  
   - Recommended: add another Set/Config node in this branch (or use environment variables) instead of referencing `Workflow Configuration` from the webhook branch.
14. **Add Aggregate node** named **Aggregate Compliance Status**:  
   - Operation: **Aggregate all item data** into one item.
15. **Add Email Send node** named **Send Compliance Report**:  
   - To: compliance report mailbox  
   - Subject: weekly date-stamped subject  
   - HTML: reference aggregated data length (e.g., `{{$json.data.length}}` depending on Aggregate output).
16. **Add Sticky Notes** (optional) to label sections:
   - “## Webhook & Workflow Config”, “## AI Status”, “## Classify and Notify”, “## Schedule Trigger”, and the “## Main / Setup” note.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Compliance/disclaimer (provided in request) |
| This workflow’s AI step currently evaluates only document metadata (type/name). For real compliance checking, pass actual document content (OCR/text extraction) to GPT. | Design note |
| The scheduled reporting branch references `$('Workflow Configuration')...` but does not execute that node in the schedule path; duplicate config in the schedule branch or store config in env/static data. | Reliability note |
| Switch “contains COMPLETE” can match “INCOMPLETE”. Use exact match or safer parsing. | Logic correctness note |