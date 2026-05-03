Fetch Real-Time Coinbase Spot Market Data with GPT-4o + Telegram

https://n8nworkflows.xyz/workflows/fetch-real-time-coinbase-spot-market-data-with-gpt-4o---telegram-8606


# Fetch Real-Time Coinbase Spot Market Data with GPT-4o + Telegram

---

### 1. Workflow Overview

This workflow, titled **"Fetch Real-Time Coinbase Spot Market Data with GPT-4o + Telegram"**, is a professional-grade AI agent designed to deliver structured, real-time spot market data from Coinbase directly to an authenticated Telegram user. It targets use cases such as receiving up-to-date cryptocurrency market data, including price, order book, trades, and candlestick information, formatted cleanly for Telegram chat consumption.

The workflow is composed of the following logical blocks:

- **1.1 Input Reception and User Validation**  
  Listens for incoming Telegram messages from users, authenticates the user based on Telegram ID, and extracts session metadata.

- **1.2 AI Orchestration and Data Fetching**  
  Acts as the core agent that interprets user requests, fetches live Coinbase market data via multiple HTTP requests in parallel, and processes this data using AI tools for formatting and reasoning.

- **1.3 Data Processing and Output Preparation**  
  Performs calculations (e.g., mid-price), intermediate reasoning to format the data into human-readable HTML, and handles message size limits by splitting long messages.

- **1.4 Output Delivery**  
  Sends the formatted market data report back to the authenticated Telegram user via the Telegram bot.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and User Validation

- **Overview:**  
  This block listens for Telegram messages, verifies the sender's identity, and attaches session metadata (session ID and message text) for downstream processing.

- **Nodes Involved:**  
  - Telegram Trigger  
  - User Authentication (Replace Telegram ID)  
  - Adds "SessionId"  

- **Node Details:**

  - **Telegram Trigger**  
    - Type: `telegramTrigger`  
    - Role: Listens for incoming Telegram updates of type "message" to trigger the workflow.  
    - Configuration: Webhook ID assigned; uses Telegram Bot credentials named `"BinanceSpotTradingAIAgent_Bot"`.  
    - Inputs: None (webhook trigger)  
    - Outputs: Raw Telegram message JSON  
    - Edge Cases: Telegram API downtime, webhook misconfiguration, unauthorized users sending messages.

  - **User Authentication (Replace Telegram ID)**  
    - Type: `code` node (JavaScript)  
    - Role: Checks if the incoming Telegram user ID matches a predefined authorized ID; filters unauthorized users by returning `{unauthorized: true}` which effectively stops the flow.  
    - Configuration: Replace placeholder `<<Replace>>` with your actual Telegram user ID.  
    - Expressions: Accesses `$input.first().json.message.from.id` to compare.  
    - Input: Raw Telegram message JSON from Telegram Trigger.  
    - Output: Passes original data if authorized, else halts flow.  
    - Edge Cases: User ID not replaced correctly, multiple authorized users not supported out-of-the-box, message without `from` field.

  - **Adds "SessionId"**  
    - Type: `set` node  
    - Role: Extracts and assigns session metadata: sets `sessionId` as Telegram chat ID and copies the message text into a `message` field for clarity downstream.  
    - Configuration: Assigns `sessionId` from `json.message.chat.id`, and `message` from `json.message.text`.  
    - Input: Authorized Telegram message JSON.  
    - Output: JSON with added session metadata fields.  
    - Edge Cases: Messages without `text` field (e.g., stickers), chats that are not one-on-one (group chats).

---

#### 2.2 AI Orchestration and Data Fetching

- **Overview:**  
  The central block where the AI agent receives the user message and orchestrates multiple concurrent HTTP requests to Coinbase’s public REST API endpoints to fetch various market data. The agent uses GPT-4o-mini to parse and structure data, but does not generate predictions or strategies.

- **Nodes Involved:**  
  - Coinbase AI Agent (Langchain Agent)  
  - OpenAI Chat Model (gpt-4o-mini)  
  - Multiple HTTP Request Tool nodes (Price, Stats, Order Book, Trades, Candles, etc.)  
  - Calculator (for midpoint price calculation)  
  - Think (AI Tool node for intermediate reasoning)  
  - Simple Memory (Langchain memory buffer)  

- **Node Details:**

  - **Coinbase AI Agent**  
    - Type: Langchain Agent node  
    - Role: Core orchestrator interpreting user input, invoking multiple HTTP request tools to fetch market data, and combining results.  
    - Configuration:  
      - Input text bound to the `message` field from earlier nodes.  
      - System message defines strict rules: fetch and present only, no analysis or predictions.  
      - Uses several downstream HTTP request tools to fetch specific data endpoints.  
      - Binds parameters like `product_id` and `granularity` from AI variables.  
    - Input: Session-enhanced user message JSON.  
    - Outputs: Aggregated raw data JSON for processing.  
    - Edge Cases: API rate limits (429 errors), incomplete data, network failures, invalid product IDs, malformed user requests.

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model node  
    - Role: Provides reasoning and formatting capabilities for the AI agent; uses `gpt-4o-mini` model.  
    - Configuration: Default with no additional options, uses OpenAI API credentials.  
    - Input: Structured data from Coinbase AI Agent.  
    - Output: Structured text (HTML) for Telegram message.  
    - Edge Cases: API errors, rate limits, prompt formatting errors.

  - **HTTP Request Tool Nodes** (examples below, all linked to AI agent):  
    - **Price (Latest)**: Fetches latest trade price and best bid/ask.  
    - **24h Stats**: Fetches 24h open, high, low, last, volume.  
    - **Order Book Depth**: Fetches level 2 order book bids and asks.  
    - **Best Bid/Ask**: Fetches level 1 best bid/ask.  
    - **Klines (Candles)**: Fetches candlestick OHLCV data with configurable granularity.  
    - **Recent Trades**: Fetches recent trade history with optional limit.  
    - **Average Price (Derived)**: Fetches best bid/ask to calculate midpoint price via Calculator node.  
    - Each node uses dynamic URLs with `product_id` and other parameters bound from AI variables.  
    - Edge Cases: Network errors, invalid parameters, empty responses, endpoint changes.

  - **Calculator**  
    - Type: Langchain Calculator tool node  
    - Role: Computes mathematical derivations such as midpoint price from bid/ask, spreads, percent changes.  
    - Input: Data from HTTP requests.  
    - Output: Numerical values for further formatting.  
    - Edge Cases: Missing or invalid numeric input fields.

  - **Think**  
    - Type: Langchain Tool (non-API AI reasoning helper)  
    - Role: Formats and reshapes JSON data, extracts relevant fields, and prepares data for final Telegram message formatting.  
    - Input: Aggregated data from API calls and Calculator.  
    - Output: Clean HTML-formatted report string.  
    - Edge Cases: Incorrect field references, malformed JSON, unexpected API data.

  - **Simple Memory**  
    - Type: Langchain memory buffer window node  
    - Role: Stores session state such as sessionId and symbol for maintaining context in multi-turn conversations.  
    - Input: Session and symbol data.  
    - Output: Provides memory context to the agent.  
    - Edge Cases: Memory overflow, session ID mismatches.

---

#### 2.3 Data Processing and Output Preparation

- **Overview:**  
  This block handles the final message formatting and ensures Telegram message size limits are respected by splitting overly long messages into chunks.

- **Nodes Involved:**  
  - Splits message is more than 4000 characters (Code node)  

- **Node Details:**

  - **Splits message is more than 4000 characters**  
    - Type: Code node (JavaScript)  
    - Role: Checks if the AI-generated message exceeds Telegram's 4000 character limit. If yes, splits it into multiple smaller messages preserving order.  
    - Configuration:  
      - Takes input from the AI agent’s output message field.  
      - Splits string in 4000-character chunks.  
      - Returns array of message chunks for sequential sending.  
    - Input: Single long string message.  
    - Output: Array of message objects with chunked text.  
    - Edge Cases: Messages exactly at limit, empty strings, Unicode character splitting.

---

#### 2.4 Output Delivery

- **Overview:**  
  Sends the formatted (and possibly chunked) HTML report back to the Telegram user who initiated the request, completing the workflow cycle.

- **Nodes Involved:**  
  - Telegram (sendMessage node)  

- **Node Details:**

  - **Telegram**  
    - Type: `telegram` node (sendMessage)  
    - Role: Sends the final HTML-formatted market data report to the Telegram chat identified by the session ID.  
    - Configuration:  
      - Text parameter bound to the processed message chunk(s).  
      - Chat ID dynamically set from the original Telegram Trigger node’s chat ID.  
      - Attribute `appendAttribution` disabled to avoid extra bot signatures.  
      - Uses same Telegram Bot credentials as the trigger node.  
    - Input: Chunked message(s) from the splitter node.  
    - Output: Sends message(s) to Telegram chat.  
    - Edge Cases: Telegram API rate limits, chat ID mismatch, message size too large even after splitting.

---

### 3. Summary Table

| Node Name                          | Node Type                                   | Functional Role                            | Input Node(s)                          | Output Node(s)                         | Sticky Note                                                                                      |
|-----------------------------------|---------------------------------------------|--------------------------------------------|--------------------------------------|--------------------------------------|-------------------------------------------------------------------------------------------------|
| Telegram Trigger                  | telegramTrigger                             | Incoming Telegram messages trigger          | -                                    | User Authentication                   | ## Trigger Incoming Telegram Command Node: Telegram Trigger Listens for new Telegram messages.   |
| User Authentication (Replace Telegram ID) | code                                      | Validates Telegram user ID                   | Telegram Trigger                     | Adds "SessionId"                      | ## Validate User Access Node: User Authentication Checks incoming Telegram ID.                   |
| Adds "SessionId"                  | set                                         | Adds sessionId and message text              | User Authentication                  | Coinbase AI Agent                    | ## Generate Session Metadata Node: Adds SessionId from Telegram chat ID.                         |
| Coinbase AI Agent                | Langchain Agent                             | AI orchestrator fetching and combining data | Adds "SessionId"                    | Splits message is more than 4000 chars | ## Main AI Agent: Data Fetcher Node: Core orchestrator fetching Coinbase data and formatting.   |
| 24h Stats1                      | httpRequestTool                             | Fetches 24h market stats                      | Coinbase AI Agent                   | Coinbase AI Agent                   | See Sticky Note10: 24h Stats endpoint details.                                                  |
| Order Book Depth1               | httpRequestTool                             | Fetches L2 order book depth                   | Coinbase AI Agent                   | Coinbase AI Agent                   | See Sticky Note7: Order Book Depth endpoint details.                                           |
| Price (Latest)1                 | httpRequestTool                             | Fetches latest trade price                    | Coinbase AI Agent                   | Coinbase AI Agent                   | See Sticky Note11: Price (Latest) endpoint details.                                            |
| Best Bid/Ask1                  | httpRequestTool                             | Fetches best bid/ask (L1)                      | Coinbase AI Agent                   | Coinbase AI Agent                   | See Sticky Note8: Best Bid/Ask endpoint details.                                               |
| Klines (Candles)1              | httpRequestTool                             | Fetches candlestick OHLCV data                 | Coinbase AI Agent                   | Coinbase AI Agent                   | See Sticky Note12: Klines (Candles) endpoint details.                                          |
| Recent Trades1                 | httpRequestTool                             | Fetches recent trades                           | Coinbase AI Agent                   | Coinbase AI Agent                   | See Sticky Note14: Recent Trades endpoint details.                                            |
| Average Price (Derived)         | httpRequestTool                             | Fetches best bid/ask to derive average price   | Coinbase AI Agent                   | Coinbase AI Agent                   | See Sticky Note13: Average Price derived from best bid/ask.                                   |
| Calculator                     | Langchain Calculator tool                   | Performs math calculations (midpoint, spread) | Coinbase AI Agent                   | Coinbase AI Agent                   | See Sticky Note15: Calculator for math operations.                                            |
| Think                         | Langchain Tool (AI reasoning helper)        | Formats and reshapes data for Telegram output | Coinbase AI Agent                   | Splits message is more than 4000 chars | See Sticky Note16: Think node for reasoning and formatting.                                  |
| Simple Memory                  | Langchain Memory Buffer Window               | Maintains session context for multi-turn flow | Coinbase AI Agent                   | Coinbase AI Agent                   | See Sticky Note9: Short-Term Memory Module stores session and symbol state.                    |
| Splits message is more than 4000 characters | code                                      | Splits long messages to respect Telegram limits | Coinbase AI Agent                   | Telegram                           | ## Handle Telegram Message Limits Node: Splits messages >4000 chars.                          |
| Telegram                      | telegram                                    | Sends final formatted message(s) to Telegram | Splits message is more than 4000 chars | -                                | ## Send Final Report to Telegram Node: Sends formatted HTML report to user.                    |
| OpenAI Chat Model              | Langchain OpenAI Chat Model                   | AI model for reasoning and formatting          | Coinbase AI Agent                   | Coinbase AI Agent                   | ## GPT Model for Reasoning Node: Uses gpt-4o-mini to interpret and format data.                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: `telegramTrigger`  
   - Parameters: Listen to updates of type "message"  
   - Credentials: Use your Telegram Bot API credentials  
   - Purpose: To receive incoming Telegram messages

2. **Add Code Node for User Authentication**  
   - Type: `code` node  
   - Script:  
     ```javascript
     if ($input.first().json.message.from.id !== <<YourTelegramID>>) {
       return { unauthorized: true };
     } else {
       return $input.all();
     }
     ```  
   - Replace `<<YourTelegramID>>` with your actual Telegram user ID.  
   - Connect input from Telegram Trigger node.

3. **Add Set Node to Assign Session Metadata**  
   - Type: `set` node  
   - Assign fields:  
     - `sessionId` = `={{ $json.message.chat.id }}`  
     - `message` = `={{ $json.message.text }}`  
   - Connect input from User Authentication node.

4. **Add Langchain Agent Node (Coinbase AI Agent)**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Parameters:  
     - Text input: `={{ $json.message }}` (from previous node)  
     - System prompt: Use the detailed system message specifying the role as Coinbase Spot Market Data Agent, API access to Coinbase endpoints, strict rules to fetch and format data only, no predictions.  
     - Configure linked AI Tools and Memory nodes as described below.  
   - Connect input from Adds "SessionId" node.

5. **Create HTTP Request Tool Nodes for Coinbase Endpoints (all linked to AI agent as tools):**  
   For each node:  
   - Type: `httpRequestTool`  
   - Parameters:  
     - URL dynamically set as:  
       `https://api.exchange.coinbase.com/products/{{$fromAI('product_id','BTC-USD','string')}}/{endpoint}`  
     - Examples:  
       - Price (Latest): `/ticker`  
       - 24h Stats: `/stats`  
       - Order Book Depth: `/book?level=2`  
       - Best Bid/Ask: `/book?level=1`  
       - Klines (Candles): `/candles` with granularity parameter  
       - Recent Trades: `/trades` with optional limit  
       - Average Price (Derived): `/book?level=1`  
     - Bind query parameters dynamically from AI variables using `$fromAI`.  
   - Connect all these nodes as AI Tools under the Coinbase AI Agent node.

6. **Add Calculator Node**  
   - Type: `@n8n/n8n-nodes-langchain.toolCalculator`  
   - Purpose: Calculate midpoint price from best bid and ask prices obtained from Average Price (Derived) endpoint.  
   - Connect as AI Tool under Coinbase AI Agent.

7. **Add Think Node**  
   - Type: `@n8n/n8n-nodes-langchain.toolThink`  
   - Purpose: Intermediate reasoning and formatting of aggregated data into clean Telegram HTML format.  
   - Connect input from Coinbase AI Agent’s tool output.

8. **Add Simple Memory Node**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Purpose: Store session state (sessionId, symbol) for multi-turn conversation support.  
   - Connect as AI Memory for Coinbase AI Agent.

9. **Add Code Node for Message Splitting**  
   - Type: `code` node  
   - Script:  
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
   - Connect input from Coinbase AI Agent node.

10. **Add Telegram Send Message Node**  
    - Type: `telegram`  
    - Parameters:  
      - Text: `={{ $json.message }}` (from the splitter node)  
      - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
      - Disable attribution (no appended bot signature)  
    - Credentials: Use your Telegram Bot API credentials  
    - Connect input from message splitter node.

11. **Configure Credentials**  
    - Set up OpenAI API credentials with access to GPT-4o-mini model for Langchain nodes.  
    - Set up Telegram Bot credentials with Telegram Bot API token.  
    - No authentication required for Coinbase public REST endpoints.

12. **Activate Workflow**  
    - Deploy and activate the workflow.  
    - Ensure the Telegram bot webhook is correctly set up and reachable.  
    - Replace the placeholder Telegram user ID in the authentication code node.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This system is a professional AI automation for Coinbase spot market data, combining real-time API fetches with GPT-4o reasoning and Telegram delivery. It explicitly forbids generating trading advice or predictions, focusing solely on data retrieval and clean presentation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Workflow description and system design notes.                                                           |
| The workflow requires multiple supporting sub-workflows (Coinbase Price Tool, Order Book Tool, Candles Tool, Trades Tool) to function fully, as the AI agent calls these tools to fetch data. Ensure all related workflows are imported and activated in your n8n environment.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note17 and internal system documentation.                                                        |
| The Telegram message size limit is 4096 characters; the code node splits messages exceeding 4000 characters to prevent delivery failures.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note4 and splitting code node.                                                                   |
| Coinbase API Rate Limits: Public endpoints allow 10 requests per second per IP (burst 15). Exceeding this returns HTTP 429 errors. The workflow should handle such errors gracefully (not explicitly handled in the workflow but should be considered).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | System message in Coinbase AI Agent node.                                                               |
| For multi-user deployments, update the User Authentication node to allow multiple Telegram user IDs or implement more advanced access control.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | User Authentication node limitation note.                                                               |
| System credits and licensing: Developed by Don Jayamaha, Treasurium Capital Limited Company; protected by U.S. copyright laws. Reuse or resale prohibited without license.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Sticky Note17; LinkedIn profile: http://linkedin.com/in/donjayamahajr                                   |
| Coinbase API documentation for reference: https://docs.cloud.coinbase.com/exchange/reference/exchangerestapi_getproductsproductidticker                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Official Coinbase API docs.                                                                              |
| OpenAI GPT-4o-mini model is used for cost-effective AI reasoning and formatting; ensure OpenAI account has access.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | OpenAI documentation and API keys setup.                                                                |

---

# Disclaimer

The text provided is extracted solely from an n8n automated workflow. It complies strictly with all applicable content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly available.

---