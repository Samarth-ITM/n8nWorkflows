Automated n8n Credential Backups to Google Drive with Scheduled Execution

https://n8nworkflows.xyz/workflows/automated-n8n-credential-backups-to-google-drive-with-scheduled-execution-4517


# Automated n8n Credential Backups to Google Drive with Scheduled Execution

### 1. Workflow Overview

This workflow automates the backup of all n8n credentials by exporting them, formatting the data, and uploading the resulting JSON file to a designated Google Drive folder. It supports two trigger modes: manual execution via a button and scheduled execution at defined intervals.

The logical flow is divided into these blocks:

- **1.1 Trigger Reception:** Receives initiation signals, either manual trigger or scheduled trigger.
- **1.2 Credentials Export:** Executes a command to export all credentials from n8n in decrypted JSON form.
- **1.3 JSON Processing:** Extracts, validates, and formats the JSON data from the command output.
- **1.4 Data Aggregation & Conversion:** Aggregates all parsed credential entries and converts them into a JSON file format.
- **1.5 Google Drive Upload:** Uploads the JSON file to a specified Google Drive folder using OAuth2 credentials.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Reception

- **Overview:** This block allows the workflow to be started either manually or on a schedule.
- **Nodes Involved:** 
  - On Click Trigger
  - Schedule Trigger

- **Node Details:**

  - **On Click Trigger**
    - Type: Manual Trigger
    - Role: Allows user to manually start the workflow via UI button.
    - Configuration: No parameters; default manual trigger.
    - Inputs: None
    - Outputs: Connects to "Execute Command Get All Cridentials"
    - Failure Modes: None typical; manual initiation.
    
  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Automatically triggers the workflow at scheduled intervals.
    - Configuration: Interval set with default placeholder (empty scheduling expression in JSON, presumably to be set by user).
    - Inputs: None
    - Outputs: Connects to "Execute Command Get All Cridentials"
    - Failure Modes: Misconfigured or missing interval can prevent automatic runs.

#### 2.2 Credentials Export

- **Overview:** Executes a shell command to export all n8n credentials in decrypted JSON format.
- **Nodes Involved:** 
  - Execute Command Get All Cridentials

- **Node Details:**

  - **Execute Command Get All Cridentials**
    - Type: Execute Command
    - Role: Runs `npx n8n export:credentials --all --decrypted` to get credentials.
    - Configuration: Command is fixed as above.
    - Inputs: Receives trigger from manual or schedule trigger.
    - Outputs: Produces command stdout with JSON string.
    - Failure Modes: Command execution failure, permission errors, or environment issues.
    - Notes: Requires n8n environment with npx and CLI commands accessible.

#### 2.3 JSON Processing

- **Overview:** Parses the raw command output, extracts JSON data, validates and beautifies it for downstream use.
- **Nodes Involved:**
  - JSON Formatting Data

- **Node Details:**

  - **JSON Formatting Data**
    - Type: Code (JavaScript)
    - Role: Extracts JSON array string from stdout, parses, and returns it as an array of JSON objects.
    - Configuration:
      - Uses regex `/\[{.*}\]/s` to extract JSON array from command stdout.
      - Parses JSON string safely with error handling.
      - Returns an error message as JSON if extraction or parsing fails.
      - Outputs each credential as a separate JSON item.
    - Inputs: Receives stdout string from command node.
    - Outputs: Array of JSON objects representing credentials.
    - Failure Modes:
      - No JSON found in stdout.
      - Invalid JSON format.
      - Unexpected command output format changes.

#### 2.4 Data Aggregation & Conversion

- **Overview:** Aggregates all credential entries into a single dataset and converts it into a JSON file binary format.
- **Nodes Involved:**
  - Aggregate Cridentials
  - Convert To File

- **Node Details:**

  - **Aggregate Cridentials**
    - Type: Aggregate
    - Role: Combines all incoming JSON items into one consolidated data array.
    - Configuration: Uses "aggregateAllItemData" to merge all items.
    - Inputs: Multiple JSON objects from previous node.
    - Outputs: Single aggregated JSON object with an array of credentials.
    - Failure Modes: Empty input, malformed JSON items.

  - **Convert To File**
    - Type: Convert To File
    - Role: Converts aggregated JSON data into a JSON file binary property named `data`.
    - Configuration:
      - Operation: toJson
      - Binary Property Name: data
    - Inputs: Aggregated JSON from previous node.
    - Outputs: Binary file ready for upload.
    - Failure Modes: Conversion failure if input data is invalid.

#### 2.5 Google Drive Upload

- **Overview:** Uploads the JSON file containing credentials backup to a specific Google Drive folder using OAuth2 authentication.
- **Nodes Involved:**
  - Google Drive Upload File

- **Node Details:**

  - **Google Drive Upload File**
    - Type: Google Drive
    - Role: Uploads a file named `n8n_backup_credentials.json` to a configured folder.
    - Configuration:
      - File Name: Fixed string `n8n_backup_credentials.json`
      - Drive: "My Drive" selected from OAuth2 authenticated account.
      - Folder ID: `1p447S9MWYcRpA6dmfDe-Kdc3-d8L2Lzr` (Tung Backup Credential folder)
      - Input Data Field: Uses binary file property `data`.
      - Credentials: Google Drive OAuth2 API named "AI Auto Google Drive account."
    - Inputs: Binary file from "Convert To File".
    - Outputs: Upload confirmation data.
    - Failure Modes:
      - Authentication errors (expired/invalid OAuth2 tokens).
      - Folder ID incorrect or access denied.
      - File size or format issues.
      - Network or API rate limit errors.

---

### 3. Summary Table

| Node Name                      | Node Type             | Functional Role                    | Input Node(s)                     | Output Node(s)                 | Sticky Note                                  |
|-------------------------------|-----------------------|----------------------------------|----------------------------------|-------------------------------|----------------------------------------------|
| On Click Trigger               | Manual Trigger        | Manual workflow start             | None                             | Execute Command Get All Cridentials |                                              |
| Schedule Trigger               | Schedule Trigger      | Scheduled workflow start          | None                             | Execute Command Get All Cridentials |                                              |
| Execute Command Get All Cridentials | Execute Command      | Export all n8n credentials        | On Click Trigger, Schedule Trigger | JSON Formatting Data            | ## Export All Credentials From N8n           |
| JSON Formatting Data           | Code                  | Extract and parse JSON output     | Execute Command Get All Cridentials | Aggregate Cridentials          |                                              |
| Aggregate Cridentials          | Aggregate             | Aggregate all credential JSON     | JSON Formatting Data              | Convert To File                |                                              |
| Convert To File                | Convert To File       | Convert JSON array to JSON file   | Aggregate Cridentials             | Google Drive Upload File       |                                              |
| Google Drive Upload File       | Google Drive          | Upload JSON file to Google Drive  | Convert To File                   | None                          | ## Google Drive Folder                        |
| Sticky Note2                  | Sticky Note           | Comment: Export All Credentials   | None                             | None                          | ## Export All Credentials From N8            |
| Sticky Note                   | Sticky Note           | Comment: Google Drive Folder      | None                             | None                          | ## Google Drive Folder                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**
   - Add a **Manual Trigger** node named `On Click Trigger` for manual execution.
   - Add a **Schedule Trigger** node named `Schedule Trigger`. Configure its interval according to your backup frequency (e.g., daily, weekly).

2. **Create Export Command Node**
   - Add an **Execute Command** node named `Execute Command Get All Cridentials`.
   - Set command to: `npx n8n export:credentials --all --decrypted`
   - Connect both `On Click Trigger` and `Schedule Trigger` nodes to this node.

3. **Create JSON Parsing Node**
   - Add a **Code** node named `JSON Formatting Data`.
   - Use JavaScript to:
     - Extract JSON array string from the `stdout` field using regex `/\[{.*}\]/s`.
     - Parse the JSON string safely; on error, return a JSON object with an error message.
     - Return the parsed array with each element as an individual JSON item.
   - Connect output of `Execute Command Get All Cridentials` to this node.

4. **Create Aggregate Node**
   - Add an **Aggregate** node named `Aggregate Cridentials`.
   - Set aggregation to `aggregateAllItemData` to combine all JSON items into one array.
   - Connect output of `JSON Formatting Data` to this node.

5. **Create Convert To File Node**
   - Add a **Convert To File** node named `Convert To File`.
   - Set operation to `toJson`.
   - Set binary property name to `data`.
   - Connect output of `Aggregate Cridentials` to this node.

6. **Create Google Drive Upload Node**
   - Add a **Google Drive** node named `Google Drive Upload File`.
   - Set file name to `n8n_backup_credentials.json`.
   - Select Drive as `My Drive`.
   - Choose folder ID corresponding to your Google Drive folder for backups.
   - Set input data field to `data` (the binary property from previous node).
   - Configure Google Drive OAuth2 credentials with appropriate permissions.
   - Connect output of `Convert To File` to this node.

7. **Add Sticky Notes (Optional)**
   - Add sticky notes near relevant nodes for documentation:
     - Near `Execute Command Get All Cridentials`: "## Export All Credentials From N8n"
     - Near `Google Drive Upload File`: "## Google Drive Folder"

8. **Activate and Test**
   - Validate OAuth2 credentials for Google Drive.
   - Test manual trigger to confirm backup is created and uploaded.
   - Configure schedule trigger timing as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                             |
|-----------------------------------------------------------------------------------------------|--------------------------------------------|
| This workflow requires the n8n environment to have CLI access to execute `npx n8n export:credentials` | n8n CLI documentation                     |
| Google Drive OAuth2 credentials must have write access to the target folder                    | Google Drive API OAuth2 setup guide        |
| The `Schedule Trigger` node’s interval must be configured explicitly for scheduled backups     | n8n Schedule Trigger documentation         |
| Ensure the Google Drive folder ID is correct and accessible by the OAuth2 authenticated account | Google Drive folder sharing and permissions|

---

_Disclaimer: The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public._