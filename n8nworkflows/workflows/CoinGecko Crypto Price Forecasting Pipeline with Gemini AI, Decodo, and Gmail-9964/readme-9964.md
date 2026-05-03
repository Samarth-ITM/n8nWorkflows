CoinGecko Crypto Price Forecasting Pipeline with Gemini AI, Decodo, and Gmail

https://n8nworkflows.xyz/workflows/coingecko-crypto-price-forecasting-pipeline-with-gemini-ai--decodo--and-gmail-9964


# CoinGecko Crypto Price Forecasting Pipeline with Gemini AI, Decodo, and Gmail

### 1. Workflow Overview

This n8n workflow automates the process of scraping cryptocurrency market data from CoinGecko, processing and storing the data, generating short-term price forecasts using AI, and sending formatted email reports. It is designed for crypto enthusiasts or analysts who want automated, periodic snapshots and forecasts without managing a complex data stack.

The workflow contains two main logical blocks:

- **1.1 Data Acquisition and Logging (Every 30 minutes):**  
  Scrapes live crypto data for selected coins using Decodo, parses HTML content with a Gemini-powered AI agent to extract clean numeric metrics, and stores the results in an n8n Data Table for historical tracking.

- **1.2 Forecasting and Email Reporting (Daily at 18:00):**  
  Loads the last 48 hours of stored data, cleans and down-samples it, then feeds it into a Gemini AI model to forecast directional price movements for the next 24 hours. The forecast results are formatted into an HTML email and sent to a configured recipient with timezone-aware timestamps.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Acquisition and Logging (Every 30 minutes)

- **Overview:**  
  This block runs every 30 minutes, loops through a configurable list of coins, scrapes their CoinGecko pages via Decodo, extracts key market metrics using an AI agent, converts the output to JSON, and logs the data into a Data Table.

- **Nodes Involved:**  
  - Schedule: Every 30m (Scrape & Log)  
  - Configure Coins  
  - Build Coin Items  
  - For Each Coin (SplitInBatches)  
  - Scrape CoinGecko (Decodo)  
  - Parse Coin Metrics (LLM)  
  - Gemini for Parsing (AI Language Model)  
  - Convert Output to JSON  
  - Insert Data (Data Table)  
  - Sticky Notes 2, 5, 7 (Documentation/Instructions)

- **Node Details:**

  - **Schedule: Every 30m (Scrape & Log)**  
    - Type: Schedule Trigger  
    - Role: Triggers the data acquisition loop every 30 minutes using server timezone.  
    - Key params: Interval of 30 minutes.  
    - Outputs to: Configure Coins.  
    - Edge cases: Scheduler misconfiguration or server timezone mismatch may cause timing issues.

  - **Configure Coins**  
    - Type: Set  
    - Role: Defines the list of coins to scrape (default: bitcoin, ethereum, solana).  
    - Config: Assigns static string variables for each coin.  
    - Outputs to: Build Coin Items.  
    - Edge cases: Users must edit coin slugs to valid CoinGecko identifiers.  

  - **Build Coin Items**  
    - Type: Code  
    - Role: Converts the configured coin strings into individual item objects for processing.  
    - Logic: Iterates over keys containing "coin", emits an item per coin.  
    - Outputs to: For Each Coin.  
    - Edge cases: Empty or invalid coin entries result in skipped items.

  - **For Each Coin**  
    - Type: SplitInBatches  
    - Role: Processes each coin item sequentially, controlling flow for scraping and data insertion.  
    - Outputs:  
      - Main output 1: Insert Data (writes parsed data).  
      - Main output 2: Scrape CoinGecko.  
    - Edge cases: Batch size defaults; large coin lists may slow process.

  - **Scrape CoinGecko (Decodo)**  
    - Type: Decodo node (community)  
    - Role: Fetches the HTML content from CoinGecko coin page via Decodo API.  
    - Config: Target URL pattern `https://www.coingecko.com/en/coins/{{ $json.coin }}` with geo set to United States.  
    - Credentials: Decodo API required.  
    - Outputs to: Parse Coin Metrics (LLM).  
    - Edge cases: Network failures, invalid coin slugs, or Decodo API limits.

  - **Parse Coin Metrics (LLM)**  
    - Type: LangChain Agent (LLM)  
    - Role: Extracts structured market data from the raw HTML using Gemini AI, following strict instructions to detect price changes including signs and convert units.  
    - Key inputs: HTML content from Decodo, coin name, strict JSON output format.  
    - Outputs to: Convert Output to JSON.  
    - Edge cases: Parsing errors, inconsistent HTML structure, or AI interpretation failures.

  - **Gemini for Parsing**  
    - Type: LangChain Gemini LLM  
    - Role: Backend AI model called by Parse Coin Metrics node to process HTML content.  
    - Credentials: Google Palm API (Gemini) required.  
    - Outputs to: Parse Coin Metrics (LLM) node as AI language model response.  
    - Edge cases: API quota, authentication errors.

  - **Convert Output to JSON**  
    - Type: Code  
    - Role: Cleans and parses the AI output string into a valid JSON object, removing markdown code fences if present.  
    - Logic: Trims, strips ```json marks, parses JSON, throws error if invalid.  
    - Outputs to: For Each Coin (to Insert Data).  
    - Edge cases: Malformed AI output causing JSON parse errors.

  - **Insert Data**  
    - Type: Data Table  
    - Role: Writes extracted data into the n8n Data Table with defined schema (coin, ts, price, change_1h, change_24h, change_7d, market_cap, volume_24h).  
    - Config: Mapping fields explicitly, no type conversion.  
    - Outputs: None (end of this branch).  
    - Edge cases: DataTable access issues, schema mismatch.

  - **Sticky Notes 2,5,7**  
    - Provide instructions about data schema, coin slug editing, and overview of this scraping and logging block.

---

#### 2.2 Forecasting and Email Reporting (Daily at 18:00)

- **Overview:**  
  This block triggers once daily at 18:00, loads the last 48 hours of stored crypto data, processes and downsamples it for each coin, invokes Gemini AI to forecast price direction windows for the next 24 hours, constructs an HTML email with localized timestamps, and sends it via Gmail.

- **Nodes Involved:**  
  - Schedule: 18:00 Daily (Forecast Email)  
  - Recipient Email + Timezone  
  - Load Data Last 48h (Data Table)  
  - Make it JSON (Code)  
  - Gemini for Forecast (LLM)  
  - Forecast Next 24h (LLM Agent)  
  - Structured Output Parser  
  - Build Email (HTML + Text) (Code)  
  - Send Email (Gmail)  
  - Sticky Notes 3,6 (Documentation)

- **Node Details:**

  - **Schedule: 18:00 Daily (Forecast Email)**  
    - Type: Schedule Trigger  
    - Role: Triggers forecast generation and email sending once daily at 18:00 server time.  
    - Outputs to: Recipient Email + Timezone.  
    - Edge cases: Server timezone mismatch affects timing.

  - **Recipient Email + Timezone**  
    - Type: Set  
    - Role: Sets recipient email address and timezone string (e.g., "GMT+0") for email scheduling and formatting.  
    - Outputs to: Load Data Last 48h.  
    - Edge cases: Invalid email or timezone formats may affect email delivery or timestamp display.

  - **Load Data Last 48h**  
    - Type: Data Table (Get operation)  
    - Role: Retrieves all stored coin price entries from the last 48 hours from the Data Table.  
    - Outputs to: Make it JSON.  
    - Edge cases: Empty or incomplete data, Data Table access errors.

  - **Make it JSON**  
    - Type: Code  
    - Role: Cleans and reformats raw data: filters last 48h only, groups by coin, sorts by timestamp, downsamples to max 120 points in 30-minute buckets, and builds a structured JSON object per coin with latest data and series.  
    - Outputs to: Forecast Next 24h (LLM).  
    - Edge cases: Missing or malformed timestamps, empty data sets.

  - **Gemini for Forecast**  
    - Type: LangChain Gemini LLM  
    - Role: AI model backend for the Forecast Next 24h agent.  
    - Credentials: Google Palm API (Gemini).  
    - Outputs to: Forecast Next 24h (LLM) agent for output parsing.  
    - Edge cases: API errors, quota limits.

  - **Forecast Next 24h (LLM)**  
    - Type: LangChain Agent  
    - Role: Receives cleaned data per coin and generates JSON forecasts of price direction for 6h, 12h, and 24h windows, including best up/down time windows and confidence level.  
    - Input: Latest snapshot, 48h series.  
    - Output: Strict JSON forecast.  
    - Outputs to: Build Email (HTML + Text).  
    - Edge cases: Forecast uncertainty, mixed signals, AI parsing errors.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser  
    - Role: Ensures forecast JSON output strictly conforms to the expected schema before passing to next node.  
    - Outputs to: Forecast Next 24h (LLM).  
    - Edge cases: Invalid or partial AI output.

  - **Build Email (HTML + Text)**  
    - Type: Code  
    - Role: Formats forecast data into an HTML email body with per-coin forecast cards, localized to recipient timezone, includes price, forecast direction, confidence, and favorable windows.  
    - Timezone parsing is robust, supporting offsets with minutes.  
    - Outputs to: Send Email (Gmail).  
    - Edge cases: Missing timezone info, malformed forecast data.

  - **Send Email (Gmail)**  
    - Type: Gmail node with OAuth2  
    - Role: Sends the formatted email to configured recipient.  
    - Subject includes localized GMT offset.  
    - Credentials: Gmail OAuth2 required.  
    - Edge cases: Email sending failures, OAuth token expiration.

  - **Sticky Notes 3,6**  
    - Explain forecast automation workflow and setup instructions including credential setup and usage.

---

### 3. Summary Table

| Node Name                     | Node Type                     | Functional Role                          | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                                                           |
|-------------------------------|-------------------------------|----------------------------------------|----------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------|
| Schedule: Every 30m (Scrape & Log) | Schedule Trigger              | Trigger scraping loop every 30 minutes |                                  | Configure Coins                  | Ensure your instance timezone matches your expectations; schedule uses server time.                  |
| Configure Coins               | Set                           | Defines coins to process                | Schedule: Every 30m              | Build Coin Items                 | Edit coin values (e.g., bitcoin, ethereum, solana). Use CoinGecko slugs. https://www.coingecko.com/ |
| Build Coin Items              | Code                          | Converts config into coin list items   | Configure Coins                 | For Each Coin                   |                                                                                                     |
| For Each Coin                 | SplitInBatches                | Processes each coin sequentially       | Build Coin Items                | Insert Data, Scrape CoinGecko   |                                                                                                     |
| Scrape CoinGecko (Decodo)     | Decodo API Node               | Fetches CoinGecko HTML content         | For Each Coin                   | Parse Coin Metrics (LLM)         |                                                                                                     |
| Parse Coin Metrics (LLM)       | LangChain Agent (Gemini AI)   | Parses HTML to extract coin metrics    | Scrape CoinGecko               | Convert Output to JSON          |                                                                                                     |
| Gemini for Parsing            | LangChain Gemini LLM          | AI model for parsing HTML               | Parse Coin Metrics (LLM)        | Parse Coin Metrics (LLM)         |                                                                                                     |
| Convert Output to JSON        | Code                          | Cleans and parses AI output to JSON    | Parse Coin Metrics (LLM)        | For Each Coin                   | Map fields to your Data Table schema. Keep numbers as pure numeric types.                            |
| Insert Data                  | Data Table                    | Stores parsed coin data                 | For Each Coin                   |                                  | Map fields to your Data Table schema. Keep numbers as pure numeric types.                            |
| Schedule: 18:00 Daily (Forecast Email) | Schedule Trigger              | Triggers daily forecast and email       |                                  | Recipient Email + Timezone      |                                                                                                     |
| Recipient Email + Timezone    | Set                           | Sets recipient email and timezone      | Schedule: 18:00 Daily           | Load Data Last 48h             |                                                                                                     |
| Load Data Last 48h           | Data Table                    | Loads last 48h of coin data             | Recipient Email + Timezone      | Make it JSON                   |                                                                                                     |
| Make it JSON                 | Code                          | Cleans, filters, groups, downsamples data | Load Data Last 48h             | Forecast Next 24h (LLM)         |                                                                                                     |
| Gemini for Forecast          | LangChain Gemini LLM          | AI model for forecasting                | Forecast Next 24h (LLM)         | Forecast Next 24h (LLM)         |                                                                                                     |
| Forecast Next 24h (LLM)       | LangChain Agent (Gemini AI)   | Generates price direction forecast JSON | Make it JSON                   | Build Email (HTML + Text)       |                                                                                                     |
| Structured Output Parser     | LangChain Output Parser       | Validates forecast JSON structure      | Forecast Next 24h (LLM)         | Forecast Next 24h (LLM)         |                                                                                                     |
| Build Email (HTML + Text)     | Code                          | Builds formatted HTML email body       | Forecast Next 24h (LLM)         | Send Email (Gmail)              |                                                                                                     |
| Send Email (Gmail)           | Gmail OAuth2                  | Sends forecast email to recipient      | Build Email (HTML + Text)       |                                  |                                                                                                     |
| Sticky Note2                 | Sticky Note                   | Notes on Data Table schema              |                                  |                                  | Map fields to your Data Table schema. Keep numbers as pure numeric types.                            |
| Sticky Note3                 | Sticky Note                   | Forecast Email Automation overview      |                                  |                                  | Forecast email automation details with timezone handling and confidence levels.                      |
| Sticky Note4                 | Sticky Note                   | Scheduler timezone reminder              |                                  |                                  | Ensure your instance timezone matches your expectations; schedule uses server time.                  |
| Sticky Note5                 | Sticky Note                   | Scraper & Logger overview                |                                  |                                  | Scraper flow description including AI parsing and Data Table logging.                               |
| Sticky Note6                 | Sticky Note                   | Automated Crypto Forecast Pipeline intro |                                  |                                  | Setup instructions and credits for Decodo, Gemini, Gmail credentials.                               |
| Sticky Note7                 | Sticky Note                   | Coin configuration note                   |                                  |                                  | Edit coin values (e.g., bitcoin, ethereum, solana). Use CoinGecko slugs.                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule: Every 30m (Scrape & Log)" node:**  
   - Type: Schedule Trigger  
   - Parameters: Interval trigger every 30 minutes (server time).  

2. **Create "Configure Coins" node:**  
   - Type: Set  
   - Assign three string variables: "coin 1" = "bitcoin", "coin 2" = "ethereum", "coin 3" = "solana".  
   - Connect output from Schedule node.  

3. **Create "Build Coin Items" node:**  
   - Type: Code  
   - Code to iterate over input JSON, extract keys containing "coin", output an item per coin with `{ coin: val }`.  
   - Connect output from Configure Coins.  

4. **Create "For Each Coin" node:**  
   - Type: SplitInBatches (default batch size)  
   - Connect output from Build Coin Items node.  

5. **Create "Scrape CoinGecko (Decodo)" node:**  
   - Type: Decodo community node  
   - Configure URL: `https://www.coingecko.com/en/coins/{{ $json.coin }}`  
   - Set geo to "United States"  
   - Credentials: Decodo API credentials required.  
   - Connect second output of For Each Coin to this node.  

6. **Create "Parse Coin Metrics (LLM)" node:**  
   - Type: LangChain Agent (Gemini)  
   - Set prompt with detailed instructions to extract coin metrics from HTML content (see above for instructions).  
   - Configure system message for AI context.  
   - Connect output from Decodo node.  

7. **Create "Gemini for Parsing" node:**  
   - Type: LangChain Gemini LLM  
   - Set credentials: Google Palm API (Gemini).  
   - Connect as AI model for "Parse Coin Metrics (LLM)" node.  

8. **Create "Convert Output to JSON" node:**  
   - Type: Code  
   - JavaScript code to trim output, remove code fences, parse JSON, and throw error if invalid.  
   - Connect output from Parse Coin Metrics (LLM).  

9. **Create "Insert Data" node:**  
   - Type: Data Table  
   - Configure Data Table ID for "Coin Prices" table.  
   - Map fields explicitly: coin (string), ts (string), price (number), change_1h (number), change_24h (number), change_7d (number), market_cap (number), volume_24h (number).  
   - Connect output from Convert Output to JSON to first output of For Each Coin node.  

10. **Create "Schedule: 18:00 Daily (Forecast Email)" node:**  
    - Type: Schedule Trigger  
    - Trigger at 18:00 daily (server time).  

11. **Create "Recipient Email + Timezone" node:**  
    - Type: Set  
    - Assign variables: recipientEmail (string, e.g., your email), timezone (string, e.g., "GMT+0").  
    - Connect output from Schedule 18:00 node.  

12. **Create "Load Data Last 48h" node:**  
    - Type: Data Table (Get operation)  
    - Data Table ID same as "Insert Data" node.  
    - Connect output from Recipient Email + Timezone.  

13. **Create "Make it JSON" node:**  
    - Type: Code  
    - Code to filter last 48h, group by coin, sort, downsample to 30m buckets max 120 points, and output structured JSON per coin with latest snapshot and series array.  
    - Connect output from Load Data Last 48h.  

14. **Create "Gemini for Forecast" node:**  
    - Type: LangChain Gemini LLM  
    - Credentials: Google Palm API (Gemini).  

15. **Create "Forecast Next 24h (LLM)" node:**  
    - Type: LangChain Agent (Gemini)  
    - Prompt to produce strict JSON forecast for next 6h, 12h, 24h price direction, best up/down windows, confidence.  
    - Connect output from Make it JSON node.  
    - Use Gemini for Forecast as AI model.  

16. **Create "Structured Output Parser" node:**  
    - Type: LangChain Output Parser  
    - Configure with example JSON schema matching forecast output.  
    - Connect AI output from Gemini for Forecast to Forecast Next 24h (LLM) to ensure valid JSON.  

17. **Create "Build Email (HTML + Text)" node:**  
    - Type: Code  
    - JavaScript to build a styled HTML email with per-coin forecast cards, localized to recipient timezone.  
    - Support various timezone formats for GMT offsets.  
    - Connect output from Forecast Next 24h (LLM).  

18. **Create "Send Email (Gmail)" node:**  
    - Type: Gmail (OAuth2)  
    - Set recipient email from input JSON or static.  
    - Use HTML body from previous node’s output.  
    - Subject includes localized GMT offset.  
    - Connect output from Build Email.  
    - Configure Gmail OAuth2 credentials.  

19. **Add Sticky Notes as documentation:**  
    - Add notes about Data Table schema, coin configuration, scheduling timezone, workflow overview, and setup instructions referencing Decodo, Gemini, Gmail credentials, and CoinGecko URLs.  

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Sign Up for Decodo [HERE](https://visit.decodo.com/discount) for discount                                             | Decodo API credential setup                                                                       |
| CoinGecko URLs use coin slugs like "bitcoin", "ethereum", "solana"                                                    | https://www.coingecko.com/                                                                       |
| Ensure your instance timezone matches your expectations; schedule triggers use server time                            | Important for correct timing of scraping and daily email                                         |
| Forecast Email Automation includes confidence levels, directional trends, and favorable trading windows              | Provides context for forecast interpretation                                                     |
| This workflow is for educational purposes only and is not financial advice                                           | Disclaimer included in email footer                                                             |
| Community node **Decodo** (@decodo/n8n-nodes-decodo) is required for scraping                                        | Must be installed for Decodo scraping node                                                      |
| Google Palm API (Gemini) credentials required for AI parsing and forecasting                                         | Must be set up with quota and authentication                                                    |
| Gmail OAuth2 credentials required for sending emails                                                                | OAuth2 setup needed for Gmail node                                                              |

---

**Disclaimer:** The provided text is derived exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.