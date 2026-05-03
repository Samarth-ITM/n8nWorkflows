Send severity-based error alerts using Telegram, email and Google Sheets

https://n8nworkflows.xyz/workflows/send-severity-based-error-alerts-using-telegram--email-and-google-sheets-12890


# Send severity-based error alerts using Telegram, email and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (provided):** Send severity-based error alerts using Telegram, email and Google Sheets  
**Workflow name (internal):** Send severity-based error alerts for n8n workflows using Telegram and email

This workflow is a **global error handler** for n8n. It listens for workflow execution failures (from other workflows that reference it as their Error Workflow), **classifies the error severity**, formats a readable alert message, and then **routes notifications** to Telegram and (for critical errors) email plus Google Sheets logging.

### Logical blocks
1.1 **Error Intake (Global Handler Entry Point)**  
Receives execution failure events via *Error Trigger*.

1.2 **Severity Classification & Context Extraction**  
Analyzes error details (HTTP codes and message text), labels severity, and extracts workflow/execution metadata.

1.3 **Alert Message Formatting**  
Builds a single Markdown-style message used by Telegram (and also reused for email body).

1.4 **Severity-Based Routing & Escalation**  
Routes to different downstream actions based on severity:  
- Critical → Email → Google Sheets logging → Telegram  
- High → Telegram  
- Normal → Telegram

1.5 **Delivery & Logging Integrations**  
Sends Telegram message, sends Gmail email, appends a row into Google Sheets for critical incidents.

---

## 2. Block-by-Block Analysis

### 2.1 Error Intake (Global Handler Entry Point)

**Overview:** Captures error events when another workflow fails, acting as a centralized error workflow. This is the only entry point.  
**Nodes involved:** `Error Trigger`

#### Node: Error Trigger
- **Type / role:** `n8n-nodes-base.errorTrigger` — triggers when a linked workflow execution errors.
- **Configuration choices:** No parameters configured (default behavior).
- **Inputs / outputs:**
  - **Input:** None (trigger node).
  - **Output:** Emits an item containing context like `workflow`, `execution`, and `execution.error` (structure depends on n8n version).
  - Connected to **Classify Error & Context**.
- **Version notes:** TypeVersion `1` (standard).
- **Edge cases / failures:**
  - If this workflow is not set as **Error Workflow** in other workflows, it will never run.
  - If the source workflow is executed in a way that doesn’t produce an error event (or error handling differs by version), fields may be missing.

---

### 2.2 Severity Classification & Context Extraction

**Overview:** Converts the raw error payload into a normalized record (severity + metadata) used everywhere else.  
**Nodes involved:** `Classify Error & Context`  
**Sticky note coverage:** “Error classification”

#### Node: Classify Error & Context
- **Type / role:** `n8n-nodes-base.code` — JavaScript transformation / enrichment.
- **Configuration choices (interpreted):**
  - Reads error details from: `const error = $json.execution?.error || {};`
  - Normalizes `message` to lowercase and reads `httpCode` as string when available.
  - Sets default severity: `🟡 Normal`
  - **Critical** if HTTP code is `401` or `403`, or message contains: `unauthorized`, `forbidden`, `authentication`
  - **High** if message contains `timeout` / `timed out` or HTTP code starts with `5`
  - Returns a single item with:
    - `severity`
    - `workflowName`, `workflowId`
    - `failedNode` (from `execution.lastNodeExecuted`)
    - `executionId`, `executionUrl`
    - `errorMessage`
    - `time` (using `$now.toFormat("yyyy-LL-dd HH:mm")`)
- **Key variables/expressions:**
  - Optional chaining: `$json.execution?.error`
  - `$now.toFormat(...)`
- **Inputs / outputs:**
  - **Input:** From `Error Trigger`.
  - **Output:** To `Format Message`.
- **Version notes:** TypeVersion `2` (Code node v2 behavior; ensure your n8n supports `$now` and Luxon formatting).
- **Edge cases / failures:**
  - `execution.lastNodeExecuted` can be undefined for early failures; `failedNode` becomes null/undefined.
  - `execution.url` might be missing depending on instance config; consider constructing it from base URL + executionId if needed.
  - Some errors may not provide `httpCode`; classification then relies solely on message text.
  - If `error.message` is non-string (rare), `.toLowerCase()` could fail; current code defaults to empty string, which is safe.

---

### 2.3 Alert Message Formatting

**Overview:** Creates a single human-readable alert text that includes severity, workflow context, and direct execution link.  
**Nodes involved:** `Format Message`  
**Sticky note coverage:** “Alert formatting”

#### Node: Format Message
- **Type / role:** `n8n-nodes-base.set` — constructs the `message` field and drops all other fields.
- **Configuration choices (interpreted):**
  - `keepOnlySet: true` ensures output contains only:
    - `message` (string)
  - Message body includes placeholders from the current item (produced by classification), e.g.:
    - `{{ $json.severity }}`
    - `{{ $json.workflowName }}`
    - `{{ $json.failedNode }}`
    - `{{ $json.time }}`
    - `{{ $json.executionId }}`
    - `{{ $json.executionUrl }}`
    - `{{ $json.errorMessage }}`
  - The message includes Markdown-like formatting (asterisks). Note: Telegram node may require parse mode to fully render Markdown (see edge cases).
- **Inputs / outputs:**
  - **Input:** From `Classify Error & Context`.
  - **Output:** To `Severity-Based Routing`.
- **Version notes:** TypeVersion `1`.
- **Edge cases / failures:**
  - Any missing fields render as blank strings in the message.
  - Emoji and Markdown formatting may not display as intended unless Telegram parse mode is configured (node-dependent).

---

### 2.4 Severity-Based Routing & Escalation

**Overview:** Splits processing into three paths based on severity. Critical goes through email then logging then Telegram; High and Normal go directly to Telegram.  
**Nodes involved:** `Severity-Based Routing`  
**Sticky note coverage:** “Severity-based routing”

#### Node: Severity-Based Routing
- **Type / role:** `n8n-nodes-base.switch` — conditional router with named outputs.
- **Configuration choices (interpreted):**
  - Evaluates **left value**: `{{ $('Classify Error & Context').item.json.severity }}`
  - Rules:
    1. Output key **“🔴 Critical ”** when equals **“🔴 Critical (Auth)”**
    2. Output key **“🟡 Normal”** when equals **“🟡 Normal”**
    3. Output key **“🟠 High”** when equals **“🟠 High (Timeout / Server)”**
  - Uses strict type validation and case-sensitive comparisons.
- **Inputs / outputs:**
  - **Input:** From `Format Message`.
  - **Outputs:**
    - Output 0 (Critical) → `Send an Email`
    - Output 1 (Normal) → `Send Alert`
    - Output 2 (High) → `Send Alert`
- **Version notes:** TypeVersion `3.4` (Switch node with rules/conditions builder).
- **Edge cases / failures:**
  - **No default/fallback output** is configured. If severity is anything unexpected (e.g., code changes, typo, new category), the item will be dropped and no alert will be sent.
  - The output key has a trailing space for “🔴 Critical ” (cosmetic, but can confuse maintenance).

---

### 2.5 Delivery & Logging Integrations (Telegram, Gmail, Google Sheets)

**Overview:** Sends notifications (Telegram always for routed paths; Email+Sheets only for critical). Logs critical incidents to Google Sheets for auditing.  
**Nodes involved:** `Send an Email`, `Add Logs to Sheet`, `Send Alert`  
**Sticky note coverage:** “Error logging” (for Google Sheets node)

#### Node: Send an Email
- **Type / role:** `n8n-nodes-base.gmail` — sends an email using Gmail credentials.
- **Configuration choices (interpreted):**
  - **To:** `YOUR_ALERT_EMAIL` (placeholder)
  - **Subject:** `Critical Error in Workflow : {{ $('Classify Error & Context').item.json.workflowName }}`
  - **Body message:** `{{ $('Format Message').item.json.message }}`
  - `emailType: text`
  - Attribution disabled: `appendAttribution: false`
- **Inputs / outputs:**
  - **Input:** From `Severity-Based Routing` (Critical path only).
  - **Output:** To `Add Logs to Sheet`.
- **Version notes:** TypeVersion `2.2`.
- **Edge cases / failures:**
  - Gmail OAuth2 permissions/auth can expire or be revoked (common failure).
  - Sending limits / rate limits may apply.
  - If `YOUR_ALERT_EMAIL` is not replaced with a valid address, node fails.
  - Because the body is “text”, Markdown will not be rendered; it will appear as plain text.

#### Node: Add Logs to Sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — appends a row for critical errors.
- **Configuration choices (interpreted):**
  - **Operation:** Append
  - **Document ID:** placeholder `YOUR_Document_ID` (must be replaced)
  - **Sheet ID/Name setting:** `sheetName` uses “id” mode with placeholder `YOUR_SHEET_ID` (must be replaced)
  - **Mapping mode:** “defineBelow” mapping with explicit columns:
    - Workflow Name, Severity, Execution ID, Execution URL, Failed Node, Error Message
  - Values are pulled from `Classify Error & Context` node output via expressions.
- **Inputs / outputs:**
  - **Input:** From `Send an Email` (critical path continues).
  - **Output:** To `Send Alert` (critical ends with Telegram).
- **Version notes:** TypeVersion `4.7`.
- **Edge cases / failures:**
  - Spreadsheet permission errors (403) if Google credential lacks access.
  - Schema mismatch: if the sheet doesn’t have matching columns/headers or the node expects a sheet ID but receives a name (or vice versa).
  - Placeholder IDs not replaced will cause runtime errors.
  - Appends can fail due to Google API quotas.

#### Node: Send Alert
- **Type / role:** `n8n-nodes-base.telegram` — sends Telegram message to a chat.
- **Configuration choices (interpreted):**
  - **Chat ID:** `YOUR_CHAT_ID` (placeholder)
  - **Text:** `{{ $('Severity-Based Routing').item.json.message }}`
    - Note: message originates from `Format Message`; it is passed through routing.
  - Attribution disabled.
  - Uses Telegram bot credentials: `n8n_Telegram_bot`.
- **Inputs / outputs:**
  - **Input:** From:
    - `Severity-Based Routing` (High and Normal paths)
    - `Add Logs to Sheet` (Critical path, after email+logging)
  - **Output:** none (terminal).
- **Version notes:** TypeVersion `1.1`.
- **Edge cases / failures:**
  - Wrong Chat ID or bot not in the chat → 403/400 errors.
  - Message formatting: Telegram may not render Markdown unless parse mode is set (not shown in config).
  - Large error messages can exceed Telegram message length limits; consider truncation.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note3 | Sticky Note | Workflow overview / documentation |  |  | ## Workflow overview; Captures errors, classifies, routes to Telegram/Email, logs critical; setup steps (credentials + placeholders + activate + set as Error Workflow). |
| Error Trigger | Error Trigger | Entry point for failed executions |  | Classify Error & Context | ## Error classification; Errors classified into Critical/High/Normal for smarter routing. |
| Classify Error & Context | Code | Determine severity + extract workflow/execution context | Error Trigger | Format Message | ## Error classification; Errors classified into Critical/High/Normal for smarter routing. |
| Sticky Note | Sticky Note | Documentation |  |  | ## Error classification; Errors analyzed and classified into Critical/High/Normal. |
| Format Message | Set | Create formatted alert message | Classify Error & Context | Severity-Based Routing | ## Alert formatting; Builds readable alert with workflow, node, time, message, link. |
| Sticky Note2 | Sticky Note | Documentation |  |  | ## Alert formatting; Builds a readable alert message with context and execution link. |
| Severity-Based Routing | Switch | Route actions by severity | Format Message | Send an Email; Send Alert | ## Severity-based routing; Critical→Email+Telegram+Logging; High→Telegram; Normal→Telegram. |
| Sticky Note1 | Sticky Note | Documentation |  |  | ## Severity-based routing; Critical→Email+Telegram+Logging; High→Telegram; Normal→Telegram. |
| Send an Email | Gmail | Email escalation for critical errors | Severity-Based Routing | Add Logs to Sheet | ## Severity-based routing; Critical→Email + Telegram + Logging. |
| Add Logs to Sheet | Google Sheets | Log critical incidents | Send an Email | Send Alert | ## Error logging; Critical errors logged to Google Sheets for auditing. |
| Sticky Note5 | Sticky Note | Documentation |  |  | ## Error logging; Critical errors logged for auditing and historical review using Google Sheets. |
| Send Alert | Telegram | Send Telegram notification | Severity-Based Routing; Add Logs to Sheet |  | ## Severity-based routing; Telegram used for all severities, critical after email+logging. |
| Sticky Note4 | Sticky Note | Branding / contact info |  |  | ## 👋 Need help with n8n automations?; https://abdallahhussein.com; Automation & Workflow Consulting |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it (example): “Send severity-based error alerts for n8n workflows using Telegram and email”.
- Keep it **inactive** until credentials and placeholders are configured.

2) **Add node: Error Trigger**
- Node type: **Error Trigger**
- Leave defaults.
- This will be the workflow’s entry point.

3) **Add node: Code** (severity + context)
- Node type: **Code**
- Name: “Classify Error & Context”
- Paste logic that:
  - Reads `$json.execution?.error`
  - Classifies:
    - Critical if 401/403 or auth/forbidden keywords
    - High if timeouts or 5xx
    - Normal otherwise
  - Outputs fields: `severity, workflowName, workflowId, failedNode, executionId, executionUrl, errorMessage, time`
- Connect: **Error Trigger → Classify Error & Context**

4) **Add node: Set** (message formatting)
- Node type: **Set**
- Name: “Format Message”
- Enable **Keep Only Set**
- Add a string field:
  - Name: `message`
  - Value: a formatted multi-line text that uses `{{$json.severity}}`, `{{$json.workflowName}}`, `{{$json.failedNode}}`, `{{$json.time}}`, `{{$json.executionId}}`, `{{$json.executionUrl}}`, `{{$json.errorMessage}}`
- Connect: **Classify Error & Context → Format Message**

5) **Add node: Switch** (routing)
- Node type: **Switch**
- Name: “Severity-Based Routing”
- Add rules that compare:
  - Left value: `{{ $('Classify Error & Context').item.json.severity }}`
  - Rule 1 equals `🔴 Critical (Auth)` (Critical output)
  - Rule 2 equals `🟡 Normal` (Normal output)
  - Rule 3 equals `🟠 High (Timeout / Server)` (High output)
- Connect: **Format Message → Severity-Based Routing**

6) **Add node: Gmail** (critical email)
- Node type: **Gmail**
- Name: “Send an Email”
- Operation: **Send**
- Configure:
  - To: your alert mailbox (replace placeholder)
  - Subject: `Critical Error in Workflow : {{ $('Classify Error & Context').item.json.workflowName }}`
  - Message/body: `{{ $('Format Message').item.json.message }}`
  - Email type: `text`
- Credentials:
  - Create/select **Gmail OAuth2** credential with permission to send email.
- Connect: **Severity-Based Routing (Critical output) → Send an Email**

7) **Add node: Google Sheets** (critical logging)
- Node type: **Google Sheets**
- Name: “Add Logs to Sheet”
- Operation: **Append**
- Select:
  - Google Sheets credentials (OAuth2) with access to the target spreadsheet.
  - Document/Spreadsheet: set **Document ID**
  - Sheet: select by **Sheet ID** (or use sheet name consistently with your configuration)
- Define columns mapping:
  - Workflow Name = from `Classify Error & Context`
  - Severity
  - Execution ID
  - Execution URL
  - Failed Node
  - Error Message
- Connect: **Send an Email → Add Logs to Sheet**

8) **Add node: Telegram**
- Node type: **Telegram**
- Name: “Send Alert”
- Operation: **Send Message**
- Configure:
  - Chat ID: your Telegram chat/group ID
  - Text: `{{ $('Severity-Based Routing').item.json.message }}`
- Credentials:
  - Create/select a **Telegram Bot API** credential (bot token).
  - Ensure the bot is added to the target chat (and has permission to post).
- Connect:
  - **Severity-Based Routing (Normal output) → Send Alert**
  - **Severity-Based Routing (High output) → Send Alert**
  - **Add Logs to Sheet → Send Alert** (so critical alerts go out after logging)

9) **Activate and link as the global Error Workflow**
- Activate this workflow.
- For each workflow you want monitored:
  - Open workflow settings → **Error workflow** → select this workflow.
- Trigger a controlled test failure in a monitored workflow to validate:
  - Telegram delivery (Normal/High/Critical)
  - Email (Critical)
  - Sheets row append (Critical)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Need help with n8n automations? … Automation & Workflow Consulting” | https://abdallahhussein.com |
| Setup reminders: configure Telegram + Email, replace placeholders (Chat ID, Email, Sheet/Document IDs), activate workflow, set as Error Workflow in other workflows, test with a forced error | Included in the workflow overview sticky note content (Sticky Note3) |