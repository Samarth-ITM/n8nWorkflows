Monitor Stock Market with AI: News Analysis & Multi-Channel Alerts via Slack & Telegram

https://n8nworkflows.xyz/workflows/monitor-stock-market-with-ai--news-analysis---multi-channel-alerts-via-slack---telegram-10284


# Monitor Stock Market with AI: News Analysis & Multi-Channel Alerts via Slack & Telegram

### 1. Workflow Overview

This workflow, titled **"Monitor Stock Market with AI: News Analysis & Multi-Channel Alerts via Slack & Telegram"**, is designed to automate the monitoring of financial news, perform AI-driven sentiment and impact analysis, integrate real-time stock data, and generate actionable trading signals. It targets traders, portfolio managers, and financial analysts who need timely and data-driven alerts for market movements.

The workflow runs every 15 minutes, fetching news from multiple RSS feeds (Bloomberg, CNBC, Reuters), combining and filtering the news for relevance based on recentness and presence of stock tickers/keywords. It then applies AI sentiment analysis to gauge market sentiment and impact, enriches the insights with stock price and volume data from Yahoo Finance, and synthesizes a comprehensive intelligence report.

Signals deemed actionable are routed to Slack (detailed alerts) and Telegram (mobile notifications), while all signals and associated trading strategies are logged in Airtable and Google Sheets for record-keeping and performance tracking. Finally, the workflow generates AI-driven trading strategy recommendations including position sizing, entry/exit plans, and risk management.

The logical blocks of the workflow are as follows:

- **1.1 Scheduled Trigger & News Collection:** Automatically triggers every 15 minutes and fetches news from multiple RSS feeds.
- **1.2 News Aggregation & Filtering:** Merges feeds and filters articles published within the last hour, extracting tickers and keywords.
- **1.3 AI Sentiment & Impact Analysis:** Sends filtered news to OpenAI for market sentiment, impact, and trading signals.
- **1.4 Stock Data Retrieval:** Fetches recent stock price and volume data from Yahoo Finance for detected tickers.
- **1.5 Intelligence Report Synthesis:** Combines AI analysis and stock data to create a detailed report with technical indicators and composite scores.
- **1.6 Signal Filtering & Notifications:** Selects actionable signals and sends alerts via Slack and Telegram.
- **1.7 Logging & Strategy Generation:** Logs signals in Airtable and Google Sheets and generates AI-driven trading strategies.
- **1.8 Strategy Storage:** Updates Airtable records with the generated trading plans.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & News Collection

- **Overview:** Automatically initiates the workflow every 15 minutes and fetches the latest financial news from three major sources.
- **Nodes Involved:**
  - Every 15 Minutes (Schedule Trigger)
  - Bloomberg Markets Feed (RSS Feed Read)
  - CNBC Breaking News (RSS Feed Read)
  - Reuters Business (RSS Feed Read)
- **Node Details:**

  - **Every 15 Minutes**
    - Type: Schedule Trigger
    - Role: Automatically starts workflow every 15 minutes.
    - Configuration: Interval set to 15 minutes.
    - Inputs: None (trigger node).
    - Outputs: Connects to all three RSS Feed nodes.
    - Edge cases: If workflow is paused or n8n instance down, trigger will not fire.

  - **Bloomberg Markets Feed**
    - Type: RSS Feed Read
    - Role: Fetches latest Bloomberg market news.
    - Configuration: URL set to Bloomberg Markets news RSS.
    - Inputs: From Schedule Trigger.
    - Outputs: To News Aggregation node.
    - Possible failures: RSS feed URL inaccessible, network errors.

  - **CNBC Breaking News**
    - Type: RSS Feed Read
    - Role: Fetches latest CNBC breaking news.
    - Configuration: URL set to CNBC RSS feed.
    - Inputs: From Schedule Trigger.
    - Outputs: To News Aggregation node.
    - Possible failures: RSS feed URL inaccessible, network errors.

  - **Reuters Business**
    - Type: RSS Feed Read
    - Role: Fetches latest Reuters business news.
    - Configuration: URL set to Reuters Business news RSS.
    - Inputs: From Schedule Trigger.
    - Outputs: To News Aggregation node.
    - Possible failures: RSS feed URL inaccessible, network errors.

---

#### 1.2 News Aggregation & Filtering

- **Overview:** Combines news items from all sources into a single stream, filters articles published within the last hour, and extracts stock tickers and predefined keywords for analysis.
- **Nodes Involved:**
  - Aggregate News Sources (Merge)
  - Filter & Extract Signals (Code)
- **Node Details:**

  - **Aggregate News Sources**
    - Type: Merge
    - Role: Combines inputs from all RSS feeds into one continuous stream.
    - Configuration: Mode set to "combine" (merges all items).
    - Inputs: From Bloomberg, CNBC, Reuters nodes.
    - Outputs: To Filter & Extract Signals node.
    - Edge cases: Empty feeds could result in no items merged.

  - **Filter & Extract Signals**
    - Type: Code (JavaScript)
    - Role: Filters articles published in the last hour and extracts potential stock tickers and keywords.
    - Configuration:
      - Filters items based on publication date (> one hour ago).
      - Extracts up to 5 unique uppercase ticker symbols (2-5 chars).
      - Searches for predefined keywords related to earnings, mergers, market events, AI, inflation, etc.
    - Key expressions: Uses JavaScript regex and array filtering.
    - Inputs: Combined news items.
    - Outputs: Filtered news with extracted metadata.
    - Edge cases:
      - Articles without pubDate or isoDate fields default to exclusion.
      - Ticker extraction may capture false positives (common uppercase words).
      - Keywords list is static; may miss emerging terms.

---

#### 1.3 AI Sentiment & Impact Analysis

- **Overview:** Sends each filtered news article to OpenAI to analyze market sentiment, impact level, affected sectors, trading signals, confidence, catalysts, and risk factors.
- **Nodes Involved:**
  - AI Sentiment Analysis (OpenAI Chat)
- **Node Details:**

  - **AI Sentiment Analysis**
    - Type: OpenAI (Chat Completion)
    - Role: Uses GPT to analyze news articles and output structured JSON with sentiment and trading signals.
    - Configuration:
      - System prompt frames the AI as a financial analyst.
      - User prompt includes article title, content, source, detected tickers, and keywords.
      - Max tokens set to 800; temperature 0.2 for focused responses.
      - Model: Chat completion resource.
    - Inputs: Filtered news articles with metadata.
    - Outputs: AI analysis JSON per article.
    - Edge cases:
      - API rate limits or connectivity issues.
      - AI output parsing errors if JSON is malformed.
      - Potential bias or inaccuracies from AI model.

---

#### 1.4 Stock Data Retrieval

- **Overview:** Fetches recent stock price and volume data for the primary ticker from Yahoo Finance API to complement AI analysis.
- **Nodes Involved:**
  - Fetch Stock Data (Code)
- **Node Details:**

  - **Fetch Stock Data**
    - Type: Code (JavaScript)
    - Role: Calls Yahoo Finance API to retrieve 5-day daily prices and volumes for the main detected ticker.
    - Configuration:
      - Picks first ticker from extracted list or defaults to 'SPY' if none.
      - Uses fetch API with User-Agent header.
      - Handles API failures by returning mock static data to keep workflow running.
    - Inputs: From filtered news articles.
    - Outputs: JSON containing chart data or fallback mock data.
    - Edge cases:
      - Network failures or API downtime.
      - Ticker not valid on Yahoo Finance.
      - Rate limiting by Yahoo Finance.
      - Fallback mock data used transparently.

---

#### 1.5 Intelligence Report Synthesis

- **Overview:** Combines AI sentiment analysis results with stock market data to create a comprehensive intelligence report including technical indicators and a composite score.
- **Nodes Involved:**
  - Synthesize Intelligence Report (Code)
- **Node Details:**

  - **Synthesize Intelligence Report**
    - Type: Code (JavaScript)
    - Role:
      - Parses AI JSON response.
      - Extracts stock price and volume arrays.
      - Calculates:
        - Latest price and price change percentage.
        - 5-day Simple Moving Average (SMA).
        - Trend determination (Uptrend if latest price > SMA).
        - Volume spike ratio (latest volume / average volume).
      - Generates composite score based on trading signal, confidence, momentum (price change), and volume spike.
      - Determines if signal is actionable based on confidence (>70) and price change or critical impact.
    - Inputs: AI analysis and stock data.
    - Outputs: Structured intelligence report JSON with news, AI, technical, and composite metrics.
    - Edge cases:
      - Missing or malformed data fields handled with defaults.
      - Division by zero avoided in calculations.
      - Composite score formula assumes specific signal labels.

---

#### 1.6 Signal Filtering & Notifications

- **Overview:** Filters intelligence reports to pass only actionable signals; sends detailed alerts to Slack and condensed alerts to Telegram.
- **Nodes Involved:**
  - Filter Actionable Signals (Switch)
  - Send Trading Alert (Slack)
  - Mobile Notification (Telegram)
- **Node Details:**

  - **Filter Actionable Signals**
    - Type: Switch
    - Role: Filters reports to select only actionable trade signals.
    - Configuration: Checks conditions on actionable field or composite score (exact condition not specified in detail).
    - Inputs: From Intelligence Report node.
    - Outputs: To Slack and Telegram alert nodes.
    - Edge cases: Improper filtering logic can miss valid signals or generate false positives.

  - **Send Trading Alert**
    - Type: Slack
    - Role: Sends detailed, formatted trading alerts to a Slack channel.
    - Configuration:
      - Uses OAuth2 authentication.
      - Message includes ticker, signal, confidence, impact, headline, source, technical indicators, AI analysis, risks, and article link.
      - Markdown enabled.
    - Inputs: Filtered actionable signals.
    - Outputs: To Airtable logging node.
    - Edge cases:
      - Slack webhook or authentication failures.
      - Message formatting errors.

  - **Mobile Notification**
    - Type: Telegram
    - Role: Sends summarized alerts to Telegram chat.
    - Configuration:
      - Chat ID must be configured.
      - Message includes signal, ticker, price, headline, confidence, composite score, and link.
    - Inputs: Filtered actionable signals.
    - Outputs: To Airtable logging node.
    - Edge cases:
      - Telegram bot authentication or chat ID misconfiguration.
      - Telegram API rate limits.

---

#### 1.7 Logging & Strategy Generation

- **Overview:** Logs signals in Airtable and Google Sheets for monitoring; generates AI-driven trading strategies based on signal data.
- **Nodes Involved:**
  - Log Signal Database (Airtable)
  - Track Performance Log (Google Sheets)
  - Generate Trading Strategy (OpenAI Chat)
- **Node Details:**

  - **Log Signal Database**
    - Type: Airtable
    - Role: Creates a new record in Airtable table "tblSignals" with detailed signal and analysis data.
    - Configuration:
      - Maps fields such as Link, Risks, Trend, Impact, Signal, Source, Ticker, Sectors, Catalyst, Headline, Sentiment, Timestamp, Actionable, Confidence, Price Change, Volume Ratio, Current Price, Composite Score.
      - Requires Airtable base and table IDs.
    - Inputs: From Slack and Telegram nodes.
    - Outputs: To Google Sheets and Trading Strategy nodes.
    - Edge cases:
      - API key or access errors.
      - Mapping errors if fields are missing.

  - **Track Performance Log**
    - Type: Google Sheets
    - Role: Appends a row to a Google Sheets document to track signal performance over time.
    - Configuration:
      - Maps date, time, price, score, change, signal, source, ticker, sentiment, confidence.
      - Requires Google Sheets document ID and sheet ID.
    - Inputs: From Airtable logging node.
    - Outputs: To Generate Trading Strategy node.
    - Edge cases:
      - Google API authentication failure.
      - Quota limits on append operations.

  - **Generate Trading Strategy**
    - Type: OpenAI (Chat Completion)
    - Role: Generates a portfolio strategy suggestion including position sizing, entry, stop loss, take profit, and horizon.
    - Configuration:
      - System prompt defines portfolio strategist role.
      - User prompt includes trading signal, ticker, price, confidence, volatility, sentiment, risks.
      - Max tokens 600; temperature 0.4.
    - Inputs: From Google Sheets logging node.
    - Outputs: To Store Trading Plan node.
    - Edge cases:
      - API rate limits or malformed responses.
      - Strategy recommendations may need validation.

---

#### 1.8 Strategy Storage

- **Overview:** Updates existing Airtable signal records with the generated trading strategy in JSON format.
- **Nodes Involved:**
  - Store Trading Plan (Airtable)
- **Node Details:**

  - **Store Trading Plan**
    - Type: Airtable
    - Role: Updates the previously created Airtable record with the trading strategy JSON.
    - Configuration:
      - Uses Record_ID from Log Signal Database node.
      - Saves Trading_Strategy field as JSON string.
    - Inputs: From Generate Trading Strategy node.
    - Outputs: None (workflow end).
    - Edge cases:
      - Record update failures.
      - JSON stringifying errors.

---

### 3. Summary Table

| Node Name                 | Node Type             | Functional Role                                 | Input Node(s)                | Output Node(s)                      | Sticky Note                                                                                      |
|---------------------------|-----------------------|------------------------------------------------|------------------------------|------------------------------------|------------------------------------------------------------------------------------------------|
| Every 15 Minutes          | Schedule Trigger      | Triggers workflow every 15 minutes              | None                         | Bloomberg Markets Feed, CNBC Breaking News, Reuters Business | Triggers the workflow automatically every 15 minutes.                                         |
| Bloomberg Markets Feed     | RSS Feed Read         | Fetches Bloomberg market news                   | Every 15 Minutes             | Aggregate News Sources              | Fetches latest market news via RSS.                                                            |
| CNBC Breaking News         | RSS Feed Read         | Fetches CNBC breaking news                       | Every 15 Minutes             | Aggregate News Sources              | Fetches latest market news via RSS.                                                            |
| Reuters Business           | RSS Feed Read         | Fetches Reuters business news                    | Every 15 Minutes             | Aggregate News Sources              | Fetches latest market news via RSS.                                                            |
| Aggregate News Sources     | Merge                 | Combines all fetched RSS feeds into one stream | Bloomberg, CNBC, Reuters     | Filter & Extract Signals            | Combines all fetched RSS feeds into one stream.                                                |
| Filter & Extract Signals   | Code                  | Filters recent news, extracts tickers & keywords | Aggregate News Sources       | AI Sentiment Analysis, Fetch Stock Data | Filters recent news, extracts tickers and keywords.                                            |
| AI Sentiment Analysis      | OpenAI Chat           | Analyzes market sentiment and impact            | Filter & Extract Signals      | Synthesize Intelligence Report     | Uses AI to analyze market sentiment and impact.                                               |
| Fetch Stock Data           | Code                  | Pulls stock prices and volumes from Yahoo Finance | Filter & Extract Signals      | Synthesize Intelligence Report     | Pulls stock prices and volumes from Yahoo Finance.                                            |
| Synthesize Intelligence Report | Code              | Merges AI and stock data into insights           | AI Sentiment Analysis, Fetch Stock Data | Filter Actionable Signals          | Merges AI and stock data into insights.                                                       |
| Filter Actionable Signals  | Switch                | Keeps only strong or high-confidence trade signals | Synthesize Intelligence Report | Send Trading Alert, Mobile Notification | Keeps only strong or high-confidence trade signals.                                           |
| Send Trading Alert         | Slack                 | Sends detailed trading alerts to Slack channel  | Filter Actionable Signals     | Log Signal Database                 | Sends detailed trading alerts to Slack channel.                                               |
| Mobile Notification       | Telegram               | Sends condensed trade alerts to Telegram mobile | Filter Actionable Signals     | Log Signal Database                 | Sends condensed trade alerts to Telegram mobile app.                                          |
| Log Signal Database        | Airtable              | Stores analyzed signals in Airtable database    | Send Trading Alert, Mobile Notification | Track Performance Log            | Stores analyzed signals in Airtable database.                                                 |
| Track Performance Log      | Google Sheets         | Logs each trade’s data into Google Sheets       | Log Signal Database          | Generate Trading Strategy           | Logs each trade’s data into Google Sheets.                                                    |
| Generate Trading Strategy  | OpenAI Chat           | Creates AI-driven strategy with entry and exit plan | Track Performance Log        | Store Trading Plan                  | Creates AI-driven strategy with entry and exit plan.                                          |
| Store Trading Plan         | Airtable              | Updates Airtable record with full trading strategy JSON | Generate Trading Strategy     | None                               | Updates Airtable record with full trading strategy JSON.                                      |
| Sticky Note               | Sticky Note           | Workflow comments and explanations               | None                         | None                               | # 🧠 AI-Powered Market Intelligence & Trading Signal Workflow ... [LinkedIn](https://www.linkedin.com/in/daniel-shashko) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: Schedule Trigger
   - Set to trigger every 15 minutes.
   - Name: "Every 15 Minutes"

2. **Create RSS Feed Read Nodes (3)**
   - Bloomberg Markets Feed:
     - URL: `https://feeds.bloomberg.com/markets/news.rss`
     - Connect input from "Every 15 Minutes"
   - CNBC Breaking News:
     - URL: `https://www.cnbc.com/id/100003114/device/rss/rss.html`
     - Connect input from "Every 15 Minutes"
   - Reuters Business:
     - URL: `https://feeds.reuters.com/reuters/businessNews`
     - Connect input from "Every 15 Minutes"

3. **Create Merge Node**
   - Type: Merge
   - Mode: Combine
   - Connect inputs from all three RSS Feed nodes.
   - Name: "Aggregate News Sources"

4. **Create Code Node to Filter & Extract Signals**
   - Paste the JavaScript code to:
     - Filter articles published within the last hour.
     - Extract stock tickers (regex: uppercase words 2-5 chars).
     - Extract predefined keywords related to market events.
   - Connect input from "Aggregate News Sources"
   - Name: "Filter & Extract Signals"

5. **Create OpenAI Chat Node for Sentiment Analysis**
   - Configure credentials for OpenAI API.
   - Use system prompt defining financial analyst role.
   - User prompt includes article title, content, source, tickers, keywords.
   - Set max tokens 800, temperature 0.2.
   - Connect input from "Filter & Extract Signals"
   - Name: "AI Sentiment Analysis"

6. **Create Code Node to Fetch Stock Data**
   - Paste JavaScript code to fetch 5-day stock price and volume data from Yahoo Finance for first ticker or default to "SPY".
   - Include error handling to provide mock data fallback.
   - Connect input from "Filter & Extract Signals"
   - Name: "Fetch Stock Data"

7. **Create Code Node to Synthesize Intelligence Report**
   - Paste JavaScript code to combine:
     - AI analysis JSON parsing.
     - Stock price and volume data extraction.
     - Calculate price change %, SMA5, trend, volume ratio.
     - Compute composite score and actionable flag.
   - Connect inputs from both "AI Sentiment Analysis" and "Fetch Stock Data"
   - Name: "Synthesize Intelligence Report"

8. **Create Switch Node to Filter Actionable Signals**
   - Configure rule to allow only signals with:
     - Confidence score > 70 and absolute price change > 2%, or
     - Impact level equals "Critical".
   - Connect input from "Synthesize Intelligence Report"
   - Name: "Filter Actionable Signals"

9. **Create Slack Node to Send Trading Alert**
   - Configure OAuth2 credentials for Slack.
   - Use markdown text message template incorporating ticker, signal, confidence, impact, headline, source, technical analysis, AI analysis, risks, and article link.
   - Connect input from "Filter Actionable Signals"
   - Name: "Send Trading Alert"

10. **Create Telegram Node for Mobile Notification**
    - Configure Telegram bot credentials.
    - Set Chat ID to your Telegram chat.
    - Use condensed alert message format with signal, ticker, price, headline, confidence, composite score, and link.
    - Connect input from "Filter Actionable Signals"
    - Name: "Mobile Notification"

11. **Create Airtable Node to Log Signal**
    - Configure Airtable credentials.
    - Set Base ID and Table ID for "tblSignals" in your Airtable.
    - Map fields including Link, Risks, Trend, Impact, Signal, Source, Ticker, Sectors, Catalyst, Headline, Sentiment, Timestamp, Actionable, Confidence, Price Change, Volume Ratio, Current Price, Composite Score.
    - Connect inputs from both "Send Trading Alert" and "Mobile Notification"
    - Name: "Log Signal Database"

12. **Create Google Sheets Node to Log Performance**
    - Configure Google Sheets credentials.
    - Set Document ID and Sheet ID.
    - Append rows with Date, Time, Price, Score, Change, Signal, Source, Ticker, Sentiment, Confidence.
    - Connect input from "Log Signal Database"
    - Name: "Track Performance Log"

13. **Create OpenAI Chat Node to Generate Trading Strategy**
    - Configure OpenAI API credentials.
    - System prompt: Portfolio strategist role.
    - User prompt: Include signal, ticker, price, confidence, volatility, sentiment, risks.
    - Max tokens 600, temperature 0.4.
    - Connect input from "Track Performance Log"
    - Name: "Generate Trading Strategy"

14. **Create Airtable Node to Store Trading Plan**
    - Update existing Airtable record using Record_ID from "Log Signal Database".
    - Store trading strategy JSON string.
    - Connect input from "Generate Trading Strategy"
    - Name: "Store Trading Plan"

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| # 🧠 AI-Powered Market Intelligence & Trading Signal Workflow: Continuously monitors Bloomberg, CNBC, Reuters every 15 minutes; combines AI sentiment and stock data; sends alerts to Slack and Telegram; logs data and creates AI-driven strategies. | Workflow high-level description inside Sticky Note (Node: Sticky Note14)                           |
| Connect with me on LinkedIn: [Daniel Shashko](https://www.linkedin.com/in/daniel-shashko)                                                                                                                                           | Author credit and professional contact                                                             |
| Slack OAuth2 authentication must be configured correctly for messaging.                                                                                                                                                             | Slack node credential requirement                                                                  |
| Telegram Chat ID must be replaced with your actual chat ID; Telegram bot token credentials are required.                                                                                                                           | Telegram node credential and parameter detail                                                      |
| Airtable base and table IDs must be set according to your Airtable setup; API key credentials required.                                                                                                                             | Airtable nodes credential and configuration                                                        |
| Google Sheets document and sheet IDs must correspond to your Google account and sheet structure; OAuth2 credentials needed.                                                                                                        | Google Sheets node credential and configuration                                                    |
| OpenAI API keys with sufficient quota for Chat Completion endpoints are required; consider rate limits and error handling for robust production use.                                                                              | OpenAI nodes configuration notes                                                                  |
| Yahoo Finance unofficial API is used via GET requests; subject to change and possible rate limits. Implement error handling as done here with fallback mock data to prevent workflow stoppage.                                      | Stock Data Fetch node notes                                                                        |
| The keyword list in filtering code can be customized or extended to capture emerging market themes.                                                                                                                                | Filter & Extract Signals node customization                                                       |
| The AI prompts can be improved or localized for better relevance or expanded outputs.                                                                                                                                              | AI Sentiment Analysis and Generate Trading Strategy nodes can be tuned                             |
| Composite score calculation uses weighted factors; adjust weights to suit your trading preferences and risk tolerance.                                                                                                            | Synthesize Intelligence Report node logic                                                         |

---

This structured documentation enables understanding, maintenance, and extension of the workflow, facilitating seamless reproduction or integration into other systems.