Daily ETH Wallet Monitoring with Etherscan, CoinGecko Pricing & Discord Alerts

https://n8nworkflows.xyz/workflows/daily-eth-wallet-monitoring-with-etherscan--coingecko-pricing---discord-alerts-8992


# Daily ETH Wallet Monitoring with Etherscan, CoinGecko Pricing & Discord Alerts

### 1. Workflow Overview

This workflow monitors the balance and token transactions of a specified Ethereum (ETH) wallet address twice daily, enriches the token balances with USD prices, and sends a detailed summary alert to a Discord channel. It integrates data from Etherscan (transaction and ETH balance), CoinGecko (token and ETH pricing), and Discord (notifications).

Logical blocks are grouped as follows:

- **1.1 Schedule Trigger:** Initiates the workflow twice daily.
- **1.2 Etherscan Data Retrieval and Processing:** Fetches token transactions and ETH balance, calculates net token balances.
- **1.3 Price Data Retrieval:** Obtains USD prices for tokens and ETH from CoinGecko.
- **1.4 Data Enrichment and Message Formatting:** Combines balances with price data and prepares the Discord message.
- **1.5 Discord Notification:** Sends the formatted balance and valuation summary to a Discord channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:** Initiates the workflow execution twice daily at specified times.
- **Nodes Involved:** `Schedule Trigger`
- **Node Details:**
  - **Type:** Schedule Trigger node; triggers workflow on a cron schedule.
  - **Configuration:** Cron expression set to trigger at 07:45 and 17:45 every day.
  - **Key expressions:** Cron schedule `"45 7,17 * * *"`.
  - **Input connections:** None (starting node).
  - **Output connections:** Triggers `HTTP Request`.
  - **Edge cases:** Cron misconfiguration or n8n scheduler downtime could prevent trigger.

#### 1.2 Etherscan Data Retrieval and Processing

- **Overview:** Retrieves Ethereum wallet token transactions and ETH balance from Etherscan, then calculates net token balances.
- **Nodes Involved:** `HTTP Request`, `Code in JavaScript`, `Code in JavaScript1`, `HTTP Request ETH`, `Code ETH Balance`, `HTTP Request ETH Price`
- **Node Details:**

  1. **HTTP Request**
     - **Type:** HTTP Request
     - **Role:** Fetches token transaction data for the specified wallet from Etherscan.
     - **Configuration:** 
       - GET request to Etherscan API endpoint for token transactions.
       - URL includes placeholders `YOUR_WALLET_HERE`, `YOUR_KEY_HERE` for wallet address and Etherscan API key.
     - **Input:** Triggered by schedule.
     - **Output:** JSON response with token transactions.
     - **Edge cases:** HTTP errors, invalid API key, rate limiting, incorrect wallet address.

  2. **Code in JavaScript**
     - **Type:** Code node (JavaScript)
     - **Role:** Parses Etherscan token transactions; calculates net token balances by summing incoming and subtracting outgoing token values.
     - **Configuration:** Uses `YOUR_WALLET_ADDRESS` placeholder in code for wallet address (must be lowercase).
     - **Key logic:** Iterates over transactions, aggregates balances per token symbol, converts raw balances to human-readable amounts considering decimals.
     - **Input:** JSON from `HTTP Request`.
     - **Output:** Array of token balances with metadata (symbol, contract, amount).
     - **Edge cases:** Missing or malformed transaction data, BigInt operations on very large numbers.

  3. **Code in JavaScript1**
     - **Type:** Code node (JavaScript)
     - **Role:** Aggregates token balances; concatenates contract addresses into a comma-separated string for price lookup.
     - **Input:** Token balances from previous node.
     - **Output:** Object containing the balances array and comma-separated contract addresses string.
     - **Edge cases:** Empty balances array; no contracts found.

  4. **HTTP Request ETH**
     - **Type:** HTTP Request
     - **Role:** Retrieves the ETH balance (in wei) for the wallet from Etherscan.
     - **Configuration:** URL includes placeholders for wallet and API key similar to token transactions.
     - **Input:** Triggered by `Code in JavaScript1`.
     - **Output:** JSON response with ETH balance.
     - **Edge cases:** Same as token HTTP request; API failures or invalid wallet.

  5. **Code ETH Balance**
     - **Type:** Code node (JavaScript)
     - **Role:** Converts ETH balance from wei (BigInt) to ETH decimal number.
     - **Input:** ETH balance JSON.
     - **Output:** ETH balance object with token symbol "ETH" and a dummy contract address.
     - **Edge cases:** Parsing errors if response structure changes; BigInt to Number conversion limitations.

  6. **HTTP Request ETH Price**
     - **Type:** HTTP Request
     - **Role:** Fetches current ETH price in USD from CoinGecko.
     - **Configuration:** Requests CoinGecko simple price API with headers including `COIN_GEKCO_KEY_HERE` for API key.
     - **Input:** Triggered by `Code ETH Balance`.
     - **Output:** JSON with ETH price.
     - **Edge cases:** API key missing, rate limiting, network errors.

#### 1.3 Price Data Retrieval

- **Overview:** Retrieves USD prices for all token contracts found and ETH.
- **Nodes Involved:** `HTTP Request1`
- **Node Details:**
  - **Type:** HTTP Request
  - **Role:** Queries CoinGecko for USD prices of all ERC-20 tokens via contract addresses.
  - **Configuration:** URL dynamically built using contract addresses from `Code in JavaScript1`.
  - **Headers:** Includes CoinGecko API key (`COIN_GECKO_KEY_HERE`).
  - **Input:** Triggered by `HTTP Request ETH Price`.
  - **Output:** JSON mapping contract addresses to USD prices.
  - **Edge cases:** Empty contract list, API rate limits, invalid API key.

#### 1.4 Data Enrichment and Message Formatting

- **Overview:** Combines token and ETH balances with price data, calculates total USD values, and formats a detailed Discord message.
- **Nodes Involved:** `Code in JavaScript2`
- **Node Details:**
  - **Type:** Code node (JavaScript)
  - **Role:** Merges ETH and token balances, enriches with USD prices, sorts tokens by USD value, computes total portfolio value.
  - **Key expressions:**
    - Combines token balances from `Code in JavaScript1` and ETH balances from `Code ETH Balance`.
    - Builds a price map including a dummy key for ETH.
    - Generates a formatted multi-line string for Discord summarizing total value, ETH details, and other token valuations.
  - **Input:** Price JSON from `HTTP Request1`, balances from previous nodes.
  - **Output:** JSON containing total USD value, enriched tokens array, and Discord message string.
  - **Edge cases:** Missing or zero prices, empty balances, formatting issues.

#### 1.5 Discord Notification

- **Overview:** Sends the formatted wallet balance and valuation summary message to a configured Discord channel via webhook.
- **Nodes Involved:** `Discord`
- **Node Details:**
  - **Type:** Discord node
  - **Role:** Posts a message to Discord using webhook authentication.
  - **Configuration:** Uses webhook credentials (`All Ops Notis Discord Webhook`).
  - **Message content:** Bound to the `discordMessage` output from previous node.
  - **Input:** Message JSON from `Code in JavaScript2`.
  - **Output:** Discord post response.
  - **Edge cases:** Webhook misconfiguration, network errors, Discord API limits.

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role                               | Input Node(s)            | Output Node(s)           | Sticky Note                                                         |
|---------------------|----------------------|-----------------------------------------------|--------------------------|--------------------------|--------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger      | Triggers workflow twice daily                  |                          | HTTP Request             | ## Daily ETH Wallet Balance sent to Discord. Currently you get two updates. One in the Morning and Evening. |
| HTTP Request        | HTTP Request         | Fetches token transactions from Etherscan     | Schedule Trigger          | Code in JavaScript       | ## Add Etherscan API Key to first 2 HTTP nodes. Add Wallet Address to Code Block as well. |
| Code in JavaScript  | Code (JavaScript)    | Calculates net token balances from txs        | HTTP Request              | Code in JavaScript1      | ## Add Etherscan API Key to first 2 HTTP nodes. Add Wallet Address to Code Block as well. |
| Code in JavaScript1 | Code (JavaScript)    | Aggregates balances and token contract list    | Code in JavaScript        | HTTP Request ETH          |                                                                    |
| HTTP Request ETH    | HTTP Request         | Fetches ETH balance in wei from Etherscan      | Code in JavaScript1       | Code ETH Balance         | ## Add Etherscan API Key to first 2 HTTP nodes. Add Wallet Address to Code Block as well. |
| Code ETH Balance    | Code (JavaScript)    | Converts wei to human-readable ETH balance     | HTTP Request ETH          | HTTP Request ETH Price   |                                                                    |
| HTTP Request ETH Price | HTTP Request       | Gets current ETH USD price from CoinGecko     | Code ETH Balance          | HTTP Request1            | ## Set Coin Gecko Keys Here                                        |
| HTTP Request1       | HTTP Request         | Fetches USD prices for tokens from CoinGecko  | HTTP Request ETH Price    | Code in JavaScript2      | ## Set Coin Gecko Keys Here                                        |
| Code in JavaScript2 | Code (JavaScript)    | Combines balances and prices; formats message  | HTTP Request1             | Discord                  |                                                                    |
| Discord             | Discord              | Sends wallet summary message to Discord        | Code in JavaScript2       |                          |                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Schedule Trigger` node:**
   - Type: Schedule Trigger
   - Set cron expression to `45 7,17 * * *` to run at 07:45 and 17:45 daily.
   - No credentials required.

2. **Create `HTTP Request` node (Token Transactions):**
   - Type: HTTP Request
   - Method: GET
   - URL: `https://api.etherscan.io/v2/api?chainid=1&module=account&action=tokentx&address=YOUR_WALLET_HERE&sort=asc&apikey=YOUR_KEY_HERE`
   - Replace `YOUR_WALLET_HERE` with your Ethereum wallet address.
   - Replace `YOUR_KEY_HERE` with your Etherscan API key.
   - Connect `Schedule Trigger` → `HTTP Request`.

3. **Create `Code in JavaScript` node (Calculate Token Balances):**
   - Type: Code (JavaScript)
   - Paste provided JavaScript code that parses token transactions and calculates balances.
   - Replace `YOUR_WALLET_ADDRESS` in code with your wallet address (lowercase).
   - Connect `HTTP Request` → `Code in JavaScript`.

4. **Create `Code in JavaScript1` node (Prepare Contract List):**
   - Type: Code (JavaScript)
   - Paste provided code that aggregates balances and concatenates contract addresses.
   - Connect `Code in JavaScript` → `Code in JavaScript1`.

5. **Create `HTTP Request ETH` node (Fetch ETH Balance):**
   - Type: HTTP Request
   - Method: GET
   - URL: `https://api.etherscan.io/v2/api?chainid=1&module=account&action=balance&address=YOUR_WALLET_HERE&apikey=YOUR_KEY_HERE`
   - Replace placeholders like before.
   - Connect `Code in JavaScript1` → `HTTP Request ETH`.

6. **Create `Code ETH Balance` node (Convert ETH Wei to ETH):**
   - Type: Code (JavaScript)
   - Paste provided code converting wei to ETH decimal.
   - Connect `HTTP Request ETH` → `Code ETH Balance`.

7. **Create `HTTP Request ETH Price` node:**
   - Type: HTTP Request
   - Method: GET
   - URL: `https://api.coingecko.com/api/v3/simple/price?ids=ethereum&vs_currencies=usd`
   - Add HTTP JSON headers:
     ```json
     {
       "accept": "application/json",
       "x-cg-demo-api-key": "COIN_GEKCO_KEY_HERE"
     }
     ```
   - Replace `COIN_GEKCO_KEY_HERE` with your CoinGecko API key.
   - Connect `Code ETH Balance` → `HTTP Request ETH Price`.

8. **Create `HTTP Request1` node (Fetch ERC20 Token Prices):**
   - Type: HTTP Request
   - Method: GET
   - URL Expression:
     ```
     https://api.coingecko.com/api/v3/simple/token_price/ethereum?contract_addresses={{ $('Code in JavaScript1').item.json.contracts }}&vs_currencies=usd
     ```
   - Add HTTP JSON headers for CoinGecko key as above.
   - Connect `HTTP Request ETH Price` → `HTTP Request1`.

9. **Create `Code in JavaScript2` node (Enrich and Format Message):**
   - Type: Code (JavaScript)
   - Paste provided code which combines balances, prices, calculates totals, and formats the Discord message.
   - Connect `HTTP Request1` → `Code in JavaScript2`.

10. **Create `Discord` node:**
    - Type: Discord
    - Authentication: Webhook
    - Credentials: Create or select existing Discord Webhook credentials.
    - Content: Set to `{{$json.discordMessage}}` (expression).
    - Connect `Code in JavaScript2` → `Discord`.

11. **Set Credentials:**
    - Configure Etherscan API key credentials or enter keys directly in HTTP Request URLs.
    - Configure CoinGecko API keys in HTTP Request headers.
    - Configure Discord webhook credentials with the webhook URL.

12. **Add Sticky Notes (Optional):**
    - Add notes near HTTP Request nodes reminding to add API keys and wallet address.
    - Add note near CoinGecko HTTP Request nodes about setting API keys.
    - Add note near Schedule Trigger about execution frequency.

13. **Test Workflow:**
    - Run manually or wait for schedule to verify correct data flow and Discord notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                         |
|------------------------------------------------------------------------------------------------|---------------------------------------|
| Add your Etherscan API key to the first two HTTP Request nodes and your wallet address in code. | Sticky Note near HTTP Request nodes.  |
| Set your CoinGecko API keys in HTTP Request headers for token and ETH price queries.           | Sticky Note near CoinGecko HTTP nodes.|
| Workflow triggers twice daily: morning and evening updates on wallet status.                    | Sticky Note near Schedule Trigger node.|
| Discord webhook must be configured with proper permissions to post messages in the target channel.| Discord node credential requirements. |

---

This documentation provides a complete reference to understand, replicate, and maintain the "Daily ETH Wallet Monitoring with Etherscan, CoinGecko Pricing & Discord Alerts" workflow.