Track Top Meme Coin Prices with Telegram Bot and CoinGecko API

https://n8nworkflows.xyz/workflows/track-top-meme-coin-prices-with-telegram-bot-and-coingecko-api-5634


# Track Top Meme Coin Prices with Telegram Bot and CoinGecko API

---

### 1. Workflow Overview

This workflow automates the tracking and reporting of the top meme coin prices via a Telegram bot using the CoinGecko API. When a user sends the command `/memecoin` to the Telegram bot, the workflow fetches the top 5 meme coins by market capitalization, formats the information into a readable message, and sends it back to the user on Telegram.

Logical blocks:

- **1.1 Input Reception:** Listens for Telegram messages and filters for the `/memecoin` command.
- **1.2 Data Retrieval:** Calls the CoinGecko API to fetch current market data for meme coins.
- **1.3 Message Formatting:** Processes and formats the API data into a Telegram-friendly message.
- **1.4 Output Delivery:** Sends the formatted message back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block captures incoming Telegram messages and filters them to process only those containing the `/memecoin` command.

- **Nodes Involved:**  
  - Telegram Trigger  
  - If /memecoin

- **Node Details:**

  - **Telegram Trigger**  
    - Type: `Telegram Trigger` (event node)  
    - Role: Listens for incoming Telegram messages and triggers the workflow when a message is received.  
    - Configuration:  
      - Listens to `message` updates only.  
      - Uses a Telegram API credential configured for bot authentication.  
    - Input/Output:  
      - No input (starts the workflow).  
      - Outputs message data JSON including the entire Telegram message object.  
    - Failure Modes:  
      - Telegram API connectivity issues or credential expiration may block triggers.  
      - Missing webhook setup can prevent messages from triggering the workflow.  
    - Version: 1

  - **If /memecoin**  
    - Type: `If` node (conditional filter)  
    - Role: Checks if the incoming message text exactly matches `/memecoin`.  
    - Configuration:  
      - Condition compares the message text field (`{{$json["message"]["text"]}}`) to the string `/memecoin`.  
    - Input/Output:  
      - Input from Telegram Trigger.  
      - Outputs to the next block if condition is true; otherwise, stops processing.  
    - Failure Modes:  
      - Expression errors if message structure differs or is empty.  
      - Case sensitivity implies other commands or typos are ignored.  
    - Version: 1

#### 1.2 Data Retrieval

- **Overview:**  
  Fetches the current market data of the top 5 meme coins in USD from the CoinGecko API.

- **Nodes Involved:**  
  - Fetch Meme Coins

- **Node Details:**

  - **Fetch Meme Coins**  
    - Type: `HTTP Request`  
    - Role: Makes an HTTP GET request to the CoinGecko API endpoint to retrieve meme coin market data.  
    - Configuration:  
      - URL: `https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&order=market_cap_desc&per_page=5&page=1&sparkline=false&category=meme-token`  
      - No authentication or additional headers required (public API).  
    - Input/Output:  
      - Input: From the If node when `/memecoin` command is detected.  
      - Output: JSON array of coin objects with market data such as name, symbol, current price, and id.  
    - Failure Modes:  
      - API downtime or rate limiting by CoinGecko.  
      - Network issues causing timeouts or malformed responses.  
      - Changes in API response structure may break downstream formatting.  
    - Version: 1

#### 1.3 Message Formatting

- **Overview:**  
  Converts the raw coin data into a Telegram-compatible Markdown message listing each coin with its name, symbol, current price, and a link to its CoinGecko page.

- **Nodes Involved:**  
  - Format Message

- **Node Details:**

  - **Format Message**  
    - Type: `Function`  
    - Role: Processes the array of coin data and constructs a formatted multiline Markdown message.  
    - Configuration:  
      - Extracts `chat_id` from the Telegram trigger node to send the response back to the correct chat.  
      - Maps each coin’s data into a message block with:  
        - Rocket emoji and coin name in bold  
        - Uppercased symbol  
        - Price in USD or "Not available" fallback  
        - URL linking to the specific coin page on CoinGecko  
      - Joins the blocks separated by two newlines for clarity.  
      - Returns an object with `chat_id` and `text` fields for the next Telegram node.  
    - Expressions/Variables:  
      - Uses `$node["Telegram Trigger"].json["message"]["chat"]["id"]` to get chat ID.  
      - Processes `items[0].json` which contains the array of coin market data.  
    - Input/Output:  
      - Input: Output from Fetch Meme Coins node.  
      - Output: JSON with `chat_id` and formatted `text`.  
    - Edge Cases:  
      - Missing or undefined fields in coin data gracefully handled with default values.  
      - If the API response is empty or malformed, the message will be blank or contain fallback text.  
      - On error, node configured to continue without interrupting the workflow.  
    - Version: 1

#### 1.4 Output Delivery

- **Overview:**  
  Sends the formatted message back to the user on Telegram using the bot.

- **Nodes Involved:**  
  - Send to Telegram

- **Node Details:**

  - **Send to Telegram**  
    - Type: `Telegram`  
    - Role: Sends a message to a Telegram chat based on the given `chat_id` and message text.  
    - Configuration:  
      - `text` parameter set dynamically to `{{$json["text"]}}`.  
      - `chatId` parameter set dynamically to `{{$json["chat_id"]}}`.  
      - Uses the same Telegram API credentials as the trigger node.  
    - Input/Output:  
      - Input: Receives the formatted message from the Function node.  
      - Output: Sends the message to Telegram; outputs the Telegram API response.  
    - Failure Modes:  
      - Failures in sending due to invalid chat ID, revoked bot access, or network issues.  
      - Telegram rate limits or API errors may cause message delivery failure.  
    - Version: 1

---

### 3. Summary Table

| Node Name         | Node Type           | Functional Role                 | Input Node(s)      | Output Node(s)   | Sticky Note                                            |
|-------------------|---------------------|--------------------------------|--------------------|------------------|--------------------------------------------------------|
| Telegram Trigger   | Telegram Trigger    | Listens for Telegram messages   | —                  | If /memecoin     |                                                        |
| If /memecoin      | If                  | Filters messages for /memecoin  | Telegram Trigger   | Fetch Meme Coins  |                                                        |
| Fetch Meme Coins   | HTTP Request        | Retrieves meme coin data        | If /memecoin       | Format Message    |                                                        |
| Format Message    | Function             | Formats API data into message   | Fetch Meme Coins   | Send to Telegram  |                                                        |
| Send to Telegram  | Telegram             | Sends message back to user      | Format Message     | —                |                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: `Telegram Trigger`  
   - Parameters:  
     - Updates: Select only `message`  
   - Credential: Add Telegram Bot API credentials (OAuth2 or Bot Token)  
   - This node will listen for incoming messages to the bot.

2. **Add If Node to Filter `/memecoin` Command**  
   - Type: `If`  
   - Condition:  
     - Check if `{{$json["message"]["text"]}}` equals `/memecoin` (string comparison, case sensitive)  
   - Connect output of Telegram Trigger to this node's input.

3. **Add HTTP Request Node to Fetch Meme Coins**  
   - Type: `HTTP Request`  
   - Parameters:  
     - HTTP Method: GET  
     - URL: `https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&order=market_cap_desc&per_page=5&page=1&sparkline=false&category=meme-token`  
     - No authentication required (public API)  
   - Connect the true output of the If node to this node.

4. **Add Function Node to Format Message**  
   - Type: `Function`  
   - Code:
     ```javascript
     const chatId = $node["Telegram Trigger"].json["message"]["chat"]["id"];
     const coins = items[0].json;

     let messageText = coins.map(coin => {
       const name = coin.name || "Unknown";
       const symbol = coin.symbol ? coin.symbol.toUpperCase() : "N/A";
       const price = coin.current_price !== undefined ? `$${coin.current_price}` : "Not available";
       const id = coin.id || "unknown";

       return `🚀 *${name}* (${symbol})\n💰 Price: ${price}\n🔗 More: https://www.coingecko.com/en/coins/${id}`;
     }).join("\n\n");

     return [{
       json: {
         chat_id: chatId,
         text: messageText
       }
     }];
     ```
   - Connect output of HTTP Request node to this node.  
   - Set "On Error" behavior to continue (optional).

5. **Add Telegram Node to Send Message**  
   - Type: `Telegram`  
   - Parameters:  
     - Text: `{{$json["text"]}}` (expression)  
     - Chat ID: `{{$json["chat_id"]}}` (expression)  
   - Credential: Use the same Telegram API credentials as the trigger node.  
   - Connect output of Function node to this node.

6. **Activate the Workflow**  
   - Ensure the workflow is active to listen for commands and respond.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                               |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------|
| The CoinGecko API endpoint used is public and does not require authentication, but rate limits may apply. | https://www.coingecko.com/en/api/documentation |
| Telegram bot requires proper webhook setup for triggers to work; ensure bot token is valid.     | Telegram Bot API documentation                 |
| Message formatting uses Markdown, so Telegram clients must support this.                         | https://core.telegram.org/bots/api#formatting-options |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.