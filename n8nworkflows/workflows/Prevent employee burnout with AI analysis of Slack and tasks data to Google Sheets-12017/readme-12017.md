Prevent employee burnout with AI analysis of Slack and tasks data to Google Sheets

https://n8nworkflows.xyz/workflows/prevent-employee-burnout-with-ai-analysis-of-slack-and-tasks-data-to-google-sheets-12017


# Prevent employee burnout with AI analysis of Slack and tasks data to Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow monitors potential employee burnout by combining **Slack communication tone**, **attendance patterns**, and **task completion behavior**, then logs anonymized results to **Google Sheets** for HR trend tracking. If an employee’s stress is predicted as **High**, it privately sends a supportive Slack DM and logs the intervention in **Postgres**.

**Target use cases:**
- Early detection of stress signals across teams
- HR trend dashboards without exposing identities (hashed employee ID)
- Automated, privacy-conscious employee outreach when risk is high

### 1.1 Schedule & Time Window
Runs on a schedule and computes the previous day’s start/end timestamps (ISO + Unix).

### 1.2 Data Aggregation per Employee (Loop)
Fetches active employees, then processes them in batches: Slack messages → tone analysis → attendance metrics → task metrics → feature consolidation.

### 1.3 AI Stress Prediction + Anonymization
Predicts stress score/level via OpenAI, hashes employee ID, and prepares data for two paths: HR dashboard and direct intervention.

### 1.4 HR Logging + Intervention
Appends anonymized results to Google Sheets. If stress is “high”, sends Slack DM and logs counseling proposal in Postgres.

---

## 2. Block-by-Block Analysis

### Block 1 — Schedule & Timeframe Computation

**Overview:**  
Triggers the workflow and calculates the analysis range for the previous day (00:00:00–23:59:59.999). The computed timestamps are reused in API requests for attendance and tasks.

**Nodes involved:**
- Run Daily at 2 AM
- Calculate Analysis Period

#### Node: **Run Daily at 2 AM**
- **Type / role:** Schedule Trigger; starts the workflow automatically.
- **Configuration (interpreted):** Configured as an interval trigger “every 2 hours” (despite the node name suggesting 2 AM daily).
- **Inputs/outputs:** No input; output feeds **Calculate Analysis Period**.
- **Version notes:** `scheduleTrigger` v1.2.
- **Edge cases / failures:**
  - Misconfiguration: The rule is not “daily at 02:00”; it is “every 2 hours”. If you truly want 2 AM daily, switch to a cron/daily rule.

#### Node: **Calculate Analysis Period**
- **Type / role:** Code node; derives yesterday’s time window.
- **Configuration choices:**
  - Creates `analysisStart` and `analysisEnd` in ISO format.
  - Also provides Unix timestamps (`analysisStartUnix`, `analysisEndUnix`) for APIs requiring epoch seconds.
- **Key variables produced:**
  - `analysisStart`, `analysisEnd` (ISO)
  - `analysisStartUnix`, `analysisEndUnix` (seconds)
- **Inputs/outputs:** Takes trigger item; outputs a single item to **Fetch Employee List**.
- **Version notes:** Code node v2.
- **Edge cases / failures:**
  - Timezone behavior depends on server runtime timezone; ISO conversion can shift day boundaries if you intended a specific locale. Consider using explicit timezone handling if needed.

**Sticky notes in this block:**
- “Workflow Overview” (global)
- “Note: Schedule” (schedule/timeframe guidance)

---

### Block 2 — Employee Retrieval & Batch Loop

**Overview:**  
Fetches active employees from an HR endpoint and processes them in batches to control rate limits and execution load.

**Nodes involved:**
- Fetch Employee List
- Start Employee Loop

#### Node: **Fetch Employee List**
- **Type / role:** HTTP Request; retrieves employees to analyze.
- **Configuration choices:**
  - GET `https://hr.example.com/api/employees`
  - Query parameter: `status=active`
  - Placeholder endpoint (meant to be replaced by real HR system).
- **Inputs/outputs:** Receives analysis period item; outputs employee list to **Start Employee Loop**.
- **Version notes:** HTTP Request v4.2.
- **Edge cases / failures:**
  - Auth not configured (common with HR APIs): will return 401/403 unless headers/credentials added.
  - Response shape mismatch: downstream expects fields like `employee_id`, `slack_user_id`, `department`. If API returns different keys, mapping must be added (e.g., Set node).

#### Node: **Start Employee Loop**
- **Type / role:** Split In Batches; iterates employees with batching.
- **Configuration choices:** `batchSize=10` to reduce API throttling risk.
- **Inputs/outputs:**
  - Input: employee list from **Fetch Employee List**
  - Output: each batch item to **Fetch Slack Messages**
  - Loopback: fed again from **Check for High Stress** (false branch) and from **Log Counseling Proposal** to continue next employee.
- **Version notes:** SplitInBatches v3.
- **Edge cases / failures:**
  - If upstream returns a single object instead of an array of employees, batching will not behave as expected.
  - Infinite/incorrect looping can happen if the “continue” wiring is wrong; here it is wired back appropriately.

**Sticky notes in this block:**
- “Note: Period Calculation” (covers aggregation/analysis stages conceptually)

---

### Block 3 — Slack Collection & Tone Analysis

**Overview:**  
Pulls Slack messages for each employee, compiles recent text into a single summary string, and asks OpenAI to assess tone (producing a stress-related signal used later).

**Nodes involved:**
- Fetch Slack Messages
- Generate Message Summary
- AI Tone Analysis
- Format Data for Privacy

#### Node: **Fetch Slack Messages**
- **Type / role:** Slack node; searches for messages.
- **Configuration choices:** Resource is `search`. (Query details are not shown in the JSON; likely missing/placeholder.)
- **Inputs/outputs:** Receives an employee item; outputs Slack search results to **Generate Message Summary**.
- **Version notes:** Slack v2.1.
- **Edge cases / failures:**
  - Missing search query: Slack “search” requires a query string; without it, this node may error or return nothing.
  - Permissions: Slack scopes like `search:read` are restricted; many workspaces disallow full search. A safer approach is reading user’s DMs/channels with explicit scopes and channel lists.
  - Rate limits (HTTP 429), especially at scale.

#### Node: **Generate Message Summary**
- **Type / role:** Code node; builds a combined text body for AI.
- **Configuration choices:**
  - Reads `messages` array from input: `const messages = $input.item.json.messages || []`
  - Takes up to 50 messages, concatenates `m.text` with newline separators
  - Outputs `messageSummary` and `messageCount`
- **Inputs/outputs:** From Slack → to **AI Tone Analysis**
- **Version notes:** Code v2.
- **Edge cases / failures:**
  - If Slack output structure differs (often Slack search returns nested results), `messages` may be undefined, resulting in empty summaries.
  - Large texts: concatenation might exceed model token limits later; consider truncation by characters/tokens.

#### Node: **AI Tone Analysis**
- **Type / role:** OpenAI node; produces tone/sentiment evaluation.
- **Configuration choices:**
  - `operation: message` (chat-style call)
  - Credentials: `openAiApi`
  - Prompt/messages content is not visible in JSON; downstream expects something like `tone_stress_score`.
- **Inputs/outputs:** From summary → to **Format Data for Privacy**
- **Version notes:** OpenAI node v1.
- **Edge cases / failures:**
  - Missing prompt mapping: if you don’t pass `messageSummary` into the prompt, results won’t reflect Slack content.
  - Output parsing: later nodes reference `$json.tone_stress_score` and `$json.response`; ensure you standardize where the model output is stored (e.g., JSON response content).
  - API errors: quota exceeded, timeouts, invalid model settings.

#### Node: **Format Data for Privacy**
- **Type / role:** Set node; intended to shape or remove sensitive fields before continuing.
- **Configuration choices:** No fields defined in JSON (currently a no-op).
- **Inputs/outputs:** Receives AI tone output; forwards to **Fetch Attendance Data**.
- **Version notes:** Set v3.2.
- **Edge cases / failures:**
  - Because it currently sets nothing, privacy shaping is not actually implemented here.
  - If the OpenAI node output overwrote prior fields, you may lose `employee_id` unless you merge carefully.

**Sticky notes in this block:**
- “Note: Period Calculation” (describes aggregation & AI analysis at a high level)

---

### Block 4 — Attendance & Task Metrics Enrichment

**Overview:**  
Fetches attendance logs and task data for the analysis period, then computes key metrics (overtime, lateness, completion rate, delays) per employee.

**Nodes involved:**
- Fetch Attendance Data
- Calculate Attendance Metrics
- Fetch Task Completion Rates
- Calculate Task Metrics
- Consolidate Features

#### Node: **Fetch Attendance Data**
- **Type / role:** HTTP Request; pulls attendance logs.
- **Configuration choices:**
  - GET `https://attendance.example.com/api/logs`
  - Query params:
    - `employee_id={{ $json.employee_id }}`
    - `from={{ $json.analysisStart }}`
    - `to={{ $json.analysisEnd }}`
  - Placeholder endpoint.
- **Inputs/outputs:** From privacy formatting → to **Calculate Attendance Metrics**
- **Version notes:** HTTP Request v4.2.
- **Edge cases / failures:**
  - Upstream must include `employee_id`, `analysisStart`, `analysisEnd` at this stage; otherwise expressions resolve to empty.
  - Response mapping: the next code node expects `attendance_logs` array; if API returns `data` or `logs`, you must map/rename.

#### Node: **Calculate Attendance Metrics**
- **Type / role:** Code node; aggregates attendance logs.
- **Configuration choices:**
  - Reads `attendance_logs` array
  - Sums `work_hours`, `overtime_hours`
  - Counts `late_arrival` and `absent`
  - Computes `avg_work_hours`
- **Outputs added:**
  - `avg_work_hours`, `total_overtime_hours`, `late_arrival_count`, `absent_count`
- **Inputs/outputs:** Attendance response → to **Fetch Task Completion Rates**
- **Version notes:** Code v2.
- **Edge cases / failures:**
  - Null/undefined numeric fields: handled with `|| 0` (good).
  - If `attendance_logs` not present, metrics default to 0, which may hide integration issues.

#### Node: **Fetch Task Completion Rates**
- **Type / role:** HTTP Request; pulls tasks for the employee and period.
- **Configuration choices:**
  - GET `https://tasks.example.com/api/tasks`
  - Query params:
    - `assignee_id={{ $json.employee_id }}`
    - `from={{ $json.analysisStart }}`
    - `to={{ $json.analysisEnd }}`
- **Inputs/outputs:** From attendance metrics → to **Calculate Task Metrics**
- **Version notes:** HTTP Request v4.2.
- **Edge cases / failures:**
  - Response mapping: next code node expects `tasks` array; update mapping if API differs.
  - Paging: many task APIs paginate; without pagination handling, metrics may be incomplete.

#### Node: **Calculate Task Metrics**
- **Type / role:** Code node; computes completion and delay metrics.
- **Configuration choices:**
  - Counts tasks with `status === 'completed'`
  - Overdue tasks: checks `task.overdue` boolean; sums `delay_days`
  - Computes:
    - `task_completion_rate = completedCount / tasks.length`
    - `avg_task_delay_days = totalDelayDays / overdueCount`
- **Inputs/outputs:** Task response → to **Consolidate Features**
- **Version notes:** Code v2.
- **Edge cases / failures:**
  - If statuses differ (e.g., `done`, `closed`), completion rate will be wrong.
  - If `overdue` is not a boolean but derived from dates, this logic needs adjustment.

#### Node: **Consolidate Features**
- **Type / role:** Code node; creates a clean feature object for final stress prediction.
- **Configuration choices:**
  - Produces a single object with:
    - Identity: `employee_id`, `slack_user_id`, `department`
    - Slack: `tone_stress_score` (default 0)
    - Attendance: `avg_work_hours`, `total_overtime_hours`, `late_arrival_count`, `absent_count`
    - Tasks: `task_completion_rate`, `overdue_task_count`
    - Period: `analysisStart`, `analysisEnd`
  - Note: `avg_task_delay_days` is calculated earlier but not included here (likely an omission).
- **Inputs/outputs:** From task metrics → to **AI Stress Level Prediction**
- **Version notes:** Code v2.
- **Edge cases / failures:**
  - If `tone_stress_score` was never set by AI Tone Analysis, it will stay 0 and weaken the model signal.
  - Missing `avg_task_delay_days` reduces feature completeness.

---

### Block 5 — AI Stress Prediction & Anonymization

**Overview:**  
Uses OpenAI to generate a structured stress assessment, then hashes employee ID and prepares anonymized payload for HR logging while keeping identifiers for the intervention branch.

**Nodes involved:**
- AI Stress Level Prediction
- Generate Anonymized Data

#### Node: **AI Stress Level Prediction**
- **Type / role:** OpenAI node; predicts overall stress score/level.
- **Configuration choices:**
  - `operation: message`
  - Uses OpenAI credentials
  - Downstream expects the model output stored in `$json.response` as JSON text (e.g., `{"stress_score":..., "stress_level":"high", ...}`).
- **Inputs/outputs:** From consolidated features → to **Generate Anonymized Data**
- **Version notes:** OpenAI v1.
- **Edge cases / failures:**
  - If the model returns non-JSON or extra text, `JSON.parse($json.response)` will fail downstream (though current code uses `JSON.parse(...||'{}')` but will still throw if it’s invalid JSON).
  - Normalization: IF node checks `stress_level` equals `"high"` lowercase; model might output `"High"`. This will cause missed interventions unless normalized.

#### Node: **Generate Anonymized Data**
- **Type / role:** Code node; hashes employee ID and parses AI response.
- **Configuration choices:**
  - Uses Node.js `crypto` SHA-256 hash of `employee_id` → `employee_hash`
  - Parses `stressData = JSON.parse($input.item.json.response || '{}')`
  - Outputs:
    - Anonymized fields for dashboard: `employee_hash`, `department`, `stress_score`, `stress_level`, `main_risk_factors`, dates/period
    - Keeps `employee_id` and `slack_user_id` for private DM branch
  - Sets `analysis_date` to *today* (execution date), not the analyzed day.
- **Inputs/outputs:** From OpenAI → to **Filter Fields for HR Dashboard**
- **Version notes:** Code v2.
- **Edge cases / failures:**
  - **JSON parsing risk:** If `response` is not strict JSON, execution fails. Consider extracting JSON via regex or using “Structured Output” approaches.
  - Hashing stability: Hash changes if `employee_id` formatting changes (e.g., numeric vs string with leading zeros).
  - `analysis_date` may be misleading; often you want the “yesterday” date.

**Sticky notes in this block:**
- “Note: Fetch Employees” (anonymization & dashboard intent)
- “Note: Stress Prediction” (action & intervention intent)

---

### Block 6 — HR Dashboard Logging + High-Stress Intervention

**Overview:**  
Writes anonymized daily results to Google Sheets, checks for high stress, then either sends a supportive Slack DM and logs to Postgres or continues looping.

**Nodes involved:**
- Filter Fields for HR Dashboard
- Save to HR Dashboard
- Check for High Stress
- Send Slack DM to High Stress Employee
- Log Counseling Proposal

#### Node: **Filter Fields for HR Dashboard**
- **Type / role:** Set node; intended to remove sensitive fields before writing to Sheets.
- **Configuration choices:** No fields defined in JSON (currently a no-op).
- **Inputs/outputs:** From anonymization → to **Save to HR Dashboard**
- **Version notes:** Set v3.2.
- **Edge cases / failures:**
  - Since it does nothing, employee_id and slack_user_id may still be present in the item (depending on node behavior and Sheets mapping). The Sheets node only maps selected columns, but the in-memory item still contains identifiers.

#### Node: **Save to HR Dashboard**
- **Type / role:** Google Sheets; appends a row for HR trend analysis.
- **Configuration choices:**
  - Operation: Append
  - Document: `YOUR_SPREADSHEET_ID`
  - Sheet name: `Stress Analysis Logs`
  - Column mapping (explicit):
    - `department`, `stress_level`, `stress_score`, `analysis_date`, `employee_hash`
- **Inputs/outputs:** From filtering → to **Check for High Stress**
- **Version notes:** Google Sheets v4.5.
- **Edge cases / failures:**
  - Spreadsheet ID not set / wrong permissions → 403.
  - Sheet/tab name mismatch → error.
  - Column headers must exist or n8n may append by position depending on settings.

#### Node: **Check for High Stress**
- **Type / role:** IF node; branches on stress level.
- **Configuration choices:**
  - Condition: `{{ $json.stress_level }}` equals `"high"` (case-sensitive).
  - True → **Send Slack DM to High Stress Employee**
  - False → **Start Employee Loop** (continue)
- **Inputs/outputs:** From Sheets append → to branch targets.
- **Version notes:** IF v2.
- **Edge cases / failures:**
  - Case mismatch (“High” vs “high”) will skip intervention. Consider lowercasing: `{{$json.stress_level.toLowerCase()}}`.

#### Node: **Send Slack DM to High Stress Employee**
- **Type / role:** Slack node; sends a private message.
- **Configuration choices:**
  - Operation: send
  - Authentication: OAuth2 (Slack credential)
  - Actual channel/user targeting text is not included in the JSON shown; must be configured to DM `slack_user_id`.
- **Inputs/outputs:** True branch → to **Log Counseling Proposal**
- **Version notes:** Slack v2.1.
- **Edge cases / failures:**
  - Requires `chat:write` and ability to open/im with user (some workspaces restrict).
  - If `slack_user_id` missing/invalid, message fails.
  - Message content should be carefully phrased and compliant with internal policy.

#### Node: **Log Counseling Proposal**
- **Type / role:** Postgres node; logs interventions to a restricted database table.
- **Configuration choices:**
  - Table: `public.counseling_logs`
  - The “columns” configuration appears corrupted (individual characters listed). As-is, it likely will not insert correct fields.
  - Intended columns (from sticky note and context): `employee_hash`, `department`, `stress_level`, `stress_score`, `notified_at`, `analysis_period_from`, `analysis_period_to`.
- **Inputs/outputs:** From Slack DM → loops back to **Start Employee Loop**.
- **Version notes:** Postgres v2.4.
- **Edge cases / failures:**
  - This node will likely fail due to malformed column mapping.
  - Missing DB credentials / network access.
  - Schema/table mismatch.

**Sticky notes in this block:**
- “Note: Fetch Employees” (privacy + Sheets)
- “Note: Stress Prediction” (intervention logic)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Documentation / overview | — | — | ### 🧠 Employee Stress Monitor & Wellness Assistant … (includes setup notes about credentials, placeholder APIs, Sheets columns, Postgres table) |
| Run Daily at 2 AM | Schedule Trigger | Time-based entrypoint | — | Calculate Analysis Period | **📅 1. Schedule & Timeframe** - Trigger: Runs daily at 02:00 AM. - Code Node: Calculates timestamp range… |
| Note: Schedule | Sticky Note | Documentation | — | — | **📅 1. Schedule & Timeframe** - Trigger: Runs daily at 02:00 AM. - Code Node: Calculates timestamp range… |
| Calculate Analysis Period | Code | Compute previous-day window | Run Daily at 2 AM | Fetch Employee List | **📅 1. Schedule & Timeframe** - Trigger: Runs daily at 02:00 AM. - Code Node: Calculates timestamp range… |
| Note: Period Calculation | Sticky Note | Documentation | — | — | **🤖 2. Data Aggregation & AI Analysis** - Slack fetch/summarize - OpenAI sentiment - Attendance/Tasks via API - Consolidation… |
| Fetch Employee List | HTTP Request | Retrieve active employees | Calculate Analysis Period | Start Employee Loop | **🤖 2. Data Aggregation & AI Analysis** - Slack fetch/summarize - OpenAI sentiment - Attendance/Tasks via API - Consolidation… |
| Start Employee Loop | Split In Batches | Batch iteration over employees | Fetch Employee List; Check for High Stress (false); Log Counseling Proposal | Fetch Slack Messages | **🤖 2. Data Aggregation & AI Analysis** - Slack fetch/summarize - OpenAI sentiment - Attendance/Tasks via API - Consolidation… |
| Fetch Slack Messages | Slack | Collect employee messages | Start Employee Loop | Generate Message Summary | **🤖 2. Data Aggregation & AI Analysis** - Slack fetch/summarize - OpenAI sentiment - Attendance/Tasks via API - Consolidation… |
| Generate Message Summary | Code | Concatenate message text | Fetch Slack Messages | AI Tone Analysis | **🤖 2. Data Aggregation & AI Analysis** - Slack fetch/summarize - OpenAI sentiment - Attendance/Tasks via API - Consolidation… |
| AI Tone Analysis | OpenAI | Tone/sentiment inference | Generate Message Summary | Format Data for Privacy | **🤖 2. Data Aggregation & AI Analysis** - Slack fetch/summarize - OpenAI sentiment - Attendance/Tasks via API - Consolidation… |
| Format Data for Privacy | Set | Shape fields / privacy (currently empty) | AI Tone Analysis | Fetch Attendance Data | **🤖 2. Data Aggregation & AI Analysis** - Slack fetch/summarize - OpenAI sentiment - Attendance/Tasks via API - Consolidation… |
| Fetch Attendance Data | HTTP Request | Pull attendance logs | Format Data for Privacy | Calculate Attendance Metrics | **🤖 2. Data Aggregation & AI Analysis** - Slack fetch/summarize - OpenAI sentiment - Attendance/Tasks via API - Consolidation… |
| Calculate Attendance Metrics | Code | Compute overtime/late/absence | Fetch Attendance Data | Fetch Task Completion Rates | **🤖 2. Data Aggregation & AI Analysis** - Slack fetch/summarize - OpenAI sentiment - Attendance/Tasks via API - Consolidation… |
| Fetch Task Completion Rates | HTTP Request | Pull tasks | Calculate Attendance Metrics | Calculate Task Metrics | **🤖 2. Data Aggregation & AI Analysis** - Slack fetch/summarize - OpenAI sentiment - Attendance/Tasks via API - Consolidation… |
| Calculate Task Metrics | Code | Compute completion/overdue metrics | Fetch Task Completion Rates | Consolidate Features | **🤖 2. Data Aggregation & AI Analysis** - Slack fetch/summarize - OpenAI sentiment - Attendance/Tasks via API - Consolidation… |
| Consolidate Features | Code | Build unified feature record | Calculate Task Metrics | AI Stress Level Prediction | **🤖 2. Data Aggregation & AI Analysis** - Slack fetch/summarize - OpenAI sentiment - Attendance/Tasks via API - Consolidation… |
| AI Stress Level Prediction | OpenAI | Predict stress score/level | Consolidate Features | Generate Anonymized Data | **❤️ 4. Action & Intervention** - Checks if stress_level is “High”… |
| Generate Anonymized Data | Code | Hash identity + parse AI output | AI Stress Level Prediction | Filter Fields for HR Dashboard | **🛡️ 3. Anonymization & Dashboard** - Hash employee ID - Log anonymized stress score… |
| Filter Fields for HR Dashboard | Set | Remove identifiers before Sheets (currently empty) | Generate Anonymized Data | Save to HR Dashboard | **🛡️ 3. Anonymization & Dashboard** - Hash employee ID - Log anonymized stress score… |
| Save to HR Dashboard | Google Sheets | Append anonymized row | Filter Fields for HR Dashboard | Check for High Stress | **🛡️ 3. Anonymization & Dashboard** - Hash employee ID - Log anonymized stress score… |
| Check for High Stress | IF | Branch on stress_level | Save to HR Dashboard | Send Slack DM to High Stress Employee (true); Start Employee Loop (false) | **❤️ 4. Action & Intervention** - Checks if stress_level is “High”… |
| Send Slack DM to High Stress Employee | Slack | Private outreach message | Check for High Stress (true) | Log Counseling Proposal | **❤️ 4. Action & Intervention** - Checks if stress_level is “High”… |
| Log Counseling Proposal | Postgres | Store intervention event | Send Slack DM to High Stress Employee | Start Employee Loop | **❤️ 4. Action & Intervention** - Checks if stress_level is “High”… |
| Note: Fetch Employees | Sticky Note | Documentation | — | — | **🛡️ 3. Anonymization & Dashboard** - Hashing - Google Sheets logging… |
| Note: Stress Prediction | Sticky Note | Documentation | — | — | **❤️ 4. Action & Intervention** - True: DM + Postgres log; False: continue loop… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name: *Monitor employee stress levels from Slack and tasks to Google Sheets* (or your preferred title).
   - (Optional) Add a Sticky Note with the overview and setup reminders.

2) **Add the trigger**
   - Node: **Schedule Trigger**
   - Configure either:
     - **Daily at 02:00** (recommended if that’s the intent), *or*
     - Interval: every **2 hours** (matches the current JSON behavior).
   - Connect to the next node.

3) **Add “Calculate Analysis Period” (Code)**
   - Node: **Code** (Run once for each item)
   - Paste logic that outputs:
     - `analysisStart`, `analysisEnd` (yesterday 00:00 to 23:59:59.999, ISO)
     - `analysisStartUnix`, `analysisEndUnix` (epoch seconds)
   - Connect to **Fetch Employee List**.

4) **Add “Fetch Employee List” (HTTP Request)**
   - Node: **HTTP Request**
   - Method: GET
   - URL: your HR system endpoint (placeholder: `https://hr.example.com/api/employees`)
   - Query: `status=active`
   - Add authentication (as needed): API key / OAuth2 / custom headers.
   - Ensure the output provides per employee at least:
     - `employee_id`
     - `slack_user_id`
     - `department`
   - Connect to **Split In Batches**.

5) **Add “Start Employee Loop” (Split In Batches)**
   - Node: **Split In Batches**
   - Batch size: 10
   - Connect its main output to **Fetch Slack Messages**.
   - Later you will connect “continue” paths back into this node.

6) **Add “Fetch Slack Messages” (Slack)**
   - Node: **Slack**
   - Resource: **Search** (or replace with a safer channel/history strategy)
   - Configure Slack OAuth2 credentials with required scopes.
   - Configure the search query to target the employee and the timeframe (must be added; it’s not defined in the JSON).
     - You may need to incorporate `analysisStartUnix/analysisEndUnix` depending on Slack capabilities.
   - Connect to **Generate Message Summary**.

7) **Add “Generate Message Summary” (Code)**
   - Node: **Code**
   - Build `messageSummary` by concatenating up to 50 message texts; output `messageCount`.
   - Connect to **AI Tone Analysis**.

8) **Add “AI Tone Analysis” (OpenAI)**
   - Node: **OpenAI**
   - Operation: **Message**
   - Credentials: OpenAI API key
   - Prompt should instruct the model to return a numeric tone stress score (e.g., 0–1 or 0–100) and place it into a predictable field (recommended: return strict JSON).
   - Ensure downstream gets a usable field like `tone_stress_score` (either via “structured output” or a follow-up parsing node).
   - Connect to **Format Data for Privacy**.

9) **Add “Format Data for Privacy” (Set)**
   - Node: **Set**
   - Configure to keep/merge required fields for later steps:
     - `employee_id`, `slack_user_id`, `department`, `analysisStart`, `analysisEnd`
     - and the derived `tone_stress_score`
   - Connect to **Fetch Attendance Data**.

10) **Add “Fetch Attendance Data” (HTTP Request)**
   - Node: **HTTP Request**
   - GET `https://attendance.example.com/api/logs` (replace with real system)
   - Query params:
     - `employee_id={{$json.employee_id}}`
     - `from={{$json.analysisStart}}`
     - `to={{$json.analysisEnd}}`
   - Map the response to an array field named `attendance_logs` (or adjust the next code node).
   - Connect to **Calculate Attendance Metrics**.

11) **Add “Calculate Attendance Metrics” (Code)**
   - Node: **Code**
   - Compute:
     - `avg_work_hours`, `total_overtime_hours`, `late_arrival_count`, `absent_count`
   - Connect to **Fetch Task Completion Rates**.

12) **Add “Fetch Task Completion Rates” (HTTP Request)**
   - Node: **HTTP Request**
   - GET `https://tasks.example.com/api/tasks` (replace)
   - Query params:
     - `assignee_id={{$json.employee_id}}`
     - `from={{$json.analysisStart}}`
     - `to={{$json.analysisEnd}}`
   - Ensure tasks array is available as `tasks` (or adjust next node).
   - Connect to **Calculate Task Metrics**.

13) **Add “Calculate Task Metrics” (Code)**
   - Node: **Code**
   - Compute:
     - `task_completion_rate`, `overdue_task_count`, `avg_task_delay_days`
   - Connect to **Consolidate Features**.

14) **Add “Consolidate Features” (Code)**
   - Node: **Code**
   - Output a single object with all signals used for prediction.
   - (Recommended) Include `avg_task_delay_days` since it is computed earlier.
   - Connect to **AI Stress Level Prediction**.

15) **Add “AI Stress Level Prediction” (OpenAI)**
   - Node: **OpenAI** → Operation: **Message**
   - Prompt the model to output strict JSON, for example:
     - `stress_score` (number)
     - `stress_level` (low/medium/high, enforce lowercase)
     - `main_risk_factors` (array of strings)
   - Ensure the OpenAI node stores the JSON in a field you will parse next (the existing workflow expects `$json.response`).
   - Connect to **Generate Anonymized Data**.

16) **Add “Generate Anonymized Data” (Code)**
   - Node: **Code**
   - Hash `employee_id` with SHA-256 → `employee_hash`
   - Parse the AI output JSON safely (recommended: add error handling if not valid JSON).
   - Output both:
     - anonymized dashboard fields
     - identifiers needed for DM (`slack_user_id`) and internal logs (`employee_id`)
   - Connect to **Filter Fields for HR Dashboard**.

17) **Add “Filter Fields for HR Dashboard” (Set)**
   - Node: **Set**
   - Keep only what HR should see in Sheets:
     - `employee_hash`, `department`, `stress_score`, `stress_level`, `analysis_date`
   - Connect to **Save to HR Dashboard**.

18) **Add “Save to HR Dashboard” (Google Sheets)**
   - Node: **Google Sheets**
   - Credentials: Google Sheets OAuth2
   - Operation: **Append**
   - Document ID: your spreadsheet ID
   - Sheet name: `Stress Analysis Logs`
   - Map columns:
     - `employee_hash`, `department`, `stress_level`, `stress_score`, `analysis_date`
   - Connect to **Check for High Stress**.

19) **Add “Check for High Stress” (IF)**
   - Node: **IF**
   - Condition: `{{$json.stress_level}} equals "high"` (or better: `{{$json.stress_level.toLowerCase()}}`)
   - True output → **Send Slack DM to High Stress Employee**
   - False output → connect back to **Start Employee Loop** (to process next batch item).

20) **Add “Send Slack DM to High Stress Employee” (Slack)**
   - Node: **Slack**
   - Operation: **Send**
   - Authentication: OAuth2
   - Configure recipient using `slack_user_id` (ensure the item at this point still contains it; you may need to branch before filtering).
   - Message: empathetic, non-diagnostic, offers resources/counseling.
   - Connect to **Log Counseling Proposal**.

21) **Add “Log Counseling Proposal” (Postgres)**
   - Node: **Postgres**
   - Credentials: Postgres connection to a restricted DB
   - Operation: Insert row into `public.counseling_logs`
   - Columns (recommended):
     - `employee_hash`, `department`, `stress_level`, `stress_score`
     - `notified_at` (now)
     - `analysis_period_from`, `analysis_period_to`
   - Connect its output back to **Start Employee Loop**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Configure credentials: OpenAI, Slack, Google Sheets, Postgres | Mentioned in “Workflow Overview” sticky note |
| Attendance and Tasks HTTP nodes are placeholders; replace with Jira/Asana/BambooHR or your APIs | Mentioned in “Workflow Overview” sticky note |
| Create Google Sheet columns (example): `employee_hash`, `stress_score`, etc., and Postgres table `counseling_logs` | Mentioned in “Workflow Overview” sticky note |
| Privacy approach: hash Employee ID for HR dashboard; DM only when “High” | Mentioned across “Workflow Overview”, “Anonymization & Dashboard”, and “Action & Intervention” sticky notes |

