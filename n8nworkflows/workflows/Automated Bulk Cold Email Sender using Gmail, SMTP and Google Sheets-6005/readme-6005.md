Automated Bulk Cold Email Sender using Gmail, SMTP and Google Sheets

https://n8nworkflows.xyz/workflows/automated-bulk-cold-email-sender-using-gmail--smtp-and-google-sheets-6005


# Automated Bulk Cold Email Sender using Gmail, SMTP and Google Sheets

### 1. Workflow Overview

This workflow automates sending bulk cold emails by integrating Google Sheets for contact management, Gmail or SMTP for sending emails, and n8n's scheduling and control nodes for flow regulation. It is designed for marketers or sales teams who want to send personalized emails in batches with tracking of sent status.

The main logical blocks are:

- **1.1 Scheduled Trigger & Email Retrieval:** Periodically triggers the workflow and fetches unsent emails from a Google Sheet.
- **1.2 Batch Processing & Looping:** Limits and batches the retrieved email list to control sending volume and process emails one by one.
- **1.3 Email Sending:** Sends emails via Gmail OAuth2 or SMTP, depending on configuration.
- **1.4 Status Update & Delay:** Updates the Google Sheet with email sent status and message ID, then waits before processing the next email.
- **1.5 Conditional Logic:** Checks if a valid email address exists before attempting to send an email.
- **1.6 User Guidance & Notes:** Provides setup instructions and usage notes via sticky notes for end users.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Email Retrieval

- **Overview:**  
  This block triggers the workflow every 6 hours and retrieves all rows from a Google Sheet where emails have not yet been sent (filtering on "Email Sent" = "No").

- **Nodes Involved:**  
  - Set Timer  
  - Get Emails  

- **Node Details:**

  - **Set Timer**  
    - Type: Schedule Trigger  
    - Config: Executes every 6 hours (hourly interval set to 6)  
    - Input: None (trigger node)  
    - Output: Triggers "Get Emails" node  
    - Failures: Workflow will not run if n8n is offline or schedule misconfigured.

  - **Get Emails**  
    - Type: Google Sheets (Read)  
    - Config: Reads rows from a Google Sheet named "Bulk Sender Template" (documentId provided), filtered where "Email Sent " column equals "No"  
    - Credentials: Google Sheets OAuth2 account  
    - Input: Trigger from Set Timer  
    - Output: List of unsent email records passed to "Limit" node  
    - Edge Cases:  
      - Google Sheets API quota limits or credential expiration  
      - If no matching rows, downstream nodes receive empty input

#### 1.2 Batch Processing & Looping

- **Overview:**  
  Limits the number of emails processed per run and processes them one by one in batches of one to prevent overload or spam detection.

- **Nodes Involved:**  
  - Limit  
  - Loop Over Items  

- **Node Details:**

  - **Limit**  
    - Type: Limit  
    - Config: Limits the number of items passing through (default no explicit limit set in parameters, but can be configured)  
    - Input: Rows from Get Emails  
    - Output: Limited subset passed to Loop Over Items  
    - Edge Cases: If set limit exceeds available items, all items processed.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Config: Batch size set to 1, processes one email at a time  
    - Input: Limited emails from Limit node  
    - Output: Single email item per iteration passed to "If" node  
    - Edge Cases: Batch size 1 can slow down processing if many emails; misconfiguration may cause infinite loops.

#### 1.3 Email Sending

- **Overview:**  
  Sends email messages either using Gmail OAuth2 integration or SMTP based on user preference. Includes conditional check for email address existence.

- **Nodes Involved:**  
  - If  
  - Send Email (Gmail)  
  - connect (SMTP)  

- **Node Details:**

  - **If**  
    - Type: If conditional  
    - Config: Checks if "Email Address " field exists and is non-empty  
    - Input: Single email item from Loop Over Items  
    - Output: True branch to Send Email node, False branch is empty (no action)  
    - Edge Cases: Empty or malformed email addresses prevent sending.

  - **Send Email**  
    - Type: Gmail  
    - Config: Uses Gmail OAuth2 credentials; sends plain text email to the address with subject and body from the input JSON  
    - Credentials: Gmail OAuth2 account  
    - Input: Email item passing If condition  
    - Output: Email sending result passed to Update Records  
    - Edge Cases: Gmail API rate limits, OAuth token expiration, email sending failures.

  - **connect**  
    - Type: Email Send (SMTP)  
    - Config: SMTP credentials used to send email with plain text format using fields from input JSON  
    - Credentials: SMTP credentials configured  
    - Input: Alternative sending method (not connected in main flow but available for use)  
    - Output: Email sending result  
    - Notes: Sticky note advises this node for non-Google email services.  
    - Edge Cases: SMTP server connection issues, authentication failures.

#### 1.4 Status Update & Delay

- **Overview:**  
  After sending an email, updates the Google Sheet row to mark email as sent with timestamp and message ID, then waits 10 seconds before processing the next.

- **Nodes Involved:**  
  - Update Records  
  - Wait  

- **Node Details:**

  - **Update Records**  
    - Type: Google Sheets (Append or Update)  
    - Config: Updates columns "Sent on" (current datetime), "Email Sent " (Yes), "Message Id" (from Send Email node), matching on "Email Address " column  
    - Credentials: Google Sheets OAuth2 account  
    - Input: Result from Send Email node  
    - Output: Triggers Wait node  
    - Edge Cases: Google Sheets API errors, concurrency issues, data mismatches with matching column.

  - **Wait**  
    - Type: Wait  
    - Config: Pauses workflow for 10 seconds  
    - Input: Post-update trigger  
    - Output: Next batch iteration via Loop Over Items  
    - Edge Cases: Delay may accumulate with many items, increasing total runtime.

#### 1.5 Conditional Logic

- **Overview:**  
  Ensures that emails are only sent if the email address field exists and is not empty, preventing errors or unintended sends.

- **Nodes Involved:**  
  - If (already described under 1.3)  

- **Node Details:** See 1.3.

#### 1.6 User Guidance & Notes

- **Overview:**  
  Provides setup instructions, external resource links, and usage notes to users via sticky notes placed in the editor for easy reference.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note4  

- **Node Details:**

  - **Sticky Note**  
    - Content: "# Email Sender" (Title/branding)  
    - Position: Top center

  - **Sticky Note1**  
    - Content:  
      - Setup guide points including timer setup, Google Sheet duplication, linking, email node selection, and limit node usage  
      - Help links to detailed setup guide, contact email, and additional workflows document  
      - Hyperlinks preserved as clickable references  
    - Position: Left side, large area for detailed instructions

  - **Sticky Note4**  
    - Content: "Utilize this node for non-Google email services."  
    - Refers to SMTP node usage advice  
    - Position: Near SMTP "connect" node  
    - Color: Emphasized with color 3 (likely orange/yellow)

---

### 3. Summary Table

| Node Name       | Node Type                   | Functional Role                          | Input Node(s)        | Output Node(s)       | Sticky Note                                                   |
|-----------------|-----------------------------|----------------------------------------|----------------------|----------------------|---------------------------------------------------------------|
| Set Timer       | Schedule Trigger            | Triggers workflow every 6 hours        | None (trigger)       | Get Emails           |                                                               |
| Get Emails      | Google Sheets (Read)        | Retrieves unsent email records          | Set Timer            | Limit                |                                                               |
| Limit           | Limit                      | Limits number of emails to process      | Get Emails           | Loop Over Items       |                                                               |
| Loop Over Items | SplitInBatches             | Processes emails one by one              | Limit                | If                   |                                                               |
| If              | If Conditional             | Checks if email address exists           | Loop Over Items       | Send Email            |                                                               |
| Send Email      | Gmail                      | Sends email via Gmail OAuth2             | If                   | Update Records        |                                                               |
| connect         | Email Send (SMTP)          | Alternative email sending via SMTP       | None (not connected)  | None (standalone)     | Utilize this node for non-Google email services.              |
| Update Records  | Google Sheets (Append/Update) | Updates sheet marking email sent       | Send Email            | Wait                  |                                                               |
| Wait            | Wait                       | Delays next email processing by 10 sec | Update Records        | Loop Over Items       |                                                               |
| Sticky Note     | Sticky Note                | Title/branding                          | None                 | None                  | # Email Sender                                                |
| Sticky Note1    | Sticky Note                | Setup guide and help information        | None                 | None                  | See detailed setup and help links inside sticky note content |
| Sticky Note4    | Sticky Note                | SMTP node usage note                     | None                 | None                  | Utilize this node for non-Google email services.              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to run every 6 hours (`hoursInterval: 6`).

2. **Add Google Sheets node (Get Emails):**  
   - Operation: Read rows  
   - Set Spreadsheet ID to your duplicated Google Sheet (e.g., `1Tq1WP6qf-YHx5odZWHFit1nFvMLSRmxM0-Fkf8mx8zk`)  
   - Sheet: Use `gid=0` or sheet name "Sheet1"  
   - Add filter: Select rows where column "Email Sent " equals "No" to fetch unsent emails  
   - Connect Schedule Trigger output to this node  
   - Attach Google Sheets OAuth2 credentials

3. **Add Limit node:**  
   - Type: Limit  
   - Configure limit to control number of emails per run (default unlimited, set as needed)  
   - Connect Get Emails output to Limit

4. **Add SplitInBatches node (Loop Over Items):**  
   - Batch Size: 1 (process one email at a time)  
   - Connect Limit output to this node

5. **Add If node:**  
   - Condition: Check if `Email Address ` field exists and is not empty  
   - Expression example: `{{$json["Email Address "]}}` exists  
   - Connect Loop Over Items output to If node

6. **Add Gmail node (Send Email):**  
   - Operation: Send email  
   - Set "Send To" to `{{$json["Email Address "]}}`  
   - Set "Subject" to `{{$json["Email Subject"]}}`  
   - Set "Message" to `{{$json["Email Body"]}}`  
   - Email type: Plain text  
   - Connect If node's "true" output to this node  
   - Attach Gmail OAuth2 credentials

7. **Add Google Sheets node (Update Records):**  
   - Operation: Append or Update  
   - Spreadsheet ID and Sheet same as Get Emails node  
   - Mapping:  
     - Update "Email Sent " column to "Yes"  
     - Update "Sent on" column to current datetime (`{{$now}}`)  
     - Update "Message Id" column with message ID from Send Email node (`{{$json.id}}`)  
     - Matching column: "Email Address " for row identification  
   - Connect Send Email output to Update Records node  
   - Attach Google Sheets OAuth2 credentials

8. **Add Wait node:**  
   - Duration: 10 seconds  
   - Connect Update Records output to Wait node

9. **Connect Wait node output back to Loop Over Items:**  
   - This creates a loop processing each email sequentially with a delay

10. **Optional: Add SMTP node (Email Send) for alternative sending:**  
    - Set parameters: To, Subject, Body from input fields  
    - Attach SMTP credentials  
    - Not connected by default; used if Gmail service is not preferred

11. **Add Sticky Notes:**  
    - Add one titled "# Email Sender" as header  
    - Add detailed setup guide sticky note with instructions and links (include links to Google Sheet template, detailed setup guide, contact email, and workflow library document)  
    - Add note near SMTP node advising its usage for non-Google email services

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Duplicate the provided [Google Sheet Template](https://docs.google.com/spreadsheets/d/1TjXelyGPg5G8lbPDI9_XOReTzmU1o52z2R3v8dYaoQM/edit?usp=sharing) and link it in the nodes. | Google Sheets template for managing bulk email contacts                                                        |
| For detailed instructions, refer to the [Detailed Setup Guide](https://drive.google.com/file/d/1o95RjqpwH_FZc28ajgxJJj5j3u6IyO0w/view?usp=sharing). | Comprehensive step-by-step setup instructions                                                                   |
| Contact support for assistance at [info.gainflow@gmail.com](mailto:info.gainflow@gmail.com).                                                     | Support email                                                                                                   |
| Discover more practical workflows [HERE](https://docs.google.com/document/d/1RACo90h-QwKA4hEZSlOQZsyw4iB5-43JM2l0s4lpuoY/edit?usp=sharing).       | Additional workflows documentation                                                                              |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.