Automated Patient Vitals Monitoring & Alerts with Philips IntelliVue & Google Sheets

https://n8nworkflows.xyz/workflows/automated-patient-vitals-monitoring---alerts-with-philips-intellivue---google-sheets-7317


# Automated Patient Vitals Monitoring & Alerts with Philips IntelliVue & Google Sheets

### 1. Workflow Overview

This workflow automates the real-time monitoring of patient vital signs using Philips IntelliVue devices and integrates the data into a Google Sheets patient database. It periodically polls the IntelliVue Gateway API, processes incoming data from multiple potential formats (HL7 messages, CSV exports, or API responses), validates and enriches the vitals with clinical assessments, saves processed records into a Google Sheet, and selectively sends clinical alert emails to nursing staff and physicians based on alert severity.

The workflow is logically structured into these functional blocks:

- **1.1 Scheduled Data Polling:** Periodically triggers data retrieval every 30 seconds.
- **1.2 Data Fetching from IntelliVue Gateway:** Retrieves current patient vitals from the Philips IntelliVue Gateway API.
- **1.3 Data Processing & Parsing:** Handles multiple data formats (HL7, CSV, API JSON), extracting standardized vital signs.
- **1.4 Data Validation & Clinical Enrichment:** Validates key fields, computes derived metrics (e.g., pulse pressure, mean arterial pressure), and classifies clinical status and alert level.
- **1.5 Data Persistence:** Appends enriched data to a Google Sheets document serving as the patient vitals database.
- **1.6 Alerting:** Based on alert level, routes critical alerts to email notifications targeted to relevant clinical staff.
- **1.7 Control Flow:** Switch node directs alert-triggered actions.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Data Polling

- **Overview:**  
Triggers the workflow execution every 30 seconds to ensure real-time data updates.

- **Nodes Involved:**  
`Poll Device Data Every 30s`

- **Node Details:**  
  - Type: Cron Trigger  
  - Role: Initiates the workflow on a fixed schedule (default every 30s implied by configuration)  
  - Configuration: Default cron settings without custom time specified, interpreted as continuous 30s trigger  
  - Inputs: None (trigger node)  
  - Outputs: Connected to `Fetch from IntelliVue Gateway` node  
  - Edge Cases: Cron misconfiguration could cause no triggering or overly frequent calls; time zone considerations might apply.

#### 1.2 Data Fetching from IntelliVue Gateway

- **Overview:**  
Performs an HTTP GET request to the Philips IntelliVue Gateway API endpoint to retrieve current patient data.

- **Nodes Involved:**  
`Fetch from IntelliVue Gateway`

- **Node Details:**  
  - Type: HTTP Request  
  - Role: Queries the IntelliVue Gateway API for current patient vitals data  
  - Configuration:  
    - URL dynamically composed using environment variable `INTELLIVUE_GATEWAY_IP` at port 8080, endpoint `/api/patients/current`  
    - Timeout set to 5000ms (5 seconds)  
    - HTTP Basic Authentication with predefined credentials (named "test - auth")  
  - Input: Triggered by cron node  
  - Output: Raw JSON or other response forwarded to `Process Device Data` node  
  - Edge Cases:  
    - Network errors, timeout, or invalid credentials causing authentication failures  
    - API downtime or malformed responses  
    - Environment variable misspecification

#### 1.3 Data Processing & Parsing

- **Overview:**  
Processes the raw data from the IntelliVue Gateway. It detects the format (HL7 messages, CSV export files, or API JSON), parses the relevant vital signs, and standardizes them into a unified data structure.

- **Nodes Involved:**  
`Process Device Data`

- **Node Details:**  
  - Type: Code (JavaScript)  
  - Role: Custom parsing logic for multiple data formats and mapping to standardized vitals fields  
  - Configuration:  
    - Reads all inputs  
    - Parses HL7 messages by splitting segments and extracting PID and OBX sections to obtain patient info and vitals  
    - Parses CSV data by splitting lines and mapping headers to vitals fields  
    - Parses API JSON responses handling variable field names and structures  
    - Returns array of processed vitals objects, filtering out null/failed parses  
  - Key Expressions: Uses JavaScript functions `processHL7Message`, `processCSVExport`, and `processGatewayResponse` internally for parsing  
  - Inputs: Raw data from HTTP request node  
  - Outputs: Standardized JSON objects with patient vitals data forwarded to validation node  
  - Edge Cases:  
    - Malformed HL7 or CSV data causing parsing errors  
    - Missing fields or unexpected data structures in API response  
    - Parsing errors are caught and logged; invalid inputs discarded (null returned)  
  - Notes: Device type hardcoded as "Philips_IntelliVue" for all data sources

#### 1.4 Data Validation & Clinical Enrichment

- **Overview:**  
Validates patient ID presence, calculates derived metrics (pulse pressure, mean arterial pressure), converts temperature units, determines clinical status and alert level, and generates textual clinical alerts.

- **Nodes Involved:**  
`Validate & Enrich Data`

- **Node Details:**  
  - Type: Code (JavaScript)  
  - Role: Data validation, enrichment, and clinical decision logic  
  - Configuration:  
    - Checks `patient_id` is present and not 'UNKNOWN'; skips records without valid ID  
    - Calculates pulse pressure and mean arterial pressure from blood pressure readings  
    - Converts temperature from Celsius to Fahrenheit with rounding  
    - Determines clinical status ("NORMAL", "ABNORMAL") based on thresholds for heart rate, SpO₂, BP, temperature, and respiration rate  
    - Determines alert level ("NORMAL", "WARNING", "CRITICAL") with stricter thresholds for critical conditions  
    - Generates an array of clinical alert descriptions for abnormal or critical vitals  
    - Adds metadata timestamps and system status  
  - Inputs: Standardized vitals JSON from previous node  
  - Outputs: Enriched JSON forwarded to Google Sheets node  
  - Edge Cases:  
    - Missing or zero values for vitals may affect calculations  
    - Potential false positives or negatives in alert classification depending on threshold appropriateness  
    - Records without valid patient ID filtered out (returns null)  

#### 1.5 Data Persistence

- **Overview:**  
Appends the enriched vital signs data to a Google Sheets spreadsheet, maintaining a centralized patient vitals database.

- **Nodes Involved:**  
`Save to Patient Database`

- **Node Details:**  
  - Type: Google Sheets (Append)  
  - Role: Data persistence storing patient vitals in cloud spreadsheet  
  - Configuration:  
    - Operation: Append row in range `PatientVitals!A:Z`  
    - Sheet ID parameterized as `{{your_patient_vitals_sheet_id}}` (to be replaced with actual sheet ID)  
    - Authentication: Service Account credentials configured for Google API access  
  - Inputs: Enriched vitals JSON  
  - Outputs: Passes data forward to switch node for alert routing  
  - Edge Cases:  
    - Google API quota limits or authentication failures  
    - Invalid sheet ID or range causing append errors  
    - Network issues

#### 1.6 Alerting

- **Overview:**  
Sends email alerts based on the clinical alert level. Critical alerts trigger emails to nursing staff and on-call physicians.

- **Nodes Involved:**  
`Switch`, `Send Clinical Alert`

- **Node Details:**  

**Switch Node**  
  - Type: Switch  
  - Role: Routes flow based on alert level field in JSON  
  - Configuration:  
    - Routes only if `alert_level` equals "CRITICAL" (case-sensitive) to send alert  
    - Other levels, including "Normal", bypass alert sending  
  - Inputs: Output from Google Sheets node  
  - Outputs:  
    - Main output 0: to `Send Clinical Alert` (for critical alerts)  
    - Main output 1: no further action  

**Send Clinical Alert Node**  
  - Type: Email Send  
  - Role: Sends plaintext email notification with patient and vitals summary  
  - Configuration:  
    - Subject includes alert level dynamically  
    - Recipient emails: nursing staff email for patient’s room (e.g., nursing-101@hospital.com) and on-call physician  
    - Sender email: intellivue-alerts@hospital.com  
    - Email body includes patient ID, room, vitals (heart rate, SpO2, BP, temperature, respiration, EtCO2), alerts summary, mean arterial pressure, clinical status, and recommended actions  
    - SMTP credentials configured (“SMTP -test”)  
  - Inputs: Connected from switch node for critical alerts  
  - Outputs: None  
  - Edge Cases:  
    - SMTP failures or invalid recipient addresses  
    - Missing vitals data fields could cause incomplete emails (mostly handled by fallback defaults)  

#### 1.7 Documentation Node

- **Overview:**  
Provides a sticky note documenting the expected Google Sheets column structure for patient vitals data.

- **Nodes Involved:**  
`Sticky Note`

- **Node Details:**  
  - Type: Sticky Note (documentation)  
  - Content: Describes Google Sheets columns such as patient_id, patient_name, room_number, timestamp, various vital sign fields, alert_level, alerts, and device_status  
  - Purpose: Helps maintainers understand data schema in Google Sheets  
  - Position: Off the main flow, informational only

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                    | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                             |
|----------------------------|--------------------|----------------------------------|------------------------------|------------------------------|-------------------------------------------------------------------------------------------------------|
| Poll Device Data Every 30s | Cron Trigger       | Initiates periodic polling       | None                         | Fetch from IntelliVue Gateway |                                                                                                       |
| Fetch from IntelliVue Gateway | HTTP Request      | Retrieves patient data via API   | Poll Device Data Every 30s   | Process Device Data           |                                                                                                       |
| Process Device Data         | Code               | Parses and standardizes vitals   | Fetch from IntelliVue Gateway | Validate & Enrich Data        |                                                                                                       |
| Validate & Enrich Data      | Code               | Validates, enriches, classifies  | Process Device Data           | Save to Patient Database      |                                                                                                       |
| Save to Patient Database    | Google Sheets      | Stores processed vitals          | Validate & Enrich Data        | Switch                       |                                                                                                       |
| Switch                     | Switch             | Routes on alert level             | Save to Patient Database      | Send Clinical Alert           |                                                                                                       |
| Send Clinical Alert         | Email Send         | Sends alert emails on critical alerts | Switch                      | None                         |                                                                                                       |
| Sticky Note                | Sticky Note        | Documents Google Sheets structure | None                         | None                         | 📊 Google Sheet Structure: Columns: patient_id, patient_name, room_number, timestamp, ecg_heart_rate, spo2_oxygen_saturation, nibp_systolic, nibp_diastolic, temperature_celsius, respiration_rate, etco2_end_tidal, cardiac_output, alert_level, alerts, device_status |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger**  
   - Add a **Cron** node named `Poll Device Data Every 30s`.  
   - Configure to trigger every 30 seconds (default settings).  
   - No credentials needed.

2. **Add HTTP Request Node**  
   - Add an **HTTP Request** node named `Fetch from IntelliVue Gateway`.  
   - Set Method: GET  
   - URL: `http://{{ $env.INTELLIVUE_GATEWAY_IP }}:8080/api/patients/current`  
   - Set Timeout: 5000 ms  
   - Authentication: HTTP Basic Auth with predefined credentials named (e.g.) “test - auth” containing valid IntelliVue Gateway credentials.  
   - Connect output of Cron node to this node.

3. **Add Code Node for Parsing**  
   - Add a **Code** node named `Process Device Data`.  
   - Paste the provided JavaScript code that:  
     - Detects data format (HL7, CSV, JSON)  
     - Parses and maps vital signs and patient info  
     - Returns an array of standardized JSON vitals objects.  
   - Connect output of HTTP Request node to this code node.

4. **Add Code Node for Validation & Enrichment**  
   - Add a **Code** node named `Validate & Enrich Data`.  
   - Paste the provided JavaScript code that:  
     - Validates patient ID presence  
     - Calculates pulse pressure and mean arterial pressure  
     - Converts temperature to Fahrenheit  
     - Determines clinical status and alert levels  
     - Generates clinical alert descriptions  
     - Adds metadata timestamps  
   - Connect output of parsing code node to this node.

5. **Add Google Sheets Node**  
   - Add a **Google Sheets** node named `Save to Patient Database`.  
   - Set Operation: Append  
   - Sheet ID: Replace `{{your_patient_vitals_sheet_id}}` with actual Google Sheets ID containing a sheet named `PatientVitals` with columns described below.  
   - Range: `PatientVitals!A:Z`  
   - Authentication: Configure with Google Service Account credentials authorized to access the sheet.  
   - Connect output of validation node to this node.

6. **Add Switch Node for Alert Routing**  
   - Add a **Switch** node named `Switch`.  
   - Configure rules:  
     - If `{{$json.alert_level}}` equals `CRITICAL` (case sensitive), output 1 (true branch).  
     - Else output 2 (false branch).  
   - Connect output of Google Sheets node to this switch node.

7. **Add Email Send Node for Alerts**  
   - Add an **Email Send** node named `Send Clinical Alert`.  
   - Configure:  
     - From Email: `intellivue-alerts@hospital.com`  
     - To Email: dynamic expression: `nursing-{{ $json.room_number }}@hospital.com, oncall-physician@hospital.com`  
     - Subject: `🏥 PHILIPS INTELLIVUE ALERT – {{ $json.alert_level }}`  
     - Text content:  
       ```
       Patient: {{ $json.patient_name }} (ID: {{ $json.patient_id }}), Room: {{ $json.room_number }}
       HR: {{ $json.heart_rate || 'N/A' }} bpm | SpO₂: {{ $json.spo2 || 'N/A' }}% | BP: {{ $json.bp_systolic || 'N/A' }}/{{ $json.bp_diastolic || 'N/A' }} mmHg | Temp: {{ $json.temperature_celsius || 'N/A' }}°C | Resp: {{ $json.respiration_rate || 'N/A' }}/min | EtCO₂: {{ $json.etco2 || 'N/A' }} mmHg
       Alerts: {{ ($json.clinical_alerts || []).join(', ') || 'None' }}
       MAP: {{ $json.mean_arterial_pressure || 'N/A' }} mmHg | Status: {{ $json.clinical_status || 'Unknown' }}
       Action: Assess patient, confirm vitals, check devices, notify physician, document actions.
       ```  
     - Authentication: SMTP credentials configured (e.g., “SMTP -test”)  
   - Connect switch node’s critical alert output to this node.

8. **Add Sticky Note for Documentation** (Optional)  
   - Add a **Sticky Note** node named `Sticky Note`.  
   - Content:  
     ```
     📊 Google Sheet Structure:
     Columns: patient_id, patient_name, room_number, timestamp, ecg_heart_rate, spo2_oxygen_saturation, nibp_systolic, nibp_diastolic, temperature_celsius, respiration_rate, etco2_end_tidal, cardiac_output, alert_level, alerts, device_status
     ```
   - Position for clarity; no input/output connections needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                      |
|-----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| The workflow assumes Philips IntelliVue Gateway IP is available via environment variable `INTELLIVUE_GATEWAY_IP`. | Setup environment variables accordingly before workflow execution. |
| Google Sheets must have a sheet named `PatientVitals` with columns matching the documented schema.              | See Sticky Note content for column details.                         |
| SMTP and Google API credentials must be configured securely via n8n credentials manager.                         | Use OAuth2 for Google Sheets with Service Account recommended.     |
| Alert email recipients are dynamically set based on patient room number.                                         | Adjust email domains and addresses to match hospital policies.     |
| Parsing supports multiple data input formats to ensure compatibility with different Philips IntelliVue data exports. | HL7 messages, CSV exports, and Gateway API JSON are supported.     |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.