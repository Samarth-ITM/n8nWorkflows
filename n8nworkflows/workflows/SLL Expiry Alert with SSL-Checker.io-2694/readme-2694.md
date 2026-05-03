SLL Expiry Alert with SSL-Checker.io

https://n8nworkflows.xyz/workflows/sll-expiry-alert-with-ssl-checker-io-2694


# SLL Expiry Alert with SSL-Checker.io

### 1. Workflow Overview

This workflow automates the monitoring of SSL certificates for a list of websites, leveraging SSL-Checker.io to verify certificate validity and expiry dates. It is designed to run on a weekly schedule, fetch URLs from a Google Sheet, check their SSL certificate status, update the sheet with the latest SSL details, and send email alerts if any certificates are expiring within 7 days.

**Target Use Cases:**  
- IT Operations and DevOps teams managing multiple websites’ SSL certificates.  
- Preventing unexpected downtime or security risks due to expired SSL certificates.  
- Automating routine SSL monitoring and alerting without manual intervention.

**Logical Blocks:**  
- **1.1 Scheduled Trigger:** Initiates the workflow weekly.  
- **1.2 URL Retrieval:** Fetches the list of URLs to monitor from Google Sheets.  
- **1.3 SSL Status Check:** Calls SSL-Checker.io API to get SSL certificate details for each URL.  
- **1.4 Data Update:** Updates the Google Sheet with the latest SSL expiry information.  
- **1.5 Expiry Evaluation:** Checks if any certificate is expiring within 7 days.  
- **1.6 Notification:** Sends alert emails for certificates nearing expiry.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow once a week on Monday at 8 AM Asia/Kolkata time.

- **Nodes Involved:**  
  - Weekly Trigger  
  - Sticky Note (descriptive)

- **Node Details:**  

  - **Weekly Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Runs weekly, specifically on Mondays at 8:00 AM.  
    - Input: None (trigger node)  
    - Output: Initiates the workflow by passing control to the next node.  
    - Edge Cases: Workflow will not run if n8n server is down or if timezone is misconfigured.  
    - Version: 1.2  

  - **Sticky Note**  
    - Purpose: Describes the trigger schedule for user clarity.  
    - No technical impact.

---

#### 1.2 URL Retrieval

- **Overview:**  
  Retrieves the list of website URLs to monitor from a Google Sheet. This list serves as the input for SSL checking.

- **Nodes Involved:**  
  - Fetch URLs (Google Sheets node)  
  - Sticky Note (instructional)

- **Node Details:**  

  - **Fetch URLs**  
    - Type: Google Sheets  
    - Configuration: Reads data from a specified Google Sheet URL and sheet tab (gid=0).  
    - Credentials: Uses Google Sheets OAuth2 credentials named "Vishal - Google Sheets".  
    - Input: Triggered by Weekly Trigger node.  
    - Output: Emits JSON objects containing URLs to be checked.  
    - Edge Cases:  
      - Authentication failure if credentials expire or are revoked.  
      - Empty or malformed sheet data may cause downstream errors.  
      - Sheet URL or tab misconfiguration leads to no data fetched.  
    - Version: 4.5  

  - **Sticky Note1**  
    - Content: Advises cloning and updating the Google Sheet URL to customize monitored URLs.

---

#### 1.3 SSL Status Check

- **Overview:**  
  For each URL fetched, this block calls the SSL-Checker.io API to retrieve SSL certificate details including host, expiry date, and days left.

- **Nodes Involved:**  
  - Check SSL (HTTP Request)  
  - Sticky Note (explanatory)

- **Node Details:**  

  - **Check SSL**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: GET (default)  
      - URL: Dynamically constructed by stripping protocol and trailing slash from the URL field, then appending to `https://ssl-checker.io/api/v1/check/`.  
      - No additional headers or authentication specified (assumes public API or API key handled elsewhere).  
    - Input: Receives URL data from Fetch URLs node.  
    - Output: JSON response with SSL details per URL.  
    - Edge Cases:  
      - API rate limits or downtime may cause request failures.  
      - Malformed URLs may cause API errors.  
      - Network timeouts or connectivity issues.  
    - Version: 4.2  

  - **Sticky Note2**  
    - Content: Describes the use of SSL-Checker.io API to fetch SSL certificate details.

---

#### 1.4 Data Update

- **Overview:**  
  Updates the Google Sheet with the latest SSL certificate information such as the expiry date and status.

- **Nodes Involved:**  
  - URLs to Monitor (Google Sheets update node)  
  - Sticky Note (instructional)

- **Node Details:**  

  - **URLs to Monitor**  
    - Type: Google Sheets  
    - Operation: Update rows in the sheet based on matching URLs.  
    - Configuration:  
      - Updates columns "URL" and "KnownExpiryDate" with values from the SSL check result (`result.host` and `result.valid_till`).  
      - Uses "URL" as the matching column to identify rows to update.  
      - Sheet URL and tab specified (gid=0).  
    - Credentials: Uses the same Google Sheets OAuth2 credentials as Fetch URLs.  
    - Input: Receives SSL check results from Check SSL node.  
    - Output: Confirmation of updated rows (not used downstream).  
    - Edge Cases:  
      - Update failures if sheet is locked or credentials invalid.  
      - Mismatched URLs causing no rows to update.  
    - Version: 4.5  

  - **Sticky Note3**  
    - Content: Explains that this node updates the Google Sheet with SSL expiry data.

---

#### 1.5 Expiry Evaluation

- **Overview:**  
  Evaluates if any SSL certificate is expiring within 7 days or less to decide if an alert should be sent.

- **Nodes Involved:**  
  - Expiry Alert (IF node)  
  - Sticky Note (explanatory)

- **Node Details:**  

  - **Expiry Alert**  
    - Type: IF (conditional)  
    - Configuration:  
      - Condition: Checks if `result.days_left` (days until expiry) is less than or equal to 7.  
      - Uses strict number comparison.  
    - Input: Receives SSL check results from URLs to Monitor node.  
    - Output:  
      - True branch: Certificates expiring soon.  
      - False branch: Certificates not expiring soon.  
    - Edge Cases:  
      - Missing or malformed `days_left` field could cause condition failure.  
      - Timezone or date parsing errors affecting days calculation.  
    - Version: 2.2  

  - **Sticky Note4**  
    - Content: Describes the logic to check for certificates expiring within 7 days.

---

#### 1.6 Notification

- **Overview:**  
  Sends an email alert via Gmail if any SSL certificate is found to be expiring soon, including details about the host and days left.

- **Nodes Involved:**  
  - Send Alert Email (Gmail node)  
  - Sticky Note (instructional)

- **Node Details:**  

  - **Send Alert Email**  
    - Type: Gmail  
    - Configuration:  
      - Recipient: `phanineeraj@quantana.com` (hardcoded, can be customized).  
      - Subject and message: Dynamic content including days left and host from SSL check result.  
      - Email type: Plain text.  
      - Attribution appended: Disabled.  
    - Credentials: Uses Gmail OAuth2 credentials named "Sabila Gmail".  
    - Input: Triggered only if Expiry Alert node condition is true.  
    - Output: Email sent confirmation.  
    - Edge Cases:  
      - Authentication failure or revoked credentials.  
      - Email sending limits or quota exceeded.  
      - Invalid email address causing delivery failure.  
    - Version: 2.1  

  - **Sticky Note5**  
    - Content: Explains that this node sends email alerts for certificates nearing expiry.

---

### 3. Summary Table

| Node Name         | Node Type           | Functional Role                      | Input Node(s)       | Output Node(s)       | Sticky Note                                                                                   |
|-------------------|---------------------|------------------------------------|---------------------|----------------------|----------------------------------------------------------------------------------------------|
| Weekly Trigger    | Schedule Trigger    | Initiates workflow weekly           | None                | Fetch URLs            | Triggers the workflow once a week.                                                           |
| Fetch URLs        | Google Sheets       | Retrieves URLs to monitor            | Weekly Trigger      | Check SSL             | Pulls the list of URLs to monitor from the Google Sheet. Ensure you clone the sheet and update URL. |
| Check SSL         | HTTP Request        | Calls SSL-Checker.io API             | Fetch URLs          | URLs to Monitor       | Uses SSL-Checker.io to verify SSL certificates and fetch details like host, expiry, days left. |
| URLs to Monitor   | Google Sheets       | Updates sheet with SSL details       | Check SSL           | Expiry Alert          | Updates the Google Sheet with SSL expiry date and certificate status.                        |
| Expiry Alert      | IF Node             | Checks if SSL expires in ≤7 days     | URLs to Monitor     | Send Alert Email      | Checks if any SSL certificate is set to expire in 7 days or less.                            |
| Send Alert Email  | Gmail               | Sends email alert for expiring cert | Expiry Alert (true) | None                  | Sends an email alert including host and days left if SSL certificate is nearing expiry.     |
| Sticky Note       | Sticky Note         | Descriptive                        | None                | None                  | Triggers the workflow once a week.                                                           |
| Sticky Note1      | Sticky Note         | Descriptive                        | None                | None                  | Pulls the list of URLs to monitor from the Google Sheet. Ensure you clone the sheet and update URL. |
| Sticky Note2      | Sticky Note         | Descriptive                        | None                | None                  | Uses SSL-Checker.io to verify SSL certificates and fetch details.                            |
| Sticky Note3      | Sticky Note         | Descriptive                        | None                | None                  | Updates the Google Sheet with SSL expiry data.                                              |
| Sticky Note4      | Sticky Note         | Descriptive                        | None                | None                  | Checks if any SSL certificate is set to expire in 7 days or less.                            |
| Sticky Note5      | Sticky Note         | Descriptive                        | None                | None                  | Sends an email alert if an SSL certificate is nearing expiry.                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Weekly Trigger`  
   - Set to trigger weekly on Monday at 8:00 AM (Asia/Kolkata timezone).  
   - No credentials required.

3. **Add a Google Sheets node to fetch URLs:**  
   - Name: `Fetch URLs`  
   - Operation: Read rows from a Google Sheet.  
   - Configure the Google Sheet URL and sheet tab (gid=0) containing the list of URLs to monitor.  
   - Use Google Sheets OAuth2 credentials (create or import as needed).  
   - Connect `Weekly Trigger` output to this node’s input.

4. **Add an HTTP Request node to check SSL:**  
   - Name: `Check SSL`  
   - Method: GET  
   - URL: Use expression to build URL:  
     ```
     https://ssl-checker.io/api/v1/check/{{ $json["URL"].replace(/^https?:\/\//, "").replace(/\/$/, "") }}
     ```  
   - No authentication needed unless SSL-Checker.io requires API keys (not specified here).  
   - Connect `Fetch URLs` output to this node’s input.

5. **Add a Google Sheets node to update SSL info:**  
   - Name: `URLs to Monitor`  
   - Operation: Update rows in the Google Sheet.  
   - Configure the same Google Sheet URL and tab as in step 3.  
   - Map columns:  
     - `URL` → `{{ $json.result.host }}`  
     - `KnownExpiryDate` → `{{ $json.result.valid_till }}`  
   - Set matching column as `URL` to update correct rows.  
   - Use the same Google Sheets OAuth2 credentials.  
   - Connect `Check SSL` output to this node’s input.

6. **Add an IF node to evaluate expiry:**  
   - Name: `Expiry Alert`  
   - Condition: Check if `{{ $json.result.days_left }}` is less than or equal to 7 (number comparison).  
   - Connect `URLs to Monitor` output to this node’s input.

7. **Add a Gmail node to send alert emails:**  
   - Name: `Send Alert Email`  
   - Configure Gmail OAuth2 credentials.  
   - Set recipient email (e.g., `phanineeraj@quantana.com`).  
   - Subject: `SSL Expiry - {{ $json.result.days_left }} Days Left - {{ $json.result.host }}`  
   - Message: Same as subject or customized text.  
   - Email type: Plain text.  
   - Connect the `true` output of `Expiry Alert` node to this node’s input.

8. **Add Sticky Notes (optional) for documentation:**  
   - Add notes describing each block for clarity.

9. **Set workflow timezone to Asia/Kolkata** in workflow settings.

10. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow designed to automate SSL certificate expiry monitoring and alerting.                                 | Use case description in workflow overview.                                                             |
| Google Sheets used as the source of URLs and for updating SSL status.                                         | Can be replaced by Airtable, CSV, or other data sources as needed.                                      |
| SSL-Checker.io API endpoint used: `https://ssl-checker.io/api/v1/check/{hostname}`                            | Official API documentation (if available) should be consulted for rate limits and authentication.       |
| Email alerts sent via Gmail OAuth2 credentials; ensure credentials are valid and have sending permissions.    | Gmail API quota and limits apply.                                                                       |
| Suggested improvements: add Slack/Teams notifications, automate renewal requests, customize expiry thresholds.| Workflow is modular and can be extended with additional nodes and integrations.                          |
| Workflow timezone set to Asia/Kolkata; adjust if running in different regions.                                | Important for correct scheduling and expiry date calculations.                                         |

---

This structured documentation provides a complete understanding of the SSL Expiry Alert workflow, enabling reproduction, modification, and troubleshooting by both human operators and automation agents.