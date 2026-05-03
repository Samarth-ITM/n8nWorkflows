Track iOS App Sizes with Trend Alerts using Google Sheets and Gmail Notifications

https://n8nworkflows.xyz/workflows/track-ios-app-sizes-with-trend-alerts-using-google-sheets-and-gmail-notifications-9617


# Track iOS App Sizes with Trend Alerts using Google Sheets and Gmail Notifications

### 1. Workflow Overview

This workflow automates the monitoring of iOS app IPA file sizes on a daily schedule. Its core purpose is to track the size of an app's IPA file over time, log size data into a Google Sheet, analyze size trends against predefined thresholds, and send alert emails via Gmail when significant size changes or thresholds are exceeded. It is designed for app developers or release managers who want automated visibility into their app binary size evolution with proactive notifications.

The workflow is logically divided into these blocks:

- **1.1 Scheduling and Configuration:** Daily trigger and static app metadata setup including IPA download URL.
- **1.2 IPA Download and Size Calculation:** Fetching the IPA file, calculating its size in multiple units, and timestamping.
- **1.3 Data Logging:** Appending the size data with metadata to a Google Sheet for historical tracking.
- **1.4 Trend Analysis and Alerting:** Analyzing size trends based on thresholds, deciding alert levels, and conditionally sending email alerts.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduling and Configuration

**Overview:**  
Triggers the workflow daily at midnight and sets the app metadata and IPA download URL for processing.

**Nodes Involved:**  
- Daily size check (Schedule Trigger)  
- App Configuration (Set)  
- Sticky Note (explanatory)

**Node Details:**

- **Daily size check**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow execution once per day at midnight (00:00:00).  
  - Configuration: Cron expression set to `"0 0 0 * * *"` (every day at 00:00).  
  - Inputs: None (trigger node)  
  - Outputs: Triggers "App Configuration" node.  
  - Edge Cases: Misfire if n8n server is down at scheduled time; ensure n8n scheduler is running.

- **App Configuration**  
  - Type: Set  
  - Role: Defines static app information and IPA download URL to be used downstream.  
  - Configuration: Four string fields set — `app_name`, `version`, `build_number`, and `ipa_url`. These must be manually updated with real values before use.  
  - Inputs: From "Daily size check"  
  - Outputs: Provides configuration JSON to "Download IPA File".  
  - Edge Cases: Workflow will fail or download fail if `ipa_url` is empty, invalid, or unreachable.

- **Sticky Note**  
  - Role: Documentation within the workflow UI, highlighting this as the central place to define monitored apps and IPA links.  
  - No technical impact.

---

#### 2.2 IPA Download and Size Calculation

**Overview:**  
Downloads the IPA file from the provided URL and computes its size in bytes, kilobytes, and megabytes. Adds timestamps for logging.

**Nodes Involved:**  
- Download IPA File (HTTP Request)  
- Calculate Sizes (Code)  
- Sticky Note (explanatory)

**Node Details:**

- **Download IPA File**  
  - Type: HTTP Request  
  - Role: Downloads the IPA file binary from the `ipa_url`.  
  - Configuration: URL sourced dynamically from `{{ $json.ipa_url }}`; response format set to "file" to handle binary data.  
  - Inputs: From "App Configuration"  
  - Outputs: Binary data of IPA file to "Calculate Sizes".  
  - Edge Cases: Download failures due to invalid URL, network issues, or authentication required on URL; file size zero if empty download.

- **Calculate Sizes**  
  - Type: Code (JavaScript)  
  - Role: Extracts the IPA file size from binary data, converts to KB and MB, adds current date and ISO timestamp, and logs size info.  
  - Key expressions:  
    - `fileSize = item.binary?.data?.fileSize || 0`  
    - `fileSizeKB = Math.round(fileSize / 1024)`  
    - `fileSizeMB = Math.round(fileSize / (1024 * 1024) * 100) / 100`  
    - Date formatted as `YYYY-MM-DD` and full ISO timestamp.  
  - Inputs: Binary IPA file from previous node  
  - Outputs: JSON object with size metrics and metadata to "Append row in sheet".  
  - Edge Cases: If no binary data or fileSize is missing, defaults to 0; ensure binary key matches "data".  
  - Logs size in MB for debugging.

- **Sticky Note1**  
  - Role: Documents that this block fetches the IPA and prepares size data for calculation.

---

#### 2.3 Data Logging

**Overview:**  
Appends the calculated size data with metadata and timestamps as a new row in a specified Google Sheets document for historical tracking.

**Nodes Involved:**  
- Append row in sheet (Google Sheets)  
- Sticky Note2 (explanatory)

**Node Details:**

- **Append row in sheet**  
  - Type: Google Sheets  
  - Role: Appends a new row with app size data to a configured Google Sheet.  
  - Configuration:  
    - Operation: Append  
    - Sheet Name: User must specify (placeholder "ENTER_SHEET_NAME")  
    - Document ID: User must specify (placeholder "ENTER_SHEET_ID")  
    - Columns: Defined below sheet row (dynamic mapping)  
  - Credentials: Uses OAuth2 credential for Google Sheets API.  
  - Inputs: JSON data from "Calculate Sizes"  
  - Outputs: Passes data onward to "Analyze Size Trends"  
  - Edge Cases:  
    - Authentication errors if OAuth2 credential expires or is invalid.  
    - Sheet or document not found if IDs incorrect.  
    - Data type mismatch if Sheet columns don’t align.

- **Sticky Note2**  
  - Role: Documents that this node saves app size data with timestamp for historical records.

---

#### 2.4 Trend Analysis and Alerting

**Overview:**  
Analyzes the logged app size data against predefined thresholds to categorize size, determine trend status, and classify alert levels. Conditionally sends an alert email if thresholds are exceeded.

**Nodes Involved:**  
- Analyze Size Trends (Code)  
- Check for Alerts (If)  
- Send Alert Email (Gmail)  

**Node Details:**

- **Analyze Size Trends**  
  - Type: Code (JavaScript)  
  - Role:  
    - Categorizes app size into SMALL (<50MB), MEDIUM (<150MB), LARGE (<300MB), VERY_LARGE (≥300MB).  
    - Sets trend status to `SIZE_ALERT_LARGE` (>500MB, CRITICAL alert), `SIZE_WARNING` (>300MB, WARNING alert), or `SIZE_NORMAL` otherwise.  
    - Returns enriched JSON with `size_category`, `trend_status`, `alert_level`, timestamp, and note.  
  - Inputs: Data appended to Google Sheets from previous node  
  - Outputs: Passes to "Check for Alerts" node  
  - Edge Cases: Thresholds are static; custom thresholds require code modification.  
  - Logs analysis summary.

- **Check for Alerts**  
  - Type: If  
  - Role: Filters out stable trend statuses; only passes records where `trend_status` is not `"STABLE"` (actually configured to exclude `"STABLE"` status).  
  - Inputs: From "Analyze Size Trends"  
  - Outputs: If condition true, triggers "Send Alert Email".  
  - Edge Cases: If `trend_status` field missing or unexpected string, may cause logic errors.

- **Send Alert Email**  
  - Type: Gmail  
  - Role: Sends alert emails with app name, current size, previous size (both currently using same field, likely a logic gap), trend status, and timestamp.  
  - Configuration:  
    - Recipient: `xyz@gmail.com` (placeholder, must be replaced)  
    - Subject and message use expressions referencing JSON fields (e.g. `{{ $json['App Name'] }}`)  
  - Credentials: Gmail OAuth2 account required.  
  - Inputs: From "Check for Alerts"  
  - Edge Cases:  
    - Email credential must be valid and authorized.  
    - Template references fields like `App Name` and `Size(Bytes)` that do not exactly match node JSON keys (case mismatch); may cause empty values.  
    - Previous size is referenced same as current size, indicating no historic comparison — potential improvement area.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                                  | Input Node(s)        | Output Node(s)          | Sticky Note                                                                                  |
|---------------------|---------------------|-------------------------------------------------|----------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Daily size check     | Schedule Trigger    | Triggers workflow daily at midnight             | None                 | App Configuration       |                                                                                              |
| App Configuration   | Set                 | Defines app metadata and IPA download URL       | Daily size check     | Download IPA File       | ** Central place to define which apps to monitor and their IPA download links **             |
| Download IPA File   | HTTP Request        | Downloads IPA binary file from URL               | App Configuration    | Calculate Sizes         | **Fetches the IPA from the configured URL and makes it ready for size calculation**          |
| Calculate Sizes     | Code                | Calculates IPA size (bytes, KB, MB) and timestamps | Download IPA File    | Append row in sheet     |                                                                                              |
| Append row in sheet | Google Sheets       | Appends size data to Google Sheet for history   | Calculate Sizes      | Analyze Size Trends     | **Saves the app size data with timestamp into Google Sheets for historical tracking**        |
| Analyze Size Trends | Code                | Categorizes size, sets trend and alert levels   | Append row in sheet  | Check for Alerts        |                                                                                              |
| Check for Alerts    | If                  | Filters for non-stable trends to trigger alerts | Analyze Size Trends  | Send Alert Email        |                                                                                              |
| Send Alert Email    | Gmail               | Sends alert emails on size warnings              | Check for Alerts     | None                   |                                                                                              |
| Sticky Note         | Sticky Note         | Documentation block                             | None                 | None                   | ** Central place to define which apps to monitor and their IPA download links **             |
| Sticky Note1        | Sticky Note         | Documentation block                             | None                 | None                   | **Fetches the IPA from the configured URL and makes it ready for size calculation**          |
| Sticky Note2        | Sticky Note         | Documentation block                             | None                 | None                   | **Saves the app size data with timestamp into Google Sheets for historical tracking**        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node**  
   - Name: `Daily size check`  
   - Set Cron Expression to `0 0 0 * * *` (runs daily at midnight).  
   - Connect output to next node.

3. **Add a Set node**  
   - Name: `App Configuration`  
   - Add the following string fields with initial placeholder values:  
     - `app_name`: e.g., `"APP_NAME"`  
     - `version`: e.g., `"1.0.0"`  
     - `build_number`: e.g., `"1"`  
     - `ipa_url`: e.g., `"ENTER_IPA_FILE_DOWNLOAD_URL"` (replace with actual IPA download URL).  
   - Connect `Daily size check` output to this node.

4. **Add an HTTP Request node**  
   - Name: `Download IPA File`  
   - Set Method to `GET`  
   - Set URL to `={{ $json.ipa_url }}` (dynamically from previous node)  
   - Under Options → Response, set Response Format to `File` to download binary data.  
   - Connect `App Configuration` output here.

5. **Add a Code node**  
   - Name: `Calculate Sizes`  
   - Use JavaScript code to:  
     - Extract file size from binary data: `item.binary.data.fileSize`  
     - Convert to KB and MB (rounded)  
     - Append current date (YYYY-MM-DD) and ISO timestamp  
     - Return enriched JSON object with app metadata and size info.  
   - Connect from `Download IPA File`.

6. **Add a Google Sheets node**  
   - Name: `Append row in sheet`  
   - Operation: `Append`  
   - Set Document ID and Sheet Name with your Google Sheet's ID and sheet tab name.  
   - Configure columns to match the fields output by `Calculate Sizes`.  
   - Set up Google Sheets OAuth2 credentials for API access.  
   - Connect from `Calculate Sizes`.

7. **Add a Code node**  
   - Name: `Analyze Size Trends`  
   - Use JavaScript code to:  
     - Compare current size (in MB) against thresholds: 50, 150, 300, 500 MB  
     - Assign size category (SMALL, MEDIUM, LARGE, VERY_LARGE)  
     - Assign trend status (SIZE_NORMAL, SIZE_WARNING, SIZE_ALERT_LARGE)  
     - Assign alert level (INFO, WARNING, CRITICAL)  
     - Add analysis timestamp and note string.  
   - Connect from `Append row in sheet`.

8. **Add an If node**  
   - Name: `Check for Alerts`  
   - Condition: Check if `trend_status` is NOT equal to `"STABLE"` (or customize as needed).  
   - Connect from `Analyze Size Trends`.

9. **Add a Gmail node**  
   - Name: `Send Alert Email`  
   - Configure:  
     - Use Gmail OAuth2 credentials.  
     - Set recipient email address (e.g., `xyz@gmail.com`).  
     - Use expressions in subject and message to include app name, size, trend status, and current time.  
     - Note: Adjust JSON field names in expressions to exactly match upstream data fields.  
   - Connect `If` node's `true` output to this node.

10. **Add Sticky Notes** for documentation:  
    - Near `App Configuration`: note this is the central place for app info and IPA URL.  
    - Near `Download IPA File`: note it fetches IPA for size calculation.  
    - Near `Append row in sheet`: note it logs data for historical tracking.

11. **Test the workflow:**  
    - Replace placeholders with real values.  
    - Trigger manually or wait for schedule.  
    - Check logs and Google Sheet for data.  
    - Verify email alerts trigger correctly when thresholds exceeded.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow uses static thresholds for size categorization; adapt thresholds in the "Analyze Size Trends" node as needed for your app. | Threshold customization advice.                                                                     |
| Email alert template references fields `App Name` and `Size(Bytes)` but input JSON uses lowercase keys like `app_name` and `file_size_bytes`. Adjust expressions accordingly to avoid empty values. | Important for email template correctness.                                                          |
| The workflow does not track historical size comparisons (previous size vs current size); consider enhancing by retrieving last size from Google Sheets for accurate trend analysis. | Suggested enhancement for better alert accuracy.                                                   |
| Google Sheets OAuth2 credentials must have write access to the target spreadsheet.                    | Credential and permissions reminder.                                                                |
| Gmail OAuth2 credentials require Gmail API enabled and proper consent scopes.                         | Credential and API setup reminder.                                                                  |
| Workflow tested on n8n version 0.202.x and above; verify compatibility if using older versions.      | Version compatibility note.                                                                         |
| More info on Google Sheets node configuration: https://docs.n8n.io/nodes/n8n-nodes-base.googlesheets/ | Official n8n documentation link.                                                                    |
| More info on Gmail node configuration: https://docs.n8n.io/nodes/n8n-nodes-base.gmail/               | Official n8n documentation link.                                                                    |

---

**Disclaimer:** The provided text is generated exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal or protected content. All handled data is legal and public.