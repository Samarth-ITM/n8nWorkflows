Verify document authenticity with Claude and record proofs on blockchain

https://n8nworkflows.xyz/workflows/verify-document-authenticity-with-claude-and-record-proofs-on-blockchain-12276


# Verify document authenticity with Claude and record proofs on blockchain

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Verify document authenticity with Claude and record proofs on blockchain  
**Workflow name (internal):** Anthropic Chat Blockchain Policy Authenticity and Claims Validation Network

**Purpose:**  
This workflow receives an uploaded document (intended as an insurance policy/claim document), extracts PDF text, uses Anthropic Claude to assess authenticity (tampering/forgery signals), then:
- If **authentic**: publishes a verification record to a blockchain endpoint and distributes proof to carriers and regulatory bodies.
- If **suspicious/forged**: flags it and emails a fraud alert to a configured compliance address.

**Primary use cases:**
- Insurance policy and claims document validation
- Compliance/audit trails with tamper-evident proofs
- Automated fraud screening + escalation to humans

### 1.1 Input Reception & Configuration
Receives the document via webhook and sets environment-style configuration values (API endpoints, alert recipient).

### 1.2 Document Text Extraction
Extracts text from the submitted PDF.

### 1.3 AI Authenticity Analysis (Claude + Structured Output)
Sends extracted text to a LangChain Agent powered by Anthropic Chat Model and enforces a strict JSON output schema.

### 1.4 Decision Routing
IF node routes authentic vs. forged flows.

### 1.5 Verified Document: Blockchain Proof + Distribution
Prepares a blockchain record, POSTs it to a blockchain API, then forwards resulting proof fields to carrier and regulator endpoints.

### 1.6 Suspicious Document: Alerting
Builds a human-readable fraud alert payload and sends an email via Gmail.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Configuration

**Overview:**  
Accepts inbound document submissions via HTTP POST webhook, then injects required configuration values (API URLs, alert email) into the workflow context.

**Nodes involved:**
- Document Submission Webhook
- Workflow Configuration

#### Node: Document Submission Webhook
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — workflow entry point; receives incoming HTTP request.
- **Key configuration:**
  - **Path:** `document-submission`
  - **Method:** `POST`
  - **Response mode:** `lastNode` (the final node executed determines HTTP response)
- **Input/Output:**
  - **Input:** external HTTP request
  - **Output:** to **Workflow Configuration**
- **Expectations / payload:**
  - For the PDF extraction node to work, the webhook request must include a **binary file** (typically `binary.data`) representing a PDF.
  - Any metadata like `policyId` should be included in JSON body fields if you want it later in the blockchain record.
- **Edge cases / failures:**
  - Missing/invalid binary data → extraction fails.
  - Large files may hit n8n payload limits or timeouts depending on instance config.
  - `responseMode: lastNode` means if execution errors mid-way, caller may get an error rather than a controlled response.

#### Node: Workflow Configuration
- **Type / role:** `Set` (n8n-nodes-base.set) — defines runtime configuration constants.
- **Key configuration choices:**
  - Adds these fields (placeholders must be replaced):
    - `blockchainApiUrl` (Blockchain API endpoint URL)
    - `carriersApiUrl` (Insurance carriers API endpoint URL)
    - `regulatoryApiUrl` (Tax/regulatory bodies API endpoint URL)
    - `alertEmail` (recipient for fraud alerts)
  - **Include other fields:** enabled (preserves incoming webhook payload fields/binary alongside config)
- **Input/Output:**
  - Input: from **Document Submission Webhook**
  - Output: to **Extract Document Content**
- **Edge cases / failures:**
  - Leaving placeholder values unchanged causes HTTP request nodes to fail (invalid URL).
  - If you expect credentials/headers for APIs, they are not configured here; you must add them on HTTP nodes.

---

### Block 2 — Document Text Extraction

**Overview:**  
Extracts text from a submitted PDF, producing a `text` field used for AI analysis.

**Nodes involved:**
- Extract Document Content

#### Node: Extract Document Content
- **Type / role:** `Extract From File` (n8n-nodes-base.extractFromFile) — parses document content.
- **Key configuration:**
  - **Operation:** `pdf` (extract text from PDF)
- **Input/Output:**
  - Input: from **Workflow Configuration** (which should still carry the binary PDF)
  - Output: to **Document Authenticity Validator**
- **Edge cases / failures:**
  - Non-PDF input or encrypted/scanned PDFs without embedded text may yield empty/poor `text`.
  - Very large PDFs can cause memory/time issues.
  - If the binary property name differs from what Extract From File expects, you must align it.

---

### Block 3 — AI Authenticity Analysis (Claude + Structured Output)

**Overview:**  
A LangChain Agent prompts Claude to assess authenticity signals and produce a strictly structured JSON result (boolean, score, findings, risk level, recommendation).

**Nodes involved:**
- Document Authenticity Validator
- Anthropic Chat Model
- Validation Result Parser

#### Node: Anthropic Chat Model
- **Type / role:** `LangChain Chat Model (Anthropic)` (`@n8n/n8n-nodes-langchain.lmChatAnthropic`) — provides Claude model to the agent.
- **Key configuration:**
  - **Model:** `claude-sonnet-4-5-20250929` (as selected in node)
  - Uses **Anthropic API** credentials (`Anthropic account`)
- **Connections:**
  - Connected to **Document Authenticity Validator** via `ai_languageModel`
- **Edge cases / failures:**
  - Invalid/expired API key or insufficient quota.
  - Model name availability depends on Anthropic account/region and node version.
  - Provider outages/timeouts.

#### Node: Validation Result Parser
- **Type / role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces JSON schema.
- **Key configuration:**
  - **Schema:** manual JSON schema requiring:
    - `isAuthentic` (boolean)
    - `confidenceScore` (number 0–100 implied by prompt; schema does not enforce bounds)
    - `findings` (array of strings)
    - `riskLevel` ∈ {low, medium, high, critical}
    - `recommendation` ∈ {approve, review, reject}
  - Required fields: all above
- **Connections:**
  - Connected to **Document Authenticity Validator** via `ai_outputParser`
- **Edge cases / failures:**
  - If the model returns invalid JSON or missing required fields, parsing fails and the workflow errors.
  - Enum mismatch (e.g., “Low” instead of “low”) will fail schema validation.

#### Node: Document Authenticity Validator
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates prompt + model + parser.
- **Key configuration:**
  - **Input text:** `={{ $json.text }}`
  - **System message:** instructs authenticity checks and requires structured JSON output fields.
  - **Prompt type:** `define`
  - **Has output parser:** enabled (wired to Validation Result Parser)
- **Input/Output:**
  - Input: from **Extract Document Content**
  - Output: to **Check Document Authenticity**
- **Edge cases / failures:**
  - Empty `text` leads to low-quality assessment or parser failure if model hallucinates structure incorrectly.
  - Larger `text` may exceed model context limits; consider truncation/summarization if needed.
  - If upstream data includes sensitive information, ensure compliance with your data policies before sending to Anthropic.

---

### Block 4 — Decision Routing

**Overview:**  
Routes the flow based on the AI result: authentic documents proceed to blockchain proof; forged documents trigger alerting.

**Nodes involved:**
- Check Document Authenticity

#### Node: Check Document Authenticity
- **Type / role:** `IF` (n8n-nodes-base.if) — boolean gate.
- **Condition:**
  - Checks: `={{ $('Document Authenticity Validator').item.json.isAuthentic }}` is **true**
  - Uses “loose” type validation (more permissive coercion)
- **Outputs:**
  - **True** → Prepare Blockchain Record
  - **False** → Flag Forged Document
- **Edge cases / failures:**
  - If `isAuthentic` is missing due to parser failure, execution likely fails earlier; if it somehow passes as undefined, IF may route unexpectedly depending on coercion.

---

### Block 5 — Verified Document: Blockchain Proof + Distribution

**Overview:**  
Builds a verification record, publishes it to a blockchain API, then forwards proof details to carriers and regulators via HTTP POST.

**Nodes involved:**
- Prepare Blockchain Record
- Publish to Blockchain
- Send Proof to Carriers
- Send Proof to Regulatory Bodies

#### Node: Prepare Blockchain Record
- **Type / role:** `Set` — constructs fields needed for blockchain and downstream distribution.
- **Key configuration & expressions:**
  - `documentHash` = `={{ $json.text.substring(0, 64) }}`
    - Note: this is **not a cryptographic hash**; it’s the first 64 chars of extracted text.
  - `timestamp` = `={{ $now.toISO() }}`
  - `validationStatus` = `VERIFIED`
  - `blockchainRecord` (object) =
    ```js
    {{
      {
        policyId: $json.policyId || "UNKNOWN",
        documentHash: $json.documentHash,
        timestamp: $json.timestamp,
        status: "VERIFIED",
        confidenceScore: $json.confidenceScore,
        riskLevel: $json.riskLevel
      }
    }}
    ```
  - Includes other fields (keeps AI outputs too)
- **Input/Output:**
  - Input: True branch from **Check Document Authenticity**
  - Output: **Publish to Blockchain**
- **Edge cases / failures:**
  - If `text` shorter than 64 chars, substring still works but yields short “hash”.
  - `policyId` must be provided earlier (e.g., webhook body) or becomes `UNKNOWN`.
  - If blockchain requires a real hash, you should replace with SHA-256 node/crypto.

#### Node: Publish to Blockchain
- **Type / role:** `HTTP Request` — publishes record to blockchain gateway/API.
- **Key configuration:**
  - **URL:** `={{ $('Workflow Configuration').first().json.blockchainApiUrl }}`
  - **Method:** POST
  - **Body (JSON):** `={{ $json.blockchainRecord }}`
  - Header: `Content-Type: application/json`
- **Input/Output:**
  - Input: from **Prepare Blockchain Record**
  - Output: to **Send Proof to Carriers**
- **Downstream data expectations:**
  - Later nodes reference `transactionId` or `txId`, and `proofUrl`; this node’s response should provide at least one of those fields.
- **Edge cases / failures:**
  - Missing auth headers/credentials for blockchain API (not configured here).
  - Non-2xx responses; consider “Continue On Fail” or explicit error handling if needed.
  - Response shape mismatch → later nodes can’t find tx id/proof URL.

#### Node: Send Proof to Carriers
- **Type / role:** `HTTP Request` — notifies insurance carriers with proof references.
- **Key configuration:**
  - **URL:** `={{ $('Workflow Configuration').first().json.carriersApiUrl }}`
  - **Method:** POST
  - **Body:**
    - `documentHash`, `timestamp`, `status: "VERIFIED"`
    - `blockchainTxId: $json.transactionId || $json.txId`
    - `proofUrl: $json.proofUrl || ""`
  - Header: `Content-Type: application/json`
- **Input/Output:**
  - Input: from **Publish to Blockchain**
  - Output: to **Send Proof to Regulatory Bodies**
- **Edge cases / failures:**
  - If blockchain response doesn’t include `transactionId` or `txId`, carriers receive null/undefined.
  - Carrier endpoint auth, rate limits, schema requirements not handled.

#### Node: Send Proof to Regulatory Bodies
- **Type / role:** `HTTP Request` — sends proof plus a compliance report summary.
- **Key configuration:**
  - **URL:** `={{ $('Workflow Configuration').first().json.regulatoryApiUrl }}`
  - **Method:** POST
  - **Body:**
    - Same proof fields as carriers
    - `complianceReport`: { `confidenceScore`, `riskLevel`, `recommendation` }
- **Input/Output:**
  - Input: from **Send Proof to Carriers**
  - Output: none (end of verified branch)
- **Edge cases / failures:**
  - Regulatory endpoints often require strong authentication/audit headers; not included.
  - Potential PII/regulated data concerns; ensure payload is appropriate.

---

### Block 6 — Suspicious Document: Alerting

**Overview:**  
Formats an alert payload (type, timestamp, message) and emails the compliance team with risk details and findings.

**Nodes involved:**
- Flag Forged Document
- Send Alert Email

#### Node: Flag Forged Document
- **Type / role:** `Set` — constructs alert fields and a readable message.
- **Key configuration & expressions:**
  - `alertType`: `FORGED_DOCUMENT_DETECTED`
  - `timestamp`: `={{ $now.toISO() }}`
  - `alertMessage`:
    ```
    ALERT: Forged or altered document detected. Risk Level: {{ $json.riskLevel }}.
    Confidence: {{ $json.confidenceScore }}%.
    Findings: {{ $json.findings.join(", ") }}
    ```
- **Input/Output:**
  - Input: False branch from **Check Document Authenticity**
  - Output: to **Send Alert Email**
- **Edge cases / failures:**
  - If `findings` is not an array (schema violation), `.join()` will fail.
  - If AI result is missing fields, message formatting fails.

#### Node: Send Alert Email
- **Type / role:** `Gmail` (n8n-nodes-base.gmail) — sends fraud alert email.
- **Key configuration:**
  - **To:** `={{ $('Workflow Configuration').first().json.alertEmail }}`
  - **Subject:** `🚨 FRAUD ALERT: Forged Document Detected`
  - **HTML message:** uses fields `alertType`, `timestamp`, `riskLevel`, `confidenceScore`, `recommendation`, and renders findings into `<li>` items via:
    - `{{ $json.findings.map(f => "<li>" + f + "</li>").join("") }}`
- **Credentials:** Gmail OAuth2 (`Gmail account`)
- **Input/Output:**
  - Input: from **Flag Forged Document**
  - Output: none (end of forged branch)
- **Edge cases / failures:**
  - Gmail OAuth token expiry/consent issues.
  - Sending limits / restricted scopes.
  - If `alertEmail` is empty or invalid, send fails.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Document Submission Webhook | Webhook | Entry point for document upload | (External) | Workflow Configuration | ## **Document Submission & Analysis** / **What:** Webhook receives documents; An Anthropic Chat Model with output parser checks authenticity  / **Why:** AI detects sophisticated forgeries beyond manual review |
| Workflow Configuration | Set | Stores API URLs + alert email; keeps original fields | Document Submission Webhook | Extract Document Content | ## **Document Submission & Analysis** / **What:** Webhook receives documents; An Anthropic Chat Model with output parser checks authenticity  / **Why:** AI detects sophisticated forgeries beyond manual review |
| Extract Document Content | Extract From File | Extracts PDF text into `text` | Workflow Configuration | Document Authenticity Validator | ## **Document Submission & Analysis** / **What:** Webhook receives documents; An Anthropic Chat Model with output parser checks authenticity  / **Why:** AI detects sophisticated forgeries beyond manual review |
| Document Authenticity Validator | LangChain Agent | Runs authenticity analysis prompt, returns structured result | Extract Document Content + (AI model/parser) | Check Document Authenticity | ## **Document Submission & Analysis** / **What:** Webhook receives documents; An Anthropic Chat Model with output parser checks authenticity  / **Why:** AI detects sophisticated forgeries beyond manual review |
| Anthropic Chat Model | LangChain Anthropic Chat Model | Provides Claude model for the agent | (Agent connection) | Document Authenticity Validator (ai_languageModel) | ## **Document Submission & Analysis** / **What:** Webhook receives documents; An Anthropic Chat Model with output parser checks authenticity  / **Why:** AI detects sophisticated forgeries beyond manual review |
| Validation Result Parser | Structured Output Parser | Enforces JSON schema of AI output | (Agent connection) | Document Authenticity Validator (ai_outputParser) | ## **Document Submission & Analysis** / **What:** Webhook receives documents; An Anthropic Chat Model with output parser checks authenticity  / **Why:** AI detects sophisticated forgeries beyond manual review |
| Check Document Authenticity | IF | Routes authentic vs forged | Document Authenticity Validator | Prepare Blockchain Record (true); Flag Forged Document (false) | ## **Blockchain Verification** / **What:** Authentic documents pass an IF node to a blockchain workflow  / **Why:** Blockchain ensures tamper-proof verification for compliance |
| Prepare Blockchain Record | Set | Builds `blockchainRecord`, timestamp, “hash” | Check Document Authenticity (true) | Publish to Blockchain | ## **Blockchain Verification** / **What:** Authentic documents pass an IF node to a blockchain workflow  / **Why:** Blockchain ensures tamper-proof verification for compliance |
| Publish to Blockchain | HTTP Request | Posts record to blockchain API | Prepare Blockchain Record | Send Proof to Carriers | ## **Blockchain Verification** / **What:** Authentic documents pass an IF node to a blockchain workflow  / **Why:** Blockchain ensures tamper-proof verification for compliance |
| Send Proof to Carriers | HTTP Request | Sends blockchain proof to carriers endpoint | Publish to Blockchain | Send Proof to Regulatory Bodies | ## **Blockchain Verification** / **What:** Authentic documents pass an IF node to a blockchain workflow  / **Why:** Blockchain ensures tamper-proof verification for compliance |
| Send Proof to Regulatory Bodies | HTTP Request | Sends proof + compliance report to regulators | Send Proof to Carriers | (None) | ## **Blockchain Verification** / **What:** Authentic documents pass an IF node to a blockchain workflow  / **Why:** Blockchain ensures tamper-proof verification for compliance |
| Flag Forged Document | Set | Formats alert payload/message | Check Document Authenticity (false) | Send Alert Email | ## **Suspicious Document Handling** / **What:** Flagged documents trigger Gmail alerts  / **Why:** Human review safeguards security and prevents false rejections . |
| Send Alert Email | Gmail | Emails compliance fraud alert | Flag Forged Document | (None) | ## **Suspicious Document Handling** / **What:** Flagged documents trigger Gmail alerts  / **Why:** Human review safeguards security and prevents false rejections . |
| Sticky Note | Sticky Note | Comment | (None) | (None) | ## Prerequisites / Anthropic API key, blockchain network access and credentials  / ## Use Cases / Supply chain documentation verification for import/export compliance  / ## Customization / Adjust AI prompts for industry-specific authenticity criteria  / ## Benefits / Eliminates manual document review time while improving fraud detection accuracy |
| Sticky Note1 | Sticky Note | Comment | (None) | (None) | ## Setup Steps / 1. Configure webhook endpoint URL for document submission / 2. Add Anthropic API key to Chat Model node for AI / 3. Set up blockchain network credentials in HTTP nodes for record preparation / 4. Connect Gmail account and specify compliance team email addresses / 5. Customize authenticity thresholds |
| Sticky Note2 | Sticky Note | Comment | (None) | (None) | ## How It Works / This workflow automates document authenticity verification by combining AI-based content analysis with immutable blockchain records... (full note content) |
| Sticky Note3 | Sticky Note | Comment | (None) | (None) | ## **Document Submission & Analysis** / **What:** Webhook receives documents; An Anthropic Chat Model with output parser checks authenticity / **Why:** AI detects sophisticated forgeries beyond manual review |
| Sticky Note4 | Sticky Note | Comment | (None) | (None) | ## **Blockchain Verification** / **What:** Authentic documents pass an IF node to a blockchain workflow / **Why:** Blockchain ensures tamper-proof verification for compliance |
| Sticky Note5 | Sticky Note | Comment | (None) | (None) | ## **Suspicious Document Handling** / **What:** Flagged documents trigger Gmail alerts / **Why:** Human review safeguards security and prevents false rejections . |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Webhook node** named **“Document Submission Webhook”**
   - Method: **POST**
   - Path: **document-submission**
   - Response mode: **Last Node**
   - Ensure your client uploads a **PDF file as binary** (commonly `binary.data`) and optionally JSON fields like `policyId`.
3. **Add Set node** named **“Workflow Configuration”**
   - Add fields (String):
     - `blockchainApiUrl` = your blockchain gateway endpoint (e.g., `https://.../publish`)
     - `carriersApiUrl` = carriers endpoint
     - `regulatoryApiUrl` = regulators endpoint
     - `alertEmail` = compliance/fraud mailbox
   - Enable **Include Other Fields**
   - Connect: **Webhook → Workflow Configuration**
4. **Add Extract From File node** named **“Extract Document Content”**
   - Operation: **PDF**
   - Connect: **Workflow Configuration → Extract Document Content**
   - If needed, configure which binary property to read (depends on your webhook upload form-field name).
5. **Add Anthropic Chat Model node** named **“Anthropic Chat Model”**
   - Select model: **claude-sonnet-4-5-20250929** (or closest available)
   - Configure **Anthropic API credentials** (API key) in n8n Credentials and select it in the node.
6. **Add Structured Output Parser node** named **“Validation Result Parser”**
   - Schema type: **Manual**
   - Paste a schema requiring:
     - `isAuthentic` boolean
     - `confidenceScore` number
     - `findings` string array
     - `riskLevel` enum: low/medium/high/critical
     - `recommendation` enum: approve/review/reject
7. **Add LangChain Agent node** named **“Document Authenticity Validator”**
   - Text input: set expression to the extracted text, e.g. `{{ $json.text }}`
   - Provide a **system message** instructing authenticity checks and requiring the structured JSON fields.
   - Enable output parsing and connect:
     - **Anthropic Chat Model → Document Authenticity Validator** (AI Language Model connection)
     - **Validation Result Parser → Document Authenticity Validator** (AI Output Parser connection)
   - Connect main flow: **Extract Document Content → Document Authenticity Validator**
8. **Add IF node** named **“Check Document Authenticity”**
   - Condition: boolean **is true**
   - Left value expression: `{{ $('Document Authenticity Validator').item.json.isAuthentic }}`
   - Connect: **Document Authenticity Validator → Check Document Authenticity**
9. **Verified path (True branch): Add Set node** named **“Prepare Blockchain Record”**
   - Enable **Include Other Fields**
   - Add fields:
     - `documentHash` = `{{ $json.text.substring(0, 64) }}` (recommended to replace with real hash later)
     - `timestamp` = `{{ $now.toISO() }}`
     - `validationStatus` = `VERIFIED`
     - `blockchainRecord` (Object) with:
       - `policyId` = `{{ $json.policyId || "UNKNOWN" }}`
       - `documentHash` = `{{ $json.documentHash }}`
       - `timestamp` = `{{ $json.timestamp }}`
       - `status` = `VERIFIED`
       - `confidenceScore` = `{{ $json.confidenceScore }}`
       - `riskLevel` = `{{ $json.riskLevel }}`
   - Connect: **IF (true) → Prepare Blockchain Record**
10. **Add HTTP Request node** named **“Publish to Blockchain”**
    - Method: **POST**
    - URL: `{{ $('Workflow Configuration').first().json.blockchainApiUrl }}`
    - Send JSON body: `{{ $json.blockchainRecord }}`
    - Header: `Content-Type: application/json`
    - Add authentication (API key/OAuth/custom headers) as required by your blockchain API.
    - Connect: **Prepare Blockchain Record → Publish to Blockchain**
11. **Add HTTP Request node** named **“Send Proof to Carriers”**
    - Method: POST
    - URL: `{{ $('Workflow Configuration').first().json.carriersApiUrl }}`
    - JSON body:
      - `documentHash`, `timestamp`, `status: "VERIFIED"`
      - `blockchainTxId: {{ $json.transactionId || $json.txId }}`
      - `proofUrl: {{ $json.proofUrl || "" }}`
    - Header: `Content-Type: application/json`
    - Connect: **Publish to Blockchain → Send Proof to Carriers**
12. **Add HTTP Request node** named **“Send Proof to Regulatory Bodies”**
    - Method: POST
    - URL: `{{ $('Workflow Configuration').first().json.regulatoryApiUrl }}`
    - JSON body: same as carriers plus:
      - `complianceReport`: `{ confidenceScore, riskLevel, recommendation }`
    - Header: `Content-Type: application/json`
    - Connect: **Send Proof to Carriers → Send Proof to Regulatory Bodies**
13. **Forged path (False branch): Add Set node** named **“Flag Forged Document”**
    - Enable **Include Other Fields**
    - Add fields:
      - `alertType` = `FORGED_DOCUMENT_DETECTED`
      - `timestamp` = `{{ $now.toISO() }}`
      - `alertMessage` = build string using `riskLevel`, `confidenceScore`, and `findings.join(", ")`
    - Connect: **IF (false) → Flag Forged Document**
14. **Add Gmail node** named **“Send Alert Email”**
    - Operation: Send email
    - To: `{{ $('Workflow Configuration').first().json.alertEmail }}`
    - Subject: `🚨 FRAUD ALERT: Forged Document Detected`
    - Body: HTML that includes findings list via `findings.map(...).join("")`
    - Configure **Gmail OAuth2 credentials** in n8n and select them here.
    - Connect: **Flag Forged Document → Send Alert Email**
15. **Activate workflow** and test:
    - POST a PDF to `/webhook/document-submission`
    - Confirm the AI output parses successfully and routes correctly.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Anthropic API key, blockchain network access and credentials | Prerequisites (Sticky Note) |
| Supply chain documentation verification for import/export compliance | Use Cases (Sticky Note) |
| Adjust AI prompts for industry-specific authenticity criteria | Customization (Sticky Note) |
| Eliminates manual document review time while improving fraud detection accuracy | Benefits (Sticky Note) |
| Setup Steps: configure webhook, add Anthropic key, set blockchain credentials in HTTP nodes, connect Gmail, customize thresholds | Setup guidance (Sticky Note1) |
| Workflow explanation: AI-based authenticity + immutable blockchain record + proof distribution | How it works (Sticky Note2) |
| Document Submission & Analysis block rationale | Sticky Note3 |
| Blockchain Verification block rationale | Sticky Note4 |
| Suspicious Document Handling block rationale | Sticky Note5 |