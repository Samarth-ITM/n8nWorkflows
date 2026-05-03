Generate AI Stock Reports w/ Fundamental, Technical, & News Analysis (Free APIs)

https://n8nworkflows.xyz/workflows/generate-ai-stock-reports-w--fundamental--technical----news-analysis--free-apis--11772


# Generate AI Stock Reports w/ Fundamental, Technical, & News Analysis (Free APIs)

### 1. Workflow Overview

This workflow automates the generation of comprehensive stock analysis reports by combining fundamental financial data, technical indicators, and news sentiment analysis. It targets financial analysts, investors, and automated trading systems that require structured, data-driven stock insights and actionable recommendations. The workflow supports both scheduled batch processing of multiple tickers and manual on-demand analysis of a single stock ticker.

The workflow is logically segmented into the following blocks:

- **1.1 Trigger & Input Reception:** Initiates workflow execution either on a schedule for a predefined list of stocks or via manual user input through a form.
- **1.2 Stock Validation & Symbol Setup:** Checks stock existence on NASDAQ or NYSE and assigns appropriate exchange context.
- **1.3 Technical Data Collection & Analysis:** Gathers historical price data, technical indicators (MACD, Bollinger Bands, Fibonacci levels, support/resistance), and visual chart analysis using AI.
- **1.4 Fundamental Financial Analysis:** Collects financial statements and key metrics from Alpha Vantage APIs and synthesizes a financial health summary.
- **1.5 News Sentiment & Trends Analysis:** Retrieves recent news articles, performs sentiment scoring and topic extraction.
- **1.6 AI Orchestration & Report Generation:** A central AI agent calls the above sub-workflows as tools, synthesizes raw data into a professional report with final recommendations.
- **1.7 Report Formatting & Delivery:** Generates an HTML email report, adjusts colors based on sentiment, and sends the report to a configured email address.
- **1.8 Optional Automated Trading:** Executes buy orders via Alpaca API if the AI recommendation is "Buy".

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Reception

- **Overview:** Starts the workflow either on a weekly schedule for a list of stocks or via a manual form submission where a user inputs a single stock ticker.
- **Nodes Involved:** `Schedule Trigger`, `Stock List`, `Split Out`, `Loop Over Items`, `On form submission`
- **Node Details:**
  - **Schedule Trigger:** Triggers weekly on Monday at 06:20 AM.
  - **Stock List:** Provides an array of stock tickers to analyze on schedule.
  - **Split Out:** Splits the stock list array into individual ticker items.
  - **Loop Over Items:** Processes each ticker individually.
  - **On form submission:** Webhook-triggered node for manual user input with a required ticker field.
- **Edge Cases:**  
  - Manual submission must include a valid ticker, otherwise no downstream processing.
  - Scheduled batch may process empty or invalid ticker list if misconfigured.

#### 1.2 Stock Validation & Symbol Setup

- **Overview:** Validates that the ticker exists on NASDAQ or NYSE exchanges and sets the appropriate exchange and chart prefix accordingly.
- **Nodes Involved:** `Set Stock Symbol and API Key`, `Test NASDAQ`, `is on nasdaq?`, `Test NYSE`, `is on NYSE?`, `Set exchange to NASDAQ`, `Set exchange to NYSE`, `Stop and Error`, `Merge1`
- **Node Details:**
  - **Set Stock Symbol and API Key:** Sets ticker and TwelveData API key (requires user to provide API key).
  - **Test NASDAQ / Test NYSE:** HTTP request nodes query TwelveData price endpoint to test ticker validity on each exchange.
  - **If nodes (`is on nasdaq?`, `is on NYSE?`):** Check if price response is non-empty to confirm listing.
  - **Set exchange nodes:** Assign exchange code and chart prefix for subsequent API calls.
  - **Stop and Error:** Halts workflow with error if ticker not found on either exchange.
  - **Merge1:** Combines outputs from exchange checks for downstream use.
- **Edge Cases:**  
  - API key misconfiguration or invalid ticker leads to error or stop.
  - Ticker existing on both exchanges is resolved by NASDAQ check first.
  
#### 1.3 Technical Data Collection & Analysis

- **Overview:** Collects technical indicators and price history, calculates support/resistance, and generates a weekly candlestick chart image for AI visual analysis.
- **Nodes Involved:** `Get Price History`, `Calculate Support Resistance`, `Get Bollinger Bands`, `Get MACD`, `Merge`, `Organizing Data`, `Get Chart URL`, `First Technical Analysis`, `Set Variable`, `Warp as JSON for GPT`, `ChatGPT 4o`, `Set Final Response`
- **Node Details:**
  - **Get Price History:** Retrieves 180 days of daily price data from TwelveData for the ticker and exchange.
  - **Calculate Support Resistance:** Custom JavaScript node calculating Fibonacci levels and local support/resistance based on price history.
  - **Get Bollinger Bands:** Fetches latest Bollinger Bands data point.
  - **Get MACD:** Fetches latest MACD indicator data point.
  - **Merge:** Combines all above technical data streams.
  - **Organizing Data:** Formats and synthesizes technical data; identifies bullish/bearish factors (e.g., MACD cross, proximity to Fibonacci levels).
  - **Get Chart URL:** Requests a weekly candlestick chart image with volume bars, EMA line, RSI from Chart-Img API.
  - **First Technical Analysis:** Uses a vision-capable AI model to analyze the chart image and produce a structured JSON describing trends, RSI, candlestick patterns, volume notes, and price zones.
  - **Set Variable:** Stores AI visual chart analysis output.
  - **Warp as JSON for GPT & ChatGPT 4o:** Wraps combined technical JSON as formatted text and sends to an OpenAI or Anthropic model to synthesize a detailed technical analysis report with specific sections (Quick Stats, Candles/EMA, RSI, Indicator Synthesis, Takeaway).
  - **Set Final Response:** Saves final AI-generated textual analysis and base64-encoded chart image.
- **Version Requirements:**  
  - Chart analysis requires AI model capable of image input.
  - TwelveData and Chart-Img API keys required.
- **Edge Cases:**
  - API failures or limits can cause missing technical data.
  - Chart image generation failure impacts visual analysis.
  - AI model response parse failures handled in downstream nodes.
  
#### 1.4 Fundamental Financial Analysis

- **Overview:** Retrieves and synthesizes key financial data from Alpha Vantage, including company overview, income statement, and balance sheet, producing a structured financial summary.
- **Nodes Involved:** `Set Variables`, `Get Company Overview`, `Wait1`, `Get Income Statement`, `Wait2`, `Get Balance Sheet`, `Merge2`, `ChatGPT 4o1`, `Code in JavaScript`
- **Node Details:**
  - **Set Variables:** Sets ticker and Alpha Vantage API key.
  - **Get Company Overview:** HTTP request for company description and valuation metrics.
  - **Wait1 and Wait2:** Delays to respect Alpha Vantage API rate limits between calls.
  - **Get Income Statement:** Retrieves recent annual income statements.
  - **Get Balance Sheet:** Retrieves recent annual balance sheets.
  - **Merge2:** Combines the three financial data responses.
  - **ChatGPT 4o1:** An AI model synthesizes raw financial data into a structured JSON including company overview, key metrics (market cap, P/E ratio, EPS), recent revenue and net income trends, and debt-to-equity ratio, plus a concise financial summary paragraph.
  - **Code in JavaScript:** Parses AI JSON string output into usable JSON format.
- **Edge Cases:**
  - API rate limits or invalid keys cause incomplete financial data.
  - AI parsing errors if response format deviates.
  - Financial data missing or outdated for ticker.

#### 1.5 News Sentiment & Trends Analysis

- **Overview:** Fetches recent news articles and sentiment data from Alpha Vantage, analyzes sentiment distribution, identifies top articles and trending topics.
- **Nodes Involved:** `Generate Variables For API`, `Set Variables1`, `Get News Data`, `Analyse API Input`
- **Node Details:**
  - **Generate Variables For API:** Generates yesterday’s date in required format for news API query.
  - **Set Variables1:** Sets wanted date, stock symbol, and Alpha Vantage API key.
  - **Get News Data:** Retrieves news sentiment data for stock symbol filtered by relevance and date.
  - **Analyse API Input:** Custom JavaScript node processes news feed:
    - Filters articles relevant to the stock.
    - Counts sentiment categories (Bullish, Neutral, Bearish).
    - Calculates weighted average sentiment score.
    - Extracts top 5 influential articles by impact score.
    - Aggregates trending news topics with article counts and average relevance.
    - Produces overall sentiment classification (Very Bullish → Very Bearish).
- **Edge Cases:**
  - No news articles available for ticker or API failure.
  - Date formatting or API key misconfigurations.
  - News articles missing ticker sentiment data.

#### 1.6 AI Orchestration & Report Generation

- **Overview:** The central AI Agent node orchestrates the entire analysis by calling the three tool sub-workflows (technical, trends/news, fundamental), synthesizes their outputs into a structured report including technical analysis paragraphs, recommendation, sentiment counts, and JSON-structured data.
- **Nodes Involved:** `When Executed by Another Workflow`, `AI Agent`, `Window Buffer Memory`, `Technical Analysis Tool`, `Trends Analysis Tool`, `Fundamental Analysis Tool`, `Structured Output Parser`, `Buy?`
- **Node Details:**
  - **When Executed by Another Workflow:** Entry point node to accept passthrough input from batch or manual triggers.
  - **Window Buffer Memory:** Stores session context keyed by a static session key (335458847), enabling stateful interactions with AI agent.
  - **AI Agent:** Core LangChain agent configured as expert financial analyst orchestrator:
    - Receives ticker input.
    - Calls sub-workflows as tools using provided syntax for technical_analysis, trends_analysis, and fundamental_analysis.
    - Synthesizes all data and writes technical analysis paragraph and final recommendation text with JSON output adhering strictly to schema.
  - **Structured Output Parser:** Validates and parses AI agent output into structured JSON following strict schema.
  - **Buy?:** Conditional node checking if recommendation title contains "Buy" to trigger optional automated trading.
- **Edge Cases:**
  - Tool sub-workflow failures or missing data impact AI synthesis.
  - Session memory issues may cause state inconsistencies.
  - AI output parsing errors catch malformed JSON.

#### 1.7 Report Formatting & Delivery

- **Overview:** Formats the AI agent's structured JSON output into a styled HTML email report, adjusts color theming based on sentiment, removes irrelevant or empty content, and sends the final report via email.
- **Nodes Involved:** `Generate HTML`, `Adjust HTML Colors`, `Send Stock Analysis`
- **Node Details:**
  - **Generate HTML:** Uses a static HTML template with embedded Handlebars-style expressions referencing the AI agent’s output JSON fields (e.g., stockSymbol, recommendationText, technicalAnalysis, fundamentalAnalysis, topArticles, hotTopics).
  - **Adjust HTML Colors:** JavaScript code node that:
    - Dynamically changes colors of titles, bars, sentiment tags, and buttons based on sentiment classification (positive, neutral, negative).
    - Removes topics with only one article to reduce noise.
    - Removes undefined or placeholder articles.
    - Includes debug logs for sentiment detection.
  - **Send Stock Analysis:** Sends the final HTML report via SMTP email using user-configured credentials and addresses.
- **Edge Cases:**
  - Missing email credentials or invalid addresses cause send failure.
  - Incomplete or malformed AI output may cause rendering issues.
  - Large reports may affect email client display.

#### 1.8 Optional Automated Trading

- **Overview:** Places a market buy order on Alpaca’s paper trading API if the AI recommendation is "Buy".
- **Nodes Involved:** `Buy?`, `Buy in Alpaca`
- **Node Details:**
  - **Buy?:** Conditional node that checks if the recommendation title contains "Buy".
  - **Buy in Alpaca:** Sends HTTP POST request to Alpaca API to place a market buy order of $7,000 notional value for the stock symbol.
  - Requires Alpaca API key and secret credentials.
- **Edge Cases:**
  - Buy action only triggers if recommendation is explicitly Buy.
  - API key or network errors may cause order failure.
  - No sell or hold actions implemented.

---

### 3. Summary Table

| Node Name                     | Node Type                               | Functional Role                                 | Input Node(s)                       | Output Node(s)                  | Sticky Note                                                                                                           |
|-------------------------------|---------------------------------------|------------------------------------------------|-----------------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger                      | Weekly batch trigger                            |                                   | Stock List                     | ## Triggers \nThis workflow starts the analysis either on a weekly schedule for a predefined list of stocks or manual. |
| Stock List                    | Set                                  | Provides batch stock tickers                    | Schedule Trigger                  | Split Out                     |                                                                                                                       |
| Split Out                    | Split Out                            | Splits ticker array into individual items      | Stock List                       | Loop Over Items                |                                                                                                                       |
| Loop Over Items              | Split In Batches                     | Processes each ticker individually              | Split Out                       | Set Ticker for AI Agent        |                                                                                                                       |
| On form submission            | Form Trigger                        | Manual input of single ticker                    |                                 | Set Ticker for AI Agent        | ### Manual Single Trigger\nRun analysis for one stock instantly via a form.                                           |
| Set Ticker for AI Agent       | Set                                  | Sets ticker for downstream processing           | Loop Over Items, On form submission | Execute Workflow             |                                                                                                                       |
| Execute Workflow              | Execute Workflow                    | Calls main sub-workflow for analysis            | Set Ticker for AI Agent           | If                            |                                                                                                                       |
| If                           | If                                  | Checks if stock ticker is in predefined list    | Execute Workflow                 | Wait                          |                                                                                                                       |
| Wait                         | Wait                                | Pause before next iteration                      | If                              | Loop Over Items                |                                                                                                                       |
| Set Stock Symbol and API Key  | Set                                  | Sets ticker and TwelveData API key              | Loop Over Items                  | Test NASDAQ                   |                                                                                                                       |
| Test NASDAQ                  | HTTP Request                        | Tests if ticker exists on NASDAQ                 | Set Stock Symbol and API Key     | is on nasdaq?                 |                                                                                                                       |
| is on nasdaq?                | If                                  | Checks if NASDAQ response is valid               | Test NASDAQ                    | Set exchange to NASDAQ, Test NYSE |                                                                                                                       |
| Test NYSE                   | HTTP Request                        | Tests if ticker exists on NYSE                   | is on nasdaq?                   | is on NYSE?                   |                                                                                                                       |
| is on NYSE?                 | If                                  | Checks if NYSE response is valid                 | Test NYSE                      | Set exchange to NYSE, Stop and Error |                                                                                                                       |
| Set exchange to NASDAQ        | Set                                  | Sets exchange to NASDAQ and chart prefix        | is on nasdaq?                   | Merge1                        |                                                                                        |
| Set exchange to NYSE          | Set                                  | Sets exchange to NYSE and chart prefix          | is on NYSE?                    | Merge1                        |                                                                                                                       |
| Stop and Error               | Stop and Error                      | Stops workflow with "Stock doesn't exist" error | is on NYSE?                    |                                |                                                                                                                       |
| Merge1                       | Merge                              | Combines exchange info                           | Set exchange to NASDAQ, Set exchange to NYSE | Get Price History, Get Chart URL, Get Bollinger Bands, Get MACD |                                                                                                                       |
| Get Price History             | HTTP Request                        | Fetches 180-day daily price history              | Merge1                         | Calculate Support Resistance   | ## Technical Analysis \nThis sub-workflow gathers technical indicator data from Twelve Data.                          |
| Calculate Support Resistance  | Code                               | Calculates Fibonacci and support/resistance levels | Get Price History              | Merge                         |                                                                                                                       |
| Get Bollinger Bands           | HTTP Request                        | Fetches Bollinger Bands data                      | Merge1                         | Merge                         |                                                                                                                       |
| Get MACD                     | HTTP Request                        | Fetches MACD indicator data                       | Merge1                         | Merge                         |                                                                                                                       |
| Merge                        | Merge                              | Combines technical indicator data streams        | Calculate Support Resistance, Get Bollinger Bands, Get MACD | Organizing Data             |                                                                                                                       |
| Organizing Data              | Code                               | Synthesizes technical indicator data, creates summary | Merge                         | Merge-2                       |                                                                                                                       |
| Get Chart URL                | HTTP Request                        | Requests weekly candlestick chart image          | Merge1                         | First Technical Analysis       | ## Visual Chart Analysis\nThis part pulls a chart and uses vision AI to analyze it                                    |
| First Technical Analysis      | LangChain OpenAI Image Analysis    | Analyzes chart image and extracts structured data | Get Chart URL                 | Set Variable                  |                                                                                                                       |
| Set Variable                 | Set                                  | Stores AI chart analysis JSON                      | First Technical Analysis       | Merge-2                       |                                                                                                                       |
| Merge-2                      | Merge                              | Combines technical JSON and AI chart analysis data | Organizing Data, Set Variable, Fundamental Analysis Tool | Warp as JSON for GPT         |                                                                                                                       |
| Fundamental Analysis Tool     | AI Tool Workflow                   | Retrieves and summarizes financial statements     | Merge2                        | ChatGPT 4o1                   | ## Fundamental Analysis \nSub-workflow fetches financial health metrics from Alpha Vantage.                           |
| Warp as JSON for GPT          | Code                               | Wraps JSON data as formatted text for AI           | Merge-2                       | ChatGPT 4o                   |                                                                                                                       |
| ChatGPT 4o1                  | LangChain OpenAI                   | Synthesizes financial data into structured summary | Fundamental Analysis Tool     | Code in JavaScript            |                                                                                                                       |
| Code in JavaScript           | Code                               | Parses AI JSON string output into JSON              | ChatGPT 4o1                   | Merge2                       |                                                                                                                       |
| ChatGPT 4o                   | LangChain OpenAI                   | Synthesizes combined technical data into analysis text | Warp as JSON for GPT          | Set Final Response            |                                                                                                                       |
| Set Final Response           | Set                                  | Stores AI-generated technical analysis and image   | ChatGPT 4o                   | AI Agent                     |                                                                                                                       |
| Window Buffer Memory          | LangChain Memory Buffer Window     | Maintains session memory for AI                      |                                 | AI Agent                     |                                                                                                                       |
| AI Agent                    | LangChain Agent                    | Orchestrates calls to sub-tools and synthesizes final report | When Executed by Another Workflow | Buy?, Generate HTML          | ## Stock Analysis (Main AI Agent) \nCore AI agent orchestrates data gathering and synthesis.                        |
| Structured Output Parser      | LangChain Output Parser Structured | Parses AI agent JSON response into structured object | AI Agent                     | AI Agent                     |                                                                                                                       |
| Buy?                        | If                                  | Checks if recommendation includes "Buy"             | AI Agent                     | Buy in Alpaca, Generate HTML  | ### Optional: Complete paper trade in Alpaca                                                                        |
| Buy in Alpaca                | HTTP Request                        | Places market buy order on Alpaca API                | Buy?                         |                                |                                                                                                                       |
| Generate HTML               | HTML                                | Generates styled HTML email report from AI output    | AI Agent                     | Adjust HTML Colors           | ### Generate report and send to email                                                                                 |
| Adjust HTML Colors           | Code                               | Updates colors by sentiment and cleans HTML content   | Generate HTML                | Send Stock Analysis          |                                                                                                                       |
| Send Stock Analysis          | Email Send                         | Sends final email report using SMTP                   | Adjust HTML Colors           |                                | ### Add SMTP Credentials and add your email                                                                           |
| Generate Variables For API   | Code                               | Generates yesterday's date for news API query          |                                 | Set Variables1               |                                                                                                                       |
| Set Variables1              | Set                                  | Sets variables for news API: wantedDate, stockSymbol, apikey | Generate Variables For API  | Get News Data                |                                                                                                                       |
| Get News Data               | HTTP Request                        | Retrieves news sentiment and articles from Alpha Vantage | Set Variables1              | Analyse API Input            | ## Trends Analysis \nSub-workflow retrieves news and analyzes sentiment.                                             |
| Analyse API Input            | Code                               | Processes news articles, sentiment counts, topics      | Get News Data                | AI Agent                    |                                                                                                                       |
| When Executed by Another Workflow | Execute Workflow Trigger       | Entry point for triggered executions                   |                                 | AI Agent                     |                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a `Schedule Trigger` node:
     - Set to trigger weekly on Monday at 06:20 AM.
   - Add a `Set` node named "Stock List":
     - Assign an array variable `tickers` with desired stock symbols (e.g., ["AAPL", "MSFT", ...]).
   - Connect `Schedule Trigger` → `Stock List`.
   - Add a `Split Out` node to split the ticker array.
   - Connect `Stock List` → `Split Out`.
   - Add a `Split In Batches` node named "Loop Over Items" set to batch size 1.
   - Connect `Split Out` → `Loop Over Items`.
   - Add a `Form Trigger` node named "On form submission":
     - Configure with a required "tickers" field.
   - Connect `Loop Over Items` and `On form submission` both to the next step.

2. **Validate Stock Exchange:**
   - Add a `Set` node named "Set Stock Symbol and API Key":
     - Assign `ticker` from input.
     - Assign your TwelveData API key as `TwelveData_API_Key`.
   - Connect both `Loop Over Items` and `On form submission` to this node.
   - Add HTTP Request nodes:
     - "Test NASDAQ": GET `https://api.twelvedata.com/price?symbol={{ticker}}&exchange=NASDAQ&apikey={{TwelveData_API_Key}}`.
     - "Test NYSE": similar for NYSE.
   - Add `If` nodes "is on nasdaq?" and "is on NYSE?" to check if price exists.
   - Add `Set` nodes to set exchange and chart prefix based on validation.
   - Add a `Stop and Error` node to stop if ticker not found.
   - Merge outputs for downstream calls.

3. **Collect Technical Indicator Data:**
   - HTTP Request nodes to TwelveData API to get:
     - Price History: symbol, interval=1day, outputsize=180.
     - Bollinger Bands: latest.
     - MACD: latest.
   - Add Code node "Calculate Support Resistance":
     - Implements Fibonacci levels and local support/resistance from price history.
   - Merge all technical data.
   - Add HTTP Request node "Get Chart URL":
     - POST to Chart-Img API with weekly candle style chart.
     - Authenticate with Chart-Img API key.
   - Add LangChain OpenAI Image Analysis node "First Technical Analysis":
     - Pass chart image (base64).
     - Model: capable of image analysis (e.g., mistralai/mistral-small).
   - Add Set node to store AI visual analysis.
   - Merge technical JSON data and AI analysis.
   - Add Code node "Warp as JSON for GPT" to format JSON for AI.
   - Add LangChain OpenAI node to synthesize technical analysis text.
   - Add Set node to store final AI technical analysis text and chart image base64.

4. **Perform Fundamental Financial Analysis:**
   - Add Set node "Set Variables":
     - Assign ticker and Alpha Vantage API key.
   - Add HTTP Request nodes to Alpha Vantage for:
     - Company Overview.
     - Income Statement.
     - Balance Sheet.
   - Add Wait nodes between requests to avoid rate limits.
   - Merge data.
   - Add LangChain OpenAI node with prompt to produce structured financial summary JSON.
   - Add Code node to parse AI JSON string into JSON object.

5. **News Sentiment and Trends Analysis:**
   - Add Code node "Generate Variables For API" to get yesterday’s date.
   - Add Set node "Set Variables1":
     - Assign wantedDate, stockSymbol, and Alpha Vantage API key.
   - Add HTTP Request node "Get News Data":
     - Alpha Vantage NEWS_SENTIMENT endpoint with ticker and date filters.
   - Add Code node "Analyse API Input":
     - Process articles, count sentiments, extract top articles and hot topics.
     - Return structured sentiment and topic JSON.

6. **AI Orchestration & Final Synthesis:**
   - Add Execute Workflow Trigger node "When Executed by Another Workflow" for entry.
   - Add LangChain Agent node "AI Agent":
     - System message defines the orchestration and tool call procedure.
     - Pass ticker input.
     - Configure three tools:
       - `technical_analysis` (calls technical sub-workflow).
       - `trends_analysis` (calls news sentiment sub-workflow).
       - `fundamental_analysis` (calls fundamental sub-workflow).
     - Synthesize technical analysis text and final recommendation JSON as output.
   - Add Window Buffer Memory node to maintain session context.
   - Add Structured Output Parser node to validate AI output JSON.
   - Add `If` node "Buy?" to check if recommendation contains "Buy".

7. **Report Generation & Delivery:**
   - Add HTML node "Generate HTML":
     - Paste the provided comprehensive HTML template.
     - Reference AI agent JSON output fields using n8n expression syntax.
   - Add Code node "Adjust HTML Colors":
     - Paste the provided JavaScript code that adjusts color theming by sentiment and cleans HTML content.
   - Add Email Send node "Send Stock Analysis":
     - Configure SMTP credentials and sender/recipient emails.
     - Use the adjusted HTML as email body.

8. **Optional Automated Trading:**
   - Add HTTP Request node "Buy in Alpaca":
     - POST to Alpaca API endpoint `/v2/orders`.
     - Payload: market buy order for $7,000 notional.
     - Use Alpaca API key and secret credentials.
   - Connect from `Buy?` node’s "true" branch.

9. **Sub-Workflow Setup:**
   - **Technical Analysis Tool:**  
     - Accepts ticker, fetches technical data, returns structured JSON.
   - **Fundamental Analysis Tool:**  
     - Accepts ticker, fetches company overview, income statement, balance sheet, returns structured financial summary.
   - **Trends Analysis Tool:**  
     - Accepts ticker, fetches news sentiment data, returns sentiment and trending topics analysis.
   - Link these sub-workflows as tools in the AI Agent node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                       | Context or Link                                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| This workflow uses a multi-agent system with a central AI orchestrator invoking specialized sub-workflows for technical, fundamental, and news sentiment analysis. All APIs used can be acquired for free.            | Sticky Note on top left (position -3248, -272)                                                                          |
| For comprehensive setup instructions and detailed explanations, visit: https://docs.google.com/document/d/1Ri_GfuIlc0QVm1aDrWJjRCg_2vWRCrw_SjRV9cBoKmw/edit?usp=sharing                                               | Setup guide link from Sticky Note                                                                                       |
| Requires API credentials for: Google Gemini (PaLM), OpenRouter (OpenAI), Alpha Vantage, TwelveData, Chart-Img, SMTP email, and optionally Alpaca for paper trading.                                                 | Multiple sticky notes near nodes requesting credential input                                                           |
| The workflow includes error handling for invalid tickers and respects API rate limits using Wait nodes in Fundamental Analysis.                                                                                   | Observed in Stock Validation and Financial Analysis blocks                                                             |
| The HTML email template is fully styled for readability with dynamic sentiment-based color coding and supports up to five influential articles and five hot topics.                                                | HTML node "Generate HTML" and Code node "Adjust HTML Colors"                                                            |
| The visual chart analysis uses a specialized AI model capable of image input to extract technical visual features from a weekly candlestick chart.                                                                  | Node "First Technical Analysis" with model "mistralai/mistral-small-3.2-24b-instruct:free"                              |

---

**Disclaimer:**  
The text and data processed in this workflow are legal and public. The workflow strictly follows content policies and does not generate or handle illegal or offensive content. All third-party API usage must comply with their respective terms of service.