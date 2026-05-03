Track AI search rankings from Perplexity via BrowserAct to Google Sheets and Slack

https://n8nworkflows.xyz/workflows/track-ai-search-rankings-from-perplexity-via-browseract-to-google-sheets-and-slack-12351


# Track AI search rankings from Perplexity via BrowserAct to Google Sheets and Slack

## 1. Workflow Overview

**Purpose:**  
This workflow automates daily GEO (Generative Engine Optimization) rank/visibility tracking by (1) generating strategic AI-search queries for a target company, (2) running those queries in Perplexity via a BrowserAct automation, (3) logging raw results into a date-stamped Google Sheet tab, and (4) producing a graded Slack report (scorecard + recommendation) using a second AI analysis pass.

**Primary use cases:**
- Daily brand visibility monitoring on AI answer engines (e.g., Perplexity)
- Tracking how often a brand is recommended, compared, or correctly described
- Producing a consistent internal GEO performance update in Slack

### 1.1 Scheduling & Daily Sheet Initialization
Runs daily, creates a new Google Sheets tab named with the date, and prepares headers.

### 1.2 Company Context Retrieval & Query Strategy Generation (AI)
Loads the company profile (name + category) from a “Main Sheet”, then uses an LLM to generate 3 intent-based queries (discovery, comparison, validation) as strict JSON.

### 1.3 Query Execution via BrowserAct + Logging
Splits the 3 queries into items, loops through them, triggers BrowserAct to run the Perplexity search, and appends “Search” + “Result” rows into the daily tab.

### 1.4 Aggregation, AI Analysis & Slack Reporting
Reads back all stored rows from the daily tab, aggregates them, uses another LLM agent to turn them into a Slack-formatted “Daily GEO Report”, then posts to Slack.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Daily Sheet Initialization

**Overview:**  
Triggers once per day, creates a new daily sheet/tab in a fixed spreadsheet, then appends the header row used for subsequent logging.

**Nodes involved:**
- Scheduled Daily
- Create sheet
- Define Headers
- add headers

#### Node: Scheduled Daily
- **Type / role:** `Schedule Trigger` — entry point, time-based execution.
- **Configuration (interpreted):** Runs on a daily schedule (configured via `rule.interval`).
- **Outputs:** Sends a single item to **Create sheet**.
- **Edge cases / failures:**
  - Misconfigured schedule interval/timezone can cause unexpected run times.
  - If workflow is inactive (`active: false`), it will not run.

#### Node: Create sheet
- **Type / role:** `Google Sheets` (operation: Create) — creates a new sheet/tab in an existing spreadsheet.
- **Key configuration:**
  - **Spreadsheet:** “GEO Results & Rank Tracking” (fixed `documentId`)
  - **Title:** `={{ $json["Readable date"] }}`
- **Inputs:** From **Scheduled Daily**.
- **Outputs:** To **Define Headers**.
- **Important dependency:** The trigger item must contain `Readable date`. In the provided JSON, no upstream node sets it; you must ensure the schedule trigger provides it (it typically doesn’t) or add a Date/Set node to generate it.
- **Edge cases / failures:**
  - Google auth/permission errors (OAuth scope, revoked token).
  - Duplicate sheet name (if same “Readable date” already exists).
  - Expression failure if `Readable date` is missing.

#### Node: Define Headers
- **Type / role:** `Set` — prepares header fields for the append operation.
- **Configuration:**
  - Adds fields: `Search` (empty string), `Result` (empty string)
- **Inputs:** From **Create sheet**.
- **Outputs:** To **add headers**.
- **Edge cases:**
  - None significant; will always output those keys.

#### Node: add headers
- **Type / role:** `Google Sheets` (operation: Append) — appends the header row to the newly created daily sheet/tab.
- **Key configuration:**
  - **Sheet target (by ID):** `={{ $('Create sheet').first().json.sheetId }}`
  - **Columns mapping:** uses the incoming `Search` and `Result` fields (created by **Define Headers**)
- **Inputs:** From **Define Headers**.
- **Outputs:** To **Get Company data**.
- **Edge cases / failures:**
  - If `Create sheet` didn’t return `sheetId` (create failed), expression breaks.
  - If spreadsheet has protected sheets/ranges, append may fail.

---

### Block 2 — Company Context Retrieval & Query Strategy Generation (AI)

**Overview:**  
Loads company name and category from the “Main Sheet”, then uses Gemini (via OpenRouter) with a structured output parser to generate exactly three GEO queries.

**Nodes involved:**
- Get Company data
- OpenRouter
- Structured Output
- Generate Search Queries
- Split AI-Generated Questions

#### Node: Get Company data
- **Type / role:** `Google Sheets` (Read) — fetches company profile data from the main tab.
- **Key configuration:**
  - **Spreadsheet:** same fixed `documentId`
  - **Sheet:** `gid=0` (“Main Sheet”)
  - Uses “specifyRange” mode (range details not shown in parameters; relies on node UI config).
- **Inputs:** From **add headers**.
- **Outputs:** To **Generate Search Queries**.
- **Data expectations:** Must include columns:
  - `Company name`
  - `Worknig category` (note the misspelling in workflow expressions)
- **Edge cases / failures:**
  - If column names don’t match exactly, expressions later will be empty.
  - Empty sheet / no rows: downstream prompt becomes incomplete.
  - Google API rate limits or auth errors.

#### Node: OpenRouter
- **Type / role:** `LangChain Chat Model (OpenRouter)` — provides the LLM used by the query generation agent and structured output parser.
- **Key configuration:**
  - **Model:** `google/gemini-3-pro-preview`
- **Connections:**
  - Supplies `ai_languageModel` to **Generate Search Queries**
  - Supplies `ai_languageModel` to **Structured Output**
- **Edge cases / failures:**
  - OpenRouter credential missing/invalid.
  - Model availability changes (preview models can be deprecated).
  - Timeouts or rate limiting.

#### Node: Structured Output
- **Type / role:** `LangChain Structured Output Parser` — enforces JSON schema-like output.
- **Key configuration:**
  - `autoFix: true` (attempts to repair malformed JSON)
  - Example schema includes keys:
    - `query_discovery`
    - `query_comparison`
    - `query_validation`
- **Connections:**
  - Receives model from **OpenRouter**
  - Provides `ai_outputParser` into **Generate Search Queries** (agent has `hasOutputParser: true`)
- **Edge cases / failures:**
  - If the model output is too malformed, auto-fix may still fail.
  - If the agent returns extra commentary, parser may reject it.

#### Node: Generate Search Queries
- **Type / role:** `LangChain Agent` — generates the 3 GEO queries as strict JSON.
- **Key configuration:**
  - **Prompt input text:**
    - `Company name : {{ $json["Company name"] }}, Working Category :{{ $json["Worknig category"] }}`
  - **System message:** Instructs 3 intent phases and strict JSON-only output.
  - **Uses output parser:** yes (connected to **Structured Output**)
- **Inputs:** From **Get Company data**.
- **Outputs:** To **Split AI-Generated Questions**.
- **Edge cases / failures:**
  - If `Worknig category` is misspelled in the sheet (or corrected to “Working Category”), prompt will be missing category unless you align naming.
  - Parser/agent mismatch (if keys differ from expected).

#### Node: Split AI-Generated Questions
- **Type / role:** `Split Out` — converts the JSON object of queries into multiple items.
- **Key configuration:**
  - `fieldToSplitOut: "output"`
- **Inputs:** From **Generate Search Queries**.
- **Outputs:** To **Loop Over Items**.
- **Behavior detail:** Expects `output` to be a splittable structure (typically an object/array). If `output` is an object with 3 keys, n8n will emit items accordingly; ensure the downstream expects `$json.output` to be the query string.
- **Edge cases / failures:**
  - If the agent output is not under `output`, the split will produce nothing.
  - If `output` is not structured as expected, items may be malformed.

---

### Block 3 — Query Execution via BrowserAct + Logging

**Overview:**  
Iterates through each generated query, runs a BrowserAct workflow (which performs the Perplexity search), then writes “Search/Result” rows into the daily Google Sheet tab.

**Nodes involved:**
- Loop Over Items
- Run GEO Results & Rank Tracking workflow
- Store Extracted Data

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — looping controller.
- **Key configuration:**
  - Default batch behavior (batch size not explicitly set in JSON).
  - `executeOnce: false` (loop runs per item batch).
- **Inputs:** From **Split AI-Generated Questions** and also loops back from **Store Extracted Data**.
- **Outputs:**
  - To **Run GEO Results & Rank Tracking workflow** (main execution path per item)
  - To **Retrieve Stored Data** (connected as an additional output; see note below)
- **Important behavioral note:** In n8n, `SplitInBatches` typically has:
  - Output 1: current batch
  - Output 2: “No Items Left” (after loop ends)
  
  The JSON wiring shows it connected to **Retrieve Stored Data** as well as **Run GEO...**. Practically, you should ensure **Retrieve Stored Data** is connected to the “No Items Left” output so analysis happens only after all queries have been processed.
- **Edge cases / failures:**
  - Incorrect wiring can cause **Retrieve Stored Data** to run too early (before all results are appended).
  - If there are zero split items, it may go straight to “No Items Left”.

#### Node: Run GEO Results & Rank Tracking workflow
- **Type / role:** `BrowserAct` — invokes a BrowserAct workflow automation.
- **Key configuration:**
  - `type: WORKFLOW`
  - `workflowId: "7+1234567890"` (must match your BrowserAct workflow)
  - Passes input mapping:
    - `input-Inputs = {{ $json.output }}`
  - `open_incognito_mode: false`
- **Inputs:** From **Loop Over Items** (each item = one query).
- **Outputs:** To **Store Extracted Data**.
- **Expected output:** The node output is later referenced as `$json.output.string` (in **Store Extracted Data**). That implies BrowserAct returns something like:
  - `output: { string: "..." }`
- **Edge cases / failures:**
  - Invalid BrowserAct API key or workflowId.
  - BrowserAct template missing (“GEO Results & Rank Tracking” in BrowserAct account).
  - Browser automation failures (site changes, bot detection, timeouts).
  - Output shape changes (breaking `$json.output.string`).

#### Node: Store Extracted Data
- **Type / role:** `Google Sheets` (Append) — logs the query and its resulting answer text.
- **Key configuration:**
  - **Sheet target (by ID):** `={{ $('Create sheet').first().json.sheetId }}`
  - Column mapping:
    - `Search = {{ $('Loop Over Items').item.json.output }}`
    - `Result = {{ $json.output.string }}`
- **Inputs:** From **Run GEO Results & Rank Tracking workflow**.
- **Outputs:** Back to **Loop Over Items** (to continue looping).
- **Edge cases / failures:**
  - If `Loop Over Items` items do not have `.json.output` as the query string, `Search` will be blank/wrong.
  - If BrowserAct output isn’t at `output.string`, `Result` will be blank/throw.
  - Google append failures (quota, permissions, protected range).

---

### Block 4 — Aggregation, AI Analysis & Slack Reporting

**Overview:**  
After all query runs, the workflow reads all rows from the daily sheet, aggregates them into a single payload, asks an LLM agent to produce a graded Slack report, then posts it to a Slack channel.

**Nodes involved:**
- Retrieve Stored Data
- Aggregate Google Sheet Rows
- OpenRouter1
- Company data analyzer
- Send Message to Team on Slack

#### Node: Retrieve Stored Data
- **Type / role:** `Google Sheets` (Read) — reads the daily tab content back for analysis.
- **Key configuration:**
  - **Sheet target (by ID):** `={{ $('Create sheet').first().json.sheetId }}`
  - “specifyRange” mode (range details handled in UI)
  - `executeOnce: true` (will only run once even if multiple items arrive)
- **Inputs:** From **Loop Over Items** (intended: after loop completes).
- **Outputs:** To **Aggregate Google Sheet Rows**.
- **Edge cases / failures:**
  - If executed before all appends finish (due to loop wiring), results may be partial.
  - If the sheet is empty or headers-only, analysis will be weak.
  - Auth/rate limits.

#### Node: Aggregate Google Sheet Rows
- **Type / role:** `Aggregate` — combines multiple rows/items into one.
- **Key configuration:**
  - `aggregate: aggregateAllItemData` (creates a single item containing combined row data)
- **Inputs:** From **Retrieve Stored Data**.
- **Outputs:** To **Company data analyzer**.
- **Edge cases / failures:**
  - Large result sets can create very large payloads; may exceed LLM context or Slack limits.

#### Node: OpenRouter1
- **Type / role:** `LangChain Chat Model (OpenRouter)` — model provider for the analysis agent.
- **Key configuration:**
  - **Model:** `google/gemini-3-pro-preview`
- **Connections:** Supplies `ai_languageModel` to **Company data analyzer**.
- **Edge cases:** Same as **OpenRouter** (preview model stability, rate limits, auth).

#### Node: Company data analyzer
- **Type / role:** `LangChain Agent` — converts raw results into a Slack-ready GEO report.
- **Key configuration:**
  - **Input text expression:**
    - `Date : {{ $('Scheduled Daily').first().json["Readable date"] }},`
    - `Company name : {{ $('Get Company data').first().json["Company name"] }},`
    - `Working Category :{{ $('Get Company data').first().json["Worknig category"] }}`
    - `question and answers : {{ $json.data }}`
  - **System message rules:**
    - Slack formatting only, no markdown headers
    - Emojis for grading (🔴 🟡 🟢)
    - Outputs final Slack message text only
- **Inputs:** From **Aggregate Google Sheet Rows** (expects aggregated data in `$json.data`).
- **Outputs:** To **Send Message to Team on Slack**.
- **Edge cases / failures:**
  - `Readable date` likely missing unless you add a date field earlier (same issue as Block 1).
  - `$json.data` must exist from aggregate node; if the aggregate output uses a different field, prompt will be empty.
  - Output size could exceed Slack message limits (especially if raw answers are long). Consider summarizing or truncating before analysis.

#### Node: Send Message to Team on Slack
- **Type / role:** `Slack` (Post message) — sends the analysis text to a channel.
- **Key configuration:**
  - Channel: `all-browseract-workflow-test` (channelId `C09KLV9DJSX`)
  - Text: `={{ $json.output }}`
- **Inputs:** From **Company data analyzer** (expects agent output at `$json.output`).
- **Edge cases / failures:**
  - Slack token missing/invalid or lacks `chat:write`.
  - Posting to a private channel without bot membership fails.
  - If `$json.output` is an object (not string), message may be wrong; ensure agent returns plain text.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Documentation | Sticky Note | Embedded documentation / requirements | — | — | ## ⚡ Workflow Overview & Setup; Requirements: Credentials BrowserAct/OpenRouter/Google Sheets/Slack; Mandatory BrowserAct template “GEO Results & Rank Tracking”; Main Sheet must contain `Company name` and `Working Category`; Links: https://docs.browseract.com |
| Sticky Note | Sticky Note | Video link | — | — | @[youtube](intc38qZ-68) |
| Step 1 Explanation | Sticky Note | Explains strategy generation phase | — | — | ### 🎯 Step 1: Strategy Generation … generates three queries (Discovery/Comparison/Validation). |
| Step 2 Explanation | Sticky Note | Explains execution & logging phase | — | — | ### 🤖 Step 2: Execution & Logging … BrowserAct searches and appends to daily sheet. |
| Step 3 Explanation | Sticky Note | Explains analysis & reporting phase | — | — | ### 📊 Step 3: Analysis & Reporting … GEO Scorecard + Slack recommendations. |
| Scheduled Daily | Schedule Trigger | Daily workflow entry point | — | Create sheet | (See Step 1 Explanation) |
| Create sheet | Google Sheets | Create a new daily tab | Scheduled Daily | Define Headers | (See Step 1 Explanation) |
| Define Headers | Set | Prepare header row fields | Create sheet | add headers | (See Step 1 Explanation) |
| add headers | Google Sheets | Append header row to daily tab | Define Headers | Get Company data | (See Step 1 Explanation) |
| Get Company data | Google Sheets | Read company profile from Main Sheet | add headers | Generate Search Queries | (See Step 1 Explanation) |
| OpenRouter | OpenRouter Chat Model | LLM for query generation + parser | — | Generate Search Queries; Structured Output | (See Step 1 Explanation) |
| Structured Output | Structured Output Parser | Enforce valid JSON for queries | OpenRouter | Generate Search Queries (as output parser) | (See Step 1 Explanation) |
| Generate Search Queries | LangChain Agent | Generate 3 intent queries (JSON) | Get Company data | Split AI-Generated Questions | (See Step 1 Explanation) |
| Split AI-Generated Questions | Split Out | Turn JSON queries into items | Generate Search Queries | Loop Over Items | (See Step 2 Explanation) |
| Loop Over Items | Split In Batches | Iterate over queries; control loop | Split AI-Generated Questions; Store Extracted Data | Run GEO Results & Rank Tracking workflow; Retrieve Stored Data | (See Step 2 Explanation) |
| Run GEO Results & Rank Tracking workflow | BrowserAct | Execute Perplexity search automation | Loop Over Items | Store Extracted Data | (See Step 2 Explanation) |
| Store Extracted Data | Google Sheets | Append Search/Result row to daily tab | Run GEO Results & Rank Tracking workflow | Loop Over Items | (See Step 2 Explanation) |
| Retrieve Stored Data | Google Sheets | Read back daily tab results | Loop Over Items | Aggregate Google Sheet Rows | (See Step 3 Explanation) |
| Aggregate Google Sheet Rows | Aggregate | Consolidate rows into one payload | Retrieve Stored Data | Company data analyzer | (See Step 3 Explanation) |
| OpenRouter1 | OpenRouter Chat Model | LLM for analysis/reporting | — | Company data analyzer | (See Step 3 Explanation) |
| Company data analyzer | LangChain Agent | Build Slack GEO report + grades | Aggregate Google Sheet Rows | Send Message to Team on Slack | (See Step 3 Explanation) |
| Send Message to Team on Slack | Slack | Post report to Slack channel | Company data analyzer | — | (See Step 3 Explanation) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials (n8n → Credentials)**
   1) **Google Sheets OAuth2** with access to the target spreadsheet.  
   2) **Slack API** credential with permission to post messages (`chat:write`) and access to the target channel.  
   3) **OpenRouter API** credential (for Gemini model access).  
   4) **BrowserAct API** credential.

2. **Create the workflow and add the trigger**
   1) Add **Schedule Trigger** node named **Scheduled Daily**.  
   2) Configure it to run daily at your preferred time.

3. **(Recommended) Add a date field for “Readable date”**
   - Add a **Date & Time** or **Set** node right after **Scheduled Daily** to create:
     - `Readable date` (e.g., `{{ $now.toFormat('yyyy-LL-dd') }}` or any human-friendly format)
   - Connect: **Scheduled Daily → (Date/Set) → Create sheet**  
   (This is necessary because multiple nodes reference `Readable date`.)

4. **Create a new daily sheet/tab**
   1) Add **Google Sheets** node named **Create sheet**.  
   2) Operation: **Create** (sheet/tab).  
   3) Select the spreadsheet **GEO Results & Rank Tracking** (your documentId).  
   4) Title: `={{ $json["Readable date"] }}`.

5. **Add headers to the daily sheet**
   1) Add **Set** node named **Define Headers** with fields:
      - `Search` = empty string
      - `Result` = empty string
   2) Add **Google Sheets** node named **add headers**:
      - Operation: **Append**
      - Sheet: by **ID**, value `={{ $('Create sheet').first().json.sheetId }}`
      - Map columns to append the `Search` and `Result` fields.
   3) Connect: **Create sheet → Define Headers → add headers**.

6. **Load company profile from the main sheet**
   1) Add **Google Sheets** node named **Get Company data**:
      - Operation: **Read/Get Many** (depending on your sheet structure)
      - Spreadsheet: same document
      - Sheet: **Main Sheet** (`gid=0`)
      - Ensure columns exist: `Company name` and category column (align naming with expressions).
   2) Connect: **add headers → Get Company data**.

7. **Set up LLM for query generation (OpenRouter + parser + agent)**
   1) Add **OpenRouter Chat Model** node named **OpenRouter**:
      - Model: `google/gemini-3-pro-preview`
   2) Add **Structured Output Parser** node named **Structured Output**:
      - Enable **autoFix**
      - Provide example JSON with keys:
        - `query_discovery`, `query_comparison`, `query_validation`
   3) Add **LangChain Agent** node named **Generate Search Queries**:
      - Text input uses company data, e.g.  
        `Company name : {{ $json["Company name"] }}, Working Category : {{ $json["Worknig category"] }}`
      - System message: rules to output JSON only with the three keys.
      - Enable/attach output parser.
   4) Connect model + parser:
      - **OpenRouter → Generate Search Queries** as `ai_languageModel`
      - **OpenRouter → Structured Output** as `ai_languageModel`
      - **Structured Output → Generate Search Queries** as `ai_outputParser`
   5) Connect main flow: **Get Company data → Generate Search Queries**.

8. **Split the 3 generated queries into items**
   1) Add **Split Out** node named **Split AI-Generated Questions**:
      - Field to split out: `output`
   2) Connect: **Generate Search Queries → Split AI-Generated Questions**.

9. **Loop through each query**
   1) Add **Split In Batches** node named **Loop Over Items** (standard loop controller).  
   2) Connect: **Split AI-Generated Questions → Loop Over Items**.

10. **Run BrowserAct automation for each query**
   1) Add **BrowserAct** node named **Run GEO Results & Rank Tracking workflow**:
      - Type: **WORKFLOW**
      - Workflow ID: set to your BrowserAct workflow/template ID
      - Map input parameter `Inputs` (or `input-Inputs`) to: `={{ $json.output }}`
   2) Connect: **Loop Over Items → Run GEO Results & Rank Tracking workflow**.

11. **Append results to the daily sheet**
   1) Add **Google Sheets** node named **Store Extracted Data** (Append):
      - Sheet ID: `={{ $('Create sheet').first().json.sheetId }}`
      - Map:
        - `Search = {{ $('Loop Over Items').item.json.output }}`
        - `Result = {{ $json.output.string }}`
   2) Connect: **Run GEO… → Store Extracted Data**.
   3) Close the loop: **Store Extracted Data → Loop Over Items**.

12. **After loop completes: read and aggregate results**
   1) Add **Google Sheets** node named **Retrieve Stored Data**:
      - Read from the same daily sheet ID: `={{ $('Create sheet').first().json.sheetId }}`
      - Set `executeOnce` to true (optional but matches provided workflow intent).
   2) Connect **Loop Over Items** “No Items Left” output → **Retrieve Stored Data** (ensure it triggers only after all queries).
   3) Add **Aggregate** node named **Aggregate Google Sheet Rows**:
      - Mode: aggregate all item data
   4) Connect: **Retrieve Stored Data → Aggregate Google Sheet Rows**.

13. **Analyze and craft the Slack report (LLM)**
   1) Add **OpenRouter Chat Model** node named **OpenRouter1** (same model).
   2) Add **LangChain Agent** node named **Company data analyzer**:
      - Text includes date, company name/category, and aggregated Q/A data.
      - System message: Slack formatting rules + scorecard structure.
   3) Connect:
      - **OpenRouter1 → Company data analyzer** as `ai_languageModel`
      - **Aggregate Google Sheet Rows → Company data analyzer** (main)

14. **Send Slack message**
   1) Add **Slack** node named **Send Message to Team on Slack**:
      - Operation: Post message to channel
      - Channel: select your channel
      - Text: `={{ $json.output }}`
   2) Connect: **Company data analyzer → Send Message to Team on Slack**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| BrowserAct help links (API key, workflow ID, templates, n8n connection) | https://docs.browseract.com |
| Video reference | @[youtube](intc38qZ-68) |
| Spreadsheet requirements: Main Sheet must include `Company name` and `Working Category` (align exact column naming with expressions) | Used by **Get Company data** and both LLM prompts |
| BrowserAct requirement: Template/workflow “GEO Results & Rank Tracking” must exist in your BrowserAct account | Required by **Run GEO Results & Rank Tracking workflow** |
| Important fix: ensure `Readable date` exists before **Create sheet** and **Company data analyzer** references it | Add Date/Set node after trigger |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.