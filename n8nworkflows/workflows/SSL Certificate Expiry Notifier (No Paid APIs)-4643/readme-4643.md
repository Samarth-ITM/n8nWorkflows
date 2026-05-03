SSL Certificate Expiry Notifier (No Paid APIs)

https://n8nworkflows.xyz/workflows/ssl-certificate-expiry-notifier--no-paid-apis--4643


# SSL Certificate Expiry Notifier (No Paid APIs)

---

### 1. Workflow Overview

This workflow, titled **SSL Certificate Expiry Notifier (No Paid APIs)**, automates the monitoring of SSL certificate expiration dates for multiple website URLs listed in a Google Sheet. Its primary use case is for IT operations and DevOps teams who need timely alerts about impending SSL certificate expirations without relying on paid API services. The workflow runs daily, fetches monitored URLs, checks SSL certificate statuses using a free public API, updates the sheet with fresh certificate data, and sends email notifications for certificates expiring within a configurable threshold.

The workflow’s logic is grouped into these key blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow daily at a set time.
- **1.2 URL Retrieval**: Reads the list of URLs to monitor from a Google Sheet.
- **1.3 SSL Certificate Check**: Queries a free SSL checking API (`ssl-checker.io`) for certificate validity details.
- **1.4 Data Update**: Updates the Google Sheet with the latest certificate information.
- **1.5 Expiry Evaluation**: Determines if a certificate is about to expire within 7 days.
- **1.6 Alerting**: Sends email notifications for certificates nearing expiry.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Initiates execution daily at 8:00 AM to automate the SSL monitoring process without manual intervention.

- **Nodes Involved:**  
  - `Daily Trigger`

- **Node Details:**  
  - **Node:** Daily Trigger  
    - Type: Schedule Trigger  
    - Configuration: Runs once daily at 8:00 AM (hour-based interval trigger)  
    - Inputs: None (start node)  
    - Outputs: Triggers the next node (`Fetch URLs`)  
    - Edge cases: Misconfiguration of the time interval could cause missed or multiple executions; ensure server time zone matches expectations.  
    - Notes: Sticky note attached explains daily triggering and adjustment options.

#### 1.2 URL Retrieval

- **Overview:**  
  Retrieves the list of website URLs to monitor from a specified Google Sheet. This provides the dynamic input for SSL checks.

- **Nodes Involved:**  
  - `Fetch URLs`

- **Node Details:**  
  - **Node:** Fetch URLs  
    - Type: Google Sheets (Read)  
    - Configuration: Reads data from a Google Sheet URL (configured with OAuth2 credentials)  
    - Key Parameters: Sheet URL and document ID point to a shared Google Sheet containing a `URL` column  
    - Inputs: Triggered by `Daily Trigger`  
    - Outputs: Emits an array of URLs (each row as JSON)  
    - Edge cases:  
      - Google Sheets API quota limits or authentication errors may interrupt workflow.  
      - Empty or missing `URL` column will cause downstream failures.  
    - Notes: Sticky note details importance of correct sheet and URL column presence.

#### 1.3 SSL Certificate Check

- **Overview:**  
  Calls the free API `ssl-checker.io` to retrieve SSL certificate details for each URL fetched. Extracts validity period and days left.

- **Nodes Involved:**  
  - `Check SSL`

- **Node Details:**  
  - **Node:** Check SSL  
    - Type: HTTP Request  
    - Configuration:  
      - URL built dynamically by stripping `http://` or `https://` prefix and trailing slash from each URL, then appending to `https://ssl-checker.io/api/v1/check/` endpoint  
      - Uses GET method (default)  
    - Inputs: Receives each URL JSON from `Fetch URLs`  
    - Outputs: JSON containing `result` object with `host`, `valid_from`, `valid_till`, `days_left`  
    - Edge cases:  
      - API downtime or rate limits could cause request failures or missing data.  
      - Invalid or malformed URLs may result in errors or empty responses.  
      - Network timeouts or connectivity issues.  
    - Notes: Sticky note explains API usage and returned data fields.

#### 1.4 Data Update

- **Overview:**  
  Updates the original Google Sheet with the latest SSL certificate details per URL. This keeps the monitoring sheet current.

- **Nodes Involved:**  
  - `URLs to Monitor`

- **Node Details:**  
  - **Node:** URLs to Monitor  
    - Type: Google Sheets (Update)  
    - Configuration:  
      - Writes back to the same Google Sheet URL and document ID as `Fetch URLs`  
      - Updates columns: `URL`, `Valid From`, `Valid Till`, `Days Left` using data from `Check SSL` output  
      - Matching rows by `URL` to update existing records without duplication  
    - Inputs: Receives SSL check results from `Check SSL`  
    - Outputs: Updated rows confirmation (not connected further)  
    - Edge cases:  
      - Sheet permission issues or quota limits may block updates.  
      - If URLs are not unique or correctly matched, wrong rows may be updated.  
    - Notes: Sticky note details update logic and mapping.

#### 1.5 Expiry Evaluation

- **Overview:**  
  Evaluates whether each certificate’s `days_left` value is less than or equal to 7 days, to decide if an alert is necessary.

- **Nodes Involved:**  
  - `Expiry Alert`

- **Node Details:**  
  - **Node:** Expiry Alert  
    - Type: If Condition  
    - Configuration: Numeric comparison: `days_left <= 7`  
    - Inputs: Receives SSL check results from `URLs to Monitor` node  
    - Outputs: Two branches—`true` (certificate expiring soon) and `false` (certificate still valid)  
    - Edge cases:  
      - Missing or non-numeric `days_left` values may cause evaluation errors.  
    - Notes: Sticky note clarifies the threshold check and filtering logic.

#### 1.6 Alerting

- **Overview:**  
  Sends an HTML email notification to preconfigured recipients if a certificate is expiring soon.

- **Nodes Involved:**  
  - `Send Email`

- **Node Details:**  
  - **Node:** Send Email  
    - Type: Email Send (SMTP)  
    - Configuration:  
      - Subject dynamically includes `days_left` and `host` info  
      - HTML body formatted with styled message highlighting expiry details  
      - Recipients: hardcoded list (`team1@gmail.com, team2@gmail.com`)  
      - Sender: `admin@gmail.com`  
      - Uses SMTP credentials for authenticated sending  
    - Inputs: Triggered only from the `true` branch of `Expiry Alert`  
    - Outputs: None (terminal node)  
    - Edge cases:  
      - SMTP authentication failures or network errors may block email delivery  
      - Invalid recipient addresses cause bounce backs  
      - HTML formatting issues might affect readability on some email clients  
    - Notes: Sticky note describes email purpose and customization.

---

### 3. Summary Table

| Node Name       | Node Type             | Functional Role              | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                                                   |
|-----------------|-----------------------|-----------------------------|----------------------|----------------------|------------------------------------------------------------------------------------------------------------------------------|
| Daily Trigger   | Schedule Trigger      | Initiates workflow daily     | None                 | Fetch URLs           | Runs the workflow every day at 8:00 AM. Adjust time or frequency as needed using cron or interval settings.                   |
| Fetch URLs      | Google Sheets (Read)  | Retrieves URLs to monitor    | Daily Trigger        | Check SSL            | Reads website URLs from a Google Sheet. Ensure the sheet has a column named `URL` containing the domains to monitor.         |
| Check SSL       | HTTP Request          | Queries SSL certificate API | Fetch URLs           | URLs to Monitor, Expiry Alert | Queries `ssl-checker.io` free API for SSL certificate data of each domain. Returns valid_from, valid_till, days_left, host info. |
| URLs to Monitor | Google Sheets (Update)| Updates sheet with SSL data | Check SSL            | Expiry Alert         | Updates the Google Sheet with new certificate data: Valid From, Valid Till, Days Left. Matches by URL to update correct rows. |
| Expiry Alert    | If Condition          | Checks expiry threshold     | URLs to Monitor       | Send Email (true)    | Checks if `days_left` ≤ 7. Only domains meeting this condition trigger email alerts.                                          |
| Send Email      | Email Send (SMTP)     | Sends alert emails          | Expiry Alert (true)   | None                 | Sends email to specified recipients if a certificate is expiring soon. Includes days left and domain name in HTML template.   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**
   - Name: `Daily Trigger`
   - Type: Schedule Trigger
   - Set to trigger once daily at 8:00 AM (using interval trigger by hour).
   - No credentials required.

3. **Add a Google Sheets node (Read operation):**
   - Name: `Fetch URLs`
   - Operation: Read
   - Configure with Google Sheets OAuth2 credentials.
   - Set Spreadsheet URL and Sheet Name (e.g., your monitoring sheet with a `URL` column).
   - Ensure it reads all rows with `URL` values.
   - Connect output from `Daily Trigger` to this node.

4. **Add an HTTP Request node:**
   - Name: `Check SSL`
   - Method: GET (default)
   - URL: Use expression to dynamically generate the URL:  
     `https://ssl-checker.io/api/v1/check/{{ $json["URL"].replace(/^https?:\/\//, "").replace(/\/$/, "") }}`
   - Connect input from `Fetch URLs`.

5. **Add a Google Sheets node (Update operation):**
   - Name: `URLs to Monitor`
   - Operation: Update
   - Use same Spreadsheet URL and Sheet Name as `Fetch URLs`.
   - Map columns:  
     - `URL` → from `$json.result.host`  
     - `Valid From` → from `$json.result.valid_from`  
     - `Valid Till` → from `$json.result.valid_till`  
     - `Days Left` → from `$json.result.days_left`  
   - Set matching column to `URL` to update existing rows.
   - Connect input from `Check SSL`.

6. **Add an If node:**
   - Name: `Expiry Alert`
   - Condition: Numeric comparison `days_left` ≤ 7  
     Expression: `{{ $json.result.days_left }} <= 7`
   - Connect input from `URLs to Monitor`.

7. **Add an Email Send node:**
   - Name: `Send Email`
   - Use SMTP credentials configured for your email provider.
   - Configure:  
     - From Email: `admin@gmail.com` (or your sender email)  
     - To Email: `team1@gmail.com,team2@gmail.com` (or your recipients)  
     - Subject: `SSL Expiry - {{ $json.result.days_left }} Days Left - {{ $json.result.host }}`  
     - HTML Body: Use the provided styled HTML template embedding domain and days left dynamically.
   - Connect input from the `true` output of `Expiry Alert`.

8. **Set node execution order explicitly:**
   - `Daily Trigger` → `Fetch URLs` → `Check SSL` → `URLs to Monitor` → `Expiry Alert` → `Send Email` (only on true condition)

9. **Test the workflow:**
   - Ensure Google Sheets credentials have read/write access to the specified sheet.
   - Confirm SMTP credentials are valid and allowed to send email.
   - Verify URLs in the sheet are correctly formatted (e.g., including protocol is optional but handled).
   - Adjust the expiry threshold by changing the value in the If node condition if needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                             | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow uses **ssl-checker.io** free public API to avoid paid service dependencies for SSL monitoring.                                                  | https://ssl-checker.io/api                                                                              |
| The Google Sheet must have a `URL` column with domain URLs to monitor. Example sheet URL needs to be replaced with your own in the nodes.                 | Google Sheets setup                                                                                      |
| Email notifications use SMTP credentials; make sure your SMTP service allows sending from the configured email address to avoid delivery issues.          | SMTP provider setup                                                                                      |
| The email body is a fully styled HTML template designed for clarity and branding; customize as needed in the `Send Email` node.                          | HTML email template embedded in node                                                                    |
| The workflow is ideal for small teams or IT admins managing multiple domains who want a flexible, self-hosted solution without recurring costs.            | Workflow description sticky note                                                                        |
| Adjust alert thresholds or email recipients via node parameters to fit operational needs.                                                                | Configuration flexibility                                                                                 |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.

---