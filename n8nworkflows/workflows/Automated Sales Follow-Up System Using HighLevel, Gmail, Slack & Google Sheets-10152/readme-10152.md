Automated Sales Follow-Up System Using HighLevel, Gmail, Slack & Google Sheets

https://n8nworkflows.xyz/workflows/automated-sales-follow-up-system-using-highlevel--gmail--slack---google-sheets-10152


# Automated Sales Follow-Up System Using HighLevel, Gmail, Slack & Google Sheets

### 1. Workflow Overview

This workflow, titled **"Automated Sales Follow-Up System Using HighLevel, Gmail, Slack & Google Sheets"**, automates the process of nurturing sales leads by detecting drop-offs in communication and initiating appropriate follow-ups. It targets sales teams and marketers who want to maintain engagement with potential clients tracked in HighLevel CRM, automatically send follow-up emails, monitor responses, and notify teams via Slack.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Manual trigger to start the workflow.
- **1.2 Data Retrieval from CRM**: Fetch all contacts from HighLevel CRM.
- **1.3 Data Validation**: Check if contact data retrieval was successful.
- **1.4 Filtering and Selection**: Filter contacts who have not responded in the last 24 hours and identify the most recent contact.
- **1.5 Follow-Up Emailing**: Send a personalized follow-up email to the selected contact.
- **1.6 Response Waiting Period**: Introduce a 24-hour wait to allow time for contact response.
- **1.7 Response Retrieval and Analysis**: Retrieve email thread from Gmail and check if the response contains a positive confirmation ("yes").
- **1.8 Notifications to Sales Team**: Notify Slack channels about either a positive response or lack of response.
- **1.9 Error Logging**: Log any errors encountered during the data fetching or validation steps to Google Sheets for audit and troubleshooting.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually through a trigger node.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**  
  - **Type:** Manual Trigger  
  - **Role:** Starts the entire workflow manually by user command.  
  - **Configuration:** No parameters; simple manual trigger.  
  - **Input/Output:** No input; outputs trigger signal to next node.  
  - **Failures:** None expected in normal operation.

---

#### 1.2 Data Retrieval from CRM

- **Overview:**  
  Retrieves all contacts from HighLevel CRM to be processed in subsequent steps.

- **Nodes Involved:**  
  - Fetch Contacts from HighLevel

- **Node Details:**  
  - **Type:** HighLevel node  
  - **Role:** Connects to HighLevel API via OAuth2 and fetches all contacts.  
  - **Configuration:** Operation set to "getAll" contacts; no filters applied.  
  - **Expressions:** None; static fetch of all contacts.  
  - **Input/Output:** Input triggered from manual trigger; output is a list of contacts.  
  - **Version Requirements:** HighLevel node v2 used.  
  - **Edge Cases:**  
    - OAuth token expiry or failure.  
    - API rate limits.  
    - Empty or malformed response.  
  - **Credentials:** Uses HighLevel OAuth2 credentials.

- **Sticky Note Summary:**  
  Retrieves contact names, emails, and metadata for processing.

---

#### 1.3 Data Validation

- **Overview:**  
  Validates whether the contact data fetch was successful to ensure workflow continues only if valid data is present.

- **Nodes Involved:**  
  - Validate Deal Fetch Success  
  - Log Errors in Google Sheets

- **Node Details:**  
  - **Validate Deal Fetch Success**  
    - **Type:** If node  
    - **Role:** Checks if the fetched data contains a non-empty "id" field.  
    - **Configuration:** Checks `{{$json.id}}` is not empty.  
    - **Input/Output:** Input from contacts fetch; two outputs: success path and failure path.  
    - **Edge Cases:** If no contacts or invalid data, triggers error logging path.  
    - **Version:** v2.2  
  - **Log Errors in Google Sheets**  
    - **Type:** Google Sheets node  
    - **Role:** Appends error details to a Google Sheet for auditing.  
    - **Configuration:** Append operation to a specific sheet named "error log sheet" in a Google Spreadsheet. Columns include error_id and error message.  
    - **Input/Output:** Input from failure output of validation node; no output connections.  
    - **Edge Cases:** Google API auth failure, quota exceeded, or sheet not found.  
    - **Credentials:** Google OAuth2 credentials — automations@techdome.ai.

- **Sticky Note Summary:**  
  Validates data existence; logs errors if fetch fails.

---

#### 1.4 Filtering and Selection

- **Overview:**  
  Filters contacts who have not responded in the last 24 hours and selects the most recent contact for follow-up.

- **Nodes Involved:**  
  - Filter Contacts with No Response  
  - Get Most Recent Contact

- **Node Details:**  
  - **Filter Contacts with No Response**  
    - **Type:** Code node (JavaScript)  
    - **Role:** Filters contacts based on last updated timestamp to identify those inactive for ≥24 hours.  
    - **Configuration:**  
      - Reads input contacts array.  
      - Calculates hours since last update (`dateUpdated`).  
      - Filters contacts where hours since last update ≥ 24.  
      - Returns filtered contacts as separate output items.  
    - **Input/Output:** Input from validation success; outputs filtered contacts.  
    - **Edge Cases:**  
      - Missing or invalid `dateUpdated` field may cause JS errors.  
      - No contacts matching filter results in empty output.  
    - **Version:** v2  
  - **Get Most Recent Contact**  
    - **Type:** Code node (JavaScript)  
    - **Role:** Selects the contact with the most recent update from filtered contacts.  
    - **Configuration:**  
      - Sorts contacts descending by `dateUpdated`.  
      - Picks first contact in sorted list.  
      - Calculates hours since last update.  
      - Prepares formatted output string.  
    - **Input/Output:** Input from filtered contacts; outputs most recent contact info plus formatted text.  
    - **Edge Cases:**  
      - Empty input array causes undefined results; should be handled gracefully.  
    - **Version:** v2

- **Sticky Note Summary:**  
  Filters inactive contacts and identifies the freshest lead.

---

#### 1.5 Follow-Up Emailing

- **Overview:**  
  Sends a personalized follow-up email to the selected contact to check engagement.

- **Nodes Involved:**  
  - Send Follow-Up Email to Contact

- **Node Details:**  
  - **Type:** Gmail node  
  - **Role:** Sends email using Gmail OAuth2 credentials.  
  - **Configuration:**  
    - Recipient email set statically to `newscctv22@gmail.com` (likely placeholder; should be dynamic per contact).  
    - Email subject includes contact name dynamically.  
    - Email HTML body personalized with contact’s first name or contact name, hours since last update, and last update date.  
    - Includes call to action to reply with “yes.”  
  - **Input/Output:** Input from most recent contact; output to wait node.  
  - **Edge Cases:**  
    - Missing email address for contact.  
    - Gmail API quota or auth failure.  
    - Static recipient may be an error or for testing.  
  - **Credentials:** Gmail OAuth2.

- **Sticky Note Summary:**  
  Sends personalized follow-up emails asking for confirmation.

---

#### 1.6 Response Waiting Period

- **Overview:**  
  Pauses workflow for a specified duration to allow time for recipient response.

- **Nodes Involved:**  
  - Wait for 24 Hours Before Next Action

- **Node Details:**  
  - **Type:** Wait node  
  - **Role:** Introduces a delay of 24 hours (42 minutes configured here, likely for testing).  
  - **Configuration:**  
    - Amount set to 42 (minutes instead of 24 hours, probable testing shortcut).  
  - **Input/Output:** Input from email send; output to email retrieval.  
  - **Edge Cases:**  
    - Workflow timeout if delay exceeds plan limits.  
    - Unexpected workflow interruption during wait.  
  - **Version:** v1

- **Sticky Note Summary:**  
  Waits 24 hours before checking for a response.

---

#### 1.7 Response Retrieval and Analysis

- **Overview:**  
  Retrieves the email thread from Gmail to check if the contact responded, specifically looking for a "yes" reply.

- **Nodes Involved:**  
  - Retrieve Email Thread for Response  
  - Check If Contact Responded with 'Yes'

- **Node Details:**  
  - **Retrieve Email Thread for Response**  
    - **Type:** Gmail node  
    - **Role:** Fetches Gmail thread data using threadId from prior email.  
    - **Configuration:**  
      - Resource: `thread`  
      - Operation: `get`  
      - Uses expression `{{$json.threadId}}` to get thread ID dynamically.  
    - **Input/Output:** Input from wait node; outputs thread details.  
    - **Edge Cases:**  
      - Missing or incorrect thread ID.  
      - Gmail API failures or auth errors.  
    - **Credentials:** Gmail OAuth2.  
  - **Check If Contact Responded with 'Yes'**  
    - **Type:** If node  
    - **Role:** Checks if the email thread's second message snippet contains the word "yes" (case sensitive).  
    - **Configuration:**  
      - Condition: string contains "yes" in `{{$json.messages[1].snippet}}`.  
    - **Input/Output:** Input from thread retrieval; two outputs: positive response path and no response path.  
    - **Edge Cases:**  
      - Email thread may have fewer than two messages, causing undefined errors.  
      - Case sensitivity may miss variants like "Yes", "YES".  
    - **Version:** v2.2

- **Sticky Note Summary:**  
  Fetches latest email thread and looks for "yes" in response.

---

#### 1.8 Notifications to Sales Team

- **Overview:**  
  Sends Slack notifications to sales team based on whether a positive response was received or not.

- **Nodes Involved:**  
  - Notify Sales Team in Slack if Response Received  
  - Notify Sales Team in Slack if No Response

- **Node Details:**  
  - **Notify Sales Team in Slack if Response Received**  
    - **Type:** Slack node  
    - **Role:** Sends congratulatory message about positive contact response.  
    - **Configuration:**  
      - Message includes contact name, email, and snippet of their reply.  
      - Sends to specific user ID representing sales team member.  
    - **Input/Output:** Input from positive response path of if node; no output.  
    - **Edge Cases:**  
      - Slack API auth failure or rate limits.  
      - User ID may change or be invalid.  
    - **Credentials:** Slack OAuth token.  
  - **Notify Sales Team in Slack if No Response**  
    - **Type:** Slack node  
    - **Role:** Sends notification about pending response and follow-up sent.  
    - **Configuration:**  
      - Message includes contact info, follow-up sent timestamp, and snippet of last message.  
      - Same user ID as above.  
    - **Input/Output:** Input from no response path of if node; no output.  
    - **Edge Cases:** Same as above.

- **Sticky Note Summary:**  
  Notifies sales team on Slack about response status.

---

#### 1.9 Error Logging

- **Covered in 1.3 Data Validation block**

---

### 3. Summary Table

| Node Name                         | Node Type             | Functional Role                            | Input Node(s)                | Output Node(s)                     | Sticky Note                                                                                   |
|----------------------------------|-----------------------|--------------------------------------------|-----------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger        | Starts the workflow manually               | None                        | Fetch Contacts from HighLevel      |                                                                                              |
| Fetch Contacts from HighLevel     | HighLevel             | Fetches all contacts from CRM              | When clicking ‘Execute workflow’ | Validate Deal Fetch Success         | Retrieves all contacts from HighLevel CRM including names, emails, metadata                  |
| Validate Deal Fetch Success       | If                    | Validates if contact data fetch succeeded | Fetch Contacts from HighLevel | Filter Contacts with No Response; Log Errors in Google Sheets | Validates if the deal fetch was successful or not                                            |
| Log Errors in Google Sheets       | Google Sheets         | Logs errors to Google Sheets                | Validate Deal Fetch Success (failure path) | None                             | Logs errors encountered during data fetching or validation                                   |
| Filter Contacts with No Response  | Code                  | Filters contacts inactive for ≥24 hours    | Validate Deal Fetch Success (success path) | Get Most Recent Contact            | Filters contacts who have not responded in the last 24 hours                                 |
| Get Most Recent Contact           | Code                  | Selects the most recently updated contact  | Filter Contacts with No Response | Send Follow-Up Email to Contact    | Retrieves the most recent contact based on last updated date                                |
| Send Follow-Up Email to Contact   | Gmail                 | Sends personalized follow-up email         | Get Most Recent Contact       | Wait for 24 Hours Before Next Action | Sends a follow-up email to the selected contact                                              |
| Wait for 24 Hours Before Next Action | Wait                  | Pauses workflow to wait for response       | Send Follow-Up Email to Contact | Retrieve Email Thread for Response  | Waits 24 hours before next action                                                           |
| Retrieve Email Thread for Response | Gmail                 | Retrieves email thread from Gmail           | Wait for 24 Hours Before Next Action | Check If Contact Responded with 'Yes' | Retrieves the email thread based on the thread ID                                           |
| Check If Contact Responded with 'Yes' | If                    | Checks if reply contains "yes"              | Retrieve Email Thread for Response | Notify Sales Team in Slack if Response Received; Notify Sales Team in Slack if No Response | Checks if the email response contains the word "yes"                                        |
| Notify Sales Team in Slack if Response Received | Slack                 | Notifies sales team of positive response   | Check If Contact Responded with 'Yes' (true path) | None                             | Sends Slack notification if response received                                               |
| Notify Sales Team in Slack if No Response | Slack                 | Notifies sales team of no response          | Check If Contact Responded with 'Yes' (false path) | None                             | Sends Slack notification if no response received                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Create HighLevel Node to Fetch Contacts**  
   - Type: HighLevel  
   - Operation: getAll contacts  
   - Credentials: Configure with HighLevel OAuth2 API credentials.  
   - Connect Manual Trigger output to this node.

3. **Add If Node to Validate Data Fetch**  
   - Type: If  
   - Condition: Check if `{{$json.id}}` is not empty (string not empty)  
   - Connect HighLevel node output to If node input.

4. **Create Google Sheets Node for Error Logging**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Use target Google Sheet ID with an "error log sheet" tab.  
   - Columns: error_id (string), error (string)  
   - Credentials: Google OAuth2 with access to the sheet.  
   - Connect If node’s failure output (no data) to this node.

5. **Create Code Node to Filter Contacts with No Response**  
   - Type: Code  
   - Paste provided JavaScript code filtering contacts with last update older than 24 hours.  
   - Connect If node’s success output (valid contacts) to this node.

6. **Create Code Node to Get Most Recent Contact**  
   - Type: Code  
   - Paste provided JavaScript code sorting contacts by last updated and preparing formatted output.  
   - Connect Filter Contacts node output to this node.

7. **Create Gmail Node to Send Follow-Up Email**  
   - Type: Gmail  
   - Operation: Send email  
   - Recipient: Ideally dynamic `{{$json.email}}` (adjust from static placeholder).  
   - Subject: `"Just Checking In - {{$json.contactName}}"`  
   - Message: Use provided HTML template with dynamic placeholders for name, hoursAgo, and dateUpdated.  
   - Credentials: Gmail OAuth2 with sending permission.  
   - Connect Get Most Recent Contact output to this node.

8. **Add Wait Node to Delay Next Action by 24 Hours**  
   - Type: Wait  
   - Amount: 24 hours (or 1440 minutes); workflow had 42 minutes for testing.  
   - Connect Gmail send node output to this node.

9. **Create Gmail Node to Retrieve Email Thread**  
   - Type: Gmail  
   - Operation: Get thread by ID  
   - Resource: Thread  
   - Thread ID: Expression from previous email’s threadId `{{$json.threadId}}`.  
   - Credentials: Gmail OAuth2.  
   - Connect Wait node output to this node.

10. **Create If Node to Check if Response Contains 'Yes'**  
    - Type: If  
    - Condition: Check if `{{$json.messages[1].snippet}}` contains `"yes"` (case sensitive).  
    - Connect Gmail thread retrieval node output to this node.

11. **Create Slack Node to Notify on Positive Response**  
    - Type: Slack  
    - Text: Use template congratulating and summarizing the positive response.  
    - User: Set to sales team user ID.  
    - Credentials: Slack OAuth2 with message sending rights.  
    - Connect If node’s "true" output to this node.

12. **Create Slack Node to Notify on No Response**  
    - Type: Slack  
    - Text: Use template notifying that no response received yet, with contact details.  
    - User: Same sales team user ID.  
    - Credentials: Slack OAuth2.  
    - Connect If node’s "false" output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow uses OAuth2 credentials for HighLevel, Gmail, Slack, and Google Sheets integrations.   | Ensure these credentials are properly configured and authorized for API access.                 |
| The Wait node duration is set to 42 minutes in testing but should be set to 24 hours in production. | Adjust accordingly to avoid premature checks or workflow timeouts.                              |
| The email recipient in the Gmail node is statically set to a fixed email (newscctv22@gmail.com).    | Replace with dynamic contact email for real deployment.                                        |
| Slack user ID is hardcoded; verify current sales team Slack IDs to ensure messages reach correct users. | Update if sales team members or user IDs change.                                               |
| Google Sheets error logging aids in identifying workflow failures and API issues.                   | Monitor the sheet regularly for error trends or recurring issues.                               |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.