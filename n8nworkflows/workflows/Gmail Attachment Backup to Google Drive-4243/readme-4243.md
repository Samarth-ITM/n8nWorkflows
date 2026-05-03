Gmail Attachment Backup to Google Drive

https://n8nworkflows.xyz/workflows/gmail-attachment-backup-to-google-drive-4243


# Gmail Attachment Backup to Google Drive

### 1. Workflow Overview

This workflow automates the backup of email attachments from a specific Gmail sender to a designated folder in Google Drive. It is designed for users who want to automatically save attachments from incoming emails without manual intervention.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception:** Monitors Gmail inbox for new emails from a specified sender.
- **1.2 Email Retrieval:** Fetches the full email details including attachments.
- **1.3 Attachment Processing:** Extracts and restructures attachments for upload.
- **1.4 Backup to Google Drive:** Uploads attachments to a specified Google Drive folder and allows for post-upload processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Watches for new emails from a defined sender address using Gmail’s trigger. This initiates the workflow whenever a matching email arrives.

- **Nodes Involved:**  
  - Gmail Trigger

- **Node Details:**

  - **Gmail Trigger**  
    - *Type & Role:* Trigger node that listens for new Gmail messages.  
    - *Configuration:* Filters emails by the sender address `akhilgadiraju@gmail.com`. Polls every minute to check for new messages.  
    - *Expressions/Variables:* Uses static sender filter; no dynamic expressions here.  
    - *Connections:* Outputs to the `Gmail` node.  
    - *Version:* 1.2  
    - *Potential Failures:* Authentication errors if OAuth token expires, Gmail API rate limits, network timeouts.  
    - *Notes:* The sender email filter is customizable (see Sticky Note).

#### 2.2 Email Retrieval

- **Overview:**  
  Retrieves full email content and attachments based on the message ID received from the trigger.

- **Nodes Involved:**  
  - Gmail

- **Node Details:**

  - **Gmail**  
    - *Type & Role:* Performs a “get” operation on the Gmail API to fetch email details including attachments.  
    - *Configuration:*  
      - `messageId` is set dynamically using the ID from the trigger (`={{ $json.id }}`).  
      - Downloads all attachments and prefixes attachment data properties with `attachment_`.  
      - Simple mode disabled for full message data.  
    - *Connections:* Outputs to the `Code` node.  
    - *Version:* 2.1  
    - *Potential Failures:* OAuth token issues, large attachments causing timeout, API limits.  
    - *Notes:* Uses the same Gmail OAuth2 credentials as the trigger.

#### 2.3 Attachment Processing

- **Overview:**  
  Converts the binary attachment data from a single combined item into multiple items each containing one attachment, preparing for individual upload.

- **Nodes Involved:**  
  - Code

- **Node Details:**

  - **Code**  
    - *Type & Role:* JavaScript code node that transforms the incoming data structure.  
    - *Configuration:*  
      - Uses JavaScript to iterate over all binary properties of the first input item and maps each to a new item with the binary data under the key `data`.  
      - Code snippet:
        ```js
        return Object.entries(items[0].binary).map(([key, value]) => {
          return {
            binary: {
              data: value
            }
          };
        });
        ```  
    - *Connections:* Outputs to the `Google Drive` node.  
    - *Version:* 2  
    - *Potential Failures:* If no attachments exist, output will be empty; binary data keys must exist.  
    - *Notes:* Essential for separating multiple attachments for upload.

#### 2.4 Backup to Google Drive

- **Overview:**  
  Uploads each attachment as a separate file into a specified Google Drive folder, naming files based on email ID and current timestamp. Provides a placeholder for post-upload actions.

- **Nodes Involved:**  
  - Google Drive  
  - Replace Me (NoOp)  
  - Sticky Notes (for user guidance)

- **Node Details:**

  - **Google Drive**  
    - *Type & Role:* Uploads binary data as files to Google Drive.  
    - *Configuration:*  
      - Filename is dynamically generated as: `{emailId}_{currentTimestamp}_backup_attachment`.  
      - Uploads to a specific folder with ID `1aZmIqT9jG-GqW_OIGT3HWvRb6JalTlBi` (named “DOcs”).  
      - Uses “My Drive” as root drive context.  
      - Reads binary data from the `data` field (set by the Code node).  
    - *Connections:* Outputs to the `Replace Me` node.  
    - *Version:* 3  
    - *Potential Failures:* Google Drive API quota/errors, permission issues, invalid folder ID, name conflicts.  
    - *Notes:* Folder destination and filename format are customizable as per sticky notes.

  - **Replace Me**  
    - *Type & Role:* No operation placeholder.  
    - *Configuration:* Empty by default.  
    - *Connections:* No outputs.  
    - *Version:* 1  
    - *Potential Failures:* None, but intended for extension with notifications or logging.  
    - *Notes:* Suggested to replace with post-upload business logic.

- **Sticky Notes:**  
  - *Change sender filter* (near Gmail Trigger) — instructs on modifying the sender email.  
  - *Change destination folder and filename format* (near Google Drive) — guides updating folder ID and filename expression.  
  - *Add post-upload logic* (near Replace Me) — suggests extending workflow after upload.

---

### 3. Summary Table

| Node Name     | Node Type                 | Functional Role                   | Input Node(s)       | Output Node(s)  | Sticky Note                                                     |
|---------------|---------------------------|---------------------------------|---------------------|-----------------|----------------------------------------------------------------|
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Watches for new emails from sender | -                   | Gmail           | Change sender filter: Modify sender field in this node         |
| Gmail         | n8n-nodes-base.gmail       | Retrieves full email and attachments | Gmail Trigger       | Code            |                                                                |
| Code          | n8n-nodes-base.code        | Splits attachments into separate items | Gmail               | Google Drive    |                                                                |
| Google Drive  | n8n-nodes-base.googleDrive | Uploads attachments to Drive folder | Code                | Replace Me      | Change destination folder and filename format                  |
| Replace Me    | n8n-nodes-base.noOp        | Placeholder for post-upload logic  | Google Drive        | -               | Add post-upload logic: Replace or extend this node             |
| Sticky Note   | n8n-nodes-base.stickyNote  | Guidance for sender filter        | -                   | -               | Change sender filter                                            |
| Sticky Note1  | n8n-nodes-base.stickyNote  | Guidance for folder & filename    | -                   | -               | Change destination folder and filename format                  |
| Sticky Note2  | n8n-nodes-base.stickyNote  | Guidance for post-upload logic    | -                   | -               | Add post-upload logic                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**  
   - Type: Gmail Trigger  
   - Credentials: Gmail OAuth2 (with appropriate token)  
   - Parameters:  
     - Filters → Sender: `akhilgadiraju@gmail.com` (or desired sender)  
     - Poll Times → Every Minute  
   - Connect output to next node.

2. **Create Gmail node**  
   - Type: Gmail  
   - Credentials: Same Gmail OAuth2 as trigger  
   - Parameters:  
     - Operation: Get  
     - Message ID: Expression → `={{ $json.id }}`  
     - Options: Enable “Download Attachments”  
     - Data Property Attachments Prefix: `attachment_`  
     - Simple: false (to get full message details)  
   - Connect output from Gmail Trigger.

3. **Create Code node**  
   - Type: Code  
   - Parameters: JavaScript code to split binary attachments:
     ```js
     return Object.entries(items[0].binary).map(([key, value]) => {
       return {
         binary: {
           data: value
         }
       };
     });
     ```  
   - Connect output from Gmail node.

4. **Create Google Drive node**  
   - Type: Google Drive  
   - Credentials: Google Drive OAuth2 (with folder write permissions)  
   - Parameters:  
     - Operation: Upload file  
     - File Name: Expression → `={{ $('Gmail Trigger').item.json.id + "_" + $now + "_backup_attachment" }}`  
     - Folder ID: Select or enter desired Google Drive folder ID (e.g., `1aZmIqT9jG-GqW_OIGT3HWvRb6JalTlBi`)  
     - Drive ID: Select “My Drive”  
     - Input Data Field Name: `data`  
   - Connect output from Code node.

5. **Create Replace Me node**  
   - Type: NoOp (No operation)  
   - Parameters: None  
   - Connect output from Google Drive node.  
   - Optionally replace or extend this node for notifications, logging, or other post-upload logic.

6. **Add Sticky Notes** (Optional but recommended for maintainability)  
   - Next to Gmail Trigger: Note to remind users to change sender filter.  
   - Next to Google Drive: Note about changing destination folder and filename format.  
   - Next to Replace Me: Note about adding post-upload logic.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                   |
|------------------------------------------------------------------------------|------------------------------------------------------------------|
| Modify sender filter to target different email addresses                    | See Sticky Note near Gmail Trigger node                          |
| Customize Google Drive folder and filename format per your backup needs     | See Sticky Note near Google Drive node                           |
| Extend post-upload logic for notifications, logging, or cleanup             | See Sticky Note near Replace Me node                             |
| Gmail OAuth2 and Google Drive OAuth2 credentials must be correctly configured | Use n8n Credentials Manager; ensure token scopes include access   |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automation workflow. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.