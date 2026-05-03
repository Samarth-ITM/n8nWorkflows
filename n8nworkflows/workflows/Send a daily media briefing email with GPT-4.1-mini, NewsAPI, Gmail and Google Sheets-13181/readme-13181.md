Send a daily media briefing email with GPT-4.1-mini, NewsAPI, Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/send-a-daily-media-briefing-email-with-gpt-4-1-mini--newsapi--gmail-and-google-sheets-13181


# Send a daily media briefing email with GPT-4.1-mini, NewsAPI, Gmail and Google Sheets

## 1. Workflow Overview

**Purpose:**  
This workflow runs daily to fetch recent NewsAPI articles about a configured topic, measures coverage volume vs. historical averages stored in Google Sheets, asks an AI agent (GPT‑4.1‑mini) to produce a short HTML media briefing (with optional anomaly investigation), then emails the briefing via Gmail and logs the day’s metrics back to Google Sheets.

**Target use cases:**  
- Daily media monitoring for a brand, executive, company, or theme  
- Early detection of unusual spikes/drops in coverage volume  
- Automated email briefing for PR / media relations teams

### 1.1 Scheduling & Time Window Setup
Creates a “last N days” timeframe and formats a date used to query NewsAPI.

### 1.2 User Configuration & News Fetch
Defines the query/topic and credentials/targets, then retrieves articles from NewsAPI.

### 1.3 Coverage Metrics Logging & Historic Baseline
Extracts today’s volume metrics, appends them to Google Sheets, retrieves historical rows, and computes an average volume.

### 1.4 Data Merge for AI Context
Combines: (a) today’s metrics, (b) historic list, (c) average volume into one payload for the AI agent.

### 1.5 AI Briefing Generation (with optional deep-dive)
GPT‑4.1‑mini selects top 10 articles, writes a compact HTML briefing, and may call an HTTP “Topic deep‑dive” tool if it detects an anomaly.

### 1.6 Email Output
Maps the AI output to an email body and sends it via Gmail.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Time Window Setup
**Overview:** Triggers once per day, computes a date 2 days in the past, then formats it for the NewsAPI `from` parameter.  
**Nodes involved:** Trigger daily → Set timeframe to last 2 days → Format date

#### Node: **Trigger daily**
- **Type / role:** Schedule Trigger — starts workflow on a timer.
- **Configuration (interpreted):** Runs on an interval rule (JSON shows `interval:[{}]`, which typically needs proper interval settings in UI).
- **Key variables used downstream:** `timestamp` is referenced later (e.g., memory session key). Also `Readable date` is referenced in email subject and AI header but is **not produced** by this node by default.
- **Outputs to:** Set timeframe to last 2 days
- **Edge cases / failures:**
  - Misconfigured schedule rule may not run at all.
  - Missing `Readable date` field causes expressions like `$('Trigger daily').item.json['Readable date']` to resolve to empty/undefined.

#### Node: **Set timeframe to last 2 days**
- **Type / role:** Date & Time — subtracts a duration from a date.
- **Configuration:** `subtractFromDate` with `duration: 2` and base date coming from `{{$json.timestamp}}`.
- **Outputs to:** Format date
- **Edge cases:**
  - If `timestamp` is absent or not a valid date, subtraction fails.

#### Node: **Format date**
- **Type / role:** Date & Time — formats date to `yyyy-MM-dd`.
- **Configuration:** Formats `{{$json.newDate}}` produced by the previous node.
- **Outputs to:** Set user config
- **Edge cases:**
  - If `newDate` missing/invalid → formatting error.

---

### Block 2 — User Configuration & News Fetch
**Overview:** Loads user-defined topic/settings and calls NewsAPI “everything” endpoint to fetch relevant articles.  
**Nodes involved:** Set user config → Fetch news articles

#### Node: **Set user config**
- **Type / role:** Set — produces a configuration object (query, API key, recipient email, sheet URL, etc.).
- **Configuration choices:**
  - Uses **raw JSON mode** to output a single JSON object with keys such as `query`, `lookback_days`, `apiKey`, `recipient_email`, `table_URL`, `from`.
  - Sets `"from"` to `{{ $json.formattedDate }}` from the Format date node.
- **Key expressions / variables:**
  - `from: "{{ $json.formattedDate }}"`
- **Outputs to:** Fetch news articles
- **Critical issue (must fix to reproduce):**
  - The JSON shown is **syntactically invalid**:
    - Missing opening quote before `apiKey`
    - Trailing commas
    - Unquoted placeholder `GOOGLE_TABLE_URL_HERE`
  - This node will fail until corrected in n8n.
- **Edge cases:**
  - If `apiKey` or `query` is blank → NewsAPI call returns 401/400.
  - If `table_URL` is wrong → Google Sheets nodes later fail.

#### Node: **Fetch news articles**
- **Type / role:** HTTP Request — calls NewsAPI.
- **Configuration:**
  - URL template:  
    `https://newsapi.org/v2/everything?q={{ $json.query }}&from={{ $json.from }}&sortBy=relevancy&apiKey={{ $json.apiKey }}`
  - No additional options shown (timeouts/retry default).
- **Outputs to:** Understand number of results
- **Edge cases / failures:**
  - 401 Unauthorized (bad API key)
  - 429 rate limiting (NewsAPI plan limits)
  - 400 if `from` format incorrect
  - Large response size or empty `articles` array

---

### Block 3 — Coverage Metrics Logging & Historic Baseline
**Overview:** Calculates today’s article volume, appends metrics to Google Sheets, fetches historic rows, converts them into a numeric history array, and computes an average coverage volume.  
**Nodes involved:** Understand number of results → (Save to Google Sheets, Get Google Sheets data, Merge) and Get Google Sheets data → Get historic values from Google Sheets → (Calculate average results, Merge)

#### Node: **Understand number of results**
- **Type / role:** Set — derives volume metrics from NewsAPI response.
- **Configuration:**
  - `date`: `{{ new Date().toISOString().slice(0,10) }}`
  - `coverageVolume`: `{{ $json["articles"] ? $json["articles"].length : 0 }}`
  - `totalResults`: `{{ $json["totalResults"] || $json["articles"]?.length || 0 }}`
  - `query`: `{{ $('Set user config').item.json.query }}`
- **Outputs to:**  
  - Save to Google Sheets  
  - Get Google Sheets data  
  - Merge (input index 0)
- **Edge cases:**
  - NewsAPI may return `totalResults` but only provide a subset of `articles` (pagination). Your `coverageVolume` will reflect fetched page size, not total universe.
  - If NewsAPI response structure changes or is error payload, expressions may evaluate to 0.

#### Node: **Save to Google Sheets**
- **Type / role:** Google Sheets — appends a new row (daily metrics log).
- **Configuration:**
  - Operation: **Append**
  - Document: by URL `{{$json.table_URL}}`
  - Sheet name/id: hardcoded `"0"` (first sheet tab by ID in many cases)
  - Columns mapped: `date`, `totalResults`, `coverageVolume`
- **Credential requirement:** Google Sheets OAuth2 or service account (depends on n8n setup).
- **Inputs:** from Understand number of results (thus expects `table_URL` to exist in that item).
- **Edge cases / failures:**
  - Permission denied to the sheet
  - Incorrect sheet ID (`0` may not exist)
  - `table_URL` missing because it was not carried forward properly

#### Node: **Get Google Sheets data**
- **Type / role:** Google Sheets — reads historical rows.
- **Configuration:**
  - Document by URL: `{{$json.table_URL}}`
  - Sheet name/id: `{{$json.sheet_ID}}` (expression)
- **Inputs:** from Understand number of results
- **Critical issue (likely):**
  - `sheet_ID` is **not set anywhere** in the workflow configuration object, so this expression will be undefined unless your sheet rows already contain `sheet_ID` (unlikely).
  - In practice you should set the sheet/tab in this node directly (or add `sheet_ID` to Set user config).
- **Outputs to:** Get historic values from Google Sheets
- **Edge cases / failures:**
  - Undefined sheet name/id → node fails
  - Wrong range / empty sheet → returns zero items

#### Node: **Get historic values from Google Sheets**
- **Type / role:** Code — normalizes Google Sheets rows into numeric history entries.
- **Configuration (logic):**
  - Builds `history` array from all incoming items:
    - keeps rows where `date` and `coverageVolume` exist
    - converts volumes to `Number(...)`
    - `totalResults` defaults to `coverageVolume` if absent
  - Returns one item: `{ history: [...] }`
- **Outputs to:**  
  - Calculate average results  
  - Merge (input index 2)
- **Edge cases:**
  - If sheet is empty → history becomes `[]`
  - Non-numeric values → `Number(...)` becomes `NaN` and breaks average math downstream unless handled

#### Node: **Calculate average results**
- **Type / role:** Code — calculates average coverage volume from `history`.
- **Configuration (logic):**
  - `history = items[0].json.history`
  - `averageVolume = sum(coverageVolume)/history.length`
  - Returns `{ averageCoverageVolume: averageVolume }`
- **Outputs to:** Merge (input index 1)
- **Edge cases / failures:**
  - **Division by zero** when `history.length === 0` → `Infinity` or error-like output.
  - `NaN` propagation if any `coverageVolume` is `NaN`.

---

### Block 4 — Data Merge for AI Context
**Overview:** Combines three parallel data streams into a single item so the AI agent can see today’s metrics plus historic context plus computed average.  
**Nodes involved:** Merge

#### Node: **Merge**
- **Type / role:** Merge — combines multiple inputs by position.
- **Configuration:**
  - Mode: `combine`
  - Combine by: `position`
  - Number of inputs: **3**
- **Inputs:**
  1. From Understand number of results (index 0)
  2. From Calculate average results (index 1)
  3. From Get historic values from Google Sheets (index 2)
- **Outputs to:** AI Agent
- **Edge cases:**
  - If any branch produces 0 items, “combine by position” may yield no merged output (depending on execution and item counts).
  - Mismatched item counts across branches can produce unexpected pairing.

---

### Block 5 — AI Briefing Generation (with optional deep-dive)
**Overview:** Uses GPT‑4.1‑mini via an AI Agent node to create a short HTML email briefing. If anomaly is detected from historic volumes, the agent may call a tool that performs an HTTP request for additional topic research.  
**Nodes involved:** AI Agent, OpenAI Chat Model, Simple Memory, Topic deep-dive

#### Node: **OpenAI Chat Model**
- **Type / role:** LangChain Chat Model wrapper for OpenAI.
- **Configuration:**
  - Model: `gpt-4.1-mini`
  - Max retries: 2
- **Connects to:** AI Agent via `ai_languageModel`
- **Credential requirement:** OpenAI API credential in n8n.
- **Edge cases:**
  - 401/403 invalid OpenAI key
  - Model not available in your region/account
  - Rate limits / timeouts

#### Node: **Simple Memory**
- **Type / role:** LangChain memory (buffer window).
- **Configuration:**
  - `sessionKey`: `{{ $('Trigger daily').item.json.timestamp }}`
  - Uses custom key mode.
- **Connects to:** AI Agent via `ai_memory`
- **Edge cases:**
  - If `timestamp` missing, session key becomes empty → memory collisions across runs.

#### Node: **Topic deep-dive**
- **Type / role:** HTTP Request Tool — callable by the AI Agent.
- **Configuration:**
  - URL comes from the agent at runtime via `$fromAI('URL', ...)`.
  - Description: “Makes an HTTP request and returns the response data.”
- **Connects to:** AI Agent via `ai_tool`
- **Edge cases / failures:**
  - Agent may produce invalid URL
  - Tool may return large payloads (token pressure)
  - NewsAPI may rate-limit or reject if key is wrong

#### Node: **AI Agent**
- **Type / role:** LangChain Agent — orchestrates prompt + tools + model to generate final HTML.
- **Configuration choices (interpreted):**
  - Prompt is fully defined in the node (“define” mode).
  - Max iterations: 3 (limits tool-calling loops).
  - Instructs:
    - Select top 10 articles from provided dataset
    - Perform anomaly detection vs. historical volume (`history`) and use tool if needed
    - Output HTML under 2000 characters
- **Key expressions/variables referenced in prompt:**
  - `$('Set user config').item.json.query`
  - `$('Fetch news articles').item.json.totalResults`
  - `$('Get historic values from Google Sheets').item.json.history`
  - `$('Set timeframe to last 2 days').item.json.newDate`
  - `$('Trigger daily').item.json['Readable date']`
  - `$json.coverageVolume` and `$json.averageCoverageVolume` (from merged item)
  - `$('Fetch news articles').item.json.articles` as DATA INPUT
- **Important security/config issue:**
  - The prompt contains a **hardcoded NewsAPI key** in the anomaly research URL (`apiKey=7199...`). This should be replaced with `{{ $('Set user config').item.json.apiKey }}` or a credential/env var to avoid leaking secrets and to keep keys consistent.
- **Input / output:**
  - Input: merged item (metrics + average + history), plus it references the NewsAPI node output by expression.
  - Output: an `output` field (typical for AI Agent) containing the HTML email body.
- **Edge cases:**
  - If merged item missing `averageCoverageVolume` or `history`, anomaly logic becomes unreliable.
  - Character limit (2000 chars) can cause truncation or omission of articles.
  - If `articles` array is empty, “Top Articles” should handle gracefully (prompt does not explicitly enforce fallback).
  - `Readable date` may be undefined → odd header/subject.

---

### Block 6 — Email Output
**Overview:** Maps AI output into a `newsContent` field and sends an email via Gmail.  
**Nodes involved:** Strip message before sending → Send the email (Gmail)

#### Node: **Strip message before sending**
- **Type / role:** Set — renames/maps AI Agent output to the email body field.
- **Configuration:**
  - `newsContent = {{ $json.output }}`
- **Outputs to:** Send the email (Gmail)
- **Edge cases:**
  - If AI Agent returns a different key than `output`, email body becomes blank.

#### Node: **Send the email (Gmail)**
- **Type / role:** Gmail — sends an email.
- **Configuration:**
  - To: `"###"` (placeholder; should be your recipient, ideally `{{ $('Set user config').item.json.recipient_email }}`)
  - Subject: `Media Briefing - {{ $('Trigger daily').item.json['Readable date'] }}`
  - Message/body: `{{ $json.newsContent }}`
- **Credential requirement:** Gmail OAuth2 credential in n8n.
- **Edge cases / failures:**
  - Invalid/expired OAuth token
  - `sendTo` placeholder not replaced
  - Gmail may strip some HTML/CSS; inline CSS is generally fine but not guaranteed

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note8 | Sticky Note | Documentation / instructions | — | — | # How this works; Configuration; credentials list |
| Sticky Note1 | Sticky Note | Section header: setup | — | — | Setup |
| Trigger daily | Schedule Trigger | Daily entry point | — | Set timeframe to last 2 days | Setup |
| Set timeframe to last 2 days | Date & Time | Compute lookback start date | Trigger daily | Format date | Setup |
| Format date | Date & Time | Format date for NewsAPI | Set timeframe to last 2 days | Set user config | Setup |
| Set user config | Set | Define query/topic + keys + targets | Format date | Fetch news articles | Setup |
| Sticky Note4 | Sticky Note | Section header: fetch news | — | — | Fetch news article |
| Fetch news articles | HTTP Request | Call NewsAPI to fetch articles | Set user config | Understand number of results | Fetch news article |
| Understand number of results | Set | Compute coverage volume and totals | Fetch news articles | Save to Google Sheets; Get Google Sheets data; Merge | Handling historic data |
| Sticky Note | Sticky Note | Section header: historical data handling | — | — | Handling historic data |
| Save to Google Sheets | Google Sheets | Append today’s metrics | Understand number of results | — | Handling historic data |
| Get Google Sheets data | Google Sheets | Read historic rows | Understand number of results | Get historic values from Google Sheets | Handling historic data |
| Get historic values from Google Sheets | Code | Normalize historic rows to numeric history array | Get Google Sheets data | Calculate average results; Merge | Handling historic data |
| Calculate average results | Code | Compute average coverage volume | Get historic values from Google Sheets | Merge | Handling historic data |
| Sticky Note2 | Sticky Note | Section header: AI analysis | — | — | Analysis & deep-dive |
| Merge | Merge | Combine metrics + average + history | Understand number of results; Calculate average results; Get historic values from Google Sheets | AI Agent | Analysis & deep-dive |
| OpenAI Chat Model | OpenAI Chat Model | LLM used by agent | — | AI Agent | Analysis & deep-dive |
| Simple Memory | Memory Buffer Window | Session memory for agent | — | AI Agent | Analysis & deep-dive |
| Topic deep-dive | HTTP Request Tool | Optional tool for anomaly research | — | AI Agent | Analysis & deep-dive |
| AI Agent | AI Agent | Generates HTML briefing; may call tool | Merge (+ model/memory/tool connections) | Strip message before sending | Analysis & deep-dive |
| Sticky Note3 | Sticky Note | Section header: output | — | — | Output |
| Strip message before sending | Set | Map AI output to email body field | AI Agent | Send the email (Gmail) | Output |
| Send the email (Gmail) | Gmail | Send HTML briefing email | Strip message before sending | — | Output |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Trigger daily” (Schedule Trigger)**
   - Set it to run **once per day** at your desired time.
   - (Optional but recommended) Add a preceding Set node to compute:
     - `timestamp: {{$now}}`
     - `Readable date: {{ $now.format('MMMM d, yyyy') }}`  
     so later expressions don’t reference missing fields.

2) **Create “Set timeframe to last 2 days” (Date & Time)**
   - Operation: **Subtract from date**
   - Base date: `{{$json.timestamp}}` (or simply `{{$now}}` if you add no timestamp field)
   - Duration: `2` days
   - Output should include a `newDate`.

3) **Create “Format date” (Date & Time)**
   - Operation: **Format date**
   - Date: `{{$json.newDate}}`
   - Format: `yyyy-MM-dd`
   - Output should include `formattedDate`.

4) **Create “Set user config” (Set node, Raw JSON mode)**
   - Output a valid JSON object, for example:
     - `query` (string): topic keyword/company
     - `apiKey` (string): NewsAPI key
     - `recipient_email` (string): briefing recipient
     - `table_URL` (string): Google Sheets document URL
     - `sheet_ID` (string or numeric as required by Google Sheets node) **(add this; the workflow expects it)**
     - `from` (string): `{{$json.formattedDate}}`
   - Ensure the JSON is valid (no trailing commas; all keys quoted).

5) **Create “Fetch news articles” (HTTP Request)**
   - Method: GET
   - URL:
     - `https://newsapi.org/v2/everything`
   - Query parameters:
     - `q = {{$json.query}}`
     - `from = {{$json.from}}`
     - `sortBy = relevancy`
     - `apiKey = {{$json.apiKey}}`
   - Connect: Set user config → Fetch news articles

6) **Create “Understand number of results” (Set)**
   - Add fields:
     - `date = {{ new Date().toISOString().slice(0,10) }}`
     - `coverageVolume = {{ $json.articles ? $json.articles.length : 0 }}`
     - `totalResults = {{ $json.totalResults || $json.articles?.length || 0 }}`
     - `query = {{ $('Set user config').item.json.query }}`
     - Also carry forward `table_URL` and `sheet_ID` (either via “Keep Only Set” = false, or explicitly set them again).

7) **Create “Save to Google Sheets” (Google Sheets)**
   - Credentials: connect Google Sheets OAuth2 (or service account).
   - Operation: **Append**
   - Document: by URL `{{$json.table_URL}}`
   - Sheet: pick the correct tab (don’t rely on `"0"` unless confirmed)
   - Map columns: `date`, `coverageVolume`, `totalResults`
   - Connect: Understand number of results → Save to Google Sheets

8) **Create “Get Google Sheets data” (Google Sheets)**
   - Credentials: same as above.
   - Operation: **Read / Get Many** (depending on node UI)
   - Document: by URL `{{$json.table_URL}}`
   - Sheet: use a known sheet/tab ID or name (recommended: set directly in node; or `{{$json.sheet_ID}}` if you configured it)
   - Connect: Understand number of results → Get Google Sheets data

9) **Create “Get historic values from Google Sheets” (Code)**
   - Paste logic that:
     - Iterates incoming rows
     - Filters rows that have `date` and `coverageVolume`
     - Converts volumes to numbers
     - Returns a single item with `{ history: [...] }`
   - Connect: Get Google Sheets data → Get historic values from Google Sheets

10) **Create “Calculate average results” (Code)**
   - Compute:
     - `averageCoverageVolume = sum(history.coverageVolume) / history.length`
   - Add a guard:
     - If `history.length === 0`, set average to `0` (recommended)
   - Connect: Get historic values from Google Sheets → Calculate average results

11) **Create “Merge” (Merge)**
   - Mode: **Combine**
   - Combine by: **Position**
   - Number of inputs: **3**
   - Connect inputs:
     1. Understand number of results → Merge (Input 1 / index 0)
     2. Calculate average results → Merge (Input 2 / index 1)
     3. Get historic values from Google Sheets → Merge (Input 3 / index 2)

12) **Create AI components**
   - **OpenAI Chat Model**
     - Model: `gpt-4.1-mini`
     - Credentials: OpenAI API key
   - **Simple Memory**
     - Session key: `{{ $('Trigger daily').item.json.timestamp }}` (or another stable daily key)
   - **Topic deep-dive (HTTP Request Tool)**
     - Leave URL to be provided by AI (`$fromAI('URL', ...)`)

13) **Create “AI Agent” (AI Agent)**
   - Prompt: replicate the workflow’s instruction (media analyst role, top 10, anomaly detection, HTML format, <2000 chars).
   - Tool usage:
     - Attach **Topic deep-dive** as a tool
   - Model:
     - Attach **OpenAI Chat Model**
   - Memory:
     - Attach **Simple Memory**
   - Important fix:
     - Replace any hardcoded NewsAPI key in the prompt with a reference to your config (or instruct the agent to use the tool with a URL you generate safely).

14) **Create “Strip message before sending” (Set)**
   - `newsContent = {{$json.output}}`
   - Connect: AI Agent → Strip message before sending

15) **Create “Send the email (Gmail)” (Gmail)**
   - Credentials: Gmail OAuth2
   - To: `{{ $('Set user config').item.json.recipient_email }}`
   - Subject: `Media Briefing - {{ $('Trigger daily').item.json['Readable date'] }}`
   - Message body: `{{$json.newsContent}}`
   - Connect: Strip message before sending → Send the email (Gmail)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow finds articles, summarizes them and sends the findings via email… saves roughly 1 hour of article research & writing…” | Workflow description sticky note (“How this works”) |
| Configuration guidance: Use “Set user config” to input topic and credentials; required credentials include Google Sheets (read/write), OpenAI model, Gmail send | Sticky note (“How this works”) |
| Section labels: Setup / Handling historic data / Analysis & deep-dive / Output | Sticky notes used to visually group the workflow |
| Security note: avoid hardcoding NewsAPI keys inside AI prompts or tool URLs | Derived from AI Agent prompt containing a fixed `apiKey=...` |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.