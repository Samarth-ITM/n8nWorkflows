Smart Stock Trading Recommendations with GPT-4, TwelveData & NewsAPI Analysis

https://n8nworkflows.xyz/workflows/smart-stock-trading-recommendations-with-gpt-4--twelvedata---newsapi-analysis-8594


# Smart Stock Trading Recommendations with GPT-4, TwelveData & NewsAPI Analysis

### 1. Workflow Overview

This workflow, titled **"Smart Stock Trading Recommendations with GPT-4, TwelveData & NewsAPI Analysis"**, is designed to provide actionable stock trading recommendations by integrating multi-timeframe technical data, real-time news sentiment analysis, and AI-driven interpretation.

Its primary use case is to receive a chat input identifying a stock (via name or symbol), gather comprehensive market data and news, analyze sentiment and technical indicators, and deliver a concise, AI-generated trade recommendation (BUY/SELL/HOLD) with key price levels and risk parameters.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Symbol Conversion:** Receives chat input and converts stock name to symbol.
- **1.2 Market Data Retrieval:** Fetches multi-interval price time series (4h, 1 day, 1 week) from TwelveData API.
- **1.3 News Retrieval and Sentiment Analysis:** Retrieves recent news articles and analyzes sentiment using OpenAI GPT-4.
- **1.4 Data Validation and Aggregation:** Checks API success, aggregates data sets for further processing.
- **1.5 Stock Price Data Transformation:** Processes raw price data via JavaScript code node to extract technical indicators and generate summaries.
- **1.6 AI-Powered Technical and Sentiment Analysis:** Uses a LangChain AI agent integrating technical data, news sentiment, stock sentiment (via Perplexity AI), and chart images to produce a trading recommendation.
- **1.7 Chat Response Generation:** Sends the AI-generated recommendation back as a chat message.
- **Error Handling:** Detects and responds to errors in data retrieval.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Symbol Conversion

- **Overview:**  
  Captures stock-related chat messages and converts stock names into standardized stock symbols for consistent API querying.

- **Nodes Involved:**  
  - When chat message received  
  - Convert to Stock Symbol

- **Node Details:**

  - **When chat message received**  
    - Type: LangChain chat trigger node  
    - Role: Entry point for chat messages containing stock names or symbols  
    - Configuration: Listens for incoming chat messages; response mode set to "responseNodes" for downstream handling  
    - Inputs: None (webhook trigger)  
    - Outputs: Chat message content forwarded for symbol conversion  
    - Edge Cases: Invalid or empty chat input; malformed messages  
    - Version: 1.3  

  - **Convert to Stock Symbol**  
    - Type: OpenAI GPT-4.1-mini node  
    - Role: Converts user-provided stock name to exact stock symbol  
    - Configuration: Prompt strictly requests only symbol output, e.g., input "Microsoft" → output "MSFT"  
    - Expressions: Uses input from chat message (`$json.chatInput`)  
    - Inputs: Chat message content from previous node  
    - Outputs: Standardized stock symbol string  
    - Edge Cases: Unknown or ambiguous stock names; GPT misinterpretation  
    - Version: 1.8  

---

#### 1.2 Market Data Retrieval

- **Overview:**  
  Queries TwelveData API for stock price time series data at three intervals: 4-hour, 1-day, and 1-week. Provides multi-timeframe market context.

- **Nodes Involved:**  
  - 4h trend  
  - 1day trend  
  - 1week trend

- **Node Details:**

  - Each node is an HTTP Request node calling the TwelveData time_series endpoint with parameters:  
    - `symbol` set dynamically to the converted stock symbol from previous block  
    - `interval` set to `4h`, `1day`, or `1week` accordingly  
  - Authentication: Uses HTTP query parameter authentication with TwelveData API key credential  
  - Inputs: Stock symbol from "Convert to Stock Symbol" node  
  - Outputs: JSON time series data including price, volume, and meta information  
  - Edge Cases: API rate limits, invalid symbol errors, network timeouts, incomplete data responses  
  - Version: 4.2  

---

#### 1.3 News Retrieval and Sentiment Analysis

- **Overview:**  
  Fetches news articles related to the stock symbol and analyzes their sentiment using GPT-4, producing a structured sentiment output.

- **Nodes Involved:**  
  - Get News  
  - News Sentiment Analyzer

- **Node Details:**

  - **Get News**  
    - Type: HTTP Request node  
    - Role: Calls NewsAPI's "everything" endpoint querying news based on stock symbol  
    - Authentication: API key via HTTP query parameter  
    - Inputs: Stock symbol from conversion node  
    - Outputs: List of news articles in JSON format  
    - Edge Cases: API quota exceeded, no news found, network failures  
    - Version: 4.2  

  - **News Sentiment Analyzer**  
    - Type: OpenAI GPT-4.1-mini node  
    - Role: Performs detailed sentiment analysis on news articles  
    - Configuration: Prompt instructs GPT to output strict JSON with sentiment category (Positive/Neutral/Negative), score (-1 to 1), and rationale  
    - Input: JSON-stringified articles from Get News  
    - Outputs: JSON sentiment object  
    - Edge Cases: Failure to parse JSON, ambiguous sentiment, GPT API errors  
    - Version: 1.8  

---

#### 1.4 Data Validation and Aggregation

- **Overview:**  
  Validates TwelveData API responses for success, aggregates multi-interval data, and routes success or error accordingly.

- **Nodes Involved:**  
  - Aggregate  
  - Check Twelvedata API Success  
  - Merge

- **Node Details:**

  - **Merge**  
    - Type: Merge node  
    - Role: Combines 4h, 1day, and 1week trend responses into one stream  
    - Configuration: Number of inputs = 3 (one per timeframe)  
    - Inputs: HTTP Request nodes for each timeframe  
    - Outputs: Combined data array  
    - Edge Cases: Missing or partial inputs  

  - **Aggregate**  
    - Type: Aggregate node  
    - Role: Aggregates all merged items into a single item containing all datasets  
    - Inputs: Merged multi-interval data  
    - Outputs: Aggregated data object with all timeframes  
    - Edge Cases: Empty input arrays  

  - **Check Twelvedata API Success**  
    - Type: Switch node  
    - Role: Checks if the 4h trend API response status equals "ok"  
    - Outputs: Routes to success or error branch  
    - Edge Cases: API returning error states, malformed responses  

---

#### 1.5 Stock Price Data Transformation

- **Overview:**  
  Processes the aggregated raw price data with a JavaScript code node to compute technical indicators, trend direction, support/resistance levels, volume and volatility metrics, and prepare a detailed summary for AI consumption.

- **Nodes Involved:**  
  - Code

- **Node Details:**

  - **Code**  
    - Type: Code node (JavaScript)  
    - Role: Converts raw time series data into structured technical analysis data for the AI agent  
    - Key Functions: Calculates moving averages, trend direction, support/resistance, volume trends, volatility, momentum indicators, multi-timeframe analysis summary  
    - Input: Aggregated multi-interval data object  
    - Output: Transformed JSON with overview, price trends, technical analysis, multi-timeframe data, and LLM-ready summary string  
    - Edge Cases: Missing data, insufficient data length, malformed input  
    - Version: 2  

---

#### 1.6 AI-Powered Technical and Sentiment Analysis

- **Overview:**  
  Uses a LangChain AI Agent node to synthesize technical data, news sentiment, additional sentiment from Perplexity AI, and stock chart images to generate a final trading recommendation.

- **Nodes Involved:**  
  - Technical Data AI Agent  
  - Get Stock Sentiment  
  - Get Chart Image for Stock  
  - Think  
  - OpenAI Chat Model

- **Node Details:**

  - **Technical Data AI Agent**  
    - Type: LangChain agent node  
    - Role: Core AI decision-maker integrating all inputs to produce a trade recommendation  
    - Input: Aggregated technical data, news sentiment, stock sentiment, chart image URL  
    - Configuration: Detailed prompt describing multi-timeframe analysis, sentiment rules, risk management, and output format (strict, no rationale, structured)  
    - Outputs: Structured textual recommendation with fields: Technical Recommendation, Entry Price, Stop-Loss, Target Price, Risk/Reward Ratio, Confidence Level  
    - Edge Cases: Missing or conflicting data, AI model timeouts, incomplete instructions  
    - Version: 2.2  

  - **Get Stock Sentiment**  
    - Type: Perplexity AI Tool node  
    - Role: Queries recent overall sentiment about the stock to supplement news sentiment  
    - Input: Original chat input (stock name/symbol)  
    - Outputs: Sentiment data for AI Agent consumption  
    - Edge Cases: API failures, ambiguous sentiment  

  - **Get Chart Image for Stock**  
    - Type: HTTP Request Tool node  
    - Role: Fetches a 1-week trading chart image URL from Chart-img API for visual context in recommendation  
    - Input: Stock symbol (with exchange prefix, e.g., NASDAQ:MSFT) dynamically set via AI override expression  
    - Outputs: Image URL for AI Agent use  
    - Edge Cases: API key missing, rate limit, invalid symbol, malformed request  

  - **Think**  
    - Type: LangChain toolThink node  
    - Role: Internal AI reasoning step (may support agent decision-making)  
    - Inputs/Outputs: Connected to AI Agent for additional processing  
    - Edge Cases: Model latency or failure  

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI chat model node (gpt-4o)  
    - Role: Language model backend supporting AI Agent operations  
    - Edge Cases: API usage limits, model errors  

---

#### 1.7 Chat Response Generation

- **Overview:**  
  Sends the final AI-generated trading recommendation back to the user as a chat message.

- **Nodes Involved:**  
  - Respond to Chat  
  - Respond with Error

- **Node Details:**

  - **Respond to Chat**  
    - Type: LangChain chat node  
    - Role: Outputs the final recommendation message to the chat interface  
    - Inputs: AI Agent's recommendation  
    - Configuration: Wait for user reply disabled (one-way response)  
    - Edge Cases: Message send failures  

  - **Respond with Error**  
    - Type: LangChain chat node  
    - Role: Sends error message if data retrieval failed (e.g., TwelveData API error)  
    - Inputs: Error details from switch node  
    - Configuration: Displays error message with details; no user reply expected  
    - Edge Cases: Error text missing or malformed  

---

### 3. Summary Table

| Node Name                  | Node Type                              | Functional Role                                    | Input Node(s)                             | Output Node(s)                            | Sticky Note                                                                                         |
|----------------------------|--------------------------------------|---------------------------------------------------|------------------------------------------|------------------------------------------|---------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Entry point for chat stock input                   | None                                     | Convert to Stock Symbol                   |                                                                                                   |
| Convert to Stock Symbol     | @n8n/n8n-nodes-langchain.openAi      | Convert stock name to symbol                        | When chat message received                | 4h trend, 1day trend, 1week trend        |                                                                                                   |
| 4h trend                   | n8n-nodes-base.httpRequest            | Fetch 4-hour price time series                      | Convert to Stock Symbol                   | Merge                                    | ## STOCK PRICE TREND<br>* Fetch using TwelveData APIs<br>* Get your free API Key from here: https://twelvedata.com/<br>* Replace your API Key under the Credentials |
| 1day trend                 | n8n-nodes-base.httpRequest            | Fetch 1-day price time series                       | Convert to Stock Symbol                   | Merge                                    | Same as above                                                                                    |
| 1week trend                | n8n-nodes-base.httpRequest            | Fetch 1-week price time series                      | Convert to Stock Symbol                   | Merge                                    | Same as above                                                                                    |
| Merge                      | n8n-nodes-base.merge                  | Combine multi-interval price data                   | 4h trend, 1day trend, 1week trend         | Aggregate                                |                                                                                                   |
| Aggregate                  | n8n-nodes-base.aggregate              | Aggregate merged data into single object            | Merge                                    | Check Twelvedata API Success             |                                                                                                   |
| Check Twelvedata API Success| n8n-nodes-base.switch                 | Validate TwelveData API response status             | Aggregate                                | Code, Get News (Success), Respond with Error (Error) |                                                                                           |
| Get News                   | n8n-nodes-base.httpRequest            | Fetch related news articles                          | Check Twelvedata API Success (Success)  | News Sentiment Analyzer                   | ## NEWS API<br>* Get your free API key from here: https://newsapi.org/<br>* Replace your API Key under the Credentials |
| News Sentiment Analyzer    | @n8n/n8n-nodes-langchain.openAi      | Analyze sentiment of news articles                   | Get News                                 | Merge1                                   |                                                                                                   |
| Merge1                     | n8n-nodes-base.merge                  | Merge news sentiment with stock price data          | Code, News Sentiment Analyzer             | Aggregate1                               |                                                                                                   |
| Aggregate1                 | n8n-nodes-base.aggregate              | Aggregate all data for AI agent processing           | Merge1                                   | Technical Data AI Agent                   |                                                                                                   |
| Code                       | n8n-nodes-base.code                   | Transform raw price data into technical indicators  | Check Twelvedata API Success (Success)  | Merge1                                   |                                                                                                   |
| Technical Data AI Agent    | @n8n/n8n-nodes-langchain.agent        | Generate final trade recommendation                  | Aggregate1, Think, OpenAI Chat Model, Get Stock Sentiment, Get Chart Image for Stock | Respond to Chat                          |                                                                                                   |
| Think                      | @n8n/n8n-nodes-langchain.toolThink   | Assist AI agent reasoning                            | Technical Data AI Agent                   | Technical Data AI Agent (loop)            |                                                                                                   |
| OpenAI Chat Model           | @n8n/n8n-nodes-langchain.lmChatOpenAi| Language model for AI agent                          | Technical Data AI Agent                   | Technical Data AI Agent                   |                                                                                                   |
| Get Stock Sentiment         | n8n-nodes-base.perplexityTool         | Get additional sentiment about stock                 | None (uses chat input)                    | Technical Data AI Agent                   |                                                                                                   |
| Get Chart Image for Stock   | n8n-nodes-base.httpRequestTool        | Fetch stock chart image for AI context               | None (dynamic symbol from AI override)  | Technical Data AI Agent                   | ## CHART-IMG API<br>* Get your free API key from here: http://chart-img.com/<br>* Replace your API Key under the Credential<br>* Optional Step. You can even delete this |
| Respond to Chat             | @n8n/n8n-nodes-langchain.chat         | Send final trade recommendation to user              | Technical Data AI Agent                   | None                                     |                                                                                                   |
| Respond with Error          | @n8n/n8n-nodes-langchain.chat         | Send error message on API failure                     | Check Twelvedata API Success (Error)     | None                                     |                                                                                                   |
| Sticky Note                | n8n-nodes-base.stickyNote             | Informational notes on TwelveData API                 | None                                     | None                                     | ## STOCK PRICE TREND<br>* Fetch using TwelveData APIs<br>* Get your free API Key from here: https://twelvedata.com/<br>* Replace your API Key under the Credentials |
| Sticky Note1               | n8n-nodes-base.stickyNote             | Informational notes on NewsAPI                         | None                                     | None                                     | ## NEWS API<br>* Get your free API key from here: https://newsapi.org/<br>* Replace your API Key under the Credentials |
| Sticky Note2               | n8n-nodes-base.stickyNote             | Informational notes on Chart-img API                   | None                                     | None                                     | ## CHART-IMG API<br>* Get your free API key from here: http://chart-img.com/<br>* Replace your API Key under the Credential<br>* Optional Step. You can even delete this |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Node: *When chat message received* (LangChain chat trigger)  
   - Configuration: Set webhook to listen for chat inputs; response mode "responseNodes".

2. **Create Stock Symbol Conversion:**  
   - Node: *Convert to Stock Symbol* (OpenAI GPT-4.1-mini)  
   - Configuration: Prompt to strictly convert stock name to symbol only.  
   - Connect output of "When chat message received" → input of this node.  
   - Set OpenAI API credentials.

3. **Create Market Data HTTP Requests:**  
   - Nodes: *4h trend*, *1day trend*, *1week trend* (HTTP Request nodes)  
   - Configuration:  
     - URL: `https://api.twelvedata.com/time_series`  
     - Authentication: HTTP Query with TwelveData API Key credential (setup API key in n8n)  
     - Query Parameters:  
       - symbol: expression from Convert to Stock Symbol output (`={{ $json.message.content }}`)  
       - interval: set respectively to `4h`, `1day`, `1week`  
   - Connect output of "Convert to Stock Symbol" to each of these nodes.

4. **Merge Market Data Responses:**  
   - Node: *Merge*  
   - Configuration: Number of inputs = 3  
   - Connect outputs of 4h, 1day, 1week trend nodes to Merge inputs.

5. **Aggregate Merged Data:**  
   - Node: *Aggregate*  
   - Configuration: Aggregate all item data into one object.  
   - Connect output of Merge to Aggregate.

6. **Check API Success:**  
   - Node: *Check Twelvedata API Success* (Switch)  
   - Configuration: Check if `$('4h trend').item.json.status === 'ok'`  
   - On Success, connect to next steps; on Error, connect to error response.

7. **Create News API Request:**  
   - Node: *Get News* (HTTP Request)  
   - Configuration:  
     - URL: `https://newsapi.org/v2/everything`  
     - Authentication: HTTP Query with NewsAPI key credential  
     - Query Parameter: `q` set to stock symbol expression (`={{ $json.message.content }}`)  
   - Connect from "Check Twelvedata API Success" success output.

8. **News Sentiment Analysis:**  
   - Node: *News Sentiment Analyzer* (OpenAI GPT-4.1-mini)  
   - Configuration: Prompt GPT to analyze news sentiment and output JSON with category, score, rationale.  
   - Input: JSON stringified news articles from Get News.  
   - Set OpenAI API credentials.  
   - Connect from Get News output.

9. **Transform Stock Data with Code Node:**  
   - Node: *Code* (JavaScript)  
   - Paste the provided JavaScript code that processes multi-timeframe data, calculates moving averages, trends, support/resistance, volume, volatility, and prepares an LLM summary.  
   - Input: Aggregated market data from "Check Twelvedata API Success" success output.  
   - Connect output to next merge.

10. **Merge News Sentiment and Technical Data:**  
    - Node: *Merge1*  
    - Connect outputs of "Code" and "News Sentiment Analyzer" to Merge1 inputs.

11. **Aggregate Combined Data for AI Agent:**  
    - Node: *Aggregate1*  
    - Aggregate all merged data into a single item for AI analysis.  
    - Connect output of Merge1 to Aggregate1.

12. **Create AI Agent Node:**  
    - Node: *Technical Data AI Agent* (LangChain agent)  
    - Configuration:  
      - Provide detailed prompt describing analysis framework, multi-timeframe priority, sentiment integration, risk management, and output strict format.  
      - Input: Aggregated data from Aggregate1, plus additional AI tools  
    - Connect output of Aggregate1 to AI Agent input.

13. **Set Up Additional AI Tools:**  
    - Node: *Get Stock Sentiment* (Perplexity AI)  
      - Query recent sentiment about the stock using chat input.  
      - Connect output to AI Agent.  
      - Set Perplexity credentials.  
    - Node: *Get Chart Image for Stock* (HTTP Request Tool)  
      - POST request to Chart-img API for 1-week chart image with symbol parameter dynamically set (e.g., "NASDAQ:MSFT")  
      - Connect output to AI Agent.  
      - Set Chart-img API credentials.

14. **Add Supporting AI Nodes:**  
    - Node: *Think* (LangChain toolThink) for internal AI reasoning linked to AI Agent.  
    - Node: *OpenAI Chat Model* (gpt-4o) supporting AI Agent's language model operations.  
    - Connect these nodes as per dependencies to AI Agent.

15. **Final Chat Response Node:**  
    - Node: *Respond to Chat* (LangChain chat)  
    - Configuration: Send AI Agent’s recommendation to user chat; no user reply expected.  
    - Connect output of AI Agent to Respond to Chat.

16. **Error Response Node:**  
    - Node: *Respond with Error* (LangChain chat)  
    - Configuration: Output error message when TwelveData API call fails.  
    - Connect error output of "Check Twelvedata API Success" to this node.

17. **Add Sticky Notes for Documentation:**  
    - Add three sticky note nodes with provided instructional content about API keys for TwelveData, NewsAPI, and Chart-img services.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Get your free TwelveData API key from https://twelvedata.com/                                  | TwelveData API credential setup                  |
| Get your free NewsAPI key from https://newsapi.org/                                            | News API credential setup                         |
| Get your free Chart-img API key from http://chart-img.com/                                     | Optional chart image generation API               |
| The workflow uses GPT-4 variants (gpt-4.1-mini and gpt-4o) for different tasks                  | OpenAI API usage                                 |
| Sentiment analysis output is strictly JSON formatted with category, score (-1 to 1), and rationale | For consistent AI interpretation                  |
| Risk management rules enforce max 2% risk per trade and minimum 1:2 reward ratio                 | Embedded in AI Agent prompt                       |
| The workflow integrates Perplexity AI for complementary stock sentiment data                    | Requires valid Perplexity API credentials         |

---

This completes the comprehensive reference documentation for the **Smart Stock Trading Recommendations with GPT-4, TwelveData & NewsAPI Analysis** workflow. It aims to facilitate understanding, reproduction, modification, and error anticipation for both human and AI users.