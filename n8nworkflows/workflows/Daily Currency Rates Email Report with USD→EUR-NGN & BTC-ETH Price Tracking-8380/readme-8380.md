Daily Currency Rates Email Report with USD→EUR/NGN & BTC/ETH Price Tracking

https://n8nworkflows.xyz/workflows/daily-currency-rates-email-report-with-usd-eur-ngn---btc-eth-price-tracking-8380


# Daily Currency Rates Email Report with USD→EUR/NGN & BTC/ETH Price Tracking

### 1. Workflow Overview

This workflow automates the delivery of a daily currency rates email report, scheduled to run every day at 8 AM. It targets users who want up-to-date exchange rates and cryptocurrency prices delivered in a professional, mobile-friendly email format. The main currencies tracked are USD→EUR, USD→NGN (Nigerian Naira), and the cryptocurrency prices of Bitcoin (BTC) and Ethereum (ETH) in USD, including their 24-hour percentage changes.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at 8 AM.
- **1.2 Data Retrieval:** Fetches fiat currency exchange rates and cryptocurrency prices from external APIs.
- **1.3 Data Merging and Formatting:** Combines the data inputs, formats them into a polished HTML and plain text email content using JavaScript.
- **1.4 Email Delivery:** Sends the formatted currency report via email using SMTP.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** This block starts the workflow automatically every day at 8:00 AM.
- **Nodes Involved:** 
  - Daily 8AM Trigger
  - Sticky Note (Currency Tracker Overview)
- **Node Details:**

  - **Daily 8AM Trigger**
    - Type: Schedule Trigger
    - Role: Initiates the workflow based on a scheduled time.
    - Configuration: Set to trigger daily at 8 AM (though the provided "interval" parameter is seconds-based, it should be configured explicitly in n8n for daily 8 AM).
    - Input Connections: None (start node).
    - Output Connections: Triggers "Get Fiat Exchange Rates" and "Get Crypto Prices" nodes.
    - Edge Cases: Misconfiguration may cause wrong trigger times; no retries if the system is down at trigger time.
    - Notes: Requires correct timezone setting in n8n instance to ensure 8 AM local time.

  - **Sticky Note (Currency Tracker Overview)**
    - Type: Sticky Note
    - Role: Provides a high-level description of the workflow purpose and currencies tracked.
    - Configuration: Static text describing the workflow and delivery format.
    - Input/Output: None.
    - Edge Cases: None.

#### 1.2 Data Retrieval

- **Overview:** This block obtains the latest currency exchange rates and cryptocurrency prices by calling public APIs.
- **Nodes Involved:**
  - Get Fiat Exchange Rates
  - Get Crypto Prices
  - Sticky Note1 (Fiat Currency Rates Info)
  - Sticky Note2 (Cryptocurrency Prices Info)
- **Node Details:**

  - **Get Fiat Exchange Rates**
    - Type: HTTP Request
    - Role: Fetches USD-based fiat currency exchange rates.
    - Configuration: Calls `https://api.exchangerate-api.com/v4/latest/USD` with GET method.
    - Variables: None dynamic; static URL.
    - Input Connections: Trigger from "Daily 8AM Trigger".
    - Output Connections: To "Merge" node.
    - Edge Cases: API downtime, rate limit, malformed response.
    - Notes: Uses free API with no signup; returns multiple currencies including EUR and NGN.

  - **Get Crypto Prices**
    - Type: HTTP Request
    - Role: Retrieves current Bitcoin and Ethereum prices in USD with 24h change.
    - Configuration: Calls `https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum&vs_currencies=usd&include_24hr_change=true` with GET method.
    - Input Connections: Trigger from "Daily 8AM Trigger".
    - Output Connections: To "Merge" node.
    - Edge Cases: API downtime, rate limit, missing data fields.
    - Notes: CoinGecko API requires no authentication.

  - **Sticky Note1 (Fiat Currency Rates Info)**
    - Type: Sticky Note
    - Role: Describes the fiat rates API and currencies fetched.
    - Configuration: Static text.
    - Input/Output: None.

  - **Sticky Note2 (Cryptocurrency Prices Info)**
    - Type: Sticky Note
    - Role: Describes the crypto prices API and data retrieved.
    - Configuration: Static text.
    - Input/Output: None.

#### 1.3 Data Merging and Formatting

- **Overview:** This block merges fiat and crypto data streams, processes the data to generate a professional HTML and plain text email body with formatted currency and crypto prices, including percentage changes and timestamps.
- **Nodes Involved:**
  - Merge
  - Format Email Content
  - Sticky Note3 (Email Formatting Details)
- **Node Details:**

  - **Merge**
    - Type: Merge
    - Role: Combines outputs of fiat and crypto HTTP requests into one data stream for processing.
    - Configuration: Default merge mode (likely "Wait" or "Combine").
    - Input Connections: From "Get Fiat Exchange Rates" and "Get Crypto Prices".
    - Output Connections: To "Format Email Content".
    - Edge Cases: One source missing or delayed; merge failure.
  
  - **Format Email Content**
    - Type: Code (JavaScript)
    - Role: Parses merged data, validates presence of both fiat and crypto data, formats HTML and text content for email.
    - Configuration:
      - Uses timezone Africa/Lagos for timestamps.
      - Extracts USD→EUR and USD→NGN rates.
      - Extracts BTC and ETH prices and 24h changes.
      - Formats numbers with locale-aware formatting.
      - Generates color-coded HTML with arrows for price changes.
      - Provides fallback plain text version.
      - Includes generation timestamps and last update date.
    - Variables:
      - `fiatData`, `cryptoData` extracted from merged inputs.
      - Formatting helper functions for numbers, percentages, colors, and dates.
    - Input Connections: From "Merge".
    - Output Connections: To "Send Daily Currency Email".
    - Edge Cases:
      - Missing fiat or crypto data throws errors.
      - Parsing failures if API response structure changes.
      - Timezone issues if server time differs.
    - Notes: Carefully handles order-independence of merged inputs.

  - **Sticky Note3 (Email Formatting Details)**
    - Type: Sticky Note
    - Role: Explains the logic and features of the email formatting code.
    - Configuration: Static descriptive text.
    - Input/Output: None.

#### 1.4 Email Delivery

- **Overview:** Sends the generated email content to the configured recipient using SMTP credentials.
- **Nodes Involved:**
  - Send Daily Currency Email
  - Sticky Note4 (Email Delivery Instructions)
- **Node Details:**

  - **Send Daily Currency Email**
    - Type: Email Send
    - Role: Delivers the formatted email to the recipient.
    - Configuration:
      - Subject and HTML body dynamically sourced from previous node output.
      - From and To email addresses set in parameters.
      - SMTP credentials configured (requires valid SMTP account).
      - Email format set to HTML.
      - Attribution disabled.
    - Input Connections: From "Format Email Content".
    - Output Connections: None (end node).
    - Edge Cases:
      - SMTP authentication errors.
      - Invalid email addresses.
      - Network issues causing send failures.
      - Recipient spam filtering.
    - Credentials: Must configure SMTP credentials with correct server, port, and authentication.

  - **Sticky Note4 (Email Delivery Instructions)**
    - Type: Sticky Note
    - Role: Provides setup guidance for the email sender and recipient configuration.
    - Configuration: Static text including SMTP setup tips for Gmail.
    - Input/Output: None.

---

### 3. Summary Table

| Node Name                  | Node Type             | Functional Role                   | Input Node(s)                    | Output Node(s)                 | Sticky Note                                                |
|----------------------------|-----------------------|---------------------------------|---------------------------------|-------------------------------|------------------------------------------------------------|
| Daily 8AM Trigger           | Schedule Trigger      | Starts workflow daily at 8 AM   | None                            | Get Fiat Exchange Rates, Get Crypto Prices | 💰 CURRENCY TRACKER WORKFLOW: Get daily exchange rates etc. |
| Sticky Note                | Sticky Note           | Workflow overview description   | None                            | None                          | 💰 CURRENCY TRACKER WORKFLOW: Get daily exchange rates etc. |
| Get Fiat Exchange Rates     | HTTP Request          | Fetches USD fiat exchange rates | Daily 8AM Trigger               | Merge                         | 🌍 FIAT CURRENCY RATES: Fetches USD rates from free API    |
| Sticky Note1               | Sticky Note           | Fiat currencies API description | None                            | None                          | 🌍 FIAT CURRENCY RATES: Fetches USD rates from free API    |
| Get Crypto Prices           | HTTP Request          | Fetches BTC and ETH prices      | Daily 8AM Trigger               | Merge                         | ₿ CRYPTOCURRENCY PRICES: Fetches Bitcoin and Ethereum etc. |
| Sticky Note2               | Sticky Note           | Crypto prices API description   | None                            | None                          | ₿ CRYPTOCURRENCY PRICES: Fetches Bitcoin and Ethereum etc. |
| Merge                      | Merge                 | Combines fiat and crypto data   | Get Fiat Exchange Rates, Get Crypto Prices | Format Email Content          |                                                            |
| Format Email Content        | Code                  | Formats email HTML and text     | Merge                          | Send Daily Currency Email     | 🎨 EMAIL FORMATTING: Professional design with gradients etc. |
| Sticky Note3               | Sticky Note           | Email formatting explanation    | None                            | None                          | 🎨 EMAIL FORMATTING: Professional design with gradients etc. |
| Send Daily Currency Email   | Email Send            | Sends the report via email      | Format Email Content            | None                          | 📨 EMAIL DELIVERY: Setup instructions for SMTP and emails  |
| Sticky Note4               | Sticky Note           | Email delivery setup instructions | None                            | None                          | 📨 EMAIL DELIVERY: Setup instructions for SMTP and emails  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node:**
   - Name: `Daily 8AM Trigger`
   - Type: Schedule Trigger
   - Configure to trigger daily at 08:00 AM in your local timezone.

2. **Create HTTP Request Node to Fetch Fiat Rates:**
   - Name: `Get Fiat Exchange Rates`
   - Type: HTTP Request
   - HTTP Method: GET
   - URL: `https://api.exchangerate-api.com/v4/latest/USD`
   - No authentication required.
   - Connect input from `Daily 8AM Trigger`.

3. **Create HTTP Request Node to Fetch Crypto Prices:**
   - Name: `Get Crypto Prices`
   - Type: HTTP Request
   - HTTP Method: GET
   - URL: `https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum&vs_currencies=usd&include_24hr_change=true`
   - No authentication required.
   - Connect input from `Daily 8AM Trigger`.

4. **Create Merge Node:**
   - Name: `Merge`
   - Type: Merge
   - Default mode (Wait for both inputs)
   - Connect input from `Get Fiat Exchange Rates` (input 2) and `Get Crypto Prices` (input 1).

5. **Create Code Node to Format Email Content:**
   - Name: `Format Email Content`
   - Type: Code (JavaScript)
   - Paste the provided JavaScript code that:
     - Extracts USD→EUR and USD→NGN rates.
     - Extracts BTC and ETH prices with 24h changes.
     - Generates HTML and plain text email content with styling and timestamps.
   - Connect input from `Merge`.

6. **Create Email Send Node:**
   - Name: `Send Daily Currency Email`
   - Type: Email Send
   - Set `From Email` to your sender email address.
   - Set `To Email` to recipient email address.
   - Set subject to `={{ $json.subject }}`
   - Set email content (HTML) to `={{ $json.html }}`
   - Disable "Append Attribution" option.
   - Configure SMTP credentials:
     - Server (e.g., smtp.gmail.com for Gmail)
     - Port (e.g., 587)
     - Username and App Password (for Gmail, use App Password, not regular password)
   - Connect input from `Format Email Content`.

7. **Add Sticky Notes (Optional but Recommended):**
   - Add descriptive sticky notes at appropriate points to document:
     - Workflow overview and currencies tracked.
     - API details for fiat and crypto rates.
     - Email formatting details.
     - Email delivery setup instructions.

8. **Test the Workflow:**
   - Execute the workflow manually or wait for the next scheduled run.
   - Verify email delivery and formatting.
   - Monitor logs for errors in API calls or email sending.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                           |
|--------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| The fiat currency data is sourced from ExchangeRate-API, a free API that requires no signup.                       | https://www.exchangerate-api.com                                          |
| Cryptocurrency prices and 24h changes are obtained from CoinGecko API, free and no API key required.                | https://www.coingecko.com/en/api                                         |
| SMTP setup for Gmail requires enabling "App Passwords" and using the SMTP server `smtp.gmail.com` on port 587.     | See Gmail docs for App Password setup                                    |
| Timezone is set to Africa/Lagos in the formatting code; adjust if sending to users in other regions.               | Modify `const TZ = 'Africa/Lagos'` in the code node                       |
| Email formatting includes professional styling with gradients, color-coded price changes, and responsive design.  | Inline CSS used for wide email client compatibility                       |
| The workflow assumes stable internet and API availability; consider adding error handling or retry logic if needed.| n8n supports retry and error workflow branches                            |

---

**Disclaimer:**  
The provided text is generated solely from an automated n8n workflow. It complies fully with content policies and does not contain any illegal, offensive, or protected material. All data handled is legal and publicly available.