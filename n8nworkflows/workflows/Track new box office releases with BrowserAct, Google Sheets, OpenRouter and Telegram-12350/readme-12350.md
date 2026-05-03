Track new box office releases with BrowserAct, Google Sheets, OpenRouter and Telegram

https://n8nworkflows.xyz/workflows/track-new-box-office-releases-with-browseract--google-sheets--openrouter-and-telegram-12350


# Track new box office releases with BrowserAct, Google Sheets, OpenRouter and Telegram

## 1. Workflow Overview

**Workflow name:** Track new box office releases from BrowserAct to Google Sheets & Telegram  
**Purpose:** On a weekly schedule, scrape new box office/movie release data via a BrowserAct workflow, compare it against an existing Google Sheets “Movie history” archive to avoid duplicates, use an OpenRouter (Gemini) agent to transform/normalize the data and generate Telegram-ready HTML posts, then append the new entries to Google Sheets and publish the post(s) to a Telegram channel.

### 1.1 Scheduling & Data Collection
- Weekly trigger starts the run.
- BrowserAct fetches raw box office data.
- Google Sheets retrieves stored/previous movie rows (for deduplication context).

### 1.2 Aggregation & AI Processing
- Data is aggregated into a single payload.
- An AI agent (OpenRouter Gemini) deduplicates, extracts fields (budget/opening/gross/cast/link/summary), and generates Telegram HTML.
- A Structured Output parser enforces a strict JSON array schema and auto-fixes malformed output.

### 1.3 Archiving & Notification
- Split the AI-produced JSON array into per-movie items.
- Append each new movie’s structured data into Google Sheets.
- Send the corresponding HTML post to Telegram (HTML parse mode).

---

## 2. Block-by-Block Analysis

### Block A — Scheduling & Scraping (Entry + BrowserAct)
**Overview:** Triggers weekly and runs a BrowserAct workflow template to scrape box office data (e.g., from IMDb box office pages) into a raw JSON/string output.

**Nodes involved:**
- **Run Weekly**
- **Extract From IMDB Box Office**

#### Node: Run Weekly
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — workflow entry point.
- **Key configuration:** Runs at an interval of **every week** (`rule.interval.field = weeks`).
- **Outputs:** One execution item per scheduled run.
- **Connections:** Outputs to **Extract From IMDB Box Office**.
- **Edge cases / failures:**
- n8n instance timezone and schedule settings may cause unexpected run times.
- If the instance is asleep/offline, runs may be delayed/missed depending on hosting.

#### Node: Extract From IMDB Box Office
- **Type / role:** BrowserAct (`n8n-nodes-browseract.browserAct`) — runs a BrowserAct “WORKFLOW”.
- **Key configuration choices:**
  - **type:** `WORKFLOW`
  - **workflowId:** `7+1234567890` (must match your BrowserAct workflow ID; appears placeholder-like)
  - **workflowConfig:** contains a removed/optional string input named `BoxOffice` (BrowserAct-side default used if blank).
  - **open_incognito_mode:** `false`
- **Inputs:** Trigger item from **Run Weekly**.
- **Outputs:** BrowserAct result object; workflow later uses:  
  - `$('Extract From IMDB Box Office').first().json.output.string` (so BrowserAct is expected to return `json.output.string` containing the scraped “Movies Data”).
- **Connections:** Outputs to **Retrieve Stored Data**.
- **Credentials:** BrowserAct API credential.
- **Edge cases / failures:**
  - Invalid BrowserAct API key, missing permissions, or wrong `workflowId`.
  - BrowserAct template changes can break the expected `output.string` shape.
  - Scraping instability: site layout changes, rate limits, timeouts.

---

### Block B — Retrieve Existing Archive (Google Sheets read)
**Overview:** Pulls the existing movie history from Google Sheets so the AI can deduplicate against already-seen movie names.

**Nodes involved:**
- **Retrieve Stored Data**

#### Node: Retrieve Stored Data
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — reads stored rows.
- **Key configuration choices:**
  - Targets document: **“Box Office Trifecta”** (Spreadsheet ID `159A6tH0vMfwE4w9upGD4C-pZB7P8E8-okS3itzuPP-M`)
  - Sheet/tab: **“Movie history”** (`gid=0`)
  - Uses explicit range mode (`options.dataLocationOnSheet.rangeDefinition = specifyRange`) — but the actual range string is not shown in the provided JSON excerpt.
  - **alwaysOutputData:** `true` (workflow continues even if sheet returns no rows)
- **Inputs:** From **Extract From IMDB Box Office** (note: the scrape result is not directly consumed here, but the execution chain ensures both are available upstream).
- **Outputs:** Sheet rows in node output; later referenced as `{{ $json.data }}` after aggregation.
- **Connections:** Outputs to **Merge All Data**.
- **Credentials:** Google Sheets OAuth2.
- **Edge cases / failures:**
  - OAuth token expiry/consent issues.
  - Wrong sheet name/tab (gid) or missing headers causes empty/misaligned data.
  - Large sheets can hit API latency; consider pagination or limiting range.

---

### Block C — Aggregate inputs for AI (Merge/Aggregate)
**Overview:** Combines the BrowserAct scrape output and the sheet data into a single item so the AI agent can see both “incoming movies” and “existing movies”.

**Nodes involved:**
- **Merge All Data**

#### Node: Merge All Data
- **Type / role:** Aggregate (`n8n-nodes-base.aggregate`) — aggregates all incoming item data.
- **Key configuration choices:**
  - `aggregate = aggregateAllItemData` (collect everything into one object/item)
- **Inputs:** From **Retrieve Stored Data** (and implicitly carrying context from prior nodes in the run).
- **Outputs:** A single aggregated item; this becomes the `$json` used by the AI node.
- **Connections:** Outputs to **Create posts for Telegram**.
- **Edge cases / failures:**
  - If the sheet read returns multiple items and BrowserAct output is not present in the aggregated context as expected, the AI prompt references could break.
  - Misunderstanding of what “aggregateAllItemData” produces can lead to unexpected `$json` structure.

---

### Block D — AI Deduplication, Extraction, and Post Generation (OpenRouter + Agent + Output Parser)
**Overview:** Uses OpenRouter (Gemini) to: (1) deduplicate movies based on existing sheet names, (2) extract structured fields for Sheets, and (3) produce Telegram-safe HTML messages. A structured output parser enforces a strict JSON array result.

**Nodes involved:**
- **OpenRouter**
- **Structured Output**
- **Create posts for Telegram**

#### Node: OpenRouter
- **Type / role:** OpenRouter Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`) — the LLM provider.
- **Key configuration choices:**
  - Model: `google/gemini-3-pro-preview`
  - No special options set.
- **Connections:**
  - Provides `ai_languageModel` to **Create posts for Telegram**
  - Provides `ai_languageModel` to **Structured Output** (as connected in the workflow graph)
- **Credentials:** OpenRouter API credential.
- **Edge cases / failures:**
  - OpenRouter billing/quota limits, model availability changes (preview models can be retired).
  - Output variability: may return non-JSON or exceed Telegram length constraints if not carefully prompted.

#### Node: Structured Output
- **Type / role:** LangChain structured output parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — validates/repairs LLM output to match schema.
- **Key configuration choices:**
  - `autoFix: true` (attempt to repair invalid JSON)
  - Schema example: JSON array of objects with:
    - `movie_name`
    - `telegram_message_html`
    - `sheet_data` containing `Name`, `Budget`, `Opening_Weekend`, `Gross_Worldwide`, `Cast`, `Link`, `Summary`
- **Connections:** Feeds into **Create posts for Telegram** via `ai_outputParser`.
- **Edge cases / failures:**
  - “AutoFix” can still fail if the LLM output is too malformed.
  - If the agent returns fields not matching the example, downstream nodes referencing `sheet_data.*` may break.

#### Node: Create posts for Telegram
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates dedup + transformation.
- **Key configuration choices:**
  - **PromptType:** define
  - **hasOutputParser:** true (uses **Structured Output**)
  - **System message:** very explicit instructions:
    - Deduplicate by comparing movie “Name” with `existing_movies` (case-insensitive)
    - Build `telegram_message_html` using Telegram-supported HTML only (`<b>`, `<i>`, `<a href>`)
    - Extract fields (Budget, Opening Weekend, Gross Worldwide) from “BoxOffice” string; use `N/A` if missing
    - Return **only** a single valid JSON array, no prose, no code fences
    - Mentions Telegram 3000 char max (Telegram message limit can vary; long HTML can still fail)
  - **User text expression:**
    - `Movies Data : {{ $('Extract From IMDB Box Office').first().json.output.string }},,`
    - `Google sheet Data : {{ $json.data }}`
- **Inputs:** From **Merge All Data**.
- **Outputs:** A JSON structure expected to include an array under `output` (later split by **Split Telegram Post**).
- **Connections:** Outputs to **Split Telegram Post**.
- **Edge cases / failures:**
  - **Expression fragility:** if BrowserAct does not provide `json.output.string`, the prompt will be blank or error.
  - **Dedup mismatch:** The system message says compare against existing movie “Name”, but the sheet column naming in later append node uses `movie_name` (see Block E). Ensure the “existing_movies” data actually contains names.
  - **Telegram HTML pitfalls:** unescaped characters, unclosed tags, invalid links can cause Telegram API errors.

---

### Block E — Split, Archive to Sheets, Send to Telegram
**Overview:** Converts the AI’s output array into individual items, appends each movie’s data to the Google Sheet, then posts the HTML message to Telegram.

**Nodes involved:**
- **Split Telegram Post**
- **Store Extracted Data**
- **Send Post to Telegram Channel**

#### Node: Split Telegram Post
- **Type / role:** Split Out (`n8n-nodes-base.splitOut`) — splits an array field into multiple items.
- **Key configuration choices:**
  - `fieldToSplitOut: "output"`
- **Inputs:** From **Create posts for Telegram**.
- **Outputs:** One item per element of `output` array; each item expected to contain:
  - `telegram_message_html`
  - `sheet_data.{...}`
- **Connections:** Outputs to **Store Extracted Data**.
- **Edge cases / failures:**
  - If the agent output does not contain `output` (e.g., returns array at root), this node will fail or emit no items. Align agent output structure with what Split Out expects.

#### Node: Store Extracted Data
- **Type / role:** Google Sheets append (`n8n-nodes-base.googleSheets`) — archives new movies.
- **Key configuration choices:**
  - Operation: `append`
  - Spreadsheet: “Box Office Trifecta”
  - Sheet: “Movie history”
  - Column mapping (notable):
    - `Cast` = `{{ $json.sheet_data.Cast }}`
    - `Link` = `{{ $json.sheet_data.Link }}`
    - `Budget` = `{{ $json.sheet_data.Budget }}`
    - `Summary` = `{{ $json.sheet_data.Summary }}`
    - `Gross_Worldwide` = `{{ $json.sheet_data.Gross_Worldwide }}`
    - `Opening_Weekend` = `{{ $json.sheet_data.Opening_Weekend }}`
    - **movie_name** = `{{ $json.sheet_data.Name }}`
    - **telegram_post** = `{{ $json.telegram_message_html }}`
- **Inputs:** Per-movie item from **Split Telegram Post**.
- **Outputs:** Append result; passed to Telegram node.
- **Connections:** Outputs to **Send Post to Telegram Channel**.
- **Credentials:** Google Sheets OAuth2.
- **Edge cases / failures:**
  - **Header mismatch risk:** The sticky note “Requirements” says headers should include `Name`, but this node writes to a column called `movie_name`. If your sheet does not have `movie_name` and `telegram_post` columns, append will fail or create unexpected mapping issues. Align sheet headers with the node mapping.
  - API errors when the sheet is protected, locked, or rate-limited.

#### Node: Send Post to Telegram Channel
- **Type / role:** Telegram (`n8n-nodes-base.telegram`) — sends the message to a channel.
- **Key configuration choices:**
  - `parse_mode: HTML`
  - `appendAttribution: false`
  - `text`: `{{ $('Split Telegram Post').first().json.telegram_post }}`
  - `chatId`: `chatId=="@Your_Telegram_Channel_ID"` (this appears incorrectly formatted; typically you set chatId to `@ChannelUsername` or numeric ID, not an equality expression)
- **Inputs:** From **Store Extracted Data**.
- **Outputs:** Telegram API response.
- **Credentials:** Telegram API credential (Bot token).
- **Edge cases / failures:**
  - **chatId misconfiguration:** should likely be `@Your_Telegram_Channel_ID` (string) without `chatId==`.
  - **Text source mismatch:** The workflow stores `telegram_post` in Google Sheets mapping, but the split items contain `telegram_message_html`. Unless you transform/rename it, `telegram_post` may not exist at split stage. Consider using `{{ $json.telegram_message_html }}` directly.
  - HTML parse errors: unsupported tags/attributes, malformed `<a href="">`.
  - Telegram message length limits; long summaries may fail.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Documentation | Sticky Note | High-level setup notes |  |  | ## ⚡ Workflow Overview & Setup; Requirements (BrowserAct/OpenRouter/Sheets/Telegram); Sheet headers; BrowserAct template “Box Office Trifecta”; Links: https://docs.browseract.com |
| Sticky Note | Sticky Note | Video/reference embed |  |  | @[youtube](OFuBm6epqdY) |
| Step 1 Explanation | Sticky Note | Explains scrape + retrieve block |  |  | ### 🕵️ Step 1: Scrape & Retrieve … |
| Run Weekly | Schedule Trigger | Weekly entry point | — | Extract From IMDB Box Office | ### 🕵️ Step 1: Scrape & Retrieve … |
| Extract From IMDB Box Office | BrowserAct | Run BrowserAct workflow to scrape box office data | Run Weekly | Retrieve Stored Data | ### 🕵️ Step 1: Scrape & Retrieve … |
| Retrieve Stored Data | Google Sheets | Read existing movie archive for dedup | Extract From IMDB Box Office | Merge All Data | ### 🕵️ Step 1: Scrape & Retrieve … |
| Merge All Data | Aggregate | Combine inputs into one payload for AI | Retrieve Stored Data | Create posts for Telegram |  |
| Step 2 Explanation | Sticky Note | Explains AI processing/dedup/parsing |  |  | ### 🧠 Step 2: AI Processing & Filtering … |
| OpenRouter | OpenRouter LLM | Provides Gemini model via OpenRouter | — (AI connection) | Create posts for Telegram; Structured Output | ### 🧠 Step 2: AI Processing & Filtering … |
| Structured Output | Structured Output Parser | Enforce JSON schema / auto-fix | OpenRouter (AI) | Create posts for Telegram (as output parser) | ### 🧠 Step 2: AI Processing & Filtering … |
| Create posts for Telegram | LangChain Agent | Dedup + extract fields + craft Telegram HTML | Merge All Data; OpenRouter; Structured Output | Split Telegram Post | ### 🧠 Step 2: AI Processing & Filtering … |
| Step 3 Explanation | Sticky Note | Explains archiving + Telegram notify |  |  | ### 💾 Step 3: Archive & Notify … |
| Split Telegram Post | Split Out | Split AI output array into per-movie items | Create posts for Telegram | Store Extracted Data | ### 💾 Step 3: Archive & Notify … |
| Store Extracted Data | Google Sheets | Append new movie rows to archive | Split Telegram Post | Send Post to Telegram Channel | ### 💾 Step 3: Archive & Notify … |
| Send Post to Telegram Channel | Telegram | Post HTML message to Telegram channel | Store Extracted Data | — | ### 💾 Step 3: Archive & Notify … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   “Track new box office releases from BrowserAct to Google Sheets & Telegram”.

2. **Add node: “Run Weekly”**
   - Type: **Schedule Trigger**
   - Configure interval: **Every 1 week** (weekly).
   - This is the entry node.

3. **Add node: “Extract From IMDB Box Office”**
   - Type: **BrowserAct**
   - Operation/type: **WORKFLOW**
   - Set **Workflow ID** to your BrowserAct workflow ID (the template referenced is “Box Office Trifecta”).
   - Leave optional input(s) blank if BrowserAct has defaults.
   - Credentials: create/select **BrowserAct API** credentials.
   - Connect: **Run Weekly → Extract From IMDB Box Office**.

4. **Add node: “Retrieve Stored Data”**
   - Type: **Google Sheets**
   - Configure to **Read/Get rows** from your spreadsheet.
   - Select Spreadsheet: create/select “Box Office Trifecta” (or your own).
   - Select Sheet/tab: “Movie history”.
   - Ensure the sheet has headers. The sticky note suggests:  
     `Name`, `Budget`, `Opening_Weekend`, `Gross_Worldwide`, `Cast`, `Link`, `Summary`  
     (If you also want to store `telegram_post` and/or `movie_name`, add those headers too, or adjust mapping later.)
   - Turn on **Always Output Data** (optional but matches the workflow).
   - Credentials: **Google Sheets OAuth2**.
   - Connect: **Extract From IMDB Box Office → Retrieve Stored Data**.

5. **Add node: “Merge All Data”**
   - Type: **Aggregate**
   - Mode: **Aggregate all item data** (aggregateAllItemData)
   - Connect: **Retrieve Stored Data → Merge All Data**.

6. **Add node: “OpenRouter”**
   - Type: **OpenRouter Chat Model**
   - Choose model: `google/gemini-3-pro-preview` (or a stable Gemini model if preferred).
   - Credentials: **OpenRouter API**.

7. **Add node: “Structured Output”**
   - Type: **Structured Output Parser**
   - Enable **Auto-fix**.
   - Define/seed the schema example as an array of objects containing:
     - `movie_name`
     - `telegram_message_html`
     - `sheet_data` with keys: `Name`, `Budget`, `Opening_Weekend`, `Gross_Worldwide`, `Cast`, `Link`, `Summary`

8. **Add node: “Create posts for Telegram”**
   - Type: **AI Agent (LangChain Agent)**
   - Prompt: set to “Define” and paste a system message that instructs:
     - Deduplicate by movie name vs existing sheet list
     - Produce Telegram HTML (`<b>`, `<i>`, `<a href="">`)
     - Extract Budget/Opening/Gross from raw box office strings
     - Return only a valid JSON array
   - In the user text field, reference:
     - Movies Data from BrowserAct output (ensure your BrowserAct node returns a known field, e.g. `output.string`)
     - Existing sheet data from the aggregated `$json`
   - Connect AI:
     - Attach **OpenRouter** as the agent’s language model input.
     - Attach **Structured Output** as the agent’s output parser.
   - Connect main: **Merge All Data → Create posts for Telegram**.

9. **Add node: “Split Telegram Post”**
   - Type: **Split Out**
   - Field to split: `output` (or adjust to match your agent’s actual output field).
   - Connect: **Create posts for Telegram → Split Telegram Post**.

10. **Add node: “Store Extracted Data”**
   - Type: **Google Sheets**
   - Operation: **Append**
   - Spreadsheet: same as above
   - Sheet: “Movie history”
   - Map columns from the split item:
     - If you keep `sheet_data.Name`, map it to your sheet’s `Name` column (recommended).
     - Map the rest: `Budget`, `Opening_Weekend`, `Gross_Worldwide`, `Cast`, `Link`, `Summary`.
     - If you want to store the Telegram HTML, add a column like `telegram_post` and map `telegram_message_html` to it.
   - Credentials: **Google Sheets OAuth2**
   - Connect: **Split Telegram Post → Store Extracted Data**.

11. **Add node: “Send Post to Telegram Channel”**
   - Type: **Telegram**
   - Operation: **Send Message**
   - **Chat ID:** set to your channel username (e.g. `@my_channel`) or numeric chat ID.
   - Text: use the per-item field (recommended): `{{ $json.telegram_message_html }}` (or your chosen field).
   - Additional fields:
     - `parse_mode`: **HTML**
     - Disable attribution if desired.
   - Credentials: **Telegram Bot** token in n8n.
   - Connect: **Store Extracted Data → Send Post to Telegram Channel**.

12. **Test run**
   - Manually execute the workflow.
   - Verify:
     - BrowserAct output field exists and contains movie list data.
     - The agent returns the expected JSON structure.
     - Split Out produces items.
     - Google Sheets append matches actual headers.
     - Telegram sends successfully (HTML valid, chatId correct).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Summary: scrapes movie box office data using BrowserAct, filters out duplicates against a Google Sheet, and posts new movie updates to Telegram while archiving data.” | From workflow sticky note “Workflow Overview & Setup”. |
| Requirements: Credentials for BrowserAct, OpenRouter (Google Gemini), Google Sheets, Telegram; Mandatory BrowserAct Template: “Box Office Trifecta” | From workflow sticky note “Workflow Overview & Setup”. |
| Google Sheet should be named “Movie history” with headers: `Name`, `Budget`, `Opening_Weekend`, `Gross_Worldwide`, `Cast`, `Link`, `Summary` | From workflow sticky note “Workflow Overview & Setup”. Consider adding `telegram_post` if you want to store messages. |
| BrowserAct documentation links | https://docs.browseract.com (also referenced for API key/workflow ID and template customization) |
| YouTube reference embed | `@[youtube](OFuBm6epqdY)` (as stored in the workflow sticky note) |