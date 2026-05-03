Daily Gemini-Powered Global Trend Analysis with GDELT, NewsAPI & Discord

https://n8nworkflows.xyz/workflows/daily-gemini-powered-global-trend-analysis-with-gdelt--newsapi---discord-10093


# Daily Gemini-Powered Global Trend Analysis with GDELT, NewsAPI & Discord

---
### 1. Workflow Overview

This workflow automates the collection, analysis, and dissemination of global trend data across multiple real-time news and event sources with AI-powered insights. It is designed for daily or more frequent monitoring of emerging global trends in technology, finance, and innovation, with a focus on cryptocurrency and related topics.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Data Collection:** Runs every 6 hours to trigger data fetching from three APIs: GDELT Global Events, Hacker News, and NewsAPI.
- **1.2 Data Aggregation & Normalization:** Merges and formats heterogeneous raw data from the three sources into a unified and consistent schema.
- **1.3 AI Trend Analysis:** Utilizes a Google Gemini-powered AI agent to extract structured insights—emerging topics, summaries, regional highlights, and notable mentions—from the normalized data.
- **1.4 Output Formatting & Delivery:** Parses the AI output into a user-friendly Discord message format and posts it automatically to a specified Discord channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Data Collection

- **Overview:**  
  This block triggers the workflow on a fixed schedule and collects raw trend data from three diverse APIs to cover a broad spectrum of global events and discussions related to technology and crypto.

- **Nodes Involved:**  
  - Every 6 Hours (Schedule Trigger)  
  - GDELT Global Event (HTTP Request)  
  - Hacker News (Free) (HTTP Request)  
  - NewsAPI (Free) (HTTP Request)  

- **Node Details:**

  - **Every 6 Hours (Schedule Trigger)**  
    - Type: Schedule Trigger  
    - Configuration: Runs daily at 8:00 AM UTC (configurable interval)  
    - Input: None (trigger node)  
    - Output: Triggers downstream API requests  
    - Edge Cases: Misconfiguration may cause no trigger or excess runs  
    - Notes: Can be adjusted to 12h or 24h intervals for slower update cycles  

  - **GDELT Global Event (HTTP Request)**  
    - Type: HTTP Request  
    - Configuration: GET request to `https://api.gdeltproject.org/api/v2/doc/doc?query=crypto&format=json`  
    - Authentication: None required  
    - Input: Trigger from schedule  
    - Output: JSON response containing global media event articles related to "crypto"  
    - Edge Cases: API downtime, network errors, rate limiting by GDELT  
    - Notes: Captures signals from global media, politics, and social activity  

  - **Hacker News (Free) (HTTP Request)**  
    - Type: HTTP Request  
    - Configuration: GET request to `https://hn.algolia.com/api/v1/search?query=startup%20OR%20trend&tags=story&hitsPerPage=10`  
    - Authentication: None required  
    - Input: Trigger from schedule  
    - Output: JSON response with trending Hacker News stories on startups and trends  
    - Edge Cases: API downtime, network errors  
    - Notes: Focuses on early tech sentiment  

  - **NewsAPI (Free) (HTTP Request)**  
    - Type: HTTP Request  
    - Configuration: GET request to `https://newsapi.org/v2/everything` with query parameters filtering for crypto, blockchain, AI, web3, etc., limited to English language and 10 articles sorted by date  
    - Authentication: NewsAPI key via HTTP Header Auth credentials  
    - Input: Trigger from schedule  
    - Output: JSON response with global news articles  
    - Edge Cases: API key expiration, quota limits, network errors  

---

#### 2.2 Data Aggregation & Normalization

- **Overview:**  
  This block merges and normalizes the disparate API responses into a single coherent dataset, standardizing fields such as title, URL, source, publication date, language, and country for uniform downstream AI processing.

- **Nodes Involved:**  
  - Merge  
  - Format Trend Data (Code)  

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Configuration: Combines three inputs from GDELT, Hacker News, and NewsAPI in "append" mode (numberInputs=3)  
    - Input: Outputs from three HTTP Request nodes  
    - Output: Single merged array containing all raw articles  
    - Edge Cases: If any input is empty or fails, resulting dataset may be incomplete  

  - **Format Trend Data (Code)**  
    - Type: Code (JavaScript)  
    - Configuration: Custom JS code that:  
      - Iterates over the merged inputs  
      - Extracts and normalizes article fields from each source  
      - Converts GDELT date format (e.g., 20250909T101500Z) to ISO 8601 standard  
      - Fills missing fields with defaults or fallbacks  
      - Sorts all articles by published date descending  
      - Outputs a unified JSON object with total articles, extraction timestamp, and standardized articles array  
    - Key Expressions: Uses regex for date parsing, array methods for merging and sorting  
    - Input: Merged raw API data  
    - Output: Cleaned, structured, and sorted dataset for AI consumption  
    - Edge Cases: Malformed dates, missing fields handled with fallbacks; extreme malformed data could break processing  

---

#### 2.3 AI Trend Analysis

- **Overview:**  
  The core analytical block invokes an AI agent powered by Google Gemini to synthesize the normalized dataset into key insights: top emerging topics, a concise global trend summary, regional highlights, and notable article mentions.

- **Nodes Involved:**  
  - AI Agent (LangChain Agent)  
  - Google Gemini Chat Model (Language Model)  
  - Structured Output Parser (LangChain Structured Output Parser)  

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent  
    - Configuration:  
      - Prompt includes instructions to analyze aggregated trend data and generate:  
        1. Top 5 emerging topics from keywords/entities  
        2. 100-150 word concise global trend summary  
        3. 2-3 regional insights  
        4. 3 notable article mentions with titles and sources  
      - Uses system message to enforce clarity, synthesis, and structured output for automated alerts  
      - Output parser enabled to validate AI response against a defined JSON schema  
      - Input: Cleaned articles array from Format Trend Data  
      - Output: Structured JSON with trend insights  
    - Edge Cases: AI model errors, invalid or malformed output (mitigated by structured parser)  

  - **Google Gemini Chat Model**  
    - Type: LangChain Language Model (Google Gemini)  
    - Configuration: Connected to Google Gemini API credentials  
    - Input: Text prompt from AI Agent node  
    - Output: AI-generated structured response  
    - Edge Cases: API unavailability, authentication errors, rate limits  

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Configuration: JSON schema defining expected AI output fields:  
      - emerging_topics (array of strings)  
      - trend_summary (string)  
      - regional_insights (array of strings)  
      - notable_mentions (array of objects with title and source)  
    - Input: Raw AI textual response  
    - Output: Parsed and validated structured JSON for downstream formatting  
    - Edge Cases: Parsing failures if AI output deviates from schema; schema helps prevent malformed data  

---

#### 2.4 Output Formatting & Delivery

- **Overview:**  
  This final block converts the AI-generated insights into a formatted Discord message with emojis and section titles, then posts this message automatically to a designated Discord channel via a bot token.

- **Nodes Involved:**  
  - Parse AI Output (Code)  
  - Send a message (Discord)  

- **Node Details:**

  - **Parse AI Output (Code)**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Extracts the structured AI output fields from `$json.output`  
      - Constructs a multiline Discord message string with sections:  
        - 🧭 Global Trend Summary  
        - 🌍 Emerging Topics  
        - 📍 Regional Insights  
        - 📰 Notable Mentions (with article titles italicized and sources shown)  
      - Adds emojis and bullet points for readability  
    - Input: Parsed AI output JSON  
    - Output: Single formatted string assigned to `content` property  
    - Edge Cases: Empty or missing AI fields handled gracefully by JS template  

  - **Send a message (Discord)**  
    - Type: Discord Node (Bot)  
    - Configuration:  
      - Uses a Discord Bot Token credential  
      - Posts message to a predefined guild (server) and channel (default #general)  
      - Sends the formatted message content from Parse AI Output node  
    - Input: Formatted message string  
    - Output: Confirmation of message delivery or error  
    - Edge Cases: Invalid bot token, permission issues, network errors, channel ID changes  

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                        | Input Node(s)               | Output Node(s)        | Sticky Note                                                                                                                                |
|-------------------------|----------------------------------|-------------------------------------|-----------------------------|-----------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Every 6 Hours           | Schedule Trigger                 | Initiates workflow every 6 hours    | None                        | Hacker News, NewsAPI, GDELT | Runs automatically every 6 hours (modifiable). Initiates data pulls from all three APIs. Adjust frequency as needed.                       |
| GDELT Global Event      | HTTP Request                    | Fetches global media event data      | Every 6 Hours               | Merge                 | Fetches real-time global event data from GDELT. No auth required.                                                                            |
| Hacker News (Free)      | HTTP Request                    | Fetches trending Hacker News stories | Every 6 Hours               | Merge                 | Pulls trending stories from Hacker News focusing on startups and innovation. No auth required.                                               |
| NewsAPI (Free)          | HTTP Request                    | Fetches global news headlines        | Every 6 Hours               | Merge                 | Fetches major news filtered by crypto and tech topics. Requires NewsAPI key.                                                                 |
| Merge                   | Merge                           | Combines API outputs into one stream | GDELT, Hacker News, NewsAPI | Format Trend Data      | Combines outputs from all APIs for normalization.                                                                                            |
| Format Trend Data       | Code                           | Normalizes and structures data       | Merge                      | AI Agent              | Cleans and formats raw data into standardized JSON schema for AI. Handles duplicates and timestamps.                                         |
| Google Gemini Chat Model| LangChain Language Model        | Provides LLM backend for AI Agent    | AI Agent                   | AI Agent              | Gemini AI engine powering trend analysis. Swap with other models as needed.                                                                 |
| Structured Output Parser| LangChain Output Parser         | Parses and validates AI output       | AI Agent                   | AI Agent              | Defines expected structured JSON output schema to prevent malformed AI responses.                                                           |
| AI Agent                | LangChain Agent                | Analyzes data and generates insights | Format Trend Data, Gemini, Parser | Parse AI Output         | Extracts top topics, summary, regional insights, notable mentions using AI analysis.                                                        |
| Parse AI Output         | Code                           | Formats AI output into Discord message | AI Agent                   | Send a message        | Transforms AI JSON into styled Discord message with emojis and sections.                                                                     |
| Send a message          | Discord Node                   | Posts message to Discord channel     | Parse AI Output            | None                  | Posts AI-curated global summary to Discord #general channel using bot token credential.                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Name: "Every 6 Hours"  
   - Type: Schedule Trigger  
   - Configure to run once daily at 08:00 UTC (adjust interval as desired)  

2. **Add HTTP Request Nodes to Fetch Data**  
   - **GDELT Global Event**  
     - Type: HTTP Request  
     - URL: `https://api.gdeltproject.org/api/v2/doc/doc?query=crypto&format=json`  
     - Method: GET  
     - No authentication required  
   - **Hacker News (Free)**  
     - Type: HTTP Request  
     - URL: `https://hn.algolia.com/api/v1/search?query=startup%20OR%20trend&tags=story&hitsPerPage=10`  
     - Method: GET  
     - No authentication required  
   - **NewsAPI (Free)**  
     - Type: HTTP Request  
     - URL: `https://newsapi.org/v2/everything`  
     - Method: GET  
     - Add Query Parameters:  
       - q: `crypto OR bitcoin OR ethereum OR solana OR defi OR nft OR blockchain OR web3 OR altcoin OR etf OR regulation`  
       - language: `en`  
       - sortBy: `publishedAt`  
       - pageSize: `10`  
     - Authentication: HTTP Header Auth using your NewsAPI key  

3. **Connect Schedule Trigger’s Output to all Three HTTP Request Nodes**  

4. **Add a Merge Node**  
   - Name: "Merge"  
   - Type: Merge  
   - Set Number of Inputs to 3  
   - Connect outputs of GDELT, Hacker News, and NewsAPI nodes to Merge inputs  

5. **Add a Code Node to Normalize Data**  
   - Name: "Format Trend Data"  
   - Type: Code  
   - Paste JavaScript logic to:  
     - Iterate over all incoming items  
     - For GDELT: parse and normalize dates, extract relevant fields  
     - For Hacker News: extract titles, URLs, timestamps  
     - For NewsAPI: extract titles, URLs, images, published dates  
     - Combine all into a uniform array, sort by date descending  
   - Connect Merge output to this node  

6. **Add LangChain Nodes for AI Analysis**  
   - **Google Gemini Chat Model**  
     - Type: LangChain Language Model  
     - Connect Google Palm API credentials (Google Gemini API)  
   - **Structured Output Parser**  
     - Type: LangChain Output Parser  
     - Define JSON schema with fields: emerging_topics, trend_summary, regional_insights, notable_mentions  
   - **AI Agent**  
     - Type: LangChain Agent  
     - Configure prompt to analyze aggregated articles and extract insights as per workflow description  
     - Connect Gemini model as languageModel and Structured Output Parser as outputParser  
     - Connect "Format Trend Data" output to AI Agent input  

7. **Add Code Node for Formatting AI Output**  
   - Name: "Parse AI Output"  
   - Type: Code  
   - Write JS code to construct a Markdown/Discord-friendly message with emojis and sections, extracting fields from AI output JSON  
   - Connect AI Agent output to this node  

8. **Add Discord Node for Message Posting**  
   - Name: "Send a message"  
   - Type: Discord Node  
   - Configure Discord Bot Token credential  
   - Set Guild ID to your Discord server ID  
   - Set Channel ID to the target channel (e.g., #general)  
   - Set the message content to the expression referencing "Parse AI Output" node's `content` field  
   - Connect "Parse AI Output" output to this node  

9. **Validate all connections and credentials**  
   - Ensure NewsAPI key and Google Gemini API credentials are correctly configured  
   - Ensure Discord Bot has permissions to send messages in the chosen channel  

10. **Test the workflow manually or wait for scheduled trigger**  
    - Check for successful data retrieval, AI insight generation, and Discord posting  

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The workflow’s core LLM is Google Gemini, but you can replace it with OpenAI or Anthropic models easily. | See "Global Trend Intelligence Agent" sticky note describing AI model flexibility.                        |
| Schedule trigger frequency is adjustable to 6, 12, or 24 hours based on desired update cadence.         | "Schedule Trigger" sticky note.                                                                          |
| The AI Agent prompt and output schema ensure consistent structured data for downstream use.            | "Global Trend Intelligence Agent" and "JSON Output Schema" sticky note.                                  |
| Discord node requires a bot token with send message permissions in the target server and channel.      | "Discord Auto-Poster" sticky note.                                                                       |
| NewsAPI requires an active API key with quota available; monitor to prevent data retrieval failures.    | "NewsAPI Global Headlines" sticky note.                                                                  |
| GDELT and Hacker News API do not require authentication but may have rate limits or occasional downtime. | "GDELT Global Media Feed" and "Hacker News Trending Insights" sticky notes.                              |
| For advanced customization, modify the AI prompt or add additional data sources as needed.             | Workflow is modular and supports extending data inputs or AI instructions.                               |
| This workflow is ideal for teams or communities tracking emerging global tech and crypto trends daily. | Can be integrated with Telegram or other platforms with minor adaptations.                               |

---

**Disclaimer:** The provided text is exclusively generated from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All processed data is public and legal information.