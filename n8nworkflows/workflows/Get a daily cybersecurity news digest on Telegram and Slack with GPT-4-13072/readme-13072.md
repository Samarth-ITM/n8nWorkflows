Get a daily cybersecurity news digest on Telegram and Slack with GPT-4

https://n8nworkflows.xyz/workflows/get-a-daily-cybersecurity-news-digest-on-telegram-and-slack-with-gpt-4-13072


# Get a daily cybersecurity news digest on Telegram and Slack with GPT-4

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow runs daily and produces a concise cybersecurity news digest. It aggregates articles from multiple news APIs, normalizes them into a single schema, captures any upstream API errors, asks GPT‑4 to select and summarize the top 5 items ranked by severity (Critical/High/Medium), then distributes the digest to **Telegram**, **Slack**, **Gmail**, and archives it to **Google Sheets**.

**Primary use cases:**
- SOC / security team daily briefings
- leadership-level summaries with severity prioritization
- automated monitoring with multi-source redundancy and failure notes

### 1.1 Scheduling & Fan-out Trigger
Runs once per day at a fixed hour, then triggers all sources in parallel.

### 1.2 News Collection (3 sources)
Fetches up to 20 matching articles per source, with request timeouts and “continue on fail” behavior.

### 1.3 Source Error Detection & Error Object Creation
Each source is checked for HTTP/errors; failures are converted into a normalized `error` record per source so the AI can mention missing sources.

### 1.4 Merge + Normalization + Context Assembly
Successful responses are merged, articles are normalized into a unified schema, then merged with collected errors into a single AI-ready payload.

### 1.5 AI Threat Digest Generation (GPT‑4)
A LangChain Agent node uses GPT‑4 as the chat model to select, rank by severity, and summarize the top 5 stories, and append a one-line source-failure note if needed.

### 1.6 Output Formatting + Delivery + Archiving
Builds a final message string for Telegram/Slack/Email and appends a row into Google Sheets.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Fan-out Trigger
**Overview:** Starts the workflow every day at a configured hour and triggers three HTTP source fetches in parallel.  
**Nodes involved:** `Daily Schedule Trigger`

#### Node: Daily Schedule Trigger
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — workflow entry point.
- **Configuration (interpreted):** Runs daily at **10:00** (instance timezone).
- **Key variables/expressions:** none.
- **Input / output:**
  - **Outputs to:** `GNews API`, `NewsAPI`, `Reddit Cybersecurity` (parallel).
- **Version notes:** Type version `1.3`.
- **Edge cases/failures:**
  - Timezone mismatches (n8n instance vs desired audience timezone).
  - If the instance is paused/down at the scheduled time, execution timing depends on n8n scheduling behavior (missed vs delayed).

---

### Block 2 — News Collection (Multi-source)
**Overview:** Pulls cybersecurity-related articles from three providers using HTTP Request nodes with 10s timeouts and query-string parameters.  
**Nodes involved:** `GNews API`, `NewsAPI`, `Reddit Cybersecurity`

#### Node: GNews API
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — fetch articles from GNews.
- **Configuration choices:**
  - GET `https://gnews.io/api/v4/search`
  - Query params: `q`, `lang=en`, `max=20`, `apikey=…`
  - Timeout: 10,000 ms
  - **continueOnFail:** enabled; also `onError: continueRegularOutput` to keep the workflow moving.
- **Key expressions/variables:** none.
- **Connections:** Output → `If`
- **Version notes:** type version `4.3`.
- **Edge cases/failures:**
  - 401/403 for invalid API key; 429 rate limiting; 5xx upstream outages.
  - “Free plan delay” messaging from GNews (not a failure but affects freshness).
  - Schema differences: some responses may not contain `articles`.

#### Node: NewsAPI
- **Type / role:** HTTP Request — fetch articles from NewsAPI.
- **Configuration:**
  - GET `https://newsapi.org/v2/everything`
  - Query params: `q`, `pageSize=20`, `sortBy=publishedAt`, `apikey=…`
  - Timeout 10,000 ms; `continueOnFail: true`
- **Connections:** Output → `If1`
- **Version notes:** type version `4.3`.
- **Edge cases/failures:**
  - NewsAPI free plan limitations (sources/domains restrictions, rate limits).
  - Response may include `status: "error"` while HTTP 200; current workflow only checks `$json.error`/`statusCode`, so some logical errors might pass as “success” unless mapped into `$json.error`.

#### Node: Reddit Cybersecurity (actually WorldNewsAPI)
- **Type / role:** HTTP Request — despite the name, it queries WorldNewsAPI.
- **Configuration:**
  - GET `https://api.worldnewsapi.com/search-news`
  - Query params: `text`, `language=en`, `api-key=…`
  - Timeout 10,000 ms; `continueOnFail: true`
- **Connections:** Output → `If2`
- **Version notes:** type version `4.3`.
- **Edge cases/failures:**
  - Provider response schema likely differs (may not return `articles` at top level).
  - If provider uses different fields (e.g., `news` or `articles`), normalization may produce zero items.

---

### Block 3 — Source Error Detection & Error Normalization
**Overview:** Each source response is evaluated; failures become standardized error records (source, error, statusCode, timestamp). Successful responses proceed to merging articles.  
**Nodes involved:** `If`, `If1`, `If2`, `Code in JavaScript`, `Code in JavaScript1`, `Code in JavaScript2`, `Merge`, `Merge1`

#### Node: If (GNews error check)
- **Type / role:** IF (`n8n-nodes-base.if`) — routing based on error/HTTP status.
- **Configuration (interpreted):**
  - Condition is **OR**:
    - String “exists”: `{{$json.error}}` exists, OR
    - Number `>= 400`: `{{$json.statusCode}} >= 400`
- **Outputs:**
  - **True (has error):** → `Code in JavaScript`
  - **False (ok):** → `Merge` (input index 0)
- **Version notes:** type version `2.3`.
- **Edge cases:**
  - Some HTTP nodes return provider-specific error structures (not always `$json.error`).
  - If `statusCode` is undefined and no `$json.error`, a failed logical response could slip through.

#### Node: If1 (NewsAPI error check)
- Same logic as `If`.
- **Outputs:**
  - True → `Code in JavaScript1`
  - False → `Merge` (input index 1)

#### Node: If2 (WorldNewsAPI error check)
- Same logic as `If`.
- **Outputs:**
  - True → `Code in JavaScript2`
  - False → `Merge` (input index 2)

#### Node: Code in JavaScript (GNews error record)
- **Type / role:** Code (`n8n-nodes-base.code`) — converts failure into a normalized error item.
- **Config choices:** Returns a single item:
  - `source: "GNews"`
  - `error: $json.error?.message || 'Request failed'`
  - `statusCode: $json.statusCode || null`
  - `timestamp: now ISO`
- **Connections:** Output → `Merge1` (input 0)
- **Version notes:** type version `2`.
- **Edge cases:** If upstream error message is not in `$json.error.message`, message becomes “Request failed”.

#### Node: Code in JavaScript1 (NewsAPI error record)
- Same as above but `source: "NewsAPI"`.
- Output → `Merge1` (input 1)

#### Node: Code in JavaScript2 (World News error record)
- Same as above but `source: "World News"`.
- Output → `Merge1` (input 2)

#### Node: Merge (merge successful source responses)
- **Type / role:** Merge (`n8n-nodes-base.merge`) — collects successful source outputs.
- **Configuration:** `numberInputs: 3` (expects up to 3 incoming success branches).
- **Connections:** Output → `Normalize Articles`
- **Version notes:** type version `3.2`.
- **Edge cases:**
  - If only 1–2 sources succeed, merge behavior is still fine, but downstream normalization may have fewer items.
  - If a “success” response doesn’t contain `articles`, normalization yields no normalized entries.

#### Node: Merge1 (merge error records)
- **Type / role:** Merge — collects error items from any failed sources.
- **Configuration:** `numberInputs: 3`
- **Connections:** Output → `Merge Articles + Errors`
- **Edge cases:** If no errors occur, this node may receive no input at all (depends on execution paths). In that scenario, the downstream merge strategy matters.

---

### Block 4 — Normalization & Context Assembly
**Overview:** Converts heterogeneous article lists into a consistent schema, merges normalized articles with normalized error records, and builds the final AI context object.  
**Nodes involved:** `Normalize Articles`, `Merge Articles + Errors`, `Final Context Builder`

#### Node: Normalize Articles
- **Type / role:** Code — flattens and standardizes article fields across sources.
- **Configuration (logic):**
  - For each incoming item (each source response):
    - Determine `source` from `item.json.source || item.json.provider || item.json.news || 'Unknown'`
    - Read `item.json.articles || []`
    - For each article: emit `{source,title,description,content,url,image}`
- **Connections:** Output → `Merge Articles + Errors` (input index 1)
- **Version notes:** type version `2`.
- **Edge cases/failures:**
  - If a provider returns `news` instead of `articles`, articles will be `[]` and nothing is emitted.
  - Titles/descriptions may be empty; downstream AI may produce weak results.
  - No deduplication is implemented despite the “Workflow Overview” note mentioning Levenshtein/80% (that logic is not present in this JSON).

#### Node: Merge Articles + Errors
- **Type / role:** Merge — combines two streams: errors (from `Merge1`) and normalized articles.
- **Configuration:** default merge settings (not explicitly set).
- **Connections:** Output → `Final Context Builder`
- **Version notes:** type version `3.2`.
- **Edge cases:**
  - If only one side emits (all sources success or all sources fail), you must ensure the merge mode supports “pass-through”; default behavior can be surprising depending on node settings/version.

#### Node: Final Context Builder
- **Type / role:** Code — constructs one single payload for the AI agent.
- **Configuration (logic):**
  - Iterates over items:
    - If `item.json.title` exists → push into `articles[]`
    - Else if `item.json.error` exists → push into `errors[]`
  - Outputs one item:
    - `{ articles, errors, generatedAt }`
- **Connections:** Output → `AI Threat Analysis Agent`
- **Version notes:** type version `2`.
- **Edge cases:**
  - Articles without `title` will be misclassified (dropped unless they have `error`).
  - If merge produced unexpected shapes (e.g., nested arrays), arrays may be empty.

---

### Block 5 — AI Threat Digest Generation (GPT‑4 via LangChain)
**Overview:** Sends the normalized context to an AI agent configured with a cybersecurity analyst system prompt. GPT‑4 selects the top 5 stories, writes short summaries/headlines, ranks by severity, and notes failed sources.  
**Nodes involved:** `OpenAI GPT-4`, `AI Threat Analysis Agent`

#### Node: OpenAI GPT-4
- **Type / role:** LangChain Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides the LLM used by the agent.
- **Configuration:**
  - Model: `gpt-4-turbo-preview`
  - `maxTokens: 2000`, `temperature: 0.3`
- **Credentials:** `OpenAi account 2` (OpenAI API credential).
- **Connections:** Provides `ai_languageModel` connection to `AI Threat Analysis Agent`.
- **Version notes:** type version `1`.
- **Edge cases/failures:**
  - Invalid API key / quota exceeded.
  - Model name deprecation (preview models can change). Consider using a stable GPT‑4o / GPT‑4.1 equivalent when available in your n8n setup.

#### Node: AI Threat Analysis Agent
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — prompt orchestration and final output.
- **Configuration choices:**
  - **System message:** “professional cybersecurity news analyst”, with strict non-hallucination and severity ranking requirement (Critical/High/Medium).
  - **User text input:** embeds:
    - `Articles: {{ JSON.stringify($json.articles) }}`
    - `Errors: {{ JSON.stringify($json.errors) }}`
    - Task instructions (select 5, summarize 2–3 lines, add headline, add failure note).
- **Inputs:**
  - Main input from `Final Context Builder`
  - LLM input from `OpenAI GPT-4` (ai_languageModel connection)
- **Outputs:** Main output → `Manipulating Outputs`
- **Version notes:** type version `1`.
- **Edge cases:**
  - Token bloat: stringifying 60 articles (3×20) with full `content` may exceed context limits. Consider trimming fields or limiting to `title/description/url`.
  - AI may output Markdown that breaks Slack/Telegram formatting if not constrained.

---

### Block 6 — Output Formatting, Delivery & Archiving
**Overview:** Wraps AI output into a final message, then sends it to Telegram/Slack/Gmail and archives to Google Sheets.  
**Nodes involved:** `Manipulating Outputs`, `Send to Telegram`, `Send to Slack`, `Send a message`, `Archive to Google Sheets`

#### Node: Manipulating Outputs
- **Type / role:** Code — converts agent output into delivery-ready objects.
- **Configuration (logic):**
  - Reads `const aiText = items[0].json.output;`
  - Builds `textMessage`:
    - Header: “🛡️ Daily Cybersecurity Brief”
    - Body: AI output
    - Footer: generated timestamp via `toLocaleString()`
  - Builds `sheetRow` (but note: it is not used by the Sheets node in this workflow)
  - Returns:
    - `json.textMessage`
    - `json.sheetRow`
    - `json.metadata.generatedAt`
    - `json.metadata.length`
- **Connections:** Output fan-out to `Send to Telegram`, `Send to Slack`, `Archive to Google Sheets`, `Send a message`.
- **Version notes:** type version `2`.
- **Edge cases/failures:**
  - Assumes AI agent output is at `items[0].json.output`. If the agent node outputs under a different property (e.g., `text`), deliveries will be empty.
  - Slack node expects `$json.digest`, but this node does **not** produce `digest`. This is a functional mismatch (see Slack & Sheets below).

#### Node: Send to Telegram
- **Type / role:** Telegram (`n8n-nodes-base.telegram`) — sends message to a chat.
- **Configuration:**
  - Text: `={{ $json.textMessage }}`
  - `chatId: YOUR_TELEGRAM_CHAT_ID` (must be replaced)
  - Parse mode: Markdown
  - Append attribution: false
- **Credentials:** `Telegram account` (Telegram bot token).
- **Version notes:** type version `1.2`.
- **Edge cases:**
  - Wrong chat ID / bot not in group.
  - Markdown parse errors if AI outputs incompatible markdown.

#### Node: Send to Slack
- **Type / role:** Slack (`n8n-nodes-base.slack`) — posts message to a channel.
- **Configuration:**
  - Authentication: OAuth2
  - Target: channel (`YOUR_SLACK_CHANNEL_ID`)
  - Text: `={{ $json.digest }}`
- **Credentials:** not shown in JSON but required (Slack OAuth2).
- **Version notes:** type version `2.2`.
- **Important issue:** Upstream `Manipulating Outputs` produces `textMessage` but **not** `digest`. Slack will likely post an empty message unless `$json.digest` exists from earlier nodes.
- **Edge cases:**
  - Missing scopes (e.g., `chat:write`, channel access).
  - Slack formatting differences vs Markdown.

#### Node: Send a message (Gmail)
- **Type / role:** Gmail (`n8n-nodes-base.gmail`) — sends an email.
- **Configuration (as provided):**
  - Subject: “Hello”
  - Message: “Hello”
  - No dynamic usage of the digest content currently.
- **Credentials:** `Gmail account` (OAuth2).
- **Version notes:** type version `2.2`.
- **Edge cases:**
  - Gmail API scopes not granted for send.
  - Missing required fields (To/From) depending on node defaults; the current configuration is incomplete for a real send in most setups.

#### Node: Archive to Google Sheets
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — appends digest metadata to a sheet.
- **Configuration:**
  - Operation: Append to `Sheet1` in `YOUR_GOOGLE_SHEET_ID`
  - Column mapping:
    - Date = `{{$now.format('yyyy-MM-dd')}}`
    - Digest = `{{$json.digest}}`
    - Timestamp = `{{$now.format('yyyy-MM-dd HH:mm:ss')}}`
    - Article Count = `{{ $('Prepare Data for AI').item.json.articleCount }}`
    - Character Count = `{{$json.digestLength}}`
- **Credentials:** `Google Sheets account` (OAuth2).
- **Version notes:** type version `4.5`.
- **Important issues:**
  - `$json.digest` and `$json.digestLength` are not produced by `Manipulating Outputs`.
  - Node references `$('Prepare Data for AI')` which does **not exist** in this workflow JSON; this will cause an expression error at runtime.
- **Edge cases:**
  - Sheet column headers must match exactly (`Date`, `Digest`, `Timestamp`, etc.).
  - Permissions: the OAuth user must have edit access to the sheet.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Schedule Trigger | scheduleTrigger | Daily entry point | — | GNews API; NewsAPI; Reddit Cybersecurity |  |
| GNews API | httpRequest | Fetch news articles (GNews) | Daily Schedule Trigger | If | ##  News Sources<br>Fetching from multiple APIs with error handling |
| NewsAPI | httpRequest | Fetch news articles (NewsAPI) | Daily Schedule Trigger | If1 | ##  News Sources<br>Fetching from multiple APIs with error handling |
| Reddit Cybersecurity | httpRequest | Fetch news articles (WorldNewsAPI) | Daily Schedule Trigger | If2 | ##  News Sources<br>Fetching from multiple APIs with error handling |
| If | if | Route GNews success vs error | GNews API | Code in JavaScript; Merge | ## Handeling Errors |
| If1 | if | Route NewsAPI success vs error | NewsAPI | Code in JavaScript1; Merge | ## Handeling Errors |
| If2 | if | Route WorldNewsAPI success vs error | Reddit Cybersecurity | Code in JavaScript2; Merge | ## Handeling Errors |
| Code in JavaScript | code | Build normalized error record (GNews) | If | Merge1 | ## Handeling Errors |
| Code in JavaScript1 | code | Build normalized error record (NewsAPI) | If1 | Merge1 | ## Handeling Errors |
| Code in JavaScript2 | code | Build normalized error record (World News) | If2 | Merge1 | ## Handeling Errors |
| Merge | merge | Merge successful source payloads | If; If1; If2 | Normalize Articles | ##  Data Processing<br>Deduplication, normalization & error handling |
| Normalize Articles | code | Normalize articles into unified schema | Merge | Merge Articles + Errors | ##  Data Processing<br>Deduplication, normalization & error handling |
| Merge1 | merge | Merge error records | Code in JavaScript; Code in JavaScript1; Code in JavaScript2 | Merge Articles + Errors | ## Handeling Errors |
| Merge Articles + Errors | merge | Combine normalized articles with error items | Merge1; Normalize Articles | Final Context Builder | ##  Data Processing<br>Deduplication, normalization & error handling |
| Final Context Builder | code | Build `{articles, errors}` AI payload | Merge Articles + Errors | AI Threat Analysis Agent |  |
| OpenAI GPT-4 | lmChatOpenAi | LLM provider for agent | — (AI connection) | AI Threat Analysis Agent |  AI Analysis<br>GPT-4 powered threat intelligence analysis with categorization |
| AI Threat Analysis Agent | agent | Select top 5 + summarize + severity rank | Final Context Builder; OpenAI GPT-4 | Manipulating Outputs |  AI Analysis<br>GPT-4 powered threat intelligence analysis with categorization |
| Manipulating Outputs | code | Format final message + metadata | AI Threat Analysis Agent | Send to Telegram; Send to Slack; Archive to Google Sheets; Send a message | ## 📤 Multi-Channel Delivery<br>Telegram, Slack, Email + Archive to Google Sheets |
| Send to Telegram | telegram | Send digest to Telegram chat | Manipulating Outputs | — | ## 📤 Multi-Channel Delivery<br>Telegram, Slack, Email + Archive to Google Sheets |
| Send to Slack | slack | Send digest to Slack channel | Manipulating Outputs | — | ## 📤 Multi-Channel Delivery<br>Telegram, Slack, Email + Archive to Google Sheets |
| Send a message | gmail | Send email (currently static) | Manipulating Outputs | — | ## 📤 Multi-Channel Delivery<br>Telegram, Slack, Email + Archive to Google Sheets |
| Archive to Google Sheets | googleSheets | Append digest log row | Manipulating Outputs | — | ## 📤 Multi-Channel Delivery<br>Telegram, Slack, Email + Archive to Google Sheets |
| Sources Note | stickyNote | Comment | — | — |  |
| Processing Note | stickyNote | Comment | — | — |  |
| AI Note | stickyNote | Comment | — | — |  |
| Delivery Note | stickyNote | Comment | — | — |  |
| Workflow Overview | stickyNote | Comment / credits / requirements | — | — |  |
| AI Note1 | stickyNote | Comment | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it: *Get a daily cybersecurity news digest on Telegram and Slack with GPT-4*.

2. **Add trigger**
   1) Add node: **Schedule Trigger**  
   2) Set it to run **daily at 10:00** (adjust timezone/instance settings as needed).  
   3) This node will be the only entry point.

3. **Add the 3 HTTP source nodes (parallel from trigger)**
   1) Add **HTTP Request** node “GNews API”  
      - URL: `https://gnews.io/api/v4/search`  
      - Query params:  
        - `q = cybersecurity OR cyber attack OR data breach OR vulnerability`  
        - `lang = en`  
        - `max = 20`  
        - `apikey = <your_gnews_key>`  
      - Options: timeout 10000ms  
      - Enable **Continue On Fail**
   2) Add **HTTP Request** node “NewsAPI”  
      - URL: `https://newsapi.org/v2/everything`  
      - Query params:  
        - `q = cybersecurity OR "data breach" OR "security vulnerability"`  
        - `pageSize = 20`  
        - `sortBy = publishedAt`  
        - `apikey = <your_newsapi_key>`  
      - Timeout 10000ms; **Continue On Fail**
   3) Add **HTTP Request** node “Reddit Cybersecurity” (optional rename to “WorldNewsAPI”)  
      - URL: `https://api.worldnewsapi.com/search-news`  
      - Query params:  
        - `text = cybersecurity OR "data breach" OR "security vulnerability"`  
        - `language = en`  
        - `api-key = <your_worldnewsapi_key>`  
      - Timeout 10000ms; **Continue On Fail**
   4) Connect **Schedule Trigger → all 3 HTTP nodes**.

4. **Add per-source error routing (3 IF nodes)**
   - For each source node, add an **IF** node with condition (OR):
     - `{{$json.error}}` **exists**
     - `{{$json.statusCode}} >= 400`
   - Connect:
     - `GNews API → If`
     - `NewsAPI → If1`
     - `Reddit Cybersecurity → If2`

5. **Add error-normalization Code nodes (3)**
   - Add **Code** node “Code in JavaScript” (for GNews error) returning an object with:
     - `source`, `error`, `statusCode`, `timestamp`
   - Duplicate for NewsAPI and WorldNews:
     - “Code in JavaScript1” source `NewsAPI`
     - “Code in JavaScript2” source `World News`
   - Connect:
     - `If (true) → Code in JavaScript`
     - `If1 (true) → Code in JavaScript1`
     - `If2 (true) → Code in JavaScript2`

6. **Merge successful source responses**
   1) Add **Merge** node (set **Number of inputs = 3**).  
   2) Connect the **false** branch (no error) of each IF into this merge:
      - `If (false) → Merge input 0`
      - `If1 (false) → Merge input 1`
      - `If2 (false) → Merge input 2`

7. **Normalize articles**
   1) Add a **Code** node “Normalize Articles” that:
      - reads each merged source response
      - extracts `articles` arrays
      - emits one item per article with fields: `source,title,description,content,url,image`
   2) Connect `Merge → Normalize Articles`.

8. **Merge errors with normalized articles**
   1) Add **Merge** node “Merge1” (Number of inputs = 3) to combine the three error records.  
      - Connect: each error Code node → `Merge1` inputs 0–2.
   2) Add **Merge** node “Merge Articles + Errors” to combine:
      - errors stream from `Merge1`
      - article stream from `Normalize Articles`
   3) Connect:
      - `Merge1 → Merge Articles + Errors`
      - `Normalize Articles → Merge Articles + Errors`

9. **Build final AI context**
   1) Add **Code** node “Final Context Builder” that outputs a single item:
      - `articles: []`, `errors: []`, `generatedAt`
   2) Connect `Merge Articles + Errors → Final Context Builder`.

10. **Add GPT‑4 model node (LangChain)**
   1) Add **OpenAI Chat Model** node “OpenAI GPT-4”  
   2) Set model `gpt-4-turbo-preview` (or a stable GPT‑4 class model available in your instance)  
   3) Set temperature `0.3`, max tokens `2000`  
   4) Configure **OpenAI credentials** (API key).

11. **Add AI Agent**
   1) Add **AI Agent** node “AI Threat Analysis Agent”  
   2) Set System Message to the cybersecurity analyst instructions (severity ranking + no hallucinations).  
   3) Set the User Text to include:
      - `Articles: {{ JSON.stringify($json.articles) }}`
      - `Errors: {{ JSON.stringify($json.errors) }}`
      - tasks (top 5, 2–3 line summary, headlines, failure note)
   4) Connect:
      - Main: `Final Context Builder → AI Threat Analysis Agent`
      - AI Language Model: `OpenAI GPT-4 → AI Threat Analysis Agent` (via the model port)

12. **Format outputs**
   1) Add **Code** node “Manipulating Outputs” that:
      - reads the agent output (ensure the correct field name)
      - produces `textMessage` and metadata
   2) Connect `AI Threat Analysis Agent → Manipulating Outputs`.

13. **Delivery nodes**
   1) Add **Telegram** node “Send to Telegram”
      - Text: `{{$json.textMessage}}`
      - chatId: your chat ID
      - Credential: Telegram bot token
   2) Add **Slack** node “Send to Slack”
      - OAuth2 authentication
      - channelId set
      - Text should reference the produced field (recommend `{{$json.textMessage}}`)
   3) Add **Gmail** node “Send a message”
      - Configure **To**, **Subject**, and set Message to `{{$json.textMessage}}`
      - Credential: Gmail OAuth2 with send scope
   4) Add **Google Sheets** node “Archive to Google Sheets”
      - Append to your spreadsheet/sheet
      - Map columns to existing fields (recommend mapping to `textMessage` and `metadata.length`)
   5) Connect `Manipulating Outputs → all 4 delivery/archive nodes`.

> **Important adjustments to make it runnable (based on the provided JSON):**
> - Change Slack text from `{{$json.digest}}` to `{{$json.textMessage}}` (or output a `digest` field in the code node).
> - Fix Google Sheets expressions: replace missing `$('Prepare Data for AI')…` and undefined `$json.digestLength` with fields that exist (e.g., `$json.metadata.length`).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Advanced Cyber Threat Intelligence Digest System | Sticky note “Workflow Overview” |
| Created by: Abdullah Dilshad / iamabdullahdishad@gmail.com | Credits from “Workflow Overview” note |
| Mentions multi-source aggregation incl. Reddit r/cybersecurity, RSS, CISA, CVE DB | Present in “Workflow Overview” note; Reddit is not actually implemented (WorldNewsAPI is used instead) |
| Mentions smart deduplication (Levenshtein, 80% threshold) | Not implemented in the provided nodes; only normalization exists |
| Mentions admin alerts, retry logic, QA checks | Not implemented beyond `continueOnFail` and 10s timeouts |
| GNews free plan has delay; upgrade link appears in sample data | Example message: `https://gnews.io/change-plan` (from pinned data) |