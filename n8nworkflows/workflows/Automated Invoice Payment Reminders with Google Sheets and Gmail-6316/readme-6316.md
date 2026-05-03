Automated Invoice Payment Reminders with Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/automated-invoice-payment-reminders-with-google-sheets-and-gmail-6316


# Automated Invoice Payment Reminders with Google Sheets and Gmail

---

### 1. Workflow Overview

The **Automated Invoice Payment Reminders with Google Sheets and Gmail** workflow automates the process of monitoring invoice due dates and sending timely reminder emails to clients. It targets small businesses, freelancers, and agencies that need to improve cash flow by reducing delayed invoice payments.

The workflow is organized into the following logical blocks:

- **1.1 Daily Trigger:** Automatically initiates the workflow once per day.
- **1.2 Invoice Data Retrieval:** Reads all invoice records from a specified Google Sheet.
- **1.3 Filtering and Preparation:** Filters invoices that are due soon or overdue and prepares personalized email content.
- **1.4 Conditional Execution:** Determines if there are any invoices to remind before proceeding.
- **1.5 Email Dispatch:** Sends personalized reminder emails to clients via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Daily Trigger

- **Overview:**  
  This block triggers the workflow daily to perform the invoice checking and reminder process automatically without manual intervention.

- **Nodes Involved:**  
  - *1. Daily Schedule Trigger*

- **Node Details:**

  - **Node Name:** 1. Daily Schedule Trigger  
  - **Type & Role:** Schedule Trigger node — triggers workflow execution on a recurring schedule.  
  - **Configuration:**  
    - Interval set to run once every day (default daily trigger, can be adjusted).  
  - **Expressions/Variables:** None.  
  - **Input:** None (start trigger).  
  - **Output:** Initiates flow to "2. Read Invoice Data (Google Sheets)".  
  - **Version:** v1.  
  - **Potential Failures:** Misconfigured schedule interval, time zone mismatches.  
  - **Notes:** Ensure time zone and interval fit your operational needs.

---

#### 1.2 Invoice Data Retrieval

- **Overview:**  
  Reads the invoice data from a Google Sheet, retrieving all necessary invoice details such as InvoiceID, ClientName, ClientEmail, Amount, DueDate, and Status.

- **Nodes Involved:**  
  - *2. Read Invoice Data (Google Sheets)*

- **Node Details:**

  - **Node Name:** 2. Read Invoice Data (Google Sheets)  
  - **Type & Role:** Google Sheets node — reads data from a specified sheet range.  
  - **Configuration:**  
    - Sheet ID set to the target Google Sheet containing invoices (replace `YOUR_GOOGLE_SHEET_ID`).  
    - Range: `Invoices!A:F` (assumes columns A to F hold invoice data).  
  - **Expressions/Variables:** None.  
  - **Input:** From schedule trigger.  
  - **Output:** Passes invoice rows to the next function node.  
  - **Version:** v2 (requires OAuth2 credentials).  
  - **Potential Failures:** Authentication errors (invalid/expired Google OAuth credentials), incorrect sheet ID or range, missing or incorrectly formatted data in the sheet.  
  - **Credentials:** Requires valid Google Sheets OAuth2 credentials.  
  - **Notes:** The Google Sheet must have exact column headers: `InvoiceID`, `ClientName`, `ClientEmail`, `Amount`, `DueDate`, `Status`.

---

#### 1.3 Filtering and Preparation

- **Overview:**  
  Processes each invoice record, filtering only those needing reminders (due within 3 days or overdue up to 7 days). It skips paid invoices and prepares personalized email subjects and bodies.

- **Nodes Involved:**  
  - *3. Filter & Prepare Reminders*

- **Node Details:**

  - **Node Name:** 3. Filter & Prepare Reminders  
  - **Type & Role:** Function node — contains JavaScript code to filter and prepare reminder data.  
  - **Configuration:**  
    - Defines `remindBeforeDays = 3` and `remindAfterDays = 7`.  
    - Normalizes dates to start of day for accurate comparisons.  
    - Checks if invoice status is paid and skips such records.  
    - Constructs email subject and body dynamically for two cases: "due soon" and "overdue".  
  - **Expressions/Variables:**  
    - Uses `items` input from Google Sheets node.  
    - Accesses fields like `InvoiceID`, `ClientName`, `ClientEmail`, `Amount`, `DueDate`, `Status`.  
  - **Input:** Invoice records from Google Sheets.  
  - **Output:** Array of filtered invoices with prepared email details.  
  - **Version:** v1.  
  - **Potential Failures:**  
    - Date parsing errors if `DueDate` is in an unexpected format.  
    - Missing critical fields (InvoiceID, ClientEmail, Amount, DueDate).  
    - Logic errors if Google Sheet column headers differ from expected names.  
  - **Notes:**  
    - Modify field names and reminder windows to suit your data structure and business rules.  
    - Logs warnings for invalid dates.

---

#### 1.4 Conditional Execution

- **Overview:**  
  Checks if the filtering step found any invoices to remind. If none, the workflow ends gracefully; otherwise, it proceeds to send emails.

- **Nodes Involved:**  
  - *4. If Invoices to Remind?*

- **Node Details:**

  - **Node Name:** 4. If Invoices to Remind?  
  - **Type & Role:** If node — evaluates whether filtered invoice list is non-empty.  
  - **Configuration:**  
    - Default condition: checks if input items length > 0.  
  - **Expressions/Variables:** Evaluates length of incoming items (`$items().length`).  
  - **Input:** Filtered invoices from Function node.  
  - **Output:**  
    - **True:** Connects to "5. Send Invoice Reminder (Gmail)".  
    - **False:** No further action.  
  - **Version:** v1.  
  - **Potential Failures:** None critical, but if misconfigured, may skip sending emails incorrectly.

---

#### 1.5 Email Dispatch

- **Overview:**  
  Sends personalized invoice reminder emails to clients using Gmail, with dynamic subjects and message bodies.

- **Nodes Involved:**  
  - *5. Send Invoice Reminder (Gmail)*

- **Node Details:**

  - **Node Name:** 5. Send Invoice Reminder (Gmail)  
  - **Type & Role:** Gmail node — sends emails via authenticated Gmail account.  
  - **Configuration:**  
    - Subject: dynamically set as `={{ $json.subject }}` from the function node.  
    - Body: uses dynamic HTML or plain text from previous node (`$json.body`).  
  - **Expressions/Variables:** Uses dynamic expressions to personalize emails.  
  - **Input:** Filtered invoice reminders from If node.  
  - **Output:** Sends email, outputs send status.  
  - **Version:** v1.  
  - **Potential Failures:**  
    - Authentication issues with Gmail OAuth2 credentials.  
    - Gmail API quota limits.  
    - Email sending errors due to invalid recipient addresses.  
  - **Credentials:** Requires Gmail OAuth2 credentials.  
  - **Notes:**  
    - Customize the email template here if richer formatting is desired.

---

### 3. Summary Table

| Node Name                   | Node Type                 | Functional Role                      | Input Node(s)                     | Output Node(s)                 | Sticky Note                                                                                          |
|-----------------------------|---------------------------|-----------------------------------|----------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------|
| 1. Daily Schedule Trigger    | Schedule Trigger          | Initiates workflow daily           | None                             | 2. Read Invoice Data (Google Sheets) |                                                                                                    |
| 2. Read Invoice Data (Google Sheets) | Google Sheets           | Reads invoice data from sheet      | 1. Daily Schedule Trigger        | 3. Filter & Prepare Reminders  | Reads invoice details. Ensure columns: InvoiceID, ClientName, ClientEmail, Amount, DueDate, Status. |
| 3. Filter & Prepare Reminders| Function                  | Filters invoices & prepares email  | 2. Read Invoice Data             | 4. If Invoices to Remind?      | Filters invoices due soon or overdue; prepares personalized email content.                          |
| 4. If Invoices to Remind?    | If                        | Checks if reminders needed         | 3. Filter & Prepare Reminders    | 5. Send Invoice Reminder (Gmail) / None |                                                                                                    |
| 5. Send Invoice Reminder (Gmail) | Gmail                     | Sends reminder emails              | 4. If Invoices to Remind? (true) | None                          | Sends personalized invoice reminder emails via Gmail.                                             |
| Sticky Note                 | Sticky Note               | Documentation and overview         | None                             | None                          | ## Flow                                                                                           |
| Sticky Note1                | Sticky Note               | Detailed workflow documentation    | None                             | None                          | See detailed notes in section 5 below.                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Workflow:**  
   Open n8n, create a new workflow named "Automated Invoice Reminder".

2. **Add Node: Daily Schedule Trigger**  
   - Type: Schedule Trigger  
   - Configure to run once every day at your preferred time (adjust interval and timezone in parameters).  
   - No input connections (start node).

3. **Add Node: Read Invoice Data (Google Sheets)**  
   - Type: Google Sheets  
   - Connect from "Daily Schedule Trigger".  
   - Set `Sheet ID` to your Google Sheet ID containing invoice data.  
   - Set `Range` to `Invoices!A:F` or the range covering your data.  
   - Select or create Google Sheets OAuth2 credentials with access to your sheet.  
   - Ensure your sheet columns include: `InvoiceID`, `ClientName`, `ClientEmail`, `Amount`, `DueDate`, and `Status`.

4. **Add Node: Filter & Prepare Reminders (Function)**  
   - Type: Function  
   - Connect from "Read Invoice Data".  
   - Paste the JavaScript code to:  
     - Normalize the current date and due dates.  
     - Filter invoices that are due soon (within 3 days) or overdue (up to 7 days).  
     - Skip invoices with status "Paid".  
     - Prepare email subject and body for each filtered invoice.  
   - Adjust variable names and reminder windows as needed to match your data.

5. **Add Node: If Invoices to Remind? (If)**  
   - Type: If node  
   - Connect from "Filter & Prepare Reminders".  
   - Condition: Check if the number of items (`$items().length`) > 0 (to detect invoices needing reminders).  
   - True branch proceeds to email sending. False branch ends workflow.

6. **Add Node: Send Invoice Reminder (Gmail)**  
   - Type: Gmail  
   - Connect from the true output of the If node.  
   - Set email `Subject` to `={{ $json.subject }}`.  
   - Set email `Body` (HTML or plain text) to `={{ $json.body }}`.  
   - Use Gmail OAuth2 credentials with permission to send emails.  
   - Set the "From Email" address as required.

7. **Activate the Workflow:**  
   - Save the workflow.  
   - Toggle the workflow to active to enable daily execution.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| This workflow addresses the common problem of delayed invoice payments by automating reminders, improving cash flow, and freeing up administrative resources. It is suitable for freelancers, SMBs, and agencies. The workflow relies on clean, well-structured Google Sheets data and properly configured credentials for Google Sheets and Gmail. Adjust date formats and column headers in the code if your source data differs. Customization of reminder messages and scheduling is straightforward within the Function and Schedule Trigger nodes.                                                                                                                                                                                                                                                                                                                                                                                                                | Detailed documentation included in the workflow’s sticky note titled "Sticky Note1".              |
| For best results, ensure the Google Sheet uses ISO date formats (YYYY-MM-DD) or formats that JavaScript Date can parse reliably to avoid date parsing errors in the Function node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | General best practice for date handling in automation workflows.                                  |
| Gmail API enforces sending quotas and rate limits; monitor usage to avoid email delivery failures. Use dedicated service accounts or SMTP providers for higher volume needs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Gmail API documentation and quota guidelines: https://developers.google.com/gmail/api/guides/push |
| n8n Credential Setup: Google Sheets OAuth2 and Gmail OAuth2 credentials must be configured with appropriate permissions and scopes. Ensure tokens are valid and refreshed as needed to avoid authentication failures.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | n8n docs on credentials: https://docs.n8n.io/credentials/google-sheets/ and https://docs.n8n.io/credentials/gmail/ |
| Troubleshooting Tips: Check n8n execution logs for errors, verify data integrity in Google Sheets, and test the Function node logic independently with sample data to debug filtering issues.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Troubleshooting guidance included in workflow sticky notes.                                      |

---

**Disclaimer:** The text provided is exclusively derived from an n8n automated workflow. This content adheres strictly to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---