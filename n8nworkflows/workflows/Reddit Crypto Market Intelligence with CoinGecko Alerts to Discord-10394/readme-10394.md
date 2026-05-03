Reddit Crypto Market Intelligence with CoinGecko Alerts to Discord

https://n8nworkflows.xyz/workflows/reddit-crypto-market-intelligence-with-coingecko-alerts-to-discord-10394


# Reddit Crypto Market Intelligence with CoinGecko Alerts to Discord

### 1. Workflow Overview

This n8n workflow titled **"Reddit Crypto Intelligence & Market Spike Detector"** is designed to monitor cryptocurrency discussions on Reddit, correlate mentioned coins with real-time market data from CoinGecko, detect significant price movements (spikes), and send alerts to a Discord channel. The workflow automates the process of sentiment and market data fusion to provide early warnings of trending tokens or volatility in the crypto market.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger:** Initiates the workflow at regular intervals (hourly by default).
- **1.2 Reddit Data Fetching & Parsing:** Retrieves the latest posts from the r/CryptoCurrency subreddit RSS feed and extracts mentions of selected cryptocurrencies.
- **1.3 Market Data Retrieval:** Fetches live price and 24-hour change data for the detected coins from the CoinGecko API.
- **1.4 Price Spike Detection:** Analyzes the market data to identify coins with significant price movements (±10% by default).
- **1.5 Alert Message Composition:** Builds a formatted message summarizing detected spikes or a default message if no spikes are found.
- **1.6 Discord Notification:** Sends the alert message to a specific Discord channel using a Discord bot integration.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  This block triggers the entire workflow automatically every hour, ensuring continuous monitoring without manual intervention.

- **Nodes Involved:**  
  - Hourly

- **Node Details:**

  - **Node Name:** Hourly  
    - **Type:** Schedule Trigger  
    - **Role:** Initiates workflow execution every hour.  
    - **Configuration:** Interval set to trigger every 1 hour; no additional parameters.  
    - **Input:** None (trigger node).  
    - **Output:** Triggers "Fetch Reddit Posts" node.  
    - **Potential Failures:** Scheduling system errors are rare; ensure n8n instance uptime for consistent triggering.  
    - **Sticky Note Content:**  
      > This node schedules the workflow to run automatically every hour.  
      > It ensures consistent monitoring of Reddit activity without manual execution.  
      > You can adjust the interval as needed (e.g., every 15 minutes or 2 hours).

#### 2.2 Reddit Data Fetching & Parsing

- **Overview:**  
  Fetches the newest posts from the r/CryptoCurrency subreddit via its public RSS feed and extracts mentions of popular cryptocurrencies from post titles.

- **Nodes Involved:**  
  - Fetch Reddit Posts  
  - Extract Coin Mentions

- **Node Details:**

  - **Node Name:** Fetch Reddit Posts  
    - **Type:** HTTP Request  
    - **Role:** Retrieves the RSS feed XML from Reddit's r/CryptoCurrency/new endpoint.  
    - **Configuration:**  
      - URL: `https://www.reddit.com/r/CryptoCurrency/new/.rss`  
      - Response Format: String (raw XML)  
      - No authentication required (public RSS).  
    - Input: Trigger from "Hourly".  
    - Output: Passes raw RSS XML string to "Extract Coin Mentions".  
    - Potential Failures: Network issues, Reddit rate limits or downtime, malformed RSS response.  
    - Sticky Note Content:  
      > This node retrieves the latest posts from the r/CryptoCurrency subreddit using Reddit’s public RSS feed.  
      > The feed provides a lightweight and free way to monitor new crypto discussions in real time.  
      > Each post includes a title, link, author, and timestamp.  
      > These are then parsed to detect potential mentions of new tokens or trending coins.

  - **Node Name:** Extract Coin Mentions  
    - **Type:** Code (JavaScript)  
    - **Role:** Parses the RSS XML to extract post titles and searches for mentions of specific coins.  
    - **Configuration:**  
      - Uses regex to extract `<title>` tags from RSS XML.  
      - Converts titles to lowercase and checks for substrings related to known coins: bitcoin (btc), ethereum (eth), solana (sol), dogecoin (doge), ripple (xrp), cardano (ada).  
      - Outputs a list of unique coins mentioned.  
    - Input: Raw RSS XML string from "Fetch Reddit Posts".  
    - Output: List of JSON objects each with a `coin` field representing a detected coin symbol/name.  
    - Potential Failures: Parsing errors if RSS format changes; no coin mentions found results in empty output (no explicit error).  
    - Sticky Note Content:  
      > This code node extracts coin tickers or symbols (e.g., $BTC, $ETH, $SOL, $MOONX) from the Reddit post titles.  
      > It uses a simple regex (\$[A-Za-z0-9]{2,10}) to detect these mentions.  
      > The results are cleaned and standardized to lowercase for easier processing.  
      > If no matches are found, the node returns a “No coin mentions detected” message.

#### 2.3 Market Data Retrieval

- **Overview:**  
  Retrieves current market data for cryptocurrencies from the CoinGecko API, focusing on price and 24-hour percentage changes.

- **Nodes Involved:**  
  - Fetch Coin Data

- **Node Details:**

  - **Node Name:** Fetch Coin Data  
    - **Type:** HTTP Request  
    - **Role:** Calls CoinGecko’s public API to get market data for the top 250 coins by market cap.  
    - **Configuration:**  
      - URL: `https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&order=market_cap_desc&per_page=250&page=1`  
      - No authentication required.  
      - Fetches `current_price`, `price_change_percentage_24h`, and other metadata.  
    - Input: List of coins from "Extract Coin Mentions" (though the API returns data for all top 250 coins, filtering occurs later).  
    - Output: Passes coin market data to "Detect Price Spike".  
    - Potential Failures: API rate limits, network issues, API changes.  
    - Sticky Note Content:  
      > This node retrieves live price data and 24-hour percentage changes for the detected coin symbols from the CoinGecko public API.  
      > CoinGecko is completely free to use with a rate limit of 10–30 calls per minute (no API key required).  
      > This step connects Reddit sentiment to real-time market data.

#### 2.4 Price Spike Detection

- **Overview:**  
  Analyzes the retrieved market data to identify cryptocurrencies with a price change greater than or equal to ±10% in the last 24 hours, flagging significant market movements.

- **Nodes Involved:**  
  - Detect Price Spike

- **Node Details:**

  - **Node Name:** Detect Price Spike  
    - **Type:** Code (JavaScript)  
    - **Role:** Filters coins with absolute 24h price change ≥10%.  
    - **Configuration:**  
      - Threshold set at 10%.  
      - Iterates over all coin data, selecting those exceeding threshold.  
      - Prepares output JSON with fields: id, symbol, name, current_price, change_24h, direction (up/down), last_updated.  
      - Returns a default message if no spikes detected or if data missing.  
    - Input: Coin market data from "Fetch Coin Data".  
    - Output: List of spikes or a message indicating no spikes.  
    - Potential Failures: Input data format changes, missing fields, empty input arrays.  
    - Sticky Note Content:  
      > This code node detects significant market movements (default ±5% in 24h).  
      > If any coin mentioned on Reddit has a large movement, it’s flagged as a potential market signal.  
      > This node filters out noise and highlights only meaningful spikes.

#### 2.5 Alert Message Composition

- **Overview:**  
  Formats the detected spike data into a clear, user-friendly message for Discord. If no spikes are detected, sends a neutral message.

- **Nodes Involved:**  
  - Compose Discord Message

- **Node Details:**

  - **Node Name:** Compose Discord Message  
    - **Type:** Code (JavaScript)  
    - **Role:** Builds a text message embedding emoji, coin name, symbol, price, change percentage, and timestamp.  
    - **Configuration:**  
      - Uses emoji 🚀 for upward spikes, 📉 for downward.  
      - Formats numbers to two decimal places.  
      - Default message: "No price spikes detected in the latest check."  
    - Input: Spike data from "Detect Price Spike".  
    - Output: JSON with a `content` field for Discord message body.  
    - Potential Failures: Missing data fields, formatting issues.  
    - Sticky Note Content:  
      > This node builds a clear, well-formatted alert message for Discord.  
      > If price spikes were detected, it lists each coin with price, change %, and timestamp.  
      > If no spikes were found, it sends a neutral “no changes” message instead.

#### 2.6 Discord Notification

- **Overview:**  
  Sends the composed alert message to a Discord channel via a Discord bot, enabling real-time alerts to community or team members.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**

  - **Node Name:** Send a message  
    - **Type:** Discord  
    - **Role:** Posts the alert message to a specified Discord guild and channel using bot credentials.  
    - **Configuration:**  
      - Content mapped dynamically from "Compose Discord Message".  
      - Target Guild ID: `1280652666577354803` (example)  
      - Target Channel ID: `1280652667408089210` (example)  
      - Uses Discord Bot API credentials (OAuth2 token or bot token).  
    - Input: Message content from "Compose Discord Message".  
    - Output: None (side effect: message sent).  
    - Potential Failures: Authentication errors, permission issues, invalid channel or guild IDs, Discord API downtime.  
    - Sticky Note Content:  
      > This node posts the composed message into your Discord channel via Webhook.  
      > You can set up the Webhook inside your Discord server (Settings → Integrations → Webhooks).  
      > This creates a simple and real-time alert delivery system.

---

### 3. Summary Table

| Node Name           | Node Type       | Functional Role                          | Input Node(s)         | Output Node(s)       | Sticky Note                                                                                                                |
|---------------------|-----------------|----------------------------------------|-----------------------|----------------------|----------------------------------------------------------------------------------------------------------------------------|
| Hourly              | Schedule Trigger| Triggers workflow every hour           | None                  | Fetch Reddit Posts    | This node schedules the workflow to run automatically every hour. It ensures consistent monitoring of Reddit activity without manual execution. You can adjust the interval as needed (e.g., every 15 minutes or 2 hours). |
| Fetch Reddit Posts   | HTTP Request    | Retrieves Reddit r/CryptoCurrency RSS  | Hourly                | Extract Coin Mentions | This node retrieves the latest posts from the r/CryptoCurrency subreddit using Reddit’s public RSS feed. The feed provides a lightweight and free way to monitor new crypto discussions in real time. Each post includes a title, link, author, and timestamp. These are then parsed to detect potential mentions of new tokens or trending coins. |
| Extract Coin Mentions| Code            | Parses RSS, extracts coin mentions     | Fetch Reddit Posts     | Fetch Coin Data       | This code node extracts coin tickers or symbols (e.g., $BTC, $ETH, $SOL, $MOONX) from the Reddit post titles. It uses a simple regex (\$[A-Za-z0-9]{2,10}) to detect these mentions. The results are cleaned and standardized to lowercase for easier processing. If no matches are found, the node returns a “No coin mentions detected” message. |
| Fetch Coin Data      | HTTP Request    | Fetches live market data from CoinGecko | Extract Coin Mentions  | Detect Price Spike    | This node retrieves live price data and 24-hour percentage changes for the detected coin symbols from the CoinGecko public API. CoinGecko is completely free to use with a rate limit of 10–30 calls per minute (no API key required). This step connects Reddit sentiment to real-time market data. |
| Detect Price Spike   | Code            | Detects coins with significant price changes | Fetch Coin Data    | Compose Discord Message | This code node detects significant market movements (default ±5% in 24h). If any coin mentioned on Reddit has a large movement, it’s flagged as a potential market signal. This node filters out noise and highlights only meaningful spikes. |
| Compose Discord Message | Code         | Formats alert message for Discord      | Detect Price Spike     | Send a message        | This node builds a clear, well-formatted alert message for Discord. If price spikes were detected, it lists each coin with price, change %, and timestamp. If no spikes were found, it sends a neutral “no changes” message instead. |
| Send a message       | Discord         | Sends alert message to Discord channel | Compose Discord Message| None                  | This node posts the composed message into your Discord channel via Webhook. You can set up the Webhook inside your Discord server (Settings → Integrations → Webhooks). This creates a simple and real-time alert delivery system. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger Node**  
   - Node Type: Schedule Trigger  
   - Parameters: Set interval to every 1 hour (adjustable as needed).  
   - This node starts the workflow automatically.

2. **Add HTTP Request Node "Fetch Reddit Posts"**  
   - URL: `https://www.reddit.com/r/CryptoCurrency/new/.rss`  
   - Response Format: String (to receive raw RSS XML)  
   - No authentication needed.  
   - Connect output of Schedule Trigger to this node.

3. **Add Code Node "Extract Coin Mentions"**  
   - Paste the provided JavaScript code that parses RSS XML, extracts `<title>` tags, and detects mentions of bitcoin, ethereum, solana, dogecoin, ripple, cardano.  
   - Input: Connect from "Fetch Reddit Posts".  
   - Output: JSON array with detected coin names in lowercase.

4. **Add HTTP Request Node "Fetch Coin Data"**  
   - URL: `https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&order=market_cap_desc&per_page=250&page=1`  
   - No authentication needed.  
   - Input: Connect from "Extract Coin Mentions".  
   - This node fetches live market data for top 250 coins.

5. **Add Code Node "Detect Price Spike"**  
   - Paste JavaScript code that filters coins with ±10% or more price change in 24h.  
   - Input: Connect from "Fetch Coin Data".  
   - Output: JSON with spike details or no spike message.

6. **Add Code Node "Compose Discord Message"**  
   - Paste JavaScript code to format alert message with emoji, coin name, symbol, price, % change, and timestamp.  
   - Input: Connect from "Detect Price Spike".  
   - Output: JSON with `content` field containing the message.

7. **Add Discord Node "Send a message"**  
   - Resource: Message  
   - Operation: Send message to channel  
   - Parameters:  
     - Content: Set expression to `{{$json["content"]}}`  
     - Guild ID: Set to your Discord server ID (e.g., `1280652666577354803`)  
     - Channel ID: Set to your target channel ID (e.g., `1280652667408089210`)  
   - Credentials: Configure Discord Bot API credentials with your bot token.  
   - Input: Connect from "Compose Discord Message".

8. **Finalize Connections**  
   - Connect all nodes sequentially as described:  
     Hourly → Fetch Reddit Posts → Extract Coin Mentions → Fetch Coin Data → Detect Price Spike → Compose Discord Message → Send a message.

9. **Test Workflow**  
   - Run manually or wait for scheduled trigger.  
   - Verify Discord messages appear correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                            | Context or Link                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| This n8n automation continuously monitors the r/CryptoCurrency subreddit for new coin mentions, correlates them with live CoinGecko market data, and detects significant market movements. When price spikes are identified, it sends real-time Discord alerts summarizing affected tokens, price, and percentage change. | Workflow purpose summary and usage context                         |
| By blending social sentiment signals with market analytics, this workflow acts as an early-warning system for trending tokens and volatility events — all using free, open APIs.                                                                                                                         | Strategic value of the workflow                                     |
| 🧩 n8n instance: Self-hosted or Cloud  💬 Discord Webhook URL: Create via Discord → Server Settings → Integrations → Webhooks  🌐 Reddit RSS Feed (Free) URL: https://www.reddit.com/r/CryptoCurrency/new/.rss  📊 CoinGecko API (Free): No API key required  ⏰ Recommended schedule: Every hour or every 30 minutes | Setup tips and external resources                                  |
| AFK Crypto Website: afkcrypto.com                                                                                                                                                                                                                                                                        | Project credit and external reference                              |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.