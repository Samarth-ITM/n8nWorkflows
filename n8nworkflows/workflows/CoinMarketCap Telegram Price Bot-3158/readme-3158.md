CoinMarketCap Telegram Price Bot

https://n8nworkflows.xyz/workflows/coinmarketcap-telegram-price-bot-3158


# CoinMarketCap Telegram Price Bot

### 1. Workflow Overview

This workflow, titled **CoinMarketCap Telegram Price Bot**, enables real-time cryptocurrency price retrieval via Telegram messages. It is designed for crypto traders, analysts, and enthusiasts who want quick access to live market data without leaving Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user messages from Telegram.
- **1.2 Session Identification:** Assigns a session ID based on the Telegram chat for tracking.
- **1.3 AI-Powered Price Query Agent:** Uses an AI agent to interpret user input, request price data from CoinMarketCap, and format the response.
- **1.4 Memory Buffer:** Maintains conversational context for better session management.
- **1.5 Price Retrieval:** Sends HTTP requests to CoinMarketCap API to fetch live crypto prices.
- **1.6 Response Delivery:** Sends the processed price information back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for incoming Telegram messages from users to trigger the workflow.
- **Nodes Involved:** `Telegram Trigger1`

##### Node: Telegram Trigger1
- **Type:** Telegram Trigger node
- **Role:** Entry point that listens for new messages sent to the Telegram bot.
- **Configuration:** 
  - Listens for `message` updates only.
  - Uses Telegram API credentials configured with the bot token.
- **Expressions/Variables:** Outputs the entire message JSON, including chat ID and text.
- **Connections:** Output connects to `Adds SessionId`.
- **Edge Cases:** 
  - Telegram API downtime or invalid bot token causes trigger failure.
  - Messages without text or unsupported message types are ignored.
- **Version:** 1.1

---

#### 1.2 Session Identification

- **Overview:** Adds a `sessionId` field to the data, using the Telegram chat ID to track user sessions.
- **Nodes Involved:** `Adds SessionId`

##### Node: Adds SessionId
- **Type:** Set node
- **Role:** Adds a new field `sessionId` equal to the Telegram chat ID for session tracking.
- **Configuration:** 
  - Assigns `sessionId` = `{{$json.message.chat.id}}`
  - Passes through all other fields unchanged.
- **Connections:** Input from `Telegram Trigger1`, output to `CoinMarketCap Price Agent`.
- **Edge Cases:** 
  - Missing or malformed chat ID in incoming message JSON.
- **Version:** 3.4

---

#### 1.3 AI-Powered Price Query Agent

- **Overview:** Acts as the core logic interpreting user input, orchestrating API calls, memory, and language model processing.
- **Nodes Involved:** `CoinMarketCap Price Agent`

##### Node: CoinMarketCap Price Agent
- **Type:** LangChain Agent node
- **Role:** Processes user input text, manages AI language model, memory, and HTTP tool integration.
- **Configuration:** 
  - Input text: `{{$json.message.text}}` (user's crypto symbol or query)
  - Prompt type: `define` (likely a predefined prompt template)
  - No additional options configured.
- **Connections:** 
  - Receives input from `Adds SessionId`.
  - Connects to three AI components:
    - Language Model: `OpenAI Chat Model`
    - Memory: `Window Buffer Memory`
    - HTTP Tool: `CoinMarketCap Price`
  - Output connects to `Telegram Send Message`.
- **Notes:** Displays sessionId in node notes for debugging.
- **Edge Cases:** 
  - AI model or API failures.
  - Unexpected user input causing prompt or parsing errors.
  - Memory overflow or mismanagement.
- **Version:** 1.7

---

#### 1.4 Memory Buffer

- **Overview:** Maintains a windowed memory buffer to keep conversational context for each session.
- **Nodes Involved:** `Window Buffer Memory`

##### Node: Window Buffer Memory
- **Type:** LangChain Memory Buffer (window)
- **Role:** Stores recent conversation history to provide context for the AI agent.
- **Configuration:** Default parameters (no custom window size specified).
- **Connections:** Connected as `ai_memory` input to `CoinMarketCap Price Agent`.
- **Edge Cases:** 
  - Memory overflow if window size is too large.
  - Loss of context if memory is cleared unexpectedly.
- **Version:** 1.3

---

#### 1.5 Price Retrieval

- **Overview:** Sends HTTP requests to CoinMarketCap API to fetch the latest cryptocurrency price based on user input.
- **Nodes Involved:** `CoinMarketCap Price`

##### Node: CoinMarketCap Price
- **Type:** LangChain HTTP Request Tool node
- **Role:** Makes authenticated API calls to CoinMarketCap to retrieve price data.
- **Configuration:** 
  - URL: `https://pro-api.coinmarketcap.com/v1/cryptocurrency/quotes/latest`
  - Query parameters:
    - `symbol` (dynamic, from AI agent input)
    - `convert` fixed to `USD`
  - Headers:
    - `Accept: application/json`
  - Authentication: HTTP Header Auth using CoinMarketCap API key credential.
  - Tool description clarifies it receives crypto symbol input and returns price.
- **Connections:** Connected as `ai_tool` input to `CoinMarketCap Price Agent`.
- **Edge Cases:** 
  - API rate limits or quota exceeded.
  - Invalid or missing API key.
  - Unsupported or invalid crypto symbol.
  - Network timeouts or errors.
- **Version:** 1.1

---

#### 1.6 Response Delivery

- **Overview:** Sends the AI-processed price message back to the Telegram user.
- **Nodes Involved:** `Telegram Send Message`

##### Node: Telegram Send Message
- **Type:** Telegram node (send message)
- **Role:** Sends a message to the Telegram chat with the price information.
- **Configuration:** 
  - Text: `{{$json.output}}` (output from AI agent)
  - Chat ID: `{{$('Telegram Trigger1').item.json.message.chat.id}}` (same chat as user)
  - No additional fields configured.
  - Uses Telegram API credentials.
- **Connections:** Input from `CoinMarketCap Price Agent`.
- **Edge Cases:** 
  - Invalid chat ID or revoked bot permissions.
  - Telegram API errors or rate limits.
- **Version:** 1

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                     | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                  |
|-------------------------|----------------------------------|-----------------------------------|------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Telegram Trigger1        | Telegram Trigger                 | Receives user messages            | —                      | Adds SessionId            |                                                                                              |
| Adds SessionId          | Set                              | Adds sessionId based on chat ID   | Telegram Trigger1       | CoinMarketCap Price Agent |                                                                                              |
| CoinMarketCap Price Agent| LangChain Agent                  | Core AI agent managing flow       | Adds SessionId          | Telegram Send Message     | Shows sessionId in notes for debugging                                                      |
| OpenAI Chat Model        | LangChain OpenAI Chat Model      | AI language model for processing  | CoinMarketCap Price Agent (ai_languageModel) | CoinMarketCap Price Agent |                                                                                              |
| Window Buffer Memory     | LangChain Memory Buffer (window) | Maintains conversation context    | CoinMarketCap Price Agent (ai_memory) | CoinMarketCap Price Agent |                                                                                              |
| CoinMarketCap Price      | LangChain HTTP Request Tool      | Fetches crypto price from API     | CoinMarketCap Price Agent (ai_tool) | CoinMarketCap Price Agent | Tool receives crypto symbol input and requests price from CoinMarketCap API                  |
| Telegram Send Message    | Telegram Send Message             | Sends price message to user       | CoinMarketCap Price Agent | —                        |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**
   - Type: Telegram Trigger
   - Configure to listen for `message` updates.
   - Set Telegram API credentials with your bot token.
   - Position: Start of workflow.

2. **Add Set node to assign sessionId:**
   - Type: Set
   - Add new field `sessionId` with value `{{$json.message.chat.id}}`.
   - Enable "Keep Only Set" = false (to keep other fields).
   - Connect output of Telegram Trigger to this node.

3. **Add LangChain Agent node:**
   - Type: LangChain Agent
   - Set parameter `text` to `{{$json.message.text}}`.
   - Set `promptType` to `define`.
   - Connect output of Set node to this agent node.

4. **Add LangChain OpenAI Chat Model node:**
   - Type: LangChain OpenAI Chat Model
   - Select model `gpt-4o-mini`.
   - Configure OpenAI API credentials.
   - Connect this node as `ai_languageModel` input to the Agent node.

5. **Add LangChain Memory Buffer (Window) node:**
   - Type: LangChain Memory Buffer Window
   - Use default parameters.
   - Connect this node as `ai_memory` input to the Agent node.

6. **Add LangChain HTTP Request Tool node:**
   - Type: LangChain Tool HTTP Request
   - Set URL to `https://pro-api.coinmarketcap.com/v1/cryptocurrency/quotes/latest`.
   - Enable sending query parameters and headers.
   - Add query parameters:
     - `symbol` (dynamic, from input)
     - `convert` = `USD`
   - Add header `Accept: application/json`.
   - Set authentication to HTTP Header Auth.
   - Configure CoinMarketCap API key credential.
   - Add tool description explaining it fetches crypto prices.
   - Connect this node as `ai_tool` input to the Agent node.

7. **Add Telegram Send Message node:**
   - Type: Telegram
   - Set `text` to `{{$json.output}}` (output from Agent).
   - Set `chatId` to `{{$('Telegram Trigger1').item.json.message.chat.id}}`.
   - Use Telegram API credentials.
   - Connect output of Agent node to this node.

8. **Verify all connections:**
   - Telegram Trigger → Adds SessionId → CoinMarketCap Price Agent → Telegram Send Message
   - OpenAI Chat Model → CoinMarketCap Price Agent (ai_languageModel)
   - Window Buffer Memory → CoinMarketCap Price Agent (ai_memory)
   - CoinMarketCap Price → CoinMarketCap Price Agent (ai_tool)

9. **Set workflow activation and test:**
   - Activate the workflow.
   - Send a crypto symbol (e.g., "BTC") to your Telegram bot.
   - Confirm you receive a formatted price response.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                      |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Create your Telegram bot using [@BotFather](https://t.me/BotFather)                             | Telegram bot creation instructions                   |
| Obtain CoinMarketCap API key at [CoinMarketCap API](https://coinmarketcap.com/api/)             | API key registration                                 |
| Configure API credentials in n8n: CoinMarketCap key under HTTP Header Auth, Telegram bot token | Credential setup instructions                        |
| Automate crypto price tracking with this Telegram bot workflow                                  | Workflow purpose summary                             |

---

This documentation provides a detailed, node-level understanding of the CoinMarketCap Telegram Price Bot workflow, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.