Generate C API HTML docs from Google Drive headers with GPT-4o and Gmail

https://n8nworkflows.xyz/workflows/generate-c-api-html-docs-from-google-drive-headers-with-gpt-4o-and-gmail-14093


# Generate C API HTML docs from Google Drive headers with GPT-4o and Gmail

## 1. Workflow Overview

This workflow generates styled HTML API documentation from C header files (`.h`) stored in Google Drive, using GPT-4o to extract structured technical content and Gmail to notify completion.

Typical use case:
- A team stores embedded C header files in a Google Drive folder.
- The workflow reads all files in that folder.
- It filters only header files.
- It processes each file one by one through OpenAI.
- It converts the extracted structure into a branded HTML document.
- It uploads each generated document to another Google Drive folder.
- After the loop completes, it sends a completion email.

### 1.1 Input Reception and File Discovery
The workflow starts manually, queries a configured Google Drive folder, and retrieves all files available in that folder.

### 1.2 Header File Filtering
A Code node filters the full file list to retain only files whose names end with `.h`. If none are found, the workflow stops with an error.

### 1.3 Per-File Iteration
A Split in Batches node turns the filtered list into a loop so each header file is processed individually. This same node also acts as the loop exit point that triggers the completion email branch once all items are consumed.

### 1.4 File Download and Text Extraction
Each selected Google Drive file is downloaded as binary data, then converted into raw text content so it can be passed to GPT-4o.

### 1.5 AI-Based Structure Extraction
The raw header content is sent to GPT-4o with a strict JSON schema request. The model is expected to return structured documentation fields such as overview, functions, enums, data types, constants, and notes.

### 1.6 Response Parsing and Normalization
The OpenAI response is cleaned, markdown fences are removed if present, and the content is parsed as JSON. If parsing fails, a fallback object is created to preserve the raw response for debugging.

### 1.7 HTML Document Generation
A long Code node builds a complete styled HTML page with navigation, metadata, sections, cards, tables, and notes. The result is emitted as binary HTML content.

### 1.8 Output Upload and Loop Continuation
The generated HTML file is uploaded to Google Drive, then the loop returns to the Split in Batches node to process the next file.

### 1.9 Completion Notification
When the batch loop finishes, the Split in Batches node exits through its completion branch, where a Code node builds an email subject and body, and a Gmail node sends the notification.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and File Discovery

**Overview:**  
This block starts the workflow manually and retrieves all files from a specified Google Drive folder. It is the entry point and determines the upstream file set for the entire run.

**Nodes Involved:**  
- When clicking ‘Execute workflow’
- Get all files from folder

### Node Details

#### When clicking ‘Execute workflow’
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry point for interactive execution.
- **Configuration choices:** No custom parameters. It simply starts the workflow when run from the editor.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none  
  - Output: `Get all files from folder`
- **Version-specific requirements:** Type version 1; standard manual trigger behavior.
- **Edge cases or potential failure types:** None at runtime beyond normal workflow execution issues.
- **Sub-workflow reference:** None.

#### Get all files from folder
- **Type and technical role:** `n8n-nodes-base.googleDrive`; searches Google Drive for files inside a specific folder.
- **Configuration choices:**  
  - Resource: file/folder search  
  - Search method: query  
  - Return all: enabled  
  - Query string: `'YOUR_FOLDER_ID' in parents and trashed=false`
- **Key expressions or variables used:** Static query string containing the folder ID placeholder.
- **Input and output connections:**  
  - Input: `When clicking ‘Execute workflow’`  
  - Output: `Filter .h files only`
- **Version-specific requirements:** Type version 3; requires valid Google Drive credentials.
- **Edge cases or potential failure types:**  
  - Invalid or missing Google Drive OAuth2 credentials  
  - Folder ID not replaced from placeholder  
  - Access denied to the folder  
  - Empty folder result  
  - Google API quota or transient connectivity failures
- **Sub-workflow reference:** None.

---

## 2.2 Header File Filtering

**Overview:**  
This block reduces the Google Drive file list to only C header files. It also enforces that at least one `.h` file exists.

**Nodes Involved:**  
- Filter .h files only

### Node Details

#### Filter .h files only
- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript filtering logic.
- **Configuration choices:**  
  - Reads all incoming items with `$input.all()`
  - Lowercases file names
  - Keeps only names ending with `.h`
  - Emits simplified JSON objects containing `id` and `fileName`
  - Throws an error if no header files are found
- **Key expressions or variables used:**  
  - `item.json.name`
  - Output shape: `{ id, fileName }`
- **Input and output connections:**  
  - Input: `Get all files from folder`  
  - Output: `Process one file at a time`
- **Version-specific requirements:** Type version 2; JavaScript Code node.
- **Edge cases or potential failure types:**  
  - Missing `name` or `id` fields from upstream data  
  - No `.h` files present, causing explicit thrown error  
  - Files with uppercase extensions are handled because names are lowercased first
- **Sub-workflow reference:** None.

---

## 2.3 Per-File Iteration

**Overview:**  
This block controls sequential processing of files and also acts as the point where the workflow exits to send a completion email after all items are processed.

**Nodes Involved:**  
- Process one file at a time

### Node Details

#### Process one file at a time
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; iterates over incoming items one by one.
- **Configuration choices:**  
  - Default batching behavior
  - `executeOnce: false`
  - No custom batch size specified, so standard one-item iteration applies in this setup
- **Key expressions or variables used:**  
  - Downstream nodes refer back to `$('Process one file at a time').item.json.fileName`
- **Input and output connections:**  
  - Input: `Filter .h files only`, and loop-back from `Save PDF to Google Drive`
  - Output 1: `Build email body` when loop completes
  - Output 2/main processing path: `Download file`
- **Version-specific requirements:** Type version 3.
- **Edge cases or potential failure types:**  
  - Incorrect loop wiring can cause premature completion or infinite loops  
  - If upload fails, loop does not continue for later files  
  - Completion branch depends on Split in Batches semantics in the installed n8n version
- **Sub-workflow reference:** None.

---

## 2.4 File Download and Text Extraction

**Overview:**  
This block downloads the current header file from Google Drive and converts its binary content into plain text.

**Nodes Involved:**  
- Download file
- Extract from File2

### Node Details

#### Download file
- **Type and technical role:** `n8n-nodes-base.googleDrive`; downloads the selected file as binary data.
- **Configuration choices:**  
  - Operation: download
  - File ID comes from the current batch item: `={{ $json.id }}`
- **Key expressions or variables used:**  
  - `{{ $json.id }}`
- **Input and output connections:**  
  - Input: `Process one file at a time`
  - Output: `Extract from File2`
- **Version-specific requirements:** Type version 3; requires Google Drive credentials.
- **Edge cases or potential failure types:**  
  - Invalid file ID  
  - Permission issues  
  - Download failures or rate limits  
  - Corrupt binary payload
- **Sub-workflow reference:** None.

#### Extract from File2
- **Type and technical role:** `n8n-nodes-base.extractFromFile`; extracts text from binary file content.
- **Configuration choices:**  
  - Operation: text extraction
  - No extra options configured
- **Key expressions or variables used:**  
  - Output text later referenced as `$('Extract from File2').item.json.data`
- **Input and output connections:**  
  - Input: `Download file`
  - Output: `Extract structured docs (GPT-4o)`
- **Version-specific requirements:** Type version 1.1.
- **Edge cases or potential failure types:**  
  - Unsupported encoding or malformed file content  
  - Empty file  
  - Binary property mismatch if upstream node output structure changes
- **Sub-workflow reference:** None.

---

## 2.5 AI-Based Structure Extraction

**Overview:**  
This block sends the raw header content to GPT-4o and asks for a strict JSON response matching a predefined documentation schema.

**Nodes Involved:**  
- Extract structured docs (GPT-4o)

### Node Details

#### Extract structured docs (GPT-4o)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; OpenAI chat/completion style node used for structured content extraction.
- **Configuration choices:**  
  - Model: `gpt-4o`
  - System message defines the JSON schema and forbids markdown/explanations
  - User content includes file name and extracted file text
- **Key expressions or variables used:**  
  - `{{ $('Process one file at a time').item.json.fileName }}`
  - `{{ $('Extract from File2').item.json.data }}`
- **Input and output connections:**  
  - Input: `Extract from File2`
  - Output: `Parse JSON response`
- **Version-specific requirements:**  
  - Type version 2.1  
  - Requires OpenAI credentials with access to `gpt-4o`
- **Edge cases or potential failure types:**  
  - Missing or invalid OpenAI credentials  
  - Model access unavailable  
  - Large header file causing token limit or truncation issues  
  - Model returning non-JSON despite prompt  
  - Timeout or API rate limiting
- **Sub-workflow reference:** None.

---

## 2.6 Response Parsing and Normalization

**Overview:**  
This block normalizes the GPT response shape and attempts to parse it into a clean JSON document object. It includes a fallback mode for debugging when parsing fails.

**Nodes Involved:**  
- Parse JSON response

### Node Details

#### Parse JSON response
- **Type and technical role:** `n8n-nodes-base.code`; defensive parser for multiple possible OpenAI response formats.
- **Configuration choices:**  
  - Reads first input item only
  - Checks several possible response paths:
    - `choices[0].message.content`
    - `output[0].content[0].text`
    - `message.content`
    - `text`
    - or serializes the JSON if `overview` already exists
  - Removes markdown code fences
  - Parses JSON
  - On failure, creates a fallback object with raw text embedded in `overview`
  - Reattaches the source file name
- **Key expressions or variables used:**  
  - `$input.first()`
  - `$('Process one file at a time')?.item?.json?.fileName`
  - fallback reference `$('Process one file at a time1')?.item?.json?.fileName`
- **Input and output connections:**  
  - Input: `Extract structured docs (GPT-4o)`
  - Output: `Generate HTML`
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential failure types:**  
  - Unexpected OpenAI payload shape  
  - Invalid JSON generated by the model  
  - Empty model response  
  - The reference to `Process one file at a time1` appears to be legacy and likely unused
- **Sub-workflow reference:** None.

---

## 2.7 HTML Document Generation

**Overview:**  
This block converts the structured JSON into a complete branded HTML document and emits it as binary content for upload.

**Nodes Involved:**  
- Generate HTML

### Node Details

#### Generate HTML
- **Type and technical role:** `n8n-nodes-base.code`; templating and binary file generation in JavaScript.
- **Configuration choices:**  
  - Reads the parsed documentation object
  - Derives:
    - `fileName`
    - `baseName`
    - `htmlName = DOC_<basename>.html`
    - current date
  - Escapes HTML content safely
  - Builds sections for:
    - Overview
    - Functions
    - Enumerators
    - Data Types & Structures
    - Constants
    - Notes
  - Embeds extensive CSS styling and layout
  - Converts final HTML string to base64
  - Outputs binary file metadata:
    - MIME type: `text/html`
    - file extension: `html`
- **Key expressions or variables used:**  
  - `doc.fileName`
  - `doc.functions`
  - `doc.enumerators`
  - `doc.data_types`
  - `doc.constants`
  - `doc.notes`
- **Input and output connections:**  
  - Input: `Parse JSON response`
  - Output: `Save PDF to Google Drive`
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential failure types:**  
  - Very large HTML output may increase memory usage  
  - Missing arrays are handled with defaults  
  - Escaping function reduces HTML injection risk from model output  
  - Branding text is hard-coded to `XYZ Technologies`
- **Sub-workflow reference:** None.

---

## 2.8 Output Upload and Loop Continuation

**Overview:**  
This block uploads the generated binary file to Google Drive and returns control to the batch loop.

**Nodes Involved:**  
- Save PDF to Google Drive

### Node Details

#### Save PDF to Google Drive
- **Type and technical role:** `n8n-nodes-base.googleDrive`; uploads the generated binary file to a target folder.
- **Configuration choices:**  
  - Drive: `MyDrive`
  - Folder ID: `YOUR_OUTPUT_FOLDER_ID`
  - File name parameter: `={{ $json.pdfName }}`
- **Key expressions or variables used:**  
  - `{{ $json.pdfName }}`
- **Input and output connections:**  
  - Input: `Generate HTML`
  - Output: `Process one file at a time` (loop-back)
- **Version-specific requirements:** Type version 3; requires Google Drive credentials.
- **Edge cases or potential failure types:**  
  - The node name and configured field indicate PDF, but upstream generates HTML  
  - `pdfName` is not produced by `Generate HTML`; it produces `htmlName` instead  
  - This likely causes missing filename issues or incorrect uploads unless corrected  
  - Folder ID must be replaced
- **Sub-workflow reference:** None.

**Important implementation inconsistency:**  
This node is configured as if it uploads a PDF, but the workflow actually generates HTML. The likely intended name expression should be `={{ $json.htmlName }}` or the node should rely on the binary metadata file name. This is the single biggest configuration issue in the workflow.

---

## 2.9 Completion Notification

**Overview:**  
After all files are processed, this block prepares and sends a completion email via Gmail.

**Nodes Involved:**  
- Build email body
- Send completion email

### Node Details

#### Build email body
- **Type and technical role:** `n8n-nodes-base.code`; constructs email subject and message text.
- **Configuration choices:**  
  - Iterates all incoming items
  - Uses:
    - `pdfName`
    - fallback `name`
    - fallback `'DOC.pdf'`
  - Builds subject: `API Documentation PDF Ready — XYZ Technologies`
  - Builds plain-text message including timestamp and generated sections
- **Key expressions or variables used:**  
  - `item.json.pdfName || item.json.name || 'DOC.pdf'`
- **Input and output connections:**  
  - Input: `Process one file at a time` completion branch
  - Output: `Send completion email`
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential failure types:**  
  - The email wording says PDF, but the actual artifact is HTML  
  - On loop completion, input item content may not include the last processed file name as expected depending on Split in Batches behavior  
  - If no useful file name is available, the fallback `DOC.pdf` is used
- **Sub-workflow reference:** None.

#### Send completion email
- **Type and technical role:** `n8n-nodes-base.gmail`; sends the notification email.
- **Configuration choices:**  
  - Recipient: `your@email.com`
  - Subject from expression: `={{ $json.subject }}`
  - Message from expression: `={{ $json.message }}`
- **Key expressions or variables used:**  
  - `{{ $json.subject }}`
  - `{{ $json.message }}`
- **Input and output connections:**  
  - Input: `Build email body`
  - Output: none
- **Version-specific requirements:** Type version 2.2; requires Gmail credentials.
- **Edge cases or potential failure types:**  
  - Missing or expired Gmail authentication  
  - Recipient not updated from placeholder  
  - Subject/message missing if upstream item shape is unexpected
- **Sub-workflow reference:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual workflow start |  | Get all files from folder | # 📄 API Documentation Generator Automatically generates **beautiful HTML documentation** from C header (`.h`) files stored in Google Drive. |
| Get all files from folder | Google Drive | Search all files in source folder | When clicking ‘Execute workflow’ | Filter .h files only | ### 📁 Step 1 — Load Files Trigger manually, then fetch all files from the configured Google Drive folder and filter down to `.h` files only. |
| Get all files from folder | Google Drive | Search all files in source folder | When clicking ‘Execute workflow’ | Filter .h files only | ⚙️ **To configure:** Open **Get all files from folder** and update the `queryString` with your own Google Drive folder ID. |
| Filter .h files only | Code | Keep only `.h` files and emit simplified metadata | Get all files from folder | Process one file at a time | ### 📁 Step 1 — Load Files Trigger manually, then fetch all files from the configured Google Drive folder and filter down to `.h` files only. |
| Filter .h files only | Code | Keep only `.h` files and emit simplified metadata | Get all files from folder | Process one file at a time | ⚙️ **To configure:** Open **Get all files from folder** and update the `queryString` with your own Google Drive folder ID. |
| Process one file at a time | Split In Batches | Loop through header files one by one and provide completion branch | Filter .h files only; Save PDF to Google Drive | Build email body; Download file | ### 🔁 Step 2 — Loop Process one `.h` file at a time using **Split in Batches**. |
| Process one file at a time | Split In Batches | Loop through header files one by one and provide completion branch | Filter .h files only; Save PDF to Google Drive | Build email body; Download file | The loop returns here after each file is saved to Drive, until all files are processed. When done, it branches to the email notification. |
| Download file | Google Drive | Download current header file as binary | Process one file at a time | Extract from File2 | ### 📥 Step 3 — Download & Read Downloads the binary file from Google Drive and extracts its raw text content, ready to be sent to GPT-4o. |
| Extract from File2 | Extract From File | Extract plain text from downloaded header file | Download file | Extract structured docs (GPT-4o) | ### 📥 Step 3 — Download & Read Downloads the binary file from Google Drive and extracts its raw text content, ready to be sent to GPT-4o. |
| Extract structured docs (GPT-4o) | OpenAI Chat Model | Send header content to GPT-4o for structured JSON extraction | Extract from File2 | Parse JSON response | ### 🤖 Step 4 — AI Extraction Sends the raw `.h` file content to **GPT-4o** with a structured prompt. |
| Extract structured docs (GPT-4o) | OpenAI Chat Model | Send header content to GPT-4o for structured JSON extraction | Extract from File2 | Parse JSON response | GPT returns a strict JSON object with: `overview` · `functions` · `enumerators` · `data_types` · `constants` · `notes` |
| Extract structured docs (GPT-4o) | OpenAI Chat Model | Send header content to GPT-4o for structured JSON extraction | Extract from File2 | Parse JSON response | ⚠️ Requires an **OpenAI credential** with GPT-4o access. |
| Parse JSON response | Code | Clean and parse model output into normalized JSON | Extract structured docs (GPT-4o) | Generate HTML | ### 🔧 Step 5 — Parse Response Strips any markdown fences from the GPT response and parses it as JSON. |
| Parse JSON response | Code | Clean and parse model output into normalized JSON | Extract structured docs (GPT-4o) | Generate HTML | 💡 If parsing fails, the raw GPT text is placed in `overview` so you can debug what GPT returned. |
| Generate HTML | Code | Build styled HTML page and output binary file | Parse JSON response | Save PDF to Google Drive | ### 🎨 Step 6 — Generate HTML Builds a fully styled HTML documentation file with: |
| Generate HTML | Code | Build styled HTML page and output binary file | Parse JSON response | Save PDF to Google Drive | - Sidebar navigation |
| Generate HTML | Code | Build styled HTML page and output binary file | Parse JSON response | Save PDF to Google Drive | - Hero header with metadata |
| Generate HTML | Code | Build styled HTML page and output binary file | Parse JSON response | Save PDF to Google Drive | - Function cards with signature, params table & examples |
| Generate HTML | Code | Build styled HTML page and output binary file | Parse JSON response | Save PDF to Google Drive | - Data types, enums, constants sections |
| Generate HTML | Code | Build styled HTML page and output binary file | Parse JSON response | Save PDF to Google Drive | Outputs as a **binary `.html` file** ready for Google Drive upload. |
| Save PDF to Google Drive | Google Drive | Upload generated document to output folder and continue loop | Generate HTML | Process one file at a time | ### 💾 Step 7 — Save to Drive Uploads the generated `DOC_filename.html` file to the configured output folder in Google Drive. |
| Save PDF to Google Drive | Google Drive | Upload generated document to output folder and continue loop | Generate HTML | Process one file at a time | ⚙️ **To configure:** Update the `folderId` in this node with your output folder ID. |
| Build email body | Code | Compose completion email text | Process one file at a time | Send completion email | ### ✉️ Completion Email Once all files are processed, the **Split in Batches** node exits the loop and triggers this branch. |
| Build email body | Code | Compose completion email text | Process one file at a time | Send completion email | Builds a summary email and sends it via **Gmail** to notify that all docs have been generated. |
| Build email body | Code | Compose completion email text | Process one file at a time | Send completion email | ⚙️ Update the recipient email in **Send completion email**. |
| Send completion email | Gmail | Send completion notification via Gmail | Build email body |  | ### ✉️ Completion Email Once all files are processed, the **Split in Batches** node exits the loop and triggers this branch. |
| Send completion email | Gmail | Send completion notification via Gmail | Build email body |  | Builds a summary email and sends it via **Gmail** to notify that all docs have been generated. |
| Send completion email | Gmail | Send completion notification via Gmail | Build email body |  | ⚙️ Update the recipient email in **Send completion email**. |
| Sticky Note — Step 1 | Sticky Note | Visual documentation |  |  | ### 📁 Step 1 — Load Files Trigger manually, then fetch all files from the configured Google Drive folder and filter down to `.h` files only. |
| Sticky Note — Step 8 | Sticky Note | Visual documentation |  |  | ### 🔁 Step 2 — Loop Process one `.h` file at a time using **Split in Batches**. The loop returns here after each file is saved to Drive, until all files are processed. When done, it branches to the email notification. |
| Sticky Note — Step 9 | Sticky Note | Visual documentation |  |  | ### 📥 Step 3 — Download & Read Downloads the binary file from Google Drive and extracts its raw text content, ready to be sent to GPT-4o. |
| Sticky Note — Step 10 | Sticky Note | Visual documentation |  |  | ### 🤖 Step 4 — AI Extraction Sends the raw `.h` file content to **GPT-4o** with a structured prompt. GPT returns a strict JSON object with: `overview` · `functions` · `enumerators` · `data_types` · `constants` · `notes` ⚠️ Requires an **OpenAI credential** with GPT-4o access. |
| Sticky Note — Step 11 | Sticky Note | Visual documentation |  |  | ### 🔧 Step 5 — Parse Response Strips any markdown fences from the GPT response and parses it as JSON. 💡 If parsing fails, the raw GPT text is placed in `overview` so you can debug what GPT returned. |
| Sticky Note — Step 12 | Sticky Note | Visual documentation |  |  | ### 🎨 Step 6 — Generate HTML Builds a fully styled HTML documentation file with sidebar navigation, hero header with metadata, function cards with signature, params table & examples, data types, enums, constants sections. Outputs as a **binary `.html` file** ready for Google Drive upload. |
| Sticky Note — Step 13 | Sticky Note | Visual documentation |  |  | ### 💾 Step 7 — Save to Drive Uploads the generated `DOC_filename.html` file to the configured output folder in Google Drive. ⚙️ **To configure:** Update the `folderId` in this node with your output folder ID. |
| Sticky Note — Email1 | Sticky Note | Visual documentation |  |  | ### ✉️ Completion Email Once all files are processed, the **Split in Batches** node exits the loop and triggers this branch. Builds a summary email and sends it via **Gmail** to notify that all docs have been generated. ⚙️ Update the recipient email in **Send completion email**. |
| Sticky Note — Overview1 | Sticky Note | Visual documentation |  |  | # 📄 API Documentation Generator Automatically generates **beautiful HTML documentation** from C header (`.h`) files stored in Google Drive. 1. Reads all files from a specified Google Drive folder 2. Filters to `.h` header files only 3. Processes each file one at a time through GPT-4o 4. GPT extracts functions, types, enums, constants & notes as structured JSON 5. Generates a professional HTML doc and saves it back to Google Drive 6. Sends a completion email when all files are processed **Requirements:** Google Drive OAuth2 credentials, OpenAI API credentials, Gmail credentials, and updated Google Drive folder ID in the **Get all files** node. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `API Documentation — Generator`.

2. **Add a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Leave default settings.
   - This is the workflow entry point.

3. **Add a Google Drive node to list source files**
   - Node type: **Google Drive**
   - Name: `Get all files from folder`
   - Credential: Google Drive OAuth2
   - Configure:
     - Resource: file/folder search
     - Search method: query
     - Return all: enabled
     - Query string: `'YOUR_FOLDER_ID' in parents and trashed=false`
   - Replace `YOUR_FOLDER_ID` with the source folder ID.
   - Connect: `Manual Trigger -> Get all files from folder`

4. **Add a Code node to keep only header files**
   - Node type: **Code**
   - Name: `Filter .h files only`
   - Paste this logic conceptually:
     - Read all incoming items
     - Keep only files whose name ends with `.h`
     - Return `id` and `fileName`
     - Throw an error if no `.h` file exists
   - Connect: `Get all files from folder -> Filter .h files only`

5. **Add a Split in Batches node**
   - Node type: **Split in Batches**
   - Name: `Process one file at a time`
   - Leave default batch behavior so one item is processed per iteration.
   - Connect: `Filter .h files only -> Process one file at a time`

6. **Add a Google Drive download node**
   - Node type: **Google Drive**
   - Name: `Download file`
   - Credential: same Google Drive credential
   - Operation: **Download**
   - File ID expression: `{{ $json.id }}`
   - Connect from the processing output of the batch node:
     - `Process one file at a time -> Download file`

7. **Add an Extract From File node**
   - Node type: **Extract From File**
   - Name: `Extract from File2`
   - Operation: **Extract Text**
   - Leave default options unless your files use unusual encoding.
   - Connect: `Download file -> Extract from File2`

8. **Add the OpenAI node**
   - Node type: **OpenAI** via the LangChain/OpenAI integration
   - Name: `Extract structured docs (GPT-4o)`
   - Credential: OpenAI API credential
   - Model: `gpt-4o`
   - Add a system message instructing the model to return strict JSON with this structure:
     - `overview`
     - `functions`
     - `enumerators`
     - `data_types`
     - `constants`
     - `notes`
   - Add a user message containing:
     - Current file name from `Process one file at a time`
     - Extracted text from `Extract from File2`
   - Use expressions equivalent to:
     - `{{ $('Process one file at a time').item.json.fileName }}`
     - `{{ $('Extract from File2').item.json.data }}`
   - Connect: `Extract from File2 -> Extract structured docs (GPT-4o)`

9. **Add a Code node to parse the model response**
   - Node type: **Code**
   - Name: `Parse JSON response`
   - Implement logic to:
     - Read the first input item
     - Try multiple possible response paths from the OpenAI node
     - Remove surrounding code fences if present
     - Parse JSON
     - On failure, return a fallback object with:
       - `overview` containing the raw response snippet
       - empty arrays for all list sections
       - empty notes
     - Reattach the current file name
   - Connect: `Extract structured docs (GPT-4o) -> Parse JSON response`

10. **Add a Code node to generate HTML**
    - Node type: **Code**
    - Name: `Generate HTML`
    - Implement logic to:
      - Read the structured doc JSON
      - Derive:
        - original `fileName`
        - `baseName`
        - output file name `DOC_<baseName>.html`
      - Build a full HTML page with:
        - sidebar navigation
        - overview
        - functions
        - enumerators
        - data types
        - constants
        - notes
      - Escape all inserted content
      - Convert the HTML string to base64
      - Return binary data with:
        - MIME type `text/html`
        - file extension `html`
        - file name equal to the generated HTML name
      - Also expose `htmlName` in JSON
    - Connect: `Parse JSON response -> Generate HTML`

11. **Add a Google Drive upload node**
    - Node type: **Google Drive**
    - Name: ideally rename to `Save HTML to Google Drive`
    - Credential: Google Drive OAuth2
    - Configure upload to the output folder:
      - Drive: `MyDrive` or your target drive
      - Folder ID: `YOUR_OUTPUT_FOLDER_ID`
    - Important:
      - Set the uploaded file name from `{{ $json.htmlName }}`
      - Do **not** leave it as `{{ $json.pdfName }}` unless you also change upstream output
    - Ensure the node uploads the binary property produced by the HTML node.
    - Connect: `Generate HTML -> Save HTML to Google Drive`

12. **Close the loop**
    - Connect the upload node back to the Split in Batches node:
      - `Save HTML to Google Drive -> Process one file at a time`
    - This makes the workflow continue with the next header file.

13. **Add a Code node for the completion email**
    - Node type: **Code**
    - Name: `Build email body`
    - Connect it from the completion output of the Split in Batches node.
    - Build:
      - `subject`
      - `message`
    - Recommended correction:
      - Change all PDF wording to HTML wording
      - Use `htmlName` rather than `pdfName`
    - Suggested content:
      - mention that HTML API documentation has been generated
      - include timestamp
      - include target Google Drive folder

14. **Add a Gmail node**
    - Node type: **Gmail**
    - Name: `Send completion email`
    - Credential: Gmail OAuth2
    - Configure:
      - Recipient email address
      - Subject: `{{ $json.subject }}`
      - Message: `{{ $json.message }}`
    - Connect: `Build email body -> Send completion email`

15. **Set credentials**
    - **Google Drive OAuth2**
      - Needed for both listing/downloading and uploading.
    - **OpenAI**
      - Must have access to `gpt-4o`.
    - **Gmail OAuth2**
      - Needed for sending the completion email.

16. **Add optional sticky notes**
    - Add notes around each stage if you want the same visual guidance:
      - Load files
      - Loop
      - Download & read
      - AI extraction
      - Parse response
      - Generate HTML
      - Save to Drive
      - Completion email
      - Overall workflow description

17. **Test with one small header file first**
    - Verify:
      - file listing works
      - `.h` filtering is correct
      - text extraction returns readable source
      - GPT returns valid JSON
      - HTML is generated with expected sections
      - Google Drive upload uses `.html`
      - completion email content matches actual output format

### Recommended corrections while rebuilding
- Rename `Save PDF to Google Drive` to `Save HTML to Google Drive`.
- Change upload name expression from `{{ $json.pdfName }}` to `{{ $json.htmlName }}`.
- Change email subject/body from “PDF” to “HTML”.
- Consider tracking all processed file names if you want a true summary instead of a generic completion message.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Branding is hard-coded throughout the AI prompt, HTML template, and email text as **XYZ Technologies**. | Update if adapting for another company. |
| The workflow title in the JSON is `API Documentation — Generator`. | Internal workflow name. |
| The workflow description field is empty. | No external description provided. |
| The HTML template uses Google Fonts (`IBM Plex Mono` and `IBM Plex Sans`). | https://fonts.googleapis.com |
| The workflow is configured for manual execution, not scheduled or event-driven operation. | Trigger design choice. |
| Binary mode is set to `separate` in workflow settings. | Relevant for file handling/debugging. |
| There is no sub-workflow node in this workflow. | All logic is contained in a single workflow. |
| Primary configuration placeholders that must be replaced: source Google Drive folder ID, output Google Drive folder ID, recipient email address. | Required before production use. |
| Main functional inconsistency: several nodes still refer to PDF even though the workflow now generates HTML. | Must be corrected for reliable operation. |