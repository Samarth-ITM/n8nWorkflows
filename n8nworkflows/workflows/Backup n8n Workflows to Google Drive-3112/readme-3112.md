Backup n8n Workflows to Google Drive

https://n8nworkflows.xyz/workflows/backup-n8n-workflows-to-google-drive-3112


# Backup n8n Workflows to Google Drive

### 1. Workflow Overview

This workflow automates the daily backup of all n8n workflows by exporting their configurations and storing them securely in a specified Google Drive folder. It targets n8n users who want to safeguard their workflow definitions and reduce the risk of data loss through automated, scheduled backups.

The workflow is logically divided into the following blocks:

- **1.1 Triggering the Backup**: Initiates the backup process either manually or automatically on a daily schedule.
- **1.2 Retrieving Workflow Data**: Fetches the list of all workflows and then retrieves detailed data for each workflow.
- **1.3 Data Transformation**: Converts the retrieved JSON workflow data into a format suitable for file upload.
- **1.4 Upload to Google Drive**: Uploads the transformed backup file to a designated Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggering the Backup

**Overview:**  
This block starts the backup process either manually via a trigger node or automatically at 2:30 AM daily using a cron schedule.

**Nodes Involved:**  
- On clicking 'execute' (Manual Trigger)  
- Run Daily at 2:30am (Cron Trigger)  
- Get Workflow List (HTTP Request)

**Node Details:**

- **On clicking 'execute'**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual initiation of the backup workflow.  
  - *Configuration:* Default manual trigger with no parameters.  
  - *Inputs:* None  
  - *Outputs:* Connected to "Get Workflow List" node.  
  - *Edge Cases:* User must manually trigger; no automatic scheduling.  
  - *Notes:* Useful for testing or on-demand backups.

- **Run Daily at 2:30am**  
  - *Type:* Cron Trigger  
  - *Role:* Automatically triggers the workflow every day at 2:30 AM.  
  - *Configuration:* Cron expression set for hour=2, minute=30.  
  - *Inputs:* None  
  - *Outputs:* Connected to "Get Workflow List" node.  
  - *Edge Cases:* Timezone differences may affect execution time; ensure n8n server timezone is correct.

- **Get Workflow List**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves the list of all workflows from the local n8n instance via REST API.  
  - *Configuration:*  
    - URL: `http://localhost:5678/rest/workflows`  
    - Authentication: Basic Auth with configured credentials ("n8n Creds").  
  - *Inputs:* Trigger nodes ("On clicking 'execute'" and "Run Daily at 2:30am").  
  - *Outputs:* Connected to "Map" node.  
  - *Edge Cases:*  
    - Authentication failure if credentials are incorrect.  
    - Connection failure if n8n API is unreachable.  
    - Large number of workflows may cause delays or timeouts.

---

#### 1.2 Retrieving Workflow Data

**Overview:**  
This block processes the list of workflows and fetches detailed data for each workflow individually.

**Nodes Involved:**  
- Map (Function)  
- Get Workflow (HTTP Request)  
- Merge (Merge Node)

**Node Details:**

- **Map**  
  - *Type:* Function  
  - *Role:* Transforms the array of workflows into individual items for further processing.  
  - *Configuration:* JavaScript code that maps each workflow object into a separate item:  
    ```js
    return items[0].json.data.map(item => {
      return {json: item}
    });
    ```  
  - *Inputs:* Output from "Get Workflow List".  
  - *Outputs:* Connected to "Get Workflow".  
  - *Edge Cases:* Assumes the response JSON contains a `data` array; failure if structure changes.

- **Get Workflow**  
  - *Type:* HTTP Request  
  - *Role:* Fetches detailed JSON data for each workflow by ID.  
  - *Configuration:*  
    - URL: `http://localhost:5678/rest/workflows/{{$node["Map"].data["id"]}}` (dynamic URL using workflow ID)  
    - Authentication: Basic Auth ("n8n Creds")  
  - *Inputs:* From "Map" node (one per workflow).  
  - *Outputs:* Connected to "Merge" node (second input).  
  - *Edge Cases:*  
    - Authentication failure.  
    - Workflow ID may be invalid or deleted between calls.  
    - API rate limits or timeouts.

- **Merge**  
  - *Type:* Merge  
  - *Role:* Combines the original workflow list and detailed workflow data into a single stream.  
  - *Configuration:* Mode set to "mergeByIndex" to pair items by their index.  
  - *Inputs:*  
    - Input 1: From "Map" node (workflow list items)  
    - Input 2: From "Get Workflow" node (detailed workflow data)  
  - *Outputs:* Connected to "FunctionItem" node.  
  - *Edge Cases:* Mismatch in item counts between inputs may cause data misalignment.

---

#### 1.3 Data Transformation

**Overview:**  
This block prepares the workflow data for upload by extracting the JSON, converting it to binary, and naming the file.

**Nodes Involved:**  
- FunctionItem (Function Item)  
- Move Binary Data (Move Binary Data)

**Node Details:**

- **FunctionItem**  
  - *Type:* Function Item  
  - *Role:* Extracts the detailed workflow data from the merged item for further processing.  
  - *Configuration:* JavaScript code:  
    ```js
    item = item.data;
    return item;
    ```  
  - *Inputs:* From "Merge" node.  
  - *Outputs:* Connected to "Move Binary Data".  
  - *Edge Cases:* Assumes `data` property exists; failure if missing.

- **Move Binary Data**  
  - *Type:* Move Binary Data  
  - *Role:* Converts JSON data into binary format suitable for file upload.  
  - *Configuration:*  
    - Mode: `jsonToBinary`  
    - Options: `useRawData` set to false (processes JSON data, not raw).  
  - *Inputs:* From "FunctionItem".  
  - *Outputs:* Connected to "Google Drive" node.  
  - *Edge Cases:* Large JSON objects may cause memory issues.

---

#### 1.4 Upload to Google Drive

**Overview:**  
This block uploads the prepared backup file to a specified Google Drive folder.

**Nodes Involved:**  
- Google Drive (Google Drive)

**Node Details:**

- **Google Drive**  
  - *Type:* Google Drive  
  - *Role:* Uploads the binary backup file to Google Drive.  
  - *Configuration:*  
    - File Name: Dynamically set as `{{$node["Merge"].data["name"]}}.json` (uses workflow name).  
    - Parent Folder ID: User must replace placeholder text with actual Google Drive folder ID.  
    - Binary Data: Enabled (uploads binary data).  
    - Resolve Data: Enabled (ensures data is resolved before upload).  
  - *Inputs:* From "Move Binary Data".  
  - *Outputs:* None (end of workflow).  
  - *Credentials:* Uses Google API credentials named "test".  
  - *Edge Cases:*  
    - Incorrect or missing folder ID causes upload failure.  
    - Authentication failure if credentials expire or are invalid.  
    - API rate limits or quota exceeded errors.

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                      | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                      |
|------------------------|---------------------|------------------------------------|-----------------------------|--------------------------|------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger      | Manual start of backup workflow    | None                        | Get Workflow List         |                                                                                                |
| Run Daily at 2:30am     | Cron Trigger        | Scheduled daily start at 2:30 AM   | None                        | Get Workflow List         |                                                                                                |
| Get Workflow List       | HTTP Request        | Fetch list of all workflows         | On clicking 'execute', Run Daily at 2:30am | Map                      | Don't forget to add your credentials for your n8n instance in this Node. Use Basic Auth for this. |
| Map                    | Function            | Split workflows list into items    | Get Workflow List            | Get Workflow, Merge       |                                                                                                |
| Get Workflow           | HTTP Request        | Fetch detailed workflow data by ID | Map                         | Merge                    | Don't forget to add your credentials for your n8n instance in this Node. Use Basic Auth for this. |
| Merge                  | Merge               | Combine workflow list and details  | Map, Get Workflow            | FunctionItem              |                                                                                                |
| FunctionItem           | Function Item       | Extract detailed workflow data     | Merge                       | Move Binary Data          |                                                                                                |
| Move Binary Data       | Move Binary Data    | Convert JSON to binary for upload  | FunctionItem                 | Google Drive              |                                                                                                |
| Google Drive           | Google Drive        | Upload backup file to Google Drive | Move Binary Data             | None                     |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Allow manual execution of the backup workflow.  
   - No parameters needed.

2. **Create Cron Trigger Node**  
   - Type: Cron Trigger  
   - Parameters: Set to trigger daily at 2:30 AM (hour=2, minute=30).  
   - Purpose: Automate daily backup execution.

3. **Create HTTP Request Node: Get Workflow List**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `http://localhost:5678/rest/workflows`  
     - Authentication: Basic Auth  
     - Credentials: Create or select credentials with username/password for your local n8n instance ("n8n Creds").  
   - Purpose: Retrieve all workflows from n8n.  
   - Connect outputs of Manual Trigger and Cron Trigger nodes to this node.

4. **Create Function Node: Map**  
   - Type: Function  
   - Parameters:  
     ```js
     return items[0].json.data.map(item => {
       return {json: item}
     });
     ```  
   - Purpose: Split the workflows list into individual items.  
   - Connect output of "Get Workflow List" to this node.

5. **Create HTTP Request Node: Get Workflow**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `http://localhost:5678/rest/workflows/{{$node["Map"].data["id"]}}` (use expression to dynamically insert workflow ID)  
     - Authentication: Basic Auth  
     - Credentials: Use same "n8n Creds" as above.  
   - Purpose: Fetch detailed data for each workflow.  
   - Connect output of "Map" node to this node.

6. **Create Merge Node**  
   - Type: Merge  
   - Parameters:  
     - Mode: `mergeByIndex`  
   - Purpose: Combine the original workflow list items and detailed workflow data.  
   - Connect outputs:  
     - Input 1: From "Map" node  
     - Input 2: From "Get Workflow" node

7. **Create Function Item Node: FunctionItem**  
   - Type: Function Item  
   - Parameters:  
     ```js
     item = item.data;
     return item;
     ```  
   - Purpose: Extract detailed workflow JSON data for upload.  
   - Connect output of "Merge" node to this node.

8. **Create Move Binary Data Node**  
   - Type: Move Binary Data  
   - Parameters:  
     - Mode: `jsonToBinary`  
     - Options: `useRawData` = false  
   - Purpose: Convert JSON data to binary format for file upload.  
   - Connect output of "FunctionItem" node to this node.

9. **Create Google Drive Node**  
   - Type: Google Drive  
   - Parameters:  
     - File Name: Use expression `{{$node["Merge"].data["name"]}}.json` to name the file after the workflow.  
     - Parent Folder ID: Replace placeholder text with your actual Google Drive folder ID (see instructions in workflow description).  
     - Binary Data: Enabled  
     - Resolve Data: Enabled  
   - Credentials: Configure Google API OAuth2 credentials ("test" or your own).  
   - Purpose: Upload the backup file to Google Drive.  
   - Connect output of "Move Binary Data" node to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| To find your Google Drive folder ID, open the folder in your browser and copy the string after `/folders/` in the URL. | See workflow description section "How to Find Your Google Drive Directory ID"                    |
| Ensure your n8n instance REST API is accessible locally at `http://localhost:5678` and that Basic Auth credentials are correctly configured. | Workflow requires local API access with Basic Auth credentials named "n8n Creds"                 |
| Backup frequency can be adjusted by modifying the Cron Trigger node's schedule.                      | Workflow description section "Adjusting Backup Frequency"                                       |
| For enhanced security, consider encrypting backups or restricting Google Drive folder permissions.  | Workflow description section "Encrypting the Backup for Extra Security"                          |
| Test your backup by manually triggering the workflow and verifying the uploaded JSON file in Google Drive. | Workflow description section "Verify That Your Backup Works"                                    |

---

This documentation provides a complete, detailed reference for understanding, reproducing, and customizing the "Backup n8n Workflows to Google Drive" workflow. It covers all nodes, their roles, configurations, and integration points, enabling both human users and automation agents to work confidently with this backup solution.