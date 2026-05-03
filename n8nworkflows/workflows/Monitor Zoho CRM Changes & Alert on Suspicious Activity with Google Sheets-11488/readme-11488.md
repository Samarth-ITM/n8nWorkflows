Monitor Zoho CRM Changes & Alert on Suspicious Activity with Google Sheets

https://n8nworkflows.xyz/workflows/monitor-zoho-crm-changes---alert-on-suspicious-activity-with-google-sheets-11488


# Monitor Zoho CRM Changes & Alert on Suspicious Activity with Google Sheets

### 1. Workflow Overview

This workflow monitors changes in specified Zoho CRM modules and alerts the security team about suspicious activities. It is designed for organizations seeking lightweight, continuous access control and audit logging within Zoho CRM without external audit tools. The workflow consists of two major logical blocks:

- **1.1 API & Data Acquisition:** Calculate a time window since the last run, retrieve Zoho CRM modules, and fetch records modified within that time window for pre-defined modules (Leads, Contacts, Accounts, Deals).

- **1.2 Data Processing & Alerting:** Normalize records, analyze them for suspicious activity based on defined patterns, create audit JSON attachments, log data into Google Sheets, and send email alerts for suspicious events. Finally, update the last processed timestamp to avoid gaps.

---

### 2. Block-by-Block Analysis

#### 1.1 API & Data Acquisition

**Overview:**  
This block calculates the polling time window, retrieves active Zoho CRM modules, filters required modules, and fetches recently modified records from each module.

**Nodes Involved:**  
- Manual Trigger (Run Now)  
- Compute Time Window  
- SET - Workflow Config  
- Zoho: List Modules  
- Prepare Modules List  
- Zoho: Search Module Records  

**Node Details:**

- **Manual Trigger (Run Now)**  
  - Type: Manual trigger node  
  - Purpose: Starts workflow execution manually for testing or on-demand runs.  
  - Input: None  
  - Output: Triggers “Compute Time Window” node.  
  - Potential issues: None; manual start only.

- **Compute Time Window**  
  - Type: Function  
  - Purpose: Calculates the 'from' and 'to' ISO8601 timestamps (with timezone) for the API query window, based on last run time stored in workflow static data. Defaults to last 6 minutes if no previous run.  
  - Key logic: Uses local timezone offset to produce timestamps; does not update lastRun here to avoid gaps.  
  - Input: Trigger from Manual Trigger node.  
  - Output: JSON with `from` and `to` times for querying Zoho.  
  - Edge cases: If static data is missing or corrupted, defaults to a safe interval.

- **SET - Workflow Config**  
  - Type: Set  
  - Purpose: Centralizes workflow parameters like polling interval (5 minutes), required modules (Leads, Contacts, Accounts, Deals), Google Sheet ID/name, dry-run flag, and batch size. Also stores overview metadata and grouping notes.  
  - Input: Output from Compute Time Window.  
  - Output: Passes config data downstream.  
  - Edge cases: Misconfiguration (wrong sheet ID, invalid modules) will cause API or logging failures.

- **Zoho: List Modules**  
  - Type: HTTP Request with OAuth2 authentication for Zoho CRM  
  - Purpose: Retrieves the full list of Zoho CRM modules currently enabled in the account via Zoho CRM API.  
  - URL: `https://www.zohoapis.in/crm/v8/settings/modules`  
  - Input: Configured OAuth2 credentials with CRM read scopes.  
  - Output: JSON response containing modules info.  
  - Failures: OAuth token expiry, API rate limits, network errors, permission issues.

- **Prepare Modules List**  
  - Type: Function  
  - Purpose: Parses the Zoho modules list, filters only required modules (Leads, Contacts, Accounts, Deals), and prepares one item per module for subsequent searching. Passes the 'from' timestamp to each.  
  - Input: Zoho modules API response, workflow 'from' timestamp.  
  - Output: Array of items, each with `module` and `from_local` keys for searching.  
  - Edge cases: Empty or malformed modules list; modules missing expected fields.

- **Zoho: Search Module Records**  
  - Type: HTTP Request with OAuth2  
  - Purpose: For each module, fetches records modified before or equal to the 'from_local' time using Zoho CRM Search API with criteria on Modified_Time.  
  - URL template uses module name and encodes date/time with timezone offset `+05:30` (India Standard Time).  
  - Input: Prepared module items from previous node.  
  - Output: Raw records data per module.  
  - Failures: API limits, invalid module name, malformed expressions for date/time.

---

#### 1.2 Data Processing & Alerting

**Overview:**  
This block normalizes records from Zoho, detects suspicious activities, creates audit JSON files, logs records to Google Sheets, sends email alerts for suspicious items, and updates the last run timestamp.

**Nodes Involved:**  
- Normalize Module Records  
- Analyze for Suspicious Activity  
- Create Audit Attachment (JSON)  
- Log to Google Sheet (append/update)  
- Check Suspicious? (If)  
- Notify Security (Email)  
- Update lastRun (staticData)  

**Node Details:**

- **Normalize Module Records**  
  - Type: Function  
  - Purpose: Extracts records from Zoho API responses and normalizes fields across modules. Sets standard properties like `module`, `recordId`, and `timestamp`.  
  - Input: Raw search results from Zoho per module.  
  - Output: Array of normalized record items with consistent keys for downstream processing.  
  - Edge cases: Missing fields, empty responses; ensures fallback values to avoid crashes.

- **Analyze for Suspicious Activity**  
  - Type: Function  
  - Purpose: Detects suspicious changes based on record fields. Flags records if:  
    - Event includes "delete"  
    - Owner or assigned user changed  
    - Email field changed  
    - Role or privilege changes  
    - More than or equal to 10 fields changed simultaneously  
  - Sets boolean `suspicious` flag and descriptive `suspicious_reasons`.  
  - Input: Normalized records.  
  - Output: Records with suspicion analysis appended.  
  - Edge cases: Variations in field names; robust string matching used.

- **Create Audit Attachment (JSON)**  
  - Type: Function  
  - Purpose: Converts each suspicious or normal record into a formatted JSON audit file as a base64-encoded binary attachment, named with timestamp.  
  - Input: Records with analysis.  
  - Output: Items appended with binary `audit` file suitable for email attachment or storage.  
  - Edge cases: Large records may increase processing time; encoding errors unlikely but possible with malformed data.

- **Log to Google Sheet (append/update)**  
  - Type: Google Sheets node  
  - Purpose: Appends or updates a row per record in a configured Google Sheet with structured audit data fields (Timestamp, Module, Record Id, User Name, Company Name, Email, Suspicious flag, Field Changes Count). Uses `Record Id` as matching column to avoid duplicates.  
  - Input: Records with audit data.  
  - Output: Confirmation of write operation.  
  - Credentials: Requires Google Sheets OAuth2 with write permissions.  
  - Edge cases: Sheet ID or name misconfiguration, API quota exceeded, network errors.

- **Check Suspicious? (If)**  
  - Type: If node  
  - Purpose: Routes items with `suspicious == true` to the alerting email node; others bypass.  
  - Input: Logged records.  
  - Output: Two branches: suspicious and non-suspicious.  
  - Edge cases: Missing or malformed suspicious flags.

- **Notify Security (Email)**  
  - Type: Gmail node (OAuth2)  
  - Purpose: Sends a formatted HTML email alert to the security team for each suspicious record, including detailed context like detected module, record ID, user info, timestamp, reasons, and recommended actions.  
  - Input: Suspicious records from If node.  
  - Output: Email send confirmation.  
  - Credentials: Gmail OAuth2 with send mail scope.  
  - Edge cases: Email quota limits, invalid recipient address, network issues.

- **Update lastRun (staticData)**  
  - Type: Function  
  - Purpose: Updates the global workflow static data key `lastRun` with current UTC ISO timestamp to mark successful processing.  
  - Input: All processed items after logging and notification.  
  - Output: Items passed unchanged.  
  - Edge cases: Static data access issues could cause duplicate processing or gaps.

---

### 3. Summary Table

| Node Name                      | Node Type            | Functional Role                                  | Input Node(s)                | Output Node(s)                  | Sticky Note                                                                                                   |
|-------------------------------|----------------------|-------------------------------------------------|-----------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------|
| Manual Trigger (Run Now)       | Manual Trigger       | Start workflow manually                         | -                           | Compute Time Window             | **Manual Trigger (Run Now)** — Starts the workflow manually for testing or on-demand execution.                |
| Compute Time Window            | Function             | Calculate polling time window                    | Manual Trigger (Run Now)     | SET - Workflow Config           | **Compute Time Window** — Calculates the ‘from’ and ‘to’ timestamps used to fetch recently modified CRM records.|
| SET - Workflow Config          | Set                  | Central configuration parameters                 | Compute Time Window          | Zoho: List Modules             | **SET - Workflow Config** — Central configuration for modules, sheet IDs, batch size, dry-run mode, and metadata.|
| Zoho: List Modules             | HTTP Request (OAuth2)| Fetch available Zoho CRM modules                 | SET - Workflow Config        | Prepare Modules List           | **Zoho: List Modules** — Retrieves the list of Zoho CRM modules via API before filtering.                      |
| Prepare Modules List           | Function             | Filter and prepare modules for record search    | Zoho: List Modules           | Zoho: Search Module Records    | **Prepare Modules List** — Filters allowed modules and prepares one item per module for sequential searching.   |
| Zoho: Search Module Records    | HTTP Request (OAuth2)| Fetch recently modified records per module      | Prepare Modules List         | Normalize Module Records       | **Zoho: Search Module Records** — Fetches recently modified CRM records for each module within the time window. |
| Normalize Module Records       | Function             | Standardize Zoho records fields                  | Zoho: Search Module Records  | Analyze for Suspicious Activity| **Normalize Module Records** — Standardizes Zoho responses and extracts record-level fields for unified processing.|
| Analyze for Suspicious Activity| Function             | Detect suspicious changes in records            | Normalize Module Records     | Create Audit Attachment (JSON), Log to Google Sheet, Check Suspicious? (If) | **Analyze for Suspicious Activity** — Detects unusual change patterns such as deletions, owner/email/role changes, or bulk edits. |
| Create Audit Attachment (JSON) | Function             | Generate audit JSON file attachment per record  | Analyze for Suspicious Activity | Log to Google Sheet, Check Suspicious? (If) | **Create Audit Attachment (JSON)** — Generates a binary JSON audit file for each processed record.              |
| Log to Google Sheet (append/update) | Google Sheets     | Append or update audit log in Google Sheets     | Analyze for Suspicious Activity, Create Audit Attachment (JSON) | Check Suspicious? (If)           | **Log to Google Sheet (append/update)** — Writes or updates the audit information into Google Sheets as a structured log entry. |
| Check Suspicious? (If)         | If                   | Route suspicious records to alerting             | Analyze for Suspicious Activity, Create Audit Attachment (JSON), Log to Google Sheet | Notify Security (Email) [True branch], Update lastRun (staticData) [False branch] | **Check Suspicious? (If)** — Routes items to alerting flow if suspicious=true, otherwise continues to finalization. |
| Notify Security (Email)        | Gmail                | Send email alert for suspicious records          | Check Suspicious? (If) (True)| Update lastRun (staticData)    | **Notify Security (Email)** — Sends a formatted HTML alert email to the security team for suspicious CRM events. |
| Update lastRun (staticData)    | Function             | Store current timestamp as last successful run  | Check Suspicious? (If), Notify Security (Email) | -                             | **Update lastRun (staticData)** — Stores the current execution timestamp to ensure future runs only process new changes. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start workflow manually for testing or on-demand.  
   - No parameters needed.

2. **Add Function Node: Compute Time Window**  
   - Calculate ‘from’ and ‘to’ timestamps in local ISO8601 format, considering last run time from workflow static data (`lastRun`).  
   - Use timezone offset to format timestamps.  
   - Return JSON with keys: `from`, `to`.

3. **Add Set Node: SET - Workflow Config**  
   - Define workflow parameters:  
     - `pollMinutes`: 5  
     - `requiredModules`: "Leads,Contacts,Accounts,Deals"  
     - `sheetId`: (Google Sheet ID where logs will be written)  
     - `sheetName`: "Sheet1"  
     - `dryRun`: true (optionally false for production)  
     - `batchSize`: 100  
     - Add workflow overview and grouping text as string parameters for documentation.  
   - Pass through incoming JSON.

4. **Add HTTP Request Node: Zoho: List Modules**  
   - Method: GET  
   - URL: `https://www.zohoapis.in/crm/v8/settings/modules`  
   - Authentication: OAuth2 with Zoho CRM scopes (`ZohoCRM.modules.ALL`, `ZohoCRM.settings.all`)  
   - Purpose: Fetch all enabled CRM modules.

5. **Add Function Node: Prepare Modules List**  
   - Extract modules array from Zoho response JSON.  
   - Filter modules to include only those in `requiredModules` list.  
   - For each module, output an item with keys: `module`, `from_local` (timestamp from previous step).  
   - Return array of module items for search.

6. **Add HTTP Request Node: Zoho: Search Module Records**  
   - Method: GET  
   - URL Template: `https://www.zohoapis.in/crm/v8/{{$json["module"]}}/search?criteria=(Modified_Time:less_equal:{{encodeURIComponent(new Date($json["from_local"]).toISOString().split('.')[0] + "+05:30")}})`  
   - OAuth2 authentication same as above.  
   - Purpose: Search for records modified within the defined time window in each module.

7. **Add Function Node: Normalize Module Records**  
   - Parse Zoho response to extract records array.  
   - Normalize key fields for each record: `module`, `recordId`, `timestamp`.  
   - Flatten nested structures as needed to standardize field names.

8. **Add Function Node: Analyze for Suspicious Activity**  
   - For each record, check for:  
     - Deletion events (event field contains "delete")  
     - Changes in owner/assigned_to field  
     - Changes in email/contact_email fields  
     - Changes in role/is_admin/profile fields  
     - More than or equal to 10 fields changed simultaneously  
   - Set `suspicious` boolean flag and `suspicious_reasons` string.  
   - Add `fields_changed_count` numeric field.

9. **Add Function Node: Create Audit Attachment (JSON)**  
   - For each record, create a JSON string with pretty-print.  
   - Base64 encode and attach as binary named `audit` with filename pattern `audit-<timestamp>.json`.  
   - Mime type: `application/json`.

10. **Add Google Sheets Node: Log to Google Sheet (append/update)**  
    - Operation: Append or update  
    - Document ID: From SET config node `sheetId`  
    - Sheet Name: From SET config node `sheetName`  
    - Matching columns: Use `Record Id` to avoid duplicates  
    - Map columns: Timestamp, Module, Record Id, User Name, Company Name, Email, Is Suspicious, Field Changes Count  
    - Credential: Google Sheets OAuth2 with edit permissions.

11. **Add If Node: Check Suspicious?**  
    - Condition: `{{ $json.suspicious }} === true`  
    - True branch: Suspicious records to alert email node  
    - False branch: Non-suspicious records proceed to lastRun update.

12. **Add Gmail Node: Notify Security (Email)**  
    - To: Security team email addresses  
    - Subject: `Zoho CRM Suspicious activity: {{$json.module}} - {{$json.recordId}}`  
    - HTML Body: Detailed alert with record info, timestamp, user, reasons, recommended actions (use provided HTML template).  
    - Credential: Gmail OAuth2 with send mail scope.

13. **Add Function Node: Update lastRun (staticData)**  
    - Store current UTC timestamp in workflow static data key `lastRun`.  
    - Return unchanged items.

14. **Connect Nodes in Sequence:**  
    - Manual Trigger → Compute Time Window → SET - Workflow Config → Zoho: List Modules → Prepare Modules List → Zoho: Search Module Records → Normalize Module Records → Analyze for Suspicious Activity → [Create Audit Attachment (JSON) & Log to Google Sheet (append/update)] → Check Suspicious? → (True) Notify Security → Update lastRun  
                                                                                                                → (False) Update lastRun

15. **Credential Setup:**  
    - Zoho OAuth2: Configure with scopes including `ZohoCRM.modules.ALL`, `ZohoCRM.settings.all`  
    - Google Sheets OAuth2: Editor access to the target spreadsheet  
    - Gmail OAuth2: Send mail permission to send alerts

16. **Testing and Activation:**  
    - Test manually with Manual Trigger  
    - Confirm Google Sheet updates and email alerts  
    - Schedule with Cron trigger at desired interval (e.g., every 5 minutes)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| This workflow provides a lightweight access-control monitoring system for Zoho CRM without needing external audit tools. It is designed for security teams to detect suspicious CRM record changes and receive timely alerts.                                                                                                                                                                                                                                                                                                                                                                                                        | Workflow Description                                                                                                     |
| Setup steps include configuring OAuth2 in Zoho CRM, adding Google Sheets and Gmail credentials in n8n, ensuring proper API scopes, and setting up the Google Sheet with required columns.                                                                                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note "Main (Yellow)"                                                                                              |
| The Gmail node uses a detailed HTML email template with embedded CSS and JavaScript expressions for dynamic formatting of timestamps and fields.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Gmail Notify Security node                                                                                               |
| API endpoints use the Indian Zoho domain `zohoapis.in`, adjust if your Zoho region differs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Zoho API nodes                                                                                                           |
| The workflow uses workflow static data key `lastRun` to track the last successful execution timestamp to avoid processing duplicate records or gaps.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Workflow static data usage                                                                                               |
| The suspicious activity detection can be customized by modifying the Function node "Analyze for Suspicious Activity" logic to add or remove detection patterns.                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Analyze for Suspicious Activity node                                                                                      |
| Google Sheet must have columns: Timestamp, Record Id, Module, Field Changes Count, Is Suspicious, Company Name, Email, User Name for correct mapping.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Google Sheets node configuration                                                                                         |
| For OAuth2 in Zoho CRM, add the n8n callback URL (e.g., `https://<your-n8n-domain>/rest/oauth2-credential/callback`) and use scopes `ZohoCRM.modules.ALL`, `ZohoCRM.settings.all` to enable full CRM API access for modules and settings.                                                                                                                                                                                                                                                                                                                                                                                                  | Zoho OAuth2 setup instructions                                                                                           |
| The workflow is structured to allow easy insertion of additional modules or expanded suspicious activity criteria by editing the Prepare Modules List and Analyze functions respectively.                                                                                                                                                                                                                                                                                                                                                                                                                                               | Workflow extensibility notes                                                                                             |
| This workflow was created and documented by a senior technical analyst specializing in n8n workflows and automation integrations.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Project credits                                                                                                          |

---

**Disclaimer:** The text above is derived exclusively from an automated n8n workflow and respects all valid content policies. It does not contain any illegal, offensive, or protected elements. All data handled is legal and public.