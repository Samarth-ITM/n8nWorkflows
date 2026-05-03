Track competitor pricing and features with BrowserAct and Google Sheets

https://n8nworkflows.xyz/workflows/track-competitor-pricing-and-features-with-browseract-and-google-sheets-13379


# Track competitor pricing and features with BrowserAct and Google Sheets

## 1. Workflow Overview

**Purpose:** This workflow monitors competitor pricing/feature pages weekly. It fetches a list of competitors from Google Sheets, scrapes current pricing-plan data with BrowserAct, compares it to the last stored scrape using an LLM, updates the “database” sheet for future baselines, generates an aggregated weekly report, archives it into a newly created weekly tab, and notifies a Slack channel on completion.

**Typical use cases**
- Competitive intelligence: track plan/price/feature changes over time.
- Weekly reporting to a team (Slack + Google Sheets archive).
- Maintaining a “current baseline” (Last_Scrape_Content/Date) per competitor.

### 1.1 Scheduling & Input Retrieval
Weekly schedule triggers the workflow, then reads competitor rows (name, URL, last scrape fields) from a Google Sheet.

### 1.2 Per-Competitor Loop: Live Scrape → AI Compare → Update Baseline
For each competitor row, BrowserAct scrapes fresh pricing/feature content, an AI agent compares it with historical content, then updates the same Google Sheet row with the new baseline fields (content/date/comparison).

### 1.3 Aggregation & Weekly Report Generation
After looping, the workflow re-reads loop results and aggregates them, then uses an AI agent to generate a single consolidated report text.

### 1.4 Archiving, Sheet Tab Creation & Notification
Creates a new weekly sheet tab (named by date), adds a header row, appends the consolidated report, then sends a Slack notification.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Competitor List Retrieval
**Overview:** Triggers weekly and loads competitor targets (including stored “last scrape” content) from the Google Sheets database tab.

**Nodes involved**
- **Weekly Trigger**
- **Fetch links & history**

#### Node: Weekly Trigger
- **Type / role:** `Schedule Trigger` — workflow entry point.
- **Configuration (interpreted):** Runs every week (`interval: weeks`).
- **Key data used later:** `$('Weekly Trigger').first().json["Readable date"]` (used to timestamp updates and to name the weekly archive tab).
- **Outputs:** To **Fetch links & history**.
- **Failure/edge cases:**
  - If instance timezone differs from expectations, “Readable date” may not match business timezone.
  - If schedule misconfigured, may not run as expected.

#### Node: Fetch links & history
- **Type / role:** `Google Sheets` — reads competitor records from the “Data” sheet.
- **Configuration (interpreted):**
  - Document: **AI Competitor Spy: Pricing & Feature Tracker**
  - Sheet/tab: **Data** (`gid=0`)
  - Operation is implicitly a “read/get many” style (not explicitly shown, but used as list input to the loop).
- **Outputs:** Rows to **Loop Over Items**.
- **Failure/edge cases:**
  - OAuth token expiration / permission issues.
  - Missing expected columns (at minimum: `Competitor Name`, `URL`, `Last_Scrape_Content`, `Last_Scrape_Date`, and a row identifier such as `row_number` used later for update matching).

---

### Block 2 — Per-Competitor Loop: Scrape → Compare → Update Database
**Overview:** Iterates through each competitor row, scrapes current pricing-plan data via BrowserAct, compares to historical data using an LLM (structured JSON output), and updates the original row in Google Sheets.

**Nodes involved**
- **Loop Over Items**
- **Extract page content**
- **OpenRouter Model**
- **Structured Output**
- **Analyze target pages**
- **Update Database**

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — controls iteration over competitor rows.
- **Configuration (interpreted):** Uses default batching options (batch size not explicitly set; n8n defaults apply).
- **Connections:**
  - Main output 0 → **Retrieve "loop" results** (a “done/continue” style path in this workflow)
  - Main output 1 → **Extract page content** (per-item processing path)
- **Edge cases:**
  - If batch size defaults to 1, execution is safe but slower.
  - If input is empty, downstream per-competitor nodes won’t run; aggregation/report may be empty.

#### Node: Extract page content
- **Type / role:** `BrowserAct` (community node `n8n-nodes-browseract.browserAct`) — runs a BrowserAct workflow/template to extract pricing-plan data from a URL.
- **Configuration (interpreted):**
  - Mode: `WORKFLOW`
  - BrowserAct Workflow ID: `75660998074542379`
  - Input mapping: `Tracking_Site` = current item URL (`={{ $json.URL }}`)
- **Inputs:** One competitor row at a time from **Loop Over Items**.
- **Outputs:** A BrowserAct response under `$json.output...` (used later as `$json.output.string`).
- **Version-specific requirements:**
  - Requires the BrowserAct node package and valid BrowserAct API credentials.
- **Failure/edge cases:**
  - Target website blocks bots / requires login / uses heavy anti-scraping.
  - BrowserAct workflow returns unexpected shape (missing `output.string`).
  - Timeout/latency for slow pages.

#### Node: OpenRouter Model
- **Type / role:** LangChain chat model connector `lmChatOpenRouter` — provides the LLM for per-competitor analysis.
- **Configuration (interpreted):**
  - Model: `google/gemini-2.5-flash`
- **Connections:**
  - `ai_languageModel` → **Analyze target pages**
  - `ai_languageModel` → **Structured Output** (so the parser can “auto-fix” with model help if needed)
- **Failure/edge cases:**
  - OpenRouter auth/rate limits.
  - Model output variability (mitigated by structured output parsing + strict system message).

#### Node: Structured Output
- **Type / role:** LangChain `Structured Output Parser` — enforces JSON schema for the per-competitor comparison output.
- **Configuration (interpreted):**
  - `autoFix: true` (attempts to repair non-valid JSON via LLM)
  - Schema example requires exactly:
    - `content` (string)
    - `comparison` (string)
- **Connections:** Supplies `ai_outputParser` into **Analyze target pages**.
- **Edge cases:**
  - If the model returns irreparable JSON, the node fails.
  - If content includes unescaped newline/quotes, parser may need auto-fix; if still invalid, workflow stops.

#### Node: Analyze target pages
- **Type / role:** LangChain `Agent` — compares current scrape vs historical data and emits strict JSON (`content`, `comparison`).
- **Configuration (interpreted):**
  - Prompt input text:
    - Current Data: `{{ $json.output.string }}` (from BrowserAct)
    - Historical Data: `{{ $('Loop Over Items').item.json.Last_Scrape_Content }}`
    - Last Scrape Date: `{{ $('Loop Over Items').item.json.Last_Scrape_Date }}`
  - System instructions enforce:
    - Output **raw JSON only**
    - Must include only keys: `content`, `comparison`
    - No markdown; escape newlines as `\\n`
    - “No historical data available…” phrase required when baseline missing
  - Uses:
    - **OpenRouter Model** as `ai_languageModel`
    - **Structured Output** as `ai_outputParser`
- **Inputs:** BrowserAct output (current scrape) plus loop item historical fields.
- **Outputs:** Parsed JSON available at `$json.output.content` and `$json.output.comparison` for the next node.
- **Failure/edge cases:**
  - Historical content may be empty, null, or extremely large (token limits).
  - BrowserAct output may not be valid JSON; the prompt assumes “JSON pricing plans”.
  - The agent may hallucinate fields if scrape is incomplete; structured schema reduces but doesn’t eliminate semantic errors.

#### Node: Update Database
- **Type / role:** `Google Sheets` — updates the competitor row with the new baseline fields.
- **Configuration (interpreted):**
  - Operation: **Update**
  - Document: **AI Competitor Spy: Pricing & Feature Tracker**
  - Sheet: **Data**
  - Matching column: `row_number` (used as a key to update the correct row)
  - Columns updated:
    - `Competitor Name` from loop item
    - `URL` from loop item
    - `Last_Scrape_Date` = weekly trigger readable date
    - `Last_Scrape_Content` = AI output `content`
    - `Comparison` = AI output `comparison`
    - `row_number` from loop item
- **Connections:** After update → **Loop Over Items** (to continue looping).
- **Failure/edge cases:**
  - If `row_number` is missing or wrong, updates may fail or overwrite wrong row.
  - If the sheet uses a different header name/case, mapping will break.
  - Google Sheets API quota/limits.

---

### Block 3 — Post-Loop Retrieval & Aggregation
**Overview:** Collects results after the loop path and aggregates them into a single object for report generation.

**Nodes involved**
- **Retrieve "loop" results**
- **Aggregate**

#### Node: Retrieve "loop" results
- **Type / role:** `Google Sheets` — reads the same “Data” sheet again (used here to feed aggregation).
- **Configuration (interpreted):**
  - Same document/sheet as earlier.
  - `executeOnce: true` (in n8n, this attempts to prevent repeated execution in loops; useful since this node is connected to the loop controller path).
- **Connections:** → **Aggregate**
- **Failure/edge cases:**
  - If it re-reads entire sheet, report includes all competitors regardless of whether they were processed successfully in this run.
  - Permissions/quota issues as with other Sheets nodes.

#### Node: Aggregate
- **Type / role:** `Aggregate` — merges multiple items into a single aggregated payload.
- **Configuration (interpreted):**
  - Mode: `aggregateAllItemData` (collect all incoming items into one structure).
- **Connections:** → **Analyze competitor data & generate report**
- **Edge cases:**
  - Large datasets can produce very large aggregated JSON (token pressure for the report LLM node).

---

### Block 4 — Weekly Consolidated Report Generation (LLM)
**Overview:** Uses an LLM to generate one structured plain-text report (inside JSON) across all competitors based on aggregated data.

**Nodes involved**
- **OpenRouter**
- **Structured Output Parser**
- **Analyze competitor data & generate report**

#### Node: OpenRouter
- **Type / role:** LangChain chat model connector `lmChatOpenRouter` — LLM for consolidated report.
- **Configuration (interpreted):**
  - Model: `google/gemini-2.5-pro`
- **Connections:**
  - `ai_languageModel` → **Analyze competitor data & generate report**
  - `ai_languageModel` → **Structured Output Parser**
- **Failure/edge cases:** Same as other OpenRouter model node (auth, quotas, variability).

#### Node: Structured Output Parser
- **Type / role:** LangChain `Structured Output Parser` — enforces `{ "text": "..." }`.
- **Configuration (interpreted):**
  - `autoFix: true`
  - Schema example contains only `text` string.
- **Connections:** Supplies `ai_outputParser` into **Analyze competitor data & generate report**.
- **Edge cases:** If the LLM outputs non-repairable JSON, run fails.

#### Node: Analyze competitor data & generate report
- **Type / role:** LangChain `Agent` — produces consolidated report text across competitors.
- **Configuration (interpreted):**
  - Input text: `Input data : {{ $json.data}}` (expects aggregated data is under `$json.data`)
  - System instructions enforce:
    - Output raw JSON only: `{ "text": "..." }`
    - No markdown, use `\\n` newlines, escape quotes
    - Structured report sections per competitor:
      - CURRENT PLAN ANALYSIS
      - COMPARISON AND CHANGE LOG
- **Inputs:** Aggregated dataset from **Aggregate**, with model+parser attached.
- **Outputs:** Parsed report at `$('Analyze competitor data & generate report').first().json.output.text`
- **Failure/edge cases:**
  - If aggregate structure is not `data` (depends on Aggregate node output format), prompt may not receive the intended input.
  - Token limits for large competitor lists or long plan texts.

---

### Block 5 — Archiving to a New Weekly Sheet Tab + Slack Notification
**Overview:** Creates a new tab named with the weekly date, inserts a header row, appends the consolidated report, and posts a Slack message.

**Nodes involved**
- **Create New Sheet**
- **Define Headers**
- **Add Headers**
- **Update records**
- **Notify on completion**

#### Node: Create New Sheet
- **Type / role:** `Google Sheets` — creates a new sheet/tab for this week’s report archive.
- **Configuration (interpreted):**
  - Operation: **Create**
  - Title: `={{ $('Weekly Trigger').first().json["Readable date"] }}`
  - Document: same competitor tracker spreadsheet
- **Connections:** → **Define Headers**
- **Failure/edge cases:**
  - If a sheet with that exact name already exists (e.g., rerun), creation may fail.
  - Permissions/quota.

#### Node: Define Headers
- **Type / role:** `Set` — prepares an item with a header field.
- **Configuration (interpreted):**
  - Sets string field `Comparative Reports` to empty string (used so append can create/ensure a column header).
- **Connections:** → **Add Headers**
- **Edge cases:** If the append node is configured differently, setting empty string could create blank row content.

#### Node: Add Headers
- **Type / role:** `Google Sheets` — appends a row, relying on auto-mapping to write headers/columns.
- **Configuration (interpreted):**
  - Operation: **Append**
  - Sheet ID: uses created sheet ID `={{ $('Create New Sheet').first().json.sheetId }}`
  - Mapping mode: auto-map input data
- **Connections:** → **Update records**
- **Failure/edge cases:**
  - Auto-mapping depends on keys present; if “Comparative Reports” is missing, header may not appear as desired.
  - Appending may create an extra blank data row if not handled carefully.

#### Node: Update records
- **Type / role:** `Google Sheets` — appends the consolidated report into the new weekly tab.
- **Configuration (interpreted):**
  - Operation: **Append**
  - Target sheet: created sheet ID
  - Writes field `Comparative Reports` with:
    - `={{ $('Analyze competitor data & generate report').first().json.output.text }}`
- **Connections:** → **Notify on completion**
- **Failure/edge cases:**
  - If the report text exceeds Google Sheets cell limits, append may fail or truncate.
  - If the prior “Add Headers” didn’t create the correct column, data may land in unexpected column.

#### Node: Notify on completion
- **Type / role:** `Slack` — sends completion notification.
- **Configuration (interpreted):**
  - Posts message: “The weekly updates to the competitor price and feature comparison are complete.”
  - Target: a specific channel (by ID)
- **Failure/edge cases:**
  - Slack token/channel permission issues.
  - Channel ID invalid or removed.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Trigger | Schedule Trigger | Weekly entry point | — | Fetch links & history | ### 🎯 Step 1: Target Retrieval\n\nThe workflow triggers weekly. It connects to a Google Sheet database to fetch a list of competitor URLs and their previously recorded data (Last Scrape Content) to establish a baseline for comparison. |
| Fetch links & history | Google Sheets | Load competitor list + history baseline | Weekly Trigger | Loop Over Items | ### 🎯 Step 1: Target Retrieval\n\nThe workflow triggers weekly. It connects to a Google Sheet database to fetch a list of competitor URLs and their previously recorded data (Last Scrape Content) to establish a baseline for comparison. |
| Loop Over Items | Split In Batches | Iterate competitors | Fetch links & history; Update Database | Retrieve "loop" results; Extract page content | ### 🕵️ Step 2: Live Scraping\n\nBrowserAct visits each competitor's pricing page in real-time. It extracts the current plan names, prices, and feature lists, ensuring the data is fresh and accurate.An AI agent compares the *live* scraped data against the *historical* data. It identifies specific changes (e.g., "Price increased by $5", "Feature X removed") and generates a concise status report for each competitor. |
| Extract page content | BrowserAct | Live scrape pricing page | Loop Over Items | Analyze target pages | ### 🕵️ Step 2: Live Scraping\n\nBrowserAct visits each competitor's pricing page in real-time. It extracts the current plan names, prices, and feature lists, ensuring the data is fresh and accurate.An AI agent compares the *live* scraped data against the *historical* data. It identifies specific changes (e.g., "Price increased by $5", "Feature X removed") and generates a concise status report for each competitor. |
| OpenRouter Model | OpenRouter Chat Model (LangChain) | LLM for per-competitor comparison | — | Analyze target pages; Structured Output | ### 🕵️ Step 2: Live Scraping\n\nBrowserAct visits each competitor's pricing page in real-time. It extracts the current plan names, prices, and feature lists, ensuring the data is fresh and accurate.An AI agent compares the *live* scraped data against the *historical* data. It identifies specific changes (e.g., "Price increased by $5", "Feature X removed") and generates a concise status report for each competitor. |
| Structured Output | Structured Output Parser (LangChain) | Enforce `{content, comparison}` JSON | OpenRouter Model | Analyze target pages | ### 🕵️ Step 2: Live Scraping\n\nBrowserAct visits each competitor's pricing page in real-time. It extracts the current plan names, prices, and feature lists, ensuring the data is fresh and accurate.An AI agent compares the *live* scraped data against the *historical* data. It identifies specific changes (e.g., "Price increased by $5", "Feature X removed") and generates a concise status report for each competitor. |
| Analyze target pages | LangChain Agent | Compare current scrape vs history | Extract page content; OpenRouter Model; Structured Output | Update Database | ### 🕵️ Step 2: Live Scraping\n\nBrowserAct visits each competitor's pricing page in real-time. It extracts the current plan names, prices, and feature lists, ensuring the data is fresh and accurate.An AI agent compares the *live* scraped data against the *historical* data. It identifies specific changes (e.g., "Price increased by $5", "Feature X removed") and generates a concise status report for each competitor. |
| Update Database | Google Sheets | Update baseline fields in Data sheet | Analyze target pages | Loop Over Items | ### 💾 Step 4: Archiving & Reporting\n\nThe workflow creates a new tab in the Google Sheet for the current week's report. It saves the AI's comparative analysis and updates the main database with the new "Last Scrape" content for the next run. |
| Retrieve "loop" results | Google Sheets | Re-read Data for aggregation | Loop Over Items | Aggregate |  |
| Aggregate | Aggregate | Merge all competitor rows into one payload | Retrieve "loop" results | Analyze competitor data & generate report |  |
| OpenRouter | OpenRouter Chat Model (LangChain) | LLM for consolidated report | — | Analyze competitor data & generate report; Structured Output Parser |  |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforce `{text}` JSON | OpenRouter | Analyze competitor data & generate report |  |
| Analyze competitor data & generate report | LangChain Agent | Build consolidated weekly report | Aggregate; OpenRouter; Structured Output Parser | Create New Sheet |  |
| Create New Sheet | Google Sheets | Create weekly report tab | Analyze competitor data & generate report | Define Headers | ### 💾 Step 4: Archiving & Reporting\n\nThe workflow creates a new tab in the Google Sheet for the current week's report. It saves the AI's comparative analysis and updates the main database with the new "Last Scrape" content for the next run. |
| Define Headers | Set | Prepare header field for new tab | Create New Sheet | Add Headers | ### 💾 Step 4: Archiving & Reporting\n\nThe workflow creates a new tab in the Google Sheet for the current week's report. It saves the AI's comparative analysis and updates the main database with the new "Last Scrape" content for the next run. |
| Add Headers | Google Sheets | Append header row to new tab | Define Headers | Update records | ### 💾 Step 4: Archiving & Reporting\n\nThe workflow creates a new tab in the Google Sheet for the current week's report. It saves the AI's comparative analysis and updates the main database with the new "Last Scrape" content for the next run. |
| Update records | Google Sheets | Append consolidated report text | Add Headers | Notify on completion | ### 💾 Step 4: Archiving & Reporting\n\nThe workflow creates a new tab in the Google Sheet for the current week's report. It saves the AI's comparative analysis and updates the main database with the new "Last Scrape" content for the next run. |
| Notify on completion | Slack | Post completion message | Update records | — | ### 💾 Step 4: Archiving & Reporting\n\nThe workflow creates a new tab in the Google Sheet for the current week's report. It saves the AI's comparative analysis and updates the main database with the new "Last Scrape" content for the next run. |
| Documentation | Sticky Note | In-canvas documentation | — | — | ## ⚡ AI Competitor Spy: Pricing & Feature Tracker\n\n**Summary:** This automation continuously monitors competitor websites for pricing and feature updates. It scrapes current data, compares it against historical records using AI, generates a detailed change report, and archives the findings in Google Sheets.\n\n### Requirements\n* **Credentials:** BrowserAct, OpenRouter, Google Sheets, Slack.\n* **Mandatory:** BrowserAct API (Template: **AI Competitor Spy: Pricing & Feature Tracker**)\n\n### How to Use\n1.  **Credentials:** Set up API keys for all services.\n2.  **BrowserAct Template:** Ensure the **AI Competitor Spy: Pricing & Feature Tracker** template is active.\n3.  **Google Sheet:** Prepare a sheet with columns for `Competitor Name`, `URL`, `Last_Scrape_Content`, and `Last_Scrape_Date`.\n4.  **Execution:** The workflow runs automatically on a weekly schedule.\n\n### Need Help?\n[How to Find Your BrowserAct API Key & Workflow ID](https://docs.browseract.com)\n[How to Connect n8n to BrowserAct](https://docs.browseract.com)\n[How to Use & Customize BrowserAct Templates](https://docs.browseract.com) |
| Step 1 Explanation | Sticky Note | In-canvas explanation | — | — | ### 🎯 Step 1: Target Retrieval\n\nThe workflow triggers weekly. It connects to a Google Sheet database to fetch a list of competitor URLs and their previously recorded data (Last Scrape Content) to establish a baseline for comparison. |
| Step 2 Explanation | Sticky Note | In-canvas explanation | — | — | ### 🕵️ Step 2: Live Scraping\n\nBrowserAct visits each competitor's pricing page in real-time. It extracts the current plan names, prices, and feature lists, ensuring the data is fresh and accurate.An AI agent compares the *live* scraped data against the *historical* data. It identifies specific changes (e.g., "Price increased by $5", "Feature X removed") and generates a concise status report for each competitor. |
| Step 4 Explanation | Sticky Note | In-canvas explanation | — | — | ### 💾 Step 4: Archiving & Reporting\n\nThe workflow creates a new tab in the Google Sheet for the current week's report. It saves the AI's comparative analysis and updates the main database with the new "Last Scrape" content for the next run. |
| Sticky Note | Sticky Note | In-canvas media link | — | — | @[youtube](yBGtCt4gIdA) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add credentials** (before configuring nodes that need them):
   - **Google Sheets OAuth2** credential with access to the target spreadsheet.
   - **BrowserAct API** credential (API key) and confirm you have access to the BrowserAct workflow/template used for scraping.
   - **OpenRouter API** credential (API key).
   - **Slack API** credential (bot/user token) with permission to post to the desired channel.

3. **Create the Google Sheet structure** (Spreadsheet: “AI Competitor Spy: Pricing & Feature Tracker”):
   - Tab named **Data**.
   - Columns (headers) should include at least:
     - `Competitor Name` (string)
     - `URL` (string)
     - `Last_Scrape_Content` (string/long text)
     - `Last_Scrape_Date` (string)
     - `Comparison` (string/long text)
     - `row_number` (number) used as update key  
       - In n8n’s Google Sheets node, `row_number` is often available as metadata depending on the read operation; ensure it is present in the rows you loop over.

4. **Add node: Weekly Trigger** (`Schedule Trigger`)
   - Set it to run **every week**.

5. **Add node: Fetch links & history** (`Google Sheets`)
   - Operation: read rows (get many) from the **Data** tab.
   - Select the spreadsheet document.
   - Connect: **Weekly Trigger → Fetch links & history**.

6. **Add node: Loop Over Items** (`Split In Batches`)
   - Keep defaults or set batch size (commonly 1–10).
   - Connect: **Fetch links & history → Loop Over Items**.

7. **Add node: Extract page content** (`BrowserAct`)
   - Mode/type: **WORKFLOW**
   - BrowserAct Workflow ID: `75660998074542379`
   - Map input `Tracking_Site` to the competitor URL with an expression:
     - `{{$json.URL}}`
   - Connect: **Loop Over Items (output 1) → Extract page content**.

8. **Add node: OpenRouter Model** (`OpenRouter Chat Model` / LangChain)
   - Model: `google/gemini-2.5-flash`

9. **Add node: Structured Output** (`Structured Output Parser` / LangChain)
   - Enable **Auto-fix**
   - Provide schema example with keys: `content`, `comparison`

10. **Add node: Analyze target pages** (`Agent` / LangChain)
    - Set “Prompt type” to define your own instructions.
    - Text input (expression) should include:
      - Current scrape: `{{$json.output.string}}`
      - Historical content: `{{$('Loop Over Items').item.json.Last_Scrape_Content}}`
      - Last scrape date: `{{$('Loop Over Items').item.json.Last_Scrape_Date}}`
    - System message: enforce **raw JSON** output with exactly `{content, comparison}`, escaping newlines.
    - Attach:
      - `ai_languageModel` input from **OpenRouter Model**
      - `ai_outputParser` input from **Structured Output**
    - Connect: **Extract page content → Analyze target pages**.

11. **Add node: Update Database** (`Google Sheets`)
    - Operation: **Update**
    - Sheet: **Data**
    - Matching column: `row_number`
    - Map fields:
      - `row_number` = `{{$('Loop Over Items').item.json.row_number}}`
      - `Competitor Name` = `{{$('Loop Over Items').item.json["Competitor Name"]}}`
      - `URL` = `{{$('Loop Over Items').item.json.URL}}`
      - `Last_Scrape_Date` = `{{$('Weekly Trigger').first().json["Readable date"]}}`
      - `Last_Scrape_Content` = `{{$json.output.content}}`
      - `Comparison` = `{{$json.output.comparison}}`
    - Connect: **Analyze target pages → Update Database**.
    - Connect loop continuation: **Update Database → Loop Over Items** (to process next competitor).

12. **Add node: Retrieve "loop" results** (`Google Sheets`)
    - Configure to read rows from the **Data** tab (same spreadsheet).
    - Enable **Execute once** (prevents repeat in loop context).
    - Connect: **Loop Over Items (output 0) → Retrieve "loop" results**.

13. **Add node: Aggregate** (`Aggregate`)
    - Set aggregation to combine all item data (aggregate all items).
    - Connect: **Retrieve "loop" results → Aggregate**.

14. **Add node: OpenRouter** (`OpenRouter Chat Model` / LangChain)
    - Model: `google/gemini-2.5-pro`

15. **Add node: Structured Output Parser** (`Structured Output Parser` / LangChain)
    - Enable **Auto-fix**
    - Schema example: `{ "text": "..." }`

16. **Add node: Analyze competitor data & generate report** (`Agent` / LangChain)
    - Text input: reference aggregated payload (ensure it matches the Aggregate output; in this workflow it’s `{{$json.data}}`).
    - System message: enforce raw JSON output with one key `text`, newlines as `\\n`, no markdown.
    - Attach:
      - `ai_languageModel` from **OpenRouter**
      - `ai_outputParser` from **Structured Output Parser**
    - Connect: **Aggregate → Analyze competitor data & generate report**.

17. **Add node: Create New Sheet** (`Google Sheets`)
    - Operation: **Create** (new tab)
    - Title: `{{$('Weekly Trigger').first().json["Readable date"]}}`
    - Connect: **Analyze competitor data & generate report → Create New Sheet**.

18. **Add node: Define Headers** (`Set`)
    - Add field `Comparative Reports` (string) with empty value.
    - Connect: **Create New Sheet → Define Headers**.

19. **Add node: Add Headers** (`Google Sheets`)
    - Operation: **Append**
    - Target sheet: use the created sheet ID (expression):
      - `{{$('Create New Sheet').first().json.sheetId}}`
    - Use auto-map input data.
    - Connect: **Define Headers → Add Headers**.

20. **Add node: Update records** (`Google Sheets`)
    - Operation: **Append**
    - Target sheet: created sheet ID (same expression as above)
    - Map `Comparative Reports` to:
      - `{{$('Analyze competitor data & generate report').first().json.output.text}}`
    - Connect: **Add Headers → Update records**.

21. **Add node: Notify on completion** (`Slack`)
    - Operation: send message to channel.
    - Message text: “The weekly updates to the competitor price and feature comparison are complete.”
    - Select channel by ID or name (ensure credential has access).
    - Connect: **Update records → Notify on completion**.

22. **Validate with a manual run**
    - Ensure the Data sheet rows have valid URLs and `row_number` resolves correctly.
    - Confirm BrowserAct returns `output.string` in the expected format.
    - Confirm the weekly tab creation doesn’t collide with an existing tab name (adjust naming if needed, e.g., include time or ISO week number).

**Referenced links from workflow notes**
- BrowserAct docs (API key/workflow IDs, connection, templates): https://docs.browseract.com
- YouTube reference embedded in canvas: `@[youtube](yBGtCt4gIdA)`