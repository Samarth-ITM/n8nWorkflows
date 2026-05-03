Real-time Crypto Market Data from Gate.io with GPT-4o and Telegram

https://n8nworkflows.xyz/workflows/real-time-crypto-market-data-from-gate-io-with-gpt-4o-and-telegram-8608


# Real-time Crypto Market Data from Gate.io with GPT-4o and Telegram

### 1. Workflow Overview

This n8n workflow, titled **"Real-time Crypto Market Data from Gate.io with GPT-4o and Telegram"**, serves as an AI-powered agent that retrieves real-time spot market data from the Gate.io cryptocurrency exchange via its REST v4 API and delivers clean, formatted reports to an authenticated Telegram user. It utilizes OpenAI's GPT-4o-mini model for message structuring and formatting, ensuring output is concise, human-readable, and suitable for Telegram's messaging constraints.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & User Validation**  
  Listens for incoming Telegram messages, authenticates the user, and extracts session metadata.

- **1.2 Data Retrieval & Orchestration (Gate AI Agent)**  
  The core orchestrator node calls multiple Gate.io public API endpoints in parallel, fetching various market data components.

- **1.3 Data Processing & Formatting via AI**  
  Utilizes the OpenAI Chat Model to interpret and format raw API data into a structured Telegram message, assisted by auxiliary tools like Calculator and Think nodes.

- **1.4 Output Handling & Delivery**  
  Checks message length against Telegram limits, splits if necessary, and sends the final formatted report back to the authenticated Telegram user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & User Validation

**Overview:**  
This block listens for incoming Telegram messages, validates the sender's Telegram ID against a predefined authorized user ID, and appends session metadata for downstream processing.

**Nodes Involved:**  
- Telegram Trigger  
- User Authentication (Replace Telegram ID)  
- Adds "SessionId"  

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for new Telegram messages (only "message" update type)  
  - Configuration: Uses webhook with a unique webhook ID; credentials configured for the Telegram bot  
  - Inputs: None (trigger node)  
  - Outputs: Raw Telegram message JSON  
  - Failure modes: Telegram API connectivity issues, webhook misconfiguration  
  - Sticky Note: Explains its role as the entry point for user commands  

- **User Authentication (Replace Telegram ID)**  
  - Type: Code (JavaScript)  
  - Role: Validates if the incoming message is from the authorized Telegram user; filters out unauthorized requests  
  - Configuration: Compares `$input.first().json.message.from.id` to a hardcoded Telegram user ID (requires manual replacement)  
  - Inputs: Telegram Trigger output  
  - Outputs: Passes original data if authorized; returns `{unauthorized: true}` if not  
  - Failure modes: Expression errors if Telegram ID not replaced; unauthorized users cause workflow termination  
  - Sticky Note: Notes the critical user validation step  

- **Adds "SessionId"**  
  - Type: Set  
  - Role: Extracts Telegram chat ID as `sessionId` and retains the message text for context  
  - Configuration: Assigns `sessionId` = `message.chat.id` and `message` = original text  
  - Inputs: User Authentication output  
  - Outputs: JSON enriched with session metadata  
  - Failure modes: Missing or malformed Telegram message fields  
  - Sticky Note: Describes session ID creation for memory and routing  

---

#### 2.2 Data Retrieval & Orchestration (Gate AI Agent)

**Overview:**  
This core block orchestrates parallel fetching of spot market data from Gate.io’s REST v4 API via multiple HTTP Request Tool nodes. The Gate AI Agent node manages these API calls and integrates their results.

**Nodes Involved:**  
- Gate AI Agent  
- 24h Stats  
- Price (Latest)  
- Order Book Depth  
- Best Bid/Ask  
- Klines (Candles)  
- Recent Trades  
- 24h Stats (Ticker)1  
- Calculator  
- Think  
- Simple Memory  

**Node Details:**

- **Gate AI Agent**  
  - Type: Langchain Agent  
  - Role: Orchestrates API calls, handles AI reasoning, and enforces rules for data fetching and output formatting  
  - Configuration: Receives user message text, uses system prompt specifying role as Gate.io Spot Market Data Agent with HTTP GET access to Gate.io API endpoints, forbidding prediction or analysis  
  - Inputs: Session metadata and user message  
  - Outputs: Aggregated raw data and formatted message candidate  
  - Failure modes: API downtime, malformed user input, AI model errors, rate limits on OpenAI or Gate.io  
  - Sticky Note: Detailed system prompt and operational rules, emphasizing no analysis, only data presentation  

- **24h Stats**  
  - Type: HTTP Request Tool  
  - Role: Calls `/spot/tickers` endpoint to fetch 24-hour stats including last price, volume, high/low, change%  
  - Configuration: Query param `currency_pair` from AI input or default `BTC_USDT`  
  - Inputs: From Gate AI Agent via AI tool connection  
  - Outputs: JSON with price and volume stats  
  - Failure modes: API errors, invalid currency pairs  

- **Price (Latest)**  
  - Type: HTTP Request Tool  
  - Role: Fetches latest trade price from `/spot/tickers` endpoint  
  - Configuration: Same as 24h Stats for `currency_pair`  
  - Inputs/Outputs: Similar to 24h Stats  

- **Order Book Depth**  
  - Type: HTTP Request Tool  
  - Role: Retrieves order book bids and asks up to a limit (default 100) using `/spot/order_book`  
  - Configuration: Params include `currency_pair`, `limit` (default 100), `with_id=true`  
  - Inputs/Outputs: Connected to Gate AI Agent as AI tool  

- **Best Bid/Ask**  
  - Type: HTTP Request Tool  
  - Role: Fetches best bid and ask prices by calling `/spot/order_book` with `limit=1`  
  - Configuration: Same as Order Book but fixed limit=1  
  - Inputs/Outputs: AI tool connection  

- **Klines (Candles)**  
  - Type: HTTP Request Tool  
  - Role: Retrieves OHLCV candlestick data from `/spot/candlesticks`  
  - Configuration: `currency_pair`, `interval` (default `15m`), `limit` (default 20) via AI input  
  - Inputs/Outputs: AI tool connection  

- **Recent Trades**  
  - Type: HTTP Request Tool  
  - Role: Gets recent public trades from `/spot/trades` endpoint  
  - Configuration: `currency_pair`, `limit` (default 100)  
  - Inputs/Outputs: AI tool connection  

- **24h Stats (Ticker)1**  
  - Type: HTTP Request Tool  
  - Role: Additional ticker stats variant from `/spot/tickers` for redundancy or extended data  
  - Configuration: Same as 24h Stats  
  - Inputs/Outputs: AI tool connection  

- **Calculator**  
  - Type: Langchain Tool Calculator  
  - Role: Performs math on fetched data, e.g., calculates spreads, midpoints, percentage changes  
  - Inputs: Receives data from Gate AI Agent  
  - Outputs: Calculated numeric values for formatting  
  - Failure modes: Incorrect input data types may cause calculation errors  
  - Sticky Note: Explains usage for spread and % change calculations  

- **Think**  
  - Type: Langchain Tool Think  
  - Role: AI reasoning helper that reshapes JSON, formats data, and prepares final message structure  
  - Inputs: From Gate AI Agent  
  - Outputs: Cleaned and formatted data for Telegram message  
  - Failure modes: AI model errors or malformed input  
  - Sticky Note: Detailed description of its function as a lightweight reasoning helper  

- **Simple Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Stores session context including sessionId and symbol for multi-turn conversation memory  
  - Inputs: Session metadata and ongoing conversation data  
  - Outputs: Provides context to Gate AI Agent for stateful interaction  
  - Failure modes: Memory overflow or loss if session data is too large  
  - Sticky Note: Describes use for multi-turn Telegram interaction  

---

#### 2.3 Output Handling & Delivery

**Overview:**  
This block ensures the final AI-generated message meets Telegram's message length constraints, splits it if necessary, and sends the report back to the authorized user.

**Nodes Involved:**  
- Splits message is more than 4000 characters  
- Send a text message  

**Node Details:**

- **Splits message is more than 4000 characters**  
  - Type: Code (JavaScript)  
  - Role: Checks if the outgoing message exceeds Telegram’s 4000 character limit; splits into safe chunks if needed  
  - Configuration: Splits the string stored in `$json.output` into 4000 character chunks  
  - Inputs: Output from Gate AI Agent (final formatted message)  
  - Outputs: One or multiple message chunks as separate items  
  - Failure modes: Input message missing or null; improper splitting logic  
  - Sticky Note: Notes handling Telegram message size limits  

- **Send a text message**  
  - Type: Telegram  
  - Role: Sends the final formatted HTML message chunks back to the Telegram chat using the bot  
  - Configuration: Uses chat ID from Telegram Trigger; appends attribution automatically  
  - Inputs: Message chunks from split node  
  - Outputs: None (terminal node)  
  - Failure modes: Telegram API errors, invalid chat ID, rate limits  
  - Sticky Note: Explains final delivery to user  

---

### 3. Summary Table

| Node Name                         | Node Type                         | Functional Role                             | Input Node(s)                       | Output Node(s)                        | Sticky Note                                                                                           |
|----------------------------------|----------------------------------|---------------------------------------------|-----------------------------------|-------------------------------------|-----------------------------------------------------------------------------------------------------|
| Telegram Trigger                 | Telegram Trigger                 | Listens for incoming Telegram messages      | None                              | User Authentication                  | Listens for new Telegram messages from users; triggers full agent process                           |
| User Authentication (Replace Telegram ID) | Code                             | Validates authorized Telegram user          | Telegram Trigger                  | Adds "SessionId"                    | Checks incoming Telegram ID against approved user list                                              |
| Adds "SessionId"                | Set                              | Adds session metadata (sessionId, message) | User Authentication              | Gate AI Agent                      | Creates sessionId using Telegram chat_id for memory and workflow routing                            |
| Gate AI Agent                   | Langchain Agent                  | Orchestrates API calls and AI message formatting | Adds "SessionId"                 | Splits message is more than 4000 characters | Core orchestrator; fetches Gate.io data, formats output with OpenAI GPT-4o-mini                      |
| 24h Stats                      | HTTP Request Tool                | Calls Gate.io `/spot/tickers` for 24h stats | Gate AI Agent                   | Gate AI Agent                      | Fetches 24-hour market stats for currency pair                                                      |
| Price (Latest)                 | HTTP Request Tool                | Calls Gate.io `/spot/tickers` for latest price | Gate AI Agent                   | Gate AI Agent                      | Retrieves latest trade price                                                                        |
| Order Book Depth               | HTTP Request Tool                | Calls Gate.io `/spot/order_book` for order book depth | Gate AI Agent                   | Gate AI Agent                      | Returns order book bids and asks up to limit                                                       |
| Best Bid/Ask                  | HTTP Request Tool                | Calls `/spot/order_book?limit=1` for best bid/ask | Gate AI Agent                   | Gate AI Agent                      | Provides top of book bid and ask prices                                                             |
| Klines (Candles)              | HTTP Request Tool                | Calls `/spot/candlesticks` for OHLCV data   | Gate AI Agent                   | Gate AI Agent                      | Retrieves candlestick data for currency pair                                                        |
| Recent Trades                 | HTTP Request Tool                | Calls `/spot/trades` for recent trades       | Gate AI Agent                   | Gate AI Agent                      | Fetches most recent 100 trades                                                                      |
| 24h Stats (Ticker)1           | HTTP Request Tool                | Variant call to `/spot/tickers` for ticker stats | Gate AI Agent                   | Gate AI Agent                      | Additional ticker stats endpoint variant                                                           |
| Calculator                   | Langchain Tool Calculator        | Performs math operations on market data      | Gate AI Agent                   | Gate AI Agent                      | Calculates spreads, midpoints, % changes                                                           |
| Think                        | Langchain Tool Think             | AI reasoning helper for data formatting      | Gate AI Agent                   | Gate AI Agent                      | Reshapes JSON and formats final Telegram message                                                   |
| Simple Memory                | Langchain Memory Buffer Window   | Stores session context for multi-turn memory | Adds "SessionId"                | Gate AI Agent                     | Stores sessionId, currency pair, and state data                                                    |
| Splits message is more than 4000 characters | Code                             | Splits output message if >4000 characters    | Gate AI Agent                   | Send a text message                 | Ensures Telegram message length limits are respected                                               |
| Send a text message          | Telegram                        | Sends formatted message to Telegram user     | Splits message is more than 4000 characters | None                             | Sends final HTML report (or split chunks) to authenticated user                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure: Select "message" updates only; set webhook ID  
   - Credentials: Link your Telegram Bot API credentials  

2. **Create User Authentication node (Code node)**  
   - Type: Code  
   - Paste JS code to compare incoming Telegram ID with your authorized user ID:  
     ```javascript
     if ($input.first().json.message.from.id !== YOUR_TELEGRAM_ID) {
       return { unauthorized: true };
     } else {
       return $input.all();
     }
     ```  
   - Connect output of Telegram Trigger → User Authentication  

3. **Create Adds "SessionId" node (Set node)**  
   - Type: Set  
   - Assign two fields:  
     - `sessionId` = `{{$json.message.chat.id}}`  
     - `message` = `{{$json.message.text}}`  
   - Connect User Authentication → Adds "SessionId"  

4. **Create Simple Memory node**  
   - Type: Langchain Memory Buffer Window  
   - No special parameters; uses default for short-term memory  
   - Connect Adds "SessionId" → Simple Memory  

5. **Create Gate AI Agent node**  
   - Type: Langchain Agent  
   - Parameters:  
     - Input text: `{{$json.message}}`  
     - System prompt defines role as Gate.io Spot Market Data Agent with HTTP GET access to Gate.io REST v4 API endpoints  
     - Rules: Only fetch data, do not analyze or predict; output clean text  
   - Credentials: OpenAI API key with GPT-4o-mini access  
   - Connect Adds "SessionId" → Gate AI Agent  
   - Connect Simple Memory → Gate AI Agent (memory input)  

6. **Create HTTP Request Tool nodes for each Gate.io endpoint:**  
   For each:  
   - URL: `https://api.gateio.ws/api/v4/<endpoint>`  
   - Method: GET  
   - Query Parameters with dynamic values from AI inputs (`$fromAI` expressions) for `currency_pair`, `interval`, `limit`, etc.  
   - Endpoints:  
     - `/spot/tickers` (24h Stats)  
     - `/spot/tickers` (Price Latest)  
     - `/spot/order_book` (Order Book Depth)  
     - `/spot/order_book` with `limit=1` (Best Bid/Ask)  
     - `/spot/candlesticks` (Klines)  
     - `/spot/trades` (Recent Trades)  
     - `/spot/tickers` (24h Stats Ticker variant)  
   - Connect all these nodes as AI tools inputs into Gate AI Agent  

7. **Create Calculator node**  
   - Type: Langchain Calculator Tool  
   - No special configuration; will receive inputs for computing spreads, midpoints, and % changes  
   - Connect as AI tool input to Gate AI Agent  

8. **Create Think node**  
   - Type: Langchain Think Tool  
   - Description: Used for intermediate AI processing and formatting  
   - Connect as AI tool input to Gate AI Agent  

9. **Create Code node named "Splits message is more than 4000 characters"**  
   - Type: Code  
   - Paste JS code that splits the message string on 4000 character boundaries:  
     ```javascript
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
   - Connect Gate AI Agent main output → this node  

10. **Create Telegram node "Send a text message"**  
    - Type: Telegram  
    - Text: `{{$json.message}}`  
    - Chat ID: `{{$node["Telegram Trigger"].json.message.chat.id}}`  
    - Enable HTML formatting and append attribution  
    - Credentials: Telegram Bot API credentials  
    - Connect splitter node → this node  

11. **Set credentials:**  
    - OpenAI API credentials for GPT-4o-mini (OpenAI account)  
    - Telegram API credentials for your bot token  
    - Replace Telegram ID in User Authentication node with your authorized user ID  

12. **Activate the workflow** and test by sending a message to the Telegram bot. The bot should respond with formatted Gate.io spot market data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow is built for production-ready real-time Gate.io Spot Market data delivery via Telegram with AI-enhanced formatting. It strictly forbids analysis, predictions, or advice, focusing solely on clean data retrieval and presentation.                                                                                                                                                 | Workflow purpose and design principle                                                             |
| The system uses OpenAI GPT-4o-mini for natural language formatting and reasoning, with auxiliary tools (Calculator, Think) for data reshaping and math operations.                                                                                                                                                                                                                            | AI tools and model versions                                                                        |
| Telegram message length limit is 4096 characters; a custom code node splits messages exceeding 4000 chars into multiple parts for safe delivery.                                                                                                                                                                                                                                           | Telegram message constraints                                                                       |
| All Gate.io endpoints used are public and require no authentication, simplifying API calls. The currency pair format is `BASE_QUOTE` with an underscore, e.g., `BTC_USDT`.                                                                                                                                                                                                                    | Gate.io API details                                                                                |
| User authentication is implemented by hardcoding the authorized Telegram user ID in a code node; this should be replaced with the actual user ID for security.                                                                                                                                                                                                                             | User authentication method                                                                         |
| The workflow includes detailed sticky notes for each critical step, including API endpoint documentation and usage notes, aiding maintainability and onboarding.                                                                                                                                                                                                                            | In-workflow documentation                                                                         |
| For support and licensing, contact Don Jayamaha on LinkedIn: [linkedin.com/in/donjayamahajr](http://linkedin.com/in/donjayamahajr) — The system and its components are proprietary to Treasurium Capital Limited Company with all rights reserved.                                                                                                                                                | Support and licensing information                                                                 |

---

**Disclaimer:**  
The provided documentation is based exclusively on an automated n8n workflow designed for legal and public data. It complies with content policies and contains no illegal or protected material.