Smart Meeting Rescheduler: Gmail + Calendar + Sheets Automation

https://n8nworkflows.xyz/workflows/smart-meeting-rescheduler--gmail---calendar---sheets-automation-8509


# Smart Meeting Rescheduler: Gmail + Calendar + Sheets Automation

---
### 1. Workflow Overview

This workflow, titled **"No-Show Follow-Up and Rescheduling"**, automates the management of meeting no-shows by integrating Gmail, Google Calendar, and Google Sheets. It identifies unbooked or missed meetings, sends follow-up emails, creates calendar placeholders for rescheduling, waits for user responses, and updates meeting records in a Google Sheet. The workflow is designed for use cases where meeting attendance tracking and timely follow-ups are critical, such as sales, recruitment, or client onboarding.

The workflow is logically divided into the following blocks:

- **1.1 Input Trigger and Data Retrieval:** Manual start and fetching meeting data from Google Sheets.
- **1.2 No-Show Check and Initial Follow-Up:** Decision node to identify unbooked meetings and send follow-up emails.
- **1.3 Calendar Placeholder Creation and Delay:** Creating a calendar event placeholder and pausing execution for 24 hours.
- **1.4 Email Thread Monitoring and Conditional Follow-Up:** Fetching Gmail threads, checking conditions, sending reminder emails, and waiting again.
- **1.5 Data Update in Google Sheets:** Modifying and appending/updating rows in Google Sheets to record outcomes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger and Data Retrieval

- **Overview:** Starts the workflow manually and retrieves meeting data rows from a Google Sheet to process.
- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’`  
  - `Get row(s) in sheet1`  
  - `Check If Unbooked`

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow manually to start the process.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None (manual start)  
    - Outputs: Triggers `Get row(s) in sheet1`  
    - Edge Cases: None inherent; requires manual intervention to start.

  - **Get row(s) in sheet1**  
    - Type: Google Sheets (Read Operation)  
    - Role: Reads data rows from a specified Google Sheet containing meeting or scheduling records.  
    - Configuration: Set up to fetch relevant rows; likely filtered or limited to necessary columns.  
    - Expressions: Uses sheet name and key columns (not explicitly detailed).  
    - Inputs: Triggered by manual trigger node.  
    - Outputs: Flows into `Check If Unbooked` node.  
    - Edge Cases: Sheet access errors, empty rows, or permission/authentication failures.

  - **Check If Unbooked**  
    - Type: If Node (Conditional Branch)  
    - Role: Determines if a meeting is unbooked/no-show by evaluating sheet data.  
    - Configuration: Condition based on field(s) indicating booking status (e.g., empty booking field).  
    - Inputs: Rows from Google Sheets node.  
    - Outputs: True branch to `Send Follow-up Email` node; false branch ends or is not connected.  
    - Edge Cases: Expression failures if expected fields are missing or null.

#### 2.2 No-Show Check and Initial Follow-Up

- **Overview:** If a meeting is unbooked, a follow-up email is sent to notify or prompt the participant.
- **Nodes Involved:**  
  - `Send Follow-up Email`  
  - `Create Calendar Placeholder`  
  - `Wait 24hr`

- **Node Details:**

  - **Send Follow-up Email**  
    - Type: Gmail Node (Send Email)  
    - Role: Sends an initial follow-up email to the meeting invitee regarding the no-show or unbooked meeting.  
    - Configuration: Uses Gmail OAuth2 credentials; email template likely includes personalized info from sheet data.  
    - Inputs: From `Check If Unbooked` (true branch).  
    - Outputs: Triggers `Create Calendar Placeholder`.  
    - Edge Cases: Gmail API rate limits, authentication errors, missing recipient email.

  - **Create Calendar Placeholder**  
    - Type: Google Calendar (Create Event)  
    - Role: Creates a placeholder event in the calendar to propose a new meeting time or block time for rescheduling.  
    - Configuration: Event details derived from sheet data or default placeholders.  
    - Inputs: From `Send Follow-up Email`.  
    - Outputs: Triggers `Wait 24hr`.  
    - Edge Cases: Calendar API permission issues, invalid event data.

  - **Wait 24hr**  
    - Type: Wait Node  
    - Role: Pauses workflow execution for 24 hours to allow time for recipient response or scheduling updates.  
    - Configuration: Fixed 24-hour wait duration.  
    - Inputs: From `Create Calendar Placeholder`.  
    - Outputs: Triggers `Get a thread`.  
    - Edge Cases: Workflow timeouts if execution limits are exceeded; long wait may require stable workflow environment.

#### 2.3 Email Thread Monitoring and Conditional Follow-Up

- **Overview:** After waiting, the workflow checks Gmail threads for responses or actions, conditionally sends further follow-ups, and waits again if necessary.
- **Nodes Involved:**  
  - `Get a thread`  
  - `If`  
  - `Send Follow-up Email1`  
  - `Wait 24 hr 1`  
  - `Get a thread1`  
  - `If1`  
  - `Edit Fields`

- **Node Details:**

  - **Get a thread**  
    - Type: Gmail Node (Get Email Thread)  
    - Role: Retrieves the email thread to assess whether the recipient has responded or engaged.  
    - Configuration: Uses thread ID or search criteria to fetch relevant messages.  
    - Inputs: From `Wait 24hr`.  
    - Outputs: Flows into `If`.  
    - Edge Cases: Thread not found, Gmail API limits.

  - **If**  
    - Type: If Node (Conditional Branch)  
    - Role: Checks conditions on the email thread, e.g., whether a reply exists or a keyword is found.  
    - Configuration: Evaluates message status or content.  
    - Inputs: From `Get a thread`.  
    - Outputs: True branch to `Send Follow-up Email1`, false branch ends or is not connected.  
    - Edge Cases: Expression errors if thread data is malformed.

  - **Send Follow-up Email1**  
    - Type: Gmail Node (Send Email)  
    - Role: Sends a second follow-up email if no satisfactory response was detected.  
    - Configuration: Uses Gmail OAuth2; possibly a different email template from the first follow-up.  
    - Inputs: From `If` (true branch).  
    - Outputs: Triggers `Wait 24 hr 1`.  
    - Edge Cases: Same as first email node.

  - **Wait 24 hr 1**  
    - Type: Wait Node  
    - Role: Another 24-hour wait to allow for recipient response after second follow-up.  
    - Configuration: Fixed 24-hour delay.  
    - Inputs: From `Send Follow-up Email1`.  
    - Outputs: Triggers `Get a thread1`.  
    - Edge Cases: Same as previous wait node.

  - **Get a thread1**  
    - Type: Gmail Node (Get Email Thread)  
    - Role: Checks the email thread again after the second wait.  
    - Configuration: Same as `Get a thread`.  
    - Inputs: From `Wait 24 hr 1`.  
    - Outputs: Flows into `If1`.  
    - Edge Cases: Same as `Get a thread`.

  - **If1**  
    - Type: If Node (Conditional Branch)  
    - Role: Checks updated thread status or content to decide whether to proceed with updating records.  
    - Configuration: Conditional on thread response or other flags.  
    - Inputs: From `Get a thread1`.  
    - Outputs: True branch to `Edit Fields`.  
    - Edge Cases: Expression or data integrity failures.

  - **Edit Fields**  
    - Type: Set Node  
    - Role: Prepares or modifies data fields to update the Google Sheet with the latest meeting or follow-up status.  
    - Configuration: Sets or updates fields such as status flags, timestamps, or notes.  
    - Inputs: From `If1` (true branch).  
    - Outputs: Triggers `Append or update row in sheet`.  
    - Edge Cases: Data mapping errors, missing fields.

#### 2.4 Data Update in Google Sheets

- **Overview:** Finalizes the process by recording the updated meeting status or notes in the Google Sheet.
- **Nodes Involved:**  
  - `Append or update row in sheet`

- **Node Details:**

  - **Append or update row in sheet**  
    - Type: Google Sheets (Write Operation)  
    - Role: Inserts or updates the corresponding row with new data reflecting follow-ups and scheduling results.  
    - Configuration: Targets specific sheet and range; uses key to find existing rows or appends new ones.  
    - Inputs: From `Edit Fields`.  
    - Outputs: Terminal node (no outputs).  
    - Edge Cases: Write permission errors, row conflicts or duplicates.

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                              | Input Node(s)                  | Output Node(s)                 | Sticky Note |
|---------------------------|------------------------|----------------------------------------------|------------------------------|-------------------------------|-------------|
| When clicking ‘Execute workflow’ | Manual Trigger          | Workflow manual start trigger                  | None                         | Get row(s) in sheet1           |             |
| Get row(s) in sheet1      | Google Sheets           | Reads meeting data from Google Sheet          | When clicking ‘Execute workflow’ | Check If Unbooked              |             |
| Check If Unbooked         | If Node                 | Checks if meeting is unbooked/no-show          | Get row(s) in sheet1          | Send Follow-up Email           |             |
| Send Follow-up Email      | Gmail (Send Email)      | Sends initial follow-up email                   | Check If Unbooked             | Create Calendar Placeholder    |             |
| Create Calendar Placeholder | Google Calendar         | Creates calendar event placeholder              | Send Follow-up Email          | Wait 24hr                     |             |
| Wait 24hr                 | Wait Node               | Waits 24 hours for recipient response          | Create Calendar Placeholder   | Get a thread                  |             |
| Get a thread              | Gmail (Get Thread)      | Retrieves email thread to check for responses  | Wait 24hr                    | If                            |             |
| If                        | If Node                 | Checks if response received in email thread    | Get a thread                 | Send Follow-up Email1          |             |
| Send Follow-up Email1     | Gmail (Send Email)      | Sends second follow-up email if no response     | If (true branch)             | Wait 24 hr 1                  |             |
| Wait 24 hr 1              | Wait Node               | Waits another 24 hours after second follow-up  | Send Follow-up Email1         | Get a thread1                 |             |
| Get a thread1             | Gmail (Get Thread)      | Retrieves email thread again after wait         | Wait 24 hr 1                 | If1                           |             |
| If1                       | If Node                 | Checks thread response again                      | Get a thread1                | Edit Fields                   |             |
| Edit Fields               | Set Node                | Prepares data fields for Google Sheets update   | If1 (true branch)            | Append or update row in sheet  |             |
| Append or update row in sheet | Google Sheets           | Updates or appends row with new meeting data    | Edit Fields                  | None                          |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: To manually start the workflow.

2. **Add Google Sheets Node to Get Rows**  
   - Name: `Get row(s) in sheet1`  
   - Operation: Read rows from your Google Sheet containing meeting data.  
   - Configure Google Sheets credentials and specify sheet name and range.

3. **Add an If Node to Check Booking Status**  
   - Name: `Check If Unbooked`  
   - Condition: Check if the row indicates the meeting is unbooked (e.g., booking status field is empty or false).  
   - Connect input from `Get row(s) in sheet1`.

4. **Add Gmail Node to Send Initial Follow-up Email**  
   - Name: `Send Follow-up Email`  
   - Action: Send email to meeting participant notifying of no-show.  
   - Configure Gmail OAuth2 credentials; prepare email template with dynamic data from sheet.  
   - Connect input from `Check If Unbooked` (true branch).

5. **Add Google Calendar Node to Create Placeholder Event**  
   - Name: `Create Calendar Placeholder`  
   - Action: Create a placeholder event for rescheduling.  
   - Configure Google Calendar credentials; set event details dynamically.  
   - Connect input from `Send Follow-up Email`.

6. **Add Wait Node for 24 Hours**  
   - Name: `Wait 24hr`  
   - Action: Pause workflow for 24 hours.  
   - Connect input from `Create Calendar Placeholder`.

7. **Add Gmail Node to Get Email Thread**  
   - Name: `Get a thread`  
   - Action: Retrieve email thread to check for replies.  
   - Configure Gmail credentials; specify thread ID or search parameters.  
   - Connect input from `Wait 24hr`.

8. **Add If Node to Check Email Thread Response**  
   - Name: `If`  
   - Condition: Check if a response has been received in the email thread.  
   - Connect input from `Get a thread`.

9. **Add Gmail Node to Send Second Follow-up Email**  
   - Name: `Send Follow-up Email1`  
   - Action: Send reminder email if no response received.  
   - Configure Gmail credentials and template.  
   - Connect input from `If` (true branch).

10. **Add Wait Node for Another 24 Hours**  
    - Name: `Wait 24 hr 1`  
    - Action: Pause workflow 24 more hours after second follow-up.  
    - Connect input from `Send Follow-up Email1`.

11. **Add Gmail Node to Get Email Thread Again**  
    - Name: `Get a thread1`  
    - Action: Retrieve email thread again to check for late responses.  
    - Configure Gmail credentials.  
    - Connect input from `Wait 24 hr 1`.

12. **Add If Node to Check Thread After Second Follow-up**  
    - Name: `If1`  
    - Condition: Check if response exists now.  
    - Connect input from `Get a thread1`.

13. **Add Set Node to Edit Data Fields**  
    - Name: `Edit Fields`  
    - Action: Prepare data fields to update Google Sheet row with final status or notes.  
    - Configure fields to set based on conditions and email thread data.  
    - Connect input from `If1` (true branch).

14. **Add Google Sheets Node to Append or Update Row**  
    - Name: `Append or update row in sheet`  
    - Action: Write updated data back to Google Sheet.  
    - Configure sheet name, range, and keys for update or append.  
    - Connect input from `Edit Fields`.

15. **Link all nodes according to the above connection order.**

16. **Configure all credentials:**  
    - Gmail OAuth2 with Send/Read permissions  
    - Google Calendar OAuth2 with Event creation permissions  
    - Google Sheets OAuth2 with read/write permissions

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow automates follow-ups for no-show meetings with staged email reminders and calendar updates. | Workflow description and use case                |
| Wait nodes are used to allow ample time for recipients to respond before subsequent follow-ups. | Important to consider n8n workflow execution limits |
| Gmail thread retrieval nodes rely on correct thread IDs or search criteria to function properly. | Gmail API documentation for thread management    |
| Google Sheets nodes must have correct sheet access and range configured to avoid errors.         | Google Sheets API and n8n Google Sheets node docs |
| Ensure OAuth2 credentials are properly set up for Gmail, Calendar, and Sheets for seamless operation. | n8n credential setup guides                        |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. The processing strictly respects current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.