Analyze Binance Futures markets with TA indicators, OpenAI news checks, and Telegram alerts

https://n8nworkflows.xyz/workflows/analyze-binance-futures-markets-with-ta-indicators--openai-news-checks--and-telegram-alerts-13011


# Analyze Binance Futures markets with TA indicators, OpenAI news checks, and Telegram alerts

## 1. Workflow Overview

**Purpose:**  
This n8n workflow scans Binance USDT-margined Futures markets every 15 minutes, detects technical-analysis-based trade opportunities (EMA200 trend filter, Bollinger Bands, RSI, volume anomaly), validates the trade against recent news using an LLM (OpenRouter/OpenAI), then **alerts via Telegram** and optionally performs **paper trades on Binance Futures Testnet** via signed REST requests.

**Target use cases:**
- Automated market scanning and TA signal generation for Binance Futures (USDT pairs)
- Risk gating using news sentiment/impact checks via an LLM
- Telegram-based trade idea distribution
- Educational “paper trading” execution loop against Binance Testnet

### 1.1 Schedule & Configuration
Runs every 15 minutes; loads user configuration (API keys, leverage, amount, Telegram channel, blacklist).

### 1.2 Market Data Collection (Candidate Selection)
Pulls Binance 24h tickers, filters to liquid USDT pairs, removes blacklisted symbols, and selects a subset of candidates to analyze.

### 1.3 Technical Analysis Core (Per-Symbol Loop)
Iterates through candidates in batches, fetches klines, computes indicators and generates a signal if conditions match, while deduplicating alerts per symbol (1-hour cooldown).

### 1.4 Risk Filters (RSI Safety + AI News Check)
Applies a fast local RSI “safety” filter, then fetches recent news and asks an LLM to confirm or skip the trade due to critical contradictory headlines.

### 1.5 Output Actions (Telegram + Paper Trading Execution Loop)
Formats a Telegram alert message. If enabled by the workflow path, it prepares signed Testnet requests (margin type, leverage, entry, SL, TP, trailing) and executes them in sequence.

---

## 2. Block-by-Block Analysis

### Block 1 — Schedule & Configuration
**Overview:** Triggers the workflow every 15 minutes and defines runtime parameters such as keys, leverage, trade amount, and blacklist.  
**Nodes involved:** `⏱️ Every 15 mins`, `📝 MAIN CONFIG`

#### Node: ⏱️ Every 15 mins
- **Type / role:** Schedule Trigger; workflow entry point.
- **Config:** Cron expression `*/15 * * * *` (every 15 minutes).
- **Outputs:** Sends one item to `📝 MAIN CONFIG`.
- **Edge cases:** None typical; relies on n8n scheduler being active.

#### Node: 📝 MAIN CONFIG
- **Type / role:** Set node; central configuration object.
- **Config choices:** Defines:
  - Numbers: `TRADE_AMOUNT_USDT` (100), `LEVERAGE` (1)
  - Strings: `BINANCE_API_KEY`, `BINANCE_SECRET`, `TELEGRAM_CHANNEL_ID`, `BLACKLIST`
- **Outputs:** Feeds `Get All Tickers`.
- **Key variables used later:**
  - `$('📝 MAIN CONFIG').first().json.TELEGRAM_CHANNEL_ID`
  - `$('📝 MAIN CONFIG').first().json.BLACKLIST`
- **Edge cases / failures:** Misconfigured keys or channel ID leads to later auth or Telegram errors (not here).

---

### Block 2 — Market Data Collection (Candidate Selection)
**Overview:** Pulls all futures tickers, filters/sorts them, removes blacklisted pairs, and produces a list of symbols to analyze.  
**Nodes involved:** `Get All Tickers`, `🔍 Filter Candidates`, `SplitInBatches`

#### Node: Get All Tickers
- **Type / role:** HTTP Request; fetches 24h ticker stats.
- **Config:** GET `https://fapi.binance.com/fapi/v1/ticker/24hr`
- **Output:** Array of ticker objects (Binance format) to `🔍 Filter Candidates`.
- **Edge cases:** Binance rate limiting (HTTP 429), transient network errors.

#### Node: 🔍 Filter Candidates
- **Type / role:** Code node; filters and ranks symbols.
- **Config choices (interpreted):**
  - Loads blacklist from `📝 MAIN CONFIG` (`BLACKLIST` is a comma-separated string).
  - Keeps only `symbol.endsWith('USDT')`
  - Sorts by `quoteVolume` descending (liquidity proxy)
  - Removes symbols found in `BLACKLIST`
  - Returns `pairs.slice(10, 150)` to avoid top stablecoin-like pairs at the very top
- **Outputs:** Emits items shaped as `{ json: { symbol } }` to `SplitInBatches`.
- **Edge cases:**
  - If `quoteVolume` missing or not numeric, sort can behave unexpectedly.
  - Blacklist parsing: extra spaces handled via `trim()`, but empty blacklist yields `[""]` which is mostly harmless.

#### Node: SplitInBatches
- **Type / role:** Split In Batches; iterates through candidate symbols.
- **Config:** `batchSize: 10`
- **Connections:**
  - **Main output 1:** (unused in this workflow’s wiring)
  - **Main output 2:** goes to `⏸️ Wait 1s` (this is the “continue” output in n8n batching patterns)
- **Important wiring note:** The workflow uses the *second* output to drive the per-batch loop.
- **Edge cases:**
  - If no candidates are produced, the loop never runs.
  - Miswiring output ports can stop analysis.

---

### Block 3 — Technical Analysis Core (Per-Symbol Loop)
**Overview:** For each symbol, waits briefly (rate limit control), fetches klines, computes TA indicators, decides whether there is a signal, and deduplicates signals per symbol for 1 hour.  
**Nodes involved:** `⏸️ Wait 1s`, `Get Klines`, `🧠 Analyze Logic`, `🚦 Signal Check`, `Loop Connector`

#### Node: ⏸️ Wait 1s
- **Type / role:** Wait; throttling to reduce API burst.
- **Config:** `amount: 0.2 seconds` (despite the name “Wait 1s”).
- **Output:** to `Get Klines`.
- **Edge cases:** None; very short wait may still hit Binance limits at scale.

#### Node: Get Klines
- **Type / role:** HTTP Request; fetches historical candles.
- **Config:**
  - GET `https://fapi.binance.com/fapi/v1/klines`
  - Query:
    - `symbol = {{ $('SplitInBatches').first().json.symbol }}`
    - `interval = 15m`
    - `limit = 300`
- **Output:** Kline array to `🧠 Analyze Logic`.
- **Edge cases:**
  - Binance returns array-of-arrays; downstream code assumes this schema.
  - Rate limiting or symbol invalid errors (if a symbol delists).

#### Node: 🧠 Analyze Logic
- **Type / role:** Code node; TA computation + signal generation + dedup.
- **Key configuration/logic:**
  - Requires at least 200 candles (EMA200); returns `{hasSignal:false}` if insufficient.
  - Calculates:
    - **EMA200** trend filter (custom EMA function)
    - **Bollinger Bands (20, 2σ)** and `bb_width`
    - **RSI(14)** and previous RSI
    - **Volume factor** vs last-20 average (`MIN_VOL_X = 1.2`)
    - **ATR(14)** for SL/TP distance
  - Signal rules (as coded):
    - LONG if uptrend, close < lower band, RSI rising and < 45, volume spike
    - SHORT if downtrend, close > upper band, RSI falling and > 55, volume spike
  - Risk/Reward:
    - `tpMultiplier = 2.5`, or `3.5` if high volatility (`bb_width > 3`) or volume factor > 2.0
  - Dedup:
    - Uses `$getWorkflowStaticData('global')`
    - `DEDUP_TIMEOUT_MS = 1 hour`
    - Suppresses signals for a symbol if one was emitted in the last hour
- **Outputs:** Emits either `{hasSignal:false, symbol}` or a populated signal object to `🚦 Signal Check`.
- **Edge cases / failures:**
  - If klines are fewer than expected, indexing can break (guarded by the `< 200` check but still assumes enough for RSI and BB windows).
  - Static data persists across executions; restarting n8n may reset it depending on setup.
  - **Signal string casing mismatch risk:** This node sets `signal` to `"Long"` / `"Short"` (capitalized), while later nodes often compare to `'LONG'`/`'SHORT'` or `'LONG'` for side logic. This can cause incorrect filtering/execution unless normalized.

#### Node: 🚦 Signal Check
- **Type / role:** IF node; branches based on `hasSignal`.
- **Config:** Boolean condition `{{ $json.hasSignal }} == true`
- **Connections:**
  - **True branch:** `🛡 RSI Safety Check.`
  - **False branch:** `Loop Connector` (continue looping)
- **Edge cases:** If `hasSignal` missing, it evaluates false and will silently loop.

#### Node: Loop Connector
- **Type / role:** NoOp; used only to connect back to batching loop.
- **Output:** back to `SplitInBatches` to fetch the next batch/symbol.
- **Edge cases:** None.

---

### Block 4 — Risk Filters (RSI Safety + AI News Check)
**Overview:** Filters out trades when RSI is too extreme for the intended direction, fetches recent news, and uses an LLM to decide whether to confirm or skip due to critical contradictory events.  
**Nodes involved:** `🛡 RSI Safety Check.`, `📰 Get News`, `📝 Format Context`, `OpenRouter/OpenAI Model`, `🤖 AI Analysis`, `🛠️ Merge & Clean Data`

#### Node: 🛡 RSI Safety Check.
- **Type / role:** Code node; fast RSI-based kill switch.
- **Logic:**
  - Uses `RSI_MAX=70`, `RSI_MIN=30`
  - If `signal === 'LONG'` and RSI > 70 → stop
  - If `signal === 'SHORT'` and RSI < 30 → stop
  - If RSI missing → passes through
- **Connections:** Output to `📰 Get News`.
- **Edge cases / failures:**
  - **Casing mismatch:** The strategy sets `signal` to `"Long"/"Short"`, but this node checks `'LONG'/'SHORT'`. As written, it will likely never block, unless upstream signal is normalized elsewhere.
  - If RSI is a string, `parseFloat` handles it.

#### Node: 📰 Get News
- **Type / role:** HTTP Request; fetches CryptoCompare news.
- **Config:**
  - GET `https://min-api.cryptocompare.com/data/v2/news/`
  - Query:
    - `categories = {{ $('🧠 Analyze Logic').first().json.symbol.replace('USDT', '') }}`
    - `lang = EN`
- **Output:** News response to `📝 Format Context`.
- **Edge cases:**
  - CryptoCompare may rate limit; response schema changes could break parsing (`Data` field expected).
  - Category mapping may be imperfect for some tickers.

#### Node: 📝 Format Context
- **Type / role:** Code node; builds prompt-ready context.
- **Config choices:**
  - Reads `Data` array from news response.
  - Creates a short text block of up to 3 headlines.
  - Merges this into the TA payload from `🧠 Analyze Logic` as `news_context`.
- **Output:** to `🤖 AI Analysis`.
- **Edge cases:** If `Data` missing, it uses “No recent news found.”

#### Node: OpenRouter/OpenAI Model
- **Type / role:** LangChain chat model provider node (OpenRouter).
- **Config:**
  - Model: `openai/gpt-3.5-turbo`
  - Temperature: `0.2`
- **Connections:** Supplies the language model input to `🤖 AI Analysis` via the `ai_languageModel` connection.
- **Version requirements:** Requires `@n8n/n8n-nodes-langchain` installed and configured; OpenRouter credentials set in n8n.
- **Edge cases:** Credential/auth errors, model name availability changes on OpenRouter.

#### Node: 🤖 AI Analysis
- **Type / role:** LangChain “chain” node; prompts the model to do risk validation.
- **Prompt behavior (interpreted):**
  - Role: “Risk Manager” to detect high-risk news events contradicting the signal.
  - Strict JSON response required:
    - Confirm: `{ "reason": "CONFIRM: ..."}`
    - Skip: `{ "reason": "SKIP: ..."}`
- **Output:** Model text to `🛠️ Merge & Clean Data`.
- **Edge cases:**
  - LLM may still return markdown/code fences; downstream node attempts to clean it.
  - No explicit enforcement that `signal` is LONG/SHORT; model sees whatever string is provided.

#### Node: 🛠️ Merge & Clean Data
- **Type / role:** Code node; combines TA + AI output into one clean object.
- **Config choices:**
  - Retrieves TA payload from `$('🧠 Analyze Logic').item.json`
  - Reads LLM output from current input item (`json.text`)
  - Strips ```json fences and attempts to parse JSON; extracts `reason`
  - Fallback behavior:
    - If parse fails, sets `ai_reason = "CHECK: <snippet>"`
    - Default confirm message if empty
  - Adds `processed_at` ISO timestamp
- **Connections:** Outputs to both:
  - `Get Exchange Rules` (paper-trade preparation path)
  - `📢 Notify Telegram` (alert path)
- **Edge cases / failures:**
  - If `🧠 Analyze Logic` item context isn’t available, it falls back to `{symbol:"ERROR"}`.
  - AI output length/format variations can reduce parse accuracy.

---

### Block 5 — Telegram Notification
**Overview:** Sends a formatted trade alert including TA metrics and AI risk reason to a Telegram channel, then continues the symbol loop.  
**Nodes involved:** `📢 Notify Telegram`, `Loop Connector`

#### Node: 📢 Notify Telegram
- **Type / role:** Telegram node; sends message.
- **Config:**
  - `chatId = {{ $('📝 MAIN CONFIG').first().json.TELEGRAM_CHANNEL_ID }}`
  - HTML parse mode enabled
  - Message includes symbol, signal, entry/TP/SL, trend, RR ratio, RSI, volatility, and `ai_reason`.
- **Output:** to `Loop Connector` (continues scanning).
- **Edge cases / failures:**
  - Telegram bot not admin in channel, wrong channel ID (e.g., missing `@`), or invalid credentials.
  - HTML formatting issues if values contain unexpected characters.

---

### Block 6 — Paper Trading Execution Loop (Binance Testnet)
**Overview:** Builds a sequence of signed Binance Futures Testnet API requests (margin type, leverage, market entry, SL, TP, trailing stop) and executes them one by one.  
**Nodes involved:** `Get Exchange Rules`, `📝 Prep String`, `🔄 Loop (SplitBatch)`, `🔐 Sign Request`, `Execute (Paper Trading)`

#### Node: Get Exchange Rules
- **Type / role:** HTTP Request; fetches exchange info (filters/precision) per symbol.
- **Config:** GET `https://fapi.binance.com/fapi/v1/exchangeInfo?symbol={{ $json.symbol }}`
- **Input:** From `🛠️ Merge & Clean Data` (must contain `symbol`).
- **Output:** to `📝 Prep String`.
- **Edge cases:** Symbol not found; response schema changes.

#### Node: 📝 Prep String
- **Type / role:** Code node; prepares order parameters and creates multiple “request items”.
- **Key configuration/logic:**
  - Loads config from `📝 MAIN CONFIG`:
    - `BINANCE_API_KEY`, `TRADE_AMOUNT_USDT`, `LEVERAGE`
  - Reads market data from `🛠️ Merge & Clean Data`
  - Uses exchange rules to compute:
    - **Price precision** from `PRICE_FILTER.tickSize`
    - **Quantity precision** from `LOT_SIZE.stepSize`
  - Calculates:
    - `totalQty = (USDT_AMOUNT * LEVERAGE) / currentPrice` rounded down to step size
    - SL/TP prices from `calc_sl`/`calc_tp` (fallback to ±2%/±4% if missing)
  - Constructs 6 request items, each containing:
    - `endpoint`, `queryString`, `apiKey`, `type`
    - Endpoints:
      1. `/fapi/v1/marginType` (ISOLATED)
      2. `/fapi/v1/leverage`
      3. `/fapi/v1/order` (MARKET entry)
      4. `/fapi/v1/order` (STOP_MARKET closePosition=true)
      5. `/fapi/v1/order` (TAKE_PROFIT_MARKET reduceOnly; half qty)
      6. `/fapi/v1/order` (TRAILING_STOP_MARKET reduceOnly; half qty)
- **Output:** list of items to `🔄 Loop (SplitBatch)`.
- **Edge cases / failures:**
  - **Signal casing mismatch:** This node checks `marketData.signal === 'LONG'` to decide BUY/SELL. Upstream produces `"Long"` / `"Short"`, so it will default to the SHORT branch behavior unless normalized—this is a critical correctness issue.
  - Rounding logic floors quantities; may result in `0` for small amounts/high prices → it returns `[]` and stops execution.
  - Binance Testnet may reject certain combinations (e.g., marginType already set, min notional, invalid stopPrice).

#### Node: 🔄 Loop (SplitBatch)
- **Type / role:** Split In Batches; iterates through prepared requests for one symbol.
- **Config:** default options; batchSize not explicitly set (n8n defaults apply).
- **Connections:**
  - Output 2 → `🔐 Sign Request` (continue)
  - Output 1 not used
- **Edge cases:** Similar port/wiring sensitivity as the earlier SplitInBatches node.

#### Node: 🔐 Sign Request
- **Type / role:** Crypto node; HMAC-SHA256 signer for Binance.
- **Config:**
  - Action: HMAC
  - Algorithm: SHA256
  - `value = {{ $json.queryString }}`
  - `secret = "YOUR_CREDENTIAL_HERE"` (placeholder)
  - Output property: `signature`
- **Output:** to `Execute (Paper Trading)`.
- **Version requirements:** Uses n8n Crypto node v1.
- **Edge cases / failures:**
  - Secret must be the Binance API secret; currently hardcoded placeholder.
  - Query string must be exactly the signed payload (ordering matters in Binance signing; current code constructs deterministic order via object key iteration, which is stable in modern JS but not formally guaranteed across all cases).

#### Node: Execute (Paper Trading)
- **Type / role:** HTTP Request; executes signed POST requests against Binance Futures Testnet.
- **Config:**
  - POST `https://testnet.binancefuture.com{{ $json.endpoint }}?{{ $json.queryString }}&signature={{ $json.signature }}`
  - Sends header `X-MBX-APIKEY: {{ $json.apiKey }}`
  - **onError:** `continueRegularOutput` (won’t stop the loop on errors)
- **Output:** loops back to `🔄 Loop (SplitBatch)` to execute next request item.
- **Edge cases / failures:**
  - Binance errors: timestamp outside recvWindow (no recvWindow set), invalid signature, insufficient margin, stop order constraints, reduceOnly constraints.
  - Because errors are continued, you may place an entry without SL/TP if later calls fail.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| ⏱️ Every 15 mins | Schedule Trigger | Periodic execution trigger | — | 📝 MAIN CONFIG | # 🤖 Crypto market analyzer & Paper trader (Scan/Analyze/Validate/Paper Trade setup steps) |
| 📝 MAIN CONFIG | Set | Central runtime configuration | ⏱️ Every 15 mins | Get All Tickers | ⚠️ **CONFIGURATION** Insert Binance TESTNET keys here. |
| Get All Tickers | HTTP Request | Fetch Binance futures 24h tickers | 📝 MAIN CONFIG | 🔍 Filter Candidates | ## 1. Market Data Collection Fetches top volume pairs, filters stablecoins, and prepares candidates. |
| 🔍 Filter Candidates | Code | Filter/sort tickers; apply blacklist; select candidates | Get All Tickers | SplitInBatches | ## 1. Market Data Collection Fetches top volume pairs, filters stablecoins, and prepares candidates. |
| SplitInBatches | Split In Batches | Iterate candidate symbols (batch size 10) | 🔍 Filter Candidates; Loop Connector | ⏸️ Wait 1s | ## 1. Market Data Collection Fetches top volume pairs, filters stablecoins, and prepares candidates. |
| ⏸️ Wait 1s | Wait | Throttle loop requests | SplitInBatches | Get Klines | ## 2. Technical Analysis Core Calculates EMA, Bollinger Bands, RSI, and Volume anomalies via JS. |
| Get Klines | HTTP Request | Fetch 15m klines (limit 300) | ⏸️ Wait 1s | 🧠 Analyze Logic | ## 2. Technical Analysis Core Calculates EMA, Bollinger Bands, RSI, and Volume anomalies via JS. |
| 🧠 Analyze Logic | Code | Compute EMA/BB/RSI/ATR; generate signal; dedup 1h | Get Klines | 🚦 Signal Check | ## 2. Technical Analysis Core Calculates EMA, Bollinger Bands, RSI, and Volume anomalies via JS. |
| 🚦 Signal Check | IF | Branch: signal found vs none | 🧠 Analyze Logic | 🛡 RSI Safety Check.; Loop Connector | ## 2. Technical Analysis Core Calculates EMA, Bollinger Bands, RSI, and Volume anomalies via JS. |
| 🛡 RSI Safety Check. | Code | Block trades in extreme RSI zones | 🚦 Signal Check (true) | 📰 Get News | ## 3. AI Sentiment Filter Scrapes news and uses LLM to validate the trade against risk factors. |
| 📰 Get News | HTTP Request | Fetch CryptoCompare news by category | 🛡 RSI Safety Check. | 📝 Format Context | ## 3. AI Sentiment Filter Scrapes news and uses LLM to validate the trade against risk factors. |
| 📝 Format Context | Code | Build short “news_context” prompt input | 📰 Get News | 🤖 AI Analysis | ## 3. AI Sentiment Filter Scrapes news and uses LLM to validate the trade against risk factors. |
| OpenRouter/OpenAI Model | LangChain Chat Model (OpenRouter) | Provides LLM model config/credentials | — | 🤖 AI Analysis (ai_languageModel) | ## 3. AI Sentiment Filter Scrapes news and uses LLM to validate the trade against risk factors. |
| 🤖 AI Analysis | LangChain Chain | News risk decision in strict JSON | 📝 Format Context; OpenRouter/OpenAI Model | 🛠️ Merge & Clean Data | ## 3. AI Sentiment Filter Scrapes news and uses LLM to validate the trade against risk factors. |
| 🛠️ Merge & Clean Data | Code | Merge TA + AI output; parse/clean JSON | 🤖 AI Analysis | Get Exchange Rules; 📢 Notify Telegram | ## 3. AI Sentiment Filter Scrapes news and uses LLM to validate the trade against risk factors. |
| 📢 Notify Telegram | Telegram | Send formatted alert to channel | 🛠️ Merge & Clean Data | Loop Connector | ## 4. Telegram Notification Sends analysis and alerts to your channel. |
| Get Exchange Rules | HTTP Request | Fetch symbol rules to compute precision | 🛠️ Merge & Clean Data | 📝 Prep String | ## 5. Paper Trading Execution Loop Prepares parameters, signs the request (HMAC SHA256), and executes on Binance Testnet. |
| 📝 Prep String | Code | Build signed request payloads (6 actions) | Get Exchange Rules | 🔄 Loop (SplitBatch) | ## 5. Paper Trading Execution Loop Prepares parameters, signs the request (HMAC SHA256), and executes on Binance Testnet. |
| 🔄 Loop (SplitBatch) | Split In Batches | Iterate each prepared REST call | 📝 Prep String; Execute (Paper Trading) | 🔐 Sign Request | ## 5. Paper Trading Execution Loop Prepares parameters, signs the request (HMAC SHA256), and executes on Binance Testnet. |
| 🔐 Sign Request | Crypto | HMAC SHA256 signature for Binance | 🔄 Loop (SplitBatch) | Execute (Paper Trading) | ## 5. Paper Trading Execution Loop Prepares parameters, signs the request (HMAC SHA256), and executes on Binance Testnet. |
| Execute (Paper Trading) | HTTP Request | POST signed request to Binance Testnet | 🔐 Sign Request | 🔄 Loop (SplitBatch) | ## 5. Paper Trading Execution Loop Prepares parameters, signs the request (HMAC SHA256), and executes on Binance Testnet. |
| Loop Connector | NoOp | Loop back connector for symbol iteration | 🚦 Signal Check (false); 📢 Notify Telegram | SplitInBatches | # 🤖 Crypto market analyzer & Paper trader (Scan/Analyze/Validate/Paper Trade setup steps) |
| Main Sticky | Sticky Note | Documentation/comment | — | — | # 🤖 Crypto market analyzer & Paper trader (Scan/Analyze/Validate/Paper Trade setup steps) |
| Warning Sticky | Sticky Note | Documentation/comment | — | — | ⚠️ **CONFIGURATION** Insert Binance TESTNET keys here. |
| Sticky Note1 | Sticky Note | Documentation/comment | — | — | ## 1. Market Data Collection Fetches top volume pairs, filters stablecoins, and prepares candidates. |
| Sticky Note2 | Sticky Note | Documentation/comment | — | — | ## 2. Technical Analysis Core Calculates EMA, Bollinger Bands, RSI, and Volume anomalies via JS. |
| Sticky Note3 | Sticky Note | Documentation/comment | — | — | ## 4. Telegram Notification Sends analysis and alerts to your channel. |
| Sticky Note4 | Sticky Note | Documentation/comment | — | — | ## 3. AI Sentiment Filter Scrapes news and uses LLM to validate the trade against risk factors. |
| Sticky Note5 | Sticky Note | Documentation/comment | — | — | ## 5. Paper Trading Execution Loop Prepares parameters, signs the request (HMAC SHA256), and executes on Binance Testnet. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1. Add **Schedule Trigger** node named `⏱️ Every 15 mins`
   2. Set cron to: `*/15 * * * *`

2. **Add configuration**
   1. Add **Set** node named `📝 MAIN CONFIG`
   2. Add fields:
      - Number: `TRADE_AMOUNT_USDT` = 100
      - Number: `LEVERAGE` = 1
      - String: `BINANCE_API_KEY` = your Testnet key
      - String: `BINANCE_SECRET` = your Testnet secret
      - String: `TELEGRAM_CHANNEL_ID` = `@your_channel` (or numeric chat ID)
      - String: `BLACKLIST` = `USDCUSDT,BUSDUSDT,USDPUSDT`
   3. Connect: `⏱️ Every 15 mins → 📝 MAIN CONFIG`

3. **Fetch tickers and filter candidates**
   1. Add **HTTP Request** `Get All Tickers`
      - GET `https://fapi.binance.com/fapi/v1/ticker/24hr`
   2. Add **Code** `🔍 Filter Candidates` with logic:
      - Filter to `USDT` symbols, sort by `quoteVolume`, remove blacklist, slice `10..150`, output `{symbol}`
   3. Add **Split In Batches** `SplitInBatches`
      - `batchSize = 10`
   4. Connect: `📝 MAIN CONFIG → Get All Tickers → 🔍 Filter Candidates → SplitInBatches`

4. **Throttle + klines**
   1. Add **Wait** `⏸️ Wait 1s`
      - seconds: `0.2`
   2. Add **HTTP Request** `Get Klines`
      - GET `https://fapi.binance.com/fapi/v1/klines`
      - Query params:
        - `symbol = {{ $('SplitInBatches').first().json.symbol }}`
        - `interval = 15m`
        - `limit = 300`
   3. Connect **SplitInBatches output 2** → `⏸️ Wait 1s → Get Klines`

5. **Technical analysis and signal branching**
   1. Add **Code** `🧠 Analyze Logic` implementing EMA200, BB(20,2), RSI(14), volume factor, ATR(14), and dedup (staticData cooldown).
   2. Add **IF** `🚦 Signal Check`
      - Condition: boolean `{{ $json.hasSignal }} is true`
   3. Add **NoOp** `Loop Connector`
   4. Connect: `Get Klines → 🧠 Analyze Logic → 🚦 Signal Check`
   5. Connect `🚦 Signal Check (false)` → `Loop Connector` → back to `SplitInBatches`

6. **RSI safety filter**
   1. Add **Code** `🛡 RSI Safety Check.`
      - Apply RSI thresholds and return empty array to stop if unsafe
   2. Connect: `🚦 Signal Check (true) → 🛡 RSI Safety Check.`

7. **News fetch + context formatting**
   1. Add **HTTP Request** `📰 Get News`
      - GET `https://min-api.cryptocompare.com/data/v2/news/`
      - Query:
        - `categories = {{ $('🧠 Analyze Logic').first().json.symbol.replace('USDT', '') }}`
        - `lang = EN`
   2. Add **Code** `📝 Format Context`
      - Build `news_context` from up to 3 headlines and merge into TA payload
   3. Connect: `🛡 RSI Safety Check. → 📰 Get News → 📝 Format Context`

8. **LLM risk check (OpenRouter)**
   1. Install/enable **n8n LangChain nodes** (`@n8n/n8n-nodes-langchain`) if not present.
   2. Create credentials for **OpenRouter** in n8n.
   3. Add **OpenRouter Chat Model** node named `OpenRouter/OpenAI Model`
      - Model: `openai/gpt-3.5-turbo`
      - Temperature: `0.2`
   4. Add **Chain LLM** node `🤖 AI Analysis`
      - Paste the prompt (risk manager, strict JSON output).
   5. Connect:
      - `📝 Format Context → 🤖 AI Analysis`
      - `OpenRouter/OpenAI Model (ai_languageModel) → 🤖 AI Analysis (ai_languageModel input)`

9. **Merge AI + TA**
   1. Add **Code** `🛠️ Merge & Clean Data`
      - Parse `json.text`, strip markdown fences, extract `reason`, merge into TA data as `ai_reason`.
   2. Connect: `🤖 AI Analysis → 🛠️ Merge & Clean Data`

10. **Telegram alert**
   1. Create Telegram Bot credentials in n8n (BotFather token).
   2. Add **Telegram** node `📢 Notify Telegram`
      - chatId: `{{ $('📝 MAIN CONFIG').first().json.TELEGRAM_CHANNEL_ID }}`
      - parse_mode: HTML
      - message template as in the workflow
   3. Connect: `🛠️ Merge & Clean Data → 📢 Notify Telegram → Loop Connector`

11. **Paper trading path (Binance Testnet)**
   1. Add **HTTP Request** `Get Exchange Rules`
      - GET `https://fapi.binance.com/fapi/v1/exchangeInfo?symbol={{ $json.symbol }}`
   2. Add **Code** `📝 Prep String`
      - Compute precision from exchange filters and build 6 request items (marginType, leverage, entry, SL, TP, trailing).
   3. Add **Split In Batches** `🔄 Loop (SplitBatch)` to iterate request items
   4. Add **Crypto** node `🔐 Sign Request`
      - HMAC SHA256 over `{{ $json.queryString }}`
      - Secret: set to Binance secret (preferably from credentials or config, not hardcoded)
      - Output property: `signature`
   5. Add **HTTP Request** `Execute (Paper Trading)`
      - POST `https://testnet.binancefuture.com{{ $json.endpoint }}?{{ $json.queryString }}&signature={{ $json.signature }}`
      - Header: `X-MBX-APIKEY = {{ $json.apiKey }}`
      - On error: continue
   6. Connect:
      - `🛠️ Merge & Clean Data → Get Exchange Rules → 📝 Prep String → 🔄 Loop (SplitBatch) (output 2) → 🔐 Sign Request → Execute (Paper Trading) → back to 🔄 Loop (SplitBatch)`

12. **Recommended fixes while reproducing (to match intent)**
   - Normalize `signal` values to a single convention everywhere, e.g. set in `🧠 Analyze Logic` to `"LONG"/"SHORT"` and update Telegram emoji logic accordingly.
   - In `🔐 Sign Request`, use `{{ $('📝 MAIN CONFIG').first().json.BINANCE_SECRET }}` as secret (or proper n8n credential) instead of `"YOUR_CREDENTIAL_HERE"`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Crypto market analyzer & Paper trader” workflow: scans pairs, computes EMA/BB/RSI, validates with AI news check, then paper trades on Testnet. | Sticky note: “Main Sticky” |
| Insert Binance TESTNET keys in `📝 MAIN CONFIG`. | Sticky note: “Warning Sticky” |
| Block labels embedded as sticky notes: 1) Market Data Collection, 2) Technical Analysis Core, 3) AI Sentiment Filter, 4) Telegram Notification, 5) Paper Trading Execution Loop. | Sticky notes: “Sticky Note1–5” |
| Paper trading execution uses signed REST requests (HMAC SHA256) and continues even if some calls fail. | Node: `Execute (Paper Trading)` (onError continue) |
| Critical implementation risk: `signal` casing inconsistency (“Long/Short” vs “LONG/SHORT”) can break RSI safety and invert paper trading side logic unless normalized. | Nodes: `🧠 Analyze Logic`, `🛡 RSI Safety Check.`, `📝 Prep String` |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.