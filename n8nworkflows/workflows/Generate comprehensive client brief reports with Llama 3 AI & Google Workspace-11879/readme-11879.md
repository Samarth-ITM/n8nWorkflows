Generate comprehensive client brief reports with Llama 3 AI & Google Workspace

https://n8nworkflows.xyz/workflows/generate-comprehensive-client-brief-reports-with-llama-3-ai---google-workspace-11879


# Generate comprehensive client brief reports with Llama 3 AI & Google Workspace

## 1. Workflow Overview

**Purpose:** Automatically generate a comprehensive, structured “client brief analysis” report when a new document is uploaded to a specific Google Drive folder. The workflow extracts text, runs AI analysis (Groq Llama 3.3 70B), performs industry research (Wikipedia + SerpAPI), composes a formatted report, saves it as a Google Doc, logs metadata to Google Sheets, and emails an account manager. If text extraction fails, it emails an error notification.

**Target use cases**
- Agencies/consultancies receiving client brief files (PDF/DOCX/TXT) and needing consistent analysis + research summaries.
- Account managers who want an automated report link and key metrics shortly after upload.
- Ops teams who need a tracking sheet of processed briefs.

### 1.1 Input Reception & Configuration
Drive trigger detects new file → configuration values are set (folder IDs, emails, sheet ID).

### 1.2 Document Type Routing & Text Extraction
File MIME type is checked → appropriate extractor runs → extraction is validated (non-empty, length > 50).

### 1.3 AI Brief Analysis (Structured)
Extracted text is analyzed by a LangChain Agent using Groq Llama → output is forced into a structured JSON schema.

### 1.4 AI Industry Research (Structured, tool-augmented)
A second agent performs market/industry research using Wikipedia + SerpAPI tools → outputs structured JSON.

### 1.5 Merge → Report Composition → Output Delivery
Analysis + research are merged → a Code node generates markdown report + stats → Google Doc is created → tracking sheet is updated → account manager receives an HTML email with summary + link.

### 1.6 Error Handling
If extraction fails validation, an error email is sent to a configured address.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Preparation
**Overview:** Watches a Drive folder for new files, injects workflow configuration constants, routes by file type, extracts text, and validates extraction quality.

**Nodes involved:**
- New File in Client Briefs Folder
- Workflow Configuration
- Check File Type
- Extract Text from PDF
- Extract Text from DOCX
- Extract Text from TXT
- Check Extraction Success
- Send Error Notification (error branch)

#### Node: New File in Client Briefs Folder
- **Type / role:** `Google Drive Trigger` — entry point; polls for new files in a specific folder.
- **Config choices:**
  - Event: *fileCreated*
  - Trigger scope: *specificFolder*
  - Poll schedule: every 5 minutes
  - Folder ID: placeholder (`Google Drive Folder ID for Client Briefs`)
- **Input/Output:**
  - No input (trigger).
  - Output to **Workflow Configuration**.
- **Edge cases / failures:**
  - Google OAuth permission issues (Drive scope, folder access).
  - Polling delay: up to 5 minutes latency.
  - Trigger may fire for unsupported MIME types; downstream routing currently only checks for PDF explicitly.

#### Node: Workflow Configuration
- **Type / role:** `Set` — centralizes runtime constants.
- **Config choices (fields created):**
  - `clientSummariesFolderId` (Drive folder for generated reports)
  - `accountManagerEmail`
  - `trackingSheetId` (Google Sheets document ID)
  - `errorNotificationEmail`
  - `includeOtherFields: true` keeps fields from the trigger item.
- **Input/Output:**
  - Input from **New File in Client Briefs Folder**
  - Output to **Check File Type**
- **Edge cases / failures:**
  - If placeholders aren’t replaced with real IDs/emails, downstream nodes will fail (Docs/Sheets/Gmail).

#### Node: Check File Type
- **Type / role:** `IF` — routes based on MIME type.
- **Config choices:**
  - Condition: `$json.mimeType === "application/pdf"`
  - True branch → PDF extractor
  - False branch → both DOCX and TXT extractors (see note below)
- **Input/Output:**
  - Input from **Workflow Configuration**
  - Output (true) to **Extract Text from PDF**
  - Output (false) to **Extract Text from DOCX** and **Extract Text from TXT**
- **Edge cases / failures:**
  - **DOCX is not explicitly checked.** Any non-PDF file will be sent to *both* DOCX and TXT extractors; one or both may fail or produce empty text.
  - Google Drive often uses MIME types like:
    - DOCX: `application/vnd.openxmlformats-officedocument.wordprocessingml.document`
    - Plain text: `text/plain`
  - Consider switching to a multi-branch switch by MIME type to avoid double-processing.

#### Node: Extract Text from PDF
- **Type / role:** `Extract from File` — parses PDF to text.
- **Config choices:** operation `pdf`.
- **Input/Output:**
  - Input from **Check File Type** (true branch)
  - Output to **Check Extraction Success**
- **Edge cases / failures:**
  - Scanned PDFs without embedded text will yield empty/short output (no OCR here).
  - Password-protected/corrupted PDFs can fail.

#### Node: Extract Text from DOCX
- **Type / role:** `Extract from File` — extracts text.
- **Config choices:** operation `text` (generic text extraction).
- **Input/Output:**
  - Input from **Check File Type** (false branch)
  - Output to **Check Extraction Success**
- **Edge cases / failures:**
  - If input is not actually DOCX, extraction may fail or return unusable text.

#### Node: Extract Text from TXT
- **Type / role:** `Extract from File` — extracts text from text-based file.
- **Config choices:** operation `text`.
- **Input/Output:**
  - Input from **Check File Type** (false branch)
  - Output to **Check Extraction Success**
- **Edge cases / failures:**
  - If input is not plain text, output can be garbage/empty.

#### Node: Check Extraction Success
- **Type / role:** `IF` — validates extraction quality.
- **Config choices:**
  - `$json.text` is not empty
  - `$json.text.length > 50`
- **Input/Output:**
  - Input from any extractor
  - True → **Analyze Client Brief**
  - False → **Send Error Notification**
- **Edge cases / failures:**
  - Very short briefs (< 50 chars) will be treated as failure.
  - If extractor outputs under a different field than `text`, condition fails (depends on Extract node output shape).

#### Node: Send Error Notification
- **Type / role:** `Gmail` — sends a formatted HTML error email.
- **Config choices:**
  - To: `$('Workflow Configuration').item.json.errorNotificationEmail`
  - Subject includes original file name
  - Body includes possible causes and actions
- **Input/Output:**
  - Input from **Check Extraction Success** (false branch)
  - No downstream nodes
- **Edge cases / failures:**
  - Gmail credential / scope errors.
  - If `errorNotificationEmail` is empty/invalid, send fails.
  - Not logging failures to the tracking sheet (only a notification is sent).

---

### Block 2 — AI Brief Analysis (Groq + Structured Output)
**Overview:** Uses a Groq-hosted Llama 3.3 70B chat model via LangChain Agent to extract structured strategic insights from the brief text.

**Nodes involved:**
- Groq Chat Model
- Structured Output Parser
- Analyze Client Brief

#### Node: Groq Chat Model
- **Type / role:** `lmChatGroq` — provides the LLM backend.
- **Config choices:**
  - Model: `llama-3.3-70b-versatile`
- **Input/Output:**
  - Connected as `ai_languageModel` into **Analyze Client Brief**
- **Version requirements:**
  - Requires `@n8n/n8n-nodes-langchain` package and node availability in your n8n instance.
- **Edge cases / failures:**
  - Groq API key missing/invalid.
  - Rate limits / timeouts on large briefs.

#### Node: Structured Output Parser
- **Type / role:** `outputParserStructured` — enforces a JSON structure for analysis output.
- **Config choices:**
  - JSON schema example with keys:
    - `summary`, `clientName`, `industry`, `projectType`, `clientNeeds[]`, `clientGoals[]`,
      `targetAudience`, `budget`, `timeline`, `challenges[]`, `risks[]`,
      `recommendations[]`, `keyQuestions[]`
- **Input/Output:**
  - Connected as `ai_outputParser` into **Analyze Client Brief**
- **Edge cases / failures:**
  - Model may produce invalid JSON → parser can throw.
  - Fields may be missing or wrong type; downstream code uses defensive checks, but email/logging expect certain fields.

#### Node: Analyze Client Brief
- **Type / role:** `LangChain Agent` — prompts the model to do deep analysis of extracted text.
- **Config choices:**
  - `text`: `={{ $json.text }}` (from extractor output)
  - System message: detailed strategic analyst instructions (12 extraction categories)
  - `hasOutputParser: true` to force structured JSON via parser node
- **Input/Output:**
  - Main input from **Check Extraction Success** (true branch)
  - Main outputs:
    - To **Deep Industry Research** (for context)
    - To **Combine Analysis and Research** (merge input 0)
  - AI connections:
    - Language model from **Groq Chat Model**
    - Output parser from **Structured Output Parser**
- **Edge cases / failures:**
  - Very long text may exceed context limits → truncated or failed completions.
  - If extracted text includes lots of artifacts (PDF layout), analysis quality degrades.

---

### Block 3 — AI Industry Research (Tools + Structured Output)
**Overview:** Performs industry and competitive research based on extracted fields from the brief analysis, using Wikipedia and SerpAPI tools, returning structured findings.

**Nodes involved:**
- Groq Research Model
- Wikipedia Research Tool
- SerpAPI Google Search
- Research Output Parser
- Deep Industry Research

#### Node: Groq Research Model
- **Type / role:** `lmChatGroq` — LLM backend for research agent.
- **Config choices:** model `llama-3.3-70b-versatile`
- **Connections:**
  - `ai_languageModel` → **Deep Industry Research**
- **Edge cases / failures:** same as other Groq node; plus longer tool-augmented runs may time out.

#### Node: Wikipedia Research Tool
- **Type / role:** LangChain Wikipedia tool.
- **Config choices:** default.
- **Connections:**
  - `ai_tool` → **Deep Industry Research**
- **Edge cases / failures:**
  - Missing network access in self-hosted environments.
  - Wikipedia queries may be too broad → irrelevant results.

#### Node: SerpAPI Google Search
- **Type / role:** LangChain SerpAPI tool for Google Search.
- **Config choices:** default options; requires SerpAPI credentials.
- **Connections:**
  - `ai_tool` → **Deep Industry Research**
- **Edge cases / failures:**
  - Missing/invalid SerpAPI key.
  - SerpAPI quota limits.
  - If you don’t want external search, remove this tool; research quality changes accordingly.

#### Node: Research Output Parser
- **Type / role:** `outputParserStructured` — enforces research JSON structure.
- **Config choices:** schema example with keys:
  - `industryTrends[]`, `competitorInsights[]`, `marketOpportunities[]`, `bestPractices[]`,
    `technologicalFactors[]`, `regulatoryConsiderations[]`,
    `budgetBenchmarks`, `timelineEstimates`
- **Connections:** `ai_outputParser` → **Deep Industry Research**
- **Edge cases / failures:**
  - Tool outputs may lead to non-JSON responses; parser can error.

#### Node: Deep Industry Research
- **Type / role:** `LangChain Agent` — runs research prompt using tools.
- **Config choices:**
  - Text prompt references the analysis results:
    - `industry`, `projectType`, `clientName` from `$('Analyze Client Brief').item.json...`
  - System message requests trends/competitors/opportunities/best practices/tech/regulatory/budgets/timelines and explicitly asks to use Wikipedia + web search tools.
  - `hasOutputParser: true`
- **Input/Output:**
  - Main input from **Analyze Client Brief**
  - Output to **Combine Analysis and Research** (merge input 1)
  - AI connections: Groq model + tools + parser
- **Edge cases / failures:**
  - If analysis output fields are missing, prompt becomes incomplete (e.g., empty industry).
  - If tools are unavailable, the agent may still respond but without external grounding.

---

### Block 4 — Merge, Report Generation, Storage, Logging, Notification
**Overview:** Combines analysis + research, generates a markdown report and metrics, stores as Google Doc, updates a tracking sheet, and emails the account manager.

**Nodes involved:**
- Combine Analysis and Research
- Generate Comprehensive Report
- Create Google Doc Report
- Log to Tracking Sheet
- Send Email to Account Manager

#### Node: Combine Analysis and Research
- **Type / role:** `Merge` — combines two streams into one item.
- **Config choices:**
  - Mode: `combine`
  - Combine by: `combineAll`
- **Input/Output:**
  - Input 0: from **Analyze Client Brief**
  - Input 1: from **Deep Industry Research**
  - Output to **Generate Comprehensive Report**
- **Edge cases / failures:**
  - If research fails but analysis succeeds (or vice versa), merge behavior depends on whether both inputs arrive; one missing input can stall execution.

#### Node: Generate Comprehensive Report
- **Type / role:** `Code` — assembles markdown report + computes stats + packages JSON payload.
- **Config choices:**
  - Uses `$input.first().json` as `analysisData`, `$input.last().json` as `researchData`.
  - Produces:
    - `reportContent` (markdown)
    - `briefAnalysis` (analysis object)
    - `industryResearch` (research object)
    - `stats` counts (needs/goals/challenges/risks/recommendations/questions/trends)
    - `originalFileName`, `analysisDate`
- **Input/Output:**
  - Input from **Combine Analysis and Research**
  - Output to **Create Google Doc Report**
- **Edge cases / failures:**
  - If merge returns unexpected ordering or only one input, `analysisData`/`researchData` assignment may be wrong.
  - Markdown is generated, but the next Google Docs node only sets title/folder; it does **not** explicitly insert `reportContent` into the document in this workflow as provided (see next node).
  - Missing fields are handled with fallbacks in many sections.

#### Node: Create Google Doc Report
- **Type / role:** `Google Docs` — creates a new Google Doc.
- **Config choices:**
  - Title: `📊 Client Brief Analysis - {{ trigger file name }}`
  - Folder ID: `Workflow Configuration.clientSummariesFolderId`
- **Input/Output:**
  - Input from **Generate Comprehensive Report**
  - Output to **Log to Tracking Sheet**
- **Important implementation note:**
  - As configured, this node **creates a doc** but does not show any parameter that writes `reportContent` into the document body. If your intention is a filled report, you typically need an additional Google Docs step such as “Append text”, “Update document”, or “Create document from markdown/plain text” (depending on node capabilities/version).
- **Edge cases / failures:**
  - Folder permission errors.
  - Google Docs API scope not granted.

#### Node: Log to Tracking Sheet
- **Type / role:** `Google Sheets` — appends or updates a log row.
- **Config choices:**
  - Operation: `appendOrUpdate`
  - Sheet name: `Brief Analysis Log`
  - Document ID: `Workflow Configuration.trackingSheetId`
  - Matching column: `Date` (note: this is unusual as a unique key; can cause overwrites if same minute)
  - Writes columns including:
    - Date, Status=Completed, Doc URL, Industry, File Name, Client Name, counts, Project Type
- **Input/Output:**
  - Input from **Create Google Doc Report**
  - Output to **Send Email to Account Manager**
- **Edge cases / failures:**
  - If “Brief Analysis Log” tab or columns don’t match, mapping fails.
  - Using `Date` as matching column can unintentionally update an existing row if timestamps collide (same value).
  - Sheet permissions / quota errors.

#### Node: Send Email to Account Manager
- **Type / role:** `Gmail` — sends formatted HTML summary + link to the created doc.
- **Config choices:**
  - To: `Workflow Configuration.accountManagerEmail`
  - Subject includes original file name
  - Body uses:
    - `briefAnalysis.clientName/industry/projectType/summary`
    - `stats.*Count`
    - Link to `$('Create Google Doc Report').item.json.documentUrl`
- **Input/Output:**
  - Input from **Log to Tracking Sheet**
  - No downstream nodes
- **Edge cases / failures:**
  - If Google Doc node didn’t return `documentUrl`, email link breaks.
  - If `briefAnalysis` isn’t present (upstream failure), email rendering will show blanks or error depending on n8n expression strictness.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| New File in Client Briefs Folder | googleDriveTrigger | Entry trigger: detect new file in specific Drive folder | — | Workflow Configuration | ## 📥 1. Input & Preparation (nodes + description + supported formats + error path) |
| Workflow Configuration | set | Central configuration constants (folder IDs, emails, sheet ID) | New File in Client Briefs Folder | Check File Type | ## 📥 1. Input & Preparation (nodes + description + supported formats + error path) |
| Check File Type | if | Route by MIME type (PDF vs other) | Workflow Configuration | Extract Text from PDF; Extract Text from DOCX; Extract Text from TXT | ## 📥 1. Input & Preparation (nodes + description + supported formats + error path) |
| Extract Text from PDF | extractFromFile | Extract text from PDF | Check File Type | Check Extraction Success | ## 📥 1. Input & Preparation (nodes + description + supported formats + error path) |
| Extract Text from DOCX | extractFromFile | Extract text from DOCX | Check File Type | Check Extraction Success | ## 📥 1. Input & Preparation (nodes + description + supported formats + error path) |
| Extract Text from TXT | extractFromFile | Extract text from TXT | Check File Type | Check Extraction Success | ## 📥 1. Input & Preparation (nodes + description + supported formats + error path) |
| Check Extraction Success | if | Validate extracted text exists and length > 50 | Extract Text from PDF/DOCX/TXT | Analyze Client Brief; Send Error Notification | ## 📥 1. Input & Preparation (nodes + description + supported formats + error path) |
| Send Error Notification | gmail | Notify on extraction failure | Check Extraction Success (false) | — | ## 📥 1. Input & Preparation (nodes + description + supported formats + error path) |
| Groq Chat Model | lmChatGroq | LLM for brief analysis | — (AI connection) | Analyze Client Brief (ai_languageModel) | ## 🤖 2. AI Processing & Research (nodes + description + model + output format) |
| Structured Output Parser | outputParserStructured | Enforce structured JSON for analysis | — (AI connection) | Analyze Client Brief (ai_outputParser) | ## 🤖 2. AI Processing & Research (nodes + description + model + output format) |
| Analyze Client Brief | agent (LangChain) | Deep strategic analysis of extracted brief | Check Extraction Success (true) | Deep Industry Research; Combine Analysis and Research | ## 🤖 2. AI Processing & Research (nodes + description + model + output format) |
| Groq Research Model | lmChatGroq | LLM for research agent | — (AI connection) | Deep Industry Research (ai_languageModel) | ## 🤖 2. AI Processing & Research (nodes + description + model + output format) |
| Wikipedia Research Tool | toolWikipedia | Tool for encyclopedia lookups | — (AI tool) | Deep Industry Research (ai_tool) | ## 🤖 2. AI Processing & Research (nodes + description + model + output format) |
| SerpAPI Google Search | toolSerpApi | Tool for web search via SerpAPI | — (AI tool) | Deep Industry Research (ai_tool) | ## 🤖 2. AI Processing & Research (nodes + description + model + output format) |
| Research Output Parser | outputParserStructured | Enforce structured JSON for research | — (AI connection) | Deep Industry Research (ai_outputParser) | ## 🤖 2. AI Processing & Research (nodes + description + model + output format) |
| Deep Industry Research | agent (LangChain) | Tool-augmented industry/market research | Analyze Client Brief | Combine Analysis and Research | ## 🤖 2. AI Processing & Research (nodes + description + model + output format) |
| Combine Analysis and Research | merge | Combine analysis + research into one payload | Analyze Client Brief; Deep Industry Research | Generate Comprehensive Report | ## 📤 3. Output & Delivery (nodes + description + output) |
| Generate Comprehensive Report | code | Build markdown report + compute stats + output JSON | Combine Analysis and Research | Create Google Doc Report | ## 📤 3. Output & Delivery (nodes + description + output) |
| Create Google Doc Report | googleDocs | Create Google Doc to store report | Generate Comprehensive Report | Log to Tracking Sheet | ## 📤 3. Output & Delivery (nodes + description + output) |
| Log to Tracking Sheet | googleSheets | Append/update log row in “Brief Analysis Log” | Create Google Doc Report | Send Email to Account Manager | ## 📤 3. Output & Delivery (nodes + description + output) |
| Send Email to Account Manager | gmail | Send HTML summary and doc link | Log to Tracking Sheet | — | ## 📤 3. Output & Delivery (nodes + description + output) |
| Main Overview | stickyNote | Documentation note | — | — | ## Auto Client Brief Analyzer (setup steps, required columns, credentials) |
| Stage 1: Input & Preparation | stickyNote | Documentation note for Stage 1 | — | — | (content as shown in workflow) |
| Stage 2: AI Processing | stickyNote | Documentation note for Stage 2 | — | — | (content as shown in workflow) |
| Stage 3: Output & Delivery | stickyNote | Documentation note for Stage 3 | — | — | (content as shown in workflow) |
| Sticky Note10 | stickyNote | Author/credit note + links | — | — | Author info, Telegram link: https://t.me/digimetalab, templates link: https://n8n.io/creators/digimetalab/ |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Google Drive folders**
   1. “Client Briefs” (incoming uploads)
   2. “Client Summaries” (generated reports)

2) **Create Google Sheet**
   - Spreadsheet with a tab named **Brief Analysis Log**
   - Columns (header row) exactly:
     `Date | File Name | Client Name | Industry | Project Type | Needs Count | Goals Count | Challenges Count | Recommendations Count | Doc URL | Status`

3) **Credentials**
   - **Google** credentials with access to:
     - Drive (read folder contents)
     - Docs (create documents)
     - Sheets (append/update rows)
     - Gmail (send emails)
   - **Groq API** credential for LangChain Groq nodes.
   - **SerpAPI** credential (optional but configured in this workflow) for web search tool.

4) **Node 1: Google Drive Trigger**
   - Add node: **Google Drive Trigger**
   - Event: *File Created*
   - Trigger on: *Specific folder*
   - Folder ID: select the “Client Briefs” folder
   - Polling: every **5 minutes**
   - Connect to next node.

5) **Node 2: Set (Workflow Configuration)**
   - Add node: **Set**
   - Add fields:
     - `clientSummariesFolderId` = (ID of “Client Summaries” folder)
     - `accountManagerEmail` = (recipient email)
     - `trackingSheetId` = (spreadsheet ID)
     - `errorNotificationEmail` = (ops/admin email)
   - Enable “Include Other Fields”.
   - Connect from Drive Trigger → this Set node → next node.

6) **Node 3: IF (Check File Type)**
   - Add node: **IF**
   - Condition (String → Equals):
     - Left: `={{ $json.mimeType }}`
     - Right: `application/pdf`
   - True output → PDF extraction.
   - False output → DOCX and TXT extraction (or preferably replace with a Switch per MIME type if you improve it).

7) **Nodes 4–6: Extractors**
   - Add **Extract from File** node “Extract Text from PDF”
     - Operation: `pdf`
     - Connect from IF true output → this node → “Check Extraction Success”
   - Add **Extract from File** node “Extract Text from DOCX”
     - Operation: `text`
     - Connect from IF false output → this node → “Check Extraction Success”
   - Add **Extract from File** node “Extract Text from TXT”
     - Operation: `text`
     - Connect from IF false output → this node → “Check Extraction Success”

8) **Node 7: IF (Check Extraction Success)**
   - Add node: **IF**
   - Conditions (AND):
     - String → notEmpty: `={{ $json.text }}`
     - Number → greater than: `={{ $json.text.length }}` > `50`
   - True output → analysis agent
   - False output → error email

9) **Node 8: Gmail (Send Error Notification)**
   - Add node: **Gmail** (Send)
   - To: `={{ $('Workflow Configuration').item.json.errorNotificationEmail }}`
   - Subject and HTML body similar to workflow (include file name/mimeType and troubleshooting)
   - Connect from IF false output → this node.

10) **AI Analysis subgraph**
   1. Add **Groq Chat Model** (`lmChatGroq`)
      - Model: `llama-3.3-70b-versatile`
      - Configure Groq credential
   2. Add **Structured Output Parser** (`outputParserStructured`)
      - Provide the analysis JSON schema example (summary, clientName, etc.)
   3. Add **LangChain Agent** “Analyze Client Brief”
      - Text: `={{ $json.text }}`
      - System message: strategic analyst instructions (12 items)
      - Enable “has output parser”
      - Connect AI:
        - Groq Chat Model → Agent (ai_languageModel)
        - Structured Output Parser → Agent (ai_outputParser)
      - Connect main flow:
        - From “Check Extraction Success” true output → Analyze Client Brief

11) **AI Research subgraph**
   1. Add **Groq Research Model** (`lmChatGroq`)
      - Model: `llama-3.3-70b-versatile`
   2. Add tools:
      - **Wikipedia Research Tool**
      - **SerpAPI Google Search** (configure SerpAPI credential)
   3. Add **Research Output Parser** (`outputParserStructured`)
      - Provide research schema example (industryTrends, competitorInsights, etc.)
   4. Add **LangChain Agent** “Deep Industry Research”
      - Prompt references analysis fields:
        - `industry`, `projectType`, `clientName` from the Analyze node output
      - Enable “has output parser”
      - Connect AI:
        - Groq Research Model → agent (ai_languageModel)
        - Wikipedia tool + SerpAPI tool → agent (ai_tool)
        - Research Output Parser → agent (ai_outputParser)
      - Connect main:
        - Analyze Client Brief → Deep Industry Research

12) **Node: Merge (Combine Analysis and Research)**
   - Add node: **Merge**
   - Mode: `combine`
   - Combine by: `combineAll`
   - Connect:
     - Analyze Client Brief → Merge input 0
     - Deep Industry Research → Merge input 1

13) **Node: Code (Generate Comprehensive Report)**
   - Add node: **Code**
   - Paste/build logic that:
     - Reads analysis + research from merge inputs
     - Generates `reportContent` markdown
     - Computes `stats`
     - Outputs JSON: `{ reportContent, briefAnalysis, industryResearch, stats, ... }`
   - Connect Merge → Code.

14) **Node: Google Docs (Create Google Doc Report)**
   - Add node: **Google Docs**
   - Operation: create document
   - Title: `📊 Client Brief Analysis - {{ $('New File in Client Briefs Folder').item.json.name }}`
   - Folder ID: `={{ $('Workflow Configuration').item.json.clientSummariesFolderId }}`
   - Connect Code → Google Docs.
   - (If you want the report body populated, add a subsequent Docs step to insert/append `{{$json.reportContent}}`.)

15) **Node: Google Sheets (Log to Tracking Sheet)**
   - Add node: **Google Sheets**
   - Operation: `appendOrUpdate`
   - Document ID: `={{ $('Workflow Configuration').item.json.trackingSheetId }}`
   - Sheet name: `Brief Analysis Log`
   - Map columns (Date, Status, Doc URL, etc.) using expressions from the Code node output + trigger name.
   - Connect Google Docs → Google Sheets.

16) **Node: Gmail (Send Email to Account Manager)**
   - Add node: **Gmail** (Send)
   - To: `={{ $('Workflow Configuration').item.json.accountManagerEmail }}`
   - Subject: `🎯 Comprehensive Client Brief Analysis Ready: {{ file name }}`
   - HTML body:
     - Include `briefAnalysis.*`, `stats.*`, and link to `documentUrl`
   - Connect Google Sheets → Gmail.

17) **Activate workflow**
   - Upload a PDF/DOCX/TXT into the watched folder to test end-to-end.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Auto Client Brief Analyzer” setup steps, required “Brief Analysis Log” columns, and credential checklist (Google + Groq + SerpAPI optional). | From sticky note: **Main Overview** |
| Author: Digimetalab; contact email: digimetalab@gmail.com; Telegram: https://t.me/digimetalab | From sticky note: **Author** |
| Other templates link: https://n8n.io/creators/digimetalab/ | From sticky note: **Author** |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.