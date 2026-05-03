Analyze crypto market sentiment and send Discord alerts with Gemini Gemini 2.5

https://n8nworkflows.xyz/workflows/analyze-crypto-market-sentiment-and-send-discord-alerts-with-gemini-gemini-2-5-12795


# Analyze crypto market sentiment and send Discord alerts with Gemini Gemini 2.5

## 1. Workflow Overview

This workflow runs on a daily schedule and produces an AI-generated crypto market sentiment report (BTC/ETH focus) from multiple free public data sources. It summarizes key metrics (prices, market cap, dominance, DXY, Fear & Greed, funding rates), asks Gemini (Gemini 2.5 Flash) to generate an “investment stance” and recommendations, then sends a richly formatted Discord embed via webhook.

### 1.1 Scheduling / Entry Point
Runs every day at a specified hour (default: 17:00 / 5PM).

### 1.2 Data Collection (Free APIs)
Fetches:
- CoinGecko global market metrics
- CoinGecko BTC/ETH/USDT prices + 24h change + market cap
- Yahoo Finance DXY (Dollar Index)
- Alternative.me Fear & Greed Index
- OKX BTC and ETH perpetual funding rates

### 1.3 Data Normalization & Prompt Construction
Merges all API responses, parses them into a single normalized object, then constructs a structured prompt for Gemini.

### 1.4 Gemini AI Analysis
Gemini analyzes the prompt and returns a structured response with sections and an explicit “INVESTMENT STANCE”.

### 1.5 Discord Alert Formatting & Delivery
Extracts stance + recommendation text, builds a Discord embed (with color/emoji based on stance), and posts it to a Discord webhook.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling / Trigger

**Overview:** Starts the workflow once per day at the configured hour and fan-outs into parallel HTTP requests.

**Nodes involved:**
- Daily Schedule Trigger (5PM)

#### Node: Daily Schedule Trigger (5PM)
- **Type / role:** `Schedule Trigger` — time-based entry point.
- **Configuration (interpreted):** Runs daily at hour `17` (server timezone as configured in n8n).
- **Connections:**
  - **Outputs to (in parallel):** Fetch Global Market Metrics, Fetch BTC ETH USDT Prices, Fetch DXY Dollar Index, Fetch Fear and Greed Index, Fetch BTC Funding Rate, Fetch ETH Funding Rate.
- **Edge cases / failures:**
  - Timezone mismatch (expected 5PM local vs server time).
  - If n8n instance is down at trigger time, execution may be missed unless using n8n’s reliability features (depends on hosting mode).

---

### Block 2 — Data Collection

**Overview:** Calls multiple public endpoints to retrieve current market and sentiment indicators.

**Nodes involved:**
- Fetch Global Market Metrics
- Fetch BTC ETH USDT Prices
- Fetch DXY Dollar Index
- Fetch Fear and Greed Index
- Fetch BTC Funding Rate
- Fetch ETH Funding Rate

#### Node: Fetch Global Market Metrics
- **Type / role:** `HTTP Request` — pulls global market stats.
- **Configuration:** GET `https://api.coingecko.com/api/v3/global`
- **Output:** CoinGecko “global” payload; later code expects `json.data.total_market_cap` and dominance percentages.
- **Connections:** Output → Combine All API Data (input index 0).
- **Edge cases / failures:**
  - CoinGecko rate-limits (HTTP 429) or intermittent downtime.
  - Response shape changes (missing `data`).
  - Network timeouts.

#### Node: Fetch BTC ETH USDT Prices
- **Type / role:** `HTTP Request` — pulls spot prices and 24h change.
- **Configuration:** GET  
  `https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum,tether&vs_currencies=usd&include_market_cap=true&include_24hr_change=true`
- **Output:** Expects `json.bitcoin.usd`, `.usd_24h_change` etc.
- **Connections:** Output → Combine All API Data (input index 1).
- **Edge cases / failures:**
  - Rate limiting / 429.
  - Missing keys if CoinGecko changes naming or request fails partially.

#### Node: Fetch DXY Dollar Index
- **Type / role:** `HTTP Request` — retrieves DXY chart data.
- **Configuration:** GET  
  `https://query1.finance.yahoo.com/v8/finance/chart/DX-Y.NYB?interval=1d&range=2d`
- **Output:** Later code expects `json.chart.result[0].indicators.quote[0].close` (array).
- **Connections:** Output → Combine All API Data (input index 2).
- **Edge cases / failures:**
  - Yahoo endpoints can be unstable or blocked in some regions.
  - `close` array may contain `null` for latest candle (market closed/incomplete), causing `toFixed` errors unless guarded (current code partially guards but can still produce issues if last value is null).
  - Potential anti-bot / throttling behavior.

#### Node: Fetch Fear and Greed Index
- **Type / role:** `HTTP Request` — retrieves sentiment index.
- **Configuration:** GET `https://api.alternative.me/fng/`
- **Output:** Code expects `json.name === "Fear and Greed Index"` and `json.data[0]`.
- **Connections:** Output → Combine All API Data (input index 3).
- **Edge cases / failures:**
  - API shape mismatch (sometimes `name` may differ or be absent).
  - `data[0].value` is a string; later nodes parse with `parseInt`.

#### Node: Fetch BTC Funding Rate
- **Type / role:** `HTTP Request` — OKX funding rate for BTC swap.
- **Configuration:** GET  
  `https://www.okx.com/api/v5/public/funding-rate?instId=BTC-USDT-SWAP`
- **Output:** Code expects `json.code === "0"` and `json.data[0].fundingRate`.
- **Connections:** Output → Combine All API Data (input index 4).
- **Edge cases / failures:**
  - OKX may rate-limit or require headers depending on region.
  - Funding rate may be absent or string not parseable.

#### Node: Fetch ETH Funding Rate
- **Type / role:** `HTTP Request` — OKX funding rate for ETH swap.
- **Configuration:** GET  
  `https://www.okx.com/api/v5/public/funding-rate?instId=ETH-USDT-SWAP`
- **Connections:** Output → Combine All API Data (input index 5).
- **Edge cases / failures:** Same as BTC funding rate.

---

### Block 3 — Merge + Normalize Metrics

**Overview:** Merges the six API responses into one execution context and parses them into a normalized, analysis-ready object with derived fields (percent changes, text “N/A” fallbacks, etc.).

**Nodes involved:**
- Combine All API Data
- Format and Parse All Data

#### Node: Combine All API Data
- **Type / role:** `Merge` — multi-input aggregation.
- **Configuration:** `numberInputs: 6` (waits for six incoming branches).
- **Connections:**
  - Inputs: Global metrics, prices, DXY, Fear&Greed, BTC funding, ETH funding.
  - Output → Format and Parse All Data.
- **Edge cases / failures:**
  - If any upstream request errors and stops execution, this merge will never receive all inputs and the workflow fails upstream.
  - If any upstream node is set to “Continue on Fail” (not shown here), merge may receive an error item or missing data; downstream code throws if core data missing.

#### Node: Format and Parse All Data
- **Type / role:** `Code` — identifies each API payload, extracts values, computes derived stats, and outputs a single normalized object.
- **Configuration choices (interpreted):**
  - Scans `$input.all()` and detects which response is which by checking distinctive fields:
    - CoinGecko prices: `json.bitcoin && json.ethereum && json.tether`
    - CoinGecko global: `json.data && json.data.total_market_cap`
    - Yahoo: `json.chart && json.chart.result`
    - Alternative.me: `json.name === "Fear and Greed Index"`
    - OKX: `json.code === "0" && json.data[0].instId`
  - Throws hard error if `cryptoPrices` or `globalData` is missing.
  - DXY parsing: takes last close, attempts to compute 24h percent change using previous close or `meta.chartPreviousClose`.
  - Funding rates: converts `fundingRate` to percent by `* 100` and formats to 4 decimals.
  - Outputs fields like `btc_price`, `market_cap_change_24h`, etc.
- **Key expressions / variables:**
  - Uses `$input.all()` and manual type detection.
  - Returns **a plain object**, not an array of `{json: ...}` items (this is important).
- **Connections:** Output → Build Gemini Analysis Prompt.
- **Edge cases / potential failures:**
  - **n8n Code node return format:** In most modern n8n versions, Code node should return an array of items (`[{ json: ... }]`). Returning a plain object can work only in certain contexts/versions or may break. This node returns a plain object.
  - **Bug in market cap calculation:**  
    `market_cap: (marketCap / 1+1234567890).toFixed(2)`  
    Due to operator precedence, this becomes `(marketCap / 1) + 1234567890`, then `.toFixed(2)`—inflating market cap by 1,234,567,890 and not converting to billions. The prompt later appends “B” (billions), so output is inconsistent/wrong.
  - DXY last close could be `null` → calling `toFixed(2)` throws. The try/catch sets `dxyValue = "N/A"` but does not always sanitize `dxyChange24h` if parsing partially succeeded.
  - Fear & Greed value “N/A” later gets `parseInt("N/A")` → `NaN` in Discord formatting logic.

---

### Block 4 — Prompt Construction + Gemini Analysis

**Overview:** Converts normalized metrics into a long, structured instruction prompt and sends it to Gemini 2.5 Flash.

**Nodes involved:**
- Build Gemini Analysis Prompt
- Gemini AI Analysis

#### Node: Build Gemini Analysis Prompt
- **Type / role:** `Code` — builds a deterministic prompt template plus embeds the day’s metrics.
- **Configuration choices (interpreted):**
  - Reads `$input.first().json` as `data`.
  - Generates a prompt with strict output format sections:
    - MARKET ANALYSIS
    - DXY & MACRO CORRELATION
    - SENTIMENT & POSITIONING
    - DOMINANCE INSIGHTS
    - INVESTMENT STANCE
    - RECOMMENDATION
    - KEY RISK FACTORS
  - Copies all original metric keys into output and adds `prompt`.
- **Connections:** Output → Gemini AI Analysis.
- **Edge cases / failures:**
  - If previous Code node returned a nonstandard structure, `$input.first().json` may not exist (depending on n8n version/runtime behavior).
  - If any metric is “N/A”, the prompt still includes it; Gemini may still comply, but downstream parsing expects strict patterns.

#### Node: Gemini AI Analysis
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — calls Google Gemini via API.
- **Configuration choices:**
  - **Model:** `models/gemini-2.5-flash`
  - **Messages payload:** Manually crafts a JSON-like structure in the message content; inserts `{{$json.prompt}}`.
  - **Credentials:** Uses a Google Gemini (PaLM) API credential.
- **Connections:** Output → Extract AI Analysis and Stance.
- **Version-specific requirements:**
  - Requires the LangChain Gemini node package available in the n8n instance and a compatible node version.
- **Edge cases / failures:**
  - Invalid/expired API key, billing/project restrictions, quota limits.
  - Model ID not available in the region/project.
  - Output format variability: Gemini may deviate from the exact requested format, which can break regex extraction downstream.

---

### Block 5 — Parse AI Output + Create Discord Embed + Send

**Overview:** Extracts the AI text, derives stance and a short action sentence, builds a Discord embed with computed indicators and AI snippets, then sends it via webhook.

**Nodes involved:**
- Extract AI Analysis and Stance
- Build Discord Alert Message
- Send to Discord Webhook

#### Node: Extract AI Analysis and Stance
- **Type / role:** `Code` — parses Gemini response and combines it with market metrics.
- **Configuration choices (interpreted):**
  - Pulls market data from the node **by name**: `$node["Format and Parse All Data"].json`
  - Reads Gemini output from `$input.first().json`
  - Attempts multiple response shapes:
    - `geminiInput[0].content.parts[0].text`
    - `geminiInput.content.parts[0].text`
  - Regex extraction:
    - Stance: `/INVESTMENT STANCE: \[?(.*?)[\]\n]/i`
    - Recommendation block: `/RECOMMENDATION:\n(.*?)$/s`
  - **Bug:** It assigns `recommendation` from `stanceMatch` (stance), not from recommendation text; variable naming is confusing:
    - `recommendation = stanceMatch ? stanceMatch[1] ... : 'NEUTRAL'`
    - `actionAdvice` is taken from the RECOMMENDATION section.
- **Connections:** Output → Build Discord Alert Message.
- **Edge cases / failures:**
  - If Gemini output doesn’t include the exact “INVESTMENT STANCE:” line, stance defaults to NEUTRAL.
  - If Gemini response structure differs, `aiAnalysis` becomes “ERROR: Could not parse AI response.”
  - Dependency on node name `Format and Parse All Data`: renaming that node will break this node.

#### Node: Build Discord Alert Message
- **Type / role:** `Code` — builds a Discord webhook payload with embeds.
- **Configuration choices (interpreted):**
  - Chooses emoji + embed color based on `data.recommendation` containing BULLISH/BEARISH, else NEUTRAL.
  - Builds extra computed labels:
    - Fear & Greed emoji + signal
    - DXY/BTC “correlation” heuristic (simple rule-based)
    - Funding “status” based on BTC funding thresholds
    - Dominance trend descriptions
  - Extracts first sentence from AI “MARKET ANALYSIS” and “DOMINANCE INSIGHTS” sections (regex).
  - Returns a plain object representing Discord webhook JSON payload: `{ embeds: [...] }`.
- **Connections:** Output → Send to Discord Webhook.
- **Edge cases / failures:**
  - `fear_greed_value` could be `"N/A"` → `parseInt` => `NaN`, comparisons fail and emoji logic falls through to `🤑` (because `else fngEmoji = "🤑"` when comparisons with NaN are false).
  - Uses `parseFloat` on potentially `"N/A"` fields (funding, DXY change) → `NaN`, then heuristic strings may default unexpectedly.
  - If `ai_analysis` is the error string, regex sections won’t match; fallback messages are used.

#### Node: Send to Discord Webhook
- **Type / role:** `HTTP Request` — posts the embed to Discord.
- **Configuration:**
  - Method: `POST`
  - URL: `YOUR-DISCORD-WEBHOOK-URL` (must be replaced)
  - Body: JSON body set to `={{ $json }}` (sends the object produced by previous node)
- **Connections:** Final node (no outputs).
- **Edge cases / failures:**
  - Invalid webhook URL or revoked webhook → 404/401.
  - Discord rate limits → HTTP 429.
  - Payload too large (Discord embed limits) if AI snippets become long (this workflow tries to keep AI snippets short by using first sentence, which helps).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | Sticky Note | Documentation / canvas annotation | — | — | ## 🚀 Crypto market sentiment analyzer; **Who is this for?** Crypto traders and investors who want daily AI-powered market analysis delivered to Discord.; **What it does:** Fetches real-time crypto data from multiple APIs, analyzes market sentiment using Gemini AI, and sends beautifully formatted alerts to Discord.; **✅ Uses Free APIs Only!** CoinGecko, Yahoo Finance, Alternative.me, and OKX - no paid subscriptions required.; **How it works:** 1. Triggers daily at scheduled time (default: 5PM) 2. Fetches BTC/ETH prices, market cap, DXY, Fear & Greed, funding rates 3. Combines all data and builds analysis prompt 4. Gemini AI analyzes and provides investment stance 5. Formats and sends rich embed to Discord webhook; **Setup steps:** - Get free Google Gemini API key from Google AI Studio - Create Discord webhook in your server channel - Update webhook URL in Send to Discord Webhook node - Connect Gemini credentials to Gemini AI Analysis node; **Configuration:** - Adjust trigger time in Daily Schedule Trigger node (default: 5PM) |
| Sticky Note - Data Collection | Sticky Note | Documentation / canvas annotation | — | — | ### 📥 Data Collection; Fetch prices, metrics, DXY, Fear & Greed, and funding rates from free APIs |
| Sticky Note - AI Output | Sticky Note | Documentation / canvas annotation | — | — | ### 🤖 AI Analysis & Output; Process data with Gemini AI and send formatted alerts to Discord |
| Daily Schedule Trigger (5PM) | Schedule Trigger | Daily workflow trigger | — | Fetch Global Market Metrics; Fetch BTC ETH USDT Prices; Fetch DXY Dollar Index; Fetch Fear and Greed Index; Fetch BTC Funding Rate; Fetch ETH Funding Rate |  |
| Fetch Global Market Metrics | HTTP Request | Pull CoinGecko global market metrics | Daily Schedule Trigger (5PM) | Combine All API Data | ### 📥 Data Collection; Fetch prices, metrics, DXY, Fear & Greed, and funding rates from free APIs |
| Fetch BTC ETH USDT Prices | HTTP Request | Pull CoinGecko prices + 24h change for BTC/ETH/USDT | Daily Schedule Trigger (5PM) | Combine All API Data | ### 📥 Data Collection; Fetch prices, metrics, DXY, Fear & Greed, and funding rates from free APIs |
| Fetch DXY Dollar Index | HTTP Request | Pull DXY chart data from Yahoo Finance | Daily Schedule Trigger (5PM) | Combine All API Data | ### 📥 Data Collection; Fetch prices, metrics, DXY, Fear & Greed, and funding rates from free APIs |
| Fetch Fear and Greed Index | HTTP Request | Pull Fear & Greed Index | Daily Schedule Trigger (5PM) | Combine All API Data | ### 📥 Data Collection; Fetch prices, metrics, DXY, Fear & Greed, and funding rates from free APIs |
| Fetch BTC Funding Rate | HTTP Request | Pull BTC funding rate from OKX | Daily Schedule Trigger (5PM) | Combine All API Data | ### 📥 Data Collection; Fetch prices, metrics, DXY, Fear & Greed, and funding rates from free APIs |
| Fetch ETH Funding Rate | HTTP Request | Pull ETH funding rate from OKX | Daily Schedule Trigger (5PM) | Combine All API Data | ### 📥 Data Collection; Fetch prices, metrics, DXY, Fear & Greed, and funding rates from free APIs |
| Combine All API Data | Merge | Waits for and merges 6 API responses | Fetch Global Market Metrics; Fetch BTC ETH USDT Prices; Fetch DXY Dollar Index; Fetch Fear and Greed Index; Fetch BTC Funding Rate; Fetch ETH Funding Rate | Format and Parse All Data |  |
| Format and Parse All Data | Code | Normalize metrics, compute derived values | Combine All API Data | Build Gemini Analysis Prompt |  |
| Build Gemini Analysis Prompt | Code | Construct structured Gemini prompt | Format and Parse All Data | Gemini AI Analysis |  |
| Gemini AI Analysis | Google Gemini (LangChain) | Run Gemini 2.5 Flash analysis | Build Gemini Analysis Prompt | Extract AI Analysis and Stance | ### 🤖 AI Analysis & Output; Process data with Gemini AI and send formatted alerts to Discord |
| Extract AI Analysis and Stance | Code | Parse Gemini output, extract stance + advice | Gemini AI Analysis | Build Discord Alert Message | ### 🤖 AI Analysis & Output; Process data with Gemini AI and send formatted alerts to Discord |
| Build Discord Alert Message | Code | Build Discord embed payload | Extract AI Analysis and Stance | Send to Discord Webhook | ### 🤖 AI Analysis & Output; Process data with Gemini AI and send formatted alerts to Discord |
| Send to Discord Webhook | HTTP Request | Deliver embed to Discord via webhook | Build Discord Alert Message | — | ### 🤖 AI Analysis & Output; Process data with Gemini AI and send formatted alerts to Discord |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Schedule Trigger**
   - Node: *Schedule Trigger*
   - Name: `Daily Schedule Trigger (5PM)`
   - Rule: daily at `17` (adjust as needed).
3. **Add HTTP Request nodes (6 total)**
   - Node 1: `Fetch Global Market Metrics`
     - Method: GET
     - URL: `https://api.coingecko.com/api/v3/global`
   - Node 2: `Fetch BTC ETH USDT Prices`
     - Method: GET
     - URL: `https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum,tether&vs_currencies=usd&include_market_cap=true&include_24hr_change=true`
   - Node 3: `Fetch DXY Dollar Index`
     - Method: GET
     - URL: `https://query1.finance.yahoo.com/v8/finance/chart/DX-Y.NYB?interval=1d&range=2d`
   - Node 4: `Fetch Fear and Greed Index`
     - Method: GET
     - URL: `https://api.alternative.me/fng/`
   - Node 5: `Fetch BTC Funding Rate`
     - Method: GET
     - URL: `https://www.okx.com/api/v5/public/funding-rate?instId=BTC-USDT-SWAP`
   - Node 6: `Fetch ETH Funding Rate`
     - Method: GET
     - URL: `https://www.okx.com/api/v5/public/funding-rate?instId=ETH-USDT-SWAP`
4. **Connect the trigger to all 6 HTTP Request nodes** (parallel fan-out).
5. **Add a Merge node**
   - Node: *Merge*
   - Name: `Combine All API Data`
   - Mode: multi-input merge (set **Number of Inputs = 6**)
   - Connect each HTTP node into the Merge node inputs (indices 0–5).
6. **Add Code node: data parsing**
   - Node: *Code*
   - Name: `Format and Parse All Data`
   - Paste the parsing JS logic (as in workflow) that:
     - detects payloads
     - extracts btc/eth/usdt price + change
     - extracts global market cap + dominance + change
     - extracts DXY last close + change
     - extracts Fear & Greed value/classification
     - extracts OKX funding rates
     - outputs a single normalized object
   - Connect: `Combine All API Data` → `Format and Parse All Data`.
   - Practical note: to maximize compatibility, return `[{ json: output }]` rather than a raw object.
7. **Add Code node: prompt builder**
   - Node: *Code*
   - Name: `Build Gemini Analysis Prompt`
   - Build the long prompt string from the normalized object and output it as `prompt` plus the original fields.
   - Connect: `Format and Parse All Data` → `Build Gemini Analysis Prompt`.
8. **Add Gemini node**
   - Node: *Google Gemini (LangChain)* (package: `@n8n/n8n-nodes-langchain.googleGemini`)
   - Name: `Gemini AI Analysis`
   - Model: `models/gemini-2.5-flash`
   - Message content: insert the prompt from `{{$json.prompt}}` in the node’s message configuration.
   - **Credentials:**
     - Create/attach Google Gemini (PaLM) API credentials using an API key from Google AI Studio.
   - Connect: `Build Gemini Analysis Prompt` → `Gemini AI Analysis`.
9. **Add Code node: extract stance + advice**
   - Node: *Code*
   - Name: `Extract AI Analysis and Stance`
   - Parse Gemini output into a raw text string, regex-extract:
     - stance line after `INVESTMENT STANCE:`
     - the text after `RECOMMENDATION:`
   - Also copy normalized market fields into output.
   - Connect: `Gemini AI Analysis` → `Extract AI Analysis and Stance`.
10. **Add Code node: build Discord payload**
    - Node: *Code*
    - Name: `Build Discord Alert Message`
    - Create a Discord webhook JSON payload with `embeds`:
      - title includes stance + date
      - fields include prices, dominance, indicators
      - include short AI snippets (first sentence)
    - Connect: `Extract AI Analysis and Stance` → `Build Discord Alert Message`.
11. **Add HTTP Request node: Discord webhook**
    - Node: *HTTP Request*
    - Name: `Send to Discord Webhook`
    - Method: POST
    - URL: your Discord webhook URL
    - Body: JSON, set body to the incoming object (e.g. `={{ $json }}`)
    - Connect: `Build Discord Alert Message` → `Send to Discord Webhook`.
12. **Activate the workflow** after confirming credentials and webhook URL.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Get free Google Gemini API key from Google AI Studio | Mentioned in Sticky Note - Overview |
| Create Discord webhook in your server channel | Mentioned in Sticky Note - Overview |
| Update webhook URL in “Send to Discord Webhook” node | Mentioned in Sticky Note - Overview |
| Adjust trigger time in “Daily Schedule Trigger (5PM)” node | Mentioned in Sticky Note - Overview |
| Data sources are free/public: CoinGecko, Yahoo Finance, Alternative.me, OKX | Mentioned in Sticky Note - Overview |

Disclaimer (as provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.