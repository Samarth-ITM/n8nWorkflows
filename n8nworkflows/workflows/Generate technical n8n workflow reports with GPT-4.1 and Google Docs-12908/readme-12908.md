Generate technical n8n workflow reports with GPT-4.1 and Google Docs

https://n8nworkflows.xyz/workflows/generate-technical-n8n-workflow-reports-with-gpt-4-1-and-google-docs-12908


# Generate technical n8n workflow reports with GPT-4.1 and Google Docs

## 1. Workflow Overview

**Purpose:** This workflow generates a **technical operation report** for a user-selected n8n workflow by (1) fetching its JSON via the n8n API, (2) normalizing/cleaning it for safe LLM consumption, (3) generating an **HTML documentation report** using **OpenAI GPT‑4.1**, (4) optionally analyzing any **Code** nodes found, and (5) uploading the resulting HTML as a **Google Docs** file in Google Drive.

**Target use cases**
- Documenting workflows you did not annotate (or forgot details about)
- Creating standardized technical documentation for audits, maintenance, handover, or governance
- Separating *general workflow description* from *code-node forensic analysis*

### 1.1 Entry & Workflow Retrieval
Starts manually and retrieves the selected workflow JSON from n8n.

### 1.2 JSON Normalization & Safety Cleanup
Removes noise/sensitive or irrelevant fields and produces:
- normalized node list
- flattened graph edges
- extracted Code node scripts (kept intact)

### 1.3 LLM Documentation Generation (Main Report)
Generates a full HTML document describing the workflow.

### 1.4 Code Node Detection & Optional Code Analysis
Detects if Code nodes exist; if yes, sends them to a second LLM pass to produce a code-focused HTML section.

### 1.5 Google Docs Creation (Upload to Drive)
Creates a Google Docs document via Google Drive multipart upload:
- with code analysis if present
- otherwise only the main report

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & Workflow Retrieval

**Overview:** Triggers the process manually and retrieves the JSON definition of a user-selected workflow via the n8n API node.

**Nodes involved:**
- Manual Trigger
- Select Workflow

#### Node: Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — interactive entry point.
- **Configuration choices:** No parameters.
- **Inputs / outputs:** No input; emits a single empty item to start the chain.
- **Connections:**
  - Output → **Select Workflow**
- **Edge cases / failures:**
  - None typical; only runs when manually executed.

#### Node: Select Workflow
- **Type / role:** `n8n-nodes-base.n8n` — calls the n8n API to fetch workflow data.
- **Configuration choices (interpreted):**
  - **Operation:** `get` (retrieve one workflow definition)
  - **Workflow selection:** uses UI “list” selector; currently points to workflow id `exOYTSuZ3nUTQLmY` (cached display name suggests another workflow in the instance)
  - **Request options:** default (empty)
- **Credentials:**
  - **n8nApi** credential named **“n8n Templates”** (API key / API access)
- **Inputs / outputs:**
  - Input: trigger item
  - Output: workflow JSON payload (either as root object or wrapped in `{ workflow: {...} }`, handled later)
- **Connections:**
  - Input ← Manual Trigger
  - Output → **Normalize Workflow JSON**
- **Edge cases / failures:**
  - Authentication/permission failure to n8n API
  - Wrong workflowId, deleted workflow, or insufficient access
  - Instance restrictions (e.g., network, self-hosted base URL issues) depending on credential setup

---

### Block 2 — JSON Normalization & Safety Cleanup

**Overview:** Cleans and restructures the retrieved workflow JSON into a minimized, LLM-friendly payload, while preserving Code node scripts (`jsCode`) intact and generating a flattened edge list to reconstruct execution order.

**Nodes involved:**
- Normalize Workflow JSON

#### Node: Normalize Workflow JSON
- **Type / role:** `n8n-nodes-base.code` — transforms the workflow JSON.
- **Configuration choices (interpreted):**
  - Reads from `items[0].json`.
  - If missing input, returns `{ error: "Workflow not found in items[0].json" }`.
  - Normalizes wrapper shape: if input is `{ workflow: {...} }` it uses `raw.workflow`; otherwise uses `raw`.
  - **Node cleanup:** keeps only a curated set of node fields:
    - `id, name, type, typeVersion, position, disabled, notes, notesInFlow, retryOnFail, continueOnFail, alwaysOutputData, executeOnce`
  - **Parameter trimming:**
    - Normal strings are trimmed up to `DEFAULT_MAX_STRING = 8000`
    - `parameters.jsCode` is **never trimmed** (`MAX_JS_CODE = null`)
  - **Trigger detection:** sets `isTrigger` if `node.type` contains `"trigger"` (case-insensitive)
  - **Code node extraction:** produces `codeNodes` array with:
    - `id, name, type, jsCode, lineCount, charCount`
  - **Connections flattening:** converts `wf.connections` into `edges[]` with:
    - `from, to, connectionType, fromOutputIndex, toInputIndex, toType`
  - **Settings subset:** keeps only workflow settings:
    - `timezone, saveDataErrorExecution, saveDataSuccessExecution, saveManualExecutions, executionTimeout`
  - Final output contains:
    - `workflowName, workflowId, active, nodeCount, edgeCount, nodes, codeNodes, edges, connections, settings`
- **Key expressions/variables used:**
  - Uses `items?.[0]?.json` (safe navigation)
  - Uses JSON stringify/parse with a replacer to trim strings but preserve `jsCode`
- **Inputs / outputs:**
  - Input: result of **Select Workflow**
  - Output: normalized workflow documentation payload
- **Connections:**
  - Input ← Select Workflow
  - Output → **Generate Report**
- **Version-specific requirements:**
  - Code node is **typeVersion 2** (modern code node execution environment).
- **Edge cases / failures:**
  - Very large workflow parameters may still cause large payloads (8000-char trimming helps, but node count can still be large)
  - If workflow JSON schema changes, picker keys may omit useful fields
  - If connections object is absent/malformed, `edges` becomes empty and later documentation must rely on “Information not available…”

---

### Block 3 — LLM Documentation Generation (Main Report)

**Overview:** Sends the normalized workflow payload to GPT‑4.1 to generate a structured **HTML** technical document.

**Nodes involved:**
- Generate Report

#### Node: Generate Report
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — “Message a model” style LLM call.
- **Configuration choices (interpreted):**
  - **Model:** `gpt-4.1`
  - **Retries:** `maxTries = 5`, `retryOnFail = true`, `waitBetweenTries = 3000ms`
  - **Prompting:**
    - System prompt defines role, analysis rules, and **strict HTML output** structure.
    - User message builds `contexto_1 = {{ JSON.stringify($json) }}` and instructs to analyze only that variable.
- **Key expressions/variables used:**
  - `{{ JSON.stringify($json) }}` embeds the entire normalized object into the prompt
- **Inputs / outputs:**
  - Input: normalized object from **Normalize Workflow JSON**
  - Output: OpenAI response object; later nodes reference:
    - `$('Generate Report').item.json.output[0].content[0].text`
- **Connections:**
  - Input ← Normalize Workflow JSON
  - Output → **Collect Code Node Info**
- **Edge cases / failures:**
  - **Prompt inconsistency risk:** the user prompt text says “generate … in Markdown” but also says “Return only … in HTML”; system prompt requires HTML. This can lead to occasional formatting drift.
  - Token/context limits if the normalized workflow is large (even with trimming)
  - OpenAI credential/model availability or quota errors
  - Output-path fragility: downstream expects `output[0].content[0].text`; response schema changes could break expressions

---

### Block 4 — Code Node Detection & Optional Code Analysis

**Overview:** Extracts Code nodes from the normalized payload, checks if any exist, and if yes performs a dedicated code-forensic LLM analysis, then merges it into the main report.

**Nodes involved:**
- Collect Code Node Info
- Are there Code Nodes?
- Analyze Code Nodes
- Merge Report and Code Node Analysis

#### Node: Collect Code Node Info
- **Type / role:** `n8n-nodes-base.code` — creates a compact list of Code nodes.
- **Configuration choices (interpreted):**
  - Reads normalized nodes from:
    - `$('Normalize Workflow JSON').first().json.nodes ?? []`
  - Filters to `type === 'n8n-nodes-base.code'`
  - Outputs:
    - `codeNodeCount`
    - `codeNodes: [{ id, name, jsCode }]`
- **Key expressions/variables used:**
  - Node reference: `$('Normalize Workflow JSON').first()`
- **Inputs / outputs:**
  - Input: comes from **Generate Report** (but it does not use that input; it reaches back to Normalize node)
  - Output: code-node summary
- **Connections:**
  - Input ← Generate Report
  - Output → **Are there Code Nodes?**
- **Edge cases / failures:**
  - If the Normalize node didn’t run (or was renamed), node reference fails
  - If normalized structure changes, `.json.nodes` may be missing

#### Node: Are there Code Nodes?
- **Type / role:** `n8n-nodes-base.if` — branching controller.
- **Configuration choices (interpreted):**
  - Condition: `{{ $json.codeNodeCount }} > 0`
  - Strict type validation enabled
- **Inputs / outputs:**
  - Input: output of **Collect Code Node Info**
  - Outputs:
    - **True branch (main output 0):** goes to **Analyze Code Nodes**
    - **False branch (main output 1):** goes to **Create Google Docs Document_1**
- **Connections:**
  - True → Analyze Code Nodes
  - False → Create Google Docs Document_1
- **Edge cases / failures:**
  - If `codeNodeCount` is missing or not numeric, strict validation may fail
  - If Code nodes exist but were not detected (type mismatch), analysis is skipped

#### Node: Analyze Code Nodes
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — second LLM pass for code-only analysis.
- **Configuration choices (interpreted):**
  - **Model:** `gpt-4.1`
  - **Retries:** `maxTries = 5`, `retryOnFail = true`, `waitBetweenTries = 3000ms`
  - **Prompting:**
    - System prompt requires **strict HTML** with title `<h2>Existing code nodes in the workflow</h2>` and a `<section>` per node.
    - User message sends `code_nodes = {{ JSON.stringify($json.codeNodes) }}` (note: expects codeNodes field on input item).
- **Inputs / outputs:**
  - Input: should contain `codeNodes` and count; however, **the IF input item comes from Collect Code Node Info, which outputs `codeNodes`** (ok).
  - Output: OpenAI response object; downstream expects:
    - `$json.output[0].content[0].text` (when merged)
- **Connections:**
  - Input ← Are there Code Nodes? (true)
  - Output → **Merge Report and Code Node Analysis**
- **Edge cases / failures:**
  - Same LLM schema fragility as main report (`output[0].content[0].text`)
  - Token limits if `jsCode` blocks are large
  - If `codeNodes` is empty but branch taken (shouldn’t happen), output may be low-quality

#### Node: Merge Report and Code Node Analysis
- **Type / role:** `n8n-nodes-base.set` — composes final HTML content.
- **Configuration choices (interpreted):**
  - Creates a string field named `complete_text`:
    - First part: main report HTML from **Generate Report**
    - Then `<br>` line break
    - Then code analysis HTML from **Analyze Code Nodes**
  - Expression used:
    - `{{ $('Generate Report').item.json.output[0].content[0].text }} <br> {{ $json.output[0].content[0].text }}`
- **Inputs / outputs:**
  - Input: output item from **Analyze Code Nodes**
  - Output: item containing `complete_text`
- **Connections:**
  - Input ← Analyze Code Nodes
  - Output → **Create Google Docs Document**
- **Edge cases / failures:**
  - If either LLM output path is missing, expression evaluation fails
  - HTML concatenation may produce invalid HTML if either section isn’t strictly HTML
- **Version-specific requirements:**
  - Set node **typeVersion 3.4** (modern “assignments” UI)

---

### Block 5 — Google Docs Creation (Upload to Drive)

**Overview:** Uploads the generated HTML as a Google Docs document to a specified Google Drive folder using the Drive multipart upload endpoint.

**Nodes involved:**
- Create Google Docs Document (with code analysis)
- Create Google Docs Document_1 (without code analysis)

#### Node: Create Google Docs Document
- **Type / role:** `n8n-nodes-base.httpRequest` — HTTP multipart upload to Google Drive.
- **Configuration choices (interpreted):**
  - **Method:** POST
  - **URL:** `https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart&supportsAllDrives=true`
  - **Authentication:** predefined credential type → `googleDriveOAuth2Api`
  - **Content-Type:** raw body, `multipart/related; boundary=foo_bar_baz`
  - **Body structure:**
    1) JSON metadata part:
       - `name`: `{{ $('Normalize Workflow JSON').item.json.workflowName }}_Technical Documentation`
       - `mimeType`: `application/vnd.google-apps.document`
       - `parents`: `["YOUR_FOLDER_ID"]` (placeholder to replace)
    2) HTML content part:
       - `{{ $json.complete_text }}`
  - **Retries:** `maxTries=5`, `retryOnFail=true`, `waitBetweenTries=3000ms`
  - **supportsAllDrives:** enabled (Shared Drives compatible)
- **Credentials:**
  - **Google Drive OAuth2** credential named **“Google Drive Templates”**
- **Inputs / outputs:**
  - Input: output from **Merge Report and Code Node Analysis**
  - Output: Google Drive API response (file metadata)
- **Connections:**
  - Input ← Merge Report and Code Node Analysis
  - Output → (none)
- **Edge cases / failures:**
  - `YOUR_FOLDER_ID` not replaced → Drive API error (invalid parent / not found)
  - OAuth scope missing (needs Drive file creation permissions)
  - Multipart boundary formatting issues if modified
  - HTML too large for upload limits (rare but possible with very large workflows)

#### Node: Create Google Docs Document_1
- **Type / role:** `n8n-nodes-base.httpRequest` — same upload, but only main report.
- **Configuration choices (interpreted):**
  - Same endpoint and multipart approach
  - Document name uses suffix `_Technical documentation` (note lowercase “documentation” vs node above)
  - HTML content is taken directly from:
    - `{{ $('Generate Report').item.json.output[0].content[0].text }}`
  - Parent folder placeholder: `["YOUR_FOLDER_ID"]`
- **Credentials:**
  - Same **Google Drive Templates** OAuth2 credential
- **Inputs / outputs:**
  - Input: from IF false branch
  - Output: Google Drive API response
- **Connections:**
  - Input ← Are there Code Nodes? (false)
  - Output → (none)
- **Edge cases / failures:**
  - Same as above (folder id, permissions, output-path fragility)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Human documentation / workflow purpose & setup instructions |  |  | # Workflow Operation Report Generator<br>This will probably sound familiar. You design a great workflow, but you don’t add any sticky notes or document it. After some time, you want to modify some of its settings, but you no longer remember exactly what each node was supposed to do.<br><br>The solution to that annoying problem is this workflow. Its purpose is to generate a report, in a Google Docs document, based on a structured analysis of the workflow selected by the user.<br><br>## **How it works / What it does**<br>1. **Manual trigger**: The workflow starts manually.<br>2. **Select and fetch the workflow to document**: Retrieves the structured information of the selected workflow.<br>3. **JSON normalization and cleanup**: The selected workflow is cleaned and transformed to keep only the relevant (and safe) information needed to generate the technical documentation.<br>4. **Documentation report generation**: A professional HTML report is created using an LLM (OpenAI GPT-4.1).<br>5. **Code node analysis**: If the workflow contains Code nodes, they are analyzed and the result is added to the main report.<br>6. **Final document creation**: The HTML report is uploaded as a Google Docs document in Google Drive.<br><br>## **How to set it up**<br>1. **Add the nodes in the following order**:<br>   * Manual Trigger<br>   * n8n node “Select Workflow” (choose the desired workflow)<br>   * Code node “Normalize Workflow JSON”<br>   * OpenAI node (Message a model) “Generate Report” (add system prompt and user prompt)<br>   * Code node “Collect Code Node Info”<br>   * If node “Are there Code Nodes?”<br>   * OpenAI node (Message a model) “Analyze Code Nodes” (add system prompt and user prompt)<br>   * Edit Fields (Set) node “Merge Report and Code Node Analysis”<br>   * HTTP Request node “Create Google Docs Document” (including Code node analysis)<br>   * HTTP Request node “Create Google Docs Document_1” (without including Code node analysis)<br>2. **Configure credentials** for n8n, OpenAI, and Google Drive.<br>3. **Run a test execution** to make sure the document is generated and archived.<br><br>## **Requirements**<br>* **Credentials**:<br>  * n8n API key<br>  * OpenAI API key<br>  * Google Drive (OAuth2) with read/write permissions<br>* **Access**:<br>  * Write access to the target Google Drive folders<br><br>## **How to customize the workflow**<br>* **Automate the trigger**: Use a Webhook Trigger to run it on-demand from external apps.<br>* **Change the prompts (system and user) in the OpenAI nodes** to generate different types of content. |
| Manual Trigger | n8n-nodes-base.manualTrigger | Manual entry point |  | Select Workflow |  |
| Select Workflow | n8n-nodes-base.n8n | Fetch selected workflow definition from n8n API | Manual Trigger | Normalize Workflow JSON |  |
| Normalize Workflow JSON | n8n-nodes-base.code | Normalize/clean workflow JSON; extract edges and Code nodes | Select Workflow | Generate Report |  |
| Generate Report | @n8n/n8n-nodes-langchain.openAi | LLM generates main HTML technical documentation | Normalize Workflow JSON | Collect Code Node Info |  |
| Collect Code Node Info | n8n-nodes-base.code | Extract Code-node scripts into compact payload | Generate Report | Are there Code Nodes? |  |
| Are there Code Nodes? | n8n-nodes-base.if | Branching: run code analysis only if Code nodes exist | Collect Code Node Info | Analyze Code Nodes; Create Google Docs Document_1 |  |
| Analyze Code Nodes | @n8n/n8n-nodes-langchain.openAi | LLM forensic analysis of Code node scripts (HTML section) | Are there Code Nodes? | Merge Report and Code Node Analysis |  |
| Merge Report and Code Node Analysis | n8n-nodes-base.set | Concatenate main report + code analysis into `complete_text` | Analyze Code Nodes | Create Google Docs Document |  |
| Create Google Docs Document | n8n-nodes-base.httpRequest | Upload final HTML (with code analysis) as Google Docs file | Merge Report and Code Node Analysis |  |  |
| Create Google Docs Document_1 | n8n-nodes-base.httpRequest | Upload main HTML (without code analysis) as Google Docs file | Are there Code Nodes? |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual example (image) |  |  | ![Mi imagen](https://lh3.googleusercontent.com/d/1ultzktRcOF1XNL0C_pAuEEkBOzXiFq2o#full-width) |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual example (image) |  |  | ![Mi imagen](https://lh3.googleusercontent.com/d/1hrx93ZVnAe_7PG_pJDlGjZ9-EuneUVTQ#full-width) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual example (image) |  |  | ![Mi imagen](https://lh3.googleusercontent.com/d/1aBgb7KGZEiGPkTsxTxBTLy3yLVaXvqqj#full-width) |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual example (image) |  |  | ![Mi imagen](https://lh3.googleusercontent.com/d/1Tvq74Ul3TfamDU0HxI5ug6xhXHSYAszu#full-width) |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual example (image) |  |  | ![Mi imagen](https://lh3.googleusercontent.com/d/1FWfXYYykClqd89shOR8Bi2QoNRA97jXD#full-width) |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual example (image) |  |  | ![Mi imagen](https://lh3.googleusercontent.com/d/1MR1REAL3Iotg4DPNKcwee6hnRInLSv1m#full-width) |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual example (image) |  |  | ![Mi imagen](https://lh3.googleusercontent.com/d/1PPdhDp1TqCM7wP_OIm-Zw39AwC-dwPWz#full-width) |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual example (image) |  |  | ![Mi imagen](https://lh3.googleusercontent.com/d/1pWplUg5-VInxFCDSESzCzvmHLqvI1TiR#full-width) |
| Sticky Note9 | n8n-nodes-base.stickyNote | Visual example (image) |  |  | ![Mi imagen](https://lh3.googleusercontent.com/d/1J4b9_sOdIKUqic1GcnDJ5eNtsXEPtcxg#full-width) |
| Sticky Note10 | n8n-nodes-base.stickyNote | Visual example (image) |  |  | ![Mi imagen](https://lh3.googleusercontent.com/d/1p46mjrexkHLLfVhT0krU4HPRlPa2sZuy#full-width) |
| Sticky Note11 | n8n-nodes-base.stickyNote | Label for examples |  |  | ## FINAL REPORT EXAMPLE |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name: **Workflow Operation Report Generator**
- Set workflow to **Inactive** initially (optional, matches provided JSON)

2) **Add “Manual Trigger”**
- Node: **Manual Trigger**
- No configuration needed

3) **Add “Select Workflow”**
- Node type: **n8n**
- Operation: **Get**
- WorkflowId: choose from list (the workflow you want to document)
- Credentials: **n8n API** credential (API key / access)
  - Must allow reading workflows via n8n API

4) **Connect:** Manual Trigger → Select Workflow

5) **Add “Normalize Workflow JSON” (Code node)**
- Node type: **Code**
- Paste the normalization script (as provided)
- Ensure it:
  - picks/keeps only key node fields
  - trims long strings but **does not trim** `parameters.jsCode`
  - outputs: `nodes`, `codeNodes`, `edges`, `connections`, `settings`, etc.

6) **Connect:** Select Workflow → Normalize Workflow JSON

7) **Add “Generate Report” (OpenAI)**
- Node type: **OpenAI (LangChain) – Message a model**
- Model: **gpt-4.1**
- Retries:
  - Enable **Retry on fail**
  - `Max tries: 5`
  - `Wait between tries: 3000 ms`
- Credentials: **OpenAI API**
- Messages:
  - **System**: paste the system prompt that requires **strict HTML** and the specified section structure
  - **User**: include `contexto_1 = {{ JSON.stringify($json) }}` and instruct model to use only that content and output HTML

8) **Connect:** Normalize Workflow JSON → Generate Report

9) **Add “Collect Code Node Info” (Code node)**
- Node type: **Code**
- Script logic:
  - `const nodes = $('Normalize Workflow JSON').first().json.nodes ?? [];`
  - filter `n.type === 'n8n-nodes-base.code'`
  - output `{ codeNodeCount, codeNodes: [{id,name,jsCode}] }`

10) **Connect:** Generate Report → Collect Code Node Info

11) **Add “Are there Code Nodes?” (IF)**
- Node type: **IF**
- Condition (Number):
  - Left: `={{ $json.codeNodeCount }}`
  - Operation: `> (gt)`
  - Right: `0`
- Keep strict validation enabled (default in recent versions)

12) **Connect:** Collect Code Node Info → Are there Code Nodes?

13) **Add “Analyze Code Nodes” (OpenAI)**
- Node type: **OpenAI (LangChain) – Message a model**
- Model: **gpt-4.1**
- Retries: same pattern (max 5, wait 3000ms)
- Credentials: **OpenAI API**
- Messages:
  - System: code-forensic analyzer prompt requiring strict HTML with `<h2>Existing code nodes in the workflow</h2>`
  - User: embed `{{ JSON.stringify($json.codeNodes) }}` and instruct to iterate in order and output per-node sections

14) **Connect:** IF (true output) → Analyze Code Nodes

15) **Add “Merge Report and Code Node Analysis” (Set / Edit Fields)**
- Node type: **Set**
- Add string field named **complete_text**
- Value expression:
  - `={{ $('Generate Report').item.json.output[0].content[0].text }}\n<br>\n\n{{ $json.output[0].content[0].text }}`
  - (Second part assumes the input item is from Analyze Code Nodes)

16) **Connect:** Analyze Code Nodes → Merge Report and Code Node Analysis

17) **Add “Create Google Docs Document” (HTTP Request) — with code analysis**
- Node type: **HTTP Request**
- Method: **POST**
- URL: `https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart&supportsAllDrives=true`
- Authentication: **Predefined credential type**
- Credential type: **Google Drive OAuth2 API**
- Content type: **Raw**
- Raw Content-Type header value:
  - `multipart/related; boundary=foo_bar_baz`
- Enable:
  - Send Query Parameters: yes
  - Send Body: yes
  - Send Headers: yes (even if empty)
- Query parameters:
  - `uploadType = multipart`
  - `supportsAllDrives = true`
- Body (raw multipart/related):
  - Metadata JSON part:
    - name: `{{ $('Normalize Workflow JSON').item.json.workflowName }}_Technical Documentation`
    - mimeType: `application/vnd.google-apps.document`
    - parents: `["YOUR_FOLDER_ID"]` → **replace with a real Drive folder ID**
  - HTML part:
    - `{{ $json.complete_text }}`

18) **Connect:** Merge Report and Code Node Analysis → Create Google Docs Document

19) **Add “Create Google Docs Document_1” (HTTP Request) — without code analysis**
- Duplicate the previous HTTP node
- In body HTML part, use only:
  - `{{ $('Generate Report').item.json.output[0].content[0].text }}`
- Keep the same folder ID replacement requirement

20) **Connect:** IF (false output) → Create Google Docs Document_1

21) **Credentials checklist**
- **n8n API credential**: can read workflows
- **OpenAI credential**: access to `gpt-4.1`
- **Google Drive OAuth2**: permissions to create files in target folder (and Shared Drives if used)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided disclaimer (global) |
| Workflow concept and setup guidance (manual trigger → fetch → normalize → LLM report → optional code analysis → Google Docs upload) | Sticky Note content embedded in the workflow |
| Example/visual assets (multiple images) | https://lh3.googleusercontent.com/d/1MR1REAL3Iotg4DPNKcwee6hnRInLSv1m#full-width ; https://lh3.googleusercontent.com/d/1ultzktRcOF1XNL0C_pAuEEkBOzXiFq2o#full-width ; https://lh3.googleusercontent.com/d/1pWplUg5-VInxFCDSESzCzvmHLqvI1TiR#full-width ; https://lh3.googleusercontent.com/d/1PPdhDp1TqCM7wP_OIm-Zw39AwC-dwPWz#full-width ; https://lh3.googleusercontent.com/d/1J4b9_sOdIKUqic1GcnDJ5eNtsXEPtcxg#full-width ; https://lh3.googleusercontent.com/d/1p46mjrexkHLLfVhT0krU4HPRlPa2sZuy#full-width ; https://lh3.googleusercontent.com/d/1aBgb7KGZEiGPkTsxTxBTLy3yLVaXvqqj#full-width ; https://lh3.googleusercontent.com/d/1hrx93ZVnAe_7PG_pJDlGjZ9-EuneUVTQ#full-width ; https://lh3.googleusercontent.com/d/1Tvq74Ul3TfamDU0HxI5ug6xhXHSYAszu#full-width ; https://lh3.googleusercontent.com/d/1FWfXYYykClqd89shOR8Bi2QoNRA97jXD#full-width |
| Placeholder to replace: `YOUR_FOLDER_ID` | Used in both Google Docs upload nodes (Drive “parents”) |