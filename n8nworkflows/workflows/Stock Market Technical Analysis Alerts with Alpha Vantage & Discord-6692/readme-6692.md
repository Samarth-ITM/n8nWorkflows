Stock Market Technical Analysis Alerts with Alpha Vantage & Discord

https://n8nworkflows.xyz/workflows/stock-market-technical-analysis-alerts-with-alpha-vantage---discord-6692


# Stock Market Technical Analysis Alerts with Alpha Vantage & Discord

### 1. Workflow Overview

This workflow, titled **"Moving Average Crossover Stock Alert Bot"**, is designed to monitor technical trading signals in the stock market by detecting **Golden Cross** and **Death Cross** events based on Simple Moving Averages (SMAs). It targets users interested in automated stock technical analysis and alerting via Discord for a selected list of stocks and ETFs.

The workflow logically divides into four main blocks:

- **1.1 Input Reception & Stock Selection**: Triggered daily on weekdays at 5 PM after market close, selects stocks for analysis.
- **1.2 Data Collection & Storage**: Fetches daily price data from Alpha Vantage, processes and stores it in a PostgreSQL database.
- **1.3 Technical Analysis Engine**: Queries historical data, computes 60-day and 120-day SMAs, detects crossover signals.
- **1.4 Alert Notification**: Formats and sends alerts to a Discord webhook based on detected signals (Golden Cross, Death Cross, or no signal).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Stock Selection

- **Overview:**  
  Initiates the workflow daily at 5 PM on weekdays and defines the list of stock ticker symbols to analyze.

- **Nodes Involved:**  
  - Trigger - Daily Close  
  - Set - Ticker List  
  - Split - Tickers  
  - Sticky Note (overview)

- **Node Details:**

  - **Trigger - Daily Close**  
    - Type: Schedule Trigger  
    - Role: Fires the workflow every weekday (Mon-Fri) at 17:00 (5 PM) to ensure the market close data is available.  
    - Configuration: Weekly interval; days 1-5; hour 17.  
    - Inputs: None  
    - Outputs: Set - Ticker List  
    - Edge cases: If the trigger fails or is delayed, alerts may be late or missed.

  - **Set - Ticker List**  
    - Type: Set  
    - Role: Defines the array of ticker symbols to monitor.  
    - Configuration: Sets `symbol` array to ["NVDA","JPM","PG","SPY"] (a subset of stocks mentioned in note).  
    - Inputs: Trigger node  
    - Outputs: Split - Tickers  
    - Notes: Contains detailed notes describing why each stock/ETF was chosen (e.g., NVDA as AI growth driver).  
    - Edge cases: If the list is empty or malformed, no data will be fetched.

  - **Split - Tickers**  
    - Type: SplitOut  
    - Role: Splits the symbol array into individual items for parallel processing.  
    - Configuration: Splits on field "symbol"  
    - Inputs: Set - Ticker List  
    - Outputs: Fetch Daily History  
    - Edge cases: Empty input array leads to no downstream processing.

  - **Sticky Note**  
    - Provides a high-level summary of the workflow purpose and schedule.

#### 2.2 Data Collection & Storage

- **Overview:**  
  Fetches daily stock price data from Alpha Vantage API, extracts yesterday's closing price, and stores it in a PostgreSQL table for historical analysis.

- **Nodes Involved:**  
  - Fetch Daily History  
  - Getting today's data (Code)  
  - Insert rows in a table (Postgres)  
  - Sticky Note1 (overview)

- **Node Details:**

  - **Fetch Daily History**  
    - Type: HTTP Request  
    - Role: Queries Alpha Vantage's TIME_SERIES_DAILY endpoint for each stock symbol.  
    - Configuration:  
      - URL: https://www.alphavantage.co/query  
      - Query parameters: function=TIME_SERIES_DAILY, symbol from split, apikey (placeholder "YOURKEYHERE"), outputsize=compact  
    - Inputs: Split - Tickers  
    - Outputs: Getting today's data  
    - Error Handling: Continues on error to avoid workflow halt on API issues.  
    - Edge cases: API rate limiting, invalid symbols, missing data, or network errors.

  - **Getting today's data**  
    - Type: Code  
    - Role: Processes Alpha Vantage JSON response to extract the previous trading day's date and closing price.  
    - Configuration: JavaScript code that:  
      - Validates the "Time Series (Daily)" object  
      - Sorts dates ascending and picks the second last date (yesterday)  
      - Returns JSON with symbol, date, and close price  
    - Inputs: Fetch Daily History  
    - Outputs: Insert rows in a table  
    - Edge cases: Missing or malformed time series, fewer than 2 days of data, API errors wrapped as JSON.

  - **Insert rows in a table**  
    - Type: PostgreSQL  
    - Role: Inserts/updates the historical stock data into the `historical_stocks` table with columns `Date`, `Close`, and `symbol`.  
    - Configuration:  
      - Table: public.historical_stocks  
      - Columns mapped from JSON fields  
      - Skip on conflict active (to avoid duplicate insertion errors)  
    - Inputs: Getting today's data  
    - Outputs: Execute a SQL query  
    - Edge cases: Database connection failures, schema mismatches, duplicate keys.

  - **Sticky Note1**  
    - Describes data collection steps and current monitored tickers.

#### 2.3 Technical Analysis Engine

- **Overview:**  
  Retrieves the last 121 days of historical price data from the database for each symbol, computes 60-day and 120-day simple moving averages (SMAs), and identifies Golden Cross or Death Cross signals by comparing current and previous SMA values.

- **Nodes Involved:**  
  - Execute a SQL query (Postgres)  
  - Compute 60/120 SMAs (Code)  
  - If (📈) (If node)  
  - If (📉) (If node)  
  - Set - Golden Cross Msg  
  - Set - Death Cross Msg  
  - Set - No Signal Msg  
  - Sticky Note2 (SMA explanation)  
  - Sticky Note4 (Signal Detection Logic)

- **Node Details:**

  - **Execute a SQL query**  
    - Type: PostgreSQL  
    - Role: Retrieves the latest 121 rows per symbol from `historical_stocks` ordered by date descending.  
    - Configuration:  
      - SQL query uses ROW_NUMBER() window function partitioned by symbol, ordered by Date DESC, filtered for selected symbols.  
    - Inputs: Insert rows in a table  
    - Outputs: Compute 60/120 SMAs  
    - Edge cases: Database query failures, empty results.

  - **Compute 60/120 SMAs**  
    - Type: Code  
    - Role: Calculates current and previous 60-day and 120-day SMAs for each symbol to detect crossover signals.  
    - Configuration: JavaScript code that:  
      - Groups data by symbol  
      - Sorts by newest date  
      - Calculates SMA for current day (offset 0) and previous day (offset 1) using a helper function  
      - Outputs JSON with symbol, sma60_current, sma120_current, sma60_previous, sma120_previous or error if insufficient data.  
    - Inputs: Execute a SQL query  
    - Outputs: If (📈)  
    - Edge cases: Insufficient data points (<121), errors in data parsing.

  - **If (📈)**  
    - Type: If  
    - Role: Detects if a Golden Cross occurred: yesterday 60-day SMA ≤ 120-day SMA AND today 60-day SMA > 120-day SMA.  
    - Inputs: Compute 60/120 SMAs  
    - Outputs:  
      - True: Set - Golden Cross Msg  
      - False: If (📉)  
    - Edge cases: Expression evaluation errors.

  - **If (📉)**  
    - Type: If  
    - Role: Detects if a Death Cross occurred: yesterday 60-day SMA ≥ 120-day SMA AND today 60-day SMA < 120-day SMA.  
    - Inputs: If (📈) false output  
    - Outputs:  
      - True: Set - Death Cross Msg  
      - False: Set - No Signal Msg  
    - Edge cases: Expression evaluation errors.

  - **Set - Golden Cross Msg**  
    - Type: Set  
    - Role: Formats the alert message for Golden Cross including emoji and stock symbol.  
    - Configuration: Sets field `content` with template string: "🟢📈 Golden Cross Alert for **{symbol}**! The 60-day SMA has crossed above the 120-day SMA."  
    - Inputs: If (📈) true output  
    - Outputs: HTTP Request  
    - Edge cases: Template variable resolution.

  - **Set - Death Cross Msg**  
    - Type: Set  
    - Role: Formats alert message for Death Cross similarly.  
    - Configuration: "🔴📉 Death Cross Alert for **{symbol}**! The 60-day SMA has crossed below the 120-day SMA."  
    - Inputs: If (📉) true output  
    - Outputs: HTTP Request

  - **Set - No Signal Msg**  
    - Type: Set  
    - Role: Sends a daily monitoring message if no crossover signals detected across all symbols.  
    - Configuration: Message includes all symbols sorted alphabetically: "🟡↔️ No crossover today for NVDA, JPM, PG, SPY. Monitoring continues…"  
    - Inputs: If (📉) false output  
    - Outputs: HTTP Request  
    - Note: Executes once per run.

  - **Sticky Note2**  
    - Explains SMA calculation and crossover detection requirements.

  - **Sticky Note4**  
    - Details the logic for Golden Cross and Death Cross detection conditions.

#### 2.4 Alert Notification

- **Overview:**  
  Sends formatted alert messages to a Discord channel via webhook for detected signals or ongoing monitoring.

- **Nodes Involved:**  
  - HTTP Request  
  - Sticky Note3 (overview)

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Posts alert content messages to Discord webhook endpoint.  
    - Configuration:  
      - Method: POST  
      - URL: Discord webhook URL (placeholder "YOURWEBHOOKHERE")  
      - Body: JSON with `content` field from previous Set node.  
    - Inputs: Set - Golden Cross Msg, Set - Death Cross Msg, Set - No Signal Msg  
    - Outputs: None  
    - Edge cases: Webhook URL invalid or revoked, network errors, Discord rate limits.

  - **Sticky Note3**  
    - Describes alert types, emoji usage, and message format for clarity.

---

### 3. Summary Table

| Node Name              | Node Type              | Functional Role                         | Input Node(s)               | Output Node(s)                | Sticky Note                                                  |
|------------------------|------------------------|---------------------------------------|-----------------------------|------------------------------|--------------------------------------------------------------|
| Sticky Note            | Sticky Note            | Overview of entire workflow            | —                           | —                            | 📊 Stock Market Analysis Workflow overview                   |
| Trigger - Daily Close  | Schedule Trigger       | Starts workflow daily at 5 PM weekdays | —                           | Set - Ticker List             |                                                              |
| Set - Ticker List      | Set                    | Defines ticker symbols array            | Trigger - Daily Close        | Split - Tickers               | Detailed ticker rationale                                   |
| Split - Tickers        | SplitOut               | Splits ticker array into individual items | Set - Ticker List            | Fetch Daily History           |                                                              |
| Fetch Daily History    | HTTP Request           | Fetches daily stock data from Alpha Vantage | Split - Tickers              | Getting today's data          |                                                              |
| Getting today's data   | Code                   | Extracts yesterday's close price       | Fetch Daily History          | Insert rows in a table        |                                                              |
| Insert rows in a table | PostgreSQL             | Stores daily close data into DB        | Getting today's data         | Execute a SQL query           |                                                              |
| Execute a SQL query    | PostgreSQL             | Retrieves last 121 days of data per symbol | Insert rows in a table       | Compute 60/120 SMAs           |                                                              |
| Compute 60/120 SMAs    | Code                   | Calculates SMAs and prepares crossover data | Execute a SQL query          | If (📈)                      | 🧮 Technical Analysis Engine explanation                      |
| If (📈)                | If                     | Detects Golden Cross condition          | Compute 60/120 SMAs          | Set - Golden Cross Msg, If (📉) | 🚦 Signal Detection Logic explanation                         |
| If (📉)                | If                     | Detects Death Cross condition           | If (📈)                     | Set - Death Cross Msg, Set - No Signal Msg | 🚦 Signal Detection Logic explanation                         |
| Set - Golden Cross Msg | Set                    | Prepares Golden Cross alert message     | If (📈) true                | HTTP Request                 |                                                              |
| Set - Death Cross Msg  | Set                    | Prepares Death Cross alert message      | If (📉) true                | HTTP Request                 |                                                              |
| Set - No Signal Msg    | Set                    | Prepares no crossover alert message     | If (📉) false               | HTTP Request                 |                                                              |
| HTTP Request           | HTTP Request           | Sends alert messages to Discord webhook | Set - Golden Cross Msg, Set - Death Cross Msg, Set - No Signal Msg | —                            | 🔔 Discord Notifications overview                            |
| Sticky Note1           | Sticky Note            | Describes data collection phase         | —                           | —                            | 📈 Data Collection Phase overview                             |
| Sticky Note2           | Sticky Note            | Explains SMA calculation and crossover detection | —                           | —                            | 🧮 Technical Analysis Engine explanation                      |
| Sticky Note3           | Sticky Note            | Overview of Discord notifications       | —                           | —                            | 🔔 Discord Notifications overview                            |
| Sticky Note4           | Sticky Note            | Details crossover signal logic           | —                           | —                            | 🚦 Signal Detection Logic explanation                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to trigger weekly on Monday to Friday at 17:00 (5 PM).

2. **Create Set Node for Ticker List**  
   - Type: Set  
   - Add a field `symbol` as an array with values: ["NVDA","JPM","PG","SPY"] (expandable).  
   - Include notes explaining ticker rationale if desired.

3. **Create SplitOut Node**  
   - Type: SplitOut  
   - Set field to split on: `symbol`.

4. **Create HTTP Request Node to fetch Alpha Vantage Data**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: https://www.alphavantage.co/query  
   - Query parameters:  
     - function=TIME_SERIES_DAILY  
     - symbol={{$json["symbol"]}} (expression)  
     - apikey=YOURKEYHERE (replace with valid Alpha Vantage API key)  
     - outputsize=compact  
   - Set error handling to continue on fail.

5. **Create Code Node to extract yesterday's close**  
   - Type: Code (JavaScript)  
   - Use provided script to parse Alpha Vantage response:  
     - Validate time series data exists  
     - Extract the second most recent date and closing price  
     - Return JSON with fields `symbol`, `date`, `close`.

6. **Create PostgreSQL Node to Insert Data**  
   - Type: PostgreSQL  
   - Set connection credentials to your PostgreSQL database.  
   - Table: `historical_stocks` in schema `public`  
   - Columns: Map `Date` to `date`, `Close` to `close`, `symbol` to `symbol`.  
   - Enable "Skip on Conflict" to avoid duplicate inserts.

7. **Create PostgreSQL Node to Query Historical Data**  
   - Type: PostgreSQL  
   - Query:  
     ```sql
     WITH numbered AS (
       SELECT
         *,
         ROW_NUMBER() OVER (
           PARTITION BY symbol
           ORDER BY "Date" DESC
         ) AS rn
       FROM public.historical_stocks
       WHERE symbol IN ('NVDA','JPM','SPY','PG')
     )
     SELECT
       id,
       symbol,
       "Date",
       "Close"
     FROM numbered
     WHERE rn <= 121
     ORDER BY symbol, "Date" DESC;
     ```  
   - Execute once per workflow run.

8. **Create Code Node to compute SMAs**  
   - Type: Code (JavaScript)  
   - Calculate 60-day and 120-day SMAs for current and previous day per symbol as per provided code.  
   - Output fields: `symbol`, `sma60_current`, `sma120_current`, `sma60_previous`, `sma120_previous`.

9. **Create If Node for Golden Cross detection (If (📈))**  
   - Type: If  
   - Condition:  
     - Left: `{{$json.sma60_previous}}` <= `{{$json.sma120_previous}}` (number less or equal)  
     - AND  
     - Left: `{{$json.sma60_current}}` > `{{$json.sma120_current}}` (number greater than)  
   - True output: Set - Golden Cross Msg  
   - False output: If (📉)

10. **Create If Node for Death Cross detection (If (📉))**  
    - Type: If  
    - Condition:  
      - Left: `{{$json.sma60_previous}}` >= `{{$json.sma120_previous}}` (number greater or equal)  
      - AND  
      - Left: `{{$json.sma60_current}}` < `{{$json.sma120_current}}` (number less than)  
    - True output: Set - Death Cross Msg  
    - False output: Set - No Signal Msg

11. **Create Set Nodes for Alert Messages**  
    - **Set - Golden Cross Msg**  
      - Set field `content` to:  
        `🟢📈 Golden Cross Alert for **{{$json["symbol"]}}**! The 60-day SMA has crossed above the 120-day SMA.`  
    - **Set - Death Cross Msg**  
      - Set field `content` to:  
        `🔴📉 Death Cross Alert for **{{$json["symbol"]}}**! The 60-day SMA has crossed below the 120-day SMA.`  
    - **Set - No Signal Msg**  
      - Set field `content` to:  
        `🟡↔️ No crossover today for {{ $items().map(i=>i.json.symbol).sort().join(", ") }}. Monitoring continues…`  
      - Configure to execute once per workflow run (Execute Once option).

12. **Create HTTP Request Node to post to Discord**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Your Discord webhook URL (replace placeholder)  
    - Body Parameters: JSON with field `content` from incoming message  
    - Connect outputs of all three Set nodes to this HTTP Request node.

13. **Connect all nodes following this order:**  
    - Trigger → Set - Ticker List → Split - Tickers → Fetch Daily History → Getting today's data → Insert rows in a table → Execute a SQL query → Compute 60/120 SMAs → If (📈) → Set - Golden Cross Msg → HTTP Request  
    - If (📈) False → If (📉) → Set - Death Cross Msg → HTTP Request  
    - If (📉) False → Set - No Signal Msg → HTTP Request

14. **Add Sticky Notes for documentation** (optional but recommended):  
    - Overview, Data Collection, Technical Analysis, Signal Logic, Discord Notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow monitors Golden Cross and Death Cross signals based on 60-day and 120-day Simple Moving Averages, key technical indicators in stock trading. | Workflow overview sticky note.                                                                     |
| Alpha Vantage API has rate limits; ensure your key has sufficient capacity or consider implementing delay/retry logic. | API usage consideration.                                                                           |
| PostgreSQL table `historical_stocks` requires columns: id (primary key), symbol (string), Date (date), Close (float).  | Database schema requirement.                                                                       |
| Discord webhook URL must be correctly configured and active for alerts to post successfully.                           | Notification integration details.                                                                  |
| The workflow runs on weekdays at 5 PM to analyze closing prices after market close.                                     | Scheduling logic.                                                                                   |
| This workflow is designed for stocks: NVDA, JPM, PG, SPY but can be extended by modifying ticker list and SQL query.  | Extensibility note.                                                                                |
| Expression syntax used in Set nodes and If nodes uses n8n's expression language with JavaScript interpolation.          | n8n expression usage.                                                                              |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.