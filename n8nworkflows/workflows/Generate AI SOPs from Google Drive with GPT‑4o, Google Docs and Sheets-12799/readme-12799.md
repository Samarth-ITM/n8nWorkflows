Generate AI SOPs from Google Drive with GPT‑4o, Google Docs and Sheets

https://n8nworkflows.xyz/workflows/generate-ai-sops-from-google-drive-with-gpt-4o--google-docs-and-sheets-12799


# Generate AI SOPs from Google Drive with GPT‑4o, Google Docs and Sheets

## 1. Workflow Overview

**Workflow name:** *Unstructured Documents to Structured SOP Converter*  
**Provided title:** *Generate AI SOPs from Google Drive with GPT‑4o, Google Docs and Sheets*

**Purpose:**  
Automatically watches a Google Drive folder for newly updated files, downloads the document, extracts its text, uses GPT‑4o (via LangChain nodes) to generate a structured SOP in JSON, refines it for quality, and—if confidence is above a threshold—creates a Google Doc, logs metadata to Google Sheets, and notifies stakeholders via Slack and Gmail.

**Primary use cases:**
- Converting messy internal docs (process notes, policies, runbooks) into standardized SOPs.
- Building an audit trail of SOP generation (Sheets log) and broadcasting new SOP availability (Slack/email).

### 1.1 Trigger & Configuration
Watches a specific Drive folder and sets workflow-wide parameters (threshold, folder IDs, sheet ID, Slack channel, stakeholder emails).

### 1.2 Text Acquisition & Normalization
Downloads the updated Drive file, extracts text (configured for PDF extraction), and normalizes fields needed downstream (text, name/id, timestamp).

### 1.3 AI SOP Generation & Refinement
Runs two AI passes: (1) generate SOP structure JSON, (2) refine and QA the SOP JSON while producing a confidence score.

### 1.4 Quality Gate & SOP Creation
Checks the confidence score against the threshold; if it passes, creates a Google Doc for the SOP.

### 1.5 Logging & Notifications
Writes/updates a row in Google Sheets and sends Slack + email notifications.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Trigger & Configuration

**Overview:**  
Starts the workflow when a file is updated in a watched Drive folder, then injects configuration constants used later (IDs, threshold, notification targets).

**Nodes involved:**
- New or Updated Document Trigger
- Workflow Configuration

#### Node: New or Updated Document Trigger
- **Type / role:** `Google Drive Trigger` (`n8n-nodes-base.googleDriveTrigger`) — polling trigger for Drive file updates.
- **Configuration (interpreted):**
  - Event: **fileUpdated**
  - Trigger scope: **specificFolder** (folder is expected but currently empty placeholder in JSON)
  - Polling: **every minute**
- **Key data produced:** Typical Drive trigger payload including `id`, `name`, etc. (downstream uses `id` and `name`).
- **Connections:**
  - **Output →** Workflow Configuration
- **Credentials:** Google Drive OAuth2 (“Google Drive account 2”)
- **Edge cases / failures:**
  - Folder not configured (`folderToWatch.value` is empty) → trigger won’t fire as intended.
  - API quota / polling limits; frequent polling can hit quota.
  - If non-supported file types are updated, later extraction may fail.

#### Node: Workflow Configuration
- **Type / role:** `Set` — centralizes constants and runtime parameters.
- **Configuration choices:**
  - Adds fields while keeping upstream fields (`includeOtherFields: true`).
  - Defines:
    - `qualityThreshold` (number): **0.8**
    - `sopFolderId` (string): placeholder for target SOP folder
    - `sheetsDocumentId` (string): placeholder for Sheets log doc
    - `slackChannel` (string): placeholder
    - `stakeholderEmails` (string): placeholder, comma-separated
- **Key expressions/variables:**
  - Values are literals/placeholders; referenced later via `$('Workflow Configuration').item.json.<field>`
- **Connections:**
  - **Input:** New or Updated Document Trigger
  - **Output →** Download Document from Drive
- **Edge cases / failures:**
  - Leaving placeholder values will cause downstream nodes to fail (Docs/Sheets/Slack/Gmail).
  - `stakeholderEmails` must be valid for Gmail node (comma-separated is fine, but invalid addresses can hard-fail or partially send depending on Gmail behavior).

---

### Block 1.2 — Text Acquisition & Normalization

**Overview:**  
Downloads the updated file from Drive into binary, extracts text (PDF mode), then shapes the payload into consistent fields required for AI and logging.

**Nodes involved:**
- Download Document from Drive
- Extract Text from Document
- Normalize Content and Metadata

#### Node: Download Document from Drive
- **Type / role:** `Google Drive` — downloads the file content.
- **Configuration choices:**
  - Operation: **download**
  - File ID: `={{ $json.id }}` (from the trigger item)
  - Binary property name: `data`
- **Connections:**
  - **Input:** Workflow Configuration (which still includes trigger fields due to includeOtherFields)
  - **Output →** Extract Text from Document
- **Credentials:** Google Drive OAuth2 (“Google Drive account 2”)
- **Edge cases / failures:**
  - If the trigger payload doesn’t include `id` (unexpected structure), expression fails.
  - Downloading Google native formats (Docs) sometimes requires export; direct download may return a non-PDF format or fail depending on node behavior and file type.
  - Large files may exceed memory/time limits.

#### Node: Extract Text from Document
- **Type / role:** `Extract From File` — converts binary into text.
- **Configuration choices:**
  - Operation: **pdf**
  - Expects binary input (by default typically reads `data`; relies on node defaults for binary property unless configured elsewhere).
- **Connections:**
  - **Input:** Download Document from Drive
  - **Output →** Normalize Content and Metadata
- **Edge cases / failures:**
  - If the downloaded file is not actually a PDF (e.g., DOCX, Google Doc export mismatch), extraction may return empty text or error.
  - Scanned PDFs without OCR: extracted text may be empty/garbled.
  - Very long PDFs may produce very large text which can exceed LLM context later.

#### Node: Normalize Content and Metadata
- **Type / role:** `Set` — creates canonical fields for downstream steps.
- **Configuration choices:**
  - Keeps other fields (`includeOtherFields: true`)
  - Sets:
    - `documentText` = `={{ $json.text }}` (from extractor)
    - `documentName` = `={{ $('New or Updated Document Trigger').item.json.name }}`
    - `documentId` = `={{ $('New or Updated Document Trigger').item.json.id }}`
    - `processedDate` = `={{ $now.toISO() }}`
- **Connections:**
  - **Input:** Extract Text from Document
  - **Output →** Generate SOP Structure
- **Edge cases / failures:**
  - If extractor output field name differs (not `text`), `documentText` becomes undefined.
  - If trigger item is not accessible by name (renamed node) expressions break.
  - Timezone/format: ISO string is stable, but downstream Sheets columns might expect a different format.

---

### Block 1.3 — AI SOP Generation & Refinement

**Overview:**  
Uses GPT‑4o twice: first to generate an SOP in a structured JSON format; second to refine and QA it, returning the same structure with a confidence score.

**Nodes involved:**
- Generate SOP Structure
- OpenAI Model for SOP Structure
- SOP Structure Output Parser
- Refine for Clarity and Consistency
- OpenAI Model for Refinement
- Refined SOP Output Parser

#### Node: SOP Structure Output Parser
- **Type / role:** `LangChain Structured Output Parser` — enforces/validates a JSON shape.
- **Configuration choices:**
  - Provides an example JSON schema including:
    - `title`, `purpose`, `scope`
    - `steps` (array of strings)
    - `responsibilities` (array of objects `{role, task}`)
    - `references` (array of strings)
    - `confidence` (number 0–1)
- **Connections:**
  - **Output (ai_outputParser) →** Generate SOP Structure (to provide parser to the agent)
- **Edge cases / failures:**
  - If the model output is not valid JSON or deviates from the schema, parsing fails.
  - Missing fields (e.g., no `confidence`) can cause downstream conditions to error.

#### Node: OpenAI Model for SOP Structure
- **Type / role:** `LangChain Chat Model (OpenAI)` — the LLM backing the first agent.
- **Configuration choices:**
  - Model: **gpt-4o**
- **Connections:**
  - **Output (ai_languageModel) →** Generate SOP Structure
- **Credentials:** OpenAI API (“n8n free OpenAI API credits”)
- **Edge cases / failures:**
  - Model access/credits/quota errors.
  - Request too large if `documentText` is long.

#### Node: Generate SOP Structure
- **Type / role:** `LangChain Agent` — prompts the model to generate SOP JSON.
- **Configuration choices:**
  - Input text: `={{ $json.documentText }}`
  - System message: instructs SOP generation and mandates “Return the structured SOP in the defined JSON format.”
  - `hasOutputParser: true` (expects structured parsing)
- **Connections:**
  - **Inputs:**
    - Main input from Normalize Content and Metadata
    - LLM input from OpenAI Model for SOP Structure
    - Output parser from SOP Structure Output Parser
  - **Output →** Refine for Clarity and Consistency
- **Edge cases / failures:**
  - If document text is empty/too short, the model may hallucinate; confidence may still appear high.
  - Parser failures halt execution.
  - Ambiguous instructions: “defined JSON format” is implied by parser example; if parser not correctly attached, output can be free-form.

#### Node: Refined SOP Output Parser
- **Type / role:** `LangChain Structured Output Parser` — validates refined SOP JSON.
- **Configuration choices:**
  - Same example schema as the first parser (includes `confidence`).
- **Connections:**
  - **Output (ai_outputParser) →** Refine for Clarity and Consistency
- **Edge cases / failures:** Same as other parser.

#### Node: OpenAI Model for Refinement
- **Type / role:** `LangChain Chat Model (OpenAI)` — LLM backing the refinement agent.
- **Configuration choices:**
  - Model: **gpt-4o**
- **Connections:**
  - **Output (ai_languageModel) →** Refine for Clarity and Consistency
- **Credentials:** OpenAI API
- **Edge cases / failures:** Same as other OpenAI node.

#### Node: Refine for Clarity and Consistency
- **Type / role:** `LangChain Agent` — QA/refinement pass and confidence assessment.
- **Configuration choices:**
  - Input text: `={{ JSON.stringify($json.output) }}`
    - Assumes previous agent output is located at `$json.output`.
  - System message: enforce clarity/consistency checks and return same JSON with confidence score (0–1).
  - `hasOutputParser: true`
- **Connections:**
  - **Input:** Generate SOP Structure
  - **Output →** Check Quality Threshold
- **Edge cases / failures:**
  - If prior node output is not in `$json.output`, `JSON.stringify($json.output)` becomes `undefined`/`null` and quality degrades or fails.
  - Confidence expected at `$json.output.confidence`; any mismatch breaks gating.

---

### Block 1.4 — Quality Gate & SOP Creation

**Overview:**  
Filters out low-confidence SOPs; if confidence meets threshold, creates a new Google Doc in the specified SOP folder.

**Nodes involved:**
- Check Quality Threshold
- Create SOP Document

#### Node: Check Quality Threshold
- **Type / role:** `IF` — gates the workflow based on confidence score.
- **Configuration choices:**
  - Condition: `={{ $json.output.confidence }}` **>=** `={{ $('Workflow Configuration').item.json.qualityThreshold }}`
  - Loose type validation enabled (n8n default here), helps if confidence is string-like.
- **Connections:**
  - **Input:** Refine for Clarity and Consistency
  - **True output →** Create SOP Document
  - **False output:** not connected (low-confidence items are dropped silently)
- **Edge cases / failures:**
  - If `output.confidence` missing or non-numeric, condition may evaluate unexpectedly or error.
  - No handling path for low confidence: consider adding a notification/log branch.

#### Node: Create SOP Document
- **Type / role:** `Google Docs` — creates a new Google Doc.
- **Configuration choices:**
  - Title: `SOP: {{ $json.output.title }}`
  - Folder ID: `={{ $('Workflow Configuration').item.json.sopFolderId }}`
- **Connections:**
  - **Input:** Check Quality Threshold (true branch)
  - **Output →** Log SOP Metadata
- **Credentials:** (Not shown in node JSON; Google Docs node typically needs Google OAuth2 credentials in n8n. If absent in UI, it will error at runtime.)
- **Edge cases / failures:**
  - If `sopFolderId` is invalid or missing permissions, doc creation fails.
  - This node creates a doc but **does not insert the SOP body** (no step that writes content into the Doc). Result: a blank doc with correct title unless defaults differ.
  - If `$json.output.title` missing, title becomes “SOP: ”.

---

### Block 1.5 — Logging & Notifications

**Overview:**  
Logs SOP metadata to a Google Sheet (append or update by Document ID), then posts a Slack message and emails stakeholders.

**Nodes involved:**
- Log SOP Metadata
- Notify Operations Team
- Email SOP to Stakeholders

#### Node: Log SOP Metadata
- **Type / role:** `Google Sheets` — appends or updates a row in a log sheet.
- **Configuration choices:**
  - Operation: **appendOrUpdate**
  - Spreadsheet ID: `={{ $('Workflow Configuration').item.json.sheetsDocumentId }}`
  - Sheet name: `SOP Log`
  - Matching column: **Document ID** (to update existing entries for the same source doc)
  - Columns written:
    - SOP Title = `={{ $json.output.title }}`
    - Document ID = `={{ $json.documentId }}`
    - Created Date = `={{ $json.processedDate }}`
    - Google Doc Link = `={{ $('Create SOP Document').item.json.documentUrl }}`
    - Confidence Score = `={{ $json.output.confidence }}`
    - Original Document Name = `={{ $json.documentName }}`
- **Connections:**
  - **Input:** Create SOP Document
  - **Output →** Notify Operations Team
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account 3”)
- **Edge cases / failures:**
  - Sheet/tab “SOP Log” must exist with compatible headers; mismatch can cause mapping issues.
  - If `documentUrl` is not returned by the Docs node (API/permissions), link will be empty.
  - If multiple updates happen quickly for same Document ID, updates can overwrite and appear “flappy”.

#### Node: Notify Operations Team
- **Type / role:** `Slack` — sends a channel message announcing a new SOP.
- **Configuration choices:**
  - Sends to channel: `={{ $('Workflow Configuration').item.json.slackChannel }}`
  - Message includes:
    - Title from `$('Create SOP Document').item.json.title`
    - Confidence Score from `$('Check Quality Threshold').item.json.confidenceScore` **(likely incorrect path)**
    - Document link from `$('Create SOP Document').item.json.documentUrl`
- **Connections:**
  - **Input:** Log SOP Metadata
  - **Output →** Email SOP to Stakeholders
- **Credentials:** Not shown; Slack node typically requires Slack OAuth/token configured in n8n.
- **Edge cases / failures / bugs:**
  - **Expression bug:** `$('Check Quality Threshold').item.json.confidenceScore` is not produced by the IF node. The confidence is at `$json.output.confidence` (from AI). This will render empty/undefined in Slack.
  - If `slackChannel` is invalid, Slack API returns channel_not_found.
  - Message includes emoji characters; if your environment restricts, it’s usually still fine.

#### Node: Email SOP to Stakeholders
- **Type / role:** `Gmail` — emails stakeholders the SOP details.
- **Configuration choices:**
  - To: `={{ $('Workflow Configuration').item.json.stakeholderEmails }}`
  - Subject: `New SOP Created: {{ $json.output.title }}`
  - HTML body includes SOP title/purpose/scope/link and a “Confidence Score” field referencing `{{ $json.output.confidenceScore }}` **(likely incorrect key)**
- **Connections:**
  - **Input:** Notify Operations Team
  - **Output:** none (end)
- **Credentials:** Gmail OAuth2 (“Gmail account 4”)
- **Edge cases / failures / bugs:**
  - **Expression bug:** confidence is `output.confidence`, not `output.confidenceScore`. Email will show blank unless the model returns both.
  - Gmail sending limits / OAuth scope issues.
  - Comma-separated recipients usually work, but malformed list can fail.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New or Updated Document Trigger | Google Drive Trigger | Poll Drive folder for updated files | — | Workflow Configuration | ## Trigger & Config |
| Workflow Configuration | Set | Define threshold + IDs for Drive/Sheets/Slack/Email | New or Updated Document Trigger | Download Document from Drive | ## Trigger & Config |
| Download Document from Drive | Google Drive | Download updated file as binary | Workflow Configuration | Extract Text from Document | ## Text Data |
| Extract Text from Document | Extract From File | Extract text from binary (PDF mode) | Download Document from Drive | Normalize Content and Metadata | ## Text Data |
| Normalize Content and Metadata | Set | Standardize fields (text/name/id/date) | Extract Text from Document | Generate SOP Structure | ## Text Data |
| Generate SOP Structure | LangChain Agent | Generate SOP JSON from text | Normalize Content and Metadata + OpenAI Model for SOP Structure + SOP Structure Output Parser | Refine for Clarity and Consistency | ## AI Logic |
| OpenAI Model for SOP Structure | OpenAI Chat Model (LangChain) | LLM for initial SOP generation | — | Generate SOP Structure | ## AI Logic |
| SOP Structure Output Parser | Structured Output Parser (LangChain) | Enforce SOP JSON schema | — | Generate SOP Structure | ## AI Logic |
| Refine for Clarity and Consistency | LangChain Agent | Refine SOP JSON + assign confidence | Generate SOP Structure + OpenAI Model for Refinement + Refined SOP Output Parser | Check Quality Threshold | ## AI Logic |
| OpenAI Model for Refinement | OpenAI Chat Model (LangChain) | LLM for SOP refinement | — | Refine for Clarity and Consistency | ## AI Logic |
| Refined SOP Output Parser | Structured Output Parser (LangChain) | Enforce refined SOP JSON schema | — | Refine for Clarity and Consistency | ## AI Logic |
| Check Quality Threshold | If | Gate on `output.confidence >= threshold` | Refine for Clarity and Consistency | Create SOP Document (true branch) | ## SOP |
| Create SOP Document | Google Docs | Create SOP doc in target folder | Check Quality Threshold (true) | Log SOP Metadata | ## SOP |
| Log SOP Metadata | Google Sheets | Append/update SOP log row | Create SOP Document | Notify Operations Team | ## Log & Notify |
| Notify Operations Team | Slack | Post Slack announcement | Log SOP Metadata | Email SOP to Stakeholders | ## Log & Notify |
| Email SOP to Stakeholders | Gmail | Email SOP summary to stakeholders | Notify Operations Team | — | ## Log & Notify |
| Sticky Note | Sticky Note | Visual label | — | — |  |
| Sticky Note1 | Sticky Note | Visual label | — | — |  |
| Sticky Note2 | Sticky Note | Visual label | — | — |  |
| Sticky Note3 | Sticky Note | Visual label | — | — |  |
| Sticky Note4 | Sticky Note | Visual label | — | — |  |
| Sticky Note5 | Sticky Note | Visual/project notes | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name it: *Unstructured Documents to Structured SOP Converter* (or your preferred name).

2. **Add Google Drive Trigger**
   - Node: **Google Drive Trigger**
   - Event: **File Updated**
   - Trigger on: **Specific Folder**
   - Select the folder that contains “raw/unstructured” documents.
   - Polling interval: **Every minute** (adjust for quotas).
   - **Credentials:** Connect Google Drive OAuth2.

3. **Add “Workflow Configuration” (Set node)**
   - Node: **Set**
   - Enable **Include Other Fields**.
   - Add fields:
     - `qualityThreshold` (Number) = `0.8`
     - `sopFolderId` (String) = *(Google Drive folder ID where SOP docs should be created)*
     - `sheetsDocumentId` (String) = *(Google Sheets spreadsheet ID for logging)*
     - `slackChannel` (String) = *(channel name or ID)*
     - `stakeholderEmails` (String) = *(comma-separated emails)*
   - Connect: **Drive Trigger → Workflow Configuration**

4. **Add “Download Document from Drive” (Google Drive node)**
   - Node: **Google Drive**
   - Operation: **Download**
   - File ID: `{{$json.id}}`
   - Options → Binary Property: `data`
   - **Credentials:** Google Drive OAuth2
   - Connect: **Workflow Configuration → Download Document from Drive**

5. **Add “Extract Text from Document”**
   - Node: **Extract From File**
   - Operation: **PDF**
   - (If the node allows selecting the binary property, set it to `data` to match the download node.)
   - Connect: **Download Document from Drive → Extract Text from Document**

6. **Add “Normalize Content and Metadata” (Set node)**
   - Node: **Set**
   - Enable **Include Other Fields**.
   - Add fields:
     - `documentText` = `{{$json.text}}`
     - `documentName` = `{{$('New or Updated Document Trigger').item.json.name}}`
     - `documentId` = `{{$('New or Updated Document Trigger').item.json.id}}`
     - `processedDate` = `{{$now.toISO()}}`
   - Connect: **Extract Text from Document → Normalize Content and Metadata**

7. **Add AI components for SOP generation**
   - Add node: **OpenAI Chat Model (LangChain)**  
     - Model: **gpt-4o**
     - **Credentials:** OpenAI API key/credits in n8n.
   - Add node: **Structured Output Parser (LangChain)**  
     - Provide an example JSON with fields: `title, purpose, scope, steps[], responsibilities[{role,task}], references[], confidence`
   - Add node: **AI Agent (LangChain)** named “Generate SOP Structure”
     - Text/input: `{{$json.documentText}}`
     - System message: SOP generator instructions (as in workflow)
     - Ensure it is configured to use:
       - The **OpenAI model** node as its Language Model connection
       - The **Structured Output Parser** as its Output Parser connection
   - Connect:
     - **Normalize Content and Metadata → Generate SOP Structure**
     - **OpenAI Model for SOP Structure → Generate SOP Structure** (AI language model connection)
     - **SOP Structure Output Parser → Generate SOP Structure** (AI output parser connection)

8. **Add AI components for refinement**
   - Add node: **OpenAI Chat Model (LangChain)** (second one)
     - Model: **gpt-4o**
   - Add node: **Structured Output Parser (LangChain)** (second one, same schema)
   - Add node: **AI Agent (LangChain)** named “Refine for Clarity and Consistency”
     - Text/input: `{{ JSON.stringify($json.output) }}`
     - System message: QA/refinement instructions
     - Attach the second OpenAI model + second parser.
   - Connect:
     - **Generate SOP Structure → Refine for Clarity and Consistency**
     - **OpenAI Model for Refinement → Refine for Clarity and Consistency** (AI language model)
     - **Refined SOP Output Parser → Refine for Clarity and Consistency** (AI output parser)

9. **Add “Check Quality Threshold” (IF node)**
   - Node: **IF**
   - Condition (Number):  
     - Left: `{{$json.output.confidence}}`  
     - Operator: **Greater than or equal (>=)**  
     - Right: `{{$('Workflow Configuration').item.json.qualityThreshold}}`
   - Connect: **Refine for Clarity and Consistency → Check Quality Threshold**
   - (Optional but recommended) Add a “false branch” handler (Slack/email/log) for low confidence.

10. **Add “Create SOP Document” (Google Docs)**
   - Node: **Google Docs**
   - Title: `SOP: {{$json.output.title}}`
   - Folder ID: `{{$('Workflow Configuration').item.json.sopFolderId}}`
   - **Credentials:** Connect Google Docs (Google OAuth2 with Docs scope).
   - Connect: **Check Quality Threshold (true) → Create SOP Document**
   - Note: If you want the SOP content inside the Doc, add an additional Google Docs “Append/Insert content” step (not present in the provided workflow).

11. **Add “Log SOP Metadata” (Google Sheets)**
   - Node: **Google Sheets**
   - Operation: **Append or Update**
   - Spreadsheet ID: `{{$('Workflow Configuration').item.json.sheetsDocumentId}}`
   - Sheet name: `SOP Log` (create this tab in advance)
   - Matching column: `Document ID`
   - Map columns:
     - SOP Title = `{{$json.output.title}}`
     - Document ID = `{{$json.documentId}}`
     - Created Date = `{{$json.processedDate}}`
     - Google Doc Link = `{{$('Create SOP Document').item.json.documentUrl}}`
     - Confidence Score = `{{$json.output.confidence}}`
     - Original Document Name = `{{$json.documentName}}`
   - **Credentials:** Google Sheets OAuth2
   - Connect: **Create SOP Document → Log SOP Metadata**

12. **Add “Notify Operations Team” (Slack)**
   - Node: **Slack**
   - Send to: **Channel**
   - Channel: `{{$('Workflow Configuration').item.json.slackChannel}}`
   - Compose message using doc title/link and confidence.
   - **Credentials:** Slack OAuth2/token.
   - Connect: **Log SOP Metadata → Notify Operations Team**
   - Recommended fix: use `{{$json.output.confidence}}` rather than referencing the IF node.

13. **Add “Email SOP to Stakeholders” (Gmail)**
   - Node: **Gmail**
   - To: `{{$('Workflow Configuration').item.json.stakeholderEmails}}`
   - Subject: `New SOP Created: {{$json.output.title}}`
   - Body (HTML): include title/purpose/scope/confidence + doc link.
   - **Credentials:** Gmail OAuth2 (with send scope).
   - Connect: **Notify Operations Team → Email SOP to Stakeholders**
   - Recommended fix: use `{{$json.output.confidence}}` (not `confidenceScore`).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Main**: This workflow converts messy internal documents into clean, structured SOPs using AI. Designed for teams with knowledge scattered across docs/notes needing standardized procedures without manual rewriting. | Sticky note “Main” |
| **Built by:** Hyrum Hurst — AI Automation Engineer, QuarterSmart. **Contact:** hyrum@quartersmart.com | Sticky note “Main” |
| **Setup notes:** 1) Connect Google Drive and Docs 2) Choose source folder 3) Configure AI tone and SOP format 4) Set notification channels 5) Review generated SOPs before rollout | Sticky note “Main” |
| Visual block labels: “Trigger & Config”, “Text Data”, “AI Logic”, “SOP”, “Log & Notify” | Sticky notes labeling sections in canvas |

**Important implementation gaps to be aware of (from the provided workflow):**
- The Google Doc is created but the SOP content is not written into it (no “insert content” step).
- Slack and Gmail nodes reference `confidenceScore`, but the AI schema uses `confidence`. This should be aligned (either change the schema to include `confidenceScore` or fix expressions to use `output.confidence`).