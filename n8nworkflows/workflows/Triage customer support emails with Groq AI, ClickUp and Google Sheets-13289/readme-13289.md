Triage customer support emails with Groq AI, ClickUp and Google Sheets

https://n8nworkflows.xyz/workflows/triage-customer-support-emails-with-groq-ai--clickup-and-google-sheets-13289


# Triage customer support emails with Groq AI, ClickUp and Google Sheets

## 1. Workflow Overview

**Workflow name:** *Customer Support Triage*  
**Purpose:** Automatically triage incoming support emails using Groq’s OpenAI-compatible API, validate/normalize the AI output, calculate assignment + SLA deadlines, create a ClickUp task, log the ticket to Google Sheets, notify the team in Slack, and send an acknowledgement email to the customer.  
**Primary use cases:** Small-to-mid support teams wanting consistent classification (category/urgency), faster routing, SLA visibility, and centralized tracking.

### 1.1 Email Intake & Normalization
- Polls a Gmail label for support emails and normalizes fields into a compact “email payload” for AI analysis.

### 1.2 AI Classification (Groq)
- Sends the email to Groq Chat Completions with a strict “return JSON only” instruction.

### 1.3 Parsing + Validation
- Parses the model response as JSON and enforces allowed values (category/urgency), minimum summary/action quality, and adds validation metadata.

### 1.4 Smart Assignment, SLA & Escalation Flags
- Maps category → team + priority + tags; computes SLA deadlines; flags possible escalations based on keywords and criticality.

### 1.5 Task Creation + Enrichment
- Creates a ClickUp task and enriches the ticket data with the created task URL/ID.

### 1.6 Logging, Notifications & Customer Acknowledgement
- Appends a row to Google Sheets, optionally notifies an escalation Slack channel, notifies a support user in Slack, and sends an acknowledgement email back to the sender.

---

## 2. Block-by-Block Analysis

### Block 1 — Email Intake & AI Preparation
**Overview:** Polls the support mailbox and extracts the minimal fields needed for classification (from/subject/body/id).  
**Nodes involved:** `Fetch Support Email list` → `Assign Email Values`

#### Node: Fetch Support Email list
- **Type / role:** Gmail Trigger (`n8n-nodes-base.gmailTrigger`) — entry point, polls Gmail.
- **Configuration (interpreted):**
  - Polling: **every minute**
  - Filter: emails with `labelIds = ["Label_7038236549786244180"]`
  - Filter: `readStatus = "read"` (only already-read messages)
  - `alwaysOutputData = true` (still outputs even when no items; can affect downstream nodes)
- **Outputs:** Main → `Assign Email Values`
- **Credentials:** Gmail OAuth2
- **Edge cases / failures:**
  - Wrong label ID or label deleted → no items or API error.
  - Polling every minute can hit Gmail API quota depending on account volume.
  - Using `readStatus=read` may miss new unread support emails unless another process marks them read.

#### Node: Assign Email Values
- **Type / role:** Set (`n8n-nodes-base.set`) — normalizes field names for later expressions.
- **Configuration (interpreted):** Creates:
  - `from` = `{{$json.From}}`
  - `subject` = `{{$json.Subject}}`
  - `body` = `{{$json.snippet}}` (uses Gmail snippet, not full body)
  - `emailid` = `{{$json.id}}`
- **Outputs:** Main → `Analysis with Grok`
- **Important variable note (bug):**
  - Downstream nodes reference `emailId` (capital “I”) in `Parse AI Response`, but this node sets `emailid` (lowercase “i”).
  - Also, `Parse AI Response` expects `emailData.emailId` while this node provides `emailid`.
- **Edge cases / failures:**
  - Gmail trigger payload fields can differ (`From`/`Subject` casing may not exist depending on node output mode). If undefined, AI prompt will contain empty strings.

**Sticky note covering this block:**  
“## Email intake & AI analysis — Fetch emails and analyze with Grok AI”

---

### Block 2 — AI Classification (Groq)
**Overview:** Sends the email text to Groq’s OpenAI-compatible endpoint and asks for strict JSON classification.  
**Nodes involved:** `Analysis with Grok`

#### Node: Analysis with Grok
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls Groq Chat Completions API.
- **Configuration (interpreted):**
  - Method: POST
  - URL: `https://api.groq.com/openai/v1/chat/completions`
  - JSON body includes:
    - `model: "llama-3.1-8b-instant"`
    - `messages[0].content`: prompt built from `from/subject/body`
    - `temperature: 0.3`
  - Authentication: predefined credential type `groqApi`
- **Inputs:** From `Assign Email Values`
- **Outputs:** Main → `Parse AI Response`
- **Edge cases / failures:**
  - Model may return non-JSON or JSON with extra text → `JSON.parse` will fail downstream.
  - HTTP 401/403 if Groq credentials invalid; 429 if rate-limited; 5xx transient failures.
  - Prompt uses `{{ $json.body }}` which is only a snippet; classification quality may drop.

**Sticky note covering this block:**  
“## Email intake & AI analysis — Fetch emails and analyze with Grok AI”

---

### Block 3 — Parse + Validate AI Output
**Overview:** Parses the model’s JSON-only response and enforces allowed schema/values; adds validation metadata and safe defaults.  
**Nodes involved:** `Parse AI Response` → `Validate AI Response`

#### Node: Parse AI Response
- **Type / role:** Code (`n8n-nodes-base.code`) — parses AI message content as JSON and merges with email metadata.
- **Configuration (interpreted):**
  - Reads Groq response at: `$input.first().json.choices[0].message.content`
  - `JSON.parse(aiResponse)`
  - Pulls email info from: `$('Assign Email Values').first().json`
  - Outputs combined object:
    - `timestamp` (ISO string)
    - `from`, `subject`
    - `emailId` (expected from upstream email data)
    - `category`, `urgency`, `summary`, `suggested_action`
- **Inputs:** From `Analysis with Grok`
- **Outputs:** Main → `Validate AI Response`
- **Edge cases / failures:**
  - **Hard failure** if AI response is not valid JSON → workflow stops here (no try/catch).
  - **Data mismatch:** upstream provides `emailid`, not `emailId`. This will produce `emailId: undefined`.
  - If Groq returns a different response structure (no `choices[0]...`) the node will error.

#### Node: Validate AI Response
- **Type / role:** Code (`n8n-nodes-base.code`) — schema validation + auto-fix.
- **Configuration (interpreted):**
  - Allowed categories:
    - Bug Report, Feature Request, Billing Question, General Inquiry, Technical Support, Marketing Email
  - Allowed urgencies: Low, Medium, High, Critical
  - Auto-fixes:
    - Invalid/missing category → set to `General Inquiry`
    - Invalid/missing urgency → set to `Medium`
    - Summary missing/too short (<10) → default text + `requires_manual_review = true`
    - Suggested action missing/too short (<5) → default guidance
  - Adds:
    - `validation_status`: `passed` or `fixed`
    - `validation_issues`: list or `none`
- **Inputs:** From `Parse AI Response`
- **Outputs:** Main → `Smart Assignment & SLA`
- **Edge cases / failures:**
  - If upstream didn’t provide `category/urgency` at all (parse partially failed), this node still attempts to fix, which is good.
  - Category/urgency capitalization differences from AI (“critical” vs “Critical”) will be treated invalid and forced to defaults (consider normalizing case if desired).

**Sticky note covering this block:**  
“## Validation & assignment — Validate AI output and assign to teams”

---

### Block 4 — Assignment, SLA, Escalation Logic
**Overview:** Determines the responsible team, ClickUp priority, tags, computes SLA deadline, and flags escalation based on keywords or critical urgency.  
**Nodes involved:** `Smart Assignment & SLA`

#### Node: Smart Assignment & SLA
- **Type / role:** Code (`n8n-nodes-base.code`) — business rules engine.
- **Configuration (interpreted):**
  - Assignment by `ticket.category`:
    - Bug Report → Engineering Team, priority `urgent` if Critical else `high`, tags `bug`, `technical`
    - Billing Question → Billing Team, priority `normal`, tags `billing`, `finance`
    - Feature Request → Product Team, priority `low`, tags `feature-request`, `product`
    - Technical Support → Support Team, priority `urgent` if Critical else `normal`, tags `support`, `technical`
    - Else → General Support, priority `normal`, tags `general`
  - SLA hours map (based on urgency):
    - Critical: 1, High: 4, Medium: 24, Low: 48
  - Deadline calculation: `now + slaHours[urgency]`
  - Escalation flags:
    - Negative sentiment keywords in `(subject + summary)`: angry, frustrated, terrible, worst
    - Churn risk keywords: cancel, refund, competitor, switching (also adds tag `churn-risk`)
    - Critical urgency always escalates
  - Outputs enriched fields:
    - `assigned_to`, `assigned_to_email`
    - `clickup_priority`, `clickup_tags`
    - `sla_deadline` (ISO), `sla_deadline_readable` (US locale), `sla_hours`
    - `requires_escalation`, `escalation_reason`
- **Inputs:** From `Validate AI Response`
- **Outputs:** Main → `Create a Clickup task`
- **Edge cases / failures:**
  - `assignToEmail` is hardcoded `user@example.com` in all branches; must be replaced.
  - SLA uses `deadline.setHours(deadline.getHours() + X)`—this is wall-clock and can behave unexpectedly across DST transitions.
  - Keyword checks are simplistic; false positives/negatives possible.

**Sticky note covering this block:**  
“## Validation & assignment — Validate AI output and assign to teams”

---

### Block 5 — ClickUp Task Creation + Result Enrichment
**Overview:** Creates a ClickUp task for the triaged ticket and then merges ClickUp response fields (task ID/URL) back into the main data object.  
**Nodes involved:** `Create a Clickup task` → `Get Clickup Task URL`

#### Node: Create a Clickup task
- **Type / role:** ClickUp (`n8n-nodes-base.clickUp`) — creates a task.
- **Configuration (interpreted):**
  - Operation: create task (implicit by node name and fields)
  - Workspace/Team/Space/List:
    - `team = 90161453486`
    - `space = 90166145857`
    - `list = 901613190616`
    - `folderless = true`
  - Task name: `{{$json.subject}}`
  - Additional fields:
    - `tags = {{$json.clickup_tags}}`
    - `status = "new ticket"`
    - `content`: rich markdown template embedding email + AI + SLA + validation details
    - `dueDate = {{$json.sla_deadline_readable}}`
- **Inputs:** From `Smart Assignment & SLA`
- **Outputs:** Main → `Get Clickup Task URL`
- **Credentials:** ClickUp API
- **Edge cases / failures:**
  - **Due date format risk:** ClickUp typically expects a timestamp (ms) for due dates; providing a locale string may fail or be ignored depending on node implementation/version.
  - Invalid list/space/team IDs or insufficient permissions → 4xx errors.
  - Tags format: ClickUp may expect array of strings; ensure `clickup_tags` matches what the node expects.

#### Node: Get Clickup Task URL
- **Type / role:** Code (`n8n-nodes-base.code`) — merges ClickUp response into prior data.
- **Configuration (interpreted):**
  - Reads “previous” data from `$('Smart Assignment & SLA').first().json`
  - Reads ClickUp create response from `$input.first().json`
  - Outputs:
    - adds `clickup_task_id`, `clickup_task_url`
    - adds `clickup_task_created: true`
- **Inputs:** From `Create a Clickup task`
- **Outputs:** Main → `Log in Google Sheet` and `Support Team Notification` (two parallel consumers)
- **Edge cases / failures:**
  - If ClickUp response shape differs (no `id`/`url`), downstream logging will break or log blanks.
  - This node assumes only one item; multiple emails in one execution may mix data if not careful (it uses `.first()` from another node, not per-item mapping).

**Sticky note covering this block:**  
“## Task creation & notifications — Create ClickUp tasks and notify via Slack”

---

### Block 6 — Logging, Conditional Escalation, Slack + Email Reply
**Overview:** Writes a tracking record to Google Sheets, then branches: if escalation required, send an escalation Slack message; always send an acknowledgement email. Separately, sends a Slack notification to support.  
**Nodes involved:** `Log in Google Sheet` → `If Urgent?` → (`Escalation Notification`) + (`Email Acknowledgement`) and `Support Team Notification`

#### Node: Log in Google Sheet
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — append row to a tracker sheet.
- **Configuration (interpreted):**
  - Operation: Append
  - Document: “Support Tickets Tracker” (ID: `1-c-1AXq0e3YoLxpKt7d0PxBY1dOUPc3ECfkf3lb9Wkw`)
  - Sheet: “Sheet1” (`gid=0`)
  - Columns mapped from current JSON:
    - Timestamp, From, Subject, Category, Urgency, Summary, Action
    - SLA Deadline, ClickUp Task ID/URL
    - Escalation Reason, Requires Escalation
    - Validation Status/Issues
- **Inputs:** From `Get Clickup Task URL`
- **Outputs:** Main → `If Urgent?`
- **Credentials:** Google Sheets OAuth2
- **Edge cases / failures:**
  - Sheet columns must exist; mismatches can cause append errors or unexpected column placement.
  - `Requires Escalation` is defined in schema as string; mapping uses boolean `requires_escalation`—may be stored as TRUE/FALSE text or fail type conversion depending on node settings (`attemptToConvertTypes` is false).

#### Node: If Urgent?
- **Type / role:** IF (`n8n-nodes-base.if`) — conditional routing.
- **Condition (interpreted):**
  - Checks `{{$json["Requires Escalation"]}} equals true` (boolean strict)
- **Inputs:** From `Log in Google Sheet`
- **Outputs:**
  - True branch → `Escalation Notification` and `Email Acknowledgement`
  - False branch → `Email Acknowledgement`
- **Critical data mismatch (bug):**
  - Your workflow consistently uses `requires_escalation` (lowercase) earlier, and the Google Sheets mapping uses `Requires Escalation` as a column name, but **the IF node checks the sheet-style key** (`"Requires Escalation"`).
  - Since the IF node runs after the Sheets append, it still receives the **same JSON item** from before/after the Sheets node (not the sheet row), so `"Requires Escalation"` likely does not exist, making the condition always false.
  - Correct check should likely be `{{$json.requires_escalation}}`.
- **Edge cases / failures:**
  - Strict boolean compare will fail if the value is `"true"` (string) instead of `true` (boolean).

#### Node: Escalation Notification
- **Type / role:** Slack (`n8n-nodes-base.slack`) — posts escalation alert to a channel.
- **Configuration (interpreted):**
  - Sends a formatted message to channel `C0AAKJ02EMR` (cached name: “escalations”)
- **Template variables used (issue):**
  - Uses `{{ $json.Subject }}`, `{{ $json.From }}`, `{{ $json.Urgency }}`, `{{ $json.Category }}`, etc.
  - But the workflow’s canonical fields are lowercase (`subject`, `from`, `urgency`, `category`, etc.).
  - Also references `{{ $json['Escalation Reason'] }}`, `{{ $json["SLA Deadline"] }}`, `{{ $json["ClickUp Task URL"] }}` which are Google Sheet column labels, not the runtime JSON keys.
- **Inputs:** From `If Urgent?` true branch
- **Outputs:** none
- **Credentials:** Slack API
- **Edge cases / failures:**
  - Message will render with blank fields unless key names are corrected.
  - Slack permission errors (missing `chat:write`) or invalid channel ID.

#### Node: Support Team Notification
- **Type / role:** Slack (`n8n-nodes-base.slack`) — direct message to a specific user.
- **Configuration (interpreted):**
  - Sends a formatted DM to user `U0ABJEE9E0G`
  - Uses mostly lowercase keys (`from`, `subject`, `category`, etc.)
- **Template variable mismatch (bug):**
  - Includes `{{ $json["Requires Escalation"] === true ? ... }}` and `{{ $json["Escalation Reason"] }}` which likely do not exist; should use `requires_escalation` / `escalation_reason`.
- **Inputs:** From `Get Clickup Task URL`
- **Outputs:** none
- **Credentials:** Slack API
- **Edge cases / failures:**
  - DM may fail if the bot cannot DM users or user ID is wrong.

#### Node: Email Acknowledgement
- **Type / role:** Gmail (`n8n-nodes-base.gmail`) — sends an acknowledgement email back to customer.
- **Configuration (interpreted):**
  - To: `{{$json.From}}` (bug: should likely be `{{$json.from}}`)
  - Subject: random case number + `{{$json.Subject}}` (bug: should likely be `{{$json.subject}}`)
  - HTML body includes a ticket summary, SLA, escalation section, and ClickUp link.
- **Template variable mismatch (bug):**
  - Mixes uppercase keys (`From`, `Subject`, `Category`, `Urgency`, `Timestamp`) with lowercase keys used elsewhere.
  - Uses `{{ $json.requires_escalation ? ... }}` correctly in one place, but also `{{ $json.urgency }}` vs `{{ $json.Urgency }}` inconsistently.
  - References `{{ $json["ClickUp Task URL"] }}` and `{{ $json["SLA Deadline"] }}` which are sheet column names, not runtime keys (`clickup_task_url`, `sla_deadline_readable`, etc.).
- **Inputs:** From `If Urgent?` (both branches)
- **Outputs:** none
- **Credentials:** Gmail OAuth2
- **Edge cases / failures:**
  - If `sendTo` is undefined because of wrong key name, Gmail node will fail.
  - HTML is accepted, but ensure the node is set to send HTML (it is providing HTML in `message` field; Gmail node supports this format).

**Sticky note covering this block:**  
“## Logging & customer response — Log to Google Sheets and send acknowledgment emails”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Fetch Support Email list | gmailTrigger | Poll support emails from Gmail label | — | Assign Email Values | ## Email intake & AI analysis<br>Fetch emails and analyze with Grok AI |
| Assign Email Values | set | Normalize email fields for AI prompt | Fetch Support Email list | Analysis with Grok | ## Email intake & AI analysis<br>Fetch emails and analyze with Grok AI |
| Analysis with Grok | httpRequest | Call Groq chat completions to classify email | Assign Email Values | Parse AI Response | ## Email intake & AI analysis<br>Fetch emails and analyze with Grok AI |
| Parse AI Response | code | Parse model JSON and merge with email metadata | Analysis with Grok | Validate AI Response | ## Email intake & AI analysis<br>Fetch emails and analyze with Grok AI |
| Validate AI Response | code | Enforce allowed values, minimum quality; add validation metadata | Parse AI Response | Smart Assignment & SLA | ## Validation & assignment<br>Validate AI output and assign to teams |
| Smart Assignment & SLA | code | Assign team/priority/tags, compute SLA deadline, set escalation flags | Validate AI Response | Create a Clickup task | ## Validation & assignment<br>Validate AI output and assign to teams |
| Create a Clickup task | clickUp | Create ClickUp task for the ticket | Smart Assignment & SLA | Get Clickup Task URL | ## Task creation & notifications<br>Create ClickUp tasks and notify via Slack |
| Get Clickup Task URL | code | Merge ClickUp task URL/ID into ticket object | Create a Clickup task | Log in Google Sheet; Support Team Notification | ## Task creation & notifications<br>Create ClickUp tasks and notify via Slack |
| Log in Google Sheet | googleSheets | Append ticket record to tracker sheet | Get Clickup Task URL | If Urgent? | ## Logging & customer response<br>Log to Google Sheets and send acknowledgment emails |
| If Urgent? | if | Branch if escalation required | Log in Google Sheet | Escalation Notification (+ Email Acknowledgement); Email Acknowledgement | ## Logging & customer response<br>Log to Google Sheets and send acknowledgment emails |
| Escalation Notification | slack | Post escalation alert to escalation channel | If Urgent? (true) | — | ## Logging & customer response<br>Log to Google Sheets and send acknowledgment emails |
| Support Team Notification | slack | DM a support user about new ticket | Get Clickup Task URL | — | ## Task creation & notifications<br>Create ClickUp tasks and notify via Slack |
| Email Acknowledgement | gmail | Send acknowledgement email to customer | If Urgent? (true/false) | — | ## Logging & customer response<br>Log to Google Sheets and send acknowledgment emails |
| Sticky Note | stickyNote | Comment block label | — | — |  |
| Sticky Note1 | stickyNote | Comment block label | — | — |  |
| Sticky Note2 | stickyNote | Comment block label | — | — |  |
| Sticky Note3 | stickyNote | Comment block label | — | — |  |
| Sticky Note4 | stickyNote | Workflow description & setup notes | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named **“Customer Support Triage”**.

2. **Add Gmail Trigger** node: `Gmail Trigger`
   - Polling: **Every minute**
   - Filters:
     - **Label**: select your support label (or paste label ID)
     - **Read status**: choose your intended behavior (recommendation: use *unread* if you want new tickets)
   - Credentials: **Gmail OAuth2** (connect the support mailbox)

3. **Add Set node**: `Set` (name it **Assign Email Values**)
   - Add fields:
     - `from` → expression from Gmail trigger output (ensure it matches your actual payload field)
     - `subject` → expression
     - `body` → expression (prefer full body if available; snippet is limited)
     - `emailId` → expression (use consistent casing; recommend `emailId`)
   - Connect: Gmail Trigger → Assign Email Values

4. **Add HTTP Request node**: `HTTP Request` (name it **Analysis with Grok**)
   - Method: POST
   - URL: `https://api.groq.com/openai/v1/chat/completions`
   - Authentication: **Groq API credential** (OpenAI-compatible bearer token)
   - Body: JSON with:
     - model `llama-3.1-8b-instant` (or your preferred Groq model)
     - messages containing the email content
     - instruction: “respond with ONLY valid JSON” and specify the schema
   - Connect: Assign Email Values → Analysis with Grok

5. **Add Code node**: `Code` (name it **Parse AI Response**)
   - Parse `choices[0].message.content` with `JSON.parse`
   - Merge parsed fields with:
     - `timestamp`, `from`, `subject`, `emailId`
   - Connect: Analysis with Grok → Parse AI Response

6. **Add Code node**: `Code` (name it **Validate AI Response**)
   - Implement:
     - Allowed category list + fallback
     - Allowed urgency list + fallback
     - Summary/action minimum length checks
     - `validation_status`, `validation_issues`
   - Connect: Parse AI Response → Validate AI Response

7. **Add Code node**: `Code` (name it **Smart Assignment & SLA**)
   - Implement routing rules for:
     - `assigned_to`, `assigned_to_email`
     - `clickup_priority`, `clickup_tags`
   - Implement SLA:
     - `sla_hours`, `sla_deadline` (ISO), `sla_deadline_readable`
   - Implement escalation flags:
     - `requires_escalation`, `escalation_reason`
   - Connect: Validate AI Response → Smart Assignment & SLA

8. **Add ClickUp node**: `ClickUp` (name it **Create a Clickup task**)
   - Credentials: ClickUp API token/OAuth
   - Select Workspace/Team, Space, and List (or paste IDs)
   - Task Name: `{{$json.subject}}`
   - Description/Content: include the merged ticket fields (from/category/urgency/summary/SLA/validation)
   - Tags: `{{$json.clickup_tags}}`
   - Priority: map to ClickUp field if supported by your node version
   - Due date: **prefer an epoch timestamp** derived from `sla_deadline` to avoid parsing issues
   - Connect: Smart Assignment & SLA → Create a Clickup task

9. **Add Code node**: `Code` (name it **Get Clickup Task URL**)
   - Merge ClickUp response (`id`, `url`) into your ticket object:
     - `clickup_task_id`, `clickup_task_url`
   - Connect: Create a Clickup task → Get Clickup Task URL

10. **Add Google Sheets node**: `Google Sheets` (name it **Log in Google Sheet**)
   - Credentials: Google Sheets OAuth2
   - Operation: **Append**
   - Select Spreadsheet + Sheet
   - Map columns to your runtime keys:
     - `Timestamp` ← `timestamp`
     - `From` ← `from`
     - `Subject` ← `subject`
     - `Category` ← `category`
     - `Urgency` ← `urgency`
     - `Summary` ← `summary`
     - `Action` ← `suggested_action`
     - `SLA Deadline` ← `sla_deadline_readable`
     - `ClickUp Task ID` ← `clickup_task_id`
     - `ClickUp Task URL` ← `clickup_task_url`
     - `Requires Escalation` ← `requires_escalation`
     - `Escalation Reason` ← `escalation_reason`
     - `Validation Status/Issues` ← `validation_status` / `validation_issues`
   - Connect: Get Clickup Task URL → Log in Google Sheet

11. **Add IF node**: `IF` (name it **If Urgent?**)
   - Condition: `{{$json.requires_escalation}}` equals `true` (boolean)
   - Connect: Log in Google Sheet → If Urgent?

12. **Add Slack node**: `Slack` (name it **Escalation Notification**)
   - Post to escalation channel (e.g., `#support-escalation`)
   - Build message using runtime keys (`from`, `subject`, `urgency`, `escalation_reason`, `clickup_task_url`, `sla_deadline_readable`)
   - Connect: If Urgent? (true) → Escalation Notification

13. **Add Slack node**: `Slack` (name it **Support Team Notification**)
   - Send to channel or DM (as desired)
   - Use runtime keys (same as above)
   - Connect: Get Clickup Task URL → Support Team Notification

14. **Add Gmail node**: `Gmail` (name it **Email Acknowledgement**)
   - Credentials: Gmail OAuth2 (the sending mailbox)
   - To: `{{$json.from}}`
   - Subject/body: include ticket details + ClickUp link
   - Connect:
     - If Urgent? (true) → Email Acknowledgement
     - If Urgent? (false) → Email Acknowledgement

15. **(Optional but recommended) Add error handling**
   - Add try/catch in the parsing Code node or an error workflow.
   - Add a fallback branch when JSON parsing fails: mark as manual review + notify internal channel.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## Customer support triage automation with AI, ClickUp and Google Sheets” + detailed “How it works” and “Setup steps” text | Sticky note describing the end-to-end intent and configuration guidance (mentions Outlook, but the actual workflow uses Gmail Trigger + Gmail send). |
| “## Email intake & AI analysis — Fetch emails and analyze with Grok AI” | Sticky note for intake + Groq block. |
| “## Validation & assignment — Validate AI output and assign to teams” | Sticky note for validation + routing block. |
| “## Task creation & notifications — Create ClickUp tasks and notify via Slack” | Sticky note for ClickUp + Slack notifications. |
| “## Logging & customer response — Log to Google Sheets and send acknowledgment emails” | Sticky note for Sheets logging + acknowledgement. |

