Track portfolio performance and risk using Google Sheets and Alpha Vantage

https://n8nworkflows.xyz/workflows/track-portfolio-performance-and-risk-using-google-sheets-and-alpha-vantage-12487


# Track portfolio performance and risk using Google Sheets and Alpha Vantage

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow tracks a stock portfolio stored in Google Sheets. On a schedule, it reads holdings (symbol, buy price, quantity, buy date), fetches daily prices from Alpha Vantage, calculates performance and risk metrics (PnL, return %, CAGR, max drawdown), classifies each position (Healthy/Watch/Risk), and writes results back to the same sheet. It also sends an email alert if the workflow fails.

**Typical use cases**
- Daily portfolio monitoring in Google Sheets
- Lightweight risk monitoring via max drawdown + CAGR rules
- Automated updating of dashboard fields in the same portfolio tab

### Logical blocks
1.1 **Input reception & scheduling**: schedule trigger → read rows from Google Sheets → iterate each holding  
1.2 **Market data retrieval**: per holding, call Alpha Vantage TIME_SERIES_DAILY  
1.3 **Performance & risk analysis + persistence**: compute metrics and classification → append/update Google Sheet row  
1.4 **Error handling & alerting**: error trigger → send Gmail notification

---

## 2. Block-by-Block Analysis

### 2.1 Input reception & scheduling

**Overview:** Runs the workflow on a time schedule, loads all portfolio rows from Google Sheets, and processes them one by one using batching.

**Nodes involved**
- Run daily portfolio update
- Read portfolio holdings
- Process each stock

#### Node: Run daily portfolio update
- **Type / role:** `Schedule Trigger` — starts workflow automatically.
- **Configuration choices:** Runs every **23 hours at minute 59** (effectively daily, drifting slightly depending on start time).
- **Inputs / outputs:** Entry node → outputs to **Read portfolio holdings**.
- **Edge cases / failures:**
  - Misinterpreting schedule: it’s not “every day at a fixed clock time” but “every 23 hours at minute 59”.
  - If n8n is down at trigger time, the run is missed unless you use queueing/external scheduling.

#### Node: Read portfolio holdings
- **Type / role:** `Google Sheets` — reads portfolio table from a given spreadsheet + tab.
- **Configuration choices (interpreted):**
  - Uses OAuth2 credentials (`automations@techdome.ai`).
  - Targets document: Google Sheets URL provided.
  - Targets sheet/tab: **“Portfolio Performance”**.
  - Operation is the node’s default “read/get rows” behavior (operation not explicitly shown, but it outputs multiple items/rows).
- **Key fields expected in each row:** `Stock`, `BuyPrice`, `Quantity`, `BuyDate` (as referenced later).
- **Inputs / outputs:** From schedule → outputs list of items to **Process each stock**.
- **Edge cases / failures:**
  - OAuth token expiration / insufficient permissions.
  - Sheet name mismatch (tab must exist).
  - Data type issues (e.g., BuyDate not parseable, BuyPrice/Quantity blank).
  - If header names differ (e.g., “Symbol” instead of “Stock”), downstream expressions break.

#### Node: Process each stock
- **Type / role:** `Split In Batches` — iterates through rows.
- **Configuration choices:** Default batching options (batch size not shown; n8n default is typically 1 unless specified).
- **Inputs / outputs:**
  - Input: all holdings from **Read portfolio holdings**.
  - Output: second output index is connected to **Fetch daily stock prices** (unusual wiring; see note below).
- **Important connection detail:** In the provided connections, **output index 1** is used (the second output). Normally, SplitInBatches uses:
  - Output 0: current batch items
  - Output 1: “No Items Left” (loop end)
  
  Here it’s wired as:
  - `Process each stock` → (main, index **1**) → `Fetch daily stock prices`
  
  This is likely a wiring mistake unless intentionally repurposed. If index 1 is truly “no items left”, then the HTTP request might run only after batches complete (or not as expected).
- **Edge cases / failures:**
  - If miswired, the loop may not process each item.
  - If batch size > 1, downstream code references `$('Process each stock').item.json...` which may not match the currently iterated item as expected.

---

### 2.2 Market data retrieval

**Overview:** For each holding, fetches daily time-series prices from Alpha Vantage to compute current price and drawdown.

**Nodes involved**
- Fetch daily stock prices

#### Node: Fetch daily stock prices
- **Type / role:** `HTTP Request` — calls Alpha Vantage endpoint.
- **Configuration choices (interpreted):**
  - Method: default (GET) with `sendQuery: true`
  - URL: `https://www.alphavantage.co/query`
  - Query parameters:
    - `function = TIME_SERIES_DAILY`
    - `symbol = {{ $json.Stock }}`
    - `apikey = "YOUR_ALPHA_VANTAGE_API_KEY"` (currently a placeholder string including quotes)
- **Key expressions / variables:**
  - `={{ $json.Stock }}` assumes the current item has a `Stock` field from Google Sheets.
- **Inputs / outputs:** From **Process each stock** → outputs Alpha Vantage JSON to **Calculate returns and risk metrics**.
- **Edge cases / failures:**
  - API key placeholder: currently hard-coded as `"YOUR_ALPHA_VANTAGE_API_KEY"`; must be replaced.
  - Rate limits (Alpha Vantage free tier is very limited). You may receive:
    - “Thank you for using Alpha Vantage! Our standard API call frequency…” (no `Time Series (Daily)`).
  - Invalid symbol → missing time series block.
  - Response ordering: Alpha Vantage date keys are not guaranteed to be in the order assumed; code uses `Object.keys()` and then `dates[0]` as “latest”.

---

### 2.3 Performance & risk analysis + persistence

**Overview:** Computes portfolio metrics from buy inputs + fetched prices, calculates max drawdown from historical closes, classifies the holding, then updates the Google Sheet row matched by Stock.

**Nodes involved**
- Calculate returns and risk metrics
- Update portfolio performance

#### Node: Calculate returns and risk metrics
- **Type / role:** `Code` — performs calculations per stock item.
- **Execution mode:** `runOnceForEachItem`
- **Main calculations performed:**
  - Extract time series: `$json['Time Series (Daily)']`
  - Determine “latestDate” as `dates[0]` (see ordering risk below)
  - `CurrentPrice` from `4. close` on latestDate
  - Pull buy inputs via node reference:
    - `buyPrice = parseFloat($('Process each stock').item.json.BuyPrice)`
    - `quantity = parseFloat($('Process each stock').item.json.Quantity)`
    - `buyDate = new Date($('Process each stock').item.json.BuyDate)`
  - Invested/current values, PnL, PnL%
  - CAGR:
    - `yearsHeld = (now - buyDate)/.../365`
    - CAGR% computed if `yearsHeld > 0`
  - Max drawdown:
    - Iterates `dates.reverse()` (oldest → newest, assuming initial order newest→oldest)
    - Tracks peak close and min drawdown, returns percent (negative number)
  - Classification:
    - Default “Watch”
    - “Healthy” if `cagr >= 15` AND `maxDrawdown > -20`
    - “Risk” if `cagr < 5` OR `maxDrawdown <= -35`
- **Outputs:** Returns a normalized object with:
  - `Stock`, `BuyPrice`, `Quantity`, `BuyDate`, `CurrentPrice`, `InvestedValue`, `CurrentValue`, `PnL`, `PnLPercent`, `CAGR`, `MaxDrawdown`, `PortfolioStatus`, `LastUpdated`
- **Input / output connections:** From **Fetch daily stock prices** → to **Update portfolio performance**
- **Edge cases / failures:**
  - **Date ordering bug risk:** `Object.keys(timeSeries)` order is not guaranteed to be newest-first; selecting `dates[0]` may not be the latest trading day. Safer: sort dates descending.
  - **Missing BuyDate output:** In the return object, it uses `BuyDate: $json.BuyDate` (but `$json` here is Alpha Vantage response). That field likely doesn’t exist; it should use the value from the sheet item reference.
  - **NaN propagation:** If BuyPrice/Quantity are blank or non-numeric, results become NaN.
  - **BuyDate parsing:** Different locales/Sheets date formats may parse incorrectly in JavaScript.
  - **API error responses:** When time series missing, it returns nulls (handled), but classification fields may be missing or inconsistent.
  - **Division by zero:** If investedValue is 0, PnLPercent becomes Infinity/NaN.

#### Node: Update portfolio performance
- **Type / role:** `Google Sheets` — append or update the calculated results back into the portfolio tab.
- **Configuration choices (interpreted):**
  - Operation: **Append or Update**
  - Match column: `Stock` (unique key)
  - Sheet/tab: **Portfolio Performance** in the specified spreadsheet
  - Writes these columns from expressions:
    - `CurrentPrice`, `InvestedValue`, `CurrentValue`, `PnL`, `PnLPercent`, `CAGR`, `MaxDrawdown`, `LastUpdated`, `PortfolioStatus`, plus `Stock`
  - Removed from mapping schema (but exists in sheet): BuyPrice, Quantity, BuyDate (not updated here).
- **Inputs / outputs:** From **Calculate returns and risk metrics** → end.
- **Edge cases / failures:**
  - If `Stock` is blank/null, matching fails and may append unwanted rows.
  - Column header mismatch in Google Sheets prevents correct mapping.
  - Permission/auth errors (OAuth scope).
  - Type conversion disabled (`attemptToConvertTypes: false`), so numeric fields may be stored as strings unless Sheets auto-detects.

---

### 2.4 Error handling & alerting

**Overview:** If any node fails during execution, an error workflow branch triggers and sends a Gmail email containing node name, error message, and timestamp.

**Nodes involved**
- Workflow error trigger
- Send error notification email

#### Node: Workflow error trigger
- **Type / role:** `Error Trigger` — runs when the workflow errors.
- **Configuration choices:** Default.
- **Inputs / outputs:** Entry for error path → outputs to **Send error notification email**.
- **Edge cases / failures:**
  - Only triggers on workflow errors; logical “bad data” that doesn’t throw may not trigger.
  - If the error happens after partial updates, sheet may be partially updated.

#### Node: Send error notification email
- **Type / role:** `Gmail` — sends an alert email.
- **Configuration choices (interpreted):**
  - To: `user@example.com` (placeholder)
  - Subject: `Error  Alert ⚠️`
  - Body (text): includes `{{$json.node.name}}`, `{{$json.error.message}}`, `{{$json.timestamp}}`
  - Uses Gmail OAuth2 credentials (“Gmail credentials”)
- **Inputs / outputs:** From **Workflow error trigger** → end.
- **Edge cases / failures:**
  - Gmail OAuth token invalid / missing scopes.
  - Sending limits / blocked by Google policies.
  - Placeholder recipient not updated.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / overview |  |  | ## Track portfolio performance and risk using Google Sheets and Alpha Vantage - Overview\n\nThis workflow automatically tracks stock portfolio performance by reading holdings from Google Sheets, fetching daily market prices from Alpha Vantage, calculating returns and risk metrics, and updating the results back to the sheet. It also classifies each holding based on performance and drawdown.\n\n### How it works\n\nThe workflow starts on a schedule and reads portfolio data such as stock symbol, buy price, quantity, and buy date from Google Sheets. Each stock is processed one by one. For every holding, the workflow fetches recent price data from Alpha Vantage and calculates metrics like invested value, current value, profit and loss, return percentage, CAGR, and maximum drawdown.\nBased on return and drawdown, the workflow assigns a simple portfolio status (Healthy, Watch, or Risk), making the output easier to understand and act on. All calculated values are then written back to the sheet for tracking or dashboard use.\n\n### Setup steps\n\nCreate a Google Sheet tab named Portfolio Performance.\n\nConfigure Google Sheets and Alpha Vantage credentials.\n\nReplace the API key with your key before running the workflow. |
| Sticky Note1 | Sticky Note | Documentation (block label) |  |  | ### Portfolio Input and Scheduling\n\nThis workflow runs on a schedule and reads stock holdings such as (symbol, buy price, quantity, buy date) from Google Sheets. Each row represents one portfolio position. |
| Sticky Note2 | Sticky Note | Documentation (block label) |  |  | ### Market data retrieval\n\nFetches daily stock price data from the Alpha Vantage API for each portfolio holding to ensure consistent performance calculations. |
| Sticky Note3 | Sticky Note | Documentation (block label) |  |  | ### Performance and risk analysis\n\nCalculates invested value, current value, PnL, return %, CAGR, and maximum drawdown. Each stock is classified as Healthy, Watch, or Risk. After calculations, the latest values are written back to the same Google Sheet, updating existing rows using the stock symbol. |
| Sticky Note8 | Sticky Note | Documentation (block label) |  |  | ## 🚨 Error Handling \n\n \nCatches any workflow failure and posts an alert to Email.  \nIncludes node name, error message, and timestamp for quick debugging. |
| Run daily portfolio update | Schedule Trigger | Scheduled start |  | Read portfolio holdings | ### Portfolio Input and Scheduling\n\nThis workflow runs on a schedule and reads stock holdings such as (symbol, buy price, quantity, buy date) from Google Sheets. Each row represents one portfolio position. |
| Read portfolio holdings | Google Sheets | Read holdings from sheet | Run daily portfolio update | Process each stock | ### Portfolio Input and Scheduling\n\nThis workflow runs on a schedule and reads stock holdings such as (symbol, buy price, quantity, buy date) from Google Sheets. Each row represents one portfolio position. |
| Process each stock | Split In Batches | Iterate over holdings | Read portfolio holdings | Fetch daily stock prices | ### Portfolio Input and Scheduling\n\nThis workflow runs on a schedule and reads stock holdings such as (symbol, buy price, quantity, buy date) from Google Sheets. Each row represents one portfolio position. |
| Fetch daily stock prices | HTTP Request | Fetch TIME_SERIES_DAILY from Alpha Vantage | Process each stock | Calculate returns and risk metrics | ### Market data retrieval\n\nFetches daily stock price data from the Alpha Vantage API for each portfolio holding to ensure consistent performance calculations. |
| Calculate returns and risk metrics | Code | Compute performance/risk metrics and classification | Fetch daily stock prices | Update portfolio performance | ### Performance and risk analysis\n\nCalculates invested value, current value, PnL, return %, CAGR, and maximum drawdown. Each stock is classified as Healthy, Watch, or Risk. After calculations, the latest values are written back to the same Google Sheet, updating existing rows using the stock symbol. |
| Update portfolio performance | Google Sheets | Append/update metrics back to sheet | Calculate returns and risk metrics |  | ### Performance and risk analysis\n\nCalculates invested value, current value, PnL, return %, CAGR, and maximum drawdown. Each stock is classified as Healthy, Watch, or Risk. After calculations, the latest values are written back to the same Google Sheet, updating existing rows using the stock symbol. |
| Workflow error trigger | Error Trigger | Catch workflow failures |  | Send error notification email | ## 🚨 Error Handling \n\n \nCatches any workflow failure and posts an alert to Email.  \nIncludes node name, error message, and timestamp for quick debugging. |
| Send error notification email | Gmail | Email alert on failure | Workflow error trigger |  | ## 🚨 Error Handling \n\n \nCatches any workflow failure and posts an alert to Email.  \nIncludes node name, error message, and timestamp for quick debugging. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the Google Sheet**
   1. Create a spreadsheet and a tab named **Portfolio Performance**.
   2. Add columns (headers) at least: `Stock`, `BuyPrice`, `Quantity`, `BuyDate`.
   3. Add output columns to be filled: `CurrentPrice`, `InvestedValue`, `CurrentValue`, `PnL`, `PnLPercent`, `CAGR`, `MaxDrawdown`, `LastUpdated`, `PortfolioStatus`.

2) **Create node: “Run daily portfolio update”**
   - Node type: **Schedule Trigger**
   - Schedule rule: interval hours = **23**, trigger at minute = **59**
   - Connect to “Read portfolio holdings”.

3) **Create Google Sheets credentials**
   - In n8n Credentials: **Google Sheets OAuth2**
   - Authenticate with an account that has access to the spreadsheet.

4) **Create node: “Read portfolio holdings”**
   - Node type: **Google Sheets**
   - Select the Document (by URL) and Sheet/Tab: **Portfolio Performance**
   - Operation: read rows (default “Read”/“Get Many” depending on UI)
   - Connect from schedule trigger → to “Process each stock”.

5) **Create node: “Process each stock”**
   - Node type: **Split In Batches**
   - Use default batch size (or set explicitly to 1 for clarity)
   - Connect from “Read portfolio holdings” → to “Fetch daily stock prices”.
   - (Recommended) Use the **first/main output** for the looped items; reserve the second output for “done”.

6) **Create node: “Fetch daily stock prices”**
   - Node type: **HTTP Request**
   - Method: GET
   - URL: `https://www.alphavantage.co/query`
   - Enable “Send Query Parameters”
   - Add parameters:
     - `function`: `TIME_SERIES_DAILY`
     - `symbol`: expression `{{ $json.Stock }}`
     - `apikey`: your Alpha Vantage key (store in environment variable or n8n credential/parameter; do **not** keep the placeholder `"YOUR_ALPHA_VANTAGE_API_KEY"`).
   - Connect to “Calculate returns and risk metrics”.

7) **Create node: “Calculate returns and risk metrics”**
   - Node type: **Code**
   - Mode: **Run once for each item**
   - Paste the logic from the workflow (adjust recommended fixes):
     - Ensure you pull BuyDate from the sheet item, not from Alpha Vantage response.
     - Sort dates descending before choosing latest close.
   - Connect to “Update portfolio performance”.

8) **Create node: “Update portfolio performance”**
   - Node type: **Google Sheets**
   - Use same Google Sheets OAuth2 credentials
   - Document + sheet: same as above
   - Operation: **Append or Update**
   - Matching column: `Stock`
   - Map output fields to sheet columns using expressions from the Code node:
     - `Stock`, `CurrentPrice`, `InvestedValue`, `CurrentValue`, `PnL`, `PnLPercent`, `CAGR`, `MaxDrawdown`, `LastUpdated`, `PortfolioStatus`

9) **Set up error alerting**
   1. Add node: **Error Trigger** named “Workflow error trigger”.
   2. Add Gmail credentials (OAuth2) in n8n.
   3. Add node: **Gmail** named “Send error notification email”
      - To: your real recipient address
      - Subject/body similar to:
        - Node: `{{ $json.node.name }}`
        - Message: `{{ $json.error.message }}`
        - Time: `{{ $json.timestamp }}`
   4. Connect Error Trigger → Gmail node.

10) **Activate and test**
   - Run once manually with a small set of holdings.
   - Verify Alpha Vantage response contains `Time Series (Daily)`.
   - Confirm rows are updated (matched by `Stock`) rather than duplicated.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a Google Sheet tab named “Portfolio Performance”. | Workflow setup requirement (sticky overview). |
| Configure Google Sheets and Alpha Vantage credentials; replace API key before running. | Workflow setup requirement (sticky overview). |
| Error handling sends an email including node name, error message, and timestamp. | Sticky “🚨 Error Handling”. |

