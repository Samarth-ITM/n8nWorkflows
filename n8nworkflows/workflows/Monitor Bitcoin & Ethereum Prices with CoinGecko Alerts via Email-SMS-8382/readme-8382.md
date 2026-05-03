Monitor Bitcoin & Ethereum Prices with CoinGecko Alerts via Email/SMS

https://n8nworkflows.xyz/workflows/monitor-bitcoin---ethereum-prices-with-coingecko-alerts-via-email-sms-8382


# Monitor Bitcoin & Ethereum Prices with CoinGecko Alerts via Email/SMS

### 1. Workflow Overview

This workflow monitors Bitcoin (BTC) and Ethereum (ETH) prices by periodically querying the CoinGecko API. It compares current prices and 24-hour percentage changes against predefined thresholds and sends alerts via email and SMS/WhatsApp when those thresholds are crossed. The workflow is designed for cryptocurrency traders or enthusiasts who want timely notifications based on significant price movements or crossing target price points.

**Logical Blocks:**

- **1.1 Scheduled Trigger:** Periodically initiates the workflow every 15 minutes.
- **1.2 Data Retrieval:** Fetches the latest BTC and ETH prices and 24h changes from CoinGecko.
- **1.3 Threshold Computation:** Analyzes the fetched data to determine if any price or change thresholds are exceeded.
- **1.4 Alert Decision:** Checks if any alert conditions are met to proceed.
- **1.5 Alert Formatting:** Prepares the alert message content for email and SMS.
- **1.6 Alert Delivery:** Sends alerts via Gmail and Twilio.
- **1.7 Documentation:** Provides a sticky note with comprehensive information about the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:** This block triggers the workflow automatically every 15 minutes to initiate the price monitoring process.
- **Nodes Involved:** 
  - Every 15 Minutes
- **Node Details:**
  - **Name:** Every 15 Minutes  
  - **Type:** Schedule Trigger  
  - **Role:** Periodic workflow starter  
  - **Configuration:** Set to trigger at 15-minute intervals using the minutes field in the schedule rule.  
  - **Input:** None (trigger node)  
  - **Output:** Triggers the “Get Crypto Prices” node  
  - **Version:** 1.1  
  - **Edge Cases:**  
    - Workflow will not run if n8n instance is offline or schedule misconfigured.  
    - Scheduling precision depends on n8n execution environment timers.

#### 2.2 Data Retrieval

- **Overview:** Queries CoinGecko’s public API for current BTC and ETH prices in USD and their 24-hour percentage changes.
- **Nodes Involved:** 
  - Get Crypto Prices
- **Node Details:**
  - **Name:** Get Crypto Prices  
  - **Type:** HTTP Request  
  - **Role:** Fetch external cryptocurrency data  
  - **Configuration:**  
    - URL: `https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum&vs_currencies=usd&include_24hr_change=true`  
    - Method: GET (default)  
    - No authentication needed (public API)  
  - **Input:** Trigger from “Every 15 Minutes”  
  - **Output:** JSON containing BTC and ETH USD prices and 24h change percentages  
  - **Version:** 4.1  
  - **Edge Cases:**  
    - API downtime or rate limiting by CoinGecko can cause failures.  
    - Network issues or malformed responses can break downstream logic.  
    - No error handling node present to retry or fallback.

#### 2.3 Threshold Computation

- **Overview:** Evaluates prices and 24h changes against preset threshold values to determine if alerts should be raised.
- **Nodes Involved:** 
  - Compute Threshold Flags
- **Node Details:**
  - **Name:** Compute Threshold Flags  
  - **Type:** Code (JavaScript)  
  - **Role:** Business logic to compute alert flags  
  - **Configuration:**  
    - Threshold constants defined:  
      - BTC_UP = 110,000 USD  
      - BTC_DOWN = 105,000 USD  
      - ETH_UP = 4,500 USD  
      - ETH_DOWN = 4,200 USD  
      - MOVE_ABS = 2.0% (absolute 24h % change threshold)  
    - Extracts BTC and ETH prices and 24h changes from input JSON  
    - Computes boolean flags for:  
      - Price crossing upper or lower thresholds  
      - Big moves in 24h % change exceeding MOVE_ABS  
    - Returns a JSON object with these flags and timestamps  
  - **Input:** JSON data from “Get Crypto Prices”  
  - **Output:** JSON with price info and boolean flags (cross_up, cross_down, big_move) for BTC and ETH  
  - **Version:** 2  
  - **Edge Cases:**  
    - If API response lacks expected fields, computations may yield NaN or false negatives.  
    - Assumes prices and changes are finite numbers.  
    - No exception handling in code node; JavaScript errors can break workflow.

#### 2.4 Alert Decision

- **Overview:** Determines if any alert flags are true to decide whether to proceed with sending alerts.
- **Nodes Involved:** 
  - Any Alert?
- **Node Details:**
  - **Name:** Any Alert?  
  - **Type:** If (Boolean OR conditions)  
  - **Role:** Conditional filter node  
  - **Configuration:**  
    - Checks boolean flags: btc.cross_up, btc.cross_down, btc.big_move, eth.cross_up, eth.cross_down, eth.big_move  
    - Returns true if any condition is true (logical OR)  
  - **Input:** Flags JSON from “Compute Threshold Flags”  
  - **Output:**  
    - True branch: proceeds to “Format Alert Email”  
    - False branch: workflow ends silently  
  - **Version:** 2  
  - **Edge Cases:**  
    - If flags are missing or undefined, condition evaluation may fail silently.  
    - No default branch handling, so no alert if no flag is true.

#### 2.5 Alert Formatting

- **Overview:** Builds the subject line and message body for email and SMS alerts based on current prices and changes.
- **Nodes Involved:** 
  - Format Alert Email
- **Node Details:**
  - **Name:** Format Alert Email  
  - **Type:** Code (JavaScript)  
  - **Role:** Compose alert messages  
  - **Configuration:**  
    - Defines helper functions to format numbers with fixed decimals and prepend sign for positive changes  
    - Constructs lines summarizing BTC and ETH prices and 24h % changes (e.g., “BTC: $110,000 (24h: +2.1%)”)  
    - Creates JSON with:  
      - Subject line including current local date/time in Africa/Lagos timezone and alert emoji  
      - HTML formatted message for email body  
      - Plain text message for SMS/WhatsApp  
  - **Input:** Alert flags JSON from “Any Alert?” (true branch)  
  - **Output:** JSON with subject, html, and text fields for downstream messaging nodes  
  - **Version:** 2  
  - **Edge Cases:**  
    - Date/time formatting depends on server timezone support.  
    - If input JSON is incomplete, may produce incorrect messages.  
    - No fallback for empty or malformed data.

#### 2.6 Alert Delivery

- **Overview:** Sends the formatted alert messages through email and SMS/WhatsApp channels.
- **Nodes Involved:** 
  - Send a message (Gmail)  
  - Send an SMS/MMS/WhatsApp message (Twilio)
- **Node Details:**

  - **Send a message**  
    - **Type:** Gmail  
    - **Role:** Sends alert email  
    - **Configuration:**  
      - Recipient: `recipient@gmail.com` (placeholder)  
      - Message body: HTML from “Format Alert Email” node  
      - Subject: from “Format Alert Email” node  
      - Appends attribution (email footer) enabled  
      - Credentials: Gmail OAuth2 (configured externally)  
    - Input: From “Format Alert Email”  
    - Output: None (final node)  
    - Version: 2.1  
    - Edge Cases:  
      - Authentication failure if OAuth token expires or revoked  
      - Gmail API rate limits or quota exceeded  
      - Invalid recipient email causes delivery failure

  - **Send an SMS/MMS/WhatsApp message**  
    - **Type:** Twilio  
    - **Role:** Sends alert as SMS, MMS, or WhatsApp message  
    - **Configuration:**  
      - Message text: plain text from “Format Alert Email” node  
      - Credentials: Twilio account (configured externally)  
    - Input: From “Format Alert Email”  
    - Output: None (final node)  
    - Version: 1  
    - Edge Cases:  
      - Twilio authentication errors (invalid SID/token)  
      - Exceeding messaging limits or account balance issues  
      - Invalid recipient phone number format  
      - Carrier or regional restrictions on WhatsApp/SMS

#### 2.7 Documentation

- **Overview:** Provides detailed explanation, usage notes, and customization tips for the workflow.
- **Nodes Involved:** 
  - Sticky Note
- **Node Details:**
  - **Name:** Sticky Note  
  - **Type:** Sticky Note  
  - **Role:** In-workflow documentation and user guidance  
  - **Content:**  
    - Workflow purpose and trigger frequency  
    - Nodes summary and typical output example  
    - Customization instructions (thresholds, channels)  
    - Pro tip for debouncing alerts using external memory or sheets  
    - Author information and contact for consulting  
    - Contains a mailto link for sales contact  
  - **Input/Output:** None  
  - **Version:** 1  
  - **Edge Cases:** None (purely informational)

---

### 3. Summary Table

| Node Name                        | Node Type             | Functional Role                          | Input Node(s)              | Output Node(s)                              | Sticky Note                                                                                                             |
|---------------------------------|-----------------------|----------------------------------------|----------------------------|---------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Every 15 Minutes                | Schedule Trigger      | Initiates workflow every 15 minutes    | -                          | Get Crypto Prices                           |                                                                                                                         |
| Get Crypto Prices               | HTTP Request          | Fetches BTC and ETH prices & 24h changes | Every 15 Minutes           | Compute Threshold Flags                     |                                                                                                                         |
| Compute Threshold Flags         | Code                  | Computes alert flags based on thresholds | Get Crypto Prices          | Any Alert?                                  |                                                                                                                         |
| Any Alert?                     | If                    | Checks if any alert condition is true  | Compute Threshold Flags     | Format Alert Email                          |                                                                                                                         |
| Format Alert Email             | Code                  | Formats email and SMS alert content    | Any Alert? (true branch)    | Send a message, Send an SMS/MMS/WhatsApp message |                                                                                                                         |
| Send a message                 | Gmail                 | Sends alert email                      | Format Alert Email          | -                                           |                                                                                                                         |
| Send an SMS/MMS/WhatsApp message | Twilio                | Sends alert SMS/WhatsApp message       | Format Alert Email          | -                                           |                                                                                                                         |
| Sticky Note                   | Sticky Note           | Documentation and workflow explanation | -                          | -                                           | ## ⚠️ Crypto Price Threshold Alerts (Email/SMS/Telegram) ... Full author info and contact at sales@daexai.com            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Name: `Every 15 Minutes`
   - Type: Schedule Trigger
   - Parameters: Set to trigger every 15 minutes under the “interval” > “minutes” field.
   - Connect output to the next node.

2. **Add HTTP Request node:**
   - Name: `Get Crypto Prices`
   - Type: HTTP Request
   - Parameters:  
     - URL: `https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum&vs_currencies=usd&include_24hr_change=true`  
     - Method: GET  
     - No Authentication required  
   - Connect input from `Every 15 Minutes` node.

3. **Add Code node for threshold computation:**
   - Name: `Compute Threshold Flags`
   - Type: Code (JavaScript)
   - Parameters (JS code):
     ```javascript
     const BTC_UP=110000, BTC_DOWN=105000, ETH_UP=4500, ETH_DOWN=4200, MOVE_ABS=2.0;
     const j=$json; const now=new Date().toISOString();
     const btcPrice=Number(j.bitcoin?.usd), btcChg=Number(j.bitcoin?.usd_24h_change);
     const ethPrice=Number(j.ethereum?.usd), ethChg=Number(j.ethereum?.usd_24h_change);
     const flags={
       btc:{price:btcPrice, chg24:btcChg, cross_up:isFinite(btcPrice)&&btcPrice>=BTC_UP, cross_down:isFinite(btcPrice)&&btcPrice<=BTC_DOWN, big_move:isFinite(btcChg)&&Math.abs(btcChg)>=MOVE_ABS},
       eth:{price:ethPrice, chg24:ethChg, cross_up:isFinite(ethPrice)&&ethPrice>=ETH_UP, cross_down:isFinite(ethPrice)&&ethPrice<=ETH_DOWN, big_move:isFinite(ethChg)&&Math.abs(ethChg)>=MOVE_ABS},
       ts: now
     };
     return [{json: flags}];
     ```
   - Connect input from `Get Crypto Prices`.

4. **Add If node for alert conditions:**
   - Name: `Any Alert?`
   - Type: If
   - Parameters:
     - Conditions: Boolean OR group with the following expressions:
       - `{{ $json.btc.cross_up }}`
       - `{{ $json.btc.cross_down }}`
       - `{{ $json.btc.big_move }}`
       - `{{ $json.eth.cross_up }}`
       - `{{ $json.eth.cross_down }}`
       - `{{ $json.eth.big_move }}`
   - Connect input from `Compute Threshold Flags`.
   - True branch connects to next node.
   - False branch ends workflow.

5. **Add Code node to format alert message:**
   - Name: `Format Alert Email`
   - Type: Code (JavaScript)
   - Parameters (JS code):
     ```javascript
     const s=(n,d=2)=>isFinite(n)?Number(n).toFixed(d):'—';
     const sign=n=>isFinite(n)&&n>0?'+':'';
     const j=$json; const lines=[
       `BTC: $${s(j.btc.price,0)} (24h: ${sign(j.btc.chg24)}${s(j.btc.chg24)}%)`,
       `ETH: $${s(j.eth.price,0)} (24h: ${sign(j.eth.chg24)}${s(j.eth.chg24)}%)`
     ];
     return [{json:{
       subject:`⚠️ Crypto Alert — ${new Date().toLocaleString('en-US',{timeZone:'Africa/Lagos'})}`,
       html:`<p>${lines.join('<br>')}</p>`,
       text: lines.join('\n')
     }}];
     ```
   - Connect input from `Any Alert?` (true branch).

6. **Add Gmail node to send email:**
   - Name: `Send a message`
   - Type: Gmail
   - Parameters:
     - Send To: Enter recipient email (e.g., `recipient@gmail.com`)
     - Subject: `={{ $json.subject }}`
     - Message: `={{ $json.html }}`
     - Options: Enable “Append Attribution”
   - Credentials: Configure Gmail OAuth2 credentials beforehand.
   - Connect input from `Format Alert Email`.

7. **Add Twilio node to send SMS/WhatsApp:**
   - Name: `Send an SMS/MMS/WhatsApp message`
   - Type: Twilio
   - Parameters:
     - Message: `={{ $json.text }}`
   - Credentials: Configure Twilio credentials with SID and Auth Token.
   - Connect input from `Format Alert Email`.

8. **Add Sticky Note for documentation (optional):**
   - Name: `Sticky Note`
   - Type: Sticky Note
   - Parameters: Paste detailed workflow description, usage notes, and contact info as in the original workflow.
   - Position anywhere convenient; no input/output connections needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                  | Context or Link                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Pro tip: Debounce alerts with a small memory store or external sheet to prevent repeated alerts when price hovers near thresholds.                                                                                                                                                                                             | Sticky Note content                                |
| Author: David Olusola. For training and 1:1 consulting, contact sales@daexai.com.                                                                                                                                                                                                                                               | Sticky Note content, mailto link                    |
| Workflow triggers every 10–15 minutes; adjust schedule node as needed for desired frequency.                                                                                                                                                                                                                                  | Sticky Note content                                |
| Typical alert output example: “BTC broke $110,000 (+2.1% 24h)”                                                                                                                                                                                                                                                                 | Sticky Note content                                |
| Add or swap alert channels easily: Email, Twilio SMS, Telegram, Slack.                                                                                                                                                                                                                                                        | Sticky Note content                                |
| CoinGecko API used is public and requires no authentication; watch for rate limits or downtime.                                                                                                                                                                                                                                | Node “Get Crypto Prices” details                    |
| Gmail node uses OAuth2 credentials; ensure tokens are valid and refreshed.                                                                                                                                                                                                                                                     | Node “Send a message” details                       |
| Twilio node requires valid account SID and Auth Token; ensure sufficient balance and correct phone number formats.                                                                                                                                                                                                           | Node “Send an SMS/MMS/WhatsApp message” details    |

---

**Disclaimer:** The text provided derives exclusively from an n8n automated workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.