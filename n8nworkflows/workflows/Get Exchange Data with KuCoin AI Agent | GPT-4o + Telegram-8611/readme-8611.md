Get Exchange Data with KuCoin AI Agent | GPT-4o + Telegram

https://n8nworkflows.xyz/workflows/get-exchange-data-with-kucoin-ai-agent---gpt-4o---telegram-8611


# Get Exchange Data with KuCoin AI Agent | GPT-4o + Telegram

---

## 1. Workflow Overview

This n8n workflow, titled **"KuCoin AI Agent v1.02"**, integrates Telegram messaging with an AI-powered agent to fetch and report real-time KuCoin Spot Market data. It is designed for users who want to query detailed market information via Telegram bot commands and receive structured, human-readable reports powered by GPT-4o mini.

**Target Use Cases:**  
- Individual or enterprise users querying KuCoin spot market data conveniently through Telegram.  
- Automated delivery of comprehensive market snapshots including prices, order book depth, trade history, and candlestick data.  
- Multi-turn conversational context retention across Telegram sessions.  

**Logical Blocks:**  
- **1.1 Input Reception & Authentication:** Receives Telegram messages and validates user identity.  
- **1.2 Session Metadata & Memory:** Assigns session context and manages short-term memory for conversation continuity.  
- **1.3 KuCoin Spot Market AI Agent:** Core logic orchestrating calls to KuCoin REST API endpoints, fetching market data.  
- **1.4 AI Processing & Formatting:** Uses GPT-4o-mini to interpret API data, reason, and generate structured HTML reports.  
- **1.5 Message Handling & Output:** Splits long messages to respect Telegram limits and sends final formatted reports back to the user.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Authentication

**Overview:**  
This block listens for incoming Telegram messages and restricts access to authorized users by validating Telegram user IDs.

**Nodes Involved:**  
- Telegram Trigger  
- User Authentication (Replace Telegram ID)  
- Adds "SessionId"  

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger Node  
  - Role: Listens for new Telegram messages (updates of type "message").  
  - Config: Uses webhook with Telegram Bot credentials (`BinanceSpotTradingAIAgent_Bot`).  
  - Inputs: None (entry point).  
  - Outputs: Raw incoming Telegram message JSON.  
  - Edge Cases: Telegram API downtime, webhook misconfiguration, message format changes.

- **User Authentication (Replace Telegram ID)**  
  - Type: Code Node (JavaScript)  
  - Role: Validates incoming Telegram user by checking `message.from.id` against a hardcoded approved ID (placeholder `<<Replace>>`).  
  - Config: Returns unauthorized if user ID mismatches, preventing further processing.  
  - Key Expression: `if ($input.first().json.message.from.id !== <<Replace>>) { return {unauthorized: true}; }`  
  - Inputs: Output of Telegram Trigger.  
  - Outputs: Passes through data only if authorized.  
  - Edge Cases: Incorrect user ID replacement, unauthorized users triggering workflow, malformed messages.

- **Adds "SessionId"**  
  - Type: Set Node  
  - Role: Creates a `sessionId` from Telegram `chat.id` and extracts the raw message text.  
  - Config: Assigns `sessionId = $json.message.chat.id`, `message = $json.message.text`.  
  - Inputs: Output from User Authentication node.  
  - Outputs: Adds metadata for session tracking downstream.  
  - Edge Cases: Missing or malformed chat ID, empty messages.

---

### 2.2 Session Metadata & Memory

**Overview:**  
Stores session context and manages short-term memory buffer to support multi-turn interactions and stateful conversation.

**Nodes Involved:**  
- Simple Memory  

**Node Details:**

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window Node  
  - Role: Maintains a memory buffer indexed by `sessionId` to store conversation state, including symbol and other parameters.  
  - Config: Default settings; no additional parameters specified.  
  - Inputs: Connected from Adds "SessionId" via AI memory interface.  
  - Outputs: Provides context memory to AI agent.  
  - Edge Cases: Memory overflow, missing sessionId, state desynchronization.

---

### 2.3 KuCoin Spot Market AI Agent

**Overview:**  
Central orchestrator AI agent that interprets user messages, calls multiple KuCoin REST API endpoints, and compiles market data.

**Nodes Involved:**  
- KuCoin AI Agent  
- 24h Stats (HTTP Request Tool)  
- Order Book Depth (HTTP Request Tool)  
- Price (Latest) (HTTP Request Tool)  
- Best Bid/Ask (HTTP Request Tool)  
- Klines (Candles) (HTTP Request Tool)  
- Average Price (via Ticker) (HTTP Request Tool)  
- Recent Trades (HTTP Request Tool)  
- Calculator (LangChain Tool Calculator)  
- Think (LangChain Tool Think)  

**Node Details:**

- **KuCoin AI Agent**  
  - Type: LangChain Agent Node  
  - Role: Receives the user message and orchestrates calls to KuCoin endpoints using embedded tools.  
  - Config: System message defines agent behavior: only fetch and present KuCoin spot market data; no predictions or advice; format output as structured human-readable report.  
  - Inputs: Receives session metadata and user message.  
  - Outputs: Passes aggregated API results to the Think and Calculator tools, then final output for formatting.  
  - Edge Cases: KuCoin API downtime, malformed user queries, rate limits, invalid symbol formats.

- **24h Stats**  
  - Type: HTTP Request Tool  
  - Role: Fetches 24-hour rolling statistics for a given symbol (e.g., open, high, low, volume).  
  - Config: GET `https://api.kucoin.com/api/v1/market/stats` with query parameter `symbol` from AI input.  
  - Inputs: Symbol parameter from AI agent context.  
  - Outputs: JSON with 24h stats.  
  - Edge Cases: Symbol not found, API rate limiting, network errors.

- **Order Book Depth**  
  - Type: HTTP Request Tool  
  - Role: Retrieves order book bids and asks for the symbol at requested depth level (`level2_100` by default).  
  - Config: GET `https://api.kucoin.com/api/v1/market/orderbook/{level}` with query `symbol`.  
  - Inputs: Level and symbol from AI context.  
  - Outputs: JSON with bids and asks arrays.  
  - Edge Cases: Invalid level param, large response size, API errors.

- **Price (Latest)**  
  - Type: HTTP Request Tool  
  - Role: Gets latest market price and best bid/ask for the symbol.  
  - Config: GET `https://api.kucoin.com/api/v1/market/orderbook/level1` with symbol query.  
  - Inputs: Symbol from AI context.  
  - Outputs: JSON with latest price data.  
  - Edge Cases: Missing symbol, API timeouts.

- **Best Bid/Ask**  
  - Type: HTTP Request Tool  
  - Role: Similar to Price (Latest), returns best bid, best ask, and last price.  
  - Config: Same endpoint as Price (Latest).  
  - Inputs/Outputs: Same as above.  
  - Edge Cases: Same as above.

- **Klines (Candles)**  
  - Type: HTTP Request Tool  
  - Role: Fetches candlestick data for symbol and interval (e.g., 15min).  
  - Config: GET `https://api.kucoin.com/api/v1/market/candles` with parameters: symbol, type (interval), startAt, endAt.  
  - Inputs: Interval and timestamps provided by AI context.  
  - Outputs: Array of OHLCV candle data.  
  - Edge Cases: Invalid intervals, date ranges, empty data.

- **Average Price (via Ticker)**  
  - Type: HTTP Request Tool  
  - Role: Obtains real-time average price proxy using best bid/ask and last price.  
  - Config: Same endpoint as Price (Latest).  
  - Inputs/Outputs: Same as above.  
  - Edge Cases: Same as above.

- **Recent Trades**  
  - Type: HTTP Request Tool  
  - Role: Retrieves most recent public trades with price, size, side, timestamp.  
  - Config: GET `https://api.kucoin.com/api/v1/market/histories` with symbol and optional limit.  
  - Inputs: Symbol and limit from AI context.  
  - Outputs: Array of recent trade objects.  
  - Edge Cases: Large limits causing timeouts, API errors.

- **Calculator**  
  - Type: LangChain Tool Calculator Node  
  - Role: Performs math operations such as calculating spreads, percentage changes, rounding.  
  - Config: No API calls; interacts with AI agent to receive numeric inputs for calculations.  
  - Inputs: Various numeric data from API results.  
  - Outputs: Computed numeric results for report.  
  - Edge Cases: Division by zero, invalid numeric inputs.

- **Think**  
  - Type: LangChain Tool Think Node  
  - Role: Lightweight AI reasoning helper to reshape JSON, filter fields, and prepare data for final output formatting.  
  - Config: No external API calls; works on upstream JSON to generate clean output.  
  - Inputs: Aggregated API data.  
  - Outputs: Structured, human-readable JSON for final report.  
  - Edge Cases: Expression or JSON parse errors.

---

### 2.4 AI Processing & Formatting

**Overview:**  
Uses OpenAI GPT-4o mini model to interpret collected data, reason about it, and generate a structured HTML report summarizing KuCoin market data.

**Nodes Involved:**  
- OpenAI Chat Model  

**Node Details:**

- **OpenAI Chat Model**  
  - Type: LangChain Language Model Node (OpenAI GPT)  
  - Role: Receives AI agent output and processes it to produce a natural language, formatted response.  
  - Config: Uses model `gpt-4.1-mini` (GPT-4o mini variant) with OpenAI API credentials.  
  - Inputs: JSON data from KuCoin AI Agent.  
  - Outputs: Formatted HTML report string to be sent back to Telegram.  
  - Edge Cases: OpenAI API rate limits, invalid prompt data, timeout errors.

---

### 2.5 Message Handling & Output

**Overview:**  
Splits long AI-generated messages exceeding Telegram's 4000-character limit into chunks and sends them sequentially back to the user via Telegram.

**Nodes Involved:**  
- Splits message is more than 4000 characters (Code Node)  
- Telegram (SendMessage)  

**Node Details:**

- **Splits message is more than 4000 characters**  
  - Type: Code Node (JavaScript)  
  - Role: Checks if the AI-generated message exceeds 4000 characters; splits it into chunks if needed.  
  - Config: Custom JS logic slicing the message every 4000 chars.  
  - Inputs: AI Chat Model output message string.  
  - Outputs: Array of message chunks for sequential sending.  
  - Edge Cases: Empty messages, malformed strings.

- **Telegram**  
  - Type: Telegram SendMessage Node  
  - Role: Sends each chunk of the final report back to the Telegram chat.  
  - Config: Uses Telegram Bot credentials, sends HTML content to the chat ID from incoming message.  
  - Inputs: Chunked messages from splitter.  
  - Outputs: Confirmation of message sent.  
  - Edge Cases: Telegram API limits, message formatting errors, network issues.

---

## 3. Summary Table

| Node Name                          | Node Type                             | Functional Role                            | Input Node(s)                               | Output Node(s)                        | Sticky Note                                                                                     |
|-----------------------------------|-------------------------------------|-------------------------------------------|---------------------------------------------|-------------------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger                  | Telegram Trigger                    | Listen for incoming Telegram messages     | None                                        | User Authentication                  | Listens for new Telegram messages from users. Triggers full agent process and passes raw input. |
| User Authentication (Replace Telegram ID) | Code                              | Validate Telegram user ID                  | Telegram Trigger                            | Adds "SessionId"                    | Checks incoming Telegram ID against approved user list.                                        |
| Adds "SessionId"                 | Set                                | Add session metadata (sessionId, message) | User Authentication                        | KuCoin AI Agent                    | Creates a sessionId using Telegram chat_id, passed downstream for memory and routing.          |
| Simple Memory                   | LangChain Memory Buffer Window      | Maintain session context and multi-turn memory | Adds "SessionId" (AI memory)               | KuCoin AI Agent                    | Stores sessionId, symbol, and state data for multi-turn Telegram interactions.                  |
| KuCoin AI Agent                 | LangChain Agent                    | Orchestrate KuCoin API calls and data aggregation | Adds "SessionId" (main), Simple Memory (ai_memory), All KuCoin HTTP Tools (ai_tool), OpenAI Chat Model (ai_languageModel) | Splits message is more than 4000 characters | Orchestrates calls to KuCoin REST endpoints; fetches market data only; no predictions.          |
| 24h Stats                      | HTTP Request Tool                   | Fetch 24h market stats from KuCoin        | KuCoin AI Agent (ai_tool)                   | KuCoin AI Agent                   | Retrieves 24-hour rolling stats for symbol.                                                    |
| Order Book Depth               | HTTP Request Tool                   | Fetch order book bids/asks                 | KuCoin AI Agent (ai_tool)                   | KuCoin AI Agent                   | Returns order book bids/asks for KuCoin symbol at specified depth.                             |
| Price (Latest)                 | HTTP Request Tool                   | Fetch latest price and bid/ask             | KuCoin AI Agent (ai_tool)                   | KuCoin AI Agent                   | Returns latest price and best bid/ask for symbol.                                             |
| Best Bid/Ask                  | HTTP Request Tool                   | Fetch best bid and ask                      | KuCoin AI Agent (ai_tool)                   | KuCoin AI Agent                   | Returns best bid/ask and last price for symbol.                                               |
| Klines (Candles)              | HTTP Request Tool                   | Fetch candlestick OHLCV data                | KuCoin AI Agent (ai_tool)                   | KuCoin AI Agent                   | Retrieves candlestick bars for symbol and interval.                                          |
| Average Price (via Ticker)    | HTTP Request Tool                   | Fetch latest average price proxy            | KuCoin AI Agent (ai_tool)                   | KuCoin AI Agent                   | Provides real-time average price proxy via bid/ask spread and last price.                      |
| Recent Trades                 | HTTP Request Tool                   | Fetch most recent public trades             | KuCoin AI Agent (ai_tool)                   | KuCoin AI Agent                   | Returns recent public trade data for symbol.                                                  |
| Calculator                   | LangChain Tool Calculator            | Perform math operations (spreads, % changes) | KuCoin AI Agent (ai_tool)                   | KuCoin AI Agent                   | Calculates spreads, percentage changes, and normalizations.                                  |
| Think                        | LangChain Tool Think                 | Reshape and clean JSON data                  | KuCoin AI Agent (ai_tool)                   | KuCoin AI Agent                   | Helps AI reshape JSON, extract fields, and prepare Telegram message formatting.               |
| OpenAI Chat Model            | LangChain Language Model (OpenAI GPT) | Generate structured HTML report              | KuCoin AI Agent (ai_languageModel)          | Splits message is more than 4000 characters | Uses GPT-4o-mini to interpret data and produce formatted report.                              |
| Splits message is more than 4000 characters | Code                              | Split long messages into 4000-char chunks    | OpenAI Chat Model                          | Telegram                          | Splits GPT output if it exceeds Telegram message size limits.                                 |
| Telegram                      | Telegram SendMessage                | Sends final report chunks back to user      | Splits message is more than 4000 characters | None                             | Sends formatted HTML report (or split chunks) via Telegram bot to authenticated user.         |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with Telegram Bot credentials.  
   - Set to listen for update type "message".  
   - Save webhook URL generated by n8n.  

2. **Create Code Node for User Authentication**  
   - Type: Code  
   - Add JavaScript to check if `$input.first().json.message.from.id` equals your Telegram user ID.  
   - If not equal, return `{ unauthorized: true }` to block. Otherwise, pass data through.  
   - Connect Telegram Trigger output to this node's input.

3. **Create Set Node to Add Session Metadata**  
   - Type: Set  
   - Assign `sessionId = $json.message.chat.id`  
   - Assign `message = $json.message.text`  
   - Connect User Authentication node output to this node.

4. **Create Simple Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Use default config.  
   - Connect Adds "SessionId" node output to this node via AI memory interface.

5. **Create HTTP Request Tool Nodes for KuCoin Endpoints**  
   For each endpoint, create an HTTP Request Tool node with these parameters:

   - **24h Stats**  
     - URL: `https://api.kucoin.com/api/v1/market/stats`  
     - Query: `symbol` = `$fromAI('parameters0_Value', 'BTC-USDT', 'string')`  
     - Method: GET  

   - **Order Book Depth**  
     - URL: `https://api.kucoin.com/api/v1/market/orderbook/` + `$fromAI('level', 'level2_100', 'string')`  
     - Query: `symbol` = `$fromAI('parameters0_Value', 'BTC-USDT', 'string')`  
     - Method: GET  

   - **Price (Latest)**  
     - URL: `https://api.kucoin.com/api/v1/market/orderbook/level1`  
     - Query: `symbol` = `$fromAI('parameters0_Value', 'BTC-USDT', 'string')`  
     - Method: GET  

   - **Best Bid/Ask**  
     - Same as Price (Latest).  

   - **Klines (Candles)**  
     - URL: `https://api.kucoin.com/api/v1/market/candles`  
     - Query:  
       - `symbol` = `$fromAI('parameters0_Value', 'BTC-USDT', 'string')`  
       - `type` = `$fromAI('parameters1_Value', '15min', 'string')`  
       - `startAt` = `$fromAI('parameters2_Value', '', 'number')`  
       - `endAt` = `$fromAI('parameters3_Value', '', 'number')`  
     - Method: GET  

   - **Average Price (via Ticker)**  
     - Same as Price (Latest).  

   - **Recent Trades**  
     - URL: `https://api.kucoin.com/api/v1/market/histories`  
     - Query:  
       - `symbol` = `$fromAI('parameters0_Value', 'BTC-USDT', 'string')`  
       - `limit` = `$fromAI('parameters1_Value', 100, 'number')`  
     - Method: GET  

6. **Create LangChain Calculator Node**  
   - Type: LangChain Tool Calculator  
   - No specific config, used by AI agent for math operations.

7. **Create LangChain Think Node**  
   - Type: LangChain Tool Think  
   - Description: Helper for JSON reshaping and output formatting.

8. **Create KuCoin AI Agent Node**  
   - Type: LangChain Agent  
   - Configure system prompt with detailed instructions to only fetch data from KuCoin endpoints, no analysis or predictions.  
   - Setup to call the above HTTP Request Tools, Calculator, and Think nodes as tools.  
   - Connect Adds "SessionId" output to AI Agent main input.  
   - Connect Simple Memory to AI Agent memory input.  
   - Connect OpenAI Chat Model for language model interface.

9. **Create OpenAI Chat Model Node**  
   - Type: LangChain Language Model Node (OpenAI GPT)  
   - Set model to `gpt-4.1-mini` (GPT-4o mini).  
   - Provide OpenAI API credentials.  
   - Connect KuCoin AI Agent ai_languageModel output to this node input.

10. **Create Code Node to Split Messages**  
    - Type: Code  
    - Add JS logic to split messages longer than 4000 characters into chunks.  
    - Connect OpenAI Chat Model output to this node.

11. **Create Telegram Node for Sending Messages**  
    - Type: Telegram SendMessage  
    - Use same Telegram Bot credentials as trigger.  
    - Configure to send message text with HTML formatting to chat ID from incoming message.  
    - Connect splitter node output to this node.

12. **Connect all nodes in order:**  
    Telegram Trigger → User Authentication → Adds "SessionId" → KuCoin AI Agent → OpenAI Chat Model → Message Splitter → Telegram SendMessage  
    Also connect KuCoin AI Agent to all HTTP Request Tools, Calculator, Think, and Simple Memory nodes as AI tools and memory.

13. **Replace placeholders:**  
    - Replace user ID in User Authentication code node with your actual Telegram user ID.  
    - Configure credentials for Telegram Bot and OpenAI API.  

14. **Test the workflow:**  
    - Trigger via Telegram message.  
    - Verify only authorized users proceed.  
    - Confirm AI agent fetches KuCoin data correctly.  
    - Confirm GPT formats report.  
    - Confirm Telegram bot sends message back properly, splitting if necessary.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The workflow connects Telegram bot with an AI agent that fetches KuCoin Spot Market data via official REST API endpoints, ensuring no fabricated analysis or trading advice is provided.                                                                                                                                                                                        | Documentation Sticky Note (Node: Sticky Note17)                                                          |
| Supported KuCoin API endpoints: 24h Stats, Order Book Depth, Latest Price, Klines (Candles), Recent Trades, Average Price via ticker (proxy).                                                                                                                                                                                                                                 | KuCoin API Reference in Sticky Notes (Nodes: Sticky Note7 to Sticky Note14)                              |
| The AI agent uses GPT-4o mini model to interpret and format data, strictly as a data presentation tool, not predictive or advisory.                                                                                                                                                                                                                                          | Sticky Note6                                                                                             |
| Telegram has a 4096 character limit per message; the workflow splits longer outputs to ensure smooth delivery.                                                                                                                                                                                                                                                               | Sticky Note4                                                                                             |
| Session ID is derived from Telegram chat ID to support multi-turn conversations and stateful interactions.                                                                                                                                                                                                                                                                  | Sticky Note2 and Sticky Note9                                                                             |
| KuCoin symbols must be formatted with a dash, e.g., BTC-USDT, not concatenated as BTCUSDT.                                                                                                                                                                                                                                                                                    | Special Notes in documentation and node descriptions                                                    |
| Workflow includes proprietary system prompt and architecture © 2025 Treasurium Capital Limited Company. Use requires license; contact Don Jayamaha via LinkedIn [linkedin.com/in/donjayamahajr](http://linkedin.com/in/donjayamahajr) for support or licensing inquiries.                                                                                                          | Licensing and credits in Sticky Note17                                                                   |

---

**Disclaimer:** The provided description is derived exclusively from an automated n8n workflow. All data handled is public and legal; no illegal or protected content is included. The workflow respects current content policies.

---