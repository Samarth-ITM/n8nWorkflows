Track employee performance KPIs from ClickUp with GPT-4.1 and Google Sheets

https://n8nworkflows.xyz/workflows/track-employee-performance-kpis-from-clickup-with-gpt-4-1-and-google-sheets-12579


# Track employee performance KPIs from ClickUp with GPT-4.1 and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Employee Performance AI Dashboard  
**Purpose:** On a schedule, fetch tasks from ClickUp, generate an AI-written structured task performance summary, normalize/score KPIs in JavaScript, then **append or update** a Google Sheets table to maintain a live employee KPI dashboard.

### 1.1 Scheduled execution (entry)
Runs automatically at the chosen interval.

### 1.2 ClickUp data collection + iteration
Gets folders from a ClickUp Space, then loops and fetches tasks (folderless tasks) and processes them one-by-one.

### 1.3 AI task analysis
Sends each task to an OpenAI model with a strict prompt to output a plain-text structured “Task Summary” including computed metrics.

### 1.4 KPI normalization and scoring
Parses the AI text into structured JSON, then recalculates KPI score and rating deterministically in code.

### 1.5 KPI reporting to Google Sheets (and loop continuation)
Writes the task-level KPI fields to a Google Sheet, then returns to the loop for the next item.

---

## 2. Block-by-Block Analysis

### Block A — Scheduled start
**Overview:** Starts the workflow automatically based on a schedule rule.  
**Nodes involved:** Schedule Trigger

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` — entry point, runs on interval.
- **Key configuration choices:** `rule.interval` is present but effectively unspecified (`[{}]`), meaning you must set an actual schedule in the UI (e.g., every day at 09:00).
- **Connections:**  
  - **Output →** Get many folders
- **Edge cases / failures:**
  - Misconfigured schedule rule may result in no executions.
  - Timezone differences between expectation and n8n instance settings.

---

### Block B — ClickUp data collection & looping
**Overview:** Collects folder data from ClickUp, then iterates to fetch tasks and process them per-item.  
**Nodes involved:** Get many folders, Loop Over Items, Get many tasks

#### Node: Get many folders
- **Type / role:** `ClickUp` (resource: folder, operation: getAll) — lists folders in a Space.
- **Key configuration choices:**
  - **Team:** `YOUR_TEAM_ID`
  - **Space:** `YOUR_SPACE_ID`
  - **Limit:** 100
  - Filters empty (no filtering)
- **Credentials:** ClickUp API credential required.
- **Connections:**  
  - **Input ←** Schedule Trigger  
  - **Output →** Loop Over Items
- **Edge cases / failures:**
  - Auth errors / revoked token.
  - Wrong Team/Space IDs yield empty results.
  - Pagination beyond limit not handled here (limit=100).

#### Node: Loop Over Items
- **Type / role:** `Split in Batches` — controls iteration across items and provides the “continue loop” mechanism.
- **Key configuration choices:** No batch size explicitly set (defaults apply). Used as a loop controller with:
  - **Output 1 (main index 0):** Start of batch (unused here)
  - **Output 2 (main index 1):** “Continue” path → Get many tasks
- **Connections:**
  - **Input ←** Get many folders
  - **Output (continue) →** Get many tasks
  - **Also receives loop-back input ←** Append or update row in sheet1 (to continue next item)
- **Edge cases / failures:**
  - If upstream produces 0 items, loop produces nothing.
  - If batch settings are not tuned, very large datasets can cause long runtimes.

#### Node: Get many tasks
- **Type / role:** `ClickUp` (operation: getAll tasks) — retrieves tasks for analysis.
- **Key configuration choices:**
  - Uses expression for List ID: `{{ $json.lists[0].id }}`
  - `folderless: true`
  - Team/Space set to the same placeholders as above
  - Filters empty (no status/date filtering)
- **Connections:**
  - **Input ←** Loop Over Items (continue output)
  - **Output →** Message a model1
- **Critical dependency note (likely issue):**
  - This node expects the incoming item (from “Get many folders”) to contain `lists[0].id`. Folder objects do not always include embedded lists; many ClickUp endpoints require separate “get lists in folder/space” calls.
  - With `folderless: true`, providing a list ID may be contradictory/ignored depending on node implementation.
- **Edge cases / failures:**
  - Expression failure if `lists` is undefined/empty.
  - Large task volumes can exceed API limits or cause long executions.
  - If tasks contain missing fields (e.g., `priority` null), downstream prompt/expressions may degrade.

---

### Block C — AI task summary generation
**Overview:** For each task, calls an OpenAI model to produce a strict, plain-text structured summary and basic metrics.  
**Nodes involved:** Message a model1

#### Node: Message a model1
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI chat/model call.
- **Key configuration choices:**
  - **Model:** `gpt-4.1-mini`
  - Prompt is a single “content” message containing:
    - ClickUp task fields interpolated from `$json` (id, name, description, status, priority, assignees, creator, due_date, timestamps, url, project/list/folder/space).
    - Inline “Metrics” calculations using n8n expressions (e.g., overdue check uses `Number($json.due_date) < Date.now()`).
    - Output formatting requirement (“Output plain text ONLY, in this format…”).
- **Credentials:** OpenAI API credential required.
- **Connections:**
  - **Input ←** Get many tasks
  - **Output →** Code in JavaScript
- **Edge cases / failures:**
  - Model may not follow the requested exact format → parser may fail.
  - Missing fields (e.g., `project.name`, `folder.name`) can break expressions if those objects are undefined.
  - Token limits if descriptions are large.
  - API timeouts/rate limits.

---

### Block D — Parse AI output + compute KPI score/rating
**Overview:** Converts the AI plain-text response into structured fields and recalculates KPI score/rating deterministically for consistency.  
**Nodes involved:** Code in JavaScript

#### Node: Code in JavaScript
- **Type / role:** `Code` — transforms incoming items, parses text, computes KPIs.
- **Key configuration choices (interpreted):**
  - Reads all incoming items: `const items = $input.all();`
  - Extracts AI text at: `item.json.output?.[0]?.content?.[0]?.text`
    - This path is specific to this OpenAI node version/output shape; it may differ across n8n versions.
  - Parses by splitting lines, tracking sections (`overview`, `metrics`, `insight`).
  - Recomputes:
    - **KPI score** based on:
      - status: done=100, in progress=60, else=30
      - priority: high +10, low -10
      - overdue: -20 if overdue > 0
      - clamps 0–100
    - **Rating** mapping:
      - ≥80 Excellent, ≥60 Needs Improvement, ≥40 Average, else Poor
  - If AI “Insight” missing, auto-generates a fallback insight.
- **Connections:**
  - **Input ←** Message a model1
  - **Output →** Append or update row in sheet1
- **Edge cases / failures:**
  - If the AI output deviates from expected labels (`Task ID:`, `Metrics:` etc.), fields stay blank and KPI may compute with defaults.
  - If the OpenAI node output shape changes, `content` extraction returns null → item skipped entirely.
  - Priority parsing assumes textual `high/low`; ClickUp priority is often an object (and AI may print nonstandard strings).

---

### Block E — Write KPI row to Google Sheets + loop continuation
**Overview:** Writes one row per parsed task into Google Sheets using “append or update”, then signals the loop controller to proceed to the next item.  
**Nodes involved:** Append or update row in sheet1

#### Node: Append or update row in sheet1
- **Type / role:** `Google Sheets` — persists KPI fields to a spreadsheet.
- **Key configuration choices:**
  - **Operation:** Append or update
  - **Document:** `YOUR_GOOGLE_SHEET_ID`
  - **Sheet (gid):** `YOUR_SHEET_GID` (displayed cached name “Employee summary”)
  - **Mapping mode:** defineBelow
  - **Matching columns:** `Employee Name`
    - Important: this means all tasks by the same “Employee Name” will update the same row rather than create per-task rows.
  - **Column mappings (examples):**
    - Employee Name ← `$json.overview.creator`
    - Task Name ← `$json.name`
    - KPI Score ← `$json.metrics.kpiScore`
    - Rating ← `$json.metrics.rating`
    - Task Link ← `$json.taskLink`
- **Credentials:** Google Sheets OAuth2 credential required.
- **Connections:**
  - **Input ←** Code in JavaScript
  - **Output →** Loop Over Items (to continue looping)
- **Edge cases / failures:**
  - If the sheet headers don’t exactly match configured column IDs, writes may fail or map incorrectly.
  - Matching on **Employee Name** will overwrite/merge many tasks into one row per employee (likely unintended for “task-level performance data”).
  - Permissions / missing OAuth scopes / expired refresh token.
  - If “Employee Name” is blank, matching behavior can be inconsistent (duplicates or update failures).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled entry point | — | Get many folders | ClickUp Task Performance → Employee KPI Sheet (How it works / setup / customization) |
| Get many folders | n8n-nodes-base.clickUp | Collect folders from ClickUp Space | Schedule Trigger | Loop Over Items | Step 1: ClickUp Data Collection — Fetches folders and tasks, loops tasks individually. |
| Loop Over Items | n8n-nodes-base.splitInBatches | Loop controller / batching | Get many folders; Append or update row in sheet1 (loop-back) | Get many tasks (continue output) | Step 1: ClickUp Data Collection — Fetches folders and tasks, loops tasks individually. |
| Get many tasks | n8n-nodes-base.clickUp | Fetch tasks for analysis | Loop Over Items | Message a model1 | Step 1: ClickUp Data Collection — Fetches folders and tasks, loops tasks individually. |
| Message a model1 | @n8n/n8n-nodes-langchain.openAi | AI summarization + initial metrics rendering | Get many tasks | Code in JavaScript | Step 2: Task Analysis & KPI Processing — AI summaries + JS recalculation. |
| Code in JavaScript | n8n-nodes-base.code | Parse AI text → structured JSON; recompute KPI/rating | Message a model1 | Append or update row in sheet1 | Step 2: Task Analysis & KPI Processing — AI summaries + JS recalculation. |
| Append or update row in sheet1 | n8n-nodes-base.googleSheets | Persist KPI fields to Google Sheets; triggers next loop | Code in JavaScript | Loop Over Items | Step 3: KPI Reporting to Sheet — Writes KPI data into Google Sheets. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / context | — | — | ClickUp Task Performance → Employee KPI Sheet (How it works / setup / customization) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation for Step 1 | — | — | Step 1: ClickUp Data Collection — Fetches folders and tasks, loops tasks individually. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation for Step 2 | — | — | Step 2: Task Analysis & KPI Processing — AI summaries + JS recalculation. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation for Step 3 | — | — | Step 3: KPI Reporting to Sheet — Writes KPI data into Google Sheets. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create workflow**
   - Name it: **Employee Performance AI Dashboard**
   - Ensure workflow execution order setting is default (this workflow uses `executionOrder: v1`).

2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Configure the interval (e.g., daily/weekly). The provided JSON does not specify a real interval; set it explicitly.

3. **Add ClickUp “Get many folders”**
   - Node: **ClickUp**
   - Resource: **Folder**
   - Operation: **Get All**
   - Set:
     - Team: your ClickUp **Team ID**
     - Space: your ClickUp **Space ID**
     - Limit: 100
   - Credentials: connect **ClickUp API**.
   - Connect: **Schedule Trigger → Get many folders**

4. **Add “Loop Over Items”**
   - Node: **Split In Batches** (rename to “Loop Over Items”)
   - Leave default options (or set a batch size if needed).
   - Connect: **Get many folders → Loop Over Items**
   - Use the **“Continue” output (second output)** later to fetch tasks.

5. **Add ClickUp “Get many tasks”**
   - Node: **ClickUp**
   - Operation: **Get All** (tasks)
   - Set Team + Space to the same IDs.
   - Set **List** to expression: `{{ $json.lists[0].id }}`
   - Enable **folderless: true**
   - Credentials: same ClickUp credential.
   - Connect: **Loop Over Items (continue output) → Get many tasks**
   - Note: if your folder payload does not include `lists[0].id`, add an extra ClickUp node to fetch lists per folder and feed the list IDs into this task query.

6. **Add OpenAI “Message a model”**
   - Node: **OpenAI (LangChain) – Message a model**
   - Model: **gpt-4.1-mini**
   - Paste the provided prompt content (ensure it remains “plain text only” output spec).
   - Credentials: connect **OpenAI API**.
   - Connect: **Get many tasks → Message a model**

7. **Add Code node (JavaScript)**
   - Node: **Code**
   - Language: JavaScript
   - Paste the provided parsing + KPI calculation code.
   - Connect: **Message a model → Code**

8. **Add Google Sheets “Append or update row”**
   - Node: **Google Sheets**
   - Operation: **Append or update**
   - Document: select your target spreadsheet (Google Sheet ID)
   - Sheet: select your target tab (gid)
   - Mapping:
     - Define columns as in the workflow (Employee Name, Task Name, Project, Folder, Status, Priority, Completed, In Progress, Pending, Overdue, Productivity, KPI Score, Rating, Summary, Task Link)
   - Matching columns: **Employee Name**
     - If you want **one row per task**, change matching to a unique key (e.g., “Task ID”) and store it in the sheet.
   - Credentials: connect **Google Sheets OAuth2**.
   - Connect: **Code → Google Sheets**

9. **Close the loop**
   - Connect: **Google Sheets → Loop Over Items**
   - This enables the “continue processing” behavior across items.

10. **(Optional) Add sticky notes**
   - Add three sticky notes describing Steps 1–3 and one global note; content can match the provided notes.

11. **Activate**
   - Turn workflow **Active** once credentials and IDs are set and the Google Sheet headers match the mapping.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ClickUp Task Performance → Employee KPI Sheet: runs on schedule, pulls tasks, AI generates structured summary, JS normalizes/recalculates KPI, writes to Google Sheets. Setup: connect ClickUp + IDs, standardize statuses/priorities, connect Sheets, match columns, configure schedule. Customization: adjust KPI logic in JS, change Sheets matching logic, extend AI prompt. | Sticky note “ClickUp Task Performance → Employee KPI Sheet” |
| Step 1: ClickUp Data Collection — Fetches folders and tasks, loops tasks individually. | Sticky note “Step 1” |
| Step 2: Task Analysis & KPI Processing — AI structured summaries, JS parses and recalculates KPI scores/ratings. | Sticky note “Step 2” |
| Step 3: KPI Reporting to Sheet — Writes task-level data into Google Sheets for live KPI/productivity reporting. | Sticky note “Step 3” |