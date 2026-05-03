Backup & Restore n8n Workflows with Telegram, Google Drive and Form Upload

https://n8nworkflows.xyz/workflows/backup---restore-n8n-workflows-with-telegram--google-drive-and-form-upload-9645


# Backup & Restore n8n Workflows with Telegram, Google Drive and Form Upload

### 1. Workflow Overview

This workflow automates the backup and restoration of all n8n workflows using multiple integration paths involving Telegram, Google Drive, and a direct file upload form. It is designed to periodically export all workflows into a JSON backup file, send it to a Telegram chat for safe keeping, and provide two restoration methods: downloading from Google Drive or uploading via a form. The restoration process intelligently checks if each workflow already exists and updates it or creates new ones accordingly.

The workflow is logically divided into three main blocks:

- **1.1 Automated Backup Trigger & Export:** Periodically fetches all workflows, aggregates and converts them into a text file, then sends the backup file to a Telegram chat.
- **1.2 Restore Trigger & File Extraction:** Supports two restoration triggers—one for downloading the backup file from Google Drive and another for uploading the backup via an n8n form. Both extract the backup text for processing.
- **1.3 Workflow Restore Processing:** Parses the backup JSON, cleans and splits it into individual workflows, then iterates over them to create or update workflows in the n8n instance with rate limiting.

---

### 2. Block-by-Block Analysis

#### 2.1 Automated Backup Trigger & Export

- **Overview:**  
  This block initiates an automated workflow backup every 3 days. It fetches all existing workflows via the n8n API, aggregates them into a single JSON array, converts the JSON to a text file, and sends the backup file to a configured Telegram chat.

- **Nodes Involved:**  
  - Schedule Backup Trigger  
  - Fetch All Workflows  
  - Aggregate Workflows  
  - Convert to Backup File  
  - Send Backup to Telegram  
  - Notes: Schedule Trigger, Fetch Workflows, Aggregate Data, Convert File, Send to Telegram

- **Node Details:**

  1. **Schedule Backup Trigger**  
     - Type: Schedule Trigger  
     - Role: Triggers the workflow automatically every 3 days (adjustable interval).  
     - Configuration: Interval set to 3 days.  
     - Inputs: None (trigger node).  
     - Outputs: Initiates workflow execution.  
     - Edge Cases: Misconfiguration of interval or disabled node stops backup.

  2. **Fetch All Workflows**  
     - Type: n8n API Node (n8n)  
     - Role: Retrieves all workflows from the n8n instance through API.  
     - Configuration: No filters, uses n8n API credential.  
     - Inputs: Trigger from Schedule Backup Trigger.  
     - Outputs: JSON array of all workflows.  
     - Edge Cases: API authentication failure, network issues.

  3. **Aggregate Workflows**  
     - Type: Aggregate  
     - Role: Combines all workflow items into a single JSON array in the `data` property.  
     - Configuration: Uses 'aggregateAllItemData' option.  
     - Inputs: From Fetch All Workflows.  
     - Outputs: Single aggregated item for file conversion.  
     - Edge Cases: Empty workflow list outputs empty array.

  4. **Convert to Backup File**  
     - Type: Convert To File  
     - Role: Converts aggregated JSON data into a text file.  
     - Configuration: Operation `toText`, source property `data`, sets filename `All-n8n-workflows.txt`.  
     - Inputs: From Aggregate Workflows.  
     - Outputs: Binary file data.  
     - Edge Cases: Large JSON size may affect memory.

  5. **Send Backup to Telegram**  
     - Type: Telegram  
     - Role: Sends the backup file as a document to a Telegram chat.  
     - Configuration: Requires Telegram API credential, `chatId` must be replaced with user’s chat ID, sends document with filename.  
     - Inputs: From Convert to Backup File.  
     - Outputs: Confirmation of sent message.  
     - Edge Cases: Invalid chat ID, bot permissions, Telegram API limits.

---

#### 2.2 Restore Trigger & File Extraction

- **Overview:**  
  This block enables restoration of workflows by either downloading a backup file from Google Drive or uploading a backup file via an n8n form. Both methods extract the file’s text content to prepare for JSON parsing.

- **Nodes Involved:**  
  - Manual Trigger  
  - Download Backup from Drive  
  - Form Restore Trigger  
  - Extract Backup Text  
  - Notes: Download from Drive, Form Trigger, Extract Text

- **Node Details:**

  1. **Manual Trigger**  
     - Type: Manual Trigger  
     - Role: Allows manual initiation of Google Drive download restore flow.  
     - Inputs: None (trigger node).  
     - Outputs: Starts download backup from Drive.  
     - Edge Cases: Requires manual user intervention.

  2. **Download Backup from Drive**  
     - Type: Google Drive  
     - Role: Downloads backup text file from Google Drive by file ID.  
     - Configuration: Operation `download`, `fileId` must be replaced with actual Drive file ID, uses Google Drive OAuth2 credential.  
     - Inputs: From Manual Trigger.  
     - Outputs: Binary data of the backup file.  
     - Edge Cases: Invalid file ID, OAuth token expiry, file not found.

  3. **Form Restore Trigger**  
     - Type: Form Trigger  
     - Role: Provides a web form to upload backup file for restore.  
     - Configuration: Single file upload field named `data`, webhook URL provided for form submission.  
     - Inputs: External file upload.  
     - Outputs: Uploaded file binary data.  
     - Edge Cases: Large file uploads, invalid file format, user errors.

  4. **Extract Backup Text**  
     - Type: Extract From File  
     - Role: Extracts raw text content from the uploaded or downloaded backup file.  
     - Configuration: Operation `text`.  
     - Inputs: From Download Backup from Drive and Form Restore Trigger.  
     - Outputs: JSON string in the `data` property.  
     - Edge Cases: Corrupt or binary files that cannot be parsed as text.

---

#### 2.3 Workflow Restore Processing

- **Overview:**  
  This block processes the extracted backup JSON string by parsing and cleaning individual workflows, then sequentially creating or updating each workflow in the n8n instance. It includes rate limiting pauses between API calls to avoid throttling.

- **Nodes Involved:**  
  - Parse Backup JSON (Code)  
  - Process Each Workflow (SplitInBatches)  
  - Check Workflow Existence (n8n API)  
  - If Workflow Exists (If)  
  - Create New Workflow (n8n API)  
  - Update Existing Workflow (n8n API)  
  - Wait for Completion (Wait)  
  - Notes: Parse JSON, Workflow Loop, Wait Delay

- **Node Details:**

  1. **Parse Backup JSON**  
     - Type: Code Node (JavaScript)  
     - Role: Parses JSON string into workflow objects, cleans unsupported properties, removes IDs, and prepares each workflow as a separate item with metadata.  
     - Configuration: Cleans settings like executionOrder, removes IDs, removes webhookIds in nodes, preserves pinData, cleans meta info.  
     - Inputs: From Extract Backup Text.  
     - Outputs: Array of sanitized workflows, each as an item with `data` property containing stringified workflow JSON.  
     - Edge Cases: Malformed JSON throws error; property name mismatch for file content requires adjustment.

  2. **Process Each Workflow**  
     - Type: SplitInBatches  
     - Role: Iterates workflows one-by-one to handle API rate limits and sequential processing.  
     - Configuration: Default batch size 1.  
     - Inputs: From Parse Backup JSON.  
     - Outputs: Single workflow item per iteration to Check Workflow Existence.  
     - Edge Cases: Large batch sizes risk API limits.

  3. **Check Workflow Existence**  
     - Type: n8n API Node (n8n)  
     - Role: Queries existing workflows by name to detect if workflow exists for update or create logic.  
     - Configuration: Filters by workflow name from current item JSON.  
     - Inputs: From Process Each Workflow (second output).  
     - Outputs: List of workflows matching the name.  
     - Edge Cases: Multiple workflows with same name possible; API failure.

  4. **If Workflow Exists**  
     - Type: If  
     - Role: Conditional node that checks if a workflow with the same name exists.  
     - Configuration: Checks if `$json.name` exists (indicating workflow found).  
     - Inputs: From Check Workflow Existence.  
     - Outputs:  
       - True branch: Workflow exists → Update Existing Workflow node.  
       - False branch: Workflow does not exist → Create New Workflow node.  
     - Edge Cases: False positives if multiple workflows share the same name.

  5. **Create New Workflow**  
     - Type: n8n API Node (n8n)  
     - Role: Creates a new workflow using JSON from the processed item.  
     - Configuration: Operation `create`, uses workflow JSON from Process Each Workflow node.  
     - Inputs: From If Workflow Exists (false branch).  
     - Outputs: Passes to Wait for Completion and loops back to Process Each Workflow.  
     - Error Handling: Continues on error without stopping entire workflow.  
     - Edge Cases: API errors, invalid workflow JSON.

  6. **Update Existing Workflow**  
     - Type: n8n API Node (n8n)  
     - Role: Updates an existing workflow by ID with new workflow JSON.  
     - Configuration: Operation `update`, workflowId from existing workflow, workflow JSON from Process Each Workflow.  
     - Inputs: From If Workflow Exists (true branch).  
     - Outputs: Passes to Wait for Completion and loops back to Process Each Workflow.  
     - Edge Cases: API errors, concurrency issues.

  7. **Wait for Completion**  
     - Type: Wait  
     - Role: Delays the workflow between API calls to avoid hitting rate limits.  
     - Configuration: Default 1-second delay, adjustable as needed.  
     - Inputs: From Create or Update Workflow nodes.  
     - Outputs: Loops back to Process Each Workflow for next iteration.  
     - Edge Cases: Insufficient delay may cause throttling.

---

### 3. Summary Table

| Node Name              | Node Type                | Functional Role                          | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                     |
|------------------------|--------------------------|----------------------------------------|----------------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| Manual Trigger         | Manual Trigger           | Manual start for Drive restore          | None                             | Download Backup from Drive      |                                                                                                |
| Schedule Backup Trigger | Schedule Trigger         | Initiates automated backup every 3 days| None                             | Fetch All Workflows             | ## ⏰ Schedule Backup Trigger - Initiates automated workflow export every 3 days. Adjust interval as needed. |
| Fetch All Workflows    | n8n API (n8n)            | Fetches all workflows via API           | Schedule Backup Trigger          | Aggregate Workflows            | ## 🔍 Fetch All Workflows - Requires n8n API credential; outputs JSON array.                    |
| Aggregate Workflows    | Aggregate                | Aggregates workflows into single JSON  | Fetch All Workflows              | Convert to Backup File          | ## 📊 Aggregate Workflows - Combines fetched workflows into one JSON array.                    |
| Convert to Backup File | Convert To File          | Converts JSON array to text file        | Aggregate Workflows              | Send Backup to Telegram         | ## 📄 Convert to Backup File - Converts JSON to text file, named All-n8n-workflows.txt.        |
| Send Backup to Telegram| Telegram                 | Sends backup file to Telegram chat      | Convert to Backup File           | None                          | ## 📤 Send Backup to Telegram - Replace '{{your_chat_id}}' with actual chat ID; requires bot credential. |
| Download Backup from Drive| Google Drive            | Downloads backup file from Drive         | Manual Trigger                  | Extract Backup Text             | ## 📥 Download Backup from Drive - Set '{{your_file_id}}' from Drive URL; requires OAuth credential. |
| Form Restore Trigger   | Form Trigger             | Receives uploaded backup file via form  | None                            | Extract Backup Text             | ## 📝 Form Restore Trigger - Allows local backup upload for restore.                           |
| Extract Backup Text    | Extract From File        | Extracts text content from file          | Download Backup from Drive, Form Restore Trigger | Parse Backup JSON             | ## 📖 Extract Backup Text - Reads text content for parsing.                                   |
| Parse Backup JSON      | Code                     | Parses and cleans JSON workflows         | Extract Backup Text             | Process Each Workflow          | ## 🔄 Parse Backup JSON - Cleans JSON, removes IDs and unsupported properties.                 |
| Process Each Workflow  | SplitInBatches           | Iterates workflows one-by-one for restore| Parse Backup JSON              | Check Workflow Existence       | ## 🔄 Process Each Workflow - Loops over workflows sequentially to avoid API limits.           |
| Check Workflow Existence| n8n API (n8n)            | Checks if workflow exists by name        | Process Each Workflow           | If Workflow Exists             |                                                                                                |
| If Workflow Exists     | If                       | Branches create or update logic          | Check Workflow Existence        | Create New Workflow, Update Existing Workflow |                                                                                |
| Create New Workflow    | n8n API (n8n)            | Creates new workflow                      | If Workflow Exists (False)      | Wait for Completion            |                                                                                                |
| Update Existing Workflow| n8n API (n8n)            | Updates existing workflow                 | If Workflow Exists (True)       | Wait for Completion            |                                                                                                |
| Wait for Completion    | Wait                     | Delays between API calls to avoid limits | Create New Workflow, Update Existing Workflow | Process Each Workflow   | ## ⏳ Wait for Completion - Default 1s delay; increase for large batches.                      |
| Note: Schedule Trigger | Sticky Note              | Documentation on Schedule Trigger        | None                          | None                           |                                                                                                |
| Note: Fetch Workflows  | Sticky Note              | Explains fetching all workflows          | None                          | None                           |                                                                                                |
| Note: Aggregate Data   | Sticky Note              | Explains aggregation step                 | None                          | None                           |                                                                                                |
| Note: Convert File     | Sticky Note              | Explains file conversion step             | None                          | None                           |                                                                                                |
| Note: Send to Telegram | Sticky Note              | Explains sending file to Telegram         | None                          | None                           |                                                                                                |
| Note: Download from Drive | Sticky Note           | Explains Google Drive download            | None                          | None                           |                                                                                                |
| Note: Form Trigger     | Sticky Note              | Explains form upload trigger              | None                          | None                           |                                                                                                |
| Note: Extract Text     | Sticky Note              | Explains extracting text from file        | None                          | None                           |                                                                                                |
| Note: Parse JSON       | Sticky Note              | Explains JSON parsing and cleaning        | None                          | None                           |                                                                                                |
| Note: Workflow Loop    | Sticky Note              | Explains looping over workflows           | None                          | None                           |                                                                                                |
| Note: Wait Delay       | Sticky Note              | Explains wait node purpose and config     | None                          | None                           |                                                                                                |
| Overview Note3         | Sticky Note              | Full workflow overview, prerequisites, and troubleshooting | None                 | None                           | Contains detailed info about setup, credentials, use cases, links, and troubleshooting tips.   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Nodes:**

   - Add a **Schedule Trigger** node, set to execute every 3 days to automate backup.
   - Add a **Manual Trigger** node for manual restore initiation.
   - Add a **Form Trigger** node configured with a single file upload field labeled `data` for direct backup upload.

2. **Fetch and Aggregate Workflows:**

   - Add an **n8n API** node named "Fetch All Workflows" with operation to list workflows and no filters.
   - Connect Schedule Trigger → Fetch All Workflows.
   - Add an **Aggregate** node set to aggregate all item data into one item.
   - Connect Fetch All Workflows → Aggregate Workflows.

3. **Convert Workflows to Backup File:**

   - Add a **Convert To File** node:
     - Operation: toText
     - Source Property: `data`
     - Set filename to `All-n8n-workflows.txt`
   - Connect Aggregate Workflows → Convert to Backup File.

4. **Send Backup File to Telegram:**

   - Add a **Telegram** node:
     - Operation: sendDocument
     - Chat ID: Replace `{{your_chat_id}}` with actual chat ID.
     - Use binary data from previous node.
     - Set filename to `All-n8n-workflows.txt`.
   - Connect Convert to Backup File → Send Backup to Telegram.
   - Configure Telegram API credentials using your bot token.

5. **Download Backup from Google Drive:**

   - Add a **Google Drive** node:
     - Operation: download
     - File ID: Replace `{{your_file_id}}` with actual Google Drive file ID.
   - Connect Manual Trigger → Download Backup from Drive.
   - Configure Google Drive OAuth2 credentials.

6. **Extract Text from Backup File:**

   - Add an **Extract From File** node:
     - Operation: text
   - Connect both Download Backup from Drive and Form Restore Trigger → Extract Backup Text.

7. **Parse and Clean JSON Backup:**

   - Add a **Code** node ("Parse Backup JSON") with provided JavaScript code to:
     - Parse JSON string from `data` property.
     - Remove unsupported settings, IDs, and webhook IDs.
     - Return one workflow per item with workflow JSON string in `data`.
   - Connect Extract Backup Text → Parse Backup JSON.

8. **Process Each Workflow Sequentially:**

   - Add a **SplitInBatches** node ("Process Each Workflow") with batch size = 1.
   - Connect Parse Backup JSON → Process Each Workflow.

9. **Check if Workflow Exists:**

   - Add an **n8n API** node ("Check Workflow Existence"):
     - Operation: list workflows.
     - Filter: by name equal to current item workflow name.
   - Connect Process Each Workflow (second output) → Check Workflow Existence.

10. **Conditional Branch:**

    - Add an **If** node ("If Workflow Exists"):
      - Condition: Check if the API returned a workflow (`$json.name` exists).
    - Connect Check Workflow Existence → If Workflow Exists.

11. **Create or Update Workflow:**

    - Add **n8n API** node ("Create New Workflow") for creating workflows:
      - Operation: create.
      - Use current workflow JSON from Process Each Workflow.
    - Add **n8n API** node ("Update Existing Workflow") for updating:
      - Operation: update.
      - Workflow ID from existing workflow.
      - Use current workflow JSON from Process Each Workflow.
    - Connect If Workflow Exists false → Create New Workflow.
    - Connect If Workflow Exists true → Update Existing Workflow.

12. **Wait Between API Calls:**

    - Add a **Wait** node ("Wait for Completion") with default 1 second delay.
    - Connect Create New Workflow → Wait for Completion.
    - Connect Update Existing Workflow → Wait for Completion.

13. **Loop Back to Process Next Workflow:**

    - Connect Wait for Completion → Process Each Workflow (main input) to continue batch processing.

14. **Credential Setup:**

    - **n8n API:** Configure with API key generated in n8n settings.
    - **Telegram API:** Use bot token from @BotFather and chat ID from @userinfobot.
    - **Google Drive OAuth2 API:** Create OAuth credentials in Google Cloud Console, enable Drive API, set redirect URI, and authorize.

15. **Replace Placeholders:**

    - Replace `{{your_chat_id}}` in Telegram node with your actual Telegram chat ID.
    - Replace `{{your_file_id}}` in Google Drive node with your backup file’s Drive ID.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| This workflow automates backup of all n8n workflows every 3 days to Telegram and supports restore via Google Drive download or form upload. Requires n8n API access, Telegram bot, and Google Drive OAuth credentials. Detailed credential setup and troubleshooting tips are included in the overview sticky note.                                                                                                                                                                                                                           | See Overview Note3 sticky note in the workflow for full setup and troubleshooting instructions.                          |
| Telegram chat ID can be obtained using the @userinfobot Telegram bot. Bot token is created via @BotFather.                                                                                                                                                                                                                                                                                                                                                                                                                            | Telegram API credential setup instructions in Overview Note3.                                                             |
| Google Drive OAuth2 credentials require creating a Google Cloud project, enabling Drive API, and setting authorized redirect URIs matching your n8n instance URLs.                                                                                                                                                                                                                                                                                                                                                                   | Google Drive OAuth2 setup instructions in Overview Note3.                                                                  |
| To avoid API rate limits during restore, use the Wait node delay (default 1s). Increase delay if processing large numbers of workflows or encountering throttling errors.                                                                                                                                                                                                                                                                                                                                                               | Wait Delay sticky note explains this.                                                                                      |
| The parse code node removes unsupported workflow settings and webhook IDs that may cause conflicts when importing workflows. Adjust property names if your Extract From File node outputs differently.                                                                                                                                                                                                                                                                                                                                | See Parse Backup JSON Code node description for details.                                                                    |
| Multiple workflows with the same name may cause update ambiguity. Consider renaming workflows to ensure unique names before restore.                                                                                                                                                                                                                                                                                                                                                                                                   | Edge case in Check Workflow Existence and If Workflow Exists nodes.                                                        |
| Backup files are JSON text files named `All-n8n-workflows.txt`. Keep backup copies safe and verify file integrity before restore.                                                                                                                                                                                                                                                                                                                                                                                                      | Backup file naming explained in Convert to Backup File node sticky note.                                                   |

---

**Disclaimer:** This documentation is generated exclusively from the provided n8n workflow JSON. It adheres strictly to content policies and contains no illegal or protected elements. All handled data is lawful and public.