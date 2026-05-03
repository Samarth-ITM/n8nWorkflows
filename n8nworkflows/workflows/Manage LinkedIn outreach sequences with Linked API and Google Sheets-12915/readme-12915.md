Manage LinkedIn outreach sequences with Linked API and Google Sheets

https://n8nworkflows.xyz/workflows/manage-linkedin-outreach-sequences-with-linked-api-and-google-sheets-12915


# Manage LinkedIn outreach sequences with Linked API and Google Sheets

## 1. Workflow Overview

**Workflow title:** Manage LinkedIn outreach sequences with Linked API and Google Sheets  
**Workflow name (in JSON):** Standard outreach sequence

This workflow automates a multi-step LinkedIn outreach sequence driven by a Google Sheets “Leads” table and executed via the **Linked API** n8n node. Every 2 hours it loads leads, limits how many new connection requests can be sent per day, routes each lead by its current status, performs the appropriate LinkedIn action (send connection request, check connection status, send messages, poll for replies), and then writes status/timestamps back to Google Sheets.

### 1.1 Trigger & Configuration
Runs on a schedule, defines all runtime parameters (sheet location, delays, daily limits).

### 1.2 Lead Loading & Filtering
Reads all rows from Google Sheets and keeps only “active” statuses.

### 1.3 Daily Throttling (NEW leads only)
Enforces a daily cap of connection requests by filtering NEW leads beyond remaining quota.

### 1.4 Per-lead Iteration & Context Normalization
Iterates through leads and creates a normalized “config_* / lead_*” payload per lead.

### 1.5 Routing by Lead Status
Switches execution into one of the sequence branches:
- NEW → send connection request
- PENDING_CONNECTION → check connection status and update accordingly
- CONNECTED → send first message and sync conversation
- AWAITING_REPLY_1/2/3 → poll conversation for replies; if none, send follow-up(s) based on timing rules

### 1.6 Branch Convergence / Loop Continuation
All branches converge and continue to the next lead batch.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Config

**Overview:** Starts the workflow every 2 hours and defines configuration values used throughout (sheet link, timing delays, daily limits).  
**Nodes involved:** `⏰ Every 2 hours`, `⚙️ Config`

#### Node: ⏰ Every 2 hours
- **Type / role:** Schedule Trigger; entry point.
- **Configuration:** Runs every **2 hours**.
- **Outputs to:** `⚙️ Config`
- **Edge cases / failures:** n8n schedule drift or disabled workflow (workflow is **inactive** in JSON).

#### Node: ⚙️ Config
- **Type / role:** Set node; defines constants.
- **Key fields:**
  - `DOCUMENT_LINK`: Google Sheet URL
  - `SHEET_NAME`: “Leads”
  - `DAILY_CONNECTION_LIMIT`: 25
  - `HOURS_TO_CHECK_IF_CONNECTION_ACCEPTED`: 24
  - `HOURS_TO_CHECK_IF_REPLIED`: 4
  - `HOURS_DELAY_AFTER_CONNECTION_ACCEPTED`: 24
  - `DAYS_DELAY_BETWEEN_MESSAGES`: 2
  - `DAYS_WAIT_FOR_CONNECTION_ACCEPTANCE`: 10
  - `DAYS_WAIT_AFTER_LAST_MESSAGE`: **string** `"4"` (inconsistent type; used later as number)
- **Outputs to:** `📥 Get All Leads`
- **Edge cases:**
  - `DAYS_WAIT_AFTER_LAST_MESSAGE` is stored as string; later treated as number in expressions. This can cause runtime expression issues depending on coercion.
  - Changing date formats in the sheet may break downstream parsing.

---

### Block 2 — Load Leads & Filter Active

**Overview:** Loads all sheet rows, then keeps only leads in outreach-related statuses.  
**Nodes involved:** `📥 Get All Leads`, `🔍 Filter Active Leads`

#### Node: 📥 Get All Leads
- **Type / role:** Google Sheets node; reads rows.
- **Configuration:**
  - Document ID from `={{ $json.DOCUMENT_LINK }}`
  - Sheet name from `={{ $json.SHEET_NAME }}`
  - Uses Google Sheets OAuth2 credential: “Google Sheets (hi@linkedapi.io)”
- **Inputs:** from `⚙️ Config`
- **Outputs to:** `🔍 Filter Active Leads`
- **Edge cases / failures:**
  - OAuth token expiration, missing permissions to the document.
  - Sheet/tab name mismatch.
  - Required columns missing (Status, LinkedIn URL, etc.) will break later expressions.

#### Node: 🔍 Filter Active Leads
- **Type / role:** Filter node; only keep leads with certain `Status`.
- **Conditions (OR):** Status equals one of:
  - `NEW`, `PENDING_CONNECTION`, `CONNECTED`, `AWAITING_REPLY_1`, `AWAITING_REPLY_2`, `AWAITING_REPLY_3`
- **Outputs to:** `📊 Limit Daily Connections`
- **Edge cases:**
  - Status typos/case differences in the sheet will exclude leads.
  - Leads in terminal statuses (REPLIED/NO_RESPONSE/DECLINED/ERROR/etc.) are intentionally ignored.

---

### Block 3 — Daily Throttling

**Overview:** Ensures no more than `DAILY_CONNECTION_LIMIT` connection requests are sent per day by limiting the count of NEW leads processed.  
**Nodes involved:** `📊 Limit Daily Connections`

#### Node: 📊 Limit Daily Connections
- **Type / role:** Code node; custom throttling logic.
- **Logic summary:**
  - Reads all items.
  - Counts how many leads have `Status === 'PENDING_CONNECTION'` with `Last action date` equal to “today” (date-only match).
  - Remaining quota = `DAILY_CONNECTION_LIMIT - sentToday` (min 0).
  - If quota is 0: filters out all `NEW` leads.
  - Else: keeps all non-NEW leads + first N NEW leads.
- **Inputs:** from `🔍 Filter Active Leads`
- **Outputs to:** `🔄 Loop Over Items`
- **Edge cases / failures:**
  - The code uses `new Date(lastActionDateStr)` while comments indicate format `M/d/yyyy H:mm:ss`. In many locales/inputs, this parsing can be unreliable. If parsing fails, sentToday may be undercounted and daily cap exceeded.
  - If “Last action date” is stored as `M/d/yyyy HH:mm` (as written by update nodes), seconds are not present; JS Date parsing may still work but is not guaranteed across environments.

---

### Block 4 — Iteration & Merge Config Into Lead Payload

**Overview:** Iterates through each selected lead and creates a normalized object that contains both lead fields and config fields for easier expressions.  
**Nodes involved:** `🔄 Loop Over Items`, `🔗 Merge Config`

#### Node: 🔄 Loop Over Items
- **Type / role:** Split In Batches; processes items sequentially.
- **Configuration:** `reset: false` (keeps internal state per execution path).
- **Connections:**
  - Receives from `📊 Limit Daily Connections`
  - Output (index 1) goes to `🔗 Merge Config`
  - Receives loop-back from `➡️ Combine Branches` to continue batches
- **Edge cases:**
  - Batch size not explicitly set; defaults can change behavior depending on node defaults/version.

#### Node: 🔗 Merge Config
- **Type / role:** Set node; “joins” config + lead row into one schema.
- **Key assignments:**
  - `config_*` fields pulled from `$('⚙️ Config').item.json...`
  - `lead_*` fields pulled from current row (e.g., `lead_url = $json['LinkedIn URL']`)
  - `lead_row_number = $json.row_number` used for updates
- **Important issues in configuration:**
  - `config_days_wait_connect` and `config_days_wait_last` values are written with a leading `= ` in the expression string (`"= {{ ... }}"`). This may evaluate incorrectly (often becomes a literal string) depending on n8n expression handling.
  - `lead_last_action` assignment contains an invalid-looking id but the value is `={{ $json['Last action date'] }}` (fine).
- **Outputs to:** `🔀 Route by Status`
- **Edge cases:**
  - If a required column is missing, expressions yield `undefined` and date parsing nodes can fail.
  - The workflow assumes date strings are in `M/d/yyyy H:mm:ss` but update nodes write `M/d/yyyy HH:mm` (no seconds).

---

### Block 5 — Route by Lead Status

**Overview:** Routes each lead into the correct branch based on `lead_status`.  
**Nodes involved:** `🔀 Route by Status`

#### Node: 🔀 Route by Status
- **Type / role:** Switch node; status-based branching.
- **Rules:** `lead_status` equals:
  - `NEW`
  - `PENDING_CONNECTION`
  - `CONNECTED`
  - `AWAITING_REPLY_1`
  - `AWAITING_REPLY_2`
  - `AWAITING_REPLY_3`
- **Outputs:**
  - `NEW` → `❓ Time to Send Connection?`
  - `PENDING_CONNECTION` → `❓ Time to Check?`
  - `CONNECTED` → `❓ Time to Send Message?`
  - `AWAITING_REPLY_*` → `❓ Time to Check Reply?` (three outputs all point to same node)
- **Edge cases:**
  - Any unexpected status will be dropped (no default route configured).

---

## Branch A — NEW → Send Connection Request

**Overview:** Sends a LinkedIn connection request when the lead is eligible to send, then updates the sheet to PENDING_CONNECTION or ERROR.  
**Nodes involved:** `❓ Time to Send Connection?`, `Send connection request`, `Wait 'Send connection request'`, `❓ Successfully Sent Connection?`, `💾 Update Sheet (PENDING)`, `💾 Update Sheet (CONNECTION ERROR)`, `➡️ Do Nothing 0`

#### Node: ❓ Time to Send Connection?
- **Type / role:** IF node; decides whether it is time to send.
- **Condition (OR):**
  - `lead_next_send` is empty, OR
  - Parsed `lead_next_send` (`DateTime.fromFormat(..., 'M/d/yyyy H:mm:ss')`) is before or equal to `$today`
- **Outputs:**
  - True → `Send connection request`
  - False → `➡️ Do Nothing 0`
- **Edge cases:**
  - Uses `$today` while other checks use `$now` (inconsistency).
  - If sheet date format differs (or no seconds), `fromFormat` may fail.

#### Node: Send connection request
- **Type / role:** Linked API node; sends connection request.
- **Parameters:**
  - `personUrl = {{$json.lead_url}}`
  - `connectionNote = {{$json.lead_connection_note}}`
- **Credential:** Linked API “Vlad’s Account”
- **Outputs to:** `Wait 'Send connection request'`
- **Failures:** Linked API auth, rate limiting, invalid person URL.

#### Node: Wait 'Send connection request'
- **Type / role:** Wait node; resumes via webhook.
- **Resume mode:** `webhook`
- **Outputs to:** `❓ Successfully Sent Connection?`
- **Edge cases:**
  - Requires n8n to be reachable at webhook URL for resume.
  - If resume never occurs, execution stays waiting.

#### Node: ❓ Successfully Sent Connection?
- **Type / role:** IF; checks for API errors.
- **Condition:** `$json.body.errors` is empty (loose validation).
- **Outputs:**
  - True → `💾 Update Sheet (PENDING)`
  - False → `💾 Update Sheet (CONNECTION ERROR)`

#### Node: 💾 Update Sheet (PENDING)
- **Type / role:** Google Sheets update; sets status and dates.
- **Writes:**
  - `Status = "PENDING_CONNECTION"`
  - `Next check date = now + config_hours_check_connection`
  - `Last action date = now`
  - Matches by `row_number`
- **Date format:** `M/d/yyyy HH:mm`
- **Outputs to:** `➡️ Do Nothing 0`
- **Failures:** sheet permissions, row_number mismatch.

#### Node: 💾 Update Sheet (CONNECTION ERROR)
- **Type / role:** Google Sheets update; sets `Status = ERROR`.
- **Writes:** `Status = "ERROR"`, `Last action date = now`
- **Outputs to:** `➡️ Do Nothing 0`

#### Node: ➡️ Do Nothing 0
- **Type / role:** NoOp; branch terminator feeding convergence.
- **Outputs to:** `➡️ Combine Branches`

---

## Branch B — PENDING_CONNECTION → Check Connection Status

**Overview:** When it’s time, checks whether the connection is now connected/pending/notConnected; handles expiration based on days since last action.  
**Nodes involved:** `❓ Time to Check?`, `Check connection status`, `Wait 'Check connection status'`, `🔀 Route by Connection Status`, `💾 Update Sheet (CONNECTED)`, `❓ Connection Expired?`, `💾 Update Sheet (EXPIRED)`, `💾 Update Sheet (STILL PENDING)`, `💾 Update Sheet (DECLINED)`, `➡️ Do Nothing 1`

#### Node: ❓ Time to Check?
- **Type / role:** IF; whether to check now.
- **Condition (OR):**
  - `lead_next_check` empty, OR
  - parsed `lead_next_check` <= `$now`
- **Outputs:**
  - True → `Check connection status`
  - False → `➡️ Do Nothing 1`
- **Edge cases:** strict validation; parsing fails if format differs.

#### Node: Check connection status
- **Type / role:** Linked API; checks relationship state.
- **Parameter:** `personUrl = {{$('🔀 Route by Status').item.json.lead_url}}` (note it references switch node output rather than merged config directly)
- **Outputs to:** `Wait 'Check connection status'`

#### Node: Wait 'Check connection status'
- **Type / role:** Wait (webhook resume)
- **Outputs to:** `🔀 Route by Connection Status`

#### Node: 🔀 Route by Connection Status
- **Type / role:** Switch; uses `$json.body.data.connectionStatus`
- **Routes:**
  - `connected` → `💾 Update Sheet (CONNECTED)`
  - `pending` → `❓ Connection Expired?`
  - `notConnected` → `💾 Update Sheet (DECLINED)`
  - `other` → `➡️ Do Nothing 1` (rule appears malformed: leftValue is `"="`)
- **Edge cases:**
  - If API response structure differs, `$json.body.data.connectionStatus` may be undefined and routing fails.

#### Node: ❓ Connection Expired?
- **Type / role:** IF; determines if pending request has timed out.
- **Condition:** `$now.diff(DateTime.fromFormat(lead_last_action,'M/d/yyyy H:mm:ss'),'days').days > config_days_wait_connect`
- **Outputs:**
  - True → `💾 Update Sheet (EXPIRED)`
  - False → `💾 Update Sheet (STILL PENDING)`
- **Edge cases:**
  - If `lead_last_action` missing/invalid → DateTime parsing error.
  - If `config_days_wait_connect` is not numeric due to the earlier `= {{ ... }}` expression issue, comparison may fail.

#### Node: 💾 Update Sheet (CONNECTED)
- **Type / role:** Sheets update after acceptance.
- **Writes:**
  - `Status = CONNECTED` (value is `"=CONNECTED"` in the node; may write literally with `=` depending on Sheets behavior)
  - `Next send date = now + config_hours_after_connect`
  - `Last action date = now`
- **Outputs to:** `➡️ Do Nothing 1`

#### Node: 💾 Update Sheet (EXPIRED)
- **Type / role:** Sheets update to terminal “connection expired”.
- **Writes:** `Status = CONNECTION_EXPIRED` (also written with leading `=`), clears message stage and next dates.
- **Outputs to:** `➡️ Do Nothing 1`

#### Node: 💾 Update Sheet (STILL PENDING)
- **Type / role:** Sheets update; sets next check date again.
- **Writes:** `Next check date = now + config_hours_check_connection`
- **Outputs to:** `➡️ Do Nothing 1`

#### Node: 💾 Update Sheet (DECLINED)
- **Type / role:** Sheets update to declined.
- **Writes:** `Status = DECLINED` (leading `=`), `Last action date = now`
- **Outputs to:** `➡️ Do Nothing 1`

#### Node: ➡️ Do Nothing 1
- **Type / role:** NoOp; convergence.
- **Outputs to:** `➡️ Combine Branches`

---

## Branch C — CONNECTED → Send First Message

**Overview:** After a post-acceptance delay, sends Message 1; if sent successfully, syncs the conversation and updates status to AWAITING_REPLY_1.  
**Nodes involved:** `❓ Time to Send Message?`, `Send message`, `Wait 'Send message'`, `❓ Successfully Sent Message?`, `Sync conversation`, `Wait 'Sync conversation'`, `❓ Successfully Synced?`, `💾 Update Sheet (AWAITING_REPLY_1)`, `➡️ Do Nothing 2`

#### Node: ❓ Time to Send Message?
- **Type / role:** IF; checks if `lead_next_send` is empty or due.
- **Condition:** empty OR parsed `lead_next_send` <= `$now`
- **Outputs:**
  - True → `Send message`
  - False → `➡️ Do Nothing 2`

#### Node: Send message
- **Type / role:** Linked API send message.
- **Parameters:**
  - `personUrl = {{$('🔀 Route by Status').item.json.lead_url}}`
  - `messageText = {{$('🔀 Route by Status').item.json.lead_message_1}}`
- **Outputs:** `Wait 'Send message'`

#### Node: Wait 'Send message'
- **Type / role:** Wait (webhook resume)
- **Outputs:** `❓ Successfully Sent Message?`

#### Node: ❓ Successfully Sent Message?
- **Type / role:** IF; checks `$json.body.errors` empty.
- **Outputs:**
  - True → `Sync conversation`
  - False → `➡️ Do Nothing 2` (no explicit “MESSAGE ERROR” for Message 1 branch)

#### Node: Sync conversation
- **Type / role:** Linked API; syncs conversation thread.
- **Parameter:** `personUrl = {{lead_url}}`
- **Outputs:** `Wait 'Sync conversation'`

#### Node: Wait 'Sync conversation'
- **Type / role:** Wait (webhook resume)
- **Outputs:** `❓ Successfully Synced?`

#### Node: ❓ Successfully Synced?
- **Type / role:** IF; checks `$json.body.errors` empty.
- **Outputs:**
  - True → `💾 Update Sheet (AWAITING_REPLY_1)`
  - False → `➡️ Do Nothing 2`

#### Node: 💾 Update Sheet (AWAITING_REPLY_1)
- **Type / role:** Sheets update.
- **Writes:**
  - `Status = AWAITING_REPLY_1` (stored as `"=AWAITING_REPLY_1"`)
  - `Next send date = now + config_days_between_msgs`
  - `Next check date = now + config_hours_check_reply`
  - `Last action date = now`
- **Outputs:** `➡️ Do Nothing 2`

#### Node: ➡️ Do Nothing 2
- **Type / role:** NoOp; convergence.
- **Outputs:** `➡️ Combine Branches`

---

## Branch D — AWAITING_REPLY_1/2/3 → Poll Replies & Send Follow-ups

**Overview:** Periodically polls the LinkedIn conversation since the last action date. If the person replied, marks REPLIED; if not, sends the next follow-up when due, otherwise schedules the next check. After the final stage, marks NO_RESPONSE.  
**Nodes involved:** `❓ Time to Check Reply?`, `Poll conversations`, `❓ Person Replied?`, `💾 Update Sheet (REPLIED)`, `❓ Time for Next Message?`, `💾 Update Sheet (next check time)`, `❓ All Messages Already Sent?`, `Send N message`, `Wait 'Send N message'`, `❓ Successfully Sent N Message?`, `💾 Update Sheet (AWAITING_REPLY_2/3)`, `💾 Update Sheet (MESSAGE ERROR)`, `💾 Update Sheet (NO_RESPONSE)`, `➡️ Do Nothing 3`

#### Node: ❓ Time to Check Reply?
- **Type / role:** IF; controls polling frequency.
- **Condition (OR):**
  - `lead_next_check` empty, OR
  - parsed `lead_next_check` <= `$now`
- **Outputs:**
  - True → `Poll conversations`
  - False → `➡️ Do Nothing 3`
- **Edge cases:** loose validation; date parsing issues still possible.

#### Node: Poll conversations
- **Type / role:** Linked API; polls conversation messages.
- **Parameters:**
  - Operation: `pollConversations`
  - Payload includes:
    - `personUrl` = lead URL
    - `since` = ISO time derived from parsing `lead_last_action`
- **Outputs:** `❓ Person Replied?`
- **Failures:** API auth, conversation not found, invalid `since`.

#### Node: ❓ Person Replied?
- **Type / role:** IF; detects inbound messages.
- **Condition:** `$json.data[0].messages.filter(item => item.sender === 'them').length > 0`
- **Outputs:**
  - True → `💾 Update Sheet (REPLIED)`
  - False → `❓ Time for Next Message?`
- **Edge cases:**
  - If `$json.data` is empty or `data[0]` missing, expression throws.
  - Sender labels may differ (`them`), depending on API response.

#### Node: 💾 Update Sheet (REPLIED)
- **Type / role:** Sheets update to terminal.
- **Writes:** `Status = REPLIED` (leading `=`), `Last action date = now`
- **Outputs:** `➡️ Do Nothing 3`

#### Node: ❓ Time for Next Message?
- **Type / role:** IF; checks if next send time has arrived.
- **Outputs:**
  - True → `❓ All Messages Already Sent?`
  - False → `💾 Update Sheet (next check time)`

#### Node: 💾 Update Sheet (next check time)
- **Type / role:** Sheets update; schedules next reply check.
- **Writes:** `Next check date = now + config_hours_check_reply`, `Last action date = now`
- **Outputs:** `➡️ Do Nothing 3`

#### Node: ❓ All Messages Already Sent?
- **Type / role:** IF; determines if lead is already at stage 3.
- **Condition:** `lead_status == AWAITING_REPLY_3`
- **Outputs:**
  - True → `💾 Update Sheet (NO_RESPONSE)`
  - False → `Send N message`

#### Node: 💾 Update Sheet (NO_RESPONSE)
- **Type / role:** Sheets update to terminal.
- **Writes:** `Status = NO_RESPONSE` (leading `=`), `Last action date = now`
- **Outputs:** `➡️ Do Nothing 3`

#### Node: Send N message
- **Type / role:** Linked API send message; chooses message 2 or 3.
- **Message selection expression:**
  - If `lead_status === 'AWAITING_REPLY_1'` → use `lead_message_2`
  - Else → use `lead_message_3`
- **Outputs:** `Wait 'Send N message'`

#### Node: Wait 'Send N message'
- **Type / role:** Wait (webhook resume)
- **Outputs:** `❓ Successfully Sent N Message?`

#### Node: ❓ Successfully Sent N Message?
- **Type / role:** IF; checks `$json.body.errors` empty.
- **Outputs:**
  - True → `💾 Update Sheet (AWAITING_REPLY_2/3)`
  - False → `💾 Update Sheet (MESSAGE ERROR)`

#### Node: 💾 Update Sheet (AWAITING_REPLY_2/3)
- **Type / role:** Sheets update; advances stage.
- **Intended writes:**
  - `Status` becomes `AWAITING_REPLY_2` or `AWAITING_REPLY_3`
  - `Next send date` based on delay between messages or after last message
  - `Next check date = now + config_hours_check_reply`
  - `Last action date = now`
- **Critical configuration bug(s):**
  - Expression references `config_config_days_between_msgs` (double “config_”) which does not exist; should likely be `config_days_between_msgs`.
  - The `Next send date` expression appears syntactically incomplete (missing closing `}}` in JSON excerpt) and likely fails at runtime.
- **Outputs:** `➡️ Do Nothing 3`

#### Node: 💾 Update Sheet (MESSAGE ERROR)
- **Type / role:** Sheets update to ERROR when follow-up send fails.
- **Writes:** `Status = ERROR` (leading `=`), `Last action date = now`
- **Outputs:** `➡️ Do Nothing 3`

#### Node: ➡️ Do Nothing 3
- **Type / role:** NoOp; convergence.
- **Outputs:** `➡️ Combine Branches`

---

### Block 6 — Convergence & Looping

**Overview:** All branches converge and continue processing the next lead in the batch.  
**Nodes involved:** `➡️ Combine Branches`, plus the Do Nothing nodes as feeders.

#### Node: ➡️ Combine Branches
- **Type / role:** NoOp; used to join branches.
- **Inputs:** from `➡️ Do Nothing 0/1/2/3`
- **Outputs:** loops back to `🔄 Loop Over Items` (to fetch/process next item)
- **Edge cases:** None functional; just control flow.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| ⏰ Every 2 hours | Schedule Trigger | Periodic trigger | — | ⚙️ Config | ### Trigger & Config<br>Runs every 2 hours. Edit `Config` node to change settings. |
| ⚙️ Config | Set | Central configuration constants | ⏰ Every 2 hours | 📥 Get All Leads | ### Trigger & Config<br>Runs every 2 hours. Edit `Config` node to change settings.<br>### Configuration<br><br>All settings in `Config` node:<br><br>- `DOCUMENT_LINK` – URL to your Google Sheet<br>- `SHEET_NAME` – Name of the sheet with leads (default: Leads)<br>- `DAILY_CONNECTION_LIMIT` – Max connection requests per day (default: 25)<br>- `HOURS_TO_CHECK_IF_CONNECTION_ACCEPTED` – Check frequency for connection acceptance (default: 24)<br>- `HOURS_TO_CHECK_IF_REPLIED` – Check frequency for message replies (default: 4)<br>- `HOURS_DELAY_AFTER_CONNECTION_ACCEPTED` – Delay before first message (default: 24)<br>- `DAYS_DELAY_BETWEEN_MESSAGES` – Delay between follow-ups (default: 2)<br>- `DAYS_WAIT_FOR_CONNECTION_ACCEPTANCE` – Timeout for connection requests (default: 10)<br>- `DAYS_WAIT_AFTER_LAST_MESSAGE` – Days to wait after last message before marking as no response (default: 4) |
| 📥 Get All Leads | Google Sheets | Read all lead rows | ⚙️ Config | 🔍 Filter Active Leads | ## LinkedIn Outreach Automation<br>## How it works<br>1. **Trigger** runs every 2 hours, loads leads from Google Sheets<br>2. **Routes** each lead by status:<br>   - `NEW` → sends connection request<br>   - `PENDING` → checks if accepted/declined/expired<br>   - `CONNECTED` → sends first message<br>   - `AWAITING_REPLY` → checks for reply, sends follow-ups<br>3. **Updates** Google Sheet with new status and timestamps<br><br>## Setup steps<br>1. **Copy** [Google Sheet template](https://docs.google.com/spreadsheets/d/141fJskisAQ7H8AxtojQ7LZrnd14EOyB26RdDq5aczEU/copy)<br>2. **Connect credentials** in n8n:<br>   - Google Sheets (OAuth2)<br>   - Linked API ([get key](https://app.linkedapi.io))<br>3. **Paste** your Sheet URL into `DOCUMENT_LINK` in Config node<br>4. **Add leads** with Status = `NEW`, fill Message 1/2/3<br>5. **Activate** workflow (toggle top-right) |
| 🔍 Filter Active Leads | Filter | Keep only active statuses | 📥 Get All Leads | 📊 Limit Daily Connections |  |
| 📊 Limit Daily Connections | Code | Enforce daily NEW connection quota | 🔍 Filter Active Leads | 🔄 Loop Over Items |  |
| 🔄 Loop Over Items | Split In Batches | Iterate leads | 📊 Limit Daily Connections; ➡️ Combine Branches | 🔗 Merge Config |  |
| 🔗 Merge Config | Set | Normalize config + lead fields | 🔄 Loop Over Items | 🔀 Route by Status |  |
| 🔀 Route by Status | Switch | Route per lead status | 🔗 Merge Config | ❓ Time to Send Connection?; ❓ Time to Check?; ❓ Time to Send Message?; ❓ Time to Check Reply? |  |
| ❓ Time to Send Connection? | IF | Check send window for connection | 🔀 Route by Status | Send connection request; ➡️ Do Nothing 0 | ### NEW → Send Connection<br>Sends connection request, updates status to `PENDING` |
| Send connection request | Linked API | Send LinkedIn connection request | ❓ Time to Send Connection? | Wait 'Send connection request' | ### NEW → Send Connection<br>Sends connection request, updates status to `PENDING` |
| Wait 'Send connection request' | Wait | Async resume for connection request | Send connection request | ❓ Successfully Sent Connection? | ### NEW → Send Connection<br>Sends connection request, updates status to `PENDING` |
| ❓ Successfully Sent Connection? | IF | Validate API success | Wait 'Send connection request' | 💾 Update Sheet (PENDING); 💾 Update Sheet (CONNECTION ERROR) | ### NEW → Send Connection<br>Sends connection request, updates status to `PENDING` |
| 💾 Update Sheet (PENDING) | Google Sheets | Write PENDING_CONNECTION + dates | ❓ Successfully Sent Connection? | ➡️ Do Nothing 0 | ### NEW → Send Connection<br>Sends connection request, updates status to `PENDING` |
| 💾 Update Sheet (CONNECTION ERROR) | Google Sheets | Write ERROR after connection failure | ❓ Successfully Sent Connection? | ➡️ Do Nothing 0 | ### NEW → Send Connection<br>Sends connection request, updates status to `PENDING` |
| ➡️ Do Nothing 0 | NoOp | Branch terminator | ❓ Time to Send Connection?; 💾 Update Sheet (PENDING); 💾 Update Sheet (CONNECTION ERROR) | ➡️ Combine Branches |  |
| ❓ Time to Check? | IF | Check window to re-check connection | 🔀 Route by Status | Check connection status; ➡️ Do Nothing 1 | ### PENDING → Check Status<br>Checks if connection accepted, declined, or expired |
| Check connection status | Linked API | Query connection status | ❓ Time to Check? | Wait 'Check connection status' | ### PENDING → Check Status<br>Checks if connection accepted, declined, or expired |
| Wait 'Check connection status' | Wait | Async resume for status check | Check connection status | 🔀 Route by Connection Status | ### PENDING → Check Status<br>Checks if connection accepted, declined, or expired |
| 🔀 Route by Connection Status | Switch | Route by API connectionStatus | Wait 'Check connection status' | 💾 Update Sheet (CONNECTED); ❓ Connection Expired?; 💾 Update Sheet (DECLINED); ➡️ Do Nothing 1 | ### PENDING → Check Status<br>Checks if connection accepted, declined, or expired |
| ❓ Connection Expired? | IF | Pending timeout logic | 🔀 Route by Connection Status | 💾 Update Sheet (EXPIRED); 💾 Update Sheet (STILL PENDING) | ### PENDING → Check Status<br>Checks if connection accepted, declined, or expired |
| 💾 Update Sheet (CONNECTED) | Google Sheets | Mark CONNECTED and schedule message | 🔀 Route by Connection Status | ➡️ Do Nothing 1 | ### PENDING → Check Status<br>Checks if connection accepted, declined, or expired |
| 💾 Update Sheet (EXPIRED) | Google Sheets | Mark CONNECTION_EXPIRED | ❓ Connection Expired? | ➡️ Do Nothing 1 | ### PENDING → Check Status<br>Checks if connection accepted, declined, or expired |
| 💾 Update Sheet (STILL PENDING) | Google Sheets | Push next check date forward | ❓ Connection Expired? | ➡️ Do Nothing 1 | ### PENDING → Check Status<br>Checks if connection accepted, declined, or expired |
| 💾 Update Sheet (DECLINED) | Google Sheets | Mark DECLINED | 🔀 Route by Connection Status | ➡️ Do Nothing 1 | ### PENDING → Check Status<br>Checks if connection accepted, declined, or expired |
| ➡️ Do Nothing 1 | NoOp | Branch terminator | ❓ Time to Check?; 💾 Update Sheet (CONNECTED); 💾 Update Sheet (EXPIRED); 💾 Update Sheet (STILL PENDING); 💾 Update Sheet (DECLINED); 🔀 Route by Connection Status | ➡️ Combine Branches |  |
| ❓ Time to Send Message? | IF | Check send window for message 1 | 🔀 Route by Status | Send message; ➡️ Do Nothing 2 | ### CONNECTED → Send Message<br>Sends first message after connection accepted |
| Send message | Linked API | Send Message 1 | ❓ Time to Send Message? | Wait 'Send message' | ### CONNECTED → Send Message<br>Sends first message after connection accepted |
| Wait 'Send message' | Wait | Async resume for message send | Send message | ❓ Successfully Sent Message? | ### CONNECTED → Send Message<br>Sends first message after connection accepted |
| ❓ Successfully Sent Message? | IF | Validate message send success | Wait 'Send message' | Sync conversation; ➡️ Do Nothing 2 | ### CONNECTED → Send Message<br>Sends first message after connection accepted |
| Sync conversation | Linked API | Sync conversation thread | ❓ Successfully Sent Message? | Wait 'Sync conversation' | ### CONNECTED → Send Message<br>Sends first message after connection accepted |
| Wait 'Sync conversation' | Wait | Async resume for sync | Sync conversation | ❓ Successfully Synced? | ### CONNECTED → Send Message<br>Sends first message after connection accepted |
| ❓ Successfully Synced? | IF | Validate sync success | Wait 'Sync conversation' | 💾 Update Sheet (AWAITING_REPLY_1); ➡️ Do Nothing 2 | ### CONNECTED → Send Message<br>Sends first message after connection accepted |
| 💾 Update Sheet (AWAITING_REPLY_1) | Google Sheets | Set stage 1, schedule follow-up/check | ❓ Successfully Synced? | ➡️ Do Nothing 2 | ### CONNECTED → Send Message<br>Sends first message after connection accepted |
| ➡️ Do Nothing 2 | NoOp | Branch terminator | ❓ Time to Send Message?; ❓ Successfully Sent Message?; ❓ Successfully Synced?; 💾 Update Sheet (AWAITING_REPLY_1) | ➡️ Combine Branches |  |
| ❓ Time to Check Reply? | IF | Throttle conversation polling | 🔀 Route by Status | Poll conversations; ➡️ Do Nothing 3 | ### AWAITING_REPLY → Follow-up<br>Checks for reply, sends follow-up messages (2 & 3) |
| Poll conversations | Linked API | Fetch conversation messages since last action | ❓ Time to Check Reply? | ❓ Person Replied? | ### AWAITING_REPLY → Follow-up<br>Checks for reply, sends follow-up messages (2 & 3) |
| ❓ Person Replied? | IF | Detect inbound response | Poll conversations | 💾 Update Sheet (REPLIED); ❓ Time for Next Message? | ### AWAITING_REPLY → Follow-up<br>Checks for reply, sends follow-up messages (2 & 3) |
| 💾 Update Sheet (REPLIED) | Google Sheets | Mark REPLIED | ❓ Person Replied? | ➡️ Do Nothing 3 | ### AWAITING_REPLY → Follow-up<br>Checks for reply, sends follow-up messages (2 & 3) |
| ❓ Time for Next Message? | IF | Check follow-up send window | ❓ Person Replied? | ❓ All Messages Already Sent?; 💾 Update Sheet (next check time) | ### AWAITING_REPLY → Follow-up<br>Checks for reply, sends follow-up messages (2 & 3) |
| 💾 Update Sheet (next check time) | Google Sheets | Schedule next reply check | ❓ Time for Next Message? | ➡️ Do Nothing 3 | ### AWAITING_REPLY → Follow-up<br>Checks for reply, sends follow-up messages (2 & 3) |
| ❓ All Messages Already Sent? | IF | If stage 3 reached, stop | ❓ Time for Next Message? | 💾 Update Sheet (NO_RESPONSE); Send N message | ### AWAITING_REPLY → Follow-up<br>Checks for reply, sends follow-up messages (2 & 3) |
| 💾 Update Sheet (NO_RESPONSE) | Google Sheets | Mark NO_RESPONSE | ❓ All Messages Already Sent? | ➡️ Do Nothing 3 | ### AWAITING_REPLY → Follow-up<br>Checks for reply, sends follow-up messages (2 & 3) |
| Send N message | Linked API | Send Message 2 or 3 | ❓ All Messages Already Sent? | Wait 'Send N message' | ### AWAITING_REPLY → Follow-up<br>Checks for reply, sends follow-up messages (2 & 3) |
| Wait 'Send N message' | Wait | Async resume for follow-up send | Send N message | ❓ Successfully Sent N Message? | ### AWAITING_REPLY → Follow-up<br>Checks for reply, sends follow-up messages (2 & 3) |
| ❓ Successfully Sent N Message? | IF | Validate follow-up send | Wait 'Send N message' | 💾 Update Sheet (AWAITING_REPLY_2/3); 💾 Update Sheet (MESSAGE ERROR) | ### AWAITING_REPLY → Follow-up<br>Checks for reply, sends follow-up messages (2 & 3) |
| 💾 Update Sheet (AWAITING_REPLY_2/3) | Google Sheets | Advance stage and schedule times | ❓ Successfully Sent N Message? | ➡️ Do Nothing 3 | ### AWAITING_REPLY → Follow-up<br>Checks for reply, sends follow-up messages (2 & 3) |
| 💾 Update Sheet (MESSAGE ERROR) | Google Sheets | Mark ERROR on follow-up failure | ❓ Successfully Sent N Message? | ➡️ Do Nothing 3 | ### AWAITING_REPLY → Follow-up<br>Checks for reply, sends follow-up messages (2 & 3) |
| ➡️ Do Nothing 3 | NoOp | Branch terminator | ❓ Time to Check Reply?; 💾 Update Sheet (REPLIED); 💾 Update Sheet (NO_RESPONSE); 💾 Update Sheet (next check time); 💾 Update Sheet (AWAITING_REPLY_2/3); 💾 Update Sheet (MESSAGE ERROR) | ➡️ Combine Branches |  |
| ➡️ Combine Branches | NoOp | Converge branches and continue loop | ➡️ Do Nothing 0/1/2/3 | 🔄 Loop Over Items |  |
| Overview | Sticky Note | Canvas documentation | — | — |  |
| Sticky Note NEW | Sticky Note | Canvas documentation | — | — |  |
| Sticky Note PENDING | Sticky Note | Canvas documentation | — | — |  |
| Sticky Note CONNECTED | Sticky Note | Canvas documentation | — | — |  |
| Sticky Note AWAITING | Sticky Note | Canvas documentation | — | — |  |
| Sticky Note Config | Sticky Note | Canvas documentation | — | — |  |
| Sticky Note Configuration | Sticky Note | Canvas documentation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named “Standard outreach sequence”.

2. **Add trigger**
   1. Add **Schedule Trigger** node named `⏰ Every 2 hours`.
   2. Set interval: **every 2 hours**.

3. **Add configuration**
   1. Add **Set** node named `⚙️ Config`.
   2. Add fields:
      - `DOCUMENT_LINK` (string): your Google Sheet URL
      - `SHEET_NAME` (string): `Leads`
      - `DAILY_CONNECTION_LIMIT` (number): 25
      - `HOURS_TO_CHECK_IF_CONNECTION_ACCEPTED` (number): 24
      - `HOURS_TO_CHECK_IF_REPLIED` (number): 4
      - `HOURS_DELAY_AFTER_CONNECTION_ACCEPTED` (number): 24
      - `DAYS_DELAY_BETWEEN_MESSAGES` (number): 2
      - `DAYS_WAIT_FOR_CONNECTION_ACCEPTANCE` (number): 10
      - `DAYS_WAIT_AFTER_LAST_MESSAGE` (number): 4 (store as number to avoid type problems)
   3. Connect: `⏰ Every 2 hours` → `⚙️ Config`.

4. **Read leads from Google Sheets**
   1. Add **Google Sheets** node named `📥 Get All Leads`.
   2. Operation: **Read / Get Many** (the node in JSON reads all rows).
   3. Set:
      - Document: expression `{{$json.DOCUMENT_LINK}}`
      - Sheet name: expression `{{$json.SHEET_NAME}}`
   4. Configure **Google Sheets OAuth2** credentials with access to the sheet.
   5. Connect: `⚙️ Config` → `📥 Get All Leads`.

5. **Filter active statuses**
   1. Add **Filter** node named `🔍 Filter Active Leads`.
   2. Create an OR group for `Status` equals:
      - NEW
      - PENDING_CONNECTION
      - CONNECTED
      - AWAITING_REPLY_1
      - AWAITING_REPLY_2
      - AWAITING_REPLY_3
   3. Connect: `📥 Get All Leads` → `🔍 Filter Active Leads`.

6. **Add daily connection limit**
   1. Add **Code** node named `📊 Limit Daily Connections`.
   2. Paste the JS logic from the workflow (or re-implement):
      - Count today’s `PENDING_CONNECTION` with Last action date = today
      - Allow only remaining quota of NEW leads
   3. Connect: `🔍 Filter Active Leads` → `📊 Limit Daily Connections`.

7. **Loop over leads**
   1. Add **Split in Batches** node named `🔄 Loop Over Items`.
   2. Keep defaults (or set batch size = 1 for strict sequential behavior).
   3. Connect: `📊 Limit Daily Connections` → `🔄 Loop Over Items`.

8. **Merge config into each lead**
   1. Add **Set** node named `🔗 Merge Config`.
   2. Map config into:
      - `config_document_link = {{$('⚙️ Config').first().json.DOCUMENT_LINK}}`
      - `config_sheet_name = {{$('⚙️ Config').first().json.SHEET_NAME}}`
      - and other timing/limit fields similarly
   3. Map lead fields into:
      - `lead_url = {{$json['LinkedIn URL']}}`
      - `lead_name = {{$json.Name}}`
      - `lead_company = {{$json.Company}}`
      - `lead_status = {{$json.Status}}`
      - `lead_next_send = {{$json['Next send date']}}`
      - `lead_next_check = {{$json['Next check date']}}`
      - `lead_last_action = {{$json['Last action date']}}`
      - `lead_message_1/2/3`, `lead_connection_note`
      - `lead_row_number = {{$json.row_number}}`
   4. Connect: `🔄 Loop Over Items` → `🔗 Merge Config`.

9. **Route by status**
   1. Add **Switch** node named `🔀 Route by Status` on `{{$json.lead_status}}`.
   2. Create outputs for each status (NEW, PENDING_CONNECTION, CONNECTED, AWAITING_REPLY_1/2/3).
   3. Connect: `🔗 Merge Config` → `🔀 Route by Status`.

10. **Implement NEW branch**
   1. Add **IF** `❓ Time to Send Connection?` checking:
      - lead_next_send empty OR lead_next_send <= now (use consistent date format)
   2. Add **Linked API** node `Send connection request` (operation `sendConnectionRequest`) with:
      - `personUrl = {{$json.lead_url}}`
      - `connectionNote = {{$json.lead_connection_note}}`
   3. Add **Wait** node `Wait 'Send connection request'` (resume by webhook).
   4. Add **IF** `❓ Successfully Sent Connection?` checking `{{$json.body.errors}}` is empty.
   5. Add **Google Sheets Update** nodes:
      - `💾 Update Sheet (PENDING)` writing Status, Next check date, Last action date by `row_number`
      - `💾 Update Sheet (CONNECTION ERROR)` writing Status=ERROR and Last action date
   6. Add **NoOp** `➡️ Do Nothing 0` for false paths and to converge.

11. **Implement PENDING_CONNECTION branch**
   1. Add **IF** `❓ Time to Check?` (lead_next_check empty OR <= now).
   2. Add **Linked API** `Check connection status` (operation `checkConnectionStatus`, personUrl lead_url).
   3. Add **Wait** `Wait 'Check connection status'`.
   4. Add **Switch** `🔀 Route by Connection Status` on `{{$json.body.data.connectionStatus}}` with cases:
      - connected, pending, notConnected
   5. For `pending`, add **IF** `❓ Connection Expired?` comparing days since last action > `DAYS_WAIT_FOR_CONNECTION_ACCEPTANCE`.
   6. Add Sheets updates:
      - CONNECTED → set Status=CONNECTED, Next send date = now + HOURS_DELAY_AFTER_CONNECTION_ACCEPTED
      - STILL PENDING → bump Next check date
      - EXPIRED → set Status=CONNECTION_EXPIRED and clear next dates
      - DECLINED → set Status=DECLINED
   7. Add **NoOp** `➡️ Do Nothing 1` to converge.

12. **Implement CONNECTED branch (Message 1)**
   1. Add **IF** `❓ Time to Send Message?` (lead_next_send empty OR <= now).
   2. Add **Linked API** `Send message` (operation `sendMessage`):
      - personUrl lead_url
      - messageText lead_message_1
   3. Add **Wait** `Wait 'Send message'`.
   4. Add **IF** `❓ Successfully Sent Message?` checking `body.errors` empty.
   5. Add **Linked API** `Sync conversation` (operation `syncConversation`).
   6. Add **Wait** `Wait 'Sync conversation'`.
   7. Add **IF** `❓ Successfully Synced?`.
   8. Add Sheets update `💾 Update Sheet (AWAITING_REPLY_1)`:
      - Status=AWAITING_REPLY_1
      - Next send date = now + DAYS_DELAY_BETWEEN_MESSAGES
      - Next check date = now + HOURS_TO_CHECK_IF_REPLIED
   9. Add **NoOp** `➡️ Do Nothing 2`.

13. **Implement AWAITING_REPLY branch**
   1. Add **IF** `❓ Time to Check Reply?` (lead_next_check empty OR <= now).
   2. Add **Linked API** `Poll conversations` (operation `pollConversations`) and pass `personUrl` and `since` from last action date.
   3. Add **IF** `❓ Person Replied?` (detect inbound message from “them”).
   4. If replied: update sheet to REPLIED.
   5. If not replied:
      - IF `❓ Time for Next Message?` (lead_next_send empty OR <= now)
      - Else update only Next check date
      - IF `❓ All Messages Already Sent?` (status == AWAITING_REPLY_3)
        - If yes: set NO_RESPONSE
        - If no: send follow-up (message 2 or 3), wait, validate, update stage and next dates
      - On send failure: set ERROR
   6. Add **NoOp** `➡️ Do Nothing 3`.

14. **Converge and continue loop**
   1. Add **NoOp** `➡️ Combine Branches`.
   2. Connect all Do Nothing nodes to `➡️ Combine Branches`.
   3. Connect `➡️ Combine Branches` back to `🔄 Loop Over Items` to continue with next lead.

15. **Credentials**
   - **Google Sheets OAuth2**: authorize a Google account that can read/write the sheet.
   - **Linked API**: configure the Linked API credential (API key from https://app.linkedapi.io) and select it in all Linked API nodes.

16. **Sheet requirements**
   Ensure columns exist (matching names used by expressions):
   - `LinkedIn URL`, `Name`, `Company`, `Status`, `Connection note`, `Message 1`, `Message 2`, `Message 3`, `Next send date`, `Next check date`, `Last action date`, and `row_number` (n8n read metadata).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Copy Google Sheet template | https://docs.google.com/spreadsheets/d/141fJskisAQ7H8AxtojQ7LZrnd14EOyB26RdDq5aczEU/copy |
| Get Linked API key | https://app.linkedapi.io |
| Workflow overview & setup steps (embedded in sticky note) | Included in the “Overview” sticky note content within the canvas |

