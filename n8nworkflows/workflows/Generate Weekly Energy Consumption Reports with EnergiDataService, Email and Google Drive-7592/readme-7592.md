Generate Weekly Energy Consumption Reports with EnergiDataService, Email and Google Drive

https://n8nworkflows.xyz/workflows/generate-weekly-energy-consumption-reports-with-energidataservice--email-and-google-drive-7592


# Generate Weekly Energy Consumption Reports with EnergiDataService, Email and Google Drive

### 1. Workflow Overview

This workflow automates the generation and distribution of weekly energy consumption reports. Every Monday at 8 AM, it fetches energy consumption data from the EnergiDataService API, processes and normalizes the data, converts it into a CSV file, then both emails the report and uploads it to Google Drive for archival and sharing.

The workflow is logically organized into the following blocks:

- **1.1 Schedule Trigger:** Initiates the workflow execution weekly on Monday at 8 AM.
- **1.2 Data Retrieval:** Calls the EnergiDataService API to fetch the latest energy consumption data.
- **1.3 Data Normalization:** Processes the raw API response to flatten and prepare data records.
- **1.4 File Conversion:** Converts normalized data into a CSV file format.
- **1.5 Report Distribution:** Sends the generated CSV report via email.
- **1.6 Report Storage:** Uploads the CSV report file to Google Drive for storage and access.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:** Triggers the entire workflow automatically every Monday at 8 AM to ensure timely weekly reports.
- **Nodes Involved:**  
  - Schedule Weekly (Mon 8AM)  
- **Node Details:**

  - **Schedule Weekly (Mon 8AM)**
    - Type: Cron Trigger  
    - Role: Initiates workflow execution on a fixed schedule.
    - Configuration: Set to trigger every Monday at 08:00 local server time.
    - Key parameters: `hour=8`, `weekday=monday`, mode=`everyWeek`.
    - Inputs: None (trigger node).
    - Outputs: Triggers "Fetch Energy Data".
    - Edge Cases: If server time zone is misconfigured, trigger time may be off; cron misconfiguration can cause missing runs.
    - Version: V1.
    - Sticky Note: "📝 - **Schedule Weekly (Mon 8AM)**: Trigger every Monday at 8AM."

#### 1.2 Data Retrieval

- **Overview:** Fetches weekly energy consumption data from an external public API.
- **Nodes Involved:**  
  - Fetch Energy Data  
- **Node Details:**

  - **Fetch Energy Data**
    - Type: HTTP Request
    - Role: Calls EnergiDataService.dk API to retrieve consumption data.
    - Configuration:  
      - URL: `https://api.energidataservice.dk/dataset/ConsumptionDE35Hour`
      - Request method: GET (default)
      - No authentication or additional headers configured.
    - Inputs: Triggered from Schedule node.
    - Outputs: JSON response passed to "Normalize Records".
    - Edge Cases: API downtime, rate limiting, unexpected response formats, network timeouts.
    - Version: V1.
    - Sticky Note: "📝 - **Fetch Energy Data**: Call EnergiDataService.dk API."

#### 1.3 Data Normalization

- **Overview:** Extracts and flattens the `records` array from the API response to prepare for file conversion.
- **Nodes Involved:**  
  - Normalize Records  
- **Node Details:**

  - **Normalize Records**
    - Type: Code (JavaScript)
    - Role: Transforms the JSON response into an array of individual record items.
    - Configuration: Inline JavaScript code:
      ```js
      const itemlist = $input.first().json.records;
      return itemlist.map(r => ({ json: r }));
      ```
      This takes the first input’s JSON, extracts the `records` property (assumed to be an array), and maps each record into a separate item.
    - Inputs: JSON from "Fetch Energy Data".
    - Outputs: Each record as a separate item for subsequent nodes.
    - Edge Cases: If `records` is missing or null, code will throw an error; malformed JSON may cause failure.
    - Version: V2.
    - Sticky Note: "📝 - **Normalize Records**: Flatten JSON response (records → items)."

#### 1.4 File Conversion

- **Overview:** Converts the normalized JSON records into a CSV file attached to the workflow.
- **Nodes Involved:**  
  - Convert to File  
- **Node Details:**

  - **Convert to File**
    - Type: Convert To File
    - Role: Transforms JSON data items into a CSV file stored in binary property `data`.
    - Configuration: Default settings (CSV format).
    - Inputs: Normalized records from "Normalize Records".
    - Outputs: Binary CSV file data forwarded to downstream nodes.
    - Edge Cases: Data with inconsistent fields may produce malformed CSV rows.
    - Version: 1.1.
    - Sticky Note: "📝 - **Convert to File**: Turn items into a CSV (binary `data`)."

#### 1.5 Report Distribution

- **Overview:** Sends the generated CSV report as an email attachment.
- **Nodes Involved:**  
  - Send Email Weekly Report  
- **Node Details:**

  - **Send Email Weekly Report**
    - Type: Email Send
    - Role: Emails the CSV report file to a specified recipient.
    - Configuration:
      - To: `test@yopmail.com`
      - From: `test@yopmail.com`
      - Subject: "Weekly Energy Consumption Report"
      - Text Body: "Please find attached the latest weekly energy consumption report."
      - Attachments: Binary data from `data` property (CSV file)
    - Inputs: Receives CSV file from "Convert to File".
    - Outputs: None (terminal node for email).
    - Credential: Uses configured email credentials (implicit).
    - Edge Cases: SMTP server errors, invalid email addresses, attachment size limits.
    - Version: V1.
    - Sticky Note: "📝 - **Email Weekly Report**: Email the CSV file."

#### 1.6 Report Storage

- **Overview:** Uploads the CSV report file to Google Drive for storage and access.
- **Nodes Involved:**  
  - Report File Upload to Google Drive  
- **Node Details:**

  - **Report File Upload to Google Drive**
    - Type: Google Drive
    - Role: Saves the CSV file to a specified Google Drive folder.
    - Configuration:
      - File name template: `energy_report_{{ $now.format('yyyy_MM_dd_HH_ii_ss') }}`
      - Drive: "My Drive"
      - Folder: Root folder (`/`)
      - Upload options: Default (file created/updated)
    - Inputs: Receives CSV binary file from "Convert to File".
    - Outputs: None (terminal node for upload).
    - Credential: Uses OAuth2 Google Drive credentials named "Google Drive account 4".
    - Edge Cases: OAuth token expiration, insufficient permissions, quota limits, network failure.
    - Version: V3.
    - Sticky Note: "📝 - **Upload File to Drive**: Save CSV to Google Drive."

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role               | Input Node(s)               | Output Node(s)                          | Sticky Note                                                     |
|-------------------------------|---------------------|------------------------------|-----------------------------|---------------------------------------|----------------------------------------------------------------|
| Schedule Weekly (Mon 8AM)      | Cron Trigger        | Schedule the weekly run      | None                        | Fetch Energy Data                     | 📝 - **Schedule Weekly (Mon 8AM)**: Trigger every Monday at 8AM.|
| Fetch Energy Data              | HTTP Request        | Fetch energy data from API   | Schedule Weekly (Mon 8AM)   | Normalize Records                    | 📝 - **Fetch Energy Data**: Call EnergiDataService.dk API.     |
| Normalize Records             | Code (JavaScript)   | Flatten JSON records         | Fetch Energy Data           | Convert to File                     | 📝 - **Normalize Records**: Flatten JSON response (records → items).|
| Convert to File               | Convert To File     | Convert JSON to CSV binary   | Normalize Records           | Send Email Weekly Report, Report File Upload to Google Drive | 📝 - **Convert to File**: Turn items into a CSV (binary `data`).|
| Send Email Weekly Report      | Email Send          | Email the CSV report         | Convert to File             | None                                | 📝 - **Email Weekly Report**: Email the CSV file.              |
| Report File Upload to Google Drive | Google Drive         | Upload CSV to Google Drive   | Convert to File             | None                                | 📝 - **Upload File to Drive**: Save CSV to Google Drive.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger Node**  
   - Name: `Schedule Weekly (Mon 8AM)`  
   - Type: Cron Trigger  
   - Parameters: Set `Trigger Times` to trigger every week on Monday at 08:00 (hour=8, weekday=monday).  
   - No credentials required.

2. **Add HTTP Request Node**  
   - Name: `Fetch Energy Data`  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://api.energidataservice.dk/dataset/ConsumptionDE35Hour`  
     - Method: GET (default)  
     - No authentication or headers.  
   - Connect the output of `Schedule Weekly (Mon 8AM)` to this node.

3. **Add Code Node for Normalization**  
   - Name: `Normalize Records`  
   - Type: Code (JavaScript)  
   - Parameters: Paste the following JavaScript:
     ```js
     const itemlist = $input.first().json.records;
     return itemlist.map(r => ({ json: r }));
     ```
   - Connect the output of `Fetch Energy Data` to this node.

4. **Add Convert To File Node**  
   - Name: `Convert to File`  
   - Type: Convert To File  
   - Parameters: Use default settings to convert JSON items to CSV.  
   - Connect the output of `Normalize Records` to this node.

5. **Add Email Send Node**  
   - Name: `Send Email Weekly Report`  
   - Type: Email Send  
   - Parameters:  
     - To Email: `test@yopmail.com`  
     - From Email: `test@yopmail.com`  
     - Subject: `Weekly Energy Consumption Report`  
     - Text: `Please find attached the latest weekly energy consumption report.`  
     - Attachments: Set to use binary property `data` (from `Convert to File`).  
   - Connect the output of `Convert to File` to this node.  
   - Configure SMTP or other email credentials as required.

6. **Add Google Drive Node**  
   - Name: `Report File Upload to Google Drive`  
   - Type: Google Drive  
   - Parameters:  
     - Name: `energy_report_{{ $now.format('yyyy_MM_dd_HH_ii_ss') }}` (use expression to timestamp filename)  
     - Drive ID: Select "My Drive"  
     - Folder ID: Select root folder (`/`)  
     - Upload options: defaults (create new file)  
   - Credentials: Set up and select Google Drive OAuth2 credentials.  
   - Connect the output of `Convert to File` to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                       |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------|
| The workflow is designed for weekly automation to reduce manual reporting tasks.                | Workflow purpose                                     |
| Email recipient and sender addresses `test@yopmail.com` are placeholders and should be replaced.| Replace with real emails before production use       |
| Google Drive OAuth2 credentials require prior setup with appropriate API scopes (drive.file).   | Google API Console                                    |
| EnergiDataService API is public; no authentication currently configured, but rate limits may apply.| API documentation https://energidataservice.dk       |
| Cron node triggers are based on the server’s timezone; ensure correct time zone configuration.  | n8n docs on Cron trigger                              |

---

**Disclaimer:** The provided text is derived solely from an n8n automated workflow export. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.