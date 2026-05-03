Organize Email Attachments into Google Drive Folders by Company with Gmail & Sheets

https://n8nworkflows.xyz/workflows/organize-email-attachments-into-google-drive-folders-by-company-with-gmail---sheets-3464


# Organize Email Attachments into Google Drive Folders by Company with Gmail & Sheets

### 1. Workflow Overview

This workflow automates the organization of email attachments from Gmail into structured Google Drive folders based on the sender’s company and the email date. It is designed for teams or individuals who need to efficiently sort invoices, receipts, or client documents by verifying sender emails against a whitelist and avoiding duplicate filenames by timestamping.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Email Retrieval:** Watches for new labeled emails with attachments in Gmail and retrieves full email data including attachments.
- **1.2 Whitelist Verification:** Checks the sender’s email against a Google Sheets whitelist to determine the associated company.
- **1.3 Company Folder Management:** Searches for or creates a Google Drive folder for the company.
- **1.4 Date-Based Folder Management:** Checks for or creates a year/month folder inside the company folder to organize files by date.
- **1.5 Attachment Processing and Upload:** Splits multiple attachments into individual items, timestamps filenames, and uploads them to the appropriate Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Email Retrieval

- **Overview:**  
  This block triggers the workflow on new Gmail messages labeled with a specific label (e.g., `Custom_Label`) and retrieves the full email content including attachments.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Gmail

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Trigger node for Gmail  
    - Configuration: Polls every 15 minutes for emails labeled with a specific Gmail label ID (`Label_2`).  
    - Inputs: None (trigger)  
    - Outputs: Emits new email metadata including message ID and sender info.  
    - Potential Failures: OAuth2 authentication errors, Gmail API rate limits, label ID misconfiguration.  
    - Notes: Requires Gmail OAuth2 credentials.

  - **Gmail**  
    - Type: Gmail node (operation: get message)  
    - Configuration: Retrieves full email data including attachments using the message ID from the trigger. Downloads attachments.  
    - Inputs: Message ID from Gmail Trigger  
    - Outputs: Full email JSON plus binary attachment data.  
    - Potential Failures: Message not found, API errors, attachment download failures.  
    - Notes: Requires same Gmail OAuth2 credentials as trigger.

---

#### 2.2 Whitelist Verification

- **Overview:**  
  This block verifies the sender’s email against a whitelist stored in Google Sheets to identify the associated company.

- **Nodes Involved:**  
  - Lookup in Sheets

- **Node Details:**

  - **Lookup in Sheets**  
    - Type: Google Sheets node (lookup operation)  
    - Configuration: Looks up the sender’s email address in the `email` column of a specified Google Sheet (whitelist). Returns the matching row containing `email` and `company`.  
    - Inputs: Sender email from Gmail Trigger  
    - Outputs: JSON with company name if found; empty if not found.  
    - Potential Failures: OAuth2 errors, sheet ID or name misconfiguration, no matching row found (empty output).  
    - Notes: Requires Google Sheets OAuth2 credentials. The whitelist sheet must have columns `email` and `company`.

---

#### 2.3 Company Folder Management

- **Overview:**  
  This block checks if a Google Drive folder exists for the company identified in the whitelist. If not, it creates the folder under a specified parent folder.

- **Nodes Involved:**  
  - Search Company Folder1  
  - Company Folder Exists (IF)  
  - Create Company Folder

- **Node Details:**

  - **Search Company Folder1**  
    - Type: Google Drive node (search folders)  
    - Configuration: Searches for folders matching the company name returned from the whitelist lookup.  
    - Inputs: Company name from Lookup in Sheets  
    - Outputs: Folder metadata if found; empty if not.  
    - Potential Failures: OAuth2 errors, search query failures.

  - **Company Folder Exists (IF)**  
    - Type: IF node  
    - Configuration: Checks if the search result is not empty (folder exists).  
    - Inputs: Output of Search Company Folder1  
    - Outputs: Two branches: true (folder exists), false (folder does not exist).  
    - Potential Failures: Expression evaluation errors.

  - **Create Company Folder**  
    - Type: Google Drive node (create folder)  
    - Configuration: Creates a folder named after the company under a fixed parent folder (ID: `18ry0AUtrpp3re6u3zQvvs0BQUGFmBKN9` - labeled “Invoices”).  
    - Inputs: Triggered only if folder does not exist (false branch of IF)  
    - Outputs: Metadata of the newly created folder.  
    - Potential Failures: OAuth2 errors, folder creation permission issues.

---

#### 2.4 Date-Based Folder Management

- **Overview:**  
  This block organizes attachments by date by checking for or creating a year/month folder inside the company folder.

- **Nodes Involved:**  
  - YYYY/MM (Set)  
  - Search For Folder  
  - Check If Folder Exists (IF)  
  - Create Month Folder

- **Node Details:**

  - **YYYY/MM (Set)**  
    - Type: Set node  
    - Configuration: Constructs a folder name string in the format `YYYY/MM` based on the email date from the Gmail Trigger.  
    - Inputs: Email date from Gmail Trigger  
    - Outputs: JSON with `folderName` property set to e.g. `2024/06`.  
    - Potential Failures: Date parsing errors if email date missing or malformed.

  - **Search For Folder**  
    - Type: Google Drive node (search folder)  
    - Configuration: Searches for a folder named `YYYY/MM` inside the company folder (folder ID passed dynamically).  
    - Inputs: Folder name from YYYY/MM, parent folder ID from company folder node  
    - Outputs: Folder metadata if found; empty if not.  
    - Potential Failures: OAuth2 errors, search failures.

  - **Check If Folder Exists (IF)**  
    - Type: IF node  
    - Configuration: Checks if the search result is not empty (folder exists).  
    - Inputs: Output of Search For Folder  
    - Outputs: Two branches: true (folder exists), false (folder does not exist).  
    - Potential Failures: Expression errors.

  - **Create Month Folder**  
    - Type: Google Drive node (create folder)  
    - Configuration: Creates the `YYYY/MM` folder inside the company folder if it does not exist.  
    - Inputs: Triggered only if folder does not exist (false branch of IF)  
    - Outputs: Metadata of the newly created folder.  
    - Potential Failures: OAuth2 errors, permission issues.

---

#### 2.5 Attachment Processing and Upload

- **Overview:**  
  This block processes each attachment individually, timestamps filenames to avoid duplicates, and uploads them to the appropriate Google Drive folder.

- **Nodes Involved:**  
  - Split Up Binary Data1 (Function)  
  - Loop Over Items (Split in Batches)  
  - Upload To Folder

- **Node Details:**

  - **Split Up Binary Data1**  
    - Type: Function node  
    - Configuration: Iterates over all binary attachments in the email, creating separate items each containing one attachment under the binary key `data`. Also passes the original filename.  
    - Inputs: Email with multiple attachments (binary data) from Gmail node  
    - Outputs: Multiple items, each with one attachment binary data and filename.  
    - Potential Failures: If no attachments present, outputs empty; expression errors in binary handling.

  - **Loop Over Items**  
    - Type: SplitInBatches node  
    - Configuration: Processes attachments one by one to avoid API rate limits or memory issues.  
    - Inputs: Items from Split Up Binary Data1  
    - Outputs: Single item per batch forwarded to upload node.  
    - Potential Failures: Batch size misconfiguration (default is 1), potential delays.

  - **Upload To Folder**  
    - Type: Google Drive node (upload file)  
    - Configuration: Uploads each attachment to the folder identified by either the existing or newly created `YYYY/MM` folder. Filenames are prefixed with the current timestamp (milliseconds since epoch) to ensure uniqueness. Also adds metadata properties `sender` and `time_received` to the file.  
    - Inputs: Single attachment item from Loop Over Items  
    - Outputs: Metadata of uploaded file.  
    - Potential Failures: OAuth2 errors, upload failures, filename conflicts (mitigated by timestamp prefix).

---

### 3. Summary Table

| Node Name             | Node Type                 | Functional Role                          | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                                       |
|-----------------------|---------------------------|----------------------------------------|------------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger         | Gmail Trigger             | Triggers workflow on labeled emails    | —                      | Lookup in Sheets          | # 1. Trigger Settings and Filters<br>Configure interval and filters to process only some emails.<br>Example Gmail filter steps.  |
| Lookup in Sheets      | Google Sheets             | Verifies sender email against whitelist| Gmail Trigger           | Search Company Folder1    | # 2. Google Sheets Whitelist Config<br>Copy example sheet or create one with columns `email` and `company`.                      |
| Search Company Folder1| Google Drive (search)     | Searches for company folder             | Lookup in Sheets        | Company Folder Exists     | # Checks if a folder with the company of the email exists<br>If it doesn't the directory is created                              |
| Company Folder Exists | IF                        | Checks if company folder exists         | Search Company Folder1  | YYYY/MM (true), Create Company Folder (false) |                                                                                                                  |
| Create Company Folder | Google Drive (create)     | Creates company folder if missing       | Company Folder Exists   | YYYY/MM                  | # Checks if a folder with the company of the email exists<br>If it doesn't the directory is created                              |
| YYYY/MM               | Set                       | Creates YYYY/MM folder name from email date | Company Folder Exists/Create Company Folder | Search For Folder          | # Checks if YYYY/MM Folder exists<br>If the directory doesn't exist it is created                                               |
| Search For Folder     | Google Drive (search)     | Searches for YYYY/MM folder              | YYYY/MM                 | Check If Folder Exists    |                                                                                                                                |
| Check If Folder Exists| IF                        | Checks if YYYY/MM folder exists          | Search For Folder       | Gmail (true), Create Month Folder (false) |                                                                                                                  |
| Create Month Folder   | Google Drive (create)     | Creates YYYY/MM folder if missing        | Check If Folder Exists  | Gmail                    |                                                                                                                                |
| Gmail                 | Gmail                     | Retrieves full email with attachments   | Check If Folder Exists  | Split Up Binary Data1     | ## Upload attachments to Drive<br>Incoming files are split into individual items with binary data under `data` key.<br>Filenames prefixed with timestamp. |
| Split Up Binary Data1 | Function                  | Splits multiple attachments into items  | Gmail                   | Loop Over Items           |                                                                                                                                |
| Loop Over Items       | SplitInBatches            | Processes attachments one by one         | Split Up Binary Data1   | Upload To Folder          |                                                                                                                                |
| Upload To Folder      | Google Drive (upload)     | Uploads attachments to Google Drive      | Loop Over Items         | Loop Over Items (loop)    |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Configure OAuth2 credentials for Gmail.  
   - Set polling interval to every 15 minutes.  
   - Add filter for label IDs to process only emails with a specific label (e.g., `Custom_Label`).  
   - Save.

2. **Create Google Sheets Lookup Node**  
   - Type: Google Sheets  
   - Configure OAuth2 credentials for Google Sheets.  
   - Set operation to lookup row by value.  
   - Set document ID to your whitelist Google Sheet.  
   - Set sheet name to the appropriate sheet (usually `Sheet1`).  
   - Lookup column: `email`  
   - Lookup value: `={{ $('Gmail Trigger').item.json.from.value[0].address }}`  
   - Save.

3. **Create Google Drive Search Node for Company Folder**  
   - Type: Google Drive (search folders)  
   - Configure OAuth2 credentials for Google Drive.  
   - Set resource to `fileFolder`.  
   - Set filter to search folders only.  
   - Query string: `={{ $('Lookup in Sheets').item.json.company }}`  
   - Save.

4. **Create IF Node to Check Company Folder Exists**  
   - Type: IF  
   - Condition: Check if output from Search Company Folder1 is not empty.  
   - True branch: Folder exists  
   - False branch: Folder does not exist  
   - Save.

5. **Create Google Drive Create Folder Node for Company Folder**  
   - Type: Google Drive (create folder)  
   - Configure OAuth2 credentials.  
   - Folder name: `={{ $('Lookup in Sheets').item.json.company }}`  
   - Parent folder ID: Set to your fixed parent folder ID (e.g., `18ry0AUtrpp3re6u3zQvvs0BQUGFmBKN9`)  
   - Save.

6. **Create Set Node to Generate YYYY/MM Folder Name**  
   - Type: Set  
   - Add field `folderName` with expression:  
     ```
     =new Date($('Gmail Trigger').item.json.date).getUTCFullYear() + '/' + String(new Date($('Gmail Trigger').item.json.date).getUTCMonth() + 1).padStart(2, '0')
     ```  
   - Save.

7. **Create Google Drive Search Node for YYYY/MM Folder**  
   - Type: Google Drive (search folders)  
   - Configure OAuth2 credentials.  
   - Search for folder named `={{ $json.folderName }}` inside the company folder (folder ID from previous step).  
   - Save.

8. **Create IF Node to Check YYYY/MM Folder Exists**  
   - Type: IF  
   - Condition: Check if output from Search For Folder is not empty.  
   - True branch: Folder exists  
   - False branch: Folder does not exist  
   - Save.

9. **Create Google Drive Create Folder Node for YYYY/MM Folder**  
   - Type: Google Drive (create folder)  
   - Configure OAuth2 credentials.  
   - Folder name: `={{ $('YYYY/MM').item.json.folderName }}`  
   - Parent folder ID: Folder ID from company folder node  
   - Save.

10. **Create Gmail Node to Retrieve Full Email**  
    - Type: Gmail (get message)  
    - Configure OAuth2 credentials.  
    - Message ID: `={{ $('Gmail Trigger').item.json.id }}`  
    - Enable download attachments option.  
    - Save.

11. **Create Function Node to Split Attachments**  
    - Type: Function  
    - Code:  
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
    - Save.

12. **Create SplitInBatches Node to Process Attachments One by One**  
    - Type: SplitInBatches  
    - Default batch size (1) is fine.  
    - Save.

13. **Create Google Drive Upload Node**  
    - Type: Google Drive (upload file)  
    - Configure OAuth2 credentials.  
    - File name: `={{ Date.now() + '-' + $('Loop Over Items').item.binary.data.fileName }}`  
    - Folder ID: Use folder ID from either Search For Folder or Create Month Folder node (use first available).  
    - Input data field name: `data` (matches binary key from function node)  
    - Add file properties:  
      - `sender`: `={{ $('Gmail').item.json.from.value[0].address }}`  
      - `time_received`: `={{ $('Gmail').item.json.date }}`  
    - Save.

14. **Connect Nodes According to Workflow Logic:**  
    - Gmail Trigger → Lookup in Sheets  
    - Lookup in Sheets → Search Company Folder1  
    - Search Company Folder1 → Company Folder Exists (IF)  
    - Company Folder Exists (true) → YYYY/MM (Set)  
    - Company Folder Exists (false) → Create Company Folder → YYYY/MM (Set)  
    - YYYY/MM (Set) → Search For Folder  
    - Search For Folder → Check If Folder Exists (IF)  
    - Check If Folder Exists (true) → Gmail  
    - Check If Folder Exists (false) → Create Month Folder → Gmail  
    - Gmail → Split Up Binary Data1  
    - Split Up Binary Data1 → Loop Over Items  
    - Loop Over Items → Upload To Folder  
    - Upload To Folder → Loop Over Items (loop for next batch)

15. **Configure Credentials:**  
    - Gmail OAuth2 for Gmail Trigger and Gmail nodes.  
    - Google Sheets OAuth2 for Lookup in Sheets.  
    - Google Drive OAuth2 for all Google Drive nodes.

16. **Test Workflow:**  
    - Send test emails matching the Gmail filter with attachments from whitelisted senders.  
    - Verify folders and files are created in Google Drive as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Example Whitelist Sheet to copy for email/company mapping                                                             | https://docs.google.com/spreadsheets/d/1tTz9BflstxVL18YG11Ny1eiDj3FcjvtZ619b_bHx8h4/copy                   |
| Instructions for creating Gmail filter to label emails with attachments                                                | See sticky note on Gmail Trigger node for detailed steps                                                    |
| Parent folder for company folders is fixed to Google Drive folder ID: `18ry0AUtrpp3re6u3zQvvs0BQUGFmBKN9` (Invoices)   | Change this ID in Create Company Folder node if needed                                                       |
| Filenames are prefixed with timestamp (milliseconds since epoch) to avoid duplicates                                   | See Upload To Folder node configuration                                                                     |
| Workflow uses OAuth2 credentials for Gmail, Google Sheets, and Google Drive integrations                               | Ensure all OAuth2 credentials are properly configured and authorized                                        |
| The workflow processes attachments one by one to avoid API rate limits and memory issues                               | Implemented via SplitInBatches node                                                                          |

---

This documentation provides a comprehensive understanding of the workflow’s structure, logic, and configuration, enabling reproduction, modification, and troubleshooting.