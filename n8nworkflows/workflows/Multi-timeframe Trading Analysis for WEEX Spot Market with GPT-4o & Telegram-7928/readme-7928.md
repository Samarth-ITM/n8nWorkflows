Multi-timeframe Trading Analysis for WEEX Spot Market with GPT-4o & Telegram

https://n8nworkflows.xyz/workflows/multi-timeframe-trading-analysis-for-weex-spot-market-with-gpt-4o---telegram-7928


# Multi-timeframe Trading Analysis for WEEX Spot Market with GPT-4o & Telegram

### 1. Workflow Overview

This workflow is designed as a **Multi-timeframe Trading Analysis AI Agent** for the **WEEX Spot Market**, leveraging **GPT-4o** and integrated with **Telegram** for user interaction. Its primary purpose is to deliver professional, structured swing-trading reports on WEEX spot market trading pairs (e.g., BTCUSDT_SPBL), combining multi-timeframe technical indicators, order flow data, and curated market sentiment from news sources.

The workflow’s logic is organized into the following main blocks:

- **1.1 Input Reception and Session Setup**  
  Handles incoming user messages from Telegram, extracts session and query details.

- **1.2 Core AI Orchestration (WEEX Quant AI Agent)**  
  Identifies user intent (Spot vs Futures) and routes queries accordingly; here, it primarily activates the Spot Market analysis tools.

- **1.3 Data Acquisition and Normalization (Raw Data AI Agent and REST Snapshot)**  
  Fetches and merges live and snapshot market data from WEEX APIs, including tickers, candles, order book depth, and trades, normalizing all into a unified format.

- **1.4 Multi-Timeframe Indicator Computation (15m, 1h, 4h, 1d AI Agent Tools)**  
  Independently analyzes candlestick data per timeframe calculating RSI, MACD, Bollinger Bands, SMA, EMA, and ADX, with qualitative labels and trend summaries.

- **1.5 Market Sentiment Analysis (News & Sentiment Tool)**  
  Aggregates and summarizes crypto market news from RSS feeds (CoinTelegraph, CoinDesk, NewsBTC) and NewsAPI, delivering sentiment context and top headlines.

- **1.6 Message Assembly and Delivery**  
  Composes the final report, splits it if exceeding Telegram’s message length limit, and sends the report back to the user via Telegram.

Supporting nodes like Think, Memory, and Calculator tools provide intermediate reasoning, state management, and numeric computations.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Session Setup

- **Overview:**  
  Listens for incoming Telegram messages, extracts user chat ID as session identifier, and captures the raw user message.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Adds "SessionId"

- **Node Details:**  

  - **Telegram Trigger**  
    - Type: Telegram Trigger  
    - Role: Entry point, catches new Telegram messages.  
    - Config: Listens on message updates; authenticates with Telegram Bot credential.  
    - Inputs: Incoming Telegram webhook messages.  
    - Outputs: User message JSON payload.  
    - Failures: Telegram API downtime, webhook misconfiguration.

  - **Adds "SessionId"**  
    - Type: Set node  
    - Role: Extracts and sets `sessionId` (user chat id) and raw `message` text for workflow processing.  
    - Config: Assigns `sessionId` = `message.chat.id`, `message` = `message.text` from Telegram payload.  
    - Inputs: Output from Telegram Trigger.  
    - Outputs: JSON with sessionId and user message.  
    - Failures: Missing chat id in message payload.

---

#### 2.2 Core AI Orchestration (WEEX Quant AI Agent)

- **Overview:**  
  The main AI orchestrator that interprets user intent, directs queries to appropriate subagents, and initiates the report generation pipeline.

- **Nodes Involved:**  
  - WEEX Quant AI Agent  
  - Splits message if more than 4000 characters  
  - Send a text message

- **Node Details:**  

  - **WEEX Quant AI Agent**  
    - Type: Langchain Agent node (custom AI orchestrator)  
    - Role: Parses user message, identifies if query pertains to Spot or Futures market, and routes accordingly (Spot in this workflow).  
    - Config: System prompt defines roles for Spot/Futures subagents, rules for analysis, and output format as plaintext for Telegram.  
    - Inputs: JSON with user message and sessionId.  
    - Outputs: Full trading report text.  
    - Failures: Misclassification of user intent, OpenAI API errors, long response truncation.

  - **Splits message if more than 4000 characters**  
    - Type: Code node  
    - Role: Splits long messages into chunks ≤ 4000 characters to comply with Telegram message length limits.  
    - Config: Splits `message` property into chunks of 4000 characters max.  
    - Inputs: Report text from WEEX Quant AI Agent.  
    - Outputs: Array of message parts.  
    - Failures: Logic errors in splitting, unexpected input formats.

  - **Send a text message**  
    - Type: Telegram node  
    - Role: Sends final (possibly split) report messages back to the user chat via Telegram bot.  
    - Config: Uses Telegram API credentials; sends text to the original chat id.  
    - Inputs: Split message parts and chat id.  
    - Outputs: Telegram API response.  
    - Failures: Telegram API errors, rate limits, chat id not valid.

---

#### 2.3 Data Acquisition and Normalization (Raw Data AI Agent and REST Snapshot)

- **Overview:**  
  Fetches baseline market data from WEEX Spot REST API (ticker, trades, order book, candles) and merges with live WebSocket streams to produce normalized, validated data for analysis.

- **Nodes Involved:**  
  - WEEX Spot Market Quant AI Agent tool  
  - WEEX SM Raw Data AI Agent Tool  
  - WEEX SM REST Snapshot Subagent Tool  
  - Fetch single ticker  
  - Fetch OrderBook Depth  
  - Fetch Trades  
  - Fetch 15m/1h/4h/1d Candles (multiple nodes)  
  - WEEX SM Raw Data Think  
  - WEEX SM REST Snapshot Think

- **Node Details:**  

  - **WEEX SM REST Snapshot Subagent Tool**  
    - Type: Langchain Agent Tool  
    - Role: Fetches core REST snapshot data for a symbol: 24h ticker, order book depth, recent trades, and OHLCV candles (15m, 1h, 4h, 1d).  
    - Config: Calls dedicated HTTP Request Tools for various endpoints, validates and normalizes data arrays, clamps bars to 200 max.  
    - Inputs: User symbol from upstream.  
    - Outputs: Normalized JSON snapshot object.  
    - Failures: API rate limits, incomplete data, invalid numeric values.

  - **WEEX SM Raw Data AI Agent Tool**  
    - Type: Langchain Agent Tool  
    - Role: Coordinates merging of REST snapshot baseline with WebSocket stream deltas (live updates) for real-time normalized data.  
    - Config: Merges datasets, ensures numeric validity, clamps array sizes, outputs unified Spot market raw data JSON.  
    - Inputs: Data from REST Snapshot and streaming data (if active).  
    - Outputs: Unified normalized market data JSON.  
    - Failures: Data merge conflicts, missing WS data, JSON parsing errors.

  - **Fetch single ticker, Fetch OrderBook Depth, Fetch Trades, Fetch [timeframe] Candles nodes**  
    - Type: HTTP Request Tool  
    - Role: REST API calls to WEEX endpoints for specific market data components.  
    - Config: URLs parameterized by symbol and timeframe, no authentication required.  
    - Inputs: Symbol and timeframe parameters.  
    - Outputs: Raw API JSON responses.  
    - Failures: API downtime, malformed responses, network timeouts.

  - **WEEX SM Raw Data Think and WEEX SM REST Snapshot Think**  
    - Type: Langchain ToolThink nodes  
    - Role: Internal validation and coordination to ensure data completeness, numeric correctness, and array sorting/clamping for REST snapshot and raw data merging.  
    - Inputs: Raw API data arrays.  
    - Outputs: Cleaned, normalized JSON payloads.  
    - Failures: Unexpected data formats, missing fields.

---

#### 2.4 Multi-Timeframe Indicator Computation (15m, 1h, 4h, 1d AI Agent Tools)

- **Overview:**  
  Independently analyzes each timeframe’s candlestick data to compute technical indicators and trend summaries, providing detailed quantitative and qualitative insights per timeframe.

- **Nodes Involved:**  
  - WEEX SM 15min AI Agent Tool  
  - WEEX SM 1hour AI Agent Tool  
  - WEEX SM 4hour AI Agent Tool  
  - WEEX SM 1day AI Agent Tool  
  - Think nodes for each timeframe (WEEX SM [timeframe] Think)  
  - Calculator nodes (Calculator, Calculator1, Calculator2, Calculator3)  
  - Memory nodes for each timeframe (WEEX SM [timeframe] Memory)

- **Node Details:**  

  - **WEEX SM [timeframe] AI Agent Tool (15m, 1h, 4h, 1d)**  
    - Type: Langchain Agent Tool  
    - Role: Processes OHLCV candle data for the specified timeframe; calculates RSI(14), MACD(12,26,9), Bollinger Bands (20,2), SMA(20/50/200), EMA(20/50/200), ADX(14).  
    - Provides numerical indicator values and qualitative labels (e.g., Overbought, Bullish Crossover).  
    - Summarizes trend context (Bullish/Bearish/Sideways).  
    - Outputs JSON with structured indicator results per timeframe.  
    - Failures: Incomplete candle data, API errors, calculation errors.

  - **Think nodes**  
    - Type: Langchain ToolThink  
    - Role: Intermediate reasoning for indicator computations, trend detection, and labeling inside each timeframe agent.  
    - Inputs: Raw candle data.  
    - Outputs: Intermediate indicator logic, no final text output.  
    - Failures: Expression or logic errors.

  - **Calculator nodes**  
    - Type: Langchain ToolCalculator  
    - Role: Performs numeric calculations required by indicator formulas.  
    - Inputs: Data from Think nodes.  
    - Outputs: Computed values for indicators.  
    - Failures: Numeric overflow, division by zero.

  - **Memory nodes**  
    - Type: Langchain MemoryBufferWindow  
    - Role: Stores recent context and intermediate results for each timeframe agent to maintain state.  
    - Failures: Memory overflow or corruption.

---

#### 2.5 Market Sentiment Analysis (News & Sentiment Tool)

- **Overview:**  
  Aggregates latest crypto market news from multiple RSS feeds and NewsAPI, analyzes sentiment, and provides a concise, professional market sentiment summary with top headlines.

- **Nodes Involved:**  
  - News & Sentiment Tool  
  - RSS CoinTelegraph News Feeds  
  - RSS Coindesk News Feeds  
  - RSS NewBTC News Feeds  
  - News Search API  
  - News & Sentiment Memory  
  - OpenAI Chat Model2

- **Node Details:**  

  - **News & Sentiment Tool**  
    - Type: Langchain Agent Tool  
    - Role: Merges RSS feed articles and NewsAPI search results, analyzes sentiment (Bullish/Bearish/Neutral), summarizes market context, and selects top 3–5 headlines with source links.  
    - Outputs formatted HTML block for Telegram.  
    - Failures: NewsAPI rate limits, incomplete RSS fetches, sentiment misclassification.

  - **RSS Feed Nodes (CoinTelegraph, Coindesk, NewsBTC)**  
    - Type: RSS Feed Read Tool  
    - Role: Fetches latest articles from specific crypto news sites.  
    - Failures: Feed unavailability, malformed XML.

  - **News Search API**  
    - Type: HTTP Request Tool with API key  
    - Role: Queries NewsAPI for recent news matching crypto symbol and terms.  
    - Failures: API key limits, query malformation.

  - **News & Sentiment Memory**  
    - Type: MemoryBufferWindow  
    - Role: Maintains recent sentiment analysis context to support incremental updates.  
    - Failures: Data staleness.

  - **OpenAI Chat Model2**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Supports News & Sentiment Tool with GPT-based analysis and summarization.  
    - Failures: OpenAI quota, latency.

---

#### 2.6 Message Assembly and Delivery

- **Overview:**  
  Post-processing of the AI-generated report, splitting if needed, and delivery through Telegram back to the user.

- **Nodes Involved:**  
  - Splits message if more than 4000 characters  
  - Send a text message

- **Node Details:**  
  See section 2.2.

---

### 3. Summary Table

| Node Name                        | Node Type                                    | Functional Role                                    | Input Node(s)                  | Output Node(s)                     | Sticky Note                                                                                       |
|---------------------------------|----------------------------------------------|---------------------------------------------------|--------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------|
| Telegram Trigger                | Telegram Trigger                             | Entry point, listens for Telegram messages        |                                | Adds "SessionId"                  |                                                                                                 |
| Adds "SessionId"               | Set Node                                     | Extracts sessionId and message text                | Telegram Trigger               | WEEX Quant AI Agent               |                                                                                                 |
| WEEX Quant AI Agent            | Langchain Agent                              | Orchestrates user intent and directs subagents    | Adds "SessionId"               | Splits message is more than 4000 characters |                                                                                                 |
| Splits message is more than 4000 characters | Code Node                          | Splits long messages for Telegram limits           | WEEX Quant AI Agent            | Send a text message               |                                                                                                 |
| Send a text message            | Telegram Node                                | Sends report back to Telegram user                  | Splits message                |                                   |                                                                                                 |
| WEEX Spot Market Quant AI Agent tool | Langchain Agent Tool                     | Specialized Spot Market analysis subagent          | WEEX Quant AI Agent             | WEEX SM REST Snapshot Subagent Tool, WEEX SM Raw Data AI Agent Tool, Indicator Tools, News & Sentiment Tool |                                                                                                 |
| WEEX SM REST Snapshot Subagent Tool | Langchain Agent Tool                     | Fetches baseline Spot market REST snapshot         | WEEX Spot Market Quant AI Agent tool | WEEX SM Raw Data AI Agent Tool |                                                                                                 |
| WEEX SM Raw Data AI Agent Tool | Langchain Agent Tool                         | Merges REST snapshot and WebSocket spot data       | WEEX SM REST Snapshot Subagent Tool | WEEX SM Raw Data Think           |                                                                                                 |
| WEEX SM Raw Data Think         | Langchain ToolThink                          | Validates and merges spot market raw data          | WEEX SM Raw Data AI Agent Tool | WEEX Spot Market Quant AI Agent tool |                                                                                                 |
| WEEX SM 15min AI Agent Tool    | Langchain Agent Tool                         | Computes 15m timeframe technical indicators         | WEEX Spot Market Quant AI Agent tool | WEEX SM 15min Think             |                                                                                                 |
| WEEX SM 1hour AI Agent Tool    | Langchain Agent Tool                         | Computes 1h timeframe technical indicators          | WEEX Spot Market Quant AI Agent tool | WEEX SM 1hour Think             |                                                                                                 |
| WEEX SM 4hour AI Agent Tool    | Langchain Agent Tool                         | Computes 4h timeframe technical indicators          | WEEX Spot Market Quant AI Agent tool | WEEX SM 4hour Think             |                                                                                                 |
| WEEX SM 1day AI Agent Tool     | Langchain Agent Tool                         | Computes 1d timeframe technical indicators          | WEEX Spot Market Quant AI Agent tool | WEEX SM 1day Think              |                                                                                                 |
| WEEX SM 15min Think            | Langchain ToolThink                          | Intermediate reasoning for 15m indicators           | WEEX SM 15min AI Agent Tool    | Calculator, WEEX SM 15min AI Agent Tool |                                                                                                 |
| WEEX SM 1hour Think            | Langchain ToolThink                          | Intermediate reasoning for 1h indicators            | WEEX SM 1hour AI Agent Tool    | Calculator1, WEEX SM 1hour AI Agent Tool |                                                                                                 |
| WEEX SM 4hour Think            | Langchain ToolThink                          | Intermediate reasoning for 4h indicators            | WEEX SM 4hour AI Agent Tool    | Calculator3, WEEX SM 4hour AI Agent Tool |                                                                                                 |
| WEEX SM 1day Think             | Langchain ToolThink                          | Intermediate reasoning for 1d indicators            | WEEX SM 1day AI Agent Tool     | Calculator2, WEEX SM 1day AI Agent Tool |                                                                                                 |
| Calculator                    | Langchain ToolCalculator                      | Numeric calculations for 15m indicators             | WEEX SM 15min Think            | WEEX SM 15min AI Agent Tool      |                                                                                                 |
| Calculator1                   | Langchain ToolCalculator                      | Numeric calculations for 1h indicators              | WEEX SM 1hour Think            | WEEX SM 1hour AI Agent Tool      |                                                                                                 |
| Calculator2                   | Langchain ToolCalculator                      | Numeric calculations for 1d indicators              | WEEX SM 1day Think             | WEEX SM 1day AI Agent Tool       |                                                                                                 |
| Calculator3                   | Langchain ToolCalculator                      | Numeric calculations for 4h indicators              | WEEX SM 4hour Think            | WEEX SM 4hour AI Agent Tool      |                                                                                                 |
| News & Sentiment Tool         | Langchain Agent Tool                         | Aggregates news, analyzes sentiment, outputs summary | RSS Nodes, News Search API, OpenAI Chat Model2 | WEEX Quant AI Agent            |                                                                                                 |
| RSS CoinTelegraph News Feeds  | RSS Feed Read Tool                           | Fetches CoinTelegraph RSS news                       |                                | News & Sentiment Tool            |                                                                                                 |
| RSS Coindesk News Feeds       | RSS Feed Read Tool                           | Fetches CoinDesk RSS news                            |                                | News & Sentiment Tool            |                                                                                                 |
| RSS NewBTC News Feeds         | RSS Feed Read Tool                           | Fetches NewsBTC RSS news                             |                                | News & Sentiment Tool            |                                                                                                 |
| News Search API               | HTTP Request Tool                            | Queries NewsAPI for crypto news                      |                                | News & Sentiment Tool            |                                                                                                 |
| OpenAI Chat Model             | Langchain OpenAI Chat Model                  | Language model for WEEX Quant AI Agent               |                                | WEEX Quant AI Agent              |                                                                                                 |
| OpenAI Chat Model1            | Langchain OpenAI Chat Model                  | Language model for Spot Market Quant AI Agent tool  |                                | WEEX Spot Market Quant AI Agent tool |                                                                                                |
| OpenAI Chat Model2            | Langchain OpenAI Chat Model                  | Language model for News & Sentiment Tool             |                                | News & Sentiment Tool            |                                                                                                 |
| OpenAI Chat Model4            | Langchain OpenAI Chat Model                  | Language model for WEEX SM Raw Data AI Agent Tool    |                                | WEEX SM Raw Data AI Agent Tool   |                                                                                                 |
| WEEX SM REST Snapshot Memory | Langchain MemoryBufferWindow                 | Maintains context for REST snapshot data             |                                | WEEX SM REST Snapshot Subagent Tool |                                                                                                 |
| WEEX SM Raw Data Memory       | Langchain MemoryBufferWindow                 | Maintains context for raw Spot market data            |                                | WEEX SM Raw Data AI Agent Tool   |                                                                                                 |
| WEEX SM 15min Memory          | Langchain MemoryBufferWindow                 | Maintains 15m timeframe indicator context            |                                | WEEX SM 15min AI Agent Tool      |                                                                                                 |
| WEEX SM 1hour Memory          | Langchain MemoryBufferWindow                 | Maintains 1h timeframe indicator context             |                                | WEEX SM 1hour AI Agent Tool      |                                                                                                 |
| WEEX SM 4hour Memory          | Langchain MemoryBufferWindow                 | Maintains 4h timeframe indicator context             |                                | WEEX SM 4hour AI Agent Tool      |                                                                                                 |
| WEEX SM 1day Memory           | Langchain MemoryBufferWindow                 | Maintains 1d timeframe indicator context             |                                | WEEX SM 1day AI Agent Tool       |                                                                                                 |
| News & Sentiment Memory       | Langchain MemoryBufferWindow                 | Maintains news sentiment analysis context            |                                | News & Sentiment Tool            |                                                                                                 |
| WEEX Quant AI Agent Memory    | Langchain MemoryBufferWindow                 | Maintains overall AI agent context                    |                                | WEEX Quant AI Agent              |                                                                                                 |
| Sticky Note10                 | Sticky Note                                  | Workflow system documentation, instructions, credits |                                |                                   | Full documentation content with installation and usage instructions, LinkedIn & legal notice.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates  
   - Credentials: Use Telegram Bot API credentials (your bot)  
   - Position: Start of workflow

2. **Create a Set node named "Adds 'SessionId'":**  
   - Assign `sessionId` = `{{$json["message"]["chat"]["id"]}}`  
   - Assign `message` = `{{$json["message"]["text"]}}`  
   - Connect Telegram Trigger output to this node

3. **Create Langchain Agent node "WEEX Quant AI Agent":**  
   - Purpose: Orchestrator to route user queries for Spot or Futures  
   - Parameters: Use system prompt defining Spot/Futures subagents, instructions, output format (plain text for Telegram)  
   - Credentials: OpenAI API key (GPT-4.1 or equivalent)  
   - Connect output of Set node to this node

4. **Create Code node "Splits message is more than 4000 characters":**  
   - JavaScript code to split text messages >4000 characters into chunks  
   - Connect output of WEEX Quant AI Agent to this node

5. **Create Telegram node "Send a text message":**  
   - Parameters: Send text from split messages to chatId from Telegram Trigger  
   - Credentials: Telegram API credentials  
   - Connect output of the splitter node here

6. **Create Langchain Agent Tool node "WEEX Spot Market Quant AI Agent tool":**  
   - Purpose: Specialized Spot market subagent to coordinate data and analysis  
   - Parameters: System message defining tool sequence (REST Snapshot, Raw Data merging, Indicator subagents, News/Sentiment) and output constraints  
   - Credentials: OpenAI API  
   - Connect WEEX Quant AI Agent to this node as a tool

7. **Create Langchain Agent Tool node "WEEX SM REST Snapshot Subagent Tool":**  
   - Purpose: Fetch baseline REST data (ticker, order book, trades, candles)  
   - Parameters: System message specifying API calls and data validation  
   - Credentials: None (public API)  
   - Connect WEEX Spot Market Quant AI Agent tool to this node

8. **Create HTTP Request nodes for:**  
   - Fetch single ticker  
   - Fetch order book depth  
   - Fetch recent trades  
   - Fetch candles for 15m, 1h, 4h, 1d  
   - Configure URLs dynamically for the symbol parameter  
   - Connect these nodes as needed to REST Snapshot Subagent Tool

9. **Create Langchain ToolThink nodes:**  
   - "WEEX SM REST Snapshot Think": Validates REST snapshot data completeness, numeric checks, sorting, clamping  
   - "WEEX SM Raw Data Think": Merges REST snapshot with WebSocket deltas (if applicable), validates merged data  
   - Connect output of REST Snapshot and Raw Data AI Agent Tool accordingly

10. **Create Langchain Agent Tool node "WEEX SM Raw Data AI Agent Tool":**  
    - Purpose: Orchestrate merging of REST snapshot and WebSocket stream data  
    - Connect REST Snapshot Subagent Tool and Raw Data Think node

11. **Create Langchain Agent Tool nodes for each timeframe indicator analysis:**  
    - "WEEX SM 15min AI Agent Tool"  
    - "WEEX SM 1hour AI Agent Tool"  
    - "WEEX SM 4hour AI Agent Tool"  
    - "WEEX SM 1day AI Agent Tool"  
    - System messages specify indicator computations and output JSON results for RSI, MACD, BBANDS, SMA, EMA, ADX  
    - Connect each to WEEX Spot Market Quant AI Agent tool as tools

12. **Create Langchain ToolThink nodes for each timeframe:**  
    - For internal reasoning on indicator calculations (15min, 1h, 4h, 1d)  
    - Connect respective AI Agent Tool outputs

13. **Create Langchain ToolCalculator nodes:**  
    - Four calculators to assist indicator computations  
    - Connect from Think nodes to calculators, then back to respective AI Agent Tool nodes

14. **Create Langchain MemoryBufferWindow nodes for context management:**  
    - For each AI Agent Tool (Spot Market, REST Snapshot, Raw Data, each timeframe, News & Sentiment, WEEX Quant AI Agent)  
    - Connect memory nodes to their respective agent or tool nodes

15. **Create News & Sentiment Tool node:**  
    - Langchain Agent Tool to merge RSS feeds and NewsAPI results and perform sentiment analysis  
    - System message defines news sources, sentiment classification, output formatting (Telegram HTML)  
    - Credentials: OpenAI API, NewsAPI key, Telegram Bot credentials

16. **Create RSS Feed Read Tool nodes:**  
    - For CoinTelegraph, CoinDesk, NewsBTC RSS feeds  
    - Connect outputs to News & Sentiment Tool

17. **Create HTTP Request Tool node "News Search API":**  
    - Query NewsAPI with dynamic query for crypto symbol and related terms  
    - Connect output to News & Sentiment Tool

18. **Create OpenAI Chat Model nodes for language model usage:**  
    - For main WEEX Quant AI Agent, Spot Market Quant AI Agent tool, News & Sentiment Tool, and Raw Data AI Agent Tool  
    - Use GPT-4.1-mini or equivalent  
    - Connect appropriately as languageModel credential nodes

19. **Connect all nodes as per dependencies:**  
    - Telegram Trigger → Adds "SessionId" → WEEX Quant AI Agent → WEEX Spot Market Quant AI Agent tool → REST Snapshot & Raw Data AI Agent → Indicator Tools → News & Sentiment Tool → Back to WEEX Quant AI Agent → Message Splitter → Send Telegram message

20. **Set up credentials:**  
    - OpenAI API key (GPT-4 or GPT-4o)  
    - Telegram Bot API token  
    - NewsAPI key  
    - No auth needed for WEEX Spot Market REST endpoints

21. **Test workflow:**  
    - Send a Telegram message with a spot market symbol (e.g., "BTCUSDT_SPBL")  
    - Verify multi-timeframe analysis report is returned

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is a proprietary AI automation system developed by Don Jayamaha for professional swing-trading reports on WEEX Spot Market. It integrates multi-timeframe technical analysis and market sentiment into Telegram reports.                                                                                                                                                     | See Sticky Note10 for detailed system documentation and installation instructions.               |
| Telegram output is strictly plain text (no HTML) except for the News & Sentiment Tool which outputs Telegram HTML snippets for headlines.                                                                                                                                                                                                                                            | Telegram message formatting requires text-only for main report.                                 |
| The system uses multiple Langchain custom nodes for Agent, Tool, Think, Calculator, and Memory functionality to modularize AI tasks and maintain state.                                                                                                                                                                                                                               | n8n Langchain nodes version 1.1 to 2.2 used.                                                    |
| Reliable baseline data is ensured by always fetching the REST snapshot before layering WebSocket updates; failure handling includes continuing with partial data and reporting data gaps in the output.                                                                                                                                                                               | Data robustness and fault tolerance.                                                            |
| News & Sentiment Tool combines trusted crypto news RSS feeds and NewsAPI search results to provide unbiased market sentiment summaries and top headlines with clickable links.                                                                                                                                                                                                      | NewsAPI website: https://newsapi.org/                                                           |
| For setup, ensure API rate limits and quota management for OpenAI and NewsAPI keys; Telegram bot must have webhook configured correctly in n8n.                                                                                                                                                                                                                                     | Webhook ID and credentials must be correctly configured in the respective nodes.                |
| Workflow includes explicit versioning and uses recommended credential setup as per n8n best practices.                                                                                                                                                                                                                                                                                 | Credential setup completed flag is true in metadata.                                           |
| © 2025 Treasurium Capital Limited Company. Unauthorized use, resale, or redistribution of this workflow is prohibited.                                                                                                                                                                                                                                                               | Legal notice in Sticky Note10.                                                                  |
| LinkedIn profile of author for support and contact: [linkedin.com/in/donjayamahajr](https://www.linkedin.com/in/donjayamahajr)                                                                                                                                                                                                                                                        | Contact for licensing and support.                                                             |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.