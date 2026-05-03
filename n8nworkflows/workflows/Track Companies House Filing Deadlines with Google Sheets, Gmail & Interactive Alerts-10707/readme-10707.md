Track Companies House Filing Deadlines with Google Sheets, Gmail & Interactive Alerts

https://n8nworkflows.xyz/workflows/track-companies-house-filing-deadlines-with-google-sheets--gmail---interactive-alerts-10707


# Track Companies House Filing Deadlines with Google Sheets, Gmail & Interactive Alerts

### 1. Workflow Overview

This workflow automates tracking and alerting for UK Companies House filing deadlines using Google Sheets as a database, Gmail for email notifications, and interactive email links to capture confirmation responses. It is designed for accounting firms managing multiple company filings, aiming to reduce manual tracking errors and ensure timely submissions to avoid penalties.

The workflow is logically divided into two main blocks:

**1.1 Daily Deadline Check and Alert System**  
- Triggered every weekday at 5 PM  
- Reads company data from a Google Sheets database  
- Fetches live filing deadlines from Companies House API for each company  
- Updates the Google Sheet with the latest due dates  
- Builds and sends an interactive, color-coded email alert listing companies and their filing deadlines, including clickable confirmation buttons

**1.2 Email Response Handler (Webhook Flow)**  
- Triggered when a recipient clicks "Yes" or "No" in the interactive email  
- Receives and processes confirmation data via webhook  
- Updates the Google Sheet with the confirmation status and timestamp  
- Sends a confirmation HTML page back to the user  

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Deadline Check and Alert System

**Overview:**  
This block performs a scheduled check of all companies, retrieves their latest filing deadlines from the Companies House API, updates the tracking database, builds a comprehensive, color-coded HTML email summarizing deadlines, and sends it to the accounting team.

**Nodes Involved:**  
- Schedule Trigger1  
- Read Company Database  
- Get Company Data  
- Update Due Dates in Sheet  
- Build Interactive Email  
- Send via Gmail1  

**Node Details:**

- **Schedule Trigger1**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow every weekday at 5 PM (Monday to Friday)  
  - Configuration: Cron expression `0 17 * * 1-5` (5 PM on weekdays)  
  - Inputs: None (trigger node)  
  - Outputs: Triggers "Read Company Database"  
  - Failure modes: Cron misconfiguration, workflow disabled  
  - Version: 1.1  

- **Read Company Database**  
  - Type: Google Sheets (Read operation)  
  - Role: Retrieves all tracked companies from the specified Google Sheet and sheet tab  
  - Configuration: Reads from a Google Sheet document with ID `1IrXd6nu3GCIH6eviMUgQrMd81ElmodDhwzb0NHSQ56E`, sheet named "Sheet2"  
  - Credentials: Google Sheets OAuth2  
  - Inputs: Trigger from Schedule Trigger1  
  - Outputs: Outputs list of companies to "Get Company Data"  
  - Failure modes: Authentication failure, sheet not found, API rate limits  
  - Version: 4.4  

- **Get Company Data**  
  - Type: HTTP Request  
  - Role: For each company, fetches live filing deadline data from Companies House API using the company's number  
  - Configuration: URL template `https://api.company-information.service.gov.uk/company/{{ $json.company_number }}` with HTTP Basic Auth using Companies House API key  
  - Credentials: HTTP Basic Auth (Company House API)  
  - Inputs: Output from "Read Company Database" (iterates per company)  
  - Outputs: Raw company data to "Update Due Dates in Sheet"  
  - Failure modes: API key invalid, request timeout, rate limiting, malformed company numbers  
  - Version: 4.1  

- **Update Due Dates in Sheet**  
  - Type: Google Sheets (Append or Update operation)  
  - Role: Updates or appends the latest filing deadlines and submission statuses back into the Google Sheet database  
  - Configuration: Matches rows by `company_number`, updates fields like `accounts_due`, `confirmation_due`, `confirmation_submitted`, `last_updated`  
  - Credentials: Google Sheets OAuth2  
  - Inputs: Data from "Get Company Data"  
  - Outputs: Passes updated company data to "Build Interactive Email"  
  - Failure modes: Sheet update conflicts, authentication errors, data mapping errors  
  - Version: 4.7  

- **Build Interactive Email**  
  - Type: Code (JavaScript)  
  - Role: Constructs an HTML email body with a styled table listing each company’s deadlines, color-coded urgency, status icons, and interactive confirmation buttons linking back to the webhook  
  - Configuration:  
    - Uses current date for report date  
    - Sorts companies by `accounts_due` date ascending  
    - Applies color coding based on days left until deadlines (Overdue, Urgent, Soon, OK)  
    - Embeds clickable links for "Yes" and "No" confirmation that target the webhook URL with query parameters  
  - Key Expressions:  
    - Filtering companies with a valid name  
    - Date calculations for urgency  
    - URL encoding of company names for links  
  - Inputs: Updated company records from "Update Due Dates in Sheet"  
  - Outputs: JSON with `emailBody`, `subject`, and total company count to "Send via Gmail1"  
  - Failure modes: JavaScript runtime errors, missing fields, invalid dates, incorrect webhook URL  
  - Version: 2  

- **Send via Gmail1**  
  - Type: Gmail (Send Email)  
  - Role: Sends the constructed interactive email to the accounting team's configured email address  
  - Configuration:  
    - Recipient email set (dynamic or placeholder "Their email")  
    - Subject from code node  
    - HTML message body from code node  
    - Attribution disabled for cleaner emails  
  - Credentials: Gmail OAuth2  
  - Inputs: Email content from "Build Interactive Email"  
  - Outputs: None (end of chain)  
  - Failure modes: Authentication errors, quota limits, email delivery failures  
  - Version: 2.1  

---

#### 2.2 Email Response Handler (Webhook Flow)

**Overview:**  
This block handles incoming HTTP requests triggered by user clicks on "Yes" or "No" confirmation buttons in the email. It records the confirmation back into Google Sheets and provides an HTML confirmation page to the user.

**Nodes Involved:**  
- Webhook - Receive Confirmation Update  
- Process Webhook Data  
- Update Google Sheet Database  
- Send Confirmation Page  

**Node Details:**

- **Webhook - Receive Confirmation Update**  
  - Type: Webhook  
  - Role: Receives GET requests triggered by email confirmation links, capturing `company_number`, `company_name`, and `confirmation_submitted` parameters  
  - Configuration: Path set to `confirmation-updates`, response mode set to use a response node for dynamic reply  
  - Inputs: External HTTP requests  
  - Outputs: Passes data to "Process Webhook Data"  
  - Failure modes: Incorrect URL, missing parameters, webhook disabled  
  - Version: 1  

- **Process Webhook Data**  
  - Type: Code (JavaScript)  
  - Role: Processes incoming webhook JSON, currently adds a dummy field `myNewField` with value 1 (placeholder for any needed transformation)  
  - Inputs: Data from webhook node  
  - Outputs: Passes processed data to "Update Google Sheet Database"  
  - Failure modes: Script errors, malformed input data  
  - Version: 2  

- **Update Google Sheet Database**  
  - Type: Google Sheets (Update operation)  
  - Role: Updates the confirmation status (`confirmation_submitted`) for the corresponding company in the Google Sheet, matching by `company_number`  
  - Configuration: Matches on `company_number`, updates `confirmation_submitted` and other related fields as needed  
  - Credentials: Google Sheets OAuth2  
  - Inputs: Processed webhook data  
  - Outputs: Passes updated record to "Send Confirmation Page"  
  - Failure modes: Sheet update conflicts, authentication errors, missing company_number  
  - Version: 4.4  

- **Send Confirmation Page**  
  - Type: Respond to Webhook  
  - Role: Sends an HTML success confirmation page back to the user who clicked the email link  
  - Configuration:  
    - Responds with styled HTML page showing company name and updated confirmation status  
    - Uses placeholders for dynamic content from JSON data  
  - Inputs: Updated data from Google Sheets update node  
  - Outputs: HTTP response to the webhook caller  
  - Failure modes: Response rendering errors, missing data fields  
  - Version: 1  

---

### 3. Summary Table

| Node Name                      | Node Type               | Functional Role                                          | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                                               |
|--------------------------------|-------------------------|----------------------------------------------------------|-------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger1               | Schedule Trigger        | Initiates daily workflow at 5 PM Mon-Fri                 | None                          | Read Company Database        | ## ⚙️ Daily Deadline Check & Alert System … Runs every weekday at 5 PM (Mon-Fri), automates company deadline checks       |
| Read Company Database           | Google Sheets           | Reads all companies from Google Sheets                    | Schedule Trigger1             | Get Company Data             | Same as above                                                                                                             |
| Get Company Data                | HTTP Request            | Fetches live filing deadlines from Companies House API   | Read Company Database         | Update Due Dates in Sheet    | Same as above                                                                                                             |
| Update Due Dates in Sheet       | Google Sheets           | Updates Google Sheet with latest filing deadlines        | Get Company Data              | Build Interactive Email      | Same as above                                                                                                             |
| Build Interactive Email         | Code                    | Builds color-coded interactive HTML email body           | Update Due Dates in Sheet     | Send via Gmail1              | Same as above                                                                                                             |
| Send via Gmail1                 | Gmail                   | Sends the interactive alert email                         | Build Interactive Email       | None                        | Same as above                                                                                                             |
| Webhook - Receive Confirmation Update | Webhook                 | Receives user confirmation clicks from email             | External HTTP request         | Process Webhook Data         | ## ✅ Email Response Handler … Handles email confirmation clicks, updates sheet, and confirms action to user                |
| Process Webhook Data            | Code                    | Processes webhook payload data                            | Webhook - Receive Confirmation Update | Update Google Sheet Database | Same as above                                                                                                             |
| Update Google Sheet Database    | Google Sheets           | Records confirmation status in Google Sheets              | Process Webhook Data          | Send Confirmation Page       | Same as above                                                                                                             |
| Send Confirmation Page          | Respond to Webhook      | Sends HTML confirmation page to user                      | Update Google Sheet Database  | None                        | Same as above                                                                                                             |
| Sticky Note2                   | Sticky Note             | Overview and purpose of entire workflow                   | None                          | None                        | 🎯 Accounting Alerts Automation … Saves 2-3 hours/week, color-coded alerts, interactive buttons, audit trail in Sheets      |
| Sticky Note3                   | Sticky Note             | Detailed explanation of daily deadline check and alert   | None                          | None                        | ⚙️ Daily Deadline Check & Alert System description                                                                         |
| Sticky Note4                   | Sticky Note             | Detailed explanation of email response webhook flow      | None                          | None                        | ✅ Email Response Handler (Webhook Flow) description                                                                      |
| Sticky Note5                   | Sticky Note             | Setup requirements and credentials overview               | None                          | None                        | 📋 Setup Requirements including Google Sheets structure and credentials                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node**  
   - Name: `Schedule Trigger1`  
   - Type: Schedule Trigger  
   - Set cron expression to `0 17 * * 1-5` (5 PM every weekday)  

3. **Add a Google Sheets node (Read)**  
   - Name: `Read Company Database`  
   - Operation: Read rows  
   - Connect input from `Schedule Trigger1`  
   - Configure with your Google Sheets OAuth2 credential  
   - Set Document ID to your company tracking Google Sheet  
   - Select the sheet/tab name where company data is stored (e.g., "Sheet2")  

4. **Add an HTTP Request node**  
   - Name: `Get Company Data`  
   - Operation: GET  
   - Connect input from `Read Company Database`  
   - URL: `https://api.company-information.service.gov.uk/company/{{ $json.company_number }}`  
   - Authentication: HTTP Basic Auth using your Companies House API key credential  

5. **Add a Google Sheets node (Append or Update)**  
   - Name: `Update Due Dates in Sheet`  
   - Operation: Append or Update  
   - Connect input from `Get Company Data`  
   - Configure with Google Sheets OAuth2 credential  
   - Set Document ID and Sheet Name same as above  
   - Define mapping columns to update fields such as `accounts_due`, `confirmation_due`, `confirmation_submitted`, `last_updated`, `company_number`, `company_name`  
   - Use `company_number` as the matching column  

6. **Add a Code node**  
   - Name: `Build Interactive Email`  
   - Connect input from `Update Due Dates in Sheet`  
   - Paste the provided JavaScript code to:  
     - Filter companies with names  
     - Sort by due date  
     - Calculate urgency colors and labels  
     - Build styled HTML table with interactive confirmation links pointing to your webhook URL (`https://your-n8n-instance/webhook/confirmation-updates`)  
   - Output JSON with `emailBody`, `subject`, and `totalCompanies` fields  

7. **Add a Gmail node**  
   - Name: `Send via Gmail1`  
   - Connect input from `Build Interactive Email`  
   - Configure Gmail OAuth2 credential  
   - Set recipient email address (e.g., your accounting team)  
   - Use expressions to set subject and HTML message body from previous node outputs  
   - Disable attribution for cleaner emails  

---

8. **Setup the Email Response Handler flow:**

9. **Add a Webhook node**  
   - Name: `Webhook - Receive Confirmation Update`  
   - HTTP Method: GET (default)  
   - Path: `confirmation-updates`  
   - Response Mode: Use response node (to send dynamic HTML confirmation page)  

10. **Add a Code node**  
    - Name: `Process Webhook Data`  
    - Connect input from the Webhook node  
    - Minimal processing (e.g., add or modify fields if needed)  

11. **Add a Google Sheets node (Update)**  
    - Name: `Update Google Sheet Database`  
    - Connect input from `Process Webhook Data`  
    - Configure with Google Sheets OAuth2 credential  
    - Document ID and Sheet Name as above  
    - Match rows by `company_number`  
    - Update `confirmation_submitted` field based on webhook query parameters  

12. **Add a Respond to Webhook node**  
    - Name: `Send Confirmation Page`  
    - Connect input from `Update Google Sheet Database`  
    - Respond with HTML content confirming the update, using placeholders for company name and confirmation status (as in provided code)  

---

13. **Connect nodes as follows:**  
- `Schedule Trigger1` → `Read Company Database` → `Get Company Data` → `Update Due Dates in Sheet` → `Build Interactive Email` → `Send via Gmail1`  
- `Webhook - Receive Confirmation Update` → `Process Webhook Data` → `Update Google Sheet Database` → `Send Confirmation Page`  

14. **Credentials to configure:**  
- Google Sheets OAuth2 with read/write access to the tracking sheet  
- Companies House API key (HTTP Basic Auth credential)  
- Gmail OAuth2 for sending emails  

15. **Set spreadsheet columns as required:**  
- `company_number`, `company_name` (manual entry)  
- `accounts_due`, `confirmation_due`, `confirmation_submitted`, `last_updated` (auto-updated by workflow)  

16. **Update the webhook URL in the email-building code to your n8n instance URL**  
- Replace `https://mega.srv967585.hstgr.cloud/webhook/confirmation-updates` with your actual webhook URL  

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                                   |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| 🎯 Accounting Alerts Automation: Saves 2-3 hours/week per firm by automating deadline tracking. | See Sticky Note2 for detailed purpose and workflow value description.                                            |
| ⚙️ Daily Deadline Check & Alert System runs every weekday at 5 PM, pulling live data from Companies House API. | See Sticky Note3.                                                                                                |
| ✅ Email Response Handler creates audit trail by updating confirmation status in Google Sheets and provides user feedback. | See Sticky Note4.                                                                                                |
| 📋 Setup Requirements: Google Sheets columns, credentials (Google Sheets OAuth2, Companies House API key, Gmail OAuth2). | See Sticky Note5.                                                                                                |
| Webhook URL must be updated in the email building code to match your n8n deployment's external URL. | Critical for confirmation links to work correctly.                                                               |
| Penalties for missed deadlines can range from £150 to £1,500 per company, emphasizing the workflow's business value. | Highlighted in Sticky Note2.                                                                                       |

---

**Disclaimer:**  
This text is exclusively derived from an n8n automation workflow. It complies strictly with current content policies and contains no illegal or protected elements. All data handled is legal and public.