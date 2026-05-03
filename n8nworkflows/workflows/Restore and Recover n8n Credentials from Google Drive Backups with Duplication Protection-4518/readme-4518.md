Restore and Recover n8n Credentials from Google Drive Backups with Duplication Protection

https://n8nworkflows.xyz/workflows/restore-and-recover-n8n-credentials-from-google-drive-backups-with-duplication-protection-4518


# Restore and Recover n8n Credentials from Google Drive Backups with Duplication Protection

### 1. Workflow Overview

This workflow, titled **"Restore and Recover n8n Credentials from Google Drive Backups with Duplication Protection"**, automates the process of restoring n8n credentials from a backup file stored on Google Drive. It is designed to extract, parse, and import credential data into an n8n instance while avoiding duplication of credentials already present. The workflow includes logical blocks for:

- **1.1 Backup Extraction:** Exporting current credentials, formatting them, and locating the backup file on Google Drive.
- **1.2 Backup Download and Parsing:** Downloading the backup JSON file from Google Drive, converting its binary data to JSON, and splitting it into individual credential entries.
- **1.3 Duplication Check and Filtering:** Iterating through each credential and checking if it should be skipped to avoid duplication.
- **1.4 Credentials Restoration:** Importing each non-duplicated credential back into n8n.
- **1.5 Flow Control and Wait:** Managing execution flow and pacing import operations.

---

### 2. Block-by-Block Analysis

#### 2.1 Backup Extraction

**Overview:**  
This block exports all existing n8n credentials decrypted and formats them as JSON. It then aggregates credential names to assist in duplication checking.

**Nodes Involved:**  
- Execute Command Get All Cridentials  
- JSON Formatting Data  
- Aggregate Cridentials  

**Node Details:**

- **Execute Command Get All Cridentials**  
  - *Type:* Execute Command  
  - *Role:* Runs shell command `npx n8n export:credentials --all --decrypted` to export decrypted credentials from n8n CLI.  
  - *Input:* Manual trigger node initiates execution.  
  - *Output:* Raw command stdout containing JSON string of credentials.  
  - *Failure cases:* Command execution failure, CLI not installed, permission issues.  
  - *Version requirements:* n8n CLI must be accessible in the runtime environment.

- **JSON Formatting Data**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Parses the raw JSON string from CLI stdout, formats it, and outputs an array of credential objects.  
  - *Key code:* Extracts JSON array using regex, parses it, maps entries to `{ json: entry }`.  
  - *Input:* Output of Execute Command node.  
  - *Output:* Parsed and structured JSON credentials.  
  - *Failure cases:* Malformed JSON in stdout, regex mismatch.  
  - *Notes:* Returns error object if JSON parsing fails.

- **Aggregate Cridentials**  
  - *Type:* Aggregate node  
  - *Role:* Aggregates the list of credential names into a single array for duplication checks later.  
  - *Input:* Output from JSON Formatting Data.  
  - *Output:* Single aggregated object containing all credential names.  
  - *Failure cases:* None significant; handles empty input gracefully.

---

#### 2.2 Backup File Retrieval and Parsing

**Overview:**  
This block locates the backup file on Google Drive, downloads it, converts the binary file contents to JSON, and prepares the data for individual processing.

**Nodes Involved:**  
- Google Drive Get Credentials File  
- Google Drive Download File  
- Convert Files To JSON  
- Split Out  
- Split Out1  

**Node Details:**

- **Google Drive Get Credentials File**  
  - *Type:* Google Drive node  
  - *Role:* Searches Google Drive for a file named `n8n_backup_credentials.json`.  
  - *Parameters:* Query string set to file name, limit 1.  
  - *Credentials:* Google Drive OAuth2.  
  - *Output:* File metadata including file ID.  
  - *Failure cases:* File not found, auth errors, API rate limits.

- **Google Drive Download File**  
  - *Type:* Google Drive node  
  - *Role:* Downloads file by ID obtained from previous node.  
  - *Parameters:* File ID from prior node output.  
  - *Credentials:* Google Drive OAuth2.  
  - *Output:* Binary file data.  
  - *Failure cases:* Download errors, invalid file ID, auth errors.

- **Convert Files To JSON**  
  - *Type:* Extract From File node  
  - *Role:* Converts binary JSON file content into JSON data under key `data`.  
  - *Parameters:* Operation set to `fromJson`, input binary property `data`.  
  - *Input:* Binary data from Google Drive download.  
  - *Output:* JSON objects extracted from file content.  
  - *Failure cases:* Invalid JSON file, file corruption.

- **Split Out**  
  - *Type:* Split Out node  
  - *Role:* Splits the array of credentials in `data` into separate items for processing.  
  - *Input:* JSON array from Convert Files To JSON.  
  - *Output:* Individual credential JSON objects.  
  - *Failure cases:* Empty or malformed input array.

- **Split Out1**  
  - *Type:* Split Out node  
  - *Role:* Additional splitting on the data field to ensure flat structure for iteration.  
  - *Input:* Output from Split Out.  
  - *Output:* Further split individual items.  
  - *Failure cases:* Similar to Split Out.

---

#### 2.3 Duplication Check and Filtering

**Overview:**  
This block iterates over each credential item from the backup, checks if the credential name already exists among the current credentials, and conditionally routes the workflow to skip or restore the credential.

**Nodes Involved:**  
- Loop Over Items  
- Check For Skipped Credentials  

**Node Details:**

- **Loop Over Items**  
  - *Type:* Split In Batches  
  - *Role:* Processes credentials one by one in batches (default batch size).  
  - *Input:* Individual credential JSON objects from Split Out1.  
  - *Output:* Single credential item per execution.  
  - *Failure cases:* Batch processing errors, empty batches.

- **Check For Skipped Credentials**  
  - *Type:* If node  
  - *Role:* Checks if the credential name is empty or already exists in the aggregated list from current credentials.  
  - *Conditions:*  
    - True if the item name is empty OR  
    - The name is contained in the current credentials array (prevents duplicates).  
  - *Input:* Current credential item from Loop Over Items and aggregated credentials from Aggregate Cridentials.  
  - *Output:*  
    - *True branch:* Loops back to skip the credential.  
    - *False branch:* Proceeds to restore the credential.  
  - *Failure cases:* Expression evaluation errors if data missing or malformed.

---

#### 2.4 Credentials Restoration

**Overview:**  
This block imports each filtered credential back into n8n using its API node, then waits briefly before processing the next credential.

**Nodes Involved:**  
- Restore N8n Credentials  
- Wait  

**Node Details:**

- **Restore N8n Credentials**  
  - *Type:* n8n API node  
  - *Role:* Imports a credential by sending its JSON data and metadata via the n8n API.  
  - *Parameters:*  
    - `data`: JSON stringified credential data from current item.  
    - `name`: Credential name.  
    - `credentialTypeName`: Credential type (e.g., API key, OAuth2).  
    - `resource`: Set to `"credential"`.  
  - *Credentials:* n8n API credentials (OAuth or API key).  
  - *Input:* Credential data from Check For Skipped Credentials false branch.  
  - *Output:* Confirmation of credential import.  
  - *Failure cases:* API auth errors, invalid credential data, duplicate import attempts.

- **Wait**  
  - *Type:* Wait node  
  - *Role:* Pauses workflow for 1 second after each credential import to avoid API throttling or rate limits.  
  - *Parameters:* Wait time set to 1 second.  
  - *Input:* Output from Restore N8n Credentials.  
  - *Output:* Triggers Loop Over Items to process the next credential.  
  - *Failure cases:* Minimal, possible delay inaccuracies.

---

#### 2.5 Trigger and Flow Control

**Overview:**  
Manages manual initiation of the workflow and orchestrates the main execution flow.

**Nodes Involved:**  
- On Click Trigger  

**Node Details:**

- **On Click Trigger**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow manually by user interaction.  
  - *Output:* Initiates the export of current credentials command.  
  - *Failure cases:* None; requires manual user start.

---

### 3. Summary Table

| Node Name                    | Node Type                 | Functional Role                            | Input Node(s)                | Output Node(s)                | Sticky Note                          |
|------------------------------|---------------------------|--------------------------------------------|-----------------------------|------------------------------|------------------------------------|
| On Click Trigger              | Manual Trigger            | Manual start of workflow                    |                             | Execute Command Get All Cridentials |                                    |
| Execute Command Get All Cridentials | Execute Command            | Export all decrypted n8n credentials       | On Click Trigger            | JSON Formatting Data          |                                    |
| JSON Formatting Data         | Code                      | Parses and formats CLI output JSON          | Execute Command Get All Cridentials | Aggregate Cridentials          |                                    |
| Aggregate Cridentials        | Aggregate                 | Aggregates credential names for duplication check | JSON Formatting Data         | Google Drive Get Credentials File |                                    |
| Google Drive Get Credentials File | Google Drive              | Finds backup JSON file on Google Drive      | Aggregate Cridentials        | Google Drive Download File    |                                    |
| Google Drive Download File   | Google Drive              | Downloads backup JSON file                   | Google Drive Get Credentials File | Convert Files To JSON          |                                    |
| Convert Files To JSON        | Extract From File         | Converts binary backup file to JSON          | Google Drive Download File   | Split Out                    |                                    |
| Split Out                   | Split Out                 | Splits JSON array into individual credential items | Convert Files To JSON        | Split Out1                   |                                    |
| Split Out1                  | Split Out                 | Further splits credential items              | Split Out                   | Loop Over Items              |                                    |
| Loop Over Items             | Split In Batches          | Processes credentials one at a time          | Split Out1                  | Check For Skipped Credentials (main) |                                    |
| Check For Skipped Credentials | If                        | Skips duplicated or empty credentials        | Loop Over Items             | Loop Over Items (true), Restore N8n Credentials (false) |                                    |
| Restore N8n Credentials     | n8n API                   | Imports credentials into n8n                  | Check For Skipped Credentials (false) | Wait                        |                                    |
| Wait                       | Wait                      | Pauses between imports to prevent throttling | Restore N8n Credentials     | Loop Over Items              |                                    |
| Sticky Note2                | Sticky Note               | Visual label: "## Import Credentials To N8n" |                             |                              |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `On Click Trigger`  
   - Type: Manual Trigger  
   - Leave default settings.

2. **Add Execute Command Node**  
   - Name: `Execute Command Get All Cridentials`  
   - Type: Execute Command  
   - Command: `npx n8n export:credentials --all --decrypted`  
   - Connect input from `On Click Trigger`.

3. **Add Code Node for JSON Formatting**  
   - Name: `JSON Formatting Data`  
   - Type: Code (JavaScript)  
   - Code: Implement parsing logic to extract JSON array from stdout string, parse it, map each item to `{ json: entry }`. Handle errors gracefully.  
   - Connect input from `Execute Command Get All Cridentials`.

4. **Add Aggregate Node**  
   - Name: `Aggregate Cridentials`  
   - Type: Aggregate  
   - Aggregate field: `name` of credentials  
   - Connect input from `JSON Formatting Data`.

5. **Add Google Drive File Search Node**  
   - Name: `Google Drive Get Credentials File`  
   - Type: Google Drive  
   - Resource: `fileFolder`  
   - Query String: `n8n_backup_credentials.json`  
   - Limit: 1  
   - Credentials: Google Drive OAuth2 account with access to the backup file  
   - Connect input from `Aggregate Cridentials`.

6. **Add Google Drive Download Node**  
   - Name: `Google Drive Download File`  
   - Type: Google Drive  
   - Operation: `download`  
   - File ID: Use expression to reference ID from previous node  
   - Credentials: Same as above  
   - Connect input from `Google Drive Get Credentials File`.

7. **Add Extract From File Node**  
   - Name: `Convert Files To JSON`  
   - Type: Extract From File  
   - Operation: `fromJson`  
   - Binary Property: `data`  
   - Destination Key: `data`  
   - Connect input from `Google Drive Download File`.

8. **Add Split Out Node (First)**  
   - Name: `Split Out`  
   - Type: Split Out  
   - Field to Split Out: `data`  
   - Connect input from `Convert Files To JSON`.

9. **Add Split Out Node (Second)**  
   - Name: `Split Out1`  
   - Type: Split Out  
   - Field to Split Out: `data`  
   - Connect input from `Split Out`.

10. **Add Split In Batches Node**  
    - Name: `Loop Over Items`  
    - Type: Split In Batches  
    - Leave default batch size (typically 1)  
    - Connect input from `Split Out1`.

11. **Add If Node for Duplication Check**  
    - Name: `Check For Skipped Credentials`  
    - Type: If  
    - Conditions:  
      - True if item `name` is empty OR  
      - The name exists in the aggregated credentials array from node `Aggregate Cridentials`  
    - Connect input from `Loop Over Items`.

12. **Loop True Branch**  
    - Connect back to `Loop Over Items` input to skip processing.

13. **Loop False Branch**  
    - Connect to a new n8n API node for restoring credentials.

14. **Add n8n API Node**  
    - Name: `Restore N8n Credentials`  
    - Type: n8n API  
    - Resource: `credential`  
    - Parameters:  
      - `name`: Expression referencing current item name  
      - `data`: JSON stringified current item data  
      - `credentialTypeName`: Expression referencing current item type  
    - Credentials: n8n API credential with sufficient permissions  
    - Connect input from `Check For Skipped Credentials` false branch.

15. **Add Wait Node**  
    - Name: `Wait`  
    - Type: Wait  
    - Duration: 1 second  
    - Connect input from `Restore N8n Credentials`.

16. **Connect Wait Node output to `Loop Over Items` input**  
    - This continues processing remaining credentials.

17. **Add Sticky Note (Optional)**  
    - Name: `Sticky Note2`  
    - Content: `## Import Credentials To N8n`  
    - Place visually near the restoration nodes for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow uses n8n API node for credential import; ensure API credentials have required scopes.   | n8n API documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n/                         |
| Google Drive OAuth2 credentials must have access to the backup file location.                     | Google Drive API setup: https://developers.google.com/drive/api/v3/about-auth                             |
| The CLI command `npx n8n export:credentials --all --decrypted` requires the n8n CLI installed in the environment where this runs. | n8n CLI docs: https://docs.n8n.io/reference/cli/                                                         |
| Wait node with 1-second delay helps prevent API rate limiting errors during bulk imports.        | Recommended for workflows interacting with APIs that enforce rate limits.                                |
| The workflow assumes the backup file is named `n8n_backup_credentials.json` in Google Drive.     | Adjust the query string if the backup file has a different name or location.                             |
| Duplication protection relies on matching credential names; consider credential name uniqueness. | Avoid duplicate names in n8n credentials to prevent import skipping errors.                              |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This treatment strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.