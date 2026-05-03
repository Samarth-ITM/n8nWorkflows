Turn Any PDF into a Clean Google Doc with Mistral OCR

https://n8nworkflows.xyz/workflows/turn-any-pdf-into-a-clean-google-doc-with-mistral-ocr-6937


# Turn Any PDF into a Clean Google Doc with Mistral OCR

---

### 1. Workflow Overview

This workflow automates the process of converting any PDF document into a clean, well-formatted Google Doc using Mistral OCR technology. It is designed primarily for users who want to digitize PDFs containing both text and images, extract accurate OCR text including inline images, clean and normalize the extracted content, and then save it as a Google Document.

**Target Use Cases:**  
- Digitizing scanned PDFs or image-heavy documents.  
- Extracting embedded text from inline images within PDFs.  
- Cleaning OCR noise and artifacts for clean documentation.  
- Automatically generating Google Docs from scanned materials.

**Logical Blocks:**

- **1.1 Input Reception:** Handles user input via a form submission with PDF upload and document naming.  
- **1.2 PDF Upload & OCR Processing:** Uploads the PDF to Mistral, fetches a signed URL, and performs OCR on the document.  
- **1.3 Inline Image Extraction and Re-OCR:** Extracts embedded images from OCR response, re-runs OCR on each image to capture small or embedded text.  
- **1.4 Text Merging and Placeholder Replacement:** Combines page text and image OCR results, replaces image placeholders with OCR text from inline images.  
- **1.5 Text Cleaning using LLM:** Cleans and normalizes the combined markdown text using a language model for improved readability and formatting.  
- **1.6 Google Docs Creation and Text Insertion:** Creates a new Google Doc with the user-defined name and inserts the cleaned text content.  
- **1.7 Supporting Documentation and Guidance:** Provides sticky notes with instructions and video resources for users.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Receives user input through a web form, requiring a PDF file upload and a document name.

- **Nodes Involved:**  
  - On form submission  
  - Sticky Note (instructions)

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point that triggers workflow upon user form submission.  
    - Config: Form titled "Document Scanner" with two fields: a required PDF file upload and a required text input for "Document Name".  
    - Input: User-submitted form data.  
    - Output: Passes file and document name JSON data to next node.  
    - Edge cases: Missing file or document name will block submission due to required fields. Form trigger webhook URL must be publicly accessible.  
    - Version: 2.2

  - **Sticky Note (Instructions)**  
    - Type: Sticky Note  
    - Role: Provides user-facing instructions on required settings and workflow overview.  
    - Content highlights:  
      - Upload PDF and enter document name in form trigger.  
      - Paste Google Drive folder ID for Google Doc creation.  
      - Ensure active Mistral and Google Docs credentials.  
      - Describes workflow steps and customization options.  
    - No inputs or outputs.

#### 1.2 PDF Upload & OCR Processing

- **Overview:**  
Uploads the submitted PDF to Mistral API, retrieves a signed URL for the uploaded file, and runs OCR on the PDF to extract text and embedded images.

- **Nodes Involved:**  
  - Upload • PDF to Mistral  
  - Fetch • Signed URL  
  - OCR • PDF via Mistral

- **Node Details:**

  - **Upload • PDF to Mistral**  
    - Type: HTTP Request  
    - Role: Uploads PDF file to Mistral OCR service.  
    - Config: POST to `https://api.mistral.ai/v1/files` with form multipart data including purpose "ocr" and the uploaded PDF binary.  
    - Credentials: Mistral Cloud API OAuth.  
    - Input: PDF file binary from form trigger.  
    - Output: JSON response with file ID for next step.  
    - Edge cases: File size limits, network errors, or auth failures.

  - **Fetch • Signed URL**  
    - Type: HTTP Request  
    - Role: Retrieves a time-limited signed URL to access the uploaded PDF.  
    - Config: GET to `https://api.mistral.ai/v1/files/{{file_id}}/url` with expiry=24 hours.  
    - Credentials: Mistral Cloud API OAuth.  
    - Input: File ID from upload node.  
    - Output: JSON containing URL for OCR request.  
    - Edge cases: Expired URL, invalid file ID, or auth errors.

  - **OCR • PDF via Mistral**  
    - Type: HTTP Request  
    - Role: Requests OCR processing on the PDF via the signed URL.  
    - Config: POST to `https://api.mistral.ai/v1/ocr` with JSON body specifying model `mistral-ocr-latest` and document URL. Includes base64 images in response.  
    - Credentials: Mistral Cloud API OAuth.  
    - Input: Signed URL from previous node.  
    - Output: OCR JSON response with pages, markdown text, and embedded image data.  
    - Edge cases: OCR processing delays, API limits, malformed response.

#### 1.3 Inline Image Extraction and Re-OCR

- **Overview:**  
Extracts all inline images embedded in the OCR markdown, re-uploads each image to Mistral, and performs OCR on each to capture any additional text.

- **Nodes Involved:**  
  - Extract • Image Placeholders  
  - OCR • Inline Images  
  - Merge • OCR + Image Data  
  - Set • Rename Columns  
  - Aggregate • Image Text

- **Node Details:**

  - **Extract • Image Placeholders**  
    - Type: Code  
    - Role: Parses OCR JSON to find all embedded inline images referenced in markdown by regex matching markdown image syntax.  
    - Config: Custom JS extracts image IDs and base64 data URIs. Outputs one item per image or a note if none found.  
    - Input: OCR output JSON.  
    - Output: Array of image objects with page index, image ID, and base64 imageDataUri.  
    - Edge cases: No embedded images scenario outputs a note.

  - **OCR • Inline Images**  
    - Type: HTTP Request  
    - Role: Sends each extracted image base64 URI to Mistral OCR for re-processing.  
    - Config: POST to Mistral OCR endpoint with document type "image_url" and base64 URI.  
    - Credentials: Mistral Cloud API OAuth.  
    - Input: Image data URI from extraction node.  
    - Output: OCR text for each image.  
    - Edge cases: Large images, OCR failures on images, API rate limits.

  - **Merge • OCR + Image Data**  
    - Type: Merge  
    - Role: Combines the original OCR PDF data and the inline image OCR results by position.  
    - Config: Combine mode, combining input streams by their position index.  
    - Input: OCR PDF and OCR inline images outputs.  
    - Output: Combined data containing both page and image OCR text.  
    - Edge cases: Input length mismatch.

  - **Set • Rename Columns**  
    - Type: Set  
    - Role: Renames and organizes key fields like pageIndex, imageId, imageDataUri, and imagePages for downstream aggregation.  
    - Input: Combined OCR data.  
    - Output: Standardized JSON with renamed properties.

  - **Aggregate • Image Text**  
    - Type: Aggregate  
    - Role: Aggregates markdown text from the OCR results of inline images for merging.  
    - Config: Aggregates the field `imagePages[0].markdown`.  
    - Input: JSON with image OCR markdown.  
    - Output: Aggregated markdown array for images.

#### 1.4 Text Merging and Placeholder Replacement

- **Overview:**  
Merges the page-level OCR markdown with the inline image OCR text, replacing image placeholders in the markdown with the corresponding inline OCR text to form a unified document.

- **Nodes Involved:**  
  - Merge • PDF & Images Markdown  
  - Enrich • Replace Placeholders

- **Node Details:**

  - **Merge • PDF & Images Markdown**  
    - Type: Merge  
    - Role: Combines the OCR markdown of the whole PDF and the aggregated markdown of inline images by position.  
    - Config: Combine mode by position index.  
    - Input: Page OCR markdown and aggregated inline image markdown.  
    - Output: Combined rows for placeholder replacement.

  - **Enrich • Replace Placeholders**  
    - Type: Code  
    - Role: Replaces markdown placeholders (such as `![alt](img-0.jpg)`) in the main document with the OCR text extracted from corresponding inline images.  
    - Config: Complex JS logic to:  
      - Identify placeholders by regex.  
      - Strip image tokens from image OCR text to prevent re-insertion loops.  
      - Replace or append inline OCR text depending on config (`replace` mode used).  
      - Output enriched markdown text with placeholders replaced by actual OCR text.  
    - Input: Combined OCR markdown rows.  
    - Output: Enriched markdown text ready for cleaning.  
    - Edge cases: Missing placeholders, no mapping found, or malformed markdown.

#### 1.5 Text Cleaning using LLM

- **Overview:**  
Uses a language model (LLM) to clean OCR-extracted markdown text, removing noise such as headers, footers, boilerplate, broken formatting, and gibberish, while preserving meaningful content.

- **Nodes Involved:**  
  - LLM Model • GPT-4.1-mini (OpenRouter)  
  - Clean • Markdown Text

- **Node Details:**

  - **LLM Model • GPT-4.1-mini**  
    - Type: LangChain Language Model node (OpenRouter GPT-4.1-mini)  
    - Role: Provides the AI language model for cleaning text.  
    - Credentials: OpenRouter API (Augra instance).  
    - Input: Raw enriched markdown text from placeholder replacement node.  
    - Output: Cleaned text for further processing.

  - **Clean • Markdown Text**  
    - Type: LangChain Chain LLM node  
    - Role: Defines the prompt and invokes the LLM to perform detailed text cleaning and normalization.  
    - Configuration:  
      - Input prompt instructs the model to remove boilerplate, formatting noise, headers/footers, repeated numbers, broken tables, etc.  
      - Preserve valid sentences, lists, code blocks, and original language.  
      - Output: Clean markdown text only, no summary or rewriting.  
    - Input: Enriched markdown text from previous node.  
    - Output: Clean, normalized markdown content.  
    - Edge cases: LLM API rate limits or timeouts, incomplete cleaning if input is malformed.

#### 1.6 Google Docs Creation and Text Insertion

- **Overview:**  
Creates a new Google Document with the user-specified name in a predefined Google Drive folder and inserts the cleaned markdown text into it.

- **Nodes Involved:**  
  - Create • Google Doc  
  - Insert • Clean Text into Doc

- **Node Details:**

  - **Create • Google Doc**  
    - Type: Google Docs node  
    - Role: Creates a new Google Doc with title from form input and in a fixed folder ID.  
    - Config:  
      - Title dynamically set from "Document Name" form field.  
      - Folder ID hardcoded as `16ooKdLhm5GzzKSSZl1wMGTHJOfJ3-r13`.  
    - Credentials: Google Docs OAuth2 account.  
    - Input: Clean markdown text trigger.  
    - Output: Created document metadata including document URL/ID.  
    - Edge cases: Invalid folder ID or insufficient permissions.

  - **Insert • Clean Text into Doc**  
    - Type: Google Docs node  
    - Role: Inserts the cleaned markdown text into the newly created Google Doc.  
    - Config:  
      - Operation: update (insert text).  
      - Document URL: dynamic from created document node output.  
      - Text: cleaned markdown text from previous step.  
    - Credentials: Google Docs OAuth2 account.  
    - Input: Created Google Doc ID and cleaned text.  
    - Output: Confirmation of text insertion.  
    - Edge cases: Insert failures due to document access or format issues.

#### 1.7 Supporting Documentation and Guidance

- **Overview:**  
Provides visual and textual user guidance including a labeled sticky note with essential starting instructions and a YouTube tutorial link.

- **Nodes Involved:**  
  - Sticky Note (Instructions)  
  - Sticky Note8 (YouTube Tutorial)

- **Node Details:**

  - **Sticky Note (Instructions)**  
    - See details in 1.1 Input Reception.

  - **Sticky Note8 (YouTube Tutorial)**  
    - Type: Sticky Note with embedded YouTube thumbnail and link.  
    - Content: Directs users to a step-by-step YouTube tutorial titled "Building an AI Personal Assistant".  
    - Link: https://youtu.be/VjIoAqvRPGs  
    - Use: Supports users learning to build or understand workflow components.  
    - No inputs or outputs.

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                      | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                  |
|----------------------------|---------------------------------|------------------------------------|-----------------------------|------------------------------|----------------------------------------------------------------------------------------------|
| On form submission         | Form Trigger                    | Entry point: receives PDF and name | None                        | Upload • PDF to Mistral       | ## Start here (required settings) • Upload your PDF and enter a document name in the form trigger. |
| Sticky Note               | Sticky Note                     | Provides workflow overview & instructions | None                        | None                         | ## Start here (required settings) • Upload your PDF and enter a document name in the form trigger. • Paste the Google Drive folder ID. • Ensure credentials are active. … |
| Upload • PDF to Mistral    | HTTP Request                   | Upload PDF to Mistral OCR service  | On form submission          | Fetch • Signed URL            |                                                                                              |
| Fetch • Signed URL         | HTTP Request                   | Get signed URL for uploaded PDF    | Upload • PDF to Mistral     | OCR • PDF via Mistral         |                                                                                              |
| OCR • PDF via Mistral      | HTTP Request                   | OCR the PDF document               | Fetch • Signed URL          | Extract • Image Placeholders, Merge • PDF & Images Markdown |                                                                                              |
| Extract • Image Placeholders | Code                          | Extract inline images from OCR markdown | OCR • PDF via Mistral       | OCR • Inline Images, Merge • OCR + Image Data |                                                                                              |
| OCR • Inline Images        | HTTP Request                   | Run OCR on each inline image       | Extract • Image Placeholders | Merge • OCR + Image Data      |                                                                                              |
| Merge • OCR + Image Data   | Merge                          | Combine OCR PDF and inline image OCR | OCR • Inline Images, Extract • Image Placeholders | Set • Rename Columns           |                                                                                              |
| Set • Rename Columns       | Set                            | Rename and organize OCR fields     | Merge • OCR + Image Data    | Aggregate • Image Text        |                                                                                              |
| Aggregate • Image Text     | Aggregate                      | Aggregate inline image markdown    | Set • Rename Columns        | Merge • PDF & Images Markdown |                                                                                              |
| Merge • PDF & Images Markdown | Merge                       | Combine page and image markdown    | OCR • PDF via Mistral, Aggregate • Image Text | Enrich • Replace Placeholders |                                                                                              |
| Enrich • Replace Placeholders | Code                        | Replace placeholders with image OCR text | Merge • PDF & Images Markdown | Clean • Markdown Text         |                                                                                              |
| LLM Model • GPT-4.1-mini  | LangChain LLM                  | Language model for text cleaning   | Enrich • Replace Placeholders | Clean • Markdown Text         |                                                                                              |
| Clean • Markdown Text      | LangChain Chain LLM            | Clean and normalize markdown text  | LLM Model • GPT-4.1-mini    | Create • Google Doc           |                                                                                              |
| Create • Google Doc        | Google Docs                    | Create a new Google Doc with title | Clean • Markdown Text       | Insert • Clean Text into Doc  | ## Start here (required settings) • Paste the Google Drive folder ID.                        |
| Insert • Clean Text into Doc | Google Docs                  | Insert cleaned markdown text into Google Doc | Create • Google Doc         | None                         |                                                                                              |
| Sticky Note8              | Sticky Note                     | YouTube tutorial link & visual aid | None                        | None                         | ## Start here: Step by Step Youtube Tutorial :star: [Building an AI Personal Assistant](https://youtu.be/VjIoAqvRPGs) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node: "On form submission"**  
   - Node Type: Form Trigger (version 2.2)  
   - Configure form title: "Document Scanner"  
   - Add fields:  
     - File upload field labeled "file", required, accept only ".pdf"  
     - Text field labeled "Document Name", placeholder "NVIDIA Annual Report Doc", required  
   - Save webhook URL for external form submission.

2. **Add a Sticky Note with Instructions**  
   - Node Type: Sticky Note  
   - Content: Include detailed instructions about required settings, workflow purpose, and customization tips.

3. **Add HTTP Request Node: "Upload • PDF to Mistral"**  
   - Node Type: HTTP Request (version 4.2)  
   - Method: POST  
   - URL: `https://api.mistral.ai/v1/files`  
   - Content-Type: multipart/form-data  
   - Body Parameters:  
     - purpose = "ocr"  
     - file = binary data from form upload "file" field  
   - Authentication: Use Mistral Cloud API credentials (set up OAuth or API key).

4. **Add HTTP Request Node: "Fetch • Signed URL"**  
   - Node Type: HTTP Request (version 4.2)  
   - Method: GET  
   - URL: `https://api.mistral.ai/v1/files/{{ $json.id }}/url` (use file ID from upload node)  
   - Query Parameter: expiry=24 (hours)  
   - Headers: Accept: application/json  
   - Authentication: Mistral Cloud API

5. **Add HTTP Request Node: "OCR • PDF via Mistral"**  
   - Node Type: HTTP Request (version 4.2)  
   - Method: POST  
   - URL: `https://api.mistral.ai/v1/ocr`  
   - Body Type: JSON  
   - JSON Body:  
     ```json
     {
       "model": "mistral-ocr-latest",
       "document": {
         "type": "document_url",
         "document_url": "{{ $json.url }}"
       },
       "include_image_base64": true
     }
     ```  
   - Authentication: Mistral Cloud API

6. **Add Code Node: "Extract • Image Placeholders"**  
   - Node Type: Code (version 2)  
   - JavaScript code: Extract markdown inline image placeholders and base64 data URIs from OCR response.  
   - Input: OCR JSON  
   - Output: Array of image objects with pageIndex, imageId, imageDataUri.

7. **Add HTTP Request Node: "OCR • Inline Images"**  
   - Node Type: HTTP Request (version 4.2)  
   - Method: POST  
   - URL: `https://api.mistral.ai/v1/ocr`  
   - Body Type: JSON  
   - JSON Body:  
     ```json
     {
       "model": "mistral-ocr-latest",
       "document": {
         "type": "image_url",
         "image_url": "{{ $json.imageDataUri }}"
       },
       "include_image_base64": true
     }
     ```  
   - Authentication: Mistral Cloud API

8. **Add Merge Node: "Merge • OCR + Image Data"**  
   - Node Type: Merge (version 3.2)  
   - Mode: Combine  
   - Combine By: Position  
   - Input: Outputs of "Extract • Image Placeholders" and "OCR • Inline Images"

9. **Add Set Node: "Set • Rename Columns"**  
   - Node Type: Set (version 3.4)  
   - Rename fields: pageIndex, imageId, imageDataUri, imagePages (array of pages) as per logic.

10. **Add Aggregate Node: "Aggregate • Image Text"**  
    - Node Type: Aggregate (version 1)  
    - Aggregate field: `imagePages[0].markdown` (collect inline image markdown text)

11. **Add Merge Node: "Merge • PDF & Images Markdown"**  
    - Node Type: Merge (version 3.2)  
    - Mode: Combine  
    - Combine By: Position  
    - Input: Outputs of "OCR • PDF via Mistral" and "Aggregate • Image Text"

12. **Add Code Node: "Enrich • Replace Placeholders"**  
    - Node Type: Code (version 2)  
    - JavaScript: Replace markdown placeholders in page OCR markdown with inline image OCR text using regex and map logic.  
    - Input: Combined OCR markdown rows  
    - Output: Enriched markdown text

13. **Add LangChain LLM Node: "LLM Model • GPT-4.1-mini"**  
    - Node Type: LangChain LLM (OpenRouter)  
    - Credentials: OpenRouter API key (Augra instance)  
    - Input: Enriched markdown text

14. **Add LangChain Chain LLM Node: "Clean • Markdown Text"**  
    - Node Type: LangChain Chain LLM (version 1.7)  
    - Prompt: Clean OCR text prompt that removes noise, placeholders, broken formatting while preserving meaningful content and original language.  
    - Input: Output from LLM Model node

15. **Add Google Docs Node: "Create • Google Doc"**  
    - Node Type: Google Docs (version 2)  
    - Parameters:  
      - Title: Use expression from form field "Document Name"  
      - Folder ID: Set to Google Drive folder ID where Docs are saved  
    - Credentials: Google Docs OAuth2

16. **Add Google Docs Node: "Insert • Clean Text into Doc"**  
    - Node Type: Google Docs (version 2)  
    - Operation: Update (insert text)  
    - Document URL: Use created document ID from previous node  
    - Text: Clean markdown text from cleaning node  
    - Credentials: Google Docs OAuth2

17. **Add Sticky Note Node for YouTube Tutorial**  
    - Node Type: Sticky Note  
    - Content: Embed YouTube link and thumbnail for a step-by-step tutorial on building an AI personal assistant.

18. **Connect Nodes According to Workflow Logic:**  
    - Connect form submission → upload → fetch URL → OCR PDF → extract images → OCR inline images → merge OCR data → rename columns → aggregate image text → merge markdown → replace placeholders → LLM model → clean markdown → create Google Doc → insert text.

19. **Activate Workflow and Test:**  
    - Ensure Mistral and Google Docs credentials are active and valid.  
    - Test by submitting a PDF and document name via the form.  
    - Monitor logs for errors such as API timeouts, authentication failures, or malformed inputs.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                       |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| "Start here (required settings)" instructions emphasize uploading PDFs, entering document name, pasting Google Drive folder ID, and ensuring active credentials. | Sticky Note at workflow start                                                          |
| Workflow uses Mistral OCR API for both PDF and inline image OCR to maximize text extraction accuracy. | Workflow functionality                                                                |
| Cleaning prompt used with LLM preserves original language and content tone while removing noise and artifacts. | LLM node prompt explanation                                                          |
| YouTube tutorial video for step-by-step guidance: "Building an AI Personal Assistant"           | https://youtu.be/VjIoAqvRPGs (linked in Sticky Note8)                                |
| Google Docs folder ID is hardcoded; update if you want to save documents elsewhere.              | Google Docs node configuration                                                       |
| Ensure that the Mistral Cloud API and Google OAuth2 credentials are correctly configured in n8n before running. | Credential setup requirement                                                         |

---

**Disclaimer:** The provided text is solely derived from an automated workflow created with n8n, an integration and automation tool. The data handled is legal and public, and the workflow adheres strictly to content policies.

---