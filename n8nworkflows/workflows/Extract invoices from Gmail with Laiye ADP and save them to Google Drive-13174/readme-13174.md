Extract invoices from Gmail with Laiye ADP and save them to Google Drive

https://n8nworkflows.xyz/workflows/extract-invoices-from-gmail-with-laiye-adp-and-save-them-to-google-drive-13174


# Extract invoices from Gmail with Laiye ADP and save them to Google Drive

This document provides a technical breakdown and implementation guide for the n8n workflow: **"Extract invoices from Gmail with Laiye ADP and save them to Google Drive."**

---

### 1. Workflow Overview
This workflow automates the end-to-end processing of financial documents. It monitors a Gmail inbox for incoming emails containing potential invoices or receipts, filters them based on file type and naming conventions, extracts structured data using the Laiye Agentic Document Processor (ADP) AI, and finally archives both the original files and the extracted data into Google Drive.

**The logic is organized into four functional blocks:**
1.  **Input & Metadata Extraction:** Detects emails and normalizes multiple attachments into individual items.
2.  **Validation & Filtering:** Segregates documents based on business rules (filename keywords, size, and file extension).
3.  **AI Extraction & Transformation:** Converts documents to Base64, invokes the Laiye ADP API, and transforms the JSON response into a downloadable Excel-compatible XML file.
4.  **Cloud Storage & Error Handling:** Routes processed data to Google Drive and moves unqualified files to a "Pending" folder for manual review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Preparation
*   **Overview:** This block triggers the workflow and prepares binary data for processing.
*   **Nodes Involved:** `Gmail Trigger`, `Extract attachment information`.
*   **Node Details:**
    *   **Gmail Trigger (v1.3):** Polling-based trigger. It is configured to monitor the `INBOX`, download attachments, and prefix them with `attachment_`. 
    *   **Extract attachment information (Code):** A critical JavaScript node that iterates through all binary attachments (e.g., `attachment_0`, `attachment_1`). It extracts metadata (MIME type, size in MB, extension) and, crucially, renames all binary keys to a unified property named `file`. This allows downstream nodes to process multiple attachments from a single email as separate items.

#### 2.2 Validation Logic
*   **Overview:** Determines if a file is a valid financial document suitable for AI processing.
*   **Nodes Involved:** `Filter documents`.
*   **Node Details:**
    *   **Filter documents (If):** Uses three conditions to route data.
        1.  **Filename (Regex):** Must match `invoice|receipt|expenses|fee`.
        2.  **Extension (Regex):** Must be a common image or document format (e.g., `pdf`, `jpg`, `xlsx`, `docx`).
        3.  **Size:** Must be less than 50MB.
    *   **Edges:** "True" leads to AI extraction; "False" leads to the unprocessed storage path.

#### 2.3 AI Processing & Data Formatting
*   **Overview:** Executes the heavy lifting of data extraction and formats the result.
*   **Nodes Involved:** `Base64 Encode Document`, `Laiye ADP HTTP Request`, `Result Processor`.
*   **Node Details:**
    *   **Base64 Encode Document (Extract from File):** Converts the binary `file` into a Base64 string required by the API.
    *   **Laiye ADP HTTP Request (HTTP Request):** A POST request to `https://adp.laiye.com/.../extract`. It sends the Base64 string along with API credentials (`app_key`, `app_secret`) in the JSON body and security headers.
    *   **Result Processor (Code):** Parses the complex JSON response from Laiye ADP. It identifies "Main Fields" and "Table Values" (line items) and constructs a Microsoft Excel-compatible XML string. It outputs this as a binary file named `data.xls`.

#### 2.4 Cloud Storage
*   **Overview:** Handles the final file upload to Google Drive for both successful and filtered-out files.
*   **Nodes Involved:** `Upload the extracted result document`, `Create a pending folder`, `Merge`, `Upload Unprocessed file`.
*   **Node Details:**
    *   **Upload the extracted result document:** Saves the generated Excel file to a specific Drive folder. The filename is dynamically generated using the sender's name and the original attachment name.
    *   **Create a pending folder:** If a file fails the filter, this node ensures a folder named "Untreated document" exists in the root of Google Drive.
    *   **Merge:** Synchronizes the folder ID from the previous node with the original binary data of the "unqualified" file.
    *   **Upload Unprocessed file:** Uploads the original file to the "Untreated document" folder for human intervention.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Entry Point | (None) | Extract attachment information | Step 1: Connect your Gmail account. You can set the execution frequency here. The default is once per day. |
| Extract attachment information | n8n-nodes-base.code | Normalization | Gmail Trigger | Filter documents | |
| Filter documents | n8n-nodes-base.if | Routing / Logic | Extract attachment information | Base64 Encode Document (T), Create a pending folder (F), Merge (F) | 1. You can edit {{ $json.attachment_name }} to use your own filter keywords. 2. We recommend not modifying file size and format settings. |
| Base64 Encode Document | n8n-nodes-base.extractFromFile | Transformation | Filter documents | Laiye ADP HTTP Request | |
| Laiye ADP HTTP Request | n8n-nodes-base.httpRequest | AI Extraction | Base64 Encode Document | Result Processor | |
| Result Processor | n8n-nodes-base.code | Excel Formatting | Laiye ADP HTTP Request | Upload the extracted result document | |
| Upload the extracted result document | n8n-nodes-base.googleDrive | Cloud Storage | Result Processor | (None) | |
| Create a pending folder | n8n-nodes-base.googleDrive | Folder Creation | Filter documents | Merge | Step 2: Connect your Google Drive. Ensure all files are safely stored in Google Drive. |
| Merge | n8n-nodes-base.merge | Data Sync | Filter documents, Create a pending folder | Upload Unprocessed file | |
| Upload Unprocessed file | n8n-nodes-base.googleDrive | Fallback Storage | Merge | (None) | |

---

### 4. Reproducing the Workflow from Scratch

1.  **Gmail Connection:**
    *   Add a **Gmail Trigger**. Set "Resource" to "Message" and "Event" to "Received".
    *   **Crucial:** Enable "Download Attachments" and set the "Data Property Attachments Prefix" to `attachment_`.
2.  **Attachment Normalization:**
    *   Add a **Code Node** (JavaScript). Use a loop to iterate through `binary` keys. Rename every attachment to `file` and extract `fileName` and `fileExtension` into the JSON body.
3.  **Filtering Logic:**
    *   Add an **If Node**. Create a "Regex" condition for `attachment_name` matching `invoice|receipt|expenses|fee`. Add a second Regex for allowed extensions and a "Number" check for `attachment_size_mb < 50`.
4.  **AI Integration (True Path):**
    *   Connect the "True" branch to an **Extract from File Node**. Set operation to "Binary to Property" and encoding to `base64`.
    *   Add an **HTTP Request Node**. Set Method to `POST` and URL to `https://adp.laiye.com/open/agentic_doc_processor/laiye/v1/app/doc/extract`. 
    *   In the Body, pass `app_key`, `app_secret`, and `file_base64` (from the previous node). Add necessary security headers (`X-Access-Key`, `X-Timestamp`, `X-Signature`).
5.  **Excel Generation:**
    *   Add a **Code Node** to parse the Laiye ADP response. Use JavaScript to build an XML string (Workbook/Worksheet tags) and return it as a binary buffer with MIME type `application/vnd.ms-excel`.
6.  **Google Drive Integration:**
    *   Connect the Excel generator to a **Google Drive Node** (Action: Upload).
    *   Connect the **If Node** "False" branch to a **Google Drive Node** (Action: Create Folder) named "Untreated document".
    *   Use a **Merge Node** (Mode: Combine) to join the Folder ID with the original binary data, then use a final **Google Drive Node** to upload the "Unprocessed" file.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Workflow Purpose** | Automates manual invoice sorting, checking, and data entry. |
| **Supported Formats** | jpeg, jpg, png, bmp, tiff, pdf, doc, docx, xls, xlsx. |
| **Default Schedule** | Once per day (Adjustable in Gmail Trigger). |
| **Manual Review** | Unqualified files are stored in "Untreated document" for manual review. |
| **API Requirements** | Requires Laiye ADP App Key and App Secret. |