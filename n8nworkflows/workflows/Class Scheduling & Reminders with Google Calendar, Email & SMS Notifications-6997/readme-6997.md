Class Scheduling & Reminders with Google Calendar, Email & SMS Notifications

https://n8nworkflows.xyz/workflows/class-scheduling---reminders-with-google-calendar--email---sms-notifications-6997


# Class Scheduling & Reminders with Google Calendar, Email & SMS Notifications

### 1. Workflow Overview

This workflow automates the process of scheduling student classes and sending timely reminders via email or SMS, while synchronizing events with Google Calendar and logging reminder activities. It is designed for educational institutions or tutors managing multiple classes and students, aiming to improve communication and scheduling efficiency.

The workflow’s logic is grouped into the following blocks:

- **1.1 Daily Trigger & Schedule Loading:** Automatically triggers daily to read the class schedule from an Excel workbook.
- **1.2 Class Filtering & Calendar Sync:** Filters classes for today and tomorrow (for evening prep), conditionally syncs relevant classes to Google Calendar.
- **1.3 Student Contact Retrieval & Reminder Creation:** Loads student contact info, matches students to classes, and generates personalized reminder messages.
- **1.4 Notification Dispatch:** Splits reminders into batches, determines preferred contact method (email or SMS), and prepares messages accordingly.
- **1.5 Activity Logging:** Reads existing reminder logs, updates logs with new sent reminders, and appends the updated log back to Excel.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Daily Trigger & Schedule Loading

- **Overview:** This block initiates the workflow daily and reads the full class schedule from a Microsoft Excel workbook.

- **Nodes Involved:**  
  - Daily Schedule Check  
  - Read Class Schedule

- **Node Details:**

1. **Daily Schedule Check**  
   - *Type:* Cron Trigger  
   - *Role:* Triggers the workflow on a daily schedule (default cron settings).  
   - *Config:* Default cron parameters (runs once per day; exact time configurable in node).  
   - *Connections:* Output → Read Class Schedule  
   - *Edge Cases:* If the cron trigger fails or is misconfigured, the workflow will not start on schedule.

2. **Read Class Schedule**  
   - *Type:* Microsoft Excel  
   - *Role:* Reads the full class schedule data from a specified Excel workbook.  
   - *Config:* Uses OAuth2 credentials for Microsoft Excel access; no filters applied (reads entire dataset).  
   - *Connections:* Output → Filter Today's Classes  
   - *Edge Cases:* Authentication failure, file access issues, or empty/malformed Excel data can cause errors.

---

#### 2.2 Class Filtering & Calendar Sync

- **Overview:** Filters the classes for today and tomorrow (after 6 PM) to identify which reminders should be sent. Also syncs today's classes to Google Calendar.

- **Nodes Involved:**  
  - Filter Today's Classes  
  - Has Classes Today?  
  - Sync to Google Calendar

- **Node Details:**

1. **Filter Today's Classes**  
   - *Type:* Code (JavaScript)  
   - *Role:* Processes schedule data to select classes happening today within reminder time windows, plus tomorrow’s classes after 6 PM for prep reminders.  
   - *Config:*  
     - Uses current date/time to compare class dates and times.  
     - Checks 'Reminder Time (Hours)' field to determine reminder window.  
     - Combines today’s immediate reminders with tomorrow’s evening prep classes.  
   - *Key Expressions:* Current date/time, string parsing of class times, conditional filtering.  
   - *Connections:* Output → Has Classes Today? and Sync to Google Calendar  
   - *Edge Cases:* Incorrect date/time formats in Excel, missing fields, or empty schedules could result in no reminders or misfires.

2. **Has Classes Today?**  
   - *Type:* If  
   - *Role:* Checks if filtered classes have a non-empty “Class Name” to continue processing.  
   - *Config:* Condition: `Class Name` field is not empty.  
   - *Connections:* True → Read Student Contacts  
   - *Edge Cases:* If no classes today, workflow branch halts here; no reminders sent.

3. **Sync to Google Calendar**  
   - *Type:* Google Calendar  
   - *Role:* Creates calendar events for the filtered classes on Google Calendar with 1-hour duration.  
   - *Config:*  
     - Start and end time computed from class date/time fields.  
     - Calendar ID fixed to a specific Google account calendar.  
     - OAuth2 credentials for Google Calendar access.  
   - *Connections:* Output → Create Student Reminders  
   - *Edge Cases:* API failures, auth errors, or date/time parsing errors can prevent calendar sync.

---

#### 2.3 Student Contact Retrieval & Reminder Creation

- **Overview:** Retrieves student contact information and generates personalized reminder messages for each student-class pair.

- **Nodes Involved:**  
  - Read Student Contacts  
  - Create Student Reminders

- **Node Details:**

1. **Read Student Contacts**  
   - *Type:* Microsoft Excel  
   - *Role:* Reads student contacts and enrollment details from Excel.  
   - *Config:* Uses same Microsoft Excel OAuth2 credentials as before; no data filters.  
   - *Connections:* Output → Create Student Reminders  
   - *Edge Cases:* Same as other Excel read nodes (auth, data format, missing fields).

2. **Create Student Reminders**  
   - *Type:* Code (JavaScript)  
   - *Role:* Matches students to classes by checking “Enrolled Classes” field and generates reminder messages tailored to reminder type (‘today’ or ‘tomorrow’).  
   - *Config:*  
     - Constructs message text with class details and prep tips.  
     - Determines preferred contact method (default email) per student.  
   - *Connections:* Output → Split Into Batches  
   - *Edge Cases:* Missing student enrollment or contact info can cause empty reminders or skipped notifications.

---

#### 2.4 Notification Dispatch

- **Overview:** Batches reminders and sends them via the students’ preferred contact method (email or SMS), preparing appropriate message formats.

- **Nodes Involved:**  
  - Split Into Batches  
  - Email or SMS?  
  - Prepare Email Reminders  
  - Prepare SMS Reminders

- **Node Details:**

1. **Split Into Batches**  
   - *Type:* SplitInBatches  
   - *Role:* Splits reminders into batches of 10 for controlled processing and rate limits.  
   - *Config:* Batch size = 10  
   - *Connections:* Output → Email or SMS?  
   - *Edge Cases:* No reminders to batch results in empty output.

2. **Email or SMS?**  
   - *Type:* If  
   - *Role:* Routes reminders based on preferred contact method.  
   - *Config:* Condition: preferredContact equals 'email'  
   - *Connections:* True → Prepare Email Reminders; False → Prepare SMS Reminders  
   - *Edge Cases:* Unexpected or missing preferredContact values may cause routing errors or skipped notifications.

3. **Prepare Email Reminders**  
   - *Type:* Code (JavaScript)  
   - *Role:* Formats HTML email body and subject for reminders, including styled layout and dynamic content.  
   - *Config:*  
     - Uses reminderType to customize subject and body.  
     - Includes class details, tips, and branding styles.  
   - *Connections:* Output → Read Reminder Log  
   - *Edge Cases:* Malformed HTML or missing email addresses can cause delivery failures.

4. **Prepare SMS Reminders**  
   - *Type:* Code (JavaScript)  
   - *Role:* Creates concise SMS message strings for reminders with essential info.  
   - *Config:* Customizes message based on reminder type and class details.  
   - *Connections:* Output → Read Reminder Log  
   - *Edge Cases:* Missing phone numbers or SMS gateway issues can cause failures.

---

#### 2.5 Activity Logging

- **Overview:** Reads existing logs from Excel, appends new reminder send entries, and saves the updated log back to Excel for audit and tracking purposes.

- **Nodes Involved:**  
  - Read Reminder Log  
  - Update Reminder Log  
  - Save Reminder Log

- **Node Details:**

1. **Read Reminder Log**  
   - *Type:* Microsoft Excel  
   - *Role:* Reads existing reminder log entries from Excel.  
   - *Config:* Uses Microsoft Excel OAuth2 credentials; no filters applied.  
   - *Connections:* Output → Update Reminder Log  
   - *Edge Cases:* Missing log file, access errors.

2. **Update Reminder Log**  
   - *Type:* Code (JavaScript)  
   - *Role:* Combines existing logs with new sent reminder entries (email and SMS), generating unique log IDs and timestamps.  
   - *Config:*  
     - Extracts sent reminder data from previous nodes.  
     - Creates comprehensive log records with status ‘Sent’.  
   - *Connections:* Output → Save Reminder Log  
   - *Edge Cases:* Missing or empty sent reminder data; concurrency issues if multiple runs overlap.

3. **Save Reminder Log**  
   - *Type:* Microsoft Excel  
   - *Role:* Appends updated log entries to the Excel worksheet for persistent storage.  
   - *Config:* Uses OAuth2 credentials, appends rows to specified workbook and worksheet by ID.  
   - *Connections:* End of workflow  
   - *Edge Cases:* Write permission issues, file locks.

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                     | Input Node(s)              | Output Node(s)                    | Sticky Note                                                                                                                                                         |
|-------------------------|------------------------|-----------------------------------|---------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Daily Schedule Check     | Cron Trigger           | Daily workflow trigger            | -                         | Read Class Schedule              | **Workflow Process:** 1. Daily Check → Triggers at scheduled time                                                                                                 |
| Read Class Schedule      | Microsoft Excel        | Load class schedule data           | Daily Schedule Check       | Filter Today's Classes           | 2. Read Schedule → Gets today's class schedule                                                                                                                    |
| Filter Today's Classes   | Code                   | Filter classes for reminders       | Read Class Schedule        | Has Classes Today?, Sync to Google Calendar | 3. Filter Classes → Identifies today's classes                                                                                                                    |
| Has Classes Today?       | If                     | Check for classes presence         | Filter Today's Classes     | Read Student Contacts            | 4. Check Students → Gets enrolled student contacts                                                                                                                |
| Read Student Contacts    | Microsoft Excel        | Load student contact info          | Has Classes Today?         | Create Student Reminders         |                                                                                                                                                                   |
| Create Student Reminders | Code                   | Generate personalized reminders    | Read Student Contacts, Sync to Google Calendar | Split Into Batches              | 6. Create Reminders → Generates personalized messages                                                                                                            |
| Split Into Batches       | SplitInBatches         | Batch reminders for processing     | Create Student Reminders   | Email or SMS?                   | 7. Send Notifications → Delivers via email/SMS                                                                                                                    |
| Email or SMS?            | If                     | Route reminders by contact method  | Split Into Batches         | Prepare Email Reminders, Prepare SMS Reminders |                                                                                                                                                                   |
| Prepare Email Reminders  | Code                   | Format and prepare email messages  | Email or SMS?              | Read Reminder Log                |                                                                                                                                                                   |
| Prepare SMS Reminders    | Code                   | Format and prepare SMS messages    | Email or SMS?              | Read Reminder Log                |                                                                                                                                                                   |
| Sync to Google Calendar  | Google Calendar        | Sync class events to calendar      | Filter Today's Classes     | Create Student Reminders         | 5. Sync Calendar → Updates Google Calendar                                                                                                                        |
| Read Reminder Log        | Microsoft Excel        | Load log of past reminders         | Prepare Email Reminders, Prepare SMS Reminders | Update Reminder Log          | 8. Log Activity → Records reminder status                                                                                                                        |
| Update Reminder Log      | Code                   | Append new reminder logs            | Read Reminder Log          | Save Reminder Log                |                                                                                                                                                                   |
| Save Reminder Log        | Microsoft Excel        | Save updated reminder logs          | Update Reminder Log        | -                               |                                                                                                                                                                   |
| Sticky Note             | Sticky Note             | Workflow process summary            | -                         | -                               | See "Workflow Process" steps above                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node: "Daily Schedule Check"**  
   - Type: Cron Trigger  
   - Configure to run once daily at your preferred time (e.g., 7:00 AM).  
   - No additional parameters needed.

2. **Create Microsoft Excel Node: "Read Class Schedule"**  
   - Operation: Read rows (default).  
   - Credentials: Configure Microsoft Excel OAuth2 with appropriate permissions.  
   - Select the workbook and worksheet containing the class schedule.  
   - Connect output from "Daily Schedule Check".

3. **Create Code Node: "Filter Today's Classes"**  
   - Paste JavaScript code that:  
     - Gets current date/time, extracts today and tomorrow dates.  
     - Filters classes for today within the reminder window based on 'Reminder Time (Hours)'.  
     - Also selects tomorrow's classes after 6 PM for prep reminders.  
     - Adds fields 'reminderType' ('today' or 'tomorrow') and 'currentTime'.  
   - Input: output from "Read Class Schedule".  
   - Output: connect to "Has Classes Today?" and "Sync to Google Calendar".

4. **Create If Node: "Has Classes Today?"**  
   - Condition: Check if `Class Name` is not empty.  
   - True branch connects to "Read Student Contacts".  
   - False branch: no further action.

5. **Create Microsoft Excel Node: "Read Student Contacts"**  
   - Operation: Read rows.  
   - Credentials: Use the same Excel OAuth2 credentials.  
   - Select workbook/worksheet with student contact info and enrollment details.  
   - Connect input from "Has Classes Today?" (true branch).

6. **Create Google Calendar Node: "Sync to Google Calendar"**  
   - Operation: Create event.  
   - Credentials: Configure Google Calendar OAuth2 with access to target calendar.  
   - Set Start: Class Date + Class Time (HH:mm:ss).  
   - Set End: Class Date + (Class Time + 1 hour).  
   - Calendar ID: Use your calendar’s email or ID.  
   - Connect input from "Filter Today's Classes".

7. **Create Code Node: "Create Student Reminders"**  
   - Paste JavaScript code that:  
     - Matches students to classes by checking 'Enrolled Classes'.  
     - Generates messages based on reminder type with class details.  
     - Assigns preferred contact method (email default).  
   - Inputs: outputs from "Read Student Contacts" and "Sync to Google Calendar".  
   - Output connects to "Split Into Batches".

8. **Create SplitInBatches Node: "Split Into Batches"**  
   - Batch Size: 10  
   - Input: from "Create Student Reminders".  
   - Output connects to "Email or SMS?".

9. **Create If Node: "Email or SMS?"**  
   - Condition: `preferredContact` equals 'email'.  
   - True branch connects to "Prepare Email Reminders".  
   - False branch connects to "Prepare SMS Reminders".

10. **Create Code Node: "Prepare Email Reminders"**  
    - Paste JavaScript code that:  
      - Formats HTML email with class info, greeting, and reminders.  
      - Sets email subject based on reminder type.  
      - Includes inline CSS styles for email display.  
    - Input: from "Email or SMS?" (true branch).  
    - Output connects to "Read Reminder Log".

11. **Create Code Node: "Prepare SMS Reminders"**  
    - Paste JavaScript code that:  
      - Creates SMS message string with class details and tip.  
    - Input: from "Email or SMS?" (false branch).  
    - Output connects to "Read Reminder Log".

12. **Create Microsoft Excel Node: "Read Reminder Log"**  
    - Operation: Read rows.  
    - Credentials: Same Microsoft Excel OAuth2 credentials.  
    - Select workbook and worksheet containing the reminder logs.  
    - Inputs: from both "Prepare Email Reminders" and "Prepare SMS Reminders".  
    - Output connects to "Update Reminder Log".

13. **Create Code Node: "Update Reminder Log"**  
    - Paste JavaScript code that:  
      - Reads existing logs.  
      - Adds new log entries for sent emails and SMS with timestamps and unique IDs.  
    - Input: from "Read Reminder Log".  
    - Output connects to "Save Reminder Log".

14. **Create Microsoft Excel Node: "Save Reminder Log"**  
    - Operation: Append rows.  
    - Credentials: Microsoft Excel OAuth2.  
    - Select workbook and worksheet for appending logs.  
    - Input: from "Update Reminder Log".  
    - This is the workflow end node.

15. **Create Sticky Note Node: "Sticky Note"** (optional)  
    - Content: Summary of workflow steps as per section 1 overview.  
    - Position it visibly for documentation purposes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                        | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow includes OAuth2 credential usage for Microsoft Excel and Google Calendar; ensure proper configuration and permissions before deployment.              | Credential setup required for Microsoft Excel and Google Calendar nodes.                        |
| Reminder messages are customizable in the code nodes; adjust text and formatting to fit branding and communication style.                                         | Code nodes "Create Student Reminders", "Prepare Email Reminders", and "Prepare SMS Reminders".  |
| Batch processing limits simultaneous sends to avoid API rate limiting or spam filtering issues. Adjust batch size as needed.                                     | "Split Into Batches" node configuration.                                                       |
| The workflow logs all sent reminders with unique IDs and timestamps for auditability; Excel file permissions must allow append operations.                      | Logging nodes: "Read Reminder Log", "Update Reminder Log", "Save Reminder Log".                 |
| For scalability, consider adding error handling nodes or retry logic to manage transient API failures or data inconsistencies.                                   | Not implemented natively in this workflow; recommended for production use.                      |
| This workflow can be adapted to other calendar systems or messaging services by replacing the Google Calendar and SMS preparation nodes accordingly.               | Node replacement possible for SMS gateway or calendar API.                                     |
| For assistance with Microsoft Excel OAuth2 permissions, see https://docs.microsoft.com/en-us/graph/auth-v2-user | Microsoft Excel OAuth2 API documentation.                                     |
| For Google Calendar API usage, refer to https://developers.google.com/calendar/api/guides/overview                                                             | Google Calendar API official guides.                                                           |

---

This documentation provides a complete and detailed reference for the "Automated Student Class Scheduling & Calendar Reminder" workflow, enabling replication, modification, and troubleshooting.