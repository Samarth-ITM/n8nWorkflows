Send Hourly Crypto Market Analysis from Binance to Telegram

https://n8nworkflows.xyz/workflows/send-hourly-crypto-market-analysis-from-binance-to-telegram-3621


# Send Hourly Crypto Market Analysis from Binance to Telegram

### 1. Workflow Overview

This workflow is designed to provide an automated hourly summary of the cryptocurrency market focusing on selected USDC trading pairs from Binance (BTCUSDC, ETHUSDC, SOLUSDC). It fetches the last 24-hour price change data, performs detailed analytics including volatility, bid-ask spread, momentum, and market comparison, then formats the results into a rich HTML message. The message highlights top gainers, losers, and key market metrics, and sends it to a specified Telegram chat or group.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger:** Initiates the workflow every hour at minute 5.
- **1.2 Data Fetching:** Retrieves 24-hour ticker data from Binance public API.
- **1.3 Data Analysis & Formatting:** Processes raw data to compute metrics, filter relevant coins, and build a formatted HTML summary message. It also handles message splitting to comply with Telegram’s character limit.
- **1.4 Telegram Messaging:** Sends the formatted summary message(s) to a Telegram chat using the Telegram Bot API.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  This block triggers the workflow execution automatically every hour at the 5th minute, ensuring timely updates.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: `Schedule Trigger`  
    - Role: Initiates the workflow on a cron schedule.  
    - Configuration: Cron expression set to `5 * * * *` (at minute 5 of every hour).  
    - Inputs: None (trigger node).  
    - Outputs: Connects to the Binance data fetching node.  
    - Edge Cases: Cron misconfiguration could cause missed or multiple triggers; ensure server time is synchronized.  
    - Version: 1.1

#### 1.2 Data Fetching

- **Overview:**  
  Fetches the last 24-hour ticker price change data for all symbols from Binance’s public API endpoint.

- **Nodes Involved:**  
  - Binance 24h Price Change

- **Node Details:**  
  - **Binance 24h Price Change**  
    - Type: `HTTP Request`  
    - Role: Calls Binance API endpoint `/api/v3/ticker/24hr` to retrieve 24h price change statistics for all trading pairs.  
    - Configuration:  
      - URL: `https://api.binance.com/api/v3/ticker/24hr`  
      - Method: GET (default)  
      - No authentication required (public endpoint).  
      - Retry on failure enabled: up to 5 retries with 5 seconds delay between tries.  
    - Inputs: Triggered by Schedule Trigger.  
    - Outputs: Passes raw JSON array of ticker data to the analysis node.  
    - Edge Cases: Network failures, API downtime, or rate limiting could cause retries or failures.  
    - Notes: No Binance API key required.  
    - Version: 1

#### 1.3 Data Analysis & Formatting

- **Overview:**  
  Processes the raw Binance data to filter relevant USDC pairs (BTC, ETH, SOL), calculates multiple market metrics (volatility, bid-ask spread, momentum, market cap, market average comparison), identifies gainers and losers, and formats a detailed HTML summary message. It also splits the message into chunks to respect Telegram’s 4096 character limit.

- **Nodes Involved:**  
  - Analyze & Format Market Data

- **Node Details:**  
  - **Analyze & Format Market Data**  
    - Type: `Function`  
    - Role: Performs all data processing, analytics, and message formatting.  
    - Configuration Highlights:  
      - Filters symbols ending with `USDC` and specifically includes `BTCUSDC`, `ETHUSDC`, `SOLUSDC`.  
      - Calculates:  
        - Volatility = ((highPrice - lowPrice) / lowPrice) * 100  
        - Bid-Ask Spread = ((askPrice - bidPrice) / bidPrice) * 100  
        - Momentum = ((lastPrice - weightedAvgPrice) / weightedAvgPrice) * 100  
        - Estimated Market Cap = lastPrice * quoteVolume  
        - Market average change and comparison per coin  
      - Formats numbers with commas and fixed decimals for readability.  
      - Builds an HTML message with emojis, bold tags, and section headings.  
      - Splits the message into chunks under 4000 characters to avoid Telegram limits.  
    - Key Expressions/Variables:  
      - `relevantSymbols` array for tracked coins (modifiable).  
      - Functions for formatting volume, money, escaping HTML, and analytics calculations.  
      - Uses `items[0].json` as input data.  
    - Inputs: Receives raw Binance data from HTTP Request node.  
    - Outputs: Array of JSON objects each containing a chunk of the formatted message under `data` property.  
    - Edge Cases:  
      - If Binance data format changes, parsing may fail.  
      - If no relevant symbols found, message may be empty.  
      - Message splitting logic must ensure no partial lines are cut.  
      - Large volumes or unexpected data types could cause formatting errors.  
    - Notes:  
      - The relevantSymbols array can be customized to track other coins ending with USDC.  
      - The message uses HTML parse mode for Telegram.  
    - Version: 1

#### 1.4 Telegram Messaging

- **Overview:**  
  Sends the formatted market summary message chunks to a Telegram chat or group using a bot token and chat ID.

- **Nodes Involved:**  
  - Send Telegram Message

- **Node Details:**  
  - **Send Telegram Message**  
    - Type: `Telegram`  
    - Role: Sends messages to Telegram using the Bot API.  
    - Configuration:  
      - Text: Uses expression `{{$json.data}}` to send each chunk of the formatted message.  
      - Chat ID: Set to a specific Telegram group or user chat ID (e.g., `-4685771678`).  
      - Additional Fields: `parse_mode` set to `HTML` for rich formatting.  
      - Credentials: Requires Telegram Bot API token configured in n8n credentials.  
    - Inputs: Receives message chunks from the Function node.  
    - Outputs: None (end node).  
    - Edge Cases:  
      - Invalid bot token or chat ID causes authentication errors.  
      - Telegram API rate limits or downtime may cause message send failures.  
      - Large messages are pre-split to avoid exceeding Telegram limits.  
    - Version: 1

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                      | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                  |
|--------------------------|--------------------|------------------------------------|------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger   | Triggers workflow hourly           | None                   | Binance 24h Price Change  | ⏱ Runs **every hour** with cron `5 * * * *` (At minute 5 of every hour)                                      |
| Binance 24h Price Change | HTTP Request       | Fetches 24h ticker data from Binance | Schedule Trigger       | Analyze & Format Market Data | **Binance** - No API key needed; uses public endpoint; requires internet access                              |
| Analyze & Format Market Data | Function          | Analyzes data, calculates metrics, formats HTML message | Binance 24h Price Change | Send Telegram Message     | Optional: Modify `relevantSymbols` array to track more coins ending with `USDC`                             |
| Send Telegram Message    | Telegram           | Sends formatted message to Telegram chat | Analyze & Format Market Data | None                     | Setup: Create bot via [@BotFather](https://t.me/BotFather), add bot to chat/group, configure token & chat ID |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set Cron Expression: `5 * * * *` (fires at minute 5 every hour)  
   - No credentials needed  
   - Position: Start of workflow

2. **Create HTTP Request node (Binance 24h Price Change)**  
   - Type: HTTP Request  
   - Set URL: `https://api.binance.com/api/v3/ticker/24hr`  
   - Method: GET (default)  
   - Enable Retry on Fail: Yes, max 5 tries, 5000ms delay between tries  
   - Connect Schedule Trigger output to this node input

3. **Create Function node (Analyze & Format Market Data)**  
   - Type: Function  
   - Paste the provided JavaScript code that:  
     - Filters for `BTCUSDC`, `ETHUSDC`, `SOLUSDC`  
     - Calculates volatility, spread, momentum, market cap, market average comparison  
     - Formats an HTML message with emojis and bold tags  
     - Splits the message into chunks under 4000 characters  
   - Connect HTTP Request output to this node input

4. **Create Telegram node (Send Telegram Message)**  
   - Type: Telegram  
   - Set Text parameter to expression: `{{$json.data}}` (to send each chunk)  
   - Set Chat ID to your Telegram group or personal chat ID (e.g., `-4685771678`)  
   - Under Additional Fields, set `parse_mode` to `HTML`  
   - Configure Telegram API credentials with your bot token (create bot via [@BotFather](https://t.me/BotFather))  
   - Connect Function node output to this node input

5. **Verify connections:**  
   - Schedule Trigger → Binance 24h Price Change → Analyze & Format Market Data → Send Telegram Message

6. **Optional customization:**  
   - In the Function node, modify the `relevantSymbols` array to add or remove coins ending with `USDC` as needed.

7. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is highly customizable; you can modify analytics, tracked pairs, or message formatting freely.  | Internal customization guidance                                                                 |
| Telegram Bot creation and setup instructions: Create bot via [@BotFather](https://t.me/BotFather) and add to chat/group. | Telegram Bot setup instructions                                                                 |
| Binance API does not require authentication for the `/api/v3/ticker/24hr` endpoint used here.                  | Binance public API usage                                                                         |
| Message output is automatically split to stay under Telegram’s 4096 character limit for compatibility.        | Telegram message length constraint                                                              |
| The workflow is suitable for crypto traders, analysts, Telegram groups, and bot developers needing hourly market insights. | Use cases                                                                                       |
| Example output includes detailed market summary with emojis and HTML formatting for readability in Telegram.  | Example message snippet provided in the description                                             |

---

This documentation provides a complete and detailed reference for understanding, reproducing, and modifying the "Send Hourly Crypto Market Analysis from Binance to Telegram" workflow.