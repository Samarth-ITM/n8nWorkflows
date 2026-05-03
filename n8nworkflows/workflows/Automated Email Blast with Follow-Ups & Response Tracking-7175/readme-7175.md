Automated Email Blast with Follow-Ups & Response Tracking

https://n8nworkflows.xyz/workflows/automated-email-blast-with-follow-ups---response-tracking-7175


# Automated Email Blast with Follow-Ups & Response Tracking

### 1. Workflow Overview

This workflow automates an email marketing campaign with structured follow-ups and response tracking. It is designed to:

- Send initial and follow-up emails to contacts listed in a Google Sheet.
- Track responses by checking an email inbox for replies.
- Update the Google Sheet to reflect the current status of each contact’s interaction.
- Manage follow-up stages dynamically based on contact replies.

Logical blocks in this workflow are:

- **1.1 Scheduled Trigger**: Starts the workflow daily at a set time.
- **1.2 Contact Data Retrieval and Iteration**: Reads contact data from Google Sheets and processes each contact individually.
- **1.3 Follow-Up Stage Determination and Email Sending**: Decides which follow-up email to send based on contact status and routes accordingly.
- **1.4 Updating Contact Status in Google Sheets**: Updates the Google Sheet with follow-up status and schedule.
- **1.5 Response Checking and Logging**: Polls an email inbox for replies and updates the status in the Google Sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview**: Initiates the workflow once daily based on a cron schedule.
- **Nodes Involved**:  
  - Daily Trigger

- **Node Details**:  
  - **Daily Trigger**  
    - Type: Schedule Trigger  
    - Configured with a cron expression to trigger daily at 9 AM.  
    - No input connections; output triggers the workflow’s next step.  
    - Edge cases: Cron misconfiguration could cause missed or multiple triggers; system downtime at trigger time can delay execution.

#### 1.2 Contact Data Retrieval and Iteration

- **Overview**: Retrieves contact data from a Google Sheet and processes each contact one by one to enable individualized handling.
- **Nodes Involved**:  
  - Fetch Contact Data  
  - Iterate Contacts

- **Node Details**:  
  - **Fetch Contact Data**  
    - Type: Google Sheets  
    - Reads data from a specified sheet ("Sheet3") within a Google Sheets document using a service account credential.  
    - No input; triggered by Daily Trigger.  
    - Output: List of contacts with fields like User Name, User Email, Follow-up stage, Reply status.  
    - Edge cases: Authentication failure, sheet access permissions, empty or malformed data rows.

  - **Iterate Contacts**  
    - Type: SplitInBatches  
    - Splits the contact list into single-item batches for sequential processing.  
    - Input: Contact list from Fetch Contact Data.  
    - Output: Single contact data per batch.  
    - Edge cases: Large datasets could slow processing; batch size defaults to 1 (single contacts).

#### 1.3 Follow-Up Stage Determination and Email Sending

- **Overview**: Evaluates each contact’s reply status and follow-up stage to decide which email to send next, then sends the appropriate follow-up email.
- **Nodes Involved**:  
  - Determine Follow-Up Stage  
  - Route by Follow-Up Stage  
  - Send Follow-Up Email 1  
  - Send Follow-Up Email 2

- **Node Details**:  
  - **Determine Follow-Up Stage**  
    - Type: If  
    - Checks if the contact's "Reply" field equals "no" (meaning no response received).  
    - Input: Individual contact data from Iterate Contacts.  
    - Output: Routes contacts who have not replied forward for follow-up decisions.  
    - Edge cases: Case sensitivity or missing "Reply" field can affect logic.

  - **Route by Follow-Up Stage**  
    - Type: Switch  
    - Routes contacts based on their numeric "Follow-up" stage (e.g., 6 or 7).  
    - Input: Contacts filtered by Determine Follow-Up Stage.  
    - Output: Different paths to send follow-up email 1 or 2.  
    - Edge cases: Unexpected follow-up stage values not handled by switch might cause contacts to halt.

  - **Send Follow-Up Email 1**  
    - Type: Gmail  
    - Sends first follow-up email ("Testing-6") as plain text to the contact’s email address.  
    - Uses Google service account authentication.  
    - Input: Contacts routed with Follow-up stage = 6.  
    - Output: Triggers sheet update with status post-sending.  
    - Edge cases: SMTP failures, invalid email addresses, quota limits.

  - **Send Follow-Up Email 2**  
    - Type: Gmail  
    - Sends second follow-up email ("Testing-7").  
    - Same configuration and authentication as above.  
    - Input: Contacts routed with Follow-up stage = 7.  
    - Output: Triggers sheet update with status post-sending.  
    - Edge cases: Same as above.

#### 1.4 Updating Contact Status in Google Sheets

- **Overview**: Updates the contact's row in the Google Sheet to reflect the latest follow-up status, including incrementing the follow-up count and scheduling the next follow-up date.
- **Nodes Involved**:  
  - Update Sheet with Follow-Up Status

- **Node Details**:  
  - **Update Sheet with Follow-Up Status**  
    - Type: Google Sheets  
    - Operation: Update rows by matching on "User Name".  
    - Updates fields: Reply status, Follow-up stage incremented by 1, "Send Mail" flag, and timestamps for submission and next follow-up date (3 days later).  
    - Inputs: Data from Send Follow-Up Email nodes (both 1 and 2).  
    - Uses service account for authentication.  
    - Edge cases: Mismatched user names, concurrent updates, permission issues.

#### 1.5 Response Checking and Logging

- **Overview**: Monitors the inbox for new email replies matching criteria (subject contains "Testing"), and updates the contact’s reply status in the Google Sheet.
- **Nodes Involved**:  
  - Check Email Responses  
  - Update Sheet with Response

- **Node Details**:  
  - **Check Email Responses**  
    - Type: Email Read (IMAP)  
    - Searches for unseen emails with subject containing "Testing".  
    - Uses IMAP credentials configured in n8n.  
    - Output: Email metadata including sender’s email address.  
    - Edge cases: IMAP connection failures, rate limits, email parsing errors.

  - **Update Sheet with Response**  
    - Type: Google Sheets  
    - Updates the "reply" field to "yes" for the matching contact by email address.  
    - Authentication via service account.  
    - Edge cases: Email address mismatches, sheet update conflicts.

---

### 3. Summary Table

| Node Name                   | Node Type              | Functional Role                             | Input Node(s)             | Output Node(s)                    | Sticky Note                                                  |
|-----------------------------|------------------------|---------------------------------------------|---------------------------|---------------------------------|--------------------------------------------------------------|
| Daily Trigger               | Schedule Trigger       | Starts workflow daily at 9 AM               |                           | Fetch Contact Data               | ## Node Details - Daily Trigger - 9 AM triggers daily.       |
| Fetch Contact Data          | Google Sheets          | Reads contacts from Google Sheet            | Daily Trigger             | Iterate Contacts                | ## Node Details - Reads contact data from Google Sheet.      |
| Iterate Contacts            | SplitInBatches         | Processes contacts one by one                | Fetch Contact Data        | Determine Follow-Up Stage       | ## Node Details - Processes each contact individually.       |
| Determine Follow-Up Stage   | If                     | Checks if contact replied (Reply == "no")  | Iterate Contacts          | Route by Follow-Up Stage        | ## Node Details - Decides follow-up based on reply status.   |
| Route by Follow-Up Stage    | Switch                 | Routes to follow-up email based on stage    | Determine Follow-Up Stage | Send Follow-Up Email 1, 2       | ## Node Details - Routes contacts by follow-up stage number. |
| Send Follow-Up Email 1      | Gmail                  | Sends first follow-up email                  | Route by Follow-Up Stage  | Update Sheet with Follow-Up Status | ## Node Details - Sends first follow-up email.              |
| Send Follow-Up Email 2      | Gmail                  | Sends second follow-up email                 | Route by Follow-Up Stage  | Update Sheet with Follow-Up Status | ## Node Details - Sends second follow-up email.             |
| Update Sheet with Follow-Up Status | Google Sheets     | Updates contact status and schedule          | Send Follow-Up Email 1, 2 | Iterate Contacts               | ## Node Details - Updates Google Sheet with new follow-up info. |
| Check Email Responses       | Email Read (IMAP)      | Checks inbox for replies                      |                           | Update Sheet with Response      | ## Node Details - Checks emails for replies with subject "Testing". |
| Update Sheet with Response  | Google Sheets          | Marks contact as replied in Google Sheet     | Check Email Responses     |                               | ## Node Details - Updates reply status in Google Sheet.      |
| Sticky Note                | Sticky Note            | Documentation node                            |                           |                               | ## Node Details - Summarizes workflow nodes and roles.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Daily Trigger" node**  
   - Type: Schedule Trigger  
   - Set cron expression to trigger daily at 9 AM (e.g., `0 9 * * *`).  
   - No credentials needed.

2. **Create "Fetch Contact Data" node**  
   - Type: Google Sheets  
   - Operation: Read rows from a specific sheet ("Sheet3").  
   - Set Document ID to your Google Sheet ID.  
   - Use a Google service account credential configured with access to the sheet.  
   - No input connections; attach as next node from Daily Trigger.

3. **Create "Iterate Contacts" node**  
   - Type: SplitInBatches  
   - Default batch size of 1 (process contacts individually).  
   - Connect input from Fetch Contact Data.

4. **Create "Determine Follow-Up Stage" node**  
   - Type: If  
   - Condition: Check if `Reply` field equals "no" (case sensitive).  
   - Connect input from second output of Iterate Contacts (batch output).

5. **Create "Route by Follow-Up Stage" node**  
   - Type: Switch  
   - Rules:  
     - When `Follow-up` field equals 6, route to Send Follow-Up Email 1.  
     - When `Follow-up` field equals 7, route to Send Follow-Up Email 2.  
   - Input from true branch of Determine Follow-Up Stage.

6. **Create "Send Follow-Up Email 1" node**  
   - Type: Gmail  
   - Send To: Use expression `{{$json["User Email"]}}`.  
   - Subject: "Testing-6"  
   - Message Body: "Testing-6" (plain text).  
   - Authentication: Google Service Account with Gmail API enabled.  
   - Input from first output of Route by Follow-Up Stage.

7. **Create "Send Follow-Up Email 2" node**  
   - Same as above but:  
   - Subject: "Testing-7"  
   - Message Body: "Testing-7".  
   - Input from second output of Route by Follow-Up Stage.

8. **Create "Update Sheet with Follow-Up Status" node**  
   - Type: Google Sheets  
   - Operation: Update rows matching on "User Name".  
   - Update columns:  
     - Reply: from Route by Follow-Up Stage output.  
     - Follow-up: increment by 1 from current value.  
     - Send Mail: set to "yes (Follow-up = X)" with current stage.  
     - Submission Date: current date in ISO format (YYYY-MM-DD).  
     - Next Follow-up Date: current date plus 3 days.  
   - Use Google service account credential.  
   - Inputs from Send Follow-Up Email 1 and Send Follow-Up Email 2 nodes.  
   - Output connects back to Iterate Contacts to continue processing next contact.

9. **Create "Check Email Responses" node**  
   - Type: Email Read (IMAP)  
   - Set search criteria to: Unseen emails with subject containing "Testing".  
   - Use IMAP credentials configured for the inbox to monitor.  
   - No inputs; can be triggered independently or scheduled separately.

10. **Create "Update Sheet with Response" node**  
    - Type: Google Sheets  
    - Operation: Update rows matching on "user email" field.  
    - Update "reply" field to "yes".  
    - Use Google service account credential.  
    - Input from Check Email Responses node.

11. **(Optional) Add Sticky Note node**  
    - Add a sticky note summarizing node roles and workflow logic for documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow requires Google Sheets API enabled and proper service account credentials for access. | Google Cloud Console setup for service accounts and API access.                                         |
| Use Gmail API or SMTP with OAuth2 for sending emails to avoid security issues with service accounts. | Gmail API documentation: https://developers.google.com/gmail/api                                      |
| IMAP credentials must have permission to read the mailbox and support search filters.                | IMAP configuration depends on email provider (e.g., Gmail, Outlook).                                    |
| The Google Sheet must have columns: User Name, User Email, Reply, Follow-up, Send Mail, Submission Date, Next Follow-up Date. | Ensure sheet schema matches node configurations exactly for proper updates.                             |
| Cron schedules should be verified for timezone correctness.                                          | Cron expression docs: https://crontab.guru/                                                             |

---

This detailed analysis and instructions enable both advanced users and automation agents to fully understand, reproduce, and maintain the "Automated Email Blast with Follow-Ups & Response Tracking" workflow.