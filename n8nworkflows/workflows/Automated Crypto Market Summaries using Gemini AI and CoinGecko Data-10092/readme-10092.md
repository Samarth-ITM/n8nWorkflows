Automated Crypto Market Summaries using Gemini AI and CoinGecko Data

https://n8nworkflows.xyz/workflows/automated-crypto-market-summaries-using-gemini-ai-and-coingecko-data-10092


# Automated Crypto Market Summaries using Gemini AI and CoinGecko Data

### 1. Workflow Overview

This workflow automates the generation and distribution of daily cryptocurrency market summaries to a Discord channel. It integrates real-time market data and sentiment analysis using CoinGecko and NewsAPI, respectively, and leverages Google Gemini AI to produce concise, insightful investment briefs. The workflow is structured into the following logical blocks:

- **1.1 Schedule Trigger:** Initiates the workflow automatically daily at 8 AM.
- **1.2 Data Acquisition:** Fetches live market data from CoinGecko and recent crypto news headlines from NewsAPI.
- **1.3 Data Merging and Formatting:** Combines and normalizes market and news data into a structured dataset suitable for AI processing.
- **1.4 AI Market Analysis:** Uses Google Gemini AI to analyze the merged data and generate a detailed market summary and sentiment report.
- **1.5 Output Parsing and Formatting:** Parses AI JSON output, formats it for readability, and prepares a message for Discord.
- **1.6 Discord Posting:** Automatically posts the formatted market briefing to a specified Discord channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  Automatically triggers the workflow every day at 8 AM, ensuring timely generation of market summaries without manual intervention.

- **Nodes Involved:**  
  - `8AM Everyday` (Schedule Trigger)

- **Node Details:**  
  - **Type:** Schedule Trigger  
  - **Configuration:** Set to trigger once daily at 8 AM local time.  
  - **Expressions:** None.  
  - **Connections:** Outputs to both `Market Data` and `Sentiment Data` nodes in parallel.  
  - **Edge Cases:** Misconfiguration may lead to missed or multiple triggers; time zone considerations might affect execution time.  
  - **Notes:** Sticky Note emphasizes modifiable timing for user preference.

#### 2.2 Data Acquisition

- **Overview:**  
  Fetches quantitative market data and qualitative sentiment data from external APIs to feed the AI analysis.

- **Nodes Involved:**  
  - `Market Data` (HTTP Request to CoinGecko)  
  - `Sentiment Data` (HTTP Request to NewsAPI)  
  - `Merge` (Merge Node combining outputs)

- **Node Details:**  

  - **Market Data:**  
    - Type: HTTP Request  
    - Config: Retrieves market data for Bitcoin, Ethereum, and Solana from CoinGecko’s public API without authentication.  
    - Key Query: Fetches current price, 24h price change (absolute and %), volume, market cap, highs and lows.  
    - Input: From `8AM Everyday` trigger  
    - Output: Merges with `Sentiment Data` via `Merge` node.  
    - Edge Cases: API downtime, rate limits, or malformed responses may cause failures.

  - **Sentiment Data:**  
    - Type: HTTP Request  
    - Config: Queries NewsAPI for latest news articles with keywords (“crypto”, “bitcoin”, “ethereum”, “solana”).  
    - Auth: Uses NewsAPI API key via HTTP header authentication.  
    - Input: From `8AM Everyday` trigger  
    - Output: Merges with `Market Data` via `Merge` node.  
    - Edge Cases: Authentication errors, rate limiting, or empty news results.

  - **Merge:**  
    - Type: Merge  
    - Role: Combines market and sentiment data outputs into one stream for downstream processing.  
    - Input: From `Market Data` and `Sentiment Data` nodes.  
    - Output: Feeds into `Merge Data` code node.  
    - Edge Cases: Ensuring synchronization; if one API fails, data inconsistency may occur.

#### 2.3 Data Merging and Formatting

- **Overview:**  
  Processes and formats the merged raw data, filtering relevant news per coin and structuring a clean dataset for AI input.

- **Nodes Involved:**  
  - `Merge Data` (Code Node)  
  - `Parse Data` (Code Node)

- **Node Details:**  

  - **Merge Data:**  
    - Type: Code  
    - Role: Extracts news articles and market data, filters news articles mentioning each coin symbol (BTC, ETH, SOL), limits to 5 articles per coin for efficiency.  
    - Key Code: Filters news by checking if article title or description contains coin symbols (case-insensitive).  
    - Input: Merged data stream from `Merge`.  
    - Output: Array of objects per coin with `market_overview` and `related_news` arrays.  
    - Edge Cases: Missing fields in news articles or market data might cause key errors; empty news arrays result in empty related_news.

  - **Parse Data:**  
    - Type: Code  
    - Role: Normalizes the `Merge Data` output into a single JSON object containing arrays of market overview and related news with coin symbols attached.  
    - Input: Output from `Merge Data`.  
    - Output: Single JSON object with two arrays: `market_overview` and `related_news`.  
    - Edge Cases: Input array empty or malformed data causing parse errors.

#### 2.4 AI Market Analysis

- **Overview:**  
  Uses Google Gemini AI to analyze structured data and produce a formatted market summary, sentiment overview, top movers, and actionable insights.

- **Nodes Involved:**  
  - `Research Analyst` (Langchain Agent)  
  - `Google Gemini Chat Model` (AI Model Node)

- **Node Details:**  

  - **Research Analyst:**  
    - Type: Langchain Agent (AI)  
    - Role: Receives parsed market and sentiment data, sends prompt to Google Gemini AI to generate structured JSON market report.  
    - Prompt: Detailed instructions to analyze market trends, top 3 coins by 24h performance, sentiment narratives, actionable takeaways, and confidence score.  
    - Output: JSON string wrapped in markdown code block per AI model's response.  
    - Input: Output from `Parse Data`.  
    - Output: Feeds into `Extract Keywords`.  
    - Edge Cases: AI may produce malformed JSON, incomplete responses, or no response; network/auth issues with Google Gemini API.  
    - Credential: Google Gemini API key configured.

  - **Google Gemini Chat Model:**  
    - Type: AI Language Model Node  
    - Role: Underlying LLM called by `Research Analyst`.  
    - Input: Prompt and data from `Research Analyst`.  
    - Output: AI-generated market summary in JSON.  
    - Edge Cases: API quota limits, response latency, or errors.

#### 2.5 Output Parsing and Formatting

- **Overview:**  
  Parses AI JSON response to extract meaningful data, filter invalid reports, merge multiple reports if any, and prepare a user-friendly Discord message.

- **Nodes Involved:**  
  - `Extract Keywords` (Function Node)

- **Node Details:**  
  - Type: Function  
  - Role: Parses AI output wrapped inside triple backticks with “json” tag, converts to JSON, filters out placeholders or invalid data, calculates average confidence score, merges multiple reports’ Discord messages.  
  - Input: AI raw output from `Research Analyst`.  
  - Output: JSON object with merged reports, confidence score, and full Discord message string.  
  - Edge Cases: Parsing failures due to malformed AI output, empty AI response, or missing fields.  
  - Logging: Errors logged in node console.

#### 2.6 Discord Posting

- **Overview:**  
  Posts the formatted market summary message to a specified Discord channel using a bot token.

- **Nodes Involved:**  
  - `Post Message` (Discord Node)

- **Node Details:**  
  - Type: Discord Node  
  - Role: Sends the AI-generated and parsed market briefing to a Discord channel (#general by default).  
  - Configuration: Uses Discord Bot API token for authentication, selects guild and channel IDs dynamically.  
  - Message Content: Markdown formatted with emojis and sections for market summary, top coins, sentiment overview, takeaway, and confidence score.  
  - Input: Output from `Extract Keywords`.  
  - Edge Cases: Authentication errors with Discord API, incorrect guild/channel IDs, message length limits, network errors.

---

### 3. Summary Table

| Node Name            | Node Type                         | Functional Role                     | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                                     |
|----------------------|----------------------------------|-----------------------------------|----------------------------|--------------------------|----------------------------------------------------------------------------------------------------------------|
| 8AM Everyday         | Schedule Trigger                 | Trigger workflow daily at 8 AM    | -                          | Market Data, Sentiment Data | Automatically runs every morning at 8AM (modifiable). Initiates workflow by fetching real-time data            |
| Market Data          | HTTP Request                    | Fetch live market data from CoinGecko | 8AM Everyday               | Merge                    | Fetches live market data for BTC, ETH, SOL; no auth required                                                  |
| Sentiment Data       | HTTP Request                    | Fetch recent crypto news headlines from NewsAPI | 8AM Everyday               | Merge                    | Retrieves latest crypto news; requires NewsAPI key                                                            |
| Merge                | Merge                          | Combine market and sentiment data | Market Data, Sentiment Data | Merge Data               | Combines outputs from CoinGecko and NewsAPI into one unified dataset                                           |
| Merge Data           | Code                           | Format and filter merged data     | Merge                      | Parse Data               | Combines market data and news articles filtered per coin; limits news articles per asset                      |
| Parse Data           | Code                           | Normalize merged data structure   | Merge Data                 | Research Analyst         | Normalizes all data into schema for LLM input                                                                  |
| Research Analyst     | Langchain Agent (AI)            | Analyze data and generate market summary | Parse Data                 | Extract Keywords          | Uses Google Gemini AI to produce structured JSON market report                                                 |
| Google Gemini Chat Model | AI Language Model Node          | AI engine for market analysis     | Research Analyst (ai_languageModel) | Research Analyst         | Primary LLM responsible for market analysis and summary generation                                            |
| Extract Keywords     | Function                       | Parse AI output and prepare message | Research Analyst           | Post Message             | Parses AI JSON output, merges reports, averages confidence score                                               |
| Post Message         | Discord Node                   | Post formatted summary to Discord | Extract Keywords           | -                        | Publishes report to Discord channel using Bot Token                                                           |
| Sticky Note          | Sticky Note                    | Documentation and explanations    | -                          | -                        | Various notes explaining each workflow segment, including API usage, AI role, and Discord posting instructions |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**
   - Type: Schedule Trigger  
   - Configure to trigger daily at 8:00 AM local time.

2. **Create an HTTP Request Node for Market Data**
   - Name: `Market Data`  
   - Type: HTTP Request  
   - URL: `https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&ids=bitcoin,ethereum,solana`  
   - Authentication: None required  
   - Connect output from `8AM Everyday`.

3. **Create an HTTP Request Node for Sentiment Data**
   - Name: `Sentiment Data`  
   - Type: HTTP Request  
   - URL: `https://newsapi.org/v2/everything`  
   - Query Parameters:  
     - `q`: `crypto OR bitcoin OR ethereum OR solana`  
     - `language`: `en`  
     - `sortBy`: `publishedAt`  
     - `pageSize`: `10`  
   - Authentication: HTTP Header Auth with NewsAPI key credential  
   - Connect output from `8AM Everyday`.

4. **Add a Merge Node**
   - Name: `Merge`  
   - Type: Merge  
   - Connect inputs from `Market Data` (index 1) and `Sentiment Data` (index 0).

5. **Add a Code Node to Merge Data**
   - Name: `Merge Data`  
   - Type: Code  
   - Input: Output of `Merge` node.  
   - Paste the JavaScript code that extracts news articles, filters by coin symbols, limits to 5 articles each, and formats market overview.  
   - Connect output to next parsing node.

6. **Add a Code Node to Parse Data**
   - Name: `Parse Data`  
   - Type: Code  
   - Input: `Merge Data` output.  
   - Paste code that normalizes the data into a single JSON with arrays for `market_overview` and `related_news`.  
   - Connect output to AI node.

7. **Add Langchain Agent Node for AI Analysis**
   - Name: `Research Analyst`  
   - Type: Langchain Agent  
   - Input: Output from `Parse Data`.  
   - Configure prompt as provided, instructing AI to analyze market trends, sentiment, top coins, actionable advice, and confidence score, returning structured JSON.  
   - Set system message to define AI persona and instructions.  
   - Connect to AI language model node.

8. **Add Google Gemini Chat Model Node**
   - Name: `Google Gemini Chat Model`  
   - Type: AI Language Model Node  
   - Credential: Configure with Google Gemini API key.  
   - Connect as AI model for `Research Analyst`.

9. **Add a Function Node to Extract and Parse AI Output**
   - Name: `Extract Keywords`  
   - Type: Function  
   - Paste the provided JavaScript code to parse AI JSON output from markdown, filter invalid reports, merge messages, and calculate average confidence.  
   - Connect output to Discord node.

10. **Add Discord Node to Post Message**
    - Name: `Post Message`  
    - Type: Discord Node  
    - Credential: Discord Bot Token (OAuth2 Bot Token recommended).  
    - Set Guild ID and Channel ID to your target Discord server and channel (e.g., `#general`).  
    - Configure message content using the parsed AI output fields with markdown and emojis, exactly as in the example.  
    - Connect input from `Extract Keywords`.

11. **Configure Credentials**
    - Set up credentials for:  
      - NewsAPI (HTTP Header Auth with API key)  
      - Google Gemini API (Google Palm API key)  
      - Discord Bot API Token

12. **Test the Workflow**
    - Manually trigger the workflow or wait for scheduled trigger to verify full end-to-end execution.  
    - Monitor logs for errors in API calls, AI parsing, or Discord posting.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Automated daily at 8AM; timing adjustable to user preference for different report cadences (e.g., 12H or weekly).                          | Sticky Note on `8AM Everyday` node                                  |
| CoinGecko API provides free, no-auth market data for BTC, ETH, SOL including pricing and volume metrics.                                  | Sticky Note1                                                       |
| NewsAPI requires an API key and supplies recent crypto news filtered by keywords for sentiment analysis.                                   | Sticky Note1                                                       |
| Data merging code filters news articles by coin symbols and limits to 5 articles per asset for AI token efficiency.                       | Sticky Note2                                                       |
| AI analysis uses Google Gemini 1.5 Pro for professional financial insights with structured JSON output.                                    | Sticky Note4                                                       |
| Discord message formatting includes emojis and markdown for readability in chat channels.                                                  | Sticky Note5                                                       |
| Final step posts the AI-generated summary to Discord with bot token authentication; default channel is #daily-market-summary.              | Sticky Note6                                                       |
| Workflow inspired by AfK Crypto project; join their Discord for community and support: https://discord.com/invite/v4DgTEUUJJ               | Sticky Note7                                                       |
| For extended functionality, consider adding CoinMarketCap or Binance API, Twitter API for social sentiment, or archiving summaries in Sheets or Notion. | Sticky Note7                                                       |

---

**Disclaimer:** The text provided is based exclusively on an automated workflow built with n8n, respecting all content policies and containing no illegal or offensive elements. All data processed is legal and publicly available.