Daily Stock Market Report with Twelve Data API Sent via WhatsApp and Email

https://n8nworkflows.xyz/workflows/daily-stock-market-report-with-twelve-data-api-sent-via-whatsapp-and-email-7726


# Daily Stock Market Report with Twelve Data API Sent via WhatsApp and Email

### 1. Workflow Overview

This workflow automates the generation and distribution of a daily stock market report highlighting the top stock price changes among selected symbols. It is designed for users who want a concise summary of stock market performance delivered promptly after market close via WhatsApp and email. The workflow includes the following logical blocks:

- **1.1 Scheduled Trigger:** Starts the workflow daily at market close time.
- **1.2 Configuration Setup:** Defines API keys, stock symbols, and recipient contact information.
- **1.3 Data Retrieval:** Fetches daily stock data from the Twelve Data API for the configured symbols.
- **1.4 Data Processing:** Parses and analyzes the fetched stock data to determine top gainers and losers.
- **1.5 Message Formatting:** Prepares formatted messages for WhatsApp and email distribution.
- **1.6 Message Delivery:** Sends the formatted stock report via WhatsApp and email to configured recipients.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow automatically every weekday at 5:00 PM, shortly after the stock market closes, ensuring timely reporting.

- **Nodes Involved:**  
  - Daily Market Close Trigger

- **Node Details:**

  - **Daily Market Close Trigger**  
    - Type: Cron Trigger  
    - Configuration: Triggers daily at 17:00 (5:00 PM) server time, Monday through Friday.  
    - Input: None (trigger node)  
    - Output: Triggers subsequent nodes in workflow.  
    - Edge Cases: Workflow will not trigger on weekends or holidays; time zone differences may affect trigger accuracy if server time is not aligned with market close time.

---

#### 2.2 Configuration Setup

- **Overview:**  
  Sets all configurable variables such as the Twelve Data API key, stock symbols to track, WhatsApp recipient number, and email recipient address.

- **Nodes Involved:**  
  - Set Configuration Variables

- **Node Details:**

  - **Set Configuration Variables**  
    - Type: Set Node  
    - Configuration: Assigns string values:
      - `twelvedata_api_key`: Placeholder for Twelve Data API key.
      - `top_symbols`: Comma-separated list of stock symbols (e.g., AAPL, MSFT, GOOGL, AMZN, TSLA).
      - `whatsapp_number`: International format phone number for WhatsApp alerts.
      - `email_recipient`: Email address for receiving the stock report.  
    - Input: Trigger output  
    - Output: JSON object with configured variables for downstream HTTP requests and messaging.  
    - Edge Cases: Missing or invalid API key or contact info will cause failures downstream.

---

#### 2.3 Data Retrieval

- **Overview:**  
  Calls the Twelve Data API to retrieve the latest daily stock quote data for the specified symbols.

- **Nodes Involved:**  
  - Fetch Stock Data from Twelve Data

- **Node Details:**

  - **Fetch Stock Data from Twelve Data**  
    - Type: HTTP Request  
    - Configuration:
      - URL constructed dynamically using expression to include symbols and API key.
      - GET method with query parameters `interval=1day`, `outputsize=1`.
      - Timeout set to 30 seconds.  
    - Input: Configuration variables from Set Configuration Variables node.  
    - Output: JSON response from Twelve Data API containing quote data for requested symbols.  
    - Edge Cases:
      - API rate limits or invalid API key may cause errors.
      - Network timeouts or API downtime.
      - Unexpected or malformed JSON responses.
    - Version-specific: Uses version 4.2 of the HTTP Request node for advanced query parameter handling.

---

#### 2.4 Data Processing

- **Overview:**  
  Processes the stock data response to extract relevant metrics, filter top movers, and prepare structured data for reporting.

- **Nodes Involved:**  
  - Process Stock Movements

- **Node Details:**

  - **Process Stock Movements**  
    - Type: Code Node (JavaScript)  
    - Configuration:
      - Parses input JSON, checks for API errors and handles single/multiple stock responses.
      - Extracts stock symbol, name, price, change, percent change, and volume.
      - Calculates absolute percent change and classifies movement as gain or loss.
      - Sorts stocks by absolute percent change.
      - Selects top 5 movers, then top 3 gainers and top 3 losers from that list.
      - Adds formatted current date.  
    - Input: JSON stock data from HTTP Request node.  
    - Output: JSON with date, total stocks analyzed, arrays of top gainers, losers, and biggest movers.  
    - Edge Cases:
      - API error messages cause workflow failure with descriptive error.
      - Missing or non-numeric data fields handled gracefully by setting null.
      - Empty or invalid data arrays handled by filtering and sorting logic.

---

#### 2.5 Message Formatting

- **Overview:**  
  Converts processed stock data into human-readable message formats optimized for WhatsApp text and HTML email.

- **Nodes Involved:**  
  - Format WhatsApp Message  
  - Format Email Content

- **Node Details:**

  - **Format WhatsApp Message**  
    - Type: Code Node (JavaScript)  
    - Configuration:
      - Builds multiline text message with Unicode emoji symbols and markdown styling.
      - Includes report date, total stocks analyzed.
      - Lists top gainers and losers with symbol, price, percent change, and absolute change.
      - Adds footer branding line.  
    - Input: Processed stock data node output.  
    - Output: JSON containing `whatsapp_message` string for sending.  
    - Edge Cases: Handles missing stock price or change data by showing 'N/A'.

  - **Format Email Content**  
    - Type: Code Node (JavaScript)  
    - Configuration:
      - Creates full HTML email with embedded CSS styling for layout and color coding gains/losses.
      - Sections for top gainers and losers with tabular data.
      - Also creates a plain text fallback message.
      - Subject line includes the report date.  
    - Input: Processed stock data node output.  
    - Output: JSON with `email_html`, `email_text`, and `email_subject` fields.  
    - Edge Cases: Gracefully handles missing numeric data by showing 'N/A'.

---

#### 2.6 Message Delivery

- **Overview:**  
  Sends the formatted stock market report messages to the configured WhatsApp number and email address.

- **Nodes Involved:**  
  - Send message (WhatsApp)  
  - Send Email Alert

- **Node Details:**

  - **Send message (WhatsApp)**  
    - Type: WhatsApp Node  
    - Configuration:
      - Sends text message using WhatsApp API.
      - Uses phone number ID (configured via expression) for sender.
      - Recipient phone number pulled from configuration variables.
      - Message body from formatted WhatsApp message node.  
    - Input: Formatted WhatsApp message node output.  
    - Credentials: WhatsApp API credentials required.  
    - Edge Cases:
      - Invalid recipient number or phone number ID will cause send failure.
      - WhatsApp API rate limits or connectivity issues.

  - **Send Email Alert**  
    - Type: Email Send Node  
    - Configuration:
      - Sends email with HTML content and subject from formatting node.
      - Recipient email from configuration variables.
      - From address is fixed as alert@gmail.com.
      - Uses SMTP credentials stored in n8n.  
    - Input: Formatted email content node output.  
    - Credentials: SMTP credentials required.  
    - Edge Cases:
      - Invalid recipient email or SMTP authentication failure.
      - Email server timeouts or rejections.

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                     | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                |
|-------------------------------|--------------------|-----------------------------------|--------------------------------|--------------------------------|----------------------------------------------------------------------------|
| Daily Market Close Trigger     | Cron Trigger       | Starts workflow daily at 5:00 PM  | None                           | Set Configuration Variables     | Triggers daily at 5:00 PM (after market close) Monday-Friday.              |
| Set Configuration Variables    | Set Node           | Defines API keys, symbols, recipients | Daily Market Close Trigger      | Fetch Stock Data from Twelve Data | Sets API key, stock symbols, and alert recipients.                         |
| Fetch Stock Data from Twelve Data | HTTP Request      | Fetches stock data from API       | Set Configuration Variables     | Process Stock Movements          | Fetches stock data for selected symbols via Twelve Data API.               |
| Process Stock Movements        | Code Node          | Processes and ranks stock data    | Fetch Stock Data from Twelve Data | Format WhatsApp Message, Format Email Content | Processes stock data to identify top gainers and losers.                   |
| Format WhatsApp Message        | Code Node          | Formats message for WhatsApp      | Process Stock Movements          | Send message                    | Sends formatted alerts via WhatsApp and Email.                            |
| Format Email Content           | Code Node          | Formats HTML and text email       | Process Stock Movements          | Send Email Alert                | Sends formatted alerts via WhatsApp and Email.                            |
| Send message                  | WhatsApp           | Sends WhatsApp stock report       | Format WhatsApp Message          | None                           | Sends formatted alerts via WhatsApp and Email.                            |
| Send Email Alert              | Email Send         | Sends email stock report          | Format Email Content             | None                           | Sends formatted alerts via WhatsApp and Email.                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node:**  
   - Name: `Daily Market Close Trigger`  
   - Type: Cron Trigger  
   - Set to trigger daily at 17:00 (5:00 PM).  
   - No inputs.

2. **Create Set Node:**  
   - Name: `Set Configuration Variables`  
   - Assign variables as strings:  
     - `twelvedata_api_key`: Enter your Twelve Data API key.  
     - `top_symbols`: Enter stock symbols (e.g., "AAPL,MSFT,GOOGL,AMZN,TSLA").  
     - `whatsapp_number`: Enter recipient WhatsApp number with country code (e.g., "+919999992211").  
     - `email_recipient`: Enter recipient email address (e.g., "abc@gmail.com").  
   - Connect: Output of `Daily Market Close Trigger` to this node.

3. **Create HTTP Request Node:**  
   - Name: `Fetch Stock Data from Twelve Data`  
   - Method: GET  
   - URL: Use expression:  
     `https://api.twelvedata.com/quote?symbol={{ $json.top_symbols }}&apikey={{ $json.twelvedata_api_key }}`  
   - Query Parameters:  
     - `interval`: 1day  
     - `outputsize`: 1  
   - Timeout: 30000 ms (30 seconds)  
   - Connect: Output of `Set Configuration Variables` to this node.

4. **Create Code Node:**  
   - Name: `Process Stock Movements`  
   - Paste provided JavaScript code that parses the API response, extracts and sorts stock data, selects top movers, and formats a structured JSON object.  
   - Connect: Output of `Fetch Stock Data from Twelve Data` to this node.

5. **Create Code Node for WhatsApp Formatting:**  
   - Name: `Format WhatsApp Message`  
   - Paste provided JavaScript code that formats the stock data into a multiline WhatsApp message with emojis and markdown style.  
   - Connect: Output of `Process Stock Movements` to this node.

6. **Create Code Node for Email Formatting:**  
   - Name: `Format Email Content`  
   - Paste provided JavaScript code that generates styled HTML email content, plain text fallback, and subject line with date.  
   - Connect: Output of `Process Stock Movements` to this node.

7. **Create WhatsApp Node:**  
   - Name: `Send message`  
   - Operation: Send  
   - `Text Body`: Use expression `{{$json.whatsapp_message}}` from `Format WhatsApp Message`.  
   - `Recipient Phone Number`: Use expression `{{$('Set Configuration Variables').item.json.whatsapp_number}}`.  
   - `Phone Number ID`: Set your WhatsApp sender phone number ID (e.g., `+919877663344`).  
   - Credentials: Configure WhatsApp API credentials in n8n.  
   - Connect: Output of `Format WhatsApp Message` to this node.

8. **Create Email Send Node:**  
   - Name: `Send Email Alert`  
   - To Email: Use expression `{{$('Set Configuration Variables').item.json.email_recipient}}`.  
   - From Email: Set a fixed sender email (e.g., `alert@gmail.com`).  
   - Subject: Use expression `{{$json.email_subject}}` from the email formatting node.  
   - HTML: Use expression `{{$json.email_html}}`.  
   - Email Format: HTML  
   - Credentials: Configure SMTP credentials in n8n.  
   - Connect: Output of `Format Email Content` to this node.

9. **Save and Activate Workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                   |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------|
| The workflow triggers daily at 5:00 PM, which is after US stock market close, Monday through Friday. | Sticky note on Cron Trigger node.                 |
| API key for Twelve Data must be obtained and inserted in the configuration node for successful data retrieval. | Twelve Data API documentation: https://twelvedata.com/docs |
| WhatsApp API requires setup with phone number IDs and credentials to send messages programmatically. | WhatsApp Business API documentation.              |
| SMTP credentials must be properly configured for email sending; test credentials before activation. | Example SMTP providers: Gmail SMTP, SendGrid.    |
| The workflow handles cases where stock data may be partially missing by displaying 'N/A' in outputs. | Code node error handling and formatting logic.    |
| For timezone consistency, verify n8n server timezone matches market timezone or adjust trigger accordingly. | N8n timezone settings documentation.              |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to relevant content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.