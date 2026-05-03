Automatically Save & Organize Outlook Email Attachments in OneDrive Folders

https://n8nworkflows.xyz/workflows/automatically-save---organize-outlook-email-attachments-in-onedrive-folders-6938


# Automatically Save & Organize Outlook Email Attachments in OneDrive Folders

### 1. Workflow Overview

This workflow automates the process of saving and organizing email attachments received in Microsoft Outlook by storing them into newly created folders in OneDrive. It is designed for users who want to automatically archive attachments from incoming emails into organized cloud storage folders without manual intervention.

**Logical Blocks:**

- **1.1 Input Reception:** Triggered by new Outlook emails every minute.
- **1.2 Filtering:** Checks if the email contains attachments (binary data).
- **1.3 Email Retrieval:** Fetches full email content including attachments.
- **1.4 Folder Creation:** Creates a new dedicated folder in OneDrive named after the email subject and timestamp.
- **1.5 Attachment Splitting:** Splits multiple attachments for individual processing.
- **1.6 Data Merging:** Combines folder metadata and attachment data.
- **1.7 File Upload:** Uploads each attachment to the corresponding OneDrive folder.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:**  
  Listens for new emails arriving in the configured Outlook account, polling every minute, and downloads attachments immediately.

- **Nodes Involved:**  
  - Microsoft Outlook Trigger

- **Node Details:**

  - **Microsoft Outlook Trigger**  
    - *Type:* Trigger node for Microsoft Outlook  
    - *Configuration:* Polls every minute; downloads attachments on trigger; no filters applied initially.  
    - *Expressions:* None directly; outputs full email data including binary if attachments exist.  
    - *Inputs:* None (trigger node)  
    - *Outputs:* Passes new email data to the Filter node  
    - *Version-specifics:* Uses version 1 of the node  
    - *Edge Cases:* Potential connectivity issues with Outlook API, authentication failures, or missing permission to access mailbox. Polling frequency may impact rate limits.

---

#### Block 1.2: Filtering

- **Overview:**  
  Filters incoming emails to process only those that have one or more attachments.

- **Nodes Involved:**  
  - Filter

- **Node Details:**

  - **Filter**  
    - *Type:* Conditional filter node  
    - *Configuration:* Checks if the `binary` property exists in the Outlook Trigger output, indicating attachment presence.  
    - *Expressions:* Uses expression `={{ $('Microsoft Outlook Trigger').item.binary}}` to check for existence of binary data.  
    - *Inputs:* Receives email data from Microsoft Outlook Trigger  
    - *Outputs:* Passes only emails with attachments to the next step  
    - *Version-specifics:* Version 2.2, supports enhanced conditions  
    - *Edge Cases:* Expression failure if the trigger output structure changes; may filter out emails with unsupported attachment types.

---

#### Block 1.3: Email Retrieval

- **Overview:**  
  Retrieves the full details of the email, including downloading attachments again with explicit request, ensuring all binary data is accessible.

- **Nodes Involved:**  
  - Get Outlook Message

- **Node Details:**

  - **Get Outlook Message**  
    - *Type:* Microsoft Outlook node (operation: get)  
    - *Configuration:* Downloads attachments explicitly, uses `messageId` from the trigger node to fetch the exact email.  
    - *Expressions:* `={{ $node["Microsoft Outlook Trigger"].json.id }}` to get the email ID dynamically.  
    - *Inputs:* Output from Filter node (filtered emails)  
    - *Outputs:* Provides detailed email content including attachments for further processing  
    - *Version-specifics:* Version 2, supports webhook ID for direct message retrieval  
    - *Edge Cases:* API limits, message ID missing or invalid, attachment download failures.

---

#### Block 1.4: Folder Creation

- **Overview:**  
  Creates a dedicated OneDrive folder for each email to organize attachments.

- **Nodes Involved:**  
  - Create Folder

- **Node Details:**

  - **Create Folder**  
    - *Type:* Microsoft OneDrive resource node (folder creation)  
    - *Configuration:* Folder name is dynamically generated using the email subject or defaults to "Untitled", appended with the current timestamp.  
    - *Expressions:* `={{ $node["Microsoft Outlook Trigger"].json.subject || "Untitled" }} - {{ $now }}`  
    - *Inputs:* Receives data from Get Outlook Message node  
    - *Outputs:* Returns metadata of the created folder (including folder ID)  
    - *Version-specifics:* Version 1  
    - *Edge Cases:* Folder name conflicts, permission denied errors, API limits, invalid characters in subject causing folder creation failure.

---

#### Block 1.5: Attachment Splitting

- **Overview:**  
  Splits the attachments (binary data) into separate items for individual upload processing.

- **Nodes Involved:**  
  - Split Out

- **Node Details:**

  - **Split Out**  
    - *Type:* Split Out node, used to separate array-type data into single items  
    - *Configuration:* Splits the `$binary` field of the email data; includes binary data for each output item.  
    - *Expressions:* Field to split out is `$binary`  
    - *Inputs:* Receives data from Get Outlook Message node  
    - *Outputs:* Provides one output item per attachment for downstream processing  
    - *Version-specifics:* Version 1  
    - *Edge Cases:* No attachments present, empty binary field, binary data corruption.

---

#### Block 1.6: Data Merging

- **Overview:**  
  Combines folder creation output and split attachment items to associate each attachment with the correct OneDrive folder.

- **Nodes Involved:**  
  - Outlook Folder Merge

- **Node Details:**

  - **Outlook Folder Merge**  
    - *Type:* Merge node (combine mode)  
    - *Configuration:* Combines all incoming data from Create Folder and Split Out nodes into one unified data set for upload.  
    - *Mode:* Combine all inputs into one output stream  
    - *Inputs:*  
      - Input 0: Create Folder output  
      - Input 1: Split Out output  
    - *Outputs:* Merged data used by Upload File OneDrive node  
    - *Version-specifics:* Version 3.1  
    - *Edge Cases:* If either input is missing or empty, merging may fail or produce incomplete data.

---

#### Block 1.7: File Upload

- **Overview:**  
  Uploads each attachment binary as a file into the corresponding OneDrive folder created earlier.

- **Nodes Involved:**  
  - Upload File OneDrive

- **Node Details:**

  - **Upload File OneDrive**  
    - *Type:* Microsoft OneDrive resource node (file upload)  
    - *Configuration:*  
      - `fileName` dynamically set from the binary file name of the current attachment.  
      - `parentId` set to the OneDrive folder ID from the merged node output (either from Create Folder or Outlook Folder Merge).  
      - Uploads binary data from the `binary` property in the merged item.  
    - *Expressions:*  
      - `={{ $binary[$binary.keys()[0]].fileName }}` for file name  
      - `={{ $node["Outlook Folder Merge"].json["id"] || $node["Create Folder"].json["id"] }}` for parent folder ID  
      - Binary property name is set dynamically to `{{$json.binary}}`  
    - *Inputs:* From Outlook Folder Merge node  
    - *Outputs:* None (end of workflow)  
    - *Version-specifics:* Version 1  
    - *Edge Cases:* Upload failure due to permissions, incorrect folder ID, binary data missing or corrupted, network timeouts.

---

### 3. Summary Table

| Node Name              | Node Type                     | Functional Role                           | Input Node(s)               | Output Node(s)          | Sticky Note                                                                                      |
|------------------------|-------------------------------|-----------------------------------------|-----------------------------|-------------------------|-------------------------------------------------------------------------------------------------|
| Microsoft Outlook Trigger | Microsoft Outlook Trigger     | Polls new emails every minute; triggers workflow | None                        | Filter                  |                                                                                                 |
| Filter                 | Filter                        | Filters emails to those containing attachments (binary) | Microsoft Outlook Trigger   | Get Outlook Message      |                                                                                                 |
| Get Outlook Message     | Microsoft Outlook             | Retrieves full email including attachments | Filter                      | Create Folder, Split Out |                                                                                                 |
| Create Folder          | Microsoft OneDrive            | Creates OneDrive folder named by email subject and time | Get Outlook Message          | Outlook Folder Merge     |                                                                                                 |
| Split Out              | Split Out                    | Splits attachments into individual items | Get Outlook Message          | Outlook Folder Merge     |                                                                                                 |
| Outlook Folder Merge   | Merge                         | Combines folder metadata and attachment data | Create Folder, Split Out     | Upload File OneDrive     |                                                                                                 |
| Upload File OneDrive   | Microsoft OneDrive            | Uploads each attachment file into OneDrive folder | Outlook Folder Merge         | None                    |                                                                                                 |
| Sticky Note            | Sticky Note                   | Describes general workflow purpose      | None                        | None                    | Outlook and Onedrive binary movement of data                                                   |
| Sticky Note1           | Sticky Note                   | Summary of workflow logic and data flow | None                        | None                    | WORKFLOW SUMMARY<br>This workflow is triggered every minute by new emails in Outlook. It first filters for emails that contain binary data (attachments). For such emails, it retrieves the full email content and then proceeds to create a new folder in OneDrive. Concurrently, it splits out the binary data (attachments) from the email. Finally, it merges the information about the newly created OneDrive folder with the attachment data and saves each attachment into the designated OneDrive folder. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Microsoft Outlook Trigger node:**  
   - Set it to poll every minute (under Poll Times).  
   - Enable downloading attachments on trigger (option: downloadAttachments = true).  
   - No filters at this stage.

3. **Add a Filter node:**  
   - Connect input from Microsoft Outlook Trigger.  
   - Configure condition: check if `binary` property exists in the incoming data (`={{ $('Microsoft Outlook Trigger').item.binary}}`).  
   - Ensure case sensitivity and strict validation enabled.

4. **Add a Microsoft Outlook node (Get Outlook Message):**  
   - Connect input from Filter node (only emails with attachments).  
   - Operation: Get message.  
   - Set `messageId` to `={{ $node["Microsoft Outlook Trigger"].json.id }}` to refer to the triggered email.  
   - Enable attachment download explicitly.

5. **Add a Microsoft OneDrive node (Create Folder):**  
   - Connect input from Get Outlook Message node.  
   - Resource: Folder.  
   - Operation: Create.  
   - Folder name: Use expression `={{ $node["Microsoft Outlook Trigger"].json.subject || "Untitled" }} - {{ $now }}` to create a folder named after the email subject or "Untitled" with a timestamp.

6. **Add a Split Out node:**  
   - Connect input from Get Outlook Message node.  
   - Set field to split out as `$binary`.  
   - Enable including binary data in output.

7. **Add a Merge node (Outlook Folder Merge):**  
   - Connect inputs:  
     - Input 0: Create Folder output  
     - Input 1: Split Out output  
   - Set mode to "Combine" to merge all data.

8. **Add a Microsoft OneDrive node (Upload File OneDrive):**  
   - Connect input from Merge node.  
   - Resource: File.  
   - Operation: Upload.  
   - File name: Use expression `={{ $binary[$binary.keys()[0]].fileName }}` to get the attachment file name.  
   - Parent ID: Use expression `={{ $node["Outlook Folder Merge"].json["id"] || $node["Create Folder"].json["id"] }}` to reference the folder ID.  
   - Binary property name: Set dynamically as `{{$json.binary}}`.  
   - Enable uploading binary data.

9. **(Optional) Add Sticky Note nodes:**  
   - Add descriptive sticky notes to document workflow purpose and block summaries for clarity.

10. **Credential Setup:**  
    - Configure Microsoft Outlook credentials with appropriate OAuth2 access permissions (Mail.Read, Mail.ReadWrite).  
    - Configure Microsoft OneDrive credentials with permissions to create folders and upload files in the target OneDrive account.

11. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow triggers every minute to check for new Outlook emails with attachments and saves them to OneDrive for organization. | General workflow purpose                                                                         |
| Folder names include email subject and timestamp to prevent conflicts and aid identification.                               | Folder creation convention                                                                       |
| Ensure OAuth2 credentials have sufficient permissions for mailbox reading and OneDrive file/folder operations.               | Credential best practices                                                                        |
| For more on Microsoft Outlook and OneDrive integrations with n8n, see official docs: https://docs.n8n.io/integrations/      | n8n official documentation on Microsoft integrations                                            |
| Error handling is recommended for production use, especially for API rate limits and network failures.                      | General advice for robustness                                                                    |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.