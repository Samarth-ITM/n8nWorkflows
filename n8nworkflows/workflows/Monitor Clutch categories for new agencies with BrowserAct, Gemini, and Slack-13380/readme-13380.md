Monitor Clutch categories for new agencies with BrowserAct, Gemini, and Slack

https://n8nworkflows.xyz/workflows/monitor-clutch-categories-for-new-agencies-with-browseract--gemini--and-slack-13380


# Monitor Clutch categories for new agencies with BrowserAct, Gemini, and Slack

## 1. Workflow Overview

**Workflow name:** Monitor Clutch categories for new agencies to Slack With BrowserAct and Gemini  
**Purpose:** On a weekly schedule, scrape a chosen Clutch.co category for listed agencies, normalize the scraped fields with an AI agent, store the cleaned results in Google Sheets, compare the “fresh scrape” against a historical database, and notify a Slack channel if new agencies are detected. Finally, it replaces the historical database with the latest extraction to prepare for the next weekly run.

**Primary use cases**
- Lead generation: detect new agencies entering a category (e.g., “Developers”).
- Market monitoring: track changing provider lists per category.
- Data hygiene: standardize messy scraped data before storing/reporting.

### 1.1 Trigger & preparation
Weekly trigger clears the “Second Extraction” sheet so the run starts with a clean staging area, then sets the target Clutch category link.

### 1.2 Scraping & itemization
BrowserAct runs an external scraping workflow/template against the target category URL and returns results as a JSON string, which is parsed and split into individual n8n items.

### 1.3 AI normalization & staging storage
For each scraped company item, an AI agent cleans/normalizes the fields into a strict schema, then records are upserted into the “Second Extraction” sheet (append-or-update by company name), with waits to reduce rate limiting.

### 1.4 Change detection (old vs. new)
The workflow reads both the historical database sheet (“Database / First Extraction”) and the new staging sheet (“Second Extraction”), aggregates each list into arrays, and uses an AI agent (Gemini via OpenRouter) to identify new entries and craft a Slack-ready message.

### 1.5 Notify & archive
It posts the Slack message, clears the historical sheet, then copies the latest extraction into the historical sheet (batched with waits) so the next run compares against the most recent baseline.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & staging reset (Market Scanning kickoff)
**Overview:** Starts weekly, clears the staging sheet, and sets the target Clutch category URL to monitor.  
**Nodes involved:** Weekly Trigger, Clear Database, Clutch Category Link

#### Node: Weekly Trigger
- **Type / role:** `Schedule Trigger` — workflow entry point.
- **Configuration (interpreted):** Runs every week (`interval: weeks`).
- **Connections:**  
  - **Out →** Clear Database
- **Failure / edge cases:** n8n instance timezone affects “weekly” timing; missed runs if instance is down.

#### Node: Clear Database
- **Type / role:** `Google Sheets` — clears staging sheet before writing fresh results.
- **Configuration:**  
  - Operation: **Clear**  
  - Spreadsheet: “The New Entrant Asset Finder”  
  - Sheet/tab: “Second Extraction”  
  - `keepFirstRow: true` (preserves headers)
- **Credentials:** Google Sheets OAuth2.
- **Connections:**  
  - **In ←** Weekly Trigger  
  - **Out →** Clutch Category Link
- **Failure / edge cases:** OAuth expiration; wrong sheet ID; clearing removes prior run’s data (intended but irreversible).

#### Node: Clutch Category Link
- **Type / role:** `Set` — defines the monitored category URL.
- **Configuration:** Sets `Target_Category_Link` to `https://clutch.co/developers`.
- **Connections:**  
  - **In ←** Clear Database  
  - **Out →** Scrape page funding data
- **Failure / edge cases:** Invalid/redirecting URL leads to scrape failure or empty results.

---

### Block 2 — BrowserAct scraping & parsing into items
**Overview:** Runs a BrowserAct scraping workflow using the target URL, then parses the returned JSON string and splits it into one item per company.  
**Nodes involved:** Scrape page funding data, Splitting Items, Loop Over Items

#### Node: Scrape page funding data
- **Type / role:** `browserAct` — executes a BrowserAct workflow (remote browser automation/scrape).
- **Configuration:**  
  - Mode: **WORKFLOW**
  - BrowserAct workflowId: `76190590932330027`
  - Input mapping: `Clutch_Target_Category` ← `{{$json.Target_Category_Link}}`
  - Mapping mode: defineBelow (explicit schema with `input-Clutch_Target_Category`)
- **Credentials:** BrowserAct API.
- **Connections:**  
  - **In ←** Clutch Category Link  
  - **Out →** Splitting Items
- **Failure / edge cases:** BrowserAct quota/timeouts; Cloudflare/anti-bot; page layout changes; BrowserAct workflow output shape changes (breaks downstream parsing path).

#### Node: Splitting Items
- **Type / role:** `Code` — parses BrowserAct output JSON string and emits multiple items.
- **Configuration (key logic):**
  - Reads: `$input.first().json.output.string`
  - Throws error if missing/empty.
  - `JSON.parse(jsonString)`; must be an array.
  - Returns `[{json: item}, ...]` to create one n8n item per company.
- **Connections:**  
  - **In ←** Scrape page funding data  
  - **Out →** Loop Over Items
- **Failure / edge cases:**
  - Path mismatch (`output.string` missing) → hard error.
  - Malformed JSON → parse error.
  - Parsed value not an array → hard error.

#### Node: Loop Over Items
- **Type / role:** `SplitInBatches` — iterates over scraped company items for processing.
- **Configuration:** default batch options (not specified).
- **Connections:**  
  - **In ←** Splitting Items (first run also fans out to sheet reads; see below)  
  - **Out (batch) →** Analyze the extracted funds for each company  
  - **Out (done) →** (first output also triggers sheet reads on the first cycle due to wiring)
- **Important wiring behavior:** This node has two outgoing “main” paths:
  1) **Output index 0** goes to *Get old extraction data* and *Get fresh extracted data* (parallel reads).  
  2) **Output index 1** goes to *Analyze the extracted funds for each company* (per-item processing).
- **Failure / edge cases:** Misconfigured batch size could slow runs; if upstream yields 0 items, the per-item path won’t run (but the sheet-read path may still depend on execution behavior).

---

### Block 3 — AI cleaning/normalization & upsert to “Second Extraction”
**Overview:** For each scraped company record, an AI agent standardizes fields (prices, rates, location, description), enforces a strict schema, waits to reduce rate limiting, then upserts into Google Sheets.  
**Nodes involved:** Analyze the extracted funds for each company, OpenRouter Chat Model, Structured Output Parser, Prevent rate limits, Upsert new records

#### Node: OpenRouter Chat Model
- **Type / role:** `LangChain Chat Model (OpenRouter)` — LLM provider for the cleaning agent.
- **Configuration:** model not explicitly set here (uses OpenRouter default for credential/node unless configured elsewhere); options empty.
- **Credentials:** OpenRouter API.
- **Connections (AI ports):**  
  - **ai_languageModel →** Analyze the extracted funds for each company  
  - **ai_languageModel →** Structured Output Parser (provides model context)
- **Failure / edge cases:** OpenRouter auth errors; provider downtime; model default may change over time.

#### Node: Structured Output Parser
- **Type / role:** `LangChain Structured Output Parser` — validates/coerces LLM output to schema.
- **Configuration:**
  - `autoFix: true` (attempts to repair minor JSON/schema issues)
  - Schema example fields:
    - `company_name`, `website_url`, `min_project_value_usd`, `hourly_rate_low`, `hourly_rate_high`, `employees_range`, `city`, `country`, `short_description`
- **Connections (AI ports):**  
  - **ai_outputParser →** Analyze the extracted funds for each company
- **Failure / edge cases:** If model output is too malformed, autoFix may fail; schema drift will break downstream mapping (`$json.output.*`).

#### Node: Analyze the extracted funds for each company
- **Type / role:** `LangChain Agent` — cleans and reformats a single scraped company record.
- **Configuration:**
  - Input text: `={{ $json }}` (passes full current item JSON)
  - System message enforces:
    - URL cleaning (keep clutch redirect; strip tracking params)
    - Min project size normalization (`Undisclosed` → 0; “$10,000+” → 10000)
    - Hourly rate splitting into low/high with special cases (`< $25` etc.)
    - Location splitting into city/country
    - Description whitespace cleanup
  - `hasOutputParser: true` using Structured Output Parser
- **Connections:**  
  - **In ←** Loop Over Items (batch output)  
  - **Out →** Prevent rate limits
- **Failure / edge cases:** LLM hallucination risk (e.g., guessing domains against rules); inconsistent input keys from scraper (system prompt expects specific field names); token limits for very long descriptions.

#### Node: Prevent rate limits
- **Type / role:** `Wait` — throttling between LLM and Sheets writes.
- **Configuration:** Wait **3** (unit in n8n Wait is typically seconds unless configured otherwise in node UI).
- **Connections:**  
  - **In ←** Analyze the extracted funds for each company  
  - **Out →** Upsert new records
- **Failure / edge cases:** Extends runtime linearly with number of companies.

#### Node: Upsert new records
- **Type / role:** `Google Sheets` — writes cleaned item to staging sheet.
- **Configuration:**
  - Operation: **Append or Update**
  - Spreadsheet: “The New Entrant Asset Finder”
  - Sheet: “Second Extraction”
  - Matching column: `company_name`
  - Column mappings from agent output:
    - `company_name: {{$json.output.company_name}}`
    - `website_url: {{$json.output.website_url}}`
    - `min_project_value_usd: {{$json.output.min_project_value_usd}}`
    - `hourly_rate_low/high`, `employees_range`, `city`, `country`, `short_description`
- **Connections:**  
  - **In ←** Prevent rate limits  
  - **Out →** Loop Over Items (to continue batching)
- **Failure / edge cases:** If `company_name` duplicates or changes spelling, updates may overwrite; schema mismatch if output parser fails; Google API quota.

---

### Block 4 — Fetch old/new sheets and aggregate to arrays
**Overview:** Reads the historical database and the fresh staging sheet, then aggregates each into a single array to make AI comparison easy.  
**Nodes involved:** Get old extraction data, Aggregate, Get fresh extracted data, Aggregate1, Sync parallel paths

#### Node: Get old extraction data
- **Type / role:** `Google Sheets` — reads historical dataset.
- **Configuration:** Reads sheet “Database (First Extarction)” (gid=0). `executeOnce: true`.
- **Connections:**  
  - **In ←** Loop Over Items (output index 0 branch)  
  - **Out →** Aggregate
- **Failure / edge cases:** If “First Extraction” is empty (e.g., first run), comparison will treat all as new (expected).

#### Node: Aggregate
- **Type / role:** `Aggregate` — combines all rows into one field.
- **Configuration:** `aggregateAllItemData` into `Old_Data`.
- **Connections:**  
  - **In ←** Get old extraction data  
  - **Out →** Sync parallel paths (branch 0)
- **Failure / edge cases:** Very large sheets could create big payloads; may exceed LLM context later.

#### Node: Get fresh extracted data
- **Type / role:** `Google Sheets` — reads staging dataset.
- **Configuration:** Reads “Second Extraction”. `executeOnce: true`.
- **Connections:**  
  - **In ←** Loop Over Items (output index 0 branch)  
  - **Out →** Aggregate1
- **Failure / edge cases:** If scraping/cleaning failed and sheet is empty, change detector should report none (or all none).

#### Node: Aggregate1
- **Type / role:** `Aggregate` — combines all staging rows.
- **Configuration:** `aggregateAllItemData` into `New_Data`.
- **Connections:**  
  - **In ←** Get fresh extracted data  
  - **Out →** Sync parallel paths (branch 1)

#### Node: Sync parallel paths
- **Type / role:** `Merge` — synchronization point to ensure both aggregates are available.
- **Configuration:** `mode: chooseBranch` (used to join two branches; in practice ensures downstream runs when inputs arrive).
- **Connections:**  
  - **In ←** Aggregate (index 0), Aggregate1 (index 1)  
  - **Out →** Detect data changes
- **Failure / edge cases:** With chooseBranch, output selection can be confusing; if one branch doesn’t execute, downstream may miss data.

---

### Block 5 — AI change detection & Slack message creation
**Overview:** Uses Gemini (via OpenRouter) to compare aggregated old vs new arrays, identify unique companies, and format a Slack message plus a structured list of leads.  
**Nodes involved:** Detect data changes, OpenRouter Chat Model1, Structured Output Parser1, Notify team

#### Node: OpenRouter Chat Model1
- **Type / role:** `LangChain Chat Model (OpenRouter)` — LLM for comparison/reporting.
- **Configuration:** Model explicitly set to `google/gemini-2.5-pro`.
- **Credentials:** OpenRouter API.
- **Connections (AI ports):**  
  - **ai_languageModel →** Detect data changes  
  - **ai_languageModel →** Structured Output Parser1
- **Failure / edge cases:** Same as other LLM node; additionally, Gemini output might be verbose without parser enforcement.

#### Node: Structured Output Parser1
- **Type / role:** `LangChain Structured Output Parser` — enforces JSON response with message + leads list.
- **Configuration:**
  - `autoFix: true`
  - Schema example:
    - `status`, `new_leads_found`, `count`, `slack_message_text`, `leads[] {name, website, match_reason}`
- **Connections:**  
  - **ai_outputParser →** Detect data changes
- **Failure / edge cases:** If the LLM can’t comply, downstream Slack message expression fails (`$json.output.slack_message_text`).

#### Node: Detect data changes
- **Type / role:** `LangChain Agent` — compares datasets and generates Slack-ready content.
- **Configuration (key expressions):**
  - Input text is a composed string:
    - `EXISTING_DB : {{ JSON.stringify($('Aggregate').first().json.Old_Data, null, 2) }}`
    - `FRESH_SCRAPE : {{ JSON.stringify($('Aggregate1').first().json.New_Data, null, 2) }}`
  - System message rules:
    - Determine uniqueness by Website URL **or** Company Name not present in existing DB.
    - Output strict JSON.
    - Slack formatting rules included (note: system prompt includes a rocket/emoji in the Slack text format).
  - `hasOutputParser: true` using Structured Output Parser1.
- **Connections:**  
  - **In ←** Sync parallel paths  
  - **Out →** Notify team
- **Failure / edge cases:** Large arrays may exceed context window; stringifying full sheets can be expensive; uniqueness logic depends on consistent column names/values.

#### Node: Notify team
- **Type / role:** `Slack` — posts report and lead list to a channel.
- **Configuration:**
  - Channel: `all-browseract-workflow-test` (channelId `C09KLV9DJSX`)
  - Message text expression:
    - `{{$json.output.slack_message_text}} ,\n{{$json.output.leads}}`
- **Connections:**  
  - **In ←** Detect data changes  
  - **Out →** Clear Old data
- **Failure / edge cases:** Slack auth/scopes; message length limits; app not in channel; leads array prints as JSON (may be less readable than formatted blocks).

---

### Block 6 — Archive: replace historical database with latest extraction
**Overview:** Clears the historical sheet, reads the staging sheet, then appends each row into the historical sheet in batches with waits to avoid API rate limits.  
**Nodes involved:** Clear Old data, Retrieve latest data, Loop Over Items1, Avoid rate limits, Replace old data with new

#### Node: Clear Old data
- **Type / role:** `Google Sheets` — clears historical baseline before repopulating.
- **Configuration:**  
  - Operation: **Clear**
  - Sheet: “Database (First Extarction)” (gid=0)
  - `keepFirstRow: true`
  - `executeOnce: true`
- **Connections:**  
  - **In ←** Notify team  
  - **Out →** Retrieve latest data
- **Failure / edge cases:** If this runs but repopulation fails, baseline becomes empty next run (causes “everything new” effect).

#### Node: Retrieve latest data
- **Type / role:** `Google Sheets` — reads “Second Extraction” to copy into baseline.
- **Configuration:** Reads staging sheet. `executeOnce: true`.
- **Connections:**  
  - **In ←** Clear Old data  
  - **Out →** Loop Over Items1

#### Node: Loop Over Items1
- **Type / role:** `SplitInBatches` — iterates through rows from “Second Extraction”.
- **Configuration:** default options.
- **Connections:**  
  - **In ←** Retrieve latest data  
  - **Out (batch) →** (none directly shown)  
  - **Out (done / second output) →** Avoid rate limits  
  - Additionally receives input back from Replace old data with new to continue looping.
- **Failure / edge cases:** The wiring suggests the node’s “continue” behavior is used; if miswired, it can loop incorrectly or not at all.

#### Node: Avoid rate limits
- **Type / role:** `Wait` — throttle before writing each row.
- **Configuration:** Wait **3**.
- **Connections:**  
  - **In ←** Loop Over Items1 (second output)  
  - **Out →** Replace old data with new

#### Node: Replace old data with new
- **Type / role:** `Google Sheets` — appends staging rows into historical sheet.
- **Configuration:**
  - Operation: **Append**
  - Sheet: “Database (First Extarction)” (gid=0)
  - Column mappings use current item fields directly:
    - `{{$json.company_name}}`, `{{$json.website_url}}`, etc.
  - No matching columns (pure append).
- **Connections:**  
  - **In ←** Avoid rate limits  
  - **Out →** Loop Over Items1 (to continue batching)
- **Failure / edge cases:** Duplicate rows can accumulate if Clear Old data didn’t run; schema mismatch if staging columns differ; Google quota.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Trigger | Schedule Trigger | Weekly workflow entry point | — | Clear Database | ### 🕵️ Step 1: Market Scanning |
| Clear Database | Google Sheets | Clear “Second Extraction” staging sheet | Weekly Trigger | Clutch Category Link | ### 🕵️ Step 1: Market Scanning |
| Clutch Category Link | Set | Define target Clutch category URL | Clear Database | Scrape page funding data | ### 🕵️ Step 1: Market Scanning |
| Scrape page funding data | BrowserAct | Scrape Clutch category listing via BrowserAct workflow | Clutch Category Link | Splitting Items | ### 🕵️ Step 1: Market Scanning |
| Splitting Items | Code | Parse BrowserAct JSON string and emit per-company items | Scrape page funding data | Loop Over Items | ### 🕵️ Step 1: Market Scanning |
| Loop Over Items | SplitInBatches | Iterate over scraped companies; also triggers sheet reads | Splitting Items; Upsert new records | Analyze the extracted funds for each company; Get old extraction data; Get fresh extracted data | ### 🧹 Step 2: Data Cleaning |
| OpenRouter Chat Model | LangChain OpenRouter Chat Model | LLM provider for cleaning agent | — (AI connection) | Analyze the extracted funds for each company; Structured Output Parser (AI) | ### 🧹 Step 2: Data Cleaning |
| Structured Output Parser | LangChain Structured Output Parser | Enforce cleaned company schema | OpenRouter Chat Model (AI) | Analyze the extracted funds for each company (AI) | ### 🧹 Step 2: Data Cleaning |
| Analyze the extracted funds for each company | LangChain Agent | Clean/standardize one scraped company entry | Loop Over Items | Prevent rate limits | ### 🧹 Step 2: Data Cleaning |
| Prevent rate limits | Wait | Throttle before Sheets upsert | Analyze the extracted funds for each company | Upsert new records | ### 🧹 Step 2: Data Cleaning |
| Upsert new records | Google Sheets | Append-or-update cleaned rows into “Second Extraction” | Prevent rate limits | Loop Over Items | ### 🧹 Step 2: Data Cleaning |
| Get old extraction data | Google Sheets | Read historical baseline sheet | Loop Over Items | Aggregate | ### 🔍 Step 3: Change Detection |
| Aggregate | Aggregate | Combine baseline rows into `Old_Data` array | Get old extraction data | Sync parallel paths | ### 🔍 Step 3: Change Detection |
| Get fresh extracted data | Google Sheets | Read staging sheet for comparison | Loop Over Items | Aggregate1 | ### 🔍 Step 3: Change Detection |
| Aggregate1 | Aggregate | Combine staging rows into `New_Data` array | Get fresh extracted data | Sync parallel paths | ### 🔍 Step 3: Change Detection |
| Sync parallel paths | Merge | Join aggregates before comparison | Aggregate; Aggregate1 | Detect data changes | ### 🔍 Step 3: Change Detection |
| OpenRouter Chat Model1 | LangChain OpenRouter Chat Model | LLM provider for change detection (Gemini) | — (AI connection) | Detect data changes; Structured Output Parser1 (AI) | ### 🔍 Step 3: Change Detection |
| Structured Output Parser1 | LangChain Structured Output Parser | Enforce comparison output schema (message + leads) | OpenRouter Chat Model1 (AI) | Detect data changes (AI) | ### 🔍 Step 3: Change Detection |
| Detect data changes | LangChain Agent | Compare old vs new; generate Slack message | Sync parallel paths | Notify team | ### 📢 Step 4: Alert & Archive |
| Notify team | Slack | Post report to Slack channel | Detect data changes | Clear Old data | ### 📢 Step 4: Alert & Archive |
| Clear Old data | Google Sheets | Clear baseline sheet before repopulating | Notify team | Retrieve latest data | ### 📢 Step 4: Alert & Archive |
| Retrieve latest data | Google Sheets | Read staging sheet to copy into baseline | Clear Old data | Loop Over Items1 | ### 📢 Step 4: Alert & Archive |
| Loop Over Items1 | SplitInBatches | Batch-iterate staging rows for copying | Retrieve latest data; Replace old data with new | Avoid rate limits | ### 📢 Step 4: Alert & Archive |
| Avoid rate limits | Wait | Throttle before baseline append | Loop Over Items1 | Replace old data with new | ### 📢 Step 4: Alert & Archive |
| Replace old data with new | Google Sheets | Append rows into baseline sheet | Avoid rate limits | Loop Over Items1 | ### 📢 Step 4: Alert & Archive |
| Documentation | Sticky Note | Workflow context and requirements | — | — | ## ⚡ The New Entrant Asset Finder (includes links: https://docs.browseract.com) |
| Step 1 Explanation | Sticky Note | Explains market scanning block | — | — | ### 🕵️ Step 1: Market Scanning |
| Step 2 Explanation | Sticky Note | Explains AI cleaning block | — | — | ### 🧹 Step 2: Data Cleaning |
| Step 3 Explanation | Sticky Note | Explains change detection block | — | — | ### 🔍 Step 3: Change Detection |
| Step 4 Explanation | Sticky Note | Explains alert + archive block | — | — | ### 📢 Step 4: Alert & Archive |
| Sticky Note | Sticky Note | Video link | — | — | @[youtube](LTNHYvObTtc) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add **Schedule Trigger**.
   2. Set interval to **Every 1 week**.

2) **Clear staging (“Second Extraction”)**
   1. Add **Google Sheets** node named **Clear Database**.
   2. Operation: **Clear**.
   3. Select your spreadsheet (create one if needed) and the tab **Second Extraction**.
   4. Enable **Keep First Row**.
   5. Connect: **Weekly Trigger → Clear Database**.
   6. Configure **Google Sheets OAuth2** credentials.

3) **Set the target category URL**
   1. Add **Set** node named **Clutch Category Link**.
   2. Add a string field `Target_Category_Link` (example: `https://clutch.co/developers`).
   3. Connect: **Clear Database → Clutch Category Link**.

4) **Scrape with BrowserAct**
   1. Add **BrowserAct** node named **Scrape page funding data**.
   2. Select **Type: WORKFLOW** and choose your BrowserAct workflow (note the **Workflow ID**).
   3. Map an input like `Clutch_Target_Category` to `={{$json.Target_Category_Link}}`.
   4. Connect: **Clutch Category Link → Scrape page funding data**.
   5. Configure **BrowserAct API** credentials.

5) **Parse BrowserAct result and split into items**
   1. Add **Code** node named **Splitting Items**.
   2. Implement logic to:
      - Read `json.output.string`
      - `JSON.parse(...)` it (must be an array)
      - Return as individual n8n items.
   3. Connect: **Scrape page funding data → Splitting Items**.

6) **Batch over scraped companies**
   1. Add **Split In Batches** node named **Loop Over Items**.
   2. Connect: **Splitting Items → Loop Over Items**.

7) **AI cleaning agent (per company)**
   1. Add **OpenRouter Chat Model** node.
   2. Add **Structured Output Parser** node with the company schema (company_name, website_url, min_project_value_usd, hourly_rate_low/high, employees_range, city, country, short_description) and enable **Auto-fix**.
   3. Add **AI Agent** node named **Analyze the extracted funds for each company**:
      - Prompt input: `={{$json}}`
      - System message: rules for URL cleanup, numeric conversions, location splitting, description cleanup.
      - Enable “has output parser” and attach the structured parser.
      - Attach the OpenRouter model to the agent (AI connection).
   4. Add **Wait** node named **Prevent rate limits** (3 seconds).
   5. Add **Google Sheets** node named **Upsert new records**:
      - Operation: **Append or Update**
      - Sheet: **Second Extraction**
      - Matching column: `company_name`
      - Map columns from `{{$json.output.<field>}}`
   6. Wire the loop:
      - **Loop Over Items (batch output) → Analyze… → Prevent rate limits → Upsert new records → Loop Over Items** (to continue batches)

8) **Read baseline + staging for comparison**
   1. Add **Google Sheets** node **Get old extraction data** (read tab “Database (First Extarction)” / gid=0).
   2. Add **Google Sheets** node **Get fresh extracted data** (read tab “Second Extraction”).
   3. Connect both from **Loop Over Items** (commonly from the “done” output so it runs after processing; in this workflow it is wired from output index 0).
   4. Add **Aggregate** node:
      - Aggregate all items into `Old_Data`
   5. Add **Aggregate** node **Aggregate1**:
      - Aggregate all items into `New_Data`

9) **Sync and detect changes with Gemini**
   1. Add **Merge** node **Sync parallel paths** to join the two aggregates.
   2. Add **OpenRouter Chat Model** node **OpenRouter Chat Model1** with model `google/gemini-2.5-pro`.
   3. Add **Structured Output Parser1** enforcing:
      - `status`, `new_leads_found`, `count`, `slack_message_text`, `leads[]`
   4. Add **AI Agent** node **Detect data changes**:
      - Text input combines both arrays using expressions:
        - `EXISTING_DB : {{ JSON.stringify($('Aggregate').first().json.Old_Data, null, 2) }}`
        - `FRESH_SCRAPE : {{ JSON.stringify($('Aggregate1').first().json.New_Data, null, 2) }}`
      - System rules: compare lists, identify uniques by URL or name, produce Slack-formatted message in JSON.
      - Attach parser and model.
   5. Connect: **Sync parallel paths → Detect data changes**

10) **Send Slack notification**
   1. Add **Slack** node **Notify team**.
   2. Choose **Channel** and select your target channel.
   3. Text: `={{$json.output.slack_message_text}} ,\n{{$json.output.leads}}`
   4. Connect: **Detect data changes → Notify team**
   5. Configure **Slack OAuth/API** credentials.

11) **Archive: replace baseline with latest extraction**
   1. Add **Google Sheets** node **Clear Old data**:
      - Operation: **Clear**
      - Sheet: baseline tab (gid=0)
      - Keep first row.
   2. Add **Google Sheets** node **Retrieve latest data** to read “Second Extraction”.
   3. Add **Split In Batches** node **Loop Over Items1**.
   4. Add **Wait** node **Avoid rate limits** (3 seconds).
   5. Add **Google Sheets** node **Replace old data with new**:
      - Operation: **Append**
      - Sheet: baseline tab
      - Map columns from `{{$json.<field>}}` (note: these come directly from sheet rows, not AI output).
   6. Connect in order:
      - **Notify team → Clear Old data → Retrieve latest data → Loop Over Items1 → Avoid rate limits → Replace old data with new → Loop Over Items1** (to continue until done)

**Credential checklist**
- BrowserAct API credential (must have access to the configured BrowserAct workflow/template).
- OpenRouter API credential (used by both LLM nodes).
- Google Sheets OAuth2 credential (read/clear/write on the spreadsheet).
- Slack credential (post to chosen channel; ensure the app is in the channel).

**Sub-workflow setup (BrowserAct)**
- You must have a BrowserAct workflow with ID `76190590932330027` (or your own).
- It must accept an input like **Clutch_Target_Category** and return a JSON string at a predictable path (this workflow expects `output.string` that parses into an array of company objects).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “The New Entrant Asset Finder” summary, requirements, and usage notes | Included in the workflow’s **Documentation** sticky note |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| Video reference | @[youtube](LTNHYvObTtc) |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.