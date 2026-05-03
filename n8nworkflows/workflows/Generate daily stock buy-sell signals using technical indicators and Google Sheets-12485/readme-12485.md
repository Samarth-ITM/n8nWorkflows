Generate daily stock buy/sell signals using technical indicators and Google Sheets

https://n8nworkflows.xyz/workflows/generate-daily-stock-buy-sell-signals-using-technical-indicators-and-google-sheets-12485


# Generate daily stock buy/sell signals using technical indicators and Google Sheets

## 1. Workflow Overview

**Purpose:**  
This workflow runs daily to generate **rule-based Buy/Sell/Neutral signals** for a predefined list of stock tickers using **technical indicators (RSI, SMA20, SMA50, MACD)** computed from **daily historical prices** fetched via **Alpha Vantage**, then **stores/upserts** the results into a **Google Sheets dashboard**.

**Target use cases:**
- Lightweight daily technical analysis dashboard in Google Sheets
- Automated watchlist monitoring for a small set of tickers
- Foundation for alerts or downstream automation based on indicator signals

### 1.1 Stock Selection & Daily Scheduling
Runs once per day and defines which tickers to process, then iterates through them to control API usage.

### 1.2 Market Data Retrieval & Normalization
Fetches daily time series for each ticker from Alpha Vantage, extracts closing prices, sorts them chronologically, and truncates to a recent window.

### 1.3 Data Sufficiency Check
Ensures there are enough data points to compute indicators reliably (at least 21 closes required by the current logic).

### 1.4 Indicator Calculation & Signal Classification
Computes RSI(14), SMA(20), SMA(50), MACD (12/26) and a MACD “signal”, then classifies into Buy/Sell/Neutral using simple rules.

### 1.5 Dashboard Update (Google Sheets)
Writes the latest values into Google Sheets using “append or update” keyed by Stock symbol.

### 1.6 Error Handling & Alerting
Catches workflow-level failures and sends an email alert via Gmail with node name, message, and timestamp.

---

## 2. Block-by-Block Analysis

### Block 1 — Stock Selection & Daily Scheduling
**Overview:** Triggers once daily, builds a stock list, and feeds each symbol into a controlled loop to avoid bursty API calls.  
**Nodes involved:**  
- Schedule Trigger – Run Daily  
- Define Stock List  
- Loop Through Stocks  

#### Node: Schedule Trigger – Run Daily
- **Type / role:** Schedule Trigger; starts the workflow on a time-based schedule.
- **Configuration choices:** Uses an `interval` rule (the JSON shows an empty interval object; in the UI this must be configured to “every day” at a specific time).
- **Input/Output:** Entry node → outputs to **Define Stock List**.
- **Version notes:** `typeVersion 1.3` (standard).
- **Failure/edge cases:**
  - Misconfigured schedule (no actual interval/time) results in the workflow not running.
  - Timezone considerations (n8n instance timezone vs desired market timezone).

#### Node: Define Stock List
- **Type / role:** Code node; generates one item per stock symbol.
- **Configuration choices:**
  - Hardcoded list: `["AAPL", "MSFT", "TSLA", "NVDA"]`
  - Outputs items shaped as: `{ json: { stock: "AAPL" } }`
- **Key variables/logic:** `stocks.map(stock => ({ json: { stock } }))`
- **Input/Output:** Trigger → outputs to **Loop Through Stocks**.
- **Version notes:** Code node `typeVersion 2`.
- **Failure/edge cases:**
  - If you add invalid tickers, Alpha Vantage may return an error payload or a response without time series.
  - Very large lists may hit Alpha Vantage rate limits unless throttled.

#### Node: Loop Through Stocks
- **Type / role:** Split In Batches; iterates over items in batches to control flow.
- **Configuration choices:** Default options; batch size is not explicitly shown (n8n default is typically 1 unless changed in UI).
- **Connections detail:** In this workflow’s wiring, the node’s **second output** (index 1) is connected forward (see notes below).
- **Input/Output:**
  - Input: stock items from **Define Stock List**
  - Output: to **Fetch Historical Prices (Alpha Vantage)** (connected from output index 1)
- **Version notes:** `typeVersion 3`.
- **Failure/edge cases / design caveat:**
  - **Potential wiring issue:** SplitInBatches typically uses:
    - Output 0: “Current batch/items”
    - Output 1: “No items left” (done)
    
    Here, the workflow connects **output 1** to the fetch step. That can cause the fetch to run only when the loop is finished (or not as intended). In most implementations, **Fetch Historical Prices** should be connected to **output 0**, and the loop should be closed by connecting the last node back to SplitInBatches to request the next batch.
  - No loop-back connection exists, so even if output 0 were used, the batching might only process one batch depending on configuration.

**Sticky note coverage (applies to nodes in this block):**  
“### Stock Selection and Scheduling … runs daily … defines the list of stock symbols … processed individually to avoid API limits …”

---

### Block 2 — Market Data Retrieval & Normalization
**Overview:** Fetches daily historical data for each symbol from Alpha Vantage and normalizes it into a `closes` array suitable for indicator computation.  
**Nodes involved:**  
- Fetch Historical Prices (Alpha Vantage)  
- Prepare Price Series  

#### Node: Fetch Historical Prices (Alpha Vantage)
- **Type / role:** HTTP Request; calls Alpha Vantage API endpoint.
- **Configuration choices:**
  - URL: `https://www.alphavantage.co/query`
  - Query params:
    - `function=TIME_SERIES_DAILY`
    - `symbol={{ $json.stock }}`
    - `outputsize=compact` (typically ~100 most recent points)
    - `apikey="YOUR_ALPHA_VANTAGE_API_KEY"` (placeholder string)
  - Sends query parameters enabled.
- **Input/Output:** Receives `{stock}` from loop → outputs Alpha Vantage JSON to **Prepare Price Series**.
- **Version notes:** `typeVersion 4.3`.
- **Failure/edge cases:**
  - **API key not set** or invalid → Alpha Vantage returns an error JSON (often under fields like `Error Message` or `Note`).
  - **Rate limiting**: free tier frequently returns `Note` about call frequency; response may not include `Time Series (Daily)`.
  - Network timeouts / DNS / transient HTTP failures.
  - Data for some symbols may be missing (delisted, unsupported exchange).

#### Node: Prepare Price Series
- **Type / role:** Code node; extracts and cleans time series into a close-price array.
- **Configuration choices:**
  - `mode: runOnceForEachItem` (process each API response individually)
  - Logic:
    - Reads `const series = $json["Time Series (Daily)"];`
    - If missing, returns `null` (drops the item).
    - Sorts dates oldest → newest.
    - Extracts numeric closes from `"4. close"`.
    - Keeps only last 60 closes.
    - Sets:
      - `stock` from `$json["Meta Data"]["2. Symbol"]`
      - `closes` = last60Closes
- **Key variables/expressions:** `$json["Time Series (Daily)"]`, `$json["Meta Data"]["2. Symbol"]`
- **Input/Output:** From HTTP response → outputs `{stock, closes}` to **Check Price Data Exists**.
- **Version notes:** Code node `typeVersion 2`.
- **Failure/edge cases:**
  - If Alpha Vantage returns `Note` or `Error Message`, `series` is undefined → item is dropped (`return null;`).
  - If `Meta Data` missing, `stock` expression will throw and fail the item (potential workflow error).
  - Close values may parse to `NaN` if response shape changes.

**Sticky note coverage (applies to nodes in this block):**  
“### Market data retrieval … Fetches historical daily price data from Alpha Vantage and prepares clean price series …”

---

### Block 3 — Data Sufficiency Check
**Overview:** Ensures enough closes exist before computing indicators, preventing invalid math (especially SMA50).  
**Nodes involved:**  
- Check Price Data Exists  

#### Node: Check Price Data Exists
- **Type / role:** IF node; gates processing based on close series length.
- **Configuration choices:**
  - Condition: `{{ $json.closes.length }} > 20`
  - Strict type validation enabled.
- **Input/Output:**
  - Input: `{stock, closes}`
  - True branch → **Calculate RSI, MA & MACD**
  - False branch: not connected (stock is silently skipped)
- **Version notes:** `typeVersion 2`.
- **Failure/edge cases:**
  - If `closes` is missing (e.g., Prepare Price Series returned unexpected output), expression may fail.
  - **Logical sufficiency mismatch:** RSI14 needs at least 15 points, SMA20 needs 20, but **SMA50 needs 50**. The check `> 20` is not sufficient to guarantee SMA50 validity. With fewer than 50 closes, SMA50 will compute over too few elements and divide by 50, yielding incorrect results or `NaN`.
  - Consider changing threshold to `>= 50` (or adjusting SMA50 logic).

---

### Block 4 — Indicator Calculation & Signal Classification
**Overview:** Calculates RSI(14), SMA20, SMA50, MACD and a MACD signal, then applies simple rules to output a final signal.  
**Nodes involved:**  
- Calculate RSI, MA & MACD  

#### Node: Calculate RSI, MA & MACD
- **Type / role:** Code node; performs all indicator computations and signal logic.
- **Configuration choices / logic:**
  - Inputs: `closes`, `stock`
  - Helpers:
    - SMA: averages last `n` elements using `arr.slice(-n)`
    - EMA: iterative EMA starting from first element
  - RSI (14):
    - Loops from `closes.length - 15` to `closes.length - 2`
    - Computes gains/losses; uses `losses || 1` to avoid division by zero
  - MAs:
    - `sma20 = SMA(closes, 20)`
    - `sma50 = SMA(closes, 50)`
  - MACD:
    - `ema12` computed on `closes.slice(-26)` with period 12
    - `ema26` computed on `closes.slice(-26)` with period 26
    - `macd = ema12 - ema26`
    - `macdSignal = EMA([macd], 9)` (see caveat below)
  - Signal rules:
    - If `rsi < 30 && macd > macdSignal` → “Buy”
    - Else if `rsi > 70 && macd < macdSignal` → “Sell”
    - Else “Neutral”
  - Output fields: stock, rsi, sma20, sma50, macd, macdSignal, finalSignal, lastUpdated (ISO string)
- **Input/Output:** From IF true branch → outputs to **Store Indicator Signals in Google Sheets**.
- **Version notes:** Code node `typeVersion 2`.
- **Failure/edge cases / correctness caveats:**
  - **SMA50 with <50 points**: will divide by 50 but sum fewer items → incorrect.
  - **MACD signal is not actually computed correctly:** `EMA([macd], 9)` is an EMA over a single value, so it equals `macd`. That makes `macd > macdSignal` and `macd < macdSignal` almost always false, pushing output to “Neutral” except for floating edge behavior. A real MACD signal requires an EMA(9) over a *series of MACD values*, not a single macd.
  - RSI loop assumes at least 15 closes; if fewer, negative index slicing/looping can behave unexpectedly.
  - If any `closes` contains `NaN`, computations propagate `NaN`.

**Sticky note coverage (applies to nodes in this block and the next):**  
“### Indicator Calculation and Dashboard Update … Calculates RSI, moving averages, and MACD … rules classify … stores in Google Sheets.”

---

### Block 5 — Dashboard Update (Google Sheets)
**Overview:** Upserts indicator values into a Google Sheet tab keyed by the Stock symbol.  
**Nodes involved:**  
- Store Indicator Signals in Google Sheets  

#### Node: Store Indicator Signals in Google Sheets
- **Type / role:** Google Sheets node; append or update a row in a sheet.
- **Configuration choices:**
  - Operation: **Append or Update**
  - Matching column(s): `Stock`
  - Sheet (tab): “Technical Indicators”
  - Document: Google Spreadsheet at the provided URL
  - Column mapping (defined explicitly):
    - Stock, RSI, SMA20, SMA50, MACD, MACD Signal, FinalSignal (Buy / Sell / Neutral), Last Updated
  - Note: There is a column labeled **`Last Updated `** with a trailing space in the name.
- **Key expressions:** `={{ $json.stock }}`, `={{ $json.rsi }}`, etc.
- **Credentials:** Google Sheets OAuth2 (`automations@techdome.ai` in JSON).
- **Input/Output:** From indicator code node → writes to Sheets → end.
- **Version notes:** `typeVersion 4.7`.
- **Failure/edge cases:**
  - OAuth credential expired/invalid → auth errors.
  - Sheet/tab name mismatch or missing columns → mapping errors.
  - Trailing space in “Last Updated ” can cause confusion when recreating columns.
  - If multiple rows share the same Stock value, “appendOrUpdate” behavior depends on n8n’s matching semantics (may update first match).

---

### Block 6 — Error Handling & Alerting
**Overview:** Captures workflow execution errors and emails an alert containing node name, error message, and timestamp.  
**Nodes involved:**  
- Error Handler Trigger  
- Send Error Alert  

#### Node: Error Handler Trigger
- **Type / role:** Error Trigger; runs when the workflow errors.
- **Configuration choices:** Default; triggers on workflow error events.
- **Input/Output:** Entry for the error path → outputs to **Send Error Alert**.
- **Version notes:** `typeVersion 1`.
- **Failure/edge cases:**
  - Only triggers on failures that cause an execution error; “soft skips” (like IF false branch or `return null`) won’t alert.
  - If the Gmail node fails too, you may miss alerts.

#### Node: Send Error Alert
- **Type / role:** Gmail node; sends an email notification.
- **Configuration choices:**
  - To: `user@example.com` (placeholder)
  - Subject: `Error  Alert ⚠️`
  - Message (text): includes:
    - Node: `{{ $json.node.name }}`
    - Message: `{{ $json.error.message }}`
    - Time: `{{ $json.timestamp }}`
- **Credentials:** Gmail OAuth2 (“Gmail credentials” in JSON).
- **Input/Output:** From Error Trigger → sends email → end.
- **Version notes:** `typeVersion 2.2`.
- **Failure/edge cases:**
  - Gmail OAuth token expired/insufficient scopes.
  - Sending limits, restricted environments, or blocked outbound SMTP/API.

**Sticky note coverage (applies to nodes in this block):**  
“## 🚨 Error Handling … Catches any workflow failure and posts an alert to Email … Includes node name, error message, and timestamp …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger – Run Daily | Schedule Trigger | Daily workflow entrypoint | — | Define Stock List | ### Stock Selection and Scheduling<br>This workflow runs daily and defines the list of stock symbols to analyze. Each stock is processed individually to avoid API limits and ensure stable execution. |
| Define Stock List | Code | Creates one item per ticker | Schedule Trigger – Run Daily | Loop Through Stocks | ### Stock Selection and Scheduling<br>This workflow runs daily and defines the list of stock symbols to analyze. Each stock is processed individually to avoid API limits and ensure stable execution. |
| Loop Through Stocks | Split In Batches | Iterates through tickers | Define Stock List | Fetch Historical Prices (Alpha Vantage) *(wired from output index 1 in JSON)* | ### Stock Selection and Scheduling<br>This workflow runs daily and defines the list of stock symbols to analyze. Each stock is processed individually to avoid API limits and ensure stable execution. |
| Fetch Historical Prices (Alpha Vantage) | HTTP Request | Calls Alpha Vantage TIME_SERIES_DAILY | Loop Through Stocks | Prepare Price Series | ### Market data retrieval<br>Fetches historical daily price data from Alpha Vantage and prepares clean price series for indicator calculations. |
| Prepare Price Series | Code | Extracts sorted closes, keeps last 60 | Fetch Historical Prices (Alpha Vantage) | Check Price Data Exists | ### Market data retrieval<br>Fetches historical daily price data from Alpha Vantage and prepares clean price series for indicator calculations. |
| Check Price Data Exists | IF | Validates sufficient closes exist | Prepare Price Series | Calculate RSI, MA & MACD (true branch) | ### Indicator Calculation and Dashboard Update<br>Calculates RSI, moving averages, and MACD using standard formulas. Simple rules classify each stock as Buy, Sell, or Neutral. And stores indicator values and signals in Google Sheets. |
| Calculate RSI, MA & MACD | Code | Computes RSI/SMA/MACD + final signal | Check Price Data Exists | Store Indicator Signals in Google Sheets | ### Indicator Calculation and Dashboard Update<br>Calculates RSI, moving averages, and MACD using standard formulas. Simple rules classify each stock as Buy, Sell, or Neutral. And stores indicator values and signals in Google Sheets. |
| Store Indicator Signals in Google Sheets | Google Sheets | Upserts results into dashboard sheet | Calculate RSI, MA & MACD | — | ### Indicator Calculation and Dashboard Update<br>Calculates RSI, moving averages, and MACD using standard formulas. Simple rules classify each stock as Buy, Sell, or Neutral. And stores indicator values and signals in Google Sheets. |
| Error Handler Trigger | Error Trigger | Catches workflow execution errors | — | Send Error Alert | ## 🚨 Error Handling<br>Catches any workflow failure and posts an alert to Email.<br>Includes node name, error message, and timestamp for quick debugging. |
| Send Error Alert | Gmail | Sends error email with details | Error Handler Trigger | — | ## 🚨 Error Handling<br>Catches any workflow failure and posts an alert to Email.<br>Includes node name, error message, and timestamp for quick debugging. |
| Sticky Note | Sticky Note | Documentation / overview | — | — | ## Workflow Overview<br>### How it works<br>This workflow calculates technical indicators for selected stocks and stores the results in a Google Sheets dashboard.<br>It runs once per day and processes each stock individually. For every stock, the workflow fetches recent daily price data from a market data API. Using this data, it calculates common technical indicators such as RSI, Moving Averages, and MACD using standard formulas.<br>After the indicators are calculated, simple rule-based logic is applied to classify each stock as Buy, Sell, or Neutral. The final values and signal are then written to Google Sheets, creating a daily-updated technical analysis dashboard.<br>Basic checks are included to ensure enough price data is available before performing calculations. If valid data is not found, the workflow safely skips that stock and continues.<br>### Setup steps<br>1. Add your market data API key in the HTTP Request node.<br>2. Review and update the stock list in the stock list code node.<br>3. Create a Google Sheets tab for storing indicator results.<br>4. Connect your Google Sheets credentials in the Sheets node.<br>5. Save and activate the workflow. |
| Sticky Note1 | Sticky Note | Documentation / block label | — | — | ### Stock Selection and Scheduling<br>This workflow runs daily and defines the list of stock symbols to analyze. Each stock is processed individually to avoid API limits and ensure stable execution. |
| Sticky Note2 | Sticky Note | Documentation / block label | — | — | ### Market data retrieval<br>Fetches historical daily price data from Alpha Vantage and prepares clean price series for indicator calculations. |
| Sticky Note3 | Sticky Note | Documentation / block label | — | — | ### Indicator Calculation and Dashboard Update<br>Calculates RSI, moving averages, and MACD using standard formulas. Simple rules classify each stock as Buy, Sell, or Neutral. And stores indicator values and signals in Google Sheets. |
| Sticky Note8 | Sticky Note | Documentation / block label | — | — | ## 🚨 Error Handling<br>Catches any workflow failure and posts an alert to Email.<br>Includes node name, error message, and timestamp for quick debugging. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Generate daily buy/sell signals using technical indicators and Google Sheets* (or your preferred name).

2. **Add “Schedule Trigger”**
   - Node: **Schedule Trigger**
   - Configure it to run **once per day** (choose your time and timezone).
   - Connect: **Schedule Trigger → Define Stock List**

3. **Add “Define Stock List” (Code node)**
   - Node: **Code**
   - Set code to output one item per ticker, e.g.:
     - Create `stocks = ["AAPL","MSFT","TSLA","NVDA"]`
     - Return `stocks.map(stock => ({ json: { stock } }))`
   - Connect: **Define Stock List → Loop Through Stocks**

4. **Add “Loop Through Stocks” (Split In Batches)**
   - Node: **Split In Batches**
   - Set **Batch Size = 1** (recommended for rate-limited APIs).
   - Important wiring (recommended correct pattern):
     - Connect **Output 0 (Items)** → **Fetch Historical Prices (Alpha Vantage)**
     - After your final processing node (Google Sheets), connect back to **Split In Batches** to fetch the next batch.
   - (If you keep the JSON’s wiring as-is, verify which output is “items” vs “done” in your n8n version.)

5. **Add “Fetch Historical Prices (Alpha Vantage)” (HTTP Request)**
   - Node: **HTTP Request**
   - Method: **GET**
   - URL: `https://www.alphavantage.co/query`
   - Enable **Send Query Parameters**
   - Add query parameters:
     - `function` = `TIME_SERIES_DAILY`
     - `symbol` = `{{ $json.stock }}`
     - `outputsize` = `compact`
     - `apikey` = your Alpha Vantage key (store in environment variable/credential if possible)
   - Connect: **Fetch Historical Prices → Prepare Price Series**

6. **Add “Prepare Price Series” (Code node)**
   - Node: **Code**
   - Mode: **Run once for each item**
   - Implement logic to:
     - Read `$json["Time Series (Daily)"]`
     - Sort dates ascending
     - Extract `"4. close"` as numbers
     - Keep last 60 closes
     - Output `{ stock, closes }`
   - Connect: **Prepare Price Series → Check Price Data Exists**

7. **Add “Check Price Data Exists” (IF)**
   - Node: **IF**
   - Condition (Number):
     - Left: `{{ $json.closes.length }}`
     - Operation: **greater than**
     - Right: `20` (as in JSON)
   - Recommended improvement: set to **>= 50** if you keep SMA50.
   - Connect **True** → **Calculate RSI, MA & MACD**
   - (Optional) Connect **False** to a logging node if you want visibility into skipped stocks.

8. **Add “Calculate RSI, MA & MACD” (Code node)**
   - Node: **Code**
   - Compute and output:
     - `rsi`, `sma20`, `sma50`, `macd`, `macdSignal`, `finalSignal`, `lastUpdated`
   - Connect: **Calculate RSI, MA & MACD → Store Indicator Signals in Google Sheets**
   - Recommended improvement: compute MACD signal from a MACD series, not a single value.

9. **Add “Store Indicator Signals in Google Sheets”**
   - Node: **Google Sheets**
   - **Credentials:** Add/choose **Google Sheets OAuth2** credential with access to the target spreadsheet.
   - Operation: **Append or Update**
   - Document: select your Google Spreadsheet
   - Sheet: select/create tab (e.g., “Technical Indicators”)
   - Matching column: `Stock`
   - Map columns:
     - Stock ← `{{ $json.stock }}`
     - RSI ← `{{ $json.rsi }}`
     - SMA20 ← `{{ $json.sma20 }}`
     - SMA50 ← `{{ $json.sma50 }}`
     - MACD ← `{{ $json.macd }}`
     - MACD Signal ← `{{ $json.macdSignal }}`
     - FinalSignal (Buy / Sell / Neutral) ← `{{ $json.finalSignal }}`
     - Last Updated (watch for exact column name; avoid trailing spaces) ← `{{ $json.lastUpdated }}`
   - (If using SplitInBatches properly) connect **Google Sheets → Split In Batches** to continue looping.

10. **Add error handling**
    - Node: **Error Trigger**
    - Node: **Gmail**
      - Credentials: Gmail OAuth2
      - To: your recipient email
      - Subject: “Error Alert”
      - Body: include `{{ $json.node.name }}`, `{{ $json.error.message }}`, `{{ $json.timestamp }}`
    - Connect: **Error Trigger → Gmail**

11. **Activate the workflow**
    - Run once manually first to confirm:
      - Alpha Vantage response contains `Time Series (Daily)`
      - Sheet columns match exactly
      - Signals are being written/upserted as expected

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Add your market data API key in the HTTP Request node. | Alpha Vantage configuration note (from workflow sticky note) |
| Review and update the stock list in the stock list code node. | Watchlist customization (from workflow sticky note) |
| Create a Google Sheets tab for storing indicator results. | Dashboard setup (from workflow sticky note) |
| Connect your Google Sheets credentials in the Sheets node. | OAuth2 setup (from workflow sticky note) |
| Save and activate the workflow. | Operational step (from workflow sticky note) |
| Google Sheet used in the JSON: “Technical Indicators” tab in a spreadsheet. | https://docs.google.com/spreadsheets/d/1Mz-woYDtXtzF2bA9IqpdYh28IPA76nFx46WHgDJOZoI/edit |
| Disclaimer (provided by user): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… | Context: content compliance statement included in request |

