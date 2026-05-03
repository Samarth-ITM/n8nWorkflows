Collect, score, and coach call quality with forms, Gemini, OpenAI, Gmail, and Google Drive

https://n8nworkflows.xyz/workflows/collect--score--and-coach-call-quality-with-forms--gemini--openai--gmail--and-google-drive-12766


# Collect, score, and coach call quality with forms, Gemini, OpenAI, Gmail, and Google Drive

## 1. Workflow Overview

**Purpose:**  
This workflow collects a call recording and metadata via an n8n Form, stores the audio in Google Drive, downloads it for processing, transcribes it with **Google Gemini**, evaluates call quality with **OpenAI (GPT‑4o)** into a structured QA scorecard, then generates coaching feedback and emails it to the **supervisor** and **agent** via **Gmail**.

**Target use cases:**
- Contact center QA automation (scorecards, pass/fail, coaching)
- Compliance/risk flagging (PII, consent/disclosure checks)
- Lightweight performance management loops (agent + supervisor delivery)

### 1.1 Input Reception & File Handling
Form submission → upload audio to Drive → download it back as binary for transcription.

### 1.2 AI Transcription & QA Evaluation
Drive binary audio → Gemini transcription → GPT‑4o agent produces structured QA JSON (scores, sentiment, risk flags, coaching, next actions) validated by a structured output parser.

### 1.3 Feedback Composition & Email Delivery
QA JSON → GPT‑4o composes scorecard summary + email subject/body (validated by a second structured parser) → email supervisor summary → email agent with supervisor CC.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & File Handling
**Overview:** Collects agent/supervisor details and the call audio file, stores it in a designated Google Drive folder, then downloads the file as binary for the transcription model.

**Nodes involved:**
- Sticky Note1 (global description)
- Sticky Note (“Form & File Handling”)
- On form submission
- Upload file (Google Drive)
- Download file (Google Drive)

#### Sticky Note1 (global description)
- **Type / role:** Sticky Note (documentation)
- **Configuration:** Describes end-to-end intent: form intake → transcription → scoring → coaching → Gmail delivery.
- **Connections:** None (visual only)
- **Edge cases:** None (non-executable)

#### Sticky Note — “Form & File Handling”
- **Type / role:** Sticky Note (documentation)
- **Configuration:** Label for the intake/upload/download portion.
- **Connections:** None

#### On form submission
- **Type / role:** `formTrigger` — entry point; hosts an n8n Form and emits submitted form data.
- **Key configuration choices:**
  - Form title: **“Calls Recording Form”**
  - Fields (all required):
    - Agent Name (text)
    - Agent Email (text)
    - Call Id (text)
    - Call Date (date)
    - Call Recording File (file; accepts e.g. `.ogg`, `.mp3`)
    - Supervisor Name (text)
    - Supervisor Email (text)
  - Attribution disabled (`appendAttribution: false`)
- **Outputs:** One item containing the submitted fields; the file field is available for downstream upload.
- **Connections:**  
  - **Main → Upload file**
- **Failure modes / edge cases:**
  - Large file upload limits (depends on n8n hosting configuration)
  - Missing/invalid email addresses (later Gmail send may fail)
  - Unsupported audio format (Gemini transcription may fail later)

#### Upload file (Google Drive)
- **Type / role:** `googleDrive` — uploads the submitted recording to a specific Drive folder.
- **Key configuration choices:**
  - Operation: **Upload** (via input field)
  - Input binary field name: `Call_Recording_File` (matches the form file field name)
  - Destination folder: configured by a Google Drive folder URL (fixed folder)
  - Uploaded file name pattern (expression):
    - `{{ $json['Call Id'] }} - {{ $json['Agent Name'] }} - {{ $json['Call Date'] }}`
  - Drive: “My Drive”
- **Inputs:** Output of “On form submission” (includes file binary).
- **Outputs:** Google Drive file metadata (notably includes an `id` used for download).
- **Connections:**  
  - **Main → Download file**
- **Failure modes / edge cases:**
  - Google Drive OAuth2 token expired / insufficient permissions
  - Folder access revoked
  - Filename contains characters not accepted by Drive (rare; mostly safe)
- **Note:** The AI Analyst system message references `{{ $('Upload file').item.json.linkShareMetadata }}` as optional metadata; the Drive Upload node output typically does **not** include `linkShareMetadata` unless you explicitly request/enable sharing metadata operations. Expect this to be `undefined` unless extended.

#### Download file (Google Drive)
- **Type / role:** `googleDrive` — downloads the just-uploaded Drive file into binary data for Gemini.
- **Key configuration choices:**
  - Operation: **Download**
  - File ID: `{{ $json.id }}` (from Upload file output)
  - Binary property name: `{{ $json.originalFilename }}`
    - This stores the binary under a dynamic key (the original filename).
- **Inputs:** “Upload file” output (Drive metadata).
- **Outputs:** Same item with binary data attached (binary key named by `originalFilename`).
- **Connections:**  
  - **Main → Transcribe a recording**
- **Failure modes / edge cases:**
  - Download permission denied
  - `originalFilename` missing (binary property name expression fails or becomes empty)
  - Large audio may cause memory/time constraints

---

### Block 2 — AI Transcription & QA Evaluation
**Overview:** Transcribes the audio using Gemini, then runs a GPT‑4o-based QA Agent to generate a structured call-quality evaluation (scores, sentiment, compliance flags, coaching) enforced by a JSON schema parser.

**Nodes involved:**
- Sticky Note2 (“AI Processing”)
- Transcribe a recording (Gemini)
- GTP-4o (OpenAI chat model)
- Memory (Buffer Window)
- Structured Output Parser (QA schema)
- AI Quality Analyst (LangChain Agent)

#### Sticky Note2 — “AI Processing”
- **Type / role:** Sticky Note (documentation label)
- **Connections:** None

#### Transcribe a recording
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — uses Gemini for audio transcription.
- **Key configuration choices:**
  - Resource: **audio**
  - Model: `models/gemini-2.5-flash`
  - Input type: **binary**
  - Binary property name (expression): `{{ $json.name }}`
- **Inputs:** From “Download file” which stored the binary under `{{ $json.originalFilename }}`.
- **Outputs:** A Gemini response object; downstream nodes reference:
  - `{{ $json.content.parts[0].text }}` (transcript text)
- **Connections:**  
  - **Main → AI Quality Analyst**
- **Failure modes / edge cases:**
  - **Binary property mismatch risk:** Download stores binary under `originalFilename`, but transcription looks for binary under `$json.name`. If `$json.name` is not exactly the same as the binary key, Gemini will fail with “binary property not found”.
    - Typical fix: set Gemini `binaryPropertyName` to `{{ $json.originalFilename }}` or set Download binaryPropertyName to a fixed value like `audio`.
  - Unsupported codec/sample rate
  - Large file → timeouts
  - Gemini API auth/quota errors

#### GTP-4o
- **Type / role:** `lmChatOpenAi` — provides GPT‑4o as the language model for the QA agent.
- **Key configuration choices:**
  - Model: `gpt-4o`
- **Connections:**  
  - Provides **AI languageModel** input to “AI Quality Analyst”.
- **Failure modes / edge cases:**
  - OpenAI credential/quota issues
  - Model access restrictions by account
  - Latency/timeouts for long transcripts

#### Memory (Buffer Window)
- **Type / role:** `memoryBufferWindow` — conversational memory for the agent (keeps last N turns).
- **Key configuration choices:**
  - Session ID type: custom key
  - Session key expression:
    - `{{ $('Transcribe a recording').item.json.content.parts[0].text }}`
  - Context window length: 20
- **Connections:**  
  - Provides **AI memory** input to “AI Quality Analyst”.
- **Potential issues:**
  - Using the entire transcript as a session key can be very large and unstable. This can degrade performance or exceed internal limits.
  - Better practice: session key = call_id (from form) or Drive file id.

#### Structured Output Parser (QA schema)
- **Type / role:** `outputParserStructured` — enforces a JSON schema for the QA agent output.
- **Key configuration choices:**
  - Manual JSON Schema defining:
    - `call_id`, `language`, `overall_score`, `verdict`
    - `sections[]` (name, score, status, note)
    - `sentiment` (customer/agent)
    - `dynamics` (AHT, interruptions, holds, transfers)
    - `risk_flags` (PII, consent/disclosure, etc.)
    - `root_cause`, `tags[]`
    - `coaching[]`, `next_actions[]`, `evidence[]`
- **Connections:**  
  - Provides **AI outputParser** input to “AI Quality Analyst”.
- **Failure modes / edge cases:**
  - If the agent returns non-JSON or schema-noncompliant fields, parsing fails.
  - Several fields in the agent’s long system message (e.g., `timeline`, `risk_notes`, `root_cause_primary`) are **not present** in this schema. If the model follows the message rather than the parser schema, it may output extra fields or mismatched names (parser may reject or drop, depending on node behavior/version).

#### AI Quality Analyst
- **Type / role:** `agent` — LangChain agent that evaluates the transcript into a structured QA scorecard.
- **Key configuration choices:**
  - Input text: `{{ $json.content.parts[0].text }}`
  - System message: detailed QA rubric, multilingual rules, risk checks, coaching requirements.
  - Output parser enabled (`hasOutputParser: true`) → uses “Structured Output Parser”.
  - Streaming disabled.
- **Inputs:**
  - Main input from Gemini transcription node.
  - AI inputs:
    - Language model from “GTP-4o”
    - Memory from “Memory”
    - Output parser from “Structured Output Parser”
- **Outputs:** `item.json.output` containing the parsed structured QA JSON (to be used by the manager step).
- **Connections:**  
  - **Main → AI QC Manager**
- **Failure modes / edge cases:**
  - Transcript empty / missing: system message says to return an error object, but the schema does not include `error` fields → parser likely fails unless schema is updated.
  - Very long transcripts may exceed model context length and lead to truncation or incomplete evaluation.
  - Non-diarized transcripts (no Agent/Customer labels) reduce scoring confidence.

---

### Block 3 — Feedback Composition & Delivery
**Overview:** Converts the QA JSON into a human-readable scorecard summary and an email (subject/body), then emails the supervisor and the agent (with supervisor CC).

**Nodes involved:**
- Sticky Note3 (“Feedback & Delivery”)
- GPT-4o1 (OpenAI chat model)
- Structured Output Parser1 (email schema)
- AI QC Manager (LLM chain)
- Email_Supervisor (Gmail)
- Email_Agent_&_Supervisor (Gmail)

#### Sticky Note3 — “Feedback & Delivery”
- **Type / role:** Sticky Note (documentation label)
- **Connections:** None

#### GPT-4o1
- **Type / role:** `lmChatOpenAi` — provides GPT‑4o as the language model for the manager chain.
- **Key configuration choices:**
  - Model: `gpt-4o`
- **Connections:**  
  - Provides **AI languageModel** input to “AI QC Manager”.

#### Structured Output Parser1
- **Type / role:** `outputParserStructured` — enforces structured output for the manager’s response.
- **Key configuration choices:**
  - Example schema (not strict JSON schema here; it’s an example structure):
    - `scorecard_summary` (HTML/Markdown table)
    - `email_subject` (string)
    - `email_body` (string)
- **Connections:**  
  - Provides **AI outputParser** input to “AI QC Manager”.
- **Failure modes / edge cases:**
  - If model returns extra prose or invalid JSON, parsing fails.
  - If you truly need strict validation, define a full JSON schema (not only an example).

#### AI QC Manager
- **Type / role:** `chainLlm` — transforms QA JSON into a formatted scorecard + agent-facing email.
- **Key configuration choices:**
  - Input text: `{{ $json.output }}`
    - This expects the previous node (“AI Quality Analyst”) to output `item.json.output` containing the QA JSON.
  - Prompt: defines QC manager role and instructs generation of:
    - formatted scorecard summary
    - professional motivating email
    - overall rating and recommendation
  - Output parser enabled (`hasOutputParser: true`) using “Structured Output Parser1”.
- **Inputs:**
  - Main: QA evaluation object
  - AI: GPT-4o1 model + Structured Output Parser1
- **Outputs:** `item.json.output.scorecard_summary`, `item.json.output.email_subject`, `item.json.output.email_body`
- **Connections:**  
  - **Main → Email_Supervisor**
- **Failure modes / edge cases:**
  - If the incoming QA JSON differs from what the prompt expects (field names mismatch), the email may be incomplete.
  - If parser fails, downstream Gmail nodes will not have `output.*`.

#### Email_Supervisor
- **Type / role:** `gmail` — sends a summary email to the supervisor first.
- **Key configuration choices:**
  - To: `{{ $('On form submission').item.json['Supervisor Email'] }}`
  - Subject: `Call Summary {{ Agent Name }} - {{ Call Date }}`
  - Body (text email):
    - Greeting with supervisor name
    - Inserts `{{ $json.output.scorecard_summary }}`
    - Sign-off: “Quality Control Manager”
  - Attribution disabled
- **Inputs:** Output of “AI QC Manager”
- **Outputs:** Gmail send result
- **Connections:**  
  - **Main → Email_Agent_&_Supervisor**
- **Failure modes / edge cases:**
  - Gmail OAuth2 errors / token expired
  - Invalid recipient email
  - If `scorecard_summary` is Markdown but emailType is `text`, formatting will be plain text (expected, but may reduce readability)

#### Email_Agent_&_Supervisor
- **Type / role:** `gmail` — sends the main coaching email to the agent and CCs supervisor.
- **Key configuration choices:**
  - To: `{{ $('On form submission').item.json['Agent Email'] }}`
  - CC: `{{ $('On form submission').item.json['Supervisor Email'] }}`
  - Subject:
    - `{{ $('AI QC Manager').item.json.output.email_subject }} - {{ Agent Name }} - {{ Call Date }}`
  - Body (text email):
    - `{{ $('AI QC Manager').item.json.output.email_body }}`
  - Attribution disabled
- **Inputs:** Output of “Email_Supervisor” (but content is pulled by expression from “AI QC Manager”, so execution order matters but data source is explicit).
- **Outputs:** Gmail send result (final node).
- **Failure modes / edge cases:**
  - Same Gmail issues as above
  - If “AI QC Manager” failed, subject/body expressions evaluate to empty → email may send blank or fail depending on Gmail node validation.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note1 | stickyNote | High-level workflow description |  |  | ## Automate call quality scoring and coaching from a simple form<br><br>This workflow helps support teams evaluate call quality and deliver structured feedback without manual review. Agents upload their recordings using an n8n Form, and the system handles transcription, scoring, risk checks, and coaching delivery automatically using Gemini, OpenAI, Google Drive, and Gmail.<br><br>### How it works<br>**Form submission**<br>Agents submit their name, email, and call recording using an n8n Form. The file is stored securely in Google Drive.<br><br>**AI transcription**<br>Gemini converts the audio into a structured transcript for analysis.<br><br>**Performance scoring**<br>An AI Agent evaluates the conversation across key criteria such as empathy, clarity, accuracy, and policy compliance, producing a weighted score out of 100.<br><br>**Sentiment and risk detection**<br>The workflow identifies customer sentiment and flags potential issues like missing consent or sensitive data exposure.<br><br>**Coaching delivery**<br>A personalized performance summary is generated and sent automatically to both the agent and supervisor via Gmail. |
| On form submission | formTrigger | Collect metadata + audio file via n8n Form | (entry) | Upload file | ## Form & File Handling |
| Upload file | googleDrive | Upload submitted call recording to Drive folder | On form submission | Download file | ## Form & File Handling |
| Download file | googleDrive | Download Drive file as binary for transcription | Upload file | Transcribe a recording | ## Form & File Handling |
| Sticky Note | stickyNote | Block label |  |  | ## Form & File Handling |
| Transcribe a recording | googleGemini (LangChain) | Transcribe audio binary to text | Download file | AI Quality Analyst | ## AI Processing |
| GTP-4o | lmChatOpenAi (LangChain) | LLM provider for QA agent |  | AI Quality Analyst (ai_languageModel) | ## AI Processing |
| Memory | memoryBufferWindow (LangChain) | Conversation memory for QA agent |  | AI Quality Analyst (ai_memory) | ## AI Processing |
| Structured Output Parser | outputParserStructured (LangChain) | Enforce QA JSON schema |  | AI Quality Analyst (ai_outputParser) | ## AI Processing |
| AI Quality Analyst | agent (LangChain) | Evaluate transcript into structured QA scorecard | Transcribe a recording (+AI inputs) | AI QC Manager | ## AI Processing |
| GPT-4o1 | lmChatOpenAi (LangChain) | LLM provider for manager chain |  | AI QC Manager (ai_languageModel) | ## Feedback & Delivery |
| Structured Output Parser1 | outputParserStructured (LangChain) | Enforce email/summary structured output |  | AI QC Manager (ai_outputParser) | ## Feedback & Delivery |
| AI QC Manager | chainLlm (LangChain) | Create scorecard summary + email subject/body | AI Quality Analyst | Email_Supervisor | ## Feedback & Delivery |
| Email_Supervisor | gmail | Send score summary to supervisor | AI QC Manager | Email_Agent_&_Supervisor | ## Feedback & Delivery |
| Email_Agent_&_Supervisor | gmail | Send coaching email to agent (CC supervisor) | Email_Supervisor | (end) | ## Feedback & Delivery |
| Sticky Note2 | stickyNote | Block label |  |  | ## AI Processing |
| Sticky Note3 | stickyNote | Block label |  |  | ## Feedback & Delivery |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name it: *Collect, score, and coach call quality with n8n Forms, Gemini, OpenAI, and Gmail* (or your preferred title).

2. **Add Trigger: “On form submission” (Form Trigger)**
   - Set **Form Title**: `Calls Recording Form`
   - Set **Description**: “Add the call recording here, with Agent Name, Email and other details”
   - Add fields (all required):
     1) Agent Name (text)  
     2) Agent Email (text)  
     3) Call Id (text)  
     4) Call Date (date)  
     5) Call Recording File (file; allow `.ogg`, `.mp3`)  
     6) Supervisor Name (text)  
     7) Supervisor Email (text)
   - Disable attribution if desired.

3. **Add Google Drive node: “Upload file”**
   - Credentials: **Google Drive OAuth2** (grant access to the target Drive/folder).
   - Operation: **Upload**
   - Input data field name: `Call_Recording_File`
   - Folder: select or paste the folder URL you want.
   - File name (expression):
     - `{{ $json['Call Id'] }} - {{ $json['Agent Name'] }} - {{ $json['Call Date'] }}`

4. **Connect:** `On form submission → Upload file`

5. **Add Google Drive node: “Download file”**
   - Credentials: same Drive OAuth2
   - Operation: **Download**
   - File ID (expression): `{{ $json.id }}`
   - Set **Binary Property Name** to a stable value (recommended), e.g. `audio`
     - (This avoids the dynamic filename mismatch issue.)

6. **Connect:** `Upload file → Download file`

7. **Add Gemini node: “Transcribe a recording”**
   - Node: **Google Gemini (PaLM) / LangChain Gemini node** with **resource = audio**
   - Credentials: **Google Gemini(PaLM) API**
   - Model: `models/gemini-2.5-flash`
   - Input type: **binary**
   - Binary property name: set to the same key you used in Download (recommended `audio`)

8. **Connect:** `Download file → Transcribe a recording`

9. **Add OpenAI chat model node: “GTP-4o”**
   - Node: **OpenAI Chat Model (LangChain)**
   - Credentials: OpenAI API key
   - Model: `gpt-4o`

10. **Add Memory node: “Memory”**
   - Node: **Buffer Window Memory**
   - Context window length: `20`
   - Session key: use something stable, recommended:
     - `{{ $('On form submission').item.json['Call Id'] }}`
     - (Avoid using the transcript as the key.)

11. **Add Output Parser: “Structured Output Parser”**
   - Node: **Structured Output Parser**
   - Schema type: manual JSON Schema
   - Paste/define a schema that matches the fields you want (sections, sentiment, risk flags, etc.).  
   - Ensure the schema includes any error fields if you want robust missing-transcript handling.

12. **Add Agent node: “AI Quality Analyst”**
   - Node: **LangChain Agent**
   - Input text: `{{ $json.content.parts[0].text }}`
   - System message: insert your QA rubric and scoring rules (as in the workflow).
   - Enable output parsing and select **Structured Output Parser**.
   - Attach AI connections:
     - Language model: **GTP-4o**
     - Memory: **Memory**
     - Output parser: **Structured Output Parser**

13. **Connect:** `Transcribe a recording → AI Quality Analyst`

14. **Add OpenAI chat model node: “GPT-4o1”**
   - Same OpenAI credentials
   - Model: `gpt-4o`

15. **Add Output Parser: “Structured Output Parser1”**
   - Define a strict schema (recommended) with:
     - `scorecard_summary` (string)
     - `email_subject` (string)
     - `email_body` (string)

16. **Add LLM Chain node: “AI QC Manager”**
   - Node: **Chain LLM**
   - Input text: `{{ $json.output }}` (expects “AI Quality Analyst” output)
   - Prompt: instruct formatting of a scorecard and writing the email
   - Enable output parser and select **Structured Output Parser1**
   - Attach AI language model: **GPT-4o1**

17. **Connect:** `AI Quality Analyst → AI QC Manager`

18. **Add Gmail node: “Email_Supervisor”**
   - Credentials: **Gmail OAuth2**
   - Operation: **Send**
   - To: `{{ $('On form submission').item.json['Supervisor Email'] }}`
   - Subject: `Call Summary {{ $('On form submission').item.json['Agent Name'] }} - {{ $('On form submission').item.json['Call Date'] }}`
   - Email type: `text` (or HTML if you generate HTML)
   - Message body (example):
     - `Hi {{ $('On form submission').item.json['Supervisor Name'] }},\n\n{{ $json.output.scorecard_summary }}\n\nThanks,\nQuality Control Manager`

19. **Connect:** `AI QC Manager → Email_Supervisor`

20. **Add Gmail node: “Email_Agent_&_Supervisor”**
   - To: `{{ $('On form submission').item.json['Agent Email'] }}`
   - CC: `{{ $('On form submission').item.json['Supervisor Email'] }}`
   - Subject:
     - `{{ $('AI QC Manager').item.json.output.email_subject }} - {{ $('On form submission').item.json['Agent Name'] }} - {{ $('On form submission').item.json['Call Date'] }}`
   - Body:
     - `{{ $('AI QC Manager').item.json.output.email_body }}`
   - Email type: `text` (or HTML consistently)

21. **Connect:** `Email_Supervisor → Email_Agent_&_Supervisor`

22. **(Optional but recommended hardening)**
   - Add an **IF** node after transcription to ensure transcript exists before QA.
   - Add error handling branches for:
     - Drive upload/download failures
     - Gemini transcription failures
     - Parser failures (invalid JSON)
     - Gmail send failures

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is currently **inactive** (`active: false`). | Enable it in n8n to accept form submissions. |
| Potential binary key mismatch: Drive Download stores binary under `originalFilename`, but Gemini transcription reads `binaryPropertyName={{ $json.name }}`. | Align both nodes to use a single fixed binary property name (e.g., `audio`). |
| The QA agent prompt requests some fields not present in the enforced QA schema. | Align prompt fields with the Structured Output Parser schema to prevent parsing failures. |
| The memory session key is set to the full transcript text. | Replace with a stable identifier such as Call Id to avoid oversized keys and fragmentation. |
| Disclaimer provided by user (French): content comes exclusively from an n8n automated workflow and respects content policies. | Project/legal context note. |