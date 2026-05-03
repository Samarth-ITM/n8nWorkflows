Build a Complete Email CRM with Google Sheets & MailerSend

https://n8nworkflows.xyz/workflows/build-a-complete-email-crm-with-google-sheets---mailersend-10179


# Build a Complete Email CRM with Google Sheets & MailerSend

### 1. Workflow Overview

This workflow builds a complete Email CRM automation by integrating Google Sheets as the data source and MailerSend as the email delivery service. It targets marketing use cases where user data and email campaign templates are managed in Google Sheets, and emails are sent to segmented users in a scheduled, automated manner.

The workflow is logically divided into two main flows:

- **1.1 Insert User Data and Prepare Transactions**  
  This flow is triggered on a schedule and manages inserting user data from segmented Google Sheets into a transaction sheet. It assigns statuses for processing and sending, ensuring data deduplication and preparing users for email campaigns.

- **1.2 Scheduled Email Sending**  
  This flow runs on a schedule to select users marked as "ready to send," validates email addresses (checks for emptiness and disposable emails), sends personalized emails using MailerSend templates, and updates transaction statuses based on outcomes.

Supporting blocks include template selection and filtering, data mapping, and status updates for different email sending scenarios.

---

### 2. Block-by-Block Analysis

#### 2.1 Insert User Data and Prepare Transactions

- **Overview:**  
  Automates the insertion of user data into the transaction sheet, associates it with the correct email template, manages data deduplication, and updates user statuses from "processing" to "sending."

- **Nodes Involved:**  
  Schedule Trigger, Setup Flow, Get all flow templates from Sheet, Filter Template, IF Template Parameters OK, Get user_id from cdp, IF user_id is not empty, Map Data, Add records By Status Processing, Wait, Insert Data By Status Processing, Remove Duplicates, Change Status to Sending

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the flow every day at 10:00 AM  
    - Config: Runs once daily at hour 10  
    - Inputs: None  
    - Outputs: Setup Flow  
    - Edge Cases: Timing issues or missed triggers if n8n is down

  - **Setup Flow**  
    - Type: Set  
    - Role: Defines the active flow ID (hardcoded as 3) to select the campaign  
    - Config: Assigns `flow_id=3`  
    - Inputs: Schedule Trigger  
    - Outputs: Get all flow templates from Sheet

  - **Get all flow templates from Sheet**  
    - Type: Google Sheets (Read rows)  
    - Role: Reads all campaign templates from the "template" sheet  
    - Config: Uses service account credentials to read sheet with ID for templates  
    - Inputs: Setup Flow  
    - Outputs: Filter Template  
    - Failure: Google Sheets API rate limits, auth errors

  - **Filter Template**  
    - Type: Filter  
    - Role: Filters templates to the one matching the selected `flow_id`  
    - Config: Checks if `flow_id` equals template `Id`  
    - Inputs: Get all flow templates from Sheet  
    - Outputs: IF Template Parameters OK

  - **IF Template Parameters OK**  
    - Type: If  
    - Role: Ensures required template parameters (`template_name`, `database_id`, `type`, `type_template_id`) are not empty  
    - Inputs: Filter Template  
    - Outputs: Get user_id from cdp (if true)  

  - **Get user_id from cdp**  
    - Type: Google Sheets (Read rows)  
    - Role: Reads users from the dynamic sheet named after `database_id` (segment sheet)  
    - Config: Uses service account; sheet name dynamically from `database_id` field  
    - Inputs: IF Template Parameters OK  
    - Outputs: IF user_id is not empty  
    - Edge Cases: Invalid sheet name, missing data, auth failures

  - **IF user_id is not empty**  
    - Type: If  
    - Role: Filters out entries without a `user_id`  
    - Inputs: Get user_id from cdp  
    - Outputs: Map Data

  - **Map Data**  
    - Type: Set  
    - Role: Maps and combines user and template data into a unified structure for transaction insertion  
    - Config: Assigns fields like `user_id`, `email`, `first_name`, `template_name`, `discount_code`, etc., using expressions referencing previous nodes  
    - Inputs: IF user_id is not empty  
    - Outputs: Add records By Status Processing

  - **Add records By Status Processing**  
    - Type: Google Sheets (Append)  
    - Role: Appends new rows to the `transaction` sheet with status `0-processing`  
    - Inputs: Map Data  
    - Outputs: Wait  
    - Edge Cases: API limits, data schema mismatches

  - **Wait**  
    - Type: Wait node  
    - Role: Waits 1 second before continuing (likely to prevent API rate issues)  
    - Inputs: Add records By Status Processing  
    - Outputs: Insert Data By Status Processing

  - **Insert Data By Status Processing**  
    - Type: Google Sheets (Read rows with filter)  
    - Role: Reads rows from `transaction` sheet where `status = 0-processing`  
    - Inputs: Wait  
    - Outputs: Remove Duplicates

  - **Remove Duplicates**  
    - Type: Remove Duplicates  
    - Role: Removes duplicate records based on `user_id`, keeping unique users for sending  
    - Inputs: Insert Data By Status Processing  
    - Outputs: Change Status to Sending

  - **Change Status to Sending**  
    - Type: Google Sheets (Update)  
    - Role: Updates status to `1-sending` for unique user entries, marking them ready for sending  
    - Inputs: Remove Duplicates  
    - Outputs: None (end of this insertion flow)  
    - Edge Cases: Update failures if matching rows not found

---

#### 2.2 Scheduled Email Sending

- **Overview:**  
  This flow runs on an hourly schedule to process users marked with status `1-sending`, validates email addresses, sends personalized emails via MailerSend, and updates their status based on sending results or email validity.

- **Nodes Involved:**  
  Schedule Trigger1, Insert Data By Status Sending, IF Type Email, If email is not empty, Disposal Check, Send an template HTML Mailerlite, Update Status: 2-Sent, No Email Update, Disposal Email Update

- **Node Details:**

  - **Schedule Trigger1**  
    - Type: Schedule Trigger  
    - Role: Triggers flow every hour (interval by hours)  
    - Inputs: None  
    - Outputs: Insert Data By Status Sending  
    - Edge Cases: Missed executions if n8n is down

  - **Insert Data By Status Sending**  
    - Type: Google Sheets (Read rows with filter)  
    - Role: Reads rows from `transaction` sheet where `status = 1-sending`  
    - Inputs: Schedule Trigger1  
    - Outputs: IF Type Email

  - **IF Type Email**  
    - Type: If  
    - Role: Checks if the `type` field equals `"email"` and `user_id` is not empty (valid email type user)  
    - Inputs: Insert Data By Status Sending  
    - Outputs: If email is not empty

  - **If email is not empty**  
    - Type: If  
    - Role: Filters out records where the email field is empty  
    - Inputs: IF Type Email  
    - Outputs: Disposal Check (if email present), No Email Update (if empty)

  - **Disposal Check**  
    - Type: If  
    - Role: Checks if the email address matches known disposable or invalid email patterns (regex filter)  
    - Inputs: If email is not empty  
    - Outputs:  
      - Send an template HTML Mailerlite (if not disposable)  
      - Disposal Email Update (if disposable)  

  - **Send an template HTML Mailerlite**  
    - Type: HTTP Request  
    - Role: Sends the email using MailerSend API with template ID and personalization data like `first_name`, `discount_code`, `gift_code`  
    - Config: POST to MailerSend API endpoint with JSON body reflecting email and template data  
    - Inputs: Disposal Check (non-disposable emails)  
    - Outputs: Update Status: 2-Sent  
    - Edge Cases: API authentication errors, rate limits, invalid template IDs  
    - Important: The "from" email must be verified in MailerSend dashboard

  - **Update Status: 2-Sent**  
    - Type: Google Sheets (Update)  
    - Role: Updates the transaction record status to `2-sent`, stores `sent_result` (MailerSend message ID) and timestamp  
    - Inputs: Send an template HTML Mailerlite

  - **No Email Update**  
    - Type: Google Sheets (Update)  
    - Role: Marks records without email as `3-no-email`  
    - Inputs: If email is not empty (false branch)

  - **Disposal Email Update**  
    - Type: Google Sheets (Update)  
    - Role: Marks records with disposable emails as `4-disposal-email`  
    - Inputs: Disposal Check (true branch)

---

### 3. Summary Table

| Node Name                      | Node Type                   | Functional Role                                     | Input Node(s)                     | Output Node(s)                         | Sticky Note                                                                                                                           |
|--------------------------------|-----------------------------|----------------------------------------------------|----------------------------------|--------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger             | Starts insertion flow daily at 10:00 AM            | None                             | Setup Flow                           | # 📝 Insert user_id                                                                                                                   |
| Setup Flow                    | Set                         | Defines active flow ID (campaign selection)        | Schedule Trigger                 | Get all flow templates from Sheet    | 📌 Node-by-Node Description (MailerSend eCRM) - Select campaign by setting `template_id`                                              |
| Get all flow templates from Sheet | Google Sheets (Read rows)   | Loads all campaign templates                        | Setup Flow                      | Filter Template                     |                                                                                                                                       |
| Filter Template               | Filter                      | Filters templates by selected `flow_id`            | Get all flow templates from Sheet| IF Template Parameters OK           |                                                                                                                                       |
| IF Template Parameters OK     | If                          | Validates required template fields                  | Filter Template                 | Get user_id from cdp                |                                                                                                                                       |
| Get user_id from cdp          | Google Sheets (Read rows)   | Reads user data from dynamic segment sheet          | IF Template Parameters OK       | IF user_id is not empty             |                                                                                                                                       |
| IF user_id is not empty       | If                          | Filters out empty user_id entries                    | Get user_id from cdp            | Map Data                          |                                                                                                                                       |
| Map Data                     | Set                         | Maps and combines user and template data            | IF user_id is not empty         | Add records By Status Processing    |                                                                                                                                       |
| Add records By Status Processing | Google Sheets (Append)      | Inserts new transaction records with status 0       | Map Data                       | Wait                              |                                                                                                                                       |
| Wait                         | Wait                        | Waits 1 second to avoid API throttling               | Add records By Status Processing| Insert Data By Status Processing    |                                                                                                                                       |
| Insert Data By Status Processing | Google Sheets (Read rows)   | Reads transaction rows with status 0-processing      | Wait                          | Remove Duplicates                  |                                                                                                                                       |
| Remove Duplicates            | Remove Duplicates          | Removes duplicate users based on `user_id`           | Insert Data By Status Processing| Change Status to Sending           |                                                                                                                                       |
| Change Status to Sending      | Google Sheets (Update)      | Updates status to 1-sending for unique users         | Remove Duplicates              | None                             |                                                                                                                                       |
| Schedule Trigger1             | Schedule Trigger             | Triggers email sending flow every hour               | None                          | Insert Data By Status Sending       | # ✉️ Sending Email                                                                                                                    |
| Insert Data By Status Sending | Google Sheets (Read rows)   | Reads transaction rows with status 1-sending         | Schedule Trigger1             | IF Type Email                    |                                                                                                                                       |
| IF Type Email                | If                          | Filters for type "email" and non-empty user_id       | Insert Data By Status Sending | If email is not empty            |                                                                                                                                       |
| If email is not empty        | If                          | Checks email field is not empty                       | IF Type Email                 | Disposal Check, No Email Update     |                                                                                                                                       |
| Disposal Check              | If                          | Checks if email is disposable using regex            | If email is not empty         | Send an template HTML Mailerlite, Disposal Email Update |                                                                                                                                       |
| Send an template HTML Mailerlite | HTTP Request                | Sends personalized email using MailerSend API        | Disposal Check               | Update Status: 2-Sent             | # ✉️ MailerSend Send Email: Sends campaign email using MailerSend template ID and personalization. Requires verified sender email.     |
| Update Status: 2-Sent        | Google Sheets (Update)      | Marks record as sent with timestamp and message ID   | Send an template HTML Mailerlite| None                            |                                                                                                                                       |
| No Email Update             | Google Sheets (Update)      | Marks records without email as 3-no-email             | If email is not empty         | None                            |                                                                                                                                       |
| Disposal Email Update       | Google Sheets (Update)      | Marks records with disposable emails as 4-disposal-email | Disposal Check               | None                            |                                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configuration: Set to trigger daily at 10:00 AM (triggerAtHour: 10)  
   - Connect output to "Setup Flow"

2. **Create a Set node named "Setup Flow"**  
   - Assign a number field `flow_id` with value 3 (or desired campaign ID)  
   - Connect output to "Get all flow templates from Sheet"

3. **Create a Google Sheets node "Get all flow templates from Sheet"**  
   - Operation: Read Rows  
   - Document ID: Google Sheet containing campaign templates  
   - Sheet Name: "template" (or corresponding)  
   - Authentication: Service Account credentials configured  
   - Connect output to "Filter Template"

4. **Create a Filter node "Filter Template"**  
   - Condition: Keep rows where template Id equals to `flow_id` from "Setup Flow"  
   - Connect output to "IF Template Parameters OK"

5. **Create an If node "IF Template Parameters OK"**  
   - Condition: Check non-empty for `template_name`, `database_id`, `type`, `type_template_id` fields  
   - True output connects to "Get user_id from cdp"

6. **Create a Google Sheets node "Get user_id from cdp"**  
   - Operation: Read Rows  
   - Document ID: Same as above or the sheet with user segments  
   - Sheet Name: Use expression to get from `database_id` field (dynamic)  
   - Authentication: Service Account  
   - Connect output to "IF user_id is not empty"

7. **Create an If node "IF user_id is not empty"**  
   - Condition: Check `user_id` is not empty  
   - True output connects to "Map Data"

8. **Create a Set node "Map Data"**  
   - Assign fields combining user and template data: user_id, email, first_name, template_name, discount_code, gift_code, journey, number, etc.  
   - Use expressions to reference previous nodes' JSON data  
   - Connect output to "Add records By Status Processing"

9. **Create a Google Sheets node "Add records By Status Processing"**  
   - Operation: Append Rows  
   - Document ID: Transaction sheet ID  
   - Sheet Name: "transaction"  
   - Data: Map fields with `status` set to "0-processing" and timestamps  
   - Connect output to "Wait"

10. **Create a Wait node "Wait"**  
    - Wait time: 1 second  
    - Connect output to "Insert Data By Status Processing"

11. **Create a Google Sheets node "Insert Data By Status Processing"**  
    - Operation: Read Rows with filter `status = 0-processing`  
    - Connect output to "Remove Duplicates"

12. **Create a Remove Duplicates node "Remove Duplicates"**  
    - Compare field: `user_id`  
    - Connect output to "Change Status to Sending"

13. **Create a Google Sheets node "Change Status to Sending"**  
    - Operation: Update Rows  
    - Filter rows by `user_id` and update `status` to "1-sending" and updated timestamp  
    - End this flow here.

14. **Create a Schedule Trigger node "Schedule Trigger1"**  
    - Type: Schedule Trigger  
    - Configuration: Trigger every 1 hour  
    - Connect output to "Insert Data By Status Sending"

15. **Create a Google Sheets node "Insert Data By Status Sending"**  
    - Operation: Read Rows with filter `status = 1-sending`  
    - Connect output to "IF Type Email"

16. **Create an If node "IF Type Email"**  
    - Condition: `type` equals "email" AND `user_id` not empty  
    - True output connects to "If email is not empty"

17. **Create an If node "If email is not empty"**  
    - Condition: `email` field not empty  
    - True output connects to "Disposal Check"  
    - False output connects to "No Email Update"

18. **Create an If node "Disposal Check"**  
    - Condition: Email regex does NOT match disposable email patterns (list provided)  
    - True output connects to "Send an template HTML Mailerlite"  
    - False output connects to "Disposal Email Update"

19. **Create an HTTP Request node "Send an template HTML Mailerlite"**  
    - Method: POST  
    - URL: `https://api.mailersend.com/v1/email`  
    - Authentication: HTTP Header with API Key (MailerSend)  
    - Headers: Content-Type: application/json, X-Requested-With: XMLHttpRequest  
    - Body (JSON): Use fields `email`, `type_template_id` (template ID), and personalization data (`first_name`, `discount_code`, `gift_code`)  
    - Connect output to "Update Status: 2-Sent"

20. **Create a Google Sheets node "Update Status: 2-Sent"**  
    - Operation: Update Rows  
    - Filter rows by `user_id`  
    - Update `status` to "2-sent", add timestamp, and store sent_result (MailerSend message ID)  
    - End flow here.

21. **Create a Google Sheets node "No Email Update"**  
    - Operation: Update Rows  
    - Filter by `user_id`  
    - Update `status` to "3-no-email" and timestamp  
    - End flow here.

22. **Create a Google Sheets node "Disposal Email Update"**  
    - Operation: Update Rows  
    - Filter by `user_id`  
    - Update `status` to "4-disposal-email" and timestamp  
    - End flow here.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The workflow uses MailerSend API for transactional email sending. The "from" email address in the HTTP Request node must be verified and approved inside MailerSend to avoid delivery issues.                                   | MailerSend Documentation: https://developers.mailersend.com/                                        |
| Google Sheets serve as the CRM database with three key sheets: `template` (email campaigns), `segment` (user data), and `transaction` (email sending status and logs).                                                       | Example Google Sheet: https://docs.google.com/spreadsheets/d/17KqltP-NqchPhZV7gk6QToqCZX6IiA5EBkDCBNsIX_0 |
| Disposable email detection uses a regex pattern that covers common disposable domains and keywords to prevent sending to invalid addresses.                                                                                   | Disposable email check regex embedded in Disposal Check node                                        |
| The workflow includes extensive status management for user records: `0-processing`, `1-sending`, `2-sent`, `3-no-email`, and `4-disposal-email` to track email campaign progress and handle exceptions gracefully.            | Status field in transaction sheet                                                                   |
| To add new campaign flows, duplicate the insertion flow and update `flow_id` in Setup Flow node to reference a new template record in the `template` sheet.                                                                   | Sticky Note in workflow describing flow duplication approach                                        |

---

**Disclaimer:**  
The text provided here is exclusively derived from an automated workflow built with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.