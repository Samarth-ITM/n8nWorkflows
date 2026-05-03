Create a daily market brief from Google Sheets, Alpha Vantage, Reddit, OpenAI, and Slack

https://n8nworkflows.xyz/workflows/create-a-daily-market-brief-from-google-sheets--alpha-vantage--reddit--openai--and-slack-12944


# Create a daily market brief from Google Sheets, Alpha Vantage, Reddit, OpenAI, and Slack

## 1. Workflow Overview

**Purpose:** Generate and deliver a **daily market intelligence brief** to Slack by combining:
- **Portfolio holdings** from Google Sheets
- **Daily stock price movement** from Alpha Vantage
- **Recent market headlines** from Google News RSS
- **Retail sentiment signals** from Reddit r/stocks RSS
- **AI synthesis** (OpenAI) into a structured summary

**Target use cases:** daily trading/portfolio monitoring, market recap for teams, lightweight “what matters today” briefing, dashboard/alert feed.

### 1.1 Scheduling & Input Reception
Runs daily at a fixed time, then loads the list of stocks to analyze.

### 1.2 Per-Stock Market Data Collection (Price)
Iterates through each stock symbol and fetches latest daily price series from Alpha Vantage, then computes trend and % change.

### 1.3 News & Social Sentiment Collection
Fetches and normalizes:
- last 24h market headlines (Google News RSS)
- r/stocks posts (Reddit RSS)

### 1.4 AI Context Assembly & Analysis
Combines price + news + reddit into a single JSON context, sends it to OpenAI with strict formatting requirements.

### 1.5 Output Structuring & Slack Delivery
Parses the AI text into structured fields and posts a formatted message to Slack.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Portfolio Input
**Overview:** Triggers daily execution and retrieves portfolio holdings from a Google Sheet to provide the stock symbols input.  
**Nodes involved:** Daily Market Brief Trigger, Read portfolio holdings, Process each stock

#### Node: Daily Market Brief Trigger
- **Type / role:** Schedule Trigger — entry point; runs on a daily schedule.
- **Configuration:** Runs every day at **08:30** (instance timezone).
- **Outputs:** Feeds execution into “Read portfolio holdings”.
- **Edge cases / failures:**
  - Timezone mismatch (n8n instance timezone vs expected local time).
  - Workflow is **inactive** (`active: false`) so it won’t run until enabled.

#### Node: Read portfolio holdings
- **Type / role:** Google Sheets — reads rows from a specific sheet containing portfolio data (expects a “Stock” column used later).
- **Configuration choices:**
  - Document: Google Sheet at URL `https://docs.google.com/spreadsheets/d/1Mz-woYDtXtzF2bA9IqpdYh28IPA76nFx46WHgDJOZoI/edit`
  - Sheet (tab): “Portfolio Performance” (gid 1013617369).
  - Uses OAuth2 credentials `automations@techdome.ai`.
- **Inputs:** From Schedule Trigger.
- **Outputs:** One item per row (each row becomes an item).
- **Key dependency:** Downstream node expects each item to contain a field **`Stock`** (case-sensitive).
- **Edge cases / failures:**
  - OAuth token expiration / missing scopes.
  - Sheet renamed / gid changed / permissions revoked.
  - Missing `Stock` column or blank symbols → breaks Alpha Vantage query.

#### Node: Process each stock
- **Type / role:** Split In Batches — controls iterating through holdings.
- **Configuration choices:**
  - Uses default batch behavior (batch size not explicitly set).
  - Connected using **output index 1** to proceed to “Stock Price (Alpha Vantage)”.
- **Inputs:** Items from Google Sheets.
- **Outputs:**
  - **Main output (index 1)** goes to Alpha Vantage request for per-stock processing.
  - **Main output (index 0)** is unused in this workflow’s wiring.
- **Edge cases / failures:**
  - If batch size defaults are unexpected, execution may send items differently than intended.
  - No “loop-back” connection exists to continue batches; as wired, this typically results in processing behavior that may not iterate as expected depending on n8n SplitInBatches semantics/version. If you intend true iteration over all rows, you usually add a loop from the end of the per-item chain back into SplitInBatches to fetch the next batch.

---

### Block 2 — Stock Price Retrieval & Normalization
**Overview:** Fetches daily time series data for the current stock symbol and computes latest close, previous close, percent change, and a bullish/bearish/neutral label.  
**Nodes involved:** Stock Price (Alpha Vantage), Normalize Stock Data

#### Node: Stock Price (Alpha Vantage)
- **Type / role:** HTTP Request — calls Alpha Vantage API.
- **Configuration choices:**
  - Method: default GET (via query params)
  - URL: `https://www.alphavantage.co/query`
  - Query parameters:
    - `function=TIME_SERIES_DAILY`
    - `symbol={{ $json.Stock }}`
    - `outputsize=compact`
    - `apikey=YOUR_API_KEY` (placeholder; must be replaced)
- **Inputs:** One item representing a holding row; must have `$json.Stock`.
- **Outputs:** Alpha Vantage response JSON (includes `Meta Data` and `Time Series (Daily)` when successful).
- **Edge cases / failures:**
  - API key missing/invalid → error payload or throttling message.
  - Rate limiting (Alpha Vantage is strict on free tiers) → returns informational JSON without expected fields.
  - Symbol invalid/empty → response missing `Time Series (Daily)`.

#### Node: Normalize Stock Data
- **Type / role:** Code — computes key metrics from Alpha Vantage daily series.
- **Configuration choices (interpreted):**
  - Reads `$json["Time Series (Daily)"]`.
  - Sorts dates descending; uses the latest two trading days.
  - Computes:
    - `close` (latest close)
    - `previousClose`
    - `changePercent` rounded to 2 decimals (string)
    - `trend`: Bullish if >0, Bearish if <0 else Neutral
  - Returns a **single object** shaped like:
    - `symbol`, `date`, `close`, `previousClose`, `changePercent`, `trend`
- **Inputs:** Alpha Vantage JSON.
- **Outputs:** Normalized one-item structure, then flows into “RSS Read”.
- **Edge cases / failures:**
  - If `Time Series (Daily)` missing (rate limit, error, unexpected payload), code throws.
  - Market holidays/weekends: still fine if series contains at least two days; fails if only one day exists.
  - `changePercent` is returned as string due to `toFixed(2)`; downstream AI prompt is fine, but numeric assumptions elsewhere would fail.

---

### Block 3 — Market News & Reddit Sentiment Collection
**Overview:** Pulls market headlines and Reddit posts via RSS, normalizes them, and filters news to the last 24 hours.  
**Nodes involved:** RSS Read, Normalize Market News, Reddit Sentiment – RSS, Normalize Reddit News

#### Node: RSS Read
- **Type / role:** RSS Feed Read — retrieves Google News RSS entries.
- **Configuration choices:**
  - URL: `https://news.google.com/rss/search?q=stock+market+OR+earnings+OR+inflation+OR+interest+rates`
- **Inputs:** From “Normalize Stock Data”.
- **Outputs:** One item per RSS entry (title, pubDate, snippet, etc.).
- **Edge cases / failures:**
  - Google News RSS availability changes, throttling, or response format changes.
  - Network timeouts.

#### Node: Normalize Market News
- **Type / role:** Code — transforms RSS items and filters to last 24 hours.
- **Configuration choices:**
  - For each item:
    - `headline`: `title`
    - `summary`: `contentSnippet` or empty string
    - `source`: constant “Google News”
    - `publishedAt`: Date object parsed from `pubDate`
    - `ageHours`: integer hours old
  - Filters entries where `publishedAt` is within last 24 hours.
- **Inputs:** Items from RSS Read.
- **Outputs:** Filtered/normalized news items to “Reddit Sentiment – RSS”.
- **Edge cases / failures:**
  - Invalid or missing `pubDate` → `new Date()` becomes “Invalid Date” and filter/math can behave unexpectedly.
  - Timezone inconsistencies in `pubDate`.

#### Node: Reddit Sentiment – RSS
- **Type / role:** RSS Feed Read — retrieves r/stocks feed.
- **Configuration choices:**
  - URL: `https://www.reddit.com/r/stocks/.rss`
- **Inputs:** From Normalize Market News.
- **Outputs:** One item per Reddit entry.
- **Edge cases / failures:**
  - Reddit RSS rate limiting / blocking non-browser user agents (can be intermittent).
  - Content fields may vary; some entries may lack `contentSnippet`.

#### Node: Normalize Reddit News
- **Type / role:** Code — turns Reddit RSS entries into compact text signals.
- **Configuration choices:**
  - Produces objects with:
    - `platform`: “Reddit”
    - `source`: “r/stocks”
    - `text`: `title + " " + contentSnippet`
    - `publishedAt`: `pubDate` (kept as-is)
- **Inputs:** Reddit RSS items.
- **Outputs:** Normalized reddit items to “Prepare AI Context”.
- **Edge cases / failures:**
  - Missing title/snippet leads to reduced text quality (but won’t usually crash).
  - `publishedAt` remains string; if later date math is needed, it isn’t parsed here.

---

### Block 4 — AI Context Assembly & OpenAI Analysis
**Overview:** Aggregates the normalized stock metrics, market headlines, and reddit signals into a single JSON payload and asks OpenAI to generate a strict-format market brief.  
**Nodes involved:** Prepare AI Context, Ai Analysis

#### Node: Prepare AI Context
- **Type / role:** Code — consolidates data from multiple upstream nodes using `$items(nodeName)`.
- **Configuration choices:**
  - Returns **one single item**:
    - `stock`: first item from “Normalize Stock Data”
    - `news`: array of all items from “Normalize Market News”
    - `reddit`: array of all items from “Normalize Reddit News”
- **Inputs:** Receives the stream from “Normalize Reddit News”, but also reaches back to other nodes using `$items()`.
- **Outputs:** One combined context item to “Ai Analysis”.
- **Key expressions / variables:**
  - `$items("Normalize Stock Data")[0]?.json`
  - `$items("Normalize Market News").map(i => i.json)`
  - `$items("Normalize Reddit News").map(i => i.json)`
- **Edge cases / failures:**
  - If upstream nodes produced zero items, arrays may be empty; `stock` may become `undefined`.
  - If execution path didn’t actually run “Normalize Stock Data” in the same execution context (possible if batching/looping isn’t correct), `$items()` may not return expected data.

#### Node: Ai Analysis
- **Type / role:** OpenAI (LangChain) — generates structured brief.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Prompt content embeds:
    - `{{ JSON.stringify($json.stock, null, 2) }}`
    - `{{ JSON.stringify($json.news, null, 2) }}`
    - `{{ JSON.stringify($json.reddit, null, 2) }}`
  - Requires strict output format with headers:
    - **Market Summary**
    - **Sentiment**
    - **Key Drivers**
    - **Risks**
    - **Actionable Insight**
  - Instruction: concise, no disclaimers, no generic advice.
- **Credentials:** OpenAI account “OpenAi account 3”.
- **Inputs:** Single context item.
- **Outputs:** An OpenAI response object containing `output[0].content[0].text` (as referenced later).
- **Edge cases / failures:**
  - OpenAI auth errors, quota exceeded, model unavailable.
  - Prompt size: too many news/reddit items could exceed token limits; consider limiting counts.
  - The model may still deviate from strict formatting; downstream parsing tries to handle some markdown variance.

---

### Block 5 — Parse AI Output & Deliver to Slack
**Overview:** Extracts sections from the AI response text, normalizes bullet formatting, then posts the daily brief to Slack.  
**Nodes involved:** Parse AI Market Output, Send actionable daily brief message

#### Node: Parse AI Market Output
- **Type / role:** Code — converts the AI-generated text into discrete fields.
- **Configuration choices (interpreted):**
  - Reads AI text from: `$json.output?.[0]?.content?.[0]?.text || ""`
  - Normalizes CRLF and whitespace.
  - Extracts sections using regex supporting both:
    - `### Market Summary` style
    - `**Market Summary**` style (the prompt requests this)
  - Cleans bullet prefixes (`-` / `•`) and blank lines.
  - Outputs:
    - `marketSummary`, `sentiment`, `keyDrivers`, `risks`, `actionableInsight`, `fullText`
- **Inputs:** OpenAI node output.
- **Outputs:** Parsed structured item to Slack node.
- **Edge cases / failures:**
  - If OpenAI output structure changes (different response schema), `rawText` may be empty.
  - Regex mismatch if model uses unexpected headers (spelling/case variation beyond what regex anticipates).
  - If sections are missing, outputs become empty strings (Slack message may look blank).

#### Node: Send actionable daily brief message
- **Type / role:** Slack — posts a formatted message (DM to slackbot user selection).
- **Configuration choices:**
  - Authentication: OAuth2 (Slack account 5)
  - Target selection: `user` = `slackbot`
  - Text template includes:
    - `{{ $json.marketSummary }}`
    - `{{ $json.sentiment }}`
    - `{{ $json.actionableInsight }}`
    - `{{ $json.keyDrivers }}`
    - `{{ $json.risks }}`
- **Inputs:** Parsed fields from Parse AI Market Output.
- **Outputs:** Slack API response.
- **Edge cases / failures:**
  - Slack OAuth token revoked / missing scopes (chat:write, etc.).
  - Posting to `slackbot` may not behave as expected in all workspaces; many implementations prefer a channel ID.
  - Message length limits if AI output is too verbose.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Market Brief Trigger | Schedule Trigger | Daily workflow entry point (08:30) | — | Read portfolio holdings | ## Workflow overview… (Creates daily market intelligence summary; setup steps: add stock symbols, add Alpha Vantage key, connect OpenAI/Slack, set schedule time) |
| Read portfolio holdings | Google Sheets | Read portfolio/stock list from spreadsheet | Daily Market Brief Trigger | Process each stock | ### Data Collection (Reads stock list; controls processing for multiple stocks) |
| Process each stock | Split In Batches | Batch/loop control for per-stock processing | Read portfolio holdings | Stock Price (Alpha Vantage) *(via output index 1)* | ### Data Collection (Reads stock list; controls processing for multiple stocks) |
| Stock Price (Alpha Vantage) | HTTP Request | Pull daily stock prices from Alpha Vantage | Process each stock | Normalize Stock Data | ### Market data collection (Collects stock prices, news, and sentiment for current market view) |
| Normalize Stock Data | Code | Compute close, change %, and trend | Stock Price (Alpha Vantage) | RSS Read | ### Market data collection (Collects stock prices, news, and sentiment for current market view) |
| RSS Read | RSS Feed Read | Fetch Google News RSS headlines | Normalize Stock Data | Normalize Market News | ### Market data collection (Collects stock prices, news, and sentiment for current market view) |
| Normalize Market News | Code | Normalize and filter headlines to last 24h | RSS Read | Reddit Sentiment – RSS | ### Market data collection (Collects stock prices, news, and sentiment for current market view) |
| Reddit Sentiment – RSS | RSS Feed Read | Fetch Reddit r/stocks RSS feed | Normalize Market News | Normalize Reddit News | ### Market data collection (Collects stock prices, news, and sentiment for current market view) |
| Normalize Reddit News | Code | Normalize Reddit posts into text signals | Reddit Sentiment – RSS | Prepare AI Context | ### Market data collection (Collects stock prices, news, and sentiment for current market view) |
| Prepare AI Context | Code | Merge stock + news + reddit into one JSON payload | Normalize Reddit News | Ai Analysis | ### Data preparation and analysis (Cleans/combines data; AI removes noise; generates summary with risks/actions) |
| Ai Analysis | OpenAI (LangChain) | Generate structured market brief from context | Prepare AI Context | Parse AI Market Output | ### Data preparation and analysis (Cleans/combines data; AI removes noise; generates summary with risks/actions) |
| Parse AI Market Output | Code | Extract sections into fields for reuse | Ai Analysis | Send actionable daily brief message | ### Output structuring and delivery (Organises AI response and delivers readable daily Slack message) |
| Send actionable daily brief message | Slack | Deliver the daily brief to Slack | Parse AI Market Output | — | ### Output structuring and delivery (Organises AI response and delivers readable daily Slack message) |
| Sticky Note | Sticky Note | Documentation/overview | — | — | ## Workflow overview… (same content as note) |
| Sticky Note1 | Sticky Note | Documentation: Data Collection block | — | — | ### Data Collection… |
| Sticky Note2 | Sticky Note | Documentation: Market data collection block | — | — | ### Market data collection… |
| Sticky Note3 | Sticky Note | Documentation: Data preparation and analysis block | — | — | ### Data preparation and analysis… |
| Sticky Note4 | Sticky Note | Documentation: Output structuring and delivery block | — | — | ### Output structuring and delivery… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it (e.g.) “Turn Stock Data and News Into a Daily Market Decision Using AI”.
   - Ensure workflow timezone is set appropriately (Settings → Timezone).

2) **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Set rule: daily at **08:30**.

3) **Add Google Sheets: Read holdings**
   - Node: **Google Sheets**
   - Operation: read/get rows (default “Read” behavior).
   - Select document by URL:
     - `https://docs.google.com/spreadsheets/d/1Mz-woYDtXtzF2bA9IqpdYh28IPA76nFx46WHgDJOZoI/edit` (or your own)
   - Select sheet/tab: “Portfolio Performance” (or your own).
   - **Credential:** Google Sheets OAuth2 (connect your Google account).
   - Ensure the sheet has a column named **`Stock`** containing ticker symbols.

4) **Add Split In Batches**
   - Node: **Split In Batches**
   - Keep defaults (or set a batch size like 1–5 to reduce rate-limit risk).
   - Connect: Schedule Trigger → Google Sheets → Split In Batches.
   - Connect Split In Batches **output index 1** to the next node (as in the provided workflow).
   - (Recommended improvement) Add a loop-back later if you want guaranteed iteration through all rows.

5) **Add HTTP Request: Alpha Vantage**
   - Node: **HTTP Request**
   - URL: `https://www.alphavantage.co/query`
   - Enable “Send Query Parameters”
   - Add parameters:
     - `function` = `TIME_SERIES_DAILY`
     - `symbol` = expression `{{ $json.Stock }}`
     - `outputsize` = `compact`
     - `apikey` = your Alpha Vantage key (replace `YOUR_API_KEY`)
   - Connect: Split In Batches → HTTP Request.

6) **Add Code: Normalize Stock Data**
   - Node: **Code**
   - Paste the logic that:
     - reads `Time Series (Daily)`
     - picks latest & previous dates
     - computes changePercent and trend
     - outputs `{ symbol, date, close, previousClose, changePercent, trend }`
   - Connect: HTTP Request → Normalize Stock Data.

7) **Add RSS Feed Read: Google News**
   - Node: **RSS Feed Read**
   - URL:
     - `https://news.google.com/rss/search?q=stock+market+OR+earnings+OR+inflation+OR+interest+rates`
   - Connect: Normalize Stock Data → RSS Read.

8) **Add Code: Normalize Market News**
   - Node: **Code**
   - Implement mapping of RSS items to `{headline, summary, source, publishedAt, ageHours}`
   - Filter to items published within last 24 hours.
   - Connect: RSS Read → Normalize Market News.

9) **Add RSS Feed Read: Reddit**
   - Node: **RSS Feed Read**
   - URL: `https://www.reddit.com/r/stocks/.rss`
   - Connect: Normalize Market News → Reddit RSS.

10) **Add Code: Normalize Reddit News**
   - Node: **Code**
   - Map to `{ platform:"Reddit", source:"r/stocks", text, publishedAt }`
   - Connect: Reddit RSS → Normalize Reddit News.

11) **Add Code: Prepare AI Context**
   - Node: **Code**
   - Build a single-item payload:
     - `stock` from `$items("Normalize Stock Data")[0].json`
     - `news` from `$items("Normalize Market News").map(i => i.json)`
     - `reddit` from `$items("Normalize Reddit News").map(i => i.json)`
   - Connect: Normalize Reddit News → Prepare AI Context.

12) **Add OpenAI (LangChain) node: Ai Analysis**
   - Node: **OpenAI** (the `@n8n/n8n-nodes-langchain.openAi` node)
   - Model: `gpt-4o-mini` (or equivalent available model)
   - Credentials: connect OpenAI API key/account in n8n credentials.
   - Prompt/body: include the role/instructions and embed:
     - `{{ JSON.stringify($json.stock, null, 2) }}`
     - `{{ JSON.stringify($json.news, null, 2) }}`
     - `{{ JSON.stringify($json.reddit, null, 2) }}`
   - Require strict headers using **bold** section titles.
   - Connect: Prepare AI Context → Ai Analysis.

13) **Add Code: Parse AI Market Output**
   - Node: **Code**
   - Extract text from `output[0].content[0].text`
   - Parse sections **Market Summary / Sentiment / Key Drivers / Risks / Actionable Insight**
   - Output those fields for Slack templating.
   - Connect: Ai Analysis → Parse AI Market Output.

14) **Add Slack node: Send message**
   - Node: **Slack**
   - Authentication: OAuth2 (connect Slack app/workspace with appropriate scopes)
   - Operation: send message (as configured in your Slack node variant)
   - Recipient: user `slackbot` (or preferably a channel like `#market-brief`)
   - Message text uses expressions:
     - `{{ $json.marketSummary }}`, `{{ $json.sentiment }}`, etc.
   - Connect: Parse AI Market Output → Slack.

15) **Activate workflow**
   - Toggle workflow active after verifying credentials and test-running once.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow creates a daily market intelligence summary… delivered to Slack as a clean, easy-to-read message.” | Workflow overview sticky note (internal documentation) |
| Setup reminders: add stock symbols in Google Sheet; add Alpha Vantage API key; connect OpenAI and Slack; set schedule time | Workflow overview sticky note (internal documentation) |
| Data Collection block: reads stock list and controls processing for multiple stocks | Sticky note “Data Collection” |
| Market data collection block: gathers stock prices, market news, and investor sentiment | Sticky note “Market data collection” |
| Data preparation and analysis block: cleans/combines data; AI removes noise and generates summary | Sticky note “Data preparation and analysis” |
| Output structuring and delivery block: structures AI response and posts a single daily Slack message | Sticky note “Output structuring and delivery” |