Crypto Market Alert System with Binance and Telegram Integration

https://n8nworkflows.xyz/workflows/crypto-market-alert-system-with-binance-and-telegram-integration-2043


# Crypto Market Alert System with Binance and Telegram Integration

### 1. Workflow Overview

This workflow is designed to monitor cryptocurrency price movements on Binance and notify users via Telegram about significant market changes. It is ideal for traders or enthusiasts who want timely alerts on notable price fluctuations to make informed decisions.

The workflow is logically divided into the following blocks:

- **1.1 Trigger Options**: Starts the workflow either manually or automatically on a schedule.
- **1.2 Fetch Market Data**: Retrieves the latest 24-hour price change data for all Binance-listed cryptocurrencies.
- **1.3 Identify Significant Changes**: Filters the data to find cryptocurrencies with price changes greater than or equal to 15% (up or down).
- **1.4 Aggregate Data**: Combines filtered results into a single dataset.
- **1.5 Format Data for Telegram**: Splits the aggregated data into message-sized chunks respecting Telegram’s message length limits.
- **1.6 Send Telegram Message**: Sends the formatted alert messages to a specified Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Options

- **Overview:**  
  Initiates the workflow execution on a scheduled interval (every 1 minute by default). This allows continuous monitoring without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow at defined intervals.  
    - Configuration: Set to trigger every 1 minute using the minutes interval in the schedule rule.  
    - Key Expressions/Variables: None.  
    - Input: None (start node).  
    - Output: Connected to the "Binance 24h Price Change" node.  
    - Version-specific: Uses typeVersion 1.1, compatible with n8n versions supporting this trigger type.  
    - Potential Failures: Misconfigured scheduling intervals, workflow disabled or not activated.  
    - Sub-workflow: None.

#### 2.2 Fetch Market Data

- **Overview:**  
  Retrieves the 24-hour price changes of all cryptocurrencies available on Binance via a public REST API.

- **Nodes Involved:**  
  - Binance 24h Price Change

- **Node Details:**

  - **Binance 24h Price Change**  
    - Type: HTTP Request  
    - Role: Fetches market data from Binance API endpoint `/api/v1/ticker/24hr`.  
    - Configuration: Uses GET method (default), no authentication (default public endpoint). URL is `https://api.binance.com/api/v1/ticker/24hr`.  
    - Key Expressions/Variables: None.  
    - Input: From "Schedule Trigger".  
    - Output: To "Filter by 10% Change rate".  
    - Version-specific: Uses typeVersion 1.  
    - Potential Failures: Network errors, Binance API downtime, rate limiting, malformed responses, or unexpected data schema changes.  
    - Sub-workflow: None.

#### 2.3 Identify Significant Changes

- **Overview:**  
  Processes the fetched market data to filter cryptocurrencies whose price changed by 15% or more in the last 24 hours, either upwards or downwards. Results are sorted descendingly by change percentage and formatted as individual message strings.

- **Nodes Involved:**  
  - Filter by 10% Change rate

- **Node Details:**

  - **Filter by 10% Change rate**  
    - Type: Function  
    - Role: Custom JavaScript code to filter and format the significant price changes.  
    - Configuration:  
      - Parses the JSON array of coin data from the previous HTTP Request node.  
      - Filters coins with absolute price change percent >= 15%.  
      - Sorts filtered coins in descending order by price change percentage.  
      - Maps each coin to an object containing a formatted message string with symbol, price change %, and last price, wrapped in triple backticks for formatting.  
    - Key Expressions/Variables:  
      - Uses `parseFloat(coin.priceChangePercent)`.  
      - Outputs array of JSON objects with `message` property.  
    - Input: From "Binance 24h Price Change".  
    - Output: To "Aggregate".  
    - Version-specific: Uses typeVersion 1.  
    - Potential Failures: JavaScript runtime errors if input data is malformed or missing properties; empty input array resulting in no messages; unexpected API schema changes.  
    - Sub-workflow: None.

#### 2.4 Aggregate Data

- **Overview:**  
  Combines multiple message items from the previous filter node into a single dataset, preparing for message splitting.

- **Nodes Involved:**  
  - Aggregate

- **Node Details:**

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Combines all individual filtered messages into one output item.  
    - Configuration: Uses "aggregate all item data" option to merge data arrays.  
    - Key Expressions/Variables: None beyond node configuration.  
    - Input: From "Filter by 10% Change rate".  
    - Output: To "Split By 1K chars".  
    - Version-specific: typeVersion 1.  
    - Potential Failures: Empty input leading to empty aggregation; large data causing performance issues.  
    - Sub-workflow: None.

#### 2.5 Format Data for Telegram

- **Overview:**  
  Splits the aggregated messages into chunks of approximately 1000 characters, respecting Telegram’s maximum message length to avoid message rejection.

- **Nodes Involved:**  
  - Split By 1K chars

- **Node Details:**

  - **Split By 1K chars**  
    - Type: Code (JavaScript)  
    - Role: Custom code to split a single large string containing all messages into multiple smaller messages capped at 1000 characters.  
    - Configuration:  
      - Iterates over the combined data messages.  
      - Adds messages to a current chunk as long as the length does not exceed 1000 characters.  
      - Pushes chunk to output array and starts a new chunk when limit is exceeded.  
    - Key Expressions/Variables:  
      - Accesses `items[0].json.data` which contains concatenated messages.  
      - Returns array of objects each with a `data` property containing the chunk string.  
    - Input: From "Aggregate".  
    - Output: To "Send Telegram Message".  
    - Version-specific: typeVersion 2 (code node version supporting modern JS).  
    - Potential Failures: Incorrect input format; if `data` property missing or malformed; possible off-by-one errors in chunk splitting; large datasets may cause performance issues.  
    - Sub-workflow: None.

#### 2.6 Send Telegram Message

- **Overview:**  
  Sends each chunked message as a Telegram message to a specified chat using a Telegram Bot.

- **Nodes Involved:**  
  - Send Telegram Message

- **Node Details:**

  - **Send Telegram Message**  
    - Type: Telegram  
    - Role: Sends a text message to a Telegram chat via Bot API.  
    - Configuration:  
      - Message text is dynamically set using an expression extracting up to the first 1000 characters from the incoming chunk’s `data` property, replacing certain patterns for formatting.  
      - Chat ID is statically set to `-1002138086614` (a Telegram group or channel ID).  
      - Credentials require Telegram Bot token setup (configured in n8n credentials).  
    - Key Expressions/Variables:  
      - Expression in `text`: `={{ $json.data.replaceAll(/(Last Price: \S+)$/gm,"$1\n").slice(0,1000) }}`  
      - Uses the prepared data chunks from the previous node.  
    - Input: From "Split By 1K chars".  
    - Output: None (end node).  
    - Version-specific: typeVersion 1.  
    - Potential Failures: Invalid or expired bot token; incorrect chat ID; Telegram API limits (rate limiting, message size exceeded); network issues; message formatting errors.  
    - Sub-workflow: None.

#### 2.7 Sticky Note (Documentation Block)

- **Overview:**  
  Provides setup instructions and notes for users configuring the workflow.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Contains detailed setup steps, including scheduling trigger activation, optional Binance API configuration, Telegram bot creation and token usage, and chat ID configuration.  
    - Content includes a link to the workflow page for extended instructions.  
    - Input/Output: None (informational only).  
    - Version-specific: typeVersion 1.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                         | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                                    |
|----------------------------|---------------------|---------------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger    | Initiates workflow on schedule        | None                         | Binance 24h Price Change     | Ensure the schedule trigger is active to desired cron time (default 1 minute).                                |
| Binance 24h Price Change    | HTTP Request        | Fetches 24h price change data from Binance | Schedule Trigger             | Filter by 10% Change rate    | Optional: Configure with Binance API details; default uses free public API.                                   |
| Filter by 10% Change rate  | Function            | Filters coins with ±15% price change and formats messages | Binance 24h Price Change      | Aggregate                   | Filters by 15% up/down change rate, sorts by descending change percent.                                       |
| Aggregate                  | Aggregate           | Combines filtered messages into one dataset | Filter by 10% Change rate    | Split By 1K chars           | Combines all filtered items for further processing.                                                           |
| Split By 1K chars          | Code                | Splits aggregated message into chunks ≤ 1000 chars | Aggregate                    | Send Telegram Message        | Splits messages to respect Telegram message size limits.                                                     |
| Send Telegram Message      | Telegram            | Sends formatted alert messages to Telegram chat | Split By 1K chars            | None                        | Requires Telegram bot token and chat ID; sends alerts to configured chat.                                    |
| Sticky Note                | Sticky Note         | Provides setup instructions and notes | None                         | None                        | Workflow setup guidelines and link to the workflow page for detailed instructions.                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to trigger every 1 minute (or desired interval) under the "Rule" tab → "Minutes" interval.  
   - Position this node as the workflow start.

3. **Add an HTTP Request Node**  
   - Name: "Binance 24h Price Change".  
   - URL: `https://api.binance.com/api/v1/ticker/24hr`  
   - Method: GET (default).  
   - No authentication needed (optional Binance API key can be added if desired).  
   - Connect the Schedule Trigger's output to this node's input.

4. **Add a Function Node**  
   - Name: "Filter by 10% Change rate".  
   - Paste the following JavaScript code in the function editor:

   ```js
   const significantChanges = [];
   for (const coin of items[0].json) {
     const priceChangePercent = parseFloat(coin.priceChangePercent);
     if (Math.abs(priceChangePercent) >= 15) {
       significantChanges.push({ 
         symbol: coin.symbol, 
         priceChangePercent, 
         lastPrice: coin.lastPrice 
       });
     }
   }
   significantChanges.sort((a, b) => b.priceChangePercent - a.priceChangePercent);
   const sortedOutput = significantChanges.map(change => ({
     json: { message: `\`\`\`${change.symbol} Price changed by ${change.priceChangePercent}% \n Last Price: ${change.lastPrice}\`\`\`` }
   }));
   return sortedOutput;
   ```

   - Connect HTTP Request node output to this node input.

5. **Add an Aggregate Node**  
   - Name: "Aggregate".  
   - Set "Aggregate" option to "aggregateAllItemData" (combines all items into one).  
   - Connect the Function node output to this Aggregate node.

6. **Add a Code Node**  
   - Name: "Split By 1K chars".  
   - Set "Type" to JavaScript.  
   - Paste this code in the editor:

   ```js
   function splitDataIntoChunks(data) {
       const chunks = [];
       let currentChunk = "";

       data.forEach(item => {
           if (item && item.message) {
               const message = item.message + "\n";
               if (currentChunk.length + message.length > 1000) {
                   chunks.push({ json: { data: currentChunk } });
                   currentChunk = message;
               } else {
                   currentChunk += message;
               }
           }
       });

       if (currentChunk) {
           chunks.push({ json: { data: currentChunk } });
       }
       return chunks;
   }

   const inputData = items[0].json.data;
   return splitDataIntoChunks(inputData);
   ```

   - Connect Aggregate node output to this Code node.

7. **Add a Telegram Node**  
   - Name: "Send Telegram Message".  
   - Operation: Send Message.  
   - Chat ID: Set to your Telegram chat ID (e.g., `-1002138086614`).  
   - Text: Use expression mode and enter:  
     ```
     {{$json.data.replaceAll(/(Last Price: \S+)$/gm,"$1\n").slice(0,1000)}}
     ```  
   - Credentials: Set up Telegram Bot credentials by creating a bot via BotFather in Telegram and entering the token here.  
   - Connect the Code node output to this Telegram node.

8. **Save and activate the workflow.**

9. **Test the workflow** manually or wait for the scheduled trigger to verify that messages are sent correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Detailed instructions on creating a Telegram bot and obtaining the required token are provided within the workflow description and linked resources. Please follow Telegram's official BotFather procedure.                                                                                                                                                                                                                                                                                                                           | Telegram BotFather: https://core.telegram.org/bots#6-botfather                                              |
| The workflow uses Binance's public API endpoint to fetch price data without authentication. For higher rate limits or private data, configure Binance API credentials in the HTTP Request node (optional).                                                                                                                                                                                                                                                                                                                           | Binance API Docs: https://binance-docs.github.io/apidocs/spot/en/#24hr-ticker-price-change-statistics       |
| Telegram messages are split to respect the 4096-character limit per message enforced by the Telegram Bot API; this workflow uses a 1000-character limit for safety and readability.                                                                                                                                                                                                                                                                                                                                                  | Telegram Bot API: https://core.telegram.org/bots/api#sendmessage                                             |
| The workflow is configured for a 1-minute interval schedule trigger by default, but this can be adjusted to any cron or interval schedule in the Schedule Trigger node.                                                                                                                                                                                                                                                                                                                                                              |                                                                                                             |
| The workflow includes a sticky note with setup steps and a link to the official n8n workflow page for further customization and community support.                                                                                                                                                                                                                                                                                                                                                                                 | Workflow page: https://n8n.io/workflows/2043-crypto-market-alert-system-with-binance-and-telegram-integration |

---

This completes the comprehensive reference documentation of the "Crypto Market Alert System with Binance and Telegram Integration" n8n workflow.