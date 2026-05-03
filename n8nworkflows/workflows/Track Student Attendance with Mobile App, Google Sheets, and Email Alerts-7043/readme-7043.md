Track Student Attendance with Mobile App, Google Sheets, and Email Alerts

https://n8nworkflows.xyz/workflows/track-student-attendance-with-mobile-app--google-sheets--and-email-alerts-7043


# Track Student Attendance with Mobile App, Google Sheets, and Email Alerts

---

### 1. Workflow Overview

This workflow enables tracking student attendance via a mobile app or QR code scanner by receiving check-in data through a webhook, formatting it, saving it in Google Sheets, sending notification emails to teachers, and confirming the check-in to the app. It is designed for educational institutions seeking an automated, no-code solution to monitor attendance in real-time.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures student check-in data via an HTTP POST webhook.
- **1.2 Data Preparation:** Formats and structures incoming JSON data with timestamp and status.
- **1.3 Data Storage:** Appends or updates the attendance record in a Google Sheets document.
- **1.4 Notification:** Sends an email alert to the relevant teacher(s) about the check-in.
- **1.5 Response Confirmation:** Returns a JSON confirmation response to the mobile app or system.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block receives incoming attendance check-in data from a mobile app or QR scanner as a POST request.

**Nodes Involved:**  
- Student Check-in  
- Sticky Note (contextual comment)

**Node Details:**  

- **Student Check-in**  
  - *Type & Role:* Webhook node acting as the entry point for the workflow.  
  - *Configuration:*  
    - HTTP Method: POST  
    - Path: `/student-checkin`  
    - Response Mode: Uses a dedicated response node downstream.  
  - *Expressions/Variables:* Expects JSON body with keys such as `student_name`, `student_id`, `class_name`.  
  - *Connections:* Output connects to the "Format Data" node.  
  - *Potential Failures:* Missing or malformed POST body, invalid JSON, network issues, invalid path requests.  
  - *Sticky Note:* "Triggered via POST request from mobile app or QR scanner"

---

#### 1.2 Data Preparation

**Overview:**  
Formats and cleans the incoming JSON data, adding current date, time, and attendance status.

**Nodes Involved:**  
- Format Data  
- Sticky Note4 (contextual comment)

**Node Details:**  

- **Format Data**  
  - *Type & Role:* Set node; restructures and enriches incoming data.  
  - *Configuration:*  
    - Extracts `student_name`, `student_id`, `class_name` from webhook JSON body.  
    - Adds current date (`yyyy-MM-dd`) and time (`HH:mm`) using `$now` helper.  
    - Adds fixed status field: `"present"`.  
  - *Expressions:* Uses n8n expressions like `{{$json.body.student_name}}` and `$now.format()`.  
  - *Connections:* Output connects to "Append or update row in sheet".  
  - *Potential Failures:* Expression evaluation errors if input data missing keys; timezone considerations for `$now`.  
  - *Sticky Note:* "Cleans and prepares incoming JSON into structured format"

---

#### 1.3 Data Storage

**Overview:**  
Appends or updates the student attendance record in a Google Sheets document.

**Nodes Involved:**  
- Append or update row in sheet  
- Sticky Note1 (contextual comment)

**Node Details:**  

- **Append or update row in sheet**  
  - *Type & Role:* Google Sheets node; writes attendance data to spreadsheet.  
  - *Configuration:*  
    - Operation: `appendOrUpdate`  
    - Document ID and Sheet ID specified (hidden in JSON but identified as placeholders).  
    - Column mapping: automatic from input JSON fields.  
    - Authentication: Service Account credentials for Google API access.  
  - *Connections:* Outputs to "Email Teacher" node.  
  - *Potential Failures:* Authentication errors, API rate limits, invalid sheet or document IDs, data mapping issues.  
  - *Sticky Note:* "Saves student check-in data into Google Sheets"

---

#### 1.4 Notification

**Overview:**  
Sends an email alert to the class teacher notifying them of the student check-in.

**Nodes Involved:**  
- Email Teacher  
- Sticky Note2 (contextual comment)

**Node Details:**  

- **Email Teacher**  
  - *Type & Role:* Email Send node; sends notification email.  
  - *Configuration:*  
    - SMTP credentials configured (example: "SMTP -test").  
    - From: `admin@google.com`  
    - To: `teachers@google.com`  
    - Subject: "Student Check-in Alert"  
    - Body text: Static message "Please check attendance sheet" (no dynamic content).  
  - *Connections:* Outputs to "Success Response".  
  - *Potential Failures:* SMTP authentication failure, network issues, invalid email addresses, throttling.  
  - *Sticky Note:* "Sends formatted check-in email to the class teacher"

---

#### 1.5 Response Confirmation

**Overview:**  
Returns a JSON response confirming the successful check-in back to the calling app or system.

**Nodes Involved:**  
- Success Response  
- Sticky Note3 (contextual comment)

**Node Details:**  

- **Success Response**  
  - *Type & Role:* Respond to Webhook node; finalizes HTTP response to client.  
  - *Configuration:*  
    - Response mode: JSON  
    - Response body includes:  
      - `success: true`  
      - `message`: includes student name from "Format Data" node.  
      - `time`: check-in time from "Format Data" node.  
  - *Expressions:* Uses expressions to dynamically insert fields from upstream node JSON (`$('Format Data').item.json.student_name`).  
  - *Connections:* No output, ends the flow.  
  - *Potential Failures:* Expression failures if upstream data missing; webhook timeout if delayed.  
  - *Sticky Note:* "Returns a confirmation response to the mobile app or system"

---

### 3. Summary Table

| Node Name                 | Node Type                | Functional Role                         | Input Node(s)          | Output Node(s)                  | Sticky Note                                                                 |
|---------------------------|--------------------------|---------------------------------------|------------------------|-------------------------------|-----------------------------------------------------------------------------|
| Student Check-in           | Webhook                  | Receives student check-in POST data   | (entry point)          | Format Data                   | Triggered via POST request from mobile app or QR scanner                    |
| Format Data               | Set                      | Formats and enriches input data       | Student Check-in       | Append or update row in sheet | Cleans and prepares incoming JSON into structured format                    |
| Append or update row in sheet | Google Sheets           | Saves attendance record to spreadsheet| Format Data            | Email Teacher                 | Saves student check-in data into Google Sheets                              |
| Email Teacher             | Email Send               | Sends notification email to teacher   | Append or update row in sheet | Success Response           | Sends formatted check-in email to the class teacher                        |
| Success Response          | Respond to Webhook       | Sends confirmation response to client | Email Teacher          | (none)                       | Returns a confirmation response to the mobile app or system                |
| Sticky Note               | Sticky Note              | Contextual comment                    | (none)                 | (none)                       | Triggered via POST request from mobile app or QR scanner                    |
| Sticky Note4              | Sticky Note              | Contextual comment                    | (none)                 | (none)                       | Cleans and prepares incoming JSON into structured format                    |
| Sticky Note1              | Sticky Note              | Contextual comment                    | (none)                 | (none)                       | Saves student check-in data into Google Sheets                              |
| Sticky Note2              | Sticky Note              | Contextual comment                    | (none)                 | (none)                       | Sends formatted check-in email to the class teacher                        |
| Sticky Note3              | Sticky Note              | Contextual comment                    | (none)                 | (none)                       | Returns a confirmation response to the mobile app or system                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node ("Student Check-in")**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `student-checkin`  
   - Response Mode: Use response node  
   - Purpose: Receive check-in data from mobile app or QR scanner  

2. **Create Set Node ("Format Data")**  
   - Type: Set  
   - Configure fields:  
     - `student_name` → Expression: `{{$json.body.student_name}}`  
     - `student_id` → Expression: `{{$json.body.student_id}}`  
     - `class_name` → Expression: `{{$json.body.class_name}}`  
     - `date` → Expression: `{{$now.format('yyyy-MM-dd')}}`  
     - `time` → Expression: `{{$now.format('HH:mm')}}`  
     - `status` → Static value: `"present"`  
   - Connect output of "Student Check-in" to this node  

3. **Create Google Sheets Node ("Append or update row in sheet")**  
   - Type: Google Sheets  
   - Operation: `appendOrUpdate`  
   - Document ID: Set your Google Sheet document ID  
   - Sheet Name/ID: Set your target sheet name or ID  
   - Columns: Auto-map input data fields (student_name, student_id, class_name, date, time, status)  
   - Authentication: Use Google Service Account credentials with Sheets API enabled  
   - Connect output of "Format Data" node to this node  

4. **Create Email Send Node ("Email Teacher")**  
   - Type: Email Send  
   - SMTP Credentials: Configure SMTP credentials (e.g., Gmail SMTP or other)  
   - From Email: `admin@google.com` (or your admin email)  
   - To Email: `teachers@google.com` (or your recipient teacher emails)  
   - Subject: `Student Check-in Alert`  
   - Body Text: `Please check attendance sheet` (static)  
   - Connect output of Google Sheets node to this node  

5. **Create Respond to Webhook Node ("Success Response")**  
   - Type: Respond to Webhook  
   - Response Mode: JSON  
   - Response Body:  
     ```json
     {
       "success": true,
       "message": "{{ $('Format Data').item.json.student_name }} checked in successfully",
       "time": "{{ $('Format Data').item.json.time }}"
     }
     ```  
   - Connect output of "Email Teacher" node to this node  

6. **Connect Nodes Sequentially:**  
   - Student Check-in → Format Data → Append or update row in sheet → Email Teacher → Success Response  

7. **Optional:** Add Sticky Notes for documentation and clarity on each block.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                  |
|------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow is designed for no-code attendance tracking using mobile apps. | Workflow purpose summary                         |
| SMTP credentials must be configured with valid email server access.          | Email Teacher node setup                         |
| Google Service Account requires Sheets API enabled and proper sharing rights.| Google Sheets node requirements                  |
| Date/time uses n8n `$now` helper function; ensure timezone is set properly.  | Data Preparation block                            |
| Webhook path `student-checkin` must be accessible from mobile app or scanner.| Webhook node deployment considerations           |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---