Monitor SSL Certificate Expiry with Google Sheets and Email Alerts

https://n8nworkflows.xyz/workflows/monitor-ssl-certificate-expiry-with-google-sheets-and-email-alerts-5768


# Monitor SSL Certificate Expiry with Google Sheets and Email Alerts

### 1. Workflow Overview

**Workflow Purpose:**  
This workflow automates the monitoring of SSL certificate expiry dates for a list of websites maintained in a Google Sheets spreadsheet. It retrieves SSL certificate details via an API, updates the spreadsheet with the current SSL status, and sends email alerts if any certificates are expiring within 14 days. This helps website administrators proactively manage SSL renewals and maintain site security.

**Target Use Cases:**  
- IT teams or website admins needing routine SSL certificate status monitoring  
- Automated alerting for SSL certificates nearing expiration  
- Maintaining a centralized Google Sheets record of SSL statuses for multiple websites  

**Logical Blocks:**

- **1.1 Scheduled Trigger:** Runs the workflow weekly on Monday at 7:00 AM.  
- **1.2 Fetch Website List:** Reads website URLs and metadata from a Google Sheets spreadsheet.  
- **1.3 Batch Processing Loop:** Processes each website URL individually to avoid rate limits and manage API calls.  
- **1.4 Get SSL Status:** Calls ssl-checker.io API to retrieve SSL certificate details per website.  
- **1.5 Update Spreadsheet:** Updates the Google Sheets spreadsheet with the latest SSL information.  
- **1.6 SSL Expiry Check:** Filters SSL certificates expiring within 14 days and generates messages accordingly.  
- **1.7 Conditional Alert:** Checks if any certificates are expiring soon and triggers an email alert if true.  
- **1.8 Send Email Alert:** Sends an email listing websites with expiring SSL certificates.  

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow every Monday at 7:00 AM to perform SSL monitoring weekly.

- **Nodes Involved:**  
  - Trigger Every Monday

- **Node Details:**  
  - **Trigger Every Monday**  
    - Type: Schedule Trigger  
    - Configuration: Runs weekly on Monday, 7:00 AM (Asia/Singapore timezone)  
    - Input: None (trigger node)  
    - Output: Starts the workflow by triggering the next node "Get Website List"  
    - Edge cases/failures: None typical; possible missed trigger if n8n instance is down  
    - Sticky Note: Explains trigger schedule  

---

#### 2.2 Fetch Website List

- **Overview:**  
  Reads a list of websites to monitor from a specific Google Sheets spreadsheet. The sheet contains columns like No, Name, Link, SSL Issued On, SSL Expired On, SSL Status.

- **Nodes Involved:**  
  - Get Website List

- **Node Details:**  
  - **Get Website List**  
    - Type: Google Sheets (Read operation)  
    - Configuration: Reads from sheet with GID=0 in a specified spreadsheet URL  
    - Authentication: Uses Google Service Account credentials  
    - Input: Trigger from Schedule node  
    - Output: JSON array of website entries, each including Name and Link  
    - Edge cases:  
      - Incorrect spreadsheet URL or permissions cause auth/read errors  
      - Empty or malformed rows may yield empty or invalid data  
    - Sticky Note: Explains purpose and expected spreadsheet format  

---

#### 2.3 Batch Processing Loop

- **Overview:**  
  Processes each website entry one by one in batches for controlled handling of API calls.

- **Nodes Involved:**  
  - Loop

- **Node Details:**  
  - **Loop**  
    - Type: SplitInBatches  
    - Configuration: Default batch size (likely 1) to iterate through each website record sequentially  
    - Input: Output from "Get Website List" node  
    - Output: Sends one website item at a time to downstream nodes  
    - Edge cases:  
      - If batch size is too large, API rate limits may be triggered  
      - If input data is empty, loop runs zero times  
    - Sticky Note: None  

---

#### 2.4 Get SSL Status

- **Overview:**  
  Calls the ssl-checker.io API for each URL to obtain SSL certificate information including validity period.

- **Nodes Involved:**  
  - Get SSL

- **Node Details:**  
  - **Get SSL**  
    - Type: HTTP Request  
    - Configuration:  
      - URL template: `https://ssl-checker.io/api/v1/check/{{ $json.Link.replace(/^https?:\/\/ /, "").replace(/\/$/, "") }}`  
      - Method: GET (default)  
      - Headers: Default headers sent  
      - SendHeaders enabled (empty headers object configured)  
    - Input: Website URL from Loop node (single item)  
    - Output: JSON containing SSL certificate details such as `valid_from`, `valid_till`, and status messages  
    - Edge cases:  
      - API downtime or rate limiting causes errors/timeouts  
      - Invalid URLs cause API errors or empty results  
      - Unexpected API response formats may break downstream usage  
    - Sticky Note: Describes API usage and SSL data retrieved  

---

#### 2.5 Update Spreadsheet

- **Overview:**  
  Updates the Google Sheets spreadsheet with the latest SSL certificate details for each website after retrieving them.

- **Nodes Involved:**  
  - Update SSL in Spreadsheet

- **Node Details:**  
  - **Update SSL in Spreadsheet**  
    - Type: Google Sheets (Update operation)  
    - Configuration:  
      - Document and Sheet specified by URL and GID  
      - Matching rows by "Name" column  
      - Updates columns: SSL Issued On (valid_from), SSL Expired On (valid_till), SSL Status (from SSL API data)  
      - Authentication: Google Service Account  
    - Input: SSL certificate data from "Get SSL" node  
    - Output: Updated spreadsheet rows confirmation  
    - Edge cases:  
      - Mismatched names cause update failure or incorrect entries  
      - Permissions or quota issues on Google Sheets API  
    - Sticky Note: Details required spreadsheet columns and update logic  

---

#### 2.6 SSL Expiry Check

- **Overview:**  
  Filters all SSL certificate records to find those expiring within 14 days and prepares a summary message.

- **Nodes Involved:**  
  - Code (custom JavaScript)

- **Node Details:**  
  - **Code**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Threshold set to 14 days  
      - Compares current date with SSL Expired On date  
      - Returns JSON with message: "All Good" if no expiring certificates, otherwise lists sites with days left  
    - Input: All SSL data from previous updated spreadsheet or looped data (depending on connection)  
    - Output: JSON message with summary string  
    - Edge cases:  
      - Invalid or missing dates cause parsing errors  
      - Timezone differences may affect calculations  
    - Sticky Note: Explains logic and usage of this node  

---

#### 2.7 Conditional Alert

- **Overview:**  
  Checks if the message from the code node indicates any SSL problems and conditionally triggers the email alert.

- **Nodes Involved:**  
  - SSL Not Good?

- **Node Details:**  
  - **SSL Not Good?**  
    - Type: If node  
    - Configuration:  
      - Condition: `$json.message != "All Good"` (string not equals)  
    - Input: Message from Code node  
    - Output: True branch triggers email alert; false branch ends workflow  
    - Edge cases:  
      - Misformatted messages may cause false positives or negatives  
    - Sticky Note: Explains purpose to detect SSL expiry issues  

---

#### 2.8 Send Email Alert

- **Overview:**  
  Sends an email alert to configured recipients if any SSL certificates are expiring soon.

- **Nodes Involved:**  
  - Send Email Alert

- **Node Details:**  
  - **Send Email Alert**  
    - Type: Email Send  
    - Configuration:  
      - Subject: "⚠️ ALERT!! SSL EXPIRED"  
      - Body: Text from previous Code node message listing expiring certificates  
      - Recipients: example@gmail.com, example2@gmail.com (placeholder emails)  
      - From email: admin@balimandira.com  
      - Email format: Text only  
      - Retry on fail: enabled  
      - Credentials: SMTP credentials configured in n8n  
    - Input: Message from "SSL Not Good?" node true branch  
    - Output: Email sent confirmation  
    - Edge cases:  
      - SMTP authentication errors  
      - Network issues causing send failure (retry enabled)  
      - Invalid recipient addresses  
    - Sticky Note: Advises checking SMTP settings and email configuration  

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                      | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                  |
|-------------------------|--------------------|------------------------------------|-----------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Trigger Every Monday     | Schedule Trigger   | Weekly workflow start trigger       | None                  | Get Website List         | This workflow trigger is scheduled to run every Monday at 7:00 AM.                           |
| Get Website List        | Google Sheets       | Fetch website list from spreadsheet | Trigger Every Monday  | Loop                    | Fetch URL Website: Reads website URLs from a connected spreadsheet.                         |
| Loop                    | SplitInBatches      | Process each website individually   | Get Website List       | Code, Get SSL            |                                                                                              |
| Get SSL                 | HTTP Request        | Retrieve SSL certificate info       | Loop                   | Update SSL in Spreadsheet| Get SSL Status: Checks SSL via ssl-checker.io API for each URL.                              |
| Update SSL in Spreadsheet| Google Sheets       | Update spreadsheet with SSL data    | Get SSL                 | Loop                    | Update Spreadsheet: Updates SSL fields in spreadsheet columns.                              |
| Code                    | Code (JavaScript)   | Check SSL expiry threshold & message| Loop                   | SSL Not Good?            | Check SSL Expiry Threshold Code: Filters certs expiring within 14 days, returns message.    |
| SSL Not Good?            | If                  | Conditional alert trigger           | Code                    | Send Email Alert         | Check Condition: Triggers email alert if any SSL certs expiring soon.                        |
| Send Email Alert         | Email Send          | Send expiration alert email         | SSL Not Good?           | None                    | Send Email Alert: Sends email notifications on SSL expiry.                                 |
| Sticky Note1             | Sticky Note         | Documentation                      | None                    | None                    | Weekly Trigger: Workflow runs every Monday at 7:00 AM.                                      |
| Sticky Note              | Sticky Note         | Documentation                      | None                    | None                    | Fetch URL Website: Reads website URLs from spreadsheet.                                    |
| Sticky Note2             | Sticky Note         | Documentation                      | None                    | None                    | Get SSL Status: Uses ssl-checker.io API to fetch SSL data.                                 |
| Sticky Note3             | Sticky Note         | Documentation                      | None                    | None                    | Update Spreadsheet: Explains required spreadsheet columns and update mechanism.            |
| Sticky Note4             | Sticky Note         | Documentation                      | None                    | None                    | Check SSL Expiry Threshold Code: Warns about certs expiring within 14 days.                 |
| Sticky Note5             | Sticky Note         | Documentation                      | None                    | None                    | Check Condition: Determines if alert email should be sent.                                 |
| Sticky Note6             | Sticky Note         | Documentation                      | None                    | None                    | Send Email Alert: Email alert configuration and usage advice.                              |
| Sticky Note7             | Sticky Note         | Documentation                      | None                    | None                    | Overview of whole workflow, steps and purpose.                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to run weekly on Mondays at 7:00 AM in your timezone (Asia/Singapore recommended)  
   - Connect output to next node.

2. **Create a Google Sheets Node to Fetch Website List**  
   - Type: Google Sheets (Read)  
   - Authentication: Google Service Account (create and configure credentials in n8n)  
   - Document ID: Use your spreadsheet ID containing columns: No, Name, Link, SSL Issued On, SSL Expired On, SSL Status  
   - Sheet Name: Use GID=0 or your sheet tab name  
   - Set to read all rows  
   - Connect output to the Loop node.

3. **Create a SplitInBatches Node (Loop)**  
   - Type: SplitInBatches  
   - Default batch size (1 recommended to avoid rate limits)  
   - Connect input from Google Sheets node  
   - Connect output to two nodes: Code node and HTTP Request node (parallel processing per batch).

4. **Create an HTTP Request Node (Get SSL)**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://ssl-checker.io/api/v1/check/{{ $json.Link.replace(/^https?:\/\//, "").replace(/\/$/, "") }}`  
   - Send headers enabled (empty or default headers)  
   - Connect input from Loop node  
   - Connect output to Google Sheets update node.

5. **Create a Google Sheets Node to Update Spreadsheet**  
   - Type: Google Sheets (Update)  
   - Authentication: Google Service Account  
   - Document ID and Sheet Name: Same as fetch node  
   - Mapping: Match rows by "Name" column  
   - Update columns:  
     - SSL Issued On = `{{$json.result.valid_from}}`  
     - SSL Expired On = `{{$json.result.valid_till}}`  
     - SSL Status = (pass through or derive from SSL data)  
   - Connect input from Get SSL node  
   - Connect output back to Loop node to continue processing.

6. **Create a Code Node to Check SSL Expiry**  
   - Type: Code (JavaScript)  
   - Input: Collect all SSL data from the Loop (may require aggregation, or modify workflow to accumulate data before this step)  
   - Script:  
     - Compare current date with SSL Expired On date for each website  
     - Threshold: 14 days  
     - Output message: "All Good" if none expiring soon, else list websites with days left  
   - Connect input from Loop node or aggregated data node  
   - Connect output to If node.

7. **Create an If Node (SSL Not Good?)**  
   - Type: If  
   - Condition: `$json.message != "All Good"`  
   - True branch connects to Send Email Alert node  
   - False branch ends workflow.

8. **Create an Email Send Node (Send Email Alert)**  
   - Type: Email Send  
   - Credentials: Configure SMTP credentials in n8n  
   - To Email: Enter recipient email addresses (comma-separated)  
   - From Email: Sender address  
   - Subject: "⚠️ ALERT!! SSL EXPIRED"  
   - Body: Use expression to insert message from Code node  
   - Email format: Text  
   - Enable retry on failure  
   - Connect input from If node true branch.

9. **Add Sticky Notes for Documentation** (Optional but recommended)  
   - Add notes describing each block as per the sticky notes in original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                                            |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| The workflow uses ssl-checker.io API: https://ssl-checker.io/ for retrieving SSL certificate details.                                                                  | API provider for SSL status data                                           |
| Google Service Account credentials must have access to the target Google Sheets document for reading and updating.                                                     | Google Cloud documentation on Service Accounts                            |
| Email SMTP credentials must be valid and authorized to send emails from the configured sender address.                                                                 | SMTP service provider setup                                               |
| Workflow timezone is set to Asia/Singapore; adjust if running in different locale.                                                                                      | Workflow settings in n8n                                                  |
| The workflow is designed to run once weekly but can be adapted to different frequencies by adjusting the schedule trigger.                                             | n8n schedule trigger documentation                                        |
| Spreadsheet columns must include: No, Name, Link, SSL Issued On, SSL Expired On, SSL Status for proper operation.                                                       | Spreadsheet template required columns                                    |
| For SSL expiry threshold customization, modify the `thresholdDays` variable in the Code node.                                                                           | Customization point in Code node                                          |
| Email recipients should be updated in the "Send Email Alert" node parameters to actual alert recipients.                                                                | Email settings                                                           |
| The workflow handles errors via retry on email send; consider adding additional error handling for HTTP requests and Google Sheets operations for production usage.     | Recommended workflow enhancements                                        |

---

**Disclaimer:**  
The content provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.