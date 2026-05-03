Match CVE Threats to Internal Assets with Google Sheets and Email Alerts

https://n8nworkflows.xyz/workflows/match-cve-threats-to-internal-assets-with-google-sheets-and-email-alerts-6410


# Match CVE Threats to Internal Assets with Google Sheets and Email Alerts

### 1. Workflow Overview

This workflow automates the process of matching new CVE (Common Vulnerabilities and Exposures) threat data to internal asset information stored in Google Sheets, enriching threat data with asset context, updating the threat records, archiving processed threats, and sending summary email alerts. It targets security operations teams who need to track CVE threats relevant to their internal infrastructure efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow on a daily schedule.
- **1.2 Data Loading and Merging**: Loads threat data and asset database from Google Sheets and merges them for correlation.
- **1.3 Threat-to-Asset Matching**: Executes custom logic to associate threats with internal assets.
- **1.4 Data Update and Archiving**: Updates Google Sheets with new threat information, archives processed threats, and deletes obsolete rows.
- **1.5 Notification**: Sends a summary email alert with the results of the matching process.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Initiates the entire workflow execution once daily to ensure up-to-date processing of CVE threats and asset information.

- **Nodes Involved:**  
  - 🔁 Daily Trigger

- **Node Details:**

  - **🔁 Daily Trigger**  
    - Type: Cron trigger  
    - Role: Triggers workflow on a scheduled basis (daily)  
    - Configuration: Default cron settings (implied daily run, no explicit timing parameters shown)  
    - Inputs: None  
    - Outputs: Triggers the "📊Threats Sheets" node  
    - Edge Cases: Misconfigured cron expressions could prevent execution; no authentication required here

---

#### 2.2 Data Loading and Merging

- **Overview:**  
  Loads current CVE threat data and internal asset data from respective Google Sheets, then merges the datasets for combined processing.

- **Nodes Involved:**  
  - 📊Threats Sheets  
  - 📊Load Asset DB  
  - Merge

- **Node Details:**

  - **📊Threats Sheets**  
    - Type: Google Sheets node  
    - Role: Retrieves CVE threat records from a Google Sheet  
    - Configuration: Reads specified sheet containing threat data  
    - Inputs: Triggered by "🔁 Daily Trigger"  
    - Outputs: Provides data to "📊Load Asset DB" and "Merge" nodes  
    - Edge Cases: Google API auth errors, empty or malformed sheet data, API rate limits

  - **📊Load Asset DB**  
    - Type: Google Sheets node  
    - Role: Retrieves internal asset information from a Google Sheet  
    - Configuration: Reads specified sheet containing asset data  
    - Inputs: Receives data from "📊Threats Sheets" node  
    - Outputs: Provides asset data to "Merge" node  
    - Edge Cases: Similar to above; potential for missing or stale asset data

  - **Merge**  
    - Type: Merge node  
    - Role: Combines threat data and asset data into a unified dataset for processing  
    - Configuration: Default merge mode (probably 'append' or 'merge by key')  
    - Inputs: Receives threat data (primary input) and asset data (secondary input)  
    - Outputs: Sends combined data to "🧠Match Threats to Assets" node  
    - Edge Cases: Mismatch in data keys or empty inputs could result in incomplete merges or errors

---

#### 2.3 Threat-to-Asset Matching

- **Overview:**  
  Processes the merged data to identify and associate CVE threats with relevant internal assets using custom logic implemented in a function node.

- **Nodes Involved:**  
  - 🧠Match Threats to Assets

- **Node Details:**

  - **🧠Match Threats to Assets**  
    - Type: Function node  
    - Role: Custom JavaScript logic to enrich threat data with related asset information  
    - Configuration: Executes once per workflow run, outputs processed data for next steps  
    - Inputs: Receives merged dataset from "Merge" node  
    - Outputs: Sends enriched data simultaneously to:  
      - 📬Send Summary Email (for notification)  
      - 📊 Apend New Threat (for appending to archive)  
      - 📊 Delete Row (for cleanup)  
    - Edge Cases: Logic errors or unhandled data structures may cause exceptions; requires robust error handling for missing fields or unexpected data formats

---

#### 2.4 Data Update and Archiving

- **Overview:**  
  Updates Google Sheets by appending new threat entries to an archive sheet, deletes obsolete threat rows from the current sheet, and maintains data integrity.

- **Nodes Involved:**  
  - 📊 Apend New Threat  
  - 🗃️ Archived_Threats  
  - 📊 Delete Row

- **Node Details:**

  - **📊 Apend New Threat**  
    - Type: Google Sheets node  
    - Role: Appends matched threat records to an archive sheet for historical tracking  
    - Configuration: Target archive sheet specified  
    - Inputs: Receives enriched threat data from "🧠Match Threats to Assets"  
    - Outputs: Connects to "🗃️ Archived_Threats" (possibly for verification or additional processing)  
    - Edge Cases: API write permission errors; partial data writes; concurrency issues

  - **🗃️ Archived_Threats**  
    - Type: Google Sheets node  
    - Role: Represents the archive sheet holding historical threat data  
    - Configuration: Reads or manages archived data as needed  
    - Inputs: Receives data from "📊 Apend New Threat"  
    - Outputs: No further output connections in this workflow  
    - Edge Cases: Data integrity risks if append fails; potential for data duplication if not managed carefully

  - **📊 Delete Row**  
    - Type: Google Sheets node  
    - Role: Deletes rows from the current threat sheet that have been processed and archived  
    - Configuration: Targets specific rows to remove obsolete data  
    - Inputs: Receives deletion instructions from "🧠Match Threats to Assets"  
    - Outputs: No further nodes connected  
    - Edge Cases: Incorrect row identification can cause data loss; permissions required for deletion

---

#### 2.5 Notification

- **Overview:**  
  Sends an email summarizing the matching results, alerting relevant stakeholders about CVE threats matched to internal assets.

- **Nodes Involved:**  
  - 📬Send Summary Email

- **Node Details:**

  - **📬Send Summary Email**  
    - Type: Email Send node  
    - Role: Dispatches summary email notifications with enriched threat information  
    - Configuration: Uses configured SMTP or email credentials (e.g., Outlook OAuth2, SMTP server)  
    - Inputs: Receives processed data from "🧠Match Threats to Assets"  
    - Outputs: None  
    - Edge Cases: Authentication failures, SMTP server downtime, email formatting errors

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                          | Input Node(s)             | Output Node(s)                               | Sticky Note |
|------------------------|---------------------|----------------------------------------|---------------------------|----------------------------------------------|-------------|
| 🔁 Daily Trigger         | Cron Trigger        | Initiate workflow daily                 | —                         | 📊Threats Sheets                             |             |
| 📊Threats Sheets         | Google Sheets       | Load CVE threat data                    | 🔁 Daily Trigger           | 📊Load Asset DB, Merge                       |             |
| 📊Load Asset DB          | Google Sheets       | Load internal asset data                | 📊Threats Sheets           | Merge                                       |             |
| Merge                   | Merge Node          | Combine threat and asset data           | 📊Threats Sheets, 📊Load Asset DB | 🧠Match Threats to Assets                  |             |
| 🧠Match Threats to Assets | Function            | Enrich threat data by matching to assets | Merge                    | 📬Send Summary Email, 📊 Apend New Threat, 📊 Delete Row |             |
| 📬Send Summary Email     | Email Send          | Send summary notification               | 🧠Match Threats to Assets  | —                                            |             |
| 📊 Apend New Threat      | Google Sheets       | Append matched threats to archive       | 🧠Match Threats to Assets  | 🗃️ Archived_Threats                          |             |
| 🗃️ Archived_Threats       | Google Sheets       | Archive sheet for threat data            | 📊 Apend New Threat        | —                                            |             |
| 📊 Delete Row            | Google Sheets       | Delete processed rows from current sheet | 🧠Match Threats to Assets  | —                                            |             |
| Sticky Note             | Sticky Note         | —                                      | —                         | —                                            |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it "M2 - Asset Enrichment Engine".**

2. **Add a Cron Trigger node:**  
   - Name: 🔁 Daily Trigger  
   - Type: Cron Trigger  
   - Configuration: Set to trigger once daily at your preferred time (e.g., 00:00)  
   - No credentials required

3. **Add a Google Sheets node to load threat data:**  
   - Name: 📊Threats Sheets  
   - Type: Google Sheets  
   - Credentials: Connect to your Google account with access to threat data sheet  
   - Operation: Read rows from the sheet containing CVE threat data  
   - Sheet ID and range: Set to the appropriate sheet/range containing current threat records  
   - Connect input from 🔁 Daily Trigger

4. **Add a Google Sheets node to load asset database:**  
   - Name: 📊Load Asset DB  
   - Type: Google Sheets  
   - Credentials: Same Google account or appropriate credentials  
   - Operation: Read rows from the sheet containing internal asset data  
   - Sheet ID and range: Set accordingly  
   - Connect input from 📊Threats Sheets (main output index 0)

5. **Add a Merge node to combine datasets:**  
   - Name: Merge  
   - Type: Merge  
   - Mode: Choose appropriate merge strategy (e.g., 'Append' or 'Merge By Key') depending on your data matching needs  
   - Connect inputs:  
     - Input 1 (primary): from 📊Threats Sheets (main output index 1)  
     - Input 2 (secondary): from 📊Load Asset DB  
   - Output connected to next function node

6. **Add a Function node to match threats to assets:**  
   - Name: 🧠Match Threats to Assets  
   - Type: Function  
   - Configuration: Add JavaScript code that processes the merged data to associate CVE threats with internal assets. The function should output three sets of data:  
     - Data for summary email  
     - Data for appending to the archive sheet  
     - Data for deletion of rows in the current threat sheet  
   - Connect input from Merge node

7. **Add a Google Sheets node to append new threats to archive:**  
   - Name: 📊 Apend New Threat  
   - Type: Google Sheets  
   - Credentials: Google account with write access to archive sheet  
   - Operation: Append rows to archive sheet containing enriched threat data  
   - Connect input from 🧠Match Threats to Assets (main output for append)

8. **Add a Google Sheets node representing the archive sheet:**  
   - Name: 🗃️ Archived_Threats  
   - Type: Google Sheets  
   - Credentials: Same as above  
   - Operation: Read or manage archive data as needed (optional for this workflow)  
   - Connect input from 📊 Apend New Threat

9. **Add a Google Sheets node to delete processed rows:**  
   - Name: 📊 Delete Row  
   - Type: Google Sheets  
   - Credentials: Google account with delete permissions  
   - Operation: Delete rows from current threat sheet that have been processed and appended to archive  
   - Connect input from 🧠Match Threats to Assets (main output for delete)

10. **Add an Email Send node to send summary emails:**  
    - Name: 📬Send Summary Email  
    - Type: Email Send  
    - Credentials: Configure SMTP or OAuth2 (e.g., Outlook OAuth2) to send emails  
    - Settings: Compose email body summarizing matched threats and assets  
    - Connect input from 🧠Match Threats to Assets (main output for email)

11. **Verify all credentials and permissions for Google Sheets and email sending are properly configured.**

12. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                           |
|-------------------------------------------------------------------------------------------------|------------------------------------------|
| The workflow relies on Google Sheets API with appropriate scopes for reading, appending, and deleting rows. Ensure OAuth2 credentials are properly configured. | Google Sheets API documentation          |
| Email node requires SMTP or OAuth2 credentials for sending alerts; verify email server settings. | Email provider documentation              |
| Custom matching logic in the Function node should be carefully tested to avoid data mismatches. Consider adding try-catch blocks and validation. | JavaScript error handling best practices |
| Cron trigger timing can be adjusted to suit operational needs (e.g., off-business hours).       | n8n Cron Trigger documentation            |
| For large datasets, consider pagination or API rate limits when reading Google Sheets data.     | Google Sheets API quotas and limits       |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.