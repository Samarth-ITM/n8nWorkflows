Automated Student Progress Reports from LMS to Parents via Gmail & Google Sheets

https://n8nworkflows.xyz/workflows/automated-student-progress-reports-from-lms-to-parents-via-gmail---google-sheets-6998


# Automated Student Progress Reports from LMS to Parents via Gmail & Google Sheets

---

### 1. Workflow Overview

This workflow automates the generation and delivery of weekly academic progress reports for students by integrating data from a Learning Management System (LMS), Google Sheets, and Gmail. Its primary use case is to send personalized performance reports to parents, summarizing student grades, assignment completion, attendance, and teacher comments, while also logging delivery status and notifying school administration.

The workflow is logically structured into the following blocks:

- **1.1 Scheduled Trigger and Student Data Retrieval**: Initiates the workflow weekly and fetches the student list from Google Sheets.
- **1.2 Batch Processing and LMS Data Fetch**: Processes students in batches to manage load and requests detailed academic data from the LMS API.
- **1.3 Academic Data Processing and Report Generation**: Analyzes LMS data to compute performance metrics, trends, and prepares a styled HTML report.
- **1.4 Email Delivery and Logging**: Sends the report to parents via Gmail, logs the delivery status in Google Sheets, and sends an administrative summary email.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Student Data Retrieval

**Overview:**  
This block triggers the workflow every Monday at 9 AM and retrieves the list of students from a Google Sheets document serving as the student database.

**Nodes Involved:**  
- Weekly Schedule Trigger  
- Get Students List  

**Node Details:**

- **Weekly Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Starts the workflow weekly on Mondays at 9:00 AM.  
  - *Configuration:* Interval set to weekly, triggering on Monday at 9 AM.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Starts the flow to “Get Students List.”  
  - *Edge Cases:* If the node is disabled or schedule misconfigured, the workflow will not start.  
  - *Version:* 1.2  

- **Get Students List**  
  - *Type:* Google Sheets Node  
  - *Role:* Retrieves the list of students from a specific sheet ("Students") in a Google Sheets document.  
  - *Configuration:*  
    - Document ID: Placeholder "YOUR_STUDENT_SHEET_ID" (must be replaced).  
    - Sheet Name: "Students".  
    - Authentication: Google Service Account credentials used.  
  - *Input:* Trigger from the schedule node.  
  - *Output:* Emits list of students with details such as student_id, student_name, parent_email, grade_level.  
  - *Edge Cases:*  
    - Invalid credentials or permissions may cause failure.  
    - Missing or malformed sheet data can result in empty or incorrect student lists.  
  - *Version:* 4.6  

---

#### 2.2 Batch Processing and LMS Data Fetch

**Overview:**  
This block splits the student list into manageable batches (5 students per batch) to prevent overloading the LMS API and fetches detailed academic data for each student via API calls.

**Nodes Involved:**  
- Split Students for Processing  
- Fetch LMS Academic Data  

**Node Details:**

- **Split Students for Processing**  
  - *Type:* SplitInBatches  
  - *Role:* Divides the student list into batches of 5 for sequential processing.  
  - *Configuration:* Batch size set to 5.  
  - *Input:* List of students from “Get Students List.”  
  - *Output:* Emits batches of student data for downstream processing.  
  - *Edge Cases:*  
    - If batch size is too large, API rate limits could be triggered.  
    - If batch size is zero or negative, the node will error.  
  - *Version:* 3  

- **Fetch LMS Academic Data**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves academic data (grades, assignments, attendance, teacher comments) for each student from the LMS API.  
  - *Configuration:*  
    - URL template uses credentials for base LMS API URL and appends student_id dynamically.  
    - Query parameters include period="current_week", include_assignments="true".  
    - Headers include Authorization with Bearer token and Content-Type JSON.  
    - Credentials: LMS API base URL and API token stored securely.  
  - *Input:* Batch student data from "Split Students for Processing."  
  - *Output:* LMS academic data JSON per student.  
  - *Edge Cases:*  
    - API token expiry or incorrect token causes auth errors.  
    - Network timeouts or API downtime.  
    - Unexpected data format could cause downstream errors.  
  - *Version:* 4.2  

---

#### 2.3 Academic Data Processing and Report Generation

**Overview:**  
Processes the LMS data to calculate performance metrics, analyze trends, and generate a personalized HTML report formatted for email.

**Nodes Involved:**  
- Process Academic Data  
- Generate HTML Report  

**Node Details:**

- **Process Academic Data**  
  - *Type:* Code (JavaScript)  
  - *Role:* Aggregates and computes student performance statistics and trends from LMS data and student info.  
  - *Configuration:*  
    - Parses grades and assignments.  
    - Calculates total and average grade.  
    - Computes assignment completion rate.  
    - Determines attendance rate.  
    - Assesses performance trend (Improving, Declining, Stable) based on recent grades.  
    - Extracts subjects needing attention (grades below 70).  
    - Includes teacher comments or defaults to "No recent comments".  
  - *Input:* Student details + LMS data.  
  - *Output:* Structured JSON with computed metrics and report date.  
  - *Key Expressions:* Uses JavaScript standard Array methods (reduce, filter, map).  
  - *Edge Cases:*  
    - Missing grades or assignments arrays defaults to safe values.  
    - Division by zero prevented by checks on array lengths.  
    - Date formatting uses ISO standard.  
  - *Version:* 2  

- **Generate HTML Report**  
  - *Type:* Code (JavaScript)  
  - *Role:* Constructs a styled HTML email report embedding the processed data.  
  - *Configuration:*  
    - Template includes header, performance overview, trend, assignment status, alerts for subjects needing attention, teacher comments, and footer.  
    - Uses inline CSS for consistent email rendering.  
    - Dynamically injects student-specific data (e.g., name, grades, dates).  
    - Prepares email subject line including student name and report date.  
  - *Input:* JSON with performance metrics from previous node.  
  - *Output:* JSON including `html_report` (HTML string) and `email_subject`.  
  - *Edge Cases:*  
    - Ensures fallback text for missing recent grades or subjects needing attention.  
    - HTML formatting should be tested across email clients.  
  - *Version:* 2  

---

#### 2.4 Email Delivery and Logging

**Overview:**  
Sends the generated HTML report to parents via Gmail, logs the delivery status in Google Sheets, and sends a summary email to the school administrator.

**Nodes Involved:**  
- Send Email to Parents  
- Log Report Delivery  
- Send Admin Summary  

**Node Details:**

- **Send Email to Parents**  
  - *Type:* Gmail Node  
  - *Role:* Sends the weekly progress report email to each student's parent email.  
  - *Configuration:*  
    - Recipient: Dynamic parent_email from JSON.  
    - Subject: Dynamic from generated email_subject.  
    - Message body: HTML content from html_report.  
    - Additional option: CC to teacher@school.edu.  
    - Authentication: Gmail OAuth2 credentials configured.  
  - *Input:* Output from the HTML report generation.  
  - *Output:* Passes success status downstream.  
  - *Edge Cases:*  
    - Gmail API rate limits or quota exceeded.  
    - Invalid parent email addresses cause delivery failure.  
    - OAuth token expiry or invalid credentials.  
  - *Version:* 2.1  

- **Log Report Delivery**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends a new row logging the report delivery status with timestamp, student info, and status.  
  - *Configuration:*  
    - Target Sheet: "Delivery_Log" in separate Google Sheets document (placeholder "YOUR_LOG_SHEET_ID").  
    - Data columns: timestamp, student_id, parent_email, student_name, overall_grade, delivery_status (set to "Sent Successfully").  
    - Authentication: Google Service Account.  
  - *Input:* Output from email sending node.  
  - *Output:* Passes to admin summary node.  
  - *Edge Cases:*  
    - Sheet ID incorrect or access denied results in logging failure.  
    - Concurrency issues if multiple runs append simultaneously.  
  - *Version:* 4.6  

- **Send Admin Summary**  
  - *Type:* Gmail Node  
  - *Role:* Sends a summary email to admin confirming completion of the weekly report sending.  
  - *Configuration:*  
    - Recipient: admin@school.edu (static).  
    - Subject: Static "Weekly Student Progress Reports Complete".  
    - Message body: HTML formatted with current date and status confirmation.  
    - Authentication: Gmail OAuth2 credentials.  
  - *Input:* Output from log report delivery node.  
  - *Output:* End of workflow.  
  - *Edge Cases:*  
    - Failure to send admin email does not affect prior steps but should be monitored.  
  - *Version:* 2.1  

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                                   | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                  |
|---------------------------|--------------------|--------------------------------------------------|----------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Weekly Schedule Trigger    | Schedule Trigger   | Initiates workflow weekly on Monday at 9 AM     | —                          | Get Students List            |                                                                                              |
| Get Students List          | Google Sheets      | Retrieves student list from Google Sheets        | Weekly Schedule Trigger     | Split Students for Processing|                                                                                              |
| Split Students for Processing | SplitInBatches    | Splits students into batches of 5                 | Get Students List           | Fetch LMS Academic Data      |                                                                                              |
| Fetch LMS Academic Data    | HTTP Request      | Fetches academic data for each student from LMS API | Split Students for Processing | Process Academic Data        |                                                                                              |
| Process Academic Data      | Code (JavaScript) | Computes performance metrics and trends           | Fetch LMS Academic Data     | Generate HTML Report         |                                                                                              |
| Generate HTML Report       | Code (JavaScript) | Builds styled HTML report and email subject       | Process Academic Data       | Send Email to Parents        |                                                                                              |
| Send Email to Parents      | Gmail             | Sends report email to parents                      | Generate HTML Report        | Log Report Delivery          |                                                                                              |
| Log Report Delivery       | Google Sheets      | Logs delivery status in Google Sheets              | Send Email to Parents       | Send Admin Summary           |                                                                                              |
| Send Admin Summary        | Gmail             | Sends summary email to admin after reports sent    | Log Report Delivery         | —                           |                                                                                              |
| Workflow Info             | Sticky Note       | Describes workflow features and schedule           | —                          | —                           | ## 📚 Student Academic Progress Report Generator\n\n### Features:\n• Weekly automated reports\n• LMS data integration\n• Parent email notifications\n• Performance trend analysis\n• HTML formatted reports\n• Admin summaries\n\n### Schedule: Every Monday at 9 AM |
| Setup Required            | Sticky Note       | Lists configuration steps needed before running    | —                          | —                           | ## ⚙️ Setup Required\n\n1. Replace YOUR_STUDENT_SHEET_ID\n2. Replace YOUR_LOG_SHEET_ID\n3. Configure LMS API credentials\n4. Set up Gmail credentials\n5. Update email addresses |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Weekly Schedule Trigger" Node**  
   - Type: Schedule Trigger  
   - Configure to trigger weekly on Mondays at 9:00 AM.

2. **Create "Get Students List" Node**  
   - Type: Google Sheets  
   - Connect input from "Weekly Schedule Trigger".  
   - Set Authentication to Google Service Account credentials.  
   - Set Document ID to your Google Sheets document containing student data (replace placeholder).  
   - Set Sheet Name to "Students".  
   - Configure to read all rows (default).

3. **Create "Split Students for Processing" Node**  
   - Type: SplitInBatches  
   - Connect input from "Get Students List".  
   - Set Batch Size to 5.

4. **Create "Fetch LMS Academic Data" Node**  
   - Type: HTTP Request  
   - Connect input from "Split Students for Processing".  
   - Configure URL as `{{ $credentials.lmsApi.baseUrl }}/api/students/{{ $json.student_id }}/grades`.  
   - Set Query Parameters: period=current_week, include_assignments=true.  
   - Add Header Parameters: Authorization: Bearer `{{ $credentials.lmsApi.apiToken }}`, Content-Type: application/json.  
   - Set Credentials: LMS API credentials with baseUrl and apiToken.

5. **Create "Process Academic Data" Node**  
   - Type: Code (JavaScript)  
   - Connect input from "Fetch LMS Academic Data".  
   - Paste the provided JavaScript code that calculates grades, trends, attendance, and flags subjects needing attention.

6. **Create "Generate HTML Report" Node**  
   - Type: Code (JavaScript)  
   - Connect input from "Process Academic Data".  
   - Paste the provided HTML template code embedding student data into a styled report.  
   - Ensure email subject is dynamically generated.

7. **Create "Send Email to Parents" Node**  
   - Type: Gmail  
   - Connect input from "Generate HTML Report".  
   - Configure "Send To" as `{{ $json.parent_email }}`.  
   - Set "Subject" as `{{ $json.email_subject }}`.  
   - Set "Message" as `{{ $json.html_report }}`.  
   - Add CC list with teacher’s email (e.g., teacher@school.edu).  
   - Use Gmail OAuth2 credentials configured for your account.

8. **Create "Log Report Delivery" Node**  
   - Type: Google Sheets  
   - Connect input from "Send Email to Parents".  
   - Configure to append rows to a "Delivery_Log" sheet in a Google Sheets document (replace placeholder ID).  
   - Map columns: timestamp (current time), student_id, parent_email, student_name, overall_grade, delivery_status ("Sent Successfully").  
   - Use Google Service Account credentials.

9. **Create "Send Admin Summary" Node**  
   - Type: Gmail  
   - Connect input from "Log Report Delivery".  
   - Set recipient to admin@school.edu.  
   - Set subject to "📊 Weekly Student Progress Reports Complete".  
   - Compose a simple HTML message summarizing the weekly run with date and status.  
   - Use Gmail OAuth2 credentials.

10. **Add Sticky Notes (Optional for Documentation)**  
    - Add a note describing workflow features and schedule.  
    - Add a note listing setup instructions including replacing placeholders and configuring credentials.

11. **Connect Nodes**  
    - Weekly Schedule Trigger → Get Students List → Split Students for Processing → Fetch LMS Academic Data → Process Academic Data → Generate HTML Report → Send Email to Parents → Log Report Delivery → Send Admin Summary.

12. **Credentials Setup**  
    - Configure Google Service Account credentials for Google Sheets nodes.  
    - Configure LMS API credentials with base URL and API token.  
    - Configure Gmail OAuth2 credentials for sending emails.

13. **Testing & Validation**  
    - Test with a small student list and monitor API responses and email delivery.  
    - Confirm logs are appended correctly.  
    - Validate email formatting across common email clients.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                       |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------|
| Workflow automates weekly student progress reporting via LMS integration and email notifications.   | Workflow overview and purpose                         |
| Ensure all placeholders such as YOUR_STUDENT_SHEET_ID and YOUR_LOG_SHEET_ID are replaced correctly.| Setup Required sticky note                            |
| Gmail OAuth2 credentials require proper consent and scope for sending emails on behalf of the user.| Gmail node credential configuration                   |
| LMS API token should be kept secure and refreshed as needed to avoid authorization failures.       | API security best practices                           |
| HTML email uses inline CSS to maximize compatibility with email clients like Gmail, Outlook, etc. | Email formatting considerations                       |
| Performance trend logic uses simple moving average comparison for last 3 grades; adjust as needed. | Performance analysis logic in "Process Academic Data" node |
| Admin summary email provides confirmation but does not include detailed error reporting.           | Potential enhancement for error monitoring            |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---