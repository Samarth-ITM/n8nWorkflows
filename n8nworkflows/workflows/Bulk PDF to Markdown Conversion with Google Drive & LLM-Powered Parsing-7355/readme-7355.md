Bulk PDF to Markdown Conversion with Google Drive & LLM-Powered Parsing

https://n8nworkflows.xyz/workflows/bulk-pdf-to-markdown-conversion-with-google-drive---llm-powered-parsing-7355


# Bulk PDF to Markdown Conversion with Google Drive & LLM-Powered Parsing

### 1. Workflow Overview

This workflow automates the bulk conversion of PDF files into Markdown format using Google Drive as the source and destination storage, combined with an AI-powered PDF parsing node. It is designed to handle multiple PDFs from a specified Google Drive folder, convert them into Markdown using a Large Language Model (LLM)-based parser, and save the results back to Google Drive. Finally, it summarizes the batch operation and sends a notification via Slack.

**Logical blocks:**

- **1.1 Input Reception:** Listing PDF files from a specified Google Drive folder.
- **1.2 Filtering:** Filtering to ensure only PDF files are processed.
- **1.3 AI Processing:** Converting each PDF to Markdown using an LLM-powered PDF parser.
- **1.4 Output Preparation:** Formatting the converted content and metadata.
- **1.5 Output Storage:** Uploading the generated Markdown files back to Google Drive.
- **1.6 Summary and Notification:** Aggregating conversion statistics and notifying completion via Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block retrieves all files from a specified Google Drive folder to serve as the input set for conversion.
- **Nodes Involved:**  
  - Google Drive - List PDFs

- **Node Details:**

  - **Google Drive - List PDFs**  
    - **Type & Role:** Google Drive node; performs file listing operation.  
    - **Configuration:** Lists files within the folder ID provided dynamically from input JSON (`{{$json.folderId}}`).  
    - **Key Expressions:** `fileId` parameter is set to the folder ID input.  
    - **Connections:** Output connects to the filter node ("Filter PDFs Only").  
    - **Potential Failures:** Auth errors (invalid or expired Google credentials), incorrect folder ID causing empty or no results, API rate limits.  
    - **Version Requirements:** Compatible with n8n Google Drive node v1+.  
    - **Notes:** Includes a sticky note "List all PDFs from specified folder" for clarity.

#### 1.2 Filtering

- **Overview:** Filters the listed files to continue processing only those with MIME type `application/pdf`.  
- **Nodes Involved:**  
  - Filter PDFs Only

- **Node Details:**

  - **Filter PDFs Only**  
    - **Type & Role:** If node; conditional filter.  
    - **Configuration:** Checks if the current item's `mimeType` equals `application/pdf`.  
    - **Key Expressions:** Condition expression: `={{ $json.mimeType }} == 'application/pdf'`.  
    - **Connections:** Input from Google Drive node; output connects to PDF conversion node.  
    - **Potential Failures:** Misclassification if MIME types are inconsistent or missing; no PDFs found results in no downstream processing.  
    - **Version Requirements:** Standard `if` node functionality.

#### 1.3 AI Processing

- **Overview:** Converts each filtered PDF into Markdown format using an LLM-based PDF vector parsing node.  
- **Nodes Involved:**  
  - PDF Vector - Convert to Markdown

- **Node Details:**

  - **PDF Vector - Convert to Markdown**  
    - **Type & Role:** Custom PDFVector node; parses PDF document into Markdown using AI.  
    - **Configuration:**  
      - `useLlm`: set to "auto" to enable automatic LLM usage.  
      - `resource`: document.  
      - `operation`: parse.  
      - `documentUrl`: dynamic, from `webViewLink` of input PDF.  
    - **Key Expressions:** `documentUrl` uses `{{$json.webViewLink}}`.  
    - **Connections:** Input from filter node; output to code node for preparing output.  
    - **Potential Failures:** LLM API errors (timeout, quota exceeded), invalid or inaccessible document URL, parsing errors, partial conversion.  
    - **Version Requirements:** Requires n8n PDFVector node with LLM support.  
    - **Notes:** Sticky note indicates "Convert each PDF to Markdown."

#### 1.4 Output Preparation

- **Overview:** Prepares the Markdown content and metadata for storage and reporting.  
- **Nodes Involved:**  
  - Prepare Output

- **Node Details:**

  - **Prepare Output**  
    - **Type & Role:** Code node; formats output data.  
    - **Configuration:** JavaScript code extracts and transforms the file name (replacing `.pdf` with `.md`), assigns Markdown content, and constructs metadata with original filename, conversion timestamp, page count, and credits used.  
    - **Key Expressions:**  
      ```js
      const fileName = $json.name.replace('.pdf', '.md');
      const content = $json.content;
      const metadata = {
        originalFile: $json.name,
        convertedAt: new Date().toISOString(),
        pageCount: $json.pageCount || 'unknown',
        credits: $json.creditsUsed || 0
      };
      return { fileName, content, metadata };
      ```  
    - **Connections:** Input from PDF Vector node; output to Google Drive upload node.  
    - **Potential Failures:** Missing or malformed input data, edge cases if file name does not contain `.pdf`, missing content or metadata fields.  
    - **Version Requirements:** Standard Code node.

#### 1.5 Output Storage

- **Overview:** Uploads the prepared Markdown files to a specified Google Drive folder.  
- **Nodes Involved:**  
  - Save Markdown Files

- **Node Details:**

  - **Save Markdown Files**  
    - **Type & Role:** Google Drive node; uploads files.  
    - **Configuration:**  
      - `name`: dynamically set to the Markdown file name from prepared output (`{{$json.fileName}}`).  
      - `content`: Markdown content (`{{$json.content}}`).  
      - `parents`: destination folder ID(s) (`{{$json.outputFolderId}}`).  
      - `operation`: upload.  
    - **Key Expressions:** Name, content, and parents parameters sourced from previous node output JSON.  
    - **Connections:** Input from prepare output node; output to summary node.  
    - **Potential Failures:** Auth errors, invalid or missing destination folder ID, upload failures (file name conflicts, quota limits).  
    - **Version Requirements:** Compatible with Google Drive node v1+.  
    - **Notes:** None explicitly.

#### 1.6 Summary and Notification

- **Overview:** Generates a summary of the batch conversion and sends a Slack notification upon completion.  
- **Nodes Involved:**  
  - Conversion Summary  
  - Send Notification

- **Node Details:**

  - **Conversion Summary**  
    - **Type & Role:** Set node; creates a summary string for all processed items.  
    - **Configuration:**  
      - Constructs a summary message with total PDFs converted and total credits used, calculated by reducing the array of items and summing their metadata credits.  
      - Expression example:  
        ```
        Converted {{ $items().length }} PDFs to Markdown
        Total credits used: {{ $items().reduce((sum, item) => sum + (item.json.metadata.credits || 0), 0) }}
        ```  
    - **Connections:** Input from save node; output to Slack notification node.  
    - **Potential Failures:** Empty item list causing zero-length summary, errors in expression evaluation.  

  - **Send Notification**  
    - **Type & Role:** Slack node; sends a message to a configured Slack channel.  
    - **Configuration:**  
      - `message`: includes the summary text and confirmation that files were saved to Google Drive.  
      - Expression example:  
        ```
        Batch Conversion Complete!

        {{ $json.summary }}

        Files saved to Google Drive.
        ```  
      - Requires Slack credentials configured.  
    - **Connections:** Input from summary node; no output.  
    - **Potential Failures:** Slack API errors, auth failures, misconfigured Slack credentials or channel.  
    - **Notes:** None explicitly.

---

### 3. Summary Table

| Node Name                      | Node Type              | Functional Role               | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                       |
|-------------------------------|------------------------|------------------------------|---------------------------|----------------------------|-------------------------------------------------------------------------------------------------|
| Workflow Info                  | Sticky Note            | Documentation note            |                           |                            | ## Batch PDF Converter\n\nThis workflow converts PDFs to Markdown in bulk.\n\nSupported sources:\n- Direct URLs\n- Google Drive\n- Dropbox\n- Local files |
| Google Drive - List PDFs       | Google Drive           | List files in folder          |                           | Filter PDFs Only            | List all PDFs from specified folder                                                             |
| Filter PDFs Only               | If                     | Filter for PDF MIME type      | Google Drive - List PDFs   | PDF Vector - Convert to Markdown |                                                                                                 |
| PDF Vector - Convert to Markdown | PDFVector             | Convert PDFs to Markdown      | Filter PDFs Only           | Prepare Output             | Convert each PDF to Markdown                                                                     |
| Prepare Output                | Code                   | Format file name and content  | PDF Vector - Convert to Markdown | Save Markdown Files        |                                                                                                 |
| Save Markdown Files           | Google Drive           | Upload Markdown files         | Prepare Output             | Conversion Summary         |                                                                                                 |
| Conversion Summary            | Set                    | Create conversion summary     | Save Markdown Files        | Send Notification          |                                                                                                 |
| Send Notification            | Slack                  | Notify completion             | Conversion Summary         |                            |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Sticky Note:**  
   - Name: `Workflow Info`  
   - Content:  
     ```
     ## Batch PDF Converter

     This workflow converts PDFs to Markdown in bulk.

     Supported sources:
     - Direct URLs
     - Google Drive
     - Dropbox
     - Local files
     ```  
   - Position it at the beginning for documentation.

2. **Google Drive - List PDFs:**  
   - Add a Google Drive node, name it `Google Drive - List PDFs`.  
   - Set operation to `list`.  
   - Set `fileId` parameter to `={{ $json.folderId }}` to dynamically read the target folder ID from input.  
   - Configure Google Drive credentials with OAuth2 for access.  
   - Position it next after the sticky note.

3. **Filter PDFs Only:**  
   - Add an If node named `Filter PDFs Only`.  
   - Configure condition: Check if `{{$json.mimeType}}` equals `"application/pdf"`.  
   - Connect input from `Google Drive - List PDFs`.  
   - Output branch `true` to next node.

4. **PDF Vector - Convert to Markdown:**  
   - Add a PDFVector node, name it `PDF Vector - Convert to Markdown`.  
   - Set `useLlm` to `"auto"` to enable LLM-based parsing.  
   - Set `resource` to `"document"`.  
   - Set `operation` to `"parse"`.  
   - Set `documentUrl` dynamically: `={{ $json.webViewLink }}`.  
   - Connect input from `Filter PDFs Only` true branch.

5. **Prepare Output:**  
   - Add a Code node named `Prepare Output`.  
   - Use this JavaScript code:  
     ```js
     const fileName = $json.name.replace('.pdf', '.md');
     const content = $json.content;
     const metadata = {
       originalFile: $json.name,
       convertedAt: new Date().toISOString(),
       pageCount: $json.pageCount || 'unknown',
       credits: $json.creditsUsed || 0
     };
     return { fileName, content, metadata };
     ```  
   - Connect input from `PDF Vector - Convert to Markdown`.

6. **Save Markdown Files:**  
   - Add a Google Drive node, name it `Save Markdown Files`.  
   - Set operation to `upload`.  
   - Set `name` to `={{ $json.fileName }}`.  
   - Set `content` to `={{ $json.content }}`.  
   - Set `parents` to `={{ $json.outputFolderId }}` (expects folder ID to be provided dynamically).  
   - Configure Google Drive credentials if different from the first node.  
   - Connect input from `Prepare Output`.

7. **Conversion Summary:**  
   - Add a Set node named `Conversion Summary`.  
   - Add a string field `summary` with expression:  
     ```
     Converted {{ $items().length }} PDFs to Markdown
     Total credits used: {{ $items().reduce((sum, item) => sum + (item.json.metadata.credits || 0), 0) }}
     ```  
   - Connect input from `Save Markdown Files`.

8. **Send Notification:**  
   - Add a Slack node named `Send Notification`.  
   - Configure Slack credentials with OAuth2 or incoming webhook.  
   - Set message to:  
     ```
     Batch Conversion Complete!

     {{ $json.summary }}

     Files saved to Google Drive.
     ```  
   - Connect input from `Conversion Summary`.

9. **Connect all nodes according to the flow:**  
   - `Google Drive - List PDFs` → `Filter PDFs Only` → `PDF Vector - Convert to Markdown` → `Prepare Output` → `Save Markdown Files` → `Conversion Summary` → `Send Notification`.

10. **Input Parameters:**  
    - Provide `folderId` for the source PDF folder in Google Drive at execution time.  
    - Provide `outputFolderId` for the destination folder to save Markdown files.  
    - Ensure Slack credentials are configured with appropriate channel access.

11. **Test the workflow:**  
    - Run with a folder containing PDFs.  
    - Verify Markdown files are generated and uploaded.  
    - Confirm Slack notification is received with correct summary.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                          |
|-------------------------------------------------------------------------------------------------|----------------------------------------|
| The workflow supports multiple PDF input sources but currently implements Google Drive listing. | Workflow Info sticky note                |
| PDF Vector node uses LLM-based parsing; ensure API keys or credentials for LLM services are set. | PDF Vector - Convert to Markdown node  |
| Slack notifications require valid Slack app credentials with chat:write permission.             | Send Notification node                  |
| Google Drive nodes require OAuth2 credentials with read/write scopes for the specified folders.  | Google Drive nodes                      |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.