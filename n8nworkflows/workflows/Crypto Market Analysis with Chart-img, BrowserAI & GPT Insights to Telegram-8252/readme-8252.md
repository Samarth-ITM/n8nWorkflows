Crypto Market Analysis with Chart-img, BrowserAI & GPT Insights to Telegram

https://n8nworkflows.xyz/workflows/crypto-market-analysis-with-chart-img--browserai---gpt-insights-to-telegram-8252


# Crypto Market Analysis with Chart-img, BrowserAI & GPT Insights to Telegram

### 1. Workflow Overview

This workflow automates a twice-daily cryptocurrency market analysis focused on four major coins: BTC, ETH, SOL, and XRP. It combines visual chart data and web-scraped news to generate concise, AI-driven summaries, delivering them via Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling & Date Calculation**: Triggers the workflow twice daily and computes yesterday's and today's dates in multiple formats for API queries.
- **1.2 Chart Image Retrieval & Conversion**: Fetches 24-hour trading charts (candlestick style) for each coin from Chart-img API and converts the images to base64 strings for AI analysis.
- **1.3 BrowserAI Web Scraping Task**: Creates and monitors a BrowserAI task to scrape and summarize recent news about the coins, with built-in retry logic.
- **1.4 AI Graph Analysis**: Uses an AI agent to analyze the base64-encoded charts, producing simple 40-word descriptions for each coin's price action.
- **1.5 AI Crypto Summarization**: Combines the graph descriptions with the web summary into a boosted, easy-to-understand 60-word summary per coin.
- **1.6 Delivery**: Sends the final summarized information to a Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling & Date Calculation

- **Overview:**  
  This block triggers the workflow twice a day (at 1 AM and 1 PM UTC) and computes formatted date strings for yesterday and today, used for querying chart and news data.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get yesterday's date

- **Node Details:**

  - **Schedule Trigger**  
    - *Type & Role:* Schedule trigger node; initiates workflow at specified times.  
    - *Configuration:* Cron expression `0 1,13 * * *` (1 AM and 1 PM UTC daily).  
    - *Input/Output:* No input; outputs timestamp.  
    - *Potential Failures:* Cron misconfiguration or n8n scheduler issues.

  - **Get yesterday's date**  
    - *Type & Role:* Code node; calculates formatted date strings for yesterday and today.  
    - *Configuration:* JavaScript computes:  
      - `formattedToday` (e.g., 09-January-2024)  
      - `isoToday` (e.g., 2024-01-09)  
      - `formattedYesterday` (e.g., 08-January-2024)  
      - `isoYesterday` (e.g., 2024-01-08)  
    - *Input:* Timestamp from Schedule Trigger.  
    - *Output:* JSON with all four date strings.  
    - *Edge Cases:* Handles date boundaries correctly; assumes valid timestamp input.

---

#### 1.2 Chart Image Retrieval & Conversion

- **Overview:**  
  Fetches candlestick chart images for BTC, ETH, SOL, XRP from Chart-img API covering the previous 24h, then converts the returned binary images to base64 strings for AI consumption.

- **Nodes Involved:**  
  - BTCUSD chart  
  - ETHUSD chart  
  - SOLUSD chart  
  - XRPUSD chart  
  - BTCUSD image converter  
  - ETHUSD image converter  
  - SOLUSD image converter  
  - XRPUSD image converter  
  - Merge  
  - Merge1

- **Node Details:**

  - **BTCUSD chart, ETHUSD chart, SOLUSD chart, XRPUSD chart**  
    - *Type & Role:* HTTP Request nodes to Chart-img API; request 1h interval candlestick charts for each coin.  
    - *Configuration:*  
      - URL: `https://api.chart-img.com/v2/tradingview/advanced-chart`  
      - Method: POST  
      - Auth: HTTP Bearer with Chart-img API key credential  
      - JSON body specifies symbol (e.g., `BINANCE:BTCUSD`), time range from `isoYesterday` to `isoToday` (UTC midnight to midnight), 800x600 px, dark theme, candle style.  
    - *Input:* Date parameters from `Get yesterday's date` node.  
    - *Output:* Binary image data in PNG format.  
    - *Edge Cases:* API rate limits, auth failures, network issues, invalid date ranges.

  - **BTCUSD image converter, ETHUSD image converter, SOLUSD image converter, XRPUSD image converter**  
    - *Type & Role:* Code nodes; convert binary image data from Chart-img into base64 strings.  
    - *Configuration:* JavaScript extracts `items[0].binary.data` and outputs as base64 JSON property (e.g., `base64BTC`).  
    - *Input:* Binary image from corresponding chart node.  
    - *Output:* JSON object with base64 string keyed by coin symbol.  
    - *Edge Cases:* Missing or malformed binary data would cause errors.

  - **Merge & Merge1**  
    - *Type & Role:* Merge nodes combining base64 strings from multiple coins.  
    - *Configuration:* Mode `combine`, combining all inputs into one dataset.  
    - *Input:* Base64 converters for BTC & ETH (Merge), SOL & XRP (Merge1).  
    - *Output:* Combined JSON with base64 strings for all coins.  
    - *Edge Cases:* Input missing any base64 string causes incomplete data.

---

#### 1.3 BrowserAI Web Scraping Task

- **Overview:**  
  Creates a web scraping task on BrowserAI to gather recent news summaries about BTC, ETH, SOL, and XRP, then monitors task status with retries until completion.

- **Nodes Involved:**  
  - Create BrowserAI task  
  - Get task results  
  - Check if finalized  
  - Wait if not finished

- **Node Details:**

  - **Create BrowserAI task**  
    - *Type & Role:* HTTP Request node; submits a new web scraping task to BrowserAI.  
    - *Configuration:*  
      - Method: POST to `https://browser.ai/api/v1/tasks`  
      - Auth: HTTP Bearer with BrowserAI API key credential  
      - JSON body specifies:  
        - `geoLocation`: US  
        - `project`: Project_1  
        - `type`: crawler_automation  
        - `inspect`: true  
        - `instructions`: Request to summarize latest crypto news on BTC, ETH, SOL, XRP up to `formattedYesterday`.  
    - *Input:* Date strings from `Get yesterday's date`.  
    - *Output:* JSON with created task's `executionId`.  
    - *Edge Cases:* API failures, auth errors, invalid instructions.

  - **Get task results**  
    - *Type & Role:* HTTP Request node; polls BrowserAI for task status and results by `executionId`.  
    - *Configuration:* GET request to `https://browser.ai/api/v1/tasks/{{ $json.executionId }}` with auth.  
    - *Input:* `executionId` from `Create BrowserAI task` or after wait.  
    - *Output:* Task status and results JSON.  
    - *Edge Cases:* Network timeouts, API errors, invalid executionId.

  - **Check if finalized**  
    - *Type & Role:* Switch node; routes workflow depending on task status.  
    - *Configuration:* Checks `$json.status` for values:  
      - `finalized` (proceed)  
      - `running` or `queued` (wait)  
      - `failed` or `stopped` (recreate task)  
    - *Input:* Status from `Get task results`.  
    - *Output:* Directs flow to next nodes accordingly.  
    - *Edge Cases:* Unexpected status values or missing status.

  - **Wait if not finished**  
    - *Type & Role:* Wait node; pauses workflow 45 seconds before polling again.  
    - *Configuration:* Wait time of 45 seconds.  
    - *Input:* Routed from `Check if finalized` for running/queued status.  
    - *Output:* Triggers `Get task results` again.  
    - *Edge Cases:* Workflow timeouts if task takes too long.

---

#### 1.4 AI Graph Analysis

- **Overview:**  
  Analyzes the base64-encoded charts with an AI agent to produce simple, 40-word descriptions highlighting trends and significant moves for each coin.

- **Nodes Involved:**  
  - AI graph analyzer  
  - Structured Output Parser2  
  - OpenRouter Chat Model5

- **Node Details:**

  - **AI graph analyzer**  
    - *Type & Role:* Langchain AI agent node using OpenRouter; processes base64 images with prompt to describe charts.  
    - *Configuration:*  
      - Prompt includes base64 strings for BTC, ETH, SOL, XRP charts.  
      - System message instructs generating 40-word descriptions per coin in a specific JSON format.  
      - Output parser enabled with JSON schema example.  
    - *Input:* Combined base64 images from `Merge all graphs`.  
    - *Output:* JSON descriptions per coin.  
    - *Edge Cases:* AI response malformed JSON, timeout, or API errors.

  - **Structured Output Parser2**  
    - *Type & Role:* Parses AI output ensuring structured JSON as per schema.  
    - *Input:* AI output from `OpenRouter Chat Model5`.  
    - *Output:* Clean structured JSON with descriptions.  
    - *Edge Cases:* Parsing failures if AI output is invalid.

  - **OpenRouter Chat Model5**  
    - *Type & Role:* Underlying AI language model node connected to `AI graph analyzer`.  
    - *Credentials:* OpenRouter API key.  
    - *Edge Cases:* API limits, network errors.

---

#### 1.5 AI Crypto Summarization

- **Overview:**  
  Combines graph descriptions with BrowserAI web summaries to generate enhanced, user-friendly 60-word summaries for each coin.

- **Nodes Involved:**  
  - Merge graphs and BrowserAI  
  - AI crypto summarizer  
  - Structured Output Parser  
  - OpenRouter Chat Model6

- **Node Details:**

  - **Merge graphs and BrowserAI**  
    - *Type & Role:* Merge node; combines AI graph analysis output and BrowserAI web summary.  
    - *Configuration:* Combine mode, all inputs merged.  
    - *Input:* Output from graph analysis and BrowserAI summary.  
    - *Output:* Combined data structure.  
    - *Edge Cases:* Missing input data leads to incomplete summaries.

  - **AI crypto summarizer**  
    - *Type & Role:* Langchain AI agent node; synthesizes graph descriptions and web summaries into concise, easy-to-understand summaries per coin.  
    - *Configuration:*  
      - Prompt includes graph descriptions and web summary.  
      - System message instructs combining information into under 60 words per coin.  
      - Output parser enabled with JSON schema example.  
    - *Input:* Merged data from previous node.  
    - *Output:* JSON with enhanced information per coin.  
    - *Edge Cases:* AI output parsing failures, API errors.

  - **Structured Output Parser**  
    - *Type & Role:* Parses AI summarizer output into structured JSON format.  
    - *Input:* AI output from `OpenRouter Chat Model6`.  
    - *Output:* Final structured summary JSON per coin.  
    - *Edge Cases:* Parsing errors on malformed AI output.

  - **OpenRouter Chat Model6**  
    - *Type & Role:* AI model node connected to summarizer.  
    - *Credentials:* OpenRouter API key.  
    - *Edge Cases:* Same as above.

---

#### 1.6 Delivery

- **Overview:**  
  Sends the final crypto market update text message to a specified Telegram chat.

- **Nodes Involved:**  
  - Send a text message

- **Node Details:**

  - **Send a text message**  
    - *Type & Role:* Telegram node; sends a formatted text message with the summaries.  
    - *Configuration:*  
      - Text built using expressions interpolating the date and coin information from the AI summarizer output.  
      - Chat ID set to the user’s Telegram chat.  
      - Attribution disabled.  
    - *Input:* Final summaries from `AI crypto summarizer`.  
    - *Output:* Message sent confirmation.  
    - *Edge Cases:* Invalid chat ID, Telegram API errors, message length limits.

---

### 3. Summary Table

| Node Name               | Node Type                               | Functional Role                     | Input Node(s)                         | Output Node(s)                        | Sticky Note                                                                                                                                                   |
|-------------------------|---------------------------------------|-----------------------------------|-------------------------------------|-------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger                      | Workflow scheduler                 | —                                   | Get yesterday's date                 | Run the analyst twice daily at 8AM and 8PM                                                                                                                   |
| Get yesterday's date    | Code                                 | Date formatting                   | Schedule Trigger                    | BTCUSD chart, ETHUSD chart, SOLUSD chart, XRPUSD chart, Create BrowserAI task |                                                                                                                                                               |
| BTCUSD chart            | HTTP Request                         | Fetch BTC 24h chart               | Get yesterday's date                | BTCUSD image converter              | Utilize Chart-img API to capture coin graphs and convert to base64                                                                                           |
| ETHUSD chart            | HTTP Request                         | Fetch ETH 24h chart               | Get yesterday's date                | ETHUSD image converter              | Utilize Chart-img API to capture coin graphs and convert to base64                                                                                           |
| SOLUSD chart            | HTTP Request                         | Fetch SOL 24h chart               | Get yesterday's date                | SOLUSD image converter              | Utilize Chart-img API to capture coin graphs and convert to base64                                                                                           |
| XRPUSD chart            | HTTP Request                         | Fetch XRP 24h chart               | Get yesterday's date                | XRPUSD image converter              | Utilize Chart-img API to capture coin graphs and convert to base64                                                                                           |
| BTCUSD image converter  | Code                                 | Convert BTC image binary to base64 | BTCUSD chart                      | Merge                               | Utilize Chart-img API to capture coin graphs and convert to base64                                                                                           |
| ETHUSD image converter  | Code                                 | Convert ETH image binary to base64 | ETHUSD chart                      | Merge                               | Utilize Chart-img API to capture coin graphs and convert to base64                                                                                           |
| SOLUSD image converter  | Code                                 | Convert SOL image binary to base64 | SOLUSD chart                      | Merge1                              | Utilize Chart-img API to capture coin graphs and convert to base64                                                                                           |
| XRPUSD image converter  | Code                                 | Convert XRP image binary to base64 | XRPUSD chart                      | Merge1                              | Utilize Chart-img API to capture coin graphs and convert to base64                                                                                           |
| Merge                   | Merge                                | Combine BTC & ETH base64 images   | BTCUSD image converter, ETHUSD image converter | AI graph analyzer                   | Review and merge all Chart-img graph analyses into one cohesive message                                                                                      |
| Merge1                  | Merge                                | Combine SOL & XRP base64 images   | SOLUSD image converter, XRPUSD image converter | Merge all graphs                   | Review and merge all Chart-img graph analyses into one cohesive message                                                                                      |
| Merge all graphs        | Merge                                | Combine all four coin base64 images | Merge, Merge1                    | AI graph analyzer                   | Review and merge all Chart-img graph analyses into one cohesive message                                                                                      |
| Create BrowserAI task   | HTTP Request                         | Submit web scraping task          | Get yesterday's date                | Get task results                   | Leverage BrowserAI's web scraping to gather essential coin info from last 24h; auto relaunch on failure                                                     |
| Get task results        | HTTP Request                         | Poll BrowserAI task status        | Create BrowserAI task, Wait if not finished | Check if finalized                | Leverage BrowserAI's web scraping to gather essential coin info from last 24h; auto relaunch on failure                                                     |
| Check if finalized      | Switch                              | Route based on BrowserAI task status | Get task results                 | Merge graphs and BrowserAI, Wait if not finished, Create BrowserAI task | WARNING: This template is for personal use only; no financial advice                                                                                         |
| Wait if not finished    | Wait                                | Pause workflow before retry       | Check if finalized                 | Get task results                   | WARNING: This template is for personal use only; no financial advice                                                                                         |
| AI graph analyzer       | Langchain Agent                     | Analyze base64 charts             | Merge all graphs                   | Merge graphs and BrowserAI         | Merge the last 24-hour analyses from both Chart-img and BrowserAI for a comprehensive summary                                                               |
| Structured Output Parser2 | Langchain Output Parser             | Parse AI graph analysis output    | OpenRouter Chat Model5             | AI graph analyzer                  | Merge the last 24-hour analyses from both Chart-img and BrowserAI for a comprehensive summary                                                               |
| OpenRouter Chat Model5  | Langchain LM Chat                   | AI model for graph analysis       | AI graph analyzer                 | Structured Output Parser2          | Merge the last 24-hour analyses from both Chart-img and BrowserAI for a comprehensive summary                                                               |
| Merge graphs and BrowserAI | Merge                             | Combine graph analysis & web summary | AI graph analyzer, Check if finalized | AI crypto summarizer             | Merge the last 24-hour analyses from both Chart-img and BrowserAI for a comprehensive summary                                                               |
| AI crypto summarizer    | Langchain Agent                     | Generate boosted summaries        | Merge graphs and BrowserAI         | Send a text message                | Send the results to your Telegram chat; alternatives include WhatsApp, email, etc.                                                                          |
| Structured Output Parser | Langchain Output Parser             | Parse AI crypto summarizer output | OpenRouter Chat Model6             | AI crypto summarizer               | Send the results to your Telegram chat; alternatives include WhatsApp, email, etc.                                                                          |
| OpenRouter Chat Model6  | Langchain LM Chat                   | AI model for crypto summarization | AI crypto summarizer               | Structured Output Parser           | Send the results to your Telegram chat; alternatives include WhatsApp, email, etc.                                                                          |
| Send a text message     | Telegram                           | Deliver final summary message     | AI crypto summarizer               | —                                 | Send the results to your Telegram chat; alternatives include WhatsApp, email, etc.                                                                          |
| Sticky Note             | Sticky Note                        | Informational notes               | —                                 | —                                 | Merge the last 24-hour analyses from both Chart-img and BrowserAI for a comprehensive summary                                                              |
| Sticky Note1            | Sticky Note                        | Informational notes               | —                                 | —                                 | Review and merge all Chart-img graph analyses into one cohesive message                                                                                      |
| Sticky Note2            | Sticky Note                        | Informational notes               | —                                 | —                                 | Utilize Chart-img's API to capture coin graphs and convert images to base64                                                                                  |
| Sticky Note3            | Sticky Note                        | Informational notes               | —                                 | —                                 | Leverage BrowserAI's web scraping capabilities to find and summarize coin info; auto relaunch failed tasks                                                  |
| Sticky Note4            | Sticky Note                        | Informational notes               | —                                 | —                                 | Run the analyst twice daily at 8AM and 8PM                                                                                                                  |
| Sticky Note5            | Sticky Note                        | Informational notes               | —                                 | —                                 | Send the results to your Telegram chat; alternatives include WhatsApp, email, etc.                                                                         |
| Sticky Note6            | Sticky Note                        | Informational notes               | —                                 | —                                 | WARNING: This template is intended for personal use only and does not constitute financial advice. Actions are user's responsibility.                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Type: Schedule Trigger  
   - Configuration: Cron expression `0 1,13 * * *` (runs at 1 AM and 1 PM UTC daily).  
   - No credentials needed.

2. **Add Code Node "Get yesterday's date"**  
   - Type: Code (JavaScript)  
   - Input: From Schedule Trigger (timestamp)  
   - Function: Compute `formattedToday`, `isoToday`, `formattedYesterday`, `isoYesterday` as detailed in the code block.  
   - Output: JSON with the four date strings.

3. **Add HTTP Request Nodes for Chart-img API (4 nodes)**  
   - Type: HTTP Request  
   - Name accordingly: BTCUSD chart, ETHUSD chart, SOLUSD chart, XRPUSD chart  
   - Method: POST  
   - URL: `https://api.chart-img.com/v2/tradingview/advanced-chart`  
   - Authentication: HTTP Bearer Auth with Chart-img API key credential  
   - JSON Body: Set symbol (e.g., "BINANCE:BTCUSD"), interval "1h", width 800, height 600, style "candle", theme "dark", timezone "Etc/UTC", range from `isoYesterday` + "T00:00:00.000Z" to `isoToday` + "T00:00:00.000Z" using expressions from "Get yesterday's date".  
   - Output: Binary image data (PNG).

4. **Create Code Nodes to Convert Image Binary to Base64 (4 nodes)**  
   - Type: Code  
   - Names: BTCUSD image converter, ETHUSD image converter, SOLUSD image converter, XRPUSD image converter  
   - Code: Extract binary data from previous node, output base64 string keyed by coin symbol.  
   - Input: Corresponding chart HTTP Request node.

5. **Create Two Merge Nodes to Combine Base64 Images**  
   - Merge: Combine BTC and ETH base64 JSON outputs (mode: combine all).  
   - Merge1: Combine SOL and XRP base64 JSON outputs (mode: combine all).

6. **Create Merge Node "Merge all graphs"**  
   - Combine outputs from Merge and Merge1 nodes into one JSON containing all four base64 strings.

7. **Create HTTP Request Node to Submit BrowserAI Task**  
   - Method: POST  
   - URL: `https://browser.ai/api/v1/tasks`  
   - Auth: HTTP Bearer Auth with BrowserAI credentials  
   - JSON Body: Define task with `"geoLocation": {"country": "us"}`, `"project": "Project_1"`, `"type": "crawler_automation"`, `"inspect": true`, and `"instructions"` asking to summarize latest crypto news on BTC, ETH, SOL, XRP up to `formattedYesterday`.  
   - Input: From "Get yesterday's date".

8. **Create HTTP Request Node "Get task results"**  
   - Method: GET  
   - URL: `https://browser.ai/api/v1/tasks/{{ $json.executionId }}` (dynamic executionId from previous task)  
   - Auth: HTTP Bearer Auth (BrowserAI).

9. **Add Switch Node "Check if finalized"**  
   - Input: Task status JSON from "Get task results"  
   - Conditions:  
     - If status == "finalized" → proceed  
     - If status == "running" or "queued" → go to Wait node  
     - If status == "failed" or "stopped" → restart BrowserAI task node  
   - Output: Multiple branches accordingly.

10. **Add Wait Node "Wait if not finished"**  
    - Wait 45 seconds before retrying to get task results.

11. **Create Langchain AI Agent Node "AI graph analyzer"**  
    - Input: Base64 image JSON from "Merge all graphs"  
    - Prompt: Include base64 images for BTC, ETH, SOL, XRP; request 40-word descriptions per coin.  
    - System message: Instructions for AI to analyze charts and output JSON with descriptions.  
    - Enable structured output parsing with JSON schema.

12. **Add Structured Output Parser for "AI graph analyzer"**  
    - To parse AI output into structured JSON.

13. **Create Merge Node "Merge graphs and BrowserAI"**  
    - Combine output from "AI graph analyzer" and BrowserAI web summary (from finalized BrowserAI task).

14. **Create Langchain AI Agent Node "AI crypto summarizer"**  
    - Input: Merged graph descriptions and web summary  
    - Prompt: Combine both inputs to produce boosted, easy-to-understand summaries under 60 words per coin.  
    - System message: Instructions for synthesis and JSON output.  
    - Enable structured output parsing.

15. **Add Structured Output Parser for "AI crypto summarizer"**  
    - Parses enhanced summaries.

16. **Create Telegram Node "Send a text message"**  
    - Configure with Telegram API credentials (bot token).  
    - Chat ID: Your target Telegram chat.  
    - Text: Use expressions to interpolate date and AI summaries for BTC, ETH, SOL, XRP.  
    - Disable attribution.

17. **Connect Nodes as per dependencies:**  
    - Schedule Trigger → Get yesterday's date → all chart HTTP requests and Create BrowserAI task  
    - Each chart HTTP request → corresponding image converter → Merge/Merge1 → Merge all graphs → AI graph analyzer  
    - Create BrowserAI task → Get task results → Check if finalized → Wait if not finished (loop) or proceed  
    - AI graph analyzer output + BrowserAI summary → Merge graphs and BrowserAI → AI crypto summarizer → Send a text message

18. **Credentials Setup:**  
    - Chart-img API key for HTTP Request nodes (Bearer token)  
    - BrowserAI API key for BrowserAI task nodes (Bearer token)  
    - OpenRouter API key for Langchain AI nodes  
    - Telegram bot token for Telegram node

---

### 5. General Notes & Resources

| Note Content                                                                                                                                          | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Utilize Chart-img API to capture coin graphs and convert images to base64 strings for AI input.                                                       | https://chart-img.com/                                                                              |
| Leverage BrowserAI for web scraping and automatic summarization of cryptocurrency news.                                                              | https://browser.ai/                                                                                 |
| Run the analyst twice daily at 8AM and 8PM (adjust cron trigger accordingly).                                                                          | Workflow scheduling note                                                                             |
| WARNING: This template is intended for personal use only and does not constitute financial advice. Users are responsible for their own actions.       | Important legal disclaimer                                                                           |
| Send results to Telegram chat; alternatives like WhatsApp or email can be integrated by replacing the delivery node.                                 | Messaging platform flexibility                                                                      |
| Chart-img API requires an API key (free tier used here).                                                                                              | https://chart-img.com/account/api                                                                    |
| BrowserAI API key required; task is automatically relaunched on failure or stoppage to ensure completion.                                            | https://browser.ai/dashboard/page/account/tab/api_key                                                |
| AI uses OpenRouter API for language models; ensure API keys and rate limits are respected.                                                           | https://openrouter.ai/                                                                               |
| Prompt engineering includes clear instructions and structured JSON schema for AI outputs to ensure machine-readable results.                        | Critical for robust AI integration                                                                  |

---

This documentation provides a detailed and methodical reference for understanding, modifying, and recreating the "Crypto Market Analysis with Chart-img, BrowserAI & GPT Insights to Telegram" workflow in n8n.