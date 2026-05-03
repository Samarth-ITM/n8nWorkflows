Send Preventive Maintenance Schedules with Google Sheets and Gmail ICS Invites

https://n8nworkflows.xyz/workflows/send-preventive-maintenance-schedules-with-google-sheets-and-gmail-ics-invites-7700


# Send Preventive Maintenance Schedules with Google Sheets and Gmail ICS Invites

### 1. Workflow Overview

This workflow automates the distribution of preventive maintenance schedules by extracting tasks from a Google Sheet and sending personalized calendar invites (.ics files) via Gmail. It is designed for organizations that want to ensure their maintenance teams receive timely, clear, and actionable reminders for scheduled tasks.

The workflow is logically divided into the following blocks:

- **1.1 Daily Trigger:** Initiates the workflow once daily at a specified time.
- **1.2 Read Maintenance Tasks:** Reads maintenance task data scheduled for the current day from a Google Sheet.
- **1.3 Generate ICS Data:** Transforms raw maintenance task data into iCalendar (ICS) compatible date-time format and event details.
- **1.4 Create ICS File:** Converts the generated event data into a standardized .ics calendar file.
- **1.5 Send Calendar Invite Email:** Sends the personalized calendar invite as an email attachment to the designated assignee.

---

### 2. Block-by-Block Analysis

#### 1.1 Daily Trigger

- **Overview:**  
  Kicks off the workflow daily at 7:00 AM to ensure maintenance tasks for the current day are processed and notified on schedule.

- **Nodes Involved:**  
  - Daily Trigger

- **Node Details:**  
  - **Type:** Schedule Trigger  
  - **Role:** Time-based entry point to start the workflow automatically every day.  
  - **Configuration:**  
    - Triggers at hour 7 (7:00 AM) daily.  
  - **Key Expressions:** None.  
  - **Input/Output:**  
    - No input (trigger node).  
    - Output connected to "Read Maintenance Tasks".  
  - **Edge Cases / Failures:**  
    - Time zone mismatches could cause trigger timing issues.  
    - Node downtime or n8n instance downtime may miss trigger.  
  - **Version:** 1.2

#### 1.2 Read Maintenance Tasks

- **Overview:**  
  Fetches maintenance tasks from a specific Google Sheet, filtering tasks scheduled for the current date.

- **Nodes Involved:**  
  - Read Maintenance Tasks

- **Node Details:**  
  - **Type:** Google Sheets  
  - **Role:** Data source node that reads rows from the configured spreadsheet.  
  - **Configuration:**  
    - Document ID: `1bYe5FlYuNNcajCWA5WW8f1wfNbVEMwIP9nyAcZtZeCc` (Google Sheet named "Preventive_Maintenance_Scheduler_final").  
    - Sheet Name: `Sheet1` (gid=0).  
    - Filter: Matches rows where the "date" column equals the current date formatted as `Day/Month/Year`. Expression: `={{ $json["Day of month"] }}/{{ $json.Month }}/{{ $json.Year }}`.  
  - **Credentials:** Google Sheets OAuth2 authentication configured.  
  - **Key Expressions:** The date filter expression dynamically selects today's tasks.  
  - **Input/Output:**  
    - Input from "Daily Trigger".  
    - Output to "Generate ICS Data".  
  - **Edge Cases / Failures:**  
    - Authentication token expiration or revocation.  
    - Sheet or document ID changes leading to data retrieval failure.  
    - Date format mismatch in the sheet causing filter to fail.  
  - **Version:** 4.6

#### 1.3 Generate ICS Data

- **Overview:**  
  Converts the Google Sheets date string to the iCalendar datetime format and prepares start/end times for the event.

- **Nodes Involved:**  
  - Generate ICS Data

- **Node Details:**  
  - **Type:** Code (JavaScript)  
  - **Role:** Data transformation node that formats dates and adds event start/end timestamps.  
  - **Configuration:**  
    - Parses date strings like `"30/July/2025"`, mapping month names to numeric values.  
    - Formats start time as 09:00 UTC and end time as 10:00 UTC for all events.  
    - Adds `dtStart` and `dtEnd` fields to each item with values like `20250730T090000Z`.  
  - **Key Expressions:** Custom JavaScript code iterating over all input items.  
  - **Input/Output:**  
    - Input from "Read Maintenance Tasks".  
    - Output to "Create ICS File".  
  - **Edge Cases / Failures:**  
    - Unexpected or malformed date strings causing parsing errors.  
    - Time zone assumptions hardcoded as UTC may not fit all use cases.  
    - Missing or null date fields may cause runtime exceptions.  
  - **Version:** 2

#### 1.4 Create ICS File

- **Overview:**  
  Converts the event data (including start/end times, location, description) into a proper .ics calendar file with MIME type `text/calendar`.

- **Nodes Involved:**  
  - Create ICS File

- **Node Details:**  
  - **Type:** Convert To File  
  - **Role:** Generates calendar invitation files in the iCal format.  
  - **Configuration:**  
    - Operation: iCal.  
    - Start: uses `{{$json.dtStart}}`.  
    - End: uses `{{$json.dtEnd}}`.  
    - Title: Static "Maintenance".  
    - Additional fields:  
      - Location: from the `location` property of the item.  
      - Description: composed from asset and task details, formatted as:  
        ```
        Asset - {{ $json.asset }}
        Task - {{ $json.task }}
        ```  
  - **Input/Output:**  
    - Input from "Generate ICS Data".  
    - Output to "Send Calendar Invite Email".  
  - **Edge Cases / Failures:**  
    - Missing or malformed date/time values prevent file creation.  
    - Missing location, asset, or task fields could produce incomplete invites.  
  - **Version:** 1.1

#### 1.5 Send Calendar Invite Email

- **Overview:**  
  Sends an email with the generated .ics file attached to the assigned maintenance person.

- **Nodes Involved:**  
  - Send Calendar Invite Email

- **Node Details:**  
  - **Type:** Gmail  
  - **Role:** Sends personalized emails with calendar invites.  
  - **Configuration:**  
    - Recipient: email address from `"Read Maintenance Tasks"` node's `email` field.  
    - Subject: includes task, asset, and date details.  
    - Message body: personalized text including asset, task, location, and date details.  
    - Attachment: the generated .ics file attached via binary data property `data`.  
    - Options: Attribution disabled.  
  - **Credentials:** Gmail OAuth2 account connected and authorized.  
  - **Key Expressions:** Uses expressions to pull data from the "Read Maintenance Tasks" node for email content.  
  - **Input/Output:**  
    - Input from "Create ICS File".  
    - No further outputs (end node).  
  - **Edge Cases / Failures:**  
    - Authentication failure or token expiration.  
    - Invalid or missing recipient email addresses.  
    - Attachment binary data missing or corrupted.  
    - Gmail API rate limits or send restrictions.  
  - **Version:** 2.1  
  - **Webhook ID:** Present but not relevant for this use case.

---

### 3. Summary Table

| Node Name                  | Node Type                | Functional Role                      | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                   |
|----------------------------|--------------------------|------------------------------------|------------------------|----------------------------|----------------------------------------------------------------------------------------------|
| Sticky Note                | Sticky Note              | Workflow title display              | —                      | —                          | ## Preventive Maintenance Scheduler via ICS Email (n8n | Google Sheets | SMTP)               |
| Sticky Note1               | Sticky Note              | Workflow overview and block summary| —                      | —                          | **Preventive Maintenance Scheduler (Google Sheets → ICS → Email)** ... detailed steps      |
| Daily Trigger              | Schedule Trigger         | Starts workflow daily at 7 AM       | —                      | Read Maintenance Tasks     |                                                                                              |
| Read Maintenance Tasks     | Google Sheets            | Reads today's maintenance tasks     | Daily Trigger          | Generate ICS Data           |                                                                                              |
| Generate ICS Data          | Code                     | Formats dates & prepares ICS fields | Read Maintenance Tasks | Create ICS File             |                                                                                              |
| Create ICS File            | Convert To File          | Creates .ics calendar file           | Generate ICS Data      | Send Calendar Invite Email |                                                                                              |
| Send Calendar Invite Email | Gmail                    | Sends email with ICS file attached  | Create ICS File        | —                          |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n with the name: `Maintenance-Scheduler-Via-ICS-Email`.

2. **Add a Schedule Trigger node:**
   - Name: `Daily Trigger`
   - Type: Schedule Trigger
   - Set to trigger daily at 7:00 AM (Hour = 7).
   - Connect output to the next node.

3. **Add a Google Sheets node:**
   - Name: `Read Maintenance Tasks`
   - Type: Google Sheets
   - Credentials: Connect your Google Sheets OAuth2 credentials.
   - Set Document ID to: `1bYe5FlYuNNcajCWA5WW8f1wfNbVEMwIP9nyAcZtZeCc`.
   - Set Sheet Name to: `Sheet1` (gid=0).
   - Add a filter on the `date` column with this expression to match today's date:
     ```
     ={{ $json["Day of month"] }}/{{ $json.Month }}/{{ $json.Year }}
     ```
   - Connect input from `Daily Trigger`.
   - Connect output to the next node.

4. **Add a Code node:**
   - Name: `Generate ICS Data`
   - Type: Code (JavaScript)
   - Paste the following code:
     ```javascript
     function pad(n) { return n < 10 ? '0' + n : n; }

     const result = [];
     for (const item of $input.all()) {
       let rawDate = item.json.date;
       const monthNames = {
         "January": "01", "February": "02", "March": "03", "April": "04",
         "May": "05", "June": "06", "July": "07", "August": "08",
         "September": "09", "October": "10", "November": "11", "December": "12"
       };
       let [day, month, year] = rawDate.split("/");
       let eventDate = `${year}${monthNames[month]}${pad(Number(day))}`;
       let dtStart = `${eventDate}T090000Z`;
       let dtEnd   = `${eventDate}T100000Z`;
       result.push({
         json: {
           ...item.json,
           dtStart: dtStart,
           dtEnd: dtEnd
         }
       });
     }
     return result;
     ```
   - Connect input from `Read Maintenance Tasks`.
   - Connect output to the next node.

5. **Add a Convert To File node:**
   - Name: `Create ICS File`
   - Type: Convert To File
   - Operation: iCal
   - Set Start to: `={{ $json.dtStart }}`
   - Set End to: `={{ $json.dtEnd }}`
   - Title: `Maintenance`
   - Additional Fields:
     - Location: `={{ $json.location }}`
     - Description:
       ```
       Asset - {{ $json.asset }}
       Task - {{ $json.task }}
       ```
   - Connect input from `Generate ICS Data`.
   - Connect output to the next node.

6. **Add a Gmail node:**
   - Name: `Send Calendar Invite Email`
   - Type: Gmail
   - Credentials: Connect Gmail OAuth2 credentials.
   - Send To: `={{ $('Read Maintenance Tasks').item.json.email }}`
   - Subject:
     ```
     Preventive Maintenance Task: {{ $('Read Maintenance Tasks').item.json.task }} for {{ $('Read Maintenance Tasks').item.json.asset }} on {{ $('Read Maintenance Tasks').item.json.date }}
     ```
   - Message:
     ```
     Hello,

     You have a scheduled preventive maintenance task today.

     Details:
     - Asset: {{ $('Read Maintenance Tasks').item.json.asset }}
     - Task: {{ $('Read Maintenance Tasks').item.json.task }}
     - Location: {{ $('Read Maintenance Tasks').item.json.location }}
     - Date: {{ $('Read Maintenance Tasks').item.json.date }}

     Please find the calendar invite attached. Add it to your calendar so you don’t miss the task.

     Thank you,
     Maintenance Team
     ```
   - Options:
     - Attachments: Enable attachment of binary property `data` (output of `Create ICS File` node).
     - Disable attribution.
   - Connect input from `Create ICS File`.

7. **Optional: Add Sticky Notes for documentation:**
   - Add notes summarizing the workflow purpose and step descriptions for clarity.

8. **Activate the workflow** to run daily and send calendar invites automatically.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                                   |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow is designed to operate on date strings formatted as "DD/Month/YYYY" e.g. "30/July/2025". Adjust code if your date format differs. | Date format dependency in the "Generate ICS Data" node.                                                          |
| Gmail OAuth2 credentials must have permissions to send emails on behalf of the configured account.  | Gmail node authentication.                                                                                        |
| Google Sheets API credentials must have read access to the specified spreadsheet document.         | Google Sheets node authentication.                                                                               |
| This workflow sends calendar invites in UTC timezone fixed between 9:00-10:00. Adjust times in code if needed. | Time zone and scheduling assumptions in the "Generate ICS Data" node.                                            |
| For troubleshooting Gmail API limits, review Google's sending limits and consider batching or delays.| Gmail rate limits considerations.                                                                                 |

---

This documentation fully describes the "Send Preventive Maintenance Schedules with Google Sheets and Gmail ICS Invites" workflow, enabling users and automation agents to understand, reproduce, and maintain the process confidently.