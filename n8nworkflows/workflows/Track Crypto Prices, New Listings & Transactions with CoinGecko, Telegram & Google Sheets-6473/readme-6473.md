Track Crypto Prices, New Listings & Transactions with CoinGecko, Telegram & Google Sheets

https://n8nworkflows.xyz/workflows/track-crypto-prices--new-listings---transactions-with-coingecko--telegram---google-sheets-6473


# Track Crypto Prices, New Listings & Transactions with CoinGecko, Telegram & Google Sheets

### 1. Workflow Overview

This workflow, named **"🚀 Crypto Sentinel: Alert, Discover & Log Automation Pack"**, automates monitoring and logging of cryptocurrency data using CoinGecko, Telegram, Binance, and Google Sheets. It targets crypto traders, analysts, and enthusiasts who want real-time alerts, discovery of new coin listings, and transaction logging for better tracking and decision-making.

The workflow is logically divided into three main functional blocks:

- **1.1 Price Monitoring and Alerts:** Periodically checks Bitcoin price and sends Telegram alerts if the price exceeds a threshold.
- **1.2 New Listing Discovery and Notifications:** Listens to an RSS feed for new crypto listings and notifies via Telegram.
- **1.3 Transaction Logging:** Periodically fetches recent Binance trades, formats the data, and logs it into Google Sheets.

Each block is independent but collectively provides a comprehensive crypto sentinel system.

---

### 2. Block-by-Block Analysis

#### 1.1 Price Monitoring and Alerts

**Overview:**  
This block triggers periodically to fetch the current Bitcoin price, evaluates if it surpasses $50,000, and sends a Telegram alert if the condition is met.

**Nodes Involved:**  
- 1. Cron (Price Monitor Trigger)  
- 2. HTTP Request (Check BTC Price)  
- 3. If (Price > $50k)  
- 4. Telegram (Send Alert)

**Node Details:**

- **1. Cron (Price Monitor Trigger)**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates periodic workflow runs (e.g., every X minutes/hours) to monitor BTC price.  
  - *Configuration:* Default scheduling parameters (not specified explicitly).  
  - *Connections:* Outputs to "2. HTTP Request (Check BTC Price)".  
  - *Failure Modes:* Scheduling misconfiguration; trigger not firing.

- **2. HTTP Request (Check BTC Price)**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves the current Bitcoin price from an external API (likely CoinGecko or similar).  
  - *Configuration:* Request method and URL not explicitly detailed but intended to fetch BTC price data.  
  - *Expressions/Variables:* Uses output from the request body/json to extract BTC price.  
  - *Input:* From Cron trigger.  
  - *Output:* Passes data to the If node.  
  - *Failure Modes:* API downtime, rate limiting, malformed response, network issues.

- **3. If (Price > $50k)**  
  - *Type:* Conditional (If) Node  
  - *Role:* Evaluates if BTC price from the HTTP Request node exceeds $50,000.  
  - *Configuration:* Condition compares extracted BTC price value to 50000 (numeric comparison).  
  - *Input:* BTC price data from HTTP Request node.  
  - *Output:* If true, passes data to Telegram alert node; else, stops here.  
  - *Failure Modes:* Expression evaluation errors if price data missing or malformed.

- **4. Telegram (Send Alert)**  
  - *Type:* Telegram Node  
  - *Role:* Sends a message alerting users that Bitcoin price exceeded $50k.  
  - *Configuration:* Uses configured Telegram Bot credentials and chat ID; message text likely includes the current price and alert info.  
  - *Input:* Triggered only on true condition from If node.  
  - *Failure Modes:* Authentication errors, Telegram API downtime, invalid chat ID.  
  - *Special:* Contains a webhookId (used for Telegram integration).  

---

#### 1.2 New Listing Discovery and Notifications

**Overview:**  
Monitors an RSS feed for new cryptocurrency listings and sends Telegram notifications on new entries.

**Nodes Involved:**  
- 1. RSS Feed (New Listing Trigger)  
- 2. Telegram (Listing Notif)

**Node Details:**

- **1. RSS Feed (New Listing Trigger)**  
  - *Type:* RSS Feed Read Trigger  
  - *Role:* Watches an RSS feed for new cryptocurrency listings or announcements.  
  - *Configuration:* RSS feed URL is configurable; note suggests replacing the URL with other exchange feeds as needed.  
  - *Input:* None (trigger node).  
  - *Output:* Feeds new listing data to Telegram node.  
  - *Failure Modes:* RSS feed URL down/unavailable, malformed RSS data, no new items.

- **2. Telegram (Listing Notif)**  
  - *Type:* Telegram Node  
  - *Role:* Sends a Telegram message notifying users about new coin listings extracted from the RSS feed.  
  - *Configuration:* Uses Telegram credentials and chat ID; message likely includes listing title, link, and date.  
  - *Input:* Receives new listing info from RSS feed trigger.  
  - *Failure Modes:* Telegram API errors, invalid credentials/chat ID.

---

#### 1.3 Transaction Logging

**Overview:**  
Periodically fetches recent Binance trades, formats the trade data, and appends it to a Google Sheets document for logging and analysis.

**Nodes Involved:**  
- 1. Cron (Transaction Logger Trigger)  
- 2. HTTP Request (Get Binance Trades)  
- 3. Function (Format Data)  
- 4. Google Sheets (Log Transactions)

**Node Details:**

- **1. Cron (Transaction Logger Trigger)**  
  - *Type:* Schedule Trigger  
  - *Role:* Periodically triggers the workflow to fetch latest Binance trades.  
  - *Configuration:* Default schedule (not detailed).  
  - *Output:* Triggers HTTP Request node.  
  - *Failure Modes:* Misconfigured schedule or trigger failure.

- **2. HTTP Request (Get Binance Trades)**  
  - *Type:* HTTP Request  
  - *Role:* Calls Binance API endpoint to retrieve recent trade transactions.  
  - *Configuration:* Includes Binance API URL for trades; may require API key or public endpoint depending on setup.  
  - *Input:* From Cron trigger.  
  - *Output:* Passes raw trade data to Function node.  
  - *Failure Modes:* API limits, authentication errors, network issues.

- **3. Function (Format Data)**  
  - *Type:* Code (JavaScript) Node  
  - *Role:* Processes raw Binance trade data into a structured format suitable for Google Sheets logging.  
  - *Configuration:* Contains custom JavaScript code to extract relevant fields (e.g., trade ID, price, quantity, timestamp).  
  - *Input:* Raw JSON trade data.  
  - *Output:* Formatted data array for Google Sheets.  
  - *Failure Modes:* Code errors, unexpected data structure, missing fields.

- **4. Google Sheets (Log Transactions)**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends formatted trade data as new rows into a specified Google Sheet.  
  - *Configuration:* Uses Google Sheets OAuth2 credentials, target spreadsheet ID, and worksheet name.  
  - *Input:* Formatted trade data from Function node.  
  - *Failure Modes:* Authentication errors, permission issues, spreadsheet not found, API quota limits.

---

### 3. Summary Table

| Node Name                        | Node Type                 | Functional Role                   | Input Node(s)                    | Output Node(s)               | Sticky Note                                              |
|---------------------------------|---------------------------|---------------------------------|---------------------------------|-----------------------------|----------------------------------------------------------|
| 1. Cron (Price Monitor Trigger) | Schedule Trigger          | Periodic trigger for price check| -                               | 2. HTTP Request (Check BTC Price) |                                                          |
| 2. HTTP Request (Check BTC Price)| HTTP Request             | Fetch current BTC price          | 1. Cron (Price Monitor Trigger)  | 3. If (Price > $50k)         |                                                          |
| 3. If (Price > $50k)             | If Node                  | Check if BTC price > $50k        | 2. HTTP Request (Check BTC Price)| 4. Telegram (Send Alert)     |                                                          |
| 4. Telegram (Send Alert)          | Telegram Node            | Send alert if BTC price high     | 3. If (Price > $50k)             | -                           | Contains webhookId for Telegram integration               |
| 1. RSS Feed (New Listing Trigger)| RSS Feed Read Trigger    | Trigger on new crypto listings   | -                               | 2. Telegram (Listing Notif)  | Note: Replace RSS URL with other exchange feeds if needed|
| 2. Telegram (Listing Notif)       | Telegram Node            | Notify new listings on Telegram  | 1. RSS Feed (New Listing Trigger)| -                           |                                                          |
| 1. Cron (Transaction Logger Trigger) | Schedule Trigger      | Periodic trigger for trades logging | -                            | 2. HTTP Request (Get Binance Trades) |                                                          |
| 2. HTTP Request (Get Binance Trades) | HTTP Request          | Fetch recent Binance trades      | 1. Cron (Transaction Logger Trigger) | 3. Function (Format Data) |                                                          |
| 3. Function (Format Data)          | Code (Function)          | Format Binance trade data        | 2. HTTP Request (Get Binance Trades) | 4. Google Sheets (Log Transactions) |                                                          |
| 4. Google Sheets (Log Transactions)| Google Sheets           | Append trade data to Google Sheet| 3. Function (Format Data)        | -                           |                                                          |
| Sticky Note                      | Sticky Note               | -                               | -                               | -                           |                                                          |
| Sticky Note1                     | Sticky Note               | -                               | -                               | -                           |                                                          |
| Sticky Note2                     | Sticky Note               | -                               | -                               | -                           |                                                          |
| Sticky Note3                     | Sticky Note               | -                               | -                               | -                           |                                                          |
| Sticky Note4                     | Sticky Note               | -                               | -                               | -                           |                                                          |
| Sticky Note5                     | Sticky Note               | -                               | -                               | -                           |                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "1. Cron (Price Monitor Trigger)"**  
   - Node type: Schedule Trigger  
   - Configure interval (e.g., every 5 minutes).  
   - No credentials needed.

2. **Create "2. HTTP Request (Check BTC Price)"**  
   - Node type: HTTP Request  
   - Method: GET  
   - URL: Use CoinGecko API endpoint for BTC price, e.g., `https://api.coingecko.com/api/v3/simple/price?ids=bitcoin&vs_currencies=usd`  
   - No auth needed.  
   - Connect output of Cron node to this node.

3. **Create "3. If (Price > $50k)"**  
   - Node type: If  
   - Condition: Numeric comparison  
   - Expression: Extract BTC price from HTTP response, e.g., `{{$json["bitcoin"]["usd"]}}` > 50000  
   - Connect HTTP Request node output to this node.

4. **Create "4. Telegram (Send Alert)"**  
   - Node type: Telegram  
   - Credentials: Set up Telegram Bot credentials using Bot Token.  
   - Chat ID: Set the target Telegram chat or channel ID.  
   - Message: Compose message including BTC price using expression, e.g., `"Bitcoin price alert: ${{$json["bitcoin"]["usd"]}}"`  
   - Connect "true" output from If node to this node.

5. **Create "1. RSS Feed (New Listing Trigger)"**  
   - Node type: RSS Feed Read Trigger  
   - URL: Set the RSS feed URL of the crypto exchange or listing source.  
   - Configure polling frequency (e.g., every 15 minutes).  
   - No auth needed.

6. **Create "2. Telegram (Listing Notif)"**  
   - Node type: Telegram  
   - Credentials: Use same or another Telegram Bot credentials.  
   - Chat ID: Target chat/channel ID.  
   - Message: Compose message using RSS feed item fields, e.g., title and link.  
   - Connect RSS Feed trigger output to this node.

7. **Create "1. Cron (Transaction Logger Trigger)"**  
   - Node type: Schedule Trigger  
   - Configure interval (e.g., every 10 minutes).  
   - No credentials needed.

8. **Create "2. HTTP Request (Get Binance Trades)"**  
   - Node type: HTTP Request  
   - Method: GET  
   - URL: Binance API endpoint for recent trades, e.g., `https://api.binance.com/api/v3/trades?symbol=BTCUSDT&limit=10`  
   - Auth: Depending on API endpoint, may not require API key for public endpoints.  
   - Connect from Cron trigger.

9. **Create "3. Function (Format Data)"**  
   - Node type: Function (Code)  
   - JavaScript code: Write code to map Binance trades JSON into an array of arrays or objects with columns matching Google Sheets structure.  
   - Connect HTTP Request output to this node.

10. **Create "4. Google Sheets (Log Transactions)"**  
    - Node type: Google Sheets  
    - Credentials: Connect Google account with OAuth2 credentials and authorize access.  
    - Action: Append Rows  
    - Spreadsheet ID: Specify target spreadsheet  
    - Sheet Name: Target worksheet/tab name  
    - Data: Use output from Function node.  
    - Connect Function node output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| To customize new listing notifications, replace the RSS feed URL with another exchange's feed.  | Mentioned in "1. RSS Feed (New Listing Trigger)" node notes.  |
| Telegram nodes require proper Bot Token and chat/channel ID configuration to work correctly.    | Telegram API documentation: https://core.telegram.org/bots/api|
| Binance API public endpoints can be used without authentication for limited data.                | Binance API docs: https://binance-docs.github.io/apidocs/spot/en/#recent-trades-list |
| Google Sheets node requires OAuth2 credentials with edit access to the target spreadsheet.      | n8n docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/ |

---

**Disclaimer:**  
The provided text is derived solely from an automated n8n workflow. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.