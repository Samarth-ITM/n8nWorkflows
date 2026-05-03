Generate consulting KPI summaries with GPT-4o-mini, Slack and Google Sheets

https://n8nworkflows.xyz/workflows/generate-consulting-kpi-summaries-with-gpt-4o-mini--slack-and-google-sheets-12857


# Generate consulting KPI summaries with GPT-4o-mini, Slack and Google Sheets

## 1. Workflow Overview

**Workflow name:** Automated Consulting KPI Reporting with AI Insights and Threshold Alerts  
**Stated title:** Generate consulting KPI summaries with GPT-4o-mini, Slack and Google Sheets

**Purpose:**  
Runs on a schedule to fetch KPI data from two client dashboards, normalizes/structures the KPI payload, logs results into Google Sheets, generates a concise narrative summary + insights using GPT‑4o‑mini, then routes notifications: a normal daily summary vs. critical alerts (Slack + email). Finally, it merges branches and schedules a follow-up meeting in Google Calendar, then performs final formatting/logging.

### 1.1 Scheduled Execution & Configuration
Triggers daily (schedule node) and sets/holds runtime configuration.

### 1.2 KPI Data Collection (Two Dashboards)
Fetches KPI data from two sources in parallel via HTTP.

### 1.3 KPI Normalization & Persistence
Combines/structures KPI data and writes rows to Google Sheets for historical logging.

### 1.4 AI Summary Generation
Uses an n8n LangChain Agent with **OpenAI GPT‑4o‑mini** as the chat model to generate KPI narrative summary and insights.

### 1.5 Threshold Routing (Critical vs Normal)
A Switch node evaluates KPI thresholds and routes to:
- **Critical path:** Slack alert + critical email
- **Normal path:** Slack summary + full report email

### 1.6 Consolidation & Follow-up Scheduling
Merges branch outputs, schedules a follow-up meeting in Google Calendar, then final “set” formatting/logging.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Execution & Workflow Configuration

**Overview:** Starts the workflow on a schedule and prepares configuration variables used by later nodes (e.g., endpoints, thresholds, sheet IDs).  
**Nodes involved:** Daily KPI Report Trigger, Workflow Configuration

#### Node: Daily KPI Report Trigger
- **Type / role:** `Schedule Trigger` — workflow entrypoint on a time schedule.
- **Configuration (interpreted):** No schedule details are provided in the JSON parameters (empty object), meaning the schedule would be defined in the UI or was omitted from export.
- **Connections:** Outputs to **Workflow Configuration**.
- **Version notes:** typeVersion **1.3**.
- **Failure/edge cases:**
  - Misconfigured schedule (never fires / fires too often).
  - Timezone assumptions (server timezone vs workspace timezone).

#### Node: Workflow Configuration
- **Type / role:** `Set` — central place to define constants and runtime parameters.
- **Configuration (interpreted):** Parameters are empty in the JSON, so currently it does not set fields. In a complete build, this is typically where you define:
  - API base URLs, tokens (prefer credentials), client identifiers
  - KPI thresholds (e.g., churn > X, revenue < Y)
  - Google Sheet ID / tab name
  - Slack channel IDs
  - Email recipients
- **Connections:** Outputs to both HTTP nodes in parallel:
  - **Fetch Client Dashboard 1 KPIs**
  - **Fetch Client Dashboard 2 KPIs**
- **Version notes:** typeVersion **3.4**.
- **Failure/edge cases:**
  - Downstream expressions expecting config fields may fail with “Cannot read properties of undefined” if you don’t set them.

---

### Block 2 — KPI Data Collection (Parallel HTTP)

**Overview:** Pulls KPI data from two separate dashboard endpoints/sources concurrently.  
**Nodes involved:** Fetch Client Dashboard 1 KPIs, Fetch Client Dashboard 2 KPIs

#### Node: Fetch Client Dashboard 1 KPIs
- **Type / role:** `HTTP Request` — retrieve KPI JSON from Dashboard 1.
- **Configuration (interpreted):** Parameters are empty in JSON; you must define:
  - Method (typically GET)
  - URL (dashboard API endpoint)
  - Auth (API key/Bearer/OAuth2 depending on source)
  - Response format (JSON)
- **Input:** From **Workflow Configuration**.
- **Output:** To **Structure KPI Data**.
- **Version notes:** typeVersion **4.3**.
- **Failure/edge cases:**
  - Auth failures (401/403), rate limits (429), timeouts
  - Non-JSON response causing parsing issues
  - Partial KPI payloads (missing keys)

#### Node: Fetch Client Dashboard 2 KPIs
- **Type / role:** `HTTP Request` — retrieve KPI JSON from Dashboard 2.
- **Configuration (interpreted):** Same as Dashboard 1; parameters empty in JSON.
- **Input:** From **Workflow Configuration**.
- **Output:** To **Structure KPI Data**.
- **Version notes:** typeVersion **4.3**.
- **Failure/edge cases:** Same as Dashboard 1 plus inconsistent schemas between dashboards.

---

### Block 3 — KPI Normalization & Persistence (Sheets)

**Overview:** Normalizes KPI payload(s) into a consistent schema and appends it to Google Sheets for historical tracking.  
**Nodes involved:** Structure KPI Data, Log KPIs to Google Sheets

#### Node: Structure KPI Data
- **Type / role:** `Set` — map raw dashboard responses into a single KPI object/row model.
- **Configuration (interpreted):** Empty in JSON. In practice, you would:
  - Select fields from dashboard responses (e.g., revenue, MRR, churn, active users)
  - Add metadata: date, client name/source, dashboard identifier
  - Ensure numeric parsing and default values (0 / null)
- **Inputs:** Two separate inbound connections:
  - From **Fetch Client Dashboard 1 KPIs**
  - From **Fetch Client Dashboard 2 KPIs**
  > Note: both feed into the same node; unless you merge explicitly, n8n will execute the node once per incoming item/branch (items may not be “combined”).
- **Output:** To **Log KPIs to Google Sheets**.
- **Version notes:** typeVersion **3.4**.
- **Failure/edge cases:**
  - If dashboards return different shapes, mapping expressions can fail.
  - Numeric strings with commas/locale may require cleaning.

#### Node: Log KPIs to Google Sheets
- **Type / role:** `Google Sheets` — append KPI rows for storage and trend analysis.
- **Configuration (interpreted):** Empty in JSON. Typically requires:
  - Operation: Append / Add row(s)
  - Spreadsheet ID and Sheet name (tab)
  - Column mapping from structured KPI fields
- **Input:** From **Structure KPI Data**.
- **Output:** To **Generate KPI Summary and Insights**.
- **Credentials:** Google Sheets OAuth2/Service Account (configured in n8n credentials).
- **Version notes:** typeVersion **4.7**.
- **Failure/edge cases:**
  - Permission errors (sheet not shared with service account / OAuth user)
  - Schema mismatch (missing columns, wrong header names)
  - Quota limits / Google API transient errors

---

### Block 4 — AI Summary Generation (LangChain Agent + GPT-4o-mini)

**Overview:** Produces a human-friendly KPI summary and actionable insights from the structured KPIs.  
**Nodes involved:** Generate KPI Summary and Insights, OpenAI GPT-4o-mini

#### Node: OpenAI GPT-4o-mini
- **Type / role:** `LM Chat OpenAI` (LangChain) — provides the chat model backing the agent.
- **Configuration (interpreted):** Empty in JSON. You must configure:
  - Model: **gpt-4o-mini**
  - OpenAI credentials (API key)
  - Temperature / max tokens (optional)
- **Connection:** Feeds the agent via the **ai_languageModel** connection to **Generate KPI Summary and Insights**.
- **Version notes:** typeVersion **1.3**.
- **Failure/edge cases:**
  - Missing/invalid API key
  - Model name not available in your account/region
  - Token limits if you pass large KPI history

#### Node: Generate KPI Summary and Insights
- **Type / role:** `LangChain Agent` — orchestrates prompt + tool/model usage to output summary text.
- **Configuration (interpreted):** Empty in JSON; in a working setup you define:
  - System instructions (tone, format, what to highlight)
  - User prompt template using KPI fields from incoming items
  - Expected outputs: summary, bullet insights, risks, next steps
- **Inputs:**
  - Main input from **Log KPIs to Google Sheets** (the KPIs)
  - Language model input from **OpenAI GPT-4o-mini**
- **Output:** To **Check Critical KPI Thresholds**.
- **Version notes:** typeVersion **3**.
- **Failure/edge cases:**
  - If prompt references fields not present, output may be low quality or expressions may fail (depending on how you template).
  - Hallucinated numbers if you don’t force the agent to only use provided values.

---

### Block 5 — Threshold Routing & Notifications

**Overview:** Evaluates KPI criticality and sends either a critical alert (Slack + email) or standard daily summary (Slack + email).  
**Nodes involved:** Check Critical KPI Thresholds, Post Summary to Slack, Email Full Report, Send Critical Alert to Slack, Email Critical Alert

#### Node: Check Critical KPI Thresholds
- **Type / role:** `Switch` — branching based on KPI thresholds.
- **Configuration (interpreted):** Empty in JSON. You must define:
  - Rules/conditions (e.g., churnRate > 0.05, MRR < target, NPS drop)
  - Which output is “critical” vs “normal”
- **Inputs:** From **Generate KPI Summary and Insights**.
- **Outputs (as wired):**
  - **Output 0 (critical branch):** Send Critical Alert to Slack, Email Critical Alert
  - **Output 1 (normal branch):** Post Summary to Slack, Email Full Report
- **Version notes:** typeVersion **3.4**.
- **Failure/edge cases:**
  - Missing KPI fields in conditions → evaluation errors or always-false
  - Type issues (string vs number comparisons)

#### Node: Post Summary to Slack
- **Type / role:** `Slack` — posts routine KPI summary to Slack.
- **Configuration (interpreted):** Parameters empty in JSON; configure:
  - Resource/operation: post message
  - Channel
  - Message content from agent output (summary/insights)
- **Input:** From **Check Critical KPI Thresholds** (normal branch).
- **Output:** To **Consolidate All Branches** (index 0 input).
- **Version notes:** typeVersion **2.4**.
- **Failure/edge cases:** Slack auth/scopes, invalid channel, message too long, rate limiting.

#### Node: Send Critical Alert to Slack
- **Type / role:** `Slack` — posts urgent alert when thresholds are exceeded.
- **Configuration (interpreted):** Same as Slack post, but message should be “alert-style” and include key KPI deltas + recommended action.
- **Input:** From **Check Critical KPI Thresholds** (critical branch).
- **Output:** Not merged in current wiring (no outgoing connection shown).
- **Version notes:** typeVersion **2.4**.
- **Failure/edge cases:** Same as Post Summary to Slack.

#### Node: Email Full Report
- **Type / role:** `Gmail` — sends the full daily KPI report via email.
- **Configuration (interpreted):** Empty in JSON; configure:
  - To/CC/BCC recipients
  - Subject line (include date/client)
  - Body (summary + KPI table + links to dashboards/sheet)
- **Input:** From **Check Critical KPI Thresholds** (normal branch).
- **Output:** To **Consolidate All Branches** (index 1 input).
- **Version notes:** typeVersion **2.2**.
- **Failure/edge cases:** OAuth consent/scopes, Gmail sending limits, invalid recipients.

#### Node: Email Critical Alert
- **Type / role:** `Gmail` — sends urgent email alert for critical KPI breaches.
- **Configuration (interpreted):** Similar to Email Full Report but shorter and urgent; include remediation steps.
- **Input:** From **Check Critical KPI Thresholds** (critical branch).
- **Output:** Not merged in current wiring (no outgoing connection shown).
- **Version notes:** typeVersion **2.2**.
- **Failure/edge cases:** Same as Email Full Report.

**Important wiring note:**  
Only the **normal** Slack+Email path is connected into **Consolidate All Branches**. The **critical** Slack+Email nodes are *not* connected to the merge node in this JSON, so the workflow may not proceed to calendar scheduling when an alert is critical (unless n8n continues on another branch independently and you accept partial completion). If you want consistent downstream steps, connect critical nodes into the merge as well (or use “Always Output Data” patterns).

---

### Block 6 — Consolidation, Calendar Follow-up, Final Formatting

**Overview:** Merges outputs from notification branches, schedules a follow-up meeting, and performs final formatting/logging.  
**Nodes involved:** Consolidate All Branches, Schedule Follow-up Meeting, Final Logging and Formatting

#### Node: Consolidate All Branches
- **Type / role:** `Merge` — joins parallel branches into a single execution path.
- **Configuration (interpreted):** Parameters empty in JSON; typical merge modes:
  - “Wait” to wait for both inputs
  - “Append” to pass items through
- **Inputs (as wired):**
  - Input 0 from **Post Summary to Slack**
  - Input 1 from **Email Full Report**
- **Output:** To **Schedule Follow-up Meeting**.
- **Version notes:** typeVersion **3.2**.
- **Failure/edge cases:**
  - If one input never arrives (e.g., because only critical path ran), merge can stall or output nothing depending on mode.
  - Item alignment issues if inputs contain different item counts.

#### Node: Schedule Follow-up Meeting
- **Type / role:** `Google Calendar` — creates a calendar event for follow-up.
- **Configuration (interpreted):** Empty in JSON; usually configure:
  - Operation: Create event
  - Calendar ID
  - Start/end time (e.g., next business day 10:00)
  - Attendees (consultant + stakeholders)
  - Description linking to the KPI sheet and summary
- **Input:** From **Consolidate All Branches**.
- **Output:** To **Final Logging and Formatting**.
- **Version notes:** typeVersion **1.3**.
- **Failure/edge cases:** Calendar permissions, timezone handling, event conflicts, invalid attendee emails.

#### Node: Final Logging and Formatting
- **Type / role:** `Set` — prepares final output payload (for logs, audits, or further integrations).
- **Configuration (interpreted):** Empty in JSON; you’d typically:
  - Store execution timestamp, report text, decision branch (critical/normal)
  - Include links to Slack message, sent email IDs, calendar event link
- **Input:** From **Schedule Follow-up Meeting**.
- **Output:** None (terminal node).
- **Version notes:** typeVersion **3.4**.
- **Failure/edge cases:** If you reference fields not returned by upstream nodes (e.g., calendar event HTML link), expressions may fail.

---

### Sticky Notes (Comments)
All sticky notes in this workflow have **empty content** (`""`). They do not add documentation or links.

Nodes: Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4, Sticky Note5

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily KPI Report Trigger | Schedule Trigger | Scheduled entrypoint | — | Workflow Configuration |  |
| Workflow Configuration | Set | Central config/constants | Daily KPI Report Trigger | Fetch Client Dashboard 1 KPIs; Fetch Client Dashboard 2 KPIs |  |
| Fetch Client Dashboard 1 KPIs | HTTP Request | Pull KPI data (source 1) | Workflow Configuration | Structure KPI Data |  |
| Fetch Client Dashboard 2 KPIs | HTTP Request | Pull KPI data (source 2) | Workflow Configuration | Structure KPI Data |  |
| Structure KPI Data | Set | Normalize/map KPIs | Fetch Client Dashboard 1 KPIs; Fetch Client Dashboard 2 KPIs | Log KPIs to Google Sheets |  |
| Log KPIs to Google Sheets | Google Sheets | Persist KPI rows | Structure KPI Data | Generate KPI Summary and Insights |  |
| Generate KPI Summary and Insights | LangChain Agent | Generate narrative KPI insights | Log KPIs to Google Sheets; OpenAI GPT-4o-mini (model connection) | Check Critical KPI Thresholds |  |
| OpenAI GPT-4o-mini | LM Chat OpenAI (LangChain) | LLM provider for agent | — | Generate KPI Summary and Insights (ai_languageModel) |  |
| Check Critical KPI Thresholds | Switch | Branch critical vs normal | Generate KPI Summary and Insights | (Critical) Send Critical Alert to Slack; Email Critical Alert; (Normal) Post Summary to Slack; Email Full Report |  |
| Post Summary to Slack | Slack | Post daily summary | Check Critical KPI Thresholds (normal) | Consolidate All Branches |  |
| Email Full Report | Gmail | Email daily report | Check Critical KPI Thresholds (normal) | Consolidate All Branches |  |
| Send Critical Alert to Slack | Slack | Post urgent Slack alert | Check Critical KPI Thresholds (critical) | — |  |
| Email Critical Alert | Gmail | Email urgent alert | Check Critical KPI Thresholds (critical) | — |  |
| Consolidate All Branches | Merge | Join notification branches | Post Summary to Slack; Email Full Report | Schedule Follow-up Meeting |  |
| Schedule Follow-up Meeting | Google Calendar | Create follow-up calendar event | Consolidate All Branches | Final Logging and Formatting |  |
| Final Logging and Formatting | Set | Final output structuring | Schedule Follow-up Meeting | — |  |
| Sticky Note | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note1 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note2 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note3 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note4 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note5 | Sticky Note | Comment container (empty) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name it: *Automated Consulting KPI Reporting with AI Insights and Threshold Alerts*

2. **Add “Daily KPI Report Trigger” (Schedule Trigger)**
   - Choose your schedule (e.g., every weekday at 08:00).
   - Ensure timezone is correct for reporting.

3. **Add “Workflow Configuration” (Set)**
   - Add fields you will reuse, for example:
     - `dashboard1Url`, `dashboard2Url`
     - `criticalChurnThreshold`, `criticalRevenueDropThreshold`
     - `sheetId`, `sheetTabName`
     - `slackChannelDaily`, `slackChannelAlerts`
     - `emailToDaily`, `emailToAlerts`
   - Connect: **Daily KPI Report Trigger → Workflow Configuration**

4. **Add two HTTP Request nodes**
   - Node A: “Fetch Client Dashboard 1 KPIs”
     - Method: GET (typical)
     - URL: `{{$json.dashboard1Url}}` (or hardcode)
     - Authentication: configure per source (Header Auth/Bearer/OAuth2)
     - Response: JSON
   - Node B: “Fetch Client Dashboard 2 KPIs”
     - Same pattern with `dashboard2Url`
   - Connect: **Workflow Configuration → both HTTP nodes** (two outgoing connections)

5. **Add “Structure KPI Data” (Set)**
   - Map both dashboard responses into a consistent schema.
   - Recommended fields:
     - `reportDate` (e.g., `{{$now}}` formatted)
     - `client`, `sourceDashboard`
     - KPI numeric fields (revenue, MRR, churn, tickets, utilization, etc.)
   - Connect:
     - **Fetch Client Dashboard 1 KPIs → Structure KPI Data**
     - **Fetch Client Dashboard 2 KPIs → Structure KPI Data**
   - If you truly need a combined single object from both dashboards, add an explicit **Merge** (or “Wait/Combine” pattern) before structuring.

6. **Add “Log KPIs to Google Sheets” (Google Sheets)**
   - Credentials: Google Sheets OAuth2 (or service account)
   - Operation: Append row(s)
   - Spreadsheet: `sheetId`
   - Sheet/tab: `sheetTabName`
   - Map columns to the structured KPI fields
   - Connect: **Structure KPI Data → Log KPIs to Google Sheets**

7. **Add LLM node “OpenAI GPT-4o-mini”**
   - Credentials: OpenAI API key in n8n credentials
   - Model: `gpt-4o-mini`
   - (Optional) set temperature low (e.g., 0.2–0.4) for consistent reporting

8. **Add “Generate KPI Summary and Insights” (LangChain Agent)**
   - Configure system/user instructions to:
     - Summarize key movements (day-over-day / vs targets if provided)
     - Flag risks and propose actions
     - Output in a structured format (e.g., sections: Summary, Risks, Actions)
   - Attach the model:
     - Connect **OpenAI GPT-4o-mini → Generate KPI Summary and Insights** via the agent’s **language model** connection.
   - Connect main flow:
     - **Log KPIs to Google Sheets → Generate KPI Summary and Insights**

9. **Add “Check Critical KPI Thresholds” (Switch)**
   - Create conditions using KPI fields (and/or agent output fields if you structured them), e.g.:
     - churnRate > `criticalChurnThreshold`
     - revenueChangePct < `-criticalRevenueDropThreshold`
   - Define outputs:
     - Output 0: critical
     - Output 1: normal
   - Connect: **Generate KPI Summary and Insights → Check Critical KPI Thresholds**

10. **Add notification nodes**
   - **Normal branch**
     - “Post Summary to Slack” (Slack)
       - Credentials: Slack OAuth2
       - Operation: Post message
       - Channel: `slackChannelDaily`
       - Text: from agent output (summary/insights)
     - “Email Full Report” (Gmail)
       - Credentials: Gmail OAuth2
       - To: `emailToDaily`
       - Subject/body: include report date + full summary and KPI list
   - **Critical branch**
     - “Send Critical Alert to Slack” (Slack)
       - Channel: `slackChannelAlerts`
       - Message: highlight breached thresholds + immediate actions
     - “Email Critical Alert” (Gmail)
       - To: `emailToAlerts`
       - Subject: “CRITICAL KPI ALERT – <client> – <date>”
   - Connect from Switch:
     - Output (critical) → Slack alert + Critical email
     - Output (normal) → Slack summary + Full report email

11. **Add “Consolidate All Branches” (Merge)**
   - Configure merge mode to fit your intent (commonly “Wait” to ensure both Slack+Email completed).
   - Connect:
     - **Post Summary to Slack → Consolidate All Branches (Input 0)**
     - **Email Full Report → Consolidate All Branches (Input 1)**
   - To include critical path in downstream scheduling, also connect:
     - **Send Critical Alert to Slack** and **Email Critical Alert** into this merge (or use a second merge), otherwise the calendar step may not run on critical days.

12. **Add “Schedule Follow-up Meeting” (Google Calendar)**
   - Credentials: Google Calendar OAuth2
   - Operation: Create event
   - Calendar: choose target calendar
   - Start/end: compute next available slot (or fixed time)
   - Description: include KPI summary + link to Google Sheet
   - Connect: **Consolidate All Branches → Schedule Follow-up Meeting**

13. **Add “Final Logging and Formatting” (Set)**
   - Build a final object with:
     - `runAt`, `branchTaken`, `slackMessageTs` (if available), `emailMessageId`, `calendarEventLink`
   - Connect: **Schedule Follow-up Meeting → Final Logging and Formatting**

14. **Activate credentials & test**
   - Run manually once with sample KPI payloads.
   - Verify Slack messages, Gmail sends, Sheets row append, Calendar event creation.
   - Add retries/error handling (e.g., Error Trigger workflow) if needed.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but contain no text. | No additional embedded notes/links were included in the workflow JSON. |
| Branch consolidation currently only merges the “normal” Slack+Email path; critical notifications are not connected to the merge. | Consider wiring critical nodes into a merge to ensure calendar scheduling and final formatting also occur on critical days. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.