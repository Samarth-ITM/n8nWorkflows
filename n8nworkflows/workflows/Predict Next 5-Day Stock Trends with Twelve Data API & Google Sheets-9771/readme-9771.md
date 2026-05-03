Predict Next 5-Day Stock Trends with Twelve Data API & Google Sheets

https://n8nworkflows.xyz/workflows/predict-next-5-day-stock-trends-with-twelve-data-api---google-sheets-9771


# Predict Next 5-Day Stock Trends with Twelve Data API & Google Sheets

---
### 1. Workflow Overview

This workflow automates the prediction of stock trends for the next 5 days by integrating data from the Twelve Data API and Google Sheets. It targets users who want to monitor stock performance daily after market close and receive summarized trend reports via email. The workflow logically divides into the following blocks:

- **1.1 Input Reception and Scheduling:** Triggers the workflow daily at market close time and reads stock ticker symbols from a Google Sheet.
- **1.2 Configuration Setup:** Sets essential variables such as API keys, stock symbols, and email recipients for subsequent nodes.
- **1.3 Data Retrieval:** Fetches the last 5 days of stock price data using the Twelve Data API.
- **1.4 Trend Analysis:** Processes fetched data to compute average prices, percentage changes, and predict the trend direction for the next 5 days.
- **1.5 Data Output and Notification:** Updates a Google Sheet with analysis results and formats a detailed HTML email report sent to users.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Scheduling

- **Overview:**  
  This block initiates the workflow automatically daily at 9:00 PM (post-market close) and retrieves the list of stock symbols from a Google Sheet.

- **Nodes Involved:**  
  - Daily Market Close Trigger  
  - Read Stock Symbols  

- **Node Details:**

  - **Daily Market Close Trigger**  
    - Type: Cron Trigger  
    - Configuration: Triggers at 21:00 (9 PM) daily.  
    - Inputs: None (start node).  
    - Outputs: Connects to "Read Stock Symbols".  
    - Edge Cases: Cron misconfiguration or time zone mismatches could cause missed triggers.  
    - Sticky Note: "Triggers daily at 9:00 PM (after market close) Monday-Friday."

  - **Read Stock Symbols**  
    - Type: Google Sheets Read  
    - Configuration: Reads a specific sheet (by sheet ID and document ID) containing stock symbols, authenticated via a Google Service Account.  
    - Inputs: Trigger from the Cron node.  
    - Outputs: Passes stock symbols to "Set Configuration Variables".  
    - Edge Cases: Authentication failure, sheet access errors, or empty symbol list.  
    - Sticky Note: "Reads stock symbols from Google Sheet."

#### 2.2 Configuration Setup

- **Overview:**  
  Sets API keys, stock symbols, and email recipient addresses into workflow variables for reuse downstream.

- **Nodes Involved:**  
  - Set Configuration Variables

- **Node Details:**

  - **Set Configuration Variables**  
    - Type: Set  
    - Configuration:  
      - `twelvedata_api_key`: Placeholder for the Twelve Data API key (to be replaced with actual key).  
      - `stock_symbols`: Dynamically assigned from the previous node's JSON symbol field.  
      - `email_recipient`: Placeholder for the email address to send reports to.  
    - Inputs: Receives stock symbols from "Read Stock Symbols".  
    - Outputs: Sends configured variables to "Fetch 5-Day Stock Data".  
    - Edge Cases: Missing API key or email address would cause failures in subsequent HTTP requests or email sending.  
    - Sticky Note: "Sets API key and stock symbols."

#### 2.3 Data Retrieval

- **Overview:**  
  Fetches the last 5 days of stock price data for each symbol from the Twelve Data API using HTTP requests.

- **Nodes Involved:**  
  - Fetch 5-Day Stock Data

- **Node Details:**

  - **Fetch 5-Day Stock Data**  
    - Type: HTTP Request  
    - Configuration:  
      - URL dynamically constructed with stock symbol, 1-day interval, output size 5, and API key from previous node.  
      - Timeout set to 30 seconds.  
    - Inputs: Receives configuration variables including API key and stock symbols.  
    - Outputs: Forwards JSON response to "Analyze Stock Trends".  
    - Edge Cases: Network issues, API rate limits, invalid API key, malformed URLs, or unexpected API error responses.  
    - Sticky Note: "Fetches 5-day stock data from Twelve Data API."

#### 2.4 Trend Analysis

- **Overview:**  
  Processes the 5-day stock data to calculate average prices, percentage changes, and predicts the trend direction for the coming days.

- **Nodes Involved:**  
  - Analyze Stock Trends

- **Node Details:**

  - **Analyze Stock Trends**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Validates API response, handles error messages.  
      - Calculates the 5-day trend slope based on closing prices.  
      - Computes average price and percentage change.  
      - Predicts trend direction ("Up", "Down", "Neutral") and magnitude of predicted change.  
      - Packages results including detailed daily values and metadata.  
      - Formats current date for reporting.  
    - Inputs: JSON stock data from HTTP request.  
    - Outputs: Transmits summarized analysis to "Update Google Sheet" and "Format Email Report".  
    - Edge Cases: API returning error JSON, missing or malformed data arrays, non-numeric price values, or empty datasets.  
    - Sticky Note: "Analyzes trends and predicts next 5-day movement."

#### 2.5 Data Output and Notification

- **Overview:**  
  Updates a Google Sheet with the analysis results and formats a comprehensive HTML email report sent to designated recipients.

- **Nodes Involved:**  
  - Update Google Sheet  
  - Format Email Report  
  - Send Email Report

- **Node Details:**

  - **Update Google Sheet**  
    - Type: Google Sheets Append  
    - Configuration:  
      - Appends analyzed data to a specified sheet by ID and document ID using service account authentication.  
      - Maps input JSON fields automatically.  
    - Inputs: Receives analysis JSON from "Analyze Stock Trends".  
    - Outputs: Forwards to "Format Email Report".  
    - Edge Cases: Sheet permission issues, incorrect mapping causing data loss, or API quota limits.  
    - Sticky Note: "Updates Google Sheet and sends email report."

  - **Format Email Report**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Generates styled HTML report including date, stock symbols, average prices, 5-day percentage changes, predicted trends, and predicted monetary changes.  
      - Also creates plain-text version for email clients.  
      - Sets email subject including report date.  
    - Inputs: Analysis data from "Update Google Sheet".  
    - Outputs: Passes formatted email content to "Send Email Report".  
    - Edge Cases: Missing data fields causing undefined content, HTML rendering issues in email clients.  
    - Sticky Note: "Updates Google Sheet and sends email report."

  - **Send Email Report**  
    - Type: Email Send  
    - Configuration:  
      - Sends email using SMTP credentials.  
      - Recipient and sender email addresses hardcoded to "abc@gmail.com" (should be parameterized).  
      - Sends HTML formatted email using generated content.  
    - Inputs: Email content and subject from "Format Email Report".  
    - Outputs: Final node, no downstream connections.  
    - Edge Cases: SMTP authentication failures, invalid email addresses, network problems.  
    - Sticky Note: "Updates Google Sheet and sends email report."

---

### 3. Summary Table

| Node Name                | Node Type         | Functional Role                                 | Input Node(s)             | Output Node(s)                      | Sticky Note                                                                  |
|--------------------------|-------------------|------------------------------------------------|---------------------------|-----------------------------------|-------------------------------------------------------------------------------|
| Daily Market Close Trigger | Cron Trigger      | Initiates workflow daily at 9 PM post-market   | None                      | Read Stock Symbols                 | Triggers daily at 9:00 PM (after market close) Monday-Friday.                  |
| Read Stock Symbols        | Google Sheets Read | Reads stock ticker symbols from Google Sheet   | Daily Market Close Trigger | Set Configuration Variables       | Reads stock symbols from Google Sheet.                                        |
| Set Configuration Variables | Set              | Sets API key, symbols, and email recipient     | Read Stock Symbols         | Fetch 5-Day Stock Data             | Sets API key and stock symbols.                                               |
| Fetch 5-Day Stock Data    | HTTP Request      | Fetches last 5-day stock data from Twelve Data API | Set Configuration Variables | Analyze Stock Trends              | Fetches 5-day stock data from Twelve Data API.                                |
| Analyze Stock Trends      | Code (JavaScript) | Processes data, calculates averages, predicts trends | Fetch 5-Day Stock Data     | Update Google Sheet, Format Email Report | Analyzes trends and predicts next 5-day movement.                             |
| Update Google Sheet       | Google Sheets Append | Appends analyzed data to Google Sheet          | Analyze Stock Trends       | Format Email Report               | Updates Google Sheet and sends email report.                                  |
| Format Email Report       | Code (JavaScript) | Generates HTML and plain-text email report     | Update Google Sheet        | Send Email Report                 | Updates Google Sheet and sends email report.                                  |
| Send Email Report         | Email Send        | Sends the email report with stock trend analysis | Format Email Report        | None                             | Updates Google Sheet and sends email report.                                  |
| Sticky Note              | Sticky Note       | Provides workflow notes and explanations       | None                      | None                             | See individual sticky notes' content above.                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node**  
   - Type: Cron Trigger  
   - Set to trigger daily at 21:00 (9 PM).  
   - No inputs, outputs connected to next node.

2. **Create Google Sheets Read Node**  
   - Type: Google Sheets  
   - Operation: Read Rows  
   - Configure credentials with Google Service Account (OAuth2 or service account key).  
   - Set Document ID and Sheet Name to the sheet containing stock symbols.  
   - Connect input from Cron Trigger node.

3. **Create Set Node for Configuration Variables**  
   - Type: Set  
   - Add variable `twelvedata_api_key` as string, placeholder `"YOUR_ACTUAL_API_KEY"`.  
   - Add variable `stock_symbols` assigned dynamically from incoming JSON field `symbol`.  
   - Add variable `email_recipient` as string, placeholder `"YOUR_ACTUAL_EMAIL_ADDRESS"`.  
   - Connect input from Google Sheets Read node.

4. **Create HTTP Request Node to Fetch Stock Data**  
   - Type: HTTP Request  
   - Method: GET  
   - URL Template: `https://api.twelvedata.com/time_series?symbol={{ $json.stock_symbols }}&interval=1day&outputsize=5&apikey={{ $json.twelvedata_api_key }}`  
   - Timeout: 30000 ms (30 seconds)  
   - Connect input from Set Configuration Variables node.

5. **Create Code Node to Analyze Stock Trends**  
   - Type: Code (JavaScript)  
   - Paste the provided JS logic to parse API response, compute averages, slopes, and predict trends.  
   - Output JSON including date, total stocks analyzed, and detailed stock analysis array.  
   - Connect input from HTTP Request node.

6. **Create Google Sheets Append Node to Update Analysis**  
   - Type: Google Sheets  
   - Operation: Append  
   - Configure with Google Service Account credentials.  
   - Set Document ID and Sheet Name where analysis results will be stored.  
   - Map input data automatically or explicitly as per analysis JSON structure.  
   - Connect input from Code Analysis node.

7. **Create Code Node to Format Email Report**  
   - Type: Code (JavaScript)  
   - Use provided JS code to generate HTML and plain-text email content including styling and summary tables.  
   - Set email subject dynamically with report date.  
   - Connect input from Google Sheets Append node.

8. **Create Email Send Node to Dispatch Report**  
   - Type: Email Send  
   - Configure SMTP credentials for sending email.  
   - Set recipient (`toEmail`) and sender (`fromEmail`) email addresses.  
   - Use `email_html`, `email_text`, and `email_subject` from previous node as inputs.  
   - Connect input from Format Email Report node.

9. **(Optional) Create Sticky Notes**  
   - Add sticky notes near each logical block to document purpose and usage as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                            |
|-----------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| The workflow requires replacing placeholders with your actual Twelve Data API key and valid Gmail addresses. | API key and email configuration must be set before running the workflow. |
| Twelve Data API Documentation: https://twelvedata.com/docs                                  | Reference for API parameters, limits, and errors.                         |
| Google Sheets API Setup: https://developers.google.com/sheets/api/quickstart/js               | Setup guide for service account credentials and Google Sheets access.     |
| SMTP Configuration: Ensure SMTP credentials (server, username, password) are valid and tested. | Required for email sending node functionality.                            |
| Date and time in the workflow assume US Eastern Time or adjust according to your market hours.| Important for scheduling triggers aligned with market close.              |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.