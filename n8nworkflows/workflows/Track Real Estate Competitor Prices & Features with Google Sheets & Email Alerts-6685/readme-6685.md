Track Real Estate Competitor Prices & Features with Google Sheets & Email Alerts

https://n8nworkflows.xyz/workflows/track-real-estate-competitor-prices---features-with-google-sheets---email-alerts-6685


# Track Real Estate Competitor Prices & Features with Google Sheets & Email Alerts

### 1. Workflow Overview

This workflow automates the tracking of competitor real estate project prices and features by periodically fetching data from competitor APIs, parsing and logging the data into a Google Sheet, and sending email alerts when a significant price change is detected. It is designed for real estate professionals or analysts who want to monitor market changes and react promptly.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Data Retrieval:** Periodically triggers the workflow and fetches competitor data from different APIs based on schedule.
- **1.2 Data Processing & Logging:** Waits for data readiness, parses the raw data to extract relevant fields, and logs it into a Google Sheet.
- **1.3 Change Detection & Alerting:** Checks for price changes compared to previous data and sends alert emails if changes are detected.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Data Retrieval

**Overview:**  
This block triggers the workflow on a schedule using a Cron node, then fetches competitor data from one of two APIs depending on the schedule expression.

**Nodes Involved:**  
- Cron  
- Fetch Competitor Data  

**Node Details:**

- **Cron**  
  - Type: Cron Trigger  
  - Configuration: Default (no parameters specified), but conditions in downstream HTTP request depend on the cron expression value `'0 0 * * *'`.  
  - Role: Triggers the workflow on a preset schedule (e.g., daily at midnight).  
  - Inputs: None (trigger node)  
  - Outputs: Connected to "Fetch Competitor Data"  
  - Edge Cases: Misconfigured cron expressions may prevent triggering; no retries on trigger failure (n8n default).

- **Fetch Competitor Data**  
  - Type: HTTP Request (GET)  
  - Configuration:  
    - URL is dynamically chosen based on the cron expression from the "Cron" node:  
      - If cron expression is `'0 0 * * *'`, URL is `https://api.competitor1.com/projects`  
      - Otherwise, URL is `https://api.competitor2.com/projects`  
    - Timeout set to 10 seconds to avoid long hangs.  
  - Role: Fetches competitor project data from the respective API endpoint.  
  - Inputs: Triggered by "Cron"  
  - Outputs: Connected to "Wait For Data"  
  - Edge Cases:  
    - HTTP errors, timeouts, or API downtime can cause failure.  
    - The dynamic URL expression depends on the correctness of the cron expression string.  
    - No explicit error handling or retries configured here.

---

#### 2.2 Data Processing & Logging

**Overview:**  
After fetching, the workflow waits momentarily to ensure data availability, processes the JSON data to extract and normalize key fields, and appends the results to a Google Sheet.

**Nodes Involved:**  
- Wait For Data  
- Parse Data  
- Log to Google Sheets  

**Node Details:**

- **Wait For Data**  
  - Type: Wait Node  
  - Configuration: Default (no delay specified, but acts as a deliberate pause to ensure data readiness or sequencing)  
  - Webhook ID present, indicating it could be used as a webhook wait, but in this workflow it functions as a standard wait node.  
  - Inputs: From "Fetch Competitor Data"  
  - Outputs: To "Parse Data"  
  - Edge Cases:  
    - If configured with a webhook, external calls may be required; none shown here.  
    - No delay parameters set, so acts as a pass-through unless configured otherwise.

- **Parse Data**  
  - Type: Function  
  - Configuration: Custom JavaScript code that maps each item in the input to a simplified JSON object with keys:  
    - `projectName` from `name`  
    - `price`  
    - `features` (array, defaults to empty if missing)  
    - `location`  
    - `timestamp` (current ISO date string)  
  - Role: Normalizes raw API data for consistent downstream usage.  
  - Inputs: From "Wait For Data"  
  - Outputs: To "Log to Google Sheets"  
  - Edge Cases:  
    - Assumes input items have `name`, `price`, and `location` fields; missing fields may cause undefined values.  
    - `features` may be missing, handled by defaulting to empty array.  
    - Timestamp is generated at runtime, ensuring freshness.

- **Log to Google Sheets**  
  - Type: Google Sheets (Append Operation)  
  - Configuration:  
    - Appends data to range `CompetitorData!A:E` in a specified Google Sheet (sheet ID placeholder present).  
    - Uses Google API credentials named "Google Sheets- test".  
    - Writes the parsed fields including timestamp to the sheet.  
  - Inputs: From "Parse Data"  
  - Outputs: To "Check Price Change"  
  - Edge Cases:  
    - Possible errors from API quota limits, credential expiry, or sheet permission issues.  
    - The sheet ID must be correctly set; otherwise, operation fails.  
    - No data validation before appending.

---

#### 2.3 Change Detection & Alerting

**Overview:**  
This block compares the current price with the previous recorded price to detect changes. If a price change is detected, it sends an email alert summarizing the update.

**Nodes Involved:**  
- Check Price Change  
- Send Alert Email  

**Node Details:**

- **Check Price Change**  
  - Type: If (Conditional)  
  - Configuration:  
    - Checks if the current price (`Parse Data.json.price`) has changed compared to the previous price (`Parse Data.json.previousPrice`).  
    - If `previousPrice` is not available, it defaults to the current price, meaning no change detected.  
  - Role: Determines whether to proceed with alerting based on price difference.  
  - Inputs: From "Log to Google Sheets"  
  - Outputs:  
    - True branch: "Send Alert Email" (if price changed)  
    - False branch: No further action (implicit)  
  - Edge Cases:  
    - The `previousPrice` field must be available in the parsed data; if not, no alert is sent.  
    - If price fields are not numeric or malformed, expression could fail or misbehave.  
    - Does not handle minor price fluctuations or thresholds, only detects any change.

- **Send Alert Email**  
  - Type: Email Send (SMTP)  
  - Configuration:  
    - Sends a text email alerting about the price change with details: project name, location, old and new price, and features listed.  
    - Subject: "Price Change Alert"  
    - To: xyz@gmaiil.com (placeholder)  
    - From: abc@gmaiil.com (placeholder)  
    - Uses SMTP credentials named "SMTP -test".  
  - Role: Notifies stakeholders of relevant price changes by email.  
  - Inputs: From "Check Price Change" (True branch)  
  - Outputs: None (end of workflow)  
  - Edge Cases:  
    - Email sending can fail due to SMTP auth errors, network issues, or invalid recipient emails.  
    - Placeholders for email addresses must be replaced with real addresses.  
    - No retry or fallback notification channels configured.

---

### 3. Summary Table

| Node Name             | Node Type                | Functional Role                                  | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                 |
|-----------------------|--------------------------|-------------------------------------------------|-------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------|
| Cron                  | Cron Trigger             | Scheduled workflow trigger                       | —                       | Fetch Competitor Data    | ## Operational Process<br>- **Set Cron**: Triggers the workflow on a scheduled basis (e.g., hourly).         |
| Fetch Competitor Data  | HTTP Request             | Fetch competitor project data from API           | Cron                    | Wait For Data           | - **Fetch Competitor Data**: Performs GET requests to retrieve competitor pricing and feature data.          |
| Wait For Data         | Wait                     | Delay to ensure data readiness                    | Fetch Competitor Data    | Parse Data              | - **Wait For Data**: Introduces a delay to ensure data is fully retrieved.                                   |
| Parse Data            | Function                 | Normalize and extract relevant fields             | Wait For Data            | Log to Google Sheets    | - **Parse Data**: Processes and extracts relevant pricing and feature details.                               |
| Log to Google Sheets  | Google Sheets            | Append parsed data to Google Sheet                 | Parse Data               | Check Price Change      | - **Log to Google Sheets**: Appends the parsed data to a Google Sheet for tracking.                          |
| Check Price Change    | If                       | Detect if price changed compared to previous data | Log to Google Sheets     | Send Alert Email        | - **Check Price Change**: Evaluates if there’s a significant price change.                                   |
| Send Alert Email      | Email Send (SMTP)        | Send email notification on price change            | Check Price Change       | —                       | - **Send Alert Email**: Sends an email notification if a price change is detected.                           |
| Sticky Note           | Sticky Note              | Workflow documentation and customization notes    | —                       | —                       | See all above notes for detailed process and customization options.                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Node**  
   - Type: Cron Trigger  
   - Parameters: Set desired schedule (default example is daily at midnight `'0 0 * * *'`).  
   - No credentials required.

2. **Create HTTP Request Node ("Fetch Competitor Data")**  
   - Type: HTTP Request (GET)  
   - Parameters:  
     - URL: Use an expression to select between two URLs based on the cron expression:  
       ```
       {{$node['Cron'].parameter.cronExpression === '0 0 * * *' ? 'https://api.competitor1.com/projects' : 'https://api.competitor2.com/projects'}}
       ```  
     - Timeout: 10000 ms (10 seconds)  
   - Connect input from Cron node.  
   - No credentials needed unless APIs require authentication (not specified).

3. **Create Wait Node ("Wait For Data")**  
   - Type: Wait  
   - Parameters: Default (no delay set)  
   - Connect input from "Fetch Competitor Data".

4. **Create Function Node ("Parse Data")**  
   - Type: Function  
   - Parameters: Use the following JavaScript code:
     ```javascript
     return items.map(item => {
       const project = item.json;
       return {
         json: {
           projectName: project.name,
           price: project.price,
           features: project.features || [],
           location: project.location,
           timestamp: new Date().toISOString(),
         }
       };
     });
     ```  
   - Connect input from "Wait For Data".

5. **Create Google Sheets Node ("Log to Google Sheets")**  
   - Type: Google Sheets (Append)  
   - Parameters:  
     - Sheet ID: Set to your Google Sheet ID (replace placeholder).  
     - Range: `CompetitorData!A:E`  
     - Operation: Append  
   - Credentials: Configure Google API credentials with access to the target sheet.  
   - Connect input from "Parse Data".

6. **Create If Node ("Check Price Change")**  
   - Type: If  
   - Parameters:  
     - Condition: Number operation "changed"  
     - Value 1: `={{$node['Parse Data'].json['price']}}`  
     - Value 2: `={{$node['Parse Data'].json['previousPrice'] || $node['Parse Data'].json['price']}}`  
   - Connect input from "Log to Google Sheets".

7. **Create Email Send Node ("Send Alert Email")**  
   - Type: Email Send (SMTP)  
   - Parameters:  
     - To Email: Replace with actual email addresses (e.g., `xyz@gmail.com`)  
     - From Email: Replace with actual sender address (e.g., `abc@gmail.com`)  
     - Subject: `"Price Change Alert"`  
     - Text:  
       ```
       🏠 Price Change Alert: {{$node['Parse Data'].json['projectName']}} at {{$node['Parse Data'].json['location']}} changed from {{$node['Parse Data'].json['previousPrice'] || 'N/A'}} to {{$node['Parse Data'].json['price']}}. Features: {{$node['Parse Data'].json['features'].join(', ')}}
       ```  
   - Credentials: Set up SMTP credentials with valid email server details.  
   - Connect input from True output of "Check Price Change".

8. **Connect Nodes Appropriately**  
   - Cron → Fetch Competitor Data → Wait For Data → Parse Data → Log to Google Sheets → Check Price Change  
   - Check Price Change (True) → Send Alert Email  
   - Check Price Change (False) → No action / end workflow.

9. **Configure Credentials**  
   - Google Sheets: OAuth2 or API key with read/write access to the specified sheet.  
   - SMTP: Valid SMTP server credentials allowing email sending.

10. **Optional: Add Sticky Note**  
    - Add a sticky note summarizing the operational process and customization tips for reference within the workflow canvas.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow can be customized to adjust the Cron schedule for different frequencies (hourly, daily, etc.) and to fetch additional competitor data fields such as availability or new features.                                          | Customization options outlined in the sticky note node within the workflow.                                      |
| Email alert content can be tailored for richer formatting (HTML) or alternative notification channels like Slack or WhatsApp can be integrated for multi-channel alerts.                                                               | Suggested enhancements from sticky note remarks.                                                                 |
| Ensure Google Sheets API quotas and SMTP server limits are monitored to prevent interruptions in data logging and alerting services.                                                                                                    | Operational best practice.                                                                                        |
| Replace all placeholder values (Google Sheet ID, email addresses, API URLs) with real values before deployment.                                                                                                                        | Critical for workflow functionality.                                                                             |
| The workflow relies on the assumption that competitor APIs return consistent JSON structures with fields `name`, `price`, `features`, and `location`. API schema changes may require updates to the "Parse Data" function node.            | Integration maintenance note.                                                                                     |
| For more advanced monitoring, consider adding error handling nodes to capture and notify on HTTP request failures or Google Sheets append errors.                                                                                        | Recommended for production-grade workflows.                                                                       |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow designed for integration and automation purposes. The processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is lawful and publicly accessible.