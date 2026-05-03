Segment PDFs by Table of Contents with Gemini AI and Chunkr.ai

https://n8nworkflows.xyz/workflows/segment-pdfs-by-table-of-contents-with-gemini-ai-and-chunkr-ai-4697


# Segment PDFs by Table of Contents with Gemini AI and Chunkr.ai

---

### 1. Workflow Overview

This workflow automates the extraction and structuring of content from PDF documents by segmenting them according to their Table of Contents (ToC). It integrates Chunkr.ai for document parsing and Google Gemini AI for intelligent ToC construction. The workflow supports two main entry points: manual trigger for a fixed Google Drive PDF and webhook trigger for external workflows providing a PDF URL. It processes the PDF into chunks, extracts or infers the ToC hierarchies, maps document sections accordingly, and outputs the results both as individual sections and as complete HTML and Markdown documents.

Logical blocks:

- **1.1 Input Reception and PDF Acquisition**: Handles manual or external triggers to download PDFs from Google Drive or a URL.
- **1.2 PDF Processing with Chunkr.ai**: Converts PDF to base64 and sends it to Chunkr.ai for chunked parsing, polling until processing completes.
- **1.3 Table of Contents Extraction and AI Processing**: Extracts potential section headers, selects initial document chunks, and uses Google Gemini AI Agent to build a nested JSON ToC.
- **1.4 Section Mapping and Content Extraction**: Maps the AI-generated ToC to Chunkr chunks, extracts section content in text, HTML, and Markdown formats.
- **1.5 Output Generation**: Produces individual section items or combined full document outputs in HTML and Markdown files.
- **1.6 Error Handling and Status Management**: Switches logic based on Chunkr task status, with error stop node.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and PDF Acquisition

**Overview:**  
Handles workflow triggering and PDF acquisition either manually or via another workflow providing a PDF URL. Downloads PDF from Google Drive (manual) or HTTP URL (webhook).

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (manual trigger, disabled)  
- When Executed by Another Workflow (webhook trigger)  
- Download PDF from Google Drive (disabled)  
- Download PDF from URL  
- Merge  

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual start of workflow for testing or fixed input  
  - Configuration: Disabled by default  
  - Input/Output: Starts workflow, outputs trigger data  
  - Edge Cases: Disabled node, must be enabled manually  
  - Sticky Note: Explains manual trigger usage and connection to Google Drive PDF

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger (Webhook-like)  
  - Role: Receives input from external workflows, expects `URL` parameter  
  - Inputs: JSON with `URL` of PDF to process  
  - Outputs: Triggers next nodes for PDF download  
  - Edge Cases: Missing or invalid URL input may cause failure  
  - Sticky Note: Explains usage for external triggers

- **Download PDF from Google Drive**  
  - Type: Google Drive  
  - Role: Downloads predefined PDF file by `fileId`  
  - Configuration: Disabled (used with manual trigger)  
  - Credentials: Google Drive OAuth2  
  - Output: Binary PDF file  
  - Edge Cases: Auth failure, file not found, rate limits

- **Download PDF from URL**  
  - Type: HTTP Request  
  - Role: Downloads PDF from given URL (from webhook input)  
  - Configuration: URL dynamically set from workflow input (`{{$json.URL}}`)  
  - Edge Cases: HTTP errors, invalid URL, network timeouts

- **Merge**  
  - Type: Merge Node  
  - Role: Combines outputs from download sources (Google Drive or URL)  
  - Configuration: Defaults to merge latest data from two sources  
  - Output: Merged binary PDF for further processing

---

#### 1.2 PDF Processing with Chunkr.ai

**Overview:**  
Converts the PDF to base64, names the file, sends it to Chunkr.ai for parsing, waits and polls for task completion.

**Nodes Involved:**  
- Convert the PDF to base64  
- Set File Name  
- POST Chunkr Task  
- Wait Before Polling the Chunkr Result  
- GET Chunkr Task  
- Status is:  
- Stop and Error  

**Node Details:**

- **Convert the PDF to base64**  
  - Type: Extract from File  
  - Role: Converts binary PDF into base64 string stored in JSON property `data`  
  - Input: Binary PDF from Merge  
  - Output: JSON with base64 encoded file  
  - Edge Cases: Binary missing or corrupt input

- **Set File Name**  
  - Type: Set  
  - Role: Defines variables `fileName`, `fileNameSnake` (snake_case), `createdAt` timestamp based on original PDF filename  
  - Uses expression to remove `.pdf` extension and convert to snake case  
  - Output: JSON with these variables for further API calls

- **POST Chunkr Task**  
  - Type: HTTP Request  
  - Role: Sends base64 PDF to Chunkr.ai API to start parsing task  
  - Configuration: POST to `https://api.chunkr.ai/api/v1/task/parse` with JSON body including base64 file and filename  
  - Authorization: Requires Chunkr.ai API key in header  
  - Output: Task ID and initial status from Chunkr.ai  
  - Edge Cases: Auth errors, API limits, invalid base64 or filename  
  - Sticky Note: Reminds to insert Chunkr.ai API key here

- **Wait Before Polling the Chunkr Result**  
  - Type: Wait  
  - Role: Waits 10 seconds before polling Chunkr.ai for task status  
  - Output: Delays workflow to allow processing time

- **GET Chunkr Task**  
  - Type: HTTP Request  
  - Role: Polls Chunkr.ai API for task status and results using Task ID from POST node  
  - Authorization: Chunkr.ai API key required  
  - Output: JSON with task status and parsed chunks when ready  
  - Edge Cases: API errors, rate limits, task failure

- **Status is:**  
  - Type: Switch  
  - Role: Routes workflow based on `status` field from Chunkr.ai response: `Succeeded`, `Processing`, or `Failed`  
  - Output: Routes to next steps or error

- **Stop and Error**  
  - Type: Stop and Error  
  - Role: Stops workflow with error message if Chunkr task failed

---

#### 1.3 Table of Contents Extraction and AI Processing

**Overview:**  
Extracts raw section headers as fallback, extracts beginning chunks for ToC detection, uses Google Gemini AI with output parsers to build a structured nested JSON Table of Contents.

**Nodes Involved:**  
- Take beginning of Document to look for Table of contents (Code)  
- Extract Sections headers as fallback (Code)  
- Google Gemini Chat Model  
- Auto-fixing Output Parser  
- Structured Output Parser  
- Table of Content Agent (AI Agent)  

**Node Details:**

- **Take beginning of Document to look for Table of contents**  
  - Type: Code  
  - Role: Extracts text content of first 10 chunks from Chunkr output to use as primary ToC source for AI  
  - Output: Single concatenated string of initial document chunks  
  - Edge Cases: Fewer than 10 chunks, missing data  
  - Sticky Note: Explains purpose and usage

- **Extract Sections headers as fallback**  
  - Type: Code  
  - Role: Iterates through all chunks and segments to extract unique `SectionHeader` segment contents as fallback headings  
  - Output: Array of unique headings  
  - Sticky Note: Explains fallback purpose if no clear ToC found

- **Google Gemini Chat Model**  
  - Type: AI Language Model (Google Gemini)  
  - Role: Processes input text to understand document structure and generate ToC JSON  
  - Model: `models/gemini-2.5-pro-preview-05-06`  
  - Credentials: Google Palm API (OAuth2)  
  - Input: Text from initial chunks and fallback headings  
  - Output: AI-generated ToC JSON  
  - Edge Cases: API errors, rate limits, malformed AI output

- **Auto-fixing Output Parser**  
  - Type: Langchain Output Parser Autofixing  
  - Role: Automatically fixes minor inconsistencies in AI output JSON  
  - Input: AI raw output  
  - Output: Cleaned structured JSON

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser  
  - Role: Validates AI output against explicit JSON schema for nested ToC (up to 3 levels)  
  - Output: Parsed JSON with hierarchical ToC array

- **Table of Content Agent**  
  - Type: AI Agent  
  - Role: Combines AI output parsing with fallback data to finalize ToC nested JSON structure  
  - Input: Cleaned ToC JSON and fallback headings  
  - Output: Final nested ToC JSON object  
  - Sticky Note: Explains combination of two sources for ToC

---

#### 1.4 Section Mapping and Content Extraction

**Overview:**  
Maps the AI-generated ToC headings to Chunkr chunks by matching headings to chunks and segments, extracts the text, HTML, and Markdown content for each section.

**Nodes Involved:**  
- Return each section individually (Code)  
- Return the whole document (Code)  

**Node Details:**

- **Return each section individually**  
  - Type: Code  
  - Role:  
    - Flattens ToC hierarchy with heading levels  
    - Identifies a chunk likely containing the PDF’s own ToC by matching heading counts  
    - Performs a two-pass mapping of headings to Chunkr segments: exact match on 'SectionHeader' segments, then content includes search  
    - Extracts section contents between heading boundaries as text, HTML, and Markdown  
    - Returns each section as an individual n8n item with heading and content in multiple formats  
  - Input: AI-generated ToC, Chunkr output chunks  
  - Output: Array of section items for downstream processing or export  
  - Edge Cases: Empty ToC, empty chunk data, missing matches  
  - Sticky Note: Notes this mode returns individual sections for granular processing

- **Return the whole document**  
  - Type: Code  
  - Role: Similar to above but compiles all sections into a single JSON array item  
  - Adds heading levels and section boundaries  
  - Prepares output suitable for generating full HTML or Markdown documents  
  - Output: Single JSON item with array of all sections and their content  
  - Edge Cases: Same as above  
  - Sticky Note: Notes this mode returns a single combined document

---

#### 1.5 Output Generation

**Overview:**  
Generates output files in HTML and Markdown formats from the mapped sections, prepares binary data for file download or further usage.

**Nodes Involved:**  
- Create HTML document (Code)  
- HTML  
- Create Markdown Document (Code)  
- Convert to File  
- Move Binary Data  

**Node Details:**

- **Create HTML document**  
  - Type: Code  
  - Role: Builds a full HTML document string from the array of processed sections  
  - Uses heading levels to create appropriate `<h1>`...`<h6>` tags  
  - Inserts section HTML content into body  
  - Adds basic CSS styling for readability  
  - Output: JSON with `fullHtmlContent` string  
  - Edge Cases: Missing or malformed input sections

- **HTML**  
  - Type: HTML Node  
  - Role: Transfers `fullHtmlContent` string into a binary property for file creation  
  - Input: JSON with HTML string  
  - Output: Binary data with `html` key

- **Create Markdown Document**  
  - Type: Code  
  - Role: Creates a full Markdown document string from sections using heading levels for `#` prefixes  
  - Appends section Markdown content  
  - Output: JSON with `fullMarkdownContent` string  
  - Edge Cases: Missing or malformed input sections

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts Markdown string to binary file with `.md` extension using filename from `Set File Name` node  
  - Input: JSON with Markdown string  
  - Output: Binary Markdown file ready for download or further use

- **Move Binary Data**  
  - Type: Move Binary Data  
  - Role: Converts JSON HTML string to binary file with `.html` extension using filename from `Set File Name` node  
  - Input: JSON with HTML string  
  - Output: Binary HTML file

---

#### 1.6 Error Handling and Status Management

**Overview:**  
Monitors Chunkr.ai task status and stops workflow with an error message if the task fails.

**Nodes Involved:**  
- Status is: (Switch)  
- Stop and Error  

**Node Details:**

- **Status is:**  
  - Routes based on Chunkr task status:  
    - `Succeeded`: proceed to ToC extraction  
    - `Processing`: continue waiting and polling  
    - `Failed`: route to error node

- **Stop and Error**  
  - Stops workflow execution with a clear error message "The chunkr Task failed!"  

---

### 3. Summary Table

| Node Name                          | Node Type                            | Functional Role                                   | Input Node(s)                      | Output Node(s)                               | Sticky Note                                                                                 |
|-----------------------------------|------------------------------------|-------------------------------------------------|-----------------------------------|----------------------------------------------|---------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger                     | Manual workflow start                            | -                                 | Download PDF from Google Drive               | Explains manual trigger usage and connection to Google Drive PDF                           |
| When Executed by Another Workflow | Execute Workflow Trigger           | External workflow trigger with PDF URL          | -                                 | Download PDF from URL                         | Explains usage for external triggers                                                     |
| Download PDF from Google Drive     | Google Drive                      | Downloads fixed PDF from Google Drive            | When clicking ‘Execute workflow’  | Merge                                        |                                                                                             |
| Download PDF from URL              | HTTP Request                      | Downloads PDF from URL                            | When Executed by Another Workflow | Merge                                        |                                                                                             |
| Merge                            | Merge                             | Merges PDF download streams                      | Download PDF from Google Drive, Download PDF from URL | Convert the PDF to base64                |                                                                                             |
| Convert the PDF to base64          | Extract From File                 | Converts binary PDF to base64 string             | Merge                            | Set File Name                                |                                                                                             |
| Set File Name                     | Set                              | Sets dynamic file name variables                  | Convert the PDF to base64          | POST Chunkr Task                             |                                                                                             |
| POST Chunkr Task                  | HTTP Request                     | Sends PDF to Chunkr.ai for parsing                | Set File Name                    | Wait Before Polling the Chunkr Result        | Reminder to insert Chunkr.ai API key                                                      |
| Wait Before Polling the Chunkr Result | Wait                          | Waits before polling Chunkr task status           | POST Chunkr Task                 | GET Chunkr Task                              |                                                                                             |
| GET Chunkr Task                  | HTTP Request                     | Polls Chunkr.ai for task status and parsed output | Wait Before Polling the Chunkr Result | Status is:                              |                                                                                             |
| Status is:                       | Switch                           | Routes workflow based on Chunkr task status       | GET Chunkr Task                 | Take beginning of Document to look for Table of contents, Wait Before Polling..., Stop and Error |                                                                                             |
| Stop and Error                   | Stop and Error                   | Stops workflow on Chunkr task failure              | Status is: (Failed)             | -                                            |                                                                                             |
| Take beginning of Document to look for Table of contents | Code                   | Extracts first 10 chunks’ text for AI ToC analysis | Status is: (Succeeded)          | Extract Sections headers as fallback          | Explains purpose of initial chunk extraction for AI                                      |
| Extract Sections headers as fallback | Code                          | Extracts unique section headers from Chunkr segments | Take beginning of Document to look for Table of contents | Table of Content Agent                   | Explains fallback extraction of section headers                                          |
| Google Gemini Chat Model          | AI Language Model (Google Gemini) | Processes text to generate ToC JSON                | Auto-fixing Output Parser        | Table of Content Agent                        |                                                                                             |
| Auto-fixing Output Parser         | Langchain Output Parser Autofixing | Fixes AI JSON output inconsistencies                | Google Gemini Chat Model1        | Structured Output Parser                      |                                                                                             |
| Structured Output Parser          | Langchain Structured Output Parser | Validates and parses AI output against JSON schema | Auto-fixing Output Parser        | Table of Content Agent                        |                                                                                             |
| Table of Content Agent            | AI Agent                         | Combines fallback and AI ToC sources to build nested ToC | Extract Sections headers as fallback, Google Gemini Chat Model | Return each section individually, Return the whole document | Explains combining two sources for ToC construction                                     |
| Return each section individually | Code                            | Maps ToC headings to Chunkr chunks, extracts sections individually | Table of Content Agent           | (end)                                        | Notes outputs individual sections with Markdown, HTML, or text                          |
| Return the whole document         | Code                            | Maps ToC headings, combines all sections into one output | Table of Content Agent           | Create HTML document, Create Markdown Document | Notes outputs whole document for download or agent use                                  |
| Create HTML document             | Code                            | Builds full HTML document from sections             | Return the whole document        | HTML                                          |                                                                                             |
| HTML                            | HTML                            | Converts HTML string to binary file                   | Create HTML document             | Move Binary Data                              |                                                                                             |
| Create Markdown Document          | Code                            | Builds full Markdown document from sections           | Return the whole document        | Convert to File                               |                                                                                             |
| Convert to File                 | Convert To File                  | Converts Markdown string to binary file               | Create Markdown Document         | (end)                                        |                                                                                             |
| Move Binary Data                | Move Binary Data                 | Converts JSON HTML string to binary file               | HTML                           | (end)                                        |                                                                                             |
| Sticky Notes (various)            | Sticky Note                     | Provides documentation and instructions               | -                               | -                                            | Multiple sticky notes explain usage, API key requirements, and processing rationale      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Create a Manual Trigger node named `When clicking ‘Execute workflow’`. Disable it by default.  
   - Create an Execute Workflow Trigger node named `When Executed by Another Workflow` with one input parameter: `URL` (string).

2. **Create PDF Download Nodes**  
   - Add a Google Drive node named `Download PDF from Google Drive`, set operation to `download`, and specify a fixed file ID. Connect it to the manual trigger. (Optional, disable by default.)  
   - Add an HTTP Request node named `Download PDF from URL`, set method to GET, URL to `={{$json.URL}}`. Connect it to the workflow trigger node.

3. **Merge Downloads**  
   - Add a Merge node named `Merge` to merge the outputs from Google Drive and HTTP Request nodes. Use default merge strategy.

4. **Convert PDF to Base64 and Set Filename**  
   - Add an Extract From File node named `Convert the PDF to base64` with operation `binaryToProperty`, source binary property `data`. Connect from `Merge`.  
   - Add a Set node named `Set File Name`, create variables:  
     - `fileName`: `={{ $('Merge').item.binary.data.fileName.replaceAll('.pdf','') }}`  
     - `fileNameSnake`: `={{ $('Merge').item.binary.data.fileName.replaceAll('.pdf','').toSnakeCase() }}`  
     - `createdAt`: `={{ $now }}`  
   Connect from `Convert the PDF to base64`.

5. **Configure Chunkr.ai API Calls**  
   - Add HTTP Request node `POST Chunkr Task`:  
     - Method: POST  
     - URL: `https://api.chunkr.ai/api/v1/task/parse`  
     - Body (JSON): includes base64 PDF `file` from previous node, `file_name` from `fileNameSnake`  
     - Headers: Authorization with your Chunkr.ai API key  
   Connect from `Set File Name`.  
   - Add Wait node `Wait Before Polling the Chunkr Result`, wait 10 seconds. Connect from `POST Chunkr Task`.  
   - Add HTTP Request node `GET Chunkr Task`:  
     - Method: GET  
     - URL: `https://api.chunkr.ai/api/v1/task/{{ $('POST Chunkr Task').item.json.task_id }}`  
     - Headers: Authorization with Chunkr.ai API key  
   Connect from `Wait Before Polling the Chunkr Result`.

6. **Status Check and Error Handling**  
   - Add Switch node `Status is:` that checks `{{$json.status}}` for values: `Succeeded`, `Processing`, `Failed`.  
   - Connect `Succeeded` output to next block, `Processing` output back to `Wait Before Polling the Chunkr Result`, `Failed` output to `Stop and Error` node with message "The chunkr Task failed!"

7. **Table of Contents Extraction**  
   - Add Code node `Take beginning of Document to look for Table of contents` to extract text of first 10 chunks from Chunkr output. Connect from `Status is:` `Succeeded` output.  
   - Add Code node `Extract Sections headers as fallback` to extract unique section headers from all chunks. Connect from previous node.

8. **AI ToC Processing**  
   - Add Google Gemini Chat Model node `Google Gemini Chat Model` configured with model `models/gemini-2.5-pro-preview-05-06`, with Google Palm API credentials. Input is initial chunks text and fallback headings.  
   - Add Auto-fixing Output Parser node `Auto-fixing Output Parser`. Connect from Google Gemini node.  
   - Add Structured Output Parser node `Structured Output Parser` with manual schema defining hierarchical ToC JSON. Connect from Auto-fixing Output Parser.  
   - Add AI Agent node `Table of Content Agent` that combines fallback headings and AI parsed ToC to produce final nested JSON ToC. Connect from Structured Output Parser and `Extract Sections headers as fallback`.

9. **Section Mapping and Output Preparation**  
   - Add Code node `Return each section individually` to perform two-pass mapping of ToC to chunks, extracting section content as individual items. Connect from `Table of Content Agent`.  
   - Add Code node `Return the whole document` to aggregate all sections into a single JSON item for full document output. Also connect from `Table of Content Agent`.

10. **Generate Output Documents**  
    - Add Code node `Create HTML document` to build full HTML from sections array. Connect from `Return the whole document`.  
    - Add HTML node `HTML` to convert HTML string to binary. Connect from `Create HTML document`.  
    - Add Move Binary Data node `Move Binary Data` to prepare HTML file binary with `.html` extension and filename from `Set File Name`. Connect from `HTML`.  
    - Add Code node `Create Markdown Document` to build full Markdown document from sections array. Connect from `Return the whole document`.  
    - Add Convert to File node `Convert to File` to convert Markdown string to binary `.md` file with filename from `Set File Name`. Connect from `Create Markdown Document`.

11. **Finalize Workflow**  
    - Connect outputs of file conversion nodes to your desired destinations (e.g., file saving, further processing).

12. **Credentials Setup**  
    - Setup Google Drive OAuth2 API credentials for Google Drive node if used.  
    - Setup Google Palm API credentials for Google Gemini nodes.  
    - Insert your Chunkr.ai API key in HTTP Request nodes for `POST Chunkr Task` and `GET Chunkr Task`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Welcome note about the Document Processing Workflow and Chunkr.ai API key requirement. Steps to create a Chunkr.ai account and retrieve API key.                                                                                  | https://chunkr.ai                                              |
| Manual trigger node explanation: how to use it to download a fixed PDF from Google Drive.                                                                                                                                         | Workflow internal documentation                               |
| External webhook trigger node explanation: requires URL input parameter pointing to PDF to process.                                                                                                                                | Workflow internal documentation                               |
| Reminder to insert Chunkr.ai API key in HTTP Request nodes for Chunkr task POST and GET.                                                                                                                                           | Workflow internal documentation                               |
| Explanation of fallback section header extraction from Chunkr output, used if no clear ToC is detected in document start.                                                                                                          | Workflow internal documentation                               |
| Explanation of initial document chunk extraction for AI Table of Contents detection.                                                                                                                                               | Workflow internal documentation                               |
| AI Agent node combines fallback headers and beginning document text to generate nested ToC JSON.                                                                                                                                  | Workflow internal documentation                               |
| Two modes of output: section-wise individual outputs or whole document combined output, depending on downstream usage.                                                                                                            | Workflow internal documentation                               |
| The workflow produces HTML and Markdown versions of the processed PDF document for download or further integration.                                                                                                               | Workflow internal documentation                               |
| The workflow includes detailed error handling for Chunkr task failure, halting execution with clear error messages.                                                                                                               | Workflow internal documentation                               |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---