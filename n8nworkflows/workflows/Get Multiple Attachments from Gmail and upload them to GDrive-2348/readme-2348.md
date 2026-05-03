Get Multiple Attachments from Gmail and upload them to GDrive

https://n8nworkflows.xyz/workflows/get-multiple-attachments-from-gmail-and-upload-them-to-gdrive-2348


# Get Multiple Attachments from Gmail and upload them to GDrive

### 1. Workflow Overview

This workflow automates the extraction of multiple attachments from new Gmail emails and uploads each attachment individually to Google Drive. It is designed for users who need to process emails containing attachments regularly and store those attachments in a cloud drive for further use or archival.

The workflow consists of three logical blocks:

- **1.1 Input Reception:** Detect new incoming Gmail messages with attachments.
- **1.2 Attachment Extraction:** Extract and split multiple binary attachments from the email payload.
- **1.3 Upload Processing:** Upload each extracted attachment file to a specified folder in Google Drive, renaming files to include sender information for better traceability.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a new email with attachments arrives in the Gmail inbox. It ensures that only emails containing attachments are processed and downloads those attachments automatically.

- **Nodes Involved:**  
  - Trigger - New Email

- **Node Details:**

  - **Node Name:** Trigger - New Email  
    - **Type:** Gmail Trigger  
    - **Technical Role:** Event listener that watches for new Gmail messages matching specific criteria.  
    - **Configuration:**  
      - Filter query set to `"has:attachment"` to only trigger on emails that include attachments.  
      - `simple` flag set to false to receive detailed email data.  
      - `downloadAttachments` option enabled to automatically download email attachments.  
      - Polling set to run every minute to check for new emails.  
    - **Key Expressions/Variables:** None explicitly, but email data with attachments is output as binary data.  
    - **Input Connections:** None (trigger node).  
    - **Output Connections:** Connects to "attach binary data outputs" node.  
    - **Version Requirements:** Uses Gmail API; requires Gmail credentials configured with appropriate scopes for reading emails and attachments.  
    - **Potential Failure Modes:**  
      - Authentication errors if Gmail credentials expire or are misconfigured.  
      - API rate limits if polling frequency is too high or account has restrictions.  
      - Missing attachment data if email format is unsupported.  
    - **Sub-workflow:** None.

#### 2.2 Attachment Extraction

- **Overview:**  
  This block processes the incoming email data to isolate each attachment as a separate binary item, enabling individual upload operations downstream.

- **Nodes Involved:**  
  - attach binary data outputs

- **Node Details:**

  - **Node Name:** attach binary data outputs  
    - **Type:** Function Node  
    - **Technical Role:** Iterates through all binary attachments in the incoming items and outputs each as a separate item with binary data and associated filename metadata.  
    - **Configuration:** Custom JavaScript code that:  
      - Loops through each incoming item.  
      - For each binary property (attachment), creates a new output item containing:  
        - `json.fileName`: original filename of the attachment.  
        - `binary.data`: the binary content of that attachment.  
      - Returns an array of output items, each representing one attachment file.  
    - **Key Expressions/Variables:**  
      - Uses `items` input array.  
      - Accesses `item.binary` keys dynamically.  
      - Outputs array named `results`.  
    - **Input Connections:** Receives output from "Trigger - New Email".  
    - **Output Connections:** Connects to "upload files to google drive".  
    - **Version Requirements:** Standard JavaScript environment in n8n Function node; no special version constraints.  
    - **Potential Failure Modes:**  
      - If no binary data exists, returns empty array, stopping further processing.  
      - If attachment metadata lacks `fileName`, downstream nodes may fail or generate incorrect names.  
      - Expression errors if unexpected data structure is encountered.  
    - **Sub-workflow:** None.

#### 2.3 Upload Processing

- **Overview:**  
  This block uploads each extracted attachment to Google Drive, renaming files to include the original filename and the sender's email address to avoid naming collisions and aid identification.

- **Nodes Involved:**  
  - upload files to google drive

- **Node Details:**

  - **Node Name:** upload files to google drive  
    - **Type:** Google Drive Node  
    - **Technical Role:** Uploads binary data as files to a specified Google Drive folder.  
    - **Configuration:**  
      - `name`: Custom expression combines the original filename (without extension), a hyphen, the sender’s email address extracted from the Gmail trigger node (`$node["Trigger - New Email"].item.json.from.value[0].address`), and the original file extension. This creates a unique, descriptive filename.  
      - `driveId`: Set to "My Drive" (default Google Drive root).  
      - `folderId`: Set to "root" (Google Drive root folder).  
      - No additional options configured.  
    - **Key Expressions/Variables:**  
      - Uses `$json.fileName` from the current item.  
      - Accesses sender email from the Gmail trigger node’s JSON path.  
    - **Input Connections:** Receives multiple items from "attach binary data outputs" (one per attachment).  
    - **Output Connections:** None (final step).  
    - **Version Requirements:** Requires Google Drive OAuth2 credentials with file upload permissions.  
    - **Potential Failure Modes:**  
      - Authentication/authorization errors if credentials expire or lack permissions.  
      - Filename expression could fail if `fileName` or sender email is missing or malformed.  
      - API quota limits or upload size restrictions.  
      - Upload failure if binary data is corrupted or missing.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                          | Input Node(s)            | Output Node(s)               | Sticky Note                                                                                   |
|----------------------------|---------------------|----------------------------------------|--------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Trigger - New Email         | Gmail Trigger       | Detect new emails with attachments     | None                     | attach binary data outputs   | has:attachment                                                                                |
| attach binary data outputs  | Function            | Extract and split attachments as items | Trigger - New Email       | upload files to google drive |                                                                                              |
| upload files to google drive| Google Drive        | Upload attachments to Google Drive     | attach binary data outputs| None                        |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: "Trigger - New Email"**  
   - Type: Gmail Trigger  
   - Credentials: Add and select your Gmail OAuth2 credentials with read and attachment permissions.  
   - Parameters:  
     - Set filter query to `has:attachment` to only trigger on emails that include attachments.  
     - Enable `downloadAttachments` option.  
     - Set `simple` to false to get detailed email data.  
     - Configure polling to check every minute (`pollTimes` → mode: everyMinute).

2. **Create Function Node: "attach binary data outputs"**  
   - Type: Function  
   - Connect input to "Trigger - New Email" output.  
   - Paste the following JavaScript code into the function editor:

   ```javascript
   let results = [];

   for (item of items) {
       for (key of Object.keys(item.binary)) {
           results.push({
               json: {
                   fileName: item.binary[key].fileName
               },
               binary: {
                   data: item.binary[key],
               }
           });
       }
   }

   return results;
   ```

   - This code iterates over all binary attachments and outputs each as a separate item.

3. **Create Google Drive Node: "upload files to google drive"**  
   - Type: Google Drive  
   - Connect input to "attach binary data outputs" output.  
   - Credentials: Add and select your Google Drive OAuth2 credentials with file upload permissions.  
   - Parameters:  
     - **Name:** Use the expression editor and enter:

       ```
       {{$json.fileName.split(".")[0] + "-" + $node["Trigger - New Email"].item.json.from.value[0].address + "." + $json.fileName.split(".")[1]}}
       ```

       This renames the file to: `[originalFilename]-[senderEmail].[extension]`.  
     - **Drive:** Select "My Drive".  
     - **Folder:** Select root folder or specify a subfolder as needed.  
     - Leave other options as default.

4. **Connect Nodes**  
   - Connect "Trigger - New Email" → "attach binary data outputs".  
   - Connect "attach binary data outputs" → "upload files to google drive".

5. **Activate the Workflow**  
   - Save and activate the workflow.  
   - Ensure credentials are authorized and have proper scopes.  
   - Test by sending an email with attachments to the Gmail account.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                         |
|------------------------------------------------------------------------------|----------------------------------------------------------------------------------------|
| Follow the official n8n documentation for Google credentials and OAuth setup | https://docs.n8n.io/integrations/builtin/credentials/google/                           |
| Workflow author contact for feedback and questions                           | ria@n8n.io                                                                             |

---

This documentation provides a detailed reference to understand, reproduce, and extend the workflow that extracts multiple attachments from Gmail and uploads them to Google Drive, ensuring smooth operation and easy troubleshooting.