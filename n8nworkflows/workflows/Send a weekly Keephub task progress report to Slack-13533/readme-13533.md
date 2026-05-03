Send a weekly Keephub task progress report to Slack

https://n8nworkflows.xyz/workflows/send-a-weekly-keephub-task-progress-report-to-slack-13533


# Send a weekly Keephub task progress report to Slack

## 1. Workflow Overview

**Purpose:** Send a weekly progress report of Keephub tasks (for a specific org unit) to Slack. The workflow fetches tasks from the past week, retrieves each task’s progress, formats a Slack-ready summary, and posts it.

**Target use cases:**
- Weekly operations reporting for a team/org unit
- Monitoring overdue/open/done tasks with a single Slack message
- Easily adjustable reporting window (weekly → monthly) by changing date logic

### 1.1 Triggers (Automatic + Manual)
Runs automatically every Monday at 09:00, with an optional manual trigger for testing.

### 1.2 Date Window Calculation (Last Week Start/End)
Computes `lastWeekStart` (now minus 7 days) and `lastWeekEnd` (start plus 7 days).

### 1.3 Keephub Task Retrieval (Org Unit + Date Filter)
Fetches tasks in the computed date range for the configured org unit.

### 1.4 Per-Task Progress Retrieval + Merge
Extracts task IDs/titles, calls Keephub to get progress per task, then merges progress back with task metadata.

### 1.5 Message Formatting + Slack Delivery
Builds a categorized weekly report (overdue/open/done + totals) and sends it to Slack.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Triggers (Automatic + Manual)
**Overview:** Starts the workflow either on schedule (weekly) or manually for debugging and ad-hoc runs.  
**Nodes involved:** `Every Monday 9am`, `Or Manually Run`

#### Node: Every Monday 9am
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — automated entry point.
- **Configuration (interpreted):**
  - Interval: weekly
  - Day: Monday (`triggerAtDay: [1]`)
  - Hour: 09:00
- **Outputs:** To `Last Week Start`
- **Edge cases / failures:**
  - Timezone depends on n8n instance settings; “9am” is instance-timezone 9am unless otherwise configured.

#### Node: Or Manually Run
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — manual entry point for testing.
- **Outputs:** To `Last Week Start`
- **Edge cases / failures:** None (but manual runs often reveal credential/config issues downstream).

---

### Block 2.2 — Date Window Calculation
**Overview:** Computes the reporting window used to filter Keephub tasks: start = now - 7 days, end = start + 7 days.  
**Nodes involved:** `Last Week Start`, `Last Week End`  
**Sticky note(s):**
- “Dates can be set per your needs, so you can easily change this into a monthly report.”

#### Node: Last Week Start
- **Type / role:** Date & Time (`n8n-nodes-base.dateTime`) — calculates start date.
- **Configuration (interpreted):**
  - Operation: subtract duration from a date
  - Base date: `{{$now}}`
  - Duration: `7` days (implicit via `duration: 7`)
  - Output field: `lastWeekStart`
- **Input:** Trigger node
- **Output:** To `Last Week End`
- **Key variables/expressions:**
  - `={{ $now }}` as the magnitude/base date
- **Edge cases / failures:**
  - If you expect “previous calendar week (Mon–Sun)”, this logic is a rolling 7-day window, not calendar-aligned.

#### Node: Last Week End
- **Type / role:** Date & Time (`n8n-nodes-base.dateTime`) — calculates end date.
- **Configuration (interpreted):**
  - Operation: add duration to a date
  - Base date: `={{ $json.lastWeekStart }}`
  - Duration: `7` days
  - Output field: `lastWeekEnd`
- **Input:** From `Last Week Start`
- **Output:** To `Get Tasks by Orgunit`
- **Key variables/expressions:**
  - `={{ $json.lastWeekStart }}`
- **Edge cases / failures:**
  - This makes `lastWeekEnd` roughly “now”; if you want “last week only” (excluding today), you may want end = `$now` and start = `$now - 7 days`, or explicitly floor/ceil to day boundaries.

---

### Block 2.3 — Keephub Task Retrieval (Org Unit + Date Filter)
**Overview:** Pulls up to 200 tasks belonging to a given org unit, filtered by start date range, sorted by due date.  
**Nodes involved:** `Get Tasks by Orgunit`  
**Sticky note(s):**
- “⚙️ Change `orgunitId` to your root org unit ID.”

#### Node: Get Tasks by Orgunit
- **Type / role:** Keephub node (`n8n-nodes-keephub.keephub`) — fetch tasks for an org unit within a date range.
- **Configuration (interpreted):**
  - Resource: `task`
  - Operation: `getTaskByOrgunit`
  - Auth: “loginCredentials” (uses Keephub Login credential)
  - `orgunitId`: placeholder `YOUR_ORG_UNIT_ID` (must be replaced)
  - Options:
    - `limit: 200`
    - Sort by: `template.dueDate` ascending (`sortOrder: 1`)
    - Date filters:
      - `startDateGte`: `={{ $('Last Week Start').first().json.lastWeekStart }}`
      - `startDateLte`: `={{ $('Last Week End').first().json.lastWeekEnd }}`
- **Input:** From `Last Week End`
- **Output:** To `Extract Task Template IDs`
- **Credentials:**
  - Keephub Login account (`keephubLoginApi`)
- **Edge cases / failures:**
  - **Auth failure:** invalid/expired login.
  - **Org unit mismatch:** wrong `orgunitId` returns empty results.
  - **Pagination/limits:** limit is 200; more tasks will be truncated unless pagination is implemented.
  - **Field availability:** sorting by `template.dueDate` assumes tasks contain that path; missing fields may affect ordering.

---

### Block 2.4 — Progress Retrieval Per Task + Merge
**Overview:** Extracts task IDs/titles, fetches progress for each task, and merges progress into the original task metadata by item position.  
**Nodes involved:** `Extract Task Template IDs`, `Get Progress`, `Merge`

#### Node: Extract Task Template IDs
- **Type / role:** Code node (`n8n-nodes-base.code`) — transforms task list into a compact list for progress calls.
- **Configuration (interpreted):**
  - For each incoming item, returns:
    - `taskTemplateId`: `item.json._id`
    - `title`: `item.json.template?.title?.en || item.json.title?.en || item.json._id`
- **Input:** From `Get Tasks by Orgunit`
- **Outputs:**
  - To `Get Progress` (to fetch progress per task)
  - To `Merge` input 2 (index 1) to preserve title/id metadata for later merge
- **Key logic details:**
  - Uses optional chaining; expects Keephub task objects to have `_id`, and optionally localized titles.
- **Edge cases / failures:**
  - If `_id` is missing, downstream `Get Progress` will fail or return error.
  - Titles might not be `en`; you may want fallback for other locales.

#### Node: Get Progress
- **Type / role:** Keephub node (`n8n-nodes-keephub.keephub`) — retrieves progress for a single task.
- **Configuration (interpreted):**
  - Resource: `task`
  - Operation: `getTaskProgress`
  - `taskId`: `={{ $json.taskTemplateId }}`
  - Auth: Bearer credential (separate from login)
- **Input:** From `Extract Task Template IDs` (one item per task)
- **Output:** To `Merge` input 1 (index 0)
- **Credentials:**
  - Keephub Bearer account (`keephubBearerApi`)
- **Edge cases / failures:**
  - **Auth failure:** invalid/expired bearer token.
  - **Rate limits/timeouts:** called once per task; large task sets may hit API limits or slow runs.
  - **Unexpected response shape:** formatter expects `fullProgress[0]` later.

#### Node: Merge
- **Type / role:** Merge node (`n8n-nodes-base.merge`) — combines per-task metadata + progress.
- **Configuration (interpreted):**
  - Mode: `combine`
  - Combine by: `position` (combineByPosition)
- **Inputs:**
  - Input 1 (index 0): from `Get Progress`
  - Input 2 (index 1): from `Extract Task Template IDs`
- **Output:** To `Format for Message`
- **Edge cases / failures:**
  - **Order dependence:** combine-by-position assumes the progress results and metadata lists stay perfectly aligned.
    - If any `Get Progress` call errors and drops an item, or if execution changes item ordering, merge alignment can break (wrong progress attached to wrong title).
  - If you need robust merging, prefer merging by a key (`taskTemplateId`) rather than position.

---

### Block 2.5 — Message Formatting + Slack Delivery
**Overview:** Aggregates task progress into totals and categorized lists (overdue/open/done) and posts a formatted Slack message.  
**Nodes involved:** `Format for Message`, `Send a message`  
**Sticky note(s):**
- “Formats the message so it can be sent to the communication channel of your choice”

#### Node: Format for Message
- **Type / role:** Code node (`n8n-nodes-base.code`) — builds Slack message text.
- **Configuration (interpreted):**
  - Creates a human-readable date string in `en-GB` (weekday/day/month/year).
  - Iterates over all merged items:
    - Reads progress from: `item.json.fullProgress?.[0] ?? {}`
    - Extracts:
      - `done` (default 0)
      - `open` as `openCount` (default 0)
      - `status` (default `'unknown'`)
    - Computes totals and per-task percent
    - Categorizes lines into:
      - overdue → 🔴
      - open → 🟡
      - done → ✅
  - Outputs a single item:
    - `summary`: formatted Slack message
    - `stats`: totals object
- **Input:** From `Merge`
- **Output:** To `Send a message`
- **Edge cases / failures:**
  - If Keephub returns progress under a different property than `fullProgress`, the report will show zeros/unknowns.
  - Very large reports may exceed Slack message size limits; consider truncation or file upload if needed.
  - Locale/timezone: `new Date()` uses server timezone.

#### Node: Send a message
- **Type / role:** Slack node (`n8n-nodes-base.slack`) — posts the message to Slack.
- **Configuration (interpreted):**
  - Message text: `={{ $json.summary }}`
  - Channel is not shown in the JSON snippet (must be set in node UI / parameters depending on Slack node configuration).
- **Input:** From `Format for Message`
- **Output:** End
- **Credentials:** Slack OAuth/token credentials (not included in JSON export here).
- **Edge cases / failures:**
  - Missing Slack credentials or insufficient scopes (e.g., `chat:write`).
  - Channel not found / bot not invited to channel.
  - Message formatting: Slack markdown is supported but some formatting differs across clients.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every Monday 9am | Schedule Trigger | Automatic weekly trigger | — | Last Week Start |  |
| Or Manually Run | Manual Trigger | Manual entry point | — | Last Week Start |  |
| Last Week Start | Date & Time | Compute start of reporting window | Every Monday 9am; Or Manually Run | Last Week End | Dates can be set per your needs, so you can easily change this into a monthly report. |
| Last Week End | Date & Time | Compute end of reporting window | Last Week Start | Get Tasks by Orgunit | Dates can be set per your needs, so you can easily change this into a monthly report. |
| Get Tasks by Orgunit | Keephub | Fetch tasks for org unit in date range | Last Week End | Extract Task Template IDs | ⚙️ Change `orgunitId` to your root org unit ID. |
| Extract Task Template IDs | Code | Map tasks to `{taskTemplateId,title}` list | Get Tasks by Orgunit | Get Progress; Merge |  |
| Get Progress | Keephub | Fetch progress per task | Extract Task Template IDs | Merge |  |
| Merge | Merge | Combine task metadata and progress | Get Progress; Extract Task Template IDs | Format for Message |  |
| Format for Message | Code | Build Slack summary text | Merge | Send a message | Formats the message so it can be sent to the communication channel of your choice |
| Send a message | Slack | Post the report to Slack | Format for Message | — |  |
| Sticky Note | Sticky Note | Comment | — | — | Dates can be set per your needs, so you can easily change this into a monthly report. |
| Sticky Note1 | Sticky Note | Comment | — | — | Formats the message so it can be sent to the communication channel of your choice |
| Sticky Note2 | Sticky Note | Comment | — | — | ## 📊 Keephub Weekly Task Report<br><br>Fetches all tasks for an org unit from the past week,<br>gets their progress, and sends a formatted Slack summary.<br><br>**Setup:**<br>1. Add your Keephub Login credential<br>2. Add your Keephub Bearer credential<br>3. Set your orgunit ID in "Get Tasks by Orgunit"<br>4. Set your Slack channel in the Slack node<br><br>Runs automatically every Monday at 9:00am. |
| Sticky Note3 | Sticky Note | Comment | — | — | ⚙️ Change `orgunitId` to your root org unit ID. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger: “Every Monday 9am”**
   - Add node: **Schedule Trigger**
   - Set schedule to **Weekly**
   - Choose **Monday**
   - Set time to **09:00** (confirm instance timezone)

2. **(Optional) Create Trigger: “Or Manually Run”**
   - Add node: **Manual Trigger**
   - This is used for testing. You’ll connect it to the same next node as the schedule trigger.

3. **Create date calculation: “Last Week Start”**
   - Add node: **Date & Time**
   - Operation: **Subtract from date**
   - Base date/value: **Now** (expression equivalent to `$now`)
   - Amount: **7 days**
   - Output field name: `lastWeekStart`
   - Connect:
     - `Every Monday 9am` → `Last Week Start`
     - `Or Manually Run` → `Last Week Start`

4. **Create date calculation: “Last Week End”**
   - Add node: **Date & Time**
   - Operation: **Add to date**
   - Base date/value: set to `lastWeekStart` from previous node (expression)
   - Amount: **7 days**
   - Output field name: `lastWeekEnd`
   - Connect: `Last Week Start` → `Last Week End`

5. **Create Keephub fetch: “Get Tasks by Orgunit”**
   - Add node: **Keephub**
   - Resource: **Task**
   - Operation: **Get Task by Orgunit**
   - Authentication: **Login Credentials**
   - Configure credential: create/select **Keephub Login** credential in n8n
   - Set `orgunitId` to your org unit (replace placeholder)
   - Options:
     - Limit: `200`
     - Sort by: `template.dueDate`
     - Sort order: ascending
     - `startDateGte`: expression pointing to `Last Week Start` → `lastWeekStart`
     - `startDateLte`: expression pointing to `Last Week End` → `lastWeekEnd`
   - Connect: `Last Week End` → `Get Tasks by Orgunit`

6. **Create mapping: “Extract Task Template IDs”**
   - Add node: **Code**
   - Paste logic that outputs one item per task with:
     - `taskTemplateId` from incoming task `_id`
     - `title` from `template.title.en`, fallback to `title.en`, fallback to `_id`
   - Connect: `Get Tasks by Orgunit` → `Extract Task Template IDs`

7. **Create progress fetch: “Get Progress”**
   - Add node: **Keephub**
   - Resource: **Task**
   - Operation: **Get Task Progress**
   - Authentication: **Bearer**
   - Configure credential: create/select **Keephub Bearer** credential in n8n
   - Task ID: expression `{{$json.taskTemplateId}}`
   - Connect: `Extract Task Template IDs` → `Get Progress`

8. **Create merge: “Merge”**
   - Add node: **Merge**
   - Mode: **Combine**
   - Combine by: **Position**
   - Connect:
     - `Get Progress` → `Merge` (Input 1)
     - `Extract Task Template IDs` → `Merge` (Input 2)

9. **Create formatter: “Format for Message”**
   - Add node: **Code**
   - Implement logic to:
     - Iterate items
     - Read progress from `fullProgress[0]`
     - Build totals + categorized lists (overdue/open/done)
     - Output a single item with `summary` and `stats`
   - Connect: `Merge` → `Format for Message`

10. **Create Slack delivery: “Send a message”**
   - Add node: **Slack**
   - Operation: **Send Message** (or equivalent in your Slack node version)
   - Credentials: connect Slack OAuth/token credential with `chat:write`
   - Choose **Channel** (must be set in node UI)
   - Text: expression `{{$json.summary}}`
   - Connect: `Format for Message` → `Send a message`

11. **Add sticky notes (optional but recommended for maintainability)**
   - Add notes reflecting:
     - Date window adjustability (weekly/monthly)
     - Setting `orgunitId`
     - Message formatting block purpose
     - Setup checklist (Keephub login, Keephub bearer, Slack channel)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Dates can be set per your needs, so you can easily change this into a monthly report. | Applies to the “Last Week Start/End” date window logic. |
| Formats the message so it can be sent to the communication channel of your choice | Applies to the message formatting step before sending to Slack. |
| ## 📊 Keephub Weekly Task Report … (setup checklist + run schedule) | High-level workflow intent + required credentials + orgunitId + Slack channel configuration. |
| ⚙️ Change `orgunitId` to your root org unit ID. | Required configuration in “Get Tasks by Orgunit”. |