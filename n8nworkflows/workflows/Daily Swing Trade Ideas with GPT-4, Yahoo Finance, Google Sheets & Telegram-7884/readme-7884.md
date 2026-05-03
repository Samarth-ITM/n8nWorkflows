Daily Swing Trade Ideas with GPT-4, Yahoo Finance, Google Sheets & Telegram

https://n8nworkflows.xyz/workflows/daily-swing-trade-ideas-with-gpt-4--yahoo-finance--google-sheets---telegram-7884


# Daily Swing Trade Ideas with GPT-4, Yahoo Finance, Google Sheets & Telegram

### 1. Workflow Overview

This workflow, titled **Daily Swing Trade Ideas with GPT-4, Yahoo Finance, Google Sheets & Telegram**, is designed to generate daily swing trade recommendations for Indian equities, specifically targeting NSE 100 stocks. It activates shortly after market close (4:05 PM IST), fetches end-of-day (EOD) stock data from Yahoo Finance via RapidAPI, processes and formats this data, then feeds it into an AI model (OpenAI GPT-4) to generate trade ideas. The resulting trade signals are parsed, logged into a Google Sheet for tracking, and simultaneously sent as alerts via Telegram.

The logical blocks of the workflow are:

- **1.1 Input Reception & Scheduling**: Triggering the workflow daily after market close.
- **1.2 Stock List Preparation**: Preparing the list of stock tickers to analyze.
- **1.3 Data Retrieval & Formatting**: Fetching EOD stock quotes and formatting them.
- **1.4 Data Validation & Prompt Construction**: Filtering valid stock data and building the AI prompt input.
- **1.5 AI Trade Idea Generation**: Calling OpenAI GPT-4 to generate swing trade recommendations.
- **1.6 Output Parsing & Distribution**: Parsing AI output, logging trades to Google Sheets, and sending Telegram alerts.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Scheduling

- **Overview:**  
  Initiates the workflow at a fixed daily time (4:05 PM IST) on all days of the week, immediately after Indian stock market closing, to ensure fresh EOD data is available.

- **Nodes Involved:**  
  - `✅ Trigger - Daily Market Close`

- **Node Details:**

  - **Node Name:** ✅ Trigger - Daily Market Close  
    - **Type:** Schedule Trigger  
    - **Configuration:**  
      - Runs every day of the week (Sunday to Saturday)  
      - Triggers at 16:05 (4:05 PM) local time (assumed IST)  
    - **Inputs:** None (starting point)  
    - **Outputs:** Connected to "Prepare Stock List (NSE 100)"  
    - **Potential Failures:** Scheduler misconfiguration or timezone drift could cause missed or mistimed runs.  
    - **Notes:** Ensures the workflow runs post-market close for accurate EOD data.

#### 1.2 Stock List Preparation

- **Overview:**  
  Generates an array of NSE 100 stock tickers (symbols) to analyze. This is the input list for data fetching.

- **Nodes Involved:**  
  - `🔢 Prepare Stock List (NSE 100)`

- **Node Details:**

  - **Node Name:** 🔢 Prepare Stock List (NSE 100)  
    - **Type:** Code (JavaScript)  
    - **Configuration:**  
      - Hardcoded array of ticker symbols (e.g., "TCS.NS", "RELIANCE.NS", "INFY.NS", "HDFCBANK.NS")  
      - Maps each ticker into an object `{ json: { ticker } }`  
    - **Inputs:** Trigger node output  
    - **Outputs:** List of ticker JSON objects  
    - **Potential Failures:** Lack of dynamic update means the list requires manual editing to stay current. Empty or malformed array would result in no data fetched downstream.

#### 1.3 Data Retrieval & Formatting

- **Overview:**  
  Fetches end-of-day stock quote data for each ticker from Yahoo Finance via RapidAPI, then parses and formats this data into a usable structure for further analysis.

- **Nodes Involved:**  
  - `Fetch Quote via Rapid API`  
  - `🧮 Format EOD Data`  
  - `📊 Filter Valid Stock Data`

- **Node Details:**

  - **Node Name:** Fetch Quote via Rapid API  
    - **Type:** HTTP Request  
    - **Configuration:**  
      - URL constructed dynamically per ticker, calling Yahoo Finance API via RapidAPI  
      - Uses RapidAPI host and API key headers (requires valid API key)  
      - Sends GET requests with parameters to retrieve market data for the symbol  
      - Proxy config enabled (`useApifyProxy: true`)  
    - **Inputs:** List of ticker JSON objects  
    - **Outputs:** Raw API JSON response per ticker  
    - **Potential Failures:**  
      - API key missing or invalid results in auth errors.  
      - Rate limiting or network issues causing timeouts.  
      - API schema changes causing parsing failures downstream.

  - **Node Name:** 🧮 Format EOD Data  
    - **Type:** Code (JavaScript)  
    - **Configuration:**  
      - Parses the API response to extract relevant stock quote fields: symbol, shortName/longName, price, open, high, low, prevClose, volume, PE ratio, 50DMA, 200DMA.  
      - Builds an array of stock objects with these properties.  
      - Limits output to 40 stocks maximum for performance.  
    - **Inputs:** API responses from previous node  
    - **Outputs:** JSON object containing array `stocks`  
    - **Potential Failures:** Missing or malformed API data may cause `undefined` references or empty results.

  - **Node Name:** 📊 Filter Valid Stock Data  
    - **Type:** Code (JavaScript)  
    - **Configuration:**  
      - Validates and extracts the `stocks` array from input JSON (handles array or object wrapping).  
      - Throws error if `stocks` array is missing.  
    - **Inputs:** Formatted stock data object  
    - **Outputs:** Cleaned JSON with `stocks` array  
    - **Potential Failures:** Input format changes upstream may cause failure to find `stocks`.

#### 1.4 Data Validation & Prompt Construction

- **Overview:**  
  Converts the cleaned stock data into a text prompt structured for GPT-4. The prompt includes formatted stock details and instructions to identify the top 3 swing trades.

- **Nodes Involved:**  
  - `🗃️ Build LLM Prompt Input`

- **Node Details:**

  - **Node Name:** 🗃️ Build LLM Prompt Input  
    - **Type:** Code (JavaScript)  
    - **Configuration:**  
      - Maps each stock object to a formatted multi-line string including symbol, price, open, high, low, previous close, volume, PE ratio, 50-day and 200-day moving averages.  
      - Joins all formatted stocks with double newlines.  
      - Constructs a prompt string with system instructions to GPT-4 as a technical analyst, requesting exactly 3 swing trade ideas in JSON format with fields: symbol, direction, entry, target, stopLoss, timeFrame, reason.  
    - **Inputs:** Cleaned stocks array  
    - **Outputs:** JSON with a `prompt` property containing the full prompt string  
    - **Potential Failures:** Excessively large input may exceed prompt size limits; malformed stock data could lead to incomplete prompt text.

#### 1.5 AI Trade Idea Generation

- **Overview:**  
  Sends the constructed prompt to OpenAI GPT-4 to generate swing trade recommendations and receives the AI’s response.

- **Nodes Involved:**  
  - `Generate Swing Trade Ideas (OpenAI)`

- **Node Details:**

  - **Node Name:** Generate Swing Trade Ideas (OpenAI)  
    - **Type:** OpenAI (via Langchain node)  
    - **Configuration:**  
      - Model: `chatgpt-4o-latest` (GPT-4 variant)  
      - Messages: includes system role with trading assistant instructions and user content from the prompt property  
      - Requires OpenAI API credentials configured  
    - **Inputs:** Prompt JSON from previous node  
    - **Outputs:** AI response JSON with the trade ideas embedded in `message.content`  
    - **Potential Failures:**  
      - OpenAI API key issues or rate limits  
      - Model unavailable or network errors  
      - AI returning malformed JSON or unexpected formatting

#### 1.6 Output Parsing & Distribution

- **Overview:**  
  Parses the AI-generated JSON trade recommendations, splits them into individual trade objects, and for each trade: sends a Telegram alert and logs the trade to Google Sheets.

- **Nodes Involved:**  
  - `Split Trade Recommendations`  
  - `Send Trade Alert to Telegram`  
  - `Log Trade to Google Sheet`

- **Node Details:**

  - **Node Name:** Split Trade Recommendations  
    - **Type:** Code (JavaScript)  
    - **Configuration:**  
      - Extracts JSON content enclosed within markdown code block ```json ... ``` in AI response text  
      - Parses JSON string into an array of trade objects  
      - Returns each trade as an individual item for downstream processing  
    - **Inputs:** AI response from OpenAI node  
    - **Outputs:** Individual trade JSON objects  
    - **Potential Failures:**  
      - Parsing failure if AI output does not follow the expected ```json ... ``` format  
      - JSON syntax errors in AI output

  - **Node Name:** Send Trade Alert to Telegram  
    - **Type:** Telegram node  
    - **Configuration:**  
      - Sends formatted message to a predefined Telegram chat ID  
      - Message includes symbol, direction, entry, target, stop loss, timeframe, and reason, with Markdown formatting  
      - Telegram API credentials required  
    - **Inputs:** Individual trade JSONs from split node  
    - **Outputs:** None (final step)  
    - **Potential Failures:**  
      - Invalid Telegram chat ID  
      - API authentication failures  
      - Message formatting errors

  - **Node Name:** Log Trade to Google Sheet  
    - **Type:** Google Sheets node  
    - **Configuration:**  
      - Appends a new row per trade to a specific Google Sheet document and sheet (by document ID and sheet gid)  
      - Columns mapped with trade properties including date (localized to Asia/Kolkata), symbol, entry, target, stop loss, direction, timeframe, reason, and status "Open"  
      - Google OAuth2 credentials required  
    - **Inputs:** Individual trade JSONs from split node  
    - **Outputs:** None (final step)  
    - **Potential Failures:**  
      - Invalid or expired Google OAuth token  
      - Sheet or document ID changes or access revoked  
      - Column mapping errors or missing fields

---

### 3. Summary Table

| Node Name                        | Node Type             | Functional Role                         | Input Node(s)                    | Output Node(s)                                 | Sticky Note                                                                                                     |
|---------------------------------|-----------------------|---------------------------------------|---------------------------------|------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| ✅ Trigger - Daily Market Close  | Schedule Trigger      | Triggers workflow daily post-market   | None                            | 🔢 Prepare Stock List (NSE 100)                | │ ⏰ Trigger - Daily Market Close               │ Triggers the workflow post EOD (e.g., 4:15 PM IST)                                                           │
| 🔢 Prepare Stock List (NSE 100)  | Code                  | Creates list of NSE 100 stock symbols | ✅ Trigger - Daily Market Close  | Fetch Quote via Rapid API                       | │ 📦 Prepare Stock List (NSE 100)              │ Generates the list of NSE 100 stock symbols to analyze                                                      │
| Fetch Quote via Rapid API        | HTTP Request          | Fetches EOD stock data from Yahoo API | 🔢 Prepare Stock List (NSE 100)  | 🧮 Format EOD Data                             | │ 🌐 Fetch EOD Data (RapidAPI)                 │ Fetches end-of-day stock data from RapidAPI                                                               │
| 🧮 Format EOD Data              | Code                  | Parses and formats fetched EOD data   | Fetch Quote via Rapid API        | 📊 Filter Valid Stock Data                      | │ 🛠️ Format EOD Data                           │ Extracts and formats the quote data from API response                                                     │
| 📊 Filter Valid Stock Data       | Code                  | Validates and extracts usable stocks  | 🧮 Format EOD Data               | 🗃️ Build LLM Prompt Input                       | │ 🧹 Filter Valid Stock Data                   │ Filters/cleans/slices usable stock entries                                                                │
| 🗃️ Build LLM Prompt Input       | Code                  | Builds the AI prompt string with stock data | 📊 Filter Valid Stock Data       | Generate Swing Trade Ideas (OpenAI)             | │ 🧾 Build LLM Prompt Input                    │ Converts JSON stock data to string for OpenAI prompt                                                      │
| Generate Swing Trade Ideas (OpenAI) | OpenAI Langchain node | Calls GPT-4 to generate swing trade ideas | 🗃️ Build LLM Prompt Input       | Split Trade Recommendations                     | │ 🤖 Generate Swing Trade Ideas (OpenAI)       │ Uses OpenAI to generate top swing trade ideas based on input data                                         │
| Split Trade Recommendations      | Code                  | Parses AI output JSON and splits trades | Generate Swing Trade Ideas (OpenAI) | Send Trade Alert to Telegram, Log Trade to Google Sheet | │ 🍴 Split JSON Output (Trade-wise)            │ Splits the LLM output into individual trade objects                                                      │
| Send Trade Alert to Telegram     | Telegram               | Sends trade alerts to Telegram chat   | Split Trade Recommendations      | None                                           | │ 📲 Send Telegram Trade Alert                 │ Sends trade alerts to Telegram user in real-time                                                        │
| Log Trade to Google Sheet        | Google Sheets          | Logs trades into Google Sheet tracker | Split Trade Recommendations      | None                                           | │ 📊 Save Trade to Google Sheet                │ Appends each trade recommendation to Google Sheet tracker                                               │

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `✅ Trigger - Daily Market Close`  
   - Type: Schedule Trigger  
   - Configure to run every day of the week at 16:05 (4:05 PM IST)  

2. **Add a Code node**  
   - Name: `🔢 Prepare Stock List (NSE 100)`  
   - Type: Code  
   - Paste the following JavaScript:  
     ```javascript
     const tickers = [
       "TCS.NS", "RELIANCE.NS", "INFY.NS", "HDFCBANK.NS"
       // Extend this list up to 100 tickers as desired
     ];
     return tickers.map(ticker => ({ json: { ticker } }));
     ```  
   - Connect `✅ Trigger - Daily Market Close` → `🔢 Prepare Stock List (NSE 100)`

3. **Add an HTTP Request node**  
   - Name: `Fetch Quote via Rapid API`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Use expression:  
     ```
     https://apidojo-yahoo-finance-v1.p.rapidapi.com/market/v2/get-quotes?region=IN&symbols={{$json["ticker"]}}
     ```  
   - Headers:  
     - `x-rapidapi-host`: `apidojo-yahoo-finance-v1.p.rapidapi.com`  
     - `x-rapidapi-key`: Your valid RapidAPI key (must be configured in credentials or node)  
     - `Accept`: `application/json`  
   - Enable proxy config if needed (`useApifyProxy: true`)  
   - Connect `🔢 Prepare Stock List (NSE 100)` → `Fetch Quote via Rapid API`

4. **Add a Code node**  
   - Name: `🧮 Format EOD Data`  
   - Type: Code  
   - Paste JavaScript to parse API response and extract key fields (adapt from existing code):  
     ```javascript
     const output = [];
     for (const item of items) {
       const result = item.json.quoteResponse?.result?.[0];
       if (!result) continue;
       output.push({
         symbol: result.symbol,
         name: result.shortName || result.longName || result.symbol,
         price: result.regularMarketPrice,
         open: result.regularMarketOpen,
         high: result.regularMarketDayHigh,
         low: result.regularMarketDayLow,
         prevClose: result.regularMarketPreviousClose,
         volume: result.regularMarketVolume,
         pe: result.trailingPE,
         fiftyDayAvg: result.fiftyDayAverage,
         twoHundredDayAvg: result.twoHundredDayAverage
       });
     }
     return { json: { stocks: output.slice(0, 40) } };
     ```  
   - Connect `Fetch Quote via Rapid API` → `🧮 Format EOD Data`

5. **Add a Code node**  
   - Name: `📊 Filter Valid Stock Data`  
   - Type: Code  
   - Paste JavaScript to validate and extract the stocks array:  
     ```javascript
     const input = $json;
     let stocks = [];
     if (Array.isArray(input) && input[0].stocks) {
       stocks = input[0].stocks;
     } else if (input.stocks) {
       stocks = input.stocks;
     } else {
       throw new Error("No valid 'stocks' array found in input.");
     }
     return [{ json: { stocks } }];
     ```  
   - Connect `🧮 Format EOD Data` → `📊 Filter Valid Stock Data`

6. **Add a Code node**  
   - Name: `🗃️ Build LLM Prompt Input`  
   - Type: Code  
   - Paste JavaScript to format a prompt string for OpenAI GPT-4:  
     ```javascript
     const stocks = $json.stocks;
     let formattedStocks = stocks.map(stock => {
       return `Symbol: ${stock.symbol}
     Price: ₹${stock.price}
     Open: ₹${stock.open}
     High: ₹${stock.high}
     Low: ₹${stock.low}
     Prev Close: ₹${stock.prevClose}
     Volume: ${stock.volume}
     PE Ratio: ${stock.pe}
     50DMA: ₹${stock.fiftyDayAvg}
     200DMA: ₹${stock.twoHundredDayAvg}`;
     }).join("\n\n");

     return [{
       json: {
         prompt: `You are a technical analyst. Based only on the EOD data provided below, identify the top 3 swing trades from this list. Return the answer in JSON format with: symbol, direction (Buy/Sell), entry, target, stopLoss, timeFrame, and reason.\n\n${formattedStocks}`
       }
     }];
     ```  
   - Connect `📊 Filter Valid Stock Data` → `🗃️ Build LLM Prompt Input`

7. **Add an OpenAI node (Langchain variant)**  
   - Name: `Generate Swing Trade Ideas (OpenAI)`  
   - Type: OpenAI (Langchain)  
   - Model: `chatgpt-4o-latest`  
   - Messages:  
     - System role with content:  
       ```
       You are a trading assistant for Indian equities. Only use the data provided to generate potential profitable swing trade setups. Do not assume or invent any details.
       ```  
     - User role with content: use expression `{{$json.prompt}}`  
   - Credentials: Configure with valid OpenAI API key  
   - Connect `🗃️ Build LLM Prompt Input` → `Generate Swing Trade Ideas (OpenAI)`

8. **Add a Code node**  
   - Name: `Split Trade Recommendations`  
   - Type: Code  
   - Paste JavaScript to parse AI output and split trades:  
     ```javascript
     const messages = $json.message.content;
     let parsed;
     try {
       parsed = JSON.parse(messages.match(/```json([\s\S]*?)```/)[1].trim());
     } catch (e) {
       throw new Error("Failed to parse OpenAI JSON output. Make sure it is enclosed in ```json ... ```.");
     }
     return parsed.map(item => ({ json: item }));
     ```  
   - Connect `Generate Swing Trade Ideas (OpenAI)` → `Split Trade Recommendations`

9. **Add a Telegram node**  
   - Name: `Send Trade Alert to Telegram`  
   - Type: Telegram  
   - Chat ID: Set your Telegram chat ID  
   - Message text (Markdown enabled):  
     ```
     📊 *{{$json.symbol}}* | {{$json.direction}}
     💸 Entry: ₹{{$json.entry}} | 🎯 Target: ₹{{$json.target}} | 🛑 SL: ₹{{$json.stopLoss}}
     ⏳ {{$json.timeFrame}}
     🧠 {{$json.reason}}
     ```  
   - Credentials: Configure Telegram API credentials  
   - Connect `Split Trade Recommendations` → `Send Trade Alert to Telegram`

10. **Add a Google Sheets node**  
    - Name: `Log Trade to Google Sheet`  
    - Type: Google Sheets  
    - Operation: Append  
    - Document ID: Your Google Sheet document ID  
    - Sheet Name/ID: Use the correct sheet gid or name ("gid=0" or "Sheet1")  
    - Columns: Map fields from trade JSON:  
      - Date: `={{ new Date().toLocaleString("en-IN", { timeZone: "Asia/Kolkata" }) }}`  
      - Symbol, Entry, Reason, Status (set "Open"), Target, Direction, Stop Loss, Time Frame mapped accordingly  
    - Credentials: Configure Google Sheets OAuth2 credentials  
    - Connect `Split Trade Recommendations` → `Log Trade to Google Sheet`

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow triggers post-market close to ensure fresh end-of-day data before generating swing trade signals.                    | Workflow scheduling detail                                                                          |
| RapidAPI Yahoo Finance API requires a valid API key and proper quota management for reliable data fetch.                           | https://rapidapi.com/apidojo/api/yahoo-finance1                                                    |
| OpenAI GPT-4 model usage requires API key and careful prompt design to ensure structured JSON output (enclosed in ```json ... ```). | https://platform.openai.com/docs/models/gpt-4                                                     |
| Google Sheets node requires OAuth2 credentials with write access to the target spreadsheet, ensure document access permissions.   | https://developers.google.com/sheets/api/guides/authorizing                                         |
| Telegram alerts require bot token and chat ID; ensure bot is added to the chat and permissions are properly set.                   | https://core.telegram.org/bots/api                                                                 |
| The included sticky note summarizes the purpose of each node for quick reference and documentation clarity.                        | Internal documentation node within workflow                                                        |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated n8n workflow designed for legal and public data handling. It complies with all applicable content policies and contains no illegal, offensive, or protected material.