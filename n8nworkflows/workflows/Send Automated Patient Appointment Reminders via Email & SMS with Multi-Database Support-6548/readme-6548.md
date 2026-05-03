Send Automated Patient Appointment Reminders via Email & SMS with Multi-Database Support

https://n8nworkflows.xyz/workflows/send-automated-patient-appointment-reminders-via-email---sms-with-multi-database-support-6548


# Send Automated Patient Appointment Reminders via Email & SMS with Multi-Database Support

### 1. Workflow Overview

This workflow automates patient appointment reminders via email and SMS, supporting multiple database backends (Google Sheets, Airtable, PostgreSQL) for storage and tracking. It is designed for healthcare providers or appointment-based services to reliably notify patients 3 days and 1 day before their scheduled appointments. The workflow logically separates into these main blocks:

- **1.1 Input Reception:** Receives new appointment data via a webhook.
- **1.2 Data Extraction & Validation:** Extracts, formats, and validates appointment details.
- **1.3 Multi-Database Storage:** Stores appointment data in one or more connected databases (Google Sheets, Airtable, PostgreSQL).
- **1.4 Reminder Scheduling & Dispatch:** Waits until 3 days and 1 day before appointments to send email and SMS reminders.
- **1.5 Status Updates:** Updates the databases to mark reminders as sent.
- **1.6 Documentation & Setup Notes:** Provides sticky notes with setup instructions and system overview.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures new appointment data securely via a webhook endpoint, triggering the workflow when a new appointment is created in an external system.

- **Nodes Involved:**  
  - Webhook: New Appointment

- **Node Details:**  
  - **Webhook: New Appointment**  
    - *Type:* Webhook  
    - *Role:* Entry point to receive appointment JSON payloads via HTTP POST.  
    - *Configuration:* Default webhook settings with a unique webhookId; no special parameters.  
    - *Inputs:* External HTTP request (JSON appointment data).  
    - *Outputs:* Passes raw data to the next node "Extract Appointment Data".  
    - *Edge Cases:* Missing or malformed payloads; webhook timeout; unauthorized requests if security not configured externally.  
    - *Version:* v1 (stable webhook node).  

#### 1.2 Data Extraction & Validation

- **Overview:**  
  Extracts required appointment fields from incoming data and performs validation and formatting to standardize the dataset for downstream tasks.

- **Nodes Involved:**  
  - Extract Appointment Data  
  - Format & Validate Data

- **Node Details:**  
  - **Extract Appointment Data**  
    - *Type:* Set  
    - *Role:* Maps and extracts specific fields (e.g., patient name, appointment date/time, contact info) from the webhook payload.  
    - *Configuration:* Defined key-value pairs to isolate relevant data fields; may include renaming or initial data shaping.  
    - *Inputs:* Raw webhook data.  
    - *Outputs:* Structured appointment data to "Format & Validate Data".  
    - *Edge Cases:* Missing fields, unexpected data types.  

  - **Format & Validate Data**  
    - *Type:* Code (JavaScript)  
    - *Role:* Runs custom scripts to validate field formats (dates, phone numbers, emails), corrects or rejects invalid data, and enriches info if needed.  
    - *Configuration:* Custom JS code block validating data integrity and formatting.  
    - *Inputs:* Extracted appointment data.  
    - *Outputs:* Validated and formatted data to storage nodes.  
    - *Edge Cases:* Validation failures causing workflow errors or skips; date parsing issues; invalid contact info.  

#### 1.3 Multi-Database Storage

- **Overview:**  
  Stores the validated appointment data into one or more databases depending on client configuration: Google Sheets, Airtable, and PostgreSQL supported in parallel.

- **Nodes Involved:**  
  - Store in Google Sheets  
  - Store in Airtable  
  - Store in PostgreSQL

- **Node Details:**  
  - **Store in Google Sheets**  
    - *Type:* Google Sheets  
    - *Role:* Inserts new appointment row into a configured Google Sheets spreadsheet.  
    - *Configuration:* Spreadsheet ID, sheet name, and mapped columns for appointment fields. Uses OAuth2 credentials.  
    - *Inputs:* Validated appointment data.  
    - *Outputs:* Triggers downstream "Wait: 3 Days Before".  
    - *Edge Cases:* API quota limits, auth token expiration, invalid spreadsheet IDs.  

  - **Store in Airtable**  
    - *Type:* Airtable  
    - *Role:* Creates new record in an Airtable base/table for appointment data.  
    - *Configuration:* Base ID, table name, field mappings; Airtable API key credentials.  
    - *Inputs:* Validated appointment data.  
    - *Outputs:* Triggers downstream "Wait: 3 Days Before".  
    - *Edge Cases:* API rate limits, base/table misconfiguration, bad API keys.  

  - **Store in PostgreSQL**  
    - *Type:* PostgreSQL  
    - *Role:* Inserts appointment data as a new row in a PostgreSQL database table.  
    - *Configuration:* Connection credentials (host, port, user, password, database), SQL insert query with parameters.  
    - *Inputs:* Validated appointment data.  
    - *Outputs:* Triggers downstream "Wait: 3 Days Before".  
    - *Edge Cases:* DB connection failures, SQL syntax errors, data type mismatches.  

#### 1.4 Reminder Scheduling & Dispatch

- **Overview:**  
  Waits until 3 days and 1 day before the appointment date/time, then sends email and SMS reminders to the patient.

- **Nodes Involved:**  
  - Wait: 3 Days Before  
  - Send 3-Day Email Reminder  
  - Send 3-Day SMS Reminder  
  - Wait: 1 Day Before  
  - Send 1-Day Email Reminder  
  - Send 1-Day SMS Reminder

- **Node Details:**  
  - **Wait: 3 Days Before**  
    - *Type:* Wait  
    - *Role:* Pauses workflow until exactly 3 days before appointment time. Uses appointment date/time fields as wait-until parameters.  
    - *Inputs:* Triggered after storing appointment data.  
    - *Outputs:* Sends parallel outputs to send 3-day reminders.  
    - *Edge Cases:* Incorrect appointment date format; daylight savings/timezone issues; workflow stop/restart causing missed waits.  

  - **Send 3-Day Email Reminder**  
    - *Type:* Email Send  
    - *Role:* Sends an email reminder to patient 3 days before appointment.  
    - *Configuration:* SMTP or email provider configured; email template with placeholders for patient name, appointment info.  
    - *Inputs:* From wait node.  
    - *Outputs:* Triggers database updates marking 3-day reminder sent.  
    - *Edge Cases:* SMTP failures, invalid email addresses, template rendering errors.  

  - **Send 3-Day SMS Reminder**  
    - *Type:* Twilio  
    - *Role:* Sends SMS reminder 3 days before appointment using Twilio API.  
    - *Configuration:* Twilio account SID, auth token; sender phone number; SMS template with variables.  
    - *Inputs:* From wait node.  
    - *Outputs:* Triggers database updates marking 3-day reminder sent.  
    - *Edge Cases:* Invalid phone number formats, Twilio API limits, message failures.  

  - **Wait: 1 Day Before**  
    - *Type:* Wait  
    - *Role:* Pauses workflow until 1 day before appointment time. Triggered after database updates from 3-day reminder nodes.  
    - *Inputs:* After 3-day reminder updates.  
    - *Outputs:* Triggers parallel sending of 1-day reminders.  
    - *Edge Cases:* Same as 3-day wait node; timing accuracy critical.  

  - **Send 1-Day Email Reminder**  
    - *Type:* Email Send  
    - *Role:* Sends 1-day-before email reminder with a similar template to the 3-day email.  
    - *Inputs:* From 1-day wait.  
    - *Outputs:* Triggers database updates marking 1-day reminder sent.  
    - *Edge Cases:* Same as 3-day email node.  

  - **Send 1-Day SMS Reminder**  
    - *Type:* Twilio  
    - *Role:* Sends 1-day-before SMS reminder via Twilio.  
    - *Inputs:* From 1-day wait.  
    - *Outputs:* Triggers database updates marking 1-day reminder sent.  
    - *Edge Cases:* Same as 3-day SMS node.  

#### 1.5 Status Updates

- **Overview:**  
  Updates appointment records in all configured databases to flag when 3-day and 1-day reminders have been sent, enabling tracking and preventing duplicate notifications.

- **Nodes Involved:**  
  - Update Google Sheets: 3-Day Sent  
  - Update Airtable: 3-Day Sent  
  - Update PostgreSQL: 3-Day Sent  
  - Update Google Sheets: 1-Day Sent  
  - Update Airtable: 1-Day Sent  
  - Update PostgreSQL: 1-Day Sent

- **Node Details:**  
  Each update node:  
  - *Type:* Corresponding database node (Google Sheets, Airtable, PostgreSQL) in update mode.  
  - *Role:* Modifies existing appointment record to set a "3-Day Reminder Sent" or "1-Day Reminder Sent" flag/column to true or timestamp.  
  - *Configuration:* Uses appointment ID or unique key to find the correct record.  
  - *Inputs:* From respective Send Reminder nodes.  
  - *Outputs:* After 3-day updates, triggers the 1-day wait node; after 1-day updates, workflow ends.  
  - *Edge Cases:* Record not found errors, concurrency conflicts, partial update failures.  

#### 1.6 Documentation & Setup Notes

- **Overview:**  
  Provides non-executable sticky notes with setup instructions, system overview, and configuration guidance for different database options and client onboarding.

- **Nodes Involved:**  
  - System Overview  
  - Database Selection Guide  
  - Google Sheets Setup  
  - Airtable Setup  
  - Client Setup Checklist

- **Node Details:**  
  - *Type:* Sticky Note  
  - *Role:* Descriptive text blocks placed for human operators to assist with deployment and maintenance.  
  - *Content Highlights:* May include steps to configure OAuth2 for Google Sheets, Airtable API keys, PostgreSQL connection strings, and general usage instructions.  
  - *Inputs/Outputs:* None.  
  - *Edge Cases:* None; purely informational.  

---

### 3. Summary Table

| Node Name                    | Node Type       | Functional Role                           | Input Node(s)               | Output Node(s)                          | Sticky Note                                     |
|------------------------------|-----------------|-----------------------------------------|-----------------------------|----------------------------------------|-------------------------------------------------|
| Webhook: New Appointment      | Webhook         | Entry point to receive new appointment data | None                        | Extract Appointment Data                |                                                 |
| Extract Appointment Data      | Set             | Extracts relevant appointment fields     | Webhook: New Appointment     | Format & Validate Data                  |                                                 |
| Format & Validate Data        | Code            | Validates and formats extracted data     | Extract Appointment Data     | Store in Google Sheets, Airtable, PostgreSQL |                                                 |
| Store in Google Sheets        | Google Sheets   | Stores appointment data in Google Sheets | Format & Validate Data       | Wait: 3 Days Before                     | See "Google Sheets Setup" sticky note            |
| Store in Airtable             | Airtable        | Stores appointment data in Airtable      | Format & Validate Data       | Wait: 3 Days Before                     | See "Airtable Setup" sticky note                  |
| Store in PostgreSQL           | PostgreSQL      | Stores appointment data in PostgreSQL    | Format & Validate Data       | Wait: 3 Days Before                     |                                                 |
| Wait: 3 Days Before           | Wait            | Waits until 3 days before appointment    | Store in Google Sheets, Airtable, PostgreSQL | Send 3-Day Email Reminder, Send 3-Day SMS Reminder |                                                 |
| Send 3-Day Email Reminder     | Email Send      | Sends email reminder 3 days before       | Wait: 3 Days Before          | Update Google Sheets: 3-Day Sent, Update Airtable: 3-Day Sent, Update PostgreSQL: 3-Day Sent |                                                 |
| Send 3-Day SMS Reminder       | Twilio          | Sends SMS reminder 3 days before         | Wait: 3 Days Before          | Update Google Sheets: 3-Day Sent, Update Airtable: 3-Day Sent, Update PostgreSQL: 3-Day Sent |                                                 |
| Update Google Sheets: 3-Day Sent | Google Sheets | Updates 3-Day reminder sent flag          | Send 3-Day Email Reminder, Send 3-Day SMS Reminder | Wait: 1 Day Before                     |                                                 |
| Update Airtable: 3-Day Sent   | Airtable        | Updates 3-Day reminder sent flag          | Send 3-Day Email Reminder, Send 3-Day SMS Reminder | Wait: 1 Day Before                     |                                                 |
| Update PostgreSQL: 3-Day Sent | PostgreSQL      | Updates 3-Day reminder sent flag          | Send 3-Day Email Reminder, Send 3-Day SMS Reminder | Wait: 1 Day Before                     |                                                 |
| Wait: 1 Day Before            | Wait            | Waits until 1 day before appointment      | Update Google Sheets: 3-Day Sent, Update Airtable: 3-Day Sent, Update PostgreSQL: 3-Day Sent | Send 1-Day Email Reminder, Send 1-Day SMS Reminder |                                                 |
| Send 1-Day Email Reminder     | Email Send      | Sends email reminder 1 day before         | Wait: 1 Day Before           | Update Google Sheets: 1-Day Sent, Update Airtable: 1-Day Sent, Update PostgreSQL: 1-Day Sent |                                                 |
| Send 1-Day SMS Reminder       | Twilio          | Sends SMS reminder 1 day before           | Wait: 1 Day Before           | Update Google Sheets: 1-Day Sent, Update Airtable: 1-Day Sent, Update PostgreSQL: 1-Day Sent |                                                 |
| Update Google Sheets: 1-Day Sent | Google Sheets | Updates 1-Day reminder sent flag          | Send 1-Day Email Reminder, Send 1-Day SMS Reminder | None                                    |                                                 |
| Update Airtable: 1-Day Sent   | Airtable        | Updates 1-Day reminder sent flag          | Send 1-Day Email Reminder, Send 1-Day SMS Reminder | None                                    |                                                 |
| Update PostgreSQL: 1-Day Sent | PostgreSQL      | Updates 1-Day reminder sent flag          | Send 1-Day Email Reminder, Send 1-Day SMS Reminder | None                                    |                                                 |
| System Overview              | Sticky Note     | Provides overview and notes about system  | None                        | None                                   |                                                 |
| Database Selection Guide      | Sticky Note     | Advises on choosing database backend      | None                        | None                                   |                                                 |
| Google Sheets Setup           | Sticky Note     | Instructions for Google Sheets configuration | None                        | None                                   |                                                 |
| Airtable Setup               | Sticky Note     | Instructions for Airtable configuration   | None                        | None                                   |                                                 |
| Client Setup Checklist        | Sticky Note     | Checklist for client onboarding           | None                        | None                                   |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node ("Webhook: New Appointment")**  
   - Type: Webhook  
   - Purpose: Receive appointment data via HTTP POST.  
   - Configure with a unique webhook URL. No authentication specified but recommend securing externally.  

2. **Create Set Node ("Extract Appointment Data")**  
   - Type: Set  
   - Configuration: Map incoming webhook JSON fields to standardized fields: patient name, appointment datetime, email, phone, appointment ID.  
   - Connect Webhook -> Extract Appointment Data.  

3. **Create Code Node ("Format & Validate Data")**  
   - Type: Code (JavaScript)  
   - Configuration:  
     - Parse and validate appointment date format.  
     - Validate email format with regex.  
     - Validate phone number format (e.g., E.164).  
     - Return error or stop workflow if validation fails.  
   - Connect Extract Appointment Data -> Format & Validate Data.  

4. **Create Storage Nodes in Parallel**  
   - **Google Sheets Node ("Store in Google Sheets")**  
     - Type: Google Sheets  
     - Configure OAuth2 credentials.  
     - Set spreadsheet ID and target sheet.  
     - Map fields to columns.  
   - **Airtable Node ("Store in Airtable")**  
     - Type: Airtable  
     - Configure API key credentials.  
     - Set base ID and table name.  
     - Map fields accordingly.  
   - **PostgreSQL Node ("Store in PostgreSQL")**  
     - Type: PostgreSQL  
     - Configure DB connection credentials.  
     - Use SQL INSERT statement with parameters.  
   - Connect Format & Validate Data to all three storage nodes in parallel.  

5. **Create Wait Node ("Wait: 3 Days Before")**  
   - Type: Wait  
   - Configure:  
     - Wait until timestamp = appointment date/time minus 3 days.  
   - Connect outputs of all three storage nodes to this Wait node (parallel triggers allowed).  

6. **Create Reminder Send Nodes for 3-Day Reminders**  
   - **Email Send Node ("Send 3-Day Email Reminder")**  
     - Configure SMTP or email provider credentials.  
     - Template email body with variables (patient name, appointment date/time).  
   - **Twilio Node ("Send 3-Day SMS Reminder")**  
     - Configure Twilio account SID, auth token, and sender number.  
     - Template SMS text with variables.  
   - Connect Wait: 3 Days Before -> Send 3-Day Email Reminder and Send 3-Day SMS Reminder in parallel.  

7. **Create Database Update Nodes for 3-Day Reminder Sent Flags**  
   - For each database (Google Sheets, Airtable, PostgreSQL), create an update node:  
     - Google Sheets: Update row to set "3-Day Reminder Sent" column.  
     - Airtable: Update record field for 3-day sent flag.  
     - PostgreSQL: Update row with 3-day reminder flag.  
   - Connect Send 3-Day Email Reminder and Send 3-Day SMS Reminder nodes to all three update nodes in parallel.  

8. **Create Wait Node ("Wait: 1 Day Before")**  
   - Type: Wait  
   - Configure: Wait until appointment date/time minus 1 day.  
   - Connect all three 3-day update nodes in parallel to this wait node.  

9. **Create Reminder Send Nodes for 1-Day Reminders**  
   - **Email Send Node ("Send 1-Day Email Reminder")**  
     - Configure SMTP/email provider.  
     - Template similar to 3-day email but indicating 1 day remaining.  
   - **Twilio Node ("Send 1-Day SMS Reminder")**  
     - Configure Twilio credentials.  
     - SMS template with 1-day reminder text.  
   - Connect Wait: 1 Day Before -> Send 1-Day Email Reminder and Send 1-Day SMS Reminder in parallel.  

10. **Create Database Update Nodes for 1-Day Reminder Sent Flags**  
    - Create update nodes for all three databases to mark 1-day reminder sent.  
    - Connect Send 1-Day Email Reminder and Send 1-Day SMS Reminder to these update nodes.  

11. **Add Sticky Notes for Documentation**  
    - Add "System Overview", "Database Selection Guide", "Google Sheets Setup", "Airtable Setup", and "Client Setup Checklist" sticky notes with relevant setup instructions and notes for operators.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                           |
|------------------------------------------------------------------------------------------------------------------------------|------------------------------------------|
| Ensure OAuth2 credentials are correctly configured for Google Sheets to avoid auth errors.                                   | Google Sheets Setup sticky note          |
| Airtable API key must have write access to the relevant base and table.                                                      | Airtable Setup sticky note                |
| Twilio account requires verified phone numbers and sufficient balance to send SMS.                                           | Twilio node configuration                 |
| Timezone consistency is critical for wait nodes to trigger reminders at correct local times.                                 | General workflow timing                   |
| Use error workflows or retry mechanisms in production to handle transient API failures or network issues.                    | Best practice for n8n production          |
| Workflow designed to be extensible for additional databases or notification channels by duplicating storage and reminder nodes. | Scalability and customization note       |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.