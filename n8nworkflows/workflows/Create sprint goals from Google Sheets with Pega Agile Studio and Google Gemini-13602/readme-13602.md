Create sprint goals from Google Sheets with Pega Agile Studio and Google Gemini

https://n8nworkflows.xyz/workflows/create-sprint-goals-from-google-sheets-with-pega-agile-studio-and-google-gemini-13602


# Create sprint goals from Google Sheets with Pega Agile Studio and Google Gemini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Create sprint goals from Google Sheets with Pega Agile Studio and Google Gemini  
**Workflow name (in n8n):** Create sprint goals using Pega Agile Studio with AI

**Purpose:**  
Generate one or two value-driven Sprint Goals (as a polished HTML email) from a list of Pega Agile Studio user story IDs stored in a Google Sheet. The workflow enriches each user story by retrieving details and parsing any linked Google attachments (Docs/Sheets/Slides and XLSX), then asks Google Gemini (via an AI Agent node) to draft a consistent “newsletter-like” HTML email and sends it via Gmail.

**Primary use case:** Product Owners / Scrum Masters using **Pega Agile Studio** who want a quick sprint-goal summary from a curated list of user stories and their supporting documents.

### Logical blocks
**1.1 Manual start + Google Sheet ingestion**  
Manual trigger → read “Userstory” column from Google Sheets → aggregate list.

**1.2 AI-driven sprint goal generation (agent + tools)**  
AI Agent receives aggregated user story IDs, calls tools (HTTP tool to retrieve user story + sub-workflow tool to fetch/parse attachments), then produces HTML.

**1.3 Email delivery**  
Send the generated HTML via Gmail.

**1.4 Sub-workflow: attachment retrieval and parsing (executed by another workflow)**  
Triggered as a sub-workflow with input `Userstory` → calls Pega Agile Studio API → iterates attachments → routes by type (Docs/Sheets/Slides; Sheets further split native vs XLSX) → fetches content with Google APIs / Drive download + XLSX extraction → merges results.

> Note: Block 1.4 is *implemented inside this JSON as nodes* starting at **“When Executed by Another Workflow”**, meaning this workflow file contains sub-workflow logic, but the main AI Agent also calls an *external* workflow tool named **“Get Attachments - Agile Studio US”**.

---

## 2. Block-by-Block Analysis

### 2.1 Manual start + Google Sheet ingestion

**Overview:**  
Starts on demand, loads user story identifiers from a Google Sheet, and aggregates them so the AI Agent can process them as a set.

**Nodes involved:**  
- When clicking ‘Execute workflow’  
- Retrieve_Data_From_Sheet  
- Aggregate

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) – interactive entry point.
- **Config choices:** No parameters.
- **Inputs/Outputs:**  
  - Output → `Retrieve_Data_From_Sheet`
- **Edge cases:** None (only manual execution).

#### Node: Retrieve_Data_From_Sheet
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) – fetch rows from a spreadsheet.
- **Config choices (interpreted):**
  - Uses **Document ID** (spreadsheet ID) and **Sheet name** selected via resource locator fields.
  - Operation is not explicitly shown in parameters (defaults apply); in typical n8n usage this node is configured to “Read” rows.
- **Key variables/fields:**
  - Expects a column named **`Userstory`** in the returned rows.
- **Inputs/Outputs:**  
  - Input ← Manual Trigger  
  - Output → `Aggregate`
- **Credentials:** Google Sheets OAuth2 (not shown but required).
- **Edge cases / failures:**
  - Missing/incorrect Spreadsheet ID or Sheet name.
  - OAuth scope issues (Sheets read access).
  - Sheet has no `Userstory` column → downstream aggregation/AI prompt becomes incomplete.

#### Node: Aggregate
- **Type / role:** Aggregate (`n8n-nodes-base.aggregate`) – collects multiple input items into aggregated fields.
- **Config choices:**
  - Aggregates field **`Userstory`** across all rows.
- **Inputs/Outputs:**  
  - Input ← `Retrieve_Data_From_Sheet`  
  - Output → `Sprint Goal Generator`
- **Edge cases:**
  - Empty sheet results → AI Agent receives no usable user stories.
  - If some rows have null `Userstory`, they may be included as empty values depending on n8n aggregation behavior.

---

### 2.2 AI-driven sprint goal generation (agent + tools)

**Overview:**  
An AI Agent (Google Gemini model) is prompted to generate sprint goals, and can call tools: (1) an HTTP request tool to fetch a single user story from Pega Agile Studio, and (2) a workflow tool to fetch attachments/content.

**Nodes involved:**  
- Sprint Goal Generator  
- Google Gemini Chat Model  
- Retrieve Agile Studio US (AI tool)  
- Call 'Get Attachments - Agile Studio US' (AI tool/workflow)

#### Node: Google Gemini Chat Model
- **Type / role:** Google Gemini chat model connector (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) – provides the LLM for the agent.
- **Config choices:** Options object empty (defaults).
- **Inputs/Outputs:**  
  - Connected via **ai_languageModel** → `Sprint Goal Generator`
- **Credentials:** Google Gemini / Google AI credentials required (not shown).
- **Edge cases:**
  - Model auth errors, quota limits, safety blocks (depending on content).
  - Large context: attachment text may exceed token limits.

#### Node: Sprint Goal Generator
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) – orchestrates LLM + tool calling and produces final HTML.
- **Config choices:**
  - **Prompt (user message)** references the aggregated `Userstory` field:
    - `For the {{ $json.Userstory }} ...`
  - Explicit tool instructions:
    - Use **Retrieve Agile Studio US** tool **one US at a time** (e.g., US-1234)
    - Call **Get Attachments - Agile Studio US**
  - **System message** (high impact constraints):
    - Role: Scrum Master/Product Owner
    - Output: 1–2 overarching Sprint Goals
    - “Beautify HTML using inline CSS”, “modern newsletter”, “professional HTML email report”
    - Output must be **raw HTML only**, starting with `<html>` and ending with `</html>`
    - No markdown fences, no preamble text
- **Inputs/Outputs:**
  - Input ← `Aggregate`
  - Output → `Send a message`
  - AI language model input ← `Google Gemini Chat Model`
  - Tool connections (ai_tool) ← `Retrieve Agile Studio US`, `Call 'Get Attachments - Agile Studio US'`
- **Edge cases / failures:**
  - Agent may not correctly iterate user stories unless the aggregated field format is clear (array vs joined string). If Aggregate outputs an array-like structure, the prompt may need explicit formatting guidance.
  - Tool-call failures (HTTP 401/404/500) can degrade output quality.
  - HTML constraints are strict; some models may occasionally prepend text—consider adding post-validation.

#### Node: Retrieve Agile Studio US
- **Type / role:** HTTP Request Tool (`n8n-nodes-base.httpRequestTool`) – callable by the agent.
- **Config choices:**
  - URL template:
    - `https://<yoururl>/userstories/{{ $fromAI('Userstory', ``, 'string') }}`
  - Authentication:
    - Generic credential type → OAuth2 API
- **Inputs/Outputs:**
  - Tool output returns HTTP response to the agent.
- **Key expressions/variables:**
  - `$fromAI('Userstory', '', 'string')` – the agent supplies `Userstory` dynamically.
- **Edge cases:**
  - If agent passes malformed ID (e.g., multiple IDs at once), endpoint may 404.
  - OAuth2 token expiration / wrong scopes.
  - Response size may be large; could hit n8n memory limits depending on story payload.

#### Node: Call 'Get Attachments - Agile Studio US'
- **Type / role:** Tool Workflow (`@n8n/n8n-nodes-langchain.toolWorkflow`) – agent calls another workflow as a tool.
- **Config choices:**
  - `workflowId`: `mQ4yQw9JQw2eCmGt` (must exist and be active/published)
  - Workflow input mapping:
    - `Userstory = {{ $fromAI('Userstory', '', 'string') }}`
- **Inputs/Outputs:**
  - Tool output becomes available to the agent for summarization.
- **Edge cases:**
  - Sub-workflow not published/active → tool call fails.
  - Mismatched expected input name/type → tool call fails.
  - Attachment parsing can produce large outputs; may exceed LLM context.

---

### 2.3 Email delivery

**Overview:**  
Sends the AI-generated HTML as an email.

**Nodes involved:**  
- Send a message

#### Node: Send a message
- **Type / role:** Gmail (`n8n-nodes-base.gmail`) – sends an email.
- **Config choices:**
  - Subject: `Sprint Goals`
  - Message body: `={{ $json.output }}`
    - Assumes the agent output is in `$json.output` (typical for some AI nodes, but you should verify actual output field naming in your n8n version).
- **Inputs/Outputs:**
  - Input ← `Sprint Goal Generator`
  - No downstream node.
- **Credentials:** Gmail OAuth2 required.
- **Edge cases:**
  - If the agent outputs HTML under a different property (e.g., `text`, `response`, etc.), the email body will be blank.
  - Gmail API may sanitize or alter HTML; inline CSS is generally OK but not guaranteed.
  - Missing “To” recipient configuration (not shown in parameters) could cause send failure depending on node defaults.

---

### 2.4 Sub-workflow: retrieve user story + parse attachments (executed by another workflow)

**Overview:**  
This block is structured like a sub-workflow entry point. Given a `Userstory` ID, it retrieves user story details from Pega Agile Studio, iterates through attachments, fetches content from Google APIs (Docs/Sheets/Slides), handles XLSX via Drive download + extraction, converts content into readable text, and merges results.

**Nodes involved:**  
- When Executed by Another Workflow  
- Get the Userstory data  
- Iterate over the attachments  
- Only select specific spreadsheets  
- Switch  
- Keep only the attachment ID of the Sheet  
- Perform the API request with the ID of sheet  
- Convert to sheet data to readable format  
- Keep only the attachment ID of .xlsx  
- Download file  
- Extract from File  
- Convert the xlsx data to more readable format  
- Merge the results  
- Only select specific document  
- Keep only the attachment ID of document  
- Perform the API request with the ID of document  
- Convert the document to more readable format  
- Only select specific presentation  
- Keep only the attachment ID of presentation  
- Perform the API request with the ID of presentation  
- Convert the presentation to more readable format  
- Merge

#### Node: When Executed by Another Workflow
- **Type / role:** Execute Workflow Trigger (`n8n-nodes-base.executeWorkflowTrigger`) – entry point for being called by another workflow.
- **Config choices:**
  - Defines an input parameter: `Userstory`
- **Inputs/Outputs:**
  - Output → `Get the Userstory data`
- **Edge cases:**
  - Caller does not pass `Userstory` → Pega request fails.

#### Node: Get the Userstory data
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) – fetches Pega Agile Studio user story JSON.
- **Config choices:**
  - URL: `https://<yoururl>/userstories/{{ $json.Userstory }}`
  - Auth: Generic OAuth2 API.
- **Inputs/Outputs:**
  - Input ← `When Executed by Another Workflow`
  - Output → `Iterate over the attachments`
- **Edge cases:**
  - 401/403 OAuth errors.
  - Story not found (404).
  - Expected structure `details.attachmentList` missing → split node yields nothing.

#### Node: Iterate over the attachments
- **Type / role:** Split Out (`n8n-nodes-base.splitOut`) – iterates over `details.attachmentList`.
- **Config choices:**
  - Field to split: `details.attachmentList`
- **Inputs/Outputs:**
  - Input ← `Get the Userstory data`
  - Outputs (parallel) → `Only select specific document`, `Only select specific spreadsheets`, `Only select specific presentation`
- **Edge cases:**
  - `details.attachmentList` not an array → runtime error or empty output depending on n8n behavior.
  - Attachments without `url` field break downstream regex extraction.

#### Node: Only select specific spreadsheets
- **Type / role:** Filter (`n8n-nodes-base.filter`) – keep only attachments whose URL contains `spreadsheets`.
- **Config choices:**
  - Condition: `$json.url contains "spreadsheets"`
- **Inputs/Outputs:**
  - Input ← `Iterate over the attachments`
  - Output → `Switch`
- **Edge cases:**
  - Google Drive “share links” not containing the expected substring; attachment skipped.

#### Node: Switch
- **Type / role:** Switch (`n8n-nodes-base.switch`) – distinguishes native Google Sheets vs XLSX based on URL query.
- **Config choices:**
  - Rule 1: URL **notContains** `rtpof=true` → treat as native Google Sheet
  - Rule 2: URL **contains** `rtpof=true` → treat as XLSX (or an Office file link style)
- **Inputs/Outputs:**
  - Input ← `Only select specific spreadsheets`
  - Output 0 → `Keep only the attachment ID of the Sheet`
  - Output 1 → `Keep only the attachment ID of .xlsx`
- **Edge cases:**
  - The heuristic `rtpof=true` is not universal; some XLSX links may not include it.
  - If both rules fail (unlikely here), item may be dropped.

#### Node: Keep only the attachment ID of the Sheet
- **Type / role:** Set (`n8n-nodes-base.set`) – extracts Google file ID from URL.
- **Config choices:**
  - Sets `url` to regex group: `{{ $json.url.match(/\/d\/([a-zA-Z0-9-_]+)/)[1] }}`
- **Inputs/Outputs:**
  - Input ← `Switch` (native sheets branch)
  - Output → `Perform the API request with the ID of sheet`
- **Edge cases:**
  - URL not matching `/d/<id>` pattern → `match(...)` returns null → expression error.

#### Node: Perform the API request with the ID of sheet
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) – calls Google Sheets API directly.
- **Config choices:**
  - URL: `https://sheets.googleapis.com/v4/spreadsheets/{{ $json.url }}`
  - Auth: predefined credential type `googleSheetsOAuth2Api`
  - Query parameter `fields` to reduce payload:
    - `properties(title),sheets(properties(title),data(rowData(values(effectiveValue,formattedValue))))`
  - Node note: response is slimmed to avoid being too large.
- **Inputs/Outputs:**
  - Input ← `Keep only the attachment ID of the Sheet`
  - Output → `Convert to sheet data to readable format`
- **Edge cases:**
  - Missing Sheets API scopes.
  - Very large sheets may still exceed payload limits even with fields filtering.

#### Node: Convert to sheet data to readable format
- **Type / role:** Code (`n8n-nodes-base.code`) – converts Sheets API structure into readable markdown-like text.
- **Config choices:**
  - Iterates over `item.json.sheets[]`, builds:
    - `spreadsheetTitle`
    - `cleanText` containing rows rendered as `| a | b |`
- **Inputs/Outputs:**
  - Input ← `Perform the API request with the ID of sheet`
  - Output → `Merge the results`
- **Edge cases:**
  - If `sheet.data[0].rowData` missing (empty sheet or different response), output may be sparse.
  - This can generate large text; may blow up prompt size later.

#### Node: Keep only the attachment ID of .xlsx
- **Type / role:** Set – extracts file ID for XLSX branch.
- **Config choices:** Same regex extraction into `url`.
- **Inputs/Outputs:**
  - Input ← `Switch` (xlsx branch)
  - Output → `Download file`
- **Edge cases:** Same regex failure risks.

#### Node: Download file
- **Type / role:** Google Drive (`n8n-nodes-base.googleDrive`) – downloads binary file content.
- **Config choices:**
  - Operation: download
  - File ID: `={{ $json.url }}`
- **Inputs/Outputs:**
  - Input ← `Keep only the attachment ID of .xlsx`
  - Output → `Extract from File` (binary)
- **Credentials:** Google Drive OAuth2.
- **Edge cases:**
  - No permission to download file.
  - Large file download timeouts.

#### Node: Extract from File
- **Type / role:** Extract From File (`n8n-nodes-base.extractFromFile`) – parses XLSX into JSON rows.
- **Config choices:**
  - Operation: `xlsx`
- **Inputs/Outputs:**
  - Input ← `Download file`
  - Output → `Convert the xlsx data to more readable format`
- **Edge cases:**
  - Corrupt XLSX, password-protected files.
  - Very large spreadsheets → memory usage.

#### Node: Convert the xlsx data to more readable format
- **Type / role:** Code – renders extracted XLSX rows to markdown-like table text.
- **Config choices:**
  - Uses headers from first item keys, prints rows as `| ... |`
  - Outputs:
    - `spreadsheetTitle: "XLSX Attachment"`
    - `cleanText`
- **Inputs/Outputs:**
  - Input ← `Extract from File`
  - Output → `Merge the results`
- **Edge cases:**
  - If extraction returns no items, output becomes only header line or minimal.

#### Node: Merge the results
- **Type / role:** Merge (`n8n-nodes-base.merge`) – merges sheet-branch results (native or xlsx) into one stream.
- **Config choices:** Default merge behavior (not explicitly configured).
- **Inputs/Outputs:**
  - Inputs ← `Convert to sheet data to readable format` and `Convert the xlsx data to more readable format`
  - Output → `Merge` (final 3-way merge)
- **Edge cases:**
  - Merge mode defaults can surprise (append vs merge by index). Here it appears intended as “pass through both into a single list”.

#### Node: Only select specific document
- **Type / role:** Filter – keep only URLs containing `document`.
- **Inputs/Outputs:**  
  - Input ← `Iterate over the attachments`  
  - Output → `Keep only the attachment ID of document`
- **Edge cases:** URL patterns differ → missed docs.

#### Node: Keep only the attachment ID of document
- **Type / role:** Set – regex extracts Google Doc ID from URL into `url`.
- **Inputs/Outputs:**  
  - Output → `Perform the API request with the ID of document`
- **Edge cases:** Regex mismatch.

#### Node: Perform the API request with the ID of document
- **Type / role:** HTTP Request – Google Docs API call with field filtering.
- **Config choices:**
  - URL: `https://docs.googleapis.com/v1/documents/{{ $json.url }}`
  - Auth: predefined `googleDocsOAuth2Api`
  - Query `fields` limits payload to title + textual content + tables.
  - Node note: slim response to avoid oversize.
- **Inputs/Outputs:**  
  - Output → `Convert the document to more readable format`
- **Edge cases:**
  - Docs API permissions/scope.
  - Some content types (drawings, embedded objects) are ignored by the chosen fields.

#### Node: Convert the document to more readable format
- **Type / role:** Code – converts Docs API structure to plain text with rudimentary table formatting.
- **Config choices:**
  - Traverses paragraphs and tables recursively.
  - Output fields: `title`, `documentId`, `text`
- **Inputs/Outputs:**  
  - Output → `Merge` (final merge input index 1)
- **Edge cases:**
  - Docs API response doesn’t always include `documentId` at top-level depending on API; might be missing.
  - Text may include many newlines; can inflate prompt.

#### Node: Only select specific presentation
- **Type / role:** Filter – keep only URLs containing `presentation`.
- **Inputs/Outputs:**  
  - Output → `Keep only the attachment ID of presentation`

#### Node: Keep only the attachment ID of presentation
- **Type / role:** Set – regex extracts Google Slides presentation ID into `url`.
- **Inputs/Outputs:**  
  - Output → `Perform the API request with the ID of presentation`

#### Node: Perform the API request with the ID of presentation
- **Type / role:** HTTP Request – Google Slides API call with field filtering.
- **Config choices:**
  - URL: `https://slides.googleapis.com/v1/presentations/{{ $json.url }}`
  - Auth: predefined `googleSlidesOAuth2Api`
  - Query `fields`: `slides(pageElements(shape(text(textElements(textRun(content))))))`
  - Note: slim response for size.
- **Inputs/Outputs:**  
  - Output → `Convert the presentation to more readable format`
- **Edge cases:**
  - Slides with speaker notes or non-shape text won’t be captured by this fields selection.

#### Node: Convert the presentation to more readable format
- **Type / role:** Code – extracts text runs per slide and returns `formattedContent`.
- **Inputs/Outputs:**  
  - Output → `Merge` (final merge input index 2)

#### Node: Merge
- **Type / role:** Merge (`n8n-nodes-base.merge`) – combines document, sheet, presentation text streams into one.
- **Config choices:**
  - `numberInputs: 3` (explicitly expects three inputs)
- **Inputs/Outputs:**
  - Input 0 ← `Merge the results` (sheets/xlsx)
  - Input 1 ← `Convert the document to more readable format`
  - Input 2 ← `Convert the presentation to more readable format`
  - Output: not connected further in this workflow JSON (likely intended as final sub-workflow output).
- **Edge cases:**
  - If one branch yields no items, merge behavior depends on mode; may stall waiting for all inputs depending on merge configuration defaults.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / setup guidance |  |  | ## Create sprint goals using Pega Agile Studio with AI; Based on the Google Sheet data...; Requirements include Pega OAuth2, AI API, Google Cloud access |
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point |  | Retrieve_Data_From_Sheet | ## Create sprint goals using Pega Agile Studio with AI; Based on the Google Sheet data...; Requirements include Pega OAuth2, AI API, Google Cloud access |
| Retrieve_Data_From_Sheet | Google Sheets | Read user story IDs from Sheet | When clicking ‘Execute workflow’ | Aggregate | ## Create sprint goals using Pega Agile Studio with AI; Based on the Google Sheet data...; Requirements include Pega OAuth2, AI API, Google Cloud access |
| Aggregate | Aggregate | Collect Userstory values | Retrieve_Data_From_Sheet | Sprint Goal Generator | ## Create sprint goals using Pega Agile Studio with AI; Based on the Google Sheet data...; Requirements include Pega OAuth2, AI API, Google Cloud access |
| Google Gemini Chat Model | LangChain Google Gemini Chat Model | LLM provider for agent |  | Sprint Goal Generator (ai_languageModel) |  |
| Retrieve Agile Studio US | HTTP Request Tool | AI tool: fetch single user story from Pega |  | Sprint Goal Generator (ai_tool) |  |
| Call 'Get Attachments - Agile Studio US' | Tool Workflow | AI tool: call sub-workflow to retrieve attachments |  | Sprint Goal Generator (ai_tool) |  |
| Sprint Goal Generator | LangChain Agent | Orchestrate tools + generate HTML sprint goals | Aggregate | Send a message |  |
| Send a message | Gmail | Email the generated HTML | Sprint Goal Generator |  |  |
| Sticky Note3 | Sticky Note | Documentation: sub-workflow attachment retrieval |  |  | ## Subworkflow: Retrieving Google Sheet/Docs/Slides attachments; Google API cannot process .xlsx so different path |
| When Executed by Another Workflow | Execute Workflow Trigger | Sub-workflow entry point (input: Userstory) |  | Get the Userstory data | ## Subworkflow: Retrieving Google Sheet/Docs/Slides attachments; Google API cannot process .xlsx so different path |
| Get the Userstory data | HTTP Request | Fetch Pega user story JSON | When Executed by Another Workflow | Iterate over the attachments | ## Subworkflow: Retrieving Google Sheet/Docs/Slides attachments; Google API cannot process .xlsx so different path |
| Iterate over the attachments | Split Out | Iterate over details.attachmentList | Get the Userstory data | Only select specific document; Only select specific spreadsheets; Only select specific presentation | ## Subworkflow: Retrieving Google Sheet/Docs/Slides attachments; Google API cannot process .xlsx so different path |
| Only select specific spreadsheets | Filter | Keep sheet attachments | Iterate over the attachments | Switch | ## Subworkflow: Retrieving Google Sheet/Docs/Slides attachments; Google API cannot process .xlsx so different path |
| Sticky Note2 | Sticky Note | Documentation: native Sheets vs XLSX handling |  |  | ## Native Sheets or .xlsx; Native: API call; XLSX: download/extract then return values |
| Switch | Switch | Route native Sheets vs XLSX | Only select specific spreadsheets | Keep only the attachment ID of the Sheet; Keep only the attachment ID of .xlsx | ## Native Sheets or .xlsx; Native: API call; XLSX: download/extract then return values |
| Keep only the attachment ID of the Sheet | Set | Extract Google Sheet file ID | Switch | Perform the API request with the ID of sheet | ## Native Sheets or .xlsx; Native: API call; XLSX: download/extract then return values |
| Perform the API request with the ID of sheet | HTTP Request | Google Sheets API fetch (filtered fields) | Keep only the attachment ID of the Sheet | Convert to sheet data to readable format | ## Native Sheets or .xlsx; Native: API call; XLSX: download/extract then return values |
| Convert to sheet data to readable format | Code | Convert Sheets API JSON to readable text | Perform the API request with the ID of sheet | Merge the results | ## Native Sheets or .xlsx; Native: API call; XLSX: download/extract then return values |
| Keep only the attachment ID of .xlsx | Set | Extract file ID for XLSX | Switch | Download file | ## Native Sheets or .xlsx; Native: API call; XLSX: download/extract then return values |
| Download file | Google Drive | Download XLSX binary | Keep only the attachment ID of .xlsx | Extract from File | ## Native Sheets or .xlsx; Native: API call; XLSX: download/extract then return values |
| Extract from File | Extract From File | Parse XLSX to JSON | Download file | Convert the xlsx data to more readable format | ## Native Sheets or .xlsx; Native: API call; XLSX: download/extract then return values |
| Convert the xlsx data to more readable format | Code | Render XLSX JSON as readable table text | Extract from File | Merge the results | ## Native Sheets or .xlsx; Native: API call; XLSX: download/extract then return values |
| Merge the results | Merge | Combine native sheet and XLSX results | Convert to sheet data to readable format; Convert the xlsx data to more readable format | Merge | ## Native Sheets or .xlsx; Native: API call; XLSX: download/extract then return values |
| Only select specific document | Filter | Keep doc attachments | Iterate over the attachments | Keep only the attachment ID of document | ## Subworkflow: Retrieving Google Sheet/Docs/Slides attachments; Google API cannot process .xlsx so different path |
| Keep only the attachment ID of document | Set | Extract Doc ID | Only select specific document | Perform the API request with the ID of document | ## Subworkflow: Retrieving Google Sheet/Docs/Slides attachments; Google API cannot process .xlsx so different path |
| Perform the API request with the ID of document | HTTP Request | Google Docs API fetch (filtered fields) | Keep only the attachment ID of document | Convert the document to more readable format | Also slim down the response so only text will be returned, otherwise the response will be to large to handle. |
| Convert the document to more readable format | Code | Convert Docs API JSON to plain text | Perform the API request with the ID of document | Merge | ## Subworkflow: Retrieving Google Sheet/Docs/Slides attachments; Google API cannot process .xlsx so different path |
| Only select specific presentation | Filter | Keep slide attachments | Iterate over the attachments | Keep only the attachment ID of presentation | ## Subworkflow: Retrieving Google Sheet/Docs/Slides attachments; Google API cannot process .xlsx so different path |
| Keep only the attachment ID of presentation | Set | Extract Slides ID | Only select specific presentation | Perform the API request with the ID of presentation | ## Subworkflow: Retrieving Google Sheet/Docs/Slides attachments; Google API cannot process .xlsx so different path |
| Perform the API request with the ID of presentation | HTTP Request | Google Slides API fetch (filtered fields) | Keep only the attachment ID of presentation | Convert the presentation to more readable format | Also slim down the response so only text will be returned, otherwise the response will be to large to handle. |
| Convert the presentation to more readable format | Code | Extract slide text per slide | Perform the API request with the ID of presentation | Merge | ## Subworkflow: Retrieving Google Sheet/Docs/Slides attachments; Google API cannot process .xlsx so different path |

---

## 4. Reproducing the Workflow from Scratch

### A) Main workflow (manual → read sheet → AI → email)

1) **Create a new workflow** named:  
   - “Create sprint goals using Pega Agile Studio with AI”

2) **Add Manual Trigger**  
   - Node: **Manual Trigger**  
   - Name: “When clicking ‘Execute workflow’”

3) **Add Google Sheets node**  
   - Node: **Google Sheets**  
   - Name: “Retrieve_Data_From_Sheet”  
   - Configure:
     - Select your **Spreadsheet (Document ID)**  
     - Select **Sheet name**  
     - Set operation to read rows (e.g., “Read” / “Get Many”, depending on your n8n UI)  
   - Ensure your sheet has a column header exactly: **Userstory** (values like `US-1234`).

4) **Connect** Manual Trigger → Retrieve_Data_From_Sheet

5) **Add Aggregate node**  
   - Node: **Aggregate**  
   - Name: “Aggregate”  
   - Configure:
     - Fields to aggregate: **Userstory**
   - Connect: Retrieve_Data_From_Sheet → Aggregate

6) **Add Google Gemini Chat Model node**  
   - Node: **Google Gemini Chat Model** (LangChain)  
   - Name: “Google Gemini Chat Model”  
   - Configure credentials for Gemini / Google AI in n8n.

7) **Add AI Agent node**  
   - Node: **AI Agent** (LangChain Agent)  
   - Name: “Sprint Goal Generator”  
   - Prompt (user message) set to:
     - “For the {{ $json.Userstory }} in the column "Userstory" retrieve the information using the following subworkflows/tools:  
       - Retrieve Agile Studio US using only 1 US at a time, so US-1234  
       - Call 'Get Attachments - Agile Studio US'  
       Use the collected information to generate general sprint goals.”
   - System message set to the provided constraints (HTML-only, inline CSS, consistent layout, etc.).
   - Connect **Google Gemini Chat Model** to the agent via the **AI Language Model** connector.

8) **Add HTTP Request Tool node (AI tool)**  
   - Node: **HTTP Request Tool**  
   - Name: “Retrieve Agile Studio US”  
   - Configure:
     - Method: GET  
     - URL: `https://<yoururl>/userstories/{{ $fromAI('Userstory', '', 'string') }}`  
     - Authentication: **OAuth2** via **Generic Credential Type**
   - In Agent node, ensure this node is connected as an **AI Tool**.

9) **Add Tool Workflow node (AI tool)**  
   - Node: **Tool Workflow**  
   - Name: “Call 'Get Attachments - Agile Studio US'”  
   - Configure:
     - Select the target workflow by ID/name (must exist): **Get Attachments - Agile Studio US**
     - Map input `Userstory` to: `{{ $fromAI('Userstory', '', 'string') }}`
   - Connect it to the agent as an **AI Tool**.

10) **Add Gmail node**  
   - Node: **Gmail**  
   - Name: “Send a message”  
   - Configure:
     - Subject: `Sprint Goals`
     - Body: `{{ $json.output }}` (verify the agent output field in your environment; adjust if needed)
     - Set recipients (To/CC) as required in node options/fields.
   - Connect: Sprint Goal Generator → Send a message

11) **Credentials to configure (main workflow):**
   - Google Sheets OAuth2 (read spreadsheet)
   - Gmail OAuth2 (send email)
   - Google Gemini credentials
   - Pega Agile Studio OAuth2 (generic OAuth2 credential on the HTTP tool)
   - Tool Workflow requires the target workflow to be active and accessible

---

### B) Attachment-handling workflow (the “Get Attachments - Agile Studio US” tool workflow)

Your main workflow expects a callable workflow that accepts **`Userstory`** and returns parsed attachment text. You can reproduce it using the nodes shown in Block 2.4:

1) **Create a new workflow** named: “Get Attachments - Agile Studio US”  
2) **Add Execute Workflow Trigger**  
   - Input: `Userstory` (string)
3) **HTTP Request** “Get the Userstory data”  
   - GET `https://<yoururl>/userstories/{{ $json.Userstory }}`  
   - OAuth2 generic credential (Pega)
4) **Split Out** “Iterate over the attachments”  
   - Field: `details.attachmentList`
5) Add three **Filter** nodes to route attachments by URL substring:
   - contains `spreadsheets` → sheets branch
   - contains `document` → docs branch
   - contains `presentation` → slides branch
6) **Sheets branch**:
   - **Switch** on `{{$json.url}}` containing `rtpof=true` (xlsx) vs not (native)
   - Native path:
     - **Set** extract file ID from `/d/<id>` into `url`
     - **HTTP Request** to Sheets API with `fields=properties(title),sheets(...)`
     - **Code** convert response to readable text
   - XLSX path:
     - **Set** extract file ID into `url`
     - **Google Drive download**
     - **Extract From File** (xlsx)
     - **Code** convert rows to readable text
   - **Merge** “Merge the results” to recombine the two paths
7) **Docs branch**:
   - **Set** extract doc ID into `url`
   - **HTTP Request** to Docs API with filtered `fields=...`
   - **Code** convert doc structure to text
8) **Slides branch**:
   - **Set** extract presentation ID into `url`
   - **HTTP Request** to Slides API with filtered `fields=...`
   - **Code** extract text per slide
9) **Final Merge (3 inputs)** to combine: sheets output + docs output + slides output
10) **Publish/activate** this workflow so the Tool Workflow node can call it.
11) **Credentials required (sub-workflow):**
   - Pega OAuth2
   - Google Sheets OAuth2
   - Google Docs OAuth2
   - Google Slides OAuth2
   - Google Drive OAuth2

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create a Google Sheet with a column named “Userstory”, and list the user stories to include. | From sticky note “Create sprint goals using Pega Agile Studio with AI” |
| Set the Google Sheet ID in “Retrieve_Data_From_Sheet”; set up credentials; publish sub-workflows for attachment handling; execute. | From sticky note “Create sprint goals using Pega Agile Studio with AI” |
| Sheets attachments: native Google Sheets can be pulled via Sheets API; XLSX must be downloaded + extracted because the API path doesn’t support it directly. | From sticky note “Native Sheets or .xlsx” |
| Sub-workflow concept: separate retrieval for Sheets, Docs, Slides attachments; XLSX handled via alternate path. | From sticky note “Subworkflow: Retrieving Google Sheet/Docs/Slides attachments” |