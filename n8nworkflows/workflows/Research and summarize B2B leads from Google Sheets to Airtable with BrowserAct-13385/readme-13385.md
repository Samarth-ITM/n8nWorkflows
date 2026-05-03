Research and summarize B2B leads from Google Sheets to Airtable with BrowserAct

https://n8nworkflows.xyz/workflows/research-and-summarize-b2b-leads-from-google-sheets-to-airtable-with-browseract-13385


# Research and summarize B2B leads from Google Sheets to Airtable with BrowserAct

## 1. Workflow Overview

**Purpose:** Automate B2B lead research from a Google Sheet of company URLs by (1) scraping each URL via BrowserAct, (2) having an LLM normalize the scraped content into a structured “page summary” saved back to Google Sheets, and (3) aggregating all collected page summaries into one consolidated “research record” that is then created in Airtable.

**Primary use cases**
- Build a lightweight B2B research pipeline from a URL list.
- Normalize heterogeneous web content (About pages, changelogs/news pages) into consistent CRM-ready fields.
- Produce one Airtable record per run, representing the synthesized company research.

### 1.1 Input Reception (Google Sheets → Items)
Reads rows from Google Sheets (expects a `Page URL` column), then iterates through them in a batch loop.

### 1.2 Automated Scraping (BrowserAct per URL)
For each `Page URL`, triggers a BrowserAct workflow to extract website content.

### 1.3 AI Normalization per Page (LLM → Structured JSON)
Feeds BrowserAct output into an LLM Agent with a strict flat JSON schema; parses/auto-fixes the structured output and updates the originating Google Sheet row (`Page Data`).

### 1.4 Post-Loop Aggregation (Google Sheets → Aggregate)
After all URLs are processed, retrieves the stored rows (now containing structured `Page Data`) and aggregates them into a single array.

### 1.5 AI Synthesis (Aggregate → Airtable-ready JSON) + Airtable Write
Uses another LLM Agent to parse the array (including stringified JSON in `Page Data`) into a single Airtable record payload, then creates a record in Airtable.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Input Retrieval
**Overview:** Starts the workflow manually and loads the source dataset from Google Sheets (the URL list).  
**Nodes involved:** `Manual execution`, `Retrieve Input Data`

#### Node: Manual execution
- **Type / role:** `Manual Trigger` (`n8n-nodes-base.manualTrigger`) — entry point for manual runs.
- **Config (interpreted):** No parameters; runs when clicked.
- **Outputs:** Sends a single trigger item to `Retrieve Input Data`.
- **Failure modes:** None (except instance-level execution restrictions).

#### Node: Retrieve Input Data
- **Type / role:** `Google Sheets` (`n8n-nodes-base.googleSheets`) — reads the database sheet.
- **Config (interpreted):**
  - Auth: Google Sheets OAuth2 credential (`Google Sheets account`)
  - Document: **“B2B Contact Research”** (Spreadsheet ID `1Y8B9x...`)
  - Sheet/tab: **“DataBase”** (gid=0)
  - Operation: not explicitly shown in JSON snippet (node defaults typically to **Read/Get Many** depending on UI state). Functionally, it outputs rows.
- **Expected input columns:** at least `Page URL`. Also later logic references `row_number`.
- **Outputs:** Row items to `Loop Over Items`.
- **Failure modes / edge cases:**
  - OAuth token expired / missing scopes
  - Sheet/tab renamed or gid changed
  - Missing `Page URL` column → downstream BrowserAct mapping fails or produces blank input.

---

### Block 2 — Iteration Control (Looping)
**Overview:** Iterates through the sheet rows and provides a “done” path once all items have been processed.  
**Nodes involved:** `Loop Over Items`

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` (`n8n-nodes-base.splitInBatches`) — controls per-row execution and loop-back.
- **Config (interpreted):**
  - Uses default options (batch size defaults to 1 unless changed in UI).
- **Connections:**
  - **Main output 0 (loop items)** → `Extract Target Page Data`
  - **Main output 1 (no more items / done)** → `Retrieve Stored Data`
  - Receives loop-back input from `Update Database` to continue with next row.
- **Failure modes / edge cases:**
  - If prior nodes error, loop stops early.
  - If batch size > 1, downstream assumptions about per-row updates may need adjustment.

---

### Block 3 — Automated Scraping (BrowserAct)
**Overview:** For each `Page URL`, triggers a BrowserAct workflow that scrapes the target page and returns extracted data.  
**Nodes involved:** `Extract Target Page Data`

#### Node: Extract Target Page Data
- **Type / role:** `BrowserAct` (`n8n-nodes-browseract.browserAct`) — runs a BrowserAct “WORKFLOW” extraction.
- **Config (interpreted):**
  - Mode: `WORKFLOW`
  - BrowserAct workflowId: `77103692630144570`
  - Mapped input field: `Opponent_Page` ← `{{ $json["Page URL"] }}`
    - In BrowserAct config schema this is `input-Opponent_Page`.
  - Credentials: `BrowserAct account`
- **Input / output:**
  - Input item must include `Page URL`.
  - Output is referenced later as `{{ $json.output.string }}` in the AI node, implying BrowserAct returns something like:
    - `output.string` containing raw scraped JSON (as text).
- **Failure modes / edge cases:**
  - Invalid workflowId or inactive BrowserAct template
  - BrowserAct auth errors / quota / rate limiting
  - Target site blocks scraping, heavy JS, timeouts
  - `Page URL` empty → BrowserAct uses default value (as hinted by description) or fails depending on BrowserAct workflow settings.

---

### Block 4 — AI Normalization per Page (LLM Agent → Structured Output → Update Sheet)
**Overview:** Converts raw scraped content into a strict flat JSON “page summary”, then writes it back into the same row in Google Sheets as `Page Data`.  
**Nodes involved:** `Analyze the Company Page`, `OpenRouter Chat Model`, `Structured Output Parser`, `Update Database`

#### Node: OpenRouter Chat Model
- **Type / role:** LangChain Chat Model via OpenRouter (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`) — provides the LLM for the per-page analysis.
- **Config (interpreted):**
  - Model: `openai/gpt-5`
  - Credentials: `OpenRouter account`
- **Connections:**
  - Supplies `ai_languageModel` to both:
    - `Analyze the Company Page`
    - `Structured Output Parser`
- **Failure modes / edge cases:**
  - OpenRouter key invalid
  - Model not available to account / region
  - Token limits if BrowserAct output is large → truncation or refusal.

#### Node: Structured Output Parser
- **Type / role:** Structured output parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces JSON schema and auto-fixes formatting.
- **Config (interpreted):**
  - `autoFix: true` (attempts to repair near-JSON outputs)
  - Schema example indicates fields:
    - `data_type`, `company_name`, `primary_date`, `core_summary`, `key_entities`, `strategic_focus`
  - Note: the agent’s system schema also includes `raw_content_snippet` and `source_url`, so actual output is expected to include those too.
- **Connections:**
  - Provides `ai_outputParser` into `Analyze the Company Page` (meaning the Agent uses it to validate/shape output).
- **Failure modes / edge cases:**
  - If the LLM returns non-repairable JSON, parsing fails.
  - If the LLM omits required keys (depending on parser strictness), downstream mapping may break.

#### Node: Analyze the Company Page
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — classifies content type (News vs Profile) and produces a flat JSON summary.
- **Config (interpreted):**
  - Input text: `{{ $json.output.string }}` (raw scraped content from BrowserAct)
  - Prompt type: “define”
  - System message specifies:
    - Detect dataset type by keys
    - For News: sort by date, summarize last 3 updates, infer focus, extract latest title/date
    - For Profile: mission, founders, keywords, find links (Careers/Pricing)
    - Output **flat JSON** (no nested arrays/objects); lists must be joined.
  - Uses output parser (`hasOutputParser: true`) → `Structured Output Parser`.
- **Outputs:** Produces `output` (structured JSON) used by `Update Database`.
- **Failure modes / edge cases:**
  - If BrowserAct output isn’t valid/meaningful text, the agent may hallucinate structure.
  - Date normalization issues (non-ISO inputs)
  - “Flat JSON” constraint: if the model outputs arrays/objects, parser may fail or auto-fix incorrectly.

#### Node: Update Database
- **Type / role:** `Google Sheets` (`n8n-nodes-base.googleSheets`) — writes the structured page summary back to the originating row.
- **Config (interpreted):**
  - Operation: `update`
  - Match column: `row_number`
  - Writes:
    - `Page URL` = `{{ $('Loop Over Items').item.json["Page URL"] }}`
    - `Page Data` = `{{ $json.output }}` (the agent’s structured output)
    - `row_number` = `{{ $('Loop Over Items').item.json.row_number }}`
  - Auth: Google Sheets OAuth2 (`Google Sheets account`)
  - Same spreadsheet/tab as input.
- **Connections:**
  - Output → loops back into `Loop Over Items` to process next URL.
- **Failure modes / edge cases:**
  - `row_number` missing (common if the read operation doesn’t include it) → update cannot match.
  - If `Page Data` is an object, Sheets node may coerce it; downstream expects it to be a **stringified JSON** later. Depending on Sheets node behavior, you may need to ensure `Page Data` is stored as a string (e.g., `JSON.stringify($json.output)`).
  - Sheet protected / insufficient permissions.

---

### Block 5 — Retrieve Stored Results & Aggregate
**Overview:** After the loop completes, reloads (or reads) stored sheet data and aggregates all items into a single array for final synthesis.  
**Nodes involved:** `Retrieve Stored Data`, `Aggregate`

#### Node: Retrieve Stored Data
- **Type / role:** `Google Sheets` — fetches rows containing stored `Page Data`.
- **Config (interpreted):**
  - Same spreadsheet/tab (`B2B Contact Research` → `DataBase`)
  - `executeOnce: true` so it runs once per workflow execution (even if multiple incoming items arrive).
- **Connections:** Output → `Aggregate`
- **Failure modes / edge cases:**
  - Same Google Sheets risks as before.
  - If the sheet contains many rows, payload can become large before LLM synthesis.

#### Node: Aggregate
- **Type / role:** `Aggregate` (`n8n-nodes-base.aggregate`) — converts multiple items into one item with a `data` array.
- **Config (interpreted):**
  - Mode: `aggregateAllItemData` (collects full item JSON for all incoming items)
- **Connections:** Output → `Analyze data and create an Airtable record`
- **Failure modes / edge cases:**
  - Large arrays can exceed memory or downstream LLM context limits.
  - If some rows have empty/invalid `Page Data`, synthesis prompt may degrade.

---

### Block 6 — AI Synthesis into Airtable Record + Airtable Create
**Overview:** Parses all per-page summaries (notably `Page Data` is expected to be a stringified JSON), synthesizes one consolidated research profile, and creates a new Airtable record.  
**Nodes involved:** `Analyze data and create an Airtable record`, `OpenRouter Chat Model1`, `Structured Output Parser1`, `Create a record`

#### Node: OpenRouter Chat Model1
- **Type / role:** OpenRouter chat model — LLM for final synthesis.
- **Config (interpreted):**
  - Model: `google/gemini-3-flash-preview`
  - Credentials: `OpenRouter account`
- **Connections:** Supplies `ai_languageModel` to:
  - `Analyze data and create an Airtable record`
  - `Structured Output Parser1`
- **Failure modes / edge cases:** same OpenRouter concerns; also model differences can affect schema compliance.

#### Node: Structured Output Parser1
- **Type / role:** Structured output parser — enforces Airtable-compatible JSON.
- **Config (interpreted):**
  - `autoFix: true`
  - Example schema includes keys:
    - `Name`, `Notes`, `Assignee`, `Status`, `Attachments`, `Attachment Summary`
- **Connections:** Provides `ai_outputParser` into `Analyze data and create an Airtable record`.
- **Failure modes / edge cases:**
  - If `Attachments` isn’t an array, Airtable node may reject.
  - If `Status` doesn’t match allowed options (Todo/In progress/Done), Airtable may fail unless typecast handles it.

#### Node: Analyze data and create an Airtable record
- **Type / role:** LangChain Agent — synthesizes one company research record from aggregated data.
- **Config (interpreted):**
  - Input text:
    - `= array of data: {{ JSON.stringify($json.data, null, 2) }}`
  - System message highlights a crucial assumption:
    - Each item has `Page Data` and **it is a STRINGIFIED JSON object**.
    - The agent must “mentally parse” that string to extract fields like `strategic_focus`, `key_entities`, and detect “News”.
  - Output: strict JSON matching Airtable fields; Markdown allowed in `Notes`.
  - Uses output parser (`hasOutputParser: true`) → `Structured Output Parser1`.
- **Connections:** Output → `Create a record`
- **Failure modes / edge cases:**
  - If `Page Data` was stored as an object (not a JSON string), the prompt assumption is wrong; synthesis quality may degrade.
  - Context window overflow if aggregated dataset is large.
  - Inconsistent company names across pages (agent must infer a single one).

#### Node: Create a record
- **Type / role:** `Airtable` (`n8n-nodes-base.airtable`) — creates a CRM record.
- **Config (interpreted):**
  - Auth: Airtable OAuth2 / PAT (`Airtable Personal Access Token account 2`)
  - Base: `BrowserAct_Test` (`appGWdWTgqsbIVXP3`)
  - Table: `Table 1` (`tblMT3dO4qmf0GpRU`)
  - Operation: `create`
  - Typecast: enabled (`typecast: true`)
  - Field mapping (expressions):
    - `Name` = `{{ $json.output.Name }}`
    - `Notes` = `{{ $json.output.Notes }}`
    - `Status` = `{{ $json.output.Status }}`
    - `Attachments` = `{{ $json.output.Attachments }}`
    - `Attachment Summary` = `{{ $json.output["Attachment Summary"] }}`
  - Note: `Assignee` exists in the parser schema but is marked removed in Airtable schema mapping (field removed in Airtable table), so it is not written.
- **Failure modes / edge cases:**
  - Airtable auth errors / base/table not found
  - Field type mismatch (e.g., Attachments must be array of attachment objects/URLs depending on Airtable API expectations)
  - `Status` option not matching allowed values (though typecast may help)

---

### Block 7 — Documentation / Sticky Notes (Non-executing)
**Overview:** Provides usage requirements, links, and a YouTube reference.  
**Nodes involved:** `Documentation`, `Step 1 Explanation`, `Step 3 Explanation`, `Sticky Note`

#### Node: Documentation (Sticky Note)
- **Type / role:** Sticky note (`n8n-nodes-base.stickyNote`) — embedded guidance.
- **Content highlights (preserved):**
  - Requirements: BrowserAct, OpenRouter, Google Sheets, Airtable credentials
  - Mandatory: BrowserAct API (Template: **B2B Contact Research**)
  - Links:
    - https://docs.browseract.com (API key & workflow ID)
    - https://docs.browseract.com (connect n8n)
    - https://docs.browseract.com (templates)

#### Node: Step 1 Explanation (Sticky Note)
- **Type / role:** Sticky note — describes scraping step.
- **Content highlights:** “Step 1: Automated Scraping…”

#### Node: Step 3 Explanation (Sticky Note)
- **Type / role:** Sticky note — describes Airtable entry creation.
- **Content highlights:** “Step 3: Database Entry…”

#### Node: Sticky Note (YouTube)
- **Type / role:** Sticky note — video reference.
- **Content:** `@[youtube](mLCuN9Of6EM)`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual execution | manualTrigger | Manual entry point | — | Retrieve Input Data | ## ⚡ Workflow Overview & Setup<br>**Summary:** Automate B2B lead research by scraping company profiles and news from a list of URLs, synthesizing the data with AI, and organizing it into a structured Airtable database.<br>**Requirements**: BrowserAct, OpenRouter, Google Sheets, Airtable. Mandatory BrowserAct template **B2B Contact Research**. Links: https://docs.browseract.com |
| Retrieve Input Data | googleSheets | Read URL rows from Google Sheets | Manual execution | Loop Over Items | ### 🌐 Step 1: Automated Scraping<br>The workflow reads a list of URLs from Google Sheets… and updates the sheet with raw JSON results. |
| Loop Over Items | splitInBatches | Iterate rows; provide “done” branch | Retrieve Input Data; Update Database | Extract Target Page Data (loop); Retrieve Stored Data (done) | ### 🌐 Step 1: Automated Scraping<br>The workflow reads a list of URLs… triggers BrowserAct… updates the sheet… |
| Extract Target Page Data | browserAct | Scrape a target URL via BrowserAct workflow | Loop Over Items | Analyze the Company Page | ### 🌐 Step 1: Automated Scraping<br>The workflow reads a list of URLs… triggers BrowserAct… updates the sheet… |
| OpenRouter Chat Model | lmChatOpenRouter | LLM for per-page normalization | — (AI connection) | Analyze the Company Page; Structured Output Parser |  |
| Structured Output Parser | outputParserStructured | Enforce/repair structured JSON for page summary | — (AI connection) | Analyze the Company Page (as output parser) |  |
| Analyze the Company Page | langchain.agent | Convert scraped content into flat JSON summary | Extract Target Page Data; (AI model + parser) | Update Database | ### 🌐 Step 1: Automated Scraping<br>The workflow reads a list of URLs… triggers BrowserAct… updates the sheet… |
| Update Database | googleSheets | Update the same row with `Page Data` | Analyze the Company Page | Loop Over Items | ### 🌐 Step 1: Automated Scraping<br>The workflow reads a list of URLs… triggers BrowserAct… updates the sheet… |
| Retrieve Stored Data | googleSheets | Load stored enriched rows (after loop) | Loop Over Items (done output) | Aggregate | ### 📂 Step 3: Database Entry<br>Once all scraping is complete, the data is aggregated… strict JSON compatible with Airtable… |
| Aggregate | aggregate | Aggregate all rows into one array | Retrieve Stored Data | Analyze data and create an Airtable record | ### 📂 Step 3: Database Entry<br>Once all scraping is complete, the data is aggregated… strict JSON compatible with Airtable… |
| OpenRouter Chat Model1 | lmChatOpenRouter | LLM for final synthesis | — (AI connection) | Analyze data and create an Airtable record; Structured Output Parser1 |  |
| Structured Output Parser1 | outputParserStructured | Enforce/repair Airtable record JSON | — (AI connection) | Analyze data and create an Airtable record (as output parser) |  |
| Analyze data and create an Airtable record | langchain.agent | Synthesize one consolidated research record | Aggregate; (AI model + parser) | Create a record | ### 📂 Step 3: Database Entry<br>Once all scraping is complete… workflow then creates a new record in Airtable… |
| Create a record | airtable | Create Airtable record | Analyze data and create an Airtable record | — | ### 📂 Step 3: Database Entry<br>Once all scraping is complete… workflow then creates a new record in Airtable… |
| Documentation | stickyNote | Setup notes + links | — | — | (same as content) Links: https://docs.browseract.com |
| Step 1 Explanation | stickyNote | Comment about scraping step | — | — | ### 🌐 Step 1: Automated Scraping… |
| Step 3 Explanation | stickyNote | Comment about Airtable step | — | — | ### 📂 Step 3: Database Entry… |
| Sticky Note | stickyNote | YouTube reference | — | — | @[youtube](mLCuN9Of6EM) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it similar to: *Automate B2B lead research from Google Sheets to Airtable With BrowserAct*.
   - Ensure workflow setting **Execution Order** is `v1` (Workflow Settings → Execution order).

2. **Add trigger**
   - Add node: **Manual Trigger**
   - Name: `Manual execution`

3. **Add Google Sheets “Retrieve Input Data”**
   - Add node: **Google Sheets**
   - Name: `Retrieve Input Data`
   - Credentials: configure **Google Sheets OAuth2** (select the account you want).
   - Document: select your spreadsheet (e.g., “B2B Contact Research”).
   - Sheet: select tab (e.g., “DataBase”).
   - Operation: read rows (Get Many / Read) so each row becomes an item.
   - Ensure the sheet contains:
     - Column **`Page URL`** (required)
     - A usable row identifier (commonly `row_number` if using n8n’s Google Sheets node with “Return All” / include row number).

4. **Add loop controller**
   - Add node: **Split In Batches**
   - Name: `Loop Over Items`
   - Batch size: typically `1` (default is fine).
   - Connect: `Manual execution` → `Retrieve Input Data` → `Loop Over Items`.

5. **Add BrowserAct scraper**
   - Add node: **BrowserAct**
   - Name: `Extract Target Page Data`
   - Credentials: configure **BrowserAct API** (API key).
   - Resource/Mode: run a **WORKFLOW**
   - Set **Workflow ID** to your BrowserAct workflow (example in JSON: `77103692630144570`)
   - In Workflow Config mapping, map:
     - `Opponent_Page` (or the input field used by your BrowserAct workflow) to expression: `{{ $json["Page URL"] }}`
   - Connect: `Loop Over Items` (main output 0) → `Extract Target Page Data`

6. **Add LLM model (OpenRouter) for per-page parsing**
   - Add node: **OpenRouter Chat Model** (LangChain)
   - Name: `OpenRouter Chat Model`
   - Credentials: configure **OpenRouter** API key
   - Model: `openai/gpt-5` (or a compatible alternative)

7. **Add Structured Output Parser for per-page JSON**
   - Add node: **Structured Output Parser**
   - Name: `Structured Output Parser`
   - Enable **Auto-fix**
   - Provide a JSON schema example containing keys like:
     - `data_type`, `company_name`, `primary_date`, `core_summary`, `key_entities`, `strategic_focus`, `raw_content_snippet`, `source_url`

8. **Add per-page LangChain Agent**
   - Add node: **AI Agent** (LangChain Agent)
   - Name: `Analyze the Company Page`
   - Text input: `{{ $json.output.string }}` (adjust if your BrowserAct output path differs)
   - System message: paste the “Senior B2B Market Analyst” instructions (News/Profile detection + flat JSON output).
   - Enable **Use Output Parser** and attach `Structured Output Parser`.
   - Attach **OpenRouter Chat Model** as the language model (AI connection).
   - Connect: `Extract Target Page Data` → `Analyze the Company Page`

9. **Write results back to Google Sheets**
   - Add node: **Google Sheets**
   - Name: `Update Database`
   - Operation: **Update**
   - Document/Sheet: same as input
   - Matching column: `row_number`
   - Set fields:
     - `Page URL` = `{{ $('Loop Over Items').item.json["Page URL"] }}`
     - `Page Data` = `{{ $json.output }}`  
       - If you want to guarantee it is stored as a string (recommended for the later synthesis prompt), use: `{{ JSON.stringify($json.output) }}`
     - `row_number` = `{{ $('Loop Over Items').item.json.row_number }}`
   - Connect: `Analyze the Company Page` → `Update Database`
   - Connect loop-back: `Update Database` → `Loop Over Items` (this advances the batch loop)

10. **Add “Retrieve Stored Data” for post-loop**
    - Add node: **Google Sheets**
    - Name: `Retrieve Stored Data`
    - Same document/sheet
    - Operation: read rows (Get Many / Read)
    - Enable **Execute Once** (node setting) so it runs once when the loop is finished.
    - Connect: `Loop Over Items` (main output 1 / done) → `Retrieve Stored Data`

11. **Aggregate all rows**
    - Add node: **Aggregate**
    - Name: `Aggregate`
    - Mode: **Aggregate All Item Data** (collect all items into one array)
    - Connect: `Retrieve Stored Data` → `Aggregate`

12. **Add LLM model (OpenRouter) for synthesis**
    - Add node: **OpenRouter Chat Model**
    - Name: `OpenRouter Chat Model1`
    - Model: `google/gemini-3-flash-preview` (or similar)
    - Credentials: same OpenRouter account

13. **Add Structured Output Parser for Airtable payload**
    - Add node: **Structured Output Parser**
    - Name: `Structured Output Parser1`
    - Auto-fix: enabled
    - Schema example with keys:
      - `Name`, `Notes`, `Assignee`, `Status`, `Attachments`, `Attachment Summary`

14. **Add synthesis LangChain Agent**
    - Add node: **AI Agent**
    - Name: `Analyze data and create an Airtable record`
    - Text input: `array of data: {{ JSON.stringify($json.data, null, 2) }}`
    - System message: paste the “Expert B2B Lead Researcher & Data Synthesizer” instructions, including the note that `Page Data` is stringified JSON.
    - Attach language model: `OpenRouter Chat Model1`
    - Attach output parser: `Structured Output Parser1`
    - Connect: `Aggregate` → `Analyze data and create an Airtable record`

15. **Create Airtable record**
    - Add node: **Airtable**
    - Name: `Create a record`
    - Credentials: configure Airtable OAuth2 or Personal Access Token with permissions for the base.
    - Base: select your base (example: `BrowserAct_Test`)
    - Table: select your table (example: `Table 1`)
    - Operation: **Create**
    - Enable **Typecast**
    - Map fields from the agent output:
      - `Name` = `{{ $json.output.Name }}`
      - `Notes` = `{{ $json.output.Notes }}`
      - `Status` = `{{ $json.output.Status }}`
      - `Attachments` = `{{ $json.output.Attachments }}`
      - `Attachment Summary` = `{{ $json.output["Attachment Summary"] }}`
    - Connect: `Analyze data and create an Airtable record` → `Create a record`

16. **(Optional) Add sticky notes**
    - Add sticky note content for requirements and links:
      - https://docs.browseract.com
    - Add the YouTube embed note: `@[youtube](mLCuN9Of6EM)`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| YouTube reference | @[youtube](mLCuN9Of6EM) |
| Mandatory BrowserAct template mentioned in notes: **B2B Contact Research** | (Referenced in sticky note “Workflow Overview & Setup”) |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.