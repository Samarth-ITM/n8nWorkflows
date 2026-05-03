Real-Time Stock Monitor with Smart Alerts via Email & Telegram for Indian & US Markets

https://n8nworkflows.xyz/workflows/real-time-stock-monitor-with-smart-alerts-via-email---telegram-for-indian---us-markets-7701


# Real-Time Stock Monitor with Smart Alerts via Email & Telegram for Indian & US Markets

### 1. Workflow Overview

This workflow implements a **Real-Time Stock Price Monitor & Smart Alerts system** for Indian (NSE/BSE) and US stock markets. Its primary purpose is to continuously track stock prices against user-defined upper and lower limits, and send timely alerts via Email and Telegram when price thresholds are crossed. It incorporates cooldown periods to prevent alert spamming and maintains alert history in Google Sheets.

**Key target use cases:**
- Day traders and investors needing instant notifications for price breakouts and breakdowns.
- Portfolio managers monitoring multiple stocks across Indian and US markets.
- Automated multi-channel alerting (Email + Telegram) with smart cooldown logic.

**Logical blocks grouped by function and flow:**

- **1.1 Market Trigger & Input Reception**  
  Nodes that schedule and initiate the workflow, and read the watchlist data from Google Sheets.

- **1.2 Data Parsing & Live Price Fetching**  
  Nodes that parse input data into structured format and fetch live stock prices from an external API.

- **1.3 Alert Decision Logic**  
  Nodes that analyze price data versus thresholds, apply cooldown logic, and determine if alerts need to be sent.

- **1.4 Alert Dispatch & History Update**  
  Nodes that send alerts through Email and Telegram, update alert history in Google Sheets, and track workflow success or failure.

- **1.5 Notification & Monitoring**  
  Nodes that notify administrators about the success or failure of alert dispatches.

- **1.6 Documentation & Setup Notes**  
  Sticky notes providing setup instructions, workflow overview, and feature descriptions.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Market Trigger & Input Reception

**Overview:**  
This block schedules the workflow to run periodically during market hours and reads the stock watchlist from a Google Sheet.

**Nodes Involved:**  
- Market Hours Trigger  
- Read Stock Watchlist

**Node Details:**

- **Market Hours Trigger**
  - *Type:* Cron Trigger  
  - *Role:* Initiates workflow execution on a schedule.  
  - *Configuration:* Default cron schedule (implied every 2 minutes during market hours; exact schedule is empty, suggesting customization needed).  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Triggers Read Stock Watchlist node.  
  - *Edge Cases:* No internal schedule set—user must configure for market hours; otherwise, workflow may run continuously or not at all.

- **Read Stock Watchlist**
  - *Type:* Google Sheets (Read Operation)  
  - *Role:* Fetches stock symbols and alert parameters from a configured Google Sheet.  
  - *Configuration:*  
    - `documentId` and `sheetName` set to `YOUR_GOOGLE_SHEET_ID_HERE` (to be replaced).  
    - Authentication via Google Service Account credential.  
  - *Inputs:* Trigger from Market Hours Trigger.  
  - *Outputs:* Passes raw watchlist data to Parse Watchlist Data node.  
  - *Edge Cases:*  
    - Google API quota limits or authentication errors.  
    - Missing or malformed sheet data.  
    - Empty watchlist leading to no alerts.

---

#### 2.2 Data Parsing & Live Price Fetching

**Overview:**  
This block processes the raw watchlist data into structured JSON objects and retrieves live stock prices from the Twelve Data API.

**Nodes Involved:**  
- Parse Watchlist Data  
- Fetch Live Stock Price

**Node Details:**

- **Parse Watchlist Data**  
  - *Type:* Code (JavaScript)  
  - *Role:* Validates and parses each row from Google Sheets into structured alert parameters.  
  - *Key Logic:*  
    - Extracts `symbol`, `upper_limit`, `lower_limit`.  
    - Optional fields: `direction` (default `'both'`), `cooldown_minutes` (default `15`), `last_alert_price` (default `0`), `last_alert_time` (empty string default).  
    - Filters out incomplete rows lacking required fields.  
  - *Inputs:* Raw rows from Read Stock Watchlist.  
  - *Outputs:* Array of structured watchlist items forwarded to Fetch Live Stock Price.  
  - *Edge Cases:*  
    - Non-numeric or missing numeric fields cause row exclusion.  
    - Empty or malformed inputs lead to empty output, halting downstream logic.

- **Fetch Live Stock Price**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves recent stock price time series data for each symbol using Twelve Data API.  
  - *Configuration:*  
    - URL template: `https://api.twelvedata.com/time_series?symbol={{ $json.symbol }}&interval={{ $json.cooldown_minutes }}min&apikey=your_api_key_add_here`  
    - API key placeholder must be replaced.  
    - Response set to never error (neverError=true) to prevent workflow stop on API errors.  
  - *Inputs:* Structured watchlist items from Parse Watchlist Data node.  
  - *Outputs:* Price data passed to Smart Alert Logic node.  
  - *Edge Cases:*  
    - API key invalid or quota exceeded causes empty or error responses.  
    - Unavailable symbols or malformed symbol strings cause no price data.  
    - Network or timeout errors.

---

#### 2.3 Alert Decision Logic

**Overview:**  
This block applies intelligent logic to detect if price crosses limits and manages cooldowns to prevent alert flooding.

**Nodes Involved:**  
- Smart Alert Logic  
- Check Alert Conditions

**Node Details:**

- **Smart Alert Logic**  
  - *Type:* Code (JavaScript)  
  - *Role:* Analyzes fetched price data vs. upper and lower limits to generate alert messages.  
  - *Key Logic:*  
    - Iterates over up to 20 recent price entries.  
    - Checks if closing price crosses upper or lower limit.  
    - Creates alert object with symbol, alert status (UP/DOWN), price, limit, timestamp, and message.  
    - Returns array of alerts or a single "No alerts" message if none triggered.  
  - *Inputs:* Price data from Fetch Live Stock Price; watchlist parameters from Parse Watchlist Data via expression.  
  - *Outputs:* Alerts or no-alert messages passed to Check Alert Conditions node.  
  - *Edge Cases:*  
    - Empty or no values array handled gracefully.  
    - Incorrect data type inputs could cause runtime errors.  
    - Potential mismatch if API returns unexpected format.

- **Check Alert Conditions**  
  - *Type:* If  
  - *Role:* Validates if alert messages are present and non-empty to proceed with alerts.  
  - *Configuration:* Condition checks if `$json.alertMessage` string is non-empty.  
  - *Inputs:* Alert objects or no-alert message from Smart Alert Logic.  
  - *Outputs:*  
    - True branch: sends alerts (Send Email Alert, Send Telegram Alert, Update Alert History).  
    - False branch: no alert; workflow stops or continues silently.  
  - *Edge Cases:*  
    - Incorrect field references in condition cause false negatives.  
    - Empty or malformed alert messages bypass alerting.

---

#### 2.4 Alert Dispatch & History Update

**Overview:**  
This block sends alert notifications via Email and Telegram, updates alert history in Google Sheets, and triggers status checks.

**Nodes Involved:**  
- Send Email Alert  
- Send Telegram Alert  
- Update Alert History  
- Alert Status Check

**Node Details:**

- **Send Email Alert**  
  - *Type:* Email Send  
  - *Role:* Sends alert email to predefined recipient.  
  - *Configuration:*  
    - Subject and body contain alert message dynamically from Smart Alert Logic.  
    - From and To emails configured statically.  
    - Uses SMTP credentials.  
  - *Inputs:* True branch from Check Alert Conditions.  
  - *Outputs:* None directly; triggers next nodes simultaneously.  
  - *Edge Cases:*  
    - SMTP auth failures or connectivity issues.  
    - Email quota or spam filtering.  
    - Missing dynamic message leads to generic or empty emails.

- **Send Telegram Alert**  
  - *Type:* Telegram  
  - *Role:* Sends instant Telegram message to specified chat ID.  
  - *Configuration:*  
    - Text uses dynamic alert message.  
    - Chat ID must be set by user.  
    - HTML parse mode enabled.  
    - Uses Telegram API credentials.  
  - *Inputs:* True branch from Check Alert Conditions.  
  - *Outputs:* None directly; triggers next nodes simultaneously.  
  - *Edge Cases:*  
    - Invalid chat ID or revoked bot token causes failure.  
    - Telegram API rate limits or connectivity issues.

- **Update Alert History**  
  - *Type:* Google Sheets (Update Operation)  
  - *Role:* Records latest alert time and price back into the Google Sheet watchlist.  
  - *Configuration:*  
    - Updates rows by automatically mapping input data.  
    - Uses same Google Sheets document and sheet as watchlist.  
    - Authentication via service account.  
  - *Inputs:* True branch from Check Alert Conditions (along with alert nodes).  
  - *Outputs:* Passes to Alert Status Check node.  
  - *Edge Cases:*  
    - Update failures due to sheet access or permissions.  
    - Data mapping errors leading to incorrect updates.

- **Alert Status Check**  
  - *Type:* If  
  - *Role:* Checks if the update was successful by validating presence of `symbol` field.  
  - *Configuration:* Condition checks if `$json.symbol` is not empty.  
  - *Inputs:* Output from Update Alert History.  
  - *Outputs:*  
    - True branch: triggers Success Notification.  
    - False branch: triggers Error Notification.  
  - *Edge Cases:*  
    - Missing or malformed response from update causes false negatives.  
    - Edge case where partial update returns empty symbol field.

---

#### 2.5 Notification & Monitoring

**Overview:**  
This block sends administrative email notifications about the overall success or failure of alert dispatching.

**Nodes Involved:**  
- Success Notification  
- Error Notification

**Node Details:**

- **Success Notification**  
  - *Type:* Email Send  
  - *Role:* Notifies admin that an alert was sent successfully.  
  - *Configuration:*  
    - Static subject "✅ Stock Monitor: Alert Sent Successfully".  
    - To and From emails configured.  
    - Uses SMTP credentials.  
  - *Inputs:* True branch of Alert Status Check.  
  - *Outputs:* None.  
  - *Edge Cases:* SMTP or email delivery failures.

- **Error Notification**  
  - *Type:* Email Send  
  - *Role:* Notifies admin that alert sending or update failed.  
  - *Configuration:*  
    - Static subject "❌ Stock Monitor: Alert Failed".  
    - To and From emails configured.  
    - Uses SMTP credentials.  
  - *Inputs:* False branch of Alert Status Check.  
  - *Outputs:* None.  
  - *Edge Cases:* SMTP or email delivery failures.

---

#### 2.6 Documentation & Setup Notes

**Overview:**  
Sticky notes provide setup guidance, workflow description, and feature highlights for users.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1

**Node Details:**

- **Sticky Note**  
  - *Content:*  
    - Describes the workflow purpose and usage scenarios.  
    - Lists required Google Sheets columns in exact order with explanations.  
  - *Role:* User guidance for correct Google Sheets setup.

- **Sticky Note1**  
  - *Content:*  
    - Detailed explanation of node-by-node workflow steps.  
    - Highlights key features such as Smart Cooldown, Multi-Market support, Dual Alerts, Auto-Update, and Error Handling.  
  - *Role:* Workflow architecture overview and user instructions.

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                            | Input Node(s)            | Output Node(s)               | Sticky Note                                                                                                                 |
|------------------------|---------------------|-------------------------------------------|--------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Market Hours Trigger   | Cron Trigger        | Initiates workflow periodically           | None                     | Read Stock Watchlist         | See Sticky Note1 for detailed workflow overview and setup instructions.                                                     |
| Read Stock Watchlist   | Google Sheets       | Reads stock watchlist from Google Sheets  | Market Hours Trigger     | Parse Watchlist Data         | See Sticky Note for Google Sheets column requirements.                                                                       |
| Parse Watchlist Data   | Code                | Parses and validates watchlist data       | Read Stock Watchlist     | Fetch Live Stock Price       |                                                                                                                             |
| Fetch Live Stock Price | HTTP Request        | Fetches live price data from API           | Parse Watchlist Data     | Smart Alert Logic            | API key must be replaced; configured to never error to maintain workflow continuity.                                         |
| Smart Alert Logic      | Code                | Compares prices to limits and generates alerts | Fetch Live Stock Price | Check Alert Conditions       |                                                                                                                             |
| Check Alert Conditions | If                  | Checks if alert messages exist             | Smart Alert Logic        | Send Email Alert, Send Telegram Alert, Update Alert History |                                                                                                                             |
| Send Email Alert       | Email Send          | Sends alert email notification             | Check Alert Conditions   | None                        | Requires SMTP credentials; dynamic message from Smart Alert Logic.                                                           |
| Send Telegram Alert    | Telegram            | Sends alert Telegram message                | Check Alert Conditions   | None                        | Requires Telegram bot credentials and valid chat ID.                                                                         |
| Update Alert History   | Google Sheets       | Updates alert timestamps in Google Sheets  | Check Alert Conditions   | Alert Status Check           | Uses service account; updates last alert price and time.                                                                     |
| Alert Status Check     | If                  | Verifies if alert history update succeeded | Update Alert History     | Success Notification, Error Notification |                                                                                                                             |
| Success Notification   | Email Send          | Notifies admin of success                    | Alert Status Check       | None                        | Requires SMTP credentials.                                                                                                   |
| Error Notification     | Email Send          | Notifies admin of failure                    | Alert Status Check       | None                        | Requires SMTP credentials.                                                                                                   |
| Sticky Note            | Sticky Note         | Provides Google Sheets setup instructions  | None                    | None                        | Contains required columns and setup instructions.                                                                            |
| Sticky Note1           | Sticky Note         | Provides workflow overview and features     | None                    | None                        | Gives detailed node descriptions and key workflow features.                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger node**  
   - Name: `Market Hours Trigger`  
   - Type: Cron Trigger  
   - Configure to run every 2 minutes (or desired frequency) during market hours via cron expressions.

2. **Create a Google Sheets node to read watchlist**  
   - Name: `Read Stock Watchlist`  
   - Type: Google Sheets (Read)  
   - Set `documentId` and `sheetName` to your Google Sheet containing stock watchlist.  
   - Configure authentication with a Google Service Account credential.  
   - Connect output of `Market Hours Trigger` to this node.

3. **Create a Code node to parse watchlist data**  
   - Name: `Parse Watchlist Data`  
   - Type: Code (JavaScript)  
   - Paste the JS code that validates and extracts symbol, limits, direction, cooldown_minutes, last alert data.  
   - Connect output of `Read Stock Watchlist` to this node.

4. **Create an HTTP Request node to fetch live stock prices**  
   - Name: `Fetch Live Stock Price`  
   - Type: HTTP Request  
   - URL: `https://api.twelvedata.com/time_series?symbol={{ $json.symbol }}&interval={{ $json.cooldown_minutes }}min&apikey=your_api_key_add_here`  
   - Replace `your_api_key_add_here` with a valid Twelve Data API key.  
   - Set option `neverError` to true to prevent workflow stops on errors.  
   - Connect output of `Parse Watchlist Data` to this node.

5. **Create a Code node for smart alert logic**  
   - Name: `Smart Alert Logic`  
   - Type: Code (JavaScript)  
   - Implement logic to iterate over price data, compare with limits, generate alert messages.  
   - Use `$input.first().json.values` to access price series.  
   - Connect output of `Fetch Live Stock Price` to this node.

6. **Create an If node to check alert conditions**  
   - Name: `Check Alert Conditions`  
   - Type: If  
   - Condition: `$json.alertMessage` is not empty (string not empty).  
   - Connect output of `Smart Alert Logic` to this node.

7. **Create an Email Send node for email alerts**  
   - Name: `Send Email Alert`  
   - Type: Email Send  
   - Subject & Text: Use expression `🚨 Stock Alert: {{ $('Smart Alert Logic').item.json.message }}`  
   - To: alert recipient email  
   - From: sender email with SMTP credential configured  
   - Connect true output of `Check Alert Conditions` to this node.

8. **Create a Telegram node for Telegram alerts**  
   - Name: `Send Telegram Alert`  
   - Type: Telegram  
   - Text: Use expression `{{ $json.alert_message }}` (ensure this matches alert message property)  
   - Chat ID: Your Telegram chat ID  
   - Use Telegram API credential  
   - Connect true output of `Check Alert Conditions` to this node.

9. **Create a Google Sheets node to update alert history**  
   - Name: `Update Alert History`  
   - Type: Google Sheets (Update)  
   - Document ID and Sheet Name: same as watchlist sheet  
   - Configure to update last alert price and last alert time columns  
   - Use service account credentials  
   - Connect true output of `Check Alert Conditions` to this node.

10. **Create an If node to check update success**  
    - Name: `Alert Status Check`  
    - Type: If  
    - Condition: `$json.symbol` is not empty  
    - Connect output of `Update Alert History` to this node.

11. **Create an Email Send node for success notification**  
    - Name: `Success Notification`  
    - Type: Email Send  
    - Subject: `✅ Stock Monitor: Alert Sent Successfully`  
    - To: admin email  
    - From: sender email with SMTP credential  
    - Connect true output of `Alert Status Check` to this node.

12. **Create an Email Send node for error notification**  
    - Name: `Error Notification`  
    - Type: Email Send  
    - Subject: `❌ Stock Monitor: Alert Failed`  
    - To: admin email  
    - From: sender email with SMTP credential  
    - Connect false output of `Alert Status Check` to this node.

13. **Create Sticky Note nodes for documentation**  
    - Name one `Sticky Note` with Google Sheets setup instructions (columns order and details).  
    - Name another `Sticky Note1` with workflow overview and feature explanations.  
    - Position these for user reference; no connections needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Google Sheets Setup requires columns in exact order: symbol, upper_limit, lower_limit, direction, cooldown_minutes, last_alert_price, last_alert_time.                                                                       | Refer to Sticky Note in workflow for details.                                                  |
| Twelve Data API requires a valid API key; free tier limits may apply. Replace placeholder in HTTP Request node.                                                                                                                | https://twelvedata.com/                                                                       |
| SMTP credentials must be configured properly for email alerts and admin notifications.                                                                                                                                        | Use your SMTP provider settings (e.g., Gmail SMTP with OAuth2 or app password).               |
| Telegram Bot token and chat ID must be correctly set to enable Telegram alerts.                                                                                                                                               | Create Telegram bot via BotFather and get chat ID by messaging or using tools like @userinfobot. |
| Smart cooldown logic prevents alert flooding by respecting cooldown_minutes per stock symbol.                                                                                                                                | Implemented in Smart Alert Logic node code.                                                   |
| Workflow designed for Indian and US markets; ensure stock symbols match API conventions (e.g., NSE: TCS, US: AAPL, BSE: RELIANCE.BSE).                                                                                       | Check Twelve Data symbol formats.                                                             |
| Error handling built-in: API errors do not halt workflow; failure notifications sent to admin email.                                                                                                                          | See Email nodes for error notifications.                                                     |
| For scalability, consider adding concurrency control or splitting watchlist for large portfolios.                                                                                                                             | Performance considerations.                                                                  |

---

This document provides a detailed, stepwise understanding of the Real-Time Stock Monitor workflow, enabling reproduction, extension, and troubleshooting for advanced users and automation agents alike.