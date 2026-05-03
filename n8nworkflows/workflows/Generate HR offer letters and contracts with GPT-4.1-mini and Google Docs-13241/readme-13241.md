Generate HR offer letters and contracts with GPT-4.1-mini and Google Docs

https://n8nworkflows.xyz/workflows/generate-hr-offer-letters-and-contracts-with-gpt-4-1-mini-and-google-docs-13241


# Generate HR offer letters and contracts with GPT-4.1-mini and Google Docs

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate HR offer letters and contracts with GPT-4.1-mini and Google Docs

This workflow generates either an **Offer Letter** or an **Employment Contract** based on an HR form submission. It ingests candidate-uploaded PDFs (Aadhar/Identity card and Resume), extracts text, determines the selected document type, loads the matching template, asks an AI model to **replace all placeholders without changing formatting**, and then saves the final filled document into **Google Docs**.

### Logical Blocks
**1.1 Form Submission & Pre-processing**  
Receives form data + uploaded files; splits them into separate items (Aadhar vs Resume) and computes FormDate and JoiningDate.

**1.2 PDF Text Extraction & Merge**  
Extracts text from each PDF and aggregates both outputs into a single payload.

**1.3 Template Selection & Loading**  
Checks whether “Offer Letter” or “Contract” was chosen and loads the corresponding template text into the execution context.

**1.4 AI Generation & Google Docs Save**  
Uses GPT-4.1-mini through the AI Agent node to fill placeholders in the selected template while preserving formatting, then updates a Google Doc with the final output.

---

## 2. Block-by-Block Analysis

### 2.1 Form Submission & Pre-processing

**Overview:**  
This block receives HR inputs (files + job info) and prepares two items for downstream extraction—one per uploaded PDF—while also generating standardized dates.

**Nodes Involved:**  
- Receive Candidate Details via Form  
- Split Documents and Calculate Dates

#### Node: Receive Candidate Details via Form
- **Type / Role:** `Form Trigger` (`n8n-nodes-base.formTrigger`) — workflow entry point.
- **Configuration (interpreted):**
  - Form title: **“Basic Details”**
  - Fields:
    - **Aadhar** (file, required)
    - **Resume** (file, required)
    - **Job Role** (text, required)
    - **Salary Details** (number, required)
    - **Type** (dropdown, required): `Offer Letter` or `Contract`
- **Key variables produced:**
  - `item.json` contains the field values (note: labels become keys; e.g. `"Job Role"`, `"Salary Details"`, `Type`)
  - `item.binary.Aadhar` and `item.binary.Resume` hold uploaded files
- **Connections:**
  - Output → **Split Documents and Calculate Dates**
- **Edge cases / failures:**
  - Missing required fields prevents submission.
  - Uploaded files not being PDFs may break later extraction.
  - Large files can cause memory/time limits in n8n.
- **Version notes:** `typeVersion 2.3` (Form Trigger v2+ behavior around field naming and file handling can differ from older versions).

#### Node: Split Documents and Calculate Dates
- **Type / Role:** `Code` (`n8n-nodes-base.code`) — transforms one submission into two items and adds computed dates.
- **Configuration (interpreted):**
  - JavaScript loops over incoming items (typically one).
  - Generates:
    - `FormDate`: current date in `YYYY-MM-DD` (UTC-based ISO slice)
    - `JoiningDate`: **1st day of next month** in `YYYY-MM-DD`
  - Splits binaries into separate items:
    - If `item.binary.Aadhar` exists → output item with `binary.Document = Aadhar` and `json.documentType = 'Aadhar'`
    - If `item.binary.Resume` exists → output item with `binary.Document = Resume` and `json.documentType = 'Resume'`
- **Key expressions / variables:**
  - Uses `item.json.JobRole` (but the form field is labeled **"Job Role"**; see edge case below)
- **Connections:**
  - Input: Form Trigger
  - Output → **Extract Text from Aadhar and Resume**
- **Edge cases / failures:**
  - **Field-name mismatch:** form uses `"Job Role"` but code reads `item.json.JobRole`. This will usually yield empty `jobRole`. (Downstream still uses the original `"Job Role"` elsewhere, so impact is limited, but inconsistent.)
  - If either file is missing or binary property names differ, that item won’t be created.
  - Date generation uses server time/UTC; could differ from HR locale expectations.
- **Version notes:** `typeVersion 2` (Code node behavior differs from older “Function” nodes).

---

### 2.2 PDF Text Extraction & Merge

**Overview:**  
Converts the two PDF binaries into extracted text and merges both results into one aggregated structure for later AI prompting.

**Nodes Involved:**  
- Extract Text from Aadhar and Resume  
- Merge Extracted Document Texts

#### Node: Extract Text from Aadhar and Resume
- **Type / Role:** `Extract From File` (`n8n-nodes-base.extractFromFile`) — OCR/text extraction from PDF binary.
- **Configuration (interpreted):**
  - Operation: **PDF**
  - Binary property name: **`Document`** (set by the prior Code node)
- **Connections:**
  - Input: Split Documents and Calculate Dates
  - Output → Merge Extracted Document Texts
- **Edge cases / failures:**
  - Non-PDF files or password-protected PDFs will fail extraction.
  - Scanned PDFs may yield poor extraction results if OCR is limited.
  - Very large PDFs can cause timeouts.
- **Version notes:** `typeVersion 1`.

#### Node: Merge Extracted Document Texts
- **Type / Role:** `Aggregate` (`n8n-nodes-base.aggregate`) — combines multiple items into one.
- **Configuration (interpreted):**
  - Mode: **Aggregate All Item Data**
  - Destination field name: **`text`**
  - Result: a single item containing `json.text` as an array of the aggregated items’ JSON data (including extracted text payloads).
- **Connections:**
  - Input: Extract Text from Aadhar and Resume
  - Output → Check Document Type Selection
- **Key downstream dependency:**
  - Later prompt references: `$('Merge Extracted Document Texts').item.json.text[1].text`
- **Edge cases / failures:**
  - **Ordering assumption:** referencing `[1]` assumes the resume (or the intended doc) is always in position 1. If the order flips, the candidate name source becomes wrong.
  - If only one file item exists (unexpected), `[1]` will be undefined and expressions may fail.
- **Version notes:** `typeVersion 1`.

---

### 2.3 Template Selection & Loading

**Overview:**  
Routes execution based on the chosen document type and injects the relevant template text into the item JSON for the AI node to use.

**Nodes Involved:**  
- Check Document Type Selection  
- Load Offer Letter Template  
- Load Contract Template

#### Node: Check Document Type Selection
- **Type / Role:** `IF` (`n8n-nodes-base.if`) — conditional routing.
- **Configuration (interpreted):**
  - Condition: `Type` equals **"Offer Letter"**
  - Left value expression: `={{ $('Receive Candidate Details via Form').item.json.Type }}`
  - True branch: Offer Letter path
  - False branch: Contract path
- **Connections:**
  - Input: Merge Extracted Document Texts
  - Output (true) → Load Offer Letter Template
  - Output (false) → Load Contract Template
- **Edge cases / failures:**
  - Strict equals check means any variations (e.g., “Offer letter”, extra spaces) will route to **Contract** branch.
- **Version notes:** `typeVersion 2.2` (IF node v2 condition handling differs from v1).

#### Node: Load Offer Letter Template
- **Type / Role:** `Set` (`n8n-nodes-base.set`) — adds `template_offer` field.
- **Configuration (interpreted):**
  - Sets `json.template_offer` to a static offer letter template containing placeholders like:
    - `[Candidate Name]`, `[Job Role]`, `[Department/Team Name]`, `[Company Name]`, `[Joining Date]`
- **Connections:**
  - Input: IF (true)
  - Output → Fill Template with AI
- **Edge cases / failures:**
  - Template placeholders must match what the AI is expected to replace; inconsistency can leave placeholders unreplaced.
- **Version notes:** `typeVersion 3.4`.

#### Node: Load Contract Template
- **Type / Role:** `Set` (`n8n-nodes-base.set`) — adds `template_contract` field.
- **Configuration (interpreted):**
  - Sets `json.template_contract` to a static contract template with placeholders similar to the offer letter plus additional fields:
    - `[Reporting Manager/Team Lead]`, `[Probation Period]`, `[Work Location]`, `[Salary Details]`
- **Connections:**
  - Input: IF (false)
  - Output → Fill Template with AI
- **Edge cases / failures:**
  - Same placeholder consistency risks as above.
  - Template contains escaped line breaks (`\n`) in the string; AI instructions demand preservation of line breaks—ensure the agent receives the template in a way that results in intended formatting.
- **Version notes:** `typeVersion 3.4`.

---

### 2.4 AI Generation & Google Docs Save

**Overview:**  
Uses an AI Agent backed by GPT-4.1-mini to fill in every placeholder in the selected template while preserving formatting, then writes the final text to a Google Doc.

**Nodes Involved:**  
- OpenAI GPT-4.1 Mini Model  
- Fill Template with AI  
- Save Document to Google Docs

#### Node: OpenAI GPT-4.1 Mini Model
- **Type / Role:** LangChain Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides the LLM to the agent.
- **Configuration (interpreted):**
  - Model: **gpt-4.1-mini**
  - Options: default
- **Credentials:**
  - OpenAI API credential named `testing` (ID present in JSON)
- **Connections:**
  - Output (AI language model) → Fill Template with AI (as `ai_languageModel`)
- **Edge cases / failures:**
  - Invalid API key, quota exceeded, model unavailable, network timeouts.
  - Organizational policies may block certain content; usually not relevant for HR templates but still possible.
- **Version notes:** `typeVersion 1.2`.

#### Node: Fill Template with AI
- **Type / Role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — constructs prompt + calls model to generate final document.
- **Configuration (interpreted):**
  - Prompt type: **Define**
  - **System message**: very strict rules:
    - Must not paraphrase/add/remove text
    - Must replace **all** placeholders
    - Must preserve **line breaks and blank lines exactly**
    - Must output only the final document
  - Provides both templates in the system message:
    - `Template_Offer Letter` from `$json.template_offer`
    - `Template_Contract` from `$json.template_contract`
  - Provides **Data** fields, including:
    - `Document Type` from form `Type`
    - `Candidate Name` from `$('Merge Extracted Document Texts').item.json.text[1].text`
    - `Job Role`, salary, probation, work location, reporting manager with fallbacks
    - `Company Name` fixed: “Incrementors Web Solution”
  - Additional `text` field includes references to Aadhar/Resume/form data and a computed joining date (but note it also separately passes Joining Date in the system message).
- **Key expressions / variables used (high impact):**
  - Candidate name source: `...json.text[1].text` (ordering-sensitive)
  - Department fallback: `|| "Automation Team"`
  - Probation fallback: `|| 6`
  - Work location fallback: `|| "Delhi"`
  - Reporting manager fallback: `|| "Team Lead"`
- **Connections:**
  - Input: Load Offer Letter Template OR Load Contract Template
  - AI model input: OpenAI GPT-4.1 Mini Model
  - Output → Save Document to Google Docs
- **Edge cases / failures:**
  - If aggregated `text[1]` is not the resume or not the candidate name, the filled document will be wrong.
  - If extracted text is messy, AI may not reliably isolate “Candidate Name” (you’re giving the AI a blob; not a structured name).
  - Strict “do not change anything” instructions can conflict with template escape sequences; formatting may still drift.
- **Version notes:** `typeVersion 2.2` (agent behavior and prompt assembly can differ across minor versions).

#### Node: Save Document to Google Docs
- **Type / Role:** `Google Docs` (`n8n-nodes-base.googleDocs`) — writes final output to a Google Doc.
- **Configuration (interpreted):**
  - Operation: **Update**
  - Action: **Insert text** with `={{ $json.output }}`
  - Document URL: `doc_url` (placeholder; must be replaced)
- **Credentials:**
  - Google Docs OAuth2 credential: “Aksh:- Google Docs account”
- **Connections:**
  - Input: Fill Template with AI
- **Edge cases / failures:**
  - Invalid/expired OAuth token, insufficient permissions, doc not shared with credential account.
  - `doc_url` not replaced with a real URL → runtime failure.
  - Large inserts may hit Google Docs API limitations.
- **Version notes:** `typeVersion 2`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Candidate Details via Form | Form Trigger | Entry point; collect HR inputs and files | — | Split Documents and Calculate Dates | ## Form Submission & Document Processing  Receives candidate details via form, splits uploaded Indentity card and Resume files, then extracts text from both PDFs and merges the data. |
| Split Documents and Calculate Dates | Code | Split Aadhar/Resume into separate items; compute FormDate/JoiningDate | Receive Candidate Details via Form | Extract Text from Aadhar and Resume | ## Form Submission & Document Processing  Receives candidate details via form, splits uploaded Indentity card and Resume files, then extracts text from both PDFs and merges the data. |
| Extract Text from Aadhar and Resume | Extract From File | Extract text from each PDF binary | Split Documents and Calculate Dates | Merge Extracted Document Texts | ## Form Submission & Document Processing  Receives candidate details via form, splits uploaded Indentity card and Resume files, then extracts text from both PDFs and merges the data. |
| Merge Extracted Document Texts | Aggregate | Combine extracted outputs into one payload | Extract Text from Aadhar and Resume | Check Document Type Selection | ## Form Submission & Document Processing  Receives candidate details via form, splits uploaded Indentity card and Resume files, then extracts text from both PDFs and merges the data. |
| Check Document Type Selection | IF | Route to offer vs contract template | Merge Extracted Document Texts | Load Offer Letter Template; Load Contract Template | ## Template Selection & Loading  Checks if the user selected "Offer Letter" or  "Contract", then loads the appropriate template  with all placeholders ready for AI filling. |
| Load Offer Letter Template | Set | Provide offer letter template string | Check Document Type Selection (true) | Fill Template with AI | ## Template Selection & Loading  Checks if the user selected "Offer Letter" or  "Contract", then loads the appropriate template  with all placeholders ready for AI filling. |
| Load Contract Template | Set | Provide contract template string | Check Document Type Selection (false) | Fill Template with AI | ## Template Selection & Loading  Checks if the user selected "Offer Letter" or  "Contract", then loads the appropriate template  with all placeholders ready for AI filling. |
| OpenAI GPT-4.1 Mini Model | LangChain Chat Model (OpenAI) | LLM provider for the agent | — | Fill Template with AI | ## AI Generation & Google Docs Save  AI fills all template placeholders with actual  candidate data while preserving formatting, then  saves the final document to Google Docs. |
| Fill Template with AI | LangChain Agent | Fill placeholders using selected template + extracted data | Load Offer Letter Template / Load Contract Template | Save Document to Google Docs | ## AI Generation & Google Docs Save  AI fills all template placeholders with actual  candidate data while preserving formatting, then  saves the final document to Google Docs. |
| Save Document to Google Docs | Google Docs | Insert generated output into a Google Doc | Fill Template with AI | — | ## AI Generation & Google Docs Save  AI fills all template placeholders with actual  candidate data while preserving formatting, then  saves the final document to Google Docs. |
| Sticky Note | Sticky Note | Documentation / canvas annotation | — | — | ## AI-Powered HR Document Generator  This workflow automatically generates professional Offer Letters or Employment Contracts for new hires. HR submits a form with the candidate's documents (Identity card & Resume), job details, and document type. The system extracts the candidate's name from uploaded files, auto-calculates the joining date, selects the appropriate template, and uses AI to intelligently fill all placeholders while preserving formatting. The final document is saved directly to Google Docs — ready to send.  ## How it works  1. HR fills a form with Identity card, Resume, Job Role, Salary, and Type.  2. System splits uploaded files and auto-generates joining date.  3. Extracts candidate name and details from PDF documents.  4. Routes to Offer Letter or Contract template based on selection.  5. AI fills all placeholders with actual data while keeping formatting.  6. Final document is saved to Google Docs automatically.  ## Setup steps  1. Connect Google Docs OAuth credentials.  2. Add OpenAI API key for GPT-4.1-mini model.  3. Update Google Docs document URL in the save node.  4. Customize Offer Letter and Contract templates as needed.  5. Share the form link with your HR team to start generating documents. |
| Sticky Note1 | Sticky Note | Documentation / canvas annotation | — | — | ## Form Submission & Document Processing  Receives candidate details via form, splits uploaded Indentity card and Resume files, then extracts text from both PDFs and merges the data. |
| Sticky Note2 | Sticky Note | Documentation / canvas annotation | — | — | ## Template Selection & Loading  Checks if the user selected "Offer Letter" or  "Contract", then loads the appropriate template  with all placeholders ready for AI filling. |
| Sticky Note3 | Sticky Note | Documentation / canvas annotation | — | — | ## AI Generation & Google Docs Save  AI fills all template placeholders with actual  candidate data while preserving formatting, then  saves the final document to Google Docs. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Receive Candidate Details via Form” (Form Trigger)**
   - Node: **Form Trigger**
   - Form title: `Basic Details`
   - Add fields:
     - File: `Aadhar` (required)
     - File: `Resume` (required)
     - Text: `Job Role` (required)
     - Number: `Salary Details` (required)
     - Dropdown: `Type` (required) options: `Offer Letter`, `Contract`

2) **Create “Split Documents and Calculate Dates” (Code)**
   - Node: **Code**
   - Paste logic that:
     - Creates `FormDate` = today (`YYYY-MM-DD`)
     - Creates `JoiningDate` = first day of next month (`YYYY-MM-DD`)
     - Outputs two items:
       - `binary.Document = binary.Aadhar`, `json.documentType = 'Aadhar'`
       - `binary.Document = binary.Resume`, `json.documentType = 'Resume'`
   - Connect: Form Trigger → Code

3) **Create “Extract Text from Aadhar and Resume”**
   - Node: **Extract From File**
   - Operation: `PDF`
   - Binary Property Name: `Document`
   - Connect: Code → Extract From File

4) **Create “Merge Extracted Document Texts”**
   - Node: **Aggregate**
   - Aggregate mode: `Aggregate All Item Data`
   - Destination field: `text`
   - Connect: Extract From File → Aggregate

5) **Create “Check Document Type Selection”**
   - Node: **IF**
   - Condition: String equals
     - Left: Expression pointing to form value `Type` (from the form trigger)
     - Right: `Offer Letter`
   - Connect: Aggregate → IF

6) **Create “Load Offer Letter Template”**
   - Node: **Set**
   - Add string field: `template_offer` = your offer template text with placeholders
   - Connect: IF (true) → Load Offer Letter Template

7) **Create “Load Contract Template”**
   - Node: **Set**
   - Add string field: `template_contract` = your contract template text with placeholders
   - Connect: IF (false) → Load Contract Template

8) **Create “OpenAI GPT-4.1 Mini Model”**
   - Node: **OpenAI Chat Model** (LangChain)
   - Model: `gpt-4.1-mini`
   - Credentials: create/select **OpenAI API** credential (API key)
   - No “main” connection required; it connects as an AI model input to the agent.

9) **Create “Fill Template with AI”**
   - Node: **AI Agent** (LangChain Agent)
   - Prompt type: `Define`
   - System message:
     - Include strict instructions to replace placeholders only and preserve formatting
     - Include both templates via expressions (from `template_offer` / `template_contract`)
     - Include Data fields from the form + extracted text + calculated dates
   - Connect:
     - Load Offer Letter Template → Fill Template with AI
     - Load Contract Template → Fill Template with AI
     - OpenAI Chat Model → Fill Template with AI (AI Language Model connection)

10) **Create “Save Document to Google Docs”**
   - Node: **Google Docs**
   - Operation: `Update`
   - Document URL: set to your real Google Doc URL (replace placeholder `doc_url`)
   - Action: Insert text with expression `{{$json.output}}`
   - Credentials: configure **Google Docs OAuth2** (authorize the Google account that owns/has access to the doc)
   - Connect: Fill Template with AI → Save Document to Google Docs

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Connect Google Docs OAuth credentials. | Setup requirement from workflow notes |
| Add OpenAI API key for GPT-4.1-mini model. | Setup requirement from workflow notes |
| Update Google Docs document URL in the save node. | Replace `doc_url` with the real document URL |
| Customize Offer Letter and Contract templates as needed. | Templates are stored in Set nodes |
| Company website included in templates. | https://www.incrementors.com/ |

