Generate AI market research reports from NewsAPI and competitor sites to Notion, Sheets and Slack

https://n8nworkflows.xyz/workflows/generate-ai-market-research-reports-from-newsapi-and-competitor-sites-to-notion--sheets-and-slack-12581


# Generate AI market research reports from NewsAPI and competitor sites to Notion, Sheets and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow automates market and competitive intelligence reporting. On a daily schedule, it fetches industry news from NewsAPI and retrieves competitor website content, then uses OpenAI (GPT‑4o) to generate a structured market analysis report (SWOT, trends, competitor movements, action plan, executive summary). The output is stored in **Notion**, logged in **Google Sheets**, and announced in **Slack**. Errors are monitored and pushed to Slack via an Error Trigger.

### 1.1 Config & Triggers
Defines when the workflow runs and what it should research (keywords, competitor URLs, industry, region).

### 1.2 Data Collection
Collects data from two sources in parallel: NewsAPI results and competitor website content.

### 1.3 AI Intelligence Layer
Merges the collected data, builds a detailed analysis prompt, calls OpenAI chat completions, and parses/structures the model output.

### 1.4 Multi-Channel Export
Creates an archive entry in Notion, appends a row to Google Sheets, and posts a completion message to Slack.

### 1.5 Error Monitoring
Catches runtime failures anywhere in the workflow and sends an alert to an admin Slack channel.

**Entry points:**
- **Schedule Trigger1** (primary)
- **Error Trigger** (secondary, system-driven error handler)

---

## 2. Block-by-Block Analysis

### Block 1 — Config & Triggers

**Overview:** Runs the workflow on a schedule and emits a single configuration item containing keywords, competitor URLs, and metadata used downstream.

**Nodes Involved:**
- Schedule Trigger1
- Research Configuration1

#### Node: Schedule Trigger1
- **Type / Role:** Schedule Trigger — starts workflow automatically.
- **Configuration (interpreted):** Runs daily at **09:00** (single interval rule with `triggerAtHour: 9`).
- **Inputs / Outputs:**
  - **Output →** Research Configuration1
- **Edge cases / failures:**
  - Timezone considerations depend on n8n instance settings (server timezone).
  - If schedule is disabled or workflow inactive, nothing runs.
- **Version notes:** `typeVersion 1.2` (standard schedule node behavior).

#### Node: Research Configuration1
- **Type / Role:** Code node — defines the research scope.
- **Configuration (interpreted):**
  - Emits one JSON object:
    - `keywords`: array of 3 example topics
    - `competitors`: array of 2 example competitor URLs
    - `industry`: `"Marketing Technology"`
    - `region`: `"Global"`
    - `timestamp`: ISO time string at runtime
- **Key variables / expressions:** JavaScript constructs `researchConfig` and returns it as `{ json: researchConfig }`.
- **Inputs / Outputs:**
  - **Input:** schedule event item (not used).
  - **Outputs →** Fetch News Data1 and Scrape Competitor Site1 (fan-out).
- **Edge cases / failures:**
  - Placeholder competitor domains may not resolve.
  - Keywords/URLs not validated; malformed URLs will break HTTP requests later.
- **Version notes:** `typeVersion 2` (Code node v2).

---

### Block 2 — Data Collection

**Overview:** Pulls fresh news via NewsAPI and fetches competitor site HTML/content, then combines both streams into a single item for AI processing.

**Nodes Involved:**
- Fetch News Data1
- Scrape Competitor Site1
- Merge Data1

#### Node: Fetch News Data1
- **Type / Role:** HTTP Request — queries NewsAPI “everything” endpoint.
- **Configuration (interpreted):**
  - **GET** `https://newsapi.org/v2/everything`
  - Query parameters:
    - `q = {{$json.keywords[0]}}` (uses only the first keyword)
    - `language = en`
    - `sortBy = publishedAt`
    - `pageSize = 20`
  - **Auth:** `httpQueryAuth` via generic credential (typically `apiKey` query parameter for NewsAPI).
- **Key expressions:**
  - `={{ $json.keywords[0] }}`
- **Inputs / Outputs:**
  - **Input:** item from Research Configuration1
  - **Output →** Merge Data1 (input index 0)
- **Edge cases / failures:**
  - Missing/invalid NewsAPI key → 401/403.
  - Rate limits / quota exceeded → 429.
  - NewsAPI can return `status: error` with message even with 200-like responses depending on configuration; downstream code assumes `.articles` exists.
  - Only first keyword is used; other keywords are ignored unless workflow is modified.
- **Version notes:** HTTP Request `typeVersion 4.2`.

#### Node: Scrape Competitor Site1
- **Type / Role:** HTTP Request — retrieves competitor page content (typically HTML).
- **Configuration (interpreted):**
  - **GET** `{{$json.competitors[0]}}` (uses only the first competitor URL)
  - No explicit auth, headers, or HTML parsing.
- **Key expressions:**
  - `={{ $json.competitors[0] }}`
- **Inputs / Outputs:**
  - **Input:** item from Research Configuration1
  - **Output →** Merge Data1 (input index 1)
- **Edge cases / failures:**
  - Competitor may block bots (403), require JS rendering (content won’t be meaningful), or redirect.
  - Large HTML payloads can bloat prompt size later if included (note: in current workflow competitor data is merged but not actually used in the prompt).
  - TLS/certificate issues or timeouts.
- **Version notes:** HTTP Request `typeVersion 4.2`.

#### Node: Merge Data1
- **Type / Role:** Merge — combines news and competitor request outputs.
- **Configuration (interpreted):**
  - Mode: **combine** (pairs items from both inputs).
- **Inputs / Outputs:**
  - **Input 0:** Fetch News Data1
  - **Input 1:** Scrape Competitor Site1
  - **Output →** Prepare AI Analysis Prompt1
- **Edge cases / failures:**
  - If one branch produces 0 items (e.g., HTTP request fails and is not handled), merge will not produce expected combined output.
  - If either branch returns multiple items, combine behavior may create unexpected pairing.
- **Version notes:** Merge `typeVersion 3`.

---

### Block 3 — AI Intelligence Layer

**Overview:** Builds a Markdown prompt from the collected news, calls OpenAI chat completions (GPT‑4o), then extracts key fields (report title, executive summary, token usage) for publishing.

**Nodes Involved:**
- Prepare AI Analysis Prompt1
- Execute OpenAI Analysis1
- Structure Analysis Results1

#### Node: Prepare AI Analysis Prompt1
- **Type / Role:** Code node — transforms raw API results into a single prompt.
- **Configuration (interpreted):**
  - Reads merged inputs:
    - `const newsData = $input.all()[0].json;`
    - `const competitorData = $input.all()[1].json;` (assigned but **not used** afterward)
  - Extracts up to 10 articles from `newsData.articles` and formats them into Markdown.
  - Creates `analysisPrompt` requesting:
    1) SWOT (3 points each)
    2) Top 5 trends
    3) Competitor movements
    4) 3 actions for next 30 days
    5) Executive summary under 300 characters
  - Outputs: `{ prompt, newsArticles, timestamp, industry }`
- **Key variables / expressions:**
  - Uses optional chaining `article.source?.name`
  - Truncates `article.content` to 500 chars when present
- **Inputs / Outputs:**
  - **Input:** Merge Data1 output (combined item)
  - **Output →** Execute OpenAI Analysis1
- **Edge cases / failures:**
  - If NewsAPI response format differs or `.articles` missing, `newsArticles` becomes `[]` and the prompt may be low-signal.
  - `industry` is read from `newsData.industry`, but NewsAPI responses don’t provide this; this relies on merge behavior and/or earlier config. With the current merge setup, `newsData` is likely the NewsAPI response only; therefore `industry` may fall back to `"Marketing Technology"`.
  - Competitor data is not incorporated—report’s “Competitor Movements” will be inferred from news only unless modified.
- **Version notes:** Code node `typeVersion 2`.

#### Node: Execute OpenAI Analysis1
- **Type / Role:** HTTP Request — calls OpenAI Chat Completions API.
- **Configuration (interpreted):**
  - **POST** `https://api.openai.com/v1/chat/completions`
  - JSON body:
    - `model: "gpt-4o"`
    - `messages`: system + user (user content is the generated prompt)
    - `temperature: 0.7`
    - `max_tokens: 4000`
  - Headers: `Content-Type: application/json`
  - **Auth:** `httpHeaderAuth` generic credential (typically `Authorization: Bearer <OPENAI_API_KEY>`).
- **Key expressions:**
  - User message content: `{{ $json.prompt | quote }}`
- **Inputs / Outputs:**
  - **Input:** output of Prepare AI Analysis Prompt1
  - **Output →** Structure Analysis Results1
- **Edge cases / failures:**
  - Missing/invalid OpenAI key → 401.
  - Model not available for the account/region → 404/400.
  - Token overflow: long prompt (especially if competitor HTML is later included) can cause 400 errors.
  - Non-2xx responses must be handled (currently not caught except by Error Trigger).
- **Version notes:** HTTP Request `typeVersion 4.2`.

#### Node: Structure Analysis Results1
- **Type / Role:** Code node — validates and structures AI output for publishing.
- **Configuration (interpreted):**
  - Reads OpenAI response:
    - `analysisText = aiResponse.choices?.[0]?.message?.content || ''`
  - Throws error if empty: `"AI analysis result is empty. Check API response."`
  - Creates:
    - `fullReport`: full markdown text
    - `metadata.model`: `aiResponse.model || 'gpt-4o'`
    - `metadata.tokensUsed`: `aiResponse.usage.total_tokens || 0`
    - `executiveSummary`: regex extraction from “### 5. Executive Summary …” else first 300 chars
    - `reportTitle`: “Market Analysis Report - <MM/DD/YYYY>”
    - `generatedAt`: UTC locale string
- **Key expressions / logic:**
  - Regex: `/### 5\. Executive Summary[\s\S]*?\n([^#]+)/`
- **Inputs / Outputs:**
  - **Input:** Execute OpenAI Analysis1
  - **Outputs →** Save Report to Notion1 and Log Data to Google Sheets1 (fan-out)
- **Edge cases / failures:**
  - If the model doesn’t follow headings exactly, regex may fail and fallback summary is used.
  - Locale-dependent date formatting may vary by runtime; reportTitle uses `toLocaleDateString('en-US')`.
- **Version notes:** Code node `typeVersion 2`.

---

### Block 4 — Multi-Channel Export

**Overview:** Publishes the report to Notion, logs metadata to Google Sheets, and posts a Slack notification once export tasks complete (both Notion and Sheets connect to the same Slack node).

**Nodes Involved:**
- Save Report to Notion1
- Log Data to Google Sheets1
- Slack Completion Notification1

#### Node: Save Report to Notion1
- **Type / Role:** Notion node — creates/updates content in Notion (configured as a page operation using a Page ID).
- **Configuration (interpreted):**
  - Title: `{{$json.reportTitle}}`
  - Destination: `pageId` is set to a selectable value but currently blank in JSON (`value: ""`) → must be configured.
  - Icon: emoji “📊”
  - Content blocks: configured with a Rich Text block UI, but the workflow JSON does not clearly map `fullReport` into Notion blocks (may result in empty/placeholder content unless configured in the UI).
- **Inputs / Outputs:**
  - **Input:** Structure Analysis Results1
  - **Output →** Slack Completion Notification1
- **Edge cases / failures:**
  - Missing Notion credentials or invalid integration access to the target page → 401/403.
  - Empty `pageId` will prevent successful write.
  - Notion block structure limitations: large markdown may need conversion into multiple blocks; otherwise truncation/formatting issues.
- **Version notes:** Notion `typeVersion 2.2`.

#### Node: Log Data to Google Sheets1
- **Type / Role:** Google Sheets node — appends a logging row.
- **Configuration (interpreted):**
  - Operation: **append**
  - Document: `documentId` is blank in JSON (`value: ""`) → must be configured.
  - Sheet name: “Market Analysis Log”
  - Columns mapped:
    - `Date = {{$json.generatedAt}}`
    - `Summary = {{$json.executiveSummary}}`
    - `Tokens Used = {{$json.metadata.tokensUsed}}`
    - `Report Title = {{$json.reportTitle}}`
- **Inputs / Outputs:**
  - **Input:** Structure Analysis Results1
  - **Output →** Slack Completion Notification1
- **Edge cases / failures:**
  - Missing OAuth credentials, revoked access, or wrong spreadsheet permissions.
  - Sheet/tab name mismatch (“Market Analysis Log” must exist).
  - Header mismatch: mapping assumes these column names exist (or n8n will create columns depending on settings, but often requires headers).
- **Version notes:** Google Sheets `typeVersion 4.5`.

#### Node: Slack Completion Notification1
- **Type / Role:** Slack node — posts a completion message to a channel.
- **Configuration (interpreted):**
  - Posts a formatted message including:
    - report title, generated time, executive summary
    - indicates full report in Notion and log updated in Sheets
  - Channel selection is enabled, but `channelId` is blank in JSON (`value: ""`) → must be configured.
- **Inputs / Outputs:**
  - **Inputs:** from Save Report to Notion1 **and** Log Data to Google Sheets1 (two incoming connections)
  - **Outputs:** none
- **Edge cases / failures:**
  - Because both Notion and Sheets connect to this node, Slack may receive **two notifications** (one per branch) unless n8n execution merges/limits (it does not by default). If you want exactly one message after both complete, add a Merge/Wait mechanism.
  - Missing Slack credentials or insufficient scopes; invalid channel ID.
- **Version notes:** Slack `typeVersion 2.2`.

---

### Block 5 — Error Monitoring

**Overview:** Any node error triggers a separate execution path that posts an alert to Slack with node name, error message, and timestamp.

**Nodes Involved:**
- Error Trigger
- Slack Error Notification1

#### Node: Error Trigger
- **Type / Role:** Error Trigger — starts when another workflow execution fails.
- **Configuration (interpreted):** Default; emits an error context object.
- **Inputs / Outputs:**
  - **Output →** Slack Error Notification1
- **Edge cases / failures:**
  - Only triggers for workflow errors within the n8n instance; if Slack itself fails, the alert may not be delivered.
- **Version notes:** `typeVersion 1`.

#### Node: Slack Error Notification1
- **Type / Role:** Slack node — sends error alert.
- **Configuration (interpreted):**
  - Message includes:
    - `{{ $json.error.message }}`
    - `{{ $json.error.node }}`
    - `{{ $now.toISO() }}`
  - Channel is selectable but `channelId` is blank in JSON → must be configured.
- **Inputs / Outputs:**
  - **Input:** Error Trigger
  - **Outputs:** none
- **Edge cases / failures:**
  - Slack credential/channel misconfiguration prevents error visibility (critical).
- **Version notes:** Slack `typeVersion 2.2`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Error Trigger | n8n-nodes-base.errorTrigger | Global error entry point | — | Slack Error Notification1 | ## 5. Error Monitoring; Catches any API failures and sends an immediate alert to the admin Slack channel. |
| Slack Error Notification1 | n8n-nodes-base.slack | Sends Slack alert on workflow error | Error Trigger | — | ## 5. Error Monitoring; Catches any API failures and sends an immediate alert to the admin Slack channel. |
| Schedule Trigger1 | n8n-nodes-base.scheduleTrigger | Scheduled workflow start | — | Research Configuration1 | ## 1. Config & Triggers; Sets the schedule and defines search keywords or competitor URLs. |
| Research Configuration1 | n8n-nodes-base.code | Defines keywords/competitors/metadata | Schedule Trigger1 | Fetch News Data1; Scrape Competitor Site1 | ## 1. Config & Triggers; Sets the schedule and defines search keywords or competitor URLs. |
| Fetch News Data1 | n8n-nodes-base.httpRequest | Fetches industry news from NewsAPI | Research Configuration1 | Merge Data1 | ## 2. Data Collection; Fetches latest industry news and scrapes competitor site content simultaneously. |
| Scrape Competitor Site1 | n8n-nodes-base.httpRequest | Fetches competitor website content | Research Configuration1 | Merge Data1 | ## 2. Data Collection; Fetches latest industry news and scrapes competitor site content simultaneously. |
| Merge Data1 | n8n-nodes-base.merge | Combines news + competitor outputs | Fetch News Data1; Scrape Competitor Site1 | Prepare AI Analysis Prompt1 | ## 2. Data Collection; Fetches latest industry news and scrapes competitor site content simultaneously. |
| Prepare AI Analysis Prompt1 | n8n-nodes-base.code | Builds markdown prompt for AI | Merge Data1 | Execute OpenAI Analysis1 | ## 3. AI Intelligence Layer; Consolidates raw data, prepares the analysis prompt, and runs the GPT-4o model. |
| Execute OpenAI Analysis1 | n8n-nodes-base.httpRequest | Calls OpenAI Chat Completions (gpt-4o) | Prepare AI Analysis Prompt1 | Structure Analysis Results1 | ## 3. AI Intelligence Layer; Consolidates raw data, prepares the analysis prompt, and runs the GPT-4o model. |
| Structure Analysis Results1 | n8n-nodes-base.code | Parses AI response + creates fields | Execute OpenAI Analysis1 | Save Report to Notion1; Log Data to Google Sheets1 | ## 3. AI Intelligence Layer; Consolidates raw data, prepares the analysis prompt, and runs the GPT-4o model. |
| Save Report to Notion1 | n8n-nodes-base.notion | Archives report to Notion | Structure Analysis Results1 | Slack Completion Notification1 | ## 4. Multi-Channel Export; Saves the structured report to Notion/Sheets and alerts the team via Slack. |
| Log Data to Google Sheets1 | n8n-nodes-base.googleSheets | Appends log row to Google Sheets | Structure Analysis Results1 | Slack Completion Notification1 | ## 4. Multi-Channel Export; Saves the structured report to Notion/Sheets and alerts the team via Slack. |
| Slack Completion Notification1 | n8n-nodes-base.slack | Posts completion message to Slack | Save Report to Notion1; Log Data to Google Sheets1 | — | ## 4. Multi-Channel Export; Saves the structured report to Notion/Sheets and alerts the team via Slack. |
| Main Overview1 | n8n-nodes-base.stickyNote | Documentation / canvas note | — | — | ## 📊 Generate market research reports from news and competitor sites to Notion and Slack; How it works; Setup steps (Credentials, Configuration, Destination, Test). |
| Section  | n8n-nodes-base.stickyNote | Documentation / section header | — | — | ## 1. Config & Triggers; Sets the schedule and defines search keywords or competitor URLs. |
| Section 6 | n8n-nodes-base.stickyNote | Documentation / section header | — | — | ## 2. Data Collection; Fetches latest industry news and scrapes competitor site content simultaneously. |
| Section 7 | n8n-nodes-base.stickyNote | Documentation / section header | — | — | ## 3. AI Intelligence Layer; Consolidates raw data, prepares the analysis prompt, and runs the GPT-4o model. |
| Section 8 | n8n-nodes-base.stickyNote | Documentation / section header | — | — | ## 4. Multi-Channel Export; Saves the structured report to Notion/Sheets and alerts the team via Slack. |
| Section 9 | n8n-nodes-base.stickyNote | Documentation / section header | — | — | ## 5. Error Monitoring; Catches any API failures and sends an immediate alert to the admin Slack channel. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name: “Generate market research reports from news and competitor sites to Notion and Slack” (or your preferred name).
   - Keep it **inactive** until credentials and IDs are set.

2) **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Configure: run daily at **09:00** (adjust to your timezone requirements).
   - Connect to the next node.

3) **Add “Research Configuration” Code node**
   - Node: **Code**
   - Paste logic to output one JSON item containing:
     - `keywords` (array)
     - `competitors` (array of URLs)
     - `industry`, `region`
     - `timestamp = new Date().toISOString()`
   - Connect **Schedule Trigger → Research Configuration**.

4) **Add NewsAPI fetch node**
   - Node: **HTTP Request**
   - Method: **GET**
   - URL: `https://newsapi.org/v2/everything`
   - Enable “Send Query Parameters”, add:
     - `q` = expression `{{$json.keywords[0]}}`
     - `language` = `en`
     - `sortBy` = `publishedAt`
     - `pageSize` = `20`
   - Authentication: **Query Auth** (generic “HTTP Query Auth”)
     - Set query API key parameter as required by NewsAPI (commonly `apiKey=<YOUR_KEY>` in credentials).
   - Connect **Research Configuration → Fetch News Data**.

5) **Add competitor site fetch node**
   - Node: **HTTP Request**
   - Method: **GET**
   - URL: expression `{{$json.competitors[0]}}`
   - (Optional but recommended) Add headers like a browser User-Agent to reduce blocking.
   - Connect **Research Configuration → Scrape Competitor Site**.

6) **Merge both data sources**
   - Node: **Merge**
   - Mode: **Combine**
   - Connect:
     - **Fetch News Data → Merge (Input 0)**
     - **Scrape Competitor Site → Merge (Input 1)**

7) **Add “Prepare AI Analysis Prompt” Code node**
   - Node: **Code**
   - Build a Markdown prompt from the NewsAPI response:
     - Take top 10 articles
     - Include title/source/date/summary
     - Add explicit instructions (SWOT, trends, competitor movements, action plan, executive summary)
   - Output JSON should include at least:
     - `prompt`
     - `timestamp`
     - `industry`
   - Connect **Merge → Prepare AI Analysis Prompt**.

8) **Add OpenAI call**
   - Node: **HTTP Request**
   - Method: **POST**
   - URL: `https://api.openai.com/v1/chat/completions`
   - Body type: **JSON**
   - JSON body fields:
     - `model: "gpt-4o"`
     - `messages: [{role:"system", ...},{role:"user", content: <prompt>}]`
     - `temperature: 0.7`
     - `max_tokens: 4000`
   - Header: `Content-Type: application/json`
   - Authentication: **Header Auth**
     - Configure credential to send `Authorization: Bearer <OPENAI_API_KEY>`
   - Connect **Prepare AI Analysis Prompt → Execute OpenAI Analysis**.

9) **Add “Structure Analysis Results” Code node**
   - Node: **Code**
   - Parse:
     - `choices[0].message.content` → `fullReport`
     - `usage.total_tokens` → `metadata.tokensUsed`
   - Create:
     - `reportTitle`
     - `generatedAt` (UTC)
     - `executiveSummary` (regex heading extraction with fallback)
   - Throw an error if content is empty.
   - Connect **Execute OpenAI Analysis → Structure Analysis Results**.

10) **Add Notion export**
   - Node: **Notion**
   - Credentials: Notion integration token with access to your destination page/database.
   - Configure destination:
     - Set a valid **Page ID** (or adapt node to create a new page in a database).
   - Map title: `{{$json.reportTitle}}`
   - Ensure the content includes `{{$json.fullReport}}` in Notion blocks (configure blocks in the node UI accordingly).
   - Connect **Structure Analysis Results → Save Report to Notion**.

11) **Add Google Sheets logging**
   - Node: **Google Sheets**
   - Credentials: Google OAuth2 with access to the spreadsheet.
   - Operation: **Append**
   - Set **Document ID** and **Sheet name** (tab) e.g. “Market Analysis Log”
   - Map columns:
     - Date = `{{$json.generatedAt}}`
     - Report Title = `{{$json.reportTitle}}`
     - Summary = `{{$json.executiveSummary}}`
     - Tokens Used = `{{$json.metadata.tokensUsed}}`
   - Connect **Structure Analysis Results → Log Data to Google Sheets**.

12) **Add Slack completion notification**
   - Node: **Slack**
   - Credentials: Slack OAuth with permission to post messages.
   - Select channel and set **Channel ID**.
   - Message text should reference:
     - `{{$json.reportTitle}}`, `{{$json.generatedAt}}`, `{{$json.executiveSummary}}`
   - Connect:
     - **Save Report to Notion → Slack Completion Notification**
     - **Log Data to Google Sheets → Slack Completion Notification**
   - If you want **only one** Slack message, insert a **Merge (wait for both)** pattern or a single branch that sends Slack after both exports complete.

13) **Add Error handling**
   - Node: **Error Trigger**
   - Add **Slack** node for error alerts:
     - Include `{{$json.error.message}}`, `{{$json.error.node}}`, and `{{$now.toISO()}}`
     - Configure admin channel ID
   - Connect **Error Trigger → Slack Error Notification**.

14) **Finalize**
   - Fill in missing IDs (Notion page/database, Google Sheet doc + tab, Slack channels).
   - Run a manual execution to validate:
     - NewsAPI returns `articles`
     - OpenAI returns `choices[0].message.content`
     - Notion block mapping actually writes `fullReport`
   - Activate the workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Generate market research reports from news and competitor sites to Notion and Slack” overview, setup steps, and required headers “Date, Title, Summary, Tokens”. | From the workflow’s main sticky note content (canvas documentation). |
| Important configuration limitation: only `keywords[0]` and `competitors[0]` are used in HTTP nodes as currently built. | Consider iterating over arrays (Split in Batches / Item Lists) if you want multi-keyword and multi-competitor coverage. |
| Slack completion node has two incoming branches; it may send duplicate notifications. | Add a merge/join strategy if exactly one completion message is required. |