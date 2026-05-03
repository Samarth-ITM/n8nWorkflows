Monitor Medical Equipment with Google Sheets & Send WhatsApp and Email Alerts

https://n8nworkflows.xyz/workflows/monitor-medical-equipment-with-google-sheets---send-whatsapp-and-email-alerts-7249


# Monitor Medical Equipment with Google Sheets & Send WhatsApp and Email Alerts

### 1. Workflow Overview

This workflow automates the monitoring of medical equipment maintenance status using data stored in a Google Sheets document. It runs daily at 6 AM, reads equipment data, processes various maintenance and usage alerts, and sends notifications via email and WhatsApp to technicians and supervisors. It also updates the equipment status back to the sheet and logs alerts for historical tracking.

**Target Use Cases:**  
- Hospital or medical facility equipment maintenance tracking  
- Automated alerting for overdue or upcoming maintenance/calibration  
- Notification escalation based on alert priority  
- Centralized logging and status updating for equipment management teams

**Logical Blocks:**  
- **1.1 Scheduled Trigger and Data Retrieval:** Initiates workflow daily and fetches equipment data from Google Sheets  
- **1.2 Alert Processing and Classification:** Processes equipment data to detect maintenance, calibration, usage, warranty, and status alerts, assigning priorities  
- **1.3 Alert Filtering and Notification Dispatch:** Filters equipment with alerts, sends email and WhatsApp notifications to technicians, and escalates critical alerts to supervisors  
- **1.4 Overdue Maintenance Handling and Status Update:** Filters overdue equipment and updates their status in Google Sheets  
- **1.5 Alert Logging:** Logs all generated alerts into a dedicated Google Sheets tab for record-keeping  
- **1.6 Workflow Control:** Includes delay/wait steps to manage execution pacing

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger and Data Retrieval

- **Overview:**  
Starts the workflow daily at 6 AM and reads the equipment data from a specified Google Sheet using a service account.

- **Nodes Involved:**  
  - Daily Equipment Check (6 AM)  
  - Read Equipment Data

- **Node Details:**  

  - **Daily Equipment Check (6 AM)**  
    - Type: Cron Trigger  
    - Configuration: Runs once daily at 6 AM (default cron with no parameters means 6 AM assumed or default)  
    - Input: None (trigger)  
    - Output: Triggers the next node once daily  
    - Edge Cases: Cron misconfiguration or time zone mismatch may cause missed runs

  - **Read Equipment Data**  
    - Type: Google Sheets (Read)  
    - Configuration: Reads all rows from the first sheet (`gid=0`) of the document identified by `YOUR_GOOGLE_SHEET_ID` using a service account for authentication  
    - Key Expressions: Sheet name fixed to first sheet, document ID must be replaced with actual Google Sheet ID  
    - Input: Trigger from Cron node  
    - Output: Outputs all rows as JSON objects to next node  
    - Edge Cases: If the document ID is invalid or permissions are insufficient, node will fail authentication or return no data  

---

#### 1.2 Alert Processing and Classification

- **Overview:**  
Processes each equipment entry to detect maintenance overdue, calibration due, usage thresholds, warranty expiration, and status conditions. It generates detailed alert messages and assigns priorities.

- **Nodes Involved:**  
  - Process Equipment Alerts  
  - Tack Break For 5 Sec (Wait node)

- **Node Details:**  

  - **Process Equipment Alerts**  
    - Type: Code (JavaScript)  
    - Role: Loops over all equipment, parses dates and numeric fields, computes alert conditions, builds alert messages and metadata, sorts alerts by priority  
    - Key Logic:  
      - Skips header row  
      - Calculates days since last maintenance/calibration and days to warranty expiry  
      - Identifies alert types: OVERDUE_MAINTENANCE, UPCOMING_MAINTENANCE, OVERDUE_CALIBRATION, UPCOMING_CALIBRATION, CRITICAL_USAGE, HIGH_USAGE, WARRANTY_EXPIRED, WARRANTY_EXPIRING, OUT_OF_SERVICE, NEEDS_REPAIR  
      - Assigns priority: Normal, Medium, High, Critical  
      - Generates email subject, body, and WhatsApp message templates  
    - Inputs: Array of equipment JSON objects from Google Sheets  
    - Outputs: Array of alert JSON objects with detailed fields  
    - Edge Cases:  
      - Date parsing errors if dates are malformed  
      - Missing numeric fields handled with defaults  
      - Empty or missing contact emails/phone numbers could cause notification failures  
      - Large dataset performance  
    - Version: n8n Code node v2

  - **Tack Break For 5 Sec**  
    - Type: Wait  
    - Role: Adds a 5-second delay after processing alerts before continuing  
    - Input: Alert array from Code node  
    - Output: Passes data forward after delay  
    - Edge Cases: Delays workflow execution, potentially causing timeout if chained with long delays

---

#### 1.3 Alert Filtering and Notification Dispatch

- **Overview:**  
Filters alerts to only those with actual issues, sends emails to technicians, escalates critical alerts to supervisors via email and WhatsApp, and sends WhatsApp alerts to technicians.

- **Nodes Involved:**  
  - Filter Equipment with Alerts  
  - Send Technician Email  
  - Filter Critical Equipment  
  - Send Critical Alert to Supervisors  
  - Send Critical Alert Massage (WhatsApp)  
  - Send message (WhatsApp)  
  - Log Maintenance Alerts

- **Node Details:**

  - **Filter Equipment with Alerts**  
    - Type: Filter  
    - Role: Passes only alerts where `alertTypes.length > 0` (i.e., equipment with issues)  
    - Input: Array of all alerts  
    - Output: Alerts with issues  
    - Edge Cases: If alertTypes is missing or empty, equipment is filtered out  

  - **Send Technician Email**  
    - Type: Email Send  
    - Role: Sends alert email to technician's email with subject and body from alert  
    - Config: Uses SMTP credentials "SMTP -test"  
    - To: Technician email from alert data  
    - From: your-email@gmail.com (placeholder, to be replaced)  
    - Input: Filtered alerts with issues  
    - Edge Cases: Missing or invalid technician email, SMTP auth failure, email quota limits  

  - **Filter Critical Equipment**  
    - Type: Filter  
    - Role: Filters alerts with priority "Critical"  
    - Input: Alerts with issues  
    - Output: Critical priority alerts only  
    - Edge Cases: Case sensitivity in priority matching  

  - **Send Critical Alert to Supervisors**  
    - Type: Email Send  
    - Role: Sends critical alert email to supervisors' fixed emails (supervisor@hospital.com, maintenance.manager@hospital.com)  
    - Config: Same SMTP "SMTP -test"  
    - Subject includes equipment name with critical alert emoji  
    - Input: Critical alerts  
    - Edge Cases: Fixed recipient emails require updating for deployment environment  

  - **Send Critical Alert Massage (WhatsApp)**  
    - Type: WhatsApp  
    - Role: Sends critical alert WhatsApp message to fixed phone number +919999887776  
    - Config: WhatsApp API credentials "WhatsApp-test"  
    - Input: Critical alerts  
    - Edge Cases: Fixed recipient number, API limits, message formatting  

  - **Send message (WhatsApp)**  
    - Type: WhatsApp  
    - Role: Sends WhatsApp alert to technician's WhatsApp number from alert data  
    - Config: WhatsApp API credentials "WhatsApp-test"  
    - Message composed with equipment name and alert emoji  
    - Input: Alerts with issues (non-filtered)  
    - Edge Cases: Missing technician WhatsApp number, API errors  

  - **Log Maintenance Alerts**  
    - Type: Google Sheets (Write)  
    - Role: Logs all alerts into the second sheet (`gid=1`) of the Google Sheet document  
    - Input: Alerts with issues  
    - Output: None (write operation)  
    - Edge Cases: Google Sheets API limits, write failures, document permissions  

---

#### 1.4 Overdue Maintenance Handling and Status Update

- **Overview:**  
Filters alerts for overdue maintenance or calibration and updates equipment status in the Google Sheet accordingly.

- **Nodes Involved:**  
  - Filter Overdue Equipment  
  - Update Equipment Status

- **Node Details:**  

  - **Filter Overdue Equipment**  
    - Type: Filter  
    - Role: Filters alerts that have alert types including 'OVERDUE_MAINTENANCE' or 'OVERDUE_CALIBRATION'  
    - Input: Alerts with issues  
    - Output: Only overdue maintenance/calibration alerts  
    - Edge Cases: AlertTypes array missing or malformed  

  - **Update Equipment Status**  
    - Type: Google Sheets (Write)  
    - Role: Updates the main equipment sheet (`gid=0`) with new status information for overdue equipment  
    - Input: Overdue alerts  
    - Output: None (write operation)  
    - Edge Cases: Permissions, concurrent writes, row indexing consistency  

---

#### 1.5 Workflow Control

- **Overview:**  
Includes a wait step to pace processing between alert creation and filtering.

- **Nodes Involved:**  
  - Tack Break For 5 Sec (covered above in Block 1.2)

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                                  | Input Node(s)              | Output Node(s)                                           | Sticky Note                                                                                                                                |
|-----------------------------|---------------------|-------------------------------------------------|----------------------------|----------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Daily Equipment Check (6 AM) | Cron Trigger        | Triggers workflow daily at 6 AM                  | None                       | Read Equipment Data                                      | ## How It Works - **Daily Equipment Check (6 AM)** - Triggers the workflow.                                                                |
| Read Equipment Data          | Google Sheets Read  | Reads equipment data from Google Sheet           | Daily Equipment Check (6 AM) | Process Equipment Alerts                                 | ## How It Works - **Read Equipment Data** - Fetches data from Google Sheet.                                                                |
| Process Equipment Alerts     | Code                | Processes and generates detailed equipment alerts| Read Equipment Data         | Tack Break For 5 Sec                                    | ## How It Works - **Process Equipment Alerts** - Identifies maintenance needs.                                                             |
| Tack Break For 5 Sec         | Wait                | Adds a 5-second delay                             | Process Equipment Alerts    | Filter Equipment with Alerts                             | ## How It Works - **Task Break For 5 Sec** - Adds a delay for processing.                                                                   |
| Filter Equipment with Alerts | Filter              | Filters alerts where alertTypes.length > 0       | Tack Break For 5 Sec        | Send Technician Email, Filter Critical Equipment, Log Maintenance Alerts, Filter Overdue Equipment, Send message | ## How It Works - **Filter Equipment with Alerts** - Filters equipment needing attention.                                                  |
| Send Technician Email        | Email Send          | Sends alert emails to technicians                 | Filter Equipment with Alerts | None                                                   | ## How It Works - **Send Technician Email** - Notifies technicians via email.                                                              |
| Filter Critical Equipment    | Filter              | Filters alerts with Critical priority             | Filter Equipment with Alerts | Send Critical Alert to Supervisors, Send Critical Alert Massage | ## How It Works - **Filter Critical Equipment** - Escalates critical issues.                                                                |
| Send Critical Alert to Supervisors | Email Send   | Sends critical alert emails to supervisors        | Filter Critical Equipment   | None                                                   | ## How It Works - **Send Critical Alert to Supervisors** - Escalates critical issues via email and WhatsApp.                               |
| Send Critical Alert Massage  | WhatsApp            | Sends critical alert WhatsApp messages             | Filter Critical Equipment   | None                                                   | ## How It Works - **Send Critical Alert to Supervisors** - Escalates critical issues via email and WhatsApp.                               |
| Send message                | WhatsApp            | Sends WhatsApp alerts to technicians               | Filter Equipment with Alerts | None                                                   | ## How It Works - **Send Message (message: send)** - Sends WhatsApp alerts to technicians.                                                  |
| Log Maintenance Alerts       | Google Sheets Write | Logs alerts data into Google Sheet                  | Filter Equipment with Alerts | None                                                   | ## How It Works - **Log Maintenance Alerts** - Logs alerts in the sheet.                                                                    |
| Filter Overdue Equipment     | Filter              | Filters alerts with overdue maintenance or calibration | Filter Equipment with Alerts | Update Equipment Status                                | ## How It Works - **Filter Overdue Equipment** - Identifies overdue maintenance.                                                           |
| Update Equipment Status      | Google Sheets Write | Updates equipment status in Google Sheet            | Filter Overdue Equipment    | None                                                   | ## How It Works - **Update Equipment Status** - Updates sheet with new statuses.                                                            |
| Sticky Note                 | Sticky Note         | Provides workflow overview and step descriptions   | None                       | None                                                   | ## How It Works - Overview and explanation of each node's role and workflow logic.                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node**  
   - Name: Daily Equipment Check (6 AM)  
   - Type: Cron  
   - Set to trigger daily at 6:00 AM (configure cron expression accordingly)  

2. **Create Google Sheets Read Node**  
   - Name: Read Equipment Data  
   - Type: Google Sheets  
   - Operation: Read Rows  
   - Document ID: Replace with your target Google Sheet ID  
   - Sheet Name: Use first sheet (`gid=0`)  
   - Authentication: Use Service Account credentials configured with access to the sheet  

3. **Create Code Node for Alert Processing**  
   - Name: Process Equipment Alerts  
   - Type: Code (JavaScript)  
   - Paste the provided JavaScript code that parses equipment data and generates alerts with priority and messages  
   - Input: Connect from Google Sheets Read node  

4. **Add Wait Node**  
   - Name: Tack Break For 5 Sec  
   - Type: Wait  
   - Duration: 5 seconds  
   - Connect from Code node  

5. **Add Filter Node to Pass Only Equipment with Alerts**  
   - Name: Filter Equipment with Alerts  
   - Type: Filter  
   - Condition: `$json.alertTypes.length > 0`  
   - Connect from Wait node  

6. **Add Email Send Node for Technicians**  
   - Name: Send Technician Email  
   - Type: Email Send  
   - Configure SMTP credentials (setup SMTP with your email provider)  
   - To: `{{$json["technicianEmail"]}}`  
   - From: Your valid email address  
   - Subject: `{{$json["emailSubject"]}}`  
   - Body: Use the body generated in `$json.emailBody`  
   - Connect from Filter Equipment with Alerts node  

7. **Add Filter Node for Critical Equipment**  
   - Name: Filter Critical Equipment  
   - Type: Filter  
   - Condition: `$json.priority === 'Critical'`  
   - Connect from Filter Equipment with Alerts node  

8. **Create Email Send Node for Supervisors**  
   - Name: Send Critical Alert to Supervisors  
   - Type: Email Send  
   - Configure same SMTP credentials  
   - To: Fixed emails (e.g., supervisor@hospital.com, maintenance.manager@hospital.com)  
   - From: Your email address  
   - Subject: Include critical alert emoji and equipment name  
   - Connect from Filter Critical Equipment node  

9. **Create WhatsApp Node for Critical Alerts**  
   - Name: Send Critical Alert Massage  
   - Type: WhatsApp  
   - Configure WhatsApp API credentials  
   - Recipient: Fixed supervisor phone number (e.g., +919999887776)  
   - Message body: Include critical alert with equipment name and alert emoji  
   - Connect from Filter Critical Equipment node  

10. **Create WhatsApp Node for Technician Alerts**  
    - Name: Send message  
    - Type: WhatsApp  
    - Configure WhatsApp API credentials  
    - Recipient: Technician WhatsApp number from alert data (`$json.technicianWhatsApp`)  
    - Message body: Alert message including equipment name and alert emoji  
    - Connect from Filter Equipment with Alerts node  

11. **Create Google Sheets Write Node for Logging Alerts**  
    - Name: Log Maintenance Alerts  
    - Type: Google Sheets  
    - Operation: Append Rows  
    - Document ID: Same as equipment sheet  
    - Sheet Name: Second sheet (`gid=1`) or named “Alert_Log”  
    - Authentication: Service Account credentials  
    - Connect from Filter Equipment with Alerts node  

12. **Add Filter Node for Overdue Maintenance/Calibration**  
    - Name: Filter Overdue Equipment  
    - Type: Filter  
    - Condition: `$json.alertTypes.includes('OVERDUE_MAINTENANCE') || $json.alertTypes.includes('OVERDUE_CALIBRATION')`  
    - Connect from Filter Equipment with Alerts node  

13. **Add Google Sheets Write Node for Updating Equipment Status**  
    - Name: Update Equipment Status  
    - Type: Google Sheets  
    - Operation: Update Rows (configure to update correct row based on Equipment ID)  
    - Document ID: Same as equipment sheet  
    - Sheet Name: First sheet (`gid=0`)  
    - Authentication: Service Account credentials  
    - Connect from Filter Overdue Equipment node  

14. **Connect Nodes in Execution Order:**  
    - Daily Equipment Check → Read Equipment Data → Process Equipment Alerts → Tack Break For 5 Sec → Filter Equipment with Alerts  
    - Filter Equipment with Alerts → Send Technician Email  
    - Filter Equipment with Alerts → Filter Critical Equipment → Send Critical Alert to Supervisors  
    - Filter Critical Equipment → Send Critical Alert Massage (WhatsApp)  
    - Filter Equipment with Alerts → Send message (WhatsApp)  
    - Filter Equipment with Alerts → Log Maintenance Alerts  
    - Filter Equipment with Alerts → Filter Overdue Equipment → Update Equipment Status  

15. **Validate and Activate Workflow**  
    - Check all credentials and document IDs are correct and have appropriate permissions  
    - Test with sample data to ensure alerts generate and notifications send correctly  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow includes a sticky note summarizing node roles and overall logic for quick reference.                                                   | Sticky Note node content within the workflow                                                        |
| Replace placeholders (`YOUR_GOOGLE_SHEET_ID`, email addresses, phone numbers) with real deployment values.                                      | Critical for operational correctness                                                                |
| SMTP and WhatsApp API credentials must be properly configured and tested prior to workflow activation.                                         | n8n credential setup documentation                                                                  |
| The Google Sheets document should have two sheets: one for equipment data (`gid=0`) and one for alert logs (`gid=1`).                          | Sheet structure requirement                                                                          |
| Alert priority order: Critical > High > Medium > Normal, used for sorting and escalation logic.                                                  | Code node logic explanation                                                                          |
| The workflow uses a service account for Google Sheets API authentication, which requires appropriate sharing of the sheet with the service email.| Google Cloud Platform - Service Account setup guide                                                 |
| The JavaScript code node assumes date fields are in parseable formats; ensure consistent date formatting in Google Sheets.                      | Date field consistency is essential to prevent parsing errors                                       |
| For WhatsApp messages, the recipient phone numbers must be in international format (+countrycode number).                                        | WhatsApp Business API requirements                                                                  |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, a tool for integration and automation. The processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.