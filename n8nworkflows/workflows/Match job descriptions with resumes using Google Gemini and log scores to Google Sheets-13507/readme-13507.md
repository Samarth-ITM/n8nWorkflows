Match job descriptions with resumes using Google Gemini and log scores to Google Sheets

https://n8nworkflows.xyz/workflows/match-job-descriptions-with-resumes-using-google-gemini-and-log-scores-to-google-sheets-13507


# Match job descriptions with resumes using Google Gemini and log scores to Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow accepts a candidate’s resume (PDF) and a job description URL via an n8n Form. It extracts text from both, uses **Google Gemini** to (1) extract a clean job description, (2) evaluate resume↔JD fit with a recruiter-style report and a **0–10 score**, (3) extract candidate identity details (name/email), and then **appends one row** with the results into **Google Sheets**.

**Primary use cases:**
- Lightweight resume screening for recruiters or hiring managers.
- Centralized logging of candidate evaluation outcomes in a sheet for ranking/triage.
- Standardized structured output for downstream automations (emailing, ATS entry, Slack alerts).

### Logical Blocks
1. **1.1 Input Reception (Form Trigger)**
2. **1.2 Content Extraction (Resume PDF → text, JD URL → page content → JD text)**
3. **1.3 Preparation & Aggregation (merge resume + JD into one aggregated object)**
4. **1.4 AI Screening (Gemini-powered recruiter agent + structured JSON parsing)**
5. **1.5 Identity Extraction & Google Sheets Logging (name/email extraction + append row)**

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (Form Trigger)

**Overview:**  
Collects the resume file and the job description link from a web form submission and fans out into two parallel branches: resume extraction and JD fetching.

**Nodes involved:**
- On form submission

#### Node: **On form submission**
- **Type / role:** `n8n-nodes-base.formTrigger` — entry point; collects user inputs.
- **Configuration (interpreted):**
  - Form title: **“Check Your Resume Score”**
  - Fields:
    - **Resume** (File, required)
    - **Job Description Link** (Text/URL, required)
- **Key variables/fields produced:**
  - Binary file property named **`Resume`**
  - JSON field **`Job Description Link`**
- **Outputs / connections:**
  - Output → **HTTP Request** (fetch JD page)
  - Output → **Extract from File2** (extract resume PDF text)
- **Edge cases / failures:**
  - Missing/invalid URL in “Job Description Link” → downstream HTTP failure.
  - Uploaded non-PDF or scanned PDF → extraction may produce empty/garbled text.
  - Large files may exceed form/upload or instance limits.

---

### 2.2 Content Extraction (Resume + JD)

**Overview:**  
Converts the uploaded resume PDF into text, and retrieves the JD page HTML/content then extracts the job description as clean text using the Gemini-backed extractor.

**Nodes involved:**
- Extract from File2
- Set Resume
- HTTP Request
- Information Extractor1
- Google Gemini Chat Model (shared model for AI nodes)

#### Node: **Extract from File2**
- **Type / role:** `n8n-nodes-base.extractFromFile` — extracts text from a PDF.
- **Configuration (interpreted):**
  - Operation: **PDF**
  - Binary property name: **Resume**
- **Input:** Form trigger item containing binary `Resume`.
- **Output:** JSON containing extracted text (commonly in a field like `text`).
- **Connections:** Output → **Set Resume**
- **Edge cases / failures:**
  - Encrypted/password-protected PDFs.
  - Image-only PDFs (no embedded text) → extraction returns little/no text.
  - Very large PDFs can cause timeouts or memory pressure.

#### Node: **Set Resume**
- **Type / role:** `n8n-nodes-base.set` — normalizes resume text into a predictable key.
- **Configuration (interpreted):**
  - Creates field **`resume`** as: `{{ $json.text }}`
- **Input:** extracted PDF text from **Extract from File2**
- **Output:** `{ resume: "<resume text>" }`
- **Connections:** Output → **Merge** (input 0)
- **Edge cases:**
  - If extractor output is not `text` (varies by node version/file type), `resume` becomes empty → weak AI evaluation.

#### Node: **HTTP Request**
- **Type / role:** `n8n-nodes-base.httpRequest` — downloads job description page content.
- **Configuration (interpreted):**
  - URL: `{{ $json['Job Description Link'] }}`
  - Defaults for method/options (GET assumed)
- **Input:** Form trigger JSON with JD link.
- **Output:** Response body in `data` (typical for this node), plus metadata.
- **Connections:** Output → **Information Extractor1**
- **Edge cases / failures:**
  - Non-200 responses (403/404), bot protection, Cloudflare, login-required pages.
  - HTML-heavy pages: extractor may capture navigation/boilerplate, hurting accuracy.
  - Redirect loops or invalid URL format.

#### Node: **Information Extractor1**
- **Type / role:** `@n8n/n8n-nodes-langchain.informationExtractor` — uses an LLM to extract a specific attribute from raw content.
- **Configuration (interpreted):**
  - Text input: `{{ $json.data }}`
  - Extracted attributes:
    - **Job description** (required)
- **Model:** Uses **Google Gemini Chat Model** via the AI connection.
- **Input:** HTTP response content.
- **Output:** Typically `{ output: { "Job description": "..." } }`
- **Connections:** Output → **Merge** (input 1)
- **Edge cases / failures:**
  - If `$json.data` is not plain text (binary/JSON), extraction may fail or return nonsense.
  - Model may hallucinate if the page content is sparse or blocked.
  - Token limits if JD pages are extremely long.

#### Node: **Google Gemini Chat Model**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — shared LLM provider for all LangChain AI nodes.
- **Configuration (interpreted):**
  - Uses configured Gemini API credentials.
  - Default model/options (not explicitly set in parameters).
- **Connected as AI model to:**
  - Information Extractor1
  - Recruiter Agent
  - Information Extractor (candidate identity)
- **Edge cases / failures:**
  - Invalid/expired API key, quota exceeded.
  - Model availability/region issues.
  - Safety blocks depending on content and Gemini settings.

---

### 2.3 Preparation & Aggregation

**Overview:**  
Combines the resume text and extracted job description into a single data structure and aggregates them so the recruiter agent can reference both as `$json.data[0]` and `$json.data[1]`.

**Nodes involved:**
- Merge
- Aggregate

#### Node: **Merge**
- **Type / role:** `n8n-nodes-base.merge` — merges two incoming branches.
- **Configuration (interpreted):**
  - Uses default merge behavior (no explicit mode shown).
  - Practically, it outputs two items (one from each input) that are then aggregated.
- **Inputs / connections:**
  - Input 0: from **Set Resume**
  - Input 1: from **Information Extractor1**
- **Output / connections:** Output → **Aggregate**
- **Edge cases:**
  - If one branch fails (e.g., JD fetch blocked), Merge may output incomplete data depending on execution behavior.
  - Misaligned item counts (not likely here—single submission—but can matter if batched).

#### Node: **Aggregate**
- **Type / role:** `n8n-nodes-base.aggregate` — aggregates all incoming items into a single item.
- **Configuration (interpreted):**
  - Mode: **aggregateAllItemData**
  - Produces a single item with a field like `data` containing an array of the merged items.
- **Input:** Items from Merge (resume item + JD item).
- **Output:** One item with `data[]`:
  - `data[0].resume`
  - `data[1].output['Job description']`
- **Connections:** Output → **Recruiter Agent**
- **Edge cases:**
  - If upstream created more than two items, indices used later (`data[0]`, `data[1]`) can shift and break mappings.

---

### 2.4 AI Evaluation & Structured Screening

**Overview:**  
Runs a recruiter-style evaluation prompt against the resume and JD, enforcing a strict structured JSON response using a schema-based output parser.

**Nodes involved:**
- Recruiter Agent
- Structured Output Parser
- Google Gemini Chat Model (AI dependency)

#### Node: **Structured Output Parser**
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces a JSON schema on LLM output.
- **Configuration (interpreted):**
  - Manual JSON schema named `resume_screening_evaluation` with required fields:
    - `candidate_strengths` (array of strings)
    - `candidate_weaknesses` (array of strings)
    - `risk_factor` { score: Low/Medium/High, explanation }
    - `reward_factor` { score: Low/Medium/High, explanation }
    - `overall_fit_rating` (integer 0–10)
    - `justification_for_rating` (string)
- **Connections:**
  - Connected to **Recruiter Agent** via `ai_outputParser`.
- **Edge cases:**
  - If the model output is not valid JSON or violates schema (e.g., decimals, missing fields), parsing fails and the workflow errors.

#### Node: **Recruiter Agent**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LLM agent that generates the screening report.
- **Configuration (interpreted):**
  - **Prompt (user text):**
    - `Candidate Resume:\n{{ $json.data[0].resume }}`
  - **System message:** detailed recruiter instructions and **includes the job description**:
    - `{{ $json.data[1].output['Job description'] }}`
  - Output parser enabled (`hasOutputParser: true`) to force schema compliance.
- **Model:** **Google Gemini Chat Model**
- **Input:** aggregated object from **Aggregate** containing `data[]`.
- **Output:** `output` object matching schema (e.g., `$json.output.overall_fit_rating`).
- **Connections:** Output → **Information Extractor** (to extract identity details)
- **Edge cases / failures:**
  - If `data[0].resume` or `data[1].output['Job description']` is empty, scoring becomes unreliable.
  - Token overflow if resume + JD too large; may truncate and reduce accuracy or fail.
  - Output parser failures when the model returns extra commentary or formatting.

---

### 2.5 Identity Extraction & Google Sheets Logging

**Overview:**  
Extracts candidate identity details from the resume text and appends a consolidated evaluation row into a Google Sheet.

**Nodes involved:**
- Information Extractor
- Append Data
- Google Gemini Chat Model (AI dependency)

#### Node: **Information Extractor**
- **Type / role:** `@n8n/n8n-nodes-langchain.informationExtractor` — extracts structured identity info from resume text.
- **Configuration (interpreted):**
  - Text: `{{ $('Set Resume').item.json.resume }}`
  - Required attributes:
    - **First Name**
    - **Last Name**
    - **Email Address**
- **Model:** **Google Gemini Chat Model**
- **Input:** comes from **Recruiter Agent** (but it references resume text directly from Set Resume via expression).
- **Output:** `{ output: { 'First Name': ..., 'Last Name': ..., 'Email Address': ... } }`
- **Connections:** Output → **Append Data**
- **Edge cases:**
  - Resumes without explicit first/last name or with nonstandard formatting.
  - Multiple emails detected; model may choose the wrong one.
  - Expression dependency on `Set Resume` item: if the workflow is modified to process multiple items, this cross-node item reference can misalign.

#### Node: **Append Data**
- **Type / role:** `n8n-nodes-base.googleSheets` — writes final results into Google Sheets.
- **Configuration (interpreted):**
  - Operation: **Append**
  - Document: **“Resume Screener”** (Spreadsheet ID `1_a0HFG...BjSs`)
  - Sheet/tab: **Sheet1** (gid=0)
  - Column mapping (explicit “define below”), writes:
    - **Date:** `{{ $now.format('yyyy-MM-dd hh:m a') }}`
    - **First Name / Last Name / Email:** from `Information Extractor` output
    - **Strengths / Weaknesses:** join arrays from `Recruiter Agent` with blank lines
    - **Risk Factor / Reward Factor:** score + explanation combined
    - **Overall Fit:** integer rating
    - **Justification:** text justification
- **Inputs:**
  - Primary input item: from **Information Extractor**
  - Also references `Recruiter Agent` output via expressions like:
    - `{{ $('Recruiter Agent').item.json.output.overall_fit_rating }}`
- **Credentials:** Google Sheets OAuth2.
- **Output:** append result metadata (row updated).
- **Edge cases / failures:**
  - Google auth expired/revoked.
  - Sheet columns renamed → mapping breaks or writes empty cells.
  - Rate limits / Google API quota.
  - `Overall Fit` column is typed as string in schema; integer is usually fine but could be coerced unexpectedly if type conversion is enabled (it is **disabled** here).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | n8n-nodes-base.formTrigger | Entry point; collect resume file + JD link | — | HTTP Request; Extract from File2 | # Smart Resume Screener — JD ↔ Resume AI Match & Sheet Logger  / How it works + links (Demo video, Sheet template, Course) |
| HTTP Request | n8n-nodes-base.httpRequest | Fetch JD page content from provided URL | On form submission | Information Extractor1 | # Data Ingestion & Content Extraction / Collect candidate inputs and convert everything into structured, usable text. |
| Extract from File2 | n8n-nodes-base.extractFromFile | Extract text from uploaded resume PDF | On form submission | Set Resume | # Data Ingestion & Content Extraction / Collect candidate inputs and convert everything into structured, usable text. |
| Set Resume | n8n-nodes-base.set | Normalize extracted resume into `resume` field | Extract from File2 | Merge | # Data Ingestion & Content Extraction / Collect candidate inputs and convert everything into structured, usable text. |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Shared Gemini LLM provider for AI nodes | — (AI connection) | Information Extractor1; Recruiter Agent; Information Extractor (AI connection) | (Same as overall workflow note area) # Smart Resume Screener — JD ↔ Resume AI Match & Sheet Logger  / How it works + links |
| Information Extractor1 | @n8n/n8n-nodes-langchain.informationExtractor | Extract clean “Job description” from fetched content | HTTP Request (+ Gemini model) | Merge | # Data Ingestion & Content Extraction / Collect candidate inputs and convert everything into structured, usable text. |
| Merge | n8n-nodes-base.merge | Combine resume item + JD item into one stream | Set Resume; Information Extractor1 | Aggregate | # AI Evaluation & Structured Screening / Evaluate candidate suitability using an AI recruiter agent. |
| Aggregate | n8n-nodes-base.aggregate | Aggregate both items into `data[]` array for agent | Merge | Recruiter Agent | # AI Evaluation & Structured Screening / Evaluate candidate suitability using an AI recruiter agent. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured JSON schema for evaluation | — (parser connection) | Recruiter Agent (as output parser) | # AI Evaluation & Structured Screening / Evaluate candidate suitability using an AI recruiter agent. |
| Recruiter Agent | @n8n/n8n-nodes-langchain.agent | Generate structured screening: strengths/weaknesses/risk/reward/score | Aggregate (+ Gemini model + output parser) | Information Extractor | # AI Evaluation & Structured Screening / Evaluate candidate suitability using an AI recruiter agent. |
| Information Extractor | @n8n/n8n-nodes-langchain.informationExtractor | Extract first/last name + email from resume | Recruiter Agent (+ Gemini model) | Append Data | # Data Structuring & Recruitment Logging / Extract candidate identity details and log final screening data. |
| Append Data | n8n-nodes-base.googleSheets | Append consolidated evaluation row into Google Sheet | Information Extractor | — | # Data Structuring & Recruitment Logging / Extract candidate identity details and log final screening data. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   **“Smart Resume Screener — JD ↔ Resume AI Match & Sheet Logger”**

2. **Add Form Trigger**
   - Node: **Form Trigger**
   - Form title: `Check Your Resume Score`
   - Add fields:
     - **Resume**: type **File**, required
     - **Job Description Link**: type **Text**, required
   - This node is the entry point.

3. **Resume branch: Extract resume text**
   - Add node: **Extract from File**
     - Operation: **PDF**
     - Binary property name: `Resume`
   - Connect: **Form Trigger → Extract from File**
   - Add node: **Set**
     - Create field `resume` (string) = `{{$json.text}}`
   - Connect: **Extract from File → Set**

4. **JD branch: Fetch and extract job description**
   - Add node: **HTTP Request**
     - URL: `{{$json['Job Description Link']}}`
     - Method: GET (default)
   - Connect: **Form Trigger → HTTP Request**
   - Add node: **Information Extractor** (LangChain)
     - Text: `{{$json.data}}`
     - Attributes to extract:
       - `Job description` (required)
   - Connect: **HTTP Request → Information Extractor (JD)**
   - Add node: **Google Gemini Chat Model**
     - Configure **Google Gemini / PaLM credentials** (API key/account).
     - Leave model/options default unless you need a specific model.
   - Connect the **Gemini Chat Model** to the JD Information Extractor via the node’s **AI Language Model** connector.

5. **Merge + aggregate inputs for the recruiter agent**
   - Add node: **Merge**
   - Connect:
     - **Set Resume → Merge (Input 0)**
     - **JD Information Extractor → Merge (Input 1)**
   - Add node: **Aggregate**
     - Mode: **Aggregate All Item Data** (`aggregateAllItemData`)
   - Connect: **Merge → Aggregate**

6. **Create structured output schema**
   - Add node: **Structured Output Parser**
     - Schema: manual
     - Define fields:
       - `candidate_strengths`: array of string
       - `candidate_weaknesses`: array of string
       - `risk_factor`: object { score: Low/Medium/High, explanation: string }
       - `reward_factor`: object { score: Low/Medium/High, explanation: string }
       - `overall_fit_rating`: integer (0–10)
       - `justification_for_rating`: string

7. **Recruiter Agent (Gemini)**
   - Add node: **AI Agent** (LangChain Agent)
   - Prompt text (user message), for example:
     - `Candidate Resume:\n{{ $json.data[0].resume }}\n`
   - System message: paste the recruiter instructions and include JD:
     - `{{ $json.data[1].output['Job description'] }}`
   - Enable structured output parsing (or connect parser as output parser).
   - Connect:
     - **Aggregate → Recruiter Agent**
     - **Gemini Chat Model → Recruiter Agent** (AI Language Model connector)
     - **Structured Output Parser → Recruiter Agent** (AI Output Parser connector)

8. **Extract candidate identity**
   - Add node: **Information Extractor** (LangChain)
     - Text: `{{ $('Set Resume').item.json.resume }}`
     - Attributes:
       - `First Name` (required)
       - `Last Name` (required)
       - `Email Address` (required)
   - Connect:
     - **Recruiter Agent → Information Extractor (Identity)**
     - **Gemini Chat Model → Information Extractor (Identity)** (AI Language Model connector)

9. **Append results to Google Sheets**
   - Add node: **Google Sheets**
     - Operation: **Append**
     - Authenticate with **Google Sheets OAuth2** credential.
     - Select Spreadsheet (create or use template) and Sheet (e.g., `Sheet1`).
     - Map columns (examples matching this workflow):
       - Date: `{{$now.format('yyyy-MM-dd hh:m a')}}`
       - First Name: `{{$json.output['First Name']}}`
       - Last Name: `{{$json.output['Last Name']}}`
       - Email: `{{$json.output['Email Address']}}`
       - Strengths: `{{ $('Recruiter Agent').item.json.output.candidate_strengths.join("\n\n") }}`
       - Weaknesses: `{{ $('Recruiter Agent').item.json.output.candidate_weaknesses.join("\n\n") }}`
       - Risk Factor: `{{ $('Recruiter Agent').item.json.output.risk_factor.score }}\n\n{{ $('Recruiter Agent').item.json.output.risk_factor.explanation }}`
       - Reward Factor: `{{ $('Recruiter Agent').item.json.output.reward_factor.score }}\n\n{{ $('Recruiter Agent').item.json.output.reward_factor.explanation }}`
       - Overall Fit: `{{ $('Recruiter Agent').item.json.output.overall_fit_rating }}`
       - Justification: `{{ $('Recruiter Agent').item.json.output.justification_for_rating }}`
   - Connect: **Information Extractor (Identity) → Google Sheets (Append)**

10. **(Optional hardening)**
   - Add validation: check that resume text and JD text are non-empty before calling the agent.
   - Add try/catch or error workflow for HTTP failures and parser errors.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Demo & Setup Video | https://drive.google.com/file/d/11YmCsQhNmDaF_O_LoTvDV4gxCbPMOGJt/view?usp=sharing |
| Sheet Template | https://docs.google.com/spreadsheets/d/1_a0HFGiv-D7_WqlmrL50CGrTW5W_8QeKW-YqbmdBjSs/edit?usp=sharing |
| Course | https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC |
| Workflow intent summary | “Smart Resume Screener ingests a candidate resume and a job description link, extracts clean text from both, runs an LLM-powered screening agent… and appends a single, validated row to a Google Sheet for tracking.” |

