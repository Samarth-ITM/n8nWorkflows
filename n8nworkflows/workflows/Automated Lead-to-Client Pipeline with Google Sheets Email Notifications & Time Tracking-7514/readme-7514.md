Automated Lead-to-Client Pipeline with Google Sheets Email Notifications & Time Tracking

https://n8nworkflows.xyz/workflows/automated-lead-to-client-pipeline-with-google-sheets-email-notifications---time-tracking-7514


# Automated Lead-to-Client Pipeline with Google Sheets Email Notifications & Time Tracking

### 1. Workflow Overview

This workflow automates a lead-to-client pipeline integrated with Google Sheets, Gmail, and Cal.com scheduling. It supports CRM activities such as tracking lead stage changes, qualifying leads, booking meetings, and monitoring project delivery times. The workflow is designed to handle real-time updates triggered via webhooks, reflecting changes made in Google Sheets tabs ("Leads" and "Clients"). It includes email notifications for lead stage changes and qualification events, updates Google Sheets accordingly, and calculates project delivery durations.

Logical blocks:

- **1.1 Lead Stage Change Handling:** Receives webhook updates when a lead’s stage changes, prepares email content, sends notification emails, and if the stage is "Won," appends the lead as a client with start date/time.
- **1.2 Lead Qualification Processing:** Triggered by webhook when a lead is marked qualified, sends a booking link email via Gmail.
- **1.3 Meeting Booking Update:** Receives webhook when a meeting is booked, updates the lead stage to "Meeting Booked" in Google Sheets.
- **1.4 Client Project Status Change Handling:** Triggered when a client’s project status changes, especially when marked "Delivered," it calculates the project duration from start to end dates and updates the client record.
- **1.5 Supporting Utilities and Configuration:** Includes Google Apps Script for sending webhook requests on Google Sheets edits, and formatting utilities for timestamps and duration calculations.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Lead Stage Change Handling

- **Overview:**  
  This block listens for webhook notifications when a lead’s stage changes in Google Sheets. It extracts relevant data, prepares a personalized email, sends the email notification, and if the stage is marked as "Won," it appends the lead to the Clients sheet with a start timestamp.

- **Nodes Involved:**  
  - Webhook: Lead Stage Changed  
  - Prepare Email Fields  
  - If (stage equals "Awaiting Proposal")  
  - Gmail: Send Stage-Change Email  
  - IF: Stage == Won  
  - Format Start Timestamp  
  - GS: Clients Append (On Won)

- **Node Details:**

  1. **Webhook: Lead Stage Changed**  
     - Type: Webhook (HTTP POST receiver)  
     - Role: Entry point for lead stage change events from Google Sheets App Script.  
     - Configuration: Listens on path `/lead-stage-changed`, expects JSON with keys: name, email, source_id or source_text, stage (new), previous_stage (optional).  
     - Input: Incoming HTTP JSON POST.  
     - Output: Passes JSON data downstream.  
     - Edge cases: Missing or malformed JSON; webhook URL misconfiguration.

  2. **Prepare Email Fields**  
     - Type: Set node  
     - Role: Extracts firstName (first word of lead name or "there"), sourcePlatform (from source_text fallback to "Unknown"), stage, and previousStage for email content.  
     - Expressions:  
       - `firstName` = first token of `body.name` or "there"  
       - `sourcePlatform` = `body.source_text` or "Unknown"  
       - `stage` and `previousStage` from input JSON  
     - Input: Output of Webhook node.  
     - Output: JSON with augmented fields for email.

  3. **If**  
     - Type: If (conditional)  
     - Role: Branches workflow on lead stage being "Awaiting Proposal".  
     - Condition: `$json.stage == "Awaiting Proposal"`  
     - Input: From Prepare Email Fields.  
     - Output: Two branches; main branch sends email; alternate branch proceeds to IF: Stage == Won node.

  4. **Gmail: Send Stage-Change Email**  
     - Type: Gmail node  
     - Role: Sends an email notifying the lead of next steps after any stage change.  
     - Configuration:  
       - Recipient: lead email from JSON body  
       - Subject: "Next Steps"  
       - Message: Personalized templated HTML message addressing the lead by name.  
       - Credentials: Gmail OAuth2 (configured)  
     - Input: From If node (stage Awaiting Proposal branch).  
     - Output: None (end of branch).  
     - Edge cases: Gmail API auth errors, invalid email addresses.

  5. **IF: Stage == Won**  
     - Type: If (conditional)  
     - Role: Checks if lead stage is "Won" to trigger client onboarding steps.  
     - Condition: `$json.stage == "Won"`  
     - Input: From If node (stage not Awaiting Proposal branch).  
     - Output: True branch proceeds to Format Start Timestamp; false branch does nothing.  
     - Edge cases: Case sensitivity; stage value mismatch.

  6. **Format Start Timestamp**  
     - Type: DateTime node  
     - Role: Formats current timestamp as "DD/MM/YYYY at HH:mm" to record client start date/time.  
     - Input: From IF: Stage == Won node.  
     - Output: Passes formatted date/time.  
     - Edge cases: None significant; relies on system time.

  7. **GS: Clients Append (On Won)**  
     - Type: Google Sheets Append Row  
     - Role: Appends a new row in Clients sheet with fields: Name, Client Email, Project Status (default "In Progress"), Start Date & Time.  
     - Configuration:  
       - Document and sheet IDs from environment variables  
       - Columns mapped explicitly  
       - Append mode with USER_ENTERED format  
       - Credentials: Google Sheets OAuth2 (configured)  
     - Input: From Format Start Timestamp node.  
     - Output: None (end of onboarding branch).  
     - Edge cases: Google Sheets API quota, missing credentials, sheet access errors.

---

#### 1.2 Lead Qualification Processing

- **Overview:**  
  This block handles webhook events when a lead is marked as qualified in the Leads sheet. It verifies the qualification status and sends a Cal.com booking link email to the lead.

- **Nodes Involved:**  
  - Webhook: Lead Qualified?  
  - IF: Is Qualified  
  - Gmail: Send Cal.com Invite

- **Node Details:**

  1. **Webhook: Lead Qualified?**  
     - Type: Webhook (HTTP POST receiver)  
     - Role: Receives updates when the "Qualified?" checkbox in Leads sheet is checked.  
     - Expected JSON keys: name, email, qualified (boolean or "TRUE"/1).  
     - Input: HTTP POST JSON.  
     - Output: Passes data downstream.

  2. **IF: Is Qualified**  
     - Type: If (boolean condition)  
     - Role: Checks if `qualified` is true/truthy.  
     - Condition: `$json.body.qualified` equals true (boolean or string).  
     - Input: From Webhook node.  
     - Output: True branch continues to send email; false branch ends.

  3. **Gmail: Send Cal.com Invite**  
     - Type: Gmail node  
     - Role: Sends a personalized email with a Cal.com booking link inviting the qualified lead to schedule a discovery call.  
     - Configuration:  
       - Recipient: lead email  
       - Subject: "Book Your Discovery Call"  
       - Message: HTML with first name and booking link via `{{CAL_COM_BOOKING_URL}}` environment variable  
       - Credentials: Gmail OAuth2 (configured)  
     - Input: From If node.  
     - Output: None (end of qualification branch).  
     - Edge cases: Gmail API errors, invalid email, missing booking URL.

---

#### 1.3 Meeting Booking Update

- **Overview:**  
  This block updates the Lead stage to "Meeting Booked" upon receiving a webhook indicating a meeting has been scheduled via Cal.com.

- **Nodes Involved:**  
  - Webhook: Meeting Booked  
  - GS: Leads Update Stage to Meeting Booked

- **Node Details:**

  1. **Webhook: Meeting Booked**  
     - Type: Webhook (HTTP POST receiver)  
     - Role: Receives JSON payload from Cal.com or related system when a meeting is booked.  
     - Input: HTTP POST JSON.  
     - Output: Passes JSON downstream.

  2. **GS: Leads Update Stage to Meeting Booked**  
     - Type: Google Sheets Update  
     - Role: Updates the "Stage" column (E) to "Meeting Booked" for the lead identified by email (column C).  
     - Configuration:  
       - Matching column: Client Email  
       - Updates "Stage" to "Meeting Booked"  
       - Uses Google Sheets OAuth2 credentials  
       - Sheet and document IDs from environment variables  
     - Input: From Webhook.  
     - Output: None (end of meeting booking branch).  
     - Edge cases: Email mismatch, sheet access errors, multiple matching rows.

---

#### 1.4 Client Project Status Change Handling

- **Overview:**  
  Handles updates when a Client's Project Status changes, specifically when marked "Delivered." It calculates the delivery duration from Start to End timestamps and updates relevant fields in the Clients sheet.

- **Nodes Involved:**  
  - Webhook: Client Status Changed  
  - IF: Delivered?  
  - GS: Clients Lookup by Email  
  - Format End Timestamp  
  - Code (Calculate Duration)  
  - GS: Clients Update End & Duration

- **Node Details:**

  1. **Webhook: Client Status Changed**  
     - Type: Webhook (HTTP POST receiver)  
     - Role: Receives notification when Clients!D (Project Status) changes.  
     - Expected keys: email, project_status (new value).  
     - Input: HTTP POST JSON.  
     - Output: Passes data downstream.

  2. **IF: Delivered?**  
     - Type: If (string equality check)  
     - Role: Checks if project_status equals "Delivered".  
     - Input: From Webhook.  
     - Output: True branch continues; false branch ends.

  3. **GS: Clients Lookup by Email**  
     - Type: Google Sheets Lookup  
     - Role: Searches Clients sheet for the row matching the client email to retrieve Start Date & Time.  
     - Configuration:  
       - Filter by "Client Email" equals email from webhook  
       - Uses Google Sheets OAuth2 credentials  
       - Document and sheet IDs from environment variables  
     - Input: From IF: Delivered? node.  
     - Output: Client row data for further processing.

  4. **Format End Timestamp**  
     - Type: DateTime node  
     - Role: Formats the current timestamp as "DD/MM/YYYY at HH:mm" for the project end time.  
     - Input: From GS: Clients Lookup by Email.  
     - Output: Formatted end timestamp.

  5. **Code**  
     - Type: Code (JavaScript)  
     - Role: Calculates duration between retrieved Start Date & Time and formatted End Date & Time.  
     - Key logic: Parses dates formatted as "dd/mm/yyyy at HH:mm", computes difference, outputs duration string in "Xd Xh Xm" format or "N/A" if invalid.  
     - Input: From Format End Timestamp and Clients Lookup.  
     - Output: Augments JSON with `endStr` and `duration`.

  6. **GS: Clients Update End & Duration**  
     - Type: Google Sheets Update  
     - Role: Updates the client row with End Date & Time and calculated Time to Deliver.  
     - Configuration:  
       - Matching column: Client Email  
       - Updates columns: End Date & Time, Time to Deliver  
       - Uses Google Sheets OAuth2 credentials  
     - Input: From Code node.  
     - Output: None (end of project fulfillment block).  
     - Edge cases: Date parsing errors, sheet update conflicts.

---

#### 1.5 Supporting Utilities and Configuration

- **Overview:**  
  Provides Google Apps Script code to be installed in the Google Sheets document to send webhook POST requests on specific cell edits in "Leads" and "Clients" tabs. Also includes sticky notes with documentation and instructions.

- **Nodes Involved:**  
  - Sticky Note1 (App Script code)  
  - Sticky Note (title: Google Sheets Automated CRM)  
  - Sticky Note2 (Lead Qualification)  
  - Sticky Note3 (Changing Stage to Meeting Booked)  
  - Sticky Note4 (Proposal Follow-Up / Client Won Flow)  
  - Sticky Note5 (Project Fulfillment Duration)

- **Node Details:**

  1. **Sticky Note1**  
     - Contains the full Google Apps Script code that:  
       - Watches edits on Leads and Clients tabs  
       - Detects changes on Lead Stage, Lead Qualified checkbox, and Client Project Status columns  
       - Posts JSON payloads to corresponding n8n webhook URLs  
     - Instructions for installation and trigger setup included.  
     - Edge cases: Script authorization required, webhook URLs must be replaced correctly.

  2. Other Sticky Notes provide section titles and brief descriptions for workflow blocks to aid user understanding.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                             | Input Node(s)                   | Output Node(s)                        | Sticky Note                                                                                      |
|-------------------------------|---------------------|--------------------------------------------|--------------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------|
| Webhook: Lead Stage Changed    | Webhook             | Entry point for lead stage change events   | —                              | Prepare Email Fields                 | Receives JSON when a lead's Stage (Leads!E) changes. Expected keys: name, email, source_id/text, stage, previous_stage |
| Prepare Email Fields           | Set                 | Prepares email variables                    | Webhook: Lead Stage Changed    | If                                  | Derives firstName and sourcePlatform for email body                                              |
| If                            | If                  | Checks if stage is "Awaiting Proposal"     | Prepare Email Fields            | Gmail: Send Stage-Change Email, IF: Stage == Won |                                                                                                 |
| Gmail: Send Stage-Change Email| Gmail               | Sends notification email on stage change   | If                            | —                                   | Sends templated email on any Stage change                                                       |
| IF: Stage == Won              | If                  | Branch if lead stage is "Won"               | If                            | Format Start Timestamp               | Branch for when the lead has been marked Won                                                    |
| Format Start Timestamp         | DateTime            | Formats current timestamp                   | IF: Stage == Won               | GS: Clients Append (On Won)          | Produces Start Date & Time as: dd/mm/yyyy at HH:MM                                              |
| GS: Clients Append (On Won)    | Google Sheets       | Appends new client row on Won stage         | Format Start Timestamp         | —                                   | Appends client with Project Status=In Progress and Start Date & Time                            |
| Webhook: Lead Qualified?       | Webhook             | Entry point for lead qualification event   | —                             | IF: Is Qualified                    | Receives JSON when Leads!H (Qualified?) is checked                                              |
| IF: Is Qualified              | If                  | Checks if lead is qualified                 | Webhook: Lead Qualified?       | Gmail: Send Cal.com Invite           |                                                                                                 |
| Gmail: Send Cal.com Invite     | Gmail               | Sends discovery call booking link email    | IF: Is Qualified              | —                                   | Sends booking link when Qualified? is checked                                                   |
| Webhook: Meeting Booked        | Webhook             | Entry point for meeting booking event       | —                             | GS: Leads Update Stage to Meeting Booked |                                                                                                 |
| GS: Leads Update Stage to Meeting Booked | Google Sheets       | Updates lead stage to "Meeting Booked"      | Webhook: Meeting Booked        | —                                   | Uses Email as key to update Stage to 'Meeting Booked'                                          |
| Webhook: Client Status Changed | Webhook             | Entry point for client project status change| —                             | IF: Delivered?                      | Receives JSON when Clients!D (Project Status) changes                                          |
| IF: Delivered?                | If                  | Checks if project status is "Delivered"    | Webhook: Client Status Changed | GS: Clients Lookup by Email          |                                                                                                 |
| GS: Clients Lookup by Email    | Google Sheets       | Looks up client row by email                 | IF: Delivered?                | Format End Timestamp                 | Fetches row to read Start Date & Time for duration calculation                                 |
| Format End Timestamp           | DateTime            | Formats current end timestamp               | GS: Clients Lookup by Email    | Code                               | End Date & Time formatted as dd/mm/yyyy at HH:mm                                               |
| Code                         | Code (JavaScript)   | Calculates delivery duration                 | Format End Timestamp           | GS: Clients Update End & Duration    | Computes duration from Start and End Date & Time                                              |
| GS: Clients Update End & Duration | Google Sheets       | Updates client with project end and duration | Code                         | —                                   | Sets End Date & Time and Time to Deliver                                                      |
| Sticky Note                   | Sticky Note         | Title and notes                             | —                             | —                                   | # Google Sheets Automated CRM                                                                  |
| Sticky Note1                  | Sticky Note         | Contains Google Apps Script code            | —                             | —                                   | # The App Script Needed (Use this for your Google Worksheet App Script)                         |
| Sticky Note2                  | Sticky Note         | Section title: Lead Qualification           | —                             | —                                   | # Lead Qualification                                                                          |
| Sticky Note3                  | Sticky Note         | Section title: Changing Stage to Meeting Booked | —                             | —                                   | # Changing Stage to Meeting Booked                                                             |
| Sticky Note4                  | Sticky Note         | Section title: Proposal Follow-Up / Client Won Flow | —                             | —                                   | # Proposal Follow-Up / Client Won Flow                                                         |
| Sticky Note5                  | Sticky Note         | Section title: Project Fulfillment Duration | —                             | —                                   | # Project Fulfillment Duration                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook: Lead Stage Changed**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `lead-stage-changed`  
   - Response: "OK"  
   - Purpose: Receive JSON when a lead stage changes in Google Sheets.

2. **Create Node: Prepare Email Fields**  
   - Type: Set  
   - Fields to set:  
     - `firstName`: Expression extracting first word from `body.name` or default "there"  
     - `sourcePlatform`: `body.source_text` or "Unknown"  
     - `stage`: `body.stage`  
     - `previousStage`: `body.previous_stage` or empty string  
   - Connect output of Webhook: Lead Stage Changed to this node.

3. **Create Node: If (Check Awaiting Proposal)**  
   - Type: If  
   - Condition: `$json.stage == "Awaiting Proposal"`  
   - Connect output of Prepare Email Fields to this node.

4. **Create Node: Gmail - Send Stage-Change Email**  
   - Type: Gmail  
   - Send To: `{{ $json.body.email }}`  
   - Subject: "Next Steps"  
   - Message (HTML): Personalized with lead name and static text about proposal delivery.  
   - Credentials: Set up Gmail OAuth2 credentials in n8n.  
   - Connect True output of "If Awaiting Proposal" to this node.

5. **Create Node: IF: Stage == Won**  
   - Type: If  
   - Condition: `$json.stage == "Won"`  
   - Connect False output of "If Awaiting Proposal" to this node.

6. **Create Node: Format Start Timestamp**  
   - Type: DateTime  
   - Value: `{{$now}}` (current time)  
   - Format: `DD/MM/YYYY 'at' HH:mm`  
   - Connect True output of "IF: Stage == Won" to this node.

7. **Create Node: GS: Clients Append (On Won)**  
   - Type: Google Sheets (Append)  
   - Document ID and Sheet Name: Use environment variables or correct IDs for your sheet.  
   - Columns to append:  
     - Name: `{{$json.body.name}}`  
     - Client Email: `{{$json.body.email}}`  
     - Project Status: "In Progress"  
     - Start Date & Time: `{{$json.data}}` (from Format Start Timestamp)  
   - Credentials: Google Sheets OAuth2  
   - Connect output of Format Start Timestamp to this node.

8. **Create Webhook: Lead Qualified?**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `lead-qualified`  
   - Response: "OK"

9. **Create Node: IF: Is Qualified**  
   - Type: If  
   - Condition: `$json.body.qualified` equals boolean true or truthy string  
   - Connect output of Webhook: Lead Qualified? to this node.

10. **Create Node: Gmail - Send Cal.com Invite**  
    - Type: Gmail  
    - Send To: `{{$json.body.email}}`  
    - Subject: "Book Your Discovery Call"  
    - Message: Personalized HTML with first name and Cal.com booking link (`{{CAL_COM_BOOKING_URL}}`).  
    - Credentials: Gmail OAuth2  
    - Connect True output of IF: Is Qualified to this node.

11. **Create Webhook: Meeting Booked**  
    - Type: Webhook  
    - HTTP Method: POST  
    - Path: `cal-booked`  
    - Response: Default

12. **Create Node: GS: Leads Update Stage to Meeting Booked**  
    - Type: Google Sheets (Update)  
    - Document and Sheet IDs from config  
    - Matching Column: Client Email  
    - Columns to update: Stage = "Meeting Booked"  
    - Credentials: Google Sheets OAuth2  
    - Connect output of Webhook: Meeting Booked to this node.

13. **Create Webhook: Client Status Changed**  
    - Type: Webhook  
    - HTTP Method: POST  
    - Path: `client-status-changed`  
    - Response: Default

14. **Create Node: IF: Delivered?**  
    - Type: If  
    - Condition: `$json.body.project_status == "Delivered"`  
    - Connect output of Webhook: Client Status Changed to this node.

15. **Create Node: GS: Clients Lookup by Email**  
    - Type: Google Sheets (Lookup)  
    - Filter: Client Email equals `$json.body.email`  
    - Credentials: Google Sheets OAuth2  
    - Connect True output of IF: Delivered? to this node.

16. **Create Node: Format End Timestamp**  
    - Type: DateTime  
    - Value: `{{$now}}`  
    - Format: `DD/MM/YYYY 'at' HH:mm`  
    - Connect output of GS: Clients Lookup by Email to this node.

17. **Create Node: Code (Calculate Duration)**  
    - Type: Code (JavaScript)  
    - Paste JavaScript code that:  
      - Parses Start Date & Time from lookup result  
      - Parses formatted End Date & Time from previous node  
      - Calculates duration in days, hours, and minutes  
      - Adds `endStr` and `duration` fields to output JSON  
    - Connect output of Format End Timestamp to this node.

18. **Create Node: GS: Clients Update End & Duration**  
    - Type: Google Sheets (Update)  
    - Document and Sheet IDs from config  
    - Matching Column: Client Email  
    - Columns to update:  
      - End Date & Time (`endStr` from Code node)  
      - Time to Deliver (`duration` from Code node)  
    - Credentials: Google Sheets OAuth2  
    - Connect output of Code node to this node.

19. **Set up Google Apps Script**  
    - In your Google Sheets document (named e.g., "ScaleFlow CRM"), open Extensions → Apps Script.  
    - Paste the provided Apps Script code (from Sticky Note1 content).  
    - Replace webhook URLs with your actual n8n webhook endpoints.  
    - Save and authorize the script.  
    - Optionally run `createInstallableTrigger()` once to enable full authorization triggers to handle edits.  

20. **Configure Credentials and Environment Variables**  
    - Gmail OAuth2 credentials for sending emails.  
    - Google Sheets OAuth2 credentials with access to your CRM spreadsheet.  
    - Environment variables or constants for:  
      - GOOGLE_SHEETS_DOC_ID  
      - CLIENTS_GID (Clients sheet gid)  
      - LEADS_GID (Leads sheet gid)  
      - CAL_COM_BOOKING_URL (discovery call booking link)  
      - SENDER_NAME (email sender display name)  
      - WEBHOOK_IDs for each webhook node.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Google Apps Script acts as a bridge to n8n, posting webhook calls on Google Sheets edits for real-time sync.  | Included in Sticky Note1 with installation instructions                                                  |
| Keeps project data in sync without polling by using onEdit triggers in Sheets.                                | Script includes optional installable trigger to overcome simple trigger limitations                      |
| Email templates are simple HTML with placeholders for personalization using n8n expressions.                  | Gmail nodes use OAuth2 credentials; ensure tokens are valid and refreshed                                |
| Cal.com booking link sends discovery call invites to qualified leads automatically.                           | Set `CAL_COM_BOOKING_URL` environment variable before use                                               |
| Duration calculation uses custom JS code parsing localized date/time formats ("dd/mm/yyyy at HH:mm").         | May require localization adjustment if your Sheets use different date formats                            |
| Workflow designed to be modular, allowing extension with other automation steps (e.g., proposals, invoicing). | Sticky notes clearly label each logical block for easier maintenance and extension                       |
| For detailed Google Sheets column setup, refer to Apps Script comments for column indexes and tabs used.      | Tabs: "Leads" and "Clients" with specified columns (A-H)                                                |

---

This documentation comprehensively describes the workflow’s structure, logic, node configurations, and environment setup, enabling advanced users or automation agents to fully understand, reproduce, and maintain the lead-to-client pipeline automation.