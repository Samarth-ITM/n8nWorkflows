Classify job applicants from CVs using Claude, PDF.co, Google Docs and Gmail

https://n8nworkflows.xyz/workflows/classify-job-applicants-from-cvs-using-claude--pdf-co--google-docs-and-gmail-12779


# Classify job applicants from CVs using Claude, PDF.co, Google Docs and Gmail

## 1. Workflow Overview

**Workflow name:** CV  
**Purpose:** Automatically receive job applications via an n8n Form, extract text from an uploaded CV (PDF or Word), summarize it with Claude (Anthropic), compare it against hiring criteria stored in Google Docs, classify the applicant as suitable/unsuitable, then send an appropriate email (and for suitable candidates, propose interview availability using Google Calendar).

### 1.1 Input Reception (Form)
Collects applicant identity details and a CV file upload.

### 1.2 File-Type Routing (PDF vs Word)
Routes processing depending on MIME type. Word files are converted via PDF.co; PDFs are extracted directly.

### 1.3 Text Extraction
- Word → PDF.co upload → convert → extract text
- PDF → Extract From File node

### 1.4 AI Summarization + Criteria Comparison
Summarizes the CV, fetches company criteria from Google Docs, and produces a final suitability summary.

### 1.5 Classification + Email Handling
Classifies into **Classified** vs **Not Classified**, then:
- **Positive agent:** checks calendar availability and emails congratulations + interview slots
- **Negative agent:** emails a polite rejection/motivation message

---

## 2. Block-by-Block Analysis

### Block 2.1 — Input Reception
**Overview:** Provides a hosted form endpoint to collect applicant details and a single CV file.  
**Nodes involved:** `On form submission`

#### Node: On form submission
- **Type / role:** `n8n-nodes-base.formTrigger` — workflow entry point via n8n Form.
- **Configuration (interpreted):**
  - Form title: **WELCOME TO OUR TEAM**
  - Description: “Fulfill the column's below.”
  - Fields (all required):
    - Full Name (text)
    - E-Mail Add (email)
    - Phone Number (number)
    - Upload CV (file; single file; accepts “PDF.” as configured)
- **Key variables / fields produced:**
  - `$json['Full Name']`
  - `$json['E-Mail Add']`
  - `$json['Phone Number ']` (note trailing space in label)
  - `$json['Upload CV ']` (note trailing space in label; contains metadata + binary mapping)
- **Outputs / connections:** → `Switch`
- **Edge cases / failures:**
  - The “acceptFileTypes” value is `"PDF."` which is not a standard MIME filter; users may still upload non-PDF depending on UI behavior/version.
  - Field labels include trailing spaces (`'Upload CV '`), which makes expressions brittle.

---

### Block 2.2 — File-Type Routing (PDF vs Word)
**Overview:** Checks uploaded file MIME type and routes to either PDF.co (Word conversion) or direct PDF extraction.  
**Nodes involved:** `Switch`

#### Node: Switch
- **Type / role:** `n8n-nodes-base.switch` — conditional router.
- **Configuration (interpreted):**
  - Rule 1 output key: `word`
    - Condition: `{{$json['Upload CV '].mimetype}} == application/vnd.openxmlformats-officedocument.wordprocessingml.document`
  - Rule 2 output key: `pdf`
    - Condition: `{{$json['Upload CV '].mimetype}} == application/pdf`
  - Output branches are renamed (using `outputKey`).
- **Inputs:** From `On form submission`
- **Outputs / connections:**
  - `word` branch → `uploading` (PDF.co)
  - `pdf` branch → `Extract from File`
- **Edge cases / failures:**
  - If the file is neither MIME type (e.g., DOC older format, image PDF, or the “acceptFileTypes” mismatch), it will not match any rule → no downstream execution.
  - Relies on `$json['Upload CV ']` exact property name (includes trailing space).

---

### Block 2.3 — Word → PDF.co Conversion + Text Extraction
**Overview:** For Word documents, uploads the file to PDF.co, converts it to PDF, then extracts text from the resulting PDF via PDF.co “Convert from PDF”.  
**Nodes involved:** `uploading`, `Converting to PDF Link`, `Extracting the content inside the Link`

#### Node: uploading
- **Type / role:** `n8n-nodes-pdfco.PDFco Api` — PDF.co file upload.
- **Configuration (interpreted):**
  - Operation: **Upload File to PDF.co**
  - Uses binary upload: enabled
  - Binary property name: `Upload_CV_`
  - Parameter `name` is set to `=` (likely unintended/placeholder)
- **Inputs:** From `Switch` (word path)
- **Outputs / connections:** → `Converting to PDF Link`
- **Edge cases / failures:**
  - Binary property name must match the actual binary field name in the incoming item. The form field is labeled “Upload CV ”; n8n often maps binary keys differently. A mismatch will cause “binary property not found”.
  - PDF.co credential/auth errors or quota limits.

#### Node: Converting to PDF Link
- **Type / role:** `n8n-nodes-pdfco.PDFco Api` — converts uploaded file URL to PDF.
- **Configuration (interpreted):**
  - Operation: **Convert to PDF**
  - URL: `{{$json.url}}` (output of previous PDF.co upload)
- **Inputs:** From `uploading`
- **Outputs / connections:** → `Extracting the content inside the Link`
- **Edge cases / failures:**
  - If `uploading` returns a different field than `url`, expression breaks.
  - Conversion can fail for protected/corrupt Word docs.

#### Node: Extracting the content inside the Link
- **Type / role:** `n8n-nodes-pdfco.PDFco Api` — extracts content from PDF.
- **Configuration (interpreted):**
  - Operation: **Convert from PDF** (PDF → text/structured output depending on PDF.co defaults)
  - URL: `{{$json.url}}` (from conversion step)
- **Inputs:** From `Converting to PDF Link`
- **Outputs / connections:** → `Summarization CV-s`
- **Edge cases / failures:**
  - Scanned PDFs may produce poor text unless OCR options are enabled (none configured here).
  - Output structure varies by PDF.co endpoint; downstream summarization must match the returned text field.

---

### Block 2.4 — PDF Direct Text Extraction
**Overview:** For PDF uploads, extracts text locally using n8n’s Extract From File node.  
**Nodes involved:** `Extract from File`

#### Node: Extract from File
- **Type / role:** `n8n-nodes-base.extractFromFile` — text extraction from binary.
- **Configuration (interpreted):**
  - Operation: **pdf**
  - Binary property name: `Upload_CV_`
- **Inputs:** From `Switch` (pdf path)
- **Outputs / connections:** → `Summarization CV-s`
- **Edge cases / failures:**
  - Same binary property risk as above: if the uploaded file is stored under a different binary key, extraction fails.
  - For scanned/image-only PDFs, extracted text may be empty.

---

### Block 2.5 — AI Summarization + Criteria Comparison
**Overview:** Uses Claude to summarize the extracted CV, loads hiring criteria from Google Docs as a tool, and compares the summary to the criteria to produce a suitability statement.  
**Nodes involved:** `Anthropic Chat Model`, `Summarization CV-s`, `CV Criteria.`, `CV COMPARATOR`

#### Node: Anthropic Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` — LLM provider for LangChain nodes.
- **Configuration (interpreted):**
  - Model: `claude-sonnet-4-5-20250929` (listed as “Claude Sonnet 4.5”)
- **Connections:**
  - Provides the **AI Language Model** input to:
    - `Summarization CV-s`
    - `CV COMPARATOR`
    - `APPLICANT Classification`
    - `POSITIVE AGENT`
    - `NEGATIVE AGENT`
- **Edge cases / failures:**
  - Model name/version may not exist or may change availability; execution fails if the model is not available in the Anthropic account/region.
  - Anthropic credential issues, rate limits, or payload size limits.

#### Node: Summarization CV-s
- **Type / role:** `@n8n/n8n-nodes-langchain.chainSummarization` — summarization chain over extracted text.
- **Configuration (interpreted):**
  - Chunking mode: **advanced** (better for long CVs)
- **Inputs:**
  - From either `Extract from File` (PDF path) or `Extracting the content inside the Link` (Word→PDF.co path)
  - LLM from `Anthropic Chat Model`
- **Outputs / connections:** → `CV COMPARATOR`
- **Edge cases / failures:**
  - If upstream extraction output isn’t in the expected text field(s), summarization may produce empty or error output.
  - Very long CVs could still hit token limits depending on summarization implementation.

#### Node: CV Criteria.
- **Type / role:** `n8n-nodes-base.googleDocsTool` — LangChain tool wrapper for Google Docs.
- **Configuration (interpreted):**
  - Operation: **get**
  - DocumentURL is set to: `1t1Lwh0yF8JcnpyjKRNKGMBGy1CO8SOsDQLO01ivIONE`
    - This looks like a **Google Doc ID**, not a full URL. Some node versions accept IDs; others require a full URL.
- **Inputs / outputs:**
  - Connected as an **AI tool** to `CV COMPARATOR` (the agent can call it).
- **Edge cases / failures:**
  - OAuth scope/permission issues (doc not shared with the connected Google account).
  - If the node expects a full URL, it will fail parsing.

#### Node: CV COMPARATOR
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — agent that compares summarized CV to criteria.
- **Configuration (interpreted):**
  - Prompt instructs it to:
    - Act as hiring professional
    - Access Google Docs tool to “remember Our Hiring Criteria”
    - Use the “previous Summarization of CV-s {{ $json.output.text }}”
    - Output a suitability summarization
  - Prompt type: **define**
- **Inputs:**
  - Main input from `Summarization CV-s`
  - AI language model from `Anthropic Chat Model`
  - AI tool: `CV Criteria.` (Google Docs)
- **Outputs / connections:** → `APPLICANT Classification`
- **Edge cases / failures:**
  - The prompt references `{{$json.output.text}}`. Depending on the summarization node’s output schema, it might be `$json.output` (string) rather than `.text`.
  - If the agent doesn’t call the tool or tool fails, comparison quality degrades.

---

### Block 2.6 — Final Classification + Branching
**Overview:** Converts the comparator output into one of two categories and branches execution accordingly.  
**Nodes involved:** `APPLICANT Classification`

#### Node: APPLICANT Classification
- **Type / role:** `@n8n/n8n-nodes-langchain.textClassifier` — LLM-based text classification.
- **Configuration (interpreted):**
  - Input text: `{{$json.output}}`
  - Categories:
    - **Classified:** suitable for work
    - **Not Classified:** not suitable
- **Inputs:**
  - Main input from `CV COMPARATOR`
  - LLM from `Anthropic Chat Model`
- **Outputs / connections:**
  - Output 1 → `POSITIVE AGENT`
  - Output 2 → `NEGATIVE AGENT`
- **Edge cases / failures:**
  - If `$json.output` is not present (comparator schema mismatch), classification fails.
  - Categories are very broad; ambiguous comparator output may flip results.

---

### Block 2.7 — Positive Handling (Email + Calendar Availability)
**Overview:** For suitable applicants, generates a congratulatory email in HTML and uses Google Calendar tool to find availability (tool is connected), then sends email using Gmail tool.  
**Nodes involved:** `POSITIVE AGENT`, `Get availability in a calendar in Google Calendar`, `Send a message in Gmail`

#### Node: Get availability in a calendar in Google Calendar
- **Type / role:** `n8n-nodes-base.googleCalendarTool` — LangChain tool wrapper for Google Calendar.
- **Configuration (interpreted):**
  - Resource: `calendar`
  - Calendar: `user@example.com` (selected from list)
- **Inputs / outputs:**
  - Connected as an **AI tool** to `POSITIVE AGENT`.
- **Edge cases / failures:**
  - Calendar ID is set to `user@example.com` as placeholder; must be replaced with a real calendar in the account.
  - OAuth permissions/scope issues.

#### Node: Send a message in Gmail
- **Type / role:** `n8n-nodes-base.gmailTool` — LangChain tool wrapper to send email.
- **Configuration (interpreted):**
  - `sendTo`, `subject`, `message` are auto-generated tool parameters using `$fromAI(...)`
  - `appendAttribution`: false
- **Inputs / outputs:**
  - Connected as an **AI tool** to both `POSITIVE AGENT` and `NEGATIVE AGENT`.
- **Edge cases / failures:**
  - If the agent does not call the tool properly (missing To/Subject/Message), email won’t send.
  - Gmail OAuth errors, “from” restrictions, or sending limits.

#### Node: POSITIVE AGENT
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — generates email and consults tools.
- **Configuration (interpreted):**
  - System/prompt content (summary):
    - Roleplay: hiring professional with “10 years on Tesla” and NYT author
    - Create appointment schedule on Google Calendar “only Monday till Friday”
    - Generate congratulatory email inviting to interview with availability created/checked
    - Uses tools:
      - Gmail Tool (send email)
      - Google Calendar Tool (availability)
    - Uses current date/time from `{{$json.now}}` (note: `now` is not created elsewhere in this workflow)
    - Applicant email: `{{ $('On form submission').item.json['E-Mail Add'] }}`
    - Applicant name: `{{ $('On form submission').item.json['Full Name'] }}`
    - Output format: HTML bullet points
- **Inputs:**
  - Main input from `APPLICANT Classification` (positive branch)
  - LLM from `Anthropic Chat Model`
  - Tools: Gmail + Google Calendar
- **Edge cases / failures:**
  - `{{$json.now}}` is likely undefined unless n8n injects it or another node sets it; consider replacing with `{{$now}}` (n8n expression) or a Date & Time node.
  - The prompt says “Create an Appointment Schedule on Google Calendar” but the workflow only provides an **availability** tool, not a “create event” tool; the agent may be unable to actually schedule.

---

### Block 2.8 — Negative Handling (Rejection Email)
**Overview:** For unsuitable applicants, generates an HTML rejection/motivation email and sends it via Gmail tool.  
**Nodes involved:** `NEGATIVE AGENT`, `Send a message in Gmail`

#### Node: NEGATIVE AGENT
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — generates rejection email and uses Gmail tool.
- **Configuration (interpreted):**
  - Main `text`: `{{$json.output}}`
  - System message instructs:
    - Express condolences / rejection politely
    - Include motivational content
    - Start with applicant’s name
    - Use Gmail tool to send
    - Applicant email/name read from `On form submission`
    - Output format: HTML
- **Inputs:**
  - Main input from `APPLICANT Classification` (negative branch)
  - LLM from `Anthropic Chat Model`
  - Tool: Gmail
- **Edge cases / failures:**
  - If `$json.output` does not contain meaningful context, the email may be generic.
  - Same Gmail tool-call dependency as positive path.

---

### Block 2.9 — Documentation / Canvas Notes (Sticky Notes)
**Overview:** Non-executable nodes that document the workflow on the canvas.  
**Nodes involved:** `Sticky Note`, `Sticky Note1`, `Sticky Note2`, `Sticky Note3`, `Sticky Note4`, `Sticky Note5`, `Sticky Note6`

- **Edge cases:** None (do not execute).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | n8n-nodes-base.formTrigger | Collect applicant info + CV upload (entry point) | — | Switch | ## Form Submission  /  ## Divider Path |
| Switch | n8n-nodes-base.switch | Route PDF vs Word branch by MIME type | On form submission | uploading; Extract from File | ## Form Submission  /  ## Divider Path |
| uploading | n8n-nodes-pdfco.PDFco Api | Upload Word file to PDF.co | Switch (word) | Converting to PDF Link | ## 1) Converting for URL  /  ## 2) Converting to PDF from URL  /  ## 3) Extracting Information |
| Converting to PDF Link | n8n-nodes-pdfco.PDFco Api | Convert uploaded file URL to PDF | uploading | Extracting the content inside the Link | ## 1) Converting for URL  /  ## 2) Converting to PDF from URL  /  ## 3) Extracting Information |
| Extracting the content inside the Link | n8n-nodes-pdfco.PDFco Api | Extract text/content from PDF URL | Converting to PDF Link | Summarization CV-s | ## 1) Converting for URL  /  ## 2) Converting to PDF from URL  /  ## 3) Extracting Information |
| Extract from File | n8n-nodes-base.extractFromFile | Extract text from uploaded PDF binary | Switch (pdf) | Summarization CV-s | ## Extract Context |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude LLM provider for all AI nodes | — | Summarization CV-s; CV COMPARATOR; APPLICANT Classification; POSITIVE AGENT; NEGATIVE AGENT |  |
| Summarization CV-s | @n8n/n8n-nodes-langchain.chainSummarization | Summarize extracted CV text | Extract from File; Extracting the content inside the Link | CV COMPARATOR | ## Summarization of CV-s  /  ## Comparator between criteria and Summarization |
| CV Criteria. | n8n-nodes-base.googleDocsTool | Tool to fetch hiring criteria from Google Docs | — | CV COMPARATOR (as tool) | ## Summarization of CV-s  /  ## Comparator between criteria and Summarization |
| CV COMPARATOR | @n8n/n8n-nodes-langchain.agent | Compare CV summary with criteria | Summarization CV-s | APPLICANT Classification | ## Summarization of CV-s  /  ## Comparator between criteria and Summarization |
| APPLICANT Classification | @n8n/n8n-nodes-langchain.textClassifier | Classify applicant (suitable/unsuitable) | CV COMPARATOR | POSITIVE AGENT; NEGATIVE AGENT | ## Classification Branches |
| POSITIVE AGENT | @n8n/n8n-nodes-langchain.agent | Congratulate + propose interview availability; uses tools | APPLICANT Classification (positive) | — | ## Expressing  /  ## Writing  /  ## Scheduling |
| NEGATIVE AGENT | @n8n/n8n-nodes-langchain.agent | Rejection email; uses Gmail tool | APPLICANT Classification (negative) | — | ## Expressing  /  ## Writing  /  ## Scheduling |
| Send a message in Gmail | n8n-nodes-base.gmailTool | Tool to send emails via Gmail (used by agents) | — | POSITIVE AGENT (as tool); NEGATIVE AGENT (as tool) | ## Expressing  /  ## Writing  /  ## Scheduling |
| Get availability in a calendar in Google Calendar | n8n-nodes-base.googleCalendarTool | Tool to read calendar availability (used by positive agent) | — | POSITIVE AGENT (as tool) | ## Expressing  /  ## Writing  /  ## Scheduling |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation | — | — | # CV CLASSIFIER  /  ## HOW IT WORKS ?  /  ### On Form Submission : Create columns for providing personal information and uploading data.  /  ### Switch Node : Separate the branches if it's in PDF Format or Google Word .  /  ### PDF.co :  We add three nodes of them :  1) uploading a file  2) Convert to PDF and  3) Convert from PDF.  /  ### Extract From File : it's on the path if the Data Uploaded it's on PDF Format who extract inside information.  /  ### Summarization CV's : Make summarization of the item from previous nodes.  /  ### CV Comparator : We give instructions to make classification between CV Criteria what Company hire and what's the Applicant's CV  /  ### Applicant Classification : Output the final results and make a division if the Applicant's are suitable for next interview or not.  /  ### POSITIVE AGENT : Express Congratulations to the Applicant via Email and also check the calendar and providing the free dates available for interview.  /  ### NEGATIVE AGENT : On politely way he's responding that applicant's didn't pass the Criteria and wish them more success on the future |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation | — | — |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation | — | — |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (name it e.g. “CV”).
2. **Add Form Trigger**
   - Node: **Form Trigger** (`On form submission`)
   - Configure form:
     - Title: `WELCOME TO OUR TEAM`
     - Description: `Fulfill the column's below.`
     - Fields (required):
       - `Full Name` (text)
       - `E-Mail Add` (email)
       - `Phone Number ` (number) *(note space if you want to match expressions exactly)*
       - `Upload CV ` (file; single file)
3. **Add a Switch node** (`Switch`)
   - Create 2 rules:
     - Word rule: `{{$json['Upload CV '].mimetype}}` equals `application/vnd.openxmlformats-officedocument.wordprocessingml.document`
     - PDF rule: `{{$json['Upload CV '].mimetype}}` equals `application/pdf`
   - Connect: **Form Trigger → Switch**
4. **PDF path: Extract text locally**
   - Add node: **Extract From File** (`Extract from File`)
   - Operation: `pdf`
   - Binary Property Name: `Upload_CV_` (ensure this matches your incoming binary key; adjust if needed)
   - Connect: **Switch (pdf output) → Extract from File**
5. **Word path: Convert via PDF.co**
   - Add node: **PDF.co** (`uploading`)
     - Operation: *Upload File to PDF.co*
     - Binary Data: true
     - Binary Property Name: `Upload_CV_` (adjust to match actual binary key)
     - Configure **PDF.co credentials** (API key)
   - Add node: **PDF.co** (`Converting to PDF Link`)
     - Operation: *Convert to PDF*
     - URL: `{{$json.url}}`
   - Add node: **PDF.co** (`Extracting the content inside the Link`)
     - Operation: *Convert from PDF*
     - URL: `{{$json.url}}`
   - Connect: **Switch (word output) → uploading → Converting to PDF Link → Extracting the content inside the Link**
6. **Add Anthropic model node**
   - Node: **Anthropic Chat Model** (`Anthropic Chat Model`)
   - Select a Claude model available in your account (the workflow uses `claude-sonnet-4-5-20250929`; adjust if not available).
   - Configure **Anthropic API credentials**.
7. **Add Summarization Chain**
   - Node: **Summarization Chain** (`Summarization CV-s`)
   - Chunking mode: `advanced`
   - Connect model: **Anthropic Chat Model → Summarization CV-s** (AI Language Model connection)
   - Connect inputs:
     - **Extract from File → Summarization CV-s**
     - **Extracting the content inside the Link → Summarization CV-s**
8. **Add Google Docs Tool for criteria**
   - Node: **Google Docs Tool** (`CV Criteria.`)
   - Operation: `get`
   - Document: use the criteria document (prefer full URL like `https://docs.google.com/document/d/<DOC_ID>/edit` or ensure the node accepts a doc ID)
   - Configure **Google Docs OAuth2 credentials**
9. **Add Comparator Agent**
   - Node: **AI Agent** (`CV COMPARATOR`)
   - Prompt: include instructions to compare summarized CV to criteria; reference the summarization output field that actually exists in your run (the workflow uses `{{$json.output.text}}`).
   - Connect:
     - Main: **Summarization CV-s → CV COMPARATOR**
     - Model: **Anthropic Chat Model → CV COMPARATOR** (AI Language Model)
     - Tool: **CV Criteria. → CV COMPARATOR** (AI Tool)
10. **Add Text Classifier**
   - Node: **Text Classifier** (`APPLICANT Classification`)
   - Input text: `{{$json.output}}` (adjust if comparator outputs a different field)
   - Categories:
     - `Classified` (suitable)
     - `Not Classified` (not suitable)
   - Connect:
     - Main: **CV COMPARATOR → APPLICANT Classification**
     - Model: **Anthropic Chat Model → APPLICANT Classification**
11. **Add Gmail Tool**
   - Node: **Gmail Tool** (`Send a message in Gmail`)
   - Operation: send message
   - Keep tool parameters as AI-driven (`$fromAI(...)`) so the agent can supply To/Subject/Message.
   - Configure **Gmail OAuth2 credentials**
12. **Add Google Calendar Tool**
   - Node: **Google Calendar Tool** (`Get availability in a calendar in Google Calendar`)
   - Resource: `calendar`
   - Select the correct calendar (replace placeholder `user@example.com`)
   - Configure **Google Calendar OAuth2 credentials**
13. **Add Positive Agent**
   - Node: **AI Agent** (`POSITIVE AGENT`)
   - Prompt: congratulate, propose interview slots Mon–Fri, use applicant name/email from `On form submission`.
   - Replace `{{$json.now}}` with `{{$now}}` (recommended) or add a Date & Time node to set `now`.
   - Connect:
     - Main: **APPLICANT Classification (positive output) → POSITIVE AGENT**
     - Model: **Anthropic Chat Model → POSITIVE AGENT**
     - Tools:
       - **Send a message in Gmail → POSITIVE AGENT** (AI Tool)
       - **Get availability in a calendar in Google Calendar → POSITIVE AGENT** (AI Tool)
14. **Add Negative Agent**
   - Node: **AI Agent** (`NEGATIVE AGENT`)
   - System message: rejection + motivation, HTML output, use Gmail tool, address applicant by name.
   - Connect:
     - Main: **APPLICANT Classification (negative output) → NEGATIVE AGENT**
     - Model: **Anthropic Chat Model → NEGATIVE AGENT**
     - Tool: **Send a message in Gmail → NEGATIVE AGENT** (AI Tool)
15. **(Optional but recommended) Add a “default/unmatched” Switch route**
   - Add a third Switch output for unknown MIME types and send an email asking for PDF/DOCX re-upload.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow relies on exact field labels with trailing spaces (e.g., `Upload CV `, `Phone Number `). Renaming fields requires updating expressions accordingly. | Form Trigger + Switch expressions |
| Binary property is set to `Upload_CV_`, which may not match the actual binary key produced by the Form Trigger. Validate by running once and checking the execution data. | PDF.co nodes + Extract From File |
| The Google Docs “DocumentURL” looks like a Doc ID; if your node version requires a full URL, use `https://docs.google.com/document/d/<ID>/edit`. | Google Docs Tool |
| `{{$json.now}}` is referenced but not created in this workflow; use `{{$now}}` or add a node to generate a timestamp. | POSITIVE AGENT |
| The positive flow mentions creating calendar appointments, but only an availability tool is configured; add a Calendar “create event” tool/node if you want real scheduling. | POSITIVE AGENT + Google Calendar Tool |
| Disclaimer (provided): “Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…” | Global |

