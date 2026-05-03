Automated Invoice Management with Nextcloud, Email and Telegram Notifications

https://n8nworkflows.xyz/workflows/automated-invoice-management-with-nextcloud--email-and-telegram-notifications-4655


# Automated Invoice Management with Nextcloud, Email and Telegram Notifications

### 1. Workflow Overview

This workflow automates the management of incoming PDF invoices stored in Nextcloud. Its main purpose is to:

- Periodically scan a specified Nextcloud folder for new invoice files.
- Download each invoice file found.
- Send the invoice as an email attachment to a fixed recipient.
- Archive the processed invoice file into a designated Nextcloud archive folder.
- Notify a Telegram chat about the successful sending of each invoice.

The workflow is designed for businesses needing an automated, hands-off method to distribute invoices stored in Nextcloud and keep track of the process via Telegram alerts.

**Logical blocks:**

- **1.1 Scheduled Trigger & Parameter Setup:** Periodic initiation and configuration of key parameters (emails, folder paths).  
- **1.2 Invoice Discovery & Validation:** Listing files in the incoming invoices Nextcloud folder and verifying their existence.  
- **1.3 Invoice Processing:** Downloading the invoice, sending it via email, and archiving it afterward.  
- **1.4 Notifications:** Sending Telegram alerts confirming successful invoice dispatch.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Parameter Setup

**Overview:**  
This block triggers the workflow at a fixed time every day and sets global parameters such as source and destination folders in Nextcloud and email addresses for sending invoices.

**Nodes Involved:**  
- Start  
- Set Parameters

**Node Details:**

- **Start (Schedule Trigger)**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the workflow daily at 8 AM.  
  - *Configuration:* Uses a simple daily interval without complex cron expressions.  
  - *Inputs:* None  
  - *Outputs:* Triggers "Set Parameters" node.  
  - *Failure Cases:* Workflow won’t trigger if n8n instance is down at scheduled time; no other edge cases.  
  - *Version:* 1

- **Set Parameters**  
  - *Type:* Set  
  - *Role:* Defines key workflow variables such as email addresses and Nextcloud folder paths.  
  - *Configuration:*  
    - from_email: info@example.com  
    - to_email: invoices@example.com  
    - nextcloud_income: /Invoices/2025  
    - nextcloud_archive: /Invoices/Archive/2025  
  - *Inputs:* Triggered by Start node  
  - *Outputs:* Passes parameters to "List Incoming Invoices" node.  
  - *Edge Cases:* Hardcoded parameters can limit flexibility; errors if folder paths are mistyped or do not exist.  
  - *Version:* 3.4

---

#### 2.2 Invoice Discovery & Validation

**Overview:**  
This block lists all files in the specified Nextcloud incoming invoices folder and filters out any empty or missing file entries.

**Nodes Involved:**  
- List Incoming Invoices  
- File Exists?

**Node Details:**

- **List Incoming Invoices**  
  - *Type:* Nextcloud (Folder List)  
  - *Role:* Lists all files in the folder defined by `nextcloud_income` parameter.  
  - *Configuration:*  
    - Resource: folder  
    - Operation: list  
    - Path: dynamically set to `/Invoices/2025` (from parameter)  
  - *Credentials:* Nextcloud API credentials configured with a valid user account.  
  - *Inputs:* Receives parameters from "Set Parameters"  
  - *Outputs:* Sends list of files to "File Exists?" node.  
  - *Failure Cases:* Authentication failure, incorrect folder path, API rate limits, network timeouts.  
  - *Version:* 1

- **File Exists?**  
  - *Type:* If  
  - *Role:* Checks if the `path` property of each file item is not empty, filtering valid files for processing.  
  - *Configuration:* Condition checks if `{{$json.path}}` is not empty.  
  - *Inputs:* List of files from "List Incoming Invoices"  
  - *Outputs:* Passes only existing files to "Download Invoice".  
  - *Edge Cases:* Files without a path or corrupted metadata will be skipped; no error thrown but no processing occurs for invalid entries.  
  - *Version:* 1

---

#### 2.3 Invoice Processing

**Overview:**  
This block downloads the invoice file from Nextcloud, emails it to the recipient, and then archives the original file to a Nextcloud archive folder.

**Nodes Involved:**  
- Download Invoice  
- Send Email  
- Archive File

**Node Details:**

- **Download Invoice**  
  - *Type:* Nextcloud (Download)  
  - *Role:* Downloads the invoice file from the path provided by the previous node.  
  - *Configuration:*  
    - Operation: download  
    - Path: dynamically set from incoming JSON `path`  
  - *Credentials:* Same Nextcloud API credentials as previous Nextcloud node.  
  - *Inputs:* File path from "File Exists?" node.  
  - *Outputs:* Passes file binary data to "Send Email".  
  - *Failure Cases:* File might be locked or deleted between listing and download; network errors or API permission issues.  
  - *Version:* 1

- **Send Email**  
  - *Type:* Email Send  
  - *Role:* Sends the downloaded invoice as an attachment via SMTP email.  
  - *Configuration:*  
    - Subject: "New Invoice"  
    - Text: "Please find the attached invoice."  
    - To: `invoices@example.com` (from "Set Parameters")  
    - From: `info@example.com` (from "Set Parameters")  
    - Attachments: uses the binary data from downloaded invoice  
  - *Credentials:* SMTP credentials configured with a valid mail server.  
  - *Inputs:* Binary invoice file from "Download Invoice"  
  - *Outputs:* Triggers "Archive File"  
  - *Failure Cases:* SMTP authentication failure, network issues, attachment size limits, invalid email addresses.  
  - *Version:* 1

- **Archive File**  
  - *Type:* Nextcloud (Move)  
  - *Role:* Moves the processed invoice file to the archive folder to avoid re-processing.  
  - *Configuration:*  
    - Operation: move  
    - Source Path: from "Download Invoice" JSON path  
    - Destination Path: `/N8N/2025/Archive` (hardcoded, should ideally use parameter for consistency)  
  - *Credentials:* Same Nextcloud API credentials.  
  - *Inputs:* Triggered after successful email sending.  
  - *Outputs:* Triggers "Notify Telegram" node.  
  - *Failure Cases:* File permissions issues, folder non-existence, simultaneous access conflicts.  
  - *Version:* 1

---

#### 2.4 Notifications

**Overview:**  
This final block sends a Telegram message confirming each invoice sent, including the file path for traceability.

**Nodes Involved:**  
- Notify Telegram

**Node Details:**

- **Notify Telegram**  
  - *Type:* Telegram  
  - *Role:* Sends a notification message to a Telegram chat or channel.  
  - *Configuration:*  
    - Text: "Invoice sent: {{path}}" where path is from "List Incoming Invoices" node's item JSON  
    - Chat ID: 617681859 (hardcoded Telegram chat)  
  - *Credentials:* Telegram API credentials configured with bot token.  
  - *Inputs:* Triggered after "Archive File" node completes.  
  - *Outputs:* None (end of workflow)  
  - *Failure Cases:* Telegram API limits, invalid chat ID, network issues, bot permissions.  
  - *Version:* 1

---

### 3. Summary Table

| Node Name             | Node Type               | Functional Role                       | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                         |
|-----------------------|-------------------------|------------------------------------|------------------------|--------------------------|--------------------------------------------------------------------------------------------------|
| Start                 | Schedule Trigger        | Starts workflow daily at 8 AM       | —                      | Set Parameters           | Run every day at 8 AM                                                                             |
| Set Parameters        | Set                     | Defines emails and Nextcloud paths  | Start                  | List Incoming Invoices    |                                                                                                  |
| List Incoming Invoices| Nextcloud (Folder List) | Lists files in incoming invoice dir | Set Parameters          | File Exists?             |                                                                                                  |
| File Exists?          | If                      | Filters files with valid paths      | List Incoming Invoices  | Download Invoice         |                                                                                                  |
| Download Invoice      | Nextcloud (Download)    | Downloads invoice file from Nextcloud | File Exists?          | Send Email               |                                                                                                  |
| Send Email            | Email Send              | Emails invoice as attachment        | Download Invoice        | Archive File             |                                                                                                  |
| Archive File          | Nextcloud (Move)        | Moves invoice to archive folder     | Send Email              | Notify Telegram          |                                                                                                  |
| Notify Telegram       | Telegram                | Sends Telegram notification         | Archive File            | —                        |                                                                                                  |
| Sticky Note           | Sticky Note             | Describes workflow functionality    | —                      | —                        | ## This workflow automatically fetches PDF invoices \n**Reading in Nextcloud folder (`/Invoice/2025`), sends them via email to a fixed recipient (`invoice@example.com`), sends a Telegram notification, and archives the file to `/Invoice/Archive/2025`. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name it "Start"  
   - Set it to trigger daily at 8 AM (use default daily interval or cron expression as needed).

3. **Add a Set node:**  
   - Name it "Set Parameters"  
   - Add the following fields with their values:  
     - `from_email`: "info@example.com"  
     - `to_email`: "invoices@example.com"  
     - `nextcloud_income`: "/Invoices/2025"  
     - `nextcloud_archive`: "/Invoices/Archive/2025"  
   - Connect "Start" node output to this node input.

4. **Add a Nextcloud node:**  
   - Name it "List Incoming Invoices"  
   - Set resource to "Folder"  
   - Operation to "List"  
   - Path: Use expression `{{$json.nextcloud_income}}` to get the incoming folder path from parameters  
   - Attach valid Nextcloud API credentials  
   - Connect "Set Parameters" node output to this node input.

5. **Add an If node:**  
   - Name it "File Exists?"  
   - Set condition: Check if `{{$json.path}}` is not empty (string condition)  
   - Connect "List Incoming Invoices" node output to this node input.

6. **Add a Nextcloud node:**  
   - Name it "Download Invoice"  
   - Operation: Download  
   - Path: Use expression `{{$json.path}}` to download current file  
   - Use same Nextcloud API credentials as before  
   - Connect "File Exists?" node's true (if) output to this node input.

7. **Add an Email Send node:**  
   - Name it "Send Email"  
   - Subject: "New Invoice"  
   - Text: "Please find the attached invoice."  
   - To Email: Use expression `{{$node["Set Parameters"].json["to_email"]}}`  
   - From Email: Use expression `{{$node["Set Parameters"].json["from_email"]}}`  
   - Attachments: Use binary data from "Download Invoice" node (typically `data`)  
   - Add SMTP credentials for a valid mail server  
   - Connect "Download Invoice" node output to this node input.

8. **Add a Nextcloud node:**  
   - Name it "Archive File"  
   - Operation: Move  
   - Path: Use expression `{{$node["Download Invoice"].json["path"]}}` as source file path  
   - To Path: For consistency, set to the parameter `{{$node["Set Parameters"].json["nextcloud_archive"]}}` (adjust from original workflow which used hardcoded path)  
   - Use same Nextcloud API credentials  
   - Connect "Send Email" node output to this node input.

9. **Add a Telegram node:**  
   - Name it "Notify Telegram"  
   - Text: Use expression: `Invoice sent: {{$node["List Incoming Invoices"].json["path"]}}`  
   - Chat ID: 617681859 (replace with your Telegram chat ID)  
   - Add Telegram API credentials (bot token)  
   - Connect "Archive File" node output to this node input.

10. **Add a Sticky Note node:**  
    - Add a large sticky note describing the workflow:  
      "This workflow automatically fetches PDF invoices from Nextcloud folder `/Invoice/2025`, sends them via email to `invoices@example.com`, sends a Telegram notification, and archives the file to `/Invoice/Archive/2025`."

11. **Connect the nodes as follows:**  
    - Start → Set Parameters → List Incoming Invoices → File Exists? (true) → Download Invoice → Send Email → Archive File → Notify Telegram

12. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow uses a fixed Telegram chat ID; update it with your own Telegram chat or channel ID and ensure bot permissions.| Telegram Bot API documentation                   |
| SMTP credentials require a properly configured outgoing mail server to send emails successfully.                      | Configure SMTP in n8n credentials                |
| Nextcloud API credentials must have proper permissions to list, download, and move files in the specified directories.| Nextcloud API authentication and permissions     |
| The workflow runs daily at 8 AM; adjust the schedule trigger if different frequency or time is needed.                 | n8n scheduling documentation                      |
| Archiving path in original workflow was hardcoded; recommended to use parameter for consistency and easier updates.  | Workflow maintainability best practices          |
| Sticky note content provides a quick overview suitable for user documentation or quick reference inside n8n editor. |                                                      |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly accessible.