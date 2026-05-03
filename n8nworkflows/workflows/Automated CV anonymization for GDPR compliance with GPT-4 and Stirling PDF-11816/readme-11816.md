Automated CV anonymization for GDPR compliance with GPT-4 and Stirling PDF

https://n8nworkflows.xyz/workflows/automated-cv-anonymization-for-gdpr-compliance-with-gpt-4-and-stirling-pdf-11816


# Automated CV anonymization for GDPR compliance with GPT-4 and Stirling PDF

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow receives a CV file via HTTP upload, converts it to PDF if needed, extracts text page-by-page using Stirling PDF, anonymizes personally identifiable information (PII) with GPT-4 while preserving content and the original language, generates a styled HTML CV, converts that HTML back to PDF, and returns the anonymized PDF in the webhook response.

**Target use cases:**
- GDPR-compliant CV handling in recruitment pipelines
- Automated PII redaction/anonymization prior to internal sharing
- Standardizing output as a clean, printable anonymized PDF

### 1.1 Input Reception & Format Decision
Receives an uploaded file via Webhook and checks whether it is already a PDF. If not, converts it to PDF.

### 1.2 PDF Normalization, Page Processing & Text Extraction
Normalizes binary field names, splits multi-page PDFs into per-page PDFs, decompresses if needed, converts each page PDF to text, then aggregates page texts into a single payload.

### 1.3 AI Anonymization (LLM + Structured Output)
Sends the full extracted text to a GPT-4 chain with strict anonymization and formatting rules. Uses structured output parsing with auto-fixing to enforce JSON output `{ "html": "..." }`.

### 1.4 Output Generation & Delivery
Extracts HTML from the structured output, builds a binary HTML file, converts it to PDF via Stirling PDF, and returns it as the webhook response.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Conversion Gate

**Overview:** Accepts an uploaded CV file via POST, then routes either directly to processing (if PDF) or to a Stirling PDF conversion step (if not PDF).  
**Nodes involved:** `Webhook`, `If1`, `Any file to pdf`, `Merge`  
**Sticky notes covering these nodes:**  
- “## File Input & Conversion …”  
- “# CV Anonymization Workflow …” (overall description)

#### Node: Webhook
- **Type / role:** `n8n-nodes-base.webhook` — Entry point; receives file upload.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `1faa18ed-f037-48a7-b1ff-ee9975836147`
  - **Binary property name:** `UploadCV0` (the uploaded file is stored in `$binary.UploadCV0`)
  - **Response mode:** `responseNode` (workflow must end with `Respond to Webhook`)
- **Inputs/Outputs:** No inputs; output goes to `If1`.
- **Edge cases / failures:**
  - Missing binary upload -> downstream nodes fail when reading `$binary.UploadCV0`.
  - Large files may exceed n8n limits or reverse proxy limits.
  - Wrong content-type/multipart encoding can cause empty binary.

#### Node: If1
- **Type / role:** `n8n-nodes-base.if` — Branches depending on uploaded file MIME type.
- **Condition logic (interpreted):**
  - Checks **`$binary.UploadCV0.mimeType != "application/pdf"`**
  - **True branch (not a PDF)** → `Any file to pdf`
  - **False branch (already a PDF)** → `Merge` (second input)
- **Inputs/Outputs:**
  - Input: from `Webhook`
  - Output 0: to `Any file to pdf`
  - Output 1: to `Merge`
- **Edge cases / failures:**
  - Some PDFs might come with non-standard MIME types (e.g., `application/x-pdf`), causing unnecessary conversion.
  - If `UploadCV0` is missing, expression resolution fails.

#### Node: Any file to pdf
- **Type / role:** `n8n-nodes-base.httpRequest` — Converts non-PDF input to PDF using Stirling PDF.
- **Endpoint:** `POST http://stirlingpdf:8080/api/v1/convert/file/pdf`
- **Body (multipart/form-data):**
  - `fileInput`: binary from field `UploadCV0`
- **Headers:**
  - `accept: */*`
  - (Per sticky note: **Authorization header required** in real deployment.)
- **Outputs:** To `Merge`.
- **Edge cases / failures:**
  - Stirling PDF unreachable / DNS mismatch (`stirlingpdf` host must resolve in your network).
  - Auth missing/invalid (401/403).
  - Unsupported input format may produce 4xx/5xx.
  - Response handling: node options are default (not explicitly “file” here); ensure it actually returns binary PDF as expected.

#### Node: Merge
- **Type / role:** `n8n-nodes-base.merge` — Joins the two branches so downstream receives exactly one PDF binary stream.
- **Configuration choices:** Default merge behavior (no explicit mode shown).
- **Inputs/Outputs:**
  - Input 1: from `Any file to pdf` (converted PDF)
  - Input 2: from `If1` false branch (original PDF)
  - Output: to `Code in JavaScript2`
- **Edge cases / failures:**
  - If merge mode expects both inputs but only one arrives, execution could stall or output nothing depending on n8n merge settings/version behavior. Validate merge mode in UI (commonly “Pass-through” is safest for this pattern).

---

### Block 2 — PDF Processing & Text Extraction

**Overview:** Ensures the PDF binary is available under `binary.data`, splits into pages, ensures page PDFs are decompressed/normalized, extracts text per page, and aggregates all pages into a single array for AI processing.  
**Nodes involved:** `Code in JavaScript2`, `Split PDF Pages1`, `Compression`, `Code in JavaScript`, `PDF → Text1`, `Code in JavaScript1`, `Aggregate1`  
**Sticky notes covering these nodes:**  
- “## PDF Processing & Text Extraction …”  
- “⚠️ STIRLING PDF AUTH REQUIRED …” (applies to Stirling HTTP nodes, see below)

#### Node: Code in JavaScript2
- **Type / role:** `n8n-nodes-base.code` — Normalizes binary field name to `data`.
- **Key logic:**
  - If `item.binary.data` exists: keep item unchanged.
  - Else, if any binary field exists (e.g., `file_0`, `UploadCV0`): rename the first binary key to `data`.
- **Inputs/Outputs:** Input from `Merge`; output to `Split PDF Pages1`.
- **Edge cases / failures:**
  - If the first binary key is not the intended PDF (unlikely here but possible), the wrong content is sent to Stirling.
  - If there is no binary at all, it passes through and next node fails.

#### Node: Split PDF Pages1
- **Type / role:** `n8n-nodes-base.httpRequest` — Splits PDF into individual pages via Stirling PDF.
- **Endpoint:** `POST http://stirlingpdf:8080/api/v1/general/split-pages`
- **Response:** Configured as **file** (binary output).
- **Body (multipart/form-data):**
  - `fileInput`: binary field `data`
  - `pages`: `"all"`
- **Headers:**
  - `accept: */*`
  - (Authorization required per sticky note.)
- **Outputs:** To `Compression`.
- **Edge cases / failures:**
  - Auth missing/invalid (401/403).
  - Large PDFs produce large multi-file responses; may stress memory/storage.
  - Output format from Stirling for split pages may be an archive/binary bundle; downstream assumes it can be handled.

#### Node: Compression
- **Type / role:** `n8n-nodes-base.compression` — Likely decompresses/unzips output from page splitting (common pattern: split-pages returns a zip).
- **Configuration:** Empty parameters (defaults). In practice you must ensure it is set to the correct operation in UI (e.g., “Extract”).
- **Inputs/Outputs:** From `Split PDF Pages1` to `Code in JavaScript`.
- **Edge cases / failures:**
  - If Stirling returns a non-archive format, extraction fails.
  - Wrong compression operation leads to missing per-page binaries.

#### Node: Code in JavaScript
- **Type / role:** `n8n-nodes-base.code` — Prepares per-page binary items for text extraction.
- **Key logic:**
  - Iterates over all items.
  - Requires `item.binary.file_0` and outputs items with:
    - `json.pageNumber = i+1`
    - `json.fileName` from binary filename or `page_i.pdf`
    - `binary.file_0` preserved
- **Inputs/Outputs:** From `Compression` to `PDF → Text1`.
- **Edge cases / failures:**
  - If decompression outputs a different binary key than `file_0`, this code drops pages (no results).
  - If pages are not ordered as expected, page numbers may be wrong.

#### Node: PDF → Text1
- **Type / role:** `n8n-nodes-base.httpRequest` — Converts each page PDF to a text file via Stirling PDF.
- **Endpoint:** `POST http://stirlingpdf:8080/api/v1/convert/pdf/text`
- **Response:** Configured as **file** (binary output).
- **Body (multipart/form-data):**
  - `fileInput`: binary `file_0`
  - `outputFormat`: `txt`
- **Headers:**
  - `accept: */*`
  - (Authorization required per sticky note.)
- **Outputs:** To `Code in JavaScript1`.
- **Edge cases / failures:**
  - OCR is not implied; scanned/image PDFs may yield empty text unless Stirling is configured for OCR.
  - Auth/network errors.
  - Per-page requests can be slow; consider concurrency limits.

#### Node: Code in JavaScript1
- **Type / role:** `n8n-nodes-base.code` — Extracts text from the response (robustly) and normalizes to `{ json: { text, pageNumber, filename } }`.
- **Key logic:**
  - Tries multiple locations for text:
    1. `item.json.text`
    2. `item.json.response`
    3. `item.json.output`
    4. Converts from binary `data` (buffer/base64/object-with-data)
- **Inputs/Outputs:** From `PDF → Text1` to `Aggregate1`.
- **Edge cases / failures:**
  - If Stirling returns the file under a different binary key (not `data`), the “binary” extraction path may not run.
  - If text encoding is not UTF-8, output may be garbled.
  - Debug `console.log` statements help in execution logs but can be noisy at scale.

#### Node: Aggregate1
- **Type / role:** `n8n-nodes-base.aggregate` — Aggregates all items into a single item holding an array of pages.
- **Configuration:**
  - Mode: “Aggregate All Item Data”
  - Destination field: `pages`
- **Output shape (interpreted):**
  - One item with `$json.pages = [{text, pageNumber, filename}, ...]`
- **Inputs/Outputs:** From `Code in JavaScript1` to `Basic LLM Chain`.
- **Edge cases / failures:**
  - If upstream produced zero page items, pages array is empty and LLM receives empty input.
  - Very large CVs may exceed prompt limits when concatenated.

---

### Block 3 — AI Anonymization (GPT + Output Parsing)

**Overview:** Concatenates all page texts and instructs GPT-4 to anonymize PII while preserving original language and content fidelity, returning **only** JSON with an HTML string. Structured parsing and auto-fixing enforce output correctness.  
**Nodes involved:** `Basic LLM Chain`, `OpenAI Chat Model`, `Auto-fixing Output Parser`, `OpenAI Chat Model1`, `Structured Output Parser`, `OpenAI Chat Model2`  
**Sticky notes covering these nodes:**  
- “## AI Anonymization …”  
- “# CV Anonymization Workflow …” (customization notes)

#### Node: Basic LLM Chain
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — Builds and runs an LLM chain using provided system instructions and an input text.
- **Key configuration:**
  - **Input text expression:**  
    `{{ $json.pages.map(p => p.text).join('\n\n') }}`
  - **Prompt type:** “define”
  - **System message:** Extensive anonymization + HTML formatting requirements, including:
    - Keep original language; do not translate.
    - Replace names/emails/phones/addresses/cities/company/university/client names with localized placeholders.
    - Preserve job titles, skills, dates, achievements, descriptions word-for-word.
    - Output must be **only** JSON: `{"html":"..."}`
    - HTML must be full HTML5 doc with embedded CSS, max width 800px, RTL support when needed.
  - **Has output parser:** enabled (uses `Auto-fixing Output Parser` connection).
- **Model connection:** Uses `OpenAI Chat Model` on `ai_languageModel` input.
- **Output parser connection:** Uses `Auto-fixing Output Parser` on `ai_outputParser`.
- **Inputs/Outputs:** Input from `Aggregate1`, output to `Code1`.
- **Edge cases / failures:**
  - Token limits: concatenated pages may exceed model context.
  - If extracted text includes tables/columns, LLM may restructure content.
  - Strict placeholder localization requirement may be inconsistently applied by the model in rare cases.
  - If the output is not valid JSON, structured parser relies on auto-fix (may still fail).

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Primary LLM used by the chain.
- **Configuration:**
  - Model: `gpt-4-turbo`
  - `maxTokens: 3000`, `temperature: 0.3`
- **Inputs/Outputs:** Connected to `Basic LLM Chain` (language model).
- **Edge cases / failures:**
  - Credential missing/invalid.
  - Model availability or deprecation.
  - Rate limits (429), timeouts.

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Defines the JSON schema the LLM must return.
- **Configuration:**
  - `autoFix: true`
  - Manual JSON schema requiring:
    - object with **only** key `html` (string)
    - `additionalProperties: false`
- **Connections:** Feeds into `Auto-fixing Output Parser` via `ai_outputParser`.
- **Edge cases / failures:**
  - If model returns extra keys, parser rejects unless auto-fix can remove them.
  - If HTML is not a string (e.g., object), parser fails.

#### Node: Auto-fixing Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserAutofixing` — Attempts to correct malformed outputs to match schema.
- **Configuration:** Default options.
- **Connections:**
  - Input parser: from `Structured Output Parser` (`ai_outputParser`)
  - Fixing model: from `OpenAI Chat Model1` (`ai_languageModel`)
  - Used by: `Basic LLM Chain` (`ai_outputParser`)
- **Edge cases / failures:**
  - If output is severely malformed, auto-fix may not recover.
  - Additional cost/latency: may invoke an LLM to repair output.

#### Node: OpenAI Chat Model1
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Model used by the auto-fixing parser.
- **Configuration:** `gpt-4.1-mini` (selected from list), default options.
- **Edge cases / failures:** Same as other OpenAI nodes (auth, rate limit).

#### Node: OpenAI Chat Model2
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — Connected to the structured parser as its language model.
- **Configuration:** `gpt-4.1-mini`, default options.
- **Note:** This is part of the parsing subsystem (not the main anonymization generation).
- **Edge cases / failures:** Same as other OpenAI nodes.

---

### Block 4 — Output Generation & Response Delivery

**Overview:** Extracts the HTML string from the chain output, builds it as a binary `.html` file, converts to PDF via Stirling, then returns the resulting PDF to the webhook caller.  
**Nodes involved:** `Code1`, `Any File → PDF2`, `Respond to Webhook`  
**Sticky notes covering these nodes:**  
- “## Output Generation …”  
- “⚠️ STIRLING PDF AUTH REQUIRED …” (applies to `Any File → PDF2`)

#### Node: Code1
- **Type / role:** `n8n-nodes-base.code` — Extracts HTML from structured output and creates a binary HTML file.
- **Key logic:**
  - Reads first input item: expects `item.output.html`
  - Throws error if missing: `"No HTML found..."`
  - Unescapes sequences:
    - `\\\"` → `"`
    - `\\n` → newline
  - Creates binary `data`:
    - MIME: `text/html`
    - Filename: `anonymized_cv.html`
- **Inputs/Outputs:** From `Basic LLM Chain` to `Any File → PDF2`.
- **Edge cases / failures:**
  - If chain output structure differs (e.g., `item.json.html` instead of `item.output.html`), this node fails.
  - Unescape logic may incorrectly handle already-unescaped strings (double-unescaping risk).
  - HTML may include characters requiring different handling (encoding issues are rare with UTF-8).

#### Node: Any File → PDF2
- **Type / role:** `n8n-nodes-base.httpRequest` — Converts generated HTML file to PDF via Stirling PDF.
- **Endpoint:** `POST http://root-stirlingpdf-1:8080/api/v1/convert/file/pdf`
  - Note the hostname differs from earlier Stirling calls (`root-stirlingpdf-1` vs `stirlingpdf`).
- **Response:** Configured as **file** (binary output).
- **Body (multipart/form-data):**
  - `fileInput`: binary `data` (the HTML created by `Code1`)
  - `outputFormat`: `pdf`
- **Headers:**
  - `accept: */*`
  - (Authorization required per sticky note.)
- **Outputs:** To `Respond to Webhook`.
- **Edge cases / failures:**
  - Hostname mismatch can break in different environments (Docker compose service name vs container name).
  - If Stirling’s HTML-to-PDF conversion is not enabled or requires additional dependencies, it may fail.
  - Auth errors, timeouts on complex HTML/CSS.

#### Node: Respond to Webhook
- **Type / role:** `n8n-nodes-base.respondToWebhook` — Returns final PDF binary to the caller.
- **Configuration:** `respondWith: binary`
- **Inputs/Outputs:** Input from `Any File → PDF2`; terminates workflow response.
- **Edge cases / failures:**
  - If upstream didn’t return binary in the expected property, response may be empty or error.
  - Consider setting content-disposition/filename headers if needed (not configured here).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Receives CV upload (binary) | — | If1 | # CV Anonymization Workflow…; ## File Input & Conversion… |
| If1 | n8n-nodes-base.if | Routes based on MIME type (PDF vs non-PDF) | Webhook | Any file to pdf; Merge | # CV Anonymization Workflow…; ## File Input & Conversion… |
| Any file to pdf | n8n-nodes-base.httpRequest | Convert non-PDF file to PDF (Stirling) | If1 (true) | Merge | # CV Anonymization Workflow…; ## File Input & Conversion…; ⚠️ STIRLING PDF AUTH REQUIRED… |
| Merge | n8n-nodes-base.merge | Merge PDF from either branch | If1 (false), Any file to pdf | Code in JavaScript2 | # CV Anonymization Workflow…; ## File Input & Conversion… |
| Code in JavaScript2 | n8n-nodes-base.code | Normalize binary key to `data` | Merge | Split PDF Pages1 | ## PDF Processing & Text Extraction… |
| Split PDF Pages1 | n8n-nodes-base.httpRequest | Split PDF into pages (Stirling) | Code in JavaScript2 | Compression | ## PDF Processing & Text Extraction…; ⚠️ STIRLING PDF AUTH REQUIRED… |
| Compression | n8n-nodes-base.compression | Decompress split-pages output (likely ZIP) | Split PDF Pages1 | Code in JavaScript | ## PDF Processing & Text Extraction… |
| Code in JavaScript | n8n-nodes-base.code | Map page binaries to `file_0` + metadata | Compression | PDF → Text1 | ## PDF Processing & Text Extraction… |
| PDF → Text1 | n8n-nodes-base.httpRequest | Convert each page PDF to text (Stirling) | Code in JavaScript | Code in JavaScript1 | ## PDF Processing & Text Extraction…; ⚠️ STIRLING PDF AUTH REQUIRED… |
| Code in JavaScript1 | n8n-nodes-base.code | Extract text from response/binary and normalize | PDF → Text1 | Aggregate1 | ## PDF Processing & Text Extraction… |
| Aggregate1 | n8n-nodes-base.aggregate | Aggregate pages into `$json.pages` array | Code in JavaScript1 | Basic LLM Chain | ## PDF Processing & Text Extraction… |
| Basic LLM Chain | @n8n/n8n-nodes-langchain.chainLlm | GPT anonymization + HTML generation | Aggregate1 | Code1 | # CV Anonymization Workflow…; ## AI Anonymization… |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Main GPT model for anonymization | — | Basic LLM Chain (ai_languageModel) | # CV Anonymization Workflow…; ## AI Anonymization… |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce `{html: string}` schema | — | Auto-fixing Output Parser (ai_outputParser) | ## AI Anonymization… |
| Auto-fixing Output Parser | @n8n/n8n-nodes-langchain.outputParserAutofixing | Fix malformed JSON to match schema | Structured Output Parser | Basic LLM Chain (ai_outputParser) | ## AI Anonymization… |
| OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM used by auto-fixing parser | — | Auto-fixing Output Parser (ai_languageModel) | ## AI Anonymization… |
| OpenAI Chat Model2 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM used by structured parser | — | Structured Output Parser (ai_languageModel) | ## AI Anonymization… |
| Code1 | n8n-nodes-base.code | Extract HTML and create binary HTML file | Basic LLM Chain | Any File → PDF2 | ## Output Generation… |
| Any File → PDF2 | n8n-nodes-base.httpRequest | Convert HTML to PDF (Stirling) | Code1 | Respond to Webhook | ## Output Generation…; ⚠️ STIRLING PDF AUTH REQUIRED… |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Return anonymized PDF to caller | Any File → PDF2 | — | ## Output Generation… |
| CV Anonymization Workflow | n8n-nodes-base.stickyNote | Documentation note | — | — | (This node is itself the note content shown) |
| File Input & Conversion | n8n-nodes-base.stickyNote | Documentation note | — | — | (This node is itself the note content shown) |
| PDF Processing & Text Extraction | n8n-nodes-base.stickyNote | Documentation note | — | — | (This node is itself the note content shown) |
| AI Anonymization1 | n8n-nodes-base.stickyNote | Documentation note | — | — | (This node is itself the note content shown) |
| Output Generation | n8n-nodes-base.stickyNote | Documentation note | — | — | (This node is itself the note content shown) |
| Stirling PDF Auth Required | n8n-nodes-base.stickyNote | Documentation note | — | — | (This node is itself the note content shown) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named **“CV Anonymization”** (inactive by default is fine).
2. **Add Webhook node**
   - Type: *Webhook*
   - Method: **POST**
   - Response mode: **Respond to Webhook node**
   - Options → **Binary Property Name:** `UploadCV0`
   - Copy the generated webhook URL for testing.
3. **Add IF node** (“If1”)
   - Condition: **String**
   - Left value: `{{$binary.UploadCV0.mimeType}}`
   - Operation: **not equals**
   - Right value: `application/pdf`
   - Connect: `Webhook → If1`
4. **Add HTTP Request node** (“Any file to pdf”)
   - Method: **POST**
   - URL: `http://stirlingpdf:8080/api/v1/convert/file/pdf`
   - Content Type: **multipart/form-data**
   - Send Headers: enabled
   - Header: `accept: */*`
   - Body parameter:
     - Name: `fileInput`
     - Type: **Binary**
     - Binary property: `UploadCV0`
   - (If Stirling is protected) Add header `Authorization: Bearer <token>` or the required scheme.
   - Connect: `If1 (true) → Any file to pdf`
5. **Add Merge node** (“Merge”)
   - Set merge mode in UI to a *pass-through / first available input* pattern (recommended), so it works when only one branch emits.
   - Connect:
     - `Any file to pdf → Merge (Input 1)`
     - `If1 (false) → Merge (Input 2)`
6. **Add Code node** (“Code in JavaScript2”)
   - Paste logic to rename the first binary key to `binary.data` if needed (as in workflow).
   - Connect: `Merge → Code in JavaScript2`
7. **Add HTTP Request node** (“Split PDF Pages1”)
   - POST `http://stirlingpdf:8080/api/v1/general/split-pages`
   - Response: set to **File**
   - multipart/form-data body:
     - `fileInput` (Binary): `data`
     - `pages` (Text): `all`
   - Header: `accept: */*` (+ Authorization if required)
   - Connect: `Code in JavaScript2 → Split PDF Pages1`
8. **Add Compression node** (“Compression”)
   - Configure operation to **Extract/Unzip** the response from split-pages (ensure it outputs one item per extracted file).
   - Connect: `Split PDF Pages1 → Compression`
9. **Add Code node** (“Code in JavaScript”)
   - Paste logic that expects each item has `binary.file_0` and adds `pageNumber` and `fileName`.
   - If your unzip outputs a different binary key, update the code accordingly.
   - Connect: `Compression → Code in JavaScript`
10. **Add HTTP Request node** (“PDF → Text1”)
    - POST `http://stirlingpdf:8080/api/v1/convert/pdf/text`
    - Response: **File**
    - multipart/form-data body:
      - `fileInput` (Binary): `file_0`
      - `outputFormat` (Text): `txt`
    - Header: `accept: */*` (+ Authorization if required)
    - Connect: `Code in JavaScript → PDF → Text1`
11. **Add Code node** (“Code in JavaScript1”)
    - Paste robust extraction logic to read text from json fields or binary `data` and output `{text, pageNumber, filename}`.
    - Connect: `PDF → Text1 → Code in JavaScript1`
12. **Add Aggregate node** (“Aggregate1”)
    - Operation: **Aggregate All Item Data**
    - Destination field: `pages`
    - Connect: `Code in JavaScript1 → Aggregate1`
13. **Add OpenAI Chat Model node** (“OpenAI Chat Model”)
    - Credentials: configure OpenAI API key in n8n credentials
    - Model: `gpt-4-turbo`
    - Options: `maxTokens=3000`, `temperature=0.3`
14. **Add Structured Output Parser node** (“Structured Output Parser”)
    - Schema (manual):
      - object with required string `html`
      - `additionalProperties: false`
    - Enable **Auto-fix**
15. **Add OpenAI Chat Model node** (“OpenAI Chat Model2”)
    - Model: `gpt-4.1-mini`
    - Connect it to **Structured Output Parser** as its language model (ai_languageModel).
16. **Add Auto-fixing Output Parser node** (“Auto-fixing Output Parser”)
    - Default options
    - Connect: `Structured Output Parser → Auto-fixing Output Parser` (ai_outputParser)
17. **Add OpenAI Chat Model node** (“OpenAI Chat Model1”)
    - Model: `gpt-4.1-mini`
    - Connect it to **Auto-fixing Output Parser** as its language model (ai_languageModel).
18. **Add Basic LLM Chain node** (“Basic LLM Chain”)
    - Input text expression: `{{$json.pages.map(p => p.text).join('\n\n')}}`
    - Add the long **system message** defining anonymization + HTML + JSON-only output constraints (as in workflow).
    - Enable output parser usage and connect:
      - `OpenAI Chat Model → Basic LLM Chain` (ai_languageModel)
      - `Auto-fixing Output Parser → Basic LLM Chain` (ai_outputParser)
    - Connect: `Aggregate1 → Basic LLM Chain`
19. **Add Code node** (“Code1”)
    - Extract `item.output.html`, unescape, create binary HTML under `binary.data` with filename `anonymized_cv.html`.
    - Connect: `Basic LLM Chain → Code1`
20. **Add HTTP Request node** (“Any File → PDF2”)
    - POST `http://root-stirlingpdf-1:8080/api/v1/convert/file/pdf` (or align host with your Stirling service)
    - Response: **File**
    - multipart/form-data:
      - `fileInput` (Binary): `data`
      - `outputFormat`: `pdf`
    - Header: `accept: */*` (+ Authorization if required)
    - Connect: `Code1 → Any File → PDF2`
21. **Add Respond to Webhook node**
    - Respond with: **Binary**
    - Connect: `Any File → PDF2 → Respond to Webhook`
22. **Credentials / environment checks**
    - OpenAI credentials must be set for all three OpenAI nodes.
    - Stirling PDF endpoints must be reachable from n8n (DNS/service name) and authenticated if required.
23. **Test**
    - Send a multipart/form-data POST to the webhook with file field mapped to `UploadCV0`.
    - Verify the response is a PDF and that PII is replaced with placeholders while language is preserved.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically removes personally identifiable information from CVs while preserving skills, experience, and achievements. | From sticky note “CV Anonymization Workflow” |
| Setup steps: configure OpenAI credentials for all three OpenAI Chat Model nodes. | From sticky note “CV Anonymization Workflow” |
| Add Stirling PDF authorization headers to HTTP Request nodes (Any file to pdf, Split PDF Pages1, PDF → Text1, Any File → PDF2). | From sticky note “CV Anonymization Workflow” and “⚠️ STIRLING PDF AUTH REQUIRED” |
| Customization: adjust anonymization rules and HTML styling requirements in the Basic LLM Chain system message; change GPT model versions for cost/quality tradeoffs. | From sticky note “CV Anonymization Workflow” |
| ⚠️ STIRLING PDF AUTH REQUIRED: All 4 Stirling PDF HTTP nodes need Authorization headers configured before this workflow will function. | From sticky note “Stirling PDF Auth Required” |
| Hostname inconsistency: Stirling is referenced as `stirlingpdf` in some nodes and `root-stirlingpdf-1` in the final conversion node. Align these for your deployment. | Observed from node URLs (integration consideration) |