Crypto RSI Alert System with EODHD, Telegram and TradingView Charts

https://n8nworkflows.xyz/workflows/crypto-rsi-alert-system-with-eodhd--telegram-and-tradingview-charts-7997


# Crypto RSI Alert System with EODHD, Telegram and TradingView Charts

### 1. Workflow Overview

This workflow implements a **Crypto RSI Alert System** that periodically or manually monitors a predefined cryptocurrency watchlist (BTC, ETH, SOL) using intraday 1-hour OHLCV data from the EOD Historical Data (EODHD) API. It calculates Wilder’s RSI(14) indicator to detect significant RSI crossings at thresholds 30 (oversold) and 70 (overbought). Upon detection of these signals, it sends formatted alerts via Telegram, including an interactive button linking to TradingView charts for the relevant symbol on Binance/USD.

**Target use cases:**  
- Crypto traders or analysts who want automated alerts for RSI-based trading signals on selected cryptocurrencies.  
- Integration of market data with messaging platforms for real-time notifications.  
- Visualization support using TradingView external chart links.

**Logical blocks:**

- **1.1 Input & Initialization:** Manual trigger + defining the crypto watchlist (symbols).  
- **1.2 Symbol Iteration & Data Retrieval:** Splitting the watchlist, looping over symbols, fetching intraday OHLCV from EODHD.  
- **1.3 RSI Calculation & Signal Detection:** Code node computes RSI(14) and detects 30/70 threshold crossings, preparing alert messages.  
- **1.4 Signal Filtering & Notification:** Conditional check for valid signals, then sending formatted alerts via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Initialization

- **Overview:**  
  Starts the workflow manually or by schedule and sets the list of symbols (watchlist) to monitor.

- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’` (Manual Trigger)  
  - `Edit Fields (watchlist)` (Set node)  
  - `Split Out` (Split Out node)

- **Node Details:**

  1. **When clicking ‘Execute workflow’**  
     - **Type:** Manual trigger node  
     - **Role:** Entry point to start the workflow manually; can be replaced or complemented by a schedule trigger in production.  
     - **Configuration:** No parameters; triggers workflow on button click.  
     - **Input/Output:** No input; outputs one item to next node.  
     - **Edge cases:** None inherent; user must manually trigger.  
     - **Sub-workflow:** None.

  2. **Edit Fields (watchlist)**  
     - **Type:** Set node  
     - **Role:** Defines the watchlist array containing crypto symbols in the format used by EODHD (e.g., `"BTC-USD.CC"`).  
     - **Configuration:** Assigns field `symbol` as an **array** of strings: `["BTC-USD.CC", "ETH-USD.CC", "SOL-USD.CC"]`.  
     - **Input/Output:** Receives trigger item; outputs one item with field `symbol` as array.  
     - **Edge cases:** Must ensure the field is typed as Array, not string; otherwise, split will fail.  
     - **Sticky Note:** Advises correct array typing and shows example output.  
     - **Sub-workflow:** None.

  3. **Split Out**  
     - **Type:** Split Out node  
     - **Role:** Explodes the array field `symbol` into multiple items, one per symbol, to allow sequential processing.  
     - **Configuration:** Field to split out: `symbol`.  
     - **Input/Output:** Input: single item with array; Output: N items each with a single symbol string.  
     - **Edge cases:** If input field missing or not an array, will fail or produce no output.  
     - **Sticky Note:** Explains explosion of array into items.  
     - **Sub-workflow:** None.

---

#### 1.2 Symbol Iteration & Data Retrieval

- **Overview:**  
  Processes each symbol individually by looping through them, requesting their intraday 1-hour OHLCV candle data from EODHD API.

- **Nodes Involved:**  
  - `Loop Over Items` (Split In Batches node)  
  - `HTTP Request (EODHD intraday 1h)` (HTTP Request node)

- **Node Details:**

  1. **Loop Over Items**  
     - **Type:** Split In Batches node  
     - **Role:** Iterates over each symbol item one by one to avoid mixing different symbols’ candle data.  
     - **Configuration:** Default batch size 1 (process one symbol at a time).  
     - **Input/Output:** Input: multiple items from `Split Out`; outputs one item per iteration.  
     - **Edge cases:** Large watchlists can increase runtime linearly; ensure batch size is 1 to isolate symbols properly.  
     - **Sticky Note:** Explains iteration logic and wiring pattern (Loop → HTTP → Code → Loop).  
     - **Sub-workflow:** None.

  2. **HTTP Request (EODHD intraday 1h)**  
     - **Type:** HTTP Request node  
     - **Role:** Fetches intraday 1-hour OHLCV data for the current symbol from EODHD API.  
     - **Configuration:**  
       - Method: GET  
       - URL: `https://eodhd.com/api/intraday/{{ $json.symbol }}` (symbol injected dynamically)  
       - Query Parameters: `interval=1h`, `fmt=json`, `api_token` from environment variable `EODHD_TOKEN`  
       - Response: Split Into Items enabled (each candle is one item)  
     - **Input/Output:** Receives one symbol item; outputs multiple candle items.  
     - **Version-specific:** Uses typeVersion 4.2 for enhanced HTTP features.  
     - **Edge cases:**  
       - API errors (invalid token, rate limits)  
       - Network timeouts  
       - Empty or malformed response  
     - **Sticky Note:** Notes token usage from env var, large output (~2-3k items).  
     - **Sub-workflow:** None.

---

#### 1.3 RSI Calculation & Signal Detection

- **Overview:**  
  Aggregates all candle data for the current symbol, computes Wilder’s RSI(14), detects crossings of 30/70 thresholds, and prepares alert messages including HTML formatting and TradingView chart URLs.

- **Nodes Involved:**  
  - `Code (RSI + message)` (Code node)

- **Node Details:**

  1. **Code (RSI + message)**  
     - **Type:** Code node (JavaScript)  
     - **Role:**  
       - Aggregates all candle items into one array  
       - Sorts candles by timestamp  
       - Calculates RSI(14) using Wilder’s method  
       - Detects signal crossings: entering or exiting oversold (≤30) and overbought (≥70) zones  
       - Prepares alert text in plain and HTML formats  
       - Constructs TradingView URL for Binance/USD chart using symbol base  
     - **Configuration:** Custom JS code embedded with constants: period=14, oversold=30, overbought=70.  
     - **Key expressions:**  
       - Input candles filtered and normalized by timestamp and close price  
       - RSI computed over closes  
       - Signal logic based on previous and current RSI  
       - Force test alert toggle (FORCE_ALERT = false by default)  
       - Symbol and URLs dynamically extracted from node context  
     - **Input/Output:** Input: candle items from HTTP request; Output: single JSON containing signal and alert info.  
     - **Edge cases:**  
       - Not enough candles to compute RSI (returns error JSON)  
       - Possible parsing errors if timestamps or closes are missing or malformed  
     - **Sticky Note:** Explains RSI computation and test toggle instructions.  
     - **Sub-workflow:** None.

---

#### 1.4 Signal Filtering & Notification

- **Overview:**  
  Checks if a valid RSI signal was detected and, if so, sends a Telegram message with the alert text and a button linking to the TradingView chart.

- **Nodes Involved:**  
  - `IF (has signal?)` (IF node)  
  - `Send a text message` (Telegram node)

- **Node Details:**

  1. **IF (has signal?)**  
     - **Type:** IF node  
     - **Role:** Filters items to pass only those where a signal exists (signal field present).  
     - **Configuration:** Checks if `$json.signal` exists (non-empty).  
     - **Input/Output:** Input from loop after RSI code; outputs to Telegram if true; discards otherwise.  
     - **Edge cases:** Potential false negatives if signal field is missing or null.  
     - **Sub-workflow:** None.

  2. **Send a text message**  
     - **Type:** Telegram node  
     - **Role:** Sends alert message to Telegram chat using bot API.  
     - **Configuration:**  
       - Text: HTML formatted alert message from `$json.alertTextHtml`  
       - Chat ID: from environment variable `TELEGRAM_CHAT_ID`  
       - Inline keyboard: button labeled “View chart” linking to TradingView URL from `$json.tradingViewUrl`  
       - Additional: Parse mode HTML, disable web page preview  
       - Credentials: Telegram bot token stored securely in n8n credentials  
     - **Input/Output:** Input from IF node; no output needed.  
     - **Edge cases:**  
       - Invalid bot token or chat ID causing authentication failure  
       - Telegram API rate limits or downtime  
       - Malformed HTML causing message send failure  
     - **Sticky Note:** Details parse mode, button, and chat ID usage.  
     - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                     | Node Type              | Functional Role                          | Input Node(s)                 | Output Node(s)                        | Sticky Note                                                                                      |
|-------------------------------|------------------------|----------------------------------------|------------------------------|-------------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger         | Entry trigger to start workflow         | —                            | Edit Fields (watchlist)              |                                                                                                |
| Edit Fields (watchlist)         | Set                    | Defines symbol watchlist array           | When clicking ‘Execute workflow’ | Split Out                           | Defines symbol array as String[]; example output: `{ symbol: ["BTC-USD.CC","ETH-USD.CC","SOL-USD.CC"] }` |
| Split Out                      | Split Out              | Expands symbol array into individual items | Edit Fields (watchlist)       | Loop Over Items                     | Explodes array into one item per symbol                                                       |
| Loop Over Items                | Split In Batches       | Processes one symbol at a time           | Split Out                    | IF (has signal?), HTTP Request (EODHD intraday 1h) | Processes one symbol per pass to avoid mixing candles; wiring Loop→HTTP→Code→Loop; Done→IF       |
| HTTP Request (EODHD intraday 1h) | HTTP Request          | Fetches intraday 1h OHLCV candle data   | Loop Over Items               | Code (RSI + message)                 | Fetches intraday 1h OHLCV; token from env var `EODHD_TOKEN`; Split Into Items enabled            |
| Code (RSI + message)           | Code                   | Computes RSI(14), detects signals, builds alert message and URL | HTTP Request (EODHD intraday 1h) | Loop Over Items                     | Computes RSI(14) (Wilder), detects 30/70 crossings, builds HTML message and TradingView URL     |
| IF (has signal?)               | IF                     | Filters items with RSI signals           | Loop Over Items               | Send a text message                 | Checks if `signal` field exists                                                               |
| Send a text message            | Telegram                | Sends alert message via Telegram         | IF (has signal?)              | —                                   | Sends HTML message; button links TradingView URL; chat ID from env var `TELEGRAM_CHAT_ID`       |
| Sticky Note — Overview         | Sticky Note             | Documentation overview                   | —                            | —                                   | Overview notes on workflow functionality and env vars required                                |
| Sticky Note — Watchlist        | Sticky Note             | Notes on watchlist array setup           | —                            | —                                   | Explains watchlist field type and example                                                    |
| Sticky Note — Split Out        | Sticky Note             | Notes on Split Out node                   | —                            | —                                   | Explains array explosion logic                                                               |
| Sticky Note — Loop             | Sticky Note             | Notes on Loop node                        | —                            | —                                   | Describes looping logic and wiring                                                           |
| Sticky Note — HTTP             | Sticky Note             | Notes on HTTP Request node                | —                            | —                                   | Explains EODHD data fetch and token usage                                                   |
| Sticky Note — Code             | Sticky Note             | Notes on Code node                        | —                            | —                                   | Explains RSI calculation and test toggle                                                    |
| Sticky Note — Telegram         | Sticky Note             | Notes on Telegram node                    | —                            | —                                   | Explains message formatting, button, and credentials                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: Manual start of workflow.

2. **Create Set node to define watchlist:**  
   - Name: `Edit Fields (watchlist)`  
   - Add field: `symbol` (type: Array)  
   - Value: `["BTC-USD.CC", "ETH-USD.CC", "SOL-USD.CC"]`  
   - Connect `When clicking ‘Execute workflow’` → `Edit Fields (watchlist)`.

3. **Create Split Out node:**  
   - Name: `Split Out`  
   - Configure "Field to split out": `symbol`  
   - Connect `Edit Fields (watchlist)` → `Split Out`.

4. **Create Split In Batches node:**  
   - Name: `Loop Over Items`  
   - Default batch size (1) to process one symbol at a time.  
   - Connect `Split Out` → `Loop Over Items`.

5. **Create HTTP Request node:**  
   - Name: `HTTP Request (EODHD intraday 1h)`  
   - Method: GET  
   - URL: `https://eodhd.com/api/intraday/{{ $json.symbol }}` (use expression)  
   - Query parameters:  
     - `interval` = `1h`  
     - `fmt` = `json`  
     - `api_token` = set to environment variable `EODHD_TOKEN` (e.g., `={{ $env.EODHD_TOKEN }}`)  
   - Enable "Split Into Items" in options (each candle is one item).  
   - Connect `Loop Over Items` (second output) → `HTTP Request (EODHD intraday 1h)`.

6. **Create Code node:**  
   - Name: `Code (RSI + message)`  
   - Paste the provided JavaScript code that:  
     - Aggregates candle items  
     - Calculates Wilder’s RSI(14)  
     - Detects 30/70 crossings  
     - Builds HTML and plain alert messages  
     - Constructs TradingView URL for Binance/USD  
   - Connect `HTTP Request (EODHD intraday 1h)` → `Code (RSI + message)`.

7. **Connect Code node back to Loop:**  
   - Connect `Code (RSI + message)` → `Loop Over Items` (main output) to continue looping.

8. **Create IF node:**  
   - Name: `IF (has signal?)`  
   - Condition: Check if `signal` field exists and is not empty (`{{$json.signal}}` exists)  
   - Connect `Loop Over Items` (main output) → `IF (has signal?)`.

9. **Create Telegram node:**  
   - Name: `Send a text message`  
   - Text: Use `{{$json.alertTextHtml}}` expression for message body (HTML formatted)  
   - Chat ID: Use environment variable `TELEGRAM_CHAT_ID` (e.g., `={{ $env.TELEGRAM_CHAT_ID }}`)  
   - Inline keyboard: Add one button  
     - Text: `View chart`  
     - URL: `{{$json.tradingViewUrl}}`  
   - Additional fields:  
     - Parse mode: HTML  
     - Disable web page preview: true  
   - Set Telegram API credentials (bot token).  
   - Connect `IF (has signal?)` (true output) → `Send a text message`.

10. **Environment Variables Setup:**  
    - Set `EODHD_TOKEN` with your EOD Historical Data API key.  
    - Set `TELEGRAM_CHAT_ID` with your Telegram chat identifier.

11. **Credentials Setup:**  
    - Create Telegram API credentials storing your Telegram bot token securely.

12. **Testing:**  
    - Use manual trigger to run workflow.  
    - Optionally, in Code node, toggle `FORCE_ALERT` to `true` for test alerts.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Environment variables `EODHD_TOKEN` and `TELEGRAM_CHAT_ID` must be configured in the n8n environment prior to running workflow.             | Critical for API authentication and Telegram chat routing.       |
| The workflow uses TradingView links targeting Binance exchange and USD pairs for visualization (e.g., BTCUSD).                              | URL pattern: https://www.tradingview.com/symbols/{BASE}USD/?exchange=BINANCE |
| To test Telegram message formatting without waiting for real signals, toggle `FORCE_ALERT` and `FORCE_SIGNAL` in the Code node.             | Useful for debugging and confirming alert delivery.              |
| RSI calculation implements Wilder’s RSI using a pure JavaScript function over sorted candles.                                                | Ensures accurate technical indicator computation within node.   |
| Telegram messages use HTML parse mode with inline keyboard buttons; malformed HTML may cause delivery errors.                               | Use only supported tags and properly escape content if modified. |
| The watchlist is defined as an array of symbols formatted as per EODHD conventions (e.g., `BTC-USD.CC`).                                   | Adhere strictly to symbol format to avoid API errors.            |

---

**Disclaimer:** The provided text derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing fully complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.