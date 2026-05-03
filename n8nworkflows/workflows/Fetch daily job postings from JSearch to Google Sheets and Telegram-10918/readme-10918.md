Fetch daily job postings from JSearch to Google Sheets and Telegram

https://n8nworkflows.xyz/workflows/fetch-daily-job-postings-from-jsearch-to-google-sheets-and-telegram-10918


# Fetch daily job postings from JSearch to Google Sheets and Telegram

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Daily Job Postings Fetcher by Query (JSearch → Google Sheets → Telegram)  
**Purpose:** Once per day, this workflow searches the JSearch API (via RapidAPI) for recent job postings matching a configurable query, deduplicates them against a Google Sheet (by job ID), appends only new jobs to the sheet, then sends a Telegram summary with the number of new jobs added and a link to the sheet.

**Typical use cases**
- Daily “job radar” for a role keyword (e.g., “Data Analyst”, “Product Manager”)
- Maintaining a clean job tracker in Google Sheets without duplicate entries
- Sending an automated daily digest to a Telegram chat/channel

### 1.1 Scheduling & Parallel Start
Runs daily at a fixed time and starts two branches in parallel: (a) load existing job IDs from the sheet, (b) build the search query.

### 1.2 Query Building & API Fetch
Defines the search keyword, URL-encodes it, calls JSearch search endpoint, and splits the response into individual job items.

### 1.3 Existing-ID Filtering & New Job Counting
Merges API jobs with existing sheet rows, filters out already-known job IDs, then counts how many new jobs remain.

### 1.4 Saving Loop (Google Sheets) & Reporting (Telegram)
Iterates through the new jobs, delays between writes to avoid rate limits, appends each job as a new row, then merges flow completion with the count and sends the Telegram report.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Parallel Start

**Overview:** Triggers once per day and launches two independent branches: fetching existing IDs from Google Sheets and preparing the API query string.

**Nodes involved:**
- Run Daily at 22:00
- Load Existing Job IDs
- Set Search Parameters

#### Node: Run Daily at 22:00
- **Type / role:** Schedule Trigger (`scheduleTrigger`) — workflow entry point.
- **Configuration:** Runs daily at **22:00** (no minute specified in node params; schedule trigger provides the execution time).
- **Outputs:** Sends execution to:
  - **Load Existing Job IDs**
  - **Set Search Parameters**
- **Edge cases / failures:**
  - Timezone behavior depends on n8n instance timezone configuration; pinned data shows “Asia/Yerevan (UTC+04:00)”.
  - If the instance timezone changes, the “22:00” run time may shift in local time.

#### Node: Load Existing Job IDs
- **Type / role:** Google Sheets node — reads existing stored jobs to deduplicate.
- **Configuration choices (interpreted):**
  - Uses Google Sheets OAuth2 credentials (“Hrayr’s Google Sheets”).
  - Document and Sheet are placeholders: `"[YOUR_GOOGLE_SHEET_HERE]"`.
  - `alwaysOutputData: true` ensures the node outputs something even if the sheet is empty (helps avoid downstream merge failures).
  - Operation is not explicitly shown in JSON; by default in this node it’s typically **Read / Get Many rows** depending on UI setup. The downstream code expects rows containing an `ID` column.
- **Inputs:** From **Run Daily at 22:00**.
- **Outputs:** To **Combine API Results with Existing IDs** (input 1 / index 0).
- **Edge cases / failures:**
  - OAuth token expiry/refresh issues.
  - Wrong Document ID / Sheet name causes “not found” or permission errors.
  - If the sheet headers do not include `ID`, downstream dedupe code may treat all jobs as new.

#### Node: Set Search Parameters
- **Type / role:** Set node — defines the search keyword(s).
- **Configuration choices:**
  - Raw JSON mode outputs:
    ```json
    { "search_query": "[YOUR_JOB_NAME_HERE]" }
    ```
- **Inputs:** From **Run Daily at 22:00**.
- **Outputs:** To **Build Search Query**.
- **Edge cases / failures:**
  - Empty `search_query` leads to an empty query string; API may return irrelevant results or errors depending on provider rules.

---

### Block 2 — Query Building & API Fetch

**Overview:** URL-encodes the user-provided search query, requests jobs from JSearch, and splits the returned array into per-job items.

**Nodes involved:**
- Build Search Query
- Fetch Jobs from JSearch API
- Split API Results

#### Node: Build Search Query
- **Type / role:** Code node — prepares URL-safe query parameter.
- **Configuration choices:**
  - JS code:
    - Reads `$json.search_query`
    - `encodeURIComponent(input)`
    - Returns `{ query }`
- **Inputs:** From **Set Search Parameters**.
- **Outputs:** To **Fetch Jobs from JSearch API**.
- **Edge cases / failures:**
  - If `search_query` is not a string, it is coerced by `|| ""` fallback; could produce unexpected queries.

#### Node: Fetch Jobs from JSearch API
- **Type / role:** HTTP Request node — calls RapidAPI JSearch endpoint.
- **Configuration choices:**
  - URL (expression):
    - `https://jsearch.p.rapidapi.com/search?query={{ $json.query }}&page=1&num_pages=10&date_posted=week`
  - Authentication:
    - `genericCredentialType` with `httpHeaderAuth`
    - Credential: “Header Auth account” (must include RapidAPI headers such as `X-RapidAPI-Key` and `X-RapidAPI-Host` depending on RapidAPI requirements).
- **Inputs:** From **Build Search Query**.
- **Outputs:** To **Split API Results**.
- **Edge cases / failures:**
  - 401/403 if RapidAPI key invalid or missing required headers.
  - 429 rate limiting from RapidAPI.
  - Provider may change response schema; downstream expects a top-level `data` array.
  - Network timeouts; consider setting retry/timeout options in node options if needed.

#### Node: Split API Results
- **Type / role:** Split Out (`splitOut`) — converts an array field into multiple items.
- **Configuration choices:**
  - `fieldToSplitOut: "data"`
  - `include: "allOtherFields"` retains other response fields alongside each split item.
- **Inputs:** From **Fetch Jobs from JSearch API**.
- **Outputs:** To **Combine API Results with Existing IDs** (input 2 / index 1).
- **Edge cases / failures:**
  - If API returns no `data` field or `data` is not an array, this node will error.
  - If `data` is empty, downstream logic should handle “no items” (but merge and code nodes need to tolerate it).

---

### Block 3 — Existing-ID Filtering & New Job Counting

**Overview:** Joins the stream of API job items with the loaded sheet rows, removes jobs whose IDs already exist in the sheet, and counts remaining new items.

**Nodes involved:**
- Combine API Results with Existing IDs
- Filter Out Existing Jobs
- Count New Jobs

#### Node: Combine API Results with Existing IDs
- **Type / role:** Merge node — synchronizes two inputs so the filter step can reference both.
- **Configuration choices:**
  - Mode: `chooseBranch`
  - `useDataOfInput: 2` meaning it passes through data from input 2 (the API jobs), while still making input 1 (sheet rows) available for referencing via `$items('Load Existing Job IDs')`.
- **Inputs:**
  - Input 1: **Load Existing Job IDs**
  - Input 2: **Split API Results**
- **Outputs:** To **Filter Out Existing Jobs**.
- **Edge cases / failures:**
  - If one branch produces no items, behavior depends on merge mode. `chooseBranch` generally forwards one side; ensure the “existing IDs” side still provides `$items()` results (the sheet node is set to always output data to help).
  - If Load Existing Job IDs fails, the workflow stops (unless error handling is added).

#### Node: Filter Out Existing Jobs
- **Type / role:** Code node (run once per item) — deduplicates based on ID.
- **Configuration choices / key variables:**
  - Collects existing IDs:
    - `const existingIds = $items('Load Existing Job IDs').map(i => i.json.ID);`
  - Current job ID:
    - `const jobId = job.data?.job_id;`
  - If exists: `return null;` to skip item.
  - Else: pass through current item.
- **Inputs:** From **Combine API Results with Existing IDs** (API job items).
- **Outputs:** To both:
  - **Count New Jobs**
  - **Loop Over New Jobs**
- **Edge cases / failures:**
  - If sheet rows use different column name than `ID` (e.g., `Id`, `job_id`), dedupe fails.
  - If API item structure differs (no `data.job_id`), `jobId` becomes `undefined`:
    - If `existingIds` includes `undefined` (possible if sheet has blank IDs), unintended skipping may occur.
    - Otherwise, undefined jobs may pass through and create bad rows.
  - Large sheets: `$items('Load Existing Job IDs')` can be large; may increase memory usage.

#### Node: Count New Jobs
- **Type / role:** Code node — aggregates count of items.
- **Configuration choices:**
  - Uses `$input.all()` to count incoming items.
  - Outputs a single item: `{ count: <number> }`
- **Inputs:** From **Filter Out Existing Jobs**.
- **Outputs:** To **Combine Count and Save Results** (input 2 / index 1).
- **Edge cases / failures:**
  - If upstream produced zero items, this node outputs `{count: 0}` only if it still executes. In n8n, a node with no input items typically does not run; to guarantee a “0” report, you’d need an alternate path or “always output data” pattern here (not configured).
  - If it doesn’t run, Telegram report may not send (depends on merge behavior).

---

### Block 4 — Saving Loop (Google Sheets) & Reporting (Telegram)

**Overview:** Processes each new job sequentially, delays slightly to reduce Google API throttling, appends rows to the sheet, then sends a Telegram summary.

**Nodes involved:**
- Loop Over New Jobs
- Delay Before Writing to Sheet
- Save Job to Google Sheet
- Combine Count and Save Results
- Send Telegram Report

#### Node: Loop Over New Jobs
- **Type / role:** Split in Batches — controls iteration over items (batch loop).
- **Configuration choices:**
  - Default options (batch size not explicitly set; default is typically 1).
  - Two outgoing connections:
    - One to **Delay Before Writing to Sheet** (write path)
    - One to **Combine Count and Save Results** (used as a “done”/flow signal path depending on how the loop completes)
- **Inputs:** From **Filter Out Existing Jobs**.
- **Outputs:**
  - To **Delay Before Writing to Sheet**
  - To **Combine Count and Save Results**
- **Edge cases / failures:**
  - If there are zero items, this node may not emit anything, which can prevent the downstream “completion” merge and Telegram report.
  - Default batch behavior can be confusing; ensure it is configured to iterate as intended (commonly batch size = 1, and it loops via connection back from the write node).

#### Node: Delay Before Writing to Sheet
- **Type / role:** Wait node — throttling guard for Google Sheets API.
- **Configuration choices:**
  - `amount: 2` (seconds by default in Wait node UI unless otherwise configured)
- **Inputs:** From **Loop Over New Jobs**.
- **Outputs:** To **Save Job to Google Sheet**.
- **Edge cases / failures:**
  - Wait increases total runtime proportional to new job count.
  - In high-volume runs, execution may exceed n8n execution time limits (on some hosting plans).

#### Node: Save Job to Google Sheet
- **Type / role:** Google Sheets node — appends a new row per new job.
- **Configuration choices:**
  - Operation: **Append**
  - Mapping mode: “define below” with explicit column mapping.
  - DocumentId and SheetName placeholders: `"[YOUR_GOOGLE_SHEET_HERE]"`
  - Writes many fields from `$('Loop Over New Jobs').item.json.data...`
  - Notable expressions:
    - `Fetched At: {{$now}}`
    - Date formatting:
      - `new Date(...job_posted_at_datetime_utc).toLocaleString('en-GB', { timeZone: 'Asia/Yerevan' })`
    - Two spreadsheet formulas are intended:
      - Apply Link: `==HYPERLINK("{{ ...job_apply_link }}","Apply now")`
      - Google Link: `==HYPERLINK("{{ ...job_google_link }}","View in Google")`
      - **Note:** These use `==` and `{{ }}` inside a string, which is atypical for n8n expressions and Google formula syntax (see edge cases below).
- **Inputs:** From **Delay Before Writing to Sheet**.
- **Outputs:** Back to **Loop Over New Jobs** (to continue batching loop).
- **Edge cases / failures:**
  - **Formula formatting risk:** Google Sheets formulas should usually start with a single `=` not `==`. Also, inside n8n expressions you typically do:
    - `=HYPERLINK("{{$json.data.job_apply_link}}","Apply now")`
    - not `{{ ... }}` templating.
  - Null/undefined fields from API can cause blank cells; that’s fine, but the Date conversion can yield “Invalid Date” if the timestamp is missing or malformed.
  - Rate limiting (429) or quota errors from Google Sheets API even with delay (especially on shared accounts).
  - If headers don’t match exactly (Status, ID, Job Title, etc.), mapping may fail or create unexpected columns.

#### Node: Combine Count and Save Results
- **Type / role:** Merge node — combines the “count” item with the “loop completion” signal before reporting.
- **Configuration choices:**
  - Mode: `chooseBranch`
  - `useDataOfInput: 2` (passes through input 2 data)
- **Inputs:**
  - Input 1 (index 0): from **Loop Over New Jobs**
  - Input 2 (index 1): from **Count New Jobs**
- **Outputs:** To **Send Telegram Report**.
- **Edge cases / failures:**
  - If **Loop Over New Jobs** emits nothing (e.g., 0 items), this merge may not run as expected depending on merge semantics.
  - If **Count New Jobs** does not run (no items), Telegram message may not have `count`.

#### Node: Send Telegram Report
- **Type / role:** Telegram node — sends summary message.
- **Configuration choices:**
  - `chatId: [YOUR_CHAT_ID_HERE]`
  - Message text (HTML parse mode):
    - `{{$json.count}} jobs fetched. | <a href="...">View</a> #Jobs_Wags`
  - `parse_mode: HTML`
  - `appendAttribution: false`
  - Hardcoded Google Sheets link in the message (not dynamically derived from the Document ID).
- **Inputs:** From **Combine Count and Save Results**.
- **Outputs:** End of workflow.
- **Edge cases / failures:**
  - Bot not admin in channel / no permission to post.
  - Wrong chat ID (user vs group vs channel IDs differ).
  - Telegram HTML parsing: malformed HTML causes message send failure.
  - If `count` is undefined, message will show “undefined jobs fetched.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Daily at 22:00 | Schedule Trigger | Daily trigger / entry point | — | Load Existing Job IDs; Set Search Parameters | ## How it works (runs daily, dedupes, writes to Sheets, Telegram summary). Setup steps: create sheet headers; add RapidAPI/Google/Telegram creds; update placeholders; enable workflow; adjust schedule time. |
| Load Existing Job IDs | Google Sheets | Load existing rows/IDs for dedupe | Run Daily at 22:00 | Combine API Results with Existing IDs | ### Existing-ID Filtering Section: Loads existing job IDs; matches; removes duplicates; counts new jobs. |
| Set Search Parameters | Set | Define job search keyword | Run Daily at 22:00 | Build Search Query | ### Query Building and Querying Section: Defines search text, encodes it; fetches job results; splits into items. |
| Build Search Query | Code | URL-encode query parameter | Set Search Parameters | Fetch Jobs from JSearch API | ### Query Building and Querying Section: Defines search text, encodes it; fetches job results; splits into items. |
| Fetch Jobs from JSearch API | HTTP Request | Call JSearch (RapidAPI) search endpoint | Build Search Query | Split API Results | ### Query Building and Querying Section: Defines search text, encodes it; fetches job results; splits into items. |
| Split API Results | Split Out | Split response `data[]` into job items | Fetch Jobs from JSearch API | Combine API Results with Existing IDs | ### Query Building and Querying Section: Defines search text, encodes it; fetches job results; splits into items. |
| Combine API Results with Existing IDs | Merge | Join branches for dedupe context | Load Existing Job IDs; Split API Results | Filter Out Existing Jobs | ### Existing-ID Filtering Section: Loads existing job IDs; matches; removes duplicates; counts new jobs. |
| Filter Out Existing Jobs | Code | Skip jobs already present in sheet | Combine API Results with Existing IDs | Count New Jobs; Loop Over New Jobs | ### Existing-ID Filtering Section: Loads existing job IDs; matches; removes duplicates; counts new jobs. |
| Count New Jobs | Code | Count remaining new items | Filter Out Existing Jobs | Combine Count and Save Results | ### Existing-ID Filtering Section: Loads existing job IDs; matches; removes duplicates; counts new jobs. |
| Loop Over New Jobs | Split in Batches | Iterate through new jobs | Filter Out Existing Jobs | Delay Before Writing to Sheet; Combine Count and Save Results | ### Saving & Reporting Section: Iterates; delays; writes to sheet; combines results; sends Telegram summary. |
| Delay Before Writing to Sheet | Wait | Throttle Google Sheets writes | Loop Over New Jobs | Save Job to Google Sheet | ### Saving & Reporting Section: Iterates; delays; writes to sheet; combines results; sends Telegram summary. |
| Save Job to Google Sheet | Google Sheets | Append each new job as a row | Delay Before Writing to Sheet | Loop Over New Jobs | ### Saving & Reporting Section: Iterates; delays; writes to sheet; combines results; sends Telegram summary. |
| Combine Count and Save Results | Merge | Combine loop completion with count | Loop Over New Jobs; Count New Jobs | Send Telegram Report | ### Saving & Reporting Section: Iterates; delays; writes to sheet; combines results; sends Telegram summary. |
| Send Telegram Report | Telegram | Send daily summary to chat | Combine Count and Save Results | — | ### Saving & Reporting Section: Iterates; delays; writes to sheet; combines results; sends Telegram summary. |
| Sticky Note1 | Sticky Note | Comment block (query + API) | — | — | ### Query Building and Querying Section: Defines search text, encodes it; fetches job results; splits into items. |
| Sticky Note2 | Sticky Note | Comment block (dedupe + count) | — | — | ### Existing-ID Filtering Section: Loads existing job IDs; matches; removes duplicates; counts new jobs. |
| Sticky Note3 | Sticky Note | Comment block (saving + reporting) | — | — | ### Saving & Reporting Section: Iterates; delays; writes to sheet; combines results; sends Telegram summary. |
| Sticky Note4 | Sticky Note | Global explanation + setup steps | — | — | ## How it works + Setup steps (sheet headers, creds, placeholders, enable workflow, schedule). |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   “Daily Job Postings Fetcher by Query (JSearch → Google Sheets → Telegram)”.

2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Name: “Run Daily at 22:00”
   - Set it to run **daily at 22:00** (ensure your n8n instance timezone is correct).

3. **Add “Set Search Parameters”**
   - Node: **Set**
   - Mode: **Raw JSON**
   - JSON:
     - `search_query`: your target keyword (e.g., `Data Analyst`)
   - Connect: **Run Daily at 22:00 → Set Search Parameters**

4. **Add “Build Search Query”**
   - Node: **Code**
   - Code (JS):
     - Read `search_query`, URL-encode it, output `{ query }`.
   - Connect: **Set Search Parameters → Build Search Query**

5. **Add “Fetch Jobs from JSearch API”**
   - Node: **HTTP Request**
   - Method: GET
   - URL (expression):  
     `https://jsearch.p.rapidapi.com/search?query={{ $json.query }}&page=1&num_pages=10&date_posted=week`
   - Authentication: **Header Auth** (Generic → HTTP Header Auth)
   - **Credentials:** create an HTTP Header Auth credential that includes RapidAPI-required headers, typically:
     - `X-RapidAPI-Key: <your key>`
     - `X-RapidAPI-Host: jsearch.p.rapidapi.com`
   - Connect: **Build Search Query → Fetch Jobs from JSearch API**

6. **Add “Split API Results”**
   - Node: **Split Out**
   - Field to split out: `data`
   - Include: “All other fields”
   - Connect: **Fetch Jobs from JSearch API → Split API Results**

7. **Add “Load Existing Job IDs”**
   - Node: **Google Sheets**
   - Credential: **Google Sheets OAuth2**
   - Select **Document** (Spreadsheet) and **Sheet**
   - Configure to **read rows** such that each output item has an `ID` field (matching your sheet header).
   - Enable **Always Output Data** on the node (important when sheet is empty).
   - Connect: **Run Daily at 22:00 → Load Existing Job IDs**

8. **Add “Combine API Results with Existing IDs”**
   - Node: **Merge**
   - Mode: **Choose Branch**
   - “Use data of input”: **Input 2**
   - Connect:
     - **Load Existing Job IDs → Merge (Input 1)**
     - **Split API Results → Merge (Input 2)**

9. **Add “Filter Out Existing Jobs”**
   - Node: **Code**
   - Mode: **Run once for each item**
   - Implement logic:
     - `existingIds = $items('Load Existing Job IDs').map(i => i.json.ID)`
     - `jobId = $input.item.json.data.job_id`
     - If included → `return null`, else return the item
   - Connect: **Merge → Filter Out Existing Jobs**

10. **Add “Count New Jobs”**
   - Node: **Code**
   - Code counts `$input.all().length` and returns a single item `{count: n}`.
   - Connect: **Filter Out Existing Jobs → Count New Jobs**

11. **Add “Loop Over New Jobs”**
   - Node: **Split In Batches**
   - Keep default batch settings (or set batch size = 1 explicitly).
   - Connect: **Filter Out Existing Jobs → Loop Over New Jobs**

12. **Add “Delay Before Writing to Sheet”**
   - Node: **Wait**
   - Amount: **2 seconds**
   - Connect: **Loop Over New Jobs → Delay Before Writing to Sheet**

13. **Add “Save Job to Google Sheet”**
   - Node: **Google Sheets**
   - Credential: same Google OAuth2 credential
   - Operation: **Append**
   - Document + Sheet: your tracker sheet
   - Map columns to job fields (ID, title, company, links, salary, etc.) from the current batch item.
   - Connect: **Delay Before Writing to Sheet → Save Job to Google Sheet**
   - Connect loopback: **Save Job to Google Sheet → Loop Over New Jobs** (so it continues to next batch)

14. **Add “Combine Count and Save Results”**
   - Node: **Merge**
   - Mode: **Choose Branch**
   - Use data of input: **Input 2**
   - Connect:
     - **Loop Over New Jobs → Combine Count and Save Results (Input 1)**
     - **Count New Jobs → Combine Count and Save Results (Input 2)**

15. **Add “Send Telegram Report”**
   - Node: **Telegram**
   - Credential: Telegram bot token credential
   - Chat ID: your target chat/channel/group ID
   - Parse mode: **HTML**
   - Text: include `{{$json.count}}` and a link to your Google Sheet.
   - Connect: **Combine Count and Save Results → Send Telegram Report**

16. **Create/verify your Google Sheet headers**
   - Ensure the sheet has the headers referenced by the mapping (including **ID**). The sticky note suggests:  
     `(Status, ID, Job Title, Company Name, Apply Link, Company Website, Source, Type, Direct Apply, Remote, Date Posted (UTC), Fetched At, Location, Google Link, Salary, Minimum, Maximum)`

17. **Replace placeholders**
   - `[YOUR_JOB_NAME_HERE]` in Set node
   - `[YOUR_GOOGLE_SHEET_HERE]` for Document ID + Sheet name selections
   - `[YOUR_CHAT_ID_HERE]` in Telegram node
   - Replace the hardcoded Google Sheets “View” link in Telegram text with your real sheet URL.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Query/API section defines search text, encodes it, fetches jobs, and splits results into items. | Sticky note: “Query Building and Querying Section” |
| Dedupe section loads existing job IDs from Sheets, removes duplicates, and counts new jobs. | Sticky note: “Existing-ID Filtering Section” |
| Saving section loops items, delays to respect Google limits, writes to Sheets, then sends Telegram summary. | Sticky note: “Saving & Reporting Section” |
| Setup steps: create sheet headers; add RapidAPI/Google/Telegram credentials; update placeholders; enable workflow; adjust schedule time. | Sticky note: “How it works / Setup steps” |
| Telegram message uses an HTML link and a hardcoded Google Sheets URL; consider making it dynamic to match the configured document. | Telegram node text contains a fixed `docs.google.com/spreadsheets/...` link |

