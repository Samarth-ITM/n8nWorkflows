Monitor SSL Certificate Expiry with Google Sheets and Multi-Channel Alert

https://n8nworkflows.xyz/workflows/monitor-ssl-certificate-expiry-with-google-sheets-and-multi-channel-alert-3493


# Monitor SSL Certificate Expiry with Google Sheets and Multi-Channel Alert

### 1. Workflow Overview

This workflow, titled **"Monitor SSL Certificate Expiry with Google Sheets and Multi-Channel Alert"**, is designed for IT administrators and professionals who need to monitor SSL certificates of multiple websites to prevent unexpected expirations. It automates the process of fetching URLs from a Google Sheet, checking their SSL certificate status via SSL-Checker.io, updating the Google Sheet with the latest SSL details, and sending alerts through email and Telegram based on certificate validity and expiry proximity.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow on a weekly basis.
- **1.2 URL Retrieval**: Fetches the list of URLs to monitor from a Google Sheet.
- **1.3 SSL Certificate Checking**: Uses SSL-Checker.io API to retrieve SSL certificate details for each URL.
- **1.4 Data Update**: Updates the Google Sheet with the latest SSL certificate information.
- **1.5 Certificate Status Evaluation and Alerting**: Classifies certificates by validity and expiry timeframes, then sends alerts via Gmail and Telegram accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow once a week on Monday at 8 AM to ensure regular SSL certificate monitoring.

- **Nodes Involved:**  
  - Weekly Trigger

- **Node Details:**  
  - **Weekly Trigger**  
    - *Type:* Schedule Trigger  
    - *Configuration:* Set to trigger weekly on Monday at 8:00 AM.  
    - *Input/Output:* No input; outputs trigger signal to "Fetch URLs" node.  
    - *Edge Cases:* If n8n instance is down or paused at trigger time, the workflow will not run until next scheduled trigger.  
    - *Version:* 1.2  

---

#### 1.2 URL Retrieval

- **Overview:**  
  Retrieves the list of URLs to monitor from a specified Google Sheet. This list forms the basis for SSL checks.

- **Nodes Involved:**  
  - Fetch URLs

- **Node Details:**  
  - **Fetch URLs**  
    - *Type:* Google Sheets  
    - *Configuration:* Reads from the first sheet (gid=0) of the Google Sheet document with ID `1aCo3vrxgheNJChElzmf4pq8h5is7E-jz4sjfV8Quprg`.  
    - *Credentials:* Uses Google Sheets OAuth2 credentials named "Google Sheets account".  
    - *Input/Output:* Receives trigger from "Weekly Trigger"; outputs list of URLs to "Check SSL".  
    - *Edge Cases:* Authentication failure, sheet access permission issues, empty or malformed sheet data.  
    - *Version:* 4.5  

---

#### 1.3 SSL Certificate Checking

- **Overview:**  
  For each URL retrieved, this block calls the SSL-Checker.io API to obtain SSL certificate details such as validity, expiry date, and days left until expiry.

- **Nodes Involved:**  
  - Check SSL  
  - Sticky Note2 (documentation)

- **Node Details:**  
  - **Check SSL**  
    - *Type:* HTTP Request  
    - *Configuration:*  
      - URL dynamically constructed by stripping protocol and trailing slash from the URL field, then calling `https://ssl-checker.io/api/v1/check/{host}`.  
      - No additional options or authentication specified (assumes public API or no auth needed).  
    - *Input/Output:* Receives URLs from "Fetch URLs"; outputs SSL check results to "URLs to Monitor" and "Switch".  
    - *Edge Cases:* API downtime, rate limiting, malformed URLs, network timeouts, unexpected API response format.  
    - *Version:* 4.2  
  - **Sticky Note2**  
    - *Purpose:* Explains that this node uses SSL-Checker.io to verify SSL certificates and fetches details like host, validity period, and days left.  

---

#### 1.4 Data Update

- **Overview:**  
  Updates the Google Sheet with the latest SSL certificate details, including expiry date and certificate status, ensuring the monitoring sheet stays current.

- **Nodes Involved:**  
  - URLs to Monitor  
  - Sticky Note3 (documentation)

- **Node Details:**  
  - **URLs to Monitor**  
    - *Type:* Google Sheets  
    - *Configuration:*  
      - Operation set to "appendOrUpdate" on the sheet named "certificate-data" (gid=636520406) in the same Google Sheet document.  
      - Columns mapped automatically with matching on "ID".  
      - Cell format set to RAW to preserve data format.  
    - *Credentials:* Uses Google Sheets OAuth2 credentials named "Google Sheets account".  
    - *Input/Output:* Receives SSL check results from "Check SSL"; outputs to "Switch" for alert evaluation.  
    - *Edge Cases:* Authentication failure, permission issues, concurrent updates causing conflicts, data format mismatches.  
    - *Version:* 4.5  
  - **Sticky Note3**  
    - *Purpose:* Notes that this node updates the Google Sheet with SSL details such as expiry date and certificate status.  

---

#### 1.5 Certificate Status Evaluation and Alerting

- **Overview:**  
  This block evaluates SSL certificate status and expiry days to classify certificates into categories (invalid, warning, notice, info). Based on classification, it sends alerts via Gmail and Telegram or logs informational messages.

- **Nodes Involved:**  
  - Switch  
  - Send Alert Email4 (invalid certificates)  
  - Telegram (invalid certificates)  
  - Send Alert Email2 (warning: expires in <30 days)  
  - Send Alert Email1 (notice: expires in <60 days)  
  - Send Alert Email6 (info: all others)  
  - Ntfy4 (info notifications)  
  - Sticky Note4, Sticky Note5, Sticky Note7, Sticky Note8, Sticky Note10 (documentation)

- **Node Details:**  
  - **Switch**  
    - *Type:* Switch  
    - *Configuration:*  
      - Routes data based on SSL certificate validity and days left until expiry.  
      - Conditions:  
        - **invalid:** `cert_valid` is false  
        - **warning:** `days_left` ≤ 30  
        - **notice:** `days_left` ≤ 60  
        - **info:** fallback for all others  
    - *Input/Output:* Receives updated SSL data from "URLs to Monitor"; outputs to respective alert nodes.  
    - *Edge Cases:* Missing or malformed `cert_valid` or `days_left` fields, unexpected data types.  
    - *Version:* 3.2  
  - **Send Alert Email4** (Invalid Certificates)  
    - *Type:* Gmail  
    - *Configuration:*  
      - Sends urgent email with subject and message indicating invalid SSL certificate and days left.  
      - Uses Gmail OAuth2 credentials named "Gmail account".  
    - *Input/Output:* Receives from "Switch" invalid output; outputs to "Telegram".  
    - *Edge Cases:* Email sending failures, authentication errors, rate limits.  
    - *Version:* 2.1  
  - **Telegram**  
    - *Type:* Telegram  
    - *Configuration:*  
      - Sends urgent message about invalid SSL certificate.  
      - Uses Telegram API credentials named "Telegram account 4".  
    - *Input/Output:* Receives from "Send Alert Email4".  
    - *Edge Cases:* Telegram API downtime, invalid credentials, message formatting issues.  
    - *Version:* 1.2  
  - **Send Alert Email2** (Warning: Expiry <30 days)  
    - *Type:* Gmail  
    - *Configuration:*  
      - Sends warning email about SSL expiry within one month.  
      - Uses Gmail OAuth2 credentials named "Gmail account".  
    - *Input/Output:* Receives from "Switch" warning output.  
    - *Edge Cases:* Same as other Gmail nodes.  
    - *Version:* 2.1  
  - **Send Alert Email1** (Notice: Expiry <60 days)  
    - *Type:* Gmail  
    - *Configuration:*  
      - Sends informational email about SSL expiry within 60 days.  
      - Uses Gmail OAuth2 credentials named "Gmail account".  
    - *Input/Output:* Receives from "Switch" notice output.  
    - *Edge Cases:* Same as other Gmail nodes.  
    - *Version:* 2.1  
  - **Send Alert Email6** (Info: No action needed)  
    - *Type:* Gmail  
    - *Configuration:*  
      - Sends info email indicating SSL check completed with no action needed.  
      - Uses Gmail OAuth2 credentials named "Gmail account".  
    - *Input/Output:* Receives from "Switch" info output; outputs to "Ntfy4".  
    - *Edge Cases:* Same as other Gmail nodes.  
    - *Version:* 2.1  
  - **Ntfy4**  
    - *Type:* ntfy (notification service)  
    - *Configuration:*  
      - Sends informational notification with tags "ssl,n8n,angie" and priority 1.  
      - Message indicates SSL expiry check completed with no further actions.  
      - No bearer token or alternate URL configured (optional).  
    - *Input/Output:* Receives from "Send Alert Email6".  
    - *Edge Cases:* ntfy service downtime, invalid topic or tags, network issues.  
    - *Version:* 1  
  - **Sticky Notes**  
    - Sticky Note4: Explains certificate grouping logic (invalid, warning, notice, info).  
    - Sticky Note5: Marks "Invalid" category.  
    - Sticky Note7: Marks "Info" category.  
    - Sticky Note8: Marks "Notice" category.  
    - Sticky Note10: Describes the Google Sheet layout for URLs to monitor.  

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role                                   | Input Node(s)       | Output Node(s)                  | Sticky Note                                                                                                  |
|--------------------|---------------------|-------------------------------------------------|---------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------|
| Weekly Trigger     | Schedule Trigger    | Triggers workflow weekly on Monday 8 AM          | -                   | Fetch URLs                    | Triggers the workflow once a week.                                                                           |
| Fetch URLs         | Google Sheets       | Retrieves URLs to monitor from Google Sheet      | Weekly Trigger      | Check SSL                    | Pulls the list of URLs to monitor from the Google Sheet. Ensure you clone the Google Sheet worksheet and update this node with its URL. This is the Sheet Layout: A1: URL, A2: n8n.io, A3: amazon.com, A4: google.com, A5: chat.openai.com |
| Check SSL          | HTTP Request        | Calls SSL-Checker.io API to get SSL details      | Fetch URLs           | URLs to Monitor, Switch       | Uses SSL-Checker.io to verify the SSL certificate of each URL. Fetches details like the host, validity period, and days remaining until expiry. |
| URLs to Monitor    | Google Sheets       | Updates Google Sheet with SSL certificate details| Check SSL            | Switch                       | Updates the Google Sheet with SSL details, including the expiry date and certificate status.                  |
| Switch             | Switch              | Classifies SSL status and routes to alert nodes  | URLs to Monitor      | Send Alert Email4, Telegram, Send Alert Email2, Send Alert Email1, Send Alert Email6, Ntfy4 | Checks certificates and groups into: invalid (certificate invalid), warning (expires in <30 days), notice (expires in <60 days), info (everything else) |
| Send Alert Email4  | Gmail               | Sends urgent email for invalid certificates      | Switch (invalid)     | Telegram                    |                                                                                                              |
| Telegram           | Telegram            | Sends urgent Telegram message for invalid certs  | Send Alert Email4    | -                           |                                                                                                              |
| Send Alert Email2  | Gmail               | Sends warning email for certificates expiring <30 days | Switch (warning)     | -                           |                                                                                                              |
| Send Alert Email1  | Gmail               | Sends notice email for certificates expiring <60 days | Switch (notice)      | -                           |                                                                                                              |
| Send Alert Email6  | Gmail               | Sends info email for certificates with no issues| Switch (info)        | Ntfy4                       |                                                                                                              |
| Ntfy4              | ntfy Notification   | Sends info notification about SSL check completion | Send Alert Email6    | -                           |                                                                                                              |
| Sticky Note2       | Sticky Note         | Documentation of SSL checking via SSL-Checker.io | -                   | -                           | Uses SSL-Checker.io to verify the SSL certificate of each URL. Fetches details like the host, validity period, and days remaining until expiry. |
| Sticky Note3       | Sticky Note         | Documentation of Google Sheet update              | -                   | -                           | Updates the Google Sheet with SSL details, including the expiry date and certificate status.                  |
| Sticky Note4       | Sticky Note         | Documentation of certificate grouping logic       | -                   | -                           | Checks certificates and groups into: invalid, warning, notice, info                                          |
| Sticky Note5       | Sticky Note         | Marks "Invalid" category                           | -                   | -                           | Invalid                                                                                                      |
| Sticky Note7       | Sticky Note         | Marks "Info" category                              | -                   | -                           | Info                                                                                                         |
| Sticky Note8       | Sticky Note         | Marks "Notice" category                            | -                   | -                           | Notice                                                                                                       |
| Sticky Note10      | Sticky Note         | Documentation of Google Sheet URL list layout     | -                   | -                           | Pulls the list of URLs to monitor from the Google Sheet. Ensure you clone the Google Sheet worksheet and update this node with its URL. This is the Sheet Layout: A1: URL, A2: n8n.io, A3: amazon.com, A4: google.com, A5: chat.openai.com |
| Sticky Note11      | Sticky Note         | Documentation of weekly trigger                    | -                   | -                           | Triggers the workflow once a week.                                                                           |
| Sticky Note6       | Sticky Note         | Overall workflow description and instructions     | -                   | -                           | Contains full workflow description, setup, customization, and notes.                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Weekly Trigger`  
   - Type: Schedule Trigger  
   - Set to trigger weekly on Monday at 8:00 AM.

3. **Add a Google Sheets node to fetch URLs:**  
   - Name: `Fetch URLs`  
   - Operation: Read rows  
   - Document ID: `1aCo3vrxgheNJChElzmf4pq8h5is7E-jz4sjfV8Quprg` (replace with your own)  
   - Sheet Name: `Sheet1` (gid=0)  
   - Credentials: Configure Google Sheets OAuth2 credentials  
   - Connect `Weekly Trigger` output to this node's input.

4. **Add an HTTP Request node to check SSL certificates:**  
   - Name: `Check SSL`  
   - Method: GET  
   - URL: Use expression to build URL:  
     ```
     https://ssl-checker.io/api/v1/check/{{ $json["URL"].replace(/^https?:\/\//, "").replace(/\/$/, "") }}
     ```  
   - Connect `Fetch URLs` output to this node.

5. **Add a Google Sheets node to update SSL details:**  
   - Name: `URLs to Monitor`  
   - Operation: Append or Update  
   - Document ID: same as above  
   - Sheet Name: `certificate-data` (gid=636520406)  
   - Columns: Auto-map input data with matching column on "ID"  
   - Cell Format: RAW  
   - Credentials: Google Sheets OAuth2 credentials  
   - Connect `Check SSL` output to this node.

6. **Add a Switch node to classify SSL certificate status:**  
   - Name: `Switch`  
   - Add rules:  
     - **invalid:** Condition: `{{$json.result.cert_valid}}` is boolean false  
     - **warning:** Condition: `{{$json.result.days_left}}` ≤ 30  
     - **notice:** Condition: `{{$json.result.days_left}}` ≤ 60  
     - Fallback output renamed to `info`  
   - Connect `URLs to Monitor` output to this node.

7. **Add Gmail nodes for alerts:**  
   - **Send Alert Email4** (invalid certificates)  
     - Subject & Message:  
       ```
       URGENT: SSL-certificate invalid, action required! - {{$json.result.days_left}} Days Left - {{$json.result.host}}
       ```  
     - Credentials: Gmail OAuth2  
     - Connect `Switch` invalid output to this node.  
   - **Send Alert Email2** (warning)  
     - Subject & Message:  
       ```
       WARNING: SSL Expiry within one month - {{$json.result.days_left}} Days Left - {{$json.result.host}}
       ```  
     - Credentials: Gmail OAuth2  
     - Connect `Switch` warning output to this node.  
   - **Send Alert Email1** (notice)  
     - Subject & Message:  
       ```
       INFO: SSL Expiry - {{$json.result.days_left}} Days Left - {{$json.result.host}}
       ```  
     - Credentials: Gmail OAuth2  
     - Connect `Switch` notice output to this node.  
   - **Send Alert Email6** (info)  
     - Subject & Message:  
       ```
       INFO: SSL Expiry check completed, took no further actions - {{$json.result.days_left}} Days Left - {{$json.result.host}}
       ```  
     - Credentials: Gmail OAuth2  
     - Connect `Switch` info output to this node.

8. **Add Telegram node for invalid certificate alerts:**  
   - Name: `Telegram`  
   - Message:  
     ```
     URGENT: SSL-certificate invalid, action required! - {{$json.result.days_left}} Days Left - {{$json.result.host}}
     ```  
   - Credentials: Telegram API  
   - Connect `Send Alert Email4` output to this node.

9. **Add ntfy notification node for info alerts:**  
   - Name: `Ntfy4`  
   - Topic: `n8n`  
   - Tags: `ssl,n8n,angie`  
   - Title:  
     ```
     INFO: SSL Expiry check completed for {{$json.result.host}}.
     ```  
   - Message:  
     ```
     INFO: SSL Expiry check completed, took no further actions - {{$json.result.days_left}} Days Left - {{$json.result.host}}.
     ```  
   - Priority: 1  
   - Connect `Send Alert Email6` output to this node.

10. **Verify all credentials are properly configured:**  
    - Google Sheets OAuth2 with access to the monitoring sheet  
    - Gmail OAuth2 with sending permissions  
    - Telegram API credentials for bot messaging

11. **Test the workflow manually or wait for the scheduled trigger.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Clone the provided Google Sheet and update the Google Sheet URL in the "URLs to Monitor" and "Fetch URLs" nodes before running the workflow.                                                                                     | Google Sheet URL: https://docs.google.com/spreadsheets/d/1aCo3vrxgheNJChElzmf4pq8h5is7E-jz4sjfV8Quprg |
| Ensure Google Sheets and Gmail credentials are properly set up with OAuth2 authentication in n8n to avoid permission issues.                                                                                                     | Credential setup in n8n                                                                             |
| Modify the trigger interval in the "Weekly Trigger" node to change monitoring frequency as needed.                                                                                                                               | Schedule Trigger node configuration                                                                |
| Customize email and Telegram alert messages in respective nodes to fit organizational communication standards.                                                                                                                  | Gmail and Telegram nodes                                                                            |
| SSL-Checker.io API is used without authentication in this workflow; verify API usage limits and consider adding authentication if required for production use.                                                                  | https://ssl-checker.io/api documentation                                                          |
| The workflow uses multiple alert channels (email, Telegram, ntfy) for redundancy and flexibility in notifications.                                                                                                              |                                                                                                    |
| Sticky notes within the workflow provide detailed explanations and instructions for each block and node, useful for onboarding and maintenance.                                                                                | n8n workflow sticky notes                                                                           |

---

This documentation provides a detailed, structured reference to understand, reproduce, and maintain the SSL Expiry Alert System workflow in n8n. It covers all nodes, their configurations, logical flow, and potential failure points to ensure robust operation and easy customization.