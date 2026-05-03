Collect API User Data and Store in Google Sheets with CSV backup

https://n8nworkflows.xyz/workflows/collect-api-user-data-and-store-in-google-sheets-with-csv-backup-10725


# Collect API User Data and Store in Google Sheets with CSV backup

### 1. Workflow Overview

This workflow automates the process of fetching user data from an external API, validating and transforming the data, and then storing it in Google Sheets with a simultaneous CSV backup file creation. It targets users and teams who want to streamline importing user information into Google Sheets for collaboration and maintain a CSV backup for offline access or further integration.

Logical blocks in the workflow:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 API Data Fetching:** HTTP request node to retrieve user data from an API endpoint.
- **1.3 API Response Validation:** Conditional check to confirm successful API response.
- **1.4 Data Transformation:** Function node to extract and format user data (name and country).
- **1.5 Data Storage:** Append transformed data to Google Sheets.
- **1.6 CSV Backup Creation:** Generate a CSV file backup of the appended data.
- **1.7 Error Handling:** Stop execution with an error message if API response validation fails.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow manually, allowing users to trigger data collection on demand. It can be replaced with automated triggers like Cron or Webhook for scheduled or event-driven execution.

**Nodes Involved:**  
- Start Workflow Manually

**Node Details:**  

- **Start Workflow Manually**  
  - Type: Manual Trigger  
  - Role: Initiates workflow execution manually  
  - Configuration: No parameters required; triggers on user action  
  - Inputs: None  
  - Outputs: Triggers the next node ("Fetch User Data from API")  
  - Edge Cases: None significant; user must manually trigger  
  - Version: 1  

---

#### 1.2 API Data Fetching

**Overview:**  
Fetches user data from an external API endpoint defined by the environment variable `BASE_URL`. The data is expected in JSON format with a specific structure containing user details.

**Nodes Involved:**  
- Fetch User Data from API

**Node Details:**  

- **Fetch User Data from API**  
  - Type: HTTP Request  
  - Role: Retrieves user data via GET request  
  - Configuration:  
    - URL: Uses environment variable `BASE_URL`  
    - Method: Default GET  
    - Options: Default, no authentication or headers specified  
  - Inputs: Trigger from manual start  
  - Outputs: API response JSON forwarded to validation node  
  - Version: 2  
  - Edge Cases:  
    - Network failures or unreachable API endpoint  
    - Unexpected response format or empty response  
    - Timeout issues if the API is slow  
  - Notes: Requires `BASE_URL` environment variable set, e.g., `https://randomuser.me/api/?results=10`  

---

#### 1.3 API Response Validation

**Overview:**  
Checks that the API response status code equals 200 (success). If true, proceeds to transform data; otherwise, stops the workflow with an error.

**Nodes Involved:**  
- Verify API Response Success  
- Stop on API Failure

**Node Details:**  

- **Verify API Response Success**  
  - Type: If (Conditional)  
  - Role: Validates HTTP status code  
  - Configuration:  
    - Condition: `$json["statusCode"] === 200`  
  - Inputs: API response from Fetch User Data node  
  - Outputs:  
    - True: proceeds to "Transform API Data to Name and Country"  
    - False: routes to "Stop on API Failure"  
  - Version: 1  
  - Edge Cases:  
    - API returns other success codes (e.g., 201) — workflow stops unless modified  
    - Missing or malformed `statusCode` field may cause expression errors  

- **Stop on API Failure**  
  - Type: Stop and Error  
  - Role: Stops workflow execution with a specific error message if API call fails  
  - Configuration:  
    - Error Message: "API request failed — status code not 200. Workflow stopped."  
  - Inputs: From failed condition branch of If node  
  - Outputs: None  
  - Version: 1  
  - Edge Cases: None; stops workflow unconditionally on failure  

---

#### 1.4 Data Transformation

**Overview:**  
Transforms the raw API JSON data into a simplified array of objects containing only `name` and `country` fields for each user.

**Nodes Involved:**  
- Transform API Data to Name and Country

**Node Details:**  

- **Transform API Data to Name and Country**  
  - Type: Function  
  - Role: Processes and maps raw user data to a simplified format  
  - Configuration:  
    - JavaScript code:
      ```javascript
      return $json.results.map(user => {
        return {
          name: `${user.name.first} ${user.name.last}`,
          country: user.location.country
        };
      });
      ```
  - Inputs: Validated API JSON data  
  - Outputs: Array of objects with `name` and `country` keys  
  - Version: 1  
  - Edge Cases:  
    - Missing `results` array or unexpected API structure causes runtime errors  
    - Missing `name` or `location` properties in user objects could cause undefined values  
    - Can customize to handle missing fields or add extra validation  
  - Notes: This node is the key customization point for adapting to different API response formats  

---

#### 1.5 Data Storage

**Overview:**  
Appends the transformed user data to a Google Sheet, specifically columns A (Name) and B (Country). Requires OAuth2 credentials and Google Sheet ID via environment variables.

**Nodes Involved:**  
- Append Data to Google Sheets

**Node Details:**  

- **Append Data to Google Sheets**  
  - Type: Google Sheets  
  - Role: Adds rows to an existing Google Sheet  
  - Configuration:  
    - Operation: Append  
    - Sheet ID: from environment variable `GOOGLE_SHEET_ID`  
    - Range: `"A:B"` to append data into columns A and B  
    - Value Input Mode: RAW to insert data as-is without interpretation  
    - Use Path For Key Row: enabled to match keys with header row  
    - Authentication: OAuth2 (configured separately)  
  - Inputs: Transformed array of user data objects  
  - Outputs: Forward data to CSV backup node  
  - Version: 1  
  - Edge Cases:  
    - Invalid or expired OAuth2 credentials cause authentication errors  
    - Incorrect or missing Google Sheet ID results in API errors  
    - If headers are missing in the sheet, data may not align correctly  
    - API rate limiting from Google Sheets may cause failures  
  - Notes: Requires setting up OAuth2 credentials and environment variable  

---

#### 1.6 CSV Backup Creation

**Overview:**  
Generates a CSV file backup of the appended user data for offline access or integration with other tools.

**Nodes Involved:**  
- Create CSV Backup File

**Node Details:**  

- **Create CSV Backup File**  
  - Type: Spreadsheet File  
  - Role: Converts data into a CSV file saved in workflow context  
  - Configuration:  
    - Operation: To File  
    - File Format: CSV  
    - File Name: `"users_backup_export"` (adds `.csv` extension automatically)  
  - Inputs: Data output from Google Sheets append node  
  - Outputs: File object available for download, email, or upload to other services (not shown in current workflow)  
  - Version: 1  
  - Edge Cases:  
    - Large datasets might cause memory or performance issues  
    - If input data is empty or malformed, the CSV will be empty or invalid  

---

### 3. Summary Table

| Node Name                   | Node Type            | Functional Role                            | Input Node(s)                | Output Node(s)                 | Sticky Note                                                                                       |
|-----------------------------|----------------------|--------------------------------------------|------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| Start Workflow Manually     | Manual Trigger       | Initiates workflow manually                 | None                         | Fetch User Data from API       | Step 1: Start - Manual trigger, replaceable with Cron or Webhook                                |
| Fetch User Data from API    | HTTP Request         | Fetches user data from API endpoint         | Start Workflow Manually       | Verify API Response Success    | Step 2: Fetch API Data - Requires setting `BASE_URL` environment variable                       |
| Verify API Response Success | If                   | Checks if API response status code is 200  | Fetch User Data from API      | Transform API Data to Name and Country, Stop on API Failure | Step 3: Validate Response - Stops workflow if status not 200                                   |
| Stop on API Failure         | Stop and Error       | Stops workflow with error message on failure | Verify API Response Success (false branch) | None                          |                                                                                                 |
| Transform API Data to Name and Country | Function             | Formats raw API data into name and country  | Verify API Response Success (true branch) | Append Data to Google Sheets  | Step 4: Transform Data - Extracts name and country fields; customizable function               |
| Append Data to Google Sheets | Google Sheets        | Appends formatted data to Google Sheet      | Transform API Data to Name and Country | Create CSV Backup File        | Step 5: Save to Google Sheets - Requires OAuth2 credentials and `GOOGLE_SHEET_ID` environment variable |
| Create CSV Backup File      | Spreadsheet File     | Creates CSV backup file from appended data  | Append Data to Google Sheets  | None                          | Step 6: Create CSV Backup - Generates downloadable CSV file automatically                       |
| Workflow Documentation      | Sticky Note          | Describes overall workflow purpose and usage | None                         | None                          | Provides detailed documentation and instructions                                              |
| Step 1 - Trigger            | Sticky Note          | Explains manual trigger usage                | None                         | None                          |                                                                                                 |
| Step 2 - API Request        | Sticky Note          | Explains API request setup                    | None                         | None                          |                                                                                                 |
| Step 3 - Validation         | Sticky Note          | Explains API response validation              | None                         | None                          |                                                                                                 |
| Step 4 - Transform          | Sticky Note          | Explains data transformation logic            | None                         | None                          |                                                                                                 |
| Step 5 - Google Sheets      | Sticky Note          | Explains Google Sheets data appending          | None                         | None                          |                                                                                                 |
| Step 6 - CSV Backup         | Sticky Note          | Explains CSV backup creation                    | None                         | None                          |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node, name it `"Start Workflow Manually"`.  
   - No configuration required. This node will start the workflow on user command.

2. **Add HTTP Request Node to Fetch API Data**  
   - Add **HTTP Request** node, name it `"Fetch User Data from API"`.  
   - Set **URL** parameter to `={{ $env.BASE_URL }}` (environment variable).  
   - Use default GET method, no authentication or headers needed unless your API requires it.  
   - Connect the output of `"Start Workflow Manually"` to this node.

3. **Add If Node to Validate API Response**  
   - Add **If** node, name it `"Verify API Response Success"`.  
   - Set condition: Number → Value1: `={{$json["statusCode"]}}`, Operation: `equal`, Value2: `200`.  
   - Connect output of `"Fetch User Data from API"` to this node.

4. **Add Stop and Error Node for Failure Handling**  
   - Add **Stop and Error** node, name it `"Stop on API Failure"`.  
   - Set error message: `"API request failed — status code not 200. Workflow stopped."`  
   - Connect the **False** branch of `"Verify API Response Success"` to this node.

5. **Add Function Node for Data Transformation**  
   - Add **Function** node, name it `"Transform API Data to Name and Country"`.  
   - Use this JavaScript code:

     ```javascript
     return $json.results.map(user => {
       return {
         name: `${user.name.first} ${user.name.last}`,
         country: user.location.country
       };
     });
     ```
   - Connect the **True** branch of `"Verify API Response Success"` to this node.

6. **Add Google Sheets Node to Append Data**  
   - Add **Google Sheets** node, name it `"Append Data to Google Sheets"`.  
   - Set operation to **Append**.  
   - Set **Sheet ID** to `={{ $env.GOOGLE_SHEET_ID }}` (environment variable).  
   - Set range to `"A:B"` to append to columns A and B.  
   - Set **Value Input Mode** to **RAW** and enable **Use Path For Key Row** to match keys with headers.  
   - Configure Google Sheets OAuth2 credentials (create or select existing OAuth2 credential with required scopes).  
   - Connect output of `"Transform API Data to Name and Country"` to this node.

7. **Add Spreadsheet File Node to Create CSV Backup**  
   - Add **Spreadsheet File** node, name it `"Create CSV Backup File"`.  
   - Set operation to **To File**.  
   - Set file format to **CSV**.  
   - Set file name to `"users_backup_export"`.  
   - Connect output of `"Append Data to Google Sheets"` to this node.

8. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Add **Sticky Note** nodes with content describing each step for clarity and future maintenance.  
   - Position them near corresponding nodes for easy reference.

9. **Set Environment Variables**  
   - `BASE_URL`: API endpoint URL (e.g., `https://randomuser.me/api/?results=10`)  
   - `GOOGLE_SHEET_ID`: The ID of the Google Sheet where data will be appended  

10. **Validate Credentials**  
    - Ensure Google Sheets OAuth2 credentials are active and authorized with proper scopes to edit the target sheet.

11. **Test Workflow**  
    - Execute the workflow manually via the trigger node.  
    - Confirm data is fetched, validated, transformed, appended to Google Sheets, and CSV backup is created without errors.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                   | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow is ideal for teams needing automated user data collection from APIs with real-time Google Sheets integration and offline CSV backups.                                                                                                            | Workflow purpose                                                                                                 |
| Set environment variables `BASE_URL` and `GOOGLE_SHEET_ID` before running. Example API: `https://randomuser.me/api/?results=10`                                                                                                                               | Environment setup                                                                                                |
| Customize the transformation function to extract different data fields or add validation for missing data.                                                                                                                                                      | Data transformation customization                                                                                |
| Replace the manual trigger with Cron or Webhook nodes to automate execution on schedules or events.                                                                                                                                                             | Automation options                                                                                               |
| Google Sheets credentials require OAuth2 setup with scopes to access and modify the target sheet.                                                                                                                                                               | Credentials setup                                                                                                |
| CSV backup file can be used for downloads, emailing, or further cloud uploads if additional nodes are added (not included here).                                                                                                                               | CSV file usage                                                                                                   |
| Error handling stops the workflow if the API responds with a non-200 status, preventing bad data from being saved.                                                                                                                                               | Error handling best practice                                                                                      |
| See detailed workflow documentation sticky note inside the workflow for usage instructions and API response format example.                                                                                                                                     | Embedded workflow documentation node                                                                             |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.