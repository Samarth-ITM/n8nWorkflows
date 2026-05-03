Sync Google Drive files to an InfraNodus Knowledge Graph

https://n8nworkflows.xyz/workflows/sync-google-drive-files-to-an-infranodus-knowledge-graph-4495


# Sync Google Drive files to an InfraNodus Knowledge Graph

---

### 1. Workflow Overview

This workflow automates the synchronization of newly added files from a specified Google Drive folder into an InfraNodus knowledge graph. Its primary use case is to continuously monitor a Google Drive folder for new files, extract their textual content (supporting multiple file formats such as PDF, plain text, markdown, and JSON), and upload that content into InfraNodus for visualization and semantic analysis.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Listens to new file creation events in a specified Google Drive folder.
- **1.2 Iteration over Files:** Processes each new file individually by retrieving its content.
- **1.3 File Type Detection and Text Extraction:** Determines the file MIME type and extracts text accordingly using dedicated extraction nodes for PDF, plain text, and markdown files.
- **1.4 Upload to InfraNodus:** Sends the extracted text content to InfraNodus via its API, categorizing entries by filename for easy management.
- **1.5 Optional Enhancements:** Includes a disabled HTTP node for improved PDF conversion via ConvertAPI, providing higher fidelity text extraction when needed.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block is responsible for triggering the workflow whenever a new file is created in a pre-defined Google Drive folder.

- **Nodes Involved:**  
  - New File Created in the Google Folder?

- **Node Details:**

  - **New File Created in the Google Folder?**  
    - Type: Google Drive Trigger  
    - Role: Polling trigger node that listens for file creation events in a specific Google Drive folder.  
    - Configuration:  
      - Event: `fileCreated`  
      - File Type: All (`all`)  
      - Poll Interval: Every 1 minute  
      - Folder to Watch: Specified by folder ID `1NFekB1_H3ADF8C5o4eN_l893d6INbfRA` (named "Polysingularity")  
      - Credential: Google Drive OAuth2 account  
    - Inputs: None (trigger node)  
    - Outputs: Emits metadata for newly created files including file IDs and names  
    - Edge Cases:  
      - Missed events if polling interval is too long or Google API quota limits reached  
      - Folder ID must be valid and accessible by the OAuth2 user  
    - Notes: Requires prior creation of the Google Drive folder and proper OAuth2 credentials  

#### 2.2 Iteration over Files

- **Overview:**  
  Handles iterating over each new file detected by the trigger, processing them individually.

- **Nodes Involved:**  
  - Loop Over Items  
  - Retrieve File  
  - Switch  

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Splits the array of new files into individual items to process one by one, enabling sequential processing and error isolation.  
    - Configuration: Default batch size (1 by default)  
    - Input: Output from the Google Drive trigger (list of new files)  
    - Output: Single file metadata per execution  
    - Edge Cases: Large batch sizes could cause timeouts or API rate limits  

  - **Retrieve File**  
    - Type: Google Drive  
    - Role: Downloads the actual file content from Google Drive using the file ID.  
    - Configuration:  
      - Operation: Download  
      - File ID: Derived dynamically from current item’s `id` field (`={{ $json.id }}`)  
      - Credential: Google Drive OAuth2  
    - Input: Single file metadata from Loop Over Items  
    - Output: Binary file content and metadata including MIME type  
    - Edge Cases:  
      - File access permissions  
      - Network timeouts  
      - Invalid or missing file ID  

  - **Switch**  
    - Type: Switch  
    - Role: Determines file type by inspecting the binary data's MIME type and routes to the corresponding extraction method.  
    - Configuration:  
      - Multiple rules matching MIME types to outputs:  
        - `application/pdf` → `pdf`  
        - `text/plain` → `text`  
        - `text/markdown` → `md`  
        - `application/json` → `json` (unused downstream)  
        - Empty MIME type → `docs` (unused downstream)  
        - `csv` → `csv` (unused downstream)  
    - Input: Retrieved file binary data  
    - Output: Routed to extraction nodes based on detected MIME type  
    - Edge Cases:  
      - Unknown or unsupported MIME types lead to no processing (empty output)  
      - Inconsistent MIME type metadata  

#### 2.3 File Type Detection and Text Extraction

- **Overview:**  
  Extracts plain text from the retrieved file content according to the file type.

- **Nodes Involved:**  
  - Extract from PDF  
  - Extract from Text File  
  - Extract from Markdown  
  - Map PDF to Text  

- **Node Details:**

  - **Extract from PDF**  
    - Type: ExtractFromFile  
    - Role: Extracts text from PDF binary content.  
    - Configuration: Operation set to `pdf` extraction.  
    - Input: Binary PDF file from Switch node’s `pdf` output  
    - Output: JSON with extracted text in a field named `text`  
    - Edge Cases:  
      - Complex PDFs with images or scanned content may yield poor extraction  

  - **Extract from Text File**  
    - Type: ExtractFromFile  
    - Role: Extracts text from plain text files.  
    - Configuration: Operation set to `text` extraction.  
    - Input: Binary plain text file from Switch node’s `text` output  
    - Output: JSON with extracted text in `text` field  
    - Edge Cases:  
      - Encoding issues if text files are not UTF-8  

  - **Extract from Markdown**  
    - Type: ExtractFromFile  
    - Role: Extracts text from markdown files, treated as plain text.  
    - Configuration: Operation set to `text` extraction.  
    - Input: Binary markdown file from Switch node’s `md` output  
    - Output: JSON with extracted text in `text` field  
    - Edge Cases:  
      - Markdown formatting is ignored, raw text extracted  

  - **Map PDF to Text**  
    - Type: Set  
    - Role: Remaps the extracted text field (`text`) from the PDF extraction node into a field named `data` to conform to InfraNodus API expectations.  
    - Configuration: Field `data` is set to the expression `={{ $json.text }}`  
    - Input: Extract from PDF output  
    - Output: JSON with field `data` containing extracted text  
    - Edge Cases: None significant  

#### 2.4 Upload to InfraNodus

- **Overview:**  
  Sends the extracted text content to InfraNodus knowledge graph API, tagging entries with the file name and specific processing options.

- **Nodes Involved:**  
  - InfraNodus Save to Graph  

- **Node Details:**

  - **InfraNodus Save to Graph**  
    - Type: HTTP Request  
    - Role: Uploads the extracted textual data to InfraNodus via its API, creating or updating a graph named `polysingularity_from_google_drive`.  
    - Configuration:  
      - URL: `https://infranodus.com/api/v1/graphAndStatements?doNotSave=false&optimize=develop&includeGraph=false&includeGraphSummary=true&includeGraph=false`  
      - Method: POST  
      - Authentication: HTTP Bearer Auth using InfraNodus API Key credential  
      - Body parameters:  
        - `name`: Fixed graph name `polysingularity_from_google_drive`  
        - `text`: Extracted text passed dynamically (`={{ $json.data }}`)  
        - `=categories`: Dynamically sets a category using the filename from the Switch node (`=[filename: {{ $('Switch').item.json.name }}]`)  
        - `contextSettings`: JSON string specifying `squareBracketsProcessing` set to `"IGNORE_BRACKETS"`  
    - Input: JSON with `data` field containing text  
    - Output: InfraNodus API response including graph summary  
    - Edge Cases:  
      - API authentication errors  
      - Network timeouts  
      - Malformed text data causing API rejection  

  - **Connection:** After successful upload, the workflow loops back to process the next file in `Loop Over Items`, enabling continuous processing.

#### 2.5 Optional Enhancements

- **Overview:**  
  Provides a disabled HTTP Request node offering higher quality PDF-to-text conversion via ConvertAPI, which can be enabled and used to replace the standard PDF extraction node.

- **Nodes Involved:**  
  - Convert File to PDF (disabled)

- **Node Details:**

  - **Convert File to PDF**  
    - Type: HTTP Request  
    - Role: Requests ConvertAPI to convert PDF files into text with better layout preservation.  
    - Configuration:  
      - URL: `https://v2.convertapi.com/convert/pdf/to/txt`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body: Sends binary PDF data from previous node  
      - Authentication: HTTP Bearer Auth with ConvertAPI key  
      - Response format: Text  
    - Input: Binary PDF data (disabled)  
    - Output: Converted text content  
    - Edge Cases:  
      - Requires valid ConvertAPI subscription  
      - API rate limits and costs apply  
      - Network failures  

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                               | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                                                        |
|-------------------------------|---------------------|-----------------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| New File Created in the Google Folder? | Google Drive Trigger | Detects new files added to specific Google Drive folder | (Trigger)                   | Loop Over Items             | ## 1. Listen to a Google Drive folder / create new file event<br>Specify the folder you want to use here                                            |
| Loop Over Items               | SplitInBatches      | Iterates over each new file individually       | New File Created in the Google Folder? | Retrieve File              | ## 2. Iterate through every file extracted<br>We get the ID of the file and iterate through them one by one                                        |
| Retrieve File                | Google Drive        | Downloads file content given a file ID         | Loop Over Items             | Switch                     |                                                                                                                                                    |
| Switch                      | Switch              | Routes file processing based on MIME type     | Retrieve File               | Extract from PDF, Extract from Text File, Extract from Markdown |                                                                                                                                                    |
| Extract from PDF              | ExtractFromFile     | Extracts text from PDF files                   | Switch (pdf output)         | Map PDF to Text            | ## 3. Extract text from the files<br>Use the built-in n8n extractor tools to convert different file formats: PDF, txt, or markdown to plain text. |
| Map PDF to Text              | Set                 | Maps extracted PDF text to 'data' field       | Extract from PDF            | InfraNodus Save to Graph   |                                                                                                                                                    |
| Extract from Text File        | ExtractFromFile     | Extracts text from plain text files            | Switch (text output)        | InfraNodus Save to Graph   |                                                                                                                                                    |
| Extract from Markdown         | ExtractFromFile     | Extracts text from markdown files              | Switch (md output)          | InfraNodus Save to Graph   |                                                                                                                                                    |
| InfraNodus Save to Graph      | HTTP Request        | Uploads extracted text to InfraNodus graph     | Map PDF to Text, Extract from Text File, Extract from Markdown | Loop Over Items             | ## 4. Save the file to InfraNodus<br>Save the text content of the file to the graph, using its name as category and additional processing settings. |
| Convert File to PDF (disabled) | HTTP Request        | Optional: Better PDF to text conversion via ConvertAPI | (Not connected)             | (Not connected)             | ## Optional: Better PDF Conversion<br>Use ConvertAPI for high-quality PDF to text conversion. Replace "Map PDF to Text" node if enabled.             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Type: Google Drive Trigger  
   - Name: `New File Created in the Google Folder?`  
   - Set event to `fileCreated`  
   - Set file type to `all`  
   - Set poll interval to every 1 minute  
   - Specify folder to watch by folder ID (e.g., `1NFekB1_H3ADF8C5o4eN_l893d6INbfRA`)  
   - Attach Google Drive OAuth2 credentials  

2. **Create SplitInBatches Node**  
   - Type: SplitInBatches  
   - Name: `Loop Over Items`  
   - Connect output of Google Drive Trigger to this node's input  
   - Use default batch size (1)  

3. **Create Google Drive Node**  
   - Type: Google Drive  
   - Name: `Retrieve File`  
   - Set operation to `download`  
   - Set File ID parameter to expression `={{ $json.id }}` (dynamic from current batch item)  
   - Attach Google Drive OAuth2 credentials  
   - Connect output of `Loop Over Items` node to this node  

4. **Create Switch Node**  
   - Type: Switch  
   - Name: `Switch`  
   - Add rules to match the binary data MIME type (`{{$binary["data"].mimeType}}`) to route:  
     - `application/pdf` → output `pdf`  
     - `text/plain` → output `text`  
     - `text/markdown` → output `md`  
     - `application/json`, empty string, and `csv` can be added if desired but are unused downstream  
   - Connect output of `Retrieve File` node to this node  

5. **Create ExtractFromFile Nodes for each supported format:**  
   - **Extract from PDF:**  
     - Type: ExtractFromFile  
     - Name: `Extract from PDF`  
     - Set operation to `pdf`  
     - Connect `pdf` output of Switch to this node  
   - **Extract from Text File:**  
     - Type: ExtractFromFile  
     - Name: `Extract from Text File`  
     - Set operation to `text`  
     - Connect `text` output of Switch to this node  
   - **Extract from Markdown:**  
     - Type: ExtractFromFile  
     - Name: `Extract from Markdown`  
     - Set operation to `text`  
     - Connect `md` output of Switch to this node  

6. **Create Set Node for PDF text mapping:**  
   - Type: Set  
   - Name: `Map PDF to Text`  
   - Add field assignment:  
     - Name: `data`  
     - Value: Expression `={{ $json.text }}`  
   - Connect output of `Extract from PDF` to this node  

7. **Create HTTP Request Node to upload to InfraNodus:**  
   - Type: HTTP Request  
   - Name: `InfraNodus Save to Graph`  
   - Set method: POST  
   - Set URL: `https://infranodus.com/api/v1/graphAndStatements?doNotSave=false&optimize=develop&includeGraph=false&includeGraphSummary=true&includeGraph=false`  
   - Authentication: HTTP Bearer Auth using InfraNodus API Key credential  
   - Body type: `Body Parameters` (JSON) with parameters:  
     - `name`: fixed string `polysingularity_from_google_drive`  
     - `text`: expression `={{ $json.data }}` (or `{{ $json.text }}` for non-PDF extractors)  
     - `=categories`: expression `=[filename: {{ $('Switch').item.json.name }}]` to use the original filename as category  
     - `contextSettings`: JSON string `{ "squareBracketsProcessing":"IGNORE_BRACKETS" }`  
   - Connect outputs:  
     - Connect `Map PDF to Text` to this node  
     - Connect outputs of `Extract from Text File` and `Extract from Markdown` directly to this node  

8. **Loop Back to Process Next File:**  
   - Connect the output of `InfraNodus Save to Graph` back to `Loop Over Items` node to process next file  

9. **Optional: Create HTTP Request Node for ConvertAPI PDF Conversion (disabled by default):**  
   - Type: HTTP Request  
   - Name: `Convert File to PDF`  
   - Method: POST  
   - URL: `https://v2.convertapi.com/convert/pdf/to/txt`  
   - Content-Type: multipart/form-data  
   - Body: binary file from `Retrieve File` node  
   - Authentication: HTTP Bearer Auth with ConvertAPI credentials  
   - Response format: text  
   - Note: Disable this node by default; can replace `Extract from PDF` node if enabled  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                            | Context or Link                                                                                                 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow listens to newly added files in a specified Google Drive folder and automatically uploads their text content to an InfraNodus knowledge graph for semantic visualization and analysis.                                                     | InfraNodus homepage: https://infranodus.com                                                                   |
| InfraNodus visualizes text as a knowledge graph, highlighting main topics and ideas. It also offers an API for programmatic graph management.                                                                                                          | InfraNodus API documentation: https://infranodus.com/api                                                      |
| When deleting files from Google Drive, manual removal from InfraNodus is required to keep graphs synchronized.                                                                                                                                        |                                                                                                               |
| Optional higher-quality PDF to text conversion can be achieved via ConvertAPI, which respects original document layout better than the standard extractor node.                                                                                       | ConvertAPI: https://convertapi.com?ref=4l54n                                                                   |
| Detailed tutorial on this workflow available at Nodus Labs support: [Upload/Sync Your Google Drive Folder with InfraNodus using n8n](https://support.noduslabs.com/hc/en-us/articles/20267019838108-Upload-Sync-Your-Google-Drive-Folder-with-InfraNodus-using-n8n) |                                                                                                               |
| Reminder: Ensure Google Drive OAuth2 credentials have proper permissions for the target folder and InfraNodus API key is valid and active.                                                                                                            |                                                                                                               |

---

This completes the comprehensive technical documentation and reconstruction guide for the "Sync Google Drive files to an InfraNodus Graph" n8n workflow.

---