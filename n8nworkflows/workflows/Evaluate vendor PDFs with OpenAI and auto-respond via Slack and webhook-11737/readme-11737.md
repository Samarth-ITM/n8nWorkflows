Evaluate vendor PDFs with OpenAI and auto-respond via Slack and webhook

https://n8nworkflows.xyz/workflows/evaluate-vendor-pdfs-with-openai-and-auto-respond-via-slack-and-webhook-11737


# Evaluate vendor PDFs with OpenAI and auto-respond via Slack and webhook

Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** AI Vendor / Partner Proposal Evaluation & Auto-Response System  
**Purpose:** Accept a vendor/partner proposal PDF via webhook, download and extract its text, clean the text, have OpenAI evaluate it into structured JSON, compute a single “risk” decision for the entire PDF (“one PDF = one decision”), then:
- **If risks exist:** send a formatted Slack alert for manual review.
- **If no risks:** auto-approve and return a JSON response to the original webhook caller.

**Target use cases**
- Intake and triage of vendor proposals at scale.
- Automated first-pass compliance/risk screening.
- Routing “clean” proposals to fast approval while escalating risky ones.

### Logical blocks
1.1 **Proposal Intake & PDF Retrieval** (Webhook → HTTP download)  
1.2 **PDF Text Extraction** (binary/file → extracted text)  
1.3 **Text Cleanup** (normalize/remove noise)  
1.4 **AI Evaluation (OpenAI)** (structured extraction into strict JSON)  
1.5 **Structured Parsing & Risk Flagging** (JSON parse + boolean risk flag)  
1.6 **Risk Path: Alerting** (build summary → Slack message)  
1.7 **No-Risk Path: Auto-Approval Response** (build payload → respond to webhook)

---

## 2. Block-by-Block Analysis

### 2.1 Proposal Intake & PDF Retrieval
**Overview:** Receives an HTTP POST containing a URL to a proposal PDF, then downloads the PDF as a file for downstream extraction.  
**Nodes Involved:** `Proposal Upload`, `Download Proposal PDF`

#### Node: Proposal Upload
- **Type / role:** `Webhook` (entry point; receives proposal submission)
- **Configuration (interpreted):**
  - Method: **POST**
  - Path: **/vendor-proposal-upload**
  - Response mode: **Using “Respond to Webhook” node** (responseNode)
- **Key variables/fields expected:**
  - Expects request body to include: `body.proposal_pdf` (a URL)
- **Connections:**
  - Output → `Download Proposal PDF`
- **Version requirements:** Webhook node v2.1
- **Failure / edge cases:**
  - Missing `body.proposal_pdf` → downstream HTTP node URL expression fails.
  - If caller expects immediate response but workflow goes down the “risk” branch, there is currently **no Respond to Webhook** on that branch (likely causing webhook timeout).
  - Webhook path must be registered/activated in n8n; workflow is currently `active: false`.

#### Node: Download Proposal PDF
- **Type / role:** `HTTP Request` (fetches the PDF from the provided URL)
- **Configuration (interpreted):**
  - URL: `{{$json.body.proposal_pdf}}`
  - Response format: **file**
  - Output property name: `data` (binary/file payload)
- **Connections:**
  - Input ← `Proposal Upload`
  - Output → `Parse PDF`
- **Version requirements:** HTTP Request node v4.3
- **Failure / edge cases:**
  - URL not reachable / 4xx/5xx responses.
  - Large PDFs may hit timeouts/memory limits.
  - If the URL requires authentication, this node is not configured for it.
  - If content-type is not a PDF (HTML, redirect, etc.), extraction may fail.

---

### 2.2 PDF Text Extraction
**Overview:** Converts the downloaded PDF file into raw text.  
**Nodes Involved:** `Parse PDF`

#### Node: Parse PDF
- **Type / role:** `Extract From File` (PDF → text extraction)
- **Configuration (interpreted):**
  - Operation: **pdf**
  - Uses the binary/file output (from `Download Proposal PDF`)
- **Connections:**
  - Input ← `Download Proposal PDF`
  - Output → `Clean Text`
- **Version requirements:** Extract From File node v1.1
- **Failure / edge cases:**
  - Scanned/image-only PDFs may extract empty/poor text (no OCR in this configuration).
  - Password-protected PDFs will fail.
  - Very large PDFs may cause execution performance issues.

---

### 2.3 Text Cleanup
**Overview:** Removes known repetitive headers/titles, collapses extra newlines, strips page-number-only lines, and trims whitespace to improve AI input quality.  
**Nodes Involved:** `Clean Text`

#### Node: Clean Text
- **Type / role:** `Code` (JS transformation)
- **Configuration choices:**
  - Reads extracted text from `$json["text"]`
  - Removes a specific repeated title string via regex:
    - `Process Design Document – Process Vendor Invoices for Vendor for ACME Systems Inc\.`
  - Collapses multiple blank lines: `/\n\s*\n/g → '\n'`
  - Removes lines that are only digits: `/^\d+\s*$/gm`
  - `trim()`
  - Returns `{ cleanText }` as the only JSON field
- **Connections:**
  - Input ← `Parse PDF`
  - Output → `Proposal Evaluation`
- **Version requirements:** Code node v2
- **Failure / edge cases:**
  - If upstream extraction doesn’t output `text`, `$json["text"]` is undefined → `.replace` will throw.
  - The title-removal is highly specific; other documents won’t be cleaned of their headers unless expanded.
  - Aggressive cleanup may remove meaningful numeric-only lines (e.g., “2025”).

---

### 2.4 AI Evaluation (OpenAI)
**Overview:** Sends cleaned proposal text to OpenAI with strict instructions to return machine-parseable JSON containing summary, key points, and risks.  
**Nodes Involved:** `Proposal Evaluation`

#### Node: Proposal Evaluation
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` (OpenAI chat/completions via LangChain wrapper)
- **Configuration choices:**
  - Model: **gpt-4o-mini**
  - Temperature: **0.3** (more deterministic)
  - Prompts:
    - **System:** Enforces “strict JSON only”, fixed keys, ignore noise, fill missing info with empty string/array.
    - **User:** Provides `{{ $json.cleanText }}` and exact JSON schema:
      - `vendor_name`, `category`, `summary`, `key_points[]`, `risks_or_concerns[]`
- **Credentials:**
  - OpenAI credential referenced as `"OPENAI Account"` with placeholder id `YOUR_OPENAI_API_KEY`
- **Connections:**
  - Input ← `Clean Text`
  - Output → `Parse OpenAI JSON`
- **Version requirements:** node v2.1
- **Failure / edge cases:**
  - Model may still output non-JSON (markdown fences, trailing commentary) → downstream parse fails.
  - Token limits: very long proposals may be truncated or rejected.
  - Credential/auth errors or usage quota issues.

---

### 2.5 Structured Parsing & Risk Flagging
**Overview:** Parses the AI response into a real JSON object and computes a single boolean `risk_flagged` for the entire proposal (true if any risks exist).  
**Nodes Involved:** `Parse OpenAI JSON`

#### Node: Parse OpenAI JSON
- **Type / role:** `Code` (validation + normalization)
- **Configuration choices:**
  - Extracts raw model text from:  
    `const rawText = $json.output[0].content[0].text;`
  - Parses JSON with `JSON.parse(rawText)`; throws explicit error if invalid.
  - Computes:
    - `risk_flagged = Array.isArray(risks_or_concerns) && risks_or_concerns.length > 0`
  - Returns **exactly one item** with:
    - `vendor_name, category, summary, key_points, risks_or_concerns, risk_flagged`
- **Connections:**
  - Input ← `Proposal Evaluation`
  - Output → `Risks?`
- **Version requirements:** Code node v2
- **Failure / edge cases:**
  - Output path assumption (`$json.output[0].content[0].text`) is specific to this OpenAI node’s response structure; if node output changes, parsing breaks.
  - If OpenAI returns JSON with leading/trailing whitespace it’s fine; but if it returns code fences (```json) parsing fails.
  - If `risks_or_concerns` is missing or not an array, `risk_flagged` becomes false (could mask schema drift).

---

### 2.6 Decision Gate (Risk vs No Risk)
**Overview:** Routes execution based on `risk_flagged`.  
**Nodes Involved:** `Risks?`

#### Node: Risks?
- **Type / role:** `IF` (branching)
- **Configuration choices:**
  - Condition group (AND):
    1. `$json.risk_flagged` **equals** `true`
    2. `$json.risk_flagged` **exists** (configured as an “exists” boolean check)
- **Connections:**
  - Input ← `Parse OpenAI JSON`
  - **True** output → `Build Risk Summary`
  - **False** output → `Auto Approve Payload`
- **Version requirements:** IF node v2.3
- **Failure / edge cases:**
  - The “exists” condition is redundant given the equals check; also it appears configured with `rightValue: false` but uses “exists” operator—this may confuse maintainers and could behave unexpectedly depending on n8n’s internal handling.
  - If `risk_flagged` is undefined, it will route to the false branch (auto-approve) unless the conditions block it.

---

### 2.7 Risk Path: Build Summary & Slack Alert
**Overview:** Formats risks into a structured alert payload and posts it to Slack for manual review.  
**Nodes Involved:** `Build Risk Summary`, `SEND ALERT`

#### Node: Build Risk Summary
- **Type / role:** `Code` (builds a human-readable Slack-ready structure)
- **Configuration choices:**
  - Creates:
    - `title`: “⚠️ Risks Identified in {vendor} Proposal”
    - `risk_count`: length of `risks_or_concerns`
    - `risks`: enumerated list “1. …”
    - `action_required`: fixed text
- **Connections:**
  - Input ← `Risks?` (true branch)
  - Output → `SEND ALERT`
- **Failure / edge cases:**
  - Assumes `risks_or_concerns` is an array; if not, `.length`/`.map` throws.
  - Vendor name empty → awkward alert title.

#### Node: SEND ALERT
- **Type / role:** `Slack` (message posting)
- **Configuration choices:**
  - Posts to a specific channel (selected by ID)
  - Message text uses expressions:
    - Title, vendor, category
    - Risks joined by newline: `{{ $json.risks.join("\n") }}`
    - Action line
- **Credentials:**
  - Slack API credential `"Slack account"` with placeholder id `YOUR_SLACK_BOT_TOKEN`
- **Connections:**
  - Input ← `Build Risk Summary`
  - No further nodes
- **Version requirements:** Slack node v2.4
- **Failure / edge cases:**
  - Missing Slack scopes/channel access.
  - Invalid channel ID.
  - Message length limits if many risks.
  - **No webhook response is sent on this branch**, so webhook callers may time out unless n8n is configured to respond immediately or you add a Respond node here.

---

### 2.8 No-Risk Path: Auto-Approval & Webhook Response
**Overview:** Builds an approval payload and returns a JSON response to the webhook caller.  
**Nodes Involved:** `Auto Approve Payload`, `Respond to Webhook`

#### Node: Auto Approve Payload
- **Type / role:** `Code` (builds approval result)
- **Configuration choices:**
  - Outputs:
    - `vendor_name`
    - `status: "APPROVED"`
    - `message: "No risks found. Proposal approved."`
- **Connections:**
  - Input ← `Risks?` (false branch)
  - Output → `Respond to Webhook`
- **Failure / edge cases:**
  - If vendor_name is empty, response is less useful but still valid.

#### Node: Respond to Webhook
- **Type / role:** `Respond to Webhook` (returns HTTP response to the original webhook request)
- **Configuration choices:**
  - Respond with: **JSON**
  - Body:
    ```json
    {
      "status": "approved",
      "vendor": "{{$json.vendor_name}}",
      "message": "No risks found. Proposal approved."
    }
    ```
- **Connections:**
  - Input ← `Auto Approve Payload`
- **Version requirements:** Respond node v1.5
- **Failure / edge cases:**
  - Only responds on the **no-risk** branch; risk branch has no response node (likely timeout).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Proposal Upload | Webhook | Entry point for proposal submissions | — | Download Proposal PDF | ## Proposal Intake & PDF Processing\nHandles proposal submission, PDF download, and text extraction.\nEnsures the uploaded document is accessible and converted into raw text for downstream AI analysis. |
| Download Proposal PDF | HTTP Request | Downloads PDF file from provided URL | Proposal Upload | Parse PDF | ## Proposal Intake & PDF Processing\nHandles proposal submission, PDF download, and text extraction.\nEnsures the uploaded document is accessible and converted into raw text for downstream AI analysis. |
| Parse PDF | Extract From File | Extracts raw text from PDF | Download Proposal PDF | Clean Text | ## Proposal Intake & PDF Processing\nHandles proposal submission, PDF download, and text extraction.\nEnsures the uploaded document is accessible and converted into raw text for downstream AI analysis. |
| Clean Text | Code | Cleans extracted PDF text (remove noise) | Parse PDF | Proposal Evaluation | ## Text Cleanup & AI Analysis\nCleans extracted PDF text and sends it to OpenAI for structured evaluation.\nRemoves noise and ensures AI receives only relevant proposal content. |
| Proposal Evaluation | OpenAI (LangChain) | Evaluates proposal and outputs strict JSON | Clean Text | Parse OpenAI JSON | ## Text Cleanup & AI Analysis\nCleans extracted PDF text and sends it to OpenAI for structured evaluation.\nRemoves noise and ensures AI receives only relevant proposal content. |
| Parse OpenAI JSON | Code | Parses AI JSON + computes `risk_flagged` | Proposal Evaluation | Risks? | ## Structured Output & Risk Detection\nParses OpenAI output into valid JSON and computes a single risk flag for the entire proposal.\nGuarantees one input PDF produces exactly one decision. |
| Risks? | IF | Branches based on `risk_flagged` | Parse OpenAI JSON | Build Risk Summary (true), Auto Approve Payload (false) | ## Structured Output & Risk Detection\nParses OpenAI output into valid JSON and computes a single risk flag for the entire proposal.\nGuarantees one input PDF produces exactly one decision. |
| Build Risk Summary | Code | Formats risks for alerting | Risks? (true) | SEND ALERT | ## Risk Identified → Alert & Review\nBuilds a structured risk summary and notifies stakeholders via Slack.\nUsed when the proposal contains missing information or potential concerns. |
| SEND ALERT | Slack | Sends Slack alert for manual review | Build Risk Summary | — | ## Risk Identified → Alert & Review\nBuilds a structured risk summary and notifies stakeholders via Slack.\nUsed when the proposal contains missing information or potential concerns. |
| Auto Approve Payload | Code | Builds approval payload | Risks? (false) | Respond to Webhook | ## No Risk → Auto Approval\nAutomatically approves proposals with no detected risks and sends a success response back to the requester. |
| Respond to Webhook | Respond to Webhook | Returns HTTP JSON response to requester | Auto Approve Payload | — | ## No Risk → Auto Approval\nAutomatically approves proposals with no detected risks and sends a success response back to the requester. |
| Sticky Note | Sticky Note | Documentation/comment node | — | — | ##  **AI Vendor / Partner Proposal Evaluation & Auto-Response System**\n\nThis workflow automates the end-to-end evaluation of vendor or partner proposal PDFs using AI. It accepts a proposal upload via webhook, downloads the PDF, extracts and cleans the text, and then uses OpenAI to analyze the proposal for key details and potential risks.\n\nThe system ensures **one PDF = one decision**, preventing duplicate or fragmented outputs. Based on AI-detected risks, the workflow either flags the proposal for manual review with a Slack alert or automatically approves it and responds back to the requester.\n\n### How it works\n\n1. A vendor proposal PDF is submitted through a webhook.\n2. The PDF is downloaded and text is extracted.\n3. The extracted text is cleaned to remove noise like headers and page numbers.\n4. OpenAI evaluates the proposal and returns structured JSON.\n5. A single risk flag is calculated for the entire proposal.\n6. If risks exist, a Slack alert is sent.\n7. If no risks exist, the proposal is auto-approved and a response is returned.\n\n### Setup steps\n\n* Configure webhook URL for proposal uploads\n* Add OpenAI API credentials\n* Connect Slack credentials for alerts\n* Deploy webhook and test with a sample PDF |
| Sticky Note1 | Sticky Note | Documentation/comment node | — | — | ## Proposal Intake & PDF Processing\nHandles proposal submission, PDF download, and text extraction.\nEnsures the uploaded document is accessible and converted into raw text for downstream AI analysis. |
| Sticky Note2 | Sticky Note | Documentation/comment node | — | — | ## Text Cleanup & AI Analysis\nCleans extracted PDF text and sends it to OpenAI for structured evaluation.\nRemoves noise and ensures AI receives only relevant proposal content. |
| Sticky Note3 | Sticky Note | Documentation/comment node | — | — | ## Structured Output & Risk Detection\nParses OpenAI output into valid JSON and computes a single risk flag for the entire proposal.\nGuarantees one input PDF produces exactly one decision. |
| Sticky Note4 | Sticky Note | Documentation/comment node | — | — | ## Risk Identified → Alert & Review\nBuilds a structured risk summary and notifies stakeholders via Slack.\nUsed when the proposal contains missing information or potential concerns. |
| Sticky Note5 | Sticky Note | Documentation/comment node | — | — | ## No Risk → Auto Approval\nAutomatically approves proposals with no detected risks and sends a success response back to the requester. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name: `AI Vendor / Partner Proposal Evaluation & Auto-Response System`
- (Optional) Set execution order setting to `v1` (default in many instances).

2) **Add “Proposal Upload” (Webhook)**
- Node type: **Webhook**
- HTTP Method: **POST**
- Path: `vendor-proposal-upload`
- Response Mode: **Response Node**
- Expected request payload example:
  - JSON body containing `proposal_pdf` URL, e.g.:
    - `{"proposal_pdf":"https://example.com/file.pdf"}`
- Connect **Proposal Upload → Download Proposal PDF**

3) **Add “Download Proposal PDF” (HTTP Request)**
- Node type: **HTTP Request**
- URL expression: `{{$json.body.proposal_pdf}}`
- Response: set to **File**
- Output property name: `data`
- Connect **Download Proposal PDF → Parse PDF**

4) **Add “Parse PDF” (Extract From File)**
- Node type: **Extract From File**
- Operation: **PDF**
- (Use default options)
- Connect **Parse PDF → Clean Text**

5) **Add “Clean Text” (Code)**
- Node type: **Code**
- Paste logic that:
  - Reads `$json.text`
  - Removes repeated title line(s)
  - Collapses blank lines
  - Removes digit-only lines
  - Returns `{ cleanText }`
- Connect **Clean Text → Proposal Evaluation**

6) **Add “Proposal Evaluation” (OpenAI / LangChain)**
- Node type: **OpenAI (LangChain)** (`@n8n/n8n-nodes-langchain.openAi`)
- Credentials: configure **OpenAI API** credential in n8n and select it here.
- Model: `gpt-4o-mini`
- Temperature: `0.3`
- System message: enforce strict JSON output and fixed keys.
- User message: include `{{$json.cleanText}}` and specify the exact JSON schema keys:
  - `vendor_name`, `category`, `summary`, `key_points`, `risks_or_concerns`
- Connect **Proposal Evaluation → Parse OpenAI JSON**

7) **Add “Parse OpenAI JSON” (Code)**
- Node type: **Code**
- Parse the OpenAI text output and compute:
  - `risk_flagged = risks_or_concerns.length > 0`
- Ensure it returns exactly one output item with fields:
  - `vendor_name, category, summary, key_points, risks_or_concerns, risk_flagged`
- Connect **Parse OpenAI JSON → Risks?**

8) **Add “Risks?” (IF)**
- Node type: **IF**
- Condition: `$json.risk_flagged` equals `true`
  - (You can keep or remove the extra “exists” condition; equals true is typically sufficient.)
- Connect:
  - **True → Build Risk Summary**
  - **False → Auto Approve Payload**

9) **Risk branch: Add “Build Risk Summary” (Code)**
- Node type: **Code**
- Build Slack-friendly fields: `title, vendor, category, summary, risk_count, risks[], action_required`
- Connect **Build Risk Summary → SEND ALERT**

10) **Risk branch: Add “SEND ALERT” (Slack)**
- Node type: **Slack**
- Credentials: configure Slack bot/token credential in n8n and select it.
- Operation: send message to a channel
- Channel: select your channel ID
- Message template: include title/vendor/category + joined risks + action.
- (Recommended) Add a **Respond to Webhook** on this branch too, to avoid webhook timeouts.

11) **No-risk branch: Add “Auto Approve Payload” (Code)**
- Node type: **Code**
- Return: `vendor_name`, `status: "APPROVED"`, message text
- Connect **Auto Approve Payload → Respond to Webhook**

12) **No-risk branch: Add “Respond to Webhook”**
- Node type: **Respond to Webhook**
- Respond with: **JSON**
- Body: include `status`, `vendor`, `message` (using expressions)
- Activate the workflow and test the webhook endpoint.

**Credential configuration checklist**
- **OpenAI**: API key set in n8n Credentials; node selects that credential.
- **Slack**: Slack bot token credential; ensure scopes allow posting to the chosen channel.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI Vendor / Partner Proposal Evaluation & Auto-Response System: end-to-end evaluation of proposal PDFs; “one PDF = one decision”; risk → Slack alert; no risk → auto-approve & respond. Setup: webhook URL, OpenAI credentials, Slack credentials, deploy webhook, test with sample PDF. | From workflow sticky note (documentation block) |
| Proposal Intake & PDF Processing: submission → download → extraction. | Sticky note over intake nodes |
| Text Cleanup & AI Analysis: clean noise, send to OpenAI for structured evaluation. | Sticky note over AI prep nodes |
| Structured Output & Risk Detection: parse OpenAI output, compute single risk flag. | Sticky note over parsing/IF nodes |
| Risk Identified → Alert & Review: build risk summary and notify via Slack. | Sticky note over risk branch |
| No Risk → Auto Approval: approve and respond back to requester. | Sticky note over approval branch |