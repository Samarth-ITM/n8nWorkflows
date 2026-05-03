Monitor SSL Certificate Expiry Dates with Google Sheets & Slack Alerts

https://n8nworkflows.xyz/workflows/monitor-ssl-certificate-expiry-dates-with-google-sheets---slack-alerts-5172


# Monitor SSL Certificate Expiry Dates with Google Sheets & Slack Alerts

### 1. Workflow Overview

This workflow automates the monitoring of SSL certificate expiry dates for a list of websites managed in a Google Sheets document. It periodically checks each domain's SSL certificate validity and updates the sheet with the remaining days until expiry. If the remaining validity falls below a defined threshold (8 days), it sends a Slack alert to notify the responsible user.

Logical blocks:

- **1.1 Scheduled Trigger**: Periodically initiates the workflow (weekly).
- **1.2 Google Sheets Data Retrieval**: Reads domain list and associated data from a specific Google Sheet.
- **1.3 SSL Certificate Checking**: Uses a custom SSL checking node to fetch SSL certificate expiry information per domain.
- **1.4 Google Sheets Update**: Updates the sheet with the latest "Days Left" value for each domain.
- **1.5 Threshold Evaluation & Slack Notification**: Checks if the certificate is about to expire soon and sends Slack alerts accordingly.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow execution on a weekly interval to ensure regular SSL expiry checks.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Node Name:** Schedule Trigger  
  - **Type:** scheduleTrigger (n8n built-in)  
  - **Configuration:**  
    - Interval set to 1 week (recurring schedule)  
  - **Input:** None (trigger node)  
  - **Output:** Initiates workflow by passing empty data to next node  
  - **Failure Types:** Scheduling misconfiguration (unlikely), n8n runtime downtime may delay execution  
  - **Version Requirements:** Compatible with n8n versions supporting scheduleTrigger v1.2  

#### 2.2 Google Sheets Data Retrieval

- **Overview:**  
  Retrieves all rows from a predefined Google Sheet containing the list of websites to monitor.

- **Nodes Involved:**  
  - Get row(s) in sheet

- **Node Details:**  
  - **Node Name:** Get row(s) in sheet  
  - **Type:** googleSheets (v4.7)  
  - **Configuration:**  
    - Document ID set to a placeholder (`YOUR_SHEET_ID_HERE`), to be replaced with actual Google Sheet ID  
    - Sheet name set to first sheet (gid=0)  
    - Operation: Read rows (default)  
  - **Credentials:** Google Sheets OAuth2 (authenticated account with access to the sheet)  
  - **Input:** Trigger output from Schedule Trigger  
  - **Output:** JSON array of rows, each containing domain info under property `Website` and a tracked `row_number`  
  - **Expressions:** Uses static configuration with caching of document and sheet details for UI convenience  
  - **Failure Types:** OAuth token expiration, insufficient permissions, sheet ID or name invalid, API quota exceeded  
  - **Version Requirements:** Google Sheets node v4.7 or higher recommended for latest features  

#### 2.3 SSL Certificate Checking

- **Overview:**  
  For each website fetched, this node performs an SSL check to determine the number of days left before the certificate expires.

- **Nodes Involved:**  
  - SSL Checker

- **Node Details:**  
  - **Node Name:** SSL Checker  
  - **Type:** Custom JS node `@custom-js/n8n-nodes-pdf-toolkit.sslChecker` (v1)  
  - **Configuration:**  
    - Domain parameter set dynamically via expression: `{{$json.Website}}` (domain from Google Sheets row)  
  - **Credentials:** CustomJS account credentials required to execute this node  
  - **Input:** Individual row JSON from Google Sheets node  
  - **Output:** Augmented JSON including an `output.daysLeft` field representing SSL certificate days remaining  
  - **Failure Types:**  
    - Domain unreachable or invalid  
    - SSL check timeout  
    - Node execution errors due to malformed data  
  - **Version Requirements:** Custom node compatibility; ensure installed and updated in n8n environment  

#### 2.4 Google Sheets Update

- **Overview:**  
  Updates the original Google Sheet row with the new "Days Left" value for the SSL certificate.

- **Nodes Involved:**  
  - Update row in sheet

- **Node Details:**  
  - **Node Name:** Update row in sheet  
  - **Type:** googleSheets (v4.7)  
  - **Configuration:**  
    - Operation: Update row  
    - Sheet name and document ID same as retrieval node  
    - Columns mapped explicitly:  
      - `Website` set to original website from "Get row(s) in sheet"  
      - `Days Left` set to `output.daysLeft` from SSL Checker output  
      - `row_number` used to identify row to update  
    - Mapping mode: "defineBelow"  
  - **Credentials:** Same Google Sheets OAuth2 credentials as in retrieval node  
  - **Input:** Data from SSL Checker node augmented with domain and row number info  
  - **Output:** Updated row confirmation  
  - **Failure Types:**  
    - Sheet locked or edited by others concurrently causing update conflicts  
    - OAuth token expiration or permission errors  
    - Row number mismatch or invalid  
  - **Version Requirements:** Same as Google Sheets read node  

#### 2.5 Threshold Evaluation & Slack Notification

- **Overview:**  
  Evaluates if the SSL certificate’s remaining days are below 8 days, and if so, sends a Slack notification to a specific user.

- **Nodes Involved:**  
  - Check Days Left Threshold (If)  
  - Send a message (Slack)

- **Node Details:**  
  - **Node Name:** Check Days Left Threshold  
  - **Type:** if node (v2.2)  
  - **Configuration:**  
    - Condition: `$json['Days Left'] < 8` (strict number comparison)  
    - Combinator: AND (single condition)  
  - **Input:** Output from Google Sheets update node  
  - **Output:**  
    - True branch: Proceed to Slack message node  
    - False branch: Ends workflow for that item  
  - **Failure Types:** Expression evaluation errors if `Days Left` is missing or not numeric  

  - **Node Name:** Send a message  
  - **Type:** slack (v2.3)  
  - **Configuration:**  
    - Text message uses expressions to insert website and days left:  
      `⏰ Reminder: SSL certificate of {{ $('Update row in sheet').item.json.Website }} will expire in {{ $json["Days Left"] }} days.`  
    - Sends direct message to a user specified by Slack user ID credential placeholder (`SLACK_USER_ID`)  
    - Select mode: user  
  - **Credentials:** Slack API OAuth2 credential with scopes for chat message posting  
  - **Input:** True branch data from If node  
  - **Output:** Confirmation of message sent  
  - **Failure Types:**  
    - Invalid Slack user ID  
    - API permission errors  
    - Rate limits or webhook failures  

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                         | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                   |
|-------------------------|---------------------------|---------------------------------------|--------------------------|----------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger        | scheduleTrigger           | Initiates workflow weekly              | —                        | Get row(s) in sheet        |                                                                                              |
| Get row(s) in sheet     | googleSheets              | Reads website list from Google Sheet  | Schedule Trigger          | SSL Checker                |                                                                                              |
| SSL Checker             | @custom-js sslChecker     | Checks SSL expiry days for each domain| Get row(s) in sheet       | Update row in sheet         |                                                                                              |
| Update row in sheet     | googleSheets              | Updates sheet with SSL expiry info    | SSL Checker              | Check Days Left Threshold   |                                                                                              |
| Check Days Left Threshold| if                       | Decides whether to alert based on days left| Update row in sheet  | Send a message             |                                                                                              |
| Send a message          | slack                     | Sends Slack alert for SSL expiry      | Check Days Left Threshold | —                          |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it appropriately (e.g., "Automated SSL Expiry Tracking and Notifications").

2. **Add a Schedule Trigger node**:  
   - Set to trigger every 1 week (interval → field: weeks, value: 1).  
   - No credentials needed.

3. **Add a Google Sheets 'Get row(s) in sheet' node**:  
   - Connect from Schedule Trigger.  
   - Set operation to read all rows (default).  
   - Specify the Google Sheets document ID (replace `YOUR_SHEET_ID_HERE` with actual ID).  
   - Set Sheet Name to the first sheet or by GID = 0.  
   - Use Google Sheets OAuth2 credentials with read access to the document.

4. **Add the Custom JS SSL Checker node**:  
   - Connect from the Google Sheets get rows node.  
   - Set the domain parameter to `{{$json.Website}}` to pass each domain from the sheet.  
   - Use the CustomJS API credentials configured in your n8n instance.

5. **Add a Google Sheets 'Update row in sheet' node**:  
   - Connect from the SSL Checker node.  
   - Set operation to "update".  
   - Use the same Google Sheets document ID and sheet name as the retrieval node.  
   - Map columns explicitly:  
     - `Website` from `$('Get row(s) in sheet').item.json.Website`  
     - `Days Left` from `$json.output.daysLeft` (output of SSL Checker)  
     - `row_number` from `$('Get row(s) in sheet').item.json.row_number` (to identify the row)  
   - Use same Google Sheets OAuth2 credentials.

6. **Add an If node named 'Check Days Left Threshold'**:  
   - Connect from the Google Sheets update node.  
   - Configure condition:  
     - Left value: `{{$json["Days Left"]}}` (cast as number)  
     - Operation: less than (<)  
     - Right value: 8  
   - This checks if the SSL certificate is expiring in less than 8 days.

7. **Add a Slack 'Send a message' node**:  
   - Connect from the true output of the If node.  
   - Configure the message text using expressions:  
     `⏰ Reminder: SSL certificate of {{ $('Update row in sheet').item.json.Website }} will expire in {{ $json["Days Left"] }} days.`  
   - Set the recipient user via Slack user ID (replace `SLACK_USER_ID` with actual Slack user ID).  
   - Use Slack OAuth2 credentials with permissions to send messages.

8. **Activate the workflow** and test by running manually or waiting for the scheduled trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Replace placeholders such as `YOUR_SHEET_ID_HERE` and `SLACK_USER_ID` with actual values before running the workflow.| Crucial for integration to function correctly.                                                  |
| Custom SSL Checker node requires installation and valid CustomJS API credentials in n8n.                           | Node source: `@custom-js/n8n-nodes-pdf-toolkit.sslChecker`                                      |
| Slack user ID can be found via Slack API methods or user profile inspection.                                       | Useful Slack API documentation: https://api.slack.com/methods/users.info                         |
| Google Sheets OAuth2 credentials must have both read and write permissions to the target document.                 | Set up in n8n credentials section with proper scopes and consent.                               |
| Workflow tags include SSL, Monitoring, Google Sheets, Slack, Alerts, Security, Automation.                          | Helpful for categorizing and searching workflows in n8n.                                        |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.