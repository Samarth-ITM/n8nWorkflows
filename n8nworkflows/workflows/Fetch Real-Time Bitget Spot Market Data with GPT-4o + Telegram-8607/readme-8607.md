Fetch Real-Time Bitget Spot Market Data with GPT-4o + Telegram

https://n8nworkflows.xyz/workflows/fetch-real-time-bitget-spot-market-data-with-gpt-4o---telegram-8607


# Fetch Real-Time Bitget Spot Market Data with GPT-4o + Telegram

### 1. Workflow Overview

This workflow, titled **"Fetch Real-Time Bitget Spot Market Data with GPT-4o + Telegram"**, is designed to serve as an AI-powered data-fetching agent that retrieves and formats real-time cryptocurrency market data from Bitget's Spot Market API. It listens for Telegram commands from authorized users, fetches multiple types of market data in parallel, processes and formats the data using GPT-4o-mini, and sends structured, readable reports back to the user on Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Authentication:** Listens for incoming Telegram messages and authenticates the user.
- **1.2 Session Metadata Creation:** Generates session context for multi-turn conversations.
- **1.3 Core Market Data Fetching Agent:** The main AI agent orchestrates multiple Bitget Spot Market API calls in parallel, processes data, and formats the output.
- **1.4 Data Processing and Formatting:** Includes calculator and reasoning nodes to parse, compute, and prepare the final Telegram message.
- **1.5 Message Size Handling:** Splits the final message if it exceeds Telegram's character limits.
- **1.6 Output Delivery:** Sends the formatted market report back to the Telegram user via bot.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Authentication

**Overview:**  
This block starts the workflow by triggering on incoming Telegram messages. It filters messages by authorized user to prevent unauthorized access.

**Nodes Involved:**  
- Telegram Trigger  
- User Authentication (Replace Telegram ID)

**Node Details:**  

- **Telegram Trigger**
  - Type: Telegram Trigger node  
  - Role: Listens for all new messages sent to the Telegram bot.  
  - Configuration: Listens for "message" updates only; uses Telegram Bot API credentials.  
  - Inputs: Incoming Telegram messages.  
  - Outputs: Raw Telegram message JSON.  
  - Potential Failures: Telegram API connectivity issues, webhook misconfiguration.  
  - Sticky Note: Explains this node listens for Telegram messages and triggers the agent.

- **User Authentication (Replace Telegram ID)**
  - Type: Code node (JavaScript)  
  - Role: Checks if the Telegram user ID matches the authorized user ID specified inside the code.  
  - Configuration: JavaScript compares incoming message sender ID with a hardcoded allowed ID; unauthorized requests return `{unauthorized: true}` stopping the workflow.  
  - Inputs: Output from Telegram Trigger.  
  - Outputs: Passes data downstream only if authorized user; otherwise workflow stops.  
  - Potential Failures: User ID not replaced with actual authorized ID results in access denial.  
  - Sticky Note: Describes purpose as user access validation.

---

#### 2.2 Session Metadata Creation

**Overview:**  
Generates a session identifier based on the Telegram chat ID and extracts the user's message for downstream processing and context tracking.

**Nodes Involved:**  
- Adds "SessionId"

**Node Details:**  

- **Adds "SessionId"**
  - Type: Set node  
  - Role: Creates two new fields: `sessionId` (from Telegram chat id) and `message` (user's text).  
  - Configuration: Assigns `sessionId` as `{{$json.message.chat.id}}` and message as `{{$json.message.text}}`.  
  - Inputs: Output from User Authentication node.  
  - Outputs: JSON object enriched with session context for multi-turn conversations.  
  - Potential Failures: Missing chat ID or message text in input JSON.  
  - Sticky Note: Explains this node creates session metadata for routing and memory.

---

#### 2.3 Core Market Data Fetching Agent (Bitget AI Agent)

**Overview:**  
This is the central orchestrator node that uses GPT-4o-mini and multiple HTTP request tools to fetch and format real-time Bitget spot market data. It calls Bitget Spot API endpoints in parallel for ticker, order book depth, recent trades, candlestick data, and historical candles, then processes and formats this data into a Telegram-ready text report.

**Nodes Involved:**  
- Bitget AI Agent  
- Ticker (24h Stats)  
- Order Book Depth  
- Recent Trades  
- Klines (Candles)  
- Historical Candles  
- Calculator  
- Think  
- Simple Memory  
- OpenAI Chat Model

**Node Details:**  

- **Bitget AI Agent**
  - Type: LangChain Agent node  
  - Role: Orchestrates data fetching and formatting using OpenAI and HTTP Request tools.  
  - Configuration:
    - Uses system prompt instructing the agent to fetch data only (no analysis).
    - Accesses Bitget Spot v2 REST API endpoints via attached HTTP tools.
    - Sends user message text as input.
  - Inputs: Session metadata and user message text.  
  - Outputs: Aggregated, formatted market data text.  
  - Integration: Calls multiple HTTP Request Tool nodes as AI tools; uses OpenAI Chat Model for text formatting; uses Simple Memory for session context; uses Calculator and Think for numeric operations and reasoning.  
  - Potential Failures: API rate limits, invalid symbols, network errors, OpenAI API limits or errors, malformed responses.  
  - Sticky Note: Describes this as the core orchestrator fetching and formatting Bitget data via multiple API endpoints.

- **Ticker (24h Stats)**
  - Type: HTTP Request Tool node  
  - Role: Fetches latest price and 24h statistics for requested symbol.  
  - Configuration: GET `https://api.bitget.com/api/v2/spot/market/tickers` with symbol query param derived from AI variables.  
  - Inputs: Called by Bitget AI Agent.  
  - Outputs: JSON with last price, open/high/low, change %, volume, best bid/ask.  
  - Potential Failures: Invalid symbol, API downtime, malformed JSON.  
  - Sticky Note: Details endpoint purpose and parameters.

- **Order Book Depth**
  - Type: HTTP Request Tool node  
  - Role: Retrieves order book bids and asks up to a specified limit.  
  - Configuration: GET `https://api.bitget.com/api/v2/spot/market/orderbook` with `symbol`, `type` (e.g., step0), and `limit`.  
  - Inputs: Called by Bitget AI Agent.  
  - Outputs: JSON arrays of bids and asks with price and size.  
  - Potential Failures: API errors, invalid parameters, empty order book.  
  - Sticky Note: Explains endpoint usage and parameter mapping.

- **Recent Trades**
  - Type: HTTP Request Tool node  
  - Role: Fetches recent public trades for the symbol.  
  - Configuration: GET `https://api.bitget.com/api/v2/spot/market/fills` with `symbol` and `limit` parameters.  
  - Inputs: Called by Bitget AI Agent.  
  - Outputs: JSON array of trades with price, size, side, timestamp.  
  - Potential Failures: API failure, empty trade data.  
  - Sticky Note: Documents endpoint details.

- **Klines (Candles)**
  - Type: HTTP Request Tool node  
  - Role: Retrieves OHLCV candlestick data for a symbol and interval.  
  - Configuration: GET `https://api.bitget.com/api/v2/spot/market/candles` with symbol, granularity, and limit params.  
  - Inputs: Called by Bitget AI Agent.  
  - Outputs: Array of candlestick data points.  
  - Potential Failures: Invalid interval/granularity, API errors.  
  - Sticky Note: Describes endpoint and params.

- **Historical Candles**
  - Type: HTTP Request Tool node  
  - Role: Retrieves older OHLCV candles ending before specified timestamp.  
  - Configuration: GET `https://api.bitget.com/api/v2/spot/market/history-candles` with symbol, granularity, endTime, and limit.  
  - Inputs: Called by Bitget AI Agent.  
  - Outputs: Array of historical candle data.  
  - Potential Failures: Missing or invalid endTime, API errors.  
  - Sticky Note: Explains endpoint and use cases.

- **Calculator**
  - Type: Calculator AI Tool node  
  - Role: Performs math operations such as spread calculation, % change, rounding.  
  - Inputs: Receives data from Bitget AI Agent.  
  - Outputs: Calculated numeric values for formatting.  
  - Potential Failures: Invalid numeric inputs or division by zero.  
  - Sticky Note: Describes math operations purpose.

- **Think**
  - Type: LangChain Tool Think node  
  - Role: Lightweight AI reasoning to reshape JSON data and format final Telegram text.  
  - Inputs: Receives multiple API results from Bitget AI Agent.  
  - Outputs: Clean, human-readable Telegram message text.  
  - Potential Failures: OpenAI API errors, malformed inputs.  
  - Sticky Note: Details node purpose as formatting helper.

- **Simple Memory**
  - Type: LangChain Memory Buffer Window node  
  - Role: Maintains short-term session memory keyed by sessionId for multi-turn interactions.  
  - Inputs: Session context from Adds "SessionId" node.  
  - Outputs: Provides context to Bitget AI Agent for stateful conversations.  
  - Potential Failures: Memory overflow or state desync.  
  - Sticky Note: Describes short-term memory role.

- **OpenAI Chat Model**
  - Type: LangChain Chat Model node using OpenAI  
  - Role: Executes GPT-4o-mini model to interpret and format messages.  
  - Inputs: Receives prompts from Bitget AI Agent.  
  - Outputs: GPT-generated text output.  
  - Potential Failures: OpenAI quota exhaustion, network errors.  
  - Sticky Note: Explains model usage for reasoning and formatting.

---

#### 2.4 Message Size Handling

**Overview:**  
Ensures that the Telegram messages do not exceed Telegram's maximum character limit (4000 chars). If the message is too long, it splits it into smaller chunks for sequential sending.

**Nodes Involved:**  
- Splits message is more than 4000 characters

**Node Details:**  

- **Splits message is more than 4000 characters**
  - Type: Code node (JavaScript)  
  - Role: Checks the final message length; if longer than 4000 characters, splits into chunks of 4000 chars or less.  
  - Configuration: Uses a custom JS function to split text safely.  
  - Inputs: Receives the final formatted message from Bitget AI Agent.  
  - Outputs: One or more JSON items each with a chunk of the message.  
  - Potential Failures: Unexpected input format, edge cases with exactly 4000 characters.  
  - Sticky Note: Explains logic for handling Telegram message size limits.

---

#### 2.5 Output Delivery

**Overview:**  
Sends the final formatted message or message chunks back to the Telegram user via the bot.

**Nodes Involved:**  
- Telegram (Send Message)

**Node Details:**  

- **Telegram (Send Message)**
  - Type: Telegram node (sendMessage)  
  - Role: Sends the prepared text messages to the Telegram chat identified by the session/chat ID.  
  - Configuration: Uses `chatId` from incoming Telegram message; disables appended attribution to keep message clean; sends HTML formatted text.  
  - Inputs: Receives one or more message chunks from splitter node.  
  - Outputs: Confirmation of Telegram message delivery.  
  - Potential Failures: Telegram API limits, invalid chat ID, network issues.  
  - Sticky Note: Describes the node as the final output sender of reports.

---

### 3. Summary Table

| Node Name                          | Node Type                                | Functional Role                       | Input Node(s)                     | Output Node(s)                        | Sticky Note                                                                                         |
|-----------------------------------|----------------------------------------|------------------------------------|----------------------------------|-------------------------------------|---------------------------------------------------------------------------------------------------|
| Telegram Trigger                  | Telegram Trigger                       | Receive incoming Telegram messages | —                                | User Authentication                 | Listens for new Telegram messages from users and triggers the full agent process.                 |
| User Authentication (Replace Telegram ID) | Code                                  | Authorize user by Telegram ID       | Telegram Trigger                 | Adds "SessionId"                    | Checks incoming Telegram ID against the approved user list.                                       |
| Adds "SessionId"                  | Set                                    | Create session context              | User Authentication              | Bitget AI Agent                    | Creates a sessionId using the Telegram chat_id; passed downstream for routing and memory.         |
| Bitget AI Agent                  | LangChain Agent                        | Core orchestrator and data fetcher | Adds "SessionId"                 | Splits message is more than 4000 characters | Core orchestrator; fetches Bitget data via multiple API calls and formats it via OpenAI.          |
| Ticker (24h Stats)                | HTTP Request Tool                      | Fetch latest price and 24h stats   | Bitget AI Agent (AI Tool)        | Bitget AI Agent (AI Tool)          | Returns latest price plus 24h stats for a symbol.                                                 |
| Order Book Depth                 | HTTP Request Tool                      | Fetch order book bids/asks          | Bitget AI Agent (AI Tool)        | Bitget AI Agent (AI Tool)          | Returns order book bids/asks up to limit.                                                         |
| Recent Trades                   | HTTP Request Tool                      | Fetch recent public trades          | Bitget AI Agent (AI Tool)        | Bitget AI Agent (AI Tool)          | Returns most recent public trades for a symbol.                                                   |
| Klines (Candles)                 | HTTP Request Tool                      | Fetch OHLCV candlestick data       | Bitget AI Agent (AI Tool)        | Bitget AI Agent (AI Tool)          | Returns OHLCV candles for specified granularity.                                                  |
| Historical Candles              | HTTP Request Tool                      | Fetch older OHLCV candles           | Bitget AI Agent (AI Tool)        | Bitget AI Agent (AI Tool)          | Returns historical candle data ending before endTime.                                            |
| Calculator                      | LangChain Calculator Tool              | Perform math operations             | Bitget AI Agent (AI Tool)        | Bitget AI Agent (AI Tool)          | Performs math like spread, % changes, rounding.                                                   |
| Think                          | LangChain Tool (Think)                 | Reshape and format data             | Bitget AI Agent (AI Tool)        | Bitget AI Agent (main)             | Lightweight reasoning helper for JSON reshaping and formatting.                                  |
| Simple Memory                  | LangChain Memory Buffer Window          | Store short-term session state      | Adds "SessionId"                 | Bitget AI Agent (AI Memory)        | Stores sessionId and symbol for multi-turn interactions.                                         |
| OpenAI Chat Model               | LangChain Chat Model                    | GPT-4o-mini language model          | Bitget AI Agent (AI LanguageModel) | Bitget AI Agent (AI Tool)          | Used for reasoning, interpreting signals, and generating structured HTML output.                  |
| Splits message is more than 4000 characters | Code                                  | Split long messages for Telegram    | Bitget AI Agent                  | Telegram                          | Splits message if exceeds 4000 characters for Telegram message limits.                            |
| Telegram                       | Telegram                               | Send message to Telegram user       | Splits message is more than 4000 | —                                 | Sends formatted HTML report or message chunks to the authenticated user via Telegram bot.         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates only.  
   - Set up Telegram Bot API credentials (OAuth2).  
   - Position: Starting point.

2. **Add User Authentication node:**  
   - Type: Code node (JavaScript)  
   - Paste code to check if incoming Telegram message sender ID matches the authorized Telegram user ID.  
   - Replace `<Replace>` in the code with your actual Telegram user ID.  
   - Connect output of Telegram Trigger to this node.

3. **Create Adds "SessionId" node:**  
   - Type: Set node  
   - Add two fields:  
     - `sessionId`: set to `{{$json.message.chat.id}}`  
     - `message`: set to `{{$json.message.text}}`  
   - Connect output of User Authentication node to this node.

4. **Add Simple Memory node:**  
   - Type: LangChain Memory Buffer Window  
   - No special parameters needed.  
   - Connect output of Adds "SessionId" node to this node (for AI memory context).

5. **Configure Bitget AI Agent node:**  
   - Type: LangChain Agent node  
   - Input property: set text to `{{$json.message}}` (user command).  
   - System prompt: Provide detailed instructions to the agent to fetch Bitget Spot API data only, no predictions or advice.  
   - Assign AI tools: add HTTP Request Tool nodes for each Bitget API endpoint (Ticker, Order Book, Recent Trades, Klines, Historical Candles).  
   - Assign Calculator and Think as AI tools for processing.  
   - Assign Simple Memory as AI memory.  
   - Assign OpenAI Chat Model (GPT-4o-mini) as AI language model.  
   - Connect output of Adds "SessionId" node to this agent.

6. **Create HTTP Request Tool nodes for Bitget API calls:**

   - **Ticker (24h Stats):**  
     - URL: `https://api.bitget.com/api/v2/spot/market/tickers`  
     - Method: GET  
     - Query params: `symbol` from AI variable `$fromAI('symbol','BTCUSDT','string')`  
     - Connect as AI tool to Bitget AI Agent.

   - **Order Book Depth:**  
     - URL: `https://api.bitget.com/api/v2/spot/market/orderbook`  
     - Method: GET  
     - Query params:  
       - `symbol`: `$fromAI('symbol','BTCUSDT','string')`  
       - `type`: `$fromAI('depthType','step0','string')`  
       - `limit`: `$fromAI('limit',100,'number')`  
     - Connect as AI tool to Bitget AI Agent.

   - **Recent Trades:**  
     - URL: `https://api.bitget.com/api/v2/spot/market/fills`  
     - Method: GET  
     - Query params:  
       - `symbol`: `$fromAI('symbol','BTCUSDT','string')`  
       - `limit`: `$fromAI('limit',100,'number')`  
     - Connect as AI tool to Bitget AI Agent.

   - **Klines (Candles):**  
     - URL: `https://api.bitget.com/api/v2/spot/market/candles`  
     - Method: GET  
     - Query params:  
       - `symbol`: `$fromAI('symbol','BTCUSDT','string')`  
       - `granularity`: `$fromAI('granularity','15min','string')`  
       - `limit`: `$fromAI('limit',20,'number')`  
     - Connect as AI tool to Bitget AI Agent.

   - **Historical Candles:**  
     - URL: `https://api.bitget.com/api/v2/spot/market/history-candles`  
     - Method: GET  
     - Query params:  
       - `symbol`: `$fromAI('symbol','BTCUSDT','string')`  
       - `granularity`: `$fromAI('granularity','15min','string')`  
       - `endTime`: `$fromAI('endTime','','number')`  
       - `limit`: `$fromAI('limit',100,'number')`  
     - Connect as AI tool to Bitget AI Agent.

7. **Add Calculator node:**  
   - Type: LangChain Calculator Tool  
   - No special configuration needed; used for math inside agent.  
   - Connect as AI tool in Bitget AI Agent.

8. **Add Think node:**  
   - Type: LangChain Think Tool  
   - No API call, used for intermediate reasoning and formatting.  
   - Connect as AI tool in Bitget AI Agent.

9. **Add OpenAI Chat Model node:**  
   - Type: LangChain Chat Model node  
   - Model: Select GPT-4o-mini or equivalent.  
   - Configure OpenAI API credentials.  
   - Connect as AI language model in Bitget AI Agent.

10. **Create Splits message is more than 4000 characters node:**  
    - Type: Code node (JavaScript)  
    - Paste JS code to split messages into <=4000 character chunks.  
    - Connect output of Bitget AI Agent to this node.

11. **Add Telegram Send Message node:**  
    - Type: Telegram node (sendMessage)  
    - Configure to send messages to `{{$json.message.chat.id}}`.  
    - Use Telegram Bot API credentials.  
    - Disable appended attribution for clean messages.  
    - Connect output of splitter node to this node.

12. **Activate the workflow.**  
    - Test by sending `/BTCUSDT` or other trading pair commands in Telegram from authorized user.  
    - Confirm receiving formatted Bitget market data report.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                 | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is proprietary and protected by U.S. copyright law. Reuse or resale is prohibited without license.                                                                                                                                              | © 2025 Treasurium Capital Limited Company                                                      |
| The core AI model used is GPT-4o-mini (OpenAI), selected for reasoning and formatting tasks.                                                                                                                                                                  | OpenAI API                                                                                        |
| The Bitget API endpoints used are only public GET endpoints, requiring no authentication; ensure API stability and rate limits compliance.                                                                                                                   | Bitget Spot REST API v2 documentation                                                           |
| Telegram message limit is 4000 characters; longer outputs are split for safe delivery.                                                                                                                                                                         | Telegram Bot API limits                                                                          |
| User must replace the Telegram ID in the authentication code node with their own Telegram user ID to secure the workflow.                                                                                                                                     | Node: User Authentication (Replace Telegram ID)                                                |
| For multi-turn conversation support, a Simple Memory node maintains session state by chat ID.                                                                                                                                                                 | LangChain Memory Buffer Window node                                                            |
| LinkedIn contact for support: Don Jayamaha - [linkedin.com/in/donjayamahajr](http://linkedin.com/in/donjayamahajr)                                                                                                                                             | Support contact                                                                                  |
| The workflow excludes any trading advice or strategy generation; it solely fetches and presents raw market data in a clean text format suitable for Telegram messaging.                                                                                       | System prompt instructions in Bitget AI Agent node                                             |
| Installation instructions recommend importing all related workflows and tools, setting up credentials for OpenAI and Telegram, and replacing Telegram ID for authentication.                                                                                  | Workflow Overview Sticky Note                                                                   |

---

**Disclaimer:**  
The provided content is generated exclusively from an automated n8n workflow export. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.