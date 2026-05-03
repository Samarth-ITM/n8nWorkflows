Binance SM Indicators Webhook Tool

https://n8nworkflows.xyz/workflows/binance-sm-indicators-webhook-tool-4747


# Binance SM Indicators Webhook Tool

---

### 1. Workflow Overview

The **Binance SM Indicators Webhook Tool** is a backend n8n workflow designed to compute key technical indicators for cryptocurrency trading pairs using Binance candlestick (kline) data. It serves as a centralized processor that accepts webhook POST requests for different time intervals (15 minutes, 1 hour, 4 hours, and 1 day), fetches raw kline data from Binance’s public API, calculates six major technical indicators (RSI, MACD, Bollinger Bands, SMA, EMA, ADX), and returns a structured JSON response with these indicators.

The workflow is organized into four parallel logical blocks, each corresponding to a timeframe (15m, 1h, 4h, 1d). Each block follows a similar pattern:

- **Input Reception via Webhook**: Receives symbol and triggers the workflow.
- **Binance API Data Fetching**: Queries Binance API for latest 40 klines.
- **Data Merging**: Merges all raw data arrays into a single array.
- **Indicator Calculations**: Calculates RSI, MACD, Bollinger Bands, SMA, EMA, ADX using code nodes.
- **Output Merging and Response**: Merges all results and responds to the webhook caller.

This modular design allows reuse of the same calculation logic across multiple timeframes while maintaining scalability and clarity.

Logical Blocks:

- **1.1 15-Minute Indicators Block**
- **1.2 1-Hour Indicators Block**
- **1.3 4-Hour Indicators Block**
- **1.4 1-Day Indicators Block**

Each block is self-contained with its own webhook entry point, API request node, multiple calculator nodes, and response node.

---

### 2. Block-by-Block Analysis

---

#### 1.1 15-Minute Indicators Block

**Overview:**  
Handles incoming POST requests for 15-minute interval indicators, fetches Binance 15m klines for a given symbol, computes six indicators, merges results, and responds with structured data.

**Nodes Involved:**  
- Webhook 15m Indicators  
- HTTP Request  
- Merge Into 1 Array  
- Calculate Bollinger Bands  
- Calculate RSI  
- Calculate MACD  
- Calculate SMA  
- Calculate EMA  
- Calculate ADX  
- Merge 15 min Indicators  
- Respond to 15m Webhook

**Node Details:**

- **Webhook 15m Indicators**  
  - Type: Webhook Trigger  
  - Role: Entry point for 15m indicator requests; listens on HTTP POST at path `39cc366c-af5f-472a-9d48-bbe30a4fe3ea`  
  - Configuration: HTTP Method POST, responseMode set to respond via Respond node  
  - Inputs: External HTTP POST payload  
  - Outputs: Passes payload json to next node  
  - Failure cases: Missing or malformed payload, invalid symbol  
  - Version: 2

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Calls Binance API endpoint `/api/v3/klines` to fetch 15m candlestick data  
  - Configuration: Sends query parameters `symbol` (from webhook JSON body), `interval=15m`, `limit=40`  
  - Inputs: From webhook node  
  - Outputs: Raw Binance kline array JSON  
  - Failure cases: Network failure, Binance API downtime, invalid symbol, rate limiting  
  - Version: 4.2

- **Merge Into 1 Array**  
  - Type: Code Node  
  - Role: Merges all incoming items into a single JSON object with property `klines` holding the array of raw klines  
  - Configuration: JavaScript code to map all inputs into one array  
  - Inputs: From HTTP Request node  
  - Outputs: Single item with merged klines array  
  - Failure cases: Empty input, unexpected input format  
  - Version: 2

- **Calculate Bollinger Bands**  
  - Type: Code Node  
  - Role: Calculates 20-period Bollinger Bands (basis, upper, lower) using standard deviation and closing prices  
  - Configuration: Uses period=20, stdMultiplier=2; extracts closes from klines, validates data length  
  - Inputs: Merged klines array  
  - Outputs: Array of objects with timestamp, close, bb_basis, bb_upper, bb_lower  
  - Failure cases: Not enough data (<20 closes), invalid klines format  
  - Version: 2

- **Calculate RSI**  
  - Type: Code Node  
  - Role: Calculates 14-period RSI from closing prices  
  - Configuration: Period=14; uses standard RSI formula with gains/losses averaging  
  - Inputs: Merged klines array  
  - Outputs: Array of objects with timestamp, close, rsi  
  - Failure cases: Not enough data (<15 closes), invalid data  
  - Version: 2

- **Calculate MACD**  
  - Type: Code Node  
  - Role: Calculates MACD line, signal line, and histogram with periods 12, 26, 9  
  - Configuration: Uses EMA helper function, validates data length (at least 35 closes)  
  - Inputs: Merged klines array  
  - Outputs: Array of objects with timestamp, close, macd, signal, histogram  
  - Failure cases: Insufficient data, invalid data format  
  - Version: 2

- **Calculate SMA**  
  - Type: Code Node  
  - Role: Calculates 20-period Simple Moving Average of close prices  
  - Configuration: Period=20  
  - Inputs: Merged klines array  
  - Outputs: Array of objects with timestamp, close, sma  
  - Failure cases: Not enough data (<20 closes)  
  - Version: 2

- **Calculate EMA**  
  - Type: Code Node  
  - Role: Calculates 20-period Exponential Moving Average  
  - Configuration: Period=20; starts with SMA for first period, then EMA formula  
  - Inputs: Merged klines array  
  - Outputs: Array of objects with timestamp, close, ema  
  - Failure cases: Not enough data (<20 closes)  
  - Version: 2

- **Calculate ADX**  
  - Type: Code Node  
  - Role: Calculates 14-period Average Directional Index (ADX) including PlusDI and MinusDI  
  - Configuration: Uses highs, lows, closes; smooths TR, +DM, -DM; calculates DI and DX; smooths into ADX  
  - Inputs: Merged klines array  
  - Outputs: Array of objects with timestamp, close, adx, plusDI, minusDI  
  - Failure cases: Insufficient data (<15 closes), invalid data  
  - Version: 2

- **Merge 15 min Indicators**  
  - Type: Merge Node  
  - Role: Merges outputs from all six indicator calculations into one composite dataset  
  - Configuration: Number of inputs = 6 (one per indicator)  
  - Inputs: Outputs from all six calculation nodes  
  - Outputs: Merged indicator data array  
  - Failure cases: Missing input from any indicator node, partial data  
  - Version: 3.1

- **Respond to 15m Webhook**  
  - Type: Respond to Webhook  
  - Role: Sends back the final merged indicator data as HTTP response to caller  
  - Configuration: Responds with all incoming items  
  - Inputs: From Merge 15 min Indicators node  
  - Outputs: HTTP response  
  - Failure cases: No data to respond, timeout  
  - Version: 1.3

---

#### 1.2 1-Hour Indicators Block

**Overview:**  
Processes webhook requests for 1-hour interval indicators, fetching 1h klines, calculating indicators, merging, and responding similarly to 15m block.

**Nodes Involved:**  
- Webhook 1h Indicators  
- HTTP Request 1h  
- Merge Into 1 Array 1h  
- Calculate Bollinger Bands(1h)  
- Calculate RSI(1h)  
- Calculate MACD(1h)  
- Calculate SMA(1h)  
- Calculate EMA(1h)  
- Calculate ADX1  
- Merge 1h Indicators  
- Respond to 1h Webhook

**Node Details:**  
All these nodes mirror the 15m block but operate on 1-hour interval data (`interval=1h`) and have analogous calculation logic with same parameters and validation. Notable differences:

- Webhook path: `78948764-5cdb-4808-8ef9-2155f10dd721`  
- HTTP Request fetches 1h klines  
- Merge and calculation nodes labeled with `(1h)` suffix  
- ADX calculation node named `Calculate ADX1` (same logic)  
- Respond node returns 1h merged data

Failure modes and version requirements are consistent with 15m block.

---

#### 1.3 4-Hour Indicators Block

**Overview:**  
Handles webhook POSTs for 4-hour interval, performs data fetch, indicator calculations, merging, and sends back the results.

**Nodes Involved:**  
- Webhook 4h Indicators  
- HTTP Request 4h  
- Merge Into 1 Array 4h  
- Calculate Bollinger Bands(4h)  
- Calculate RSI(4h)  
- Calculate MACD(4h)  
- Calculate SMA(4h)  
- Calculate EMA(4h)  
- Calculate ADX (4h)  
- Merge 4h Indicators  
- Respond to 4h Webhook

**Node Details:**  
Same structure as previous blocks, but with `interval=4h` for API requests, and node names suffixed with `(4h)`. Calculation code is identical in logic and parameters to other timeframes but operates on 4h data.  
Failure cases and version requirements match prior blocks.

---

#### 1.4 1-Day Indicators Block

**Overview:**  
Processes webhook requests for daily interval data, fetches daily klines, calculates all six indicators, merges, and responds.

**Nodes Involved:**  
- Webhook 1d Indicators  
- HTTP Request 1d  
- Merge Into 1 Array 1d  
- Calculate Bollinger Bands(1d)  
- Calculate RSI(1d)  
- Calculate MACD(1d)  
- Calculate SMA(1d)  
- Calculate EMA(1d)  
- Calculate ADX (1d)  
- Merge 1d Indicators  
- Respond to 1d Webhook

**Node Details:**  
This block mirrors the others but focuses on 1-day interval klines (`interval=1d`). Node names are suffixed `(1d)`. The indicator calculation code is consistent with other timeframes.  
Failure modes and version requirements remain the same.

---

### 3. Summary Table

| Node Name                | Node Type                | Functional Role                              | Input Node(s)                       | Output Node(s)               | Sticky Note                                                                                   |
|--------------------------|--------------------------|----------------------------------------------|-----------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Webhook 15m Indicators   | Webhook Trigger          | Entry point for 15m indicator requests       | External HTTP POST                 | HTTP Request                 | Entry point for 15m data, triggers API pull and calculations                                 |
| HTTP Request             | HTTP Request             | Fetches 15m kline data from Binance API      | Webhook 15m Indicators            | Merge Into 1 Array           | Fetches Binance API klines for 15m interval                                                  |
| Merge Into 1 Array       | Code Node                | Merges raw klines into single array           | HTTP Request                     | Calculate Bollinger Bands etc.| Merges raw API data for indicator calculations                                               |
| Calculate Bollinger Bands| Code Node                | Calculates 20-period Bollinger Bands          | Merge Into 1 Array               | Merge 15 min Indicators      | Calculates BBANDS for 15m timeframe                                                          |
| Calculate RSI            | Code Node                | Calculates 14-period RSI                       | Merge Into 1 Array               | Merge 15 min Indicators      | Calculates RSI for 15m timeframe                                                             |
| Calculate MACD           | Code Node                | Calculates MACD line, signal, histogram       | Merge Into 1 Array               | Merge 15 min Indicators      | Calculates MACD for 15m timeframe                                                            |
| Calculate SMA            | Code Node                | Calculates 20-period SMA                       | Merge Into 1 Array               | Merge 15 min Indicators      | Calculates SMA for 15m timeframe                                                             |
| Calculate EMA            | Code Node                | Calculates 20-period EMA                       | Merge Into 1 Array               | Merge 15 min Indicators      | Calculates EMA for 15m timeframe                                                             |
| Calculate ADX            | Code Node                | Calculates 14-period ADX and DI values        | Merge Into 1 Array               | Merge 15 min Indicators      | Calculates ADX for 15m timeframe                                                             |
| Merge 15 min Indicators  | Merge Node               | Merges all 6 indicators for 15m timeframe     | All indicator code nodes         | Respond to 15m Webhook       | Merges all indicator outputs                                                                 |
| Respond to 15m Webhook   | Respond to Webhook       | Sends final 15m indicator JSON response       | Merge 15 min Indicators          | HTTP Response               | Responds to webhook caller with indicators                                                   |
| Webhook 1h Indicators    | Webhook Trigger          | Entry point for 1h indicator requests         | External HTTP POST               | HTTP Request 1h              | Entry point for 1h data, triggers API pull and calculations                                  |
| HTTP Request 1h          | HTTP Request             | Fetches 1h kline data from Binance API        | Webhook 1h Indicators            | Merge Into 1 Array 1h       | Fetches Binance API klines for 1h interval                                                  |
| Merge Into 1 Array 1h    | Code Node                | Merges raw 1h klines into single array         | HTTP Request 1h                 | Calculate Bollinger Bands(1h) etc.| Merges raw API data for 1h indicator calculations                                           |
| Calculate Bollinger Bands(1h)| Code Node            | Calculates 20-period Bollinger Bands (1h)     | Merge Into 1 Array 1h           | Merge 1h Indicators         | Calculates BBANDS for 1h timeframe                                                          |
| Calculate RSI(1h)        | Code Node                | Calculates 14-period RSI (1h)                   | Merge Into 1 Array 1h           | Merge 1h Indicators         | Calculates RSI for 1h timeframe                                                             |
| Calculate MACD(1h)       | Code Node                | Calculates MACD (1h)                            | Merge Into 1 Array 1h           | Merge 1h Indicators         | Calculates MACD for 1h timeframe                                                            |
| Calculate SMA(1h)        | Code Node                | Calculates SMA (1h)                             | Merge Into 1 Array 1h           | Merge 1h Indicators         | Calculates SMA for 1h timeframe                                                             |
| Calculate EMA(1h)        | Code Node                | Calculates EMA (1h)                             | Merge Into 1 Array 1h           | Merge 1h Indicators         | Calculates EMA for 1h timeframe                                                             |
| Calculate ADX1           | Code Node                | Calculates ADX (1h)                             | Merge Into 1 Array 1h           | Merge 1h Indicators         | Calculates ADX for 1h timeframe                                                             |
| Merge 1h Indicators      | Merge Node               | Merges all 6 indicators for 1h timeframe       | All 1h indicator nodes          | Respond to 1h Webhook       | Merges all 1h indicator outputs                                                             |
| Respond to 1h Webhook    | Respond to Webhook       | Sends final 1h indicator JSON response         | Merge 1h Indicators             | HTTP Response               | Responds to webhook caller with 1h indicators                                               |
| Webhook 4h Indicators    | Webhook Trigger          | Entry point for 4h indicator requests           | External HTTP POST              | HTTP Request 4h             | Entry point for 4h data, triggers API pull and calculations                                  |
| HTTP Request 4h          | HTTP Request             | Fetches 4h kline data from Binance API          | Webhook 4h Indicators           | Merge Into 1 Array 4h       | Fetches Binance API klines for 4h interval                                                  |
| Merge Into 1 Array 4h    | Code Node                | Merges raw 4h klines into single array           | HTTP Request 4h                | Calculate Bollinger Bands(4h) etc.| Merges raw API data for 4h indicator calculations                                           |
| Calculate Bollinger Bands(4h)| Code Node            | Calculates 20-period Bollinger Bands (4h)       | Merge Into 1 Array 4h          | Merge 4h Indicators         | Calculates BBANDS for 4h timeframe                                                          |
| Calculate RSI(4h)        | Code Node                | Calculates 14-period RSI (4h)                     | Merge Into 1 Array 4h          | Merge 4h Indicators         | Calculates RSI for 4h timeframe                                                             |
| Calculate MACD(4h)       | Code Node                | Calculates MACD (4h)                              | Merge Into 1 Array 4h          | Merge 4h Indicators         | Calculates MACD for 4h timeframe                                                            |
| Calculate SMA(4h)        | Code Node                | Calculates SMA (4h)                               | Merge Into 1 Array 4h          | Merge 4h Indicators         | Calculates SMA for 4h timeframe                                                             |
| Calculate EMA(4h)        | Code Node                | Calculates EMA (4h)                               | Merge Into 1 Array 4h          | Merge 4h Indicators         | Calculates EMA for 4h timeframe                                                             |
| Calculate ADX (4h)       | Code Node                | Calculates ADX (4h)                               | Merge Into 1 Array 4h          | Merge 4h Indicators         | Calculates ADX for 4h timeframe                                                             |
| Merge 4h Indicators      | Merge Node               | Merges all 6 indicators for 4h timeframe         | All 4h indicator nodes          | Respond to 4h Webhook       | Merges all 4h indicator outputs                                                             |
| Respond to 4h Webhook    | Respond to Webhook       | Sends final 4h indicator JSON response           | Merge 4h Indicators             | HTTP Response               | Responds to webhook caller with 4h indicators                                               |
| Webhook 1d Indicators    | Webhook Trigger          | Entry point for 1d indicator requests             | External HTTP POST              | HTTP Request 1d             | Entry point for 1d data, triggers API pull and calculations                                  |
| HTTP Request 1d          | HTTP Request             | Fetches 1d kline data from Binance API            | Webhook 1d Indicators           | Merge Into 1 Array 1d       | Fetches Binance API klines for 1d interval                                                  |
| Merge Into 1 Array 1d    | Code Node                | Merges raw 1d klines into single array             | HTTP Request 1d                | Calculate Bollinger Bands(1d) etc.| Merges raw API data for 1d indicator calculations                                           |
| Calculate Bollinger Bands(1d)| Code Node            | Calculates 20-period Bollinger Bands (1d)         | Merge Into 1 Array 1d          | Merge 1d Indicators         | Calculates BBANDS for 1d timeframe                                                          |
| Calculate RSI(1d)        | Code Node                | Calculates 14-period RSI (1d)                       | Merge Into 1 Array 1d          | Merge 1d Indicators         | Calculates RSI for 1d timeframe                                                             |
| Calculate MACD(1d)       | Code Node                | Calculates MACD (1d)                                | Merge Into 1 Array 1d          | Merge 1d Indicators         | Calculates MACD for 1d timeframe                                                            |
| Calculate SMA(1d)        | Code Node                | Calculates SMA (1d)                                 | Merge Into 1 Array 1d          | Merge 1d Indicators         | Calculates SMA for 1d timeframe                                                             |
| Calculate EMA(1d)        | Code Node                | Calculates EMA (1d)                                 | Merge Into 1 Array 1d          | Merge 1d Indicators         | Calculates EMA for 1d timeframe                                                             |
| Calculate ADX (1d)       | Code Node                | Calculates ADX (1d)                                 | Merge Into 1 Array 1d          | Merge 1d Indicators         | Calculates ADX for 1d timeframe                                                             |
| Merge 1d Indicators      | Merge Node               | Merges all 6 indicators for 1d timeframe           | All 1d indicator nodes          | Respond to 1d Webhook       | Merges all 1d indicator outputs                                                             |
| Respond to 1d Webhook    | Respond to Webhook       | Sends final 1d indicator JSON response             | Merge 1d Indicators             | HTTP Response               | Responds to webhook caller with 1d indicators                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Nodes (4 total):**  
   - For each timeframe (15m, 1h, 4h, 1d), add a Webhook node.  
   - Configure HTTP Method to POST.  
   - Set unique webhook paths (e.g., `/15m-indicators`, `/1h-indicators`, `/4h-indicators`, `/1d-indicators`).  
   - Set response mode to "Respond to Webhook" to use respond nodes downstream.

2. **Add HTTP Request Nodes (4 total):**  
   - For each timeframe, add HTTP Request node.  
   - Set URL to `https://api.binance.com/api/v3/klines`.  
   - Set Query Parameters:  
     - `symbol`: `={{$json.body.symbol}}` (taken from webhook input)  
     - `interval`: the corresponding timeframe string (`15m`, `1h`, `4h`, `1d`)  
     - `limit`: `40`  
   - Connect each HTTP Request node to its respective Webhook node.

3. **Add Code Nodes to Merge Raw API Data (4 total):**  
   - Add a Code node for each timeframe to merge multiple incoming items into one array.  
   - Use this JavaScript snippet:  
     ```js
     const klines = $input.all().map(item => item.json);
     return [{ json: { klines } }];
     ```  
   - Connect HTTP Request node output to this Code node input.

4. **Add 6 Code Nodes per timeframe for Indicator Calculations:**  
   - For each timeframe, create the following nodes with the provided scripts adapted for respective timeframe data:

     - **Calculate Bollinger Bands**: 20-period, std multiplier 2.  
     - **Calculate RSI**: 14-period RSI.  
     - **Calculate MACD**: periods 12, 26, 9.  
     - **Calculate SMA**: 20-period SMA.  
     - **Calculate EMA**: 20-period EMA.  
     - **Calculate ADX**: 14-period ADX including +DI and -DI.

   - Each node receives input from the merge Code node (klines array).  
   - Ensure the JavaScript logic matches the provided calculations, including data validation and error throwing for insufficient data.

5. **Add Merge Node per timeframe:**  
   - Add a Merge node configured with `numberInputs` set to 6.  
   - Connect outputs of all 6 indicator code nodes to this Merge node.

6. **Add Respond to Webhook Node per timeframe:**  
   - Connect the Merge node output to a Respond to Webhook node.  
   - Configure to respond with all incoming items.

7. **Connect Workflow:**  
   - For each timeframe:  
     `Webhook -> HTTP Request -> Merge Into 1 Array (Code) -> 6 Indicator Calculators -> Merge Indicators -> Respond`

8. **Credential Setup:**  
   - No credentials needed for Binance public API calls.

9. **Testing:**  
   - Test each webhook endpoint with a POST request containing:  
     ```json
     { "symbol": "BTCUSDT" }
     ```  
   - Validate the JSON response includes all six indicators with expected fields.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow processes Binance public API kline data and calculates RSI, MACD, Bollinger Bands, SMA, EMA, and ADX indicators for 15m, 1h, 4h, and 1d intervals. | Overall project purpose                                                                                 |
| Each timeframe block is modular and separates input reception, API calls, indicator calculations, and response for scalability.          | Architecture design                                                                                     |
| The workflow expects valid Binance spot trading pair symbols from webhook requests.                                                     | Input requirements                                                                                      |
| Binance API endpoint used: `https://api.binance.com/api/v3/klines`                                                                       | Binance API documentation: https://binance-docs.github.io/apidocs/spot/en/#kline-candlestick-data     |
| Indicator calculation formulas are implemented manually in JavaScript for full customization and consistent output formatting.         | Indicator algorithm source                                                                              |
| Error handling in code nodes includes validation for sufficient data length and proper array formats to avoid runtime exceptions.       | Robustness considerations                                                                              |
| Refer to the sticky note content within the workflow for comments on node roles and logic.                                               | Node inline documentation                                                                               |
| Author and licensing: Don Jayamaha via LinkedIn [http://linkedin.com/in/donjayamahajr](http://linkedin.com/in/donjayamahajr)             | Licensing and support info                                                                              |
| Proprietary design and calculations; reuse or modification requires permission.                                                           | Licensing and intellectual property notice                                                            |

---

**Disclaimer:**  
The provided workflow is an automated n8n integration tool, compliant with all applicable content policies and legal standards. It uses publicly accessible data and standard financial formulas to deliver technical indicators for cryptocurrency trading purposes.

---