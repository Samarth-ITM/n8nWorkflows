Generate Crypto Trading Signals with Multi-Timeframe Data, News Sentiment & Gemini AI

https://n8nworkflows.xyz/workflows/generate-crypto-trading-signals-with-multi-timeframe-data--news-sentiment---gemini-ai-11366


# Generate Crypto Trading Signals with Multi-Timeframe Data, News Sentiment & Gemini AI

---

### 1. Workflow Overview

This workflow automates the generation of cryptocurrency trading signals by aggregating multi-timeframe market data, analyzing relevant news sentiment, and synthesizing insights through AI models (Google Gemini and a specialized Crypto Agent). It is designed for traders or automated systems that seek data-driven, sentiment-aware trading signals on crypto assets.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggered by a Telegram message, initiating the workflow.
- **1.2 Market Data Retrieval:** Fetches candlestick data at multiple timeframes (15 min, 1 hour, 1 day) via HTTP requests.
- **1.3 Data Preparation:** Edits and formats retrieved market data fields, then combines them.
- **1.4 News Gathering and Sentiment Analysis:** Retrieves crypto-related news, filters it, and analyzes sentiment using an AI model.
- **1.5 Signal Generation:** Merges market and news data, processes information with Google Gemini chat model and a specialized Crypto Agent to generate trading signals.
- **1.6 Output Dispatch:** Sends generated trading signals back via Telegram message.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Receives user input via Telegram to trigger the workflow.
- **Nodes Involved:** Telegram Trigger → Code in JavaScript
- **Node Details:**

  - **Telegram Trigger**  
    - Type: Trigger (Telegram)  
    - Role: Listens for incoming Telegram messages to start the workflow.  
    - Configuration: Default webhook setup, no parameters specified, uses webhook ID.  
    - Inputs: None (trigger node)  
    - Outputs: Passes Telegram message data downstream.  
    - Edge Cases: Telegram API rate limits, message format errors.

  - **Code in JavaScript**  
    - Type: Function (Code)  
    - Role: Processes Telegram input, prepares parameters for subsequent requests.  
    - Configuration: Empty parameters; likely contains JavaScript code (not shown) that extracts or formats input.  
    - Inputs: Telegram Trigger output  
    - Outputs: Triggers parallel HTTP requests for market data and news.  
    - Edge Cases: Expression errors if input format unexpected; missing or malformed Telegram messages.

#### 2.2 Market Data Retrieval

- **Overview:** Retrieves candlestick data at 15-minute, 1-hour, and 1-day intervals via HTTP requests.
- **Nodes Involved:** 15 min, 1 hour, 1 day
- **Node Details:**

  - All three are **HTTP Request** nodes configured to call respective APIs (likely exchange or market data providers).  
  - Inputs: Code in JavaScript node triggers them in parallel.  
  - Outputs: Raw candlestick data passed downstream.  
  - Configuration specifics (URLs, headers, params) are not detailed but must be set according to the chosen data provider’s API.  
  - Edge Cases: Network timeouts, API authentication errors, rate limits, malformed data responses.

#### 2.3 Data Preparation

- **Overview:** Edits and formats the raw candlestick data from each timeframe, then combines them into a consolidated dataset.
- **Nodes Involved:** Edit Fields, Edit Fields1, Edit Fields2 → Merge → Combine candlestick
- **Node Details:**

  - **Edit Fields (15 min), Edit Fields1 (1 hour), Edit Fields2 (1 day)**  
    - Type: Set node  
    - Role: Selects and formats relevant fields (e.g., open, close, high, low, volume) from raw HTTP responses.  
    - Inputs: Corresponding HTTP Request nodes.  
    - Outputs: Formatted datasets for merging.  
    - Edge Cases: Missing fields, empty data sets causing empty outputs.

  - **Merge**  
    - Type: Merge  
    - Role: Combines the three formatted datasets into one unified data set, preserving all inputs (likely using 'Merge By Index' or 'Merge By Key').  
    - Inputs: Outputs of the three Edit Fields nodes.  
    - Outputs: Combined candlestick data.  
    - Edge Cases: Mismatched data lengths, missing inputs.

  - **Combine candlestick**  
    - Type: Code (JavaScript)  
    - Role: Further processes/aggregates merged candlestick data, possibly creating features or indicators for AI input.  
    - Inputs: Merge node output.  
    - Outputs: Processed data forwarded for AI processing.  
    - Edge Cases: Code errors, unexpected data formats.

#### 2.4 News Gathering and Sentiment Analysis

- **Overview:** Fetches crypto news, filters relevant information, and analyzes sentiment using Google Gemini AI models.
- **Nodes Involved:** News → Filtering News → News sentiment Analyzer → Merge1
- **Node Details:**

  - **News**  
    - Type: HTTP Request  
    - Role: Fetches latest crypto-related news from a news API.  
    - Inputs: Code in JavaScript (parallel with market data fetch).  
    - Outputs: Raw news data.  
    - Edge Cases: API errors, empty results.

  - **Filtering News**  
    - Type: Code (JavaScript)  
    - Role: Filters and extracts relevant news items, possibly removing duplicates or irrelevant articles.  
    - Inputs: News node output.  
    - Outputs: Filtered news articles.  
    - Edge Cases: Logical errors in filter code, empty filtered results.

  - **News sentiment Analyzer**  
    - Type: Google Gemini AI (LangChain node)  
    - Role: Analyzes sentiment of filtered news articles using an AI language model.  
    - Inputs: Filtering News output.  
    - Outputs: Sentiment scores or summaries.  
    - Edge Cases: AI model errors, API limits, malformed input.

  - **Merge1**  
    - Type: Merge  
    - Role: Combines processed candlestick data (from Combine candlestick) and news sentiment results into one dataset for final AI analysis.  
    - Inputs: Combine candlestick and News sentiment Analyzer outputs.  
    - Outputs: Consolidated input for Crypto Agent.  
    - Edge Cases: Failed merges if any input missing.

#### 2.5 Signal Generation

- **Overview:** Uses Google Gemini Chat Model and a Crypto Agent AI to synthesize data into actionable crypto trading signals.
- **Nodes Involved:** Google Gemini Chat Model1 → Crypto Agent → Send a text message
- **Node Details:**

  - **Google Gemini Chat Model1**  
    - Type: Google Gemini AI Chat Model (LangChain)  
    - Role: Performs intermediate AI processing or summarization of inputs before passing to Crypto Agent.  
    - Inputs: Connected to Crypto Agent node as language model input.  
    - Outputs: Processed text or data for Crypto Agent.  
    - Edge Cases: API errors, input size limits.

  - **Crypto Agent**  
    - Type: LangChain Agent Node  
    - Role: Main AI agent that generates crypto trading signals based on combined market and news data.  
    - Inputs: Merge1 node output and Google Gemini Chat Model1 as language model.  
    - Outputs: Trading signals (text or structured data).  
    - Edge Cases: AI reasoning failures, execution errors.

#### 2.6 Output Dispatch

- **Overview:** Sends the generated trading signals back to the user via Telegram.
- **Nodes Involved:** Send a text message
- **Node Details:**

  - **Send a text message**  
    - Type: Telegram node (send message)  
    - Role: Delivers the AI-generated trading signals to Telegram user or group.  
    - Inputs: Crypto Agent output.  
    - Outputs: None (endpoint).  
    - Configuration: Uses Telegram credentials (OAuth2) and chat ID from trigger or static config.  
    - Edge Cases: Telegram API errors, message formatting issues.

---

### 3. Summary Table

| Node Name              | Node Type                             | Functional Role                               | Input Node(s)                     | Output Node(s)                     | Sticky Note                        |
|------------------------|-------------------------------------|-----------------------------------------------|----------------------------------|----------------------------------|----------------------------------|
| Telegram Trigger       | Telegram Trigger                    | Receive Telegram message to start workflow    | None                             | Code in JavaScript               |                                  |
| Code in JavaScript     | Code (JavaScript)                   | Process input, trigger data/news requests     | Telegram Trigger                 | 15 min, 1 hour, 1 day, News     |                                  |
| 15 min                 | HTTP Request                       | Fetch 15-minute candlestick data               | Code in JavaScript               | Edit Fields                     |                                  |
| 1 hour                 | HTTP Request                       | Fetch 1-hour candlestick data                   | Code in JavaScript               | Edit Fields1                    |                                  |
| 1 day                  | HTTP Request                       | Fetch 1-day candlestick data                    | Code in JavaScript               | Edit Fields2                    |                                  |
| Edit Fields            | Set                               | Format 15-min data                             | 15 min                         | Merge                          |                                  |
| Edit Fields1           | Set                               | Format 1-hour data                             | 1 hour                        | Merge                          |                                  |
| Edit Fields2           | Set                               | Format 1-day data                              | 1 day                         | Merge                          |                                  |
| Merge                  | Merge                             | Combine formatted multi-timeframe data        | Edit Fields, Edit Fields1, Edit Fields2 | Combine candlestick        |                                  |
| Combine candlestick    | Code (JavaScript)                  | Aggregate/prepare combined candlestick data   | Merge                         | Merge1                         |                                  |
| News                   | HTTP Request                      | Fetch crypto news                              | Code in JavaScript             | Filtering News                 |                                  |
| Filtering News         | Code (JavaScript)                 | Filter relevant news data                       | News                         | News sentiment Analyzer        |                                  |
| News sentiment Analyzer| Google Gemini AI (LangChain)      | Analyze sentiment of news                       | Filtering News               | Merge1                         |                                  |
| Merge1                 | Merge                            | Merge market data and news sentiment           | Combine candlestick, News sentiment Analyzer | Crypto Agent              |                                  |
| Google Gemini Chat Model1 | Google Gemini AI Chat Model       | AI intermediate processing for Crypto Agent   | — (used as LM for Crypto Agent) | Crypto Agent (languageModel input) |                                  |
| Crypto Agent           | LangChain Agent                   | Generate crypto trading signals                 | Merge1                        | Send a text message            |                                  |
| Send a text message    | Telegram                          | Send generated signals via Telegram             | Crypto Agent                  | None                          |                                  |
| Sticky Note            | Sticky Note                      | Comments or notes (various positions)          | None                         | None                          | Multiple sticky notes present but empty content |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Purpose: Start workflow on incoming Telegram messages  
   - Configuration: Set up webhook with Telegram bot credentials and enable webhook listening.

2. **Add Code in JavaScript Node:**  
   - Connect Telegram Trigger → Code in JavaScript  
   - Purpose: Parse Telegram message, prepare parameters for API calls  
   - Add JavaScript code to extract user input or parameters for market data and news requests.

3. **Add HTTP Request Nodes for Market Data:**  
   - Create three HTTP Request nodes named "15 min", "1 hour", and "1 day".  
   - Connect Code in JavaScript → each HTTP Request node in parallel.  
   - Configure endpoints, headers, query parameters to fetch candlestick data for the respective timeframes from your crypto data provider.

4. **Add Set Nodes to Edit Fields:**  
   - Create three Set nodes named "Edit Fields" (for 15 min), "Edit Fields1" (for 1 hour), and "Edit Fields2" (for 1 day).  
   - Connect each HTTP Request node to its corresponding Set node.  
   - Configure fields to retain only necessary data (e.g., open, close, high, low, volume, timestamp).

5. **Add Merge Node:**  
   - Create a Merge node named "Merge".  
   - Connect the three Set nodes ("Edit Fields", "Edit Fields1", "Edit Fields2") as inputs.  
   - Configure merge mode to combine all inputs appropriately (e.g., by index or key).

6. **Add Code Node "Combine candlestick":**  
   - Connect Merge node → Combine candlestick (Code node).  
   - Insert JavaScript code to further process combined candlestick data for AI input (e.g., feature extraction, normalization).

7. **Add HTTP Request Node for News:**  
   - Create HTTP Request node named "News".  
   - Connect Code in JavaScript → News node in parallel with market data requests.  
   - Configure to fetch latest crypto news from a news API.

8. **Add Code Node "Filtering News":**  
   - Connect News → Filtering News (Code).  
   - Add JavaScript code to filter news articles, keeping only relevant and recent items.

9. **Add Google Gemini AI Node "News sentiment Analyzer":**  
   - Connect Filtering News → News sentiment Analyzer.  
   - Configure Google Gemini credentials and set up prompts or parameters to analyze sentiment of news articles.

10. **Add Merge Node "Merge1":**  
    - Connect Combine candlestick and News sentiment Analyzer → Merge1.  
    - Configure to combine market data and sentiment analysis for AI agent input.

11. **Add Google Gemini Chat Model Node "Google Gemini Chat Model1":**  
    - Add and configure with Google Gemini credentials.  
    - This node is set as language model input for Crypto Agent.

12. **Add LangChain Agent Node "Crypto Agent":**  
    - Connect Merge1 → Crypto Agent main input.  
    - Set AI language model input to Google Gemini Chat Model1 node.  
    - Configure agent with prompt templates and parameters to generate trading signals.

13. **Add Telegram Node "Send a text message":**  
    - Connect Crypto Agent → Send a text message.  
    - Configure Telegram credentials and set chat ID or user ID to send the message.  
    - Format message to include trading signals generated.

14. **Test the Workflow:**  
    - Trigger via Telegram message.  
    - Verify data retrieval, AI processing, and message dispatch.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow integrates multiple Google Gemini AI nodes leveraging LangChain for advanced AI processing. Requires proper Google Gemini API credentials and LangChain node setup in n8n. | Workflow AI integration |
| Telegram nodes require bot token configuration and webhook setup. Refer to official Telegram Bot API for credentials. | https://core.telegram.org/bots/api |
| HTTP Request nodes must be configured with correct API endpoints and authentication for crypto market data and news sources (e.g., CoinGecko, CryptoCompare, NewsAPI). | Market data & News APIs |
| The workflow assumes JSON response parsing and error handling in code nodes; ensure robust error checks to prevent failures. | Best practices for error handling |
| Sticky notes are present in the workflow for annotations but contain no textual content. Use them as placeholders for comments or future documentation. | Workflow documentation aid |

---

Disclaimer: The provided text originates solely from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---