Generate structured exams from PDFs with OpenAI GPT-4o and HTML to PDF

https://n8nworkflows.xyz/workflows/generate-structured-exams-from-pdfs-with-openai-gpt-4o-and-html-to-pdf-13599


# Generate structured exams from PDFs with OpenAI GPT-4o and HTML to PDF

## 1. Workflow Overview

**Purpose:**  
This workflow (“ExamForge AI – PDF to Structured Exam Generator”) accepts a **PDF upload via Webhook**, validates request parameters and file size, extracts and cleans the PDF text, checks approximate token length to prevent model overflow, uses **OpenAI GPT-4o-mini** to generate **structured exam content (MCQ + essay) as JSON**, then formats the result into **two HTML documents** (Exam + Answer Key), converts both to **PDF via HTMLCSS to PDF**, and finally sends **download links via Telegram**. It also returns structured error responses to the webhook caller when validation fails.

**Target use cases:**
- Generating assessment papers from course handouts or lecture PDFs (text-based PDFs)
- Quickly producing MCQs, essay prompts, and answer guides with configurable difficulty/language
- Automated delivery of exam PDFs to a Telegram chat

### 1.1 Input Reception & Request Validation
Receives multipart/form-data (PDF + parameters), validates required fields and constraints (counts, difficulty, language), and enforces a max file size (5MB).

### 1.2 PDF Text Extraction, Cleaning & Length Safety Gate
Extracts text from the uploaded PDF, cleans it (normalize whitespace/remove weird characters), estimates token usage, and blocks overly long content.

### 1.3 AI Generation (Structured JSON)
Sends cleaned text + user parameters to OpenAI, requesting **JSON-only** output with a fixed schema.

### 1.4 JSON Parsing, HTML Composition, PDF Rendering & Delivery
Parses the model output JSON, renders exam and answer key as HTML, converts each to PDF, and sends links via Telegram.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Request Validation
**Overview:**  
Accepts a PDF and generation parameters over HTTP, then validates file size and required fields. If invalid, the workflow responds immediately with structured error JSON.

**Nodes involved:**
- Webhook
- Validate Parameter & PDF
- Condition Valid
- Respond Valid Parameter

#### Node: **Webhook**
- **Type / Role:** `n8n-nodes-base.webhook` — Entry point (HTTP POST).
- **Key configuration:**
  - Method: **POST**
  - Response mode: **Respond via “Respond to Webhook” node** (`responseMode: responseNode`)
  - Expects multipart/form-data with binary file in `binary.data` and fields in `body.*`.
- **Outputs:** Sends request data to **Validate Parameter & PDF**.
- **Edge cases / failures:**
  - Wrong method/content-type (no binary file).
  - Large uploads can be blocked by n8n or reverse proxy limits before node logic runs.
  - If a later Respond node isn’t hit on some branches, caller may hang (ensure all error branches respond).

#### Node: **Validate Parameter & PDF**
- **Type / Role:** `n8n-nodes-base.code` — Validates parameters and file size.
- **What it checks (interpreted from code):**
  - Reads:
    - `mcq_count`, `essay_count`, `difficulty`, `language` from `{{$input.first().json.body...}}`
    - file size from `{{$input.first().binary.data.fileSize}}`
  - File size rule: **must be < 5MB** (`5242880` bytes).  
    The code attempts to parse `fileSize` which may be formatted as “123kb/1.2mb” or raw bytes.
  - Required fields: mcq_count, essay_count, difficulty, language
  - Numeric checks: mcq_count and essay_count must be numeric
  - Difficulty allowed values: `easy | medium | hard`
  - Upper bounds:
    - `mcq_count` max 50
    - `essay_count` intended max 20, but the code checks `> 50` while the message says “maximum is 20” (**logic/message mismatch**).
- **Output:** One item:
  - `isValid` boolean
  - `errors` array
  - parameters echoed back
  - `data` set to the binary file object (`$input.first().binary.data`) for downstream PDF extraction
- **Connections:** Output → **Condition Valid**
- **Edge cases / failures:**
  - If `binary.data.fileSize` is undefined or in an unexpected format, parsing may yield `NaN`, bypassing size enforcement.
  - `if (!mcq)` treats `0` as missing (fine here, since 0 questions likely invalid anyway).
  - The essay max rule likely needs fixing to `if (essay && essay > 20)`.

#### Node: **Condition Valid**
- **Type / Role:** `n8n-nodes-base.if` — Routes valid vs invalid requests.
- **Condition:** `{{$json.isValid}} == true`
- **True branch:** → **Extract from File**  
- **False branch:** → **Respond Valid Parameter**
- **Edge cases:**
  - If `isValid` missing/non-boolean, strict validation may fail; current code always sets it.

#### Node: **Respond Valid Parameter**
- **Type / Role:** `n8n-nodes-base.respondToWebhook` — Returns parameter validation error to caller.
- **Response:** JSON:
  - `status: "error"`
  - `message: "Invalid parameters"`
  - `details: {{$json.errors}}`
- **Connections:** Terminal response on invalid parameter path.
- **Edge cases:**
  - Because Webhook is set to respond via response node, you must ensure this node is reached for invalid requests (it is).

---

### Block 2 — PDF Text Extraction, Cleaning & Length Safety Gate
**Overview:**  
Extracts text from the validated PDF, cleans it, estimates token usage, and blocks documents that are too long for safe prompting.

**Nodes involved:**
- Extract from File
- Clean Text
- Length Estimation Layer
- Validate Token > 8.000
- Respond to Webhook

#### Node: **Extract from File**
- **Type / Role:** `n8n-nodes-base.extractFromFile` — Extracts text from a PDF.
- **Key configuration:**
  - Operation: **pdf**
  - Binary property name: `={{ $json.data }}`  
    (It is trying to point the extractor to the binary content passed forward as `data`.)
- **Input:** Output of Validate Parameter & PDF (valid branch) containing `data`.
- **Output:** Extracted content, typically in `json.text` (depends on node’s output format/version).
- **Edge cases / failures:**
  - **Potential misconfiguration:** Extract From File typically expects the *name of a binary property* (e.g., `"data"`), not the binary object itself. If `binaryPropertyName` must be a string, set it to `"data"` and ensure the binary is at `binary.data`.  
  - Encrypted/scanned PDFs (image-only) will not yield meaningful text without OCR.
  - Very large PDFs can cause performance/timeouts.

#### Node: **Clean Text**
- **Type / Role:** `n8n-nodes-base.code` — Normalizes extracted text.
- **Cleaning steps:**
  - Source text: `$json.text || $json.data || ""`
  - Removes non-printable chars (keeps ASCII printable + newlines)
  - Collapses excessive newlines and whitespace
  - Trims
  - Hard truncation: **max 15,000 characters**
- **Output:** `cleaned_text`
- **Connections:** → **Length Estimation Layer**
- **Edge cases:**
  - ASCII-only filter may remove non-Latin characters (problem if `language` is non-English and PDF contains unicode). Consider relaxing the regex for unicode support.
  - Truncation can cut content mid-sentence and reduce exam quality.

#### Node: **Length Estimation Layer**
- **Type / Role:** `n8n-nodes-base.code` — Estimates token usage and gates oversized documents.
- **Logic:**
  - `estimatedTokens = ceil(text.length / 4)` (rough heuristic)
  - If `estimatedTokens > 8000`: returns `{ error: true, message: ..., estimated_tokens }`
  - Else returns `{ error: false, cleaned_text, estimated_tokens }`
- **Connections:** → **Validate Token > 8.000**
- **Edge cases:**
  - Token heuristic is approximate; real token count depends on language/content.
  - The node name suggests “> 8.000”; the threshold is actually 8000 tokens.

#### Node: **Validate Token > 8.000**
- **Type / Role:** `n8n-nodes-base.if` — Branches on the length gate.
- **Condition:** `{{$json.error}} == true`
- **True branch (too long):** → **Respond to Webhook**
- **False branch (ok):** → **Message a model**
- **Edge cases:**
  - If upstream didn’t set `error`, strict boolean compare could behave unexpectedly.

#### Node: **Respond to Webhook**
- **Type / Role:** `n8n-nodes-base.respondToWebhook` — Returns “too long / invalid size upload” error.
- **Response JSON:**
  - `status: "error"`
  - `message: "Invalid size upload"`
  - `details: "Document too long. Please upload a shorter PDF (recommended max 15–20 pages)."`
- **Note:** The message says “Invalid size upload” but this branch is about **token length**, not file bytes.
- **Edge cases:**
  - If you also want a distinct error for >5MB, ensure that branch responds separately (currently handled by parameter validation).

---

### Block 3 — AI Generation (Structured JSON)
**Overview:**  
Prompts OpenAI with cleaned PDF text and requested parameters to generate MCQs and essay questions in a strict JSON schema.

**Nodes involved:**
- Message a model
- Parse JSON

#### Node: **Message a model**
- **Type / Role:** `@n8n/n8n-nodes-langchain.openAi` — Sends prompt to OpenAI chat model.
- **Model:** `gpt-4o-mini`
- **Prompt strategy (interpreted):**
  - System-style instruction: “expert assessment designer”
  - Enforces **JSON-only** output with schema:
    - `mcq[]` items with `question`, `options{A..D}`, `answer`, `difficulty`
    - `essay[]` items with `question`, `expected_points`, `difficulty`
  - Dynamic user prompt uses:
    - `{{$json.cleaned_text}}` from the length gate output
    - `{{$('Webhook').item.json.body.mcq_count}}`, `essay_count`, `difficulty`, `language` from the original request
- **Connections:** → **Parse JSON**
- **Credentials:** OpenAI API credential (“OpenAi account”).
- **Edge cases / failures:**
  - Model may still wrap JSON in code fences; handled downstream.
  - Output may be invalid JSON (trailing commas, smart quotes, extra text).
  - Prompt uses curly quotes in the schema (`“mcq”` etc.); this can confuse models and produce invalid JSON. Prefer plain quotes (`"`).
  - If the workflow receives multiple items, referencing `$('Webhook').item` assumes the first webhook item; usually fine for webhook-triggered single-item runs.

#### Node: **Parse JSON**
- **Type / Role:** `n8n-nodes-base.code` — Extracts and parses JSON from model output.
- **Key logic:**
  - Reads: `$input.first().json.output[0].content[0].text` (LangChain node output structure)
  - Strips ```json and ``` fences
  - `JSON.parse(raw)`; throws error if invalid
  - Returns parsed object as the node JSON
- **Connections:** → **Format Exam Text**
- **Edge cases / failures:**
  - If OpenAI node output shape changes (node version/model), path may break. Consider defensive extraction (fallbacks).
  - If model returns smart quotes or non-JSON, this hard-fails the execution.

---

### Block 4 — HTML Composition, PDF Rendering & Delivery
**Overview:**  
Transforms structured exam JSON into HTML for the exam and answer key, converts both HTML documents to PDFs, then sends the resulting PDF URLs to a Telegram chat.

**Nodes involved:**
- Format Exam Text
- Convert HTML to PDF - Exam
- Convert HTML to PDF - Answer
- Send a text message (Telegram)
- Send a text message1 (Telegram)

#### Node: **Format Exam Text**
- **Type / Role:** `n8n-nodes-base.code` — Builds `exam_html` and `answer_key_html`.
- **HTML output behavior:**
  - Exam:
    - Title: “Exam Paper”
    - MCQ section: numbered questions + A–D options
    - Essay section: numbered questions
  - Answer key:
    - MCQ answers list (number → letter)
    - Essay guide using `expected_points`
- **Connections:** Splits to both:
  - → **Convert HTML to PDF - Exam**
  - → **Convert HTML to PDF - Answer**
- **Edge cases:**
  - No CSS included; PDFs may look plain.
  - HTML is not escaped—if model outputs HTML-like text, it could affect rendering (usually low risk, but possible).
  - Assumes each MCQ has `options.A-D` and `answer`; missing fields become empty strings.

#### Node: **Convert HTML to PDF - Exam**
- **Type / Role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf` — Renders HTML to PDF via HTMLCSS.toPDF service.
- **Input:** `html_content = {{$json.exam_html}}`
- **Output:** Typically includes a `pdf_url` (as referenced downstream).
- **Connections:** → **Send a text message**
- **Credentials:** HTMLCSS.toPDF API credential (“HTML to PDF account”).
- **Edge cases / failures:**
  - External API timeouts or rate limits.
  - If the service returns a different field name than `pdf_url`, Telegram message will be wrong.

#### Node: **Convert HTML to PDF - Answer**
- **Type / Role:** `n8n-nodes-htmlcsstopdf.htmlcsstopdf`
- **Input:** `html_content = {{$json.answer_key_html}}`
- **Connections:** → **Send a text message1**
- **Credentials:** Same as above.
- **Edge cases:** Same as above.

#### Node: **Send a text message**
- **Type / Role:** `n8n-nodes-base.telegram` — Sends Exam PDF link to Telegram.
- **Config:**
  - Chat ID: `123456789` (static)
  - Text: `Download Exam Here : {{$json.pdf_url}}`
- **Credentials:** Telegram bot credential (“EngineN8N”).
- **Edge cases:**
  - Wrong chat ID or bot not allowed in that chat.
  - If `pdf_url` missing, message contains blank link.

#### Node: **Send a text message1**
- **Type / Role:** `n8n-nodes-base.telegram` — Sends Answer PDF link to Telegram.
- **Config:**
  - Chat ID: `123456789`
  - Text: `Download Answer Here : {{$json.pdf_url}}`
- **Credentials / edge cases:** Same as above.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Receives PDF + parameters via HTTP POST | — | Validate Parameter & PDF | # 🚀 ExamForge AI … (full ExamForge AI note) / ## Step 1 : Get File, Parsing and Validation file PDF |
| Validate Parameter & PDF | n8n-nodes-base.code | Validates parameters + file size (5MB) | Webhook | Condition Valid | # 🚀 ExamForge AI … / ## Step 1 : Get File, Parsing and Validation file PDF |
| Condition Valid | n8n-nodes-base.if | Branch: valid vs invalid request | Validate Parameter & PDF | Extract from File; Respond Valid Parameter | # 🚀 ExamForge AI … / ## Step 1 : Get File, Parsing and Validation file PDF |
| Respond Valid Parameter | n8n-nodes-base.respondToWebhook | Returns invalid parameter errors | Condition Valid (false) | — | # 🚀 ExamForge AI … / ## Step 1 : Get File, Parsing and Validation file PDF |
| Extract from File | n8n-nodes-base.extractFromFile | Extracts text from uploaded PDF | Condition Valid (true) | Clean Text | # 🚀 ExamForge AI … / ## Step 1 : Get File, Parsing and Validation file PDF |
| Clean Text | n8n-nodes-base.code | Cleans/normalizes extracted text, truncates | Extract from File | Length Estimation Layer | # 🚀 ExamForge AI … / ## Step 1 : Get File, Parsing and Validation file PDF |
| Length Estimation Layer | n8n-nodes-base.code | Estimates tokens and flags too-long docs | Clean Text | Validate Token > 8.000 | # 🚀 ExamForge AI … / ## Step 1 : Get File, Parsing and Validation file PDF |
| Validate Token > 8.000 | n8n-nodes-base.if | Branch: too long vs proceed | Length Estimation Layer | Respond to Webhook; Message a model | # 🚀 ExamForge AI … / ## Step 1 : Get File, Parsing and Validation file PDF |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Returns “document too long” error | Validate Token > 8.000 (true) | — | # 🚀 ExamForge AI … / ## Step 1 : Get File, Parsing and Validation file PDF |
| Message a model | @n8n/n8n-nodes-langchain.openAi | Generates structured exam JSON from text | Validate Token > 8.000 (false) | Parse JSON | # 🚀 ExamForge AI … / ## Step 2: Generate Exam and Answer |
| Parse JSON | n8n-nodes-base.code | Strips code fences and parses JSON | Message a model | Format Exam Text | # 🚀 ExamForge AI … / ## Step 2: Generate Exam and Answer |
| Format Exam Text | n8n-nodes-base.code | Builds exam & answer key HTML | Parse JSON | Convert HTML to PDF - Exam; Convert HTML to PDF - Answer | # 🚀 ExamForge AI … / ## Step 2: Generate Exam and Answer |
| Convert HTML to PDF - Answer | n8n-nodes-htmlcsstopdf.htmlcsstopdf | Renders answer key HTML to PDF | Format Exam Text | Send a text message1 | # 🚀 ExamForge AI … / ## Step 3: Convert HTML to PDF and Send Telegram |
| Convert HTML to PDF - Exam | n8n-nodes-htmlcsstopdf.htmlcsstopdf | Renders exam HTML to PDF | Format Exam Text | Send a text message | # 🚀 ExamForge AI … / ## Step 3: Convert HTML to PDF and Send Telegram |
| Send a text message1 | n8n-nodes-base.telegram | Sends Answer PDF link to Telegram | Convert HTML to PDF - Answer | — | # 🚀 ExamForge AI … / ## Step 3: Convert HTML to PDF and Send Telegram |
| Send a text message | n8n-nodes-base.telegram | Sends Exam PDF link to Telegram | Convert HTML to PDF - Exam | — | # 🚀 ExamForge AI … / ## Step 3: Convert HTML to PDF and Send Telegram |
| Sticky Note1 | n8n-nodes-base.stickyNote | Workspace documentation note | — | — | # 🚀 ExamForge AI … (contains features/requirements/example curl) |
| Sticky Note | n8n-nodes-base.stickyNote | Labels Step 1 block | — | — | ## Step 1 : Get File, Parsing and Validation file PDF |
| Sticky Note2 | n8n-nodes-base.stickyNote | Labels Step 2 block | — | — | ## Step 2: Generate Exam and Answer |
| Sticky Note3 | n8n-nodes-base.stickyNote | Labels Step 3 block | — | — | ## Step 3: Convert HTML to PDF and Send Telegram |

> Note: Sticky notes are listed as nodes in the workflow; they don’t execute.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Webhook trigger**
   - Add node: **Webhook**
   - Method: **POST**
   - Response: **Respond to Webhook node**
   - Set a path (e.g., `/examforge`)
   - Expect **multipart/form-data** with:
     - File field: `file` (n8n stores it under `binary.data` by default)
     - Fields: `mcq_count`, `essay_count`, `difficulty`, `language`

2. **Add “Validate Parameter & PDF” (Code node)**
   - Add node: **Code**
   - Implement validations:
     - Required fields: mcq_count, essay_count, difficulty, language
     - Numeric checks for counts
     - Difficulty in `easy|medium|hard`
     - Max file size: 5MB using `binary.data.fileSize`
     - Max counts (mcq ≤ 50; essay ≤ 20 — fix the mismatch if desired)
   - Output fields:
     - `isValid`, `errors[]`, `mcq_count`, `essay_count`, `difficulty`, `language`
     - `data` referencing the PDF binary (or keep it in `binary.data` and avoid remapping)

3. **Add “Condition Valid” (IF node)**
   - Condition: `isValid equals true`
   - **True** → extraction path
   - **False** → respond error path

4. **Add “Respond Valid Parameter” (Respond to Webhook)**
   - Respond with JSON containing `errors` (e.g., `details: {{$json.errors}}`)
   - Connect from **Condition Valid (false)**

5. **Add “Extract from File”**
   - Node: **Extract from File**
   - Operation: **PDF**
   - Configure binary property correctly:
     - If your PDF is in `binary.data`, set **Binary Property Name = `data`**
   - Connect from **Condition Valid (true)**

6. **Add “Clean Text” (Code node)**
   - Normalize extracted text (remove weird chars, collapse whitespace)
   - Truncate to a safe length (15,000 chars as in workflow)
   - Output: `cleaned_text`

7. **Add “Length Estimation Layer” (Code node)**
   - Estimate tokens: `ceil(cleaned_text.length / 4)`
   - If > 8000 tokens, output `{ error: true, message }`
   - Else output `{ error: false, cleaned_text, estimated_tokens }`

8. **Add “Validate Token > 8.000” (IF node)**
   - Condition: `error equals true`
   - **True** → respond “too long”
   - **False** → OpenAI generation

9. **Add “Respond to Webhook” (Respond to Webhook node)**
   - Return JSON error for documents too long (token overflow prevention)
   - Connect from **Validate Token > 8.000 (true)**

10. **Add “Message a model” (OpenAI / LangChain node)**
   - Node: **OpenAI (LangChain) – Message a model**
   - Credential: configure **OpenAI API key** in n8n Credentials
   - Model: `gpt-4o-mini` (or your preferred)
   - Prompt:
     - Instruct to return **ONLY valid JSON** with the required schema
     - Include dynamic fields:
       - Material: `{{$json.cleaned_text}}`
       - Counts/difficulty/language from webhook body, e.g. `{{$('Webhook').item.json.body.mcq_count}}`
   - Connect from **Validate Token > 8.000 (false)**

11. **Add “Parse JSON” (Code node)**
   - Read the model text output (ensure you match your node’s actual output structure)
   - Strip ```json fences if present
   - `JSON.parse()` and throw a clear error if invalid
   - Output parsed object with keys `mcq`, `essay`

12. **Add “Format Exam Text” (Code node)**
   - Build:
     - `exam_html` (MCQ + essay questions)
     - `answer_key_html` (MCQ answers + essay expected points)
   - Output both strings

13. **Add PDF conversion nodes**
   - Add node: **HTMLCSS to PDF** (or equivalent)
   - Credential: configure HTMLCSS.toPDF API key in n8n Credentials
   - Node A: “Convert HTML to PDF - Exam”
     - `html_content = {{$json.exam_html}}`
   - Node B: “Convert HTML to PDF - Answer”
     - `html_content = {{$json.answer_key_html}}`
   - Connect both from **Format Exam Text**

14. **Add Telegram delivery (optional)**
   - Add node: **Telegram → Send Message**
   - Credential: Telegram bot token in n8n Credentials
   - Chat ID: your target chat (replace `123456789`)
   - Exam message text: `Download Exam Here : {{$json.pdf_url}}`
   - Answer message text: `Download Answer Here : {{$json.pdf_url}}`
   - Connect:
     - Exam PDF node → Exam Telegram node
     - Answer PDF node → Answer Telegram node

15. **(Recommended) Ensure webhook always gets a final response**
   - Current design responds only on error paths; success path sends Telegram messages but does **not** respond to the HTTP caller.
   - Add another **Respond to Webhook** node on the success path (e.g., after formatting or after PDF generation) to return URLs.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “ExamForge AI – Automated PDF to Structured Exam Generator (MCQ + Essay + Answer Key)” | Workflow branding/content from Sticky Note1 |
| Accounts required: OpenAI API key, Telegram Bot (optional), “PDF Munk / HTMLCSS.toPDF” API key | Requirements section from Sticky Note1 |
| Webhook expects POST multipart/form-data with parameters: file, mcq_count, essay_count, difficulty, language | Sticky Note1 “Webhook Configuration” |
| Example curl request (as provided) | Sticky Note1 example (update URL/path to your webhook) |
| Disclaimer (French): content comes exclusively from an n8n automated workflow; legal/public data; no illegal/offensive/protected elements | Provided by user in prompt context |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.