Send One-Time Email Alerts for Urgent Tasks with Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/send-one-time-email-alerts-for-urgent-tasks-with-google-sheets-and-gmail-9324


# Send One-Time Email Alerts for Urgent Tasks with Google Sheets and Gmail

### 1. Workflow Overview

This workflow automates the process of sending one-time email alerts for tasks marked as urgent in a Google Sheets document. It is designed for project managers or teams tracking tasks in a spreadsheet who want to be immediately notified by email when a task’s priority becomes "Urgent," ensuring prompt attention without duplicate notifications.

The workflow is logically divided into three main blocks:

- **1.1 Trigger, Filter & Row Info**: Detects changes in the "Priority" column of the Google Sheet, filters for urgent tasks that have not yet been notified, and retrieves the row number of the changed item.
- **1.2 Send Email Alert**: Sends a personalized email via Gmail with details about the urgent task.
- **1.3 Update Google Sheet**: Updates the Google Sheet row to mark that a notification has been sent, preventing duplicate alerts.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger, Filter & Row Info

- **Overview:**  
  This block listens for changes in the Google Sheet, specifically monitoring the "Priority" column. When a change occurs, it filters the data to identify rows where "Priority" equals "Urgent," the "Notified" column is empty (meaning no prior alert sent), and the row number is known. It then fetches the row number to enable further processing.

- **Nodes Involved:**  
  - Trigger when Urgent status (Google Sheets Trigger)  
  - Get the row number (Google Sheets)  
  - Condition to send the email (IF node)

- **Node Details:**

  1. **Trigger when Urgent status**  
     - Type: Google Sheets Trigger  
     - Role: Watches for changes in the "Priority" column in a specific Google Sheet and sheet tab (gid=0). Polls every minute.  
     - Configuration: Monitors the "Priority" column only to minimize unnecessary triggers; sheet and document IDs are specified.  
     - Inputs: None (trigger node)  
     - Outputs: Emits changed row data when detected  
     - Potential Failures: Authentication errors if Google Sheets OAuth2 token expires; delayed triggers if polling interval is too long; missing or renamed columns cause failure.  
     - Credentials: Google Sheets OAuth2 (named "Google Sheets Trigger account 4")

  2. **Get the row number**  
     - Type: Google Sheets (read operation)  
     - Role: Retrieves the full data row from the sheet to obtain the row number and other details.  
     - Configuration: Reads the same sheet and tab as the trigger; no filters used, fetches entire row based on trigger data context.  
     - Inputs: Trigger node output  
     - Outputs: Full row data including row_number field.  
     - Potential Failures: Sheet access issues; mismatch between trigger data and actual sheet data; OAuth token expiration.  
     - Credentials: Google Sheets OAuth2 (named "Google Sheets account 3")

  3. **Condition to send the email**  
     - Type: IF node  
     - Role: Applies logical filtering on the incoming data to ensure only urgent tasks that have not been notified and for which the row number exists proceed further.  
     - Configuration:  
       - Condition 1: Priority equals "Urgent" (case-sensitive strict string match)  
       - Condition 2: row_number is not empty  
       - Condition 3: Notified field is empty (meaning no prior notification sent)  
     - Inputs: Output of "Get the row number" node  
     - Outputs: Passes data forward only if all conditions met  
     - Potential Failures: Expression evaluation errors if fields missing; case sensitivity issues; empty or malformed data.

#### 2.2 Send Email Alert

- **Overview:**  
  This block sends a personalized email alert to a fixed recipient via Gmail, including task details such as task name, owner, deadline, status, and next steps. The email is formatted with HTML for readability.

- **Nodes Involved:**  
  - Email alert (Gmail)

- **Node Details:**

  1. **Email alert**  
     - Type: Gmail node (send email)  
     - Role: Sends an email notification about the urgent task to a fixed recipient email address.  
     - Configuration:  
       - Recipient: your.email@example.com (placeholder, should be replaced)  
       - Subject: "A critical task has been flagged in the project sheet"  
       - Message: HTML formatted body including task details injected via expressions from the JSON data of the task (Task, Owner, Deadline, Status, Next step). Includes polite request for review.  
       - Options: Default sending options, no attachments or CC.  
     - Inputs: Output from IF node (only urgent, non-notified tasks)  
     - Outputs: Email sent confirmation passed to next node  
     - Potential Failures: Gmail OAuth token expiration; Gmail API rate limits; incorrect recipient address; network issues.  
     - Credentials: Gmail OAuth2 (named "Gmail account")

#### 2.3 Update Google Sheet

- **Overview:**  
  After sending an email alert, this block updates the same row in the Google Sheet by setting the "Notified" column to "Yes." This update prevents duplicate emails from being sent for the same task if the sheet changes multiple times.

- **Nodes Involved:**  
  - Update row avoiding spam (Google Sheets)

- **Node Details:**

  1. **Update row avoiding spam**  
     - Type: Google Sheets (update operation)  
     - Role: Updates the specific row identified by row_number to mark it as notified.  
     - Configuration:  
       - Operation: Update  
       - Matching column: row_number (ensures precise row update)  
       - Columns updated: Notified = "Yes", row_number passed through unchanged  
       - Sheet and document IDs same as other Google Sheets nodes  
     - Inputs: Output from "Email alert" node  
     - Outputs: Confirmation of update operation  
     - Potential Failures: OAuth token expiration; row_number mismatch or missing; sheet write permission issues; concurrent writes causing conflicts.  
     - Credentials: Google Sheets OAuth2 (named "Google Sheets account 3")

---

### 3. Summary Table

| Node Name                | Node Type             | Functional Role                       | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                                                                                                                                                                        |
|--------------------------|-----------------------|-------------------------------------|---------------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note1             | Sticky Note           | Prerequisites note                   | None                      | None                     | Required: • Google sheets credential • Gmail credential                                                                                                                                                                                            |
| Sticky Note              | Sticky Note           | Overview of Trigger & Filtering      | None                      | None                     | 1) Trigger, Filter & Row Info: Trigger on Priority column change; filter Priority=Urgent, Notified empty, row_number exists; best practices described                                                                                               |
| Sticky Note2             | Sticky Note           | Email sending best practices         | None                      | None                     | 2) Send Email: personalized message with task details; use line breaks for formatting; personalize subject and content                                                                                                                            |
| Sticky Note3             | Sticky Note           | Google Sheet update best practices   | None                      | None                     | 3) Update Google Sheet: set Notified to Yes or timestamp to track sent alerts; avoid duplicates; make system state-aware                                                                                                                           |
| Trigger when Urgent status | Google Sheets Trigger | Detect changes in Priority column   | None                      | Get the row number        | See Sticky Note for trigger and filtering details                                                                                                                                                                                                 |
| Get the row number       | Google Sheets         | Retrieve full row data including row number | Trigger when Urgent status | Condition to send the email | See Sticky Note for trigger and filtering details                                                                                                                                                                                                 |
| Condition to send the email | IF                    | Filter for urgent, non-notified rows | Get the row number         | Email alert              | See Sticky Note for trigger and filtering details                                                                                                                                                                                                 |
| Email alert              | Gmail                 | Send personalized email notification | Condition to send the email | Update row avoiding spam | See Sticky Note for email sending best practices                                                                                                                                                                                                  |
| Update row avoiding spam | Google Sheets         | Mark row as notified in Google Sheet | Email alert               | None                     | See Sticky Note for Google Sheet update best practices                                                                                                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Type: Google Sheets Trigger  
   - Configure credentials with OAuth2 for Google Sheets access.  
   - Set Document ID to the target Google Sheet (e.g., "1a4_uY_GzzCkLRdF2yy9mT0TQj5bXsrxOBEWnGO68M9U").  
   - Set Sheet Name to the relevant tab (gid=0).  
   - Set "Columns to Watch" to ["Priority"].  
   - Set polling mode to "Every Minute" for near real-time detection.

2. **Add Google Sheets Node to Get Full Row**  
   - Type: Google Sheets (read)  
   - Use a separate Google Sheets OAuth2 credential if desired.  
   - Configure to read from the same Document ID and Sheet Name (gid=0).  
   - No filter needed; this node will receive trigger data to retrieve full row details including row_number.

3. **Add IF Node to Filter Rows**  
   - Type: IF  
   - Conditions (AND combinator):  
     - Priority equals "Urgent" (case-sensitive string).  
     - row_number is not empty (exists).  
     - Notified is empty (meaning no prior notification sent).  
   - Connect output of "Get the row number" node to this IF node.

4. **Add Gmail Node to Send Email**  
   - Type: Gmail (send)  
   - Configure Gmail OAuth2 credentials.  
   - Set "Send To" to the desired recipient email address (replace "your.email@example.com").  
   - Set Subject to "A critical task has been flagged in the project sheet".  
   - Compose Message body as HTML with placeholders for:  
     - Task: {{$json["Task"]}}  
     - Owner: {{$json["Owner"]}}  
     - Deadline: {{$json["Deadline"]}}  
     - Status: {{$json["Status"]}}  
     - Next step: {{$json["Next step"]}}  
   - Connect IF node’s true output to this node.

5. **Add Google Sheets Node to Update Row**  
   - Type: Google Sheets (update)  
   - Use same Document ID and Sheet Name as previous Google Sheets nodes.  
   - Set operation to "Update".  
   - Matching Column: "row_number" (to update correct row).  
   - Set columns to update:  
     - Notified = "Yes" (or a timestamp if preferred)  
     - row_number = {{$json["row_number"]}} (pass-through)  
   - Connect Gmail node output to this update node.

6. **Add Sticky Notes for Documentation (Optional)**  
   - Add sticky notes summarizing prerequisites (Google Sheets and Gmail credentials), trigger and filtering logic, email formatting best practices, and update logic to track notifications.

7. **Test Workflow**  
   - Ensure Google Sheets columns: Task, Owner, Deadline, Status, Next step, Priority, Notified, row_number exist and are properly formatted.  
   - Change the "Priority" column of a task to "Urgent".  
   - Confirm an email is sent only once and the "Notified" column updates correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Use the "Notified" column to prevent duplicate email alerts. Update this field after sending the alert.      | Best practice to make workflow idempotent and avoid spam emails                                    |
| Polling interval set to "every minute" balances responsiveness and API quota usage for Google Sheets trigger. | Adjust polling based on project requirements and API limits                                        |
| Gmail message body uses HTML formatting with line breaks and bold tags for clarity.                           | Improves readability of email alerts                                                               |
| Workflow requires Google Sheets and Gmail OAuth2 credentials to be configured in n8n before activation.       | Ensure credentials have sufficient scopes and permissions                                          |
| This workflow is inactive by default; activate after proper testing.                                          | Prevents unintended emails during setup                                                            |
| Source Spreadsheet URL: https://docs.google.com/spreadsheets/d/1a4_uY_GzzCkLRdF2yy9mT0TQj5bXsrxOBEWnGO68M9U | Spreadsheet must have columns: Task, Owner, Deadline, Status, Next step, Priority, Notified, row_number |

---

**Disclaimer:**  
The text above is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.