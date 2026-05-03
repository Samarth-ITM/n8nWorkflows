Binance SM Price-24hrStats-OrderBook-Kline Tool

https://n8nworkflows.xyz/workflows/binance-sm-price-24hrstats-orderbook-kline-tool-4742


# Binance SM Price-24hrStats-OrderBook-Kline Tool

### 1. Workflow Overview

This workflow, titled **"Binance SM Price-24hrStats-OrderBook-Kline Tool"**, is designed to serve as a specialized market data agent for Binance Spot Market trading pairs. Its primary purpose is to collect, aggregate, and format comprehensive real-time market structure data including:

- Current trade price
- 24-hour price statistics (OHLC, volume, percentage change)
- Live order book depth (top bids and asks)
- Latest candlestick (kline) data across multiple key timeframes (15m, 1h, 4h, 1d)

The workflow is intended for use as a sub-agent called by higher-level workflows or agents (such as financial analysts or quant AI agents) to provide a unified market snapshot formatted for Telegram messaging.

#### Logical Blocks:

- **1.1 Trigger and Input Reception**  
  Handles receiving the input message and session data from a parent workflow.

- **1.2 AI Agent with Integrated Tools**  
  The core agent node that extracts the trading symbol from the input prompt, triggers all four market data API tools in parallel, and merges their outputs.

- **1.3 Market Data Tools**  
  Four separate HTTP Request tool nodes each querying a different Binance API endpoint:
  - Current Price
  - 24hr Stats
  - Order Book
  - Klines (candlestick data for multiple intervals)

- **1.4 AI Language Model Formatting**  
  An OpenAI chat model node formats the raw JSON market data into a clean, human-readable Telegram HTML message.

- **1.5 Session Memory**  
  A simple memory buffer to maintain session context (like last used symbol) between calls.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Trigger and Input Reception

- **Overview:**  
  Receives the workflow trigger from an external parent workflow, passing the input prompt and session identifier.

- **Nodes Involved:**  
  - `When Executed by Another Workflow`

- **Node Details:**  
  - Type: `Execute Workflow Trigger`  
  - Configuration: Accepts inputs named `message` (user prompt text) and `sessionId` (session context ID)  
  - Inputs: External workflow trigger  
  - Outputs: Triggers downstream processing in the agent node  
  - Edge Cases: Missing or malformed input may cause failures; no internal validation shown here  
  - Notes: This node enables this workflow to be used as a sub-agent rather than a standalone user-facing workflow.

---

#### 1.2 AI Agent with Integrated Tools

- **Overview:**  
  Central AI agent node that processes the input prompt, extracts the trading symbol, calls all four Binance data tools in parallel, and merges results into a single response formatted for Telegram.

- **Nodes Involved:**  
  - `Binance SM Price-24hrStats-OrderBook-Kline Agent`

- **Node Details:**  
  - Type: `LangChain Agent`  
  - Configuration:  
    - Accepts input text from the trigger node (`message` JSON field)  
    - System message instructs to extract symbol, call all four tools (current price, 24hr stats, order book, klines for 15m, 1h, 4h, 1d), combine outputs, format for Telegram HTML  
  - Inputs: Triggered by `When Executed by Another Workflow`  
  - Outputs: Calls to `getCurrentPrice`, `get24hrStats`, `getOrderBook`, and `getKlines`; also linked to `Simple Memory` and `OpenAI Chat Model` for managing context and formatting  
  - Version: 1.9  
  - Edge Cases:  
    - If no symbol found in prompt, agent asks for symbol input explicitly  
    - Failure in any tool invocation may affect output completeness  
  - Notes: This is the main reasoning and orchestrator node, integrating all market data tools.

---

#### 1.3 Market Data Tools

- **Overview:**  
  Four HTTP Request nodes fetch specific Binance Spot Market data endpoints, each configured as a LangChain tool for the agent to call.

- **Nodes Involved:**  
  - `getCurrentPrice`  
  - `get24hrStats`  
  - `getOrderBook`  
  - `getKlines`

- **Node Details:**

  1. **getCurrentPrice**  
     - Type: `toolHttpRequest`  
     - Role: Fetches latest trade price for a symbol  
     - Configuration: Calls `https://api.binance.com/api/v3/ticker/price` with query param `symbol`  
     - Input: `{ "symbol": "BTCUSDT" }` (symbol passed dynamically)  
     - Output: JSON with latest price  
     - Edge Cases: Invalid symbol or API downtime can cause errors  

  2. **get24hrStats**  
     - Type: `toolHttpRequest`  
     - Role: Retrieves 24-hour price stats including open, high, low, close, volume, and % change  
     - Configuration: Calls `https://api.binance.com/api/v3/ticker/24hr` with `symbol` query  
     - Input: `{ "symbol": "BTCUSDT" }`  
     - Output: JSON summary of 24h stats  
     - Edge Cases: Similar to above  

  3. **getOrderBook**  
     - Type: `toolHttpRequest`  
     - Role: Gets current order book (bids and asks) for the symbol  
     - Configuration: Calls `https://api.binance.com/api/v3/depth` with `symbol` and `limit` (default 100)  
     - Input: `{ "symbol": "BTCUSDT", "limit": 100 }`  
     - Output: JSON arrays of bids and asks, each [price, quantity]  
     - Edge Cases: Limit must be between 1 and 5000, too large requests may cause latency or failures  

  4. **getKlines**  
     - Type: `toolHttpRequest`  
     - Role: Fetches latest OHLCV candlestick data for multiple intervals  
     - Configuration: Calls `https://api.binance.com/api/v3/klines` with `symbol`, `interval`, and optional `limit` (usually 1)  
     - Input Examples:  
       `{ "symbol": "BTCUSDT", "interval": "15m", "limit": 1 }`  
       `{ "symbol": "BTCUSDT", "interval": "1h",  "limit": 1 }`  
       `{ "symbol": "BTCUSDT", "interval": "4h",  "limit": 1 }`  
       `{ "symbol": "BTCUSDT", "interval": "1d",  "limit": 1 }`  
     - Output: Array of arrays, each representing a candle with open time, OHLC, volume, etc.  
     - Edge Cases: Missing or invalid intervals, API rate limits, incomplete data for very new symbols  

---

#### 1.4 AI Language Model Formatting

- **Overview:**  
  Converts raw JSON data from the market tools into a formatted, labeled, human-readable Telegram HTML message block.

- **Nodes Involved:**  
  - `OpenAI Chat Model`

- **Node Details:**  
  - Type: `lmChatOpenAi` (OpenAI GPT-4.1-mini model)  
  - Role: Parsing, formatting, labeling numeric results for clear presentation  
  - Configuration: Uses GPT-4.1-mini with no additional options specified  
  - Credentials: OpenAI API key required  
  - Input: Raw JSON outputs from the agent node after data aggregation  
  - Output: Clean Telegram HTML text summarizing market data  
  - Edge Cases: API failures, rate limits, or unexpected input formats could cause formatting errors  

---

#### 1.5 Session Memory

- **Overview:**  
  Maintains session context and symbol continuity across multiple calls to the agent.

- **Nodes Involved:**  
  - `Simple Memory`

- **Node Details:**  
  - Type: `memoryBufferWindow`  
  - Role: Stores recent session messages and symbol extraction results  
  - Configuration: Default, no parameters specified  
  - Input: Linked to the agent node for session context  
  - Output: Provides context memory to agent for summarization or continuity  
  - Edge Cases: Memory overflow or session ID mismanagement may cause context loss  

---

### 3. Summary Table

| Node Name                              | Node Type                             | Functional Role                                      | Input Node(s)                      | Output Node(s)                             | Sticky Note                                                                                                                              |
|--------------------------------------|-------------------------------------|-----------------------------------------------------|----------------------------------|-------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow    | Execute Workflow Trigger             | Receives input from parent workflow                  | External trigger                 | Binance SM Price-24hrStats-OrderBook-Kline Agent | ## Triggered by Parent Workflow: Node activates only when called by parent workflow, expects `message` and `sessionId` inputs            |
| Binance SM Price-24hrStats-OrderBook-Kline Agent | LangChain Agent                     | Extracts symbol, calls all market data tools in parallel, merges results | When Executed by Another Workflow | getCurrentPrice, get24hrStats, getOrderBook, getKlines, Simple Memory, OpenAI Chat Model | ## Main Reasoning Agent: Accepts prompt, extracts trading pair, calls all four tools, merges results, formats response                   |
| getCurrentPrice                      | toolHttpRequest                     | Fetches latest trade price for symbol                | Binance SM Price-24hrStats-OrderBook-Kline Agent | Binance SM Price-24hrStats-OrderBook-Kline Agent | ## getCurrentPrice: Calls `/api/v3/ticker/price`, input `{ "symbol": "BTCUSDT" }`, returns last trade price                              |
| get24hrStats                        | toolHttpRequest                     | Fetches 24h OHLC and volume stats                     | Binance SM Price-24hrStats-OrderBook-Kline Agent | Binance SM Price-24hrStats-OrderBook-Kline Agent | ## get24hrStats: Calls `/api/v3/ticker/24hr`, input `{ "symbol": "BTCUSDT" }`, returns 24h stats including price change and volume       |
| getOrderBook                       | toolHttpRequest                     | Retrieves order book depth (top bids and asks)       | Binance SM Price-24hrStats-OrderBook-Kline Agent | Binance SM Price-24hrStats-OrderBook-Kline Agent | ## getOrderBook: Calls `/api/v3/depth`, input `{ "symbol": "BTCUSDT", "limit": 100 }`, returns top 100 bids and asks                    |
| getKlines                          | toolHttpRequest                     | Fetches latest candlestick data for multiple intervals | Binance SM Price-24hrStats-OrderBook-Kline Agent | Binance SM Price-24hrStats-OrderBook-Kline Agent | ## getKlines: Calls `/api/v3/klines` for 15m,1h,4h,1d intervals, input includes symbol, interval, limit=1, returns OHLCV for candles      |
| Simple Memory                      | memoryBufferWindow                  | Maintains session and symbol context                  | Binance SM Price-24hrStats-OrderBook-Kline Agent | Binance SM Price-24hrStats-OrderBook-Kline Agent | ## Simple Memory: Stores session and symbol context to ensure continuity across tool calls                                               |
| OpenAI Chat Model                  | lmChatOpenAi                       | Formats raw JSON into clean Telegram HTML summary    | Binance SM Price-24hrStats-OrderBook-Kline Agent | -                                         | ## OpenAI Formatter: Parses JSON results, converts numeric data into labeled text blocks, suitable for Telegram messaging               |
| Sticky Note (multiple)             | stickyNote                        | Informational comments explaining nodes and flow     | -                                | -                                         | Multiple sticky notes provide detailed documentation and usage context for nodes (see detailed notes in section 5)                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add `Execute Workflow Trigger` node named `When Executed by Another Workflow`.  
   - Configure inputs with two parameters: `message` (string), `sessionId` (string).  
   - This node enables the workflow to be triggered externally with these inputs.

2. **Create AI Agent Node:**  
   - Add `LangChain Agent` node named `Binance SM Price-24hrStats-OrderBook-Kline Agent`.  
   - Configure input text as `={{ $json.message }}` to receive the prompt.  
   - Paste the detailed system message instructing the agent to:  
     - Extract symbol from input  
     - Call all four market data tools in parallel  
     - Fetch klines for 15m, 1h, 4h, and 1d intervals  
     - Merge outputs and format for Telegram HTML  
   - Set prompt type to `define`.  
   - Connect this node’s input from `When Executed by Another Workflow`.

3. **Create Market Data Tool Nodes (4 total):**

   - **getCurrentPrice:**  
     - Type: `@n8n/n8n-nodes-langchain.toolHttpRequest`  
     - Name: `getCurrentPrice`  
     - URL: `https://api.binance.com/api/v3/ticker/price`  
     - Send query parameters: `symbol` (required)  
     - Tool Description: Fetches current trade price for the symbol.

   - **get24hrStats:**  
     - Type: `toolHttpRequest`  
     - Name: `get24hrStats`  
     - URL: `https://api.binance.com/api/v3/ticker/24hr`  
     - Send query parameters: `symbol` (required)  
     - Tool Description: Fetches 24-hour OHLC and volume stats.

   - **getOrderBook:**  
     - Type: `toolHttpRequest`  
     - Name: `getOrderBook`  
     - URL: `https://api.binance.com/api/v3/depth`  
     - Send query parameters: `symbol`, `limit` (default 100)  
     - Tool Description: Fetches top 100 bids and asks.

   - **getKlines:**  
     - Type: `toolHttpRequest`  
     - Name: `getKlines`  
     - URL: `https://api.binance.com/api/v3/klines`  
     - Send query parameters: `symbol`, `interval` (required), `limit` (optional)  
     - Tool Description: Fetches latest OHLCV candles for specified intervals.

4. **Connect Market Data Tools to Agent:**  
   - In the agent node settings, add these four HTTP Request nodes as tools.  
   - Configure the agent to call them in parallel.  
   - For `getKlines`, the agent should invoke it four times with intervals `15m`, `1h`, `4h`, and `1d`, each with `limit`=1.

5. **Create Memory Node:**  
   - Add `memoryBufferWindow` node named `Simple Memory`.  
   - Connect this memory node as the agent's session memory node to maintain context.

6. **Create OpenAI Chat Model Node:**  
   - Add `lmChatOpenAi` node named `OpenAI Chat Model`.  
   - Set model to `gpt-4.1-mini`.  
   - Connect this node to receive the aggregated JSON output from the agent node.  
   - Provide OpenAI API credentials.  
   - This node formats raw JSON data into Telegram-ready HTML text.

7. **Connect Workflow:**  
   - Connect output of `When Executed by Another Workflow` to `Binance SM Price-24hrStats-OrderBook-Kline Agent`.  
   - Connect agent outputs to all four market data nodes and `Simple Memory`.  
   - Connect agent output also to `OpenAI Chat Model` for final formatting.

8. **Credential Setup:**  
   - No authentication required for Binance public API.  
   - Configure OpenAI API credentials for the `OpenAI Chat Model` node.

9. **Activation and Testing:**  
   - Activate the workflow.  
   - Test by triggering via another workflow or manual run with JSON input such as:  
     ```
     {
       "message": "BTCUSDT",
       "sessionId": "12345"
     }
     ```  
   - Verify the formatted Telegram HTML output summarizing market data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                             |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| This workflow is designed as a sub-agent, not for direct user interaction. It is intended to be triggered by other workflows such as "Binance SM Financial Analyst Tool" or "Binance Spot Market Quant AI Agent".                                   | Sticky Note at trigger node                                  |
| OpenAI Chat Model node uses the `gpt-4.1-mini` model to parse and format JSON data into clean text blocks labeled with meaningful statistics for Telegram. API key required.                                                                   | Sticky Note at OpenAI Chat Model node                        |
| The agent node calls all four Binance API endpoints in parallel and merges results for unified output. It always fetches klines for four intervals: 15 minutes, 1 hour, 4 hours, and 1 day.                                                      | Sticky Note at Agent node                                    |
| Binance API public endpoints require no authentication but are subject to rate limits. Limit parameter in order book queries can be between 1 and 5000; default 100 is used here for performance balance.                                         | Sticky Notes at individual HTTP Request nodes                |
| Telegram Output Example:  
```
📊 BTCUSDT Market Overview

💰 Price: $63,220  
📈 24h Change: +2.3% | Volume: 45,210 BTC  

📉 Order Book  
• Top Bid: $63,190  
• Top Ask: $63,230  

🕰️ Latest Candles  
• 15m: O: $63,000 | C: $63,220 | Vol: 320 BTC  
• 1h : O: $62,700 | C: $63,300 | Vol: 980 BTC  
• 4h : O: $61,800 | C: $63,500 | Vol: 2,410 BTC  
• 1d : O: $59,200 | C: $63,220 | Vol: 7,850 BTC
``` | Provided in main sticky note documentation                   |
| For detailed installation, credential setup, and usage instructions see the comprehensive workflow documentation embedded as sticky note.                                                                                                   | Sticky Note with large documentation block                   |
| Licensing: Proprietary to Treasurium Capital Limited Company. Redistribution or replication without license is prohibited.                                                                                                                    | Sticky Note with licensing information                        |
| Author Contact: Don Jayamaha – LinkedIn: [http://linkedin.com/in/donjayamahajr](http://linkedin.com/in/donjayamahajr)                                                                                                                           | Sticky Note with author contact                               |

---

**Disclaimer:** The text provided here is a structured analysis and documentation derived exclusively from an automated n8n workflow. It respects all current content policies and contains no illegal or protected content. All data processed is legal and public.