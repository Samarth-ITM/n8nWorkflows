Gmail to Google Drive Email Export Workflow

https://n8nworkflows.xyz/workflows/gmail-to-google-drive-email-export-workflow-4236


# Gmail to Google Drive Email Export Workflow

### 1. Workflow Overview

This workflow, titled **"Gmail to Google Drive Email Export Workflow"**, automates the process of exporting emails from a specific Gmail sender and saving them as a CSV file in Google Drive. It is designed for use cases such as personal email archiving, audit logging, and team report aggregation.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger initiates the workflow and fetches emails from Gmail filtered by sender.
- **1.2 Data Parsing and Transformation:** Extracts relevant email fields and converts the timestamp to a specified timezone format.
- **1.3 File Conversion:** Converts the processed data into a CSV file format.
- **1.4 File Upload:** Uploads the generated CSV file to a specified folder in Google Drive.
- **1.5 Workflow Termination:** Marks the end of the workflow execution.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Starts workflow execution manually and retrieves all emails from Gmail filtered by a specific sender.
- **Nodes Involved:**  
  - Start Workflow  
  - Gmail  

- **Node Details:**

  - **Start Workflow**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow  
    - Configuration: Default manual trigger without additional parameters  
    - Inputs: None  
    - Outputs: Connects to Gmail node  
    - Failure Types: None expected  
    - Notes: Provides a manual way to initiate email fetching

  - **Gmail**  
    - Type: Gmail node (OAuth2 authenticated)  
    - Role: Fetches all emails filtered by sender email address  
    - Configuration:  
      - Operation: Get all emails  
      - Filters: Sender email set to "akhilgadiraju@gmail.com" (modifiable)  
      - Return all: true (fetches all matching emails)  
      - Credentials: Gmail OAuth2 linked to user "Akhil"  
    - Inputs: Triggered by Start Workflow  
    - Outputs: Passes all retrieved emails to "Parse Data" node  
    - Failure Types: Authentication errors, API rate limits, no emails found  
    - Sticky Note:  
      - "Change Sender Email: Update the `sender` field in the Gmail node."  

#### 1.2 Data Parsing and Transformation

- **Overview:** Extracts key fields from each email and converts the email timestamp into a specified localized string format.
- **Nodes Involved:**  
  - Parse Data  
  - Convert Time Field  

- **Node Details:**

  - **Parse Data**  
    - Type: Set node  
    - Role: Extracts and assigns relevant email fields into a simplified structure  
    - Configuration:  
      - Assigns fields:  
        - id ← email id  
        - subject ← email subject  
        - message ← email text content  
        - time ← email date as ISO string  
    - Inputs: Emails from Gmail node  
    - Outputs: Passes structured JSON to Convert Time Field  
    - Failure Types: Expression errors if fields are missing or malformed  
    - Sticky Note:  
      - "Add More Email Fields: Modify the Set node to include more fields."  

  - **Convert Time Field**  
    - Type: Code node (JavaScript)  
    - Role: Converts ISO timestamp in `time` field to localized string in Asia/Kolkata timezone  
    - Configuration:  
      - Uses JavaScript Date object with toLocaleString specifying:  
        - Timezone: 'Asia/Kolkata' (modifiable)  
        - Format: long month, numeric day/year, 12-hour time with minutes  
      - Returns updated JSON with localized `time` field  
    - Inputs: Parsed data from Set node  
    - Outputs: Passes updated JSON to Convert to File node  
    - Failure Types: Invalid date formats, code execution errors  
    - Sticky Note:  
      - "Change Time Zone: Edit timeZone in the Code node."  

#### 1.3 File Conversion

- **Overview:** Converts the JSON data stream into a CSV file format for export.
- **Nodes Involved:**  
  - Convert to File  

- **Node Details:**

  - **Convert to File**  
    - Type: ConvertToFile node  
    - Role: Transforms JSON data into a downloadable file (CSV)  
    - Configuration:  
      - Default format assumed as CSV (no explicit format override)  
      - No additional options set  
    - Inputs: Localized data from Convert Time Field  
    - Outputs: Passes file binary data to Google Drive node  
    - Failure Types: Conversion errors if data structure incompatible  
    - Sticky Note:  
      - "Change File Format: Use a different format in the Convert to File node."  

#### 1.4 File Upload

- **Overview:** Uploads the generated CSV file to a specified folder in Google Drive.
- **Nodes Involved:**  
  - Google Drive  

- **Node Details:**

  - **Google Drive**  
    - Type: Google Drive node (OAuth2 authenticated)  
    - Role: Uploads the CSV file to Google Drive root folder  
    - Configuration:  
      - File name: Dynamic, formatted as current timestamp + "_n8n_export.csv"  
      - Drive: "My Drive" selected  
      - Folder: Root folder ("root")  
      - Credentials: Google Drive OAuth2 linked to user "Akhil"  
    - Inputs: File data from Convert to File node  
    - Outputs: Passes to End Workflow node  
    - Failure Types: Authentication errors, quota exceeded, invalid folder ID  
    - Sticky Note:  
      - "Rename Output File: Adjust the name in the Google Drive node"  
      - "Change Upload Folder: Set a different folderId in the Google Drive node."  

#### 1.5 Workflow Termination

- **Overview:** Marks the end of the workflow run.
- **Nodes Involved:**  
  - End Workflow  

- **Node Details:**

  - **End Workflow**  
    - Type: NoOp node  
    - Role: Terminates workflow cleanly  
    - Configuration: None  
    - Inputs: From Google Drive node  
    - Outputs: None  
    - Failure Types: None expected  

---

### 3. Summary Table

| Node Name         | Node Type               | Functional Role               | Input Node(s)    | Output Node(s)    | Sticky Note                                                                                 |
|-------------------|-------------------------|------------------------------|------------------|-------------------|---------------------------------------------------------------------------------------------|
| Start Workflow    | Manual Trigger          | Workflow entry point          | None             | Gmail             |                                                                                             |
| Gmail             | Gmail                   | Fetch emails filtered by sender | Start Workflow   | Parse Data        | Change Sender Email: Update the `sender` field in the Gmail node.                           |
| Parse Data        | Set                     | Extract key email fields      | Gmail            | Convert Time Field | Add More Email Fields: Modify the Set node to include more fields.                          |
| Convert Time Field | Code                    | Convert ISO time to localized string | Parse Data       | Convert to File   | Change Time Zone: Edit timeZone in the Code node.                                          |
| Convert to File   | ConvertToFile           | Convert JSON data to CSV file | Convert Time Field| Google Drive      | Change File Format: Use a different format in the Convert to File node.                    |
| Google Drive      | Google Drive            | Upload CSV file to Drive      | Convert to File  | End Workflow      | Rename Output File: Adjust the name in the Google Drive node. Change Upload Folder: Set a different folderId in the Google Drive node. |
| End Workflow      | NoOp                    | Workflow termination          | Google Drive     | None              |                                                                                             |
| Sticky Note       | Sticky Note             | Notes and instructions        | None             | None              | See individual sticky notes in respective nodes.                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Purpose: Start workflow manually  
   - No parameters needed  

2. **Create Gmail Node**  
   - Node Type: Gmail  
   - Set operation to "Get All" emails  
   - Add filter: sender equals "akhilgadiraju@gmail.com" (modify as needed)  
   - Enable "Return All" to true  
   - Configure Gmail OAuth2 credentials (set up Gmail OAuth2 with appropriate scopes)  
   - Connect Manual Trigger output to Gmail input  

3. **Create Set Node ("Parse Data")**  
   - Node Type: Set  
   - Add fields with expressions:  
     - id = `{{$json["id"]}}`  
     - subject = `{{$json["subject"]}}`  
     - message = `{{$json["text"]}}`  
     - time = `{{$json["date"]}}`  
   - Connect Gmail output to Set node input  

4. **Create Code Node ("Convert Time Field")**  
   - Node Type: Code (JavaScript)  
   - Paste JavaScript code:  
     ```javascript
     return $input.all().map(item => {
       const isoTime = item.json.time;
       const date = new Date(isoTime).toLocaleString('en-US', {
         timeZone: 'Asia/Kolkata',  // Change timezone as needed
         year: 'numeric',
         month: 'long',
         day: 'numeric',
         hour: 'numeric',
         minute: '2-digit',
         hour12: true
       });
       return {
         json: {
           ...item.json,
           time: date
         }
       };
     });
     ```  
   - Connect Set node output to Code node input  

5. **Create ConvertToFile Node ("Convert to File")**  
   - Node Type: ConvertToFile  
   - Default output format CSV (adjust if needed)  
   - Connect Code node output to ConvertToFile input  

6. **Create Google Drive Node**  
   - Node Type: Google Drive  
   - Operation: Upload file  
   - File name: `={{ $now + "_n8n_export.csv" }}`  
   - Drive: Select "My Drive" or another drive  
   - Folder: Select "root" or desired folder  
   - Credentials: Configure Google Drive OAuth2 credentials  
   - Connect ConvertToFile node output to Google Drive node input  

7. **Create NoOp Node ("End Workflow")**  
   - Node Type: NoOp  
   - Connect Google Drive node output to NoOp node input  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                               |
|-----------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Use Cases: Personal email archiving, audit logs, team reports aggregation                      | Sticky Note near Start Workflow node                           |
| Modify sender email in Gmail node to change data source                                       | Sticky Note near Gmail node                                    |
| Adjust time zone in Code node for correct localized timestamps                                | Sticky Note near Convert Time Field node                       |
| Change output file format in Convert to File node if CSV is not desired                        | Sticky Note near Convert to File node                          |
| Rename output file and change upload folder in Google Drive node                              | Sticky Note near Google Drive node                             |

---

**Disclaimer:** The provided text originates exclusively from an n8n automated workflow. This processing strictly adheres to existing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.