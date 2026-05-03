Get Real-time Crypto Market Data from Bybit with GPT-4.1-mini in Telegram

https://n8nworkflows.xyz/workflows/get-real-time-crypto-market-data-from-bybit-with-gpt-4-1-mini-in-telegram-8610


# Get Real-time Crypto Market Data from Bybit with GPT-4.1-mini in Telegram

### 1. Workflow Overview

This workflow, titled **"Get Real-time Crypto Market Data from Bybit with GPT-4.1-mini in Telegram"**, serves as an intelligent Telegram bot that fetches real-time cryptocurrency spot market data from the Bybit exchange via its REST v5 API and returns structured, human-readable reports to authorized Telegram users. It leverages GPT-4.1-mini to format and present the gathered data cleanly in Telegram messages without performing any market analysis or prediction.

The workflow is logically divided into the following blocks:

- **1.1 Telegram Input & User Authentication:** Listens for user commands on Telegram and validates user identity.
- **1.2 Session Management:** Assigns session metadata based on Telegram chat context.
- **1.3 Memory Buffer:** Maintains short-term session memory for multi-turn interactions.
- **1.4 Bybit Data Fetching Agent:** Core AI agent orchestrating multiple Bybit API calls to fetch market data.
- **1.5 Data Processing & Formatting Tools:** Calculator and Think nodes to process raw API data and prepare it for presentation.
- **1.6 Message Splitting:** Ensures Telegram message length limits are respected by splitting long outputs.
- **1.7 Telegram Output:** Sends the finalized formatted report back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Input & User Authentication

**Overview:**  
This block listens for incoming Telegram messages and filters access by validating the Telegram user ID to authorize only approved users.

**Nodes Involved:**  
- Telegram Trigger  
- User Authentication (Replace Telegram ID)

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for new Telegram messages (update type: message) to start the workflow.  
  - Configuration: Uses an existing Telegram bot credential; listens for all messages.  
  - Inputs: External Telegram messages  
  - Outputs: Raw Telegram message JSON  
  - Failure modes: Telegram API downtime, webhook misconfiguration  
  - Sticky Note: Explains the trigger's purpose to start the agent on incoming messages.

- **User Authentication (Replace Telegram ID)**  
  - Type: Code node (JavaScript)  
  - Role: Checks if the Telegram sender's user ID matches the allowed ID; otherwise stops workflow.  
  - Configuration: Hardcoded Telegram ID to compare with incoming message sender ID.  
  - Key Expression: `if ($input.first().json.message.from.id !== <<Replace>>) { return {unauthorized: true}; }`  
  - Inputs: Telegram Trigger output  
  - Outputs: Passes data downstream only if authorized; else stops workflow silently.  
  - Failure modes: Misconfigured Telegram ID, unexpected message format  
  - Sticky Note: Describes user access validation role.

#### 1.2 Session Management

**Overview:**  
Generates session metadata based on Telegram chat ID, assigning a sessionId and extracting the user message text for downstream use.

**Nodes Involved:**  
- Adds "SessionId"

**Node Details:**

- **Adds "SessionId"**  
  - Type: Set node  
  - Role: Creates session variables: `sessionId` (Telegram chat id) and copies user message text.  
  - Configuration: Assigns `sessionId = $json.message.chat.id` and `message = $json.message.text`  
  - Inputs: User Authentication output  
  - Outputs: JSON with sessionId and message properties added  
  - Failure modes: Missing chat id or message text in input  
  - Sticky Note: Explains session metadata creation for routing and memory.

#### 1.3 Memory Buffer

**Overview:**  
Maintains short-term memory of the session context for multi-turn conversations or tracking symbol/state across messages.

**Nodes Involved:**  
- Simple Memory

**Node Details:**

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Stores `sessionId`, symbol, and other state data for conversation continuity.  
  - Configuration: Default buffer window, no special parameters set.  
  - Inputs: Session metadata plus AI Agent context  
  - Outputs: Memory context usable by AI Agent node  
  - Failure modes: Memory overflow, improper sessionId linkage  
  - Sticky Note: Describes memory role in multi-turn Telegram interactions.

#### 1.4 Bybit Data Fetching Agent

**Overview:**  
Central AI Agent node using GPT-4.1-mini to interpret user queries, orchestrate multiple HTTP API calls to Bybit Spot API, and format the gathered data into a clean Telegram-ready HTML text report.

**Nodes Involved:**  
- OpenAI Chat Model  
- Bybit AI Agent  
- Multiple HTTP Request Tool nodes (Bybit API endpoints)  
- Calculator (for math operations)  
- Think (for reasoning and data reshaping)

**Node Details:**

- **OpenAI Chat Model**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Processes AI language understanding and reasoning with GPT-4.1-mini.  
  - Configuration: Model set to `gpt-4.1-mini` with no extra options.  
  - Inputs: User message text from session metadata  
  - Outputs: Structured prompts and responses to/from Bybit AI Agent  
  - Failure modes: OpenAI API quota, invalid prompts  
  - Sticky Note: Notes model usage for reasoning and HTML generation.

- **Bybit AI Agent**  
  - Type: LangChain Agent  
  - Role: Core orchestrator executing multiple Bybit API calls (via attached tools), enforcing fetch-only behavior, and formatting output.  
  - Configuration: System prompt defines strict rules — fetch data only, no analysis or predictions, output clean Telegram HTML text.  
  - Inputs: Session message text, memory context, OpenAI language model  
  - Outputs: Combined formatted report text  
  - Failure modes: API connection errors, invalid user input symbols, timeout, partial data returns  
  - Sticky Note: Extensive documentation on Bybit API endpoints used, rules, and output format.

- **HTTP Request Tool Nodes (Bybit API calls):**  
  Each node calls a specific Bybit endpoint with parameters dynamically extracted from AI input variables:  

  - *24h Stats (Ticker):* `/v5/market/tickers?category=spot&symbol=...`  
  - *Price (Latest):* `/v5/market/tickers` (same endpoint, focused on latest price)  
  - *Order Book Depth:* `/v5/market/orderbook?category=spot&symbol=...&limit=...`  
  - *Best Bid/Ask:* `/v5/market/orderbook?limit=1`  
  - *Klines (Candles):* `/v5/market/kline` with interval, limit, start/end  
  - *Ticker (Latest & Stats):* `/v5/market/tickers` (similar to 24h stats)  
  - *Recent Trades:* `/v5/market/recent-trade` with limit  

  - Role: Fetch fresh market data from Bybit to supply to AI agent  
  - Inputs: Symbol, interval, limit parameters dynamically set by AI agent variables  
  - Outputs: Raw JSON market data  
  - Failure modes: API rate limits, invalid symbols, missing parameters, network failures  
  - Sticky Notes: Detailed explanation of each endpoint’s parameters, expected returns, and usage.

- **Calculator**  
  - Type: LangChain Tool Calculator  
  - Role: Perform inline math (spreads, midpoints, percentages) on Bybit API data for formatting  
  - Inputs: Raw numeric fields from API responses  
  - Outputs: Computed values for report  
  - Failure modes: Invalid numeric inputs, division by zero  
  - Sticky Note: Explains calculator’s role for math operations in workflow.

- **Think**  
  - Type: LangChain Tool Think  
  - Role: Lightweight reasoning helper to reshape JSON, extract necessary fields, and prepare data for final Telegram message formatting.  
  - Inputs: Raw API responses  
  - Outputs: Cleaned and structured data for reporting  
  - Failure modes: Expression errors, invalid input data  
  - Sticky Note: Describes Think node purpose as an AI tool without API calls.

#### 1.5 Message Splitting

**Overview:**  
Ensures that if the AI’s formatted output exceeds Telegram’s message character limit (~4096), it is split into sequential chunks safely.

**Nodes Involved:**  
- Splits message is more than 4000 characters (Code node)

**Node Details:**

- **Splits message is more than 4000 characters**  
  - Type: Code node (JavaScript)  
  - Role: Checks message length and splits into ≤4000 character chunks if needed.  
  - Configuration: Splits input string from AI output into array of message chunks.  
  - Inputs: Formatted report text from Bybit AI Agent  
  - Outputs: Array of message parts for sequential Telegram sending  
  - Failure modes: Unexpected input format, Unicode character splitting edge cases  
  - Sticky Note: Explains Telegram’s message length limits and chunking logic.

#### 1.6 Telegram Output

**Overview:**  
Sends the final formatted (and split, if needed) HTML report messages back to the authenticated user on Telegram.

**Nodes Involved:**  
- Telegram (Send Message)

**Node Details:**

- **Telegram**  
  - Type: Telegram node (Send Message)  
  - Role: Sends text messages with HTML formatting to the user’s Telegram chat ID.  
  - Configuration: Uses same Telegram bot credentials; disables attribution; sends to chat ID from original message.  
  - Inputs: Message chunks from splitter node  
  - Outputs: Confirmation of message sent  
  - Failure modes: Telegram API errors, network issues, invalid chat id  
  - Sticky Note: States this node sends final report to user.

---

### 3. Summary Table

| Node Name                              | Node Type                               | Functional Role                          | Input Node(s)                         | Output Node(s)                        | Sticky Note                                                                                                                   |
|--------------------------------------|---------------------------------------|----------------------------------------|-------------------------------------|-------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger                     | Telegram Trigger                      | Listens for new Telegram messages      | -                                   | User Authentication                 | Listens for new Telegram messages from users; triggers full agent process and passes raw user input downstream.               |
| User Authentication (Replace Telegram ID) | Code (JavaScript)                     | Validates Telegram user ID             | Telegram Trigger                    | Adds "SessionId"                   | Checks incoming Telegram ID against approved user list.                                                                       |
| Adds "SessionId"                    | Set                                   | Assigns sessionId and extracts message | User Authentication                 | Bybit AI Agent                    | Creates sessionId using Telegram chat id; passes message downstream.                                                          |
| Simple Memory                      | LangChain Memory Buffer Window        | Maintains session state/memory         | Bybit AI Agent                     | Bybit AI Agent                    | Stores sessionId, symbol, and other state data for multi-turn Telegram interactions.                                           |
| OpenAI Chat Model                   | LangChain LM Chat OpenAI               | Processes AI language understanding   | Bybit AI Agent                     | Bybit AI Agent                    | GPT-4.1-mini used for reasoning, formatting, and recommending trades.                                                         |
| Bybit AI Agent                     | LangChain Agent                      | Core orchestrator fetching Bybit data | Adds "SessionId", Simple Memory, OpenAI Chat Model, HTTP Request Tools, Calculator, Think | Splits message is more than 4000 characters | Fetches multiple Bybit API endpoints; formats output into Telegram HTML; no analysis or predictions.                            |
| 24h Stats1                        | HTTP Request Tool                     | Fetches 24h stats from Bybit API       | Bybit AI Agent                     | Bybit AI Agent                    | Returns latest price and 24h stats for a Spot symbol.                                                                          |
| Price (Latest)1                   | HTTP Request Tool                     | Fetches latest price from Bybit API    | Bybit AI Agent                     | Bybit AI Agent                    | Returns latest price and 24h stats for a symbol.                                                                               |
| Order Book Depth1                | HTTP Request Tool                     | Fetches order book depth                | Bybit AI Agent                     | Bybit AI Agent                    | Returns order book bids/asks up to specified limit for a symbol.                                                               |
| Best Bid/Ask1                   | HTTP Request Tool                     | Fetches best bid/ask (top of book)     | Bybit AI Agent                     | Bybit AI Agent                    | Returns best bid and ask prices with sizes.                                                                                    |
| Klines (Candles)1               | HTTP Request Tool                     | Fetches candlestick OHLCV data         | Bybit AI Agent                     | Bybit AI Agent                    | Returns candlestick bars for symbol and interval.                                                                              |
| Ticker (Latest & Stats)          | HTTP Request Tool                     | Fetches latest ticker stats             | Bybit AI Agent                     | Bybit AI Agent                    | Returns latest price, best bid/ask, 24h high/low, % change, and volume.                                                       |
| Recent Trades1                  | HTTP Request Tool                     | Fetches recent public trades            | Bybit AI Agent                     | Bybit AI Agent                    | Returns recent trades for a symbol with details.                                                                               |
| Calculator                      | LangChain Tool Calculator             | Performs math operations on data       | Bybit AI Agent                     | Bybit AI Agent                    | Calculates spreads, percentages, midpoints, etc.                                                                               |
| Think                          | LangChain Tool Think                  | Performs lightweight reasoning         | Bybit AI Agent                     | Bybit AI Agent                    | Helps reshape JSON, extract fields, and prepare data for reporting.                                                           |
| Splits message is more than 4000 characters | Code (JavaScript)                    | Splits long messages into chunks       | Bybit AI Agent                    | Telegram                         | Checks if message exceeds 4000 chars; splits to comply with Telegram limits.                                                  |
| Telegram                      | Telegram (Send Message)                | Sends formatted messages to user       | Splits message is more than 4000 characters | -                               | Sends final formatted HTML report (or message chunks) to authenticated Telegram user.                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram Bot API credentials (OAuth2 or token)  
   - Set to listen for `message` updates only  
   - Position at the start of the workflow

2. **Create User Authentication node (Code node)**  
   - Add a Code node connected from Telegram Trigger  
   - JavaScript code:  
     ```js
     if ($input.first().json.message.from.id !== <<YourTelegramUserID>>) {
       return { unauthorized: true };
     } else {
       return $input.all();
     }
     ```
   - Replace `<<YourTelegramUserID>>` with the Telegram user ID allowed to use the bot

3. **Create Adds "SessionId" node (Set node)**  
   - Connect from User Authentication  
   - Assign two fields:  
     - `sessionId` = `{{$json["message"]["chat"]["id"]}}`  
     - `message` = `{{$json["message"]["text"]}}`  
   - This prepares session context for downstream nodes

4. **Create Simple Memory node (LangChain Memory Buffer Window)**  
   - Connect from Adds "SessionId"  
   - Use default settings for memory buffer  
   - Used to keep session context (symbol, interval, etc.)

5. **Create OpenAI Chat Model node (LangChain LM Chat OpenAI)**  
   - Connect from Simple Memory  
   - Configure with OpenAI API credentials (API key)  
   - Model: `gpt-4.1-mini`  
   - No additional options required

6. **Create multiple HTTP Request Tool nodes for Bybit API calls**  
   - For each of the following endpoints, create an HTTP Request Tool node with:  
     - URL set to respective Bybit endpoint (e.g., `https://api.bybit.com/v5/market/tickers`)  
     - Method: GET  
     - Query Parameters:  
       - `category` = `spot`  
       - `symbol` = `{{$fromAI('symbol', 'BTCUSDT', 'string')}}` (dynamic from AI input)  
       - Additional params per endpoint (e.g., `limit`, `interval`) also set dynamically via `$fromAI` expressions  
   - Endpoints to create:  
     - 24h Stats (Ticker)  
     - Price (Latest)  
     - Order Book Depth  
     - Best Bid/Ask (limit=1)  
     - Klines (Candles)  
     - Ticker (Latest & Stats)  
     - Recent Trades  

7. **Create Calculator node (LangChain Tool Calculator)**  
   - Connect from HTTP nodes as needed  
   - Used for computations like spreads, midpoints, percentages  
   - No special configuration needed; use inputs from previous nodes

8. **Create Think node (LangChain Tool Think)**  
   - Connect from Calculator and HTTP nodes  
   - Used to reshape data and prepare for reporting  
   - No API calls, just reasoning and formatting

9. **Create Bybit AI Agent node (LangChain Agent)**  
   - Connect from Adds "SessionId", Simple Memory, OpenAI Chat Model, HTTP Request Tools, Calculator, and Think nodes as AI tools and memories  
   - Configure with a system prompt:  
     ```
     You are the Bybit Spot Market Data Agent...
     [Use the detailed system message from the original workflow]
     ```
   - Set promptType: define  
   - This node orchestrates API calls and formatting

10. **Create Splits message is more than 4000 characters node (Code node)**  
    - Connect from Bybit AI Agent output  
    - JavaScript code:  
      ```js
      const input = $json.output;
      const chunkSize = 4000;
      function splitMessage(text, size) {
        const result = [];
        for (let i = 0; i < text.length; i += size) {
          result.push(text.substring(i, i + size));
        }
        return result;
      }
      if (input.length <= chunkSize) {
        return [{ json: { message: input } }];
      } else {
        const chunks = splitMessage(input, chunkSize);
        return chunks.map(chunk => ({ json: { message: chunk } }));
      }
      ```

11. **Create Telegram (Send Message) node**  
    - Connect from splitter node  
    - Use the same Telegram Bot credentials  
    - Send message text with HTML enabled (ensure `parse_mode` is set to `HTML`)  
    - Set chat ID dynamically: `{{$json.message.chat.id}}` from Telegram Trigger

12. **Connect all nodes according to dependencies:**  
    - Telegram Trigger → User Authentication → Adds "SessionId" → Bybit AI Agent → Splits message → Telegram Send  
    - Memory and OpenAI Chat Model linked as AI memory and language model to Bybit AI Agent  
    - HTTP Request nodes, Calculator, Think connected as AI tools to Bybit AI Agent

13. **Test the workflow**  
    - Trigger via Telegram message from authorized user  
    - Confirm data retrieval, formatting, and message delivery works as expected

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                     | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The workflow strictly fetches and formats Bybit Spot market data; it does not perform analysis, predictions, or trading advice.                                                                                                                | System prompt in Bybit AI Agent node                                                                     |
| Telegram message length limit is approximately 4096 characters; this workflow splits messages safely at 4000 characters to avoid Telegram API errors.                                                                                            | Code node “Splits message is more than 4000 characters”                                                  |
| Bybit REST v5 Spot API endpoints are used extensively; refer to official Bybit API docs for detailed parameter and response descriptions: https://bybit-exchange.github.io/docs/spot/v5/                                                        | Bybit API documentation                                                                                   |
| The workflow uses GPT-4.1-mini as a lightweight but advanced language model suitable for reasoning and formatting tasks.                                                                                                                        | OpenAI Chat Model node configuration                                                                     |
| Workflow creator: Don Jayamaha, Treasurium Capital Ltd. Proprietary architecture and prompts protected by U.S. copyright law.                                                                                                                  | [LinkedIn](http://linkedin.com/in/donjayamahajr)                                                         |
| The agent’s system prompt enforces strict rules to avoid fabricating data and to output clean Telegram HTML text, ensuring reliability and consistency in user interactions.                                                                      | System prompt content in Bybit AI Agent node                                                            |
| Sessions are tracked via Telegram chat IDs, enabling multi-turn conversations and stateful interactions, which is critical for complex or follow-up queries.                                                                                     | Adds "SessionId" node and Simple Memory node                                                             |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow built using n8n, an integration and automation tool. This process strictly adheres to content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.