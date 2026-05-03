Collect LinkedIn details and generate CV feedback with Gemini and Google Workspace

https://n8nworkflows.xyz/workflows/collect-linkedin-details-and-generate-cv-feedback-with-gemini-and-google-workspace-12888


# Collect LinkedIn details and generate CV feedback with Gemini and Google Workspace

## 1. Workflow Overview

**Purpose:** Collect candidate information via an n8n Form (LinkedIn URL + contact details + optional CV PDF), log the submission to Google Sheets, upload the CV to Google Drive, extract PDF text, then use **Google Gemini (via LangChain Agents)** to generate (1) structured CV/LinkedIn recommendations and (2) a cover letter + branding suggestions. Outputs are written into **Google Docs**, and the candidate receives an email containing a Google Drive link to the uploaded CV.

**Target use cases:** recruiters, HR teams, career coaches, training programs needing fast standardized profile/CV feedback and cover letters.

### Logical Blocks
**1.1 Form Intake (Trigger)**  
Receives the submission and attached CV file.

**1.2 File Processing + Logging + Delivery**  
Extracts PDF text, logs the submission to Sheets, uploads the PDF to Drive, and emails the Drive link.

**1.3 AI Generation & Google Docs Output**  
Uses Gemini-powered agents to create:  
- A “CV / Resume Recommendations” Google Doc (from extracted CV text)  
- A “Cover Letter” Google Doc (from the first agent’s output)

---

## 2. Block-by-Block Analysis

### 2.1 Form Intake (Trigger)

**Overview:** Captures candidate inputs from an n8n Form, including an uploaded PDF. This event starts the workflow and provides the source fields referenced by later expressions.

**Nodes involved:**
- **On form submission** (Form Trigger)

#### Node: On form submission
- **Type / role:** `n8n-nodes-base.formTrigger` — entry point (form-based webhook trigger).
- **Key configuration (interpreted):**
  - Form title: **“Submit Your LinkedIn Profile”**
  - Description: asks for LinkedIn URL.
  - Fields:
    - LinkedIn Profile URL (required)
    - First & Last Name (required)
    - Email Address (required, email type)
    - Phone Number (required)
    - Upload CV / Resume (optional file, accepts `.pdf`)
  - Attribution disabled (`appendAttribution: false`)
- **Key data produced:**
  - `submittedAt`
  - Field values by label, e.g.:
    - `$json['LinkedIn Profile URL']`
    - `$json['First & Last Name']`
    - `$json['Email Address']`
    - `$json['Phone Number']`
    - `$json['Upload CV / Resume']` (file metadata array)
  - Binary file is also available and later referenced under a normalized binary property name.
- **Outputs / connections:**
  - Main output goes to:
    - **Extract from File**
    - **Upload file**
- **Version-specific notes:** Form Trigger v2.3.
- **Edge cases / failures:**
  - Missing/invalid required fields prevents submission.
  - PDF upload is optional; downstream nodes expecting a PDF can fail if not provided.

---

### 2.2 File Processing, Logging and Delivery

**Overview:** Extracts text from the uploaded PDF, logs submission metadata to Google Sheets, uploads the PDF to Google Drive, and sends an email to the candidate containing the Drive link (and attempts to attach a file).

**Nodes involved:**
- **Extract from File**
- **Append row in sheet**
- **Upload file**
- **Send a message**

#### Node: Extract from File
- **Type / role:** `n8n-nodes-base.extractFromFile` — extracts text from an uploaded PDF.
- **Key configuration:**
  - Operation: **PDF text extraction**
  - `joinPages: true` (returns one combined text output)
  - Binary property name: **`Upload_CV___Resume`**
    - This implies the form upload is mapped into this binary key (n8n normalizes the field name).
- **Inputs / outputs:**
  - Input: binary PDF from **On form submission**
  - Output: JSON including extracted `text`
  - Connected to: **Append row in sheet**
- **Version:** v1
- **Edge cases / failures:**
  - If no PDF uploaded: node fails due to missing binary property.
  - Scanned/image-only PDFs may extract poorly (empty/garbled text).
  - Large PDFs can cause performance/timeouts.

#### Node: Append row in sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — logs submission into a Google Sheet.
- **Key configuration:**
  - Operation: **Append** row into spreadsheet **“Submit Form Results”** → `Sheet1` (gid=0).
  - Columns mapped using expressions from the trigger:
    - `Date & Time` = `{{ $('On form submission').item.json.submittedAt }}`
    - `Phone Number` = `{{ $('On form submission').item.json['Phone Number'] }}`
    - `Email Address` = `{{ $('On form submission').item.json['Email Address'] }}`
    - `First & Last Name` = `{{ $('On form submission').item.json['First & Last Name'] }}`
    - `Upload CV / Resume` = `{{ $('On form submission').item.json['Upload CV / Resume'][0].filename }}`
    - `LinkedIn Profile URL` = `{{ $('On form submission').item.json['LinkedIn Profile URL'] }}`
  - The `#` column is present in schema and mapping, set to `"0"` (static).
- **Inputs / outputs:**
  - Input: comes from **Extract from File** (but values are pulled from **On form submission** via node references).
  - Output: appended row result.
  - Connected to: **LinkedIn Analyst**
- **Credentials:** Google Sheets OAuth2.
- **Version:** v4.7
- **Edge cases / failures:**
  - If the upload field is empty, `[0].filename` expression errors.
  - Spreadsheet permissions / OAuth scope issues.
  - Sheet schema mismatches (renamed columns) break mapping.

#### Node: Upload file
- **Type / role:** `n8n-nodes-base.googleDrive` — uploads the CV PDF to Drive.
- **Key configuration:**
  - Upload name: `{{ $json['First & Last Name'] }} - LinkedIn CV PDF`
  - Destination folder: **“CV Refiner & Generator”** (folderId `1NQ_ZMlpWXfjgpvllFsk54nHdTLs4YB2v`)
  - Input binary field: `Upload_CV___Resume`
- **Inputs / outputs:**
  - Input: from **On form submission**
  - Output: Drive file metadata including `webContentLink`, `name` (used by email)
  - Connected to: **Send a message**
- **Credentials:** Google Drive OAuth2.
- **Version:** v3
- **Edge cases / failures:**
  - Missing PDF → upload fails.
  - Drive folder permission issues.
  - `webContentLink` may be absent depending on Drive settings; sharing might be restricted by default.

#### Node: Send a message
- **Type / role:** `n8n-nodes-base.gmail` — sends confirmation email to the candidate.
- **Key configuration:**
  - To: `{{ $('On form submission').item.json['Email Address'] }}`
  - Subject: `{{ $json.name }}` (Drive file name from **Upload file**)
  - Body includes Drive link: `{{ $json.webContentLink }}`
  - Attempts to add attachments using:
    - `attachmentsBinary[].property = {{ $('On form submission').item.json['Upload CV / Resume'][0].filename }}`
- **Inputs / outputs:**
  - Input: from **Upload file** (so `$json.webContentLink` is from Drive upload output)
  - Output: Gmail send result (not used further)
- **Credentials:** Gmail OAuth2.
- **Version:** v2.1
- **Edge cases / failures:**
  - **Attachment configuration is likely incorrect:** Gmail node expects a *binary property name* (e.g., `Upload_CV___Resume`), not a filename string. As configured, it may not attach anything and may error.
  - If `webContentLink` isn’t accessible publicly, recipient may get “Access denied”.
  - Gmail API sending limits / OAuth scope issues.

---

### 2.3 AI Generation & Documents

**Overview:** Two Gemini-powered LangChain agents generate structured outputs and write them into Google Docs using “Google Docs Tool” nodes exposed to the agents as tools.

**Nodes involved:**
- **Google Gemini Chat Model**
- **LinkedIn Analyst**
- **Google_Docs**
- **Google_Docs_Update**
- **Google Gemini Chat Model1**
- **Recruiter / HR**
- **Google_Docs_Title**
- **Google_Docs_Body**

#### Node: Google Gemini Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — provides the LLM backend for the first agent.
- **Key configuration:**
  - Default options (model selection not shown; uses credential’s defaults).
- **Connections:**
  - Connected via **ai_languageModel** to **LinkedIn Analyst**.
- **Credentials:** Google Gemini (PaLM) API.
- **Edge cases / failures:**
  - API key/quota errors.
  - Safety filters may block output depending on content.
  - Latency/timeouts on long PDF text.

#### Node: LinkedIn Analyst
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — agent that analyzes extracted CV/LinkedIn text and generates CV recommendations.
- **Key configuration:**
  - Input text: `{{ $('Extract from File').item.json.text }}`
  - System message instructs the agent to:
    - Score the text out of 100%
    - Provide brief recommendations (3–5 lines)
    - Suggest Canva CV design ideas
    - Extract and refine structured CV fields (name, headline, experience, education, skills, languages, etc.)
    - Produce ATS-friendly refinements
    - Write output to Google Doc using tools:
      - **Google_Docs**: create doc with title `{{ First & Last Name }} - CV / Resume Recommendations`
      - **Google_Docs_Update**: insert formatted content
      - Mentions updating a sheet “Google_Sheets_CV_URLs” (but **no such node exists** in this workflow)
- **Tools available (via ai_tool connections):**
  - **Google_Docs** (create doc)
  - **Google_Docs_Update** (update doc)
- **Connections:**
  - Main output → **Recruiter / HR** (passes `$json.output`)
- **Version:** v3
- **Edge cases / failures:**
  - If extracted text is empty/poor, output quality is low.
  - Tool usage depends on agent correctly calling tools; failures can occur if tool parameters are missing.
  - The referenced “Google_Sheets_CV_URLs” update step is not implemented (likely intended but missing).

#### Node: Google_Docs
- **Type / role:** `n8n-nodes-base.googleDocsTool` — tool for the agent to create a Google Doc.
- **Key configuration:**
  - FolderId: `1NQ_ZMlpWXfjgpvllFsk54nHdTLs4YB2v`
  - Title is set through `$fromAI('Title', ..., 'string')` (agent supplies it).
- **Connections:**
  - Provided as **ai_tool** to **LinkedIn Analyst**.
- **Credentials:** Google Docs OAuth2.
- **Version:** v2
- **Edge cases / failures:**
  - Wrong folder permissions.
  - Agent may provide empty/invalid title.

#### Node: Google_Docs_Update
- **Type / role:** `n8n-nodes-base.googleDocsTool` — tool for inserting content into an existing doc.
- **Key configuration:**
  - Operation: **update**
  - Document URL/ID from agent: `$fromAI('Doc_ID_or_URL', ..., 'string')`
  - Action: insert text from `$fromAI('actionFields0_Text', ..., 'string')`
  - “Simplify” boolean from `$fromAI('Simplify', ..., 'boolean')` (agent-controlled)
- **Connections:**
  - Provided as **ai_tool** to **LinkedIn Analyst**.
- **Credentials:** Google Docs OAuth2.
- **Edge cases / failures:**
  - Invalid doc URL/ID from agent.
  - Insert operations can fail if doc not found or permissions missing.

#### Node: Google Gemini Chat Model1
- **Type / role:** Gemini LLM backend for the second agent.
- **Connections:**
  - **ai_languageModel** → **Recruiter / HR**
- **Credentials:** same Gemini account.
- **Edge cases:** same as first model.

#### Node: Recruiter / HR
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — produces cover letter + refined recruiter-friendly suggestions based on prior output.
- **Key configuration:**
  - Input text: `{{ $json.output }}` (from **LinkedIn Analyst**)
  - System message instructs:
    - Enhance all data for candidate use
    - Create cover letter
    - Suggest 2–3 colors for CV/profile image branding
    - Write outputs to Google Docs via tools:
      - **Google_Docs_Title** to create doc titled `{{ First & Last Name }} Cover Letter`
      - **Google_Docs_Body** to insert formatted content
      - Mentions updating “Google_Sheets_URLs” (but **no such node exists**)
- **Tools available:**
  - **Google_Docs_Title**
  - **Google_Docs_Body**
- **Connections:**
  - No downstream main nodes (workflow ends here for this branch).
- **Version:** v3
- **Edge cases / failures:**
  - If first agent output is missing, this agent has little context.
  - Missing sheet-update nodes (referenced but absent) means URLs are not logged anywhere except potentially inside text/doc.

#### Node: Google_Docs_Title
- **Type / role:** `n8n-nodes-base.googleDocsTool` — create doc for cover letter.
- **Key configuration:**
  - FolderId: same shared folder
  - Title from `$fromAI('Title', ..., 'string')`
- **Connections:**
  - Provided as **ai_tool** to **Recruiter / HR**
- **Credentials:** Google Docs OAuth2.
- **Version:** v2
- **Edge cases:** same as Google_Docs.

#### Node: Google_Docs_Body
- **Type / role:** `n8n-nodes-base.googleDocsTool` — insert cover letter and suggestions into the created doc.
- **Key configuration:** update/insert text with `$fromAI(...)` fields (same pattern as Google_Docs_Update).
- **Connections:**
  - Provided as **ai_tool** to **Recruiter / HR**
- **Credentials:** Google Docs OAuth2.
- **Version:** v2
- **Edge cases:** invalid doc URL/ID, permissions.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | n8n-nodes-base.formTrigger | Entry point: collects LinkedIn URL, contact info, PDF | — | Extract from File; Upload file | ## Automate linkedIn-to-CV feedback  from a simple form / This workflow collects candidate details through an n8n Form (LinkedIn URL, name, email, phone, and optional CV upload). It then stores submissions in Google Sheets, uploads the CV to Google Drive, extracts the CV text (PDF), and uses Google Gemini to generate structured CV recommendations and LinkedIn improvement notes in Google Docs. Finally, it drafts a cover letter and sends the candidate a confirmation email via Gmail with the generated output link. / Best for recruiters, career coaches, HR teams, and training programs that want fast, consistent CV and LinkedIn feedback. / ## Form intake |
| Extract from File | n8n-nodes-base.extractFromFile | PDF → text extraction | On form submission | Append row in sheet | ## File processing, logging and delivery |
| Append row in sheet | n8n-nodes-base.googleSheets | Log submission metadata to Google Sheets | Extract from File | LinkedIn Analyst | ## File processing, logging and delivery |
| Upload file | n8n-nodes-base.googleDrive | Upload CV PDF to Google Drive | On form submission | Send a message | ## File processing, logging and delivery |
| Send a message | n8n-nodes-base.gmail | Email candidate with Drive link (and attempted attachment) | Upload file | — | ## File processing, logging and delivery |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM backend for agent 1 | — | LinkedIn Analyst (ai_languageModel) | ## AI generation & documents |
| LinkedIn Analyst | @n8n/n8n-nodes-langchain.agent | Generate CV/LinkedIn recommendations; write to Google Docs | Append row in sheet | Recruiter / HR | ## AI generation & documents |
| Google_Docs | n8n-nodes-base.googleDocsTool | Tool: create recommendations document | — | LinkedIn Analyst (ai_tool) | ## AI generation & documents |
| Google_Docs_Update | n8n-nodes-base.googleDocsTool | Tool: insert recommendations content | — | LinkedIn Analyst (ai_tool) | ## AI generation & documents |
| Google Gemini Chat Model1 | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM backend for agent 2 | — | Recruiter / HR (ai_languageModel) | ## AI generation & documents |
| Recruiter / HR | @n8n/n8n-nodes-langchain.agent | Generate cover letter + branding suggestions; write to Google Docs | LinkedIn Analyst | — | ## AI generation & documents |
| Google_Docs_Title | n8n-nodes-base.googleDocsTool | Tool: create cover letter document | — | Recruiter / HR (ai_tool) | ## AI generation & documents |
| Google_Docs_Body | n8n-nodes-base.googleDocsTool | Tool: insert cover letter content | — | Recruiter / HR (ai_tool) | ## AI generation & documents |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas annotation | — | — | ## Automate linkedIn-to-CV feedback  from a simple form / This workflow collects candidate details through an n8n Form (LinkedIn URL, name, email, phone, and optional CV upload). It then stores submissions in Google Sheets, uploads the CV to Google Drive, extracts the CV text (PDF), and uses Google Gemini to generate structured CV recommendations and LinkedIn improvement notes in Google Docs. Finally, it drafts a cover letter and sends the candidate a confirmation email via Gmail with the generated output link. / Best for recruiters, career coaches, HR teams, and training programs that want fast, consistent CV and LinkedIn feedback. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas annotation | — | — | ## Form intake |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas annotation | — | — | ## File processing, logging and delivery |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas annotation | — | — | ## AI generation & documents |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Form Trigger**
   1. Add node: **Form Trigger** (“On form submission”).
   2. Set Form Title: *Submit Your LinkedIn Profile*.
   3. Add fields:
      - Text: “LinkedIn Profile URL” (required)
      - Text: “First & Last Name” (required, placeholder optional)
      - Email: “Email Address” (required)
      - Text: “Phone Number” (required)
      - File: “Upload CV / Resume” (accept `.pdf`, optional)
   4. Options: disable attribution if desired.

2) **Extract text from uploaded PDF**
   1. Add node: **Extract From File**.
   2. Operation: **PDF**.
   3. Enable **Join pages**.
   4. Set **Binary Property Name** to the binary key that corresponds to the form upload. In this workflow it is `Upload_CV___Resume` (verify in your execution data; you may need to adjust).
   5. Connect: **On form submission → Extract from File**.

3) **Log submission to Google Sheets**
   1. Add node: **Google Sheets**.
   2. Credentials: connect Google Sheets OAuth2.
   3. Operation: **Append**.
   4. Select Spreadsheet (create one if needed) and Sheet tab.
   5. Map columns using expressions referencing the trigger node, e.g.:
      - Date & Time = `{{$('On form submission').item.json.submittedAt}}`
      - LinkedIn Profile URL = `{{$('On form submission').item.json['LinkedIn Profile URL']}}`
      - First & Last Name = `{{$('On form submission').item.json['First & Last Name']}}`
      - Email Address = `{{$('On form submission').item.json['Email Address']}}`
      - Phone Number = `{{$('On form submission').item.json['Phone Number']}}`
      - Upload CV / Resume = `{{$('On form submission').item.json['Upload CV / Resume'][0].filename}}` (guard this if file is optional)
   6. Connect: **Extract from File → Google Sheets**.

4) **Upload PDF to Google Drive**
   1. Add node: **Google Drive** (operation: Upload).
   2. Credentials: Google Drive OAuth2.
   3. Set file name: `{{$json['First & Last Name']}} - LinkedIn CV PDF` (coming from the trigger payload).
   4. Choose destination folder (create folder if needed).
   5. Input binary field name: `Upload_CV___Resume` (must match step 2).
   6. Connect: **On form submission → Google Drive**.

5) **Send candidate email (Gmail)**
   1. Add node: **Gmail** → Send.
   2. Credentials: Gmail OAuth2.
   3. To: `{{$('On form submission').item.json['Email Address']}}`
   4. Subject: `{{$json.name}}` (expects Drive output).
   5. Message: include `{{$json.webContentLink}}`.
   6. Connect: **Google Drive → Gmail**.
   7. If you want to attach the original PDF, set attachment binary property to `Upload_CV___Resume` (not the filename), and ensure the Gmail node receives that binary (you may need to pass-through binary from the trigger branch).

6) **Set up Gemini model for Agent 1**
   1. Add node: **Google Gemini Chat Model** (LangChain).
   2. Credentials: Google Gemini / PaLM API.
   3. Keep defaults or select a specific Gemini model if your node version supports it.

7) **Agent 1: “LinkedIn Analyst”**
   1. Add node: **AI Agent** (LangChain Agent).
   2. Set input text to: `{{$('Extract from File').item.json.text}}`
   3. Paste your system message instructions (CV/LinkedIn analysis + scoring + structured output).
   4. Connect **Google Gemini Chat Model → LinkedIn Analyst** using the **ai_languageModel** connection.
   5. Add Google Docs tools (next steps) and connect them via **ai_tool** to the agent.
   6. Connect main: **Google Sheets → LinkedIn Analyst** (as in the original flow) or directly from Extract step if you prefer.

8) **Google Docs tools for Agent 1**
   1. Add node: **Google Docs Tool** (“Google_Docs”) configured to **create a doc** in your target folder. Title should be AI-provided (use `$fromAI` field mapping).
   2. Add node: **Google Docs Tool** (“Google_Docs_Update”) configured to **update a doc** by URL/ID (AI-provided) and insert AI-provided text.
   3. Connect both tools to **LinkedIn Analyst** via **ai_tool** connections.
   4. Credentials: Google Docs OAuth2.

9) **Set up Gemini model for Agent 2**
   1. Add node: **Google Gemini Chat Model** (second instance).
   2. Credentials: same or different Gemini account.

10) **Agent 2: “Recruiter / HR”**
   1. Add node: **AI Agent**.
   2. Input text: `{{$json.output}}` (expects output from Agent 1).
   3. System message: recruiter-focused refinement + cover letter + color suggestions.
   4. Connect **LinkedIn Analyst → Recruiter / HR** (main).
   5. Connect **Google Gemini Chat Model1 → Recruiter / HR** via **ai_languageModel**.

11) **Google Docs tools for Agent 2**
   1. Add **Google Docs Tool** (“Google_Docs_Title”) to create the cover letter doc in the same folder (title from AI).
   2. Add **Google Docs Tool** (“Google_Docs_Body”) to update/insert the cover letter content (doc URL/ID + text from AI).
   3. Connect both tools to **Recruiter / HR** via **ai_tool**.
   4. Credentials: Google Docs OAuth2.

12) **(Optional but implied) Add missing Google Sheets “URL update” nodes**
   - The agents’ system messages mention updating Sheet2 with doc URLs (“Google_Sheets_CV_URLs”, “Google_Sheets_URLs”), but the workflow does not include these nodes. If you need this:
     1. Add Google Sheets “Update” operations targeting Sheet2.
     2. Decide a matching key (email, timestamp, or an ID).
     3. Capture the created Doc URL from the Google Docs tool output and write it to the appropriate column.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Automate linkedIn-to-CV feedback from a simple form… Best for recruiters, career coaches, HR teams, and training programs…” | Sticky note describing intended value and audience |
| “## Form intake” | Canvas grouping label |
| “## File processing, logging and delivery” | Canvas grouping label |
| “## AI generation & documents” | Canvas grouping label |