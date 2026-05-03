Log Jira worklogs nightly from Google Sheets with Gmail and GPT-4o-mini

https://n8nworkflows.xyz/workflows/log-jira-worklogs-nightly-from-google-sheets-with-gmail-and-gpt-4o-mini-14779


# Log Jira worklogs nightly from Google Sheets with Gmail and GPT-4o-mini

Let me analyze this n8n workflow carefully and produce a comprehensive, structured reference document.

The workflow is called "Log Jira worklogs nightly from Google Sheets with Gmail and GPT-4o-mini". It's a scheduled automation that:

1. Triggers every night at 10 PM
2. Reads a Google Sheet with Jira worklog entries
3. Filters for pending rows with dates ≤ today
4. If nothing to process, sends a "nothing to do" email
5. If there are rows, processes them one at a time in a loop:
   - Builds the Jira API payload
   - Calls the Jira REST API to add a worklog
   - Handles the response (success/failure)
   - Updates the Google Sheet with the result
6. After all rows processed, builds a summary
7. Uses GPT-4o-mini to generate a friendly summary
8. Sends a final email report

Let me map out all nodes and their connections:

Nodes list:
1. Every Night 10 PM (scheduleTrigger) → Read Log Sheet
2. Read Log Sheet (googleSheets) → Filter Pending + Today
3. Filter Pending + Today (code) → Anything To Process?
4. Anything To Process? (if) → true: Email: Nothing To Do, false: Split Into Rows
5. Email: Nothing To Do (gmail)
6. Split Into Rows (code) → Loop Rows
7. Loop Rows (splitInBatches) → output 0: Build Summary, output 1: Build Payload
8. Build Payload (code) → Jira: Add Worklog
9. Jira: Add Worklog (httpRequest) → Handle Response
10. Handle Response (code) → Edit Fields
11. Edit Fields (set) → Update Sheet
12. Update Sheet (googleSheets) → Loop Rows (loops back)
13. Build Summary (code) → AI Summary
14. AI Summary (openAi) → Email: Final Report
15. Email: Final Report (gmail)

Sticky notes:
- SN: Jira Daily Worklog Automation - covers overview
- SN: Setup - setup instructions
- SN: Trigger + Read + Filter - covers Every Night 10 PM, Read Log Sheet, Filter Pending + Today, Anything To Process?
- SN: No Pending Rows - covers Email: Nothing To Do
- SN: Loop — One Row at a Time - covers Loop Rows, Split Into Rows
- SN: Build Payload - covers Build Payload
- SN: Jira API - covers Jira: Add Worklog
- SN: Handle Response + Update Sheet - covers Handle Response, Edit Fields, Update Sheet
- SN: Summary + Email - covers Build Summary, AI Summary, Email: Final Report

Let me now structure the document carefully.

For the sticky note coverage, I need to check positions to see which nodes are visually covered:

SN: Trigger + Read + Filter (position [0,496], width 1064, height 528) → covers x:0-1064, y:496-1024
- Every Night 10 PM: position [224,768] → within range
- Read Log Sheet: position [496,768] → within range
- Filter Pending + Today: position [688,768] → within range
- Anything To Process?: position [912,768] → within range (912 < 1064)

SN: No Pending Rows (position [1120,432], width 284, height 320) → covers x:1120-1404, y:432-752
- Email: Nothing To Do: position [1200,576] → within range

SN: Loop — One Row at a Time (position [1120,816], width 512, height 336) → covers x:1120-1632, y:816-1152
- Split Into Rows: position [1184,976] → within range
- Loop Rows: position [1440,976] → within range

SN: Build Payload (position [1696,816], width 360, height 336) → covers x:1696-2056, y:816-1152
- Build Payload: position [1792,992] → within range

SN: Jira API (position [2080,816], width 324, height 336) → covers x:2080-2404, y:816-1152
- Jira: Add Worklog: position [2208,992] → within range

SN: Handle Response + Update Sheet (position [2432,816], width 628, height 428) → covers x:2432-3060, y:816-1244
- Handle Response: position [2512,992] → within range
- Edit Fields: position [2752,992] → within range
- Update Sheet: position [2928,1088] → within range

SN: Summary + Email (position [1696,432], width 1016, height 360) → covers x:1696-2712, y:432-792
- Build Summary: position [1776,608] → within range
- AI Summary: position [2064,608] → within range
- Email: Final Report: position [2480,608] → within range

Now let me write the full document.### 1. Workflow Overview

This workflow automates the nightly creation of Jira worklogs based on rows maintained in a Google Sheet. It is designed for teams or individuals who track their daily Jira time entries in a spreadsheet and want those entries posted to Jira automatically, without manual intervention.

**Logical Blocks:**

| Block | Purpose |
|---|---|
| **1.1 Scheduled Trigger & Sheet Read** | Fires every night at 22:00, reads all rows from a "daily jira logs" Google Sheet. |
| **1.2 Filter & Gate** | Keeps only rows where `status = pending` and `date ≤ today`; if none exist, sends a "nothing to do" email and stops. |
| **1.3 Row-by-Row Loop** | Iterates over each qualifying row one at a time via `Split In Batches`. |
| **1.4 Payload Construction** | Normalises date, time, and text fields into a Jira-compatible worklog payload. |
| **1.5 Jira API Call** | POSTs a worklog entry to Jira Cloud REST API v3 with `continueOnFail` so a single failure does not break the loop. |
| **1.6 Response Handling & Sheet Update** | Interprets the Jira response, marks the row "Completed" or keeps it "Pending" with an incremented retry count, then writes the result back to the Google Sheet and loops back. |
| **1.7 Summary, AI Polish & Email Report** | After the loop ends, aggregates all results, asks GPT-4o-mini to write a friendly 2–3 sentence update, and emails an HTML report. |

---

### 2. Block-by-Block Analysis

---

#### Block 1.1 — Scheduled Trigger & Sheet Read

**Overview:**  
A cron-based schedule starts the workflow at 22:00 every night. The first action reads the entire "Sheet1" from the "daily jira logs" Google Sheet.

**Nodes Involved:**  
- Every Night 10 PM  
- Read Log Sheet

**Node Details:**

| Node | Type & Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|---|---|---|---|---|---|---|
| Every Night 10 PM | `n8n-nodes-base.scheduleTrigger` — Entry point | Cron expression: `0 0 22 * * *` (22:00 daily) | — | None | Triggers Read Log Sheet | n8n timezone must match the desired local time; misconfiguration leads to off-by-one-hour issues (DST). |
| Read Log Sheet | `n8n-nodes-base.googleSheets` (v4.5) — Reads spreadsheet data | Operation: Read rows; Document: "daily jira logs" (`1zj0LlOO4f0pOqmnvCocHMdibKVzyBglKcoCDHujjhYc`); Sheet: "Sheet1" (`gid=0`) | — | Every Night 10 PM | Filter Pending + Today | OAuth2 token expiry or revocation; empty sheet returns zero items; sheet must have a header row with expected column names (`date`, `status`, `ticket_id`, `log_time`, `started_at`, `log_text`, `retry_count`, `row_number`). |

---

#### Block 1.2 — Filter & Gate

**Overview:**  
A Code node filters rows to keep only those with `status = pending` and `date ≤ today`. An IF node then decides whether any work remains; if not, a Gmail node sends a no-op notification and the workflow ends.

**Nodes Involved:**  
- Filter Pending + Today  
- Anything To Process?  
- Email: Nothing To Do

**Node Details:**

| Node | Type & Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|---|---|---|---|---|---|---|
| Filter Pending + Today | `n8n-nodes-base.code` (v2) — Custom JS filter | Computes `todayStr` as `YYYY-MM-DD`. Attaches `sheet_row` (index + 2, because Google Sheets is 1-indexed with a header). Returns `{ nothingToDo: true/false, todayStr, rows: []/pending[] }` | `rowStatus === 'pending' && rowDate <= todayStr` | Read Log Sheet | Anything To Process? | Rows with blank or malformed `date` are excluded; case-insensitive match on `status`; `sheet_row` is computed from input order, not from any `row_number` column (important if the sheet is filtered/sorted differently). |
| Anything To Process? | `n8n-nodes-base.if` (v2) — Boolean branch | Condition: `$json.nothingToDo` equals `true` (strict boolean) | `={{ $json.nothingToDo }}` | Filter Pending + Today | **true →** Email: Nothing To Do; **false →** Split Into Rows | If the Code node returns an unexpected shape, the IF node may fall through both branches or error. |
| Email: Nothing To Do | `n8n-nodes-base.gmail` (v2.2) — Sends no-op email | Recipient: `your@email.com`; Subject: "Jira Worklog — No Rows Today"; HTML body references `$json.todayStr` | `{{ $json.todayStr }}` | Anything To Process? (true branch) | None (workflow ends) | Gmail OAuth2 token must be valid; hard-coded recipient must be updated before use. |

---

#### Block 1.3 — Row-by-Row Loop

**Overview:**  
The filtered rows are split into individual items, then processed one at a time using `Split In Batches`. Output 0 (done) proceeds to the summary; output 1 (next item) passes to payload construction. After the sheet update, execution loops back to `Loop Rows`.

**Nodes Involved:**  
- Split Into Rows  
- Loop Rows

**Node Details:**

| Node | Type & Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|---|---|---|---|---|---|---|
| Split Into Rows | `n8n-nodes-base.code` (v2) — Explodes array into items | Reads `d.rows` from input; initialises `__loopResults = []` in global static data; returns each row as a separate item | `allSessions.__loopResults = []` | Anything To Process? (false branch) | Loop Rows | If `rows` is empty despite `nothingToDo = false`, returns zero items and the loop never fires. |
| Loop Rows | `n8n-nodes-base.splitInBatches` (v3) — Batches of 1 | Default batch size 1; Output 0 = done, Output 1 = next item | — | Split Into Rows; also receives feedback from Update Sheet | **Output 0 →** Build Summary; **Output 1 →** Build Payload | If any node after Output 1 errors with `continueOnFail = false`, the loop halts prematurely. |

---

#### Block 1.4 — Payload Construction

**Overview:**  
Normalises date, start time, and time-spent values into Jira's expected formats. Stores current row data in global static data so it survives the subsequent HTTP call (whose response overwrites the item payload).

**Nodes Involved:**  
- Build Payload

**Node Details:**

| Node | Type & Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|---|---|---|---|---|---|---|
| Build Payload | `n8n-nodes-base.code` (v2) — Data transformation | Defaults `startedISO` to `{date}T09:00:00.000+0000`. If `started_at` provided, attempts two parse strategies. Normalises `log_time`: accepts `1h30m`, `45m`, or decimal hours (`1.5` → `1h30m`). Fallback `timeSpent = '1h'`. Stores `{ ticket_id, date, timeSpent, log_text, retry_count, row_number }` in `allSessions.__currentRow` | `allSessions.__currentRow`, `jiraIssueKey`, `startedISO`, `timeSpent`, `logComment` | Loop Rows (Output 1) | Jira: Add Worklog | Invalid `started_at` strings silently fall back to 09:00 UTC; `log_time` not matching any pattern defaults to `1h`; `row_number` may be undefined if the sheet doesn't contain it, so `sheet_row` from Filter node is used as fallback. |

---

#### Block 1.5 — Jira API Call

**Overview:**  
Sends a POST request to Jira Cloud REST API 3 to create a worklog on the specified issue. Uses HTTP Basic Auth (email + API token) and is set to `continueOnFail` so that one failure does not abort the batch.

**Nodes Involved:**  
- Jira: Add Worklog

**Node Details:**

| Node | Type & Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|---|---|---|---|---|---|---|
| Jira: Add Worklog | `n8n-nodes-base.httpRequest` (v4.2) — Outbound HTTP | Method: POST; URL: `https://YOUR-DOMAIN.atlassian.net/rest/api/3/issue/{{ $json.jiraIssueKey }}/worklog`; Auth: HTTP Basic Auth (credential "Unnamed credential"); Body: JSON with `timeSpent`, `started`, `comment` (Atlassian Document format); Timeout: 15000 ms; `continueOnFail: true` | `$json.jiraIssueKey`, `$json.timeSpent`, `$json.startedISO`, `$json.logComment` | Build Payload | Handle Response | **Must replace `YOUR-DOMAIN`** with actual Atlassian domain; 401 if API token is wrong or user lacks worklog permission; 404 if issue key doesn't exist; 400 if `timeSpent` is malformed; Jira may rate-limit; on failure, the response body is passed forward (error details stored later). |

---

#### Block 1.6 — Response Handling & Sheet Update

**Overview:**  
Interprets the Jira response: presence of `id` or `self` indicates success. The row is marked "Completed" or left "Pending" with an incremented retry count and a truncated error message. The row is then written back to the Google Sheet (matched on `row_number`), and execution loops back for the next item.

**Nodes Involved:**  
- Handle Response  
- Edit Fields  
- Update Sheet

**Node Details:**

| Node | Type & Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|---|---|---|---|---|---|---|
| Handle Response | `n8n-nodes-base.code` (v2) — Result interpretation | Reads `allSessions.__currentRow` (populated by Build Payload). Determines `isSuccess` from `jiraResp.id || jiraResp.self`. Appends result object to `allSessions.__loopResults`. | `rowData`, `isSuccess`, `allSessions.__loopResults` | Jira: Add Worklog | Edit Fields | If static data is cleared between executions, `__currentRow` may be empty; Jira errors return JSON without `id`/`self`, which is treated as failure. |
| Edit Fields | `n8n-nodes-base.set` (v3.4) — Field shaping | Maps: `status`, `error_message`, `retry_count`, `ticket_id`, `sheet_row` | `$json.status`, `$json.error_message`, `$json.retry_count`, `$json.ticket_id`, `$json.sheet_row` | Handle Response | Update Sheet | No transformation logic; serves as a clean pass-through with explicit field names. |
| Update Sheet | `n8n-nodes-base.googleSheets` (v4.5) — Write-back | Operation: update; Matching column: `row_number`; Updates `status`, `ticket_id`, `retry_count`, `error_message`; Document/Sheet same as Read Log Sheet | `$json.status`, `$json.ticket_id`, `$json.sheet_row`, `$json.retry_count`, `$json.error_message` | Edit Fields | Loop Rows (loops back) | Matching by `row_number` ensures correct row even if `ticket_id` appears multiple times; if `row_number` is missing or 0, the update may target the wrong row or fail. OAuth2 credentials must remain valid across the entire loop duration. |

---

#### Block 1.7 — Summary, AI Polish & Email Report

**Overview:**  
After all rows are processed, the loop's Output 0 triggers Build Summary, which aggregates success/failure counts and total time from `__loopResults`. The summary is passed to GPT-4o-mini, which writes a concise friendly update. An HTML email is then sent with the full report.

**Nodes Involved:**  
- Build Summary  
- AI Summary  
- Email: Final Report

**Node Details:**

| Node | Type & Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases |
|---|---|---|---|---|---|---|
| Build Summary | `n8n-nodes-base.code` (v2) — Aggregation | Iterates `__loopResults`, counts successes/failures, parses `timeSpent` back to minutes, computes total time string. Clears `__loopResults` afterwards. | `allSessions.__loopResults`, `success`, `failed`, `totalStr`, `failedTickets`, `ticket_ids` | Loop Rows (Output 0) | AI Summary | If `__loopResults` is empty (no rows processed), `success = 0`, `failed = 0`, `totalStr = '0m'`; time parsing regex assumes `Xh Ym` format. |
| AI Summary | `@n8n/n8n-nodes-langchain.openAi` (v1.8) — LLM call | Model: `gpt-4o-mini`; Single user message combining success/fail counts, total time, failed tickets, ticket IDs; Prompt asks for 2–3 sentence friendly update. | `$json.success`, `$json.failed`, `$json.totalStr`, `$json.failedTickets`, `$json.ticket_ids` | Build Summary | Email: Final Report | OpenAI API key must be set; token limits or content policy may truncate; if the model returns no content, the email will have an empty AI section. |
| Email: Final Report | `n8n-nodes-base.gmail` (v2.2) — Notification | Recipient: `your@email.com`; Subject: "Jira Daily Worklog Report"; HTML body includes success/fail counts, total time, failed tickets list, and AI-generated message. References `$('Build Summary').first().json.*` and `$json.message.content` (from AI). | `$('Build Summary').first().json.today`, `.success`, `.failed`, `.totalStr`, `.failedTickets`; `$json.message.content` | AI Summary | None (workflow ends) | Same Gmail OAuth2 constraints; if AI node output schema changes, `$json.message.content` may break; HTML must be reviewed for XSS if ticket IDs contain special characters. |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every Night 10 PM | scheduleTrigger (v1.2) | Nightly cron entry point | — | Read Log Sheet | **① Trigger + Read + Filter** — Runs every night at **10 PM** (`0 0 22 * * *`). Reads all rows · keeps only: `status = pending`, `date ≤ today` (catches past missed rows). Assigns `sheet_row` for precise update later. |
| Read Log Sheet | googleSheets (v4.5) | Read all rows from Google Sheet | Every Night 10 PM | Filter Pending + Today | **① Trigger + Read + Filter** — Runs every night at **10 PM** (`0 0 22 * * *`). Reads all rows · keeps only: `status = pending`, `date ≤ today` (catches past missed rows). Assigns `sheet_row` for precise update later. |
| Filter Pending + Today | code (v2) | Filter rows for pending status and date ≤ today; attach sheet_row index | Read Log Sheet | Anything To Process? | **① Trigger + Read + Filter** — Runs every night at **10 PM** (`0 0 22 * * *`). Reads all rows · keeps only: `status = pending`, `date ≤ today` (catches past missed rows). Assigns `sheet_row` for precise update later. |
| Anything To Process? | if (v2) | Branch: no pending rows → email; pending rows → split | Filter Pending + Today | Email: Nothing To Do (true), Split Into Rows (false) | **① Trigger + Read + Filter** — Runs every night at **10 PM** (`0 0 22 * * *`). Reads all rows · keeps only: `status = pending`, `date ≤ today` (catches past missed rows). Assigns `sheet_row` for precise update later. |
| Email: Nothing To Do | gmail (v2.2) | Send "nothing to log today" email | Anything To Process? (true) | — | **② No Pending Rows** — Sends a "nothing to log" email and stops. |
| Split Into Rows | code (v2) | Convert filtered array into individual items; initialise loop results | Anything To Process? (false) | Loop Rows | **③ Loop — One Row at a Time** — Output 0 → done → Build Summary. Output 1 → next item → Build Payload. After sheet update, loops back for next row. |
| Loop Rows | splitInBatches (v3) | Process one row per iteration; output 0 = loop done, output 1 = next row | Split Into Rows; Update Sheet | Build Summary (output 0), Build Payload (output 1) | **③ Loop — One Row at a Time** — Output 0 → done → Build Summary. Output 1 → next item → Build Payload. After sheet update, loops back for next row. |
| Build Payload | code (v2) | Normalise date/time into Jira format; store current row in static data | Loop Rows (output 1) | Jira: Add Worklog | **④ Build Payload** — Uses `row.date` for timestamp (not today). Normalises `log_time` → Jira format (`1h30m`). Stores row in static data so it survives the HTTP call. |
| Jira: Add Worklog | httpRequest (v4.2) | POST worklog to Jira REST API v3 | Build Payload | Handle Response | **⑤ Jira API** — `POST /rest/api/3/issue/{ticket_id}/worklog`. Auth: HTTP Basic (email + API token). `continueOnFail: true` — one failure won't stop the loop. |
| Handle Response | code (v2) | Interpret Jira response; mark success or failure; push to loop results | Jira: Add Worklog | Edit Fields | **⑥ Handle Response + Update Sheet** — Reads row from static data (safe after HTTP call). ✅ Success → sets `status = Completed`. ❌ Fail → keeps `Pending`, increments `retry_count`, logs error. Matches by `row_number` — works even if ticket_id appears twice. |
| Edit Fields | set (v3.4) | Pass through shaped fields for sheet update | Handle Response | Update Sheet | **⑥ Handle Response + Update Sheet** — Reads row from static data (safe after HTTP call). ✅ Success → sets `status = Completed`. ❌ Fail → keeps `Pending`, increments `retry_count`, logs error. Matches by `row_number` — works even if ticket_id appears twice. |
| Update Sheet | googleSheets (v4.5) | Write status, retry count, error message back to sheet row | Edit Fields | Loop Rows | **⑥ Handle Response + Update Sheet** — Reads row from static data (safe after HTTP call). ✅ Success → sets `status = Completed`. ❌ Fail → keeps `Pending`, increments `retry_count`, logs error. Matches by `row_number` — works even if ticket_id appears twice. |
| Build Summary | code (v2) | Aggregate all loop results into success/fail/total time | Loop Rows (output 0) | AI Summary | **⑦ Summary + Email** — Reads all results from `__loopResults[]` static array. GPT-4o-mini writes a friendly 2–3 sentence team update. Sends HTML email: success count · fail count · total time · AI note. |
| AI Summary | @n8n/n8n-nodes-langchain.openAi (v1.8) | Generate friendly 2–3 sentence summary via GPT-4o-mini | Build Summary | Email: Final Report | **⑦ Summary + Email** — Reads all results from `__loopResults[]` static array. GPT-4o-mini writes a friendly 2–3 sentence team update. Sends HTML email: success count · fail count · total time · AI note. |
| Email: Final Report | gmail (v2.2) | Send HTML daily report email | AI Summary | — | **⑦ Summary + Email** — Reads all results from `__loopResults[]` static array. GPT-4o-mini writes a friendly 2–3 sentence team update. Sends HTML email: success count · fail count · total time · AI note. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Log Jira worklogs nightly from Google Sheets with Gmail and GPT-4o-mini".

2. **Add Schedule Trigger node**  
   - Type: `Schedule Trigger`  
   - Name: "Every Night 10 PM"  
   - Mode: Cron expression  
   - Expression: `0 0 22 * * *`  
   - Timezone: Verify it matches your desired timezone in n8n settings.

3. **Add Google Sheets — Read node**  
   - Type: `Google Sheets` (v4.5)  
   - Name: "Read Log Sheet"  
   - Operation: Read rows  
   - Document: Select or paste the Google Sheets document ID (`1zj0LlOO4f0pOqmnvCocHMdibKVzyBglKcoCDHujjhYc`)  
   - Sheet: "Sheet1" (`gid=0`)  
   - Credential: Create/select a **Google Sheets OAuth2** credential.  
   - Connect: "Every Night 10 PM" → "Read Log Sheet"

4. **Add Code node — Filter**  
   - Type: `Code` (v2)  
   - Name: "Filter Pending + Today"  
   - Language: JavaScript  
   - Paste the filter logic: compute `todayStr`, filter rows where `status === 'pending'` and `date <= todayStr`, assign `sheet_row = index + 2`, return `{ nothingToDo, todayStr, rows }`.  
   - Connect: "Read Log Sheet" → "Filter Pending + Today"

5. **Add IF node**  
   - Type: `IF` (v2)  
   - Name: "Anything To Process?"  
   - Condition: Boolean — `$json.nothingToDo` equals `true` (strict)  
   - Connect: "Filter Pending + Today" → "Anything To Process?"  
   - True branch → "Email: Nothing To Do"  
   - False branch → "Split Into Rows"

6. **Add Gmail node — Nothing To Do**  
   - Type: `Gmail` (v2.2)  
   - Name: "Email: Nothing To Do"  
   - Credential: Create/select **Gmail OAuth2** credential.  
   - Send To: `your@email.com` (replace before use)  
   - Subject: `Jira Worklog — No Rows Today`  
   - Message (HTML):  
     ```html
     <p>💤 <strong>Jira Worklog — {{ $json.todayStr }}</strong></p>
     <p>No pending rows found. Nothing to log today.</p>
     ```  
   - Connect: "Anything To Process?" (true) → "Email: Nothing To Do"

7. **Add Code node — Split Into Rows**  
   - Type: `Code` (v2)  
   - Name: "Split Into Rows"  
   - Logic: Read `rows` from input, initialise `__loopResults = []` in global static data, return each row as a separate item.  
   - Connect: "Anything To Process?" (false) → "Split Into Rows"

8. **Add Split In Batches node**  
   - Type: `Split In Batches` (v3)  
   - Name: "Loop Rows"  
   - Batch size: 1 (default)  
   - Connect: "Split Into Rows" → "Loop Rows"  
   - Output 0 → "Build Summary" (loop done)  
   - Output 1 → "Build Payload" (next row)

9. **Add Code node — Build Payload**  
   - Type: `Code` (v2)  
   - Name: "Build Payload"  
   - Logic: Normalise `date` + `started_at` to ISO string with `+0000` offset; parse `log_time` into Jira `timeSpent` format; default `timeSpent = '1h'`, `startedISO = {date}T09:00:00.000+0000`, `logComment = row.log_text || 'Daily worklog'`; store current row in `allSessions.__currentRow`.  
   - Connect: "Loop Rows" (output 1) → "Build Payload"

10. **Add HTTP Request node — Jira Add Worklog**  
    - Type: `HTTP Request` (v4.2)  
    - Name: "Jira: Add Worklog"  
    - Method: POST  
    - URL: `https://YOUR-DOMAIN.atlassian.net/rest/api/3/issue/{{ $json.jiraIssueKey }}/worklog` (replace `YOUR-DOMAIN`)  
    - Authentication: Generic Credential Type → HTTP Basic Auth  
    - Create/select **HTTP Basic Auth** credential: username = Jira account email, password = Jira API token  
    - Body: JSON, expression:  
      ```
      { "timeSpent": $json.timeSpent, "started": $json.startedISO, "comment": { "type": "doc", "version": 1, "content": [{ "type": "paragraph", "content": [{ "type": "text", "text": $json.logComment }] }] } }
      ```  
    - Options: Timeout 15000 ms  
    - **continueOnFail: true**  
    - Connect: "Build Payload" → "Jira: Add Worklog"

11. **Add Code node — Handle Response**  
    - Type: `Code` (v2)  
    - Name: "Handle Response"  
    - Logic: Read `allSessions.__currentRow`; determine `isSuccess` from `jiraResp.id || jiraResp.self`; build result object with `status`, `error_message` (truncated to 200 chars), `retry_count`, `jiraSuccess`, `ticket_id`, `timeSpent`, `sheet_row`; push to `allSessions.__loopResults`.  
    - Connect: "Jira: Add Worklog" → "Handle Response"

12. **Add Set node — Edit Fields**  
    - Type: `Set` (v3.4)  
    - Name: "Edit Fields"  
    - Assignments:  
      - `status` (string) = `{{ $json.status }}`  
      - `error_message` (string) = `{{ $json.error_message }}`  
      - `retry_count` (number) = `{{ $json.retry_count }}`  
      - `ticket_id` (string) = `{{ $json.ticket_id }}`  
      - `sheet_row` (number) = `{{ $json.sheet_row }}`  
    - Connect: "Handle Response" → "Edit Fields"

13. **Add Google Sheets — Update node**  
    - Type: `Google Sheets` (v4.5)  
    - Name: "Update Sheet"  
    - Operation: Update  
    - Same Document ID and Sheet as step 3  
    - Matching column: `row_number`  
    - Columns to update: `status`, `ticket_id`, `retry_count`, `error_message` (mapped from incoming fields)  
    - Credential: Same Google Sheets OAuth2 credential  
    - Connect: "Edit Fields" → "Update Sheet"  
    - Then connect: "Update Sheet" → "Loop Rows" (creates the loop-back)

14. **Add Code node — Build Summary**  
    - Type: `Code` (v2)  
    - Name: "Build Summary"  
    - Logic: Read `allSessions.__loopResults`; count successes/failures; parse `timeSpent` strings back to total minutes; compute `totalStr`; collect `failedTickets` and `ticket_ids`; clear `__loopResults`.  
    - Connect: "Loop Rows" (output 0) → "Build Summary"

15. **Add OpenAI node — AI Summary**  
    - Type: `OpenAI` (Langchain, v1.8)  
    - Name: "AI Summary"  
    - Credential: Create/select **OpenAI API** credential  
    - Model: `gpt-4o-mini`  
    - Messages: Single user message built from `$json.success`, `$json.failed`, `$json.totalStr`, `$json.failedTickets`, `$json.ticket_ids` — prompt asks for a friendly 2–3 sentence update.  
    - Connect: "Build Summary" → "AI Summary"

16. **Add Gmail node — Final Report**  
    - Type: `Gmail` (v2.2)  
    - Name: "Email: Final Report"  
    - Credential: Same Gmail OAuth2 credential as step 6  
    - Send To: `your@email.com` (replace before use)  
    - Subject: `Jira Daily Worklog Report`  
    - Message (HTML):  
      ```html
      <p>📅 <strong>Jira Daily Report — {{ $('Build Summary').first().json.today }}</strong></p>
      <p>✅ Success: <strong>{{ $('Build Summary').first().json.success }} ticket(s)</strong></p>
      <p>❌ Failed: <strong>{{ $('Build Summary').first().json.failed }} ticket(s)</strong></p>
      <p>⏳ Total: <strong>{{ $('Build Summary').first().json.totalStr }}</strong></p>
      {{ $('Build Summary').first().json.failed > 0 ? '<p>⚠️ Failed: <code>' + $('Build Summary').first().json.failedTickets + '</code></p>' : '' }}
      <p>🤖 {{ $json.message.content }}</p>
      ```  
    - Connect: "AI Summary" → "Email: Final Report"

17. **Credential checklist**  
    - **Google Sheets OAuth2** — authorise access to the spreadsheet.  
    - **Gmail OAuth2** — authorise sending emails from the chosen account.  
    - **HTTP Basic Auth (Jira)** — username = Atlassian account email, password = Jira API token (generate at https://id.atlassian.com/manage-profile/security/api-tokens).  
    - **OpenAI API** — API key with access to `gpt-4o-mini` model.

18. **Google Sheet structure requirement**  
    The sheet "Sheet1" in the "daily jira logs" document must contain at minimum these columns:  
    `ticket_id`, `date` (YYYY-MM-DD), `status` (pending/completed), `log_time` (e.g. "1h30m", "45m", "1.5"), `started_at` (e.g. "09:00"), `log_text`, `retry_count`, `row_number`, `error_message`. The header row is expected by the Read node.

19. **Hard-coded values to update before first run**  
    - `YOUR-DOMAIN.atlassian.net` in the Jira HTTP Request URL.  
    - `your@email.com` in both Gmail nodes.  
    - Verify the Google Sheet document ID matches your own (or re-select the sheet in both Google Sheets nodes).

20. **Activate the workflow**  
    After all nodes are connected and credentials are verified, click **Active** in n8n. The workflow will now run nightly at 22:00.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Jira Daily Worklog Automation** — Logs time to Jira every night from a Google Sheet. No manual entries needed. → Reads sheet → filters pending rows → posts to Jira API → updates status → sends email report. **Requires:** Google Sheets · Gmail · Jira Basic Auth · OpenAI API | Workflow overview sticky note |
| **⚙️ Setup** — 1. Set your **Google Sheet ID** in `Read Log Sheet` and `Update Sheet`. 2. Add your **Jira domain** to the URL in `Jira: Add Worklog`. 3. Set `sendTo` email in both Gmail nodes. 4. Credentials needed: Google Sheets OAuth2, Gmail OAuth2, HTTP Basic Auth (Jira email + API token), OpenAI API key. | Setup sticky note |
| Jira API tokens are generated at Atlassian account security settings: https://id.atlassian.com/manage-profile/security/api-tokens | Required for HTTP Basic Auth credential |
| The workflow uses `continueOnFail: true` on the Jira HTTP Request node so that a single failed worklog does not halt the entire batch — all rows are attempted and results are recorded individually. | Error resilience design |
| `sheet_row` is computed as `index + 2` (1-indexed sheet minus header row). If the sheet is re-sorted or filtered between reads, row positions may shift. The `row_number` column (if present) is used as the matching key in the Update operation to mitigate this. | Data integrity consideration |
| Global static data (`$getWorkflowStaticData('global')`) is used to persist `__currentRow` and `__loopResults` across the loop and HTTP call boundaries. This data is cleared at the end of each execution in the Build Summary node. | Architecture note |
| The `log_time` field accepts three formats: `XhYm` (e.g. `1h30m`), `Xm` (e.g. `45m`), or decimal hours (e.g. `1.5` → `1h30m`). Any other format defaults to `1h`. | Input format documentation |
| The `started_at` field accepts time-only strings like `09:00` or `14:30`. Invalid values silently fall back to `09:00:00 UTC` on the row's date. | Input format documentation |
| The AI Summary node uses the `gpt-4o-mini` model via the Langchain OpenAI integration. Ensure the connected OpenAI API key has sufficient quota and access to that model. | LLM integration note |