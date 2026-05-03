Score job applications and write AI feedback with OpenAI and Notion

https://n8nworkflows.xyz/workflows/score-job-applications-and-write-ai-feedback-with-openai-and-notion-12323


# Score job applications and write AI feedback with OpenAI and Notion

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Score job applications and write AI feedback with OpenAI and Notion  
**Purpose:** Automatically monitor a Notion database of job applications, download each candidate’s resume (PDF or DOCX), extract text, evaluate the candidate against the matching job description stored in another Notion database, and write AI-generated scoring + feedback back into Notion.

**Target use cases**
- Hiring pipelines where resumes are stored as URLs in Notion.
- Automatic first-pass screening/scoring to help recruiters triage candidates.
- Standardized candidate feedback fields (score, summary, skills, decision).

### 1.1 Trigger & Candidate Selection
Watches the “TD Careers” Notion database for new entries and selects one candidate that has not yet received AI comments.

### 1.2 Validation / Idempotency Guard
Ensures the candidate has not already been processed (AI Comments empty). If processed, workflow stops.

### 1.3 Data Preparation: Fetch Job + Download Resume
Fetches the relevant job description from the “Open Jobs” Notion database (using the candidate’s position field), then downloads the resume file.

### 1.4 File Type Routing (DOCX vs PDF)
Determines which branch to run (DOCX processing or PDF processing) based on the resume URL/file name.

### 1.5 DOCX Branch: Convert → Extract → Format → AI Analyze → Normalize → Update Notion
Converts DOCX to an intermediate format, extracts HTML/text, cleans/formats data, calls OpenAI agent, formats AI output, updates Notion fields.

### 1.6 PDF Branch: Extract → Format → AI Analyze → Normalize → Update Notion
Extracts PDF text, cleans/formats data, calls OpenAI agent, formats AI output, updates Notion fields.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Candidate Selection
**Overview:** Detects new applications in Notion and fetches the next unprocessed candidate (AI Comments empty).  
**Nodes involved:** `On New Candidate`, `Get Pending Candidate`

#### Node: On New Candidate
- **Type / role:** Notion Trigger (`n8n-nodes-base.notionTrigger`) — polling trigger to detect changes/new items.
- **Configuration (interpreted):**
  - Polling schedule: **every minute**
  - Watches Notion database: **TD Careers** (databaseId `1d921223-e79e-8164-8cd4-fa013f4dd093`)
- **Connections:**
  - Output → `Get Pending Candidate`
- **Failure/edge cases:**
  - Notion auth/permissions errors (integration not shared with DB).
  - Polling may detect multiple changes; workflow still only processes one candidate because the next node limits results.
  - If database schema changes (property names), downstream nodes break.

#### Node: Get Pending Candidate
- **Type / role:** Notion node (`n8n-nodes-base.notion`) — query DB for items needing processing.
- **Configuration:**
  - Resource: Database Page, Operation: **Get All**
  - Database: **TD Career** (same ID as trigger)
  - Filter: `AI Comments` (rich_text) **is empty**
  - Limit: **1** (process one candidate per run)
- **Key data expected downstream:**
  - Candidate page `id`
  - Candidate properties including `property_position`, `property_resume_file`, `property_ai_comments` (names as produced by the Notion node mapping)
- **Connections:**
  - Output → `Check Processing Status`
- **Failure/edge cases:**
  - If `AI Comments` property type is not rich_text or renamed, filter returns nothing.
  - If multiple candidates are pending, only the first is handled per run.

**Sticky note covering this block**
- “### 1. Trigger & Validation  
  Detects new items and ensures we don't process the same candidate twice.”

---

### Block 2 — Validation / Idempotency Guard
**Overview:** Re-checks “AI Comments” emptiness and stops if already processed (defensive safeguard).  
**Nodes involved:** `Check Processing Status`, `Stop`

#### Node: Check Processing Status
- **Type / role:** IF node (`n8n-nodes-base.if`) — conditional routing.
- **Configuration:**
  - Condition: checks whether `{{$json.property_ai_comments}}` is **empty** (string empty operation).
- **Connections:**
  - True branch (empty) → `Get Full Record`
  - False branch (not empty) → `Stop`
- **Failure/edge cases:**
  - If the Notion node output field name differs (e.g., property mapping changes), expression can evaluate incorrectly.
  - If `property_ai_comments` is undefined rather than empty string, behavior depends on strict validation; could route unexpectedly.

#### Node: Stop
- **Type / role:** NoOp (`n8n-nodes-base.noOp`) — terminates processing path.
- **Connections:** none
- **Failure/edge cases:** none (used intentionally).

---

### Block 3 — Data Preparation: Candidate Record + Job Description + Resume Download
**Overview:** Loads the full Notion page for the candidate, prepares the resume file URL, fetches the job description matching the candidate’s position, and downloads the resume.  
**Nodes involved:** `Get Full Record`, `Prepare Resume URL`, `Fetch Job Description`, `Download Resume`

#### Node: Get Full Record
- **Type / role:** Notion node — retrieves a single page with full properties.
- **Configuration:**
  - Operation: **Get** database page
  - Page ID: `={{ $('Get Pending Candidate').item.json.id }}`
- **Connections:**
  - Output → `Prepare Resume URL`
- **Failure/edge cases:**
  - Page might have been deleted/moved between query and get.
  - Permissions could allow query but not page retrieval (rare but possible with Notion setups).

#### Node: Prepare Resume URL
- **Type / role:** Set node (`n8n-nodes-base.set`) — standardizes/keeps resume URL in a predictable field.
- **Configuration:**
  - Sets field `property_resume_file` to `={{ $json.property_resume_file }}`
  - (Effectively passes through; can be a normalization step in case upstream fields change.)
- **Connections:**
  - Output → `Fetch Job Description`
- **Failure/edge cases:**
  - If `property_resume_file` is empty/null, resume download will fail later.

#### Node: Fetch Job Description
- **Type / role:** Notion node — queries the “Open Jobs” DB to find the job description for the candidate’s position.
- **Configuration:**
  - Operation: **Get All** from database **Open Jobs** (databaseId `19a21223-e79e-80a3-8049-c04ca8c01f9c`)
  - Filter: `Task name` (title) **contains** candidate position:
    - `={{ $('Get Pending Candidate').item.json.property_position }}`
  - `returnAll: true`
  - `alwaysOutputData: true` (workflow continues even if query returns empty)
- **Connections:**
  - Output → `Download Resume`
- **Failure/edge cases:**
  - If no matching job page exists, downstream AI prompt may lack job requirements (quality degradation).
  - “Contains” matching can yield multiple job pages; downstream logic must pick one (not shown explicitly in nodes—risk of multi-item behavior).
  - Property name `Task name|title` must match Notion DB schema exactly.

#### Node: Download Resume
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — downloads the resume file from a URL.
- **Configuration:**
  - URL: `={{ $('Prepare Resume URL').item.json.property_resume_file }}`
  - Response: **File**, and **Full Response** enabled
- **Connections:**
  - Output → `Identify File Type`
- **Failure/edge cases:**
  - Resume URL requires auth (e.g., private drive links) → 401/403.
  - Large files/timeouts.
  - Content-Type mismatch; a HTML page could download instead of the file.

**Sticky note covering this block**
- “### 2. Data Preparation  
  Fetches the full job description and downloads the resume file.”

---

### Block 4 — File Type Detection & Routing
**Overview:** Determines whether the resume is DOCX or PDF and routes to the corresponding extraction + AI pipeline.  
**Nodes involved:** `Identify File Type`, `Route by Extension`

#### Node: Identify File Type
- **Type / role:** Code node (`n8n-nodes-base.code`) — inspects `file_name` and sets `fileType`.
- **Configuration (interpreted from code):**
  - For each item:
    - If `json.file_name` missing: set `error` and `fileType = null`
    - Else detect extension from `file_name`:
      - `pdf` → `fileType = 'pdf'`
      - `docx` → `fileType = 'docx'`
      - otherwise → `null`
- **Connections:**
  - Output → `Route by Extension`
- **Failure/edge cases:**
  - HTTP Request node may not provide `file_name` in `item.json` depending on n8n version/response handling; detection may fail.
  - If URL contains query params, extension detection via file name may not work.
  - Note: The next node routes by **URL contains** anyway, so this node may be redundant unless other nodes rely on `fileType`.

#### Node: Route by Extension
- **Type / role:** Switch (`n8n-nodes-base.switch`) — routes by checking the resume URL string.
- **Configuration:**
  - Rule 1: if `{{$('Prepare Resume URL').item.json.property_resume_file}}` **contains** `.docx` → DOCX branch
  - Rule 2: if it **contains** `pdf` → PDF branch
- **Connections:**
  - Output 0 → `Convert DOCX to Text`
  - Output 1 → `Extract PDF Content`
- **Failure/edge cases:**
  - URL could be uppercase (`.PDF`) or not contain extension at all.
  - The PDF rule checks `contains "pdf"` (not `.pdf`) → can false-match unrelated URLs containing “pdf”.
  - If neither rule matches, execution stops silently (no default route configured).

---

### Block 5 — DOCX Processing Branch
**Overview:** Converts DOCX, extracts text, formats it, runs AI scoring with GPT-4 Turbo, normalizes AI output, then updates Notion.  
**Nodes involved:** `Convert DOCX to Text`, `Extract DOCX Content`, `Format DOCX Data`, `OpenAI DOCX Model`, `Analyze Candidate (DOCX)`, `Format AI Output (DOCX)`, `Update Notion (DOCX)`

#### Node: Convert DOCX to Text
- **Type / role:** Convert to File (`n8n-nodes-base.convertToFile`)
- **Configuration:**
  - Operation: **rtf**
  - Used as an intermediate conversion step before extracting HTML.
- **Connections:**
  - Output → `Extract DOCX Content`
- **Failure/edge cases:**
  - Some DOCX files (encrypted/corrupt) may fail conversion.
  - If the downloaded binary isn’t actually a DOCX, conversion fails.

#### Node: Extract DOCX Content
- **Type / role:** Extract From File (`n8n-nodes-base.extractFromFile`)
- **Configuration:**
  - Operation: **html**
- **Connections:**
  - Output → `Format DOCX Data`
- **Failure/edge cases:**
  - Extraction may produce noisy HTML; depends on DOCX formatting.
  - Very large resumes could exceed later token limits.

#### Node: Format DOCX Data
- **Type / role:** Code node — preprocessing / cleanup.
- **Configuration:**
  - Mode: run once per item
  - JS code is present but truncated in provided JSON: “Resume Content Analysis and Preprocessing...”
- **Expected role (from naming):**
  - Clean extracted HTML, derive structured resume text, possibly attach job description from earlier node context.
- **Connections:**
  - Output → `Analyze Candidate (DOCX)`
- **Failure/edge cases:**
  - If code assumes fields that aren’t present (job description item vs candidate item mismatch), it can throw.
  - Multi-item input from `Fetch Job Description` could complicate pairing (resume item vs job items).

#### Node: OpenAI DOCX Model
- **Type / role:** OpenAI Chat Model (LangChain) (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
- **Configuration:**
  - Model: `gpt-4-turbo`
  - Temperature: `0.1` (stable/consistent scoring)
- **Connections:**
  - Provides `ai_languageModel` input to `Analyze Candidate (DOCX)`
- **Version requirements:**
  - Requires n8n’s LangChain nodes package and compatible n8n version supporting `lmChatOpenAi` typeVersion `1.2`.
- **Failure/edge cases:**
  - Missing/invalid OpenAI credentials.
  - Rate limits / token limits, especially if resume text is long.

#### Node: Analyze Candidate (DOCX)
- **Type / role:** AI Agent (LangChain) (`@n8n/n8n-nodes-langchain.agent`)
- **Configuration:**
  - Prompt type: **define**
  - Prompt text begins: “You are a senior technical recruiter...” (truncated in provided JSON)
  - Uses the connected OpenAI model via `ai_languageModel`.
- **Connections:**
  - Output → `Format AI Output (DOCX)`
- **Failure/edge cases:**
  - If prompt expects job description/resume variables not present, output quality or structure can break.
  - Agent responses may not be valid JSON; downstream formatting code must handle.

#### Node: Format AI Output (DOCX)
- **Type / role:** Code node — normalizes agent output into fields used by Notion update.
- **Configuration:**
  - Mode: run once per item
  - JS code truncated: “Final JSON Processing...”
- **Expected output fields (based on Update Notion mapping):**
  - `status` (for Feedback select)
  - `comments` (AI Comments rich text)
  - `verdictComment` (used oddly as `{{$json.verdictComment[0]}}` as a key)
  - `overallScore` (stringified)
  - `skillsExtractedText`
- **Connections:**
  - Output → `Update Notion (DOCX)`
- **Failure/edge cases:**
  - If AI output deviates (missing keys), Notion update fails or writes blanks.
  - The dynamic property key usage `{{$json.verdictComment[0]}}` is risky: if it’s not a valid Notion property name, update fails.

#### Node: Update Notion (DOCX)
- **Type / role:** Notion node — writes results back to candidate page.
- **Configuration:**
  - Operation: **Update** database page
  - Page ID: `={{ $('Get Pending Candidate').item.json.id }}`
  - Properties set:
    - `Feedback` (select) = `{{$json.status}}`
    - `AI Comments` (rich_text) = `{{$json.comments}}`
    - One property entry has **dynamic key**: `={{ $json.verdictComment[0] }}` (no explicit value shown in JSON; likely incomplete/erroneous mapping)
    - `Resume Score` (rich_text) = `{{$json.overallScore.toString()}}`
    - `Top Skills Detected` (rich_text) = `{{$json.skillsExtractedText}}`
- **Connections:** none
- **Failure/edge cases:**
  - Notion property types must match (Feedback must be Select, others Rich text).
  - Select value must already exist in Notion options unless Notion node supports creating new select options (often it does not).
  - The dynamic key line is a major failure risk (invalid property key).

**Sticky note covering this block**
- “### 3. DOCX Processing Branch  
  Converts DOCX to text, analyzes via AI, and updates Notion.”

---

### Block 6 — PDF Processing Branch
**Overview:** Extracts PDF text, formats it, runs AI scoring with GPT-4 Turbo, normalizes output, then updates Notion.  
**Nodes involved:** `Extract PDF Content`, `Format PDF Data`, `OpenAI PDF Model`, `Analyze Candidate (PDF)`, `Format AI Output (PDF)`, `Update Notion (PDF)`

#### Node: Extract PDF Content
- **Type / role:** Extract From File (`n8n-nodes-base.extractFromFile`)
- **Configuration:**
  - Operation: **pdf**
- **Connections:**
  - Output → `Format PDF Data`
- **Failure/edge cases:**
  - Scanned/image PDFs: extraction may return little/no text (would require OCR, not present here).
  - Password-protected PDFs fail extraction.

#### Node: Format PDF Data
- **Type / role:** Code node — preprocessing / cleanup for PDF text.
- **Configuration:**
  - Mode: run once per item
  - JS code truncated: “Resume Content Analysis and Preprocessing...”
- **Connections:**
  - Output → `Analyze Candidate (PDF)`
- **Failure/edge cases:**
  - Token explosion if preprocessing doesn’t trim repeated headers/footers.
  - Similar multi-item job-description pairing risk.

#### Node: OpenAI PDF Model
- **Type / role:** OpenAI Chat Model (LangChain)
- **Configuration:**
  - Model: `gpt-4-turbo`
  - Temperature: `0.1`
- **Connections:**
  - Provides `ai_languageModel` to `Analyze Candidate (PDF)`
- **Failure/edge cases:** same as DOCX model node.

#### Node: Analyze Candidate (PDF)
- **Type / role:** AI Agent (LangChain)
- **Configuration:**
  - Prompt type: define
  - Prompt starts “You are a senior technical recruiter...” (truncated)
- **Connections:**
  - Output → `Format AI Output (PDF)`
- **Failure/edge cases:** same structural-output risks as DOCX branch.

#### Node: Format AI Output (PDF)
- **Type / role:** Code node — normalizes AI output.
- **Configuration:**
  - Mode: run once per item
  - JS code truncated: “Final JSON Processing...”
- **Expected output fields (based on Notion update):**
  - `status`, `comments`, `rawAnalysis.verdictComment`, `overallScore`, `skillsExtractedText`
- **Connections:**
  - Output → `Update Notion (PDF)`
- **Failure/edge cases:**
  - If `rawAnalysis.verdictComment` missing, Notion “One Line Summary” becomes blank.

#### Node: Update Notion (PDF)
- **Type / role:** Notion node — writes PDF branch results back to the candidate record.
- **Configuration:**
  - Operation: Update database page
  - Page ID: `={{ $('Get Pending Candidate').item.json.id }}`
  - Sets:
    - `Feedback` (select) = `{{$json.status}}`
    - `AI Comments` (rich_text) = `{{$json.comments}}`
    - `One Line Summary` (rich_text) = `{{$json.rawAnalysis.verdictComment}}`
    - `Resume Score` (rich_text) = `{{$json.overallScore.toString()}}`
    - `Top Skills Detected` (rich_text) = `{{$json.skillsExtractedText}}`
- **Failure/edge cases:**
  - Select option mismatch.
  - Property name mismatches in Notion DB schema.

**Sticky note covering this block**
- “### 4. PDF Processing Branch  
  Extracts PDF text, analyzes via AI, and updates Notion.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On New Candidate | notionTrigger | Poll Notion DB for new/changed candidate items | — | Get Pending Candidate | ## How it works … (watches Notion, checks status, downloads resume, analyzes with OpenAI + Open Jobs, updates Notion). Setup steps include Notion creds, DB IDs, and required fields. / “### 1. Trigger & Validation …” |
| Get Pending Candidate | notion | Fetch next unprocessed candidate (AI Comments empty) | On New Candidate | Check Processing Status | Same as above / “### 1. Trigger & Validation …” |
| Check Processing Status | if | Guard: ensure candidate not already processed | Get Pending Candidate | Get Full Record (true), Stop (false) | Same as above / “### 1. Trigger & Validation …” |
| Stop | noOp | Stop execution for already-processed candidates | Check Processing Status | — | Same as above / “### 1. Trigger & Validation …” |
| Get Full Record | notion | Retrieve full candidate page properties | Check Processing Status (true) | Prepare Resume URL | “### 2. Data Preparation …” |
| Prepare Resume URL | set | Normalize/pass-through resume URL field | Get Full Record | Fetch Job Description | “### 2. Data Preparation …” |
| Fetch Job Description | notion | Find job description in Open Jobs DB matching candidate position | Prepare Resume URL | Download Resume | “### 2. Data Preparation …” |
| Download Resume | httpRequest | Download resume as binary file | Fetch Job Description | Identify File Type | “### 2. Data Preparation …” |
| Identify File Type | code | Detect file extension from `file_name` and set `fileType` | Download Resume | Route by Extension | “### 2. Data Preparation …” |
| Route by Extension | switch | Route to DOCX vs PDF pipeline based on resume URL contains | Identify File Type | Convert DOCX to Text; Extract PDF Content | “### 2. Data Preparation …” |
| Convert DOCX to Text | convertToFile | Convert DOCX into intermediate format (rtf) | Route by Extension (docx) | Extract DOCX Content | “### 3. DOCX Processing Branch …” |
| Extract DOCX Content | extractFromFile | Extract HTML/text from converted file | Convert DOCX to Text | Format DOCX Data | “### 3. DOCX Processing Branch …” |
| Format DOCX Data | code | Clean/structure extracted DOCX content for AI prompt | Extract DOCX Content | Analyze Candidate (DOCX) | “### 3. DOCX Processing Branch …” |
| OpenAI DOCX Model | lmChatOpenAi (LangChain) | Chat model for DOCX agent | — | Analyze Candidate (DOCX) (ai_languageModel) | “### 3. DOCX Processing Branch …” |
| Analyze Candidate (DOCX) | agent (LangChain) | Score and write feedback vs job requirements | Format DOCX Data + OpenAI DOCX Model | Format AI Output (DOCX) | “### 3. DOCX Processing Branch …” |
| Format AI Output (DOCX) | code | Normalize AI output to fields for Notion update | Analyze Candidate (DOCX) | Update Notion (DOCX) | “### 3. DOCX Processing Branch …” |
| Update Notion (DOCX) | notion | Write DOCX scoring/feedback back to candidate page | Format AI Output (DOCX) | — | “### 3. DOCX Processing Branch …” |
| Extract PDF Content | extractFromFile | Extract text from PDF | Route by Extension (pdf) | Format PDF Data | “### 4. PDF Processing Branch …” |
| Format PDF Data | code | Clean/structure extracted PDF text for AI prompt | Extract PDF Content | Analyze Candidate (PDF) | “### 4. PDF Processing Branch …” |
| OpenAI PDF Model | lmChatOpenAi (LangChain) | Chat model for PDF agent | — | Analyze Candidate (PDF) (ai_languageModel) | “### 4. PDF Processing Branch …” |
| Analyze Candidate (PDF) | agent (LangChain) | Score and write feedback vs job requirements | Format PDF Data + OpenAI PDF Model | Format AI Output (PDF) | “### 4. PDF Processing Branch …” |
| Format AI Output (PDF) | code | Normalize AI output to fields for Notion update | Analyze Candidate (PDF) | Update Notion (PDF) | “### 4. PDF Processing Branch …” |
| Update Notion (PDF) | notion | Write PDF scoring/feedback back to candidate page | Format AI Output (PDF) | — | “### 4. PDF Processing Branch …” |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the Notion databases (or confirm existing)**
   - DB A: **TD Careers** (candidates)
     - Required properties (per sticky note and nodes):
       - `AI Comments` (Rich text)
       - `Resume Score` (Rich text)
       - `Top Skills Detected` (Rich text)
       - `Feedback` (Select)
       - `Resume file` (URL or Files & media that yields a URL in n8n as `property_resume_file`)
       - `Position` (Text/Select mapped to `property_position`)
       - `One Line Summary` (Rich text) (used by PDF branch)
   - DB B: **Open Jobs**
     - `Task name` (Title) containing the position name used by candidates.

2) **Add credentials**
   - In n8n, add **Notion API credentials** (and share both databases with the Notion integration).
   - Add **OpenAI credentials** usable by LangChain OpenAI Chat Model nodes.

3) **Add trigger**
   - Node: **Notion Trigger** named `On New Candidate`
   - Configure:
     - Database: `TD Careers`
     - Polling: every minute
   - Connect to next step.

4) **Fetch one pending candidate**
   - Node: **Notion** named `Get Pending Candidate`
   - Operation: Database Page → **Get All**
   - Database: `TD Careers`
   - Filter: `AI Comments` **is empty**
   - Limit: `1`
   - Connect `On New Candidate` → `Get Pending Candidate`.

5) **Idempotency check**
   - Node: **IF** named `Check Processing Status`
   - Condition: String → **is empty** on `{{$json.property_ai_comments}}`
   - Connect `Get Pending Candidate` → `Check Processing Status`.
   - Add a **NoOp** node named `Stop`, connect IF false output → `Stop`.

6) **Get full candidate record**
   - Node: **Notion** named `Get Full Record`
   - Operation: Database Page → **Get**
   - Page ID: `={{ $('Get Pending Candidate').item.json.id }}`
   - Connect IF true output → `Get Full Record`.

7) **Prepare resume URL**
   - Node: **Set** named `Prepare Resume URL`
   - Set field `property_resume_file` = `={{ $json.property_resume_file }}`
   - Connect `Get Full Record` → `Prepare Resume URL`.

8) **Fetch matching job description**
   - Node: **Notion** named `Fetch Job Description`
   - Operation: Database Page → **Get All**
   - Database: `Open Jobs`
   - Filter: `Task name` (Title) **contains** `={{ $('Get Pending Candidate').item.json.property_position }}`
   - Enable “Always output data” (so workflow continues even if no match).
   - Connect `Prepare Resume URL` → `Fetch Job Description`.

9) **Download resume**
   - Node: **HTTP Request** named `Download Resume`
   - URL: `={{ $('Prepare Resume URL').item.json.property_resume_file }}`
   - Response: **File** and enable **Full Response**
   - Connect `Fetch Job Description` → `Download Resume`.

10) **Detect and route by file type**
   - Node: **Code** named `Identify File Type`
     - Implement extension detection (pdf/docx) from `file_name` if present.
   - Node: **Switch** named `Route by Extension`
     - Rule for DOCX: URL contains `.docx`
     - Rule for PDF: URL contains `pdf` (prefer `.pdf` and case-insensitive if you improve it)
   - Connect: `Download Resume` → `Identify File Type` → `Route by Extension`.

11) **DOCX branch nodes**
   1. **Convert to File** named `Convert DOCX to Text`
      - Operation: `rtf`
   2. **Extract From File** named `Extract DOCX Content`
      - Operation: `html`
   3. **Code** named `Format DOCX Data`
      - Clean/trim extracted content; prepare structured input for AI (resume + job description).
   4. **OpenAI Chat Model (LangChain)** named `OpenAI DOCX Model`
      - Model: `gpt-4-turbo`
      - Temperature: `0.1`
   5. **AI Agent (LangChain)** named `Analyze Candidate (DOCX)`
      - Prompt: “senior technical recruiter…” (your rubric + output schema)
      - Connect `OpenAI DOCX Model` to the Agent’s **ai_languageModel** input.
   6. **Code** named `Format AI Output (DOCX)`
      - Parse/normalize agent output into: `status`, `comments`, `overallScore`, `skillsExtractedText`, etc.
   7. **Notion Update** named `Update Notion (DOCX)`
      - Page ID: `={{ $('Get Pending Candidate').item.json.id }}`
      - Map fields:
        - Feedback (Select) ← `status`
        - AI Comments (Rich text) ← `comments`
        - Resume Score (Rich text) ← `overallScore` as string
        - Top Skills Detected (Rich text) ← `skillsExtractedText`
      - Avoid dynamic property keys unless you are certain they match a real Notion property.

   - Connect in order from `Route by Extension` DOCX output.

12) **PDF branch nodes**
   1. **Extract From File** named `Extract PDF Content`
      - Operation: `pdf`
   2. **Code** named `Format PDF Data`
   3. **OpenAI Chat Model (LangChain)** named `OpenAI PDF Model`
      - Model: `gpt-4-turbo`, temperature `0.1`
   4. **AI Agent (LangChain)** named `Analyze Candidate (PDF)` with same rubric prompt logic
   5. **Code** named `Format AI Output (PDF)`
   6. **Notion Update** named `Update Notion (PDF)`
      - Page ID: `={{ $('Get Pending Candidate').item.json.id }}`
      - Map:
        - Feedback ← `status`
        - AI Comments ← `comments`
        - One Line Summary ← `rawAnalysis.verdictComment`
        - Resume Score ← `overallScore`
        - Top Skills Detected ← `skillsExtractedText`

   - Connect in order from `Route by Extension` PDF output.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Watches Notion for new job applications, checks status, downloads resume, analyzes with OpenAI vs Open Jobs, updates Notion with score/summary/disqualifiers. | Sticky note “How it works” (applies to whole workflow). |
| Setup: configure Notion credentials, update database IDs in trigger + Fetch Job Description, ensure Notion fields exist: AI Comments, Resume Score, Top Skills Detected, Feedback. | Sticky note “Setup steps” (applies to whole workflow). |