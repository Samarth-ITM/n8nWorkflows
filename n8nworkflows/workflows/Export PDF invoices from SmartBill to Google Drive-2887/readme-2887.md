Export PDF invoices from SmartBill to Google Drive

https://n8nworkflows.xyz/workflows/export-pdf-invoices-from-smartbill-to-google-drive-2887


# Export PDF invoices from SmartBill to Google Drive

### 1. Workflow Overview

This workflow automates the export of invoice PDFs from the Smartbill API and saves them to Google Drive. It is designed for users who want to back up or archive invoices monthly in a structured folder system. The workflow dynamically creates a Google Drive folder named after the previous month (format `YYYY-MM`), generates a range of invoice numbers with proper formatting, retrieves each invoice PDF from Smartbill, and uploads them to the designated folder.

Logical blocks:

- **1.1 Trigger and Data Setup:** Manual trigger and initialization of parameters such as invoice number range, series, and Google Drive folder IDs.
- **1.2 Folder Handling:** Calculation of the target folder name based on last month’s date, searching for the folder in Google Drive, and creating it if it does not exist.
- **1.3 Invoice Generation:** Generation of invoice items with padded invoice numbers and associated metadata.
- **1.4 Invoice PDF Retrieval and Upload:** Looping over each invoice, retrieving the PDF from Smartbill via API, setting the correct MIME type, uploading the file to Google Drive, and pacing requests with a wait node.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Data Setup

- **Overview:**  
  This block initiates the workflow manually and sets essential parameters such as invoice number range, invoice series, and Google Drive folder IDs.

- **Nodes Involved:**  
  - When clicking "Execute Workflow"  
  - SetData

- **Node Details:**

  - **When clicking "Execute Workflow"**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Triggers the next node `SetData`.  
    - Edge cases: None expected; manual trigger requires user action.

  - **SetData**  
    - Type: Set  
    - Role: Defines workflow parameters such as invoice number start/end, invoice series, parent Google Drive folder ID, and target folder name placeholder.  
    - Configuration: Contains static or user-editable fields for:  
      - `startInvoiceNumber` (e.g., 1)  
      - `endInvoiceNumber` (e.g., 20)  
      - `invoiceSeries` (e.g., "A")  
      - `parentFolderId` (Google Drive folder ID where monthly folders will be created)  
      - `folderName` (initially empty, to be set later)  
    - Inputs: Trigger from manual node  
    - Outputs: Passes parameters to `Set Default Folder Name` node  
    - Edge cases: Incorrect or missing parameters here will affect downstream logic.

#### 1.2 Folder Handling

- **Overview:**  
  This block calculates the folder name for last month, searches for this folder in Google Drive, and creates it if it does not exist.

- **Nodes Involved:**  
  - Set Default Folder Name  
  - Search Drive Folder  
  - If exists  
  - Create Drive Folder  
  - Set Folder ID

- **Node Details:**

  - **Set Default Folder Name**  
    - Type: Code (JavaScript)  
    - Role: Calculates last month’s date and formats it as `YYYY-MM` to use as the folder name.  
    - Configuration: Uses JavaScript date functions to get current date, subtract one month, and format accordingly.  
    - Inputs: Receives parameters from `SetData`  
    - Outputs: Passes folder name to `Search Drive Folder`  
    - Edge cases: Handles year boundary correctly (e.g., January to December previous year).

  - **Search Drive Folder**  
    - Type: Google Drive node (Search)  
    - Role: Searches for a folder with the calculated name inside the specified parent folder.  
    - Configuration:  
      - Search query filters by folder name and parent folder ID.  
      - Returns folder metadata if found.  
    - Inputs: Folder name and parent folder ID from previous node  
    - Outputs: Passes results to `If exists` node  
    - Edge cases: API errors, folder not found (empty result).

  - **If exists**  
    - Type: If  
    - Role: Checks if the folder was found by evaluating if search results are non-empty.  
    - Configuration: Condition checks length of search results > 0.  
    - Inputs: Search results from `Search Drive Folder`  
    - Outputs:  
      - True branch: Folder exists → passes folder ID to `Set Folder ID`  
      - False branch: Folder does not exist → triggers `Create Drive Folder`  
    - Edge cases: Unexpected API response formats.

  - **Create Drive Folder**  
    - Type: Google Drive node (Create)  
    - Role: Creates a new folder with the calculated name inside the parent folder.  
    - Configuration:  
      - Folder name from `Set Default Folder Name`  
      - Parent folder ID from `SetData`  
    - Inputs: Triggered if folder does not exist  
    - Outputs: Passes new folder ID to `Set Folder ID`  
    - Edge cases: API quota limits, permission errors.

  - **Set Folder ID**  
    - Type: Set  
    - Role: Stores the folder ID (existing or newly created) for use in subsequent nodes.  
    - Configuration: Sets a variable `folderId` with the folder ID value.  
    - Inputs: From either `If exists` (true) or `Create Drive Folder`  
    - Outputs: Passes folder ID to `Set the loop` node  
    - Edge cases: Missing folder ID would cause downstream failures.

#### 1.3 Invoice Generation

- **Overview:**  
  This block generates a list of invoice items by iterating over the specified invoice number range, padding numbers, and attaching metadata such as invoice series and folder ID.

- **Nodes Involved:**  
  - Set the loop

- **Node Details:**

  - **Set the loop**  
    - Type: Code (JavaScript)  
    - Role: Creates an array of invoice objects, each containing:  
      - Padded invoice number (e.g., "0013")  
      - Invoice series (e.g., "A")  
      - Folder ID for Google Drive upload  
    - Configuration:  
      - Reads `startInvoiceNumber`, `endInvoiceNumber`, `invoiceSeries`, and `folderId` from previous nodes.  
      - Pads invoice numbers to a fixed length (e.g., 4 digits).  
      - Outputs an array of invoice items for batch processing.  
    - Inputs: Folder ID and parameters from previous nodes  
    - Outputs: Passes array to `Loop Over Items`  
    - Edge cases: Invalid number ranges, non-numeric inputs.

#### 1.4 Invoice PDF Retrieval and Upload

- **Overview:**  
  This block processes each invoice item in batches, retrieves the invoice PDF from Smartbill API, sets the correct MIME type, uploads the file to Google Drive, and waits briefly between uploads to avoid rate limits.

- **Nodes Involved:**  
  - Loop Over Items  
  - Smartbill get invoice pdf  
  - Set mime  
  - Upload file to Google Drive  
  - Wait a sec

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes invoice items one at a time or in small batches to control API load.  
    - Configuration: Batch size typically set to 1 for sequential processing.  
    - Inputs: Array of invoice items from `Set the loop`  
    - Outputs: Passes each invoice item to `Smartbill get invoice pdf`  
    - Edge cases: Large batch sizes may cause API throttling.

  - **Smartbill get invoice pdf**  
    - Type: HTTP Request  
    - Role: Calls Smartbill API to retrieve the PDF for the given invoice number and series.  
    - Configuration:  
      - HTTP Method: GET  
      - URL: Constructed dynamically using invoice number and series from input data.  
      - Authentication: HTTP Basic Auth with Smartbill credentials.  
      - Response Format: Binary (PDF file).  
    - Inputs: Invoice number and series from `Loop Over Items`  
    - Outputs: Passes binary PDF data to `Set mime`  
    - Edge cases: Authentication failures, invoice not found (404), network timeouts.

  - **Set mime**  
    - Type: Code (JavaScript)  
    - Role: Sets the MIME type of the binary data explicitly to `application/pdf` for correct upload.  
    - Configuration: Modifies binary data metadata.  
    - Inputs: Binary PDF from `Smartbill get invoice pdf`  
    - Outputs: Passes data to `Upload file to Google Drive`  
    - Edge cases: Missing binary data or corrupted response.

  - **Upload file to Google Drive**  
    - Type: Google Drive node (Upload)  
    - Role: Uploads the PDF file to the target Google Drive folder with a structured filename.  
    - Configuration:  
      - Folder ID: from `Set Folder ID`  
      - Filename: constructed from invoice series and padded invoice number (e.g., `A-0013.pdf`)  
      - MIME type: `application/pdf`  
    - Inputs: PDF binary data with MIME from `Set mime`  
    - Outputs: Passes control to `Wait a sec`  
    - Edge cases: Permission errors, quota exceeded, filename conflicts.

  - **Wait a sec**  
    - Type: Wait  
    - Role: Pauses workflow execution briefly (e.g., 1 second) to avoid API rate limits.  
    - Configuration: Fixed wait time (default or configured).  
    - Inputs: From `Upload file to Google Drive`  
    - Outputs: Loops back to `Loop Over Items` for next invoice  
    - Edge cases: None significant; ensures pacing.

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                          | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                          |
|-----------------------------|---------------------|----------------------------------------|-------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger      | Starts workflow manually                | None                          | SetData                       |                                                                                                    |
| SetData                     | Set                 | Defines invoice range, series, folder IDs | When clicking "Execute Workflow" | Set Default Folder Name        | Modify parameters here to adjust invoice range, series, and Google Drive folder.                   |
| Set Default Folder Name     | Code                | Calculates last month folder name      | SetData                       | Search Drive Folder            |                                                                                                    |
| Search Drive Folder         | Google Drive Search  | Searches for existing folder            | Set Default Folder Name        | If exists                     |                                                                                                    |
| If exists                  | If                  | Checks if folder exists                  | Search Drive Folder            | Set Folder ID (true), Create Drive Folder (false) |                                                                                                    |
| Create Drive Folder         | Google Drive Create  | Creates folder if not found              | If exists (false)              | Set Folder ID                 |                                                                                                    |
| Set Folder ID               | Set                 | Stores folder ID for uploads             | If exists (true), Create Drive Folder | Set the loop                  |                                                                                                    |
| Set the loop                | Code                | Generates invoice list with padded numbers | Set Folder ID                 | Loop Over Items               |                                                                                                    |
| Loop Over Items             | SplitInBatches      | Processes invoices one by one            | Set the loop                  | Smartbill get invoice pdf (main), End (empty) |                                                                                                    |
| Smartbill get invoice pdf   | HTTP Request        | Retrieves invoice PDF from Smartbill API | Loop Over Items               | Set mime                     | Requires Smartbill API credentials (HTTP Basic Auth).                                              |
| Set mime                   | Code                | Sets MIME type to application/pdf       | Smartbill get invoice pdf     | Upload file to Google Drive   |                                                                                                    |
| Upload file to Google Drive | Google Drive Upload | Uploads PDF to Google Drive folder       | Set mime                     | Wait a sec                   | Requires Google Drive API credentials. Filename format: `{series}-{paddedNumber}.pdf`.             |
| Wait a sec                 | Wait                | Pauses between uploads to avoid rate limits | Upload file to Google Drive   | Loop Over Items               |                                                                                                    |
| Sticky Note                | Sticky Note         | Instructional note                       | None                         | None                         |                                                                                                    |
| Sticky Note1               | Sticky Note         | Instructional note                       | None                         | None                         |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: `When clicking "Execute Workflow"`  
   - No parameters needed.

2. **Create Set Node for Parameters**  
   - Type: Set  
   - Name: `SetData`  
   - Add fields:  
     - `startInvoiceNumber` (Number, e.g., 1)  
     - `endInvoiceNumber` (Number, e.g., 20)  
     - `invoiceSeries` (String, e.g., "A")  
     - `parentFolderId` (String, Google Drive folder ID)  
     - `folderName` (String, leave empty or placeholder)  
   - Connect output of manual trigger to this node.

3. **Create Code Node to Calculate Folder Name**  
   - Type: Code  
   - Name: `Set Default Folder Name`  
   - JavaScript code to:  
     - Get current date  
     - Subtract one month (handle year boundary)  
     - Format as `YYYY-MM`  
     - Set `folderName` variable in output  
   - Connect output of `SetData` to this node.

4. **Create Google Drive Search Node**  
   - Type: Google Drive (Search)  
   - Name: `Search Drive Folder`  
   - Parameters:  
     - Query: Search for folder with name equal to `folderName` inside `parentFolderId`  
     - Use expression to pass folder name and parent folder ID from previous node.  
   - Connect output of `Set Default Folder Name` to this node.

5. **Create If Node to Check Folder Existence**  
   - Type: If  
   - Name: `If exists`  
   - Condition: Check if search results length > 0  
   - Connect output of `Search Drive Folder` to this node.

6. **Create Google Drive Create Folder Node**  
   - Type: Google Drive (Create)  
   - Name: `Create Drive Folder`  
   - Parameters:  
     - Folder name: from `folderName` variable  
     - Parent folder ID: from `parentFolderId`  
   - Connect false branch of `If exists` to this node.

7. **Create Set Node to Store Folder ID**  
   - Type: Set  
   - Name: `Set Folder ID`  
   - Parameters:  
     - Set variable `folderId` to:  
       - If folder exists: folder ID from search results  
       - If folder created: folder ID from create node output  
   - Connect true branch of `If exists` and output of `Create Drive Folder` to this node.

8. **Create Code Node to Generate Invoice Items**  
   - Type: Code  
   - Name: `Set the loop`  
   - JavaScript code to:  
     - Read `startInvoiceNumber`, `endInvoiceNumber`, `invoiceSeries`, and `folderId`  
     - Loop from start to end number  
     - Pad invoice number to 4 digits (e.g., `0013`)  
     - Create array of objects with fields: `invoiceNumber`, `invoiceSeries`, `folderId`  
     - Return array as output  
   - Connect output of `Set Folder ID` to this node.

9. **Create SplitInBatches Node**  
   - Type: SplitInBatches  
   - Name: `Loop Over Items`  
   - Parameters: Batch size = 1 (to process invoices sequentially)  
   - Connect output of `Set the loop` to this node.

10. **Create HTTP Request Node to Retrieve PDF**  
    - Type: HTTP Request  
    - Name: `Smartbill get invoice pdf`  
    - Parameters:  
      - HTTP Method: GET  
      - URL: Construct using expressions to insert invoice series and padded invoice number into Smartbill API endpoint URL  
      - Authentication: HTTP Basic Auth with Smartbill credentials  
      - Response Format: File (binary)  
    - Connect output of `Loop Over Items` to this node.

11. **Create Code Node to Set MIME Type**  
    - Type: Code  
    - Name: `Set mime`  
    - JavaScript code to:  
      - Set binary data MIME type to `application/pdf`  
    - Connect output of `Smartbill get invoice pdf` to this node.

12. **Create Google Drive Upload Node**  
    - Type: Google Drive (Upload)  
    - Name: `Upload file to Google Drive`  
    - Parameters:  
      - Folder ID: from `folderId` variable  
      - Filename: Constructed as `{invoiceSeries}-{paddedInvoiceNumber}.pdf` using expressions  
      - MIME Type: `application/pdf`  
      - Binary Data: from previous node  
    - Connect output of `Set mime` to this node.

13. **Create Wait Node**  
    - Type: Wait  
    - Name: `Wait a sec`  
    - Parameters: Wait time (e.g., 1 second)  
    - Connect output of `Upload file to Google Drive` to this node.

14. **Loop Back**  
    - Connect output of `Wait a sec` back to `Loop Over Items` to process next invoice.

15. **Credentials Setup**  
    - Configure Smartbill API credentials for HTTP Basic Auth in the HTTP Request node.  
    - Configure Google Drive OAuth2 credentials for Google Drive nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Modify the `SetData` node to adjust invoice number range, invoice series, and Google Drive folder. | Workflow usage instructions.                                                                     |
| Smartbill API requires HTTP Basic Authentication credentials configured in n8n credentials.    | Smartbill API documentation: https://smartbill.ro/api/                                           |
| Google Drive nodes require OAuth2 credentials with permissions to search, create folders, and upload files. | Google Drive API documentation: https://developers.google.com/drive/api/v3/about-sdk             |
| The folder naming convention uses last month’s date in `YYYY-MM` format to organize invoices monthly. | Ensures consistent archival structure.                                                          |
| The workflow uses a wait node to avoid hitting API rate limits during batch processing.          | Helps prevent throttling or temporary bans.                                                     |

---

This document provides a complete, structured reference for understanding, reproducing, and modifying the "Export PDF invoices from SmartBill to Google Drive" workflow. It covers all nodes, logic blocks, configuration details, and potential failure points.