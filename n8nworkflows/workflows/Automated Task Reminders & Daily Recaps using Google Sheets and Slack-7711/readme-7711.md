Automated Task Reminders & Daily Recaps using Google Sheets and Slack

https://n8nworkflows.xyz/workflows/automated-task-reminders---daily-recaps-using-google-sheets-and-slack-7711


# Automated Task Reminders & Daily Recaps using Google Sheets and Slack

### 1. Workflow Overview

This workflow automates task reminders and daily progress recaps by integrating Google Sheets as a task database and Slack as the communication channel. It targets individual or team productivity management, enabling timely alerts about upcoming task deadlines and providing end-of-day summaries of completed versus pending tasks.

The workflow is logically divided into two main functional blocks:

- **1.1 Automated Task Reminder Cycle:**  
  Periodically (every 15 minutes) fetches tasks from Google Sheets, checks if any are due within the next 30 minutes, sends Slack reminders for those tasks, and updates the sheet to prevent duplicate alerts.

- **1.2 Daily Recap Cycle:**  
  Triggers once daily at 6 PM to fetch all tasks, analyze their statuses, and send a Slack message summarizing completed and pending tasks for the day.

Each block is triggered by its own Cron node and uses Google Sheets nodes for data retrieval and update, combined with Slack nodes for messaging.

---

### 2. Block-by-Block Analysis

#### 2.1 Automated Task Reminder Cycle

- **Overview:**  
  This block runs every 15 minutes to identify tasks approaching their due time and sends Slack reminders to the relevant channel, then marks those tasks as reminded to avoid repeated notifications.

- **Nodes Involved:**  
  - Start: Cron Trigger  
  - Fetch Tasks from Google Sheets  
  - Check Task Deadlines (IF)  
  - Send a Slack Reminder  
  - Update Last Reminder Sent

- **Node Details:**

  - **Start: Cron Trigger**  
    - Type: Cron Trigger  
    - Role: Initiates the workflow every 15 minutes to check task deadlines.  
    - Configuration: Default 15-minute interval (implied by note).  
    - Inputs: None  
    - Outputs: To "Fetch Tasks from Google Sheets"  
    - Failures: Cron trigger errors are rare but could occur if system time misconfigured.

  - **Fetch Tasks from Google Sheets**  
    - Type: Google Sheets node (read operation)  
    - Role: Retrieves all rows from the “Tasks” sheet tab.  
    - Configuration: Document ID and Sheet Tab are dynamically set via variables `GOOGLE_SHEET_WORKSHEET` and `SHEET_TAB`.  
    - Credentials: Uses Google Sheets OAuth2 credentials identified as "YOUR_GOOGLE_SHEET_ACCOUNT".  
    - Inputs: From Cron Trigger  
    - Outputs: To "Check Task Deadlines"  
    - Failures: Possible auth failures, sheet access errors, or missing worksheet/tab.  
    - Edge Cases: Empty rows or malformed data in date fields.

  - **Check Task Deadlines (IF Node)**  
    - Type: If node  
    - Role: Filters tasks due within the next 30 minutes.  
    - Configuration: Compares "Due Date" field against current time plus 30 minutes, using date-time comparison with format `yyyy-MM-dd hh:mm`.  
    - Inputs: Task rows from Google Sheets  
    - Outputs: Two branches—true (due soon) to send reminder; false ignored.  
    - Failures: Expression evaluation failures if "Due Date" missing or improperly formatted.  
    - Edge Cases: Timezone differences, tasks without due dates, or past due dates.

  - **Send a Slack Reminder**  
    - Type: Slack node  
    - Role: Sends a formatted reminder message to a Slack channel about an imminent task.  
    - Configuration: Message text includes Task Name, Due Date, and Why it matters; channel selected dynamically (parameter not hardcoded).  
    - Credentials: Slack API OAuth2 under "YOUR_SLACK".  
    - Inputs: Filtered tasks from IF node.  
    - Outputs: To "Update Last Reminder Sent"  
    - Failures: Slack API rate limits, invalid channel ID, or credential expiration.  
    - Edge Cases: Task fields missing or empty, causing incomplete messages.

  - **Update Last Reminder Sent**  
    - Type: Google Sheets node (update operation)  
    - Role: Updates the “Last Reminder Sent” column with the current timestamp to prevent duplicate reminders.  
    - Configuration: Matches on "Task ID", updates "Last Reminder Sent" with current time formatted as `yyyy-MM-dd HH:mm`. Uses dynamic `GID` for document ID.  
    - Credentials: Same Google Sheets OAuth2 credentials.  
    - Inputs: Output from Slack reminder node.  
    - Outputs: None (end of this branch)  
    - Failures: Sheet update errors, concurrency issues if multiple updates overlap.  
    - Edge Cases: Missing or duplicate Task IDs could cause update mismatches.

---

#### 2.2 Daily Recap Cycle

- **Overview:**  
  Runs once daily at 6 PM to summarize task progress by fetching all tasks and sending a recap Slack message indicating how many tasks were completed and how many remain pending.

- **Nodes Involved:**  
  - Daily Recap Trigger  
  - Fetch Completed Tasks  
  - Send a Slack Reminder1 (Daily Recap message)

- **Node Details:**

  - **Daily Recap Trigger**  
    - Type: Cron Trigger  
    - Role: Starts the daily recap workflow at 6 PM each day.  
    - Configuration: Scheduled daily at 18:00 (assumed from notes).  
    - Inputs: None  
    - Outputs: To "Fetch Completed Tasks"  
    - Failures: Similar to other cron triggers; rare system or time misconfiguration.

  - **Fetch Completed Tasks**  
    - Type: Google Sheets node (read operation)  
    - Role: Retrieves the entire “Tasks” sheet to assess task statuses for the recap.  
    - Configuration: Uses fixed document and sheet IDs, reading from gid=0 (first sheet).  
    - Credentials: Same Google Sheets OAuth2 credentials.  
    - Inputs: From daily cron trigger  
    - Outputs: To "Send a Slack Reminder1"  
    - Failures: Auth errors, access restrictions, or sheet data issues.

  - **Send a Slack Reminder1**  
    - Type: Slack node  
    - Role: Sends a formatted message summarizing number of completed and pending tasks.  
    - Configuration: Message uses JavaScript expressions to count occurrences of "Completed" versus "In Progress" or "Not Started" statuses in the "Status" field.  
    - Credentials: Slack API OAuth2  
    - Inputs: Tasks data from Sheets  
    - Outputs: None (end of workflow)  
    - Failures: Slack API errors, message formatting errors if fields missing.

---

#### 2.3 Documentation and Setup Note

- **Sticky Note Node**  
  - Type: Sticky Note  
  - Role: Provides comprehensive documentation for Google Sheets setup, Slack integration, workflow logic, benefits, and use cases.  
  - Content:  
    - Instructions for Google Sheets tabs and columns required  
    - Slack integration setup and channel configuration  
    - Workflow node sequencing explanation  
    - Benefits and practical use cases  
  - Position: Off to the side for reference only, no functional impact.

---

### 3. Summary Table

| Node Name                 | Node Type             | Functional Role                             | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                   |
|---------------------------|-----------------------|---------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| Start: Cron Trigger       | Cron Trigger          | Triggers task check every 15 minutes        | -                           | Fetch Tasks from Google Sheets| Runs every 15 minutes to check tasks in Google Sheets.                                                      |
| Fetch Tasks from Google Sheets | Google Sheets      | Reads all tasks from Tasks sheet             | Start: Cron Trigger          | Check Task Deadlines         | Reads all tasks from the `Tasks` sheet.                                                                     |
| Check Task Deadlines      | If Node               | Filters tasks due in next 30 minutes         | Fetch Tasks from Google Sheets| Send a Slack Reminder, Update Last Reminder Sent| Checks if task is due within the next 30 minutes.                                                           |
| Send a Slack Reminder     | Slack Node            | Sends reminder message for due tasks         | Check Task Deadlines (true)  | Update Last Reminder Sent    | Posts reminders with task name, due date, and reason.                                                       |
| Update Last Reminder Sent | Google Sheets (Update)| Marks tasks as reminded with timestamp       | Send a Slack Reminder        | -                           | Updates the `Last Reminder Sent` column in Sheets to avoid duplicate reminders.                              |
| Daily Recap Trigger       | Cron Trigger          | Triggers daily recap at 6 PM                  | -                           | Fetch Completed Tasks        | Triggers daily recap at 6 PM.                                                                                |
| Fetch Completed Tasks     | Google Sheets         | Reads all tasks for daily recap               | Daily Recap Trigger          | Send a Slack Reminder1       | Reads all tasks for recap.                                                                                   |
| Send a Slack Reminder1    | Slack Node            | Sends daily summary of completed vs pending  | Fetch Completed Tasks        | -                           | Posts a daily summary with completed vs pending tasks.                                                      |
| Sticky Note               | Sticky Note           | Documentation and setup instructions          | -                           | -                           | # 📝 Rize Lite – Google Sheets + Slack Productivity Tracker (full detailed setup and benefits information)  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Setup:**

   - Create a Google Sheet with at least one tab named "Tasks".
   - Columns should include:  
     `Task ID`, `Task Name`, `Assigned To`, `Start Time`, `End Time`, `Duration (mins)`, `Due Date` (format: yyyy-MM-dd HH:mm), `Status` (values: Not Started, In Progress, Completed), `Last Reminder Sent`, `Why it matters`.

2. **Connect Google Sheets Credentials in n8n:**

   - Set up OAuth2 credentials for Google Sheets and name as "YOUR_GOOGLE_SHEET_ACCOUNT".
   - Ensure the n8n instance has access to the Google Sheet document.

3. **Create Slack Credentials in n8n:**

   - Set up Slack API OAuth2 credentials named "YOUR_SLACK".
   - Configure the Slack app with permissions to post messages to channels.
   - Note your target Slack channel name or ID.

4. **Add Node: Start: Cron Trigger**

   - Type: Cron Trigger  
   - Set to run every 15 minutes.  
   - No input parameters.

5. **Add Node: Fetch Tasks from Google Sheets**

   - Type: Google Sheets (Read)  
   - Document ID: Set to your Google Sheet document ID (use a variable or fixed string).  
   - Sheet Name: Set to "Tasks" tab or dynamically via workflow variables.  
   - Use OAuth2 credentials "YOUR_GOOGLE_SHEET_ACCOUNT".  
   - Connect input from "Start: Cron Trigger".

6. **Add Node: Check Task Deadlines (If Node)**

   - Type: If  
   - Condition:  
     - DateTime check where:  
       `Due Date` < current time plus 30 minutes  
     - Use expression:  
       ```  
       {{$json["Due Date"]}} < {{$now.plus({ minutes: 30 }).format('yyyy-MM-dd hh:mm')}}  
       ```  
   - Connect input from "Fetch Tasks from Google Sheets".

7. **Add Node: Send a Slack Reminder**

   - Type: Slack  
   - Credentials: "YOUR_SLACK"  
   - Message text:  
     ```
     ⚡ Reminder: Task *{{$json["Task Name"]}}* is due at {{$json["Due Date"]}}.|Reason: {{$json["Why it matters"]}}
     ```  
   - Channel: Select appropriate Slack channel parameter.  
   - Connect input from "Check Task Deadlines" true output.

8. **Add Node: Update Last Reminder Sent**

   - Type: Google Sheets (Update)  
   - Document ID: Same as before (variable or fixed).  
   - Sheet Name: "Tasks" tab or gid=0.  
   - Operation: Update row matching `Task ID` with current timestamp in `Last Reminder Sent` column:  
     ```
     {{$now.format('yyyy-MM-dd HH:mm')}}
     ```  
   - Connect input from "Send a Slack Reminder".

9. **Add Node: Daily Recap Trigger**

   - Type: Cron Trigger  
   - Schedule: Daily at 18:00 (6 PM).  
   - No input.

10. **Add Node: Fetch Completed Tasks**

    - Type: Google Sheets (Read)  
    - Document ID and Sheet Name same as above.  
    - Connect input from "Daily Recap Trigger".

11. **Add Node: Send a Slack Reminder1 (Daily Recap)**

    - Type: Slack  
    - Credentials: "YOUR_SLACK"  
    - Message text:  
      ```
      📊 Daily Recap:
      Completed tasks today: {{ ($json["Status"] || "").match(/Completed/g)?.length || 0 }}
      Pending tasks: {{ ($json["Status"] || "").match(/In Progress|Not Started/g)?.length || 0 }}
      ```  
    - Channel: Same Slack channel as reminders.  
    - Connect input from "Fetch Completed Tasks".

12. **Optional: Add Sticky Note node for documentation**

    - Paste full workflow setup, benefits, and usage notes as described.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| This workflow replaces paid productivity tools with a zero-cost system based on Google Sheets and Slack, enabling simple task tracking and timely reminders.                                                                             | Workflow benefits section in Sticky Note      |
| Google Sheets acts as the database where tasks are maintained and updated, while Slack serves as the communication interface for reminders and recaps.                                                                                   | Sticky Note documentation                      |
| Ensure all date/time fields use consistent formatting (`yyyy-MM-dd HH:mm`) and consider timezone settings to avoid incorrect triggers.                                                                                                  | Workflow edge case consideration               |
| Slack API credentials require proper channel permissions and token scopes to post messages successfully; watch for rate limits if many reminders are sent.                                                                              | Slack node note                                |
| The workflow can be extended with additional features such as weekly summaries, AI-generated insights, or personalized reminders per assigned user.                                                                                     | Sticky Note suggestions                         |
| For large teams, consider adding filtering or looping mechanisms to send reminders directly to assigned users instead of a general channel.                                                                                             | Use case notes                                 |
| Google Sheets operations rely on consistent "Task ID" uniqueness to update timestamps correctly and avoid duplicate reminders.                                                                                                          | Update node edge case                           |
| Workflow expressions assume the presence of all required fields; missing or empty fields may cause expression errors or incomplete messages.                                                                                            | Expression failure edge case                     |
| For further customization, consult n8n documentation on Google Sheets and Slack node capabilities: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.google-sheets/ and https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.slack/ | Official n8n documentation                      |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow built with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is lawful and public.