Track Student Arrival with iOS Automation, Google Sheets & Email Alerts

https://n8nworkflows.xyz/workflows/track-student-arrival-with-ios-automation--google-sheets---email-alerts-7044


# Track Student Arrival with iOS Automation, Google Sheets & Email Alerts

### 1. Workflow Overview

This workflow, titled **"Geofenced Student Activity Tracker via iOS Automation"**, is designed to track student arrivals at a school using location data sent from an iOS device. It integrates iOS location updates with Google Sheets for logging, and sends email notifications to teachers and parents when a student arrives at the school campus.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives location updates via a webhook triggered by an iOS Shortcut.
- **1.2 Data Extraction & Processing:** Parses and formats incoming location data, extracting relevant metadata such as student identity, location, and timestamp.
- **1.3 Arrival Verification:** Checks if the incoming activity marks the student as "arrived".
- **1.4 Logging to Google Sheets:** Appends arrival data to a Google Sheet for record-keeping.
- **1.5 Geofence Confirmation:** Verifies that the location corresponds to the "School Campus" geofence.
- **1.6 Notifications:** Sends arrival alert emails to the teacher and location updates to the parent.
- **1.7 Response:** Sends a JSON response back to the iOS device confirming successful processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Accepts POST requests from iOS automation containing student location data.
- **Nodes Involved:**  
  - Location Update Webhook
  - Sticky Note

- **Node Details:**

  - **Location Update Webhook**  
    - *Type:* Webhook  
    - *Role:* Entry point; receives POST requests at the `/student-location` endpoint.  
    - *Config:* HTTP Method: POST, Response Mode: Respond via dedicated node.  
    - *Input:* External HTTP POST from iOS Shortcut with JSON body.  
    - *Output:* Passes raw request data to next node.  
    - *Edge Cases:* Missing or malformed data may cause downstream failures if not validated.  
    - *Sticky Note:* "Triggered via iOS Shortcut when student location updates."

---

#### 2.2 Data Extraction & Processing

- **Overview:** Extracts relevant student and location metadata from the incoming request and adds timestamp details.
- **Nodes Involved:**  
  - Process Location Data  
  - Sticky Note1

- **Node Details:**

  - **Process Location Data**  
    - *Type:* Set Node  
    - *Role:* Maps incoming JSON fields to workflow variables; adds formatted timestamps (`date`, `time`, `timestamp`).  
    - *Config:* Extracts `student_name`, `student_id`, `location_name`, `activity_type`, `latitude`, `longitude` from incoming JSON body; uses `$now.format()` for timestamps.  
    - *Input:* JSON from webhook node.  
    - *Output:* JSON with clean, structured data for conditional checks and logging.  
    - *Edge Cases:* If any expected fields are missing, expressions will evaluate to empty strings; no explicit validation.  
    - *Sticky Note:* "Extracts coordinates and metadata."

---

#### 2.3 Arrival Verification

- **Overview:** Determines if the student's activity indicates arrival.
- **Nodes Involved:**  
  - Student Arrived?  
  - Sticky Note2

- **Node Details:**

  - **Student Arrived?**  
    - *Type:* If Node  
    - *Role:* Checks if `activity_type` equals `"arrived"` (case-insensitive).  
    - *Config:* Condition: `$json.activity_type == "arrived"`.  
    - *Input:* Data from "Process Location Data".  
    - *Output:*  
      - True branch: continues workflow; student has arrived.  
      - False branch: no further processing (no output connection).  
    - *Edge Cases:* Case sensitivity disabled; other activity types ignored.  
    - *Sticky Note:* "Checks if student entered school zone."

---

#### 2.4 Logging to Google Sheets

- **Overview:** Records student arrival details in a Google Sheet via Google Sheets API.
- **Nodes Involved:**  
  - Log School Arrival  
  - Sticky Note3

- **Node Details:**

  - **Log School Arrival**  
    - *Type:* HTTP Request  
    - *Role:* Appends arrival data to a Google Sheets spreadsheet using Google Sheets API v4.  
    - *Config:*  
      - URL includes spreadsheet ID placeholder `YOUR_SHEET_ID` and endpoint to append values (`Activities` sheet).  
      - HTTP method: POST (implied by append operation).  
      - Value input option set to RAW.  
      - Authentication: OAuth2 credential for Google Sheets.  
    - *Input:* JSON data from arrival verification node.  
    - *Output:* Passes response to geofence verification node.  
    - *Edge Cases:*  
      - API quota limits or invalid credentials.  
      - Spreadsheet ID must be replaced with actual sheet ID.  
    - *Sticky Note:* "Adds arrival data to Google Sheet."

---

#### 2.5 Geofence Confirmation

- **Overview:** Confirms that the student is physically at the "School Campus" location before sending notifications.
- **Nodes Involved:**  
  - At School?  
  - Sticky Note4

- **Node Details:**

  - **At School?**  
    - *Type:* If Node  
    - *Role:* Checks if `location_name` equals `"School Campus"` (case-insensitive).  
    - *Config:* Condition: `$json.location_name == "School Campus"`.  
    - *Input:* Output from Google Sheets logging.  
    - *Output:*  
      - True branch: sends notifications.  
      - False branch: sends notification only to parent (see connections).  
    - *Edge Cases:* Location naming must be consistent; misspellings or case mismatches may block notification.  
    - *Sticky Note:* "Double-checks geofence condition before notifying."

---

#### 2.6 Notifications

- **Overview:** Sends email alerts to the teacher (if at school) and parent (always) about the student's location.
- **Nodes Involved:**  
  - Notify Teacher  
  - Notify Parent  
  - Sticky Note6

- **Node Details:**

  - **Notify Teacher**  
    - *Type:* Email Send  
    - *Role:* Sends an email alert to teacher about student arrival.  
    - *Config:*  
      - Subject: "Student Arrival Alert"  
      - To: teacher@google.com (placeholder)  
      - From: admin@google.com (placeholder)  
      - SMTP credentials used for sending.  
    - *Input:* True branch from "At School?" node.  
    - *Output:* Passes to success response node.  
    - *Edge Cases:* SMTP connection failure, invalid email addresses.  
    - *Sticky Note:* "Sends email if student is confirmed at school."

  - **Notify Parent**  
    - *Type:* Email Send  
    - *Role:* Sends an email with child's location update to parent.  
    - *Config:*  
      - Subject: "Your Child's Location Update"  
      - To: parent@google.com (placeholder)  
      - From: admin@google.com (placeholder)  
      - SMTP credentials used for sending.  
    - *Input:* False branch from "At School?" node.  
    - *Output:* Passes to success response node.  
    - *Edge Cases:* Same as "Notify Teacher".  

---

#### 2.7 Response

- **Overview:** Returns a JSON response to the iOS device confirming successful processing.
- **Nodes Involved:**  
  - Success Response  
  - Sticky Note5

- **Node Details:**

  - **Success Response**  
    - *Type:* Respond to Webhook  
    - *Role:* Sends a 200 OK JSON response including student name, location, and time extracted from processed data.  
    - *Config:*  
      - Response body uses expressions referencing "Process Location Data" node fields.  
      - Response format: JSON with keys `success`, `message`, `location`, and `time`.  
    - *Input:* From notification nodes.  
    - *Output:* Ends workflow.  
    - *Edge Cases:* If referenced nodes fail or data is missing, expressions may return empty fields.  
    - *Sticky Note:* "Returns a 200 response to the triggering device."

---

### 3. Summary Table

| Node Name             | Node Type            | Functional Role                            | Input Node(s)           | Output Node(s)                | Sticky Note                                                  |
|-----------------------|----------------------|-------------------------------------------|-------------------------|------------------------------|--------------------------------------------------------------|
| Location Update Webhook| Webhook              | Receives POST requests from iOS Shortcut | -                       | Process Location Data         | Triggered via iOS Shortcut when student location updates     |
| Process Location Data  | Set                  | Extracts and formats location metadata    | Location Update Webhook  | Student Arrived?              | Extracts coordinates and metadata                            |
| Student Arrived?       | If                   | Checks if activity is "arrived"            | Process Location Data    | Log School Arrival (true)     | Checks if student entered school zone                        |
| Log School Arrival     | HTTP Request         | Appends arrival data to Google Sheet      | Student Arrived?         | At School?                   | Adds arrival data to Google Sheet                            |
| At School?             | If                   | Checks if location is "School Campus"      | Log School Arrival       | Notify Teacher (true)         | Double-checks geofence condition before notifying           |
|                       |                      |                                           |                         | Notify Parent (false)         |                                                              |
| Notify Teacher         | Email Send           | Sends email alert to teacher               | At School? (true)        | Success Response             | Sends email if student is confirmed at school               |
| Notify Parent          | Email Send           | Sends email alert to parent                | At School? (false)       | Success Response             |                                                              |
| Success Response       | Respond to Webhook   | Returns JSON response to iOS device        | Notify Teacher, Notify Parent | -                      | Returns a 200 response to the triggering device             |
| Sticky Note            | Sticky Note          | Comment on webhook trigger                  | -                       | -                            | Triggered via iOS Shortcut when student location updates     |
| Sticky Note1           | Sticky Note          | Comment on data extraction                   | -                       | -                            | Extracts coordinates and metadata                            |
| Sticky Note2           | Sticky Note          | Comment on arrival check                      | -                       | -                            | Checks if student entered school zone                        |
| Sticky Note3           | Sticky Note          | Comment on logging to Google Sheets          | -                       | -                            | Adds arrival data to Google Sheet                            |
| Sticky Note4           | Sticky Note          | Comment on geofence check                      | -                       | -                            | Double-checks geofence condition before notifying           |
| Sticky Note5           | Sticky Note          | Comment on webhook response                   | -                       | -                            | Returns a 200 response to the triggering device             |
| Sticky Note6           | Sticky Note          | Comment on notification emails                 | -                       | -                            | Sends email if student is confirmed at school               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Location Update Webhook`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `student-location`  
   - Response Mode: `Response Node`  
   - Purpose: Receive location data from iOS Shortcut.

2. **Create Set Node**  
   - Name: `Process Location Data`  
   - Type: Set  
   - Add the following string fields with expressions:  
     - `student_name` = `{{$json["body"]["student_name"]}}`  
     - `student_id` = `{{$json["body"]["student_id"]}}`  
     - `location_name` = `{{$json["body"]["location_name"]}}`  
     - `activity_type` = `{{$json["body"]["activity_type"]}}`  
     - `latitude` = `{{$json["body"]["latitude"]}}`  
     - `longitude` = `{{$json["body"]["longitude"]}}`  
     - `timestamp` = `{{$now.format("yyyy-MM-dd HH:mm:ss")}}`  
     - `date` = `{{$now.format("yyyy-MM-dd")}}`  
     - `time` = `{{$now.format("HH:mm")}}`  
   - Connect output of `Location Update Webhook` to this node.

3. **Create If Node**  
   - Name: `Student Arrived?`  
   - Type: If  
   - Condition:  
     - Check if `$json.activity_type` equals `"arrived"` (case-insensitive)  
   - Connect output of `Process Location Data` to this node.

4. **Create HTTP Request Node**  
   - Name: `Log School Arrival`  
   - Type: HTTP Request  
   - Method: POST (implied by append operation)  
   - URL: `https://sheets.googleapis.com/v4/spreadsheets/YOUR_SHEET_ID/values/Activities:append?valueInputOption=RAW` (replace `YOUR_SHEET_ID` with actual Google Sheet ID)  
   - Authentication: Select or create Google Sheets OAuth2 credential.  
   - Send Body: Enable, send JSON data from input.  
   - Connect `Student Arrived?` True output to this node.

5. **Create If Node**  
   - Name: `At School?`  
   - Type: If  
   - Condition: Check if `$json.location_name` equals `"School Campus"` (case-insensitive)  
   - Connect output of `Log School Arrival` to this node.

6. **Create Email Send Node**  
   - Name: `Notify Teacher`  
   - Type: Email Send  
   - Subject: `"Student Arrival Alert"`  
   - To Email: `"teacher@google.com"` (replace with actual teacher email)  
   - From Email: `"admin@google.com"` (set appropriate sender)  
   - Credentials: Configure SMTP credentials for email sending.  
   - Connect `At School?` True output to this node.

7. **Create Email Send Node**  
   - Name: `Notify Parent`  
   - Type: Email Send  
   - Subject: `"Your Child's Location Update"`  
   - To Email: `"parent@google.com"` (replace with actual parent email)  
   - From Email: `"admin@google.com"`  
   - Credentials: Use same SMTP credentials.  
   - Connect `At School?` False output to this node.

8. **Create Respond to Webhook Node**  
   - Name: `Success Response`  
   - Type: Respond to Webhook  
   - Respond With: JSON  
   - Response Body:  
     ```json
     {
       "success": true,
       "message": "Location updated for {{$node["Process Location Data"].json.student_name}}",
       "location": "{{$node["Process Location Data"].json.location_name}}",
       "time": "{{$node["Process Location Data"].json.time}}"
     }
     ```  
   - Connect output of both `Notify Teacher` and `Notify Parent` to this node.

9. **Connect all nodes as per above steps** ensuring the workflow branches and merges correctly.

10. **Test the workflow** by sending a POST request with sample JSON payload from an iOS Shortcut or HTTP client:  
    ```json
    {
      "student_name": "John Doe",
      "student_id": "12345",
      "location_name": "School Campus",
      "activity_type": "arrived",
      "latitude": 12.345678,
      "longitude": 98.7654321
    }
    ```

11. **Replace placeholders:**  
    - Google Sheets spreadsheet ID in the HTTP Request URL.  
    - SMTP email addresses and credentials.  
    - Verify Google Sheets OAuth2 credentials have correct scopes (`https://www.googleapis.com/auth/spreadsheets`).  

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                         |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow is triggered by an iOS Shortcut that posts location updates to the webhook.       | iOS Automation Integration                              |
| Google Sheets API v4 is used with OAuth2 for appending data; ensure proper API credentials.      | https://developers.google.com/sheets/api               |
| SMTP credentials must be set up with valid email provider settings for sending notifications.    | SMTP configuration details depend on email provider.  |
| Placeholder emails (teacher@google.com, parent@google.com) should be replaced by actual addresses.| Email notification configuration                        |
| Timestamp formatting uses n8n’s `$now.format()` function with `yyyy-MM-dd HH:mm:ss` pattern.     | n8n Date/Time functions                                  |
| Ensure consistent naming in location data for geofence checks, e.g., "School Campus".            | Geofence logic depends on exact string matching.        |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.