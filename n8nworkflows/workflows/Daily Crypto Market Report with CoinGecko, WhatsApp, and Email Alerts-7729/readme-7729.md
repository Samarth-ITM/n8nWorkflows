Daily Crypto Market Report with CoinGecko, WhatsApp, and Email Alerts

https://n8nworkflows.xyz/workflows/daily-crypto-market-report-with-coingecko--whatsapp--and-email-alerts-7729


# Daily Crypto Market Report with CoinGecko, WhatsApp, and Email Alerts

### 1. Workflow Overview

This workflow automates the generation and distribution of a daily cryptocurrency market report highlighting the top gainers and losers among the top 100 cryptocurrencies by market capitalization. Its primary use case is to provide automated alerts via WhatsApp and email, delivering timely and formatted market insights for traders, analysts, or crypto enthusiasts.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger and Configuration Setup:** Initiates the workflow daily and sets key configuration variables such as currency, WhatsApp number, and email recipient.
- **1.2 Data Retrieval:** Fetches real-time crypto market data from the CoinGecko API for the top 100 cryptocurrencies.
- **1.3 Data Processing:** Processes and ranks cryptocurrencies by their 24-hour price movements, categorizing them into gainers and losers.
- **1.4 Message Formatting:** Creates formatted messages tailored for WhatsApp and email, including both plain text and rich HTML content.
- **1.5 Alert Distribution:** Sends the generated report via WhatsApp and email to configured recipients.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Configuration Setup

**Overview:**  
This block triggers the workflow daily at midnight UTC and sets the essential configuration parameters required for subsequent API calls and message delivery.

**Nodes Involved:**  
- Daily Crypto Trigger  
- Set Configuration Variables  

**Node Details:**

- **Daily Crypto Trigger**  
  - *Type & Role:* Cron node; schedules the workflow execution.  
  - *Configuration:* Set to trigger daily at 00:00 UTC (hour 0).  
  - *Key Expressions:* None; static schedule.  
  - *Connections:* Outputs to "Set Configuration Variables".  
  - *Edge Cases:* Cron misconfiguration could cause missed or multiple triggers. Timezone assumptions must be noted (UTC).  
  - *Sticky Note:* "Triggers daily at 00:00 UTC."

- **Set Configuration Variables**  
  - *Type & Role:* Set node; defines static variables for use downstream.  
  - *Configuration:*  
    - `vs_currency = "usd"` (currency for API queries)  
    - `whatsapp_number = "+919988665533"` (recipient number for WhatsApp message)  
    - `email_recipient = "abc@gmail.com"` (email alert recipient)  
  - *Key Expressions:* Static string assignments.  
  - *Connections:* Outputs to "Fetch Crypto Data from CoinGecko".  
  - *Edge Cases:* Incorrect phone number or email format may cause delivery failures later.  
  - *Sticky Note:* "Configure phone numbers, and email addresses here"

---

#### 2.2 Data Retrieval

**Overview:**  
Fetches the latest 24-hour market data of the top 100 cryptocurrencies by market capitalization using CoinGecko’s public API.

**Nodes Involved:**  
- Fetch Crypto Data from CoinGecko  

**Node Details:**

- **Fetch Crypto Data from CoinGecko**  
  - *Type & Role:* HTTP Request node; fetches market data JSON.  
  - *Configuration:*  
    - URL built dynamically using `vs_currency` from Set Configuration Variables:  
      `https://api.coingecko.com/api/v3/coins/markets?vs_currency={{ $json.vs_currency }}&order=market_cap_desc&per_page=100&page=1&sparkline=false&locale=en&price_change_percentage=24h`  
    - Timeout set to 30 seconds.  
  - *Key Expressions:* URL uses mustache template referencing previous node's JSON.  
  - *Connections:* Outputs to "Process Crypto Movements".  
  - *Edge Cases:*  
    - API rate limiting or downtime can cause errors or empty responses.  
    - Timeout or network errors possible.  
    - Response format changes by CoinGecko could break parsing.  
  - *Sticky Note:* "Fetches 24h data for top 100 cryptos using CoinGecko API"

---

#### 2.3 Data Processing

**Overview:**  
Processes the raw API data to calculate and categorize cryptocurrencies based on their 24-hour price movement percentage, preparing datasets for top gainers, losers, and biggest movers.

**Nodes Involved:**  
- Process Crypto Movements  

**Node Details:**

- **Process Crypto Movements**  
  - *Type & Role:* Code node (JavaScript); transforms and ranks the data.  
  - *Configuration:*  
    - Iterates all input items; parses key fields: name, symbol, current price, price change, percent change, volume, high and low prices.  
    - Calculates absolute percent change and classifies each coin as 'gain' or 'loss'.  
    - Sorts all coins by the absolute percent change descending.  
    - Extracts top 100 movers, gainers, and losers.  
    - Sets a fixed current date/time string in IST timezone (02:33 PM IST, August 22, 2025) for reporting purposes (note: hardcoded date).  
  - *Key Expressions:* JavaScript code with array operations and sorting.  
  - *Connections:* Outputs to "Format WhatsApp Message" and "Format Email Content".  
  - *Edge Cases:*  
    - Missing or malformed data fields in API response.  
    - NaN or null prices/changes handled with fallbacks.  
    - Hardcoded date/time may cause confusion or require manual update.  
  - *Sticky Note:* "Processes and ranks cryptos by biggest 24h movements"

---

#### 2.4 Message Formatting

**Overview:**  
Generates user-friendly formatted content for both WhatsApp and email communication channels based on processed crypto data.

**Nodes Involved:**  
- Format WhatsApp Message  
- Format Email Content  

**Node Details:**

- **Format WhatsApp Message**  
  - *Type & Role:* Code node (JavaScript); formats plain-text message for WhatsApp.  
  - *Configuration:*  
    - Constructs a multi-line string including date, total analyzed coins, top gainers, and top losers with price and percentage changes.  
    - Uses emoji and markdown-style formatting for readability on WhatsApp.  
  - *Key Expressions:* Template literals with array iteration over gainers and losers.  
  - *Connections:* Outputs to "Send message" (WhatsApp send).  
  - *Edge Cases:* Large messages may exceed WhatsApp limits; null or missing values replaced with 'N/A'.  
  - *Sticky Note:* None.

- **Format Email Content**  
  - *Type & Role:* Code node (JavaScript); generates HTML and plain text for email alerts.  
  - *Configuration:*  
    - Builds a styled HTML email with tables listing top gainers and losers, including rank, symbol, price, change, percent change, and volume.  
    - Applies CSS styling for visual clarity and branding.  
    - Also prepares a plain-text summary version for email clients that do not support HTML.  
    - Email subject line dynamically includes report date.  
  - *Key Expressions:* Template literals with embedded loops for table rows.  
  - *Connections:* Outputs to "Send Email Alert".  
  - *Edge Cases:* Email clients may render HTML differently; large tables could increase email size.  
  - *Sticky Note:* None.

---

#### 2.5 Alert Distribution

**Overview:**  
Dispatches the formatted report via WhatsApp and email to configured recipients.

**Nodes Involved:**  
- Send message (WhatsApp)  
- Send Email Alert  

**Node Details:**

- **Send message**  
  - *Type & Role:* WhatsApp node; sends the formatted WhatsApp message.  
  - *Configuration:*  
    - Message body sourced from previous node's `whatsapp_message`.  
    - Recipient phone number dynamically from configuration variables.  
    - Phone number ID statically set to `+919876543234` (this is the WhatsApp Business API phone number ID).  
  - *Credentials:* Uses WhatsApp API credentials ("WhatsApp-test").  
  - *Connections:* Terminal node.  
  - *Edge Cases:*  
    - Invalid recipient number or phone number ID may cause failures.  
    - WhatsApp API quota or connectivity issues.  
  - *Sticky Note:* "Sends alerts via WhatsApp/Telegram and Email with formatted reports"

- **Send Email Alert**  
  - *Type & Role:* Email Send node; sends the HTML email alert.  
  - *Configuration:*  
    - "To" address dynamically set from configuration variables.  
    - "From" email statically set to `alert@gmail.com`.  
    - Subject and HTML content taken from previous node.  
    - Email format set to HTML.  
  - *Credentials:* SMTP credentials configured ("SMTP -test").  
  - *Connections:* Terminal node.  
  - *Edge Cases:*  
    - SMTP authentication failure or network errors.  
    - Invalid recipient email address.  
  - *Sticky Note:* "Sends alerts via WhatsApp/Telegram and Email with formatted reports"

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                            | Input Node(s)                | Output Node(s)                 | Sticky Note                                                        |
|----------------------------|--------------------|--------------------------------------------|-----------------------------|-------------------------------|-------------------------------------------------------------------|
| Daily Crypto Trigger        | Cron               | Daily workflow trigger                      |                             | Set Configuration Variables   | Triggers daily at 00:00 UTC.                                      |
| Set Configuration Variables | Set                | Define static variables for API and alerts | Daily Crypto Trigger         | Fetch Crypto Data from CoinGecko | Configure phone numbers, and email addresses here                |
| Fetch Crypto Data from CoinGecko | HTTP Request   | Fetch crypto market data from CoinGecko API| Set Configuration Variables  | Process Crypto Movements       | Fetches 24h data for top 100 cryptos using CoinGecko API         |
| Process Crypto Movements    | Code               | Process and rank cryptos by 24h movement   | Fetch Crypto Data from CoinGecko | Format WhatsApp Message, Format Email Content | Processes and ranks cryptos by biggest 24h movements             |
| Format WhatsApp Message     | Code               | Format WhatsApp message content             | Process Crypto Movements      | Send message                   |                                                                   |
| Format Email Content        | Code               | Generate HTML and plain-text email content | Process Crypto Movements      | Send Email Alert               |                                                                   |
| Send message               | WhatsApp           | Send WhatsApp alert                         | Format WhatsApp Message       |                               | Sends alerts via WhatsApp/Telegram and Email with formatted reports |
| Send Email Alert           | Email Send         | Send email alert                           | Format Email Content          |                               | Sends alerts via WhatsApp/Telegram and Email with formatted reports |
| Sticky Note                | Sticky Note        | Informational notes                         |                             |                               | Triggers daily at 00:00 UTC.                                      |
| Sticky Note1               | Sticky Note        | Informational notes                         |                             |                               | Configure phone numbers, and email addresses here                |
| Sticky Note2               | Sticky Note        | Informational notes                         |                             |                               | Fetches 24h data for top 100 cryptos using CoinGecko API         |
| Sticky Note3               | Sticky Note        | Informational notes                         |                             |                               | Processes and ranks cryptos by biggest 24h movements             |
| Sticky Note4               | Sticky Note        | Informational notes                         |                             |                               | Sends alerts via WhatsApp/Telegram and Email with formatted reports |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Node** named "Daily Crypto Trigger"  
   - Set to trigger daily at hour `0` (midnight UTC).

2. **Add a Set Node** named "Set Configuration Variables"  
   - Connect from "Daily Crypto Trigger".  
   - Set variables:  
     - `vs_currency` = `"usd"`  
     - `whatsapp_number` = `"+919988665533"` (replace with your target number)  
     - `email_recipient` = `"abc@gmail.com"` (replace with your target email)

3. **Add an HTTP Request Node** named "Fetch Crypto Data from CoinGecko"  
   - Connect from "Set Configuration Variables".  
   - Configure:  
     - Method: GET  
     - URL:  
       ```
       https://api.coingecko.com/api/v3/coins/markets?vs_currency={{ $json.vs_currency }}&order=market_cap_desc&per_page=100&page=1&sparkline=false&locale=en&price_change_percentage=24h
       ```  
     - Set timeout: 30000 ms (30 seconds)

4. **Add a Code Node** named "Process Crypto Movements"  
   - Connect from "Fetch Crypto Data from CoinGecko".  
   - Paste JavaScript code that:  
     - Iterates through the API response JSON, extracts and calculates price changes and percent changes.  
     - Sorts by absolute percent change.  
     - Selects top 100 gainers and losers.  
     - Sets a date string (update this date/time as needed or replace with dynamic date).  
   - Ensure the output JSON follows the structure with fields: `date`, `total_stocks`, `top_gainers`, `top_losers`, `biggest_movers`.

5. **Add a Code Node** named "Format WhatsApp Message"  
   - Connect from "Process Crypto Movements".  
   - Implement JavaScript code to build a text message with emojis and markdown for:  
     - Date  
     - Total cryptos analyzed  
     - Top gainers and losers with prices and percent changes.

6. **Add a WhatsApp Node** named "Send message"  
   - Connect from "Format WhatsApp Message".  
   - Configure:  
     - Text Body: `={{ $json.whatsapp_message }}`  
     - Recipient Phone Number: `={{ $('Set Configuration Variables').item.json.whatsapp_number }}`  
     - Phone Number ID: Set your WhatsApp Business API phone ID (replace `+919876543234`)  
   - Add WhatsApp API credentials.

7. **Add a Code Node** named "Format Email Content"  
   - Connect from "Process Crypto Movements".  
   - Implement JavaScript code to build:  
     - A styled HTML email with tables for top gainers and losers.  
     - Plain-text fallback summary.  
     - Dynamic email subject line including the report date.

8. **Add an Email Send Node** named "Send Email Alert"  
   - Connect from "Format Email Content".  
   - Configure:  
     - To Email: `={{ $('Set Configuration Variables').item.json.email_recipient }}`  
     - From Email: `alert@gmail.com` (update as appropriate)  
     - Subject: `={{ $json.email_subject }}`  
     - HTML: `={{ $json.email_html }}`  
     - Email Format: HTML  
   - Add SMTP credentials with valid access.

9. **Add Sticky Notes** as needed for documentation and clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                              |
|-----------------------------------------------------------------------------------------------|----------------------------------------------|
| The workflow uses CoinGecko's free public API; consider API rate limits and usage policies.    | https://www.coingecko.com/api/documentations/v3 |
| WhatsApp message size limits should be considered to avoid message truncation or failures.    | WhatsApp Business API documentation           |
| The date/time in the processing code is hardcoded to a fixed IST timestamp; update as needed or replace with dynamic date generation. | Internal workflow note                         |
| SMTP credentials must be valid and allowed to send emails to the target recipient.             | SMTP provider documentation                    |
| WhatsApp API requires a verified Business Account with phone number ID properly configured.   | https://developers.facebook.com/docs/whatsapp |
| Styling in the email content uses inline CSS for compatibility across major email clients.    | General email HTML styling best practices     |

---

This detailed documentation enables users and automation agents to fully understand, reproduce, and maintain the "Automated Daily Crypto Market Report - Top Gainers & Losers" workflow. It covers node configurations, data flow, potential failure points, and integration nuances.