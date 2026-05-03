QuickBooks Customer Welcome Emails with Google Sheets Tracking

https://n8nworkflows.xyz/workflows/quickbooks-customer-welcome-emails-with-google-sheets-tracking-6705


# QuickBooks Customer Welcome Emails with Google Sheets Tracking

---
### 1. Workflow Overview

This workflow automates the sending of personalized welcome emails to new customers created in QuickBooks Online, while tracking processed customers via Google Sheets to avoid duplicate emails. It targets QuickBooks users who want to streamline customer onboarding communications without manual intervention.

The workflow consists of these logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow automatically on a customizable schedule.
- **1.2 Data Retrieval:** Concurrently fetches all customers from QuickBooks and reads already processed customer IDs from Google Sheets.
- **1.3 New Customer Identification:** Compares the QuickBooks customer list to the processed IDs to isolate only new customers.
- **1.4 Logging:** For each new customer, logs detailed info to a Google Sheet and appends their ID to the tracking sheet.
- **1.5 Email Generation:** Dynamically creates a personalized HTML welcome email for each new customer.
- **1.6 Email Sending:** Sends the personalized welcome email via a configured email service.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** This block triggers the workflow execution periodically based on a user-defined schedule.

- **Nodes Involved:**  
  - Scheduler  
  - Start Both Branches  
  - Sticky Note (Set Your Schedule Node)

- **Node Details:**

  - **Scheduler**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow at intervals (e.g., every hour or day).  
    - Configuration: User sets interval and optionally specific hour/minute.  
    - Input: None (trigger node)  
    - Output: Triggers "Start Both Branches" node.  
    - Edge Cases: Misconfiguration may cause missed or too frequent runs.

  - **Start Both Branches (NoOp)**  
    - Type: No Operation (NoOp) node  
    - Role: Splits workflow execution into two parallel branches: fetching QuickBooks customers and reading Google Sheets processed IDs.  
    - Configuration: Default (no parameters).  
    - Input: Trigger from Scheduler.  
    - Output: Two parallel outputs leading to "Get many customers" and "Read Old Customers".  
    - Edge Cases: None typical.

  - **Sticky Note (Set Your Schedule Node)**  
    - Pure documentation node describing how to configure the Scheduler node.

---

#### 1.2 Data Retrieval

- **Overview:** Retrieves all customers from QuickBooks Online and reads processed customer IDs from Google Sheets concurrently.

- **Nodes Involved:**  
  - Get many customers (QuickBooks)  
  - Read Old Customers (Google Sheets)  
  - Sticky Notes for each node explaining setup

- **Node Details:**

  - **Get many customers**  
    - Type: QuickBooks node  
    - Role: Fetches up to 1000 customer records from QuickBooks Online.  
    - Configuration: Operation "getAll" with limit 1000, no filters applied.  
    - Credentials: QuickBooks OAuth2.  
    - Input: From "Start Both Branches".  
    - Output: Customer list JSON to "Find New Customers".  
    - Edge Cases: OAuth token expiration, API rate limits, large data sets exceeding 1000 limit.

  - **Read Old Customers**  
    - Type: Google Sheets node  
    - Role: Reads customer IDs already processed, from "Processed IDs" sheet, column A.  
    - Configuration: Document ID points to the configured Google Sheet; sheet name "Processed IDs"; reads range A:A.  
    - Credentials: Google Sheets OAuth2.  
    - Input: From "Start Both Branches".  
    - Output: List of processed customer IDs to "Find New Customers".  
    - Edge Cases: Sheet access permission issues, empty sheet, incorrect range.

  - **Sticky Notes (QuickBooks and Google Sheets nodes)**  
    - Provide instructions on credential setup and node purpose.

---

#### 1.3 New Customer Identification

- **Overview:** Compares fetched QuickBooks customers against processed IDs to identify only new customers who have not been emailed yet.

- **Nodes Involved:**  
  - Find New Customers (Compare Datasets)  
  - Sticky Note explaining this node's function

- **Node Details:**

  - **Find New Customers**  
    - Type: Compare Datasets node  
    - Role: Performs comparison between QuickBooks customers (Input 1) and processed IDs from Google Sheets (Input 2) using the fields "Id" vs "CustomerIds".  
    - Configuration: Merge by fields "Id" (QuickBooks) and "CustomerIds" (Google Sheets).  
    - Input: Two inputs from "Get many customers" and "Read Old Customers".  
    - Output: Main output contains only new customers (those not found in processed IDs).  
    - Edge Cases: Mismatch in ID formats, empty inputs, failure if inputs are missing.

---

#### 1.4 Logging

- **Overview:** Logs new customer details and their IDs into separate Google Sheets tabs for record-keeping and tracking.

- **Nodes Involved:**  
  - Log New Customer Details (Google Sheets)  
  - Log New Customer ID for Tracking (Google Sheets)  
  - Sticky Notes describing setup for each node

- **Node Details:**

  - **Log New Customer Details**  
    - Type: Google Sheets node  
    - Role: Appends new customer details (Name, Company, Email, Phone, ID) to "New Customer Logs" tab.  
    - Configuration: Document ID set to the same Google Sheet; sheet name "New Customer Logs"; fields mapped from JSON properties.  
    - Credentials: Google Sheets OAuth2.  
    - Input: From "Find New Customers" output (new customers only).  
    - Output: Data forwarded to "Email Template".  
    - Edge Cases: Google Sheets API errors, data mapping mismatches.

  - **Log New Customer ID for Tracking**  
    - Type: Google Sheets node  
    - Role: Appends new customer IDs to "Processed IDs" tab to prevent reprocessing.  
    - Configuration: Document ID same as above; sheet name "Processed IDs"; appends customer Id field.  
    - Credentials: Google Sheets OAuth2.  
    - Input: Also from "Find New Customers" output (in parallel with above).  
    - Output: None downstream.  
    - Edge Cases: Same as above, potential duplicate IDs if workflow concurrency is not managed.

---

#### 1.5 Email Generation

- **Overview:** Generates a personalized, responsive HTML welcome email tailored to each new customer’s details.

- **Nodes Involved:**  
  - Email Template (Code node)  
  - Sticky Note with customization instructions

- **Node Details:**

  - **Email Template**  
    - Type: Code node (JavaScript)  
    - Role: Builds HTML email content dynamically using customer data fields such as GivenName, DisplayName, CompanyName, Email, Location, etc.  
    - Configuration: Custom JS code iterates over items, constructs greeting, opening line, confirmation details, and full HTML email template with placeholders for logo URL, support email, company links.  
    - Input: From "Log New Customer Details".  
    - Output: Items enriched with `emailHtml` property for sending.  
    - Edge Cases: Missing customer fields (e.g., email, name), malformed HTML if input data contains unexpected characters.  
    - Version: Requires "Execute Once" mode enabled to process all input items together.

  - **Sticky Note**  
    - Explains where to customize the email template placeholders (logo URL, support email, dashboard link, company name).

---

#### 1.6 Email Sending

- **Overview:** Sends the personalized welcome email to each new customer using a configured SMTP or email service.

- **Nodes Involved:**  
  - Send Personalized Welcome Email (Email Send)  
  - Sticky Note with sender setup instructions

- **Node Details:**

  - **Send Personalized Welcome Email**  
    - Type: Email Send node  
    - Role: Sends the HTML email generated by the previous node to the customer's email address.  
    - Configuration:  
      - From Email: Configured sender address (example: company.ebtech@gmail.com).  
      - To Email: Customer’s PrimaryEmailAddr.Address dynamically set.  
      - Subject: Personalized with company name and customer display name.  
      - HTML Body: Uses `emailHtml` from previous node.  
    - Credentials: SMTP or email service credentials (e.g., Gmail, Outlook).  
    - Input: From "Email Template".  
    - Output: None downstream.  
    - Edge Cases: Invalid email addresses, SMTP authentication failure, network issues, email quota limits.

  - **Sticky Note**  
    - Guides configuration of credentials, sender email, and subject line customization.

---

### 3. Summary Table

| Node Name                          | Node Type             | Functional Role                               | Input Node(s)                  | Output Node(s)                                      | Sticky Note                                                                                                   |
|-----------------------------------|-----------------------|----------------------------------------------|-------------------------------|----------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Scheduler                         | Schedule Trigger      | Triggers workflow on schedule                 | None                          | Start Both Branches                                | ## Set Your Schedule Node: Configure trigger interval and time.                                              |
| Start Both Branches               | NoOp                  | Splits workflow into parallel data fetches   | Scheduler                     | Get many customers, Read Old Customers             |                                                                                                               |
| Get many customers                | QuickBooks            | Fetches all customers from QuickBooks         | Start Both Branches           | Find New Customers                                 | ## Get many customers Node: Connect QuickBooks credentials, no other config needed.                          |
| Read Old Customers                | Google Sheets         | Reads processed customer IDs from Google Sheets | Start Both Branches           | Find New Customers                                 | ## Read Old Customers Node: Connect Google Sheets credentials; sheet "Processed IDs".                        |
| Find New Customers                | Compare Datasets      | Compares QuickBooks customers vs processed IDs | Get many customers, Read Old Customers | Log New Customer Details, Log New Customer ID for Tracking | ## Find New Customers Node: Compares datasets by Id and CustomerIds; outputs only new customers.              |
| Log New Customer Details          | Google Sheets         | Logs new customer details into log sheet      | Find New Customers            | Email Template                                    | ## Log New Customer Details Node: Append to "New Customer Logs" sheet; Google Sheets credentials required.   |
| Log New Customer ID for Tracking  | Google Sheets         | Adds new customer IDs to processed IDs sheet  | Find New Customers            | None                                               | ## Log New Customer ID for Tracking Node: Append to "Processed IDs" sheet; credentials required.             |
| Email Template                   | Code                  | Generates personalized HTML email content     | Log New Customer Details      | Send Personalized Welcome Email                    | ## Email Template Node: Customize logo URL, links, support email, company name in code.                      |
| Send Personalized Welcome Email  | Email Send            | Sends personalized welcome email to customer | Email Template                | None                                               | ## Send Personalized Welcome Email Node: Connect SMTP/email credentials; configure sender and subject line. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheet:**
   - Create a Google Sheet with two tabs:
     - Tab 1: `Processed IDs` with column header `CustomerIds` in cell A1.
     - Tab 2: `New Customer Logs` with headers: `Customer_Name`, `Company_Name`, `Email_ID`, `Phone_No`, `Customer_ID`.

2. **Create the Scheduler Node:**
   - Add a **Schedule Trigger** node.
   - Configure interval (e.g., every hour or day).
   - No credentials required.

3. **Add a No Operation Node:**
   - Add a **NoOp** node named "Start Both Branches".
   - Connect the Scheduler node’s output to this node.
   - This will split execution into two parallel branches.

4. **Add QuickBooks Node to Fetch Customers:**
   - Add a **QuickBooks** node named "Get many customers".
   - Set operation to `getAll`.
   - Set limit to 1000.
   - Connect "Start Both Branches" output (first output) to this node.
   - Add QuickBooks OAuth2 credentials.

5. **Add Google Sheets Node to Read Processed IDs:**
   - Add a **Google Sheets** node named "Read Old Customers".
   - Operation: read (default).
   - Configure Document ID pointing to your Google Sheet.
   - Set Sheet Name to `Processed IDs`.
   - Set data range to column A (`A:A`).
   - Connect "Start Both Branches" output (second output) to this node.
   - Add Google Sheets OAuth2 credentials.

6. **Add Compare Datasets Node to Find New Customers:**
   - Add a **Compare Datasets** node named "Find New Customers".
   - Configure merge by fields: QuickBooks "Id" and Google Sheets "CustomerIds".
   - Connect output of "Get many customers" to input 1.
   - Connect output of "Read Old Customers" to input 2.

7. **Add Google Sheets Node to Log New Customer Details:**
   - Add a **Google Sheets** node named "Log New Customer Details".
   - Operation: Append.
   - Document ID: same Google Sheet.
   - Sheet Name: `New Customer Logs`.
   - Map fields:
     - `Customer_Name` = `{{$json.GivenName}}`
     - `Company_Name` = `{{$json.CompanyName}}`
     - `Email_ID` = `{{$json.PrimaryEmailAddr.Address}}`
     - `Phone_No` = `{{$json.PrimaryPhone.FreeFormNumber}}`
     - `Customer_ID` = `{{$json.Id}}`
   - Connect main output of "Find New Customers" to this node.

8. **Add Google Sheets Node to Log New Customer IDs for Tracking:**
   - Add a **Google Sheets** node named "Log New Customer ID for Tracking".
   - Operation: Append.
   - Document ID: same Google Sheet.
   - Sheet Name: `Processed IDs`.
   - Map field:
     - `CustomerIds` = `{{$json.Id}}`
   - Connect main output of "Find New Customers" to this node (parallel with above).

9. **Add Code Node to Create Email Template:**
   - Add a **Code** node named "Email Template".
   - Paste the provided JavaScript code that dynamically builds the HTML email with placeholders.
   - Ensure "Execute Once" is enabled.
   - Connect output of "Log New Customer Details" to this node.

10. **Add Email Send Node to Send Emails:**
    - Add an **Email Send** node named "Send Personalized Welcome Email".
    - Set:
      - From Email: Your sender email (e.g., company.ebtech@gmail.com).
      - To Email: `{{$json.PrimaryEmailAddr.Address}}`.
      - Subject: A Warm Welcome to [Your Company Name], {{$json.DisplayName}}!
      - HTML: `{{$json.emailHtml}}`.
    - Connect output of "Email Template" to this node.
    - Add SMTP or other email credentials.

11. **Activate Workflow:**
    - Save the workflow.
    - Activate it.
    - The workflow will now run on the set schedule.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| This workflow was crafted by Elegant Biztech to automate QuickBooks new customer welcome emails. It uses Google Sheets as a simple database to prevent duplicate emails. Customize email template placeholders with your company's logo URL, support email, website link, and company name. Ensure you set up your Google Sheet tabs exactly as described for proper functioning.                                                                                                                                            | https://www.elegantbiztech.com                            |
| For scheduling, use the Scheduler node to set desired intervals (hourly, daily, etc.).                                                                                                                                                                                                                                                                                                                                                                                                                                  | See Sticky Note on Scheduler node                          |
| Google Sheets must have two tabs: "Processed IDs" (for tracking emailed customers) and "New Customer Logs" (for detailed logs).                                                                                                                                                                                                                                                                                                                                                                                        | Setup instructions in Step 1                              |
| Email Template customization is critical: update logo URL, dashboard link, support email, and footer company name in the Code node.                                                                                                                                                                                                                                                                                                                                                                                     | See Sticky Note on Email Template node                    |
| When sending emails, ensure SMTP or email credentials are correctly configured with valid sending addresses and limits.                                                                                                                                                                                                                                                                                                                                                                                                | See Sticky Note on Send Personalized Welcome Email node  |
| For further assistance or custom workflow development, contact Elegant Biztech.                                                                                                                                                                                                                                                                                                                                                                                                                                          | info@elegantbiztech.com                                   |

---

**Disclaimer:** The content above is derived exclusively from an automated n8n workflow export. It complies fully with content policies and does not include illegal, offensive, or protected material. All data handled is legal and public.