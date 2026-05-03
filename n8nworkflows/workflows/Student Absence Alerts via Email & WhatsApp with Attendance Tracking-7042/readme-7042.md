Student Absence Alerts via Email & WhatsApp with Attendance Tracking

https://n8nworkflows.xyz/workflows/student-absence-alerts-via-email---whatsapp-with-attendance-tracking-7042


# Student Absence Alerts via Email & WhatsApp with Attendance Tracking

### 1. Workflow Overview

This workflow automates the daily monitoring and alerting of student absences by integrating attendance data with student contact records. It targets educational institutions aiming to proactively communicate unexcused absences to parents via email and WhatsApp, while tracking attendance trends and generating daily reports.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow every day at 10:30 AM.
- **1.2 Data Retrieval:** Reads attendance records for the current day and loads student contact information.
- **1.3 Absence Processing:** Identifies unexcused absent students, computes absence patterns over the past 30 days, and categorizes alert levels.
- **1.4 Notification Preparation:** Constructs personalized email and WhatsApp messages based on alert severity.
- **1.5 Notification Sending:** Sends absence alerts through email and WhatsApp channels.
- **1.6 Reporting:** Generates a summary attendance report and appends it to a report workbook.
- **1.7 Metadata & Documentation:** Contains workflow description and usage notes as a sticky note.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:** Starts the workflow automatically at a specific time each day to ensure timely attendance processing.
- **Nodes Involved:**  
  - *Daily Attendance Check - 10:30 AM*

- **Node Details:**  
  - **Name:** Daily Attendance Check - 10:30 AM  
  - **Type:** Schedule Trigger  
  - **Configuration:** Triggers daily at 10:30 AM (hour 10, minute 30).  
  - **Inputs:** None (trigger node).  
  - **Outputs:** Triggers downstream nodes to read attendance and student contact data.  
  - **Potential Failures:** None typical; depends on n8n scheduler service uptime.  
  - **Version:** 1.2

---

#### 2.2 Data Retrieval

- **Overview:** Fetches the attendance data for the current day and retrieves all student contact details from separate Excel workbooks for further processing.
- **Nodes Involved:**  
  - *Read Today's Attendance*  
  - *Read Student Contacts*

- **Node Details:**

  - **Read Today's Attendance**  
    - Type: Microsoft Excel node (v2)  
    - Role: Reads attendance worksheet filtered to only include records matching today's date (based on ISO date string).  
    - Configuration: Workbook specified by ID (parameterized as `attendance-workbook`), filter applied on "Date" column with today's date dynamically calculated.  
    - Inputs: Trigger from schedule node.  
    - Outputs: Attendance records for today.  
    - Credentials: Microsoft Excel OAuth2.  
    - Edge Cases: Empty attendance records if no data for today, Excel API authentication or network errors.

  - **Read Student Contacts**  
    - Type: Microsoft Excel node (v2)  
    - Role: Reads entire student contact list (no filters).  
    - Configuration: Workbook specified by ID (`student-records-workbook`).  
    - Inputs: Trigger from schedule node.  
    - Outputs: Student contact details for lookup.  
    - Credentials: Microsoft Excel OAuth2.  
    - Edge Cases: Empty contact list, auth errors, data format inconsistencies.

---

#### 2.3 Absence Processing

- **Overview:** Cross-references attendance and student data to identify unexcused absences, calculates absence frequency over the last 30 days, estimates attendance rates, and assigns alert levels.
- **Nodes Involved:**  
  - *Process Absent Students*

- **Node Details:**  
  - Type: Code node (JavaScript)  
  - Role: Performs custom logic to filter and enrich absent student data.  
  - Key Logic:  
    - Iterates today's attendance records, selects those with status 'Absent' and no excused flag.  
    - Finds corresponding student contact info.  
    - Counts absences for the same student over the last 30 days.  
    - Calculates attendance rate as percentage.  
    - Assigns alert level based on absence thresholds:  
      - High (🚨): 5+ absences  
      - Medium (⚠️): 3-4 absences  
      - Low (ℹ️): 1-2 absences  
  - Inputs: Attendance and student contacts data from previous nodes.  
  - Outputs: List of absent students with enriched data for messaging and reporting.  
  - Edge Cases: Missing student contact, malformed dates, empty inputs, code execution failures.  
  - Version: 2

---

#### 2.4 Notification Preparation

- **Overview:** Generates customized email and WhatsApp messages tailored to each absent student's alert level and contact details.
- **Nodes Involved:**  
  - *Prepare Absence Email*  
  - *Prepare Absence SMS*

- **Node Details:**

  - **Prepare Absence Email**  
    - Type: Code node (JavaScript)  
    - Role: Creates an email subject and body incorporating alert icons, student and parent names, absence details, and attendance summary with appropriate alert messages.  
    - Uses conditional logic to vary message content by alert level.  
    - Outputs email metadata: recipient address, subject, body text.  
    - Inputs: Single absent student record from Process Absent Students.  
    - Edge Cases: Missing email address, empty data fields, string interpolation errors.  
    - Version: 2

  - **Prepare Absence SMS**  
    - Type: Code node (JavaScript)  
    - Role: Constructs WhatsApp message text with alert emoji, student and parent info, attendance summary, and alert-specific instructions.  
    - Formatted with markdown style for WhatsApp display.  
    - Outputs phone number and message text.  
    - Inputs: Single absent student record.  
    - Edge Cases: Missing phone number, message length constraints, invalid characters for WhatsApp.  
    - Version: 2

---

#### 2.5 Notification Sending

- **Overview:** Sends the prepared absence alerts via email and WhatsApp to parents.
- **Nodes Involved:**  
  - *Send Absence Email*  
  - *Send Absence WhatsApp*

- **Node Details:**

  - **Send Absence Email**  
    - Type: Email Send node  
    - Role: Sends plaintext email to parent email address with subject and body from previous node.  
    - Configured to send from "attendance@school.edu".  
    - Credentials: SMTP credentials (named "SMTP -test").  
    - Inputs: Email data from Prepare Absence Email node.  
    - Edge Cases: SMTP authentication failure, invalid email addresses, email delivery issues.  
    - Version: 2.1

  - **Send Absence WhatsApp**  
    - Type: HTTP Request node  
    - Role: Sends WhatsApp message via Facebook Graph API endpoint for WhatsApp Business.  
    - Configured with POST request, JSON body containing recipient phone and message text.  
    - Headers include Authorization Bearer token (placeholder: "YOUR_ACCESS_TOKEN") and Content-Type application/json.  
    - Inputs: Phone and message from Prepare Absence SMS node.  
    - Edge Cases: API authentication failure, invalid phone number format, rate limits, network errors.  
    - Version: 4.2

---

#### 2.6 Reporting

- **Overview:** Aggregates daily absence data to produce a summary report, including counts by alert level, average attendance rates, and list of students at risk. Saves the report to a dedicated Excel workbook.
- **Nodes Involved:**  
  - *Generate Attendance Report*  
  - *Save Attendance Report*

- **Node Details:**

  - **Generate Attendance Report**  
    - Type: Code node (JavaScript)  
    - Role: Processes all absent student records to compute:  
      - Total absences  
      - Counts of high, medium, and low alerts  
      - Average attendance rate  
      - List of students with 5+ recent absences (at risk)  
    - Outputs a single JSON report object.  
    - Inputs: All absent student data from Process Absent Students (via Prepare nodes).  
    - Edge Cases: Division by zero if no absences, empty input arrays, malformed data.  
    - Version: 2

  - **Save Attendance Report**  
    - Type: Microsoft Excel node (v2)  
    - Role: Appends the generated report as a new row in a specified worksheet in the "attendance-reports-workbook".  
    - Inputs: Report JSON from Generate Attendance Report.  
    - Credentials: Microsoft Excel OAuth2.  
    - Edge Cases: Excel API errors, permission issues, worksheet ID mismatch.  
    - Version: 2

---

#### 2.7 Metadata & Documentation

- **Overview:** Provides an embedded sticky note explaining workflow purpose, schedule, alert levels, and process steps for user reference.
- **Nodes Involved:**  
  - *Workflow Info*

- **Node Details:**  
  - Type: Sticky Note  
  - Content: Multi-line markdown text summarizing workflow functionality, schedule (daily 10:30 AM), alert definitions, and tracking details.  
  - Inputs/Outputs: None (informational only).  
  - Version: 1

---

### 3. Summary Table

| Node Name                   | Node Type                | Functional Role                      | Input Node(s)                      | Output Node(s)                              | Sticky Note                                                                |
|-----------------------------|--------------------------|------------------------------------|----------------------------------|---------------------------------------------|---------------------------------------------------------------------------|
| Daily Attendance Check - 10:30 AM | Schedule Trigger          | Initiates workflow daily at 10:30 AM | None                             | Read Today's Attendance, Read Student Contacts |                                                                           |
| Read Today's Attendance      | Microsoft Excel           | Reads today's attendance records   | Daily Attendance Check - 10:30 AM | Process Absent Students                      |                                                                           |
| Read Student Contacts        | Microsoft Excel           | Reads all student contact details  | Daily Attendance Check - 10:30 AM | Process Absent Students                      |                                                                           |
| Process Absent Students      | Code                     | Identifies absent students and computes patterns | Read Today's Attendance, Read Student Contacts | Prepare Absence Email, Prepare Absence SMS, Generate Attendance Report |                                                                           |
| Prepare Absence Email        | Code                     | Builds personalized absence email  | Process Absent Students           | Send Absence Email                          |                                                                           |
| Prepare Absence SMS          | Code                     | Builds personalized WhatsApp message | Process Absent Students           | Send Absence WhatsApp                        |                                                                           |
| Send Absence Email           | Email Send               | Sends absence alert emails          | Prepare Absence Email             | None                                        |                                                                           |
| Send Absence WhatsApp        | HTTP Request             | Sends absence alert WhatsApp messages | Prepare Absence SMS               | None                                        |                                                                           |
| Generate Attendance Report   | Code                     | Creates daily attendance summary report | Process Absent Students           | Save Attendance Report                       |                                                                           |
| Save Attendance Report       | Microsoft Excel           | Appends attendance report to workbook | Generate Attendance Report        | None                                        |                                                                           |
| Workflow Info               | Sticky Note              | Provides workflow description and metadata | None                             | None                                        | **Absence Tracking Workflow** Scheduled 10:30 AM; Alert Levels: High 🚨, Medium ⚠️, Low 📋; Steps: attendance check, absence ID, messaging, reporting |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Name: "Daily Attendance Check - 10:30 AM"  
   - Set trigger to daily at 10:30 AM (hour 10, minute 30).

2. **Create Microsoft Excel Node to Read Today's Attendance**  
   - Type: Microsoft Excel (v2)  
   - Name: "Read Today's Attendance"  
   - Resource: Worksheet  
   - Workbook: Select or enter the attendance workbook ID (parameterized as `attendance-workbook`).  
   - Set filter: Date column equals dynamic expression `={{new Date().toISOString().split('T')[0]}}` to get today's date.  
   - Credentials: Assign Microsoft Excel OAuth2 credentials.  
   - Connect trigger node output to this node.

3. **Create Microsoft Excel Node to Read Student Contacts**  
   - Type: Microsoft Excel (v2)  
   - Name: "Read Student Contacts"  
   - Resource: Worksheet  
   - Workbook: Select or enter the student records workbook ID (parameterized as `student-records-workbook`).  
   - No filters applied (fetch all contacts).  
   - Credentials: Assign Microsoft Excel OAuth2 credentials.  
   - Connect trigger node output to this node.

4. **Create Code Node to Process Absent Students**  
   - Type: Code (JavaScript)  
   - Name: "Process Absent Students"  
   - Input: Connect outputs of both "Read Today's Attendance" and "Read Student Contacts".  
   - Paste the custom JS code to:  
     - Filter unexcused absences for today.  
     - Match student contacts.  
     - Calculate recent absence counts (last 30 days).  
     - Compute attendance rate and alert level.  
   - Output: Array of enriched absent student objects.

5. **Create Code Node to Prepare Absence Email**  
   - Type: Code (JavaScript)  
   - Name: "Prepare Absence Email"  
   - Input: Connect from "Process Absent Students".  
   - Use JS code to build email subject, body text including alert icons and personalized messages based on alert level.  
   - Output: Email metadata with to address, subject, and body.

6. **Create Code Node to Prepare Absence SMS/WhatsApp Message**  
   - Type: Code (JavaScript)  
   - Name: "Prepare Absence SMS"  
   - Input: Connect from "Process Absent Students".  
   - Use JS code to compose WhatsApp message with markdown formatting and alert-specific notes.  
   - Output: Phone number and message text.

7. **Create Email Send Node to Send Emails**  
   - Type: Email Send  
   - Name: "Send Absence Email"  
   - Connect input from "Prepare Absence Email".  
   - Configure:  
     - From Email: attendance@school.edu  
     - To Email: `={{$json.to}}`  
     - Subject: `={{$json.subject}}`  
     - Text: `={{$json.body}}`  
   - Credentials: Set SMTP credentials.  
   - Set email format to plain text.

8. **Create HTTP Request Node to Send WhatsApp Messages**  
   - Type: HTTP Request  
   - Name: "Send Absence WhatsApp"  
   - Connect input from "Prepare Absence SMS".  
   - Configure:  
     - Method: POST  
     - URL: `https://graph.facebook.com/v17.0/FROM_PHONE_NUMBER_ID/messages` (replace `FROM_PHONE_NUMBER_ID` with your WhatsApp Business phone number ID).  
     - Headers:  
       - Authorization: Bearer YOUR_ACCESS_TOKEN (replace with valid token)  
       - Content-Type: application/json  
     - Body (JSON):  
       ```json
       {
         "messaging_product": "whatsapp",
         "to": "{{ $json.phone }}",
         "type": "text",
         "text": {
           "body": "{{ $json.message }}"
         }
       }
       ```  
     - Send Body and Headers enabled.

9. **Create Code Node to Generate Attendance Report**  
   - Type: Code (JavaScript)  
   - Name: "Generate Attendance Report"  
   - Input: Connect from "Process Absent Students".  
   - JS code to:  
     - Aggregate absent student counts by alert level.  
     - Compute average attendance rate.  
     - List students at risk (5+ absences).  
     - Output a single JSON report object.

10. **Create Microsoft Excel Node to Save Attendance Report**  
    - Type: Microsoft Excel (v2)  
    - Name: "Save Attendance Report"  
    - Input: Connect from "Generate Attendance Report".  
    - Resource: Worksheet  
    - Workbook: Set to attendance reports workbook ID (parameterized as `attendance-reports-workbook`).  
    - Worksheet: Specify worksheet ID for report storage.  
    - Operation: Append data.  
    - Credentials: Microsoft Excel OAuth2.

11. **Create Sticky Note Node for Workflow Info**  
    - Type: Sticky Note  
    - Name: "Workflow Info"  
    - Content:  
      ```
      ### **Absence Tracking Workflow**

      **🕙 Schedule:** Daily at 10:30 AM

      **📊 Process:**
      1. Check today's attendance records
      2. Identify absent students (unexcused)
      3. Calculate 30-day absence patterns
      4. Send alerts via Email & WhatsApp
      5. Generate daily attendance reports

      **🚦 Alert Levels:**
      - **High (🚨):** 5+ absences in 30 days
      - **Medium (⚠️):** 3-4 absences in 30 days  
      - **Low (📋):** 1-2 absences in 30 days

      **📈 Tracking:**
      - Attendance rates calculation
      - Pattern analysis
      - Risk identification
      - Historical reporting
      ```
    - Position it clearly for user reference; no connections required.

12. **Connect Nodes Appropriately:**  
    - Schedule Trigger → Read Today's Attendance  
    - Schedule Trigger → Read Student Contacts  
    - Read Today's Attendance → Process Absent Students  
    - Read Student Contacts → Process Absent Students  
    - Process Absent Students → Prepare Absence Email  
    - Process Absent Students → Prepare Absence SMS  
    - Process Absent Students → Generate Attendance Report  
    - Prepare Absence Email → Send Absence Email  
    - Prepare Absence SMS → Send Absence WhatsApp  
    - Generate Attendance Report → Save Attendance Report

13. **Credential Setup:**  
    - Microsoft Excel OAuth2 credentials for all Excel nodes.  
    - SMTP credentials for sending emails.  
    - Facebook Graph API access token and WhatsApp Business phone number ID for WhatsApp messaging.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                          |
|----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| Workflow runs daily at 10:30 AM to ensure attendance monitoring is timely without manual intervention.                           | Scheduling detail                                        |
| Alert levels are defined to prioritize parental outreach: High (5+ absences), Medium (3-4), Low (1-2) for graduated urgency.      | Alert level strategy                                    |
| WhatsApp messaging requires a valid Facebook Graph API token and WhatsApp Business phone number ID configured in HTTP Request.   | WhatsApp Business API docs: https://developers.facebook.com/docs/whatsapp/api |
| Email sending requires SMTP credentials configured in n8n for reliable delivery.                                                 | SMTP setup considerations                               |
| Attendance data and student contacts must be maintained and synchronized in Excel workbooks accessible via Microsoft Graph API.  | Microsoft Excel OAuth2 API integration                    |
| The workflow’s code nodes must be maintained carefully; date handling and filtering logic are critical to accurate notifications.| Code maintenance and debugging tips                      |
| For testing, run nodes individually with sample data to validate absence detection and messaging content before full deployment. | Testing recommendation                                  |

---

**Disclaimer:** The provided content is exclusively derived from an automated n8n workflow and adheres strictly to content policies. It contains no illegal, offensive, or protected material. All data handled are legitimate and public.