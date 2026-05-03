Organize Gmail Attachments in Google Drive Folders based on sender’s email

https://n8nworkflows.xyz/workflows/organize-gmail-attachments-in-google-drive-folders-based-on-sender-s-email-6277


# Organize Gmail Attachments in Google Drive Folders based on sender’s email

### 1. Workflow Overview

This workflow is designed to automate the organization of Gmail email attachments into Google Drive folders, categorizing the attachments based on the sender's email address. It monitors incoming Gmail messages (via trigger or manual execution), downloads attachments, verifies if a Google Drive folder named after the sender's email exists, creates the folder if necessary, and uploads the attachments into the corresponding folder. The workflow ensures no duplicate folders are created by checking folder existence prior to creation.

Logical blocks:

- **1.1 Input Reception:** Receives emails via Gmail Trigger or manual trigger, and optionally fetches emails in batch.
- **1.2 Email Processing Loop:** Iterates over each email to process attachments individually.
- **1.3 Attachment Extraction:** Extracts attachments from emails and processes each attachment separately.
- **1.4 Folder Management:** Checks if a Google Drive folder exists for the sender's email; creates it if missing.
- **1.5 Uploading Attachments:** Uploads the extracted attachments to the appropriate Google Drive folder.
- **1.6 Workflow Execution Control:** Supports execution triggered by manual input or other workflows.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block handles starting the workflow by detecting new Gmail messages either automatically every minute or manually. It also supports fetching emails based on date filters.

**Nodes Involved:**  
- Gmail Trigger  
- When clicking ‘Execute workflow’  
- Get emails  

**Node Details:**

- **Gmail Trigger**  
  - Type: Trigger node for new Gmail emails  
  - Configuration: Polls Gmail every minute for new emails without filters applied  
  - Inputs: None (trigger)  
  - Outputs: Email items representing new messages  
  - Failure cases: Authentication errors, Gmail API quota limits, network timeout  
  - Credentials: OAuth2 Gmail account  
 
- **When clicking ‘Execute workflow’**  
  - Type: Manual trigger node  
  - Configuration: No parameters required  
  - Inputs: None (manual start)  
  - Outputs: Triggers execution flow  
  - Failure cases: None expected  

- **Get emails**  
  - Type: Gmail node for fetching emails  
  - Configuration: Retrieves all emails received between July 6, 2025 and July 9, 2025; downloads attachments  
  - Inputs: Trigger from manual start  
  - Outputs: List of emails matching filter  
  - Failure cases: Authentication issues, API rate limits, malformed date filters  
  - Credentials: OAuth2 Gmail account  

---

#### 2.2 Email Processing Loop

**Overview:**  
Iterates through each email item to allow individual processing of attachments and metadata.

**Nodes Involved:**  
- Loop Over Items  
- Get email  

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches node (batch processing)  
  - Configuration: Processes items one by one without reset  
  - Inputs: List of emails from Get emails or Execute Workflow  
  - Outputs: Single email item per iteration  
  - Failure cases: Batch reset misconfiguration, lost context between batches  

- **Get email**  
  - Type: Gmail node to fetch a single email by ID  
  - Configuration: Downloads attachments for the given message ID from the loop  
  - Inputs: Single email ID from Loop Over Items  
  - Outputs: Detailed email item with attachments  
  - Failure cases: Invalid message ID, auth errors, attachment download failure  
  - Credentials: OAuth2 Gmail account  

---

#### 2.3 Attachment Extraction

**Overview:**  
Determines if the email contains attachments and extracts each attachment as a separate item for uploading.

**Nodes Involved:**  
- Contain attachments?  
- Split Out  
- Get attachments  

**Node Details:**

- **Contain attachments?**  
  - Type: If node  
  - Configuration: Checks if the `$binary` property (attachments) is non-empty  
  - Inputs: Email item with binary attachment data  
  - Outputs: Yes branch if attachments exist, No branch to skip  
  - Failure cases: Expression evaluation errors if binary data missing  

- **Split Out**  
  - Type: SplitOut node  
  - Configuration: Splits the binary attachments into separate items  
  - Field to split: `$binary`  
  - Inputs: Emails with attachments from Contain attachments?  
  - Outputs: One item per attachment  
  - Failure cases: Missing binary field, empty lists  

- **Get attachments**  
  - Type: Code node (JavaScript)  
  - Configuration: Iterates over all binary attachments and returns each as separate items with fileName and binary data fields  
  - Inputs: Items from Split Out  
  - Outputs: Items with JSON filename and binary data for each attachment  
  - Failure cases: Script errors, empty binary data  

---

#### 2.4 Folder Management

**Overview:**  
Checks if a Google Drive folder exists for the email sender; creates one if it does not exist.

**Nodes Involved:**  
- Search folders  
- Exist?  
- Create folder  
- Folder ID  

**Node Details:**

- **Search folders**  
  - Type: Google Drive node (search files/folders)  
  - Configuration: Searches for folders named exactly as the sender's email address inside a predefined parent folder ("Email Attachments")  
  - Inputs: Email item with sender’s email (from Get email)  
  - Outputs: Folder information if found, empty if not  
  - Failure cases: API errors, permission issues  
  - Credentials: OAuth2 Google Drive account  

- **Exist?**  
  - Type: If node  
  - Configuration: Checks if the search result is non-empty (folder exists)  
  - Inputs: Search folders output  
  - Outputs: Yes branch (folder exists), No branch (folder missing)  
  - Failure cases: Expression errors if search returns unexpected data  

- **Create folder**  
  - Type: Google Drive node (folder creation)  
  - Configuration: Creates a new folder named as sender's email inside the "Email Attachments" parent folder  
  - Inputs: No folder found branch from Exist?  
  - Outputs: Newly created folder’s metadata  
  - Failure cases: API limits, permission denied  
  - Credentials: OAuth2 Google Drive account  

- **Folder ID**  
  - Type: Set node  
  - Configuration: Stores the folder ID from either the search result or newly created folder for downstream operations  
  - Inputs: Yes branch from Exist? or Create folder  
  - Outputs: Item with `folder_id` field set  
  - Failure cases: Missing folder metadata  

---

#### 2.5 Uploading Attachments

**Overview:**  
Uploads each extracted attachment to the corresponding Google Drive folder identified for the sender.

**Nodes Involved:**  
- Merge  
- Execute Workflow  
- When Executed by Another Workflow  
- Upload attachment  

**Node Details:**

- **Merge**  
  - Type: Merge node  
  - Configuration: Combines all extracted attachment items into a single stream for upload  
  - Inputs: Attachment items from Get attachments and Folder ID objects  
  - Outputs: Merged items ready for upload  
  - Failure cases: Mismatched input lengths, data loss  

- **Execute Workflow**  
  - Type: Execute Workflow node (recursive call)  
  - Configuration: Executes the same workflow "Organize Email Attachments in Google Drive Folders" in "each" mode, waits for subworkflow completion  
  - Inputs: Merged items  
  - Outputs: Processed items after recursive execution  
  - Failure cases: Infinite recursion if not controlled, workflow errors  
  - Sub-workflow: Self-reference to current workflow  

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger node  
  - Configuration: Triggered when called by another workflow; passes input through  
  - Inputs: From Execute Workflow recursive calls  
  - Outputs: Items to Upload attachment node  
  - Failure cases: Trigger misfire, data mismatch  

- **Upload attachment**  
  - Type: Google Drive node (file upload)  
  - Configuration: Uploads a single attachment to the folder specified by `folder_id`; uses binary data and filename from input  
  - Inputs: Attachment item with folder ID and binary file data  
  - Outputs: Uploaded file metadata  
  - Failure cases: API permission denied, upload failures, invalid folder ID  
  - Credentials: OAuth2 Google Drive account  

---

#### 2.6 Workflow Execution Control & Notes

**Overview:**  
Manages workflow execution modes and documents the workflow purpose.

**Nodes Involved:**  
- Sticky Note  

**Node Details:**

- **Sticky Note**  
  - Type: Sticky note (documentation node)  
  - Configuration: Describes workflow purpose and notes on folder duplication prevention  
  - Position: Visual annotation, no data input/output  

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                            | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                         |
|-------------------------------|--------------------------------|-------------------------------------------|---------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------|
| Gmail Trigger                 | Gmail Trigger                  | Start workflow on new Gmail email         | None                           | Contain attachments?, Split Out | ## Organize Gmail Attachments in Google Drive Folders based on sender’s email                      |
| When clicking ‘Execute workflow’ | Manual Trigger                 | Manual start of workflow                   | None                           | Get emails                     | ## Organize Gmail Attachments in Google Drive Folders based on sender’s email                      |
| Get emails                   | Gmail                         | Fetch emails by date range, download attachments | When clicking ‘Execute workflow’ | Loop Over Items                |                                                                                                  |
| Loop Over Items              | SplitInBatches                | Process emails one by one                   | Get emails, Execute Workflow    | Get email                     |                                                                                                  |
| Get email                   | Gmail                         | Fetch detailed email with attachments     | Loop Over Items                 | Contain attachments?, Split Out |                                                                                                  |
| Contain attachments?         | If                            | Check if email contains attachments       | Get email                      | Search folders (yes), Loop Over Items (no) |                                                                                                  |
| Split Out                   | SplitOut                      | Split attachments into separate items      | Contain attachments?            | Get attachments               |                                                                                                  |
| Get attachments             | Code                          | Extract each attachment as separate item  | Split Out                      | Merge                         |                                                                                                  |
| Search folders             | Google Drive                  | Search Drive folders for sender’s email    | Contain attachments?            | Exist?                        |                                                                                                  |
| Exist?                     | If                            | Check folder existence                      | Search folders                 | Folder ID (yes), Create folder (no) |                                                                                                  |
| Create folder              | Google Drive                  | Create folder named after sender’s email   | Exist? (no)                    | Folder ID                     |                                                                                                  |
| Folder ID                  | Set                           | Store folder ID for uploads                 | Exist? (yes), Create folder    | Merge                        |                                                                                                  |
| Merge                      | Merge                         | Combine extracted attachments and folder ID | Folder ID, Get attachments      | Execute Workflow             |                                                                                                  |
| Execute Workflow           | Execute Workflow              | Recursively process each attachment upload | Merge                         | Loop Over Items               |                                                                                                  |
| When Executed by Another Workflow | Execute Workflow Trigger      | Triggered when called by another workflow  | Execute Workflow               | Upload attachment             |                                                                                                  |
| Upload attachment          | Google Drive                  | Upload attachment to sender’s folder        | When Executed by Another Workflow | None                        |                                                                                                  |
| Sticky Note                | Sticky Note                   | Describes workflow purpose and notes       | None                          | None                         | ## Organize Gmail Attachments in Google Drive Folders based on sender’s email; avoids duplicates |


---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Set to poll every minute without filter  
   - Use OAuth2 credentials for Gmail  

2. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters needed  

3. **Create Get emails Node**  
   - Type: Gmail  
   - Operation: getAll  
   - Filter emails with receivedAfter = “2025-07-06T00:00:00” and receivedBefore = “2025-07-09T23:00:00”  
   - Options: downloadAttachments = true  
   - Connect output from manual trigger to this node  
   - Use OAuth2 Gmail credentials  

4. **Create Loop Over Items Node**  
   - Type: SplitInBatches  
   - Options: reset = false  
   - Connect output from Get emails node (or Execute Workflow node in recursive case) as input  

5. **Create Get email Node**  
   - Type: Gmail  
   - Operation: get  
   - Parameter: messageId = `={{ $json.id }}`  
   - Options: downloadAttachments = true  
   - Connect output of Loop Over Items (second output) to this node  
   - Use OAuth2 Gmail credentials  

6. **Create Contain attachments? Node**  
   - Type: If  
   - Condition: Check if `$binary` is not empty  
   - Connect output from Get email node  

7. **Create Search folders Node**  
   - Type: Google Drive  
   - Resource: fileFolder  
   - Operation: search  
   - Query string: `={{ $('Get email').item.json.from.value[0].address }}`  
   - FolderId: Set to your parent folder (e.g., “Email Attachments” folder ID)  
   - Limit: 1  
   - Connect “true” output of Contain attachments? to this node  
   - Use OAuth2 Google Drive credentials  

8. **Create Exist? Node**  
   - Type: If  
   - Condition: Check if search result is not empty  
   - Connect output from Search folders node  

9. **Create Create folder Node**  
   - Type: Google Drive  
   - Resource: folder  
   - Name: `={{ $('Get email').item.json.from.value[0].address }}`  
   - Parent folder: same as Search folders node  
   - Connect “false” output from Exist? node to this node  
   - Use OAuth2 Google Drive credentials  

10. **Create Folder ID Node**  
    - Type: Set  
    - Assign field: `folder_id` from `{{$json.id}}`  
    - Connect “true” output from Exist? and output from Create folder node to this node (merge outputs)  

11. **Create Split Out Node**  
    - Type: SplitOut  
    - Field to split: `$binary`  
    - Connect “true” output of Contain attachments? node  

12. **Create Get attachments Node**  
    - Type: Code  
    - JavaScript code:  
      ```js
      let results = [];
      for (const item of items) {
          for (const key of Object.keys(item.binary)) {
              results.push({
                  json: { fileName: item.binary[key].fileName },
                  binary: { data: item.binary[key] },
              });
          }
      }
      return results;
      ```  
    - Connect output from Split Out node  

13. **Create Merge Node**  
    - Type: Merge  
    - Mode: Combine (combineAll)  
    - Connect outputs from Folder ID node and Get attachments node  

14. **Create Execute Workflow Node**  
    - Type: Execute Workflow  
    - Mode: Each  
    - Wait for subworkflow: true  
    - Workflow ID: Set to this workflow’s ID (self-reference)  
    - Connect output from Merge node  

15. **Create When Executed by Another Workflow Node**  
    - Type: Execute Workflow Trigger  
    - Input source: passthrough  
    - Connect output from Execute Workflow node  

16. **Create Upload attachment Node**  
    - Type: Google Drive  
    - Operation: upload  
    - Name: `={{ $data.fileName }}`  
    - Folder ID: `={{ $json.folder_id }}`  
    - Input Data Field Name: `data`  
    - Connect output from When Executed by Another Workflow node  
    - Use OAuth2 Google Drive credentials  

17. **Create Sticky Note Node**  
    - Add explanatory note about the workflow purpose and folder duplication avoidance  

18. **Connect Gmail Trigger and Manual Trigger outputs to the proper starting nodes:**  
    - Gmail Trigger → Contain attachments?, Split Out  
    - Manual Trigger → Get emails  

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow automatically organizes Gmail attachments into Google Drive folders named after the sender's email address.| Workflow purpose summary                                                                             |
| Avoids duplicate folders by checking for the existence of a folder before creating a new one.                        | Important design note                                                                                |
| Uses OAuth2 authentication for both Gmail and Google Drive integrations.                                             | Credential setup requirement                                                                         |
| Google Drive parent folder "Email Attachments" must exist and its folder ID configured in Search folders and Create folder nodes. | Folder structure prerequisite                                                                        |
| Watch out for Gmail API limits and Google Drive API quota when processing large volumes of emails or attachments.   | Operational consideration                                                                            |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.