Automate systematic literature reviews with Google Drive, GPT-4, Gemini, Qdrant and Airtable

https://n8nworkflows.xyz/workflows/automate-systematic-literature-reviews-with-google-drive--gpt-4--gemini--qdrant-and-airtable-12827


# Automate systematic literature reviews with Google Drive, GPT-4, Gemini, Qdrant and Airtable

## 1. Workflow Overview

**Title:** Automate systematic literature reviews with Google Drive, GPT-4, Gemini, Qdrant and Airtable

This workflow automates a **Systematic Literature Review (SLR)** pipeline for PDF papers stored in **Google Drive**. It iterates through PDFs in an “INBOX” folder, extracts full text, applies a strict **LLM-based inclusion/exclusion filter**, and then:

- **Excluded papers:** are logged to **Airtable** and moved to an “Excluded” folder.
- **Included papers:** are scored and classified into one of three mechanism categories, logged to **Airtable**, embedded with **Google Gemini embeddings**, inserted into one of three **Qdrant collections**, and moved to an “Included” folder.

### 1.1 Entry & Paper Discovery (Drive → list files)
Manual trigger starts a Google Drive search to fetch all files in a target folder.

### 1.2 Iteration Control (batch loop per file)
A Split-in-batches node iterates through file items and re-enters the loop after each paper is processed (included or excluded).

### 1.3 PDF Retrieval & Text Extraction
Each PDF is downloaded, then text is extracted from the binary PDF.

### 1.4 Text Cleanup (bibliography trimming)
A Code node truncates text at “References/Bibliography…” unless an appendix is detected after the references.

### 1.5 Stage 2 Decision Agent (SLR inclusion/exclusion)
An LLM Chain (“SLR Agent”) produces **structured JSON** (via Structured Output Parser) including DOI/Author/Title and a strict decision.

### 1.6 Decision Routing + Logging (Airtable + Drive move)
An IF node routes excluded papers to Airtable logging + move to excluded folder; included papers continue to scoring.

### 1.7 Stage 3 Scoring + Mechanism Classification
A second LLM Chain (“Scoring Agent”) returns a structured JSON with mechanism category, abstract, and a 0–20 score.

### 1.8 Stage 4 Vector Storage (Qdrant) + File Organization
A Switch routes included papers into one of three Qdrant collections. Documents are loaded via Default Data Loader nodes and embedded via Gemini Embeddings nodes. Finally, the PDF is moved to the included folder and the loop continues.

---

## 2. Block-by-Block Analysis

### Block A — Entry & Google Drive File Discovery
**Overview:** Starts the workflow manually and retrieves all files from a specific Google Drive folder (INBOX) to be processed.

**Nodes Involved:**
- When clicking ‘Execute workflow’
- Search files and folders

#### Node: When clicking ‘Execute workflow’
- **Type / Role:** Manual Trigger; entry point.
- **Configuration:** No parameters.
- **Outputs:** Triggers **Search files and folders**.
- **Failure modes:** None (except workflow disabled / permission issues in n8n instance).

#### Node: Search files and folders
- **Type / Role:** Google Drive node; lists files in a folder.
- **Configuration choices:**
  - **Resource:** fileFolder
  - **Operation:** search (query)
  - **Folder:** `INBOX` (Drive folder ID `167cyFTL95Xo0z5hWO250raqIvryIeHp8`)
  - **Return all:** enabled
- **Credentials:** Google Drive OAuth2 (“Jannik.hiller02”).
- **Outputs:** List of file items containing at least `id` used later.
- **Edge cases / failures:**
  - OAuth token expired / insufficient Drive scopes.
  - Folder ID invalid or not accessible.
  - Non-PDF files returned (later PDF extraction may fail).

---

### Block B — Looping / Batch Control
**Overview:** Iterates through each file item. The workflow is designed to return to the loop after moving a file to included/excluded folders.

**Nodes Involved:**
- Loop Over Items

#### Node: Loop Over Items
- **Type / Role:** SplitInBatches; provides iterative processing.
- **Configuration choices:**
  - `reset: false` (keeps internal cursor across loopbacks during one run)
  - Batch size is not explicitly shown (defaults apply unless set in UI).
- **Connections:**
  - Input: **Search files and folders**
  - Output (index 1) → **Download PDF**
  - Loopback inputs from:
    - Move file to Excluded Folder
    - Move file to included folder / included folder1 / included folder2
- **Edge cases / failures:**
  - If there is no connection from output index 0 to continue/stop, behavior relies on n8n’s batching semantics; misconfiguration can stall.
  - If downstream nodes error for one item, the loop stops unless error handling is configured globally.

---

### Block C — PDF Download & Text Extraction
**Overview:** Downloads each Drive file as binary PDF and extracts its text content for downstream LLM processing.

**Nodes Involved:**
- Download PDF
- Extract PDF Text

#### Node: Download PDF
- **Type / Role:** Google Drive; downloads file binary.
- **Configuration choices:**
  - Operation: **download**
  - File ID: `={{ $json.id }}` (from the current loop item)
- **Credentials:** Google Drive OAuth2 (“Google Drive account”).
- **Connections:** Output → **Extract PDF Text**
- **Edge cases / failures:**
  - File not found / permissions.
  - File is not a PDF or is Google Doc-type that can’t be downloaded as PDF as-is (depending on Drive settings).
  - Large PDFs can cause memory/timeouts.

#### Node: Extract PDF Text
- **Type / Role:** Extract From File; OCR/text extraction from PDF binary.
- **Configuration choices:**
  - Operation: **pdf**
  - `keepSource: binary` (preserves binary in output for potential later use; here primarily text is used)
- **Connections:** Output → **Cut of bibliography**
- **Edge cases / failures:**
  - Scanned PDFs with no embedded text may yield empty output unless OCR is supported/added elsewhere.
  - Corrupt PDFs can fail extraction.

---

### Block D — Text Cleanup / Bibliography Cutting
**Overview:** Removes references/bibliography sections to reduce noise for LLM evaluation, but keeps them if an appendix exists after references.

**Nodes Involved:**
- Cut of bibliography

#### Node: Cut of bibliography
- **Type / Role:** Code node (JavaScript); normalizes structure and truncates text.
- **Key configuration / logic:**
  - Uses `inputFieldName = 'text'` (expects extracted text in `item.json.text`)
  - Initializes defaults to avoid expression failures later:
    - Ensures `item.json.info` exists
    - Ensures `item.json.info.Custom` exists
    - Ensures `item.json.metadata` exists
  - Cut-off keywords: `References`, `Bibliography`, `Works Cited`, `LITERATURVERZEICHNIS`
  - Protected keywords after cut point: `Appendix`, `Appendices`, `Anhang`
  - Sets:
    - `item.json.text_cleaned` (final text used by LLMs and vector store)
    - `item.json.cleaning_status` (debug string)
- **Connections:** Output → **SLR Agent**
- **Edge cases / failures:**
  - If extracted text is missing, the node sets `text_cleaned` to empty and continues (LLM may then exclude due to lack of evidence).
  - Regex assumptions may miss non-standard headings (e.g., “REFERENCES” without newline formatting).

---

### Block E — Stage 2: SLR Decision Agent (Include/Exclude)
**Overview:** Uses an LLM chain with a strict prompt to decide inclusion/exclusion based on 4 IC and 4 EC criteria, returning structured JSON.

**Nodes Involved:**
- OpenAI Chat Model
- Structured Output Parser
- Google Gemini Chat Model1 (present but not actually used by the decision chain)
- SLR Agent

#### Node: OpenAI Chat Model
- **Type / Role:** LangChain Chat Model (OpenAI-compatible); provides the language model for SLR Agent.
- **Configuration choices:**
  - Model: `openai/gpt-oss-20b` (selected via expression)
  - Credentials named “Huggingface” (suggests OpenAI-compatible endpoint provided by HF or similar gateway).
  - Built-in tools: none.
- **Connections:** `ai_languageModel` → **SLR Agent**
- **Edge cases / failures:**
  - Wrong base URL / incompatible OpenAI API semantics on the provider.
  - Rate limits and context window overflow (full paper text can exceed limits).

#### Node: Structured Output Parser
- **Type / Role:** Structured output parser; validates and optionally auto-fixes JSON.
- **Configuration choices:**
  - `autoFix: true`
  - Schema example includes fields (DOI, Author, Titel, Abstract, Decision, etc.)
- **Connections:**
  - `ai_outputParser` → **SLR Agent**
  - Also has an `ai_languageModel` input coming from **Google Gemini Chat Model1**, but this parser is already used by SLR Agent; the Gemini model connection is unusual and likely unused in practice.
- **Edge cases / failures:**
  - If the LLM returns non-JSON or incomplete JSON, autoFix attempts repair but can still fail.
  - Schema example includes “Included or excluded” vs the SLR Agent prompt expects exactly “Included” or “Excluded”; inconsistencies may cause parsing instability.

#### Node: Google Gemini Chat Model1
- **Type / Role:** Gemini chat model node.
- **Configuration:** Default options; credentials “Google Gemini(PaLM) Api account”.
- **Connections:** `ai_languageModel` → **Structured Output Parser**
- **Practical note:** The workflow’s SLR Agent uses **OpenAI Chat Model** as its language model. This Gemini model appears connected to the parser node but does not meaningfully drive the chain unless manually rewired. Consider removing or explicitly using it.
- **Edge cases:** API auth errors, region restrictions, quota.

#### Node: SLR Agent
- **Type / Role:** LangChain LLM Chain; applies strict IC/EC filter and outputs structured decision JSON.
- **Configuration choices (interpreted):**
  - Input text: combines:
    - `info.Author`, `info.Creator`, `info.Title`
    - `metadata`
    - `text_cleaned`
  - Prompt: German/English mixed strict SLR rubric; demands JSON-only output with Decision and PRISMA-style reasoning.
  - `hasOutputParser: true` (paired with Structured Output Parser).
- **Connections:**
  - Main output → **If**
  - Inputs:
    - LLM model from **OpenAI Chat Model**
    - Output parser from **Structured Output Parser**
- **Edge cases / failures:**
  - Missing metadata fields (mitigated by the Code node’s initialization).
  - Token overflow due to large PDFs.
  - Prompt requires very strict format; without parser, downstream IF would fail.

---

### Block F — Stage 3: Decision Routing & Exclusion Logging
**Overview:** Branches based on `output.Decision`. Excluded papers are logged and moved to an excluded Drive folder.

**Nodes Involved:**
- If
- Log Excluded Paper
- Move file to Excluded Folder

#### Node: If
- **Type / Role:** IF condition; checks inclusion decision.
- **Condition:** `={{ $json.output.Decision }}` equals `"Included"`.
- **Connections:**
  - True → **Scoring Agent**
  - False → **Log Excluded Paper**
- **Edge cases / failures:**
  - If `output.Decision` missing or spelled differently (e.g., “included”), strict equals will route to excluded path.
  - If the output parser failed and `output` isn’t present, the IF expression can error.

#### Node: Log Excluded Paper
- **Type / Role:** Airtable; creates a record for excluded papers.
- **Configuration choices:**
  - Base: `SLR` (appMIPges4F2ST4Sm)
  - Table: `Table 1` (tbltF6eQKLmLth73e)
  - Fields mapped (key ones):
    - DOI, Titel, Author from current `$json.output...`
    - Abstract/Decision/Reasoning/Notes from `$('SLR Agent').item.json.output...`
    - Score set to `0`
- **Credentials:** Airtable token “SLR Base”.
- **Connections:** Output → **Move file to Excluded Folder**
- **Edge cases / failures:**
  - Field name mismatches in Airtable (e.g., “Titel” vs “Title”).
  - Airtable rate limits.
  - If SLR Agent output lacks certain fields, Airtable create may fail depending on constraints.

#### Node: Move file to Excluded Folder
- **Type / Role:** Google Drive; moves processed file to excluded folder.
- **Configuration:**
  - Operation: move
  - File ID: `={{ $('Loop Over Items').item.json.id }}`
  - Destination folder: `Process Excluded` (ID `1PCQCVbps32P08Ojng944kqXmMDiGfU6e`)
- **Credentials:** Google Drive OAuth2 (“Jannik.hiller02”).
- **Connections:** Output → **Loop Over Items** (loopback)
- **Edge cases / failures:**
  - File already in destination (Drive behavior varies).
  - No permission to move between folders (shared drives can be stricter).

---

### Block G — Stage 3: Included Paper Scoring + Mechanism Classification
**Overview:** For included papers, a second LLM chain produces a mechanism label, abstract, and quality score, then logs the results.

**Nodes Involved:**
- OpenAI Chat Model1
- Structured Output Parser1
- Google Gemini Chat Model3 (present but not actually used by the scoring chain)
- Scoring Agent
- Log Included folder

#### Node: OpenAI Chat Model1
- **Type / Role:** LangChain Chat Model; model for Scoring Agent.
- **Configuration:** Same as OpenAI Chat Model (model `openai/gpt-oss-20b`, credentials “Huggingface”).
- **Connections:** `ai_languageModel` → **Scoring Agent**
- **Edge cases:** same as earlier (rate limits, token overflow).

#### Node: Structured Output Parser1
- **Type / Role:** Structured output parser for scoring JSON.
- **Configuration:**
  - `autoFix: true`
  - Schema example expects DOI/Author/Titel/Mechanism/Abstract/Decision/Decision Reasoning/Score/Additional notes
- **Connections:**
  - `ai_outputParser` → **Scoring Agent**
  - `ai_languageModel` input from **Google Gemini Chat Model3** (again, likely unnecessary unless rewired)
- **Edge cases:** parser repair may produce incorrect numeric types; keep Airtable field types aligned.

#### Node: Google Gemini Chat Model3
- **Type / Role:** Gemini chat model node.
- **Connections:** `ai_languageModel` → **Structured Output Parser1**
- **Practical note:** Not the scoring chain’s LLM model (that is OpenAI Chat Model1). This is likely leftover wiring.
- **Edge cases:** quota/auth.

#### Node: Scoring Agent
- **Type / Role:** LangChain LLM Chain; scores and classifies included papers.
- **Configuration choices:**
  - Input text: from `$('Cut of bibliography').item.json...` (Custom, metadata, text_cleaned)
  - Prompt: defines 3 mechanism categories and 10 scoring criteria (0–2 each), total 0–20.
  - Structured JSON output required.
- **Connections:**
  - Main output → **Route to sub topic** AND → **Log Included folder** (parallel)
- **Edge cases / failures:**
  - Score must be integer 0–20; LLM may output float/string; parser tries to fix.
  - Mechanism label must exactly match the Switch rules; otherwise it won’t route to any Qdrant collection.

#### Node: Log Included folder
- **Type / Role:** Airtable create record for included papers.
- **Configuration:**
  - Writes DOI/Author/Titel/Abstract/Mechanism/Score/Score Reasoning, and also:
    - Decision Reasoning from SLR Agent (PRISMA reasoning)
    - Additional notes from SLR Agent
  - Decision is hard-coded to `"Included"`
- **Connections:** No downstream node.
- **Edge cases / failures:**
  - Airtable “Mechanism” is an options field; must match exactly one of the configured option values.

---

### Block H — Mechanism Routing to Qdrant Collections
**Overview:** Routes included papers into a Qdrant collection based on mechanism label.

**Nodes Involved:**
- Route to sub topic
- Vector Store - collection 1
- Vector Store - collection 2
- Vector Store - collection 3

#### Node: Route to sub topic
- **Type / Role:** Switch; selects branch by `output.Mechanism`.
- **Rules (exact matches):**
  - `Self-Referential Prompting`
  - `Reflective Evaluation`
  - `Iterative Self-Correction / Debate Mechanismen`
- **Connections:** Each output routes to one Vector Store node.
- **Edge cases / failures:**
  - If mechanism string differs by spacing/case/punctuation, no route matches (document won’t be inserted, file may never be moved, loop may stall).
  - Consider adding a default branch (not present).

#### Node: Vector Store - collection 1 / 2 / 3
- **Type / Role:** Qdrant Vector Store insert; stores embeddings + document payload.
- **Configuration choices:**
  - Mode: **insert**
  - Collection: each node points to a different Qdrant collection (names include German descriptions).
  - Credentials: Qdrant API “Self-hosted”.
- **Connections:**
  - Each Vector Store receives:
    - `ai_document` from its corresponding Default Data Loader node
    - `ai_embedding` from its corresponding Embeddings Google Gemini node
  - Main output goes to a corresponding “Move file to included folderX” node.
- **Edge cases / failures:**
  - Collection must already exist (depending on Qdrant node settings/version).
  - Payload size limits; embedding failures for extremely long documents.
  - Network/TLS/auth errors against self-hosted Qdrant.

---

### Block I — Document Preparation + Embeddings for Qdrant
**Overview:** Builds the document text to embed and insert, and generates Gemini embeddings (separate pipeline per collection).

**Nodes Involved:**
- Default Data Loader
- Default Data Loader1
- Default Data Loader2
- Embeddings Google Gemini2
- Embeddings Google Gemini1
- Embeddings Google Gemini

#### Nodes: Default Data Loader / Default Data Loader1 / Default Data Loader2
- **Type / Role:** LangChain document loader; creates a Document object from provided text.
- **Configuration:**
  - `jsonMode: expressionData`
  - `jsonData` concatenates:
    - `$('Cut of bibliography').item.json.info.Custom`
    - `$('Cut of bibliography').item.json.metadata`
    - `$('Cut of bibliography').item.json.text_cleaned`
- **Connections:**
  - Each loader’s `ai_document` output goes to its corresponding Vector Store node (1/2/3).
- **Edge cases / failures:**
  - If `info.Custom` is an object, concatenation may produce `[object Object]`. If you expect readable metadata, stringify it explicitly in the expression.
  - Very large documents can cause embedding model failures; consider chunking.

#### Nodes: Embeddings Google Gemini / 1 / 2
- **Type / Role:** Gemini embeddings provider.
- **Configuration:** Default; credentials “Google Gemini(PaLM) Api account”.
- **Connections:**
  - Each node’s `ai_embedding` output feeds one Vector Store node.
- **Edge cases / failures:**
  - API quota/rate limits.
  - Input length constraints; embedding endpoints often have max tokens/chars.

---

### Block J — File Organization for Included Papers (Drive move)
**Overview:** After inserting into Qdrant, moves the processed file into the “Processed Included” folder and returns to the loop.

**Nodes Involved:**
- Move file to included folder
- Move file to included folder1
- Move file to included folder2

#### Nodes: Move file to included folder / 1 / 2
- **Type / Role:** Google Drive move.
- **Configuration (all three are effectively identical):**
  - File ID: `={{ $('Loop Over Items').item.json.id }}`
  - Destination folder: `Processed Included` (ID `19W3qW76F2gxlMPkZnKA64HYCA-Od2OcK`)
- **Connections:** Each outputs back to **Loop Over Items**.
- **Why three nodes?** One per mechanism branch (collection 1/2/3). Functionally they do the same move.
- **Edge cases / failures:** same as excluded move.

---

## 3. Summary Table (All Nodes)

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | manualTrigger | Entry point | — | Search files and folders | ## Stage 1: Paper Input & Extraction |
| Search files and folders | googleDrive | List files from INBOX folder | When clicking ‘Execute workflow’ | Loop Over Items | ## Stage 1: Paper Input & Extraction |
| Loop Over Items | splitInBatches | Iterate over Drive files | Search files and folders; Move file to Excluded Folder; Move file to included folder; Move file to included folder1; Move file to included folder2 | Download PDF (batch output) | ## Stage 1: Paper Input & Extraction |
| Download PDF | googleDrive | Download PDF binary | Loop Over Items | Extract PDF Text | ## Stage 1: Paper Input & Extraction |
| Extract PDF Text | extractFromFile | Extract text from PDF binary | Download PDF | Cut of bibliography | ## Stage 1: Paper Input & Extraction |
| Cut of bibliography | code | Trim references; ensure default fields | Extract PDF Text | SLR Agent | ## Stage 1: Paper Input & Extraction |
| OpenAI Chat Model | lmChatOpenAi | LLM for SLR decision | — | SLR Agent (ai_languageModel) | ## Stage 2: LLM-Based SLR Review |
| Structured Output Parser | outputParserStructured | Parse/fix decision JSON | Google Gemini Chat Model1 (ai_languageModel) | SLR Agent (ai_outputParser) | ## Stage 2: LLM-Based SLR Review |
| Google Gemini Chat Model1 | lmChatGoogleGemini | Unused/aux model connection to parser | — | Structured Output Parser (ai_languageModel) | ## Stage 2: LLM-Based SLR Review |
| SLR Agent | chainLlm | Apply IC/EC criteria; produce decision JSON | Cut of bibliography; OpenAI Chat Model (ai); Structured Output Parser (ai) | If | ## Stage 2: LLM-Based SLR Review |
| If | if | Route Included vs Excluded | SLR Agent | Scoring Agent (true); Log Excluded Paper (false) | ## Stage 3: Decision Routing & Logging |
| Log Excluded Paper | airtable | Create Airtable record for excluded | If (false) | Move file to Excluded Folder | ## Stage 3: Decision Routing & Logging |
| Move file to Excluded Folder | googleDrive | Move file to excluded folder | Log Excluded Paper | Loop Over Items | ## Stage 4: Vector Store & File Organization |
| OpenAI Chat Model1 | lmChatOpenAi | LLM for scoring/classification | — | Scoring Agent (ai_languageModel) | ## Stage 3: Decision Routing & Logging |
| Structured Output Parser1 | outputParserStructured | Parse/fix scoring JSON | Google Gemini Chat Model3 (ai_languageModel) | Scoring Agent (ai_outputParser) | ## Stage 3: Decision Routing & Logging |
| Google Gemini Chat Model3 | lmChatGoogleGemini | Unused/aux model connection to parser | — | Structured Output Parser1 (ai_languageModel) | ## Stage 3: Decision Routing & Logging |
| Scoring Agent | chainLlm | Mechanism classification + 0–20 score | If (true); OpenAI Chat Model1 (ai); Structured Output Parser1 (ai) | Route to sub topic; Log Included folder | ## Stage 3: Decision Routing & Logging |
| Log Included folder | airtable | Create Airtable record for included | Scoring Agent | — | ## Stage 3: Decision Routing & Logging |
| Route to sub topic | switch | Route by mechanism label | Scoring Agent | Vector Store - collection 1; Vector Store - collection 2; Vector Store - collection 3 | ## Stage 4: Vector Store & File Organization |
| Default Data Loader | documentDefaultDataLoader | Build doc for collection 1 | — (references Cut of bibliography by expression) | Vector Store - collection 1 (ai_document) | ## Stage 4: Vector Store & File Organization |
| Embeddings Google Gemini2 | embeddingsGoogleGemini | Embeddings for collection 1 | — | Vector Store - collection 1 (ai_embedding) | ## Stage 4: Vector Store & File Organization |
| Vector Store - collection 1 | vectorStoreQdrant | Insert into Qdrant collection 1 | Route to sub topic; Default Data Loader (ai); Embeddings Google Gemini2 (ai) | Move file to included folder2 | ## Stage 4: Vector Store & File Organization |
| Default Data Loader1 | documentDefaultDataLoader | Build doc for collection 2 | — (references Cut of bibliography by expression) | Vector Store - collection 2 (ai_document) | ## Stage 4: Vector Store & File Organization |
| Embeddings Google Gemini1 | embeddingsGoogleGemini | Embeddings for collection 2 | — | Vector Store - collection 2 (ai_embedding) | ## Stage 4: Vector Store & File Organization |
| Vector Store - collection 2 | vectorStoreQdrant | Insert into Qdrant collection 2 | Route to sub topic; Default Data Loader1 (ai); Embeddings Google Gemini1 (ai) | Move file to included folder1 | ## Stage 4: Vector Store & File Organization |
| Default Data Loader2 | documentDefaultDataLoader | Build doc for collection 3 | — (references Cut of bibliography by expression) | Vector Store - collection 3 (ai_document) | ## Stage 4: Vector Store & File Organization |
| Embeddings Google Gemini | embeddingsGoogleGemini | Embeddings for collection 3 | — | Vector Store - collection 3 (ai_embedding) | ## Stage 4: Vector Store & File Organization |
| Vector Store - collection 3 | vectorStoreQdrant | Insert into Qdrant collection 3 | Route to sub topic; Default Data Loader2 (ai); Embeddings Google Gemini (ai) | Move file to included folder | ## Stage 4: Vector Store & File Organization |
| Move file to included folder | googleDrive | Move file to included folder (branch 3) | Vector Store - collection 3 | Loop Over Items | ## Stage 4: Vector Store & File Organization |
| Move file to included folder1 | googleDrive | Move file to included folder (branch 2) | Vector Store - collection 2 | Loop Over Items | ## Stage 4: Vector Store & File Organization |
| Move file to included folder2 | googleDrive | Move file to included folder (branch 1) | Vector Store - collection 1 | Loop Over Items | ## Stage 4: Vector Store & File Organization |
| 📋 Overview: SLR Paper Review Agent | stickyNote | Documentation / overview | — | — | (This is the sticky note itself) |
| Stage 1: Paper Input & Extraction | stickyNote | Stage label | — | — | (This is the sticky note itself) |
| Stage 2: LLM-Based SLR Review | stickyNote | Stage label | — | — | (This is the sticky note itself) |
| Stage 3: Decision Routing & Logging | stickyNote | Stage label | — | — | (This is the sticky note itself) |
| Stage 4: Vector Store & File Organization | stickyNote | Stage label | — | — | (This is the sticky note itself) |

---

## 4. Reproducing the Workflow from Scratch (n8n)

1. **Create a new workflow** and name it:  
   **“Automate systematic literature reviews with Google Drive, GPT-4, Gemini, Qdrant and Airtable”**

2. **Add Manual Trigger**
   - Node: **Manual Trigger**
   - Name: “When clicking ‘Execute workflow’”

3. **Add Google Drive node to list files**
   - Node: **Google Drive**
   - Name: “Search files and folders”
   - Resource: **File/Folder**
   - Operation: **Search**
   - Search scope: **Files**
   - Folder: select your **INBOX** folder (store its folder ID)
   - Return All: **true**
   - Credentials: configure **Google Drive OAuth2** with access to that folder
   - Connect: Manual Trigger → Search files and folders

4. **Add Split in Batches**
   - Node: **Split In Batches**
   - Name: “Loop Over Items”
   - Keep `Reset` = **false**
   - Connect: Search files and folders → Loop Over Items

5. **Add Google Drive Download**
   - Node: **Google Drive**
   - Name: “Download PDF”
   - Operation: **Download**
   - File ID: expression `{{$json.id}}`
   - Credentials: Google Drive OAuth2
   - Connect: Loop Over Items (batch output) → Download PDF

6. **Add PDF text extraction**
   - Node: **Extract from File**
   - Name: “Extract PDF Text”
   - Operation: **PDF**
   - Options: enable **Keep Source (binary)**
   - Connect: Download PDF → Extract PDF Text

7. **Add Code node for bibliography trimming**
   - Node: **Code**
   - Name: “Cut of bibliography”
   - Paste logic that:
     - reads `item.json.text`
     - writes `item.json.text_cleaned`
     - initializes `item.json.info.Custom` and `item.json.metadata` if missing
   - Connect: Extract PDF Text → Cut of bibliography

8. **Add SLR LLM chain (decision)**
   - Node: **LLM Chain** (LangChain)
   - Name: “SLR Agent”
   - Text input: include `info.*`, `metadata`, and `text_cleaned` via expressions from the Code node output.
   - Prompt: paste your strict IC/EC rubric and demand JSON-only output with fields:
     - DOI, Author, Titel, Decision, Decision Reasoning
   - Enable **Output Parser** on the chain.

9. **Add Structured Output Parser (decision)**
   - Node: **Structured Output Parser**
   - Name: “Structured Output Parser”
   - Enable **Auto-fix**
   - Provide a schema example matching the expected JSON.
   - Connect parser to the SLR Agent via the node’s **ai_outputParser** connection.

10. **Add Chat Model for SLR Agent**
   - Node: **OpenAI Chat Model** (LangChain)
   - Name: “OpenAI Chat Model”
   - Select model (example used): `openai/gpt-oss-20b`
   - Credentials:
     - Configure an **OpenAI-compatible** credential (may be OpenAI, Azure OpenAI, or HF Inference in OpenAI-compatible mode).
   - Connect model to SLR Agent via **ai_languageModel**.

11. **Add IF routing**
   - Node: **IF**
   - Name: “If”
   - Condition: String equals  
     - Left: `{{$json.output.Decision}}`
     - Right: `Included`
   - Connect: SLR Agent → If

12. **Excluded branch: Airtable logging**
   - Node: **Airtable**
   - Name: “Log Excluded Paper”
   - Operation: **Create**
   - Base/Table: select your SLR base/table
   - Map fields (examples):
     - DOI: `{{$json.output.DOI}}`
     - Titel: `{{$json.output.Titel}}`
     - Author: `{{$json.output.Author}}`
     - Abstract/Decision/Reasoning: from `$('SLR Agent').item.json.output...`
     - Score: `0`
   - Credentials: Airtable Personal Access Token (PAT)
   - Connect: If (false) → Log Excluded Paper

13. **Excluded branch: Move file**
   - Node: **Google Drive**
   - Name: “Move file to Excluded Folder”
   - Operation: **Move**
   - File ID: `{{ $('Loop Over Items').item.json.id }}`
   - Destination folder: your “Process Excluded”
   - Connect: Log Excluded Paper → Move file to Excluded Folder
   - Loopback: Move file to Excluded Folder → Loop Over Items

14. **Included branch: Scoring LLM chain**
   - Node: **LLM Chain**
   - Name: “Scoring Agent”
   - Input text: use `$('Cut of bibliography').item.json.info.Custom`, `.metadata`, `.text_cleaned`
   - Prompt: mechanism classification + 10 criteria scoring (0–2), sum 0–20; output JSON.
   - Enable Output Parser.
   - Connect: If (true) → Scoring Agent

15. **Add Structured Output Parser (scoring)**
   - Node: **Structured Output Parser**
   - Name: “Structured Output Parser1”
   - Auto-fix: enabled
   - Schema example includes Mechanism, Abstract, Score (0–20 integer).
   - Connect to Scoring Agent via **ai_outputParser**.

16. **Add Chat Model for Scoring Agent**
   - Node: **OpenAI Chat Model**
   - Name: “OpenAI Chat Model1”
   - Same model selection as above (or a different one).
   - Connect to Scoring Agent via **ai_languageModel**.

17. **Included logging in Airtable**
   - Node: **Airtable**
   - Name: “Log Included folder”
   - Operation: Create
   - Map:
     - Decision: hardcode “Included”
     - Mechanism/Abstract/Score from Scoring Agent output
     - Decision Reasoning from SLR Agent output (PRISMA)
     - Score Reasoning from Scoring Agent decision reasoning
   - Connect: Scoring Agent → Log Included folder

18. **Mechanism routing switch**
   - Node: **Switch**
   - Name: “Route to sub topic”
   - Add 3 rules (string equals) on `{{$json.output.Mechanism}}`:
     1) Self-Referential Prompting  
     2) Reflective Evaluation  
     3) Iterative Self-Correction / Debate Mechanismen
   - Connect: Scoring Agent → Route to sub topic

19. **Create 3 Qdrant vector store insert nodes**
   - Node: **Qdrant Vector Store**
   - Names:
     - “Vector Store - collection 1”
     - “Vector Store - collection 2”
     - “Vector Store - collection 3”
   - Mode: **Insert**
   - Choose the Qdrant collection for each mechanism category.
   - Credentials: configure Qdrant URL + API key (self-hosted or cloud).
   - Connect:
     - Switch output 1 → Vector Store 1
     - Switch output 2 → Vector Store 2
     - Switch output 3 → Vector Store 3

20. **Create 3 Default Data Loader nodes**
   - Node: **Default Data Loader**
   - Names: “Default Data Loader”, “Default Data Loader1”, “Default Data Loader2”
   - Provide the document content as one expression concatenating Custom + metadata + text_cleaned.
   - Connect each loader’s **ai_document** to the corresponding Vector Store node.

21. **Create 3 Gemini Embeddings nodes**
   - Node: **Google Gemini Embeddings**
   - Names: “Embeddings Google Gemini2”, “Embeddings Google Gemini1”, “Embeddings Google Gemini”
   - Credentials: Google Gemini / PaLM API key credential.
   - Connect each embeddings node’s **ai_embedding** to the corresponding Vector Store node.

22. **Add “move to included folder” nodes (one per branch)**
   - Node: **Google Drive**
   - Names: “Move file to included folder”, “…1”, “…2”
   - Operation: **Move**
   - File ID: `{{ $('Loop Over Items').item.json.id }}`
   - Destination folder: “Processed Included”
   - Connect:
     - Vector Store 1 → Move included folder2 → Loop Over Items
     - Vector Store 2 → Move included folder1 → Loop Over Items
     - Vector Store 3 → Move included folder → Loop Over Items

23. **(Optional cleanup) Remove or rewire unused Gemini chat model nodes**
   - In the provided workflow, “Google Gemini Chat Model1/3” are connected to output parsers, while the chains use OpenAI chat models. If you intend Gemini to be the LLM, connect Gemini directly to the chain’s **ai_languageModel** instead.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is designed as an “SLR Paper Review Agent”: downloads PDFs from Drive, applies strict IC/EC screening, logs decisions to Airtable, and stores included papers in Qdrant for semantic search. | From sticky note “📋 Overview: SLR Paper Review Agent” |
| Setup reminders: connect Google Drive, Airtable base/table, Qdrant collections, LLM credentials; customize inclusion/exclusion and scoring prompts; adjust folders. | From sticky note “📋 Overview: SLR Paper Review Agent” |
| Disclaimer (provided by user): “Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… Toutes les données manipulées sont légales et publiques.” | Compliance/context note |

