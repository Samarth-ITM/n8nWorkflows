Send Multi-Channel Medication Reminders to Patients with Google Sheets & WhatsApp

https://n8nworkflows.xyz/workflows/send-multi-channel-medication-reminders-to-patients-with-google-sheets---whatsapp-7326


# Send Multi-Channel Medication Reminders to Patients with Google Sheets & WhatsApp

### 1. Workflow Overview

This workflow automates the process of sending multi-channel medication reminders to patients by integrating Google Sheets and WhatsApp, along with email notifications. It targets healthcare providers who manage patient prescriptions and wish to ensure patients take their medications on time via scheduled reminders.

The workflow is logically divided into two main blocks:

- **1.1 Schedule Creation:** Watches for newly sent prescriptions in a Google Sheet, filters unscheduled ones, generates a detailed medication reminder schedule based on prescription data, saves these reminders back to Google Sheets, and marks prescriptions as processed to avoid duplication.

- **1.2 Reminder Sending:** Runs every 10 minutes to fetch all scheduled reminders, identifies due reminders based on current time, sends reminders via WhatsApp and email simultaneously, and marks those reminders as sent to prevent repeated notifications.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Creation

**Overview:**  
This block monitors prescription data in a Google Sheet, filters for prescriptions that have been sent but have no reminder schedule yet, generates a timeline of medication reminders based on dosage frequency and duration, stores these reminders in a Google Sheet, and finally marks the prescriptions as processed.

**Nodes Involved:**  
- Watch Sheet For Trigger  
- New Sent Prescriptions (Filter)  
- Create Reminder Schedule (Code)  
- Save Reminders (Google Sheets)  
- Mark as Processed (Google Sheets)  
- Sticky Note (explanatory)

**Node Details:**

- **Watch Sheet For Trigger**  
  - Type: Google Sheets Trigger  
  - Role: Listens for updates (rowUpdate event) on the "Prescriptions" sheet to detect new or updated prescription entries.  
  - Config: Polls every hour; uses OAuth2 credentials for Google Sheets access.  
  - Inputs: None (trigger node)  
  - Outputs: Row data on prescription updates.  
  - Edge cases: OAuth token expiry, permission issues, network downtime.  

- **New Sent Prescriptions**  
  - Type: Filter  
  - Role: Filters rows where `prescription_status` equals "sent" AND `reminders_created` is not "yes" (case-sensitive).  
  - Key Expressions:  
    - Left: `{{$json.prescription_status}}` equals "sent"  
    - Left: `{{$json.reminders_created}}` not equals "yes"  
  - Inputs: Output of Google Sheets Trigger  
  - Outputs: Filtered prescriptions requiring reminder scheduling.  
  - Edge cases: Missing fields, case sensitivity causing mismatches.  

- **Create Reminder Schedule**  
  - Type: Code (JavaScript)  
  - Role: Automatically generates a schedule of reminder timestamps using prescription details such as start date, frequency per day, and duration in days.  
  - Logic Highlights:  
    - Defaults: 3 times daily, 7 days duration if not specified.  
    - Time slots predefined for 1 to 4 times daily (e.g., 08:00, 14:00, 20:00 for 3 times).  
    - For each day and time slot, creates reminder objects with detailed info.  
    - Only future reminders (after current time) are created.  
  - Inputs: Filtered prescription data  
  - Outputs: Enhanced JSON with a `reminders` array and `total_reminders`.  
  - Edge cases: Invalid/missing date formats, times per day outside 1-4, timezone issues.  

- **Save Reminders**  
  - Type: Google Sheets (Append or Update)  
  - Role: Saves the generated reminders into the "Reminders" sheet in a Google Sheets document identified by ID.  
  - Config: Uses Service Account authentication, auto-maps input data to columns.  
  - Inputs: Output of Create Reminder Schedule node.  
  - Outputs: Confirmation of sheet append/update operation.  
  - Edge cases: Google Sheets API quota exceeded, invalid document ID, network errors.  

- **Mark as Processed**  
  - Type: Google Sheets (Update)  
  - Role: Updates the corresponding prescription row in the "Prescriptions" sheet to mark `reminders_created` as "yes" to prevent reprocessing.  
  - Matching Column: `prescription_id` ensures correct row update.  
  - Inputs: Output of Save Reminders node.  
  - Outputs: Confirmation of update.  
  - Edge cases: Prescription ID mismatch, update conflicts, API errors.  

- **Sticky Note**  
  - Content:  
    ```
    ### **Part 1: Schedule Creation**
    1. **Watch Sheet** → Monitors for "sent" prescriptions
    2. **Filter New** → Only processes unscheduled prescriptions  
    3. **Create Schedule** → Generates reminder times automatically
    4. **Save Reminders** → Stores schedule in sheet
    5. **Mark Processed** → Prevents duplicate scheduling
    ```
  - Role: Documentation within the workflow for clarity.

---

#### 2.2 Reminder Sending

**Overview:**  
This block runs on a 10-minute interval, fetches all scheduled reminders, identifies which ones are currently due (within a 10-minute buffer), sends reminders via WhatsApp and email simultaneously, and marks reminders as sent to avoid repeat notifications.

**Nodes Involved:**  
- Check Every 10 Minutes (Cron)  
- Get All Reminders (Google Sheets)  
- Find Due Reminders (Code)  
- Send WhatsApp  
- Send email  
- Mark as Sent (Google Sheets)  
- Sticky Note1 (explanatory)

**Node Details:**

- **Check Every 10 Minutes**  
  - Type: Cron  
  - Role: Triggers the reminder sending process every 10 minutes.  
  - Config: Default cron without extra parameters; triggers main flow.  
  - Inputs: None (trigger)  
  - Outputs: Triggers Get All Reminders.  
  - Edge cases: Cron timing drift, execution overlap if processing delays.  

- **Get All Reminders**  
  - Type: Google Sheets (Read)  
  - Role: Retrieves all rows from the "Reminders" sheet to assess which reminders are due.  
  - Config: Service Account authentication; uses document ID for the reminders Google Sheet.  
  - Inputs: Trigger from Cron node.  
  - Outputs: Array of reminders with details and statuses.  
  - Edge cases: Large volume leading to API limits; incomplete data rows.  

- **Find Due Reminders**  
  - Type: Code (JavaScript)  
  - Role: Filters the reminders to those whose `status` is "scheduled" and `reminder_time` falls within the current time and the next 10 minutes (buffer).  
  - Logic Highlights:  
    - Computes time difference between now and reminder time in minutes.  
    - Constructs a personalized message including patient name, medication, dosage, time slot, and day number.  
  - Inputs: List of all reminders  
  - Outputs: List of reminders due now with a formatted message field.  
  - Edge cases: Timezone mismatches, wrongly formatted dates, missing status or fields.  

- **Send WhatsApp**  
  - Type: WhatsApp node (API integration)  
  - Role: Sends medication reminder messages to the patient's WhatsApp number.  
  - Config:  
    - Uses WhatsApp API credentials.  
    - Message body templated dynamically (Note: JSON uses placeholders in the example, but actual message uses code-generated `message` field).  
    - Phone numbers are derived from input JSON fields.  
  - Inputs: Due reminders from code node.  
  - Outputs: Confirmation of message sent.  
  - Edge cases: Invalid phone numbers, WhatsApp API limits, connectivity issues, message formatting errors.  

- **Send email**  
  - Type: Email Send  
  - Role: Sends email reminders to patients as an alternative notification channel.  
  - Config:  
    - Uses SMTP credentials.  
    - Email subject: "💊 Medication Reminder"  
    - To address: patient email from JSON.  
    - From address: reminders@hospital.com  
    - Email body: personalized medication reminder text.  
  - Inputs: Same due reminders as WhatsApp node.  
  - Outputs: Confirmation of email sent.  
  - Edge cases: Invalid email addresses, SMTP authentication failures, spam filtering.  

- **Mark as Sent**  
  - Type: Google Sheets (Update)  
  - Role: Updates the reminders in the "Reminders" sheet, setting status to indicate the reminder was sent.  
  - Matching Columns: `prescription_id` and `reminder_time` to identify each reminder uniquely.  
  - Inputs: Outputs from both messaging nodes (WhatsApp and email).  
  - Outputs: Confirmation of update to prevent duplicate sends.  
  - Edge cases: Concurrency issues if multiple reminders processed simultaneously, update conflicts.  

- **Sticky Note1**  
  - Content:  
    ```
    ### **Part 2: Send Reminders** 
    6. **Cron Timer** → Checks every 10 minutes
    7. **Get Reminders** → Retrieves all scheduled reminders
    8. **Find Due** → Identifies reminders due now
    9. **Send Messages** → WhatsApp + Email simultaneously
    10. **Mark Sent** → Updates status to prevent duplicates
    ```
  - Role: Documentation within the workflow for clarity.

---

### 3. Summary Table

| Node Name                | Node Type                | Functional Role                          | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                     |
|--------------------------|--------------------------|----------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Watch Sheet For Trigger   | Google Sheets Trigger    | Triggers on prescription sheet updates| None                        | New Sent Prescriptions       | Part 1: Schedule Creation step 1                                                                |
| New Sent Prescriptions    | Filter                   | Filters prescriptions with status "sent" and no reminders created | Watch Sheet For Trigger     | Create Reminder Schedule      | Part 1: Schedule Creation step 2                                                                |
| Create Reminder Schedule  | Code                     | Generates reminder schedule timestamps | New Sent Prescriptions       | Save Reminders               | Part 1: Schedule Creation step 3                                                                |
| Save Reminders            | Google Sheets            | Saves generated reminders in sheet    | Create Reminder Schedule     | Mark as Processed            | Part 1: Schedule Creation step 4                                                                |
| Mark as Processed         | Google Sheets            | Marks prescriptions as processed       | Save Reminders               |                             | Part 1: Schedule Creation step 5                                                                |
| Check Every 10 Minutes    | Cron                     | Periodic trigger every 10 minutes      | None                        | Get All Reminders            | Part 2: Send Reminders step 6                                                                    |
| Get All Reminders         | Google Sheets            | Retrieves all reminders from sheet     | Check Every 10 Minutes       | Find Due Reminders           | Part 2: Send Reminders step 7                                                                    |
| Find Due Reminders        | Code                     | Filters reminders due to send now      | Get All Reminders            | Send WhatsApp, Send email    | Part 2: Send Reminders step 8                                                                    |
| Send WhatsApp             | WhatsApp                 | Sends reminder message via WhatsApp    | Find Due Reminders           | Mark as Sent                | Part 2: Send Reminders step 9                                                                    |
| Send email               | Email Send                | Sends reminder email                    | Find Due Reminders           | Mark as Sent                | Part 2: Send Reminders step 9                                                                    |
| Mark as Sent             | Google Sheets            | Marks reminders as sent                 | Send WhatsApp, Send email    |                             | Part 2: Send Reminders step 10                                                                   |
| Sticky Note              | Sticky Note              | Documentation for schedule creation    | None                        | None                        | See Part 1 content above                                                                          |
| Sticky Note1             | Sticky Note              | Documentation for reminder sending     | None                        | None                        | See Part 2 content above                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node ("Watch Sheet For Trigger")**  
   - Node Type: Google Sheets Trigger  
   - Event: `rowUpdate` on the "Prescriptions" sheet  
   - Polling Interval: Every hour  
   - Authentication: OAuth2 credentials for Google Sheets  
   - Document ID and Sheet Name: Set IDs according to your Google Sheet containing prescriptions  

2. **Add Filter Node ("New Sent Prescriptions")**  
   - Node Type: Filter  
   - Conditions:  
     - `$json.prescription_status` equals `"sent"` (case sensitive)  
     - `$json.reminders_created` not equals `"yes"` (case sensitive)  
   - Connect input from "Watch Sheet For Trigger"  

3. **Add Code Node ("Create Reminder Schedule")**  
   - Node Type: Code (JavaScript)  
   - Paste the provided code that:  
     - Parses `times_per_day`, `duration_days`, `start_date`  
     - Uses predefined time slots for reminders  
     - Generates future reminder objects with fields including `prescription_id`, `patient_name`, `patient_phone`, `patient_email`, `medication`, `dosage`, `reminder_time`, `day_number`, `time_slot`, and `status: 'scheduled'`  
   - Connect input from "New Sent Prescriptions"  

4. **Add Google Sheets Node ("Save Reminders")**  
   - Node Type: Google Sheets (Append or Update)  
   - Operation: Append or Update  
   - Sheet Name: `"Reminders"`  
   - Document ID: Your Google Sheet ID where reminders are stored  
   - Authentication: Service Account credentials for Google Sheets API  
   - Input Mapping: Auto-map the reminder fields  
   - Connect input from "Create Reminder Schedule"  

5. **Add Google Sheets Node ("Mark as Processed")**  
   - Node Type: Google Sheets (Update)  
   - Operation: Update  
   - Sheet Name: `"Prescriptions"`  
   - Document ID: Same as the prescriptions sheet  
   - Authentication: Service Account credentials  
   - Matching Column: `"prescription_id"`  
   - Update Field: Set `reminders_created` to `"yes"`  
   - Connect input from "Save Reminders"  

6. **Add Cron Node ("Check Every 10 Minutes")**  
   - Node Type: Cron  
   - Schedule: Every 10 minutes  
   - No inputs (trigger node)  

7. **Add Google Sheets Node ("Get All Reminders")**  
   - Node Type: Google Sheets (Read)  
   - Operation: Read all rows from `"Reminders"` sheet  
   - Document ID: Your reminders sheet ID  
   - Authentication: Service Account credentials  
   - Connect input from "Check Every 10 Minutes"  

8. **Add Code Node ("Find Due Reminders")**  
   - Node Type: Code (JavaScript)  
   - Paste provided code which:  
     - Filters reminders with status `"scheduled"`  
     - Checks if `reminder_time` is within past 10 minutes (buffer)  
     - Constructs a message string with medication details  
   - Connect input from "Get All Reminders"  

9. **Add WhatsApp Node ("Send WhatsApp")**  
   - Node Type: WhatsApp  
   - Operation: Send message  
   - Phone Number ID: Set your WhatsApp Business phone number ID  
   - Recipient Phone Number: Use expression to read from input JSON (`patient_phone`)  
   - Message Body: Use the `message` field created by the code node (dynamic content)  
   - Authentication: WhatsApp API credentials  
   - Connect input from "Find Due Reminders"  

10. **Add Email Send Node ("Send email")**  
    - Node Type: Email Send  
    - Subject: `"💊 Medication Reminder"`  
    - To: Expression using `$json.patient_email`  
    - From: `"reminders@hospital.com"` (or your domain)  
    - Email Body: Template similar to WhatsApp message, personalized with patient and medication info  
    - Authentication: SMTP credentials  
    - Connect input from "Find Due Reminders"  

11. **Add Google Sheets Node ("Mark as Sent")**  
    - Node Type: Google Sheets (Update)  
    - Operation: Update  
    - Sheet Name: `"Reminders"`  
    - Document ID: Reminders Google Sheet ID  
    - Authentication: Service Account credentials  
    - Matching Columns: `"prescription_id"` and `"reminder_time"`  
    - Update Field: Set status to `"sent"` or equivalent to mark reminders as delivered  
    - Connect inputs from both "Send WhatsApp" and "Send email" to this node (merge outputs)  

12. **Add Sticky Notes**  
    - Add two Sticky Note nodes to document:  
      - Part 1: Schedule Creation steps 1–5  
      - Part 2: Send Reminders steps 6–10  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                | Context or Link                                                       |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| This workflow uses Google Sheets as the central data store for prescriptions and reminders, enabling easy management and auditing.                                                                                                                                                                        | Core workflow design                                                 |
| WhatsApp messaging requires a valid WhatsApp Business API setup with appropriate phone number ID and permissions.                                                                                                                                                                                         | WhatsApp API integration                                              |
| Email sending requires SMTP credentials configured in n8n, with a verified sending domain to avoid spam filtering.                                                                                                                                                                                        | SMTP email configuration                                             |
| Reminder scheduling defaults to 3 times daily for 7 days if no specific frequency/duration provided. Time slots are predefined but can be customized in the code node.                                                                                                                                     | Scheduling logic details                                             |
| The workflow includes robust filtering to avoid duplicate reminders by marking processed prescriptions and sent reminders.                                                                                                                                                                               | Workflow integrity                                                  |
| Timezone considerations: Date and time comparisons assume consistent timezone handling; ensure your Google Sheets date fields and server time are aligned to prevent missed or early reminders.                                                                                                             | Important for time-sensitive workflows                              |
| For further help on Google Sheets integration and OAuth2 setup see: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googlesheets/                                                                                                                                                           | Official n8n docs                                                    |
| For WhatsApp node details and best practices refer to: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.whatsapp/                                                                                                                                                                           | Official n8n docs                                                    |
| For SMTP email node configuration see: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.email-send/                                                                                                                                                                                         | Official n8n docs                                                    |

---

This comprehensive analysis and reproduction guide provides all necessary details to understand, maintain, or replicate the medication reminder workflow using n8n, Google Sheets, WhatsApp, and email channels.