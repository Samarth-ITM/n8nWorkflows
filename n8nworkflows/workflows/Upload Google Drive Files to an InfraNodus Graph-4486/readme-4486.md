Upload Google Drive Files to an InfraNodus Graph

https://n8nworkflows.xyz/workflows/upload-google-drive-files-to-an-infranodus-graph-4486


# Upload Google Drive Files to an InfraNodus Graph

### 1. Workflow Overview

This workflow automates the process of uploading files stored in a specified Google Drive folder into an InfraNodus knowledge graph. It is designed to extract textual content from various file formats (PDF, plain text, Markdown, JSON, CSV, etc.) and push this content with metadata into InfraNodus, where it can be visualized and analyzed as a graph of interconnected ideas.

**Target Use Cases:**  
- Syncing documents from Google Drive to InfraNodus for knowledge visualization and analysis.  
- Automating content ingestion from multiple file formats into a graph database.  
- Structuring unstructured document data for AI-powered insights.

**Logical Blocks:**  
- **1.1 Trigger & Google Drive File Retrieval:** Initiates manually and fetches files from a specified Google Drive folder.  
- **1.2 File Processing Loop:** Iterates over each file to download and process it.  
- **1.3 File Type Switching & Extraction:** Determines the file type and extracts plain text accordingly using built-in extractors.  
- **1.4 Text Mapping & InfraNodus Upload:** Maps extracted text and metadata into a format accepted by InfraNodus and uploads it via HTTP request.  
- **1.5 Optional High-Quality PDF Conversion:** (Disabled) Provides an alternative PDF-to-text conversion using an external API for better text extraction quality.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Google Drive File Retrieval  
**Overview:** This block starts the workflow manually and retrieves all files from a given Google Drive folder for processing.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Search Google Drive (Google Drive API node)  
- Sticky Note (instructional)

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts workflow execution on demand.  
  - *Config:* No parameters; simple manual start.  
  - *Connections:* Outputs to "Search Google Drive."  
  - *Potential Failures:* None typically, except accidental multiple triggers.  

- **Search Google Drive**  
  - *Type:* Google Drive node (resource: fileFolder)  
  - *Role:* Lists all files inside a particular folder ID in Google Drive.  
  - *Config:* Folder ID is set to "1ep1yoEl7vxMdrTmvdlDnOXRlXwdxZTzX" (must be replaced with user’s folder ID). Returns all files matching query "*".  
  - *Credentials:* Requires Google Drive OAuth2 credentials.  
  - *Connections:* Outputs all files to "Loop Over Items" node.  
  - *Edge Cases:*  
    - Invalid folder ID or permissions errors.  
    - Empty folder returns empty list; downstream nodes must handle no items gracefully.  

- **Sticky Note (Instructional)**  
  - Content explains the need to create a Google Drive folder and upload files there, highlighting where to specify the folder ID.

---

#### 1.2 File Processing Loop  
**Overview:** Iterates over each retrieved file, downloading it individually for further processing.

**Nodes Involved:**  
- Loop Over Items (Split in Batches)  
- Retrieve File (Google Drive Download)  
- Sticky Note (instructional)

**Node Details:**  

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Role:* Processes files one by one to avoid overload and manage sequential processing.  
  - *Config:* Default options; batch size is likely 1 (default).  
  - *Connections:* On first output path, loops back after upload; second output path flows to "Retrieve File."  
  - *Edge Cases:* Empty input (no files) leads to no iterations; handles gracefully.  

- **Retrieve File**  
  - *Type:* Google Drive node (operation: download)  
  - *Role:* Downloads the actual file content using file ID from each iteration.  
  - *Config:* File ID dynamically set from current item JSON (`{{$json.id}}`).  
  - *Credentials:* Google Drive OAuth2 credentials.  
  - *Connections:* Forwards the downloaded file's binary data to the "Switch" node.  
  - *Edge Cases:*  
    - File may be deleted between listing and download, causing errors.  
    - Large files may cause timeouts or memory issues.  

- **Sticky Note (Instructional)**  
  - Explains the iteration over every file and extraction of file IDs.

---

#### 1.3 File Type Switching & Extraction  
**Overview:** Determines the MIME type of each downloaded file and applies the correct extraction method to convert it into plain text.

**Nodes Involved:**  
- Switch (MIME Type check)  
- Extract from PDF  
- Extract from Text File  
- Extract from Markdown  
- Sticky Note (instructional)

**Node Details:**  

- **Switch**  
  - *Type:* Switch node  
  - *Role:* Routes the file according to MIME type to the appropriate extractor.  
  - *Config:* Checks `$binary["data"].mimeType` against known types:  
    - `application/pdf` → outputKey "pdf"  
    - `text/plain` → outputKey "text"  
    - `text/markdown` → outputKey "md"  
    - `application/json` → outputKey "json" (no downstream action here)  
    - Empty MIME → outputKey "docs" (no downstream)  
    - `csv` → outputKey "csv" (no downstream)  
  - *Connections:* Routes PDFs to "Extract from PDF," plain text to "Extract from Text File," markdown to "Extract from Markdown."  
  - *Edge Cases:*  
    - Unsupported MIME types are not handled (e.g., images).  
    - MIME detection failure leads to no output.  

- **Extract from PDF**  
  - *Type:* ExtractFromFile node  
  - *Role:* Extracts text from PDF binary.  
  - *Config:* Operation set to "pdf."  
  - *Connections:* Outputs extracted text to "Map PDF to Text."  
  - *Edge Cases:* Complex PDFs with images or scanned text may result in incomplete extraction.  

- **Extract from Text File**  
  - *Type:* ExtractFromFile node  
  - *Role:* Extracts text from plain text files.  
  - *Config:* Operation set to "text."  
  - *Connections:* Outputs extracted text to "InfraNodus Save to Graph."  

- **Extract from Markdown**  
  - *Type:* ExtractFromFile node  
  - *Role:* Extracts text from Markdown files.  
  - *Config:* Operation set to "text."  
  - *Connections:* Outputs extracted text to "InfraNodus Save to Graph."  

- **Sticky Note (Instructional)**  
  - Notes the use of n8n’s built-in extractor tools for multiple file types.

---

#### 1.4 Text Mapping & InfraNodus Upload  
**Overview:** Maps extracted text into a JSON structure with metadata and uploads it to InfraNodus via HTTP API, associating the text with the original file name as a category.

**Nodes Involved:**  
- Map PDF to Text (Set node)  
- InfraNodus Save to Graph (HTTP Request node)  
- Sticky Note (instructional)

**Node Details:**  

- **Map PDF to Text**  
  - *Type:* Set node  
  - *Role:* Normalizes PDF extracted text into a `data` property for uniform downstream processing.  
  - *Config:* Sets `data` to the extracted text (`{{$json.text}}`).  
  - *Connections:* Outputs to "InfraNodus Save to Graph."  
  - *Edge Cases:* If extraction fails, `data` may be empty or undefined.  

- **InfraNodus Save to Graph**  
  - *Type:* HTTP Request node  
  - *Role:* Posts text data to InfraNodus API to save or update a graph.  
  - *Config:*  
    - URL: InfraNodus API endpoint with parameters to optimize and include summaries.  
    - Method: POST  
    - Authentication: HTTP Bearer Auth with InfraNodus API Key.  
    - Body parameters:  
      - `name`: fixed graph name "graphrag_from_google_drive" (creates or updates this graph)  
      - `text`: the extracted text from file (`{{$json.data}}`)  
      - `categories`: dynamically set to `[filename: <file name>]` to tag content by source file.  
      - `contextSettings`: disables processing of square brackets (to avoid interference with InfraNodus syntax).  
  - *Connections:* Loops back to "Loop Over Items" to process next file.  
  - *Edge Cases:*  
    - API key invalid or expired causes auth failures.  
    - Network errors or API rate limiting can interrupt uploads.  
    - Extremely large texts may hit API limits.  

- **Sticky Note (Instructional)**  
  - Explains the use of file name as category and the option to specify existing or new graph name.

---

#### 1.5 Optional High-Quality PDF Conversion (Disabled)  
**Overview:** Provides an alternative to n8n’s default PDF extractor by calling ConvertAPI to convert PDFs to text with improved layout preservation.

**Nodes Involved:**  
- Convert File to PDF (HTTP Request node) [Disabled]  
- Sticky Note (instructional)

**Node Details:**  

- **Convert File to PDF**  
  - *Type:* HTTP Request node (disabled)  
  - *Role:* Sends PDF binary to ConvertAPI’s PDF-to-text conversion endpoint to get better text extraction results.  
  - *Config:*  
    - Method: POST  
    - Content-Type: multipart/form-data with file binary data  
    - Authentication: HTTP Bearer Auth with ConvertAPI key  
    - Response: text/plain  
  - *Connections:* None active; disabled.  
  - *Edge Cases:* Requires valid ConvertAPI subscription and key; disabled by default to avoid extra cost.  

- **Sticky Note (Instructional)**  
  - Explains why and how to use ConvertAPI as an alternative for better PDF conversion quality.

---

### 3. Summary Table

| Node Name                | Node Type               | Functional Role                           | Input Node(s)           | Output Node(s)                            | Sticky Note                                                                                          |
|--------------------------|-------------------------|-----------------------------------------|------------------------|------------------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger          | Starts workflow manually                 | —                      | Search Google Drive                      |                                                                                                    |
| Search Google Drive       | Google Drive            | Lists files in specified Drive folder   | When clicking ‘Test workflow’ | Loop Over Items                          | ## 1. Retrieve the files from a Google Drive folder\n\n### You need to create the folder first and upload the files there. \n\n🚨 Specify the folder you want to use here |
| Loop Over Items           | SplitInBatches          | Iterates over each file                  | Search Google Drive     | (1) Retrieve File (2) Loop Over Items (loop back) | ## 2. Iterate through every file extracted\n\n### We get the ID of the file and iterate through them one by one |
| Retrieve File             | Google Drive            | Downloads file content                   | Loop Over Items         | Switch                                 |                                                                                                    |
| Switch                   | Switch                  | Routes based on file MIME type           | Retrieve File           | Extract from PDF / Extract from Text File / Extract from Markdown | ## 3. Extract text from the files\n\n### Use the built-in n8n extractor tools to convert different file formats: PDF, txt, or markdown to plain text. |
| Extract from PDF          | ExtractFromFile         | Extracts text from PDF                   | Switch (pdf)            | Map PDF to Text                        |                                                                                                    |
| Extract from Text File    | ExtractFromFile         | Extracts text from plain text files     | Switch (text)           | InfraNodus Save to Graph               |                                                                                                    |
| Extract from Markdown     | ExtractFromFile         | Extracts text from markdown files       | Switch (md)             | InfraNodus Save to Graph               |                                                                                                    |
| Map PDF to Text           | Set                     | Normalizes PDF extracted text            | Extract from PDF        | InfraNodus Save to Graph               |                                                                                                    |
| InfraNodus Save to Graph  | HTTP Request            | Posts extracted text to InfraNodus API  | Extract from Text File / Extract from Markdown / Map PDF to Text | Loop Over Items (loop back)               | ## 4. Save the file to InfraNodus\n\n### Save the text content of the file to the graph, using its name as category (so you can manually delete it after) and additional processing settings (in the `body` field of the POST request.\n\nNote: you can use an existing graph name (the `name` field) or specify a new name and the graph will be created. |
| Convert File to PDF       | HTTP Request (disabled) | Optional: High-quality PDF to text conversion | —                      | —                                      | ## Optional: Better PDF Conversion\n\n### Standard Map PDF to Text node will split your PDF files into very short chunks, which deteriorates retrieval. \n\nUse can use [ConvertAPI](https://convertapi.com?ref=4l54n) which is a high-quality convertor that will respect the layout of the original document and not cut the paragraphs into short chunks. \n\nHere is an HTTP node that makes a request to their API to convert the PDF into text. If you have a ConvertAPI account, you can replace the \"Map PDF to Text\" node in step 4 with this node. |
| Sticky Note               | Sticky Note             | Instructional                           | —                      | —                                      | # Upload Google Drive files to an InfraNodus graph\n\n## This workflow uploads the files in your Google drive to your [InfraNodus graph](https://infranodus.com). \n\n### InfraNodus visualizes your text as a knowledge graph, showing the main topics and ideas inside. It also provides API access to your knownledge graphs, so you can use them as \"experts\" for your AI agent n8n workflows.\n\nYou need an [InfraNodus](https://infranodus.com) account to use this workflow.\n\nDetailed tutorial on this workflow: [https://support.noduslabs.com/hc/en-us/articles/20267019838108-Upload-Sync-Your-Google-Drive-Folder-with-InfraNodus-using-n8n](https://support.noduslabs.com/hc/en-us/articles/20267019838108-Upload-Sync-Your-Google-Drive-Folder-with-InfraNodus-using-n8n)\n\n🚨 launch it only once to avoid having duplicate content added\n\n\n![InfraNodus Graph](https://infranodus.com/images/front/infranodus-overview.jpg) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node named "When clicking ‘Test workflow’". No parameters needed.  

2. **Add Google Drive Search Node**  
   - Add a "Google Drive" node named "Search Google Drive".  
   - Set resource to "fileFolder" and operation to list files.  
   - Set folderId to your target Google Drive folder ID (replace "1ep1yoEl7vxMdrTmvdlDnOXRlXwdxZTzX").  
   - Set "Return All" to true and query string to "*".  
   - Connect "When clicking ‘Test workflow’" output to this node input.  
   - Configure Google Drive OAuth2 credentials.

3. **Add SplitInBatches Node**  
   - Add a "SplitInBatches" node named "Loop Over Items".  
   - Connect output of "Search Google Drive" to this node.  
   - Use default batch size (1 by default).  

4. **Add Google Drive Download Node**  
   - Add a "Google Drive" node named "Retrieve File".  
   - Set operation to "download".  
   - Set fileId parameter to `={{ $json.id }}` (dynamic from current item).  
   - Connect first output of "Loop Over Items" to this node.  
   - Use the same Google Drive OAuth2 credentials.

5. **Add Switch Node for MIME Type**  
   - Add a "Switch" node named "Switch".  
   - Set condition to check binary data mimeType: `{{$binary["data"].mimeType}}`.  
   - Add cases for:  
     - "application/pdf" → output "pdf"  
     - "text/plain" → output "text"  
     - "text/markdown" → output "md"  
     - "application/json" → output "json" (optional)  
     - "" (empty) → output "docs" (optional)  
     - "csv" → output "csv" (optional)  
   - Connect output of "Retrieve File" to this Switch node.

6. **Add ExtractFromFile Nodes for Text Extraction**  
   - Add "Extract from PDF" node:  
     - Operation: "pdf"  
     - Connect Switch output "pdf" to this node.  
   - Add "Extract from Text File" node:  
     - Operation: "text"  
     - Connect Switch output "text" to this node.  
   - Add "Extract from Markdown" node:  
     - Operation: "text"  
     - Connect Switch output "md" to this node.  

7. **Add Set Node to Normalize PDF Text**  
   - Add a "Set" node named "Map PDF to Text".  
   - Set a field "data" to value `={{ $json.text }}` from extracted PDF text.  
   - Connect "Extract from PDF" output to "Map PDF to Text".  

8. **Add HTTP Request Node to Save to InfraNodus**  
   - Add an "HTTP Request" node named "InfraNodus Save to Graph".  
   - Set method to POST.  
   - Set URL to:  
     `https://infranodus.com/api/v1/graphAndStatements?doNotSave=false&optimize=develop&includeGraph=false&includeGraphSummary=true&includeGraph=false`  
   - Authentication: HTTP Bearer Auth using your InfraNodus API key credentials.  
   - Body parameters (send as form or JSON):  
     - `name`: "graphrag_from_google_drive" (graph name)  
     - `text`: `={{ $json.data }}` or `={{ $json.text }}` depending on previous node  
     - `=categories`: `=[filename: {{ $('Switch').item.json.name }}]` (file name as category)  
     - `contextSettings`: `={{{ "squareBracketsProcessing":"IGNORE_BRACKETS" }}}` (prevents bracket processing)  
   - Connect outputs from:  
     - "Map PDF to Text" (for PDFs)  
     - "Extract from Text File" (for text files)  
     - "Extract from Markdown" (for markdown files)  
   - Connect output of this node back to the second output of "Loop Over Items" node to continue processing next file.

9. **(Optional) Add ConvertAPI HTTP Node for Better PDF Conversion**  
   - Add an HTTP Request node named "Convert File to PDF".  
   - Set to POST to ConvertAPI endpoint for pdf-to-txt conversion.  
   - Authentication: HTTP Bearer Auth with ConvertAPI key.  
   - Content-Type: multipart/form-data, send binary "data".  
   - Use this node instead of "Extract from PDF" and "Map PDF to Text" if you have a ConvertAPI account and want improved PDF text extraction.  
   - Note: This node is disabled by default.

10. **Add Sticky Notes for Documentation**  
    - Add sticky notes as per the original workflow to guide users at each step, especially for folder ID setup, iteration explanation, extraction notes, and InfraNodus API usage.

11. **Test Workflow**  
    - Manually trigger the workflow.  
    - Confirm files from the chosen Google Drive folder are processed and uploaded to the InfraNodus graph.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                           | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow uploads the files in your Google drive to your InfraNodus graph. InfraNodus visualizes your text as a knowledge graph, showing the main topics and ideas inside. It also provides API access to your knowledge graphs for AI workflows.                      | [InfraNodus](https://infranodus.com)                                                                         |
| You need an InfraNodus account to use this workflow.                                                                                                                                                                                                                   | https://infranodus.com                                                                                        |
| Detailed tutorial on this workflow: [Upload/Sync Your Google Drive Folder with InfraNodus using n8n](https://support.noduslabs.com/hc/en-us/articles/20267019838108-Upload-Sync-Your-Google-Drive-Folder-with-InfraNodus-using-n8n)                                       | https://support.noduslabs.com/hc/en-us/articles/20267019838108-Upload-Sync-Your-Google-Drive-Folder-with-InfraNodus-using-n8n |
| Optional higher quality PDF to text conversion available via ConvertAPI. ConvertAPI respects layout better than the built-in extractor and avoids splitting PDFs into very short chunks. Requires ConvertAPI subscription and key.                                       | https://convertapi.com?ref=4l54n                                                                              |
| 🚨 Launch this workflow only once per folder update to avoid duplicate content ingestion in InfraNodus.                                                                                                                                                                | —                                                                                                           |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.