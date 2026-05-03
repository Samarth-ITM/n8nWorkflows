Monitor Bank Transactions with Multi-Channel Alerts for Accounting Teams

https://n8nworkflows.xyz/workflows/monitor-bank-transactions-with-multi-channel-alerts-for-accounting-teams-10110


# Monitor Bank Transactions with Multi-Channel Alerts for Accounting Teams

### 1. Workflow Overview

This workflow monitors bank transactions in real-time and generates multi-channel alerts targeted at accounting and finance teams to aid in fraud detection and risk management. It periodically fetches recent transactions from a banking API, enriches them with risk-calculation logic, categorizes alerts by severity, logs alerts to Google Sheets, and sends notifications via email and Slack. The workflow also handles API errors gracefully and compiles daily summary statistics.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger & Data Retrieval:** Periodic fetching of recent bank transactions via API.
- **1.2 Error Handling:** Detection and alerting on API call failures.
- **1.3 Data Enrichment & Risk Scoring:** Transform raw transaction data to compute adjusted risk scores and categorize alerts.
- **1.4 Alert Categorization & Logging:** Branch transactions into critical, high, or medium priority, logging each to Google Sheets.
- **1.5 Multi-Channel Notification:** Send emails and Slack messages for critical and high priority alerts.
- **1.6 Summary Generation & Logging:** Aggregate alert statistics and log daily summaries to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger & Data Retrieval

- **Overview:**  
  This block initiates the workflow every 5 minutes, fetching the most recent bank transactions via an HTTP request to the bank’s API.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Fetch Transactions

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow periodically every 5 minutes.  
    - Configuration: Interval set to 5 minutes.  
    - Input: None (trigger node).  
    - Output: Triggers the next node "Fetch Transactions".  
    - Edge Cases: If n8n scheduler is down, no trigger will occur; minimal failure risk.

  - **Fetch Transactions**  
    - Type: HTTP Request  
    - Role: Retrieves transactions created in the last 5 minutes, filtered by account ID and limited to 100 records.  
    - Configuration:  
      - URL: `https://api.bank.com/transactions`  
      - Query Parameters:  
        - `from_date`: ISO timestamp 5 minutes before current time (dynamic via expression).  
        - `account_id`: Fixed account identifier "ACC-891234".  
        - `limit`: 100 transactions maximum.  
      - Timeout: 30 seconds.  
      - Error Handling: Configured to continue on error, passing error info downstream.  
    - Input: Trigger from Schedule Trigger.  
    - Output: JSON array of bank transactions or error object.  
    - Edge Cases: API timeout, network errors, invalid or missing data, rate limiting.  
    - Failure Handling: Passes error info to "API Error?" node for detection.

---

#### 1.2 Error Handling

- **Overview:**  
  Checks if the API response contains errors, then processes and alerts the DevOps team if needed.

- **Nodes Involved:**  
  - API Error? (If node)  
  - Handle API Error (Code node)  
  - Send Error Alert (Email node)

- **Node Details:**

  - **API Error?**  
    - Type: If  
    - Role: Detects if the API response contains an error field.  
    - Configuration: Checks if `$json.error` exists and is non-empty.  
    - Input: Output from "Fetch Transactions".  
    - Output: Two branches: error present → "Handle API Error", else → "Enrich & Transform Data".  
    - Edge Cases: Missing error field, unexpected error formats.

  - **Handle API Error**  
    - Type: Code (JavaScript)  
    - Role: Formats error information for alerting.  
    - Configuration: Extracts error message and appends metadata such as timestamp, workflow name, and severity.  
    - Input: Error branch from "API Error?".  
    - Output: Structured error object for email.  
    - Edge Cases: Unexpected error data structure.

  - **Send Error Alert**  
    - Type: Email Send  
    - Role: Sends high severity alert email to DevOps and CTO about API failure.  
    - Configuration:  
      - From: alerts@company.com  
      - To: devops@company.com  
      - CC: cto@company.com  
      - Subject includes alert emoji and error indication.  
      - SMTP credentials configured (test SMTP).  
    - Input: Output from "Handle API Error".  
    - Output: None (final endpoint).  
    - Edge Cases: Email delivery failure, SMTP auth errors.

---

#### 1.3 Data Enrichment & Risk Scoring

- **Overview:**  
  Enriches each transaction with calculated fields such as adjusted risk score incorporating multiple heuristics (weekend, night time, international, payment method), categorizes risk level, and flags velocity anomalies.

- **Nodes Involved:**  
  - Enrich & Transform Data

- **Node Details:**

  - **Enrich & Transform Data**  
    - Type: Code (JavaScript)  
    - Role: Processes all incoming transactions for risk scoring and categorization.  
    - Configuration:  
      - Parses transaction timestamp to determine weekend and night time.  
      - Flags international transactions by country code.  
      - Adjusts base risk score with weighted increments for risk factors.  
      - Categorizes risk as CRITICAL, HIGH, MEDIUM, LOW_RISK based on thresholds.  
      - Detects velocity anomalies based on count of similar vendor transactions in last 24 hours.  
      - Adds metadata such as processed timestamp and alert priority number.  
    - Input: Transactions from "Fetch Transactions" (error-free branch).  
    - Output: Array of enriched transaction objects with new fields.  
    - Edge Cases: Missing or malformed transaction data, invalid timestamps, missing risk_score fields.

---

#### 1.4 Alert Categorization & Logging

- **Overview:**  
  Routes enriched transactions into conditional branches based on risk categories and appends relevant records into dedicated Google Sheets tabs representing alert severity.

- **Nodes Involved:**  
  - Critical Alert? (If)  
  - High Priority? (If)  
  - Medium Priority? (If)  
  - Log Critical to Sheet  
  - Log High Priority to Sheet  
  - Log Medium Priority to Sheet

- **Node Details:**

  - **Critical Alert?**  
    - Type: If  
    - Role: Tests if risk_category equals "CRITICAL".  
    - Output: True → "Log Critical to Sheet", False → continue to next condition.

  - **High Priority?**  
    - Type: If  
    - Role: Tests if risk_category equals "HIGH".  
    - Output: True → "Log High Priority to Sheet".

  - **Medium Priority?**  
    - Type: If  
    - Role: Tests if risk_category equals "MEDIUM".  
    - Output: True → "Log Medium Priority to Sheet".

  - **Log Critical to Sheet**  
    - Type: Google Sheets  
    - Role: Appends critical alert transaction data into "Critical_Alerts" sheet.  
    - Configuration:  
      - Columns mapped: Timestamp, Transaction_ID, Amount, Vendor, Adjusted_Risk_Score, Risk_Category, Country, Velocity_Anomaly, Status ("PENDING_REVIEW").  
      - Uses Google Service Account authentication.  
      - Document ID and sheet name fixed.  
    - Input: True branch from "Critical Alert?".  
    - Output: Next node "Send Critical Email".  
    - Edge Cases: Authentication errors, API limits, missing Google Sheets.

  - **Log High Priority to Sheet**  
    - Type: Google Sheets  
    - Role: Appends high priority alert data to "High_Priority_Alerts" sheet.  
    - Configuration similar to critical log but fewer columns and "NEEDS_REVIEW" status.  
    - Input: True branch from "High Priority?".  
    - Output: "Send High Priority Email".  
    - Edge Cases as above.

  - **Log Medium Priority to Sheet**  
    - Type: Google Sheets  
    - Role: Appends medium priority alerts to "Medium_Alerts" sheet with limited columns and "LOGGED" status.  
    - Input: True branch from "Medium Priority?".  
    - Output: None further downstream.  
    - Edge Cases as above.

---

#### 1.5 Multi-Channel Notification

- **Overview:**  
  Sends alert notifications via email and Slack for critical and high priority transactions, including formatted messages and appropriate recipients.

- **Nodes Involved:**  
  - Send Critical Email  
  - Send High Priority Email  
  - Send Critical Slack Alert  
  - Send High Priority Slack  
  - Merge All Alerts

- **Node Details:**

  - **Send Critical Email**  
    - Type: Email Send  
    - Role: Sends HTML email to CFO and CEO for critical alerts with transaction amount in subject.  
    - Configuration:  
      - From: critical-alerts@company.com  
      - To: cfo@company.com, ceo@company.com  
      - CC: compliance@company.com  
      - SMTP credentials (test).  
    - Input: Output from "Log Critical to Sheet".  
    - Output: "Send Critical Slack Alert".

  - **Send High Priority Email**  
    - Type: Email Send  
    - Role: Sends alert email to finance team for high priority alerts.  
    - Configuration:  
      - From: alerts@company.com  
      - To: finance@company.com  
      - CC: cfo@company.com  
      - SMTP credentials (test).  
    - Input: Output from "Log High Priority to Sheet".  
    - Output: "Send High Priority Slack".

  - **Send Critical Slack Alert**  
    - Type: Slack  
    - Role: Posts detailed critical alert message to a Slack channel with @channel mention for immediate review.  
    - Configuration:  
      - Channel ID fixed to a specific channel.  
      - Message includes transaction amount, vendor, risk score, transaction ID, country, and explicit call to action.  
      - Slack API credentials configured.  
    - Input: Output from "Send Critical Email".  
    - Output: Merges to "Merge All Alerts".

  - **Send High Priority Slack**  
    - Type: Slack  
    - Role: Posts high priority alert to Slack channel with review timeframe.  
    - Configuration similar to critical Slack alert but less urgent tone.  
    - Input: Output from "Send High Priority Email".  
    - Output: Merges to "Merge All Alerts".

  - **Merge All Alerts**  
    - Type: Merge  
    - Role: Combines alert flows from critical and high priority slack notifications into a single flow for summary generation.  
    - Input: From "Send Critical Slack Alert" and "Send High Priority Slack".  
    - Output: "Generate Summary Stats".

---

#### 1.6 Summary Generation & Logging

- **Overview:**  
  Aggregates all categorized transactions to produce daily summary statistics and logs them to a dedicated Google Sheet for reporting.

- **Nodes Involved:**  
  - Generate Summary Stats (Code node)  
  - Log Summary to Sheet (Google Sheets)

- **Node Details:**

  - **Generate Summary Stats**  
    - Type: Code (JavaScript)  
    - Role: Computes counts of critical, high, and medium alerts; total and average amounts and risk scores; counts international and velocity anomalies; flags if immediate action is required.  
    - Input: Combined alerts from "Merge All Alerts" and implicitly from medium alerts (though medium alerts do not proceed past logging).  
    - Output: Summary JSON object with all computed metrics and timestamp.  
    - Edge Cases: Empty input array, division by zero.

  - **Log Summary to Sheet**  
    - Type: Google Sheets  
    - Role: Appends the summary statistics to "Daily_Summary" tab in Google Sheets.  
    - Configuration:  
      - Columns mapped: Timestamp, total transactions, counts per category, total amount, average risk score, international count, velocity anomalies.  
      - Uses Google Service Account credentials.  
    - Input: Output from "Generate Summary Stats".  
    - Output: None (end node).  
    - Edge Cases: Google API errors, auth failures.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                          | Input Node(s)              | Output Node(s)              | Sticky Note                               |
|--------------------------|---------------------|----------------------------------------|----------------------------|-----------------------------|-------------------------------------------|
| Schedule Trigger         | Schedule Trigger    | Initiates workflow every 5 minutes      | -                          | Fetch Transactions          | **Schedule Trigger** - Runs every 5 minutes |
| Fetch Transactions       | HTTP Request       | Fetches recent transactions from API    | Schedule Trigger            | API Error?                  | **Fetch Transactions** - HTTP Request with retry logic |
| API Error?               | If                 | Detects API errors                      | Fetch Transactions          | Handle API Error, Enrich & Transform Data | **API Error?** - IF condition for error detection |
| Handle API Error         | Code                | Formats errors for alerts               | API Error?                  | Send Error Alert            | **Handle API Error** - Code node for error processing |
| Send Error Alert         | Email Send          | Sends error email to DevOps             | Handle API Error            | -                           | **Send Error Alert** - Email to DevOps team |
| Enrich & Transform Data  | Code                | Calculates risk scores and enrich data | API Error?                  | Critical Alert?, High Priority?, Medium Priority? | **Enrich & Transform Data** - Advanced risk calculation |
| Critical Alert?          | If                  | Checks for critical risk category       | Enrich & Transform Data     | Log Critical to Sheet       | **Critical Alert?**, **High Priority?**, **Medium Priority?** - IF conditions for priority levels |
| High Priority?           | If                  | Checks for high risk category            | Enrich & Transform Data     | Log High Priority to Sheet  | (see above)                              |
| Medium Priority?         | If                  | Checks for medium risk category          | Enrich & Transform Data     | Log Medium Priority to Sheet | (see above)                              |
| Log Critical to Sheet    | Google Sheets       | Logs critical alerts to sheet            | Critical Alert?             | Send Critical Email         | **Log Critical to Sheet**, **Log High Priority to Sheet**, **Log Medium Priority to Sheet** - Google Sheets append |
| Log High Priority to Sheet | Google Sheets     | Logs high priority alerts to sheet       | High Priority?              | Send High Priority Email    | (see above)                              |
| Log Medium Priority to Sheet | Google Sheets   | Logs medium priority alerts to sheet     | Medium Priority?            | -                           | (see above)                              |
| Send Critical Email      | Email Send          | Emails critical alerts to executives     | Log Critical to Sheet       | Send Critical Slack Alert   | **Send Critical Email**, **Send High Priority Email** - Email notifications |
| Send High Priority Email | Email Send          | Emails high priority alerts to finance   | Log High Priority to Sheet  | Send High Priority Slack    | (see above)                              |
| Send Critical Slack Alert | Slack              | Posts critical alerts to Slack channel   | Send Critical Email         | Merge All Alerts            | **Send Critical Slack Alert**, **Send High Priority Slack** - Slack notifications |
| Send High Priority Slack | Slack               | Posts high priority alerts to Slack      | Send High Priority Email    | Merge All Alerts            | (see above)                              |
| Merge All Alerts         | Merge               | Combines alert branches                  | Send Critical Slack Alert, Send High Priority Slack | Generate Summary Stats | **Merge All Alerts** - Combines all branches |
| Generate Summary Stats   | Code                | Computes daily alert summary statistics  | Merge All Alerts            | Log Summary to Sheet        | **Generate Summary Stats** - Code node for analytics |
| Log Summary to Sheet     | Google Sheets       | Logs summary statistics to sheet         | Generate Summary Stats      | -                           | **Log Summary to Sheet** - Summary statistics storage |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Node: Schedule Trigger  
   - Set interval to every 5 minutes.

2. **Create Fetch Transactions HTTP Request**  
   - Node: HTTP Request  
   - URL: `https://api.bank.com/transactions`  
   - Query Parameters:  
     - `from_date`: Expression `{{$now.minus({ minutes: 5 }).toISO()}}`  
     - `account_id`: `"ACC-891234"`  
     - `limit`: `"100"`  
   - Timeout: 30000ms (30 seconds)  
   - Error Handling: Continue on error enabled.

3. **Create API Error? If Node**  
   - Condition: Check if `$json.error` exists and is non-empty.

4. **Create Handle API Error Code Node**  
   - JavaScript code that extracts error message, appends timestamp, workflow name, severity.

5. **Create Send Error Alert Email Node**  
   - SMTP credentials configured (e.g., test SMTP).  
   - From: alerts@company.com  
   - To: devops@company.com  
   - CC: cto@company.com  
   - Subject: "🚨 Workflow Error - API Failure"  
   - Connect Handle API Error node output to this node.

6. **Create Enrich & Transform Data Code Node**  
   - JavaScript code to:  
     - Determine weekend, night time.  
     - Flag international transactions.  
     - Adjust risk score with weighted factors.  
     - Categorize risk level (CRITICAL, HIGH, MEDIUM, LOW_RISK).  
     - Detect velocity anomalies.  
     - Add processed timestamp and alert priority number.

7. **Create Critical Alert? If Node**  
   - Condition: `$json.risk_category` equals `"CRITICAL"`.

8. **Create High Priority? If Node**  
   - Condition: `$json.risk_category` equals `"HIGH"`.

9. **Create Medium Priority? If Node**  
   - Condition: `$json.risk_category` equals `"MEDIUM"`.

10. **Create Log Critical to Sheet Google Sheets Node**  
    - Authenticate with Google Service Account.  
    - Document ID: your Google Sheet ID.  
    - Sheet Name: "Critical_Alerts".  
    - Operation: Append.  
    - Map columns: Timestamp, Transaction_ID, Amount, Vendor, Adjusted_Risk_Score, Risk_Category, Country, Velocity_Anomaly, Status (set to "PENDING_REVIEW").

11. **Create Log High Priority to Sheet Google Sheets Node**  
    - Same as above but Sheet Name: "High_Priority_Alerts", Status: "NEEDS_REVIEW".

12. **Create Log Medium Priority to Sheet Google Sheets Node**  
    - Same as above but Sheet Name: "Medium_Alerts", Status: "LOGGED".

13. **Create Send Critical Email Node**  
    - SMTP credentials.  
    - From: critical-alerts@company.com  
    - To: cfo@company.com, ceo@company.com  
    - CC: compliance@company.com  
    - Subject: "🚨 CRITICAL: Transaction Alert - ${{ $json.amount }}"  
    - Connect output of "Log Critical to Sheet" to this node.

14. **Create Send High Priority Email Node**  
    - SMTP credentials.  
    - From: alerts@company.com  
    - To: finance@company.com  
    - CC: cfo@company.com  
    - Subject: "⚠️ High Priority Transaction Alert - ${{ $json.amount }}"  
    - Connect output of "Log High Priority to Sheet" to this node.

15. **Create Send Critical Slack Alert Node**  
    - Slack credentials configured.  
    - Channel ID: fixed channel ID.  
    - Message text with amount, vendor, risk score, transaction ID, country, @channel mention.  
    - Connect output of "Send Critical Email" to this node.

16. **Create Send High Priority Slack Alert Node**  
    - Slack credentials.  
    - Channel ID: same as critical.  
    - Message with amount, vendor, risk score, transaction ID, review timeframe.  
    - Connect output of "Send High Priority Email" to this node.

17. **Create Merge All Alerts Node**  
    - Mode: Combine.  
    - Connect outputs from both Slack alert nodes here.

18. **Create Generate Summary Stats Code Node**  
    - JavaScript code to count alerts by category, sum amounts, average risk score, count international and velocity anomalies, flag immediate action need.

19. **Create Log Summary to Sheet Google Sheets Node**  
    - Google Service Account authentication.  
    - Document ID: same Google Sheet.  
    - Sheet Name: "Daily_Summary".  
    - Append operation.  
    - Map summary fields: Timestamp, Total_Transactions, Critical_Count, High_Count, Medium_Count, Total_Amount, Avg_Risk_Score, International_Count, Velocity_Anomalies.

20. **Connect Nodes in Order:**  
    - Schedule Trigger → Fetch Transactions → API Error?  
    - API Error? error branch → Handle API Error → Send Error Alert  
    - API Error? success branch → Enrich & Transform Data → Critical Alert?, High Priority?, Medium Priority? branches → respective log nodes → email nodes → Slack nodes → Merge All Alerts → Generate Summary Stats → Log Summary to Sheet.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                               |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------|
| Use Google Service Account authentication for Google Sheets nodes for seamless and secure access. | Google Sheets API documentation               |
| SMTP credentials should be configured securely for email nodes (test SMTP used here).              | n8n Email Send node documentation              |
| Slack integration requires Slack API token with proper scopes (chat:write) and channel access.     | Slack API docs: https://api.slack.com/        |
| Risk scoring logic includes weekend, night time, international flags, and payment method weighting.| Customizable in the "Enrich & Transform Data" node. |
| Velocity anomaly detection based on vendor transaction count in last 24h flags high-risk activity. | Adjust threshold as per business needs.       |
| The workflow handles API error gracefully by notifying DevOps and continuing without stopping.     | Good practice for production reliability.     |
| Summary statistics support management reporting and operational decision-making.                    | Stored in "Daily_Summary" Google Sheet tab.   |
| Slack alert messages use @channel to ensure immediate visibility for critical alerts.               | Use with caution to avoid alert fatigue.       |

---

This documentation fully describes the workflow "Monitor Bank Transactions with Multi-Channel Alerts for Accounting Teams" enabling understanding, modification, reproduction, and troubleshooting.