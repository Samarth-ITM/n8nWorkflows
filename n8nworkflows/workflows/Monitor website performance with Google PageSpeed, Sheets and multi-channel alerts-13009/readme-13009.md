Monitor website performance with Google PageSpeed, Sheets and multi-channel alerts

https://n8nworkflows.xyz/workflows/monitor-website-performance-with-google-pagespeed--sheets-and-multi-channel-alerts-13009


# Monitor website performance with Google PageSpeed, Sheets and multi-channel alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow periodically monitors a list of websites (stored in Google Sheets) by running Google PageSpeed tests, storing results back to Sheets, updating tracking data, and sending multi-channel alerts (Gmail, Discord, and WhatsApp via Rapiwa). It also includes pacing logic (wait 10s) and a mechanism to decide whether to run a test or skip/loop based on sheet row state.

**Target use cases:**
- Routine performance and availability monitoring for multiple URLs
- Logging PageSpeed metrics over time into Google Sheets
- Triggering alerts when tests run (and potentially when results indicate issues, depending on the logic inside the “Process Results” / “Update data” nodes)

### 1.1 Scheduled Start & Data Acquisition
Runs on a schedule, reads rows from Google Sheets, and limits the number of processed rows per execution.

### 1.2 Batching / Iteration Control
Iterates through sheet rows in batches; includes “empty response” checks to decide next action.

### 1.3 Decision & PageSpeed Execution Paths
Two HTTP Request nodes (“PageSpeed Test” and “PageSpeed Test2”) are used based on branching conditions.

### 1.4 Results Processing & Persistence
Processes PageSpeed responses, saves results to Sheets, and updates tracking columns.

### 1.5 Multi-channel Alerting + Rate Limiting
Sends notifications (Gmail, Discord, Rapiwa/WhatsApp) and waits 10 seconds before continuing the loop.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Start & Read Monitoring Targets
**Overview:** Starts the workflow on a schedule and fetches website monitoring rows from Google Sheets, then limits the number of processed rows to avoid quotas/timeouts.

**Nodes involved:**
- Schedule Trigger
- Get Data Form Sheet
- Limit (10)

#### Node: Schedule Trigger
- **Type / role:** Schedule Trigger (time-based entry point)
- **Configuration choices:** Not specified in the provided export (parameters empty). Typically defines interval/cron.
- **Inputs / outputs:**  
  - **Output →** Get Data Form Sheet
- **Version requirements:** typeVersion **1.2**
- **Edge cases / failures:**
  - Misconfigured schedule (never fires / fires too often)
  - Timezone mismatches (n8n instance timezone vs expected schedule)

#### Node: Get Data Form Sheet
- **Type / role:** Google Sheets node (read rows / fetch data)
- **Configuration choices:** Not specified (parameters empty). Intended to read a sheet containing URLs and metadata.
- **Inputs / outputs:**  
  - **Input ←** Schedule Trigger  
  - **Output →** Limit (10)
- **Version requirements:** typeVersion **4.7**
- **Edge cases / failures:**
  - OAuth/Service Account credential failure
  - Incorrect spreadsheet ID/sheet name/range
  - API quota exceeded
  - Empty dataset (feeds into later “check empty response” logic)

#### Node: Limit (10)
- **Type / role:** Limit node (cap number of items)
- **Configuration choices:** Parameters empty in export; implied purpose is limiting to **10 items** (based on node name).
- **Inputs / outputs:**  
  - **Input ←** Get Data Form Sheet  
  - **Output →** Loop Over Items
- **Version requirements:** typeVersion **1**
- **Edge cases / failures:**
  - If limit is not actually configured, node name may be misleading
  - Limiting too aggressively may delay coverage of all URLs

---

### Block 2 — Batching / Iteration Control & Empty-Response Gate
**Overview:** Iterates through items (sheet rows). Uses an IF node to detect empty/invalid iteration output; depending on outcome, either runs an immediate test path or computes elapsed days and decides whether to test.

**Nodes involved:**
- Loop Over Items
- If (check empty response)

#### Node: Loop Over Items
- **Type / role:** Split In Batches (iteration controller)
- **Configuration choices:** Parameters empty; typically defines batch size and “continue” behavior.
- **Connections:**  
  - **Input ←** Limit (10)  
  - **Output (main index 1) →** If (check empty response)  
  - Additionally receives loopbacks from:
    - If3 (false path) → Loop Over Items
    - Wait 10s → Loop Over Items
- **Version requirements:** typeVersion **3**
- **Edge cases / failures:**
  - Incorrect batch size can cause too many requests per run or too slow coverage
  - Loopbacks can create long executions; risk of timeout depending on n8n settings

#### Node: If (check empty response)
- **Type / role:** IF node (validates that the current batch/item has data)
- **Configuration choices:** Not specified; implied condition checks for empty response from SplitInBatches.
- **Connections:**  
  - **Input ←** Loop Over Items  
  - **True path (index 0) →** PageSpeed Test2  
  - **False path (index 1) →** Code (Calculate Days)
- **Version requirements:** typeVersion **2.2**
- **Edge cases / failures:**
  - If condition references a field that sometimes doesn’t exist, expression evaluation may fail
  - If “empty response” logic is inverted, workflow may test on wrong branch

---

### Block 3 — Timing/Eligibility Decision (Days Calculation + IF3)
**Overview:** Calculates time-based criteria (e.g., days since last check) and decides whether to run a PageSpeed test now or skip and move on.

**Nodes involved:**
- Code (Calculate Days)
- If3

#### Node: Code (Calculate Days)
- **Type / role:** Code node (custom JS transformation)
- **Configuration choices:** Parameters empty in export; expected to compute a “days since last run” or similar metric based on sheet data.
- **Connections:**  
  - **Input ←** If (check empty response) (false path)  
  - **Output →** If3
- **Version requirements:** typeVersion **2**
- **Edge cases / failures:**
  - Missing date fields / invalid date formats from Sheets
  - Timezone issues when computing day differences
  - Code node runtime errors (throws, undefined access)

#### Node: If3
- **Type / role:** IF node (eligibility gate for running a test)
- **Configuration choices:** Not specified; likely checks computed days threshold or row flags.
- **Connections:**  
  - **Input ←** Code (Calculate Days)  
  - **True path (index 0) →** PageSpeed Test  
  - **False path (index 1) →** Loop Over Items (skip/continue)
- **Version requirements:** typeVersion **2.2**
- **Edge cases / failures:**
  - Misconfigured threshold logic can lead to never testing or testing too frequently
  - Expression failures if the expected computed field is missing

---

### Block 4 — PageSpeed Calls (Two Request Variants)
**Overview:** Executes Google PageSpeed Insights API calls. Two separate HTTP Request nodes are present, likely for different device strategies (mobile/desktop) or different parameter sets depending on branch.

**Nodes involved:**
- PageSpeed Test2
- PageSpeed Test

#### Node: PageSpeed Test2
- **Type / role:** HTTP Request (external API call)
- **Configuration choices:** Empty in export; intended to call PageSpeed Insights endpoint with URL + key + strategy.
- **Connections:**  
  - **Input ←** If (check empty response) (true path)  
  - **Output →** Process Results
- **Version requirements:** typeVersion **4.2**
- **Edge cases / failures:**
  - Missing/invalid API key, or endpoint misconfiguration
  - Rate limiting / quota exhaustion
  - Non-200 responses; JSON parse issues if response is HTML/error
  - URL encoding problems for tested URLs

#### Node: PageSpeed Test
- **Type / role:** HTTP Request (external API call)
- **Configuration choices:** Empty in export; similar to PageSpeed Test2 but used after eligibility calculation.
- **Connections:**  
  - **Input ←** If3 (true path)  
  - **Output →** Process Results
- **Version requirements:** typeVersion **4.2**
- **Edge cases / failures:** same as PageSpeed Test2

---

### Block 5 — Process, Save, Update
**Overview:** Converts raw PageSpeed response into structured metrics, saves a results row into Google Sheets, then updates tracking data (e.g., last checked date, status, scores) and triggers alerting.

**Nodes involved:**
- Process Results
- Save Results
- Update data

#### Node: Process Results
- **Type / role:** Code node (extract/transform metrics)
- **Configuration choices:** Empty in export; expected to parse Lighthouse/PageSpeed JSON and map fields for saving and alerting.
- **Connections:**  
  - **Inputs ←** PageSpeed Test and PageSpeed Test2  
  - **Output →** Save Results
- **Version requirements:** typeVersion **2**
- **Edge cases / failures:**
  - PageSpeed response schema changes (missing nested fields)
  - Division/rounding errors, NaNs
  - If multiple items are processed, ensure the node handles item-wise mapping correctly

#### Node: Save Results
- **Type / role:** Google Sheets node (append/write metrics)
- **Configuration choices:** Empty in export; intended to persist each test run’s metrics.
- **Connections:**  
  - **Input ←** Process Results  
  - **Output →** Update data
- **Version requirements:** typeVersion **4.6**
- **Edge cases / failures:**
  - Writing to wrong sheet/range; permission issues
  - Data type mismatches (dates/numbers)
  - Quota limits for writes

#### Node: Update data
- **Type / role:** Google Sheets node (update original row tracking fields)
- **Configuration choices:** Empty in export; likely updates “last checked”, “last score”, “alert sent”, etc.
- **Connections:**  
  - **Input ←** Save Results  
  - **Outputs →** Send a message2 (Gmail), Rapiwa, Send a message (Discord) in parallel
- **Version requirements:** typeVersion **4.7**
- **Edge cases / failures:**
  - Update requires a row identifier; if absent, updates may fail or overwrite wrong row
  - Partial failure: if update succeeds but alerts fail (or vice versa) you may get inconsistent state

---

### Block 6 — Multi-channel Alerting + Wait + Continue Loop
**Overview:** Sends alerts via Gmail, WhatsApp (Rapiwa), and Discord. After Discord sends, a 10-second wait is applied and the loop continues.

**Nodes involved:**
- Send a message2 (Gmail)
- Rapiwa
- Send a message (Discord)
- Wait 10s

#### Node: Send a message2
- **Type / role:** Gmail node (send email)
- **Configuration choices:** Empty in export; intended to send an email alert containing the results.
- **Connections:**  
  - **Input ←** Update data  
  - **No outgoing connection** (ends branch)
- **Version requirements:** typeVersion **2.1**
- **Edge cases / failures:**
  - OAuth token expired / insufficient scopes
  - Gmail sending limits
  - Invalid recipient addresses

#### Node: Rapiwa
- **Type / role:** Rapiwa node (3rd-party WhatsApp messaging integration)
- **Configuration choices:** Empty in export; expected to send WhatsApp messages.
- **Connections:**  
  - **Input ←** Update data  
  - **No outgoing connection** (ends branch)
- **Version requirements:** typeVersion **1** (community/third-party node)
- **Edge cases / failures:**
  - Credential/API token errors
  - Provider downtime or rate limiting
  - Message formatting constraints

#### Node: Send a message
- **Type / role:** Discord node (send message)
- **Configuration choices:** Empty in export; sends an alert to a Discord channel.
- **Connections:**  
  - **Input ←** Update data  
  - **Output →** Wait 10s
- **Version requirements:** typeVersion **2**
- **Edge cases / failures:**
  - Invalid bot token/webhook config
  - Missing permissions in target channel
  - Rate limits if many URLs per run

#### Node: Wait 10s
- **Type / role:** Wait node (delay execution)
- **Configuration choices:** Empty in export; implied wait duration is **10 seconds** (from node name).
- **Connections:**  
  - **Input ←** Send a message (Discord)  
  - **Output →** Loop Over Items (continue to next item)
- **Version requirements:** typeVersion **1.1**
- **Edge cases / failures:**
  - Long runs if many items; may exceed execution time limit on some hosting plans

---

### Sticky Notes
There are 6 sticky note nodes (Sticky Note, Sticky Note1..5, Sticky Note3..4), but **all have empty content** in the provided workflow export. They do not add documented instructions or links.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Starts workflow on a schedule | — | Get Data Form Sheet |  |
| Get Data Form Sheet | Google Sheets | Read monitoring targets (URLs/metadata) | Schedule Trigger | Limit (10) |  |
| Limit (10) | Limit | Caps number of rows processed per run | Get Data Form Sheet | Loop Over Items |  |
| Loop Over Items | Split In Batches | Iterates through rows with loopbacks | Limit (10); If3; Wait 10s | If (check empty response) |  |
| If (check empty response) | IF | Detects empty/invalid batch output | Loop Over Items | PageSpeed Test2; Code (Calculate Days) |  |
| PageSpeed Test2 | HTTP Request | Runs PageSpeed API (variant A) | If (check empty response) | Process Results |  |
| Code (Calculate Days) | Code | Computes elapsed time / eligibility fields | If (check empty response) | If3 |  |
| If3 | IF | Decides whether to run PageSpeed or skip | Code (Calculate Days) | PageSpeed Test; Loop Over Items |  |
| PageSpeed Test | HTTP Request | Runs PageSpeed API (variant B) | If3 | Process Results |  |
| Process Results | Code | Extracts metrics and prepares output for Sheets/alerts | PageSpeed Test; PageSpeed Test2 | Save Results |  |
| Save Results | Google Sheets | Writes/append performance results | Process Results | Update data |  |
| Update data | Google Sheets | Updates tracking fields on source row | Save Results | Send a message2; Rapiwa; Send a message |  |
| Send a message2 | Gmail | Sends email alert | Update data | — |  |
| Rapiwa | Rapiwa (community) | Sends WhatsApp alert | Update data | — |  |
| Send a message | Discord | Sends Discord alert | Update data | Wait 10s |  |
| Wait 10s | Wait | Delay to reduce rate-limit pressure | Send a message | Loop Over Items |  |
| Sticky Note | Sticky Note | Annotation (empty) | — | — |  |
| Sticky Note1 | Sticky Note | Annotation (empty) | — | — |  |
| Sticky Note2 | Sticky Note | Annotation (empty) | — | — |  |
| Sticky Note3 | Sticky Note | Annotation (empty) | — | — |  |
| Sticky Note4 | Sticky Note | Annotation (empty) | — | — |  |
| Sticky Note5 | Sticky Note | Annotation (empty) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Schedule Trigger” (Schedule Trigger node)**
   - Set the schedule you want (e.g., every hour / daily / cron).
   - This is the workflow entry node.

2) **Create “Get Data Form Sheet” (Google Sheets node)**
   - Authenticate Google Sheets (OAuth2 or Service Account).
   - Operation: *Read/Get many rows* (configure Spreadsheet, Sheet/Tab, Range).
   - Ensure each row includes at least:
     - A **URL** to test
     - A **row identifier** strategy (row number or unique ID) for later updates
     - Optional: lastChecked timestamp, thresholds, enabled flag
   - Connect: **Schedule Trigger → Get Data Form Sheet**

3) **Create “Limit (10)” (Limit node)**
   - Set **Max Items = 10** (to match the node name intent).
   - Connect: **Get Data Form Sheet → Limit (10)**

4) **Create “Loop Over Items” (Split In Batches node)**
   - Set **Batch Size = 1** (common for URL-by-URL processing) or as desired.
   - Connect: **Limit (10) → Loop Over Items**

5) **Create “If (check empty response)” (IF node)**
   - Configure a condition to detect whether the current item exists.
   - Example approach: check that `{{$json.url}}` *is not empty* (adapt to your sheet column).
   - Connect: **Loop Over Items → If (check empty response)**

6) **Create “PageSpeed Test2” (HTTP Request node)**
   - Method: GET
   - URL: Google PageSpeed Insights endpoint (commonly `https://www.googleapis.com/pagespeedonline/v5/runPagespeed`)
   - Add query parameters such as:
     - `url` = the website URL from the sheet (expression)
     - `strategy` = e.g., `mobile` or `desktop`
     - `key` = your Google API key (or use OAuth if you prefer; API key is common)
   - Connect: **If (check empty response) [true] → PageSpeed Test2**

7) **Create “Code (Calculate Days)” (Code node)**
   - Implement JS to compute days since last check (based on a sheet column like `lastChecked`).
   - Output fields used by the next IF node (e.g., `daysSinceLastCheck`, `shouldRun`).
   - Connect: **If (check empty response) [false] → Code (Calculate Days)**

8) **Create “If3” (IF node)**
   - Condition based on the computed values (e.g., run only if daysSinceLastCheck >= N, or if enabled flag is true).
   - Connect: **Code (Calculate Days) → If3**
   - Connect skip path: **If3 [false] → Loop Over Items** (to continue)

9) **Create “PageSpeed Test” (HTTP Request node)**
   - Same endpoint as PageSpeed Test2, but configure it differently if needed (e.g., different strategy, categories, or locale).
   - Connect: **If3 [true] → PageSpeed Test**

10) **Create “Process Results” (Code node)**
   - Parse PageSpeed response JSON and extract metrics you want (e.g., performance score, LCP, CLS, TBT).
   - Normalize data into flat fields for Sheets and alert messages.
   - Connect:  
     - **PageSpeed Test → Process Results**  
     - **PageSpeed Test2 → Process Results**

11) **Create “Save Results” (Google Sheets node)**
   - Operation: *Append row* (or *Add row*) into a “Results/Logs” tab.
   - Map extracted metrics to columns.
   - Connect: **Process Results → Save Results**

12) **Create “Update data” (Google Sheets node)**
   - Operation: *Update row* in the original “targets” tab.
   - Use row number/ID from the original item to update fields like:
     - lastChecked timestamp
     - lastPerformanceScore
     - status / lastError
   - Connect: **Save Results → Update data**

13) **Create alert nodes (three parallel branches from “Update data”)**
   - **“Send a message2” (Gmail node)**: configure From/To/Subject/Body using fields from Process Results.
   - **“Send a message” (Discord node)**: configure channel and message content.
   - **“Rapiwa” (Rapiwa node)**: configure recipient and message body.
   - Connect: **Update data → Gmail**, **Update data → Discord**, **Update data → Rapiwa**

14) **Create “Wait 10s” (Wait node)**
   - Set wait time to **10 seconds**.
   - Connect: **Send a message (Discord) → Wait 10s**
   - Connect: **Wait 10s → Loop Over Items** (continue iterating)

15) **(Optional) Add Sticky Notes**
   - In the provided workflow they are present but empty; you can add documentation notes for URLs, thresholds, and credential setup.

**Credential configuration checklist**
- Google Sheets credentials (OAuth2 or Service Account)
- Gmail OAuth2 credentials (scopes for sending email)
- Discord credentials (bot token or webhook depending on node configuration)
- Rapiwa credentials (provider API token/config)
- Google PageSpeed API key (typically stored as an n8n credential, environment variable, or static parameter)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but contain no text in the provided export. | Workflow canvas annotations are empty, so no additional operator guidance is embedded. |