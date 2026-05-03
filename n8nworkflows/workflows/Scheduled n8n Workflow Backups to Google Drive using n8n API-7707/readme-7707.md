Scheduled n8n Workflow Backups to Google Drive using n8n API

https://n8nworkflows.xyz/workflows/scheduled-n8n-workflow-backups-to-google-drive-using-n8n-api-7707


# Scheduled n8n Workflow Backups to Google Drive using n8n API

---

### 1. Workflow Overview

This n8n workflow automates scheduled backups of all existing n8n workflows by exporting their full definitions and storing them as JSON files in a specified Google Drive folder. It is designed for users who want to maintain off-platform backups of their workflows regularly, ensuring data resilience and version history.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger**: Initiates the workflow execution automatically on a defined interval (weekly).
- **1.2 Fetch Workflows List**: Retrieves the complete list of all workflows available in the current n8n instance.
- **1.3 Retrieve Individual Workflow Definitions**: For each workflow ID fetched, fetches the full workflow data.
- **1.4 Convert Workflow JSON to File**: Converts the JSON data of each workflow into a binary file format with a sanitized filename.
- **1.5 Upload Backup Files to Google Drive**: Uploads the converted workflow JSON files to a predetermined Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  This block triggers the entire backup process automatically on a schedule without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger1

- **Node Details:**  
  - **Node Name:** Schedule Trigger1  
  - **Type:** Schedule Trigger  
  - **Role:** Initiates workflow execution according to a time interval.  
  - **Configuration:**  
    - Interval set to run weekly (default weekly interval, exact timing configurable in n8n UI).  
  - **Connections:**  
    - Output connected to "Get many workflows1" node.  
  - **Edge Cases / Failure Types:**  
    - Misconfiguration of schedule could lead to unintended trigger frequency.  
    - Workflow runtime errors after trigger do not affect scheduling but cause backup failure.  
  - **Version Requirements:** Version 1.2 used (minor version differences mostly affect UI features).

#### 2.2 Fetch Workflows List

- **Overview:**  
  Queries the n8n API to retrieve metadata for all workflows currently defined in the instance. This is the foundational step to know which workflows to back up.

- **Nodes Involved:**  
  - Get many workflows1

- **Node Details:**  
  - **Node Name:** Get many workflows1  
  - **Type:** n8n Node (n8n API)  
  - **Role:** Retrieves a list of all workflows.  
  - **Configuration:**  
    - Operation: List (default fetch all workflows without filters).  
    - No filters applied, fetching all workflows.  
  - **Connections:**  
    - Input from "Schedule Trigger1" output.  
    - Output to "Get a workflow1" node, passing each workflow’s ID downstream.  
  - **Edge Cases / Failure Types:**  
    - API rate limits could cause failures if the instance has many workflows.  
    - Empty workflows list results in no further processing (no backups).  
    - API auth errors if n8n credentials are invalid.  
  - **Version Requirements:** Version 1 used.

#### 2.3 Retrieve Individual Workflow Definitions

- **Overview:**  
  For each workflow ID retrieved, fetches the complete workflow definition including nodes, parameters, and metadata.

- **Nodes Involved:**  
  - Get a workflow1

- **Node Details:**  
  - **Node Name:** Get a workflow1  
  - **Type:** n8n Node (n8n API)  
  - **Role:** Downloads a specific workflow’s full data by ID.  
  - **Configuration:**  
    - Operation: Get workflow by ID.  
    - Workflow ID dynamically set from the JSON item’s `id` field passed from "Get many workflows1".  
  - **Connections:**  
    - Input from "Get many workflows1".  
    - Output to "Move Binary Data (JSON → File)".  
  - **Edge Cases / Failure Types:**  
    - Invalid or deleted workflow IDs may cause 404 errors.  
    - API auth errors.  
    - Timeout if workflow data is large or API slow.  
    - Expression failures if `id` field is missing or malformed.  
  - **Version Requirements:** Version 1 used.

#### 2.4 Convert Workflow JSON to File

- **Overview:**  
  Converts the retrieved workflow JSON data into a binary file suitable for upload, generating a safe filename using the workflow's name and ID.

- **Nodes Involved:**  
  - Move Binary Data (JSON → File)

- **Node Details:**  
  - **Node Name:** Move Binary Data (JSON → File)  
  - **Type:** Move Binary Data  
  - **Role:** Converts JSON workflow data into a binary file attachment to enable file uploads.  
  - **Configuration:**  
    - Mode: jsonToBinary conversion.  
    - File name generated dynamically using workflow name and ID, with unsafe characters removed via regex `[^a-zA-Z0-9_\-]`. Filename formatted as: `<workflowName>_<workflowID>.json`.  
  - **Connections:**  
    - Input from "Get a workflow1".  
    - Output to "Upload file1".  
  - **Edge Cases / Failure Types:**  
    - Missing or invalid workflow name/ID could cause filename issues.  
    - Large workflows may impact processing time or memory.  
  - **Version Requirements:** Version 1 used.

#### 2.5 Upload Backup Files to Google Drive

- **Overview:**  
  Uploads each converted workflow JSON file into a specified Google Drive folder for centralized storage and easy access.

- **Nodes Involved:**  
  - Upload file1

- **Node Details:**  
  - **Node Name:** Upload file1  
  - **Type:** Google Drive node  
  - **Role:** Uploads the binary file to Google Drive under the specified folder.  
  - **Configuration:**  
    - File name: Set dynamically from previous node’s workflow name.  
    - Drive: Uses "My Drive".  
    - Folder ID: User must specify the target Google Drive folder ID (`YOUR_FOLDER_ID`).  
    - Input data field: Uses the binary file data from the previous node.  
  - **Connections:**  
    - Input from "Move Binary Data (JSON → File)".  
    - No further outputs (terminal upload node).  
  - **Edge Cases / Failure Types:**  
    - Invalid or missing Google Drive credentials cause upload failures.  
    - Incorrect folder ID leads to upload errors or misplaced files.  
    - API rate limits or quota exceeded errors.  
    - Filename conflicts in Drive could cause overwrites unless file versioning is enabled.  
  - **Version Requirements:** Version 3 used (due to Google Drive node updates).  
  - **Credential Requirements:** Requires configured Google Drive OAuth2 credentials with write permissions.

---

### 3. Summary Table

| Node Name                     | Node Type                 | Functional Role                    | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                                      |
|-------------------------------|---------------------------|----------------------------------|--------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger1              | Schedule Trigger          | Initiates scheduled workflow run |                          | Get many workflows1        | ⏰ Scheduled trigger<br>Allows the flow to run automatically according to a defined interval.                    |
| Get many workflows1            | n8n API (n8n node)        | Fetch all workflows list          | Schedule Trigger1        | Get a workflow1            | 📥 Fetch all workflows<br>Queries the n8n API and retrieves the complete list of existing workflows.            |
| Get a workflow1                | n8n API (n8n node)        | Fetch single workflow definition  | Get many workflows1      | Move Binary Data (JSON → File) | 📂 Fetch a specific workflow<br>Downloads full data of each workflow one by one.                                |
| Move Binary Data (JSON → File) | Move Binary Data          | Convert JSON to binary file       | Get a workflow1          | Upload file1               | 📦 Convert to file<br>Transforms each downloaded workflow into a JSON file with safe filenames.                 |
| Upload file1                  | Google Drive              | Upload backup file to Google Drive | Move Binary Data (JSON → File) |                            | ☁️ Upload to Google Drive<br>Saves files into the configured Google Drive folder for centralized backups.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Name: `Schedule Trigger1`
   - Type: Schedule Trigger
   - Configure to run on a weekly interval (select weekly schedule, set desired weekday/time).
   - No input connections.

2. **Create an n8n node to list workflows:**
   - Name: `Get many workflows1`
   - Type: n8n (core node to call n8n API)
   - Operation: `List`
   - Filters: None (fetch all workflows)
   - Connect input from `Schedule Trigger1` output (main).
  
3. **Create an n8n node to get individual workflows:**
   - Name: `Get a workflow1`
   - Type: n8n (core node)
   - Operation: `Get`
   - Workflow ID: Set expression `{{$json["id"]}}` to dynamically get the workflow ID from the previous node.
   - Connect input from `Get many workflows1` output (main).

4. **Create a Move Binary Data node to convert JSON to file:**
   - Name: `Move Binary Data (JSON → File)`
   - Type: Move Binary Data
   - Mode: `jsonToBinary`
   - File Name: Use expression to generate a safe filename:  
     `={{ $json.name.replace(/[^a-zA-Z0-9_\-]/g, '') + '_' + $json.id + '.json' }}`
   - Connect input from `Get a workflow1` output (main).

5. **Create a Google Drive upload node:**
   - Name: `Upload file1`
   - Type: Google Drive
   - Operation: Upload File
   - Drive: Select "My Drive"
   - Folder ID: Enter your target Google Drive folder ID (replace placeholder `YOUR_FOLDER_ID` with actual ID)
   - File Name: Use expression `={{ $('Get a workflow1').item.json.name }}`
   - Input Data Field Name: Use the binary data from the previous node (`data` by default)
   - Connect input from `Move Binary Data (JSON → File)` output (main).

6. **Credential Setup:**
   - For the Google Drive node, configure OAuth2 credentials with write access to the target Drive.
   - For n8n internal API calls, ensure API authentication is properly configured or the node runs with appropriate permissions.

7. **Final Connections:**
   - `Schedule Trigger1` → `Get many workflows1`
   - `Get many workflows1` → `Get a workflow1`
   - `Get a workflow1` → `Move Binary Data (JSON → File)`
   - `Move Binary Data (JSON → File)` → `Upload file1`

8. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                             | Context or Link                                                               |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| The Google Drive folder ID must be replaced with your actual folder ID to ensure uploads go to the correct location.                                     | [Google Drive Folder ID Docs](https://support.google.com/drive/answer/2423485) |
| Filenames are sanitized to avoid invalid characters that may cause upload issues or overwrite risks.                                                     | Regex used in filename expression: `/[^a-zA-Z0-9_\-]/g`                        |
| This workflow requires n8n instance API permissions to list and get workflows. Ensure API tokens or credentials are properly configured.                 | n8n API Authentication documentation                                          |
| Scheduled trigger interval can be customized as needed (daily, monthly) via the Schedule Trigger node settings.                                          | n8n Schedule Trigger node docs                                                |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---