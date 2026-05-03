Extract and Format PDF Data from Google Drive

https://n8nworkflows.xyz/workflows/extract-and-format-pdf-data-from-google-drive-9061


# Extract and Format PDF Data from Google Drive

---

## 1. Workflow Overview

This workflow automates the process of extracting and cleaning text data from PDF files stored in a Google Drive folder. It is designed for use cases such as archiving, data extraction from reports or invoices, and any scenario requiring automated PDF text processing.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception (File Discovery):** Triggering the workflow manually and locating PDF files in a specified Google Drive folder.
- **1.2 Retrieval Stage (File Download):** Downloading the identified PDF files from Google Drive.
- **1.3 Processing Stage (Data Extraction):** Extracting raw text content from the downloaded PDF files.
- **1.4 Formatting Stage (Data Parsing & Cleaning):** Cleaning and formatting the extracted text using custom JavaScript code to prepare it for downstream use.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (File Discovery)

**Overview:**  
This block starts the workflow on demand and searches a designated Google Drive folder for PDF files, identifying all files that match the `.pdf` extension.

**Nodes Involved:**  
- Start  
- Get PDF Files/File

**Node Details:**

- **Start**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow manually by user action.  
  - Configuration: Default manual trigger with no parameters.  
  - Inputs: None  
  - Outputs: Connected to "Get PDF Files/File" node.  
  - Potential Failures: None, unless user neglects to trigger workflow.

- **Get PDF Files/File**  
  - Type: Google Drive  
  - Role: Searches for PDF files within a specified Google Drive folder.  
  - Configuration:  
    - Search Query: `*.pdf` to target PDF files only.  
    - Filter: Folder filter applied to restrict search to a specific folder (folder ID must be set).  
    - Fields Requested: `id`, `name` of files.  
    - Return All: True (returns all matching files).  
  - Expressions: Folder ID set dynamically (must be configured).  
  - Inputs: Connected from "Start".  
  - Outputs: Sends file metadata (id, name) to "Download Retrieval Files/File".  
  - Credential: Uses Google Drive OAuth2 credentials named "Template".  
  - Potential Failures:  
    - Invalid or expired Google credentials (authentication error).  
    - Incorrect or empty folder ID (no files found).  
    - No PDF files present in folder.  
  - Edge Cases: Empty folder, permission restrictions.

---

### 2.2 Retrieval Stage (File Download)

**Overview:**  
This block downloads each PDF file found in the previous step, converting it to plain text format to facilitate extraction.

**Nodes Involved:**  
- Download Retrieval Files/File

**Node Details:**

- **Download Retrieval Files/File**  
  - Type: Google Drive  
  - Role: Downloads files by their ID from Google Drive, converting PDFs to plain text.  
  - Configuration:  
    - Operation: Download  
    - File ID: Dynamically set via expression `{{$json.id}}` from previous node output.  
    - Google File Conversion: Converts Google Docs to `text/plain` format (applied for PDFs).  
  - Inputs: Receives file metadata from "Get PDF Files/File".  
  - Outputs: Sends binary file data to "Extract Files/File's Data".  
  - Credential: Uses Google Drive OAuth2 credentials named "Template".  
  - Potential Failures:  
    - File ID invalid or deleted.  
    - Permission denied for file access.  
    - Conversion errors if file is corrupted or unsupported.  
  - Edge Cases: Large files causing timeout, network issues.

---

### 2.3 Processing Stage (Data Extraction)

**Overview:**  
Extracts raw text content from the downloaded PDF binary data, preparing it for cleaning and formatting.

**Nodes Involved:**  
- Extract Files/File's Data  
- Get PDF Data Only

**Node Details:**

- **Extract Files/File's Data**  
  - Type: Extract From File  
  - Role: Extracts text content from PDF files.  
  - Configuration:  
    - Operation: PDF extraction mode enabled.  
  - Inputs: Receives binary PDF data from "Download Retrieval Files/File".  
  - Outputs: Provides extracted text under the field `text` along with other file data.  
  - Potential Failures:  
    - Malformed PDF that cannot be parsed.  
    - Extraction failure due to unsupported PDF features.  
  - Edge Cases: Very large PDFs, encrypted PDFs.

- **Get PDF Data Only**  
  - Type: Set  
  - Role: Isolates the extracted text field (`text`) from the full extraction output, simplifying data for the next step.  
  - Configuration:  
    - Sets a new JSON property `text` equal to the extracted text from the previous node (`{{$json.text}}`).  
  - Inputs: From "Extract Files/File's Data".  
  - Outputs: Passes cleaned data structure to "Data Parser & Cleaner".  
  - Potential Failures:  
    - Missing `text` field if extraction failed or empty PDF.  
    - Expression evaluation errors if input data structure changes.

---

### 2.4 Formatting Stage (Data Parsing & Cleaning)

**Overview:**  
Cleans and formats the raw extracted text to remove unwanted characters such as newlines and prepare the data for further use or export.

**Nodes Involved:**  
- Data Parser & Cleaner  
- Done !

**Node Details:**

- **Data Parser & Cleaner**  
  - Type: Code (JavaScript)  
  - Role: Processes the raw text string by removing newline characters and optionally other cleanup tasks.  
  - Configuration:  
    - Custom JavaScript code that:  
      - Checks if input is a string.  
      - Replaces all newline characters (`\n`) with spaces.  
      - Logs original and cleaned text for debugging.  
      - Returns an object containing the cleaned text as `cleanedText`.  
  - Key Expressions:  
    - Input accessed via `$input.first().json.text`  
  - Inputs: Receives JSON with raw text from "Get PDF Data Only".  
  - Outputs: Sends cleaned text JSON to "Done !".  
  - Potential Failures:  
    - Input is not a string, causing errors or empty output.  
    - JavaScript syntax errors in code node.  
    - Input path changes breaking the code.  
  - Version Specific: Uses n8n Code node v2 syntax.

- **Done !**  
  - Type: No Operation (NoOp)  
  - Role: Terminal node indicating workflow completion.  
  - Configuration: None.  
  - Inputs: Receives cleaned text from "Data Parser & Cleaner".  
  - Outputs: None.  
  - Purpose: Can be used to inspect final output or for future extensions.

---

## 3. Summary Table

| Node Name                  | Node Type             | Functional Role                  | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                                  |
|----------------------------|-----------------------|---------------------------------|--------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| Start                      | Manual Trigger        | Initiate workflow manually       | —                        | Get PDF Files/File        |                                                                                                              |
| Get PDF Files/File          | Google Drive          | Search PDF files in folder       | Start                    | Download Retrieval Files/File | See Sticky Note6 for setup instructions on folder, creds, and search configuration.                          |
| Download Retrieval Files/File | Google Drive        | Download PDF files               | Get PDF Files/File        | Extract Files/File's Data | See Sticky Note5 for download node configuration details.                                                   |
| Extract Files/File's Data   | Extract From File     | Extract text from PDF            | Download Retrieval Files/File | Get PDF Data Only         |                                                                                                              |
| Get PDF Data Only           | Set                   | Isolate extracted text           | Extract Files/File's Data | Data Parser & Cleaner     |                                                                                                              |
| Data Parser & Cleaner       | Code (JavaScript)     | Clean and format extracted text  | Get PDF Data Only         | Done !                   | See Sticky Note5 for code node usage and troubleshooting.                                                   |
| Done !                     | No Operation (NoOp)  | Workflow completion marker       | Data Parser & Cleaner     | —                        |                                                                                                              |
| Sticky Note2                | Sticky Note           | Thank you and feedback request   | —                        | —                        | Expresses gratitude and invites feedback on workflow improvements.                                          |
| Sticky Note3                | Sticky Note           | Troubleshooting tips             | —                        | —                        | Provides common fixes and debug checklist for credential and node issues.                                   |
| Sticky Note4                | Sticky Note           | Step-by-step setup guide placeholder | —                      | —                        | Contains placeholders for detailed setup instructions.                                                     |
| Sticky Note5                | Sticky Note           | Detailed configuration notes     | —                        | —                        | Explains download node and code node configurations, plus testing instructions.                             |
| Sticky Note6                | Sticky Note           | Google Drive preparation steps   | —                        | —                        | Advises on folder preparation, credential connection, and search node setup.                                |
| Sticky Note7                | Sticky Note           | Customization suggestions        | —                        | —                        | Suggests modifying data fields and parser code for customization.                                          |
| Sticky Note8                | Sticky Note           | Workflow flow explanation        | —                        | —                        | Explains the four main stages of the workflow in detail.                                                   |
| Sticky Note9                | Sticky Note           | Quick demo and use case overview | —                        | —                        | Summarizes input/output and use cases of the workflow.                                                     |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node ("Start")**  
   - Type: Manual Trigger  
   - No special configuration needed.

2. **Add Google Drive Node ("Get PDF Files/File")**  
   - Set Operation: Search  
   - Resource: File/Folder  
   - Search Query: `*.pdf`  
   - Add Filter: Folder  
     - Operation: In Folder  
     - Select the Google Drive folder containing PDFs.  
   - Fields to Return: `id`, `name`  
   - Return All: Enabled  
   - Connect Manual Trigger output to this node input.  
   - Configure Google Drive OAuth2 credentials (create new if not existing):  
     - Provide Client ID and Client Secret from Google Cloud Console.  
     - Authenticate and authorize access to Google Drive.

3. **Add Google Drive Node ("Download Retrieval Files/File")**  
   - Set Operation: Download  
   - File ID: Use expression `{{$json.id}}` to get the file ID dynamically from previous node.  
   - Google File Conversion: Enable conversion to `text/plain` format for PDFs.  
   - Connect output of "Get PDF Files/File" to this node's input.  
   - Use the same Google Drive OAuth2 credentials as above.

4. **Add Extract From File Node ("Extract Files/File's Data")**  
   - Operation: PDF extraction enabled.  
   - Connect output of "Download Retrieval Files/File" to this node.  
   - No credentials needed.

5. **Add Set Node ("Get PDF Data Only")**  
   - In Parameters, assign one field:  
     - Name: `text`  
     - Type: String  
     - Value: Expression `{{$json.text}}` to extract the text field from previous node's output.  
   - Connect output of "Extract Files/File's Data" to this node.

6. **Add Code Node ("Data Parser & Cleaner")**  
   - Language: JavaScript  
   - Paste the following script:  
     ```javascript
     function removeNewlines(text) {
       if (typeof text !== 'string') {
         console.error("Input must be a string.");
         return "";
       }
       return text.replace(/\n/g, ' ');
     }
     const inputText = $input.first().json.text;
     const cleanedText = removeNewlines(inputText);
     return { cleanedText };
     ```  
   - Connect output of "Get PDF Data Only" to this node.

7. **Add No Operation Node ("Done !")**  
   - This node serves as the workflow endpoint.  
   - Connect output of "Data Parser & Cleaner" to this node.

8. **Testing the Workflow**  
   - Save the workflow.  
   - Ensure Google Drive folder contains PDF files.  
   - Trigger the workflow manually using the "Start" node.  
   - Verify nodes execute without errors; inspect the output in the "Done !" node for cleaned text results.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                             | Context or Link                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------|
| 🙏 Thank you for trying this workflow! Feedback is welcome on improvements, new features, or additional workflows.                                                                                                       | Sticky Note2                           |
| 🔍 Troubleshooting tips include checking Google credentials, folder selection, file presence, and verifying outputs at each node.                                                                                        | Sticky Note3                           |
| 🛠️ Step-by-step setup guide placeholders are included to assist with quick deployment and configuration.                                                                                                               | Sticky Note4                           |
| 📋 The workflow operates in four stages: Input (file search), Retrieval (download), Processing (extraction), and Formatting (parsing and cleaning).                                                                       | Sticky Note8                           |
| 💾 Customization options: Modify the "Get PDF Data Only" node to extract more metadata fields; adjust the "Data Parser & Cleaner" JavaScript code to suit specific formatting or parsing needs.                            | Sticky Note7                           |
| 📁 Use case overview: Automates extraction of text from PDFs stored in Google Drive folders, ideal for archiving or data processing tasks involving PDFs such as invoices or reports.                                    | Sticky Note9                           |

---

**Disclaimer:** The content described derives exclusively from an n8n automated workflow. It complies fully with content policies, containing no illegal or offensive materials. All handled data is public and legal.

---