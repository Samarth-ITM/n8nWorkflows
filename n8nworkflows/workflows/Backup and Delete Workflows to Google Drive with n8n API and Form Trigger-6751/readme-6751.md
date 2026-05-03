Backup and Delete Workflows to Google Drive with n8n API and Form Trigger

https://n8nworkflows.xyz/workflows/backup-and-delete-workflows-to-google-drive-with-n8n-api-and-form-trigger-6751


# Backup and Delete Workflows to Google Drive with n8n API and Form Trigger

### 1. Workflow Overview

This workflow automates the process of backing up an n8n workflow as a JSON file to Google Drive and then deleting that workflow from the n8n instance. It is designed for users who want to keep their n8n environment clean and performant by removing unused or obsolete workflows while preserving their full definitions securely on Google Drive.

**Target Use Cases:**
- Self-hosted n8n instances managing multiple workflows.
- Users needing automated archival and cleanup of workflows.
- Teams requiring backups before workflow deletion for audit or recovery.

**Logical Blocks:**

- **1.1 Input Reception:**  
  Receives a workflow URL from the user via a form submission trigger.

- **1.2 Workflow Retrieval:**  
  Extracts the workflow ID from the URL and fetches the full workflow JSON using the n8n API.

- **1.3 JSON File Generation:**  
  Converts the fetched workflow JSON into a binary file formatted as a UTF-8 encoded `.json` file.

- **1.4 Backup Upload:**  
  Uploads the generated JSON file to a specified Google Drive folder using Google Drive API.

- **1.5 Workflow Deletion:**  
  Deletes the original workflow from the n8n instance via the API.

- **1.6 Notification:**  
  Sends a Telegram message to confirm successful backup and deletion.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures user input containing the full URL of the workflow to back up and delete.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **On form submission**  
    - Type: Form Trigger  
    - Role: Triggers workflow on submission of a custom form.  
    - Configuration:  
      - Form title: "🧾 Backup & Delete Workflow"  
      - Form fields: One required text input field prompting for "Paste the full workflow URL below."  
      - Button label: "Backup & Delete"  
      - No attribution appended.  
    - Inputs: HTTP webhook call when form is submitted.  
    - Outputs: Passes form data (workflow URL) to next node.  
    - Edge cases: Invalid or missing URL input; users must submit a valid full workflow URL (e.g., `https://your-n8n-instance.com/workflow/abc123`).  
    - Version: Uses Form Trigger v2.2.

#### 1.2 Workflow Retrieval

- **Overview:**  
  Extracts the workflow ID from the provided URL and fetches the corresponding workflow JSON from the n8n API.

- **Nodes Involved:**  
  - Get a Workflow

- **Node Details:**  
  - **Get a Workflow**  
    - Type: n8n node (n8n-nodes-base.n8n)  
    - Role: Uses n8n API to get a workflow's details by its ID.  
    - Configuration:  
      - Operation: "get"  
      - Workflow ID: Extracted dynamically from the URL submitted in the form field using expression mode "url".  
    - Credentials: API Key authentication with user’s n8n API key and base URL (e.g., `https://your-n8n-instance.com/api/v1`).  
    - Input: Receives the workflow URL from the form node.  
    - Output: Produces the full workflow JSON object.  
    - Edge cases:  
      - Invalid API credentials or missing API key.  
      - Workflow ID not found or malformed URL.  
      - API timeout or server errors.  
    - Version: API node v1.

#### 1.3 JSON File Generation

- **Overview:**  
  Converts the workflow JSON into a UTF-8 encoded `.json` file with an appropriate filename for upload.

- **Nodes Involved:**  
  - JSON to File

- **Node Details:**  
  - **JSON to File**  
    - Type: Code node (JavaScript)  
    - Role: Creates a binary file from the workflow JSON for Google Drive upload.  
    - Configuration:  
      - Generates filename by sanitizing workflow name (removes non-alphanumeric, replaces with underscores) or defaults to "workflow.json".  
      - Converts JSON data to formatted string with indentation.  
      - Uses helper function `prepareBinaryData` to convert string to binary with MIME type `application/json`.  
      - Outputs binary data property named `data` and JSON property `fileName`.  
    - Inputs: Receives workflow JSON from "Get a Workflow" node.  
    - Outputs: Binary file data for upload.  
    - Edge cases:  
      - Workflow name missing or containing special characters (handled by sanitization).  
      - Errors in JSON.stringify or buffer preparation (rare).  
    - Version: Code node v2.

#### 1.4 Backup Upload

- **Overview:**  
  Uploads the binary JSON file to a pre-configured Google Drive folder using a service account.

- **Nodes Involved:**  
  - Upload Workflow

- **Node Details:**  
  - **Upload Workflow**  
    - Type: Google Drive node  
    - Role: Uploads a file to Google Drive.  
    - Configuration:  
      - File name: Dynamically set from the workflow name provided by the previous node.  
      - Drive: "My Drive" (default)  
      - Folder ID: Fixed Google Drive folder ID (`1Wv8HI8jUu-XbpC4mRJH78WNMAD_xM8f8`) where backups are stored.  
      - Authentication: Service account credentials for Google API.  
      - Binary property to upload: `data` (from "JSON to File").  
    - Inputs: Receives binary file data from "JSON to File".  
    - Outputs: Passes workflow JSON forward including file upload metadata.  
    - Edge cases:  
      - Invalid Google Drive credentials or expired tokens.  
      - Folder ID incorrect or access denied.  
      - File name conflicts (managed by overwriting or Google Drive behavior).  
    - Version: Google Drive node v3.

#### 1.5 Workflow Deletion

- **Overview:**  
  Deletes the specified workflow from the n8n instance via the API using the workflow ID.

- **Nodes Involved:**  
  - Delete a Workflow

- **Node Details:**  
  - **Delete a Workflow**  
    - Type: n8n node (n8n-nodes-base.n8n)  
    - Role: Deletes a workflow by its ID using the n8n API.  
    - Configuration:  
      - Operation: "delete"  
      - Workflow ID: Dynamically set from the earlier fetched workflow JSON (`id`).  
    - Credentials: Uses the same API Key credential as "Get a Workflow".  
    - Inputs: Receives workflow JSON from "Upload Workflow".  
    - Outputs: Confirmation of deletion.  
    - Edge cases:  
      - API errors or permissions preventing deletion.  
      - Workflow already deleted or not found.  
      - API timeouts.  
    - Version: API node v1.

#### 1.6 Notification

- **Overview:**  
  Sends a Telegram message to a predefined chat confirming the workflow backup and deletion.

- **Nodes Involved:**  
  - Send a text message

- **Node Details:**  
  - **Send a text message**  
    - Type: Telegram node (n8n-nodes-base.telegram)  
    - Role: Sends a notification message to Telegram.  
    - Configuration:  
      - Text: Template message including workflow name, ID, Google Drive backup link placeholder, and current date/time.  
      - Chat ID: Fixed chat ID `8019473661` (user must replace with their own).  
      - Credentials: Telegram Bot API credentials configured.  
    - Inputs: Receives workflow deletion confirmation data.  
    - Outputs: Final node, no further outputs.  
    - Edge cases:  
      - Invalid Telegram bot token or revoked bot permissions.  
      - Chat ID incorrect or bot not a member of chat.  
      - Message formatting errors (unlikely).  
    - Version: Telegram node v1.2.

---

### 3. Summary Table

| Node Name          | Node Type              | Functional Role                | Input Node(s)      | Output Node(s)     | Sticky Note                                                                                                          |
|--------------------|------------------------|-------------------------------|--------------------|--------------------|----------------------------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger           | Receives workflow URL input   | -                  | Get a Workflow     | ## 📝 1. Form Node: "Get Workflow URL" - Instruction and example input. Link to documentation included.              |
| Get a Workflow      | n8n API Node           | Fetches workflow JSON          | On form submission | JSON to File       | ## 🔍 2. n8n Node: "Get Workflow" - API Key auth required. Link to documentation included.                           |
| JSON to File       | Code                   | Converts JSON to binary file   | Get a Workflow     | Upload Workflow    | ## 📦 5. Code Node: "Generate .json File" - Formats JSON and prepares binary data. Link to documentation included.    |
| Upload Workflow     | Google Drive           | Uploads JSON backup to Drive   | JSON to File       | Delete a Workflow  | ## 📁 Google Drive Node: "Upload Workflow Backup" - Config for folder ID and binary property. Link included.         |
| Delete a Workflow   | n8n API Node           | Deletes workflow from n8n      | Upload Workflow    | Send a text message| ## 🗑️ 5. n8n Node: "Delete Workflow" - Uses API key auth. Link to documentation included.                            |
| Send a text message | Telegram               | Sends Telegram notification    | Delete a Workflow  | -                  | ## 📬 6. Telegram Node: "Send Notification" - Sends confirmation message. Link to documentation included.            |
| Sticky Note         | Sticky Note            | Description and overview       | -                  | -                  | ## 🔍 Description: Workflow purpose, what it does, setup instructions, security notes, and activation instructions.  |
| Sticky Note1        | Sticky Note            | Form node instructions         | -                  | -                  | ## 📝 1. Form Node: "Get Workflow URL" - Instructions and link.                                                       |
| Sticky Note2        | Sticky Note            | Get Workflow node instructions | -                  | -                  | ## 🔍 2. n8n Node: "Get Workflow" - Instructions and link.                                                           |
| Sticky Note4        | Sticky Note            | Google Drive node instructions | -                  | -                  | ## 📁 Google Drive Node: "Upload Workflow Backup" - Instructions and link.                                          |
| Sticky Note5        | Sticky Note            | Delete Workflow node instructions| -                 | -                  | ## 🗑️ 5. n8n Node: "Delete Workflow" - Instructions and link.                                                      |
| Sticky Note6        | Sticky Note            | Telegram node instructions     | -                  | -                  | ## 📬 6. Telegram Node: "Send Notification" - Instructions and link.                                                |
| Sticky Note7        | Sticky Note            | Code node instructions         | -                  | -                  | ## 📦 5. Code Node: "Generate .json File" - Instructions and link.                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("On form submission")**  
   - Type: Form Trigger  
   - Configure form title: "🧾 Backup & Delete Workflow"  
   - Add one required text field: Label "Paste the full workflow URL below." with placeholder "https://n8n-miinstancia.com/workflow/abc123"  
   - Button label: "Backup & Delete"  
   - No attribution appended.  
   - Position this node as the workflow start.

2. **Create n8n API Node ("Get a Workflow")**  
   - Type: n8n API node  
   - Operation: Get  
   - Workflow ID: Set mode to "url" and map from form field input `={{ $json['Paste the full workflow URL below.'] }}`  
   - Credentials: Use API Key credential with your n8n API key and base URL (e.g., `https://your-n8n-instance.com/api/v1`)  
   - Connect output of Form Trigger to this node’s input.

3. **Create Code Node ("JSON to File")**  
   - Type: Code node (JavaScript)  
   - Paste the following JavaScript code:

     ```javascript
     const fileName = `${$json.name.replace(/[^a-z0-9_-]/gi, "_") || "workflow"}.json`;
     const jsonContent = JSON.stringify($json, null, 2);

     return [
       {
         binary: {
           data: await this.helpers.prepareBinaryData(Buffer.from(jsonContent), fileName, 'application/json'),
         },
         json: {
           fileName,
         },
       },
     ];
     ```

   - Connect output of "Get a Workflow" to this node’s input.

4. **Create Google Drive Node ("Upload Workflow")**  
   - Type: Google Drive node  
   - Operation: Upload file  
   - Set Drive to "My Drive" or your preferred drive.  
   - Folder ID: Enter your target Google Drive folder ID, e.g., `1Wv8HI8jUu-XbpC4mRJH78WNMAD_xM8f8`  
   - Name: Use expression `={{ $('Get a Workflow').item.json.name }}` to dynamically name the file.  
   - Binary property: `data` (from previous Code node)  
   - Authentication: Use Google Service Account credentials with access to your Drive folder.  
   - Connect output of "JSON to File" to this node’s input.

5. **Create n8n API Node ("Delete a Workflow")**  
   - Type: n8n API node  
   - Operation: Delete  
   - Workflow ID: Use expression `={{ $('Get a Workflow').item.json.id }}`  
   - Credentials: Use the same API Key credential as before.  
   - Connect output of "Upload Workflow" to this node’s input.

6. **Create Telegram Node ("Send a text message")**  
   - Type: Telegram node  
   - Chat ID: Enter your Telegram chat ID (e.g., `8019473661`)  
   - Text: Use the following template (modify the Google Drive link placeholder before using):

     ```
     ✅ Workflow Backup Successful

     The workflow titled: {{ $json.name }}
     with ID: {{ $json.id }}
     has been successfully backed up to your selected Google Drive Folder.

     📄 You can view the backup at the following link:
     (USE YOUR GOOGLE DRIVE FOLDER LINK HERE)

     📅 Backup Date: {{ $now }}
     ```

   - Credentials: Telegram Bot API credentials configured.  
   - Connect output of "Delete a Workflow" to this node’s input.

7. **Save and Activate the Workflow**

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow backs up entire workflow data to Google Drive. Keep your Drive folder permissions secure to prevent unauthorized access. | Security best practice |
| Use the official n8n API key credential and ensure your API key has permissions to get and delete workflows. | n8n API docs |
| Google Drive folder ID can be found from the folder’s URL in Google Drive UI. | Google Drive |
| Telegram Bot and Chat ID must be set up correctly to receive notifications. | Telegram Bot API docs |
| Form Trigger documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.formtrigger/ | Official n8n docs |
| n8n API node documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.n8n/ | Official n8n docs |
| Google Drive node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive/file-operations/#upload-a-file | Official n8n docs |
| Telegram node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegram/ | Official n8n docs |

---

**Disclaimer:**  
The text provided originates exclusively from an automated n8n workflow developed with the n8n integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.