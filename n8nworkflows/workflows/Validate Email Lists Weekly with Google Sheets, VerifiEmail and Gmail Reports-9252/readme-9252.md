Validate Email Lists Weekly with Google Sheets, VerifiEmail and Gmail Reports

https://n8nworkflows.xyz/workflows/validate-email-lists-weekly-with-google-sheets--verifiemail-and-gmail-reports-9252


# Validate Email Lists Weekly with Google Sheets, VerifiEmail and Gmail Reports

---

### 1. Workflow Overview

This workflow automates the weekly validation and hygiene of email lists stored in Google Sheets. It is designed to run every Friday at 5 PM, validating potentially thousands of email addresses to improve deliverability, reduce bounce rates, and ensure compliance with email sending best practices.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling and Input Reception:** Triggers the workflow on a weekly schedule and reads the email list from a Google Sheet.
- **1.2 Loop Processing and Email Validation:** Processes each email individually in a loop, validating via VerifiEmail API and branching logic based on validation results.
- **1.3 Updating Google Sheets:** Updates the original sheet with validation status, timestamps, and notes based on validation outcomes.
- **1.4 Aggregation and Statistics Calculation:** After processing all emails, calculates detailed statistics and a health score of the email list.
- **1.5 Reporting:** Sends a professional, richly formatted HTML email report summarizing the validation results.
- **1.6 Workflow Control and Loop Management:** Uses a Merge node to orchestrate looping until all emails are processed.

This structured design enables scalable, automated email hygiene with actionable insights delivered to stakeholders.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduling and Input Reception

- **Overview:**  
  This block triggers the workflow weekly and reads the email data from a Google Sheet for validation.

- **Nodes Involved:**  
  - Weekly Schedule (Friday 5PM)  
  - Read Email List

- **Node Details:**

1. **Weekly Schedule (Friday 5PM)**  
   - **Type:** Schedule Trigger  
   - **Role:** Initiates the workflow every Friday at 5:00 PM using a cron expression `0 17 * * 5`.  
   - **Configuration:** Cron schedule set for weekly trigger.  
   - **Inputs:** None (trigger node).  
   - **Outputs:** Triggers the Google Sheets read node.  
   - **Edge Cases:** Workflow will not run if the n8n instance is offline at trigger time.  
   - **Version:** 1.2  

2. **Read Email List**  
   - **Type:** Google Sheets  
   - **Role:** Reads entire email list from the specified Google Sheet tab.  
   - **Configuration:** Reads from `gid=0` (default sheet). Columns expected include `row_number`, `name`, `email`, `status`, `checked_at`, `notes`.  
   - **Credentials:** Google Sheets OAuth2 required.  
   - **Inputs:** Trigger from schedule node.  
   - **Outputs:** Sends all rows as array to SplitInBatches node.  
   - **Edge Cases:**  
     - Sheet must have exact column headers (case-sensitive).  
     - Empty or malformed sheets may cause no items to process.  
     - API quota or auth errors possible if credentials invalid or expired.  
   - **Version:** 4.7

---

#### 2.2 Loop Processing and Email Validation

- **Overview:**  
  Processes each email individually in batches, validates using VerifiEmail API, and branches logic based on validation results.

- **Nodes Involved:**  
  - Process Each Email (SplitInBatches)  
  - Validate Email Address (VerifiEmail)  
  - Check Validation Result (IF)  
  - Process Valid Email (Code)  
  - Process Invalid Email (Code)

- **Node Details:**

1. **Process Each Email**  
   - **Type:** SplitInBatches  
   - **Role:** Breaks the input list into individual emails (batch size 1) for sequential processing.  
   - **Configuration:** Default batch size (1).  
   - **Inputs:** From Read Email List.  
   - **Outputs:** Two outputs:  
       - Main (for looping to Validate Email Address)  
       - Second output for statistics collection after loop completion.  
   - **Edge Cases:** Large lists may take time; API rate limits apply.  
   - **Version:** 3

2. **Validate Email Address**  
   - **Type:** VerifiEmail API Node  
   - **Role:** Sends email address to VerifiEmail API to verify deliverability and format.  
   - **Configuration:** Uses expression to pass current email (`{{$json.email}}`).  
   - **Credentials:** VerifiEmail API key required.  
   - **Inputs:** Single email from SplitInBatches.  
   - **Outputs:** Validation result JSON with fields like `valid`, and detailed flags.  
   - **Edge Cases:**  
     - API quota exceeded or invalid API key.  
     - Timeout or network errors.  
     - Invalid input data causing API errors.  
   - **Version:** 1

3. **Check Validation Result**  
   - **Type:** IF Node  
   - **Role:** Branches workflow into two paths based on validation `valid` boolean.  
   - **Configuration:** Condition: `{{$json.valid}} === true` → TRUE branch, else FALSE branch.  
   - **Inputs:** Validation result from VerifiEmail node.  
   - **Outputs:**  
       - TRUE: Valid email processing.  
       - FALSE: Invalid email processing.  
   - **Edge Cases:** Unexpected or missing fields could cause condition to fail.  
   - **Version:** 2.2

4. **Process Valid Email**  
   - **Type:** Code (JavaScript)  
   - **Role:** Processes valid emails, checks for disposable flag to mark as risky if needed, and prepares output object.  
   - **Configuration:**  
     - Sets status as "valid" or "risky" (if disposable).  
     - Adds notes and timestamp in localized format.  
     - Merges original row data with validation results.  
   - **Inputs:** TRUE branch from IF node, plus access to original loop data.  
   - **Outputs:** Object with validation status and metadata for sheet update.  
   - **Edge Cases:** Missing flags or unexpected data formats.  
   - **Version:** 2

5. **Process Invalid Email**  
   - **Type:** Code (JavaScript)  
   - **Role:** Processes invalid emails assigning specific reasons based on validation details and returns structured object.  
   - **Configuration:**  
     - Checks failure reasons: MX record, format, spoofing.  
     - Sets status "invalid" and appropriate notes.  
     - Adds localized timestamp.  
     - Merges original data for updating.  
   - **Inputs:** FALSE branch from IF node, original loop data.  
   - **Outputs:** Structured invalid email data for sheet update.  
   - **Edge Cases:** Incomplete validation detail fields.  
   - **Version:** 2

---

#### 2.3 Updating Google Sheets

- **Overview:**  
  Updates the original Google Sheet rows with validation results separately for valid and invalid emails.

- **Nodes Involved:**  
  - Update Valid Status  
  - Update Invalid Status

- **Node Details:**

1. **Update Valid Status**  
   - **Type:** Google Sheets (Update Operation)  
   - **Role:** Updates rows for valid or risky emails with status, notes, and timestamp.  
   - **Configuration:**  
     - Matches rows using `row_number`.  
     - Updates columns: `name`, `email`, `status`, `checked_at`, `notes`.  
     - Uses values from the Process Valid Email node output.  
   - **Credentials:** Google Sheets OAuth2 (possibly different account than Read node).  
   - **Inputs:** From Process Valid Email.  
   - **Outputs:** To Merge node.  
   - **Edge Cases:** If row_number missing or sheet locked, update may fail.  
   - **Version:** 4.7

2. **Update Invalid Status**  
   - **Type:** Google Sheets (Update Operation)  
   - **Role:** Updates rows for invalid emails similarly.  
   - **Configuration:** Matches and updates same columns as valid update node.  
   - **Credentials:** Same as Update Valid Status.  
   - **Inputs:** From Process Invalid Email.  
   - **Outputs:** To Merge node.  
   - **Edge Cases:** Same as above.  
   - **Version:** 4.7

---

#### 2.4 Workflow Control and Loop Management

- **Overview:**  
  The Merge node combines valid and invalid processing branches and loops the flow back to process all emails sequentially. When all emails processed, outputs accumulated data.

- **Nodes Involved:**  
  - Merge1

- **Node Details:**

1. **Merge1**  
   - **Type:** Merge  
   - **Role:** Consolidates outputs from both update nodes, feeds data back to the SplitInBatches node to continue processing the next email.  
   - **Configuration:** Default merge mode, inputs connected as:  
     - Input 1: From Update Valid Status  
     - Input 2: From Update Invalid Status  
   - **Inputs:** Two from update nodes.  
   - **Outputs:** Back to SplitInBatches main input for looping, and after all processing completes, triggers Calculate Statistics node.  
   - **Edge Cases:** Missing input on one branch causes merge to wait indefinitely. Both branches must complete each loop iteration.  
   - **Version:** 3.2

---

#### 2.5 Aggregation and Statistics Calculation

- **Overview:**  
  After all emails are processed, this block calculates summary statistics and an overall health score of the email list.

- **Nodes Involved:**  
  - Calculate Statistics

- **Node Details:**

1. **Calculate Statistics**  
   - **Type:** Code (JavaScript)  
   - **Role:** Aggregates all processed email data to compute counts and percentages of valid, invalid, risky, and unknown statuses. Also calculates a health score between 0 and 100 based on weighted formula.  
   - **Configuration:**  
     - Counts total emails and categorizes by status.  
     - Calculates percentages with zero division guard.  
     - Health score formula:  
       `(valid% * 100) - (invalid% * 20) - (risky% * 10)` capped between 0 and 100.  
     - Assigns health status label and color based on score thresholds.  
     - Prepares summary text and detailed categorized email lists.  
   - **Inputs:** Accumulated array of all processed emails from Merge node.  
   - **Outputs:** Statistics object used for report generation.  
   - **Edge Cases:** Empty lists yield zero percentages and health score zero.  
   - **Version:** 2

---

#### 2.6 Reporting

- **Overview:**  
  Sends a comprehensive, branded HTML email report summarizing the validation results and statistics.

- **Nodes Involved:**  
  - Send Weekly Report

- **Node Details:**

1. **Send Weekly Report**  
   - **Type:** Gmail (Send Email)  
   - **Role:** Sends an HTML email report to configured recipients with validation summary and detailed insights.  
   - **Configuration:**  
     - Recipient email set via parameter (default: marketing.manager@company.com).  
     - Subject includes current date dynamically.  
     - HTML body contains:  
       - Color-coded health score badge with status.  
       - Stat cards for valid, invalid, risky, and total emails.  
       - Action items for flagged emails.  
       - List of invalid emails with reasons.  
       - Link to Google Sheet report.  
       - Campaign impact insights and branding.  
       - Footer with automated system note and timestamp.  
   - **Credentials:** Gmail OAuth2 required.  
   - **Inputs:** Statistics object from Calculate Statistics node.  
   - **Outputs:** None (final action).  
   - **Edge Cases:**  
     - Email delivery issues (spam folder, wrong recipient).  
     - Gmail API quota or auth errors.  
   - **Version:** 2.1

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                          | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                                      |
|-------------------------|---------------------------|----------------------------------------|---------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------|
| Weekly Schedule (Friday 5PM) | Schedule Trigger           | Triggers workflow weekly at Friday 5PM| None                      | Read Email List             |                                                                                                                                         |
| Read Email List         | Google Sheets              | Reads email list rows from Google Sheet| Weekly Schedule           | Process Each Email          | Setup instructions and column requirements (Sticky Note1)                                                       |
| Process Each Email      | SplitInBatches             | Processes emails one by one in loop    | Read Email List            | Validate Email Address, Calculate Statistics | Explanation of email validation process (Sticky Note2)                                                        |
| Validate Email Address  | VerifiEmail API            | Validates email via external API       | Process Each Email         | Check Validation Result     |                                                                                                                                         |
| Check Validation Result | IF                         | Branches flow based on validation result| Validate Email Address     | Process Valid Email, Process Invalid Email | True/False branching explanation (Sticky Note3)                                                                 |
| Process Valid Email     | Code                      | Prepares valid email data output       | Check Validation Result    | Update Valid Status         |                                                                                                                                         |
| Process Invalid Email   | Code                      | Prepares invalid email data output     | Check Validation Result    | Update Invalid Status       |                                                                                                                                         |
| Update Valid Status     | Google Sheets (Update)     | Updates sheet rows for valid emails    | Process Valid Email        | Merge1                     |                                                                                                                                         |
| Update Invalid Status   | Google Sheets (Update)     | Updates sheet rows for invalid emails  | Process Invalid Email      | Merge1                     |                                                                                                                                         |
| Merge1                  | Merge                     | Combines valid and invalid branches, loops| Update Valid Status, Update Invalid Status | Process Each Email (loop), Calculate Statistics | Merge & loop completion explanation (Sticky Note4)                                                              |
| Calculate Statistics    | Code                      | Calculates summary statistics & health score| Merge1                    | Send Weekly Report          | Analytics & reporting explanation (Sticky Note5)                                                                |
| Send Weekly Report      | Gmail                     | Sends HTML email report with stats     | Calculate Statistics       | None                       |                                                                                                                                         |
| Sticky Note             | Sticky Note               | Workflow overview and benefits summary | None                      | None                       | Entire workflow description and benefits (Sticky Note)                                                          |
| Sticky Note1            | Sticky Note               | Setup instructions and sheet config    | None                      | None                       | Setup required before activation (Sticky Note1)                                                                 |
| Sticky Note2            | Sticky Note               | Email validation process explanation    | None                      | None                       | Email validation steps and processing logic (Sticky Note2)                                                      |
| Sticky Note3            | Sticky Note               | Branching logic description             | None                      | None                       | True/False branching explanation (Sticky Note3)                                                                 |
| Sticky Note4            | Sticky Note               | Merge node and loop explanation         | None                      | None                       | Merge & loop control details (Sticky Note4)                                                                      |
| Sticky Note5            | Sticky Note               | Analytics and reporting explanation     | None                      | None                       | Statistics calculation and reporting overview (Sticky Note5)                                                    |
| Sticky Note6            | Sticky Note               | Customization options                    | None                      | None                       | How to customize schedule, recipients, add Slack, archive invalid emails, rate limiting, email design (Sticky Note6) |
| Sticky Note7            | Sticky Note               | Troubleshooting guide                    | None                      | None                       | Common errors and fixes for sheet, loop, statistics, email delivery, validation API, merge node (Sticky Note7)  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Set to cron expression `0 17 * * 5` (every Friday at 5 PM).  
   - Name it "Weekly Schedule (Friday 5PM)".

3. **Add a Google Sheets node to read data:**  
   - Name: "Read Email List".  
   - Configure credentials with Google Sheets OAuth2.  
   - Set Document ID to your Google Sheet containing the email list.  
   - Set Sheet Name to the sheet tab ID (`gid=0`).  
   - Ensure the sheet has columns: `row_number`, `name`, `email`, `status`, `checked_at`, `notes`.  
   - Connect "Weekly Schedule" output to this node.

4. **Add SplitInBatches node:**  
   - Name: "Process Each Email".  
   - Batch size: 1 (default).  
   - Connect "Read Email List" output to this node.

5. **Add VerifiEmail node:**  
   - Name: "Validate Email Address".  
   - Configure VerifiEmail API credentials.  
   - Set email parameter to expression: `{{$json.email}}`.  
   - Connect "Process Each Email" output to this node.

6. **Add IF node:**  
   - Name: "Check Validation Result".  
   - Condition: `{{$json.valid}} === true` (boolean check).  
   - Connect "Validate Email Address" output to this node.

7. **Add Code node for valid emails:**  
   - Name: "Process Valid Email".  
   - JavaScript code to set status "valid" or "risky" if disposable, add notes, timestamp, and merge with original data.  
   - Use `$input.first().json` for validation result and access original loop data.  
   - Connect TRUE branch of IF node to this node.

8. **Add Code node for invalid emails:**  
   - Name: "Process Invalid Email".  
   - JavaScript code to set status "invalid" with specific reasons (no MX, invalid format, spoof), timestamp, merge with original data.  
   - Connect FALSE branch of IF node to this node.

9. **Add two Google Sheets nodes to update rows:**  
   - Name one "Update Valid Status" and the other "Update Invalid Status".  
   - Both configured to update rows by matching `row_number`.  
   - Set columns to update: `name`, `email`, `status`, `checked_at`, `notes`.  
   - Configure Google Sheets OAuth2 credentials (can be same or different).  
   - Connect "Process Valid Email" to "Update Valid Status".  
   - Connect "Process Invalid Email" to "Update Invalid Status".

10. **Add Merge node:**  
    - Name: "Merge1".  
    - Connect "Update Valid Status" to Merge input 1.  
    - Connect "Update Invalid Status" to Merge input 2.

11. **Connect Merge output back to "Process Each Email" node:**  
    - This creates the loop for processing the next batch/email.  

12. **Add Code node to calculate statistics:**  
    - Name: "Calculate Statistics".  
    - Configure JavaScript to receive accumulated items, count each status, calculate percentages, compute health score, create summary and email details.  
    - Connect Merge node's "done" output (after all batches processed) to this node.

13. **Add Gmail node to send report:**  
    - Name: "Send Weekly Report".  
    - Configure Gmail OAuth2 credentials.  
    - Set recipient email (e.g., marketing.manager@company.com).  
    - Subject: dynamic with date (e.g., `📧 Email Hygiene Report - {{ $now.format("MMM DD, YYYY") }}`).  
    - Set message body to the detailed HTML template using statistics data bindings (health score, counts, lists, summary).  
    - Connect "Calculate Statistics" output to this node.

14. **Activate the workflow and test:**  
    - Add test emails to Google Sheet.  
    - Run the workflow manually.  
    - Verify Google Sheet updates and email delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow operates fully automated every Friday at 5 PM to validate email lists, improving deliverability and compliance. It uses VerifiEmail API for validation and sends rich HTML reports via Gmail.                                                                                                                                                                                       | Workflow overview (Sticky Note)                         |
| Setup requires a Google Sheet with exact columns: `row_number`, `name`, `email`, `status`, `checked_at`, `notes`. OAuth2 credentials for Google Sheets, VerifiEmail, and Gmail are mandatory.                                                                                                                                                                                                    | Setup instructions (Sticky Note1)                       |
| Validation checks include format compliance, MX record presence, SMTP mailbox existence, disposable email detection, and catch-all domain identification.                                                                                                                                                                                                                                       | Validation process details (Sticky Note2)               |
| Branching is done with an IF node on the `valid` property; valid and invalid emails are processed differently and updated accordingly in the sheet.                                                                                                                                                                                                                                            | Branching logic (Sticky Note3)                          |
| The Merge node is central to loop control, combining both branches and feeding data back to process all emails sequentially. The "done" output triggers final statistics calculation and reporting.                                                                                                                                                                                            | Merge & loop explanation (Sticky Note4)                 |
| Statistics calculation includes counts, percentages, and a health score formula with color-coded statuses for quick assessment of list quality.                                                                                                                                                                                                                                                | Analytics & reporting (Sticky Note5)                    |
| Customization options include schedule changes, multiple email recipients, Slack notifications, archiving invalid emails, CRM exports, rate limiting, and email report design customization.                                                                                                                                                                                                 | Customization tips (Sticky Note6)                       |
| Troubleshooting common issues such as column mismatches, loop behavior, statistics accuracy, email delivery failures, and API errors with fixes and debugging tips.                                                                                                                                                                                                                              | Troubleshooting guide (Sticky Note7)                    |
| VerifiEmail API documentation and account setup available at https://verifi.email. Gmail OAuth2 credentials must be authorized with the proper scopes to send emails on behalf of the user.                                                                                                                                                                                                    | External API resources                                  |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---