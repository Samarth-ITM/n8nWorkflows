Create anti-cheating tests from PDFs with GPT-4o-mini and Google Docs

https://n8nworkflows.xyz/workflows/create-anti-cheating-tests-from-pdfs-with-gpt-4o-mini-and-google-docs-13135


# Create anti-cheating tests from PDFs with GPT-4o-mini and Google Docs

## 1. Workflow Overview

**Purpose:** Generate two anti-cheating versions of a student test (Group A and Group B) from teacher-provided PDF material, format them into a single Google Doc (with a page break between versions), share the doc publicly (read-only), and email the link to the requester.

**Target use cases:**
- Teachers creating quick assessments from lesson PDFs
- Producing two parallel test versions to reduce cheating
- Automated document creation + delivery via email

### 1.1 Input Reception & PDF Extraction
Collects teacher metadata + a PDF upload via an n8n Form, then extracts the PDF text for downstream AI generation.

### 1.2 AI Question Generation (Structured JSON)
Sends extracted PDF text to OpenAI (GPT‑4o‑mini) with strict instructions and a JSON Schema output requirement to produce 10 questions split into Group A and Group B.

### 1.3 Test Formatting (Printable Text Blocks)
Transforms the AI JSON output into two printable plain-text sections (Group A and Group B) including headers (Subject/Grade/Date) and answer/work lines.

### 1.4 Google Docs Creation, Sharing, and Population
Creates a new Google Doc, shares it as “anyone with the link can read,” inserts Group A, inserts a page break, then inserts Group B.

### 1.5 Email Notification
Emails the requester a link to the generated Google Doc.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & PDF Extraction

**Overview:** Captures form fields (teacher name, subject, grade, date, email) and a PDF upload, then extracts text from the PDF to feed the AI model.

**Nodes Involved:**
- Form Trigger
- Extract PDF

#### Node: Form Trigger
- **Type / Role:** `n8n-nodes-base.formTrigger` — Entry point that hosts an n8n form and outputs submitted fields + uploaded file binary.
- **Configuration (interpreted):**
  - Form title: **AI Test Generator**
  - Description: “Generate test questions from PDF materials”
  - Fields (all required):
    - Name (text)
    - Subject (text)
    - Grade (text, placeholder “e.g. 10th Grade”)
    - Date (date)
    - Upload PDF (file; accepts `.pdf`)
    - Email (email)
  - Attribution disabled (`appendAttribution: false`)
- **Key variables/fields produced:**
  - `$json.Name`, `$json.Subject`, `$json.Grade`, `$json.Date`, `$json.Email`
  - Binary file property for upload is referenced later as **`Upload_PDF`** (must match the internal field name created by the form).
- **Connections:**
  - **Output →** Extract PDF
- **Version-specific notes:** Node typeVersion **2.5** (n8n Form Trigger UI/field behavior can differ between major versions).
- **Edge cases / failures:**
  - Missing/invalid PDF upload (user error) prevents downstream extraction.
  - Form field naming: if n8n changes the binary property name or the form field label changes, downstream node must be updated.

#### Node: Extract PDF
- **Type / Role:** `n8n-nodes-base.extractFromFile` — Extracts text content from an uploaded file (PDF operation).
- **Configuration (interpreted):**
  - Operation: **PDF**
  - Binary property name: **Upload_PDF**
- **Connections:**
  - **Input ←** Form Trigger
  - **Output →** AI Generate
- **Version-specific notes:** typeVersion **1.1**
- **Edge cases / failures:**
  - Extraction can fail on scanned/image-only PDFs (no embedded text).
  - Large PDFs may produce very long text that increases token usage and may exceed model limits unless truncated upstream.
  - Corrupt PDFs or password-protected PDFs will fail extraction.

---

### Block 2 — AI Question Generation (Structured JSON)

**Overview:** Uses GPT‑4o‑mini to generate 10 test questions from the extracted PDF text, enforcing a strict schema and split into two distinct groups to support anti-cheating.

**Nodes Involved:**
- AI Generate

#### Node: AI Generate
- **Type / Role:** `@n8n/n8n-nodes-langchain.openAi` — Calls OpenAI chat model through n8n’s LangChain-based node; returns structured content.
- **Configuration (interpreted):**
  - Model: **gpt-4o-mini**
  - Max tokens: **3000**
  - Output formatting: **JSON Schema** enforced (`textFormat.type = json_schema`)
  - System prompt defines:
    - English only, professional tone
    - No invention beyond source material
    - Avoid repeating concepts across questions
    - 10 questions total: 5 in Group A + 5 in Group B
    - Per group: 2 fill-in (easy), 2 multiple-choice w/ 4 options (medium), 1 problem (hard)
    - Groups A and B must be substantially different
    - Return only valid JSON matching schema
  - User content: `={{ $json.text }}` (the extracted PDF text)
- **Schema expectations (important for downstream Code node):**
  - Root object with:
    - `test_title` (string)
    - `questions` (array of objects)
  - Each question requires:
    - `group`: "A" or "B"
    - `number`: integer
    - `type`: "fill-in" | "multiple-choice" | "problem"
    - `text`: string
    - `options`: array of strings (still required even for non-multiple-choice per schema)
    - `correct_answer`: string
    - `points`: integer
- **Connections:**
  - **Input ←** Extract PDF
  - **Output →** Format Test
- **Credentials:** OpenAI API credential required (`openAiApi`)
- **Version-specific notes:** typeVersion **2.1** (LangChain/OpenAI node output structure can vary by node version).
- **Edge cases / failures:**
  - **Schema mismatch:** If the model returns invalid JSON or violates schema, the node may error or produce output that breaks the Code node assumptions.
  - **Token overflow:** Very long extracted text can push context limits; results may truncate or fail.
  - **Content gaps:** If PDF lacks enough instructional content, model may struggle to comply with “do not invent,” yielding fewer/poorer questions or refusal-like outputs.
  - **Cost/latency:** Depends on PDF size; larger prompts increase latency and spend.

---

### Block 3 — Test Formatting (Printable Text Blocks)

**Overview:** Converts the AI-generated JSON questions into two formatted plain-text test versions (A and B), including a standardized header and answer/work lines.

**Nodes Involved:**
- Format Test

#### Node: Format Test
- **Type / Role:** `n8n-nodes-base.code` — JavaScript post-processing of AI output into printable strings.
- **Configuration (interpreted):**
  - Reads:
    - AI output from input item
    - Form fields from the **Form Trigger** node
  - Produces:
    - `textA`: formatted Group A test
    - `textB`: formatted Group B test
- **Key expressions/variables:**
  - `const ai = $input.first().json;`
  - `const qs = ai.output[0].content[0].text.questions;`
    - Assumes OpenAI node returns a structure containing `output[0].content[0].text.questions`
  - `const form = $('Form Trigger').item.json;`
  - Header template includes:
    - `SUBJECT`, `GRADE`, `DATE`, and blank `STUDENT NAME`
  - Formatting rules:
    - Multiple-choice prints `a)`–`d)` if exactly 4 options
    - Adds “Answer:” line for all questions
    - Adds “Work:” line for `problem` type
    - Prints `Points: X`
  - Splits questions into `group === "A"` and `group === "B"`
- **Connections:**
  - **Input ←** AI Generate
  - **Output →** Create Doc
- **Version-specific notes:** typeVersion **2**
- **Edge cases / failures:**
  - **Brittle output path:** If AI node output structure differs (common across versions/settings), `ai.output[0]...` may be undefined and throw.
  - If a question’s `options` is missing/empty for multiple-choice, options won’t render (but schema says options are required—still, model may fail).
  - If group labels aren’t exactly “A”/“B”, filtering yields empty tests.
  - If numbering is not sequential or duplicates exist, formatting will preserve whatever number is provided.

---

### Block 4 — Google Docs Creation, Sharing, and Population

**Overview:** Creates a Google Doc titled from form fields, sets it to be readable by anyone with the link, then inserts Group A content, adds a page break, and inserts Group B content.

**Nodes Involved:**
- Create Doc
- Share Doc
- Insert Group A
- Page Break
- Insert Group B

#### Node: Create Doc
- **Type / Role:** `n8n-nodes-base.googleDocs` — Creates a new Google Document.
- **Configuration (interpreted):**
  - Title expression:
    - `Subject - Grade - Date` from Form Trigger:
      - `={{ $('Form Trigger').item.json.Subject }} - {{ $('Form Trigger').item.json.Grade }} - {{ $('Form Trigger').item.json.Date }}`
  - (Sticky note mentions optional folder ID, but not configured in shown parameters.)
- **Connections:**
  - **Input ←** Format Test
  - **Output →** Share Doc
- **Credentials:** Google Docs OAuth2 required
- **Version-specific notes:** typeVersion **2**
- **Edge cases / failures:**
  - OAuth consent/scopes missing for Docs creation.
  - Title expression fails if form fields are missing or renamed.

#### Node: Share Doc
- **Type / Role:** `n8n-nodes-base.googleDrive` — Applies sharing permissions to the created doc via Google Drive.
- **Configuration (interpreted):**
  - Operation: **share**
  - File ID: `={{ $json.id }}` (expects Create Doc output to contain `id`)
  - Permission: `type=anyone`, `role=reader` (public link readable)
- **Connections:**
  - **Input ←** Create Doc
  - **Output →** Insert Group A
- **Credentials:** Google Drive OAuth2 required
- **Version-specific notes:** typeVersion **3**
- **Edge cases / failures:**
  - Domain policies may block “anyone with link” sharing.
  - If doc ID is not present at `$json.id`, sharing fails.

#### Node: Insert Group A
- **Type / Role:** `n8n-nodes-base.googleDocs` — Inserts formatted text for Group A into the doc body.
- **Configuration (interpreted):**
  - Operation: **update**
  - Document URL/ID: `={{ $('Create Doc').item.json.id }}`
  - Action: Insert text = `={{ $('Format Test').item.json.textA }}`
- **Connections:**
  - **Input ←** Share Doc
  - **Output →** Page Break
- **Credentials:** Google Docs OAuth2 required
- **Version-specific notes:** typeVersion **2**
- **Edge cases / failures:**
  - If doc ID reference is wrong, update fails.
  - Large inserted text may hit Google Docs API request limits in extreme cases.

#### Node: Page Break
- **Type / Role:** `n8n-nodes-base.googleDocs` — Inserts a page break between Group A and Group B.
- **Configuration (interpreted):**
  - Operation: **update**
  - Document URL/ID: `={{ $('Create Doc').item.json.id }}`
  - Action: Insert object `pageBreak`
- **Connections:**
  - **Input ←** Insert Group A
  - **Output →** Insert Group B
- **Credentials:** Google Docs OAuth2 required
- **Version-specific notes:** typeVersion **2**
- **Edge cases / failures:**
  - Same doc ID/auth issues as other Docs update nodes.

#### Node: Insert Group B
- **Type / Role:** `n8n-nodes-base.googleDocs` — Inserts formatted text for Group B after the page break.
- **Configuration (interpreted):**
  - Operation: **update**
  - Document URL/ID: `={{ $('Create Doc').item.json.id }}`
  - Action: Insert text = `={{ $('Format Test').item.json.textB }}`
- **Connections:**
  - **Input ←** Page Break
  - **Output →** Send Email
- **Credentials:** Google Docs OAuth2 required
- **Version-specific notes:** typeVersion **2**
- **Edge cases / failures:**
  - Same as Insert Group A; additionally if `textB` is empty (AI failure), the doc will contain only Group A.

---

### Block 5 — Email Notification

**Overview:** Sends a confirmation email to the requester containing the Google Doc link and summary details.

**Nodes Involved:**
- Send Email

#### Node: Send Email
- **Type / Role:** `n8n-nodes-base.gmail` — Sends an email via Gmail OAuth2.
- **Configuration (interpreted):**
  - To: `={{ $('Form Trigger').item.json.Email }}`
  - Subject: `={{ $('Form Trigger').item.json.Subject }} - Test Ready`
  - HTML message includes:
    - Greeting using `Name`
    - A link to: `https://docs.google.com/document/d/{{ $('Create Doc').item.json.id }}`
    - Summary bullets (Subject, Grade, Questions)
- **Connections:**
  - **Input ←** Insert Group B
  - **Output:** none (end of workflow)
- **Credentials:** Gmail OAuth2 required
- **Version-specific notes:** typeVersion **2.2**
- **Edge cases / failures:**
  - Gmail scopes not granted or OAuth token expired.
  - Recipient address invalid.
  - Organization policies may restrict sending.
  - Note: The doc is shared “anyone reader,” so the emailed link should work without Google sign-in; if sharing fails, recipients may get access denied.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Form Trigger | n8n-nodes-base.formTrigger | Collects form input + PDF upload (entry point) | — | Extract PDF | ## 📝 Form & PDF Processing<br><br>Captures user input and extracts text from PDF document. |
| Extract PDF | n8n-nodes-base.extractFromFile | Extracts text from uploaded PDF | Form Trigger | AI Generate | ## 📝 Form & PDF Processing<br><br>Captures user input and extracts text from PDF document. |
| AI Generate | @n8n/n8n-nodes-langchain.openAi | Generates structured test questions via GPT-4o-mini | Extract PDF | Format Test | ## 🤖 AI Question Generation<br><br>OpenAI generates 10 questions (5 per group) based on PDF content.<br><br>**Model:** GPT-4o-mini  <br>**Cost:** ~$0.002 per test |
| Format Test | n8n-nodes-base.code | Formats AI JSON into printable Group A & B text blocks | AI Generate | Create Doc | ## 🤖 AI Question Generation<br><br>OpenAI generates 10 questions (5 per group) based on PDF content.<br><br>**Model:** GPT-4o-mini  <br>**Cost:** ~$0.002 per test |
| Create Doc | n8n-nodes-base.googleDocs | Creates a Google Doc titled from form fields | Format Test | Share Doc | ## 📄 Google Docs Output<br><br>Creates formatted document with both test versions and auto-shares with link. |
| Share Doc | n8n-nodes-base.googleDrive | Shares doc as “anyone with link can read” | Create Doc | Insert Group A | ## 📄 Google Docs Output<br><br>Creates formatted document with both test versions and auto-shares with link. |
| Insert Group A | n8n-nodes-base.googleDocs | Inserts Group A formatted content | Share Doc | Page Break | ## 📄 Google Docs Output<br><br>Creates formatted document with both test versions and auto-shares with link. |
| Page Break | n8n-nodes-base.googleDocs | Inserts a page break separator | Insert Group A | Insert Group B | ## 📄 Google Docs Output<br><br>Creates formatted document with both test versions and auto-shares with link. |
| Insert Group B | n8n-nodes-base.googleDocs | Inserts Group B formatted content | Page Break | Send Email | ## 📄 Google Docs Output<br><br>Creates formatted document with both test versions and auto-shares with link. |
| Send Email | n8n-nodes-base.gmail | Emails the doc link to requester | Insert Group B | — | ## ✉️ Email Notification<br><br>Sends professional email with document link. |
| 📝 Overview | n8n-nodes-base.stickyNote | Documentation / workflow notes | — | — | ## How it works<br><br>1. **User submits form** with PDF teaching materials<br>2. **PDF text extraction** pulls content from document  <br>3. **AI generates questions** (10 total: 5 Group A, 5 Group B)<br>4. **Google Doc created** with both test versions<br>5. **Email sent** with document link<br><br>Total time: ~60 seconds<br><br>## Setup steps<br><br>1. Add OpenAI API key (for question generation)<br>2. Add Google Docs OAuth2 (for document creation)<br>3. Add Google Drive OAuth2 (for sharing)<br>4. Add Gmail OAuth2 (for email notifications)<br>5. (Optional) Set Google Drive folder ID in "Create Doc" node<br>6. Test with sample PDF<br><br>**Cost:** ~$0.002 per test (GPT-4o-mini) |
| Group: Input | n8n-nodes-base.stickyNote | Visual grouping label | — | — |  |
| Group: AI | n8n-nodes-base.stickyNote | Visual grouping label | — | — |  |
| Group: Docs | n8n-nodes-base.stickyNote | Visual grouping label | — | — |  |
| Group: Email | n8n-nodes-base.stickyNote | Visual grouping label | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: *Generate student tests from PDFs using AI and Google Docs* (or your preferred name).
   - Keep it inactive until credentials are set.

2. **Add “Form Trigger” (Form Trigger node)**
   - Node type: **Form Trigger**
   - Form title: **AI Test Generator**
   - Description: **Generate test questions from PDF materials**
   - Add required fields:
     1) Name (text, required)  
     2) Subject (text, required)  
     3) Grade (text, required; placeholder “e.g. 10th Grade”)  
     4) Date (date, required)  
     5) Upload PDF (file, required; accept “.pdf”)  
     6) Email (email, required)
   - Options: disable attribution if available.

3. **Add “Extract PDF” (Extract From File node)**
   - Node type: **Extract From File**
   - Operation: **PDF**
   - Binary property name: set to the file field binary property (in this workflow it is **`Upload_PDF`**).
   - Connect: **Form Trigger → Extract PDF**

4. **Add “AI Generate” (OpenAI / LangChain node)**
   - Node type: **OpenAI (LangChain)**
   - Credentials: create/select **OpenAI API** credential.
   - Model: **gpt-4o-mini**
   - Max tokens: **3000**
   - Prompting:
     - Add a **system** message with the full rules (English only, do not invent, 10 questions split A/B, etc.).
     - Add a **user** message whose content is the extracted PDF text:
       - Expression: `{{ $json.text }}`
   - Enforce structured output:
     - Set output format to **JSON Schema**
     - Use a schema that includes `test_title` and `questions[]` with required fields:
       - group (A/B), number, type, text, options, correct_answer, points
   - Connect: **Extract PDF → AI Generate**

5. **Add “Format Test” (Code node)**
   - Node type: **Code**
   - Paste logic that:
     - Reads the AI questions
     - Reads form fields from the Form Trigger node
     - Produces `{ textA, textB }`
   - Critical adjustment check:
     - Ensure the code reads the AI response from the correct path for your OpenAI node version. In the provided workflow it expects:
       - `ai.output[0].content[0].text.questions`
   - Connect: **AI Generate → Format Test**

6. **Add “Create Doc” (Google Docs node)**
   - Node type: **Google Docs**
   - Credentials: **Google Docs OAuth2**
   - Operation: create document (default for “Create Doc” in many setups)
   - Title expression:
     - `{{ $('Form Trigger').item.json.Subject }} - {{ $('Form Trigger').item.json.Grade }} - {{ $('Form Trigger').item.json.Date }}`
   - Optional (if supported by your node version): set a target Drive folder ID.
   - Connect: **Format Test → Create Doc**

7. **Add “Share Doc” (Google Drive node)**
   - Node type: **Google Drive**
   - Credentials: **Google Drive OAuth2**
   - Operation: **Share**
   - File ID: expression `{{ $json.id }}` (expects Create Doc to output the doc ID as `id`)
   - Permission:
     - Type: **anyone**
     - Role: **reader**
   - Connect: **Create Doc → Share Doc**

8. **Add “Insert Group A” (Google Docs node)**
   - Node type: **Google Docs**
   - Credentials: Google Docs OAuth2
   - Operation: **Update**
   - Document URL/ID: `{{ $('Create Doc').item.json.id }}`
   - Action: **Insert text**
     - Text: `{{ $('Format Test').item.json.textA }}`
   - Connect: **Share Doc → Insert Group A**

9. **Add “Page Break” (Google Docs node)**
   - Node type: **Google Docs**
   - Operation: **Update**
   - Document URL/ID: `{{ $('Create Doc').item.json.id }}`
   - Action: **Insert pageBreak**
   - Connect: **Insert Group A → Page Break**

10. **Add “Insert Group B” (Google Docs node)**
   - Node type: **Google Docs**
   - Operation: **Update**
   - Document URL/ID: `{{ $('Create Doc').item.json.id }}`
   - Action: **Insert text**
     - Text: `{{ $('Format Test').item.json.textB }}`
   - Connect: **Page Break → Insert Group B**

11. **Add “Send Email” (Gmail node)**
   - Node type: **Gmail**
   - Credentials: **Gmail OAuth2**
   - To: `{{ $('Form Trigger').item.json.Email }}`
   - Subject: `{{ $('Form Trigger').item.json.Subject }} - Test Ready`
   - Message (HTML): include the doc link:
     - `https://docs.google.com/document/d/{{ $('Create Doc').item.json.id }}`
   - Connect: **Insert Group B → Send Email**

12. **Credentials and scopes checklist**
   - OpenAI: valid API key, billing enabled.
   - Google Docs OAuth2: scopes for creating/updating Docs.
   - Google Drive OAuth2: scopes for changing sharing permissions.
   - Gmail OAuth2: scopes for sending email.

13. **Test end-to-end**
   - Submit the form with a small text-based PDF first.
   - Verify:
     - AI returns valid JSON per schema
     - Doc is created and contains both groups separated by a page break
     - Sharing is “anyone with the link”
     - Email arrives and link opens successfully

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Total time: ~60 seconds” | From workflow overview sticky note |
| Setup requires OpenAI + Google Docs + Google Drive + Gmail credentials | From workflow overview sticky note |
| Optional: set Google Drive folder ID in “Create Doc” node | From workflow overview sticky note |
| Estimated cost: “~$0.002 per test (GPT-4o-mini)” | From workflow overview + AI grouping sticky note |