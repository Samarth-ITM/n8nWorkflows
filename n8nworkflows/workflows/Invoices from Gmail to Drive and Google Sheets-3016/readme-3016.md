Invoices from Gmail to Drive and Google Sheets

https://n8nworkflows.xyz/workflows/invoices-from-gmail-to-drive-and-google-sheets-3016


# Invoices from Gmail to Drive and Google Sheets

### 1. Workflow Overview

This workflow automates the processing of invoice emails received in Gmail by saving their PDF attachments to Google Drive and extracting key invoice data into a Google Sheets spreadsheet using AI. It is designed for users who want to streamline invoice management, financial record keeping, and document organization by eliminating manual download and data entry tasks.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Filtering**: Watches Gmail for unread emails with attachments, filtering for invoice-related emails.
- **1.2 File Handling and Storage**: Uploads PDF attachments to Google Drive, renames files with meaningful names, and moves them to a designated folder.
- **1.3 Invoice Data Extraction**: Downloads the stored PDF, extracts text content, and uses OpenAI GPT-4o to parse invoice details.
- **1.4 Data Structuring and Storage**: Parses AI output into structured data and appends it to a Google Sheets spreadsheet.
- **1.5 Email Status Update**: Marks processed emails as read to avoid reprocessing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

- **Overview:**  
  This block triggers the workflow on new unread Gmail emails with attachments and filters to process only those emails that contain invoice attachments.

- **Nodes Involved:**  
  - Gmail Trigger1  
  - Setup1  
  - Only invoice mails with attachments

- **Node Details:**

  - **Gmail Trigger1**  
    - Type: Gmail Trigger  
    - Role: Watches Gmail inbox for unread emails with attachments, polling every minute.  
    - Configuration: Filters for unread emails (`readStatus: unread`), downloads attachments automatically.  
    - Inputs: None (trigger node)  
    - Outputs: Email data including attachments and headers.  
    - Potential Failures: Authentication errors, Gmail API rate limits, network timeouts.

  - **Setup1**  
    - Type: Set  
    - Role: Sets a workflow variable `url_to_drive_folder` with the Google Drive folder ID where invoices will be stored.  
    - Configuration: Assigns string value `"1fCWCdqrFP3WrjjLc-gJtxMaiaF5lh8Ko"` to `url_to_drive_folder`.  
    - Inputs: Output from Gmail Trigger1  
    - Outputs: Passes email data along with the folder ID.  
    - Edge Cases: Folder ID must be valid and accessible by the Google Drive credentials.

  - **Only invoice mails with attachments**  
    - Type: If  
    - Role: Filters emails to continue only if the email content type contains `"multipart/mixed"` or if attachments exist.  
    - Configuration: Checks if the email header `content-type` contains `"multipart/mixed"` OR if attachments array is not empty.  
    - Inputs: Output from Setup1  
    - Outputs: True branch proceeds with emails having attachments; False branch ends workflow for others.  
    - Edge Cases: Emails without attachments or with unexpected content types will be skipped.

---

#### 2.2 File Handling and Storage

- **Overview:**  
  This block uploads the PDF attachment to Google Drive, renames the file with a meaningful name based on the email subject and current date, and moves the file to the specified Drive folder.

- **Nodes Involved:**  
  - Upload PDF to Drive1  
  - Rename file1  
  - Move to the correct folder1

- **Node Details:**

  - **Upload PDF to Drive1**  
    - Type: HTTP Request  
    - Role: Uploads the PDF attachment binary data to Google Drive via Drive API v3.  
    - Configuration:  
      - POST to `https://www.googleapis.com/upload/drive/v3/files` with `uploadType=media` query parameter.  
      - Sends binary data from the attachment field chosen dynamically: if the first attachment is a PDF (`application/pdf`), use `attachment_0`, else `attachment_1`.  
      - Uses Google Drive OAuth2 credentials.  
      - Retries up to 5 times on failure.  
    - Inputs: True branch from "Only invoice mails with attachments" (email with attachments)  
    - Outputs: JSON response with uploaded file metadata including file ID.  
    - Edge Cases:  
      - Non-PDF attachments may cause incorrect upload if not handled properly.  
      - API quota limits or authentication failures.  
      - Attachment binary data must be present and valid.

  - **Rename file1**  
    - Type: Google Drive  
    - Role: Renames the uploaded file to a descriptive name combining the email subject and current date.  
    - Configuration:  
      - Operation: Update file metadata.  
      - File ID: Uses the uploaded file ID from previous node.  
      - New file name: `{{ $('Setup1').item.json.subject }}_invoice_{{ $now.format('yyyy-MM-dd') }}.pdf`  
    - Inputs: Output of Upload PDF to Drive1  
    - Outputs: Updated file metadata.  
    - Edge Cases: File ID must be valid; date formatting depends on system time.

  - **Move to the correct folder1**  
    - Type: Google Drive  
    - Role: Moves the renamed file into the designated Google Drive folder.  
    - Configuration:  
      - Operation: Move file.  
      - File ID: From Rename file1 output.  
      - Drive ID: "My Drive" (default Google Drive root).  
      - Folder ID: Uses the folder ID set in Setup1 (`1fCWCdqrFP3WrjjLc-gJtxMaiaF5lh8Ko`).  
    - Inputs: Output of Rename file1  
    - Outputs: File metadata after move.  
    - Edge Cases: Folder must exist and be accessible; file must exist; permission errors possible.

---

#### 2.3 Invoice Data Extraction

- **Overview:**  
  Downloads the stored PDF from Google Drive, extracts its text content, and uses OpenAI GPT-4o to extract structured invoice data.

- **Nodes Involved:**  
  - Gmail (mark as read)  
  - Google Drive (download)  
  - Extract from File2  
  - OpenAI Model  
  - Structured Output Parser  
  - Apply Data Extraction Rules  
  - Sticky Note2 (documentation)  
  - Sticky Note6 (reminder for attributes)

- **Node Details:**

  - **Gmail**  
    - Type: Gmail  
    - Role: Marks the processed email as read to prevent reprocessing.  
    - Configuration:  
      - Operation: Mark as read.  
      - Message ID: Uses the original email ID from Gmail Trigger1.  
    - Inputs: Output from Move to the correct folder1 (true branch)  
    - Outputs: Confirmation of marking email as read.  
    - Edge Cases: Email must exist; authentication required.

  - **Google Drive**  
    - Type: Google Drive  
    - Role: Downloads the PDF file from Google Drive for text extraction.  
    - Configuration:  
      - Operation: Download file.  
      - File ID: From Move to the correct folder1 output.  
    - Inputs: Output from Move to the correct folder1  
    - Outputs: Binary PDF data.  
    - Edge Cases: File must exist; permission and network errors possible.

  - **Extract from File2**  
    - Type: Extract From File  
    - Role: Extracts text content from the downloaded PDF.  
    - Configuration:  
      - Operation: PDF text extraction.  
    - Inputs: Output from Google Drive (download)  
    - Outputs: Extracted text in JSON format.  
    - Edge Cases: Corrupted PDFs, unsupported formats, extraction errors.

  - **OpenAI Model**  
    - Type: LangChain OpenAI Model  
    - Role: Runs GPT-4o model to interpret extracted text and extract invoice data.  
    - Configuration:  
      - Model: gpt-4o  
      - Temperature: 0 (deterministic output)  
      - Credentials: OpenAI API key configured in n8n.  
    - Inputs: Output from Extract from File2  
    - Outputs: Raw AI response text.  
    - Edge Cases: API rate limits, authentication errors, unexpected AI output.

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses AI output into a JSON object with defined schema for invoice data.  
    - Configuration:  
      - JSON Schema:  
        - Invoice date (date)  
        - Invoice description (string)  
        - Total price (number)  
        - Fichero (string, hyperlink)  
    - Inputs: AI raw output from OpenAI Model  
    - Outputs: Structured JSON data.  
    - Edge Cases: Parsing errors if AI output does not conform to schema.

  - **Apply Data Extraction Rules**  
    - Type: LangChain Chain LLM  
    - Role: Defines the prompt template for AI to extract invoice data from the text.  
    - Configuration:  
      - Prompt: Extracts invoice date, description (uses renamed file name), total price, and a hyperlink to the Drive file.  
      - Uses output parser (Structured Output Parser).  
    - Inputs: Extracted text from Extract from File2 and AI model output parser.  
    - Outputs: Structured invoice data JSON.  
    - Edge Cases: Missing data in invoice, incomplete extraction.

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Documentation on using LLMs for data extraction and benefits of structured output parser.  
    - Content includes link: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm/

  - **Sticky Note6**  
    - Type: Sticky Note  
    - Role: Reminder that additional attributes can be added by modifying the prompt or schema.

---

#### 2.4 Data Structuring and Storage

- **Overview:**  
  Maps the structured AI output and appends it as a new row in a Google Sheets spreadsheet for invoice record keeping.

- **Nodes Involved:**  
  - Map Output  
  - Append to Reconciliation Sheet

- **Node Details:**

  - **Map Output**  
    - Type: Set  
    - Role: Extracts the `output` field from the AI structured JSON to prepare for Google Sheets insertion.  
    - Configuration:  
      - Mode: Raw JSON output  
      - JSON Output: `={{ $json.output }}`  
    - Inputs: Output from Apply Data Extraction Rules  
    - Outputs: Clean JSON with invoice data fields.  
    - Edge Cases: If AI output is empty or malformed, mapping may fail.

  - **Append to Reconciliation Sheet**  
    - Type: Google Sheets  
    - Role: Appends the invoice data as a new row in the specified Google Sheets document.  
    - Configuration:  
      - Operation: Append  
      - Document ID: Spreadsheet ID `"1gIUnjSWUhsoTOVVd4ZoVjARCGQfGE8s7FWcju3lNajM"`  
      - Sheet Name: `gid=0` (default first sheet)  
      - Columns: Auto-mapped to columns: Invoice date, Invoice Description, Total price, Fichero  
      - Credentials: Google Sheets OAuth2  
    - Inputs: Output from Map Output  
    - Outputs: Confirmation of row append.  
    - Edge Cases: Spreadsheet must exist and be accessible; column names must match; API limits.

---

### 3. Summary Table

| Node Name                      | Node Type                           | Functional Role                          | Input Node(s)                   | Output Node(s)                            | Sticky Note                                                                                                  |
|--------------------------------|-----------------------------------|----------------------------------------|--------------------------------|------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky Note                    | Sticky Note                       | Setup instructions reminder            | None                           | None                                     | ## Setup 1. Setup your **Gmail** and **Google Drive** credentials 2. Setup your **Google Sheets** credentials 3. Setup your **Openai** api key |
| Gmail Trigger1                 | Gmail Trigger                    | Trigger on unread Gmail emails with attachments | None                           | Setup1                                   |                                                                                                              |
| Setup1                        | Set                              | Sets Google Drive folder ID variable   | Gmail Trigger1                 | Only invoice mails with attachments      |                                                                                                              |
| Only invoice mails with attachments | If                               | Filters emails to those with attachments | Setup1                        | Upload PDF to Drive1, Gmail               |                                                                                                              |
| Upload PDF to Drive1           | HTTP Request                     | Uploads PDF attachment to Google Drive | Only invoice mails with attachments | Rename file1                             |                                                                                                              |
| Rename file1                  | Google Drive                     | Renames uploaded file with subject and date | Upload PDF to Drive1          | Move to the correct folder1               |                                                                                                              |
| Move to the correct folder1    | Google Drive                     | Moves file to designated Drive folder  | Rename file1                  | Gmail, Google Drive                       |                                                                                                              |
| Gmail                         | Gmail                           | Marks email as read after processing   | Move to the correct folder1   | None                                     |                                                                                                              |
| Google Drive                  | Google Drive                    | Downloads PDF file from Drive           | Move to the correct folder1   | Extract from File2                        |                                                                                                              |
| Extract from File2             | Extract From File                | Extracts text from PDF                   | Google Drive                  | Apply Data Extraction Rules               |                                                                                                              |
| OpenAI Model                  | LangChain OpenAI Model          | Runs GPT-4o to extract invoice data     | Extract from File2            | Apply Data Extraction Rules               |                                                                                                              |
| Structured Output Parser       | LangChain Output Parser         | Parses AI output into structured JSON   | OpenAI Model                 | Apply Data Extraction Rules               |                                                                                                              |
| Apply Data Extraction Rules    | LangChain Chain LLM             | Defines prompt and extracts invoice fields | Extract from File2, OpenAI Model, Structured Output Parser | Map Output                              |                                                                                                              |
| Sticky Note2                  | Sticky Note                     | Documentation on LLM data extraction    | None                         | None                                     | ## 3. Use LLMs to Extract Values from Data [Read more about Basic LLM Chain](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm/) |
| Sticky Note6                  | Sticky Note                     | Reminder to add more attributes if needed | None                         | None                                     | **Need more attributes?** Change it here!                                                                    |
| Map Output                   | Set                             | Maps AI output JSON for Google Sheets   | Apply Data Extraction Rules   | Append to Reconciliation Sheet            |                                                                                                              |
| Append to Reconciliation Sheet | Google Sheets                   | Appends invoice data to spreadsheet     | Map Output                   | None                                     |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Configure to watch for unread emails (`readStatus: unread`)  
   - Enable attachment download  
   - Set polling interval to every minute  
   - Connect Gmail OAuth2 credentials

2. **Create Set Node (Setup1)**  
   - Type: Set  
   - Add a string field `url_to_drive_folder` with your Google Drive folder ID (e.g., `"1fCWCdqrFP3WrjjLc-gJtxMaiaF5lh8Ko"`)  
   - Connect output of Gmail Trigger to this node

3. **Create If Node (Only invoice mails with attachments)**  
   - Type: If  
   - Condition: Check if email header `content-type` contains `"multipart/mixed"` OR attachments array is not empty  
   - Connect Setup1 output to this node

4. **Create HTTP Request Node (Upload PDF to Drive1)**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://www.googleapis.com/upload/drive/v3/files`  
   - Query Parameter: `uploadType=media`  
   - Content Type: Binary Data  
   - Input Data Field Name: Use expression to select attachment binary field where mimeType is `application/pdf` (e.g., `={{ $binary.attachment_0.mimeType === "application/pdf" ? "attachment_0" : "attachment_1" }}`)  
   - Authentication: Use Google Drive OAuth2 credentials  
   - Set max retries to 5  
   - Connect True output of If node here

5. **Create Google Drive Node (Rename file1)**  
   - Type: Google Drive  
   - Operation: Update file metadata  
   - File ID: Use uploaded file ID from previous node (`={{ $json.id }}`)  
   - New File Name: Use expression combining email subject and current date, e.g., `={{ $('Setup1').item.json.subject }}_invoice_{{ $now.format('yyyy-MM-dd') }}.pdf`  
   - Connect output of Upload PDF to Drive1

6. **Create Google Drive Node (Move to the correct folder1)**  
   - Type: Google Drive  
   - Operation: Move file  
   - File ID: From Rename file1 output (`={{ $json.id }}`)  
   - Drive ID: "My Drive"  
   - Folder ID: Use the folder ID set in Setup1 (`1fCWCdqrFP3WrjjLc-gJtxMaiaF5lh8Ko`)  
   - Connect output of Rename file1

7. **Create Gmail Node (Mark as read)**  
   - Type: Gmail  
   - Operation: Mark email as read  
   - Message ID: Use original email ID from Gmail Trigger1 (`={{ $('Gmail Trigger1').item.json.id }}`)  
   - Connect output of Move to the correct folder1 (True branch)

8. **Create Google Drive Node (Download)**  
   - Type: Google Drive  
   - Operation: Download file  
   - File ID: From Move to the correct folder1 output (`={{ $json.id }}`)  
   - Connect output of Move to the correct folder1 (True branch)

9. **Create Extract From File Node (Extract from File2)**  
   - Type: Extract From File  
   - Operation: PDF text extraction  
   - Connect output of Google Drive (download)

10. **Create LangChain OpenAI Model Node (OpenAI Model)**  
    - Type: LangChain OpenAI Model  
    - Model: gpt-4o  
    - Temperature: 0  
    - Connect output of Extract from File2  
    - Configure OpenAI API credentials

11. **Create LangChain Structured Output Parser Node (Structured Output Parser)**  
    - Type: LangChain Output Parser Structured  
    - JSON Schema:  
      ```json
      {
        "Invoice date": { "type": "date" },
        "Invoice description": { "type": "string" },
        "Total price": { "type": "number" },
        "Fichero": { "type": "string" }
      }
      ```  
    - Connect AI Model output to this node's input parser

12. **Create LangChain Chain LLM Node (Apply Data Extraction Rules)**  
    - Type: LangChain Chain LLM  
    - Prompt:  
      ```
      Given the following invoice in the <invoice> xml tags, extract the following information as listed below.
      If you cannot the information for a specific item, then leave blank and skip to the next.

      * Invoice date
      * Invoice Description: {{ $('Rename file1').item.json.name }}
      * Total price
      * Fichero: =HYPERLINK("https://drive.google.com/file/d/{{ $('Move to the correct folder1').item.json.id }}/view", "Ver Documento")

      <invoice>{{ $json.text }}</invoice>
      ```
    - Enable output parser and select the Structured Output Parser node  
    - Connect Extract from File2 output to this node's input text  
    - Connect OpenAI Model output to AI language model input  
    - Connect Structured Output Parser output to AI output parser input

13. **Create Set Node (Map Output)**  
    - Type: Set  
    - Mode: Raw  
    - JSON Output: `={{ $json.output }}`  
    - Connect output of Apply Data Extraction Rules

14. **Create Google Sheets Node (Append to Reconciliation Sheet)**  
    - Type: Google Sheets  
    - Operation: Append  
    - Document ID: Your Google Sheets spreadsheet ID (e.g., `"1gIUnjSWUhsoTOVVd4ZoVjARCGQfGE8s7FWcju3lNajM"`)  
    - Sheet Name: Use sheet ID or name (e.g., `gid=0`)  
    - Columns: Auto-map input data to columns: Invoice date, Invoice Description, Total price, Fichero  
    - Connect output of Map Output  
    - Configure Google Sheets OAuth2 credentials

15. **Connect False branch of If node (Only invoice mails with attachments) to Gmail node (Mark as read)**  
    - This ensures emails without attachments are marked as read to avoid reprocessing.

16. **Add Sticky Notes for Documentation**  
    - Add sticky notes with setup instructions and LLM usage tips as per original workflow for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Setup your **Gmail**, **Google Drive**, **Google Sheets** credentials, and **OpenAI** API key before running workflow. | Setup instructions sticky note at workflow start.                                                                                |
| Use GPT-4o model with temperature 0 for deterministic invoice data extraction.                                         | OpenAI Model node configuration.                                                                                                |
| Structured Output Parser ensures AI output conforms to expected JSON schema for easy insertion into Google Sheets.    | Sticky Note2 content and LangChain Output Parser node.                                                                           |
| For more info on LangChain LLM chains, see: [n8n LangChain Chain LLM docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm/) | Sticky Note2 includes this link.                                                                                                 |
| Modify the prompt and JSON schema in the Chain LLM and Output Parser nodes to extract additional invoice attributes.  | Sticky Note6 reminder.                                                                                                           |
| The workflow runs every minute to process new emails as they arrive, ensuring timely invoice processing.               | Gmail Trigger1 polling configuration.                                                                                            |
| Ensure Google Drive folder ID and Google Sheets document ID are correctly set and accessible by the connected accounts. | Setup1 node and Append to Reconciliation Sheet node configuration.                                                              |

---

This documentation provides a complete, detailed understanding of the "Invoices from Gmail to Drive and Google Sheets" workflow, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.