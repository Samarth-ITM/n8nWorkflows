Score call quality and send coaching emails with Gemini, OpenAI, and Gmail

https://n8nworkflows.xyz/workflows/score-call-quality-and-send-coaching-emails-with-gemini--openai--and-gmail-12765


# Score call quality and send coaching emails with Gemini, OpenAI, and Gmail

## 1. Workflow Overview

**Title:** Score call quality and send coaching emails with Gemini, OpenAI, and Gmail  
**Workflow name (in JSON):** Automate call quality scoring and coaching with Gemini, OpenAI, and Gmail  
**Purpose:** Automatically monitor a Google Drive folder for newly uploaded call recordings, transcribe them with Google Gemini, evaluate call quality with an OpenAI-powered QA agent, generate a coaching email + scorecard summary, and send the email via Gmail.

### 1.1 Input Reception & File Handling
Watches a specific Google Drive folder and downloads any newly created file (typically call audio).

### 1.2 AI Processing (Transcription → QA Scoring)
Transcribes audio using Gemini, then runs a LangChain Agent (“AI Quality Analyst”) backed by GPT‑4o to output a structured JSON scorecard.

### 1.3 Output Formatting & Delivery
A second LLM chain (“QC Manager”) converts the scorecard into a formatted scorecard summary and a coaching email (structured output), then sends it via Gmail.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & File Handling
**Overview:** Detects new files in a target Drive folder and downloads the file content as binary for downstream transcription.  
**Nodes involved:** Google Drive Trigger, Download file

#### Node: Google Drive Trigger
- **Type / role:** `googleDriveTrigger` — polling trigger for newly created Drive files.
- **Configuration (interpreted):**
  - Event: **fileCreated**
  - Trigger scope: **specific folder**
  - Folder URL: `https://drive.google.com/drive/folders/1FfTNxyRZ4S4Ae_XVfDW50S4uHVB5e-FZ`
  - Polling: **every minute**
  - File type: **all**
- **Key expressions/variables:** None (emits file metadata, including `id`).
- **Connections:**
  - **Output →** Download file (main)
- **Credentials:** Google Drive OAuth2 (“Google Drive account”)
- **Failure/edge cases:**
  - OAuth expiration / insufficient permissions to the folder
  - High-frequency uploads could cause missed items if polling overlaps or rate limits apply
  - Non-audio files will still trigger (since `fileType=all`) and may break later transcription

#### Node: Download file
- **Type / role:** `googleDrive` — downloads the newly created file.
- **Configuration (interpreted):**
  - Operation: **download**
  - File ID: `={{ $json.id }}` (from trigger output)
  - Binary property name: `data` (downloaded content stored in `binary.data`)
- **Connections:**
  - **Input ←** Google Drive Trigger
  - **Output →** Transcribe (main)
- **Credentials:** Google Drive OAuth2 (“Google Drive account”)
- **Failure/edge cases:**
  - File not accessible (permissions, shared-drive quirks)
  - Large file download timeouts
  - Non-audio content downloaded successfully but fails transcription
  - Missing/changed metadata fields used later (e.g., permissions array in downstream prompt)

---

### Block 2 — AI Processing (Transcription → QA Scoring)
**Overview:** Converts audio to transcript via Gemini, then uses a GPT‑4o-backed agent with memory and a strict JSON schema output parser to produce a machine-readable evaluation.  
**Nodes involved:** Transcribe, GPT-4o, Memory, Structured Output Parser, AI Quality Analyst

#### Node: Transcribe
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — Gemini audio resource transcription.
- **Configuration (interpreted):**
  - Resource: **audio**
  - Input type: **binary**
  - Model: `models/gemini-2.5-flash`
  - Uses incoming binary (from Download file) as audio input
- **Key expressions/variables:** None explicitly; output used downstream as `content.parts[0].text`.
- **Connections:**
  - **Input ←** Download file (main)
  - **Output →** AI Quality Analyst (main)
- **Credentials:** Google Gemini(PaLM) API account
- **Failure/edge cases:**
  - Unsupported audio codec/container
  - Audio too large / too long (model limits)
  - Empty/low-quality audio producing empty transcript
  - API quota/rate limiting

#### Node: GPT-4o
- **Type / role:** `lmChatOpenAi` — language model provider for the QA agent.
- **Configuration (interpreted):**
  - Model: **gpt-4o**
  - Options: default
- **Connections:**
  - **AI language model →** AI Quality Analyst
- **Credentials:** OpenAI API account
- **Failure/edge cases:**
  - OpenAI auth or quota errors
  - Model name not available in region/account

#### Node: Memory
- **Type / role:** `memoryBufferWindow` — conversation memory buffer for the agent.
- **Configuration (interpreted):**
  - Session key (custom): `={{ $('Transcribe').item.json.content.parts[0].text }}`
  - Context window length: **20**
- **Connections:**
  - **AI memory →** AI Quality Analyst
- **Important note (edge case):**
  - Using the **entire transcript text as a session key** can be problematic:
    - Very large keys (performance/storage)
    - Slight transcript changes create new sessions (no reuse)
    - Potentially sensitive content in the session identifier
  - Expression dependency: if `Transcribe` output path changes or is empty, memory may fail.

#### Node: Structured Output Parser
- **Type / role:** `outputParserStructured` — enforces a strict JSON schema for the QA output.
- **Configuration (interpreted):**
  - Manual JSON schema defining fields like:
    - `call_id`, `language`, `overall_score`, `verdict`
    - `sections[]` with pass/fail/na
    - sentiment, dynamics, risk_flags, root_cause, tags, coaching, next_actions, evidence
- **Connections:**
  - **AI output parser →** AI Quality Analyst
- **Failure/edge cases:**
  - If the agent returns invalid JSON or missing required structure, parsing fails
  - Enum mismatches (e.g., verdict not exactly `pass|fail`)
  - Integer type mismatches (model outputs string)

#### Node: AI Quality Analyst
- **Type / role:** `agent` — LangChain agent that evaluates transcript and produces structured QA JSON.
- **Configuration (interpreted):**
  - Input text: `={{ $json.content.parts[0].text }}` (from Transcribe output)
  - System message: detailed rubric for contact center QA, multilingual handling, compliance flags, coaching, and strict JSON-only output.
  - Output parser enabled: **Yes** (Structured Output Parser)
  - Streaming: disabled
- **Key expressions/variables used in prompt:**
  - Transcript: `{{ $json.content.parts[0].text }}`
  - Optional metadata reference: `{{ $('Download file').item.json.linkShareMetadata }}`
  - Policy snippets: `{{ $json.policy }}` (not produced elsewhere in this workflow; may be undefined)
- **Connections:**
  - **Input ←** Transcribe (main)
  - **Uses:** GPT‑4o (ai_languageModel), Memory (ai_memory), Structured Output Parser (ai_outputParser)
  - **Output →** AI Quality Control (QC) Manager (main)
- **Failure/edge cases:**
  - If `policy` is not present, the prompt references it but it will be empty/undefined (usually safe, but can confuse output)
  - Transcript missing/empty: system message asks to return an error object, but the schema shown in the parser does not include an `error` field—this can cause parser failures if the agent follows the “return error” instruction.
  - Any mismatch between the system prompt’s “schema below” and the actual parser schema can cause invalid output.

---

### Block 3 — Output Formatting & Delivery
**Overview:** Converts the structured QA JSON into an email subject/body and sends it via Gmail.  
**Nodes involved:** AI Quality Control (QC) Manager, GPT-4o1, Structured Output Parser1, Send a message

#### Node: GPT-4o1
- **Type / role:** `lmChatOpenAi` — language model provider for the QC Manager chain.
- **Configuration (interpreted):**
  - Model: **gpt-4o**
- **Connections:**
  - **AI language model →** AI Quality Control (QC) Manager
- **Credentials:** OpenAI API account
- **Failure/edge cases:** Same as GPT-4o node (auth/quota/model availability)

#### Node: Structured Output Parser1
- **Type / role:** `outputParserStructured` — ensures the QC Manager returns the expected email payload.
- **Configuration (interpreted):**
  - Schema example (not strict manual schema here, but used to guide structure):
    - `scorecard_summary` (HTML/Markdown table)
    - `email_subject`
    - `email_body`
- **Connections:**
  - **AI output parser →** AI Quality Control (QC) Manager
- **Failure/edge cases:**
  - If the chain returns content not matching the expected JSON keys, parsing/consumption may fail downstream (especially `Send a message` expects `$json.output.email_*`).

#### Node: AI Quality Control (QC) Manager
- **Type / role:** `chainLlm` — takes QA JSON and generates a formatted scorecard + coaching email.
- **Configuration (interpreted):**
  - Input text: `={{ $json.output }}` (expects previous node to provide parsed output under `output`)
  - Prompt: role instructions and an example input JSON including expressions referencing Google Drive file metadata.
  - Output parser enabled: **Yes** (Structured Output Parser1)
- **Key expressions referenced inside the prompt example:**
  - `{{ $('Download file').item.json.name }}` as `agent_name` (this is actually the file name; may not be a real agent name)
  - `{{ $('Download file').item.json.permissions[1].displayName }}` as supervisor name (fragile: permissions array may not have index 1)
  - `{{ $('Download file').item.json.id }}` as call_id
  - `{{$now}}` as call_date
- **Connections:**
  - **Input ←** AI Quality Analyst (main)
  - **Uses:** GPT-4o1 (ai_languageModel), Structured Output Parser1 (ai_outputParser)
  - **Output →** Send a message (main)
- **Failure/edge cases:**
  - If AI Quality Analyst output is not located at `$json.output`, this will break (path mismatch).
  - `permissions[1]` may be undefined → results in blank supervisor name or expression error depending on n8n settings.
  - If Google Drive file metadata does not include `permissions` (depends on Drive API scopes and node output settings).

#### Node: Send a message
- **Type / role:** `gmail` — sends the coaching email.
- **Configuration (interpreted):**
  - To: `user@example.com` (static; should be replaced with agent email logic)
  - Subject: `={{ $json.output.email_subject }}`
  - Message: `={{ $json.output.email_body }}`
  - Email type: **text**
  - Attribution: disabled
- **Connections:**
  - **Input ←** AI Quality Control (QC) Manager
- **Credentials:** Gmail OAuth2 (“Gmail account”)
- **Failure/edge cases:**
  - Gmail OAuth token expiration / missing scopes (send email)
  - Sending limits / rate limiting
  - If QC Manager output isn’t in `$json.output.email_*`, subject/body will be empty or expression will fail
  - If you intend HTML emails, `emailType` is currently set to `text`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Google Drive Trigger | n8n-nodes-base.googleDriveTrigger | Detect new files in a Drive folder | — | Download file | ## Input & File Handling |
| Download file | n8n-nodes-base.googleDrive | Download new Drive file as binary | Google Drive Trigger | Transcribe | ## Input & File Handling |
| Transcribe | @n8n/n8n-nodes-langchain.googleGemini | Transcribe audio to text | Download file | AI Quality Analyst | ## AI Processing |
| GPT-4o | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for QA agent | — | AI Quality Analyst (ai_languageModel) | ## AI Processing |
| Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Agent memory (buffer window) | — | AI Quality Analyst (ai_memory) | ## AI Processing |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce QA JSON schema | — | AI Quality Analyst (ai_outputParser) | ## AI Processing |
| AI Quality Analyst | @n8n/n8n-nodes-langchain.agent | Evaluate transcript, score call, produce JSON | Transcribe | AI Quality Control (QC) Manager | ## AI Processing |
| GPT-4o1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for QC/email generation | — | AI Quality Control (QC) Manager (ai_languageModel) | ## Output & Delivery |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce email payload structure | — | AI Quality Control (QC) Manager (ai_outputParser) | ## Output & Delivery |
| AI Quality Control (QC) Manager | @n8n/n8n-nodes-langchain.chainLlm | Convert QA JSON into scorecard + coaching email | AI Quality Analyst | Send a message | ## Output & Delivery |
| Send a message | n8n-nodes-base.gmail | Send coaching email via Gmail | AI Quality Control (QC) Manager | — | ## Output & Delivery |
| Sticky Note | n8n-nodes-base.stickyNote | Workflow description / setup notes | — | — | ## Automate call quality scoring and coaching with AI  \nThis workflow helps contact centers and support teams automatically evaluate call quality and send coaching feedback without manual review. It uses AI to transcribe calls, score performance, detect risks, and generate clear feedback for agents.  \n**How it works**  \nFetch recordings  \nThe workflow watches a Google Drive folder and downloads new call audio files automatically.  \n**Transcribe with AI**  \nEach recording is converted into a structured transcript using the Gemini model.  \n**Score performance**  \nAn AI Agent evaluates the transcript against key criteria such as empathy, solution quality, clarity, and policy adherence.  \n**Detect risks and insights**  \nThe workflow flags potential issues (e.g. missing consent or sensitive data) and extracts sentiment for both agent and customer.  \n**Send coaching email**  \nA personalized performance summary with strengths and improvement points is generated and sent to the agent via Gmail.  \n**Setup steps**  \nConnect Google Drive and select your recordings folder  \nAdd your Gemini API key for transcription  \nAdd your OpenAI key for scoring and feedback generation  \nConnect Gmail to send automated coaching emails  \n(Optional) Customize scoring criteria inside the AI Agent node |
| Sticky Note1 | n8n-nodes-base.stickyNote | Block label | — | — | ## Input & File Handling |
| Sticky Note2 | n8n-nodes-base.stickyNote | Block label | — | — | ## AI Processing |
| Sticky Note3 | n8n-nodes-base.stickyNote | Block label | — | — | ## Output & Delivery |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: Google Drive Trigger**
   - Type: *Google Drive Trigger*
   - Event: *File Created*
   - Trigger on: *Specific Folder*
   - Folder to watch: paste the folder URL (or select via picker)
   - Polling: *Every minute*
   - Credentials: connect **Google Drive OAuth2** with access to that folder
3. **Add node: Download file**
   - Type: *Google Drive*
   - Operation: *Download*
   - File ID: use expression `{{ $json.id }}` (from trigger)
   - Options → Binary Property Name: `data`
   - Credentials: same Google Drive OAuth2
   - Connect: **Google Drive Trigger → Download file**
4. **Add node: Transcribe**
   - Type: *Google Gemini (LangChain)*
   - Resource: *Audio*
   - Input type: *Binary*
   - Model: `models/gemini-2.5-flash`
   - Credentials: set up **Google Gemini / PaLM API** credential
   - Connect: **Download file → Transcribe**
5. **Add node: Structured Output Parser (QA)**
   - Type: *Structured Output Parser (LangChain)*
   - Schema type: *Manual*
   - Paste the JSON schema that defines the QA result (call_id, overall_score, sections, sentiment, risk_flags, coaching, etc.)
6. **Add node: Memory**
   - Type: *Memory Buffer Window*
   - Session ID type: *Custom Key*
   - Session key expression: `{{ $('Transcribe').item.json.content.parts[0].text }}`
   - Context window length: `20`
7. **Add node: GPT-4o (LLM)**
   - Type: *OpenAI Chat Model (LangChain)*
   - Model: `gpt-4o`
   - Credentials: configure **OpenAI API**
8. **Add node: AI Quality Analyst**
   - Type: *AI Agent (LangChain)*
   - Prompt type: *Define*
   - Text input: `{{ $json.content.parts[0].text }}`
   - System message: use the provided QA analyst rubric (ensure it matches the parser schema)
   - Enable output parser: **true**
   - Connect AI inputs:
     - **GPT-4o → AI Quality Analyst** (ai_languageModel)
     - **Memory → AI Quality Analyst** (ai_memory)
     - **Structured Output Parser → AI Quality Analyst** (ai_outputParser)
   - Connect main flow: **Transcribe → AI Quality Analyst**
9. **Add node: Structured Output Parser1 (Email payload)**
   - Type: *Structured Output Parser (LangChain)*
   - Provide a schema/example with: `scorecard_summary`, `email_subject`, `email_body`
10. **Add node: GPT-4o1**
   - Type: *OpenAI Chat Model (LangChain)*
   - Model: `gpt-4o`
   - Credentials: same OpenAI credential (or another)
11. **Add node: AI Quality Control (QC) Manager**
   - Type: *LLM Chain (LangChain)*
   - Prompt type: *Define*
   - Input text: `{{ $json.output }}`
   - Messages: include the QC Manager role prompt and the example JSON (with expressions referencing Download file metadata)
   - Enable output parser: **true**
   - Connect AI inputs:
     - **GPT-4o1 → AI Quality Control (QC) Manager** (ai_languageModel)
     - **Structured Output Parser1 → AI Quality Control (QC) Manager** (ai_outputParser)
   - Connect main flow: **AI Quality Analyst → AI Quality Control (QC) Manager**
12. **Add node: Send a message**
   - Type: *Gmail*
   - Operation: *Send*
   - To: set a test email first (replace later with dynamic agent email)
   - Subject: `{{ $json.output.email_subject }}`
   - Message: `{{ $json.output.email_body }}`
   - Email type: `text` (switch to HTML if you generate HTML)
   - Credentials: connect **Gmail OAuth2** with send permissions
   - Connect: **AI Quality Control (QC) Manager → Send a message**
13. **Validate with a test**
   - Upload an audio file into the watched Drive folder.
   - Confirm the workflow runs end-to-end and that `$json.output.email_subject` / `$json.output.email_body` exist at the Gmail node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automate call quality scoring and coaching with AI: watches a Google Drive folder, transcribes with Gemini, scores with an AI agent, flags risks, and emails coaching via Gmail. Setup: connect Drive, Gemini API, OpenAI, Gmail; customize scoring criteria in AI Agent. | Workflow sticky note (global description) |