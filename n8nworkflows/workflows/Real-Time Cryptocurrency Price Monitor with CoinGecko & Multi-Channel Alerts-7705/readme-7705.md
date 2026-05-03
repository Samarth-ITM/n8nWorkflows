Real-Time Cryptocurrency Price Monitor with CoinGecko & Multi-Channel Alerts

https://n8nworkflows.xyz/workflows/real-time-cryptocurrency-price-monitor-with-coingecko---multi-channel-alerts-7705


# Real-Time Cryptocurrency Price Monitor with CoinGecko & Multi-Channel Alerts

### 1. Workflow Overview

This workflow implements a **Real-Time Cryptocurrency Price Monitor with Multi-Channel Alerts** using CoinGecko’s free API and Google Sheets as a watchlist source. It is designed for crypto traders, DeFi investors, and portfolio managers who require instant notifications on cryptocurrency price movements, including breakout (price rising above a threshold) and breakdown (price falling below a threshold) alerts.

The workflow operates continuously via a cron trigger and includes logic for:

- Fetching and parsing a user-maintained crypto watchlist from Google Sheets  
- Mapping crypto symbols and trading pairs to CoinGecko coin IDs  
- Retrieving live crypto prices with market data such as 24h change and market cap  
- Applying smart alert logic with cooldown periods to avoid alert spam  
- Sending alerts through Email, Telegram, and Discord channels  
- Updating alert history back to Google Sheets  
- Admin notification on workflow success or failure for monitoring  

**Logical Blocks:**

- **1.1 Trigger & Input Reception:** Scheduling periodic execution and reading watchlist data  
- **1.2 Data Parsing & Mapping:** Converting sheet data into structured crypto queries  
- **1.3 Live Price Fetching:** Querying CoinGecko API for current prices and market info  
- **1.4 Smart Alert Evaluation:** Logic to decide if price meets alert conditions and cooldown rules  
- **1.5 Alert Condition Check:** Filtering alerts based on message presence  
- **1.6 Multi-Channel Alert Dispatch:** Sending notifications via Email, Telegram, and Discord  
- **1.7 Alert History Update:** Recording alert timestamps and prices to Google Sheets  
- **1.8 Workflow Health Monitoring:** Sending success or error notifications to admin  
- **1.9 Documentation & Setup Notes:** Sticky notes providing usage instructions and examples

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Reception

**Overview:**  
Schedules workflow execution every minute to ensure continuous crypto price monitoring and reads the user’s crypto watchlist from Google Sheets.

**Nodes Involved:**  
- 24/7 Crypto Trigger (Cron)  
- Read Crypto Watchlist (Google Sheets)

**Node Details:**

- **24/7 Crypto Trigger**  
  - Type: Cron Trigger  
  - Config: Default settings (runs every minute) to ensure real-time monitoring  
  - No inputs, outputs to Read Crypto Watchlist  
  - Edge Cases: Node downtime or cron misconfiguration could cause missed checks  

- **Read Crypto Watchlist**  
  - Type: Google Sheets Read  
  - Config: Reads data from "Sheet1" of a specified Google Sheet document by ID  
  - Authentication: Service Account credential for Google API access  
  - Input: Trigger from Cron node  
  - Output: Raw watchlist rows (including headers) to Parse Crypto Data  
  - Edge Cases: Sheet missing, invalid document ID, or auth failure may cause errors or empty data  

---

#### 1.2 Data Parsing & Mapping

**Overview:**  
Processes the raw Google Sheets data, skipping the header row, mapping common crypto symbols and pair formats to CoinGecko coin IDs, and preparing structured data for API calls.

**Nodes Involved:**  
- Parse Crypto Data (Code)

**Node Details:**

- **Parse Crypto Data**  
  - Type: Code (JavaScript)  
  - Config: Custom JS code to:  
    - Skip header row  
    - Validate required columns (Symbol, Upper Limit, Lower Limit)  
    - Map various symbol formats (e.g., BTC, btc/usdt) to CoinGecko IDs using a predefined dictionary  
    - Parse numeric fields for price limits and cooldowns  
    - Provide defaults for direction (“both”) and cooldown (10 minutes)  
  - Input: Raw Google Sheets rows  
  - Output: Array of JSON items with coin_id, price limits, directions, cooldowns, and last alert data  
  - Edge Cases: Missing or malformed rows are skipped; unsupported symbols fallback to base part before “/” or “-”  
  - Version Note: Uses n8n v2 code node features (async support not required)  

---

#### 1.3 Live Price Fetching

**Overview:**  
Retrieves live cryptocurrency prices and market metrics (USD price, 24h change, market cap) from the CoinGecko public API.

**Nodes Involved:**  
- Fetch Live Crypto Price (HTTP Request)

**Node Details:**

- **Fetch Live Crypto Price**  
  - Type: HTTP Request  
  - Config:  
    - URL dynamically built using mapped `coin_id` from previous node  
    - Queries CoinGecko’s `/simple/price` endpoint for USD price, 24h change, and market cap  
    - Response configured to never error to avoid workflow failure on API issues  
  - Input: Parsed crypto data with coin_id  
  - Output: Raw JSON response from CoinGecko, passed to Smart Crypto Alert Logic  
  - Edge Cases: API downtime, rate limiting; malformed or missing data handled downstream  

---

#### 1.4 Smart Alert Evaluation

**Overview:**  
Analyzes live price data to determine if breakout or breakdown alerts should be triggered based on user-defined thresholds and cooldown periods.

**Nodes Involved:**  
- Smart Crypto Alert Logic (Code)

**Node Details:**

- **Smart Crypto Alert Logic**  
  - Type: Code (JavaScript)  
  - Config:  
    - Parses CoinGecko API JSON response  
    - Extracts current price, 24h change percentage, market cap  
    - Checks price against upper and lower limits per configured direction (“above”, “below”, “both”)  
    - Applies cooldown logic comparing last alert time against cooldown_minutes to prevent alert flooding  
    - Constructs rich alert messages with emoji, formatted numbers, percentage pump/dump calculations, timestamps  
    - Returns alert items only if triggered, including data for updating Google Sheets  
  - Input: Live price API responses  
  - Output: Alert objects with detailed alert messages and metadata  
  - Edge Cases: Invalid JSON responses, missing price data skip processing; cooldown prevents duplicate alerts  

---

#### 1.5 Alert Condition Check

**Overview:**  
Filters alerts to only pass those where a meaningful alert message exists, effectively gating the alert dispatch nodes.

**Nodes Involved:**  
- Check Crypto Alert Conditions (If)

**Node Details:**

- **Check Crypto Alert Conditions**  
  - Type: If  
  - Config: Checks if `alert_message` field is not empty  
  - Input: Alerts from Smart Crypto Alert Logic  
  - Output: If true, routes to alert dispatch nodes; else, no further action  
  - Edge Cases: Empty alert messages prevent unnecessary downstream processing  

---

#### 1.6 Multi-Channel Alert Dispatch

**Overview:**  
Sends triggered alerts through Email, Telegram, and Discord channels and updates alert history in Google Sheets.

**Nodes Involved:**  
- Send Crypto Email Alert (Email Send)  
- Send Telegram Crypto Alert (Telegram)  
- Send Discord Crypto Alert (Discord)  
- Update Crypto Alert History (Google Sheets)

**Node Details:**

- **Send Crypto Email Alert**  
  - Type: Email Send  
  - Config:  
    - Subject dynamically built with symbol, alert type, and current price  
    - Recipient: crypto-alerts@gmail.com (example)  
    - From: your-email@gmail.com (configured SMTP credential)  
  - Input: Alerts from Condition Check  
  - Output: None (side effect)  
  - Edge Cases: SMTP auth failure, email deliverability issues  

- **Send Telegram Crypto Alert**  
  - Type: Telegram  
  - Config:  
    - Sends alert_message text to configured Telegram chat ID  
    - Parse mode set to HTML for formatting  
  - Input: Alerts from Condition Check  
  - Output: None  
  - Edge Cases: Invalid chat ID, Telegram API downtime  

- **Send Discord Crypto Alert**  
  - Type: Discord  
  - Config:  
    - Sends alert messages to specified Discord guild (server) via bot  
    - Uses Discord Bot API credential with appropriate permissions  
  - Input: Alerts from Condition Check  
  - Output: None  
  - Edge Cases: Bot permissions, guild ID correctness, Discord API limits  

- **Update Crypto Alert History**  
  - Type: Google Sheets Update  
  - Config:  
    - Updates the same "Sheet1" document with new last alert price and time  
    - Uses “autoMapInputData” to map updated fields to the sheet columns  
    - Authentication via service account  
  - Input: Alerts from Condition Check  
  - Output: Passes to Crypto Alert Status Check  
  - Edge Cases: Sheet write permissions, row matching failures  

---

#### 1.7 Workflow Health Monitoring

**Overview:**  
Checks if alerts were processed successfully by verifying presence of symbol data, then sends admin notifications on success or failure.

**Nodes Involved:**  
- Crypto Alert Status Check (If)  
- Success Notification (Email Send)  
- Error Notification (Email Send)

**Node Details:**

- **Crypto Alert Status Check**  
  - Type: If  
  - Config: Checks if `symbol` field is non-empty in the input JSON  
  - Input: Output of Update Crypto Alert History node  
  - Output: True branch triggers Success Notification; false branch triggers Error Notification  

- **Success Notification**  
  - Type: Email Send  
  - Config:  
    - Notifies admin@yourcompany.com of successful alert execution  
    - Uses SMTP credentials  
  - Input: True output from Status Check  

- **Error Notification**  
  - Type: Email Send  
  - Config:  
    - Notifies admin@yourcompany.com of failure  
    - Uses SMTP credentials  
  - Input: False output from Status Check  

- Edge Cases: Email delivery failures for notifications  

---

#### 1.8 Documentation & Setup Notes

**Overview:**  
Sticky notes provide detailed explanations, usage instructions, and example watchlist formats for user reference.

**Nodes Involved:**  
- Sticky Note (Real-Time Cryptocurrency Price Monitor & Smart Alerts)  
- Crypto Workflow Guide (Sticky Note)  
- Crypto Setup Examples (Sticky Note)

**Node Details:**

- Sticky notes contain:  
  - Overview and project description  
  - Workflow step explanations  
  - Example Google Sheets data format and supported symbol mappings  
  - Instructions on alert parameters and multi-channel alerting  

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                          | Input Node(s)              | Output Node(s)                                                  | Sticky Note                                                                                      |
|----------------------------|--------------------|----------------------------------------|----------------------------|----------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| 24/7 Crypto Trigger         | Cron Trigger       | Periodic execution trigger (every min) | -                          | Read Crypto Watchlist                                           |                                                                                                  |
| Read Crypto Watchlist       | Google Sheets      | Reads crypto watchlist from Google Sheets | 24/7 Crypto Trigger        | Parse Crypto Data                                              |                                                                                                  |
| Parse Crypto Data           | Code               | Parses symbols, maps to CoinGecko IDs  | Read Crypto Watchlist       | Fetch Live Crypto Price                                        |                                                                                                  |
| Fetch Live Crypto Price     | HTTP Request       | Fetches live crypto prices & market data | Parse Crypto Data           | Smart Crypto Alert Logic                                      |                                                                                                  |
| Smart Crypto Alert Logic    | Code               | Determines if alerts should trigger     | Fetch Live Crypto Price     | Check Crypto Alert Conditions                                  |                                                                                                  |
| Check Crypto Alert Conditions| If                 | Filters alerts with meaningful messages | Smart Crypto Alert Logic    | Send Crypto Email Alert, Send Telegram Crypto Alert, Send Discord Crypto Alert, Update Crypto Alert History |                                                                                                  |
| Send Crypto Email Alert     | Email Send         | Sends email alerts                      | Check Crypto Alert Conditions | -                                                              |                                                                                                  |
| Send Telegram Crypto Alert  | Telegram           | Sends Telegram alerts                   | Check Crypto Alert Conditions | -                                                              |                                                                                                  |
| Send Discord Crypto Alert   | Discord            | Sends Discord alerts                    | Check Crypto Alert Conditions | -                                                              |                                                                                                  |
| Update Crypto Alert History | Google Sheets      | Updates alert history in Google Sheets | Check Crypto Alert Conditions | Crypto Alert Status Check                                      |                                                                                                  |
| Crypto Alert Status Check   | If                 | Checks success of alert update          | Update Crypto Alert History | Success Notification (if true), Error Notification (if false) |                                                                                                  |
| Success Notification        | Email Send         | Sends success notification to admin    | Crypto Alert Status Check   | -                                                              |                                                                                                  |
| Error Notification          | Email Send         | Sends error notification to admin      | Crypto Alert Status Check   | -                                                              |                                                                                                  |
| Sticky Note                | Sticky Note        | Workflow description and overview       | -                          | -                                                              | ## 🚀 Real-Time Cryptocurrency Price Monitor & Smart Alerts... (overview and purpose details)     |
| Crypto Workflow Guide       | Sticky Note        | Detailed workflow step explanations     | -                          | -                                                              | ## 🔧 How It Works... (stepwise workflow logic and crypto-specific features)                      |
| Crypto Setup Examples       | Sticky Note        | Example watchlist formats and symbols   | -                          | -                                                              | ## 💎 Crypto Watchlist Examples... (supported symbols and sheet format details)                   |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create Cron Trigger Node**  
- Node Type: Cron  
- Name: "24/7 Crypto Trigger"  
- Set to trigger every 1 minute (default)  
- No credentials needed  

**Step 2: Add Google Sheets Node to Read Watchlist**  
- Node Type: Google Sheets  
- Name: "Read Crypto Watchlist"  
- Operation: Read rows from Sheet1  
- Document ID: Your Google Sheet ID containing crypto watchlist  
- Authentication: Use Google API credentials (Service Account recommended)  
- Connect "24/7 Crypto Trigger" output to this node's input  

**Step 3: Add Code Node to Parse Crypto Data**  
- Node Type: Code (JavaScript)  
- Name: "Parse Crypto Data"  
- Paste the provided JS code that:  
  - Skips header row  
  - Maps symbols to CoinGecko IDs  
  - Parses numeric limits and cooldowns  
  - Sets default directions and cooldowns  
- Connect "Read Crypto Watchlist" output to this node  

**Step 4: Add HTTP Request Node to Fetch Live Prices**  
- Node Type: HTTP Request  
- Name: "Fetch Live Crypto Price"  
- Method: GET  
- URL: `https://api.coingecko.com/api/v3/simple/price?ids={{ $json.coin_id }}&vs_currencies=usd&include_24hr_change=true&include_market_cap=true`  
- Set "Never Error" on response to true  
- Connect "Parse Crypto Data" output to this node  

**Step 5: Add Code Node for Smart Alert Logic**  
- Node Type: Code (JavaScript)  
- Name: "Smart Crypto Alert Logic"  
- Paste the provided JS logic to:  
  - Parse API response  
  - Compare current price with upper/lower limits  
  - Apply cooldown check  
  - Construct alert messages with detailed info  
  - Return alert items only if triggered  
- Connect "Fetch Live Crypto Price" output to this node  

**Step 6: Add If Node to Check Alert Conditions**  
- Node Type: If  
- Name: "Check Crypto Alert Conditions"  
- Condition: `alert_message` field is not empty  
- Connect "Smart Crypto Alert Logic" output to this node  

**Step 7: Add Email Send Node for Email Alerts**  
- Node Type: Email Send  
- Name: "Send Crypto Email Alert"  
- Subject Template: `🚨 CRYPTO ALERT: {{ $json.symbol }} {{ $json.alert_type === 'upper' ? '🚀 BREAKOUT' : '📉 BREAKDOWN' }} - ${{ $json.current_price }}`  
- To: crypto-alerts@gmail.com (replace with actual recipient)  
- From: your-email@gmail.com (set SMTP credentials)  
- Connect "Check Crypto Alert Conditions" true output to this node  

**Step 8: Add Telegram Node for Telegram Alerts**  
- Node Type: Telegram  
- Name: "Send Telegram Crypto Alert"  
- Chat ID: Your Telegram Chat ID  
- Text: `{{ $json.alert_message }}`  
- Parse Mode: HTML  
- Connect "Check Crypto Alert Conditions" true output to this node  

**Step 9: Add Discord Node for Discord Alerts**  
- Node Type: Discord  
- Name: "Send Discord Crypto Alert"  
- Guild ID: Your Discord Server ID  
- Use Discord Bot API credentials with permissions to post messages  
- Connect "Check Crypto Alert Conditions" true output to this node  

**Step 10: Add Google Sheets Node to Update Alert History**  
- Node Type: Google Sheets  
- Name: "Update Crypto Alert History"  
- Operation: Update rows in the same Sheet1  
- Use auto mapping for input data to match columns  
- Use Service Account credentials  
- Connect "Check Crypto Alert Conditions" true output to this node  

**Step 11: Add If Node to Check Alert Update Success**  
- Node Type: If  
- Name: "Crypto Alert Status Check"  
- Condition: Check if `symbol` field is not empty  
- Connect "Update Crypto Alert History" output to this node  

**Step 12: Add Email Send Node for Success Notification**  
- Node Type: Email Send  
- Name: "Success Notification"  
- Subject: "✅ Crypto Monitor: Alert Sent Successfully"  
- To: admin@yourcompany.com  
- From: your-email@gmail.com  
- Connect "Crypto Alert Status Check" true output to this node  

**Step 13: Add Email Send Node for Error Notification**  
- Node Type: Email Send  
- Name: "Error Notification"  
- Subject: "❌ Crypto Monitor: Alert Failed"  
- To: admin@yourcompany.com  
- From: your-email@gmail.com  
- Connect "Crypto Alert Status Check" false output to this node  

**Step 14: Add Sticky Notes for Documentation (Optional)**  
- Create Sticky Note nodes to document:  
  - Workflow overview and purpose  
  - Workflow step guidance  
  - Watchlist format examples and symbol mapping  

**Final:**  
- Ensure all credentials (Google API, SMTP, Telegram, Discord) are correctly configured and tested.  
- Test workflow with sample watchlist data in Google Sheets.  
- Activate workflow for continuous operation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                               | Context or Link                                                                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Uses CoinGecko free API for real-time price and market data: includes 24h change and market cap to enhance alert quality.                                                                                                 | https://www.coingecko.com/en/api                                                                                                                               |
| Watchlist is maintained in Google Sheets with columns: Symbol, Upper Limit, Lower Limit, Direction, Cooldown (minutes), Last Price, Last Alert Time.                                                                       | Example watchlist format provided in "Crypto Setup Examples" sticky note                                                                                         |
| Multi-channel alerts improve reliability and user reach: Email for formal alerts, Telegram for mobile push, Discord for community or team notifications.                                                                   | Telegram Bot API and Discord Bot API credentials are required and must be set up with proper permissions.                                                       |
| Cooldown periods prevent alert flooding, configurable per crypto in minutes, defaulting to 10 minutes.                                                                                                                     | Critical to avoid spam in volatile markets.                                                                                                                    |
| Alerts include rich formatting with emojis, percentage pump/dump calculations, timestamps, 24h price changes, and market cap for better trader context.                                                                     | Message formatting uses HTML in Telegram and custom formatting in email.                                                                                        |
| Admin email notifications on workflow success or failure enable monitoring for operational issues.                                                                                                                        | Ensure SMTP credentials and admin email are correctly configured.                                                                                               |
| This workflow supports over 1000 cryptocurrencies via CoinGecko and handles common symbol pair formats automatically (e.g., BTC/USDT → bitcoin).                                                                             | Symbol mapping is extensible in the Parse Crypto Data code node.                                                                                               |
| Sticky notes embedded in the workflow provide comprehensive instructions, examples, and setup tips for ease of use and maintenance.                                                                                        | Review sticky notes regularly for updates or modifications.                                                                                                   |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. All data handled is legal and public. The workflow respects all applicable content policies and contains no illegal, offensive, or protected elements.