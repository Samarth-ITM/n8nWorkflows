Monitor daily HR risks and standup summaries with Monday.com and GPT-4o-mini

https://n8nworkflows.xyz/workflows/monitor-daily-hr-risks-and-standup-summaries-with-monday-com-and-gpt-4o-mini-12269


# Monitor daily HR risks and standup summaries with Monday.com and GPT-4o-mini

## 1. Workflow Overview

**Purpose:**  
This workflow runs every morning to monitor HR tasks stored in a Monday.com board, detect operational risks (overdue, stuck, unassigned tasks), and automatically email either:
- an **escalation risk report** (if risks exist), or
- a **normal daily standup summary** (if everything is on track).

**Target use cases:**
- Daily HR operations pulse-check (task progress, blockers, ownership gaps)
- Leadership-ready escalation when HR delivery is at risk
- Automated standup-style summary without manual reporting

### 1.1 Trigger & Data Retrieval
Runs on a schedule and fetches all HR items from a Monday.com board/group.

### 1.2 Task Filtering & Normalization
Removes completed tasks (“Done”), and reshapes Monday.com items into a simplified task list.

### 1.3 Risk Detection & Metrics
Computes counts and detailed lists for:
- overdue tasks (more than 2 days late),
- “Stuck” tasks,
- unassigned tasks (no owner).

### 1.4 Decision + AI Content Generation + Email Delivery
If any risk exists → generate AI risk report and email escalation.  
Else → generate AI daily standup summary and email it.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Data Source
**Overview:** Runs daily at a fixed time and loads HR tasks from Monday.com for analysis.  
**Nodes involved:** `Daily Trigger`, `Get HR Tasks`

#### Node: Daily Trigger
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — entry point.
- **Configuration (interpreted):**
  - Runs every day at **08:30** (server / n8n instance timezone).
- **Inputs / outputs:**
  - **Input:** none (trigger)
  - **Output:** to `Get HR Tasks`
- **Version notes:** typeVersion **1.3**.
- **Potential failures / edge cases:**
  - Timezone mismatch (instance timezone vs expected local HR timezone).
  - Workflow is currently **inactive** (`active: false`), so it will not run until activated.

#### Node: Get HR Tasks
- **Type / role:** Monday.com node (`n8n-nodes-base.mondayCom`) — reads board items.
- **Configuration (interpreted):**
  - **Resource:** Board Item
  - **Operation:** Get All
  - **Board ID:** `5025720609`
  - **Group ID:** `new_group29179`
  - **Return all:** enabled
  - Uses Monday.com API credential: **“Monday.com account 3”**
- **Inputs / outputs:**
  - **Input:** `Daily Trigger`
  - **Output:** list of Monday.com items → `Filter Active Tasks`
- **Version notes:** typeVersion **1**.
- **Potential failures / edge cases:**
  - Invalid board/group IDs, or group renamed/removed.
  - Monday API permission errors (token scope/user access).
  - Large boards: returning all items can be slow or hit API rate limits.
  - Column ordering is assumed later (owner uses `column_values[1]`), which can break if board columns change.

**Sticky note coverage (for this block):**
- “### Trigger and Data Source  
  This workflow runs daily and fetches active HR tasks from Monday.com.”

---

### Block 2 — Task Filtering & Preparation
**Overview:** Filters out completed tasks and converts the Monday.com payload into a compact array of task objects used downstream.  
**Nodes involved:** `Filter Active Tasks`, `Transform Tasks`

#### Node: Filter Active Tasks
- **Type / role:** Code node (`n8n-nodes-base.code`) — removes tasks with status “Done”.
- **Configuration (interpreted):**
  - JavaScript filters incoming items:
    - Reads `status` from `item.json.column_values` where `c.type === 'status'`
    - Keeps items where status exists and is **not** `"Done"`
- **Key logic (variables/expressions):**
  - `item.json.column_values.find(c => c.type === 'status')?.text`
- **Inputs / outputs:**
  - **Input:** `Get HR Tasks`
  - **Output:** filtered items → `Transform Tasks`
- **Version notes:** typeVersion **2** (modern Code node behavior).
- **Potential failures / edge cases:**
  - If Monday response structure changes or `column_values` missing → could throw if not an array (current code assumes it is).
  - If “Done” status label is different (e.g., “Completed”) → tasks won’t be filtered correctly.
  - If multiple status columns exist, `find(c.type==='status')` may select an unexpected one.

#### Node: Transform Tasks
- **Type / role:** Code node (`n8n-nodes-base.code`) — normalizes items into `{ tasks: [...] }`.
- **Configuration (interpreted):**
  - Outputs a **single item** with a `tasks` array derived from all input items.
  - Each task object includes:
    - `name`: item name
    - `owner`: `column_values[1]?.text` (second column in array)
    - `status`: first column with `type === 'status'` → `.text`
    - `due_date`: first column with `type === 'date'` → `.text`
- **Key logic (variables/expressions):**
  - Owner extraction is **positional**: `i.json.column_values[1]?.text || null`
  - Status/date extraction is by type: `find(c => c.type === 'status'/'date')?.text`
- **Inputs / outputs:**
  - **Input:** `Filter Active Tasks`
  - **Output:** one aggregated item → `Build HR Metrics`
- **Version notes:** typeVersion **2**.
- **Potential failures / edge cases:**
  - **Column position dependency** for owner is brittle; if board columns reorder, owner may become wrong/null.
  - Date parsing later depends on the format in `due_date` text; Monday date text may be locale-formatted.

**Sticky note coverage (for this block):**
- “### Task filtering and preparation  
  Filters out completed tasks and prepares HR task data for analysis.”

---

### Block 3 — Risk Detection & Metrics
**Overview:** Computes risk metrics and detailed risk lists (overdue >2 days, stuck, unassigned). Produces a single structured object used for branching and AI prompts.  
**Nodes involved:** `Build HR Metrics`, `Any HR Risks?`

#### Node: Build HR Metrics
- **Type / role:** Code node (`n8n-nodes-base.code`) — calculates risk counts and arrays.
- **Configuration (interpreted):**
  - Reads `items[0].json.tasks` from previous node (expects exactly one aggregated item).
  - Builds:
    - `unassigned`: tasks where `owner` is null/empty
    - `stuck`: tasks where `status === 'Stuck'`
    - `overdue`: tasks where due date exists and is **more than 2 days in the past**
  - Adds `days_overdue` to overdue entries.
- **Key logic (variables/expressions):**
  - `const days = Math.floor((today - due) / 86400000); if (days > 2) ...`
  - Output shape:
    - `json.tasks`
    - `json.metrics.{ overdue_count, stuck_count, unassigned_count, overdue, stuck, unassigned }`
- **Inputs / outputs:**
  - **Input:** `Transform Tasks`
  - **Output:** → `Any HR Risks?`
- **Version notes:** typeVersion **2**.
- **Potential failures / edge cases:**
  - **Date parsing risk:** `new Date(t.due_date)` may return Invalid Date if Monday’s `.text` is not ISO-compatible (common in non-US locales).
  - Timezone effects: “days overdue” can be off by 1 depending on time of day/timezone.
  - Assumes `items[0]` exists; if no tasks returned at all, it will throw.

#### Node: Any HR Risks?
- **Type / role:** IF node (`n8n-nodes-base.if`) — branches based on combined risk count.
- **Configuration (interpreted):**
  - Condition: `overdue_count + stuck_count + unassigned_count > 0`
  - Uses strict type validation.
- **Key expressions:**
  - `{{ $json.metrics.overdue_count + $json.metrics.stuck_count + $json.metrics.unassigned_count }}`
- **Inputs / outputs:**
  - **Input:** `Build HR Metrics`
  - **True output (risk exists):** → `AI Risk Report`
  - **False output (no risk):** → `AI Daily Summary`
- **Version notes:** typeVersion **2**.
- **Potential failures / edge cases:**
  - If `metrics` missing (due to upstream error/shape change), expression evaluation fails.
  - If counts are strings (unlikely here) strict validation could fail comparisons.

**Sticky note coverage (for this block):**
- “### Risk Detection and Metrics  
  Checks for overdue, stuck, and unassigned tasks and builds HR risk metrics.”

---

### Block 4 — AI Analysis + Email Delivery
**Overview:** Uses OpenAI (GPT-4o-mini) to generate either a risk escalation report or a daily standup summary, then emails the resulting text via Gmail.  
**Nodes involved:** `AI Risk Report`, `Escalation Email`, `AI Daily Summary`, `Daily HR Summary Email`

#### Node: AI Risk Report
- **Type / role:** OpenAI (LangChain) node (`@n8n/n8n-nodes-langchain.openAi`) — generates escalation report text.
- **Configuration (interpreted):**
  - Model: **gpt-4o-mini** (selected via model list resolver)
  - System message: “You are an HR operations analyst.”
  - User prompt requests “HR Risk Report” with:
    1) Critical blockers  
    2) Overdue tasks with days overdue  
    3) Unassigned tasks  
    4) Recommended HR actions  
  - Injects data: `JSON.stringify($json.metrics, null, 2)`
  - Credentials: **“OpenAi account 3”**
- **Inputs / outputs:**
  - **Input:** True branch from `Any HR Risks?`
  - **Output:** to `Escalation Email`
- **Version notes:** typeVersion **2.1**.
- **Potential failures / edge cases:**
  - OpenAI auth/quota/model access errors.
  - Prompt may exceed token limits if metrics lists are large (many tasks).
  - Output mapping assumes a specific response structure used later (`$json.output[0].content[0].text`).

#### Node: Escalation Email
- **Type / role:** Gmail node (`n8n-nodes-base.gmail`) — sends escalation email.
- **Configuration (interpreted):**
  - To: `user@example.com`
  - Subject: `🚨 HR Risks Detected – Immediate Action Required`
  - Body (text): `{{ $json.output[0].content[0].text }}`
  - Credentials: **“Gmail credentials”** (OAuth2)
- **Inputs / outputs:**
  - **Input:** `AI Risk Report`
  - **Output:** none (terminal)
- **Version notes:** typeVersion **2.2**.
- **Potential failures / edge cases:**
  - Gmail OAuth token expired/insufficient scopes.
  - If OpenAI node output schema differs, body expression may be undefined and send blank email or fail.
  - Deliverability/recipient restrictions in Google Workspace environments.

#### Node: AI Daily Summary
- **Type / role:** OpenAI (LangChain) node (`@n8n/n8n-nodes-langchain.openAi`) — generates normal daily summary.
- **Configuration (interpreted):**
  - Model: `gpt-4o-mini`
  - System message: “You are an HR operations assistant.”
  - Prompt: “Create a concise HR daily standup summary grouped by status.”
  - Injects tasks: `JSON.stringify($json.tasks, null, 2)`
  - Credentials: **“OpenAi account 3”**
- **Inputs / outputs:**
  - **Input:** False branch from `Any HR Risks?`
  - **Output:** to `Daily HR Summary Email`
- **Version notes:** typeVersion **2.1**.
- **Potential failures / edge cases:**
  - Token/size issues if the tasks array is very large.
  - Same response-structure assumption as above for the email body mapping.

#### Node: Daily HR Summary Email
- **Type / role:** Gmail node (`n8n-nodes-base.gmail`) — sends daily summary email.
- **Configuration (interpreted):**
  - To: `user@example.com`
  - Subject: `Daily HR Action Summary`
  - Body (text): `{{ $json.output[0].content[0].text }}`
  - Credentials: **“Gmail credentials”**
- **Inputs / outputs:**
  - **Input:** `AI Daily Summary`
  - **Output:** none (terminal)
- **Version notes:** typeVersion **2.2**.
- **Potential failures / edge cases:**
  - Same Gmail OAuth and expression-shape risks as escalation email.

**Sticky note coverage (for this block):**
- “### AI analysis and decision  
  Uses AI to decide whether there are HR risks or a normal daily update. And  
  sends escalation emails if risks exist or a daily summary if everything is on track.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Trigger | Schedule Trigger | Daily workflow entry point at 08:30 | — | Get HR Tasks | ### Trigger and Data Source<br>This workflow runs daily and fetches active HR tasks from Monday.com. |
| Get HR Tasks | Monday.com | Fetch all items from HR board/group | Daily Trigger | Filter Active Tasks | ### Trigger and Data Source<br>This workflow runs daily and fetches active HR tasks from Monday.com. |
| Filter Active Tasks | Code | Remove tasks with status “Done” | Get HR Tasks | Transform Tasks | ### Task filtering and preparation<br>Filters out completed tasks and prepares HR task data for analysis. |
| Transform Tasks | Code | Normalize/aggregate items into `tasks[]` | Filter Active Tasks | Build HR Metrics | ### Task filtering and preparation<br>Filters out completed tasks and prepares HR task data for analysis. |
| Build HR Metrics | Code | Compute overdue/stuck/unassigned metrics | Transform Tasks | Any HR Risks? | ### Risk Detection and Metrics<br>Checks for overdue, stuck, and unassigned tasks and builds HR risk metrics. |
| Any HR Risks? | IF | Branch based on total risk count > 0 | Build HR Metrics | AI Risk Report (true), AI Daily Summary (false) | ### Risk Detection and Metrics<br>Checks for overdue, stuck, and unassigned tasks and builds HR risk metrics.<br>### AI analysis and decision<br>Uses AI to decide whether there are HR risks or a normal daily update. And sends escalation emails if risks exist or a daily summary if everything is on track. |
| AI Risk Report | OpenAI (LangChain) | Generate escalation-focused HR risk report | Any HR Risks? (true) | Escalation Email | ### AI analysis and decision<br>Uses AI to decide whether there are HR risks or a normal daily update. And sends escalation emails if risks exist or a daily summary if everything is on track. |
| Escalation Email | Gmail | Email escalation report to stakeholders | AI Risk Report | — | ### AI analysis and decision<br>Uses AI to decide whether there are HR risks or a normal daily update. And sends escalation emails if risks exist or a daily summary if everything is on track. |
| AI Daily Summary | OpenAI (LangChain) | Generate normal daily standup summary | Any HR Risks? (false) | Daily HR Summary Email | ### AI analysis and decision<br>Uses AI to decide whether there are HR risks or a normal daily update. And sends escalation emails if risks exist or a daily summary if everything is on track. |
| Daily HR Summary Email | Gmail | Email daily summary to stakeholders | AI Daily Summary | — | ### AI analysis and decision<br>Uses AI to decide whether there are HR risks or a normal daily update. And sends escalation emails if risks exist or a daily summary if everything is on track. |
| Workflow overview | Sticky Note | Documentation / overview | — | — | ## Workflow Overview<br>…(see note in section 5) |
| Sticky Note | Sticky Note | Visual grouping label | — | — | ### Trigger and Data Source<br>This workflow runs daily and fetches active HR tasks from Monday.com. |
| Sticky Note1 | Sticky Note | Visual grouping label | — | — | ### Task filtering and preparation<br>Filters out completed tasks and prepares HR task data for analysis. |
| Sticky Note2 | Sticky Note | Visual grouping label | — | — | ### Risk Detection and Metrics<br>Checks for overdue, stuck, and unassigned tasks and builds HR risk metrics. |
| Sticky Note3 | Sticky Note | Visual grouping label | — | — | ### AI analysis and decision<br>Uses AI to decide whether there are HR risks or a normal daily update. And sends escalation emails if risks exist or a daily summary if everything is on track. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“Monitor daily HR actions, blockers, and risks using Monday.com and AI”** (or your preferred name).
   - Ensure workflow settings use **Execution Order: v1** (Workflow settings → Execution).

2. **Add trigger**
   - Add node: **Schedule Trigger**
   - Configure: run daily at **08:30**
   - Connect: `Daily Trigger` → `Get HR Tasks`

3. **Add Monday.com retrieval**
   - Add node: **Monday.com**
   - Resource: **Board Item**
   - Operation: **Get All**
   - Set **Board ID** to your HR board (example in workflow: `5025720609`)
   - Set **Group ID** to the target group (example: `new_group29179`)
   - Enable **Return All**
   - Credentials:
     - Create/select **Monday.com API** credentials (token/user with board access)
   - Connect to next: `Get HR Tasks` → `Filter Active Tasks`

4. **Filter out completed tasks**
   - Add node: **Code**
   - Paste logic equivalent to:
     - Find status column (`type === 'status'`) and exclude `"Done"`
   - Connect: `Filter Active Tasks` → `Transform Tasks`

5. **Transform into normalized `tasks[]`**
   - Add node: **Code**
   - Configure it to output **one item** with `json.tasks = items.map(...)`
   - Map at minimum:
     - `name` from item name
     - `owner` from the appropriate owner/person column
     - `status` from status column text
     - `due_date` from date column text
   - Important: avoid positional mapping if possible; prefer selecting the correct column by `id` or `type` to reduce breakage when board columns move.
   - Connect: `Transform Tasks` → `Build HR Metrics`

6. **Build metrics (overdue/stuck/unassigned)**
   - Add node: **Code**
   - Compute:
     - `unassigned` where owner is empty
     - `stuck` where status equals `"Stuck"`
     - `overdue` where due date is more than **2 days** behind “today”
   - Output:
     - `tasks` (pass-through)
     - `metrics` object with counts and lists
   - Connect: `Build HR Metrics` → `Any HR Risks?`

7. **Add risk decision branch**
   - Add node: **IF**
   - Condition: Number comparison  
     - Left value expression: `metrics.overdue_count + metrics.stuck_count + metrics.unassigned_count`
     - Operation: `>`  
     - Right value: `0`
   - Connect:
     - **True** → `AI Risk Report`
     - **False** → `AI Daily Summary`

8. **Add OpenAI node for risk report**
   - Add node: **OpenAI (LangChain)**
   - Model: **gpt-4o-mini**
   - Messages:
     - System: “You are an HR operations analyst.”
     - User: request a risk report and inject `metrics` as JSON
   - Credentials:
     - Create/select **OpenAI API** credentials
   - Connect: `AI Risk Report` → `Escalation Email`

9. **Add Gmail node for escalation email**
   - Add node: **Gmail → Send**
   - To: escalation recipient(s)
   - Subject: “HR Risks Detected – Immediate Action Required”
   - Email type: **text**
   - Body expression:
     - Map from the OpenAI node output (match your node’s actual output structure). In this workflow it is:
       - `{{$json.output[0].content[0].text}}`
   - Credentials:
     - Create/select **Gmail OAuth2** credentials (ensure send permissions)
   - This is a terminal node on the True branch.

10. **Add OpenAI node for daily summary**
   - Add node: **OpenAI (LangChain)**
   - Model: **gpt-4o-mini**
   - Messages:
     - System: “You are an HR operations assistant.”
     - User: “Create a concise HR daily standup summary grouped by status.” + inject `tasks` as JSON
   - Connect: `AI Daily Summary` → `Daily HR Summary Email`

11. **Add Gmail node for daily summary**
   - Add node: **Gmail → Send**
   - To: daily recipient(s)
   - Subject: “Daily HR Action Summary”
   - Email type: **text**
   - Body expression: same mapping approach as escalation branch (in this workflow: `{{$json.output[0].content[0].text}}`)

12. **Validate & activate**
   - Run once manually to validate:
     - Monday.com returns expected column structures
     - dates parse correctly for overdue logic
     - OpenAI output mapping resolves correctly for Gmail body
   - Activate workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## Workflow Overview … (How it works / Setup steps)” including: connect Monday.com, ensure status values include “Done” and “Stuck”, connect OpenAI, configure recipients, adjust schedule timing. | Sticky note: “Workflow overview” |
| Disclaimer: *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…* | Provided by user (workflow context) |