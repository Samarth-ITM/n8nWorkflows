Generate Professional Documents with Claude AI Skills and Upload to Google Drive

https://n8nworkflows.xyz/workflows/generate-professional-documents-with-claude-ai-skills-and-upload-to-google-drive-11049


# Generate Professional Documents with Claude AI Skills and Upload to Google Drive

---

## 1. Workflow Overview

This workflow automates the generation of professional documents using Anthropic’s Claude AI Skills API (v2), supporting multiple file formats (PPTX, PDF, DOCX, XLSX). Users submit a prompt and select a desired file type via a form, which triggers the workflow to:

- Process the prompt with a corresponding Claude AI Skill specialized for the selected document type.
- Extract the generated file’s ID from the API response.
- Download the generated file content.
- Upload the file to a specified folder in Google Drive.

### Logical Blocks

- **1.1 Input Reception:** Form trigger captures user prompt and file type.
- **1.2 File Type Routing:** Switch node routes workflow based on selected file type.
- **1.3 Document Generation:** HTTP Request nodes invoke Claude AI Skills API to generate document files for each format.
- **1.4 File ID Extraction:** Code nodes extract the file_id from nested API responses.
- **1.5 File Download:** HTTP Request nodes download files using the extracted file_id.
- **1.6 Upload to Google Drive:** Google Drive nodes upload the downloaded files into a configured folder.
- **1.7 Setup & Instructions:** Sticky notes provide setup instructions and workflow guidance.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

- **Overview:** Waits for user input with a text prompt and desired file type selection.
- **Nodes Involved:** `On form submission`
- **Node Details:**

  - **On form submission**
    - *Type:* Form Trigger
    - *Role:* Entry point capturing user input via a web form.
    - *Configuration:*  
      - Form titled "Use Anthropic Skills"  
      - Two fields:  
        - `Prompt` (required text input)  
        - `File Type` (required dropdown with options: PPTX, PDF, DOCX, XLSX)
    - *Input connections:* None (trigger node)  
    - *Output connections:* Connects to `Switch` node.
    - *Potential failure:* Network/webhook failure, missing required fields.
    - *Notes:* Webhook ID assigned for external triggering.

---

### 2.2 File Type Routing

- **Overview:** Routes workflow execution based on user-selected file type.
- **Nodes Involved:** `Switch`
- **Node Details:**

  - **Switch**
    - *Type:* Switch
    - *Role:* Condition-based branch routing.
    - *Configuration:*  
      - Checks the exact value of the `File Type` field from the form submission.  
      - Routes to one of four outputs: PPTX, PDF, DOCX, XLSX.
    - *Input connections:* From `On form submission`
    - *Output connections:*  
      - PPTX → `Create PPTX`  
      - PDF → `Create PDF`  
      - DOCX → `Create DOCX`  
      - XLSX → `Create XLSX`
    - *Potential failure:* Incorrect or missing file type input; case sensitivity enforced.

---

### 2.3 Document Generation

- **Overview:** Calls Anthropic Claude AI Skills API to generate documents in the selected format.
- **Nodes Involved:** `Create PPTX`, `Create PDF`, `Create DOCX`, `Create XLSX`
- **Node Details:**

  - Common Configuration for all four:
    - *Type:* HTTP Request
    - *Role:* POST request to Anthropic `/v1/messages` endpoint to invoke Claude Skills.
    - *Authentication:* HTTP Header Auth with Anthropic API key.
    - *Headers:*  
      - `anthropic-version`: 2023-06-01  
      - `anthropic-beta`: includes `code-execution-2025-08-25` and `skills-2025-10-02`
    - *Request Body:* JSON containing:  
      - `model`: "claude-haiku-4-5-20251001"  
      - `max_tokens`: varies (mostly 4096, PPTX uses 50000)  
      - `system`: Detailed professional instructions tailored per document type (PPTX, PDF, DOCX, XLSX) specifying formatting standards, layout, and design principles.  
      - `container.skills`: Specifies skill type and latest version (e.g., `pptx`, `pdf`, `docx`, `xlsx`)  
      - `messages.user.content`: User prompt from form submission  
      - `tools`: Includes `code_execution` skill for enhanced capabilities.
    - *Input connections:* From `Switch` node branch.
    - *Output connections:* Connects to respective file ID extraction node.
    - *Edge cases/failures:* API auth errors, rate limits, malformed prompt, timeout or exceeded max tokens.

---

### 2.4 File ID Extraction

- **Overview:** Extracts `file_id` from sometimes deeply nested API response objects to enable file download.
- **Nodes Involved:** `Extract PPTX file_id`, `Extract PDF file_id`, `Extract DOCX file_id`, `Extract XLSX file_id`
- **Node Details:**

  - *Type:* Code (JavaScript)
  - *Role:* Recursively searches response JSON for a `file_id` or equivalent identifier.
  - *Logic:*  
    - Checks common known paths (`file_id`, `fileId`, `id` starting with "file-")  
    - Recurses through nested objects and arrays  
    - Also checks binary data if present  
    - Returns the first found `file_id`.
  - *Input connections:* From respective `Create` nodes.
  - *Output connections:* To respective download nodes.
  - *Edge cases:* Missing or malformed response, no `file_id` found, circular references handled.

---

### 2.5 File Download

- **Overview:** Downloads the generated document file content from Anthropic API using the extracted `file_id`.
- **Nodes Involved:** `Download PPTX`, `Download PDF`, `Download DOCX`, `Download XLSX`
- **Node Details:**

  - *Type:* HTTP Request
  - *Role:* GET request to `/v1/files/{file_id}/content` endpoint to retrieve file content as a binary.
  - *Authentication:* HTTP Header Auth with Anthropic API key.
  - *Headers:*  
    - `x-api-key` with API key (note: placeholder key shown in JSON, must be replaced)  
    - `anthropic-version`: 2023-06-01  
    - `anthropic-beta`: `files-api-2025-04-14`
  - *Request Options:*  
    - Response format set to `file` to handle binary download.
  - *Input connections:* From respective file ID extraction nodes.
  - *Output connections:* To corresponding Google Drive upload nodes.
  - *Edge cases:* Invalid or expired file_id, API errors, network timeouts.

---

### 2.6 Upload to Google Drive

- **Overview:** Uploads the downloaded file binary to a fixed Google Drive folder.
- **Nodes Involved:** `Upload PPTX`, `Upload PDF`, `Upload DOCX`, `Upload XLSX`
- **Node Details:**

  - *Type:* Google Drive
  - *Role:* Upload binary file to Google Drive.
  - *Configuration:*  
    - Upload destination: Folder ID `1tkCr7xdraoZwsHqeLm7FZ4aRWY94oLbZ` (configured folder in "My Drive")  
    - File name taken dynamically from binary data filename.  
    - Credentials: Google Drive OAuth2 account named "Google Drive account (n3w.it)".
  - *Input connections:* From respective Download nodes.
  - *Output connections:* None (terminal nodes).
  - *Edge cases:* Google Drive API errors, auth token expiration, insufficient permissions, file naming conflicts.

---

### 2.7 Setup & Instructions (Sticky Notes)

- **Overview:** Provides guidance and setup instructions for the workflow.
- **Nodes Involved:** `Sticky Note1`, `Sticky Note2`, `Sticky Note6`, `Sticky Note`
- **Node Details:**

  - Sticky Note1:  
    - Explains overall workflow purpose, usage with Anthropic API and Google Drive, and setup steps.
  - Sticky Note6:  
    - Describes Step 1: Add `x-api-key` header with Anthropic API key, get all skills, switch by file type.
  - Sticky Note:  
    - Describes Step 2: Create the desired documents and download using the file_id.
  - Sticky Note2:  
    - Describes Step 3: Connect Google Drive for upload.

---

## 3. Summary Table

| Node Name           | Node Type             | Functional Role                          | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                   |
|---------------------|-----------------------|----------------------------------------|-----------------------|------------------------|---------------------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger          | Receives user prompt and file type     | None                  | Switch                 |                                                                                                               |
| Switch              | Switch                | Routes workflow by file type            | On form submission    | Create PPTX, Create PDF, Create DOCX, Create XLSX |                                                                                                               |
| Create PPTX          | HTTP Request          | Generate PPTX document via Claude AI   | Switch (PPTX)          | Extract PPTX file_id    |                                                                                                               |
| Create PDF           | HTTP Request          | Generate PDF document via Claude AI    | Switch (PDF)           | Extract PDF file_id     |                                                                                                               |
| Create DOCX          | HTTP Request          | Generate DOCX document via Claude AI   | Switch (DOCX)          | Extract DOCX file_id    |                                                                                                               |
| Create XLSX          | HTTP Request          | Generate XLSX document via Claude AI   | Switch (XLSX)          | Extract XLSX file_id    |                                                                                                               |
| Extract PPTX file_id  | Code (JavaScript)     | Extract file_id from PPTX generation   | Create PPTX            | Download PPTX           |                                                                                                               |
| Extract PDF file_id   | Code (JavaScript)     | Extract file_id from PDF generation    | Create PDF             | Download PDF            |                                                                                                               |
| Extract DOCX file_id  | Code (JavaScript)     | Extract file_id from DOCX generation   | Create DOCX            | Download DOCX           |                                                                                                               |
| Extract XLSX file_id  | Code (JavaScript)     | Extract file_id from XLSX generation   | Create XLSX            | Download XLSX           |                                                                                                               |
| Download PPTX         | HTTP Request          | Download PPTX file content              | Extract PPTX file_id   | Upload PPTX             |                                                                                                               |
| Download PDF          | HTTP Request          | Download PDF file content               | Extract PDF file_id    | Upload PDF              |                                                                                                               |
| Download DOCX         | HTTP Request          | Download DOCX file content              | Extract DOCX file_id   | Upload DOCX             |                                                                                                               |
| Download XLSX         | HTTP Request          | Download XLSX file content              | Extract XLSX file_id   | Upload XLSX             |                                                                                                               |
| Upload PPTX           | Google Drive          | Upload PPTX file to Google Drive        | Download PPTX          | None                   |                                                                                                               |
| Upload PDF            | Google Drive          | Upload PDF file to Google Drive         | Download PDF           | None                   |                                                                                                               |
| Upload DOCX           | Google Drive          | Upload DOCX file to Google Drive        | Download DOCX          | None                   |                                                                                                               |
| Upload XLSX           | Google Drive          | Upload XLSX file to Google Drive        | Download XLSX          | None                   |                                                                                                               |
| Sticky Note1          | Sticky Note           | Workflow overview and usage instructions| None                  | None                   | Explains workflow purpose and setup with Anthropic and Google Drive                                          |
| Sticky Note6          | Sticky Note           | Step 1 instructions for API key header and skill routing | None | None | Step 1: Add x-api-key header, get skills, switch by file type                                                  |
| Sticky Note           | Sticky Note           | Step 2 instructions on document creation and download | None | None | Step 2: Create documents and download using file_id                                                           |
| Sticky Note2          | Sticky Note           | Step 3 instructions for Google Drive connection | None | None | Step 3: Connect Google Drive                                                                                   |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**
   - Type: Form Trigger
   - Name: `On form submission`
   - Configure form with title "Use Anthropic Skills"
   - Add two fields:
     - `Prompt` (Text, required)
     - `File Type` (Dropdown, required) with options: PPTX, PDF, DOCX, XLSX
   - Save and note webhook URL for external access.

2. **Add Switch Node**
   - Type: Switch
   - Name: `Switch`
   - Connect input from `On form submission`
   - Add four outputs with conditions:
     - Output 1: `File Type` equals "PPTX"
     - Output 2: `File Type` equals "PDF"
     - Output 3: `File Type` equals "DOCX"
     - Output 4: `File Type` equals "XLSX"

3. **Create HTTP Request Nodes to Generate Documents**
   
   For each file type (PPTX, PDF, DOCX, XLSX):

   - Add HTTP Request node named accordingly (e.g., `Create PPTX`).
   - Connect each from corresponding Switch output.
   - Set:
     - Method: POST
     - URL: `https://api.anthropic.com/v1/messages`
     - Authentication: HTTP Header Auth
       - Header name: `x-api-key`
       - Value: Your Anthropic API key
     - Additional headers:
       - `anthropic-version`: `2023-06-01`
       - `anthropic-beta`: `code-execution-2025-08-25,skills-2025-10-02`
     - Request Body Type: JSON
     - Body: Use the provided JSON templates per document type with:
       - `model`: `"claude-haiku-4-5-20251001"`
       - `max_tokens`: 4096 (except PPTX uses 50000)
       - `system`: Paste the respective professional instructions (see original JSON for details)
       - `container.skills`: skill type matching file type (`pptx`, `pdf`, `docx`, `xlsx`)
       - `messages.user.content`: Expression referencing `Prompt` field from form submission, e.g. `{{$node["On form submission"].json["Prompt"]}}`
       - `tools`: include `code_execution_20250825` tool

4. **Add Code Nodes to Extract `file_id`**

   For each file type:

   - Add a Code node named `Extract <File Type> file_id`
   - Connect from the respective `Create <File Type>` HTTP Request node.
   - Paste the recursive JavaScript code provided (see node details above) that searches response JSON and binary data for a `file_id`.
   - This code outputs an object with `{ file_id: <extracted_id> }`.

5. **Add HTTP Request Nodes to Download Files**

   For each file type:

   - Add HTTP Request node named `Download <File Type>`
   - Connect input from the corresponding `Extract <File Type> file_id` Code node.
   - Set:
     - Method: GET
     - URL: `https://api.anthropic.com/v1/files/{{$json["file_id"]}}/content`
     - Authentication: HTTP Header Auth with the same Anthropic API key.
     - Headers:
       - `x-api-key`: Your Anthropic API key
       - `anthropic-version`: `2023-06-01`
       - `anthropic-beta`: `files-api-2025-04-14`
     - Response Format: File (binary)
     - Ensure the URL expression dynamically uses the extracted `file_id`.
   
6. **Add Google Drive Upload Nodes**

   For each file type:

   - Add Google Drive node named `Upload <File Type>`
   - Connect input from the corresponding `Download <File Type>` node.
   - Configure:
     - Operation: Upload
     - Drive: Select your Google Drive account
     - Folder ID: Set to your target folder (e.g., `1tkCr7xdraoZwsHqeLm7FZ4aRWY94oLbZ`)
     - File Name: Use the binary file name (e.g., expression `{{$binary.data.fileName}}`)
     - Credentials: Use your Google Drive OAuth2 credentials.

7. **Add Sticky Notes (Optional but Recommended)**

   - Create sticky notes with summarized instructions:
     - Workflow overview and setup steps.
     - Step 1: Add API key header and skill routing.
     - Step 2: Document creation and downloading.
     - Step 3: Google Drive connection instructions.

8. **Activate and Test Workflow**

   - Activate the workflow.
   - Submit prompts via the exposed form, select file type.
   - Verify documents are generated, downloaded, and uploaded correctly to Google Drive.

---

## 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Workflow uses Anthropic Claude AI Skills API v2 with specialized skills for PPTX, PDF, DOCX, XLSX generation.                             | Anthropic API documentation: https://www.anthropic.com/api                                                  |
| Google Drive folder ID must be configured to a valid folder where files will be saved.                                                    | Google Drive API docs: https://developers.google.com/drive/api                                             |
| The API key placeholders must be replaced with valid Anthropic API keys.                                                                  | Ensure API keys have appropriate permissions and are kept secure.                                          |
| Max tokens and model versions are set to future dates (e.g., 2025) – verify compatibility with your Anthropic account settings.          | Monitor API version updates for breaking changes.                                                          |
| The code nodes use recursive JavaScript to extract `file_id` robustly, handling nested responses and binary data.                         | Helps minimize errors due to unexpected API response structures.                                           |
| Ensure Google Drive OAuth2 credentials are authorized for Drive folder access and have upload permissions.                                | Use OAuth2 credentials node in n8n for Google Drive.                                                       |
| The workflow includes comprehensive embedded instructions as sticky notes to assist users configuring and understanding workflow steps. |                                                                                                            |

---

**Disclaimer:**  
The text provided derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible.

---