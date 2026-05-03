Real-Time Crypto Price Bot for Telegram with Gemini AI & CoinGecko

https://n8nworkflows.xyz/workflows/real-time-crypto-price-bot-for-telegram-with-gemini-ai---coingecko-9230


# Real-Time Crypto Price Bot for Telegram with Gemini AI & CoinGecko

### 1. Workflow Overview

This workflow implements a **Real-Time Crypto Price Bot for Telegram** leveraging **Gemini AI** and **CoinGecko** APIs. Its primary purpose is to receive user inquiries about cryptocurrency prices via Telegram messages, interpret these queries using Gemini AI (Google Gemini model), retrieve the requested crypto price data from CoinGecko, format the results, and send the response back to the user on Telegram in real-time.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception:** Capture incoming Telegram messages triggering the workflow.
- **1.2 AI Query Interpretation:** Use Gemini AI to parse and understand the user's crypto price request.
- **1.3 Ticker Extraction:** Extract and set the requested cryptocurrency ticker symbol.
- **1.4 Price Retrieval:** Query CoinGecko API for the latest price of the requested ticker.
- **1.5 Formatting:** Process and format the retrieved data for user-friendly Telegram output.
- **1.6 Output Delivery:** Send the formatted crypto price information back to Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming Telegram messages that trigger the workflow. It serves as the entry point for user interaction.

- **Nodes Involved:**  
  - Telegram Trigger1

- **Node Details:**  
  - **Telegram Trigger1**  
    - Type: Telegram Trigger  
    - Role: Listens for new messages in a configured Telegram bot or chat.  
    - Configuration: Uses a webhook ID for incoming Telegram updates; no extra parameters specified, implying default listening to all messages.  
    - Inputs: None (trigger)  
    - Outputs: Sends message data downstream.  
    - Edge Cases: Potential webhook misconfiguration, Telegram bot authorization errors, or message format inconsistencies.  
    - Version: 1.2

---

#### 2.2 AI Query Interpretation

- **Overview:**  
  This block uses Gemini AI to interpret the user's message, understanding what cryptocurrency ticker or price information is requested.

- **Nodes Involved:**  
  - Message a model1

- **Node Details:**  
  - **Message a model1**  
    - Type: Google Gemini (via LangChain integration)  
    - Role: Sends the Telegram message text to Gemini AI for natural language understanding.  
    - Configuration: Default parameters; presumably uses configured OpenAI/Gemini credentials for API access.  
    - Inputs: Receives Telegram message data from Telegram Trigger1.  
    - Outputs: Provides AI-interpreted output, likely containing identified ticker symbol(s).  
    - Expressions: Likely uses expressions to extract user text from Telegram message payload (not explicitly shown).  
    - Edge Cases: API rate limits, auth failures, unexpected AI response format, or misunderstood input.  
    - Version: 1

---

#### 2.3 Ticker Extraction

- **Overview:**  
  This block sets the cryptocurrency ticker symbol extracted or inferred by Gemini AI for subsequent price lookup.

- **Nodes Involved:**  
  - Set Ticker1

- **Node Details:**  
  - **Set Ticker1**  
    - Type: Set node  
    - Role: Define or extract the ticker symbol variable from the AI response.  
    - Configuration: Presumably sets a variable with the ticker symbol extracted from Gemini output.  
    - Inputs: Receives AI response from Message a model1.  
    - Outputs: Passes ticker symbol forward for price retrieval.  
    - Expressions: Uses expressions to parse and assign the ticker symbol.  
    - Edge Cases: Missing or ambiguous ticker info from AI output, expression evaluation errors.  
    - Version: 3.4

---

#### 2.4 Price Retrieval

- **Overview:**  
  Queries the CoinGecko API to obtain the latest cryptocurrency price based on the ticker symbol.

- **Nodes Involved:**  
  - Get Price - Coingecko1

- **Node Details:**  
  - **Get Price - Coingecko1**  
    - Type: HTTP Request  
    - Role: Sends an API request to CoinGecko to fetch current price data for the specified ticker.  
    - Configuration:  
      - HTTP Method: Presumably GET  
      - URL: CoinGecko API endpoint (not explicitly shown) using the ticker symbol variable.  
      - Authentication: None (CoinGecko API is public)  
      - Response format: JSON expected.  
    - Inputs: Receives ticker symbol from Set Ticker1.  
    - Outputs: Returns raw price data to formatting node.  
    - Edge Cases: Network errors, invalid ticker symbol, API downtime, unexpected response structure.  
    - Version: 4.2

---

#### 2.5 Formatting

- **Overview:**  
  Formats the raw price data into a user-friendly message for Telegram delivery.

- **Nodes Involved:**  
  - Telegram Formatting1

- **Node Details:**  
  - **Telegram Formatting1**  
    - Type: Code node (JavaScript)  
    - Role: Processes the CoinGecko API response, formats it into a nicely readable text message.  
    - Configuration: Custom JavaScript code to extract price, format decimals, and prepare message string.  
    - Inputs: Receives raw price JSON from Get Price - Coingecko1.  
    - Outputs: Sends formatted text message.  
    - Edge Cases: Parsing errors if API response format changes, empty or null price data.  
    - Version: 2

---

#### 2.6 Output Delivery

- **Overview:**  
  Sends the formatted cryptocurrency price information back to the user on Telegram.

- **Nodes Involved:**  
  - Send COIN INFO to Telegram1

- **Node Details:**  
  - **Send COIN INFO to Telegram1**  
    - Type: Telegram node (send message)  
    - Role: Sends a text message to the Telegram chat/user who made the request.  
    - Configuration: Uses Telegram credentials and webhook ID for sending messages. The message text is set from the output of the formatting node.  
    - Inputs: Receives formatted text from Telegram Formatting1.  
    - Outputs: None (final node)  
    - Edge Cases: Telegram API errors, chat ID mismatches, message length limits.  
    - Version: 1.2

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role               | Input Node(s)        | Output Node(s)         | Sticky Note           |
|----------------------------|--------------------------------|------------------------------|----------------------|------------------------|-----------------------|
| Telegram Trigger1           | Telegram Trigger               | Input Reception              | -                    | Message a model1       | Sticky Note - Telegram Trigger |
| Message a model1            | Google Gemini (LangChain)      | AI Query Interpretation      | Telegram Trigger1     | Set Ticker1            | Sticky Note - Gemini  |
| Set Ticker1                | Set                           | Ticker Extraction            | Message a model1      | Get Price - Coingecko1 | Sticky Note - Set Ticker |
| Get Price - Coingecko1      | HTTP Request                  | Price Retrieval              | Set Ticker1           | Telegram Formatting1   | Sticky Note - CoinGecko |
| Telegram Formatting1        | Code                          | Formatting                   | Get Price - Coingecko1| Send COIN INFO to Telegram1 | Sticky Note - Formatting |
| Send COIN INFO to Telegram1 | Telegram                      | Output Delivery              | Telegram Formatting1  | -                      | Sticky Note - Send Telegram |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure webhook with your Telegram Bot token and set it to listen for new messages.  
   - No special filters needed; default triggers on all messages.

2. **Create Google Gemini (LangChain) node**  
   - Type: Google Gemini (LangChain)  
   - Configure with your Gemini or OpenAI credentials.  
   - Connect input from Telegram Trigger node.  
   - Set the prompt or input to send the Telegram message text to Gemini AI for natural language understanding.

3. **Create Set node (Set Ticker1)**  
   - Type: Set  
   - Connect input from Google Gemini node.  
   - Use expressions to extract the cryptocurrency ticker symbol from the AI response. For example, parse the AI output JSON or text to isolate the ticker (e.g., "BTC", "ETH").  
   - Save this ticker symbol as a variable (e.g., `ticker`).

4. **Create HTTP Request node (Get Price - Coingecko1)**  
   - Type: HTTP Request  
   - Connect input from Set node.  
   - Configure:  
     - Method: GET  
     - URL: `https://api.coingecko.com/api/v3/simple/price?ids={{$json["ticker"]}}&vs_currencies=usd`  
     - Replace `{{$json["ticker"]}}` with the ticker variable.  
     - No authentication needed.  
   - Set response format to JSON.

5. **Create Code node (Telegram Formatting1)**  
   - Type: Code  
   - Connect input from HTTP Request node.  
   - Write JavaScript to parse CoinGecko’s JSON response and format a user-friendly message.  
   - Example snippet:  
     ```javascript
     const ticker = Object.keys(items[0])[0];
     const price = items[0][ticker].usd;
     return [{ json: { message: `Current price of ${ticker.toUpperCase()} is $${price}` } }];
     ```
   - Output the formatted message text.

6. **Create Telegram node (Send COIN INFO to Telegram1)**  
   - Type: Telegram  
   - Connect input from Code node.  
   - Configure with the same Telegram credentials as the trigger.  
   - Set the chat ID dynamically to the original sender (usually from the Telegram Trigger node data).  
   - Bind the message text to the formatted output from the Code node.

7. **Link all nodes sequentially:**  
   - Telegram Trigger → Google Gemini → Set Ticker → HTTP Request (CoinGecko) → Code (Formatting) → Telegram (Send message).

8. **Credentials Setup:**  
   - Configure Telegram API credentials (bot token) in n8n credentials.  
   - Configure Google Gemini or OpenAI API credentials for the Gemini node.

9. **Test the workflow:**  
   - Send a message to your Telegram bot asking for a crypto price (e.g., "What's the price of BTC?").  
   - Confirm that the bot replies with the current price.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow uses Gemini AI (Google Gemini) to interpret natural language queries, improving understanding.    | Gemini AI integration via LangChain node        |
| CoinGecko API is public and free for price retrieval; no API key required.                                    | https://www.coingecko.com/en/api                |
| Telegram bot setup requires creating a bot via BotFather and obtaining a bot token.                           | https://core.telegram.org/bots                   |
| Proper error handling and rate limit management are recommended for production usage.                         | Adds robustness to the bot                        |
| Sticky notes in the workflow serve as visual guides for block functions but contain no content or comments.   | Workflow visual aid in n8n editor                 |

---

**Disclaimer:**  
The provided text is exclusively extracted from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.