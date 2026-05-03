Automated Payment Reminders with Grouped QuickBooks Invoices via Email

https://n8nworkflows.xyz/workflows/automated-payment-reminders-with-grouped-quickbooks-invoices-via-email-6959


# Automated Payment Reminders with Grouped QuickBooks Invoices via Email

### 1. Workflow Overview

This workflow automates the process of sending payment reminder emails to customers with unpaid invoices in QuickBooks Online. It is designed for businesses that want to streamline their accounts receivable process by automatically fetching unpaid invoices, grouping them by customer, generating a consolidated invoice summary in a professional HTML email format, and sending reminders to customers’ billing email addresses.

**Target Use Cases:**  
- Small to medium businesses using QuickBooks Online for invoicing  
- Automating overdue payment reminders to improve cash flow  
- Reducing manual effort and errors in payment follow-ups  
- Enhancing customer communication with clear, consolidated invoice details  

**Logical Blocks:**  
- **1.1 Scheduler Trigger:** Initiates the workflow on a set schedule.  
- **1.2 Fetch Unpaid Invoices:** Connects to QuickBooks Online and retrieves all unpaid invoices.  
- **1.3 Group Invoices by Customer:** Processes the invoices to group them by individual customers for consolidated communication.  
- **1.4 Email Content Generation:** Builds a dynamic, visually appealing HTML email template listing all unpaid invoices per customer with totals.  
- **1.5 Send Reminder Email:** Sends the personalized email to the customer’s billing email address via SMTP.  
- **1.6 Documentation & Setup Guidance:** Several sticky notes provide setup instructions and customization tips for users.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduler Trigger

- **Overview:**  
  Initiates the workflow execution on a customizable recurring schedule.

- **Nodes Involved:**  
  - Scheduler

- **Node Details:**  
  - **Type & Role:** Schedule Trigger node; triggers workflow based on defined intervals.  
  - **Configuration:** Interval set to run periodically (default is every minute with empty interval array, typically adjusted by user).  
  - **Inputs:** None (start node).  
  - **Outputs:** Passes trigger to “Get Unpaid Invoices” node.  
  - **Version:** v1.  
  - **Edge Cases:** Misconfiguration could cause too frequent or infrequent execution. Ensure schedule aligns with business needs (e.g., daily or weekly).  
  - **Sticky Note:** Reminds user to configure schedule frequency (Sticky Note near Scheduler).

#### 2.2 Fetch Unpaid Invoices

- **Overview:**  
  Retrieves all invoices from QuickBooks Online, filtering for unpaid invoices to process for reminders.

- **Nodes Involved:**  
  - Get Unpaid Invoices

- **Node Details:**  
  - **Type & Role:** QuickBooks node; fetches invoice data via QuickBooks Online API using OAuth2 credentials.  
  - **Configuration:**  
    - Resource: Invoice  
    - Operation: Get All  
    - Return All: True (fetch all matching invoices)  
    - Filters: No explicit filter set in JSON, but can be customized (e.g., by date) in node options.  
  - **Credentials:** OAuth2 for QuickBooks Online.  
  - **Inputs:** Trigger from Scheduler.  
  - **Outputs:** Outputs full invoice list to next node.  
  - **Version:** v1.  
  - **Edge Cases:**  
    - API rate limits or authentication failure.  
    - Empty invoice list (no unpaid invoices).  
    - Partial data or paginated results if API limits apply.  
  - **Sticky Note:** Instructs to connect QuickBooks account and optionally adjust date filters.

#### 2.3 Group Invoices by Customer

- **Overview:**  
  Processes the entire invoice list to group invoices by customer, collecting only those with outstanding balances to prepare for email generation.

- **Nodes Involved:**  
  - Get Customer Wise Invoice list

- **Node Details:**  
  - **Type & Role:** Code node running JavaScript to process JSON input.  
  - **Configuration:**  
    - Extracts invoices from “Get Unpaid Invoices” node.  
    - Identifies a single customer (currently only processes the first customer in the list).  
    - Filters invoices for that customer where Balance > 0.  
    - Returns a JSON object containing customer name, customer ID, and array of unpaid invoices for that customer.  
  - **Inputs:** Invoice list from “Get Unpaid Invoices”.  
  - **Outputs:** Structured data for one customer with their invoices.  
  - **Version:** v2 (code node).  
  - **Edge Cases:**  
    - Currently processes only the first customer’s invoices; does not loop through all customers—limitation for multiple customers.  
    - If no invoices or no balance > 0, returns empty list.  
    - Errors in JSON parsing or missing data properties could cause failure.  
  - **Note:** This node could be enhanced to iterate over all customers for full automation.

#### 2.4 Email Content Generation

- **Overview:**  
  Dynamically builds a professional HTML email summarizing all unpaid invoices for the customer, including a personalized greeting, invoice table, total amount due, and payment link.

- **Nodes Involved:**  
  - Invoice Template

- **Node Details:**  
  - **Type & Role:** Code node generating HTML email content.  
  - **Configuration:**  
    - Accesses grouped invoice data from “Get Customer Wise Invoice list”.  
    - Iterates over each invoice to build HTML table rows with invoice number, date, due date, and amount.  
    - Sums total amount due across invoices.  
    - Constructs full HTML email with inline CSS styles for responsiveness and branding.  
    - Includes placeholders for company payment portal link and company address/footer (editable by user).  
    - Sets generated HTML to `emailBody` property for use in Send Email node.  
  - **Inputs:** Customer invoice data from previous code node.  
  - **Outputs:** JSON object with `emailBody` and customer info for email sending.  
  - **Version:** v2.  
  - **Edge Cases:**  
    - Malformed invoice data could cause exceptions.  
    - Missing or invalid invoice fields (e.g., DocNumber, TotalAmt) could break layout or calculations.  
    - User must update payment link and company info for correctness.  
  - **Sticky Note:** Instructions for personalizing payment link and company details.

#### 2.5 Send Reminder Email

- **Overview:**  
  Sends the generated payment reminder email to the customer’s billing email address.

- **Nodes Involved:**  
  - Send Reminder Email

- **Node Details:**  
  - **Type & Role:** Email Send node; sends email using SMTP credentials.  
  - **Configuration:**  
    - Subject dynamically set to “Unpaid Invoice Reminder for : [Customer Name]”.  
    - HTML body set via expression to `{{$json.emailBody}}`.  
    - Plain text body empty (not used).  
    - Recipient email dynamically set to the first invoice’s billing email address from invoices array.  
    - Sender email address is a placeholder string (“placeholderEmail”) needing user update.  
  - **Credentials:** SMTP email account (e.g., Gmail, Outlook).  
  - **Inputs:** Email content and customer info from “Invoice Template” node.  
  - **Outputs:** None (end node).  
  - **Version:** v1.  
  - **Edge Cases:**  
    - Incorrect or missing SMTP credentials cause auth failures.  
    - Recipient email missing or invalid leads to send errors.  
    - Multiple invoices per customer use billing email of first invoice only; assumes uniform billing email.  
  - **Sticky Note:** Instructions to select or create email credentials, no change needed in fields.

#### 2.6 Documentation & Setup Guidance

- **Overview:**  
  Several sticky notes provide stepwise instructions to configure and personalize the workflow, including scheduling, QuickBooks connection, template customization, and activation.

- **Nodes Involved:**  
  - Sticky Note1 (full workflow overview and instructions)  
  - Sticky Note (scheduler configuration)  
  - Sticky Note2 (QuickBooks connection)  
  - Sticky Note3 (email template personalization)  
  - Sticky Note4 (email config and workflow activation)

- **Node Details:**  
  - **Type & Role:** Sticky Note nodes containing rich text instructions for user guidance.  
  - **Content Highlights:**  
    - Workflow purpose and operation overview.  
    - Authentication setup for QuickBooks and SMTP.  
    - Personalization points in email template code.  
    - Scheduling and activation recommendations.  
    - Contact info and support links.  
  - **Inputs/Outputs:** None; purely informational.

---

### 3. Summary Table

| Node Name                   | Node Type         | Functional Role                                  | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                                 |
|-----------------------------|-------------------|-------------------------------------------------|---------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------|
| Scheduler                   | Schedule Trigger  | Initiates workflow on schedule                   | None                      | Get Unpaid Invoices          | ### 1. Set Your Schedule: Configure frequency; recommended once a day or week.                              |
| Get Unpaid Invoices          | QuickBooks Node   | Fetches all unpaid invoices from QuickBooks     | Scheduler                 | Get Customer Wise Invoice list | ### 2. Connect Your QuickBooks Account: Select or create credentials; optionally adjust filters (TxnDate).  |
| Get Customer Wise Invoice list | Code Node         | Groups invoices by customer, filters unpaid     | Get Unpaid Invoices        | Invoice Template            |                                                                                                             |
| Invoice Template            | Code Node         | Generates personalized HTML email with invoice table | Get Customer Wise Invoice list | Send Reminder Email         | ### 3. Personalize Your Email Template: Update payment link (line 115) and company info (line 120).         |
| Send Reminder Email          | Email Send Node   | Sends the generated reminder email               | Invoice Template          | None                        | ### 4. Configure & Activate: Select email credentials; no changes needed in To/HTML; save and activate.      |
| Sticky Note1                | Sticky Note       | Workflow overview and detailed setup instructions | None                     | None                        | Full detailed workflow explanation and setup guide.                                                         |
| Sticky Note                 | Sticky Note       | Scheduler setup guidance                          | None                      | None                        | ### 1. Set Your Schedule: Configure frequency; recommended once a day or week.                              |
| Sticky Note2                | Sticky Note       | QuickBooks connection instructions               | None                      | None                        | ### 2. Connect Your QuickBooks Account: Select or create credentials; optionally adjust filters (TxnDate).  |
| Sticky Note3                | Sticky Note       | Email template personalization instructions      | None                      | None                        | ### 3. Personalize Your Email Template: Update payment link (line 115) and company info (line 120).         |
| Sticky Note4                | Sticky Note       | Email sending and activation instructions        | None                      | None                        | ### 4. Configure & Activate: Select email credentials; no changes needed in To/HTML; save and activate.      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Scheduler Node:**  
   - Node Type: Schedule Trigger  
   - Configure interval as desired (e.g., daily at 9 AM)  
   - Position: Start node

2. **Create “Get Unpaid Invoices” Node:**  
   - Node Type: QuickBooks  
   - Resource: Invoice  
   - Operation: Get All  
   - Return All: True  
   - Filters: Optional; set `TxnDate` date range if needed (e.g., last 90 days)  
   - Credentials: Connect or create QuickBooks OAuth2 credentials  
   - Connect Scheduler output to this node

3. **Create “Get Customer Wise Invoice list” Node:**  
   - Node Type: Code  
   - Language: JavaScript  
   - Paste code that:  
     - Reads invoices from “Get Unpaid Invoices”  
     - Extracts first customer’s ID and name  
     - Filters invoices for that customer with balance > 0  
     - Returns object with customer name, ID, and invoice array  
   - Connect “Get Unpaid Invoices” output to this node

4. **Create “Invoice Template” Node:**  
   - Node Type: Code  
   - Language: JavaScript  
   - Paste code that:  
     - Accepts customer and invoices data  
     - Iterates invoices to create HTML table rows  
     - Computes total due  
     - Constructs full HTML email with inline styles, greeting, invoice table, total, footer, and payment link  
     - Sets generated HTML as `emailBody` property  
   - **IMPORTANT:** Replace placeholders for payment portal link and company address/footer inside code before activating  
   - Connect “Get Customer Wise Invoice list” output to this node

5. **Create “Send Reminder Email” Node:**  
   - Node Type: Email Send  
   - SMTP Credentials: Connect or create SMTP (Gmail/Outlook/etc.)  
   - From Email: Set your sender email address (replace placeholder)  
   - To Email: Use expression to extract customer billing email from invoices: `={{ $json.invoices[0].json.BillEmail.Address }}`  
   - Subject: Use expression: `=Unpaid Invoice Reminder for : {{$json.customer}}`  
   - HTML: Use expression: `={{ $json.emailBody }}`  
   - Plain Text: Leave empty or set as desired  
   - Connect “Invoice Template” output to this node

6. **Add Sticky Notes (Optional):**  
   - Create sticky notes with setup instructions or workflow overview for user guidance. Use rich text format to include headings and bullet points.

7. **Activate Workflow:**  
   - Save the workflow  
   - Set the schedule in the Scheduler node as desired  
   - Activate workflow toggle to enable automation

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                        |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| This workflow was created by **Prompt-Wizards** to automate QuickBooks payment reminders and improve customer communication with consolidated invoice emails.                                                                                                                                                                                               | Workflow Sticky Note1 content                                         |
| Replace the payment portal link in the email template code at line 115 (`href="https://your-payment-portal-link.com"`) with your actual payment URL to allow customers to pay online easily.                                                                                                                                                                     | Sticky Note3 content                                                  |
| Update your company name and address in the email footer at line 120 of the email template code to reflect your branding and contact info.                                                                                                                                                                                                                     | Sticky Note3 content                                                  |
| For support and custom automation solutions, visit [Elegant Biztech](https://www.elegantbiztech.com/) or email [sales@elegantbiztech.com](mailto:sales@elegantbiztech.com).                                                                                                                                                                                    | Sticky Note1 content                                                  |
| Recommended schedule frequency is once daily or weekly to balance timely reminders with customer experience.                                                                                                                                                                                                                                                  | Sticky Note near Scheduler node                                       |
| Ensure that QuickBooks OAuth2 credentials are properly configured and authorized to allow invoice data retrieval.                                                                                                                                                                                                                                            | Sticky Note2 content                                                  |
| SMTP email credentials must be valid and authorized to send emails on behalf of your domain or email provider to avoid delivery issues or spam filtering.                                                                                                                                                                                                   | Sticky Note4 content                                                  |
| Current implementation processes only the first customer found in unpaid invoices; to scale for multiple customers, modify the “Get Customer Wise Invoice list” node to loop or split invoices by all customers and iterate email sending accordingly.                                                                                                        | Workflow logic observation                                           |
| The email template uses inline CSS and a responsive design suitable for most email clients; however, testing across clients (Outlook, Gmail, mobile) is recommended before production deployment.                                                                                                                                                               | Best practice note                                                   |

---

**Disclaimer:**  
This document is based solely on an n8n automation workflow. All data handled is legal and public. The workflow respects content and usage policies. No illegal or protected data is included.