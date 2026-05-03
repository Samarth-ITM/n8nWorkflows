Automated Multi-tier Expense Approval & Reimbursement with JotForm, Slack & Google Sheets

https://n8nworkflows.xyz/workflows/automated-multi-tier-expense-approval---reimbursement-with-jotform--slack---google-sheets-9600


# Automated Multi-tier Expense Approval & Reimbursement with JotForm, Slack & Google Sheets

### 1. Workflow Overview

This workflow automates the multi-tier expense approval and reimbursement process for an organization by integrating JotForm, Slack, Google Sheets, and Gmail. It captures employee-submitted expense reports via a JotForm form, validates them against company policy, routes them for appropriate approval based on amount and category, logs all actions in Google Sheets, and sends notification emails accordingly.

Logical blocks organized by function and data flow:

- **1.1 Input Reception:** Captures new expense submissions from JotForm.
- **1.2 Data Parsing & Normalization:** Extracts and normalizes key data fields from the form submission.
- **1.3 Policy Validation:** Checks expense details against company spending policies.
- **1.4 Compliance Check:** Determines if the expense violates policies and routes accordingly.
- **1.5 Approval Routing:** Routes expenses for auto-approval, manager, or director approval based on thresholds.
- **1.6 Notification & Logging:** Sends Slack notifications for approvals, logs expense data to Google Sheets, and sends approval/rejection emails to employees.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for new expense form submissions via JotForm webhook to initiate the workflow.

- **Nodes Involved:**  
  - JotForm Trigger

- **Node Details:**  
  - **JotForm Trigger**  
    - Type: Trigger node (Webhook listener)  
    - Configuration: Listens for submissions on form ID `252860582136459` with webhook ID `expense-submission`.  
    - Credentials: Authenticated with a JotForm API account.  
    - Inputs: Incoming HTTP webhook request from JotForm.  
    - Outputs: JSON object containing raw form submission data.  
    - Edge cases: Network failures, JotForm API downtime, malformed submissions.  
    - Sticky Note: Provides instructions and a link to create JotForm forms ([JotForm](https://www.jotform.com/?partner=mediajade)).

#### 1.2 Data Parsing & Normalization

- **Overview:**  
  Extracts essential expense details from the raw form data and normalizes field names and formats for downstream processing.

- **Nodes Involved:**  
  - Parse Form Data

- **Node Details:**  
  - **Parse Form Data**  
    - Type: Code node (JavaScript)  
    - Configuration: Extracts fields like submission ID, employee name/email/ID, amount, category, merchant, date, description, receipt URL; uses alternate keys if some are missing; parses amount as float; adds submission timestamp.  
    - Key expressions: Uses `$input.first().json` to access data; fallback field names for backward compatibility.  
    - Inputs: Raw JSON from JotForm Trigger.  
    - Outputs: Cleaned and normalized JSON object.  
    - Edge cases: Missing fields, invalid number parsing, inconsistent field names.

#### 1.3 Policy Validation

- **Overview:**  
  Validates the expense against company policies including allowed categories, maximum amounts, and approval thresholds.

- **Nodes Involved:**  
  - Validate Policy

- **Node Details:**  
  - **Validate Policy**  
    - Type: Code node (JavaScript)  
    - Configuration: Defines policy rules (max amounts per category, approved categories, auto/manager/director approval thresholds); checks if category is approved; checks amount limits; determines approval requirement level (`auto`, `manager`, `director`); flags policy violations; sets compliance status and initial workflow status to `pending`.  
    - Key expressions: Uses JavaScript logic to inspect data fields and return enriched JSON.  
    - Inputs: Parsed form data.  
    - Outputs: JSON enriched with compliance info and approval routing.  
    - Edge cases: Unknown categories, missing policy definitions, unexpected data types.

#### 1.4 Compliance Check

- **Overview:**  
  Checks if policy violations exist; if yes, flags the expense as rejected; if no, routes the expense to approval routing.

- **Nodes Involved:**  
  - Check Violations  
  - Set Rejection

- **Node Details:**  
  - **Check Violations**  
    - Type: If node  
    - Configuration: Condition checks if `policyViolations.length === 0` (no violations).  
    - Inputs: Policy validation output.  
    - Outputs:  
      - True (no violations): routes to approval routing.  
      - False: routes to rejection handling.  
    - Edge cases: Unexpected data structure, missing `policyViolations` field.

  - **Set Rejection**  
    - Type: Set node  
    - Configuration: Sets `status` to `rejected` and `rejectionReason` to a concatenated string of policy violations.  
    - Inputs: Policy validation data with violations.  
    - Outputs: JSON updated with rejection information.  
    - Edge cases: Empty violation list (should not occur here).

#### 1.5 Approval Routing

- **Overview:**  
  Routes expenses based on required approval level: auto-approved, manager approval, or director approval.

- **Nodes Involved:**  
  - Route Auto  
  - Auto Approve  
  - Route Manager  
  - Slack Manager  
  - Slack Director

- **Node Details:**  
  - **Route Auto**  
    - Type: If node  
    - Configuration: Checks if `requiresApproval === 'auto'`.  
    - Outputs:  
      - True: routes to Auto Approve node.  
      - False: routes to Route Manager node.  
    - Edge cases: Missing or unexpected `requiresApproval` values.

  - **Auto Approve**  
    - Type: Set node  
    - Configuration: Sets `status` to `approved`, `approvedBy` to `"System (Auto-Approved)"`, and `approvedAt` to current ISO timestamp.  
    - Outputs: Updated JSON for approved auto-routed expenses.

  - **Route Manager**  
    - Type: If node  
    - Configuration: Checks if `requiresApproval === 'manager'`.  
    - Outputs:  
      - True: sends Slack notification to manager channel.  
      - False: sends Slack notification to director channel.  
    - Edge cases: Missing or unexpected `requiresApproval` values.

  - **Slack Manager**  
    - Type: Slack node  
    - Configuration: Sends message to Slack channel ID `C01234567` about manager approval needed, including employee, amount, and category details.  
    - Credentials: Slack API account.  
    - Outputs: Passes data to logging.

  - **Slack Director**  
    - Type: Slack node  
    - Configuration: Sends message to Slack channel ID `C98765432` about director approval needed for high-value expense, including employee, amount, and category details.  
    - Credentials: Slack API account.  
    - Outputs: Passes data to logging.

#### 1.6 Notification & Logging

- **Overview:**  
  Logs the expense data and status to Google Sheets and sends an email notification to the employee based on final approval status.

- **Nodes Involved:**  
  - Log to Sheets  
  - Is Approved  
  - Approved (Email)  
  - Rejection Email

- **Node Details:**  
  - **Log to Sheets**  
    - Type: Google Sheets node  
    - Configuration: Appends or updates expense record in Google Sheet named "Sheet1" within a specified document ID (`YOUR_GOOGLE_SHEET_ID` placeholder). Uses `submissionId` as matching column, auto-maps input data.  
    - Credentials: Google Sheets OAuth2 account.  
    - Inputs: Expense data after approval or rejection handling.  
    - Outputs: Passes data to approval email decision.

  - **Is Approved**  
    - Type: If node  
    - Configuration: Checks if `status === 'approved'`.  
    - Outputs:  
      - True: routes to approved email notification.  
      - False: routes to rejection email notification.

  - **Approved (Email)**  
    - Type: Gmail node  
    - Configuration: Sends approval email to employee’s email address with subject and message confirming the approved amount.  
    - Credentials: Gmail OAuth2 account.  
    - Inputs: Approved expense data.

  - **Rejection Email**  
    - Type: Gmail node  
    - Configuration: Sends rejection email to employee’s email address explaining the rejection reason and amount.  
    - Credentials: Gmail OAuth2 account.  
    - Inputs: Rejected expense data.

---

### 3. Summary Table

| Node Name        | Node Type           | Functional Role                         | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                  |
|------------------|---------------------|---------------------------------------|-----------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| JotForm Trigger  | JotForm Trigger     | Triggers on new expense submission    | -                           | Parse Form Data                 | 📩 Trigger: New Expense Submission Captures employee expense details and receipt upload from JotForm. Create your form for free on [JotForm](https://www.jotform.com/?partner=mediajade) |
| Parse Form Data  | Code                | Parses and normalizes form data        | JotForm Trigger             | Validate Policy                 | 🧾 Parse & Normalize Data Extracts key fields (name, email, amount, category, merchant, receipt URL) for further processing. |
| Validate Policy  | Code                | Validates expense against company policy | Parse Form Data             | Check Violations                | ⚙️ Policy Validation Checks compliance against company rules: - Allowed categories - Max amount limits - Auto/Manager/Director approval routing |
| Check Violations | If                  | Checks for policy violations           | Validate Policy             | Route Auto, Set Rejection       | 🚫 Compliance Check If violations exist → reject Else → continue for approval routing        |
| Set Rejection    | Set                 | Sets rejection status and reason       | Check Violations (false)    | Log to Sheets                  |                                                                                                |
| Route Auto       | If                  | Routes expense for auto approval or next steps | Check Violations (true)    | Auto Approve, Route Manager     | ⚡ Routing Decision Determines if expense should be: - Auto-approved - Sent for manager approval - Sent for director approval |
| Auto Approve     | Set                 | Auto-approves expense                   | Route Auto (true)           | Log to Sheets                  |                                                                                                |
| Route Manager    | If                  | Routes expense for manager or director approval | Route Auto (false)          | Slack Manager, Slack Director   |                                                                                                |
| Slack Manager    | Slack                | Sends Slack notification for manager approval | Route Manager (true)        | Log to Sheets                  |                                                                                                |
| Slack Director   | Slack                | Sends Slack notification for director approval | Route Manager (false)       | Log to Sheets                  |                                                                                                |
| Log to Sheets    | Google Sheets        | Logs expense data and status            | Auto Approve, Set Rejection, Slack Manager, Slack Director | Is Approved                  | 📊 Audit Logging Appends expense status, approver details, and decision to Google Sheets for tracking. |
| Is Approved      | If                  | Checks if expense is approved           | Log to Sheets               | Approved (true), Rejection Email (false) | 🔍 Approval Check Branches workflow to send approval or rejection emails based on final status. |
| Approved         | Gmail                | Sends approval email to employee       | Is Approved (true)          | -                             |                                                                                                |
| Rejection Email  | Gmail                | Sends rejection email to employee      | Is Approved (false)         | -                             |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **JotForm Trigger node:**  
   - Add node `JotForm Trigger`.  
   - Set **Form** to your expense submission form ID.  
   - Use your JotForm API credentials.  
   - Name it `JotForm Trigger`.  
   - This node listens for new expense submissions.

3. **Parse Form Data (Code node):**  
   - Add a `Code` node after `JotForm Trigger`.  
   - Paste the JS code to extract and normalize fields (employeeName, employeeEmail, amount, category, merchant, date, description, receiptUrl).  
   - Name it `Parse Form Data`.

4. **Validate Policy (Code node):**  
   - Add a `Code` node after `Parse Form Data`.  
   - Define company policy rules for categories, max amounts, and approval thresholds.  
   - Implement logic to identify violations and approval routing (`auto`, `manager`, `director`).  
   - Name it `Validate Policy`.

5. **Check Violations (If node):**  
   - Add an `If` node after `Validate Policy`.  
   - Set condition: `policyViolations.length == 0`.  
   - True path: continues approval routing.  
   - False path: routes to rejection handling.  
   - Name it `Check Violations`.

6. **Set Rejection (Set node):**  
   - On the false output of `Check Violations`, add a `Set` node.  
   - Set `status` = `"rejected"` and `rejectionReason` = joined string of policy violations.  
   - Name it `Set Rejection`.

7. **Route Auto (If node):**  
   - On the true output of `Check Violations`, add an `If` node.  
   - Condition: `requiresApproval == 'auto'`.  
   - True: routes to auto-approval.  
   - False: routes to manager/director routing.  
   - Name it `Route Auto`.

8. **Auto Approve (Set node):**  
   - On true output of `Route Auto`, add `Set` node.  
   - Set `status` = `"approved"`, `approvedBy` = `"System (Auto-Approved)"`, `approvedAt` = current timestamp.  
   - Name it `Auto Approve`.

9. **Route Manager (If node):**  
   - On false output of `Route Auto`, add an `If` node.  
   - Condition: `requiresApproval == 'manager'`.  
   - True: send Slack notification to manager channel.  
   - False: send Slack notification to director channel.  
   - Name it `Route Manager`.

10. **Slack Manager (Slack node):**  
    - On true output of `Route Manager`, add `Slack` node.  
    - Use Slack API credentials.  
    - Set channel ID to manager channel (`C01234567`).  
    - Message includes employee name, amount, category.  
    - Name it `Slack Manager`.

11. **Slack Director (Slack node):**  
    - On false output of `Route Manager`, add `Slack` node.  
    - Use Slack API credentials.  
    - Set channel ID to director channel (`C98765432`).  
    - Message indicates high-value expense needing director approval.  
    - Name it `Slack Director`.

12. **Log to Sheets (Google Sheets node):**  
    - Connect outputs of `Auto Approve`, `Set Rejection`, `Slack Manager`, and `Slack Director` to this node.  
    - Configure operation as `Append or Update` in Google Sheets.  
    - Map data fields automatically, use `submissionId` as matching column.  
    - Specify your Google Sheets document ID and sheet name (`Sheet1`).  
    - Use Google Sheets OAuth2 credentials.  
    - Name it `Log to Sheets`.

13. **Is Approved (If node):**  
    - Add `If` node after `Log to Sheets`.  
    - Condition: `status == 'approved'`.  
    - True: routes to approval email.  
    - False: routes to rejection email.  
    - Name it `Is Approved`.

14. **Approved (Gmail node):**  
    - On true output of `Is Approved`, add `Gmail` node.  
    - Use Gmail OAuth2 credentials.  
    - Send email to employee email with subject "Expense Approved - $amount".  
    - Compose approval message confirming amount.  
    - Name it `Approved`.

15. **Rejection Email (Gmail node):**  
    - On false output of `Is Approved`, add `Gmail` node.  
    - Use Gmail OAuth2 credentials.  
    - Send email to employee email with subject "Expense Rejected - $amount".  
    - Compose rejection message explaining policy violations.  
    - Name it `Rejection Email`.

16. **Connect all nodes as per described flow.**

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                     |
|------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Create your own expense submission form for free on JotForm using this partner link:                       | https://www.jotform.com/?partner=mediajade         |
| Slack channel IDs must be replaced with your actual team’s channels for manager and director notifications. | Slack workspace configuration                       |
| Google Sheets document ID must be updated to your actual sheet ID for logging.                             | Google Sheets URL parameter                         |
| Gmail OAuth2 credentials require user consent for sending emails on behalf of the workflow.                | Gmail API and OAuth2 setup                          |
| Approval thresholds and policy limits can be customized in the `Validate Policy` code node according to your company rules. | Editable in workflow code node                       |

---

This documentation fully describes the automated multi-tier expense approval workflow, enabling users and AI agents to understand, reproduce, and extend the process reliably.