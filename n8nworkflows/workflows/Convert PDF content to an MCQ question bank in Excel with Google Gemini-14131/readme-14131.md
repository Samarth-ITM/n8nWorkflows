Convert PDF content to an MCQ question bank in Excel with Google Gemini

https://n8nworkflows.xyz/workflows/convert-pdf-content-to-an-mcq-question-bank-in-excel-with-google-gemini-14131


# Convert PDF content to an MCQ question bank in Excel with Google Gemini

## 1. Workflow Overview

This workflow receives a PDF through an HTTP webhook, extracts its text, splits the text into chunks, asks Google Gemini to generate multiple-choice questions from each chunk, converts the AI output into structured rows, exports the result to an Excel file, uploads the file to Google Drive, and optionally delivers it through Telegram and email.

Its main use case is automated question-bank generation for educators, trainers, and content creators who want to turn PDF content into MCQs with answer keys and explanations.

### 1.1 Input Reception and Validation
The workflow starts from a `Webhook` node that accepts a POST request. A validation code node checks the uploaded PDF metadata, especially file size, and derives a sanitized output filename.

### 1.2 Validation Routing and Early HTTP Response
An `IF` node decides whether the uploaded file is acceptable. If valid, the workflow immediately returns a success response to the webhook caller while processing continues asynchronously; if invalid, it returns an error response.

### 1.3 PDF Text Extraction and Chunking
The PDF binary is parsed into text using `Extract from File`. A code node splits the extracted text into fixed-size chunks so the AI model receives manageable prompts.

### 1.4 AI-Based MCQ Generation
Each chunk is sent to a Google Gemini chat/model node with a strict prompt requiring valid JSON output only. The model is instructed to produce up to three MCQs per chunk.

### 1.5 Output Cleanup and Structuring
A code node removes Markdown fences if present, parses the returned JSON, and emits one item per generated question.

### 1.6 File Generation, Storage, and Delivery
The structured question rows are converted into an XLSX file, uploaded to Google Drive, sent to Telegram as a document, and sent by email as an attachment.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception and File Validation

**Overview:**  
This block receives the inbound PDF through an HTTP endpoint and validates whether the file is acceptable for processing. It also prepares metadata used later for naming the output Excel file.

**Nodes Involved:**  
- Webhook  
- Validate Parameter & PDF  
- Condition Valid  
- Respond Success  
- Respond Failed  

#### Webhook
- **Type and technical role:** `n8n-nodes-base.webhook` — entry point for external HTTP requests.
- **Configuration choices:**  
  - Method: `POST`
  - Response mode: `responseNode`, meaning the workflow must explicitly answer using `Respond to Webhook` nodes.
  - Path: a generated unique path segment.
- **Key expressions or variables used:** None in the node itself.
- **Input and output connections:**  
  - No input; this is the trigger node.
  - Output goes to `Validate Parameter & PDF`.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**  
  - Wrong HTTP method
  - Missing binary file payload
  - Caller timeout expectations if they assume the response means processing is complete
- **Sub-workflow reference:** None.

#### Validate Parameter & PDF
- **Type and technical role:** `n8n-nodes-base.code` — custom JavaScript validation and metadata preparation.
- **Configuration choices:**  
  - Reads `$input.first().binary.data.fileSize`
  - Converts file size strings like KB/MB into bytes
  - Rejects files `>= 5,242,880` bytes (5 MB)
  - Builds:
    - `isValid`
    - `errors`
    - `data`
    - `fileName`
- **Key expressions or variables used:**  
  - `$input.first().binary.data`
  - `$input.first().binary.data.fileName.split('.').slice(0, -1).join('.').replaceAll(' ', '-')`
- **Input and output connections:**  
  - Input from `Webhook`
  - Output to `Condition Valid`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - If `binary.data` is absent, the code will fail
  - If `fileSize` is missing or in an unexpected format, parsing may produce `NaN`
  - The node does not explicitly validate MIME type or extension
  - The comment says “ensures file exists,” but the code only checks size and assumes the binary exists
- **Sub-workflow reference:** None.

#### Condition Valid
- **Type and technical role:** `n8n-nodes-base.if` — routes execution depending on validation result.
- **Configuration choices:**  
  - Checks `{{$json.isValid}} == true`
- **Key expressions or variables used:**  
  - `={{ $json.isValid }}`
- **Input and output connections:**  
  - Input from `Validate Parameter & PDF`
  - True output to:
    - `Extract from File`
    - `Respond Success`
  - False output to:
    - `Respond Failed`
- **Version-specific requirements:** Type version `2.2`, condition options version `2`.
- **Edge cases or potential failure types:**  
  - If `isValid` is undefined, strict comparison will evaluate to false
- **Sub-workflow reference:** None.

#### Respond Success
- **Type and technical role:** `n8n-nodes-base.respondToWebhook` — returns HTTP success to the original caller.
- **Configuration choices:**  
  - Response code: `200`
  - Respond with JSON
  - Body:
    ```json
    {
      "status": "success",
      "message": "oke"
    }
    ```
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from `Condition Valid` true branch
  - No output
- **Version-specific requirements:** Type version `1.4`.
- **Edge cases or potential failure types:**  
  - Response is sent before the downstream processing completes, so the caller may receive success even if later nodes fail
- **Sub-workflow reference:** None.

#### Respond Failed
- **Type and technical role:** `n8n-nodes-base.respondToWebhook` — returns validation failure to the original caller.
- **Configuration choices:**  
  - Respond with JSON
  - Body uses expression to include the `errors` array
- **Key expressions or variables used:**  
  - `{{ $json.errors }}`
- **Input and output connections:**  
  - Input from `Condition Valid` false branch
  - No output
- **Version-specific requirements:** Type version `1.4`.
- **Edge cases or potential failure types:**  
  - Error messages are returned as rendered array text, not necessarily a structured JSON array depending on expression rendering
- **Sub-workflow reference:** None.

---

### Block 2 — PDF Text Extraction and Chunking

**Overview:**  
This block converts the uploaded PDF into plain text and breaks the text into 2,000-character chunks. This reduces prompt size and allows the AI model to process long documents incrementally.

**Nodes Involved:**  
- Extract from File  
- Chunk  

#### Extract from File
- **Type and technical role:** `n8n-nodes-base.extractFromFile` — extracts textual content from a PDF binary.
- **Configuration choices:**  
  - Operation: `pdf`
  - Binary property name: `={{ $json.data }}`
- **Key expressions or variables used:**  
  - `={{ $json.data }}`
- **Input and output connections:**  
  - Input from `Condition Valid` true branch
  - Output to `Chunk`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - This node normally expects the *name of a binary property*, not the binary object itself. In many n8n setups, this expression is likely incorrect unless the upstream item structure matches exactly in a special way.
  - If the incoming binary property is not correctly referenced, extraction will fail
  - Scanned PDFs with no text layer may produce little or no text
  - Complex layouts may extract poorly
- **Sub-workflow reference:** None.

#### Chunk
- **Type and technical role:** `n8n-nodes-base.code` — slices extracted text into fixed-size items.
- **Configuration choices:**  
  - Uses `$json.text || $json.data || ""`
  - Splits text into chunks of `2000` characters
  - Emits one item per chunk as `{ json: { chunk } }`
- **Key expressions or variables used:**  
  - `$json.text`
  - `$json.data`
- **Input and output connections:**  
  - Input from `Extract from File`
  - Output to `Message a model`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Character-based chunking may split sentences or concepts mid-way
  - Empty extraction produces zero or low-value chunks
  - No overlap between chunks, so some context can be lost at boundaries
- **Sub-workflow reference:** None.

---

### Block 3 — AI MCQ Generation and Cleanup

**Overview:**  
This block sends each chunk to Google Gemini with strict formatting instructions and then normalizes the response into one item per question. It is the core intelligence layer of the workflow.

**Nodes Involved:**  
- Message a model  
- Cleaner  

#### Message a model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.googleGemini` — prompts Google Gemini/Gemma to generate MCQs.
- **Configuration choices:**  
  - Model: `models/gemma-3-1b-it`
  - Prompt instructs the model to:
    - act as an expert teacher and exam creator
    - generate MCQs from the chunk
    - provide exactly 4 options
    - include one correct answer and explanation
    - avoid duplicates
    - generate max 3 questions per chunk
    - return valid JSON only
  - Content source: `{{ $json.chunk }}`
- **Key expressions or variables used:**  
  - `{{ $json.chunk }}`
- **Input and output connections:**  
  - Input from `Chunk`
  - Output to `Cleaner`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - API credential errors
  - Quota/rate-limit errors
  - Model may ignore formatting instructions and return malformed JSON
  - The configured model is a Gemma model via Gemini integration; availability depends on account, region, and node version
  - Large prompt accumulation across many chunks can increase latency and cost
- **Sub-workflow reference:** None.

#### Cleaner
- **Type and technical role:** `n8n-nodes-base.code` — parses and flattens AI output.
- **Configuration choices:**  
  - Reads `item.json.content.parts[0].text`
  - Removes code fences like ```json
  - Parses the cleaned string as JSON
  - Assumes the parsed result is an array of question objects
  - Emits each question as a separate output item
  - Silently skips items that fail parsing
- **Key expressions or variables used:**  
  - `item.json?.content?.parts?.[0]?.text`
  - `JSON.parse(clean)`
- **Input and output connections:**  
  - Input from `Message a model`
  - Output to `Convert to File`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - If the AI output is not valid JSON, that chunk is discarded
  - If the AI returns a single object instead of an array, the loop logic will not behave as intended
  - Silent failure can hide quality issues because invalid chunks are simply skipped
  - If no valid questions survive parsing, downstream file generation may still run but contain no rows
- **Sub-workflow reference:** None.

---

### Block 4 — File Generation and Storage

**Overview:**  
This block converts the normalized MCQ items into an Excel workbook and uploads the result to Google Drive. It creates the reusable document artifact that is later distributed.

**Nodes Involved:**  
- Convert to File  
- Upload file  

#### Convert to File
- **Type and technical role:** `n8n-nodes-base.convertToFile` — turns structured items into an XLSX binary.
- **Configuration choices:**  
  - Operation: `xlsx`
  - Header row enabled
  - Sheet name: `Sheet 1`
  - File name:
    `result-n8n-{{$('Validate Parameter & PDF').first().json.fileName }}.xlsx`
- **Key expressions or variables used:**  
  - `$('Validate Parameter & PDF').first().json.fileName`
- **Input and output connections:**  
  - Input from `Cleaner`
  - Output to:
    - `Send a document`
    - `Send email`
    - `Upload file`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - If there are zero incoming items, the generated file may be empty or the node may behave unexpectedly depending on n8n version
  - Column order follows incoming JSON key order
- **Sub-workflow reference:** None.

#### Upload file
- **Type and technical role:** `n8n-nodes-base.googleDrive` — uploads the generated Excel file to a Google Drive folder.
- **Configuration choices:**  
  - File name matches the generated result name
  - Drive: `My Drive`
  - Folder: configured folder named `n8n`
- **Key expressions or variables used:**  
  - `=result-n8n-{{$('Validate Parameter & PDF').first().json.fileName }}.xlsx`
- **Input and output connections:**  
  - Input from `Convert to File`
  - No output
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**  
  - Google OAuth permission issues
  - Folder access revoked or folder deleted
  - If the binary property expected by the node is not present, upload fails
  - Duplicate filenames may create multiple files unless overwrite behavior is configured elsewhere
- **Sub-workflow reference:** None.

---

### Block 5 — Delivery and Notification

**Overview:**  
This block distributes the final Excel file through Telegram and email. It is intended as the delivery layer for end users after processing completes.

**Nodes Involved:**  
- Send a document  
- Send email  

#### Send a document
- **Type and technical role:** `n8n-nodes-base.telegram` — sends the XLSX as a Telegram document.
- **Configuration choices:**  
  - Operation: `sendDocument`
  - Binary data enabled
  - Chat ID is hardcoded to `12345678`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from `Convert to File`
  - No output
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Hardcoded chat ID means delivery is not dynamic
  - Invalid bot token or missing permissions
  - Bot cannot message a user/chat unless previously started or allowed
  - File size limits may apply depending on Telegram API/account context
- **Sub-workflow reference:** None.

#### Send email
- **Type and technical role:** `n8n-nodes-base.emailSend` — sends the XLSX as an email attachment.
- **Configuration choices:**  
  - Subject: fixed title describing the result
  - Body text: `Here attachment result`
  - Attachments property: `data`
  - To/From email both hardcoded as `user@example.com`
  - Format: plain text
- **Key expressions or variables used:**  
  - Attachment binary property name: `data`
- **Input and output connections:**  
  - Input from `Convert to File`
  - No output
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**  
  - Hardcoded recipient and sender values must be replaced
  - SMTP authentication or relay restrictions
  - Some SMTP providers reject spoofed `fromEmail`
  - Attachment binary property must match the output of `Convert to File`
- **Sub-workflow reference:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook | Receives incoming POST request with PDF payload |  | Validate Parameter & PDF | ## 1. Input & File Validation<br>Handles incoming PDF, validates file size (max 5MB) and ensures file exists before processing. |
| Validate Parameter & PDF | Code | Validates PDF size and prepares metadata including sanitized filename | Webhook | Condition Valid | ## 1. Input & File Validation<br>Handles incoming PDF, validates file size (max 5MB) and ensures file exists before processing. |
| Condition Valid | If | Routes valid vs invalid requests | Validate Parameter & PDF | Extract from File; Respond Success; Respond Failed | ## 1. Input & File Validation<br>Handles incoming PDF, validates file size (max 5MB) and ensures file exists before processing. |
| Respond Success | Respond to Webhook | Returns early success HTTP response | Condition Valid |  | ## 1. Input & File Validation<br>Handles incoming PDF, validates file size (max 5MB) and ensures file exists before processing. |
| Respond Failed | Respond to Webhook | Returns validation failure HTTP response | Condition Valid |  | ## 1. Input & File Validation<br>Handles incoming PDF, validates file size (max 5MB) and ensures file exists before processing. |
| Extract from File | Extract from File | Extracts text from uploaded PDF | Condition Valid | Chunk | ## 2. PDF Text Extraction & Chunking<br>Extracts text from PDF and splits it into manageable chunks for AI processing. |
| Chunk | Code | Splits extracted text into 2,000-character chunks | Extract from File | Message a model | ## 2. PDF Text Extraction & Chunking<br>Extracts text from PDF and splits it into manageable chunks for AI processing. |
| Message a model | Google Gemini | Generates MCQs from each text chunk | Chunk | Cleaner | ## 3. AI MCQ Generation (Google Gemini)<br>Generates structured multiple choice questions using AI and cleans the output into valid JSON format. |
| Cleaner | Code | Removes fences, parses JSON, and emits one row per question | Message a model | Convert to File | ## 3. AI MCQ Generation (Google Gemini)<br>Generates structured multiple choice questions using AI and cleans the output into valid JSON format. |
| Convert to File | Convert to File | Creates XLSX from structured MCQ items | Cleaner | Send a document; Send email; Upload file | ## 4. Output Generation & Storage<br>Converts structured data into Excel format and uploads the file to Google Drive. |
| Upload file | Google Drive | Uploads generated Excel file to Google Drive | Convert to File |  | ## 4. Output Generation & Storage<br>Converts structured data into Excel format and uploads the file to Google Drive. |
| Send a document | Telegram | Sends XLSX to Telegram chat | Convert to File |  | ## 5. Delivery & Notification<br>Sends the generated Excel file to the user via Telegram and Email. |
| Send email | Send Email | Sends XLSX as email attachment | Convert to File |  | ## 5. Delivery & Notification<br>Sends the generated Excel file to the user via Telegram and Email. |
| Sticky Note | Sticky Note | Visual documentation for input/validation block |  |  |  |
| Sticky Note1 | Sticky Note | Visual documentation for extraction/chunking block |  |  |  |
| Sticky Note2 | Sticky Note | Visual documentation for AI generation block |  |  |  |
| Sticky Note3 | Sticky Note | Visual documentation for file generation/storage block |  |  |  |
| Sticky Note4 | Sticky Note | Visual documentation for delivery block |  |  |  |
| Sticky Note5 | Sticky Note | General workflow documentation and setup notes |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Convert PDF to MCQ Question Bank in Excel with AI (Gemini)`.

2. **Add a Webhook node** named `Webhook`.
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Response Mode: `Using Respond to Webhook Node`
   - Path: generate or paste a unique path
   - Expect the request to include a PDF file as binary input

3. **Add a Code node** named `Validate Parameter & PDF` and connect `Webhook -> Validate Parameter & PDF`.
   - Paste logic that:
     - reads the uploaded binary file metadata
     - parses `fileSize`
     - rejects files over 5 MB
     - outputs:
       - `isValid`
       - `errors`
       - `data`
       - `fileName`
   - Use the same behavior as the original workflow:
     - replace spaces in filename with `-`
     - remove extension for the base output name
   - Important assumption: the inbound file must be available as `binary.data`.

4. **Add an IF node** named `Condition Valid` and connect `Validate Parameter & PDF -> Condition Valid`.
   - Condition:
     - left value: `{{ $json.isValid }}`
     - operation: `equals`
     - right value: `true`

5. **Add a Respond to Webhook node** named `Respond Success`.
   - Connect from the **true** branch of `Condition Valid`
   - Set:
     - Response format: JSON
     - HTTP status code: `200`
     - Response body:
       ```json
       {
         "status": "success",
         "message": "oke"
       }
       ```

6. **Add another Respond to Webhook node** named `Respond Failed`.
   - Connect from the **false** branch of `Condition Valid`
   - Set:
     - Response format: JSON
     - Response body using expression:
       ```json
       {
         "status": "failed",
         "message": "{{ $json.errors }}"
       }
       ```

7. **Add an Extract from File node** named `Extract from File`.
   - Connect from the **true** branch of `Condition Valid`
   - Operation: `PDF`
   - Configure the binary property containing the uploaded file
   - The JSON shows `={{ $json.data }}`, but in practice you should verify this carefully:
     - if the file is stored as `binary.data`, the property name should usually be `data`
   - This is one of the most important implementation details to test.

8. **Add a Code node** named `Chunk` and connect `Extract from File -> Chunk`.
   - Use code that:
     - reads extracted text from `$json.text` or `$json.data`
     - splits it every `2000` characters
     - returns one item per chunk as:
       - `{ json: { chunk } }`

9. **Add a Google Gemini node** named `Message a model` and connect `Chunk -> Message a model`.
   - Node type: `Google Gemini` from LangChain/n8n AI nodes
   - Configure credentials:
     - create or select `Google Gemini(PaLM) Api account`
     - use your Gemini API key
   - Model:
     - select `models/gemma-3-1b-it` if available in your environment
   - Add one user message with content equivalent to:
     - expert teacher/exam creator role
     - generate MCQs from `{{ $json.chunk }}`
     - exactly 4 options
     - one correct answer
     - include explanation
     - max 3 questions per chunk
     - return only valid JSON, no markdown
   - Ensure the chunk variable is inserted with expression syntax.

10. **Add a Code node** named `Cleaner` and connect `Message a model -> Cleaner`.
    - Add code that:
      - reads model output from `item.json.content.parts[0].text`
      - strips code fences
      - parses JSON
      - iterates over the resulting array
      - emits each question object as a separate item
    - Preserve fields:
      - `question`
      - `option_a`
      - `option_b`
      - `option_c`
      - `option_d`
      - `correct_answer`
      - `explanation`

11. **Add a Convert to File node** named `Convert to File` and connect `Cleaner -> Convert to File`.
    - Operation: `xlsx`
    - Header row: enabled
    - Sheet name: `Sheet 1`
    - File name expression:
      `result-n8n-{{$('Validate Parameter & PDF').first().json.fileName }}.xlsx`

12. **Add a Google Drive node** named `Upload file` and connect `Convert to File -> Upload file`.
    - Configure Google Drive OAuth2 credentials
    - Operation: upload/create file
    - Drive: `My Drive`
    - Folder: choose the target folder, here it is a folder named `n8n`
    - File name:
      `result-n8n-{{$('Validate Parameter & PDF').first().json.fileName }}.xlsx`

13. **Add a Telegram node** named `Send a document` and connect `Convert to File -> Send a document`.
    - Configure Telegram Bot credentials
    - Operation: `sendDocument`
    - Binary Data: enabled
    - Chat ID: replace `12345678` with the real target chat ID
    - Ensure the binary property used by the node matches the XLSX file binary output

14. **Add an Email Send node** named `Send email` and connect `Convert to File -> Send email`.
    - Configure SMTP credentials
    - To Email: replace `user@example.com`
    - From Email: replace `user@example.com` with a valid sender for your SMTP server
    - Subject:
      `Turn Any PDF into a Structured Question Bank in Excel using AI (MCQ + Answer Key)`
    - Body text:
      `Here attachment result`
    - Attachments property: `data`
    - Email format: text

15. **Optionally add sticky notes** to document the workflow blocks:
    - Input & validation
    - PDF extraction & chunking
    - AI generation
    - Output generation & storage
    - Delivery & notification
    - General setup instructions

16. **Configure credentials** before activating:
    - **Google Gemini / PaLM API**
      - valid API key
      - model availability checked
    - **Google Drive OAuth2**
      - access to the destination folder
    - **Telegram Bot**
      - valid bot token
      - bot allowed to message target chat
    - **SMTP**
      - authenticated sender
      - permitted `fromEmail`

17. **Test with a small text-based PDF first**.
    - Verify:
      - webhook receives binary file
      - `Extract from File` reads the correct binary property
      - Gemini returns valid JSON
      - `Cleaner` parses it successfully
      - XLSX file contains columns and rows as expected

18. **Only then activate the workflow**.

### Sub-workflow setup
This workflow does **not** use any Execute Workflow or sub-workflow nodes. No sub-workflow configuration is required.

### Input/output expectations
- **Expected input:** HTTP POST to webhook with a binary PDF file
- **Primary artifact produced:** XLSX file containing MCQ rows
- **Optional delivery outputs:** Google Drive upload, Telegram document, email attachment
- **Webhook response behavior:** immediate success/failure response, not a completion callback

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Convert PDF to MCQ Question Bank in Excel with AI (Gemini). This workflow automatically converts any PDF into a structured Multiple Choice Question (MCQ) bank using Google Gemini AI. | General workflow description |
| It extracts text from the PDF, processes it into manageable chunks, generates MCQs with answer keys and explanations, and exports the results into an Excel file. | General workflow description |
| The final file is uploaded to Google Drive and can be sent directly via Email or Telegram. | Output and delivery context |
| This workflow is ideal for educators, trainers, and content creators who want to automate question generation and save time. | Use-case context |
| How it works: 1. A PDF file is uploaded via webhook. 2. The workflow validates the file (max 5MB and file existence). 3. The PDF content is extracted and split into smaller chunks. 4. Each chunk is processed by Google Gemini to generate MCQs. 5. The AI response is cleaned and converted into structured JSON. 6. All questions are compiled and converted into an Excel file. 7. The file is uploaded to Google Drive. 8. The result is sent to the user via Email or Telegram. | General process summary |
| Setup: 1. Import the workflow into your n8n instance. 2. Configure credentials: Google Gemini API Key, Google Drive OAuth, Email (SMTP or Gmail), Telegram Bot Token. 3. Update the Gemini node. 4. Activate the workflow. 5. Send a POST request to the webhook with: file (PDF), email (optional), telegram_chat_id (optional). 6. Run the workflow and receive the generated Excel file. | Setup notes |
| Important implementation note: the current workflow does not actually use dynamic `email` or `telegram_chat_id` from the webhook request; both delivery targets are hardcoded and must be modified if dynamic recipient routing is desired. | Architecture caveat |
| Important implementation note: the success webhook response is sent before AI generation, file creation, upload, Telegram sending, and email sending finish. | Runtime behavior caveat |
| Important implementation note: the PDF extraction node configuration should be tested carefully because `Extract from File` typically expects a binary property name such as `data`, not the binary object itself. | Technical caveat |