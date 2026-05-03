Real-time OKX Spot Market Data with GPT-4o & Telegram

https://n8nworkflows.xyz/workflows/real-time-okx-spot-market-data-with-gpt-4o---telegram-8609


# Real-time OKX Spot Market Data with GPT-4o & Telegram

---

### 1. Workflow Overview

This workflow, titled **"Real-time OKX Spot Market Data with GPT-4o & Telegram"**, is designed to provide real-time, structured spot market data from the OKX cryptocurrency exchange directly to authenticated Telegram users. It integrates multiple OKX public REST API endpoints to fetch detailed market data such as tickers, order book depth, recent trades, candlestick (Kline) data, and mark prices. The workflow uses GPT-4o-mini (an OpenAI language model) as an AI agent to format, reason about, and present this raw data in a clean, human-readable style optimized for Telegram messages.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception & User Authentication**: Handles incoming Telegram messages, authenticates the user by Telegram ID, and creates session metadata.
- **1.2 AI Agent & Data Fetching**: The core agent that interprets user input, calls multiple OKX API endpoints in parallel, and orchestrates the retrieval of various market data.
- **1.3 Utility Processing & Formatting**: Uses AI tools and calculator nodes to process, reshape, and format fetched data into structured text.
- **1.4 Output Handling & Telegram Delivery**: Splits messages longer than Telegram limits into chunks and sends formatted reports back via Telegram bot.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & User Authentication

- **Overview:**  
  This block listens for incoming Telegram messages, verifies if the sender is authorized, and prepares session metadata for downstream processing.

- **Nodes Involved:**  
  - Telegram Trigger  
  - User Authentication (Replace Telegram ID)  
  - Adds "SessionId"  

- **Node Details:**

  1. **Telegram Trigger**  
     - Type: Telegram Trigger Node  
     - Role: Listens for new Telegram messages (updates of type "message").  
     - Configuration: Uses a webhook to receive Telegram messages from users.  
     - Credentials: Telegram Bot API token configured for the bot named "BinanceSpotTradingAIAgent_Bot".  
     - Inputs: External Telegram message webhook.  
     - Outputs: Emits the full message JSON.  
     - Edge Cases: Telegram API downtime, invalid webhook setup, message format changes.  
     - Sticky Note: "Listens for new Telegram messages from users. Triggers the full agent process."

  2. **User Authentication (Replace Telegram ID)**  
     - Type: Code Node (JavaScript)  
     - Role: Checks if the incoming Telegram user ID matches an authorized ID.  
     - Configuration: Hardcoded check against a single Telegram user ID (placeholder `<<Replace>>` to be replaced with actual ID).  
     - Expression: Checks `$input.first().json.message.from.id` against allowed ID.  
     - Input: Data from Telegram Trigger.  
     - Output: Passes data downstream if authorized; otherwise outputs `{unauthorized: true}` which halts further processing.  
     - Edge Cases: Unauthorized user messages, missing message fields.  
     - Sticky Note: "Checks incoming Telegram ID against the approved user list."

  3. **Adds "SessionId"**  
     - Type: Set Node  
     - Role: Creates session metadata by assigning sessionId based on Telegram chat ID and copies the message text into a simpler field.  
     - Configuration:  
       - Assigns `sessionId = $json.message.chat.id`  
       - Assigns `message = $json.message.text`  
     - Input: Authorized Telegram message JSON.  
     - Output: Passes session metadata and user message for AI processing.  
     - Sticky Note: "Creates a sessionId using the Telegram chat_id for memory and routing."

---

#### Block 1.2: AI Agent & Data Fetching

- **Overview:**  
  The core AI agent interprets the user's command, orchestrates calls to multiple OKX API endpoints (price, order book, trades, candles, mark price, stats), and uses AI tools to reason and format the raw data.

- **Nodes Involved:**  
  - OKX AI Agent  
  - 24h Stats (Ticker)  
  - Order Book Depth  
  - Price (Latest)  
  - Best Bid/Ask  
  - Klines (Candles)  
  - Average / Mark Price  
  - Recent Trades  
  - Calculator  
  - Think (AI Tool)  
  - Simple Memory  
  - OpenAI Chat Model  

- **Node Details:**

  1. **OKX AI Agent**  
     - Type: LangChain Agent Node  
     - Role: Receives user message and sessionId, calls multiple OKX API tools in parallel, applies logical orchestration rules, and formats output text.  
     - Configuration:  
       - System message defines it as "OKX Spot Market Data Agent" with strict rules: only fetch and present data, no analysis or predictions.  
       - Uses HTTP GET to OKX public API endpoints with query parameters like `instId`.  
       - Calls downstream HTTP Request Tool nodes as AI tools.  
     - Inputs: User message text and sessionId from "Adds SessionId".  
     - Outputs: Clean, structured text report with market data.  
     - Edge Cases: API failures or missing data replaced with "N/A", strict no-advice policy enforced.  
     - Sticky Note: "Core orchestrator; fetches and presents market data from OKX endpoints."

  2. **24h Stats (Ticker)**  
     - Type: HTTP Request Tool  
     - Role: Fetches 24-hour ticker statistics from OKX API for specified instrument (e.g., last price, open, high, low, volume).  
     - Configuration: GET `https://www.okx.com/api/v5/market/ticker` with `instId` query from AI inputs.  
     - Input: Parameters passed dynamically from AI agent.  
     - Output: JSON containing 24h stats.  
     - Sticky Note: Detailed notes on endpoint usage and parameters.

  3. **Order Book Depth**  
     - Type: HTTP Request Tool  
     - Role: Retrieves order book bids and asks up to configurable depth (`sz`, default 25).  
     - Configuration: GET `https://www.okx.com/api/v5/market/books` with `instId` and `sz` from AI inputs.  
     - Output: JSON with `bids`, `asks`, and timestamp.  
     - Sticky Note: Explains difference with Binance and max size limit.

  4. **Price (Latest)**  
     - Type: HTTP Request Tool  
     - Role: Returns latest trade price and related ticker info (last price, high, low, ask/bid).  
     - Configuration: GET `https://www.okx.com/api/v5/market/ticker` with `instId`.  
     - Output: JSON with latest prices.  
     - Sticky Note: Notes on required `instId` format.

  5. **Best Bid/Ask**  
     - Type: HTTP Request Tool  
     - Role: Provides best bid and ask prices with sizes for instrument.  
     - Configuration: GET same endpoint as Price (Latest) but agent extracts bid/ask fields.  
     - Output: JSON with `bidPx`, `bidSz`, `askPx`, `askSz`.  
     - Sticky Note: Notes on data included in ticker response.

  6. **Klines (Candles)**  
     - Type: HTTP Request Tool  
     - Role: Retrieves OHLCV candlestick bars for a given time interval and limit (max 100).  
     - Configuration: GET `https://www.okx.com/api/v5/market/candles` with parameters `instId`, `bar` (e.g., 15m), and `limit`.  
     - Output: JSON array of candle data.  
     - Sticky Note: Explains parameters and differences with Binance.

  7. **Average / Mark Price**  
     - Type: HTTP Request Tool  
     - Role: Fetches mark price (fair/average price reference) for instrument.  
     - Configuration: GET `https://www.okx.com/api/v5/market/mark-price` with `instType=SPOT` and `instId`.  
     - Output: JSON array with mark price.  
     - Sticky Note: Mark price used as average price substitute.

  8. **Recent Trades**  
     - Type: HTTP Request Tool  
     - Role: Returns the most recent public trades for the given instrument (up to 100).  
     - Configuration: GET `https://www.okx.com/api/v5/market/trades` with `instId` and optional limit.  
     - Output: JSON array of trades.  
     - Sticky Note: Notes on required formatting.

  9. **Calculator**  
     - Type: LangChain Calculator Tool Node  
     - Role: Performs math operations such as spread calculation, percentage changes, and rounding.  
     - Configuration: No API call; calculations based on data fed from previous nodes.  
     - Output: Numeric results used by AI Agent.  
     - Sticky Note: Used for computing spreads, changes, and normalizing values.

  10. **Think**  
      - Type: LangChain Think Tool Node  
      - Role: AI reasoning helper for reshaping JSON, selecting fields, formatting data before final output.  
      - Configuration: No API call; purely internal reasoning.  
      - Input/Output: Works on data from HTTP nodes → prepares for final report.  
      - Sticky Note: Lightweight reasoning helper.

  11. **Simple Memory**  
      - Type: LangChain Memory Buffer Window  
      - Role: Stores session state including sessionId, symbol, and other context to support multi-turn conversations.  
      - Configuration: Connected to AI Agent as memory.  
      - Sticky Note: Supports multi-turn Telegram interactions and state tracking.

  12. **OpenAI Chat Model**  
      - Type: LangChain OpenAI Chat Model Node  
      - Role: Uses GPT-4o-mini for natural language reasoning, interpreting signals, generating structured HTML or text.  
      - Configuration: Connected as AI language model to the OKX AI Agent.  
      - Credentials: OpenAI API key configured.  
      - Sticky Note: Used to interpret signals and produce structured output.

---

#### Block 1.3: Utility Processing & Formatting

- **Overview:**  
  This block manages processing of the AI agent's output, ensuring message size limits are respected, and prepares the final formatted messages for Telegram.

- **Nodes Involved:**  
  - Splits message is more than 4000 characters  

- **Node Details:**

  1. **Splits message is more than 4000 characters**  
     - Type: Code Node (JavaScript)  
     - Role: Checks if AI output message exceeds Telegram's 4000 character limit; if so, splits it into smaller chunks to avoid message rejection.  
     - Configuration: Splits the text into 4000-character segments preserving message order.  
     - Input: AI Agent output text.  
     - Output: Array of JSON messages each containing a chunk of the original message.  
     - Edge Cases: Handles very long AI outputs gracefully.  
     - Sticky Note: Ensures Telegram message size limits are respected.

---

#### Block 1.4: Output Handling & Telegram Delivery

- **Overview:**  
  This final block sends the formatted text (or split chunks) back to the original Telegram user via the bot.

- **Nodes Involved:**  
  - Telegram  

- **Node Details:**

  1. **Telegram**  
     - Type: Telegram Node (Send Message)  
     - Role: Sends the final formatted message(s) to the authenticated Telegram chat ID.  
     - Configuration:  
       - Text parameter set dynamically from the message content.  
       - Chat ID derived from the original Telegram Trigger node's message chat.id.  
       - Option to disable attribution appended by Telegram node.  
     - Credentials: Same Telegram Bot API token as Telegram Trigger.  
     - Edge Cases: Telegram API rate limits, message delivery failures.  
     - Sticky Note: Sends structured market reports to the user.

---

### 3. Summary Table

| Node Name                         | Node Type                             | Functional Role                                    | Input Node(s)                           | Output Node(s)                  | Sticky Note                                                                                                          |
|----------------------------------|-------------------------------------|---------------------------------------------------|---------------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger                 | Telegram Trigger                    | Listens for incoming Telegram messages             | External Telegram webhook             | User Authentication             | Listens for new Telegram messages from users. Triggers the full agent process.                                       |
| User Authentication (Replace Telegram ID) | Code                              | Validates Telegram user ID                          | Telegram Trigger                     | Adds "SessionId"                | Checks incoming Telegram ID against the approved user list.                                                          |
| Adds "SessionId"                 | Set                                 | Adds session metadata (sessionId and message)      | User Authentication                 | OKX AI Agent                   | Creates a sessionId using the Telegram chat_id for memory and routing.                                               |
| OKX AI Agent                    | LangChain Agent                     | Core AI orchestration; fetches & formats market data | Adds "SessionId"                   | Splits message is more than 4000 characters | Core orchestrator; fetches and presents market data from OKX endpoints.                                              |
| 24h Stats                       | HTTP Request Tool                   | Fetches 24h ticker stats                            | OKX AI Agent (AI tool)               | OKX AI Agent                  | Detailed notes on endpoint usage and parameters.                                                                      |
| Order Book Depth                | HTTP Request Tool                   | Fetches order book bids/asks                        | OKX AI Agent (AI tool)               | OKX AI Agent                  | Explains difference with Binance and max size limit.                                                                  |
| Price (Latest)                  | HTTP Request Tool                   | Fetches latest trade price                          | OKX AI Agent (AI tool)               | OKX AI Agent                  | Notes on required `instId` format.                                                                                    |
| Best Bid/Ask                   | HTTP Request Tool                   | Fetches best bid and ask prices                     | OKX AI Agent (AI tool)               | OKX AI Agent                  | Notes on data included in ticker response.                                                                            |
| Klines (Candles)               | HTTP Request Tool                   | Fetches OHLCV candlestick data                      | OKX AI Agent (AI tool)               | OKX AI Agent                  | Explains parameters and differences with Binance.                                                                     |
| Average / Mark Price            | HTTP Request Tool                   | Fetches mark price (fair price)                      | OKX AI Agent (AI tool)               | OKX AI Agent                  | Mark price used as average price substitute.                                                                           |
| Recent Trades                  | HTTP Request Tool                   | Fetches recent public trades                         | OKX AI Agent (AI tool)               | OKX AI Agent                  | Notes on required formatting.                                                                                          |
| Calculator                    | LangChain Calculator Tool           | Performs math operations                             | OKX AI Agent (AI tool)               | OKX AI Agent                  | Used for computing spreads, changes, and normalizing values.                                                         |
| Think                         | LangChain Think Tool                | AI reasoning helper for formatting                   | OKX AI Agent (AI tool)               | OKX AI Agent                  | Lightweight reasoning helper.                                                                                          |
| Simple Memory                 | LangChain Memory Buffer Window      | Stores session state for multi-turn interaction     | OKX AI Agent (AI memory)             | OKX AI Agent                  | Supports multi-turn Telegram interactions and state tracking.                                                        |
| Splits message is more than 4000 characters | Code                              | Splits long messages into 4000-char chunks          | OKX AI Agent                       | Telegram                       | Ensures Telegram message size limits are respected.                                                                   |
| Telegram                     | Telegram (Send Message)             | Sends formatted messages to Telegram user           | Splits message is more than 4000 characters | None                         | Sends structured market reports to the user.                                                                          |
| OpenAI Chat Model              | LangChain OpenAI Chat Model         | GPT-4o-mini model for reasoning and text generation | OKX AI Agent (ai_languageModel)     | OKX AI Agent                  | Used to interpret signals and produce structured output.                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials**  
   - Register a Telegram bot with BotFather.  
   - Generate and save the Telegram Bot API token.  
   - In n8n, create Telegram API credentials with this token.

2. **Create OpenAI API Credentials**  
   - Obtain OpenAI API key with access to GPT-4o-mini or GPT-4.1-mini model.  
   - Add OpenAI API credentials in n8n.

3. **Set up Telegram Trigger Node**  
   - Create a Telegram Trigger node.  
   - Configure to listen for `message` updates.  
   - Connect to Telegram API credentials.  
   - Deploy webhook to receive Telegram messages.

4. **Add User Authentication Code Node**  
   - Add a Code node after Telegram Trigger.  
   - Paste JavaScript code to verify `message.from.id` against your Telegram user ID.  
   - If unauthorized, return `{unauthorized:true}` to stop workflow.  
   - Connect Telegram Trigger → User Authentication.

5. **Create "Adds SessionId" Set Node**  
   - Add a Set node.  
   - Assign `sessionId` to `{{$json["message"]["chat"]["id"]}}`.  
   - Assign `message` to `{{$json["message"]["text"]}}`.  
   - Connect User Authentication → Adds "SessionId".

6. **Create OKX AI Agent Node**  
   - Add LangChain Agent node.  
   - Set model to GPT-4o-mini (or gpt-4.1-mini).  
   - Paste system prompt defining agent as OKX Spot Market Data Agent with detailed OKX API endpoint rules and output format requirements.  
   - Connect Adds "SessionId" → OKX AI Agent.

7. **Add HTTP Request Tool Nodes for OKX API Endpoints**  
   - Add nodes for each OKX endpoint used by the agent:  
     - 24h Stats (GET /api/v5/market/ticker with `instId`)  
     - Order Book Depth (GET /api/v5/market/books with `instId`, `sz`)  
     - Price (Latest) (GET /api/v5/market/ticker with `instId`)  
     - Best Bid/Ask (GET /api/v5/market/ticker with `instId`)  
     - Klines (Candles) (GET /api/v5/market/candles with `instId`, `bar`, `limit`)  
     - Average / Mark Price (GET /api/v5/market/mark-price with `instType=SPOT`, `instId`)  
     - Recent Trades (GET /api/v5/market/trades with `instId`, `limit`)  
   - Configure each node with dynamic query parameters injected by the AI agent via `$fromAI` expressions.  
   - Connect all these nodes as AI tools to the OKX AI Agent node.

8. **Add Calculator and Think Tool Nodes**  
   - Add Calculator node to perform math operations on market data.  
   - Add Think node for JSON reshaping and formatting assistance.  
   - Connect both as AI tools to the OKX AI Agent.

9. **Add Simple Memory Node**  
   - Add LangChain Memory Buffer Window node.  
   - Connect it as AI memory to the OKX AI Agent to store session state.

10. **Add OpenAI Chat Model Node**  
    - Add LangChain OpenAI Chat Model node with GPT-4o-mini.  
    - Connect as AI language model to the OKX AI Agent.

11. **Add Message Splitting Code Node**  
    - Add a Code node to split outgoing messages longer than 4000 characters into smaller chunks.  
    - Paste JavaScript splitting logic provided.  
    - Connect OKX AI Agent → Splitting Node.

12. **Add Telegram Send Message Node**  
    - Add Telegram node (send message).  
    - Configure `text` to receive chunked messages from splitter node.  
    - Set `chatId` dynamically from the original Telegram Trigger message chat ID.  
    - Connect Splitting Node → Telegram Send Message node.  
    - Use the same Telegram API credentials.

13. **Test the Workflow**  
    - Deploy all nodes.  
    - Send messages to the Telegram bot from the authorized user ID.  
    - Confirm receipt of structured market data reports.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow is part of a production-ready AI automation system providing real-time OKX Spot market data with formatting optimized for Telegram delivery. It requires installing and activating several supporting workflows for complete functionality.                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | See Sticky Note17 for full system documentation overview.                                         |
| Always replace the placeholder in the User Authentication node with your actual Telegram user ID to secure access.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | User Authentication Code Node instruction.                                                        |
| OKX API endpoints require instrument IDs in uppercase with a hyphen, e.g., "BTC-USDT".                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Multiple sticky notes detailing OKX API parameter requirements.                                  |
| The Telegram message size limit is approximately 4000 characters; messages exceeding this length are automatically split by the workflow to avoid errors.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note and Code Node for splitting messages.                                                |
| The AI agent enforces strict rules: it must not provide trading advice or predictions, only fetch and present clean market data. Missing or failed data responses are replaced with "N/A".                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | System message prompt in OKX AI Agent node.                                                       |
| For support and licensing, contact Don Jayamaha via LinkedIn: [linkedin.com/in/donjayamahajr](http://linkedin.com/in/donjayamahajr) - © 2025 Treasurium Capital Limited Company. The workflow and architecture are proprietary.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Sticky Note17 - Licensing and contact information.                                               |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, adhering strictly to content policies. It contains no illegal, offensive, or protected elements. All data handled is legal and public.

---