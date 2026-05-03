Verify property ownership with blockchain, GPT-4 fraud detection, and compliance tracking

https://n8nworkflows.xyz/workflows/verify-property-ownership-with-blockchain--gpt-4-fraud-detection--and-compliance-tracking-12036


# Verify property ownership with blockchain, GPT-4 fraud detection, and compliance tracking

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow registers property-related transactions (property registration, lease agreements, and transfers) by generating cryptographic hashes, performing **AI-based fraud detection** and **AI-based compliance validation**, writing an immutable proof to a **blockchain API**, and storing/auditing records in **n8n Data Tables** and **Google Sheets**. It also exposes a separate **verification API webhook** to validate ownership/transaction proof later.

**Target use cases:**  
Real estate firms, legal practices, property management companies needing fraud mitigation, compliance checks, tamper-resistant records, and audit trails.

### 1.1 Intake & Configuration
- Collect transaction details through an n8n Form.
- Centralize environment configuration (API URL/key, DataTable name).

### 1.2 AI Fraud Assessment (Pre-processing Gate)
- GPT-based structured fraud scoring and risk factors.
- If high risk, mark the record for manual review (but still proceeds).

### 1.3 AI Compliance Validation (Pre-processing)
- GPT-based structured compliance evaluation (compliant, violations, missing fields).
- Feeds into transaction routing.

### 1.4 Transaction Type Routing & Hash Generation
- Decide whether to produce a “property hash” vs “lease hash”.
- Create timestamped SHA-256 hash for immutability.

### 1.5 Blockchain Registration + Storage + Audit Log
- Send hash + metadata to blockchain API.
- Store enriched record in n8n Data Table.
- Append/update an audit entry in Google Sheets.

### 1.6 Verification API (Ownership/Proof Verification)
- External POST endpoint to verify a propertyId.
- Queries stored ledger records, calls blockchain verification endpoint, returns JSON.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Intake & Workflow Configuration
**Overview:** Receives property transaction data and injects configuration needed downstream (blockchain endpoint/key, target table name).  
**Nodes involved:** `Property Registration Form`, `Workflow Configuration`

#### Node: Property Registration Form
- **Type / Role:** `Form Trigger` — entry point for submission-based intake.
- **Configuration (interpreted):**
  - Form title: *Blockchain Property Registration*
  - Description: registers properties/leases/transactions for immutable proof.
  - Fields:
    - `transactionType` dropdown: Property Registration / Lease Agreement / Property Transfer
    - `propertyId`, `ownerName`, `ownerAddress`, `propertyValue` (number), `location`, `legalDescription`, `additionalDetails` (textarea)
  - Response text: confirms blockchain registration and mentions transaction hash.
  - Attribution disabled.
- **Key variables:** outputs the submitted fields directly in `$json`.
- **Connections:** `Property Registration Form` → `Workflow Configuration`
- **Version notes:** Form Trigger `typeVersion 2.4`.
- **Edge cases / failures:**
  - Missing required fields: depends on form field required flags (not specified); downstream nodes assume fields exist.
  - `propertyValue` non-numeric or empty may reduce AI quality and can break validations if you later enforce numeric conditions.

#### Node: Workflow Configuration
- **Type / Role:** `Set` — injects environment constants/placeholders into the item.
- **Configuration (interpreted):**
  - Adds:
    - `blockchainApiUrl` (placeholder)
    - `blockchainApiKey` (placeholder)
    - `dataTableName` = `PropertyLedger`
  - `includeOtherFields: true` keeps all form fields.
- **Key expressions:** none; literal placeholders.
- **Connections:**
  - Input: `Property Registration Form`
  - Outputs: `Check Transaction Type` and `AI Fraud Detection Agent` (fan-out)
- **Version notes:** Set `typeVersion 3.4`.
- **Edge cases / failures:**
  - If placeholders aren’t replaced, downstream HTTP requests and Data Table ops will fail (invalid URL, auth).

**Sticky note context:**  
- “Prerequisites / Use Cases / Customization / Benefits” applies broadly to this block and the overall workflow.

---

### Block 2.2 — AI Fraud Risk Assessment
**Overview:** Uses a LangChain Agent backed by OpenAI chat model to produce a structured fraud risk report; flags high-risk transactions for manual review.  
**Nodes involved:** `OpenAI Chat Model`, `Fraud Risk Output Parser`, `AI Fraud Detection Agent`, `Check Fraud Risk Level`, `Flag High Risk Transaction`

#### Node: OpenAI Chat Model
- **Type / Role:** `lmChatOpenAi` — LLM provider for the fraud agent.
- **Configuration (interpreted):**
  - Model: `gpt-4.1-mini`
  - Built-in tools: none
- **Credentials:** OpenAI API credential required.
- **Connections:** Provides `ai_languageModel` to `AI Fraud Detection Agent`.
- **Version notes:** `typeVersion 1.3`.
- **Edge cases / failures:**
  - Invalid OpenAI credentials, quota/rate limits, model availability changes.
  - Latency/timeouts on large prompts.

#### Node: Fraud Risk Output Parser
- **Type / Role:** `outputParserStructured` — forces JSON output matching a schema.
- **Configuration (interpreted):**
  - Manual JSON schema requiring:
    - `riskLevel` (string: low/medium/high)
    - `riskScore` (number 0–100)
    - `riskFactors` (array of strings)
    - `recommendation` (string)
    - `reasoning` (string)
- **Connections:** Provides `ai_outputParser` to `AI Fraud Detection Agent`.
- **Version notes:** `typeVersion 1.3`.
- **Edge cases / failures:**
  - If the model returns invalid JSON or mismatched schema, parsing fails and stops execution (unless error handling is added).

#### Node: AI Fraud Detection Agent
- **Type / Role:** `LangChain Agent` — produces structured fraud assessment.
- **Configuration (interpreted):**
  - Prompt includes property fields from `$json`.
  - System message defines fraud criteria and required structured output.
  - Output parser enabled (connected).
- **Key expressions/variables:** uses `{{ $json.propertyId }}`, `{{ $json.ownerName }}`, etc.
- **Connections:**
  - Input: `Workflow Configuration`
  - Output: `Check Fraud Risk Level`
  - AI links: `OpenAI Chat Model` + `Fraud Risk Output Parser`
- **Version notes:** `typeVersion 3.1`.
- **Edge cases / failures:**
  - Garbage-in/garbage-out: incomplete owner address/legal description reduces accuracy.
  - Prompt includes `propertyValue`; if missing, LLM may infer incorrectly.

#### Node: Check Fraud Risk Level
- **Type / Role:** `IF` — routes based on fraud risk.
- **Configuration (interpreted):**
  - Condition: `$('AI Fraud Detection Agent').item.json.riskLevel == "high"`
- **Connections (n8n semantics):**
  - **True (high risk):** → `Flag High Risk Transaction`
  - **False (not high):** → `AI Compliance Validator`
- **Version notes:** `typeVersion 2.3`.
- **Edge cases / failures:**
  - If agent output is missing `riskLevel`, the comparison may evaluate false or error depending on execution/data state.

#### Node: Flag High Risk Transaction
- **Type / Role:** `Set` — enrich record with flags for manual review.
- **Configuration (interpreted):**
  - Adds:
    - `fraudRiskFlag` = `HIGH_RISK`
    - `requiresManualReview` = `true`
    - `fraudRiskScore` = `{{$json.riskScore}}`
    - `fraudRiskFactors` = `{{$json.riskFactors}}` (stored as string field here)
  - Keeps other fields.
- **Connections:** → `Check Transaction Type`
- **Version notes:** `typeVersion 3.4`.
- **Edge cases / failures:**
  - `riskFactors` is an array; storing into a “string” field may serialize unexpectedly. Consider storing as JSON/stringify explicitly if needed.

**Sticky note context:**  
- “Assess Fraud Risk … Uses GPT-4 …” applies to this block.

---

### Block 2.3 — AI Compliance Validation
**Overview:** Validates regulatory completeness and flags violations/missing fields; then proceeds to transaction routing and hashing.  
**Nodes involved:** `OpenAI Chat Model for Compliance`, `Compliance Output Parser`, `AI Compliance Validator`

#### Node: OpenAI Chat Model for Compliance
- **Type / Role:** `lmChatOpenAi` — LLM provider for the compliance agent.
- **Configuration (interpreted):**
  - Model: `gpt-4.1-mini`
- **Credentials:** OpenAI API credential required.
- **Connections:** Provides `ai_languageModel` to `AI Compliance Validator`.
- **Version notes:** `typeVersion 1.3`.
- **Edge cases:** same as fraud model (auth, rate limits).

#### Node: Compliance Output Parser
- **Type / Role:** `outputParserStructured` — structured compliance output.
- **Configuration (interpreted):**
  - Uses an example JSON schema (not a strict manual schema definition like the fraud parser):
    - `compliant` (boolean)
    - `complianceScore` (number)
    - `violations` (array)
    - `missingFields` (array)
    - `recommendation` (string)
- **Connections:** Provides `ai_outputParser` to `AI Compliance Validator`.
- **Version notes:** `typeVersion 1.3`.
- **Edge cases / failures:**
  - If model output doesn’t match expected structure, parsing can fail.

#### Node: AI Compliance Validator
- **Type / Role:** `LangChain Agent` — produces compliance assessment.
- **Configuration (interpreted):**
  - Prompt includes key property fields (no “additionalDetails” here).
  - System message lists validation checks and required structured output.
  - Output parser enabled (connected).
- **Connections:**
  - Input: from `Check Fraud Risk Level` (false path)
  - Output: `Check Transaction Type`
  - AI links: `OpenAI Chat Model for Compliance` + `Compliance Output Parser`
- **Version notes:** `typeVersion 3.1`.
- **Edge cases / failures:**
  - Legal requirements vary by jurisdiction; results are advisory unless mapped to real rule logic.
  - Missing field detection depends on the model interpretation, not deterministic validation.

---

### Block 2.4 — Transaction Type Routing & Hash Generation
**Overview:** Routes transactions by type and generates SHA-256 hashes (property vs lease) with a timestamp to prepare for blockchain registration.  
**Nodes involved:** `Check Transaction Type`, `Generate Property Hash`, `Generate Lease Hash`

#### Node: Check Transaction Type
- **Type / Role:** `IF` — selects hashing path.
- **Configuration (interpreted):**
  - Uses OR condition:
    - transactionType == “Property Registration”
    - OR transactionType == “Property Transfer”
  - Left value reads from: `$('Workflow Configuration').item.json.transactionType`
- **Connections:**
  - **True:** → `Generate Property Hash`
  - **False:** → `Generate Lease Hash` (covers Lease Agreement and any other values)
- **Version notes:** `typeVersion 2.3`.
- **Edge cases / failures:**
  - It references `Workflow Configuration` node output rather than the current `$json.transactionType`. If later branches modify `transactionType`, this IF will still use the original value from the config node.
  - If multiple items are processed, cross-item referencing can be risky unless item linking is consistent.

#### Node: Generate Property Hash
- **Type / Role:** `Code` — computes SHA-256 hash for property registration/transfer.
- **Configuration (interpreted):**
  - Node.js `crypto` to hash a JSON string containing:
    - `propertyId, ownerName, ownerAddress, propertyValue, location, legalDescription, transactionType, timestamp`
  - Outputs all original fields +:
    - `propertyHash`
    - `blockchainTimestamp` (ISO string)
- **Connections:** → `Register on Blockchain`
- **Version notes:** Code node `typeVersion 2`.
- **Edge cases / failures:**
  - The hash depends on field ordering in the JSON string produced by `JSON.stringify` of the object literal (stable here because keys are defined in code).
  - Any whitespace/casing changes in input fields changes hash (expected behavior).
  - If a field is `undefined`, it will be omitted from JSON, affecting determinism across clients.

#### Node: Generate Lease Hash
- **Type / Role:** `Code` — computes SHA-256 hash for leases.
- **Configuration (interpreted):**
  - Hashes a `|`-delimited string of:
    - propertyId, ownerName, ownerAddress, location, additionalDetails (lease terms), transactionType, timestamp
  - Outputs all original fields +:
    - `leaseHash`
    - `blockchainTimestamp`
- **Connections:** → `Register on Blockchain`
- **Version notes:** Code node `typeVersion 2`.
- **Edge cases / failures:**
  - Uses `|| ''` defaults: missing values become empty strings, which may hide missing-data issues and produce hashes that look valid.
  - Delimiter collisions: if a field contains `|`, it changes meaning; safer to hash JSON like the property path.

**Sticky note context:**  
- “Validate Transaction Type …” applies to `Check Transaction Type`.  
- “Generate Cryptographic Hashes & Log Compliance Results …” applies to both hash nodes and downstream registration/storage.

---

### Block 2.5 — Blockchain Registration, Storage, and Audit Logging
**Overview:** Sends the hash to a blockchain API, stores records in Data Tables, and logs an auditable row in Google Sheets.  
**Nodes involved:** `Register on Blockchain`, `Store Property Records`, `Log to Audit Sheet`

#### Node: Register on Blockchain
- **Type / Role:** `HTTP Request` — registers the transaction hash on a blockchain service.
- **Configuration (interpreted):**
  - Method: POST
  - URL: `$('Workflow Configuration').first().json.blockchainApiUrl`
  - Headers:
    - `Authorization: Bearer <blockchainApiKey>`
    - `Content-Type: application/json`
  - Body parameters:
    - `hash`: `{{$json.propertyHash || $json.leaseHash}}`
    - `transactionType`, `propertyId`, `ownerName`
    - `timestamp`: **uses `{{$json.timestamp}}`**
  - Response: JSON expected.
- **Connections:** → `Store Property Records`
- **Version notes:** HTTP node `typeVersion 4.3`.
- **Edge cases / failures:**
  - **Bug/field mismatch:** upstream code nodes create `blockchainTimestamp`, not `timestamp`. This body may send `timestamp: null/undefined` unless another node provides `timestamp`.
  - API errors: 401 (bad key), 404 (bad URL), 429/5xx, non-JSON responses (parsing failure).
  - If blockchain API expects specific fields (e.g., `propertyHash`), your generic `hash` may not match.

#### Node: Store Property Records
- **Type / Role:** `Data Table` — persists ledger entries in n8n’s Data Tables.
- **Configuration (interpreted):**
  - Operation: (implied create/insert) with defined column mapping.
  - DataTable ID/Name: from `Workflow Configuration.dataTableName` (set to `PropertyLedger`).
  - Stores (examples):
    - `propertyHash`: from `propertyHash || leaseHash`
    - `blockchainTimestamp`: from `$('Register on Blockchain').item.json.timestamp`
    - `blockchainTransactionId`: from `$('Register on Blockchain').item.json.transactionId`
    - plus property fields, additionalDetails, legalDescription, status, etc.
- **Connections:** → `Log to Audit Sheet`
- **Version notes:** `typeVersion 1.1`.
- **Edge cases / failures:**
  - If Data Table schema doesn’t contain these columns, insert may fail or drop fields depending on n8n Data Tables behavior.
  - Uses blockchain response `timestamp`—ensure the blockchain API actually returns `timestamp`. If it returns `blockchainTimestamp` instead, you’ll store blank.
  - If `status` is not produced by prior nodes, it will be empty unless blockchain response is merged (it isn’t explicitly merged here).

#### Node: Log to Audit Sheet
- **Type / Role:** `Google Sheets` — append/update audit log for compliance and traceability.
- **Configuration (interpreted):**
  - Operation: `appendOrUpdate`
  - Matching column: `Property ID`
  - Sheet/document: placeholders for Spreadsheet ID + sheet name.
  - Writes columns:
    - `Status`: `{{$json.status}}`
    - `Timestamp`: `{{$now.toISO()}}`
    - `Owner Name`, `Property ID`, `Transaction Type`
    - `Transaction ID`: `{{$json.transactionId}}`
    - `Blockchain Hash`: `{{$json.blockchainHash}}`
- **Credentials:** Google Sheets OAuth2 required.
- **Connections:** none (end of registration flow).
- **Version notes:** `typeVersion 4.7`.
- **Edge cases / failures:**
  - Potential field mismatches: upstream stores `propertyHash` and blockchain may return `transactionId`; but this node expects `$json.blockchainHash` and `$json.transactionId`. Unless the incoming item contains those exact keys, audit columns may be blank.
  - Append/update requires sheet header columns matching configured schema names.
  - OAuth permission issues, spreadsheet sharing, quota limits.

---

### Block 2.6 — Verification API (Webhook → Lookup → Blockchain Verify → Response)
**Overview:** Exposes a POST endpoint to verify a stored property record against blockchain proof, returning a JSON result.  
**Nodes involved:** `Verification API Webhook`, `Query Property Records`, `Verify on Blockchain`, `Return Verification Result`

#### Node: Verification API Webhook
- **Type / Role:** `Webhook` — external API entry point.
- **Configuration (interpreted):**
  - Path: `POST /verify-property`
  - Response mode: `lastNode` (response comes from the last node in the chain)
- **Connections:** → `Query Property Records`
- **Version notes:** `typeVersion 2.1`.
- **Edge cases / failures:**
  - No authentication configured (public endpoint unless protected by n8n or external gateway).
  - Expects `body.propertyId`; missing body will break lookup.

#### Node: Query Property Records
- **Type / Role:** `Data Table` — fetches stored ledger entries for a propertyId.
- **Configuration (interpreted):**
  - Operation: `get`, returnAll: true
  - Filter: `propertyId == {{$json.body.propertyId}}`
  - DataTable ID: `Workflow Configuration.dataTableName`
- **Connections:** → `Verify on Blockchain`
- **Version notes:** `typeVersion 1.1`.
- **Edge cases / failures:**
  - This verification branch does **not** connect to `Workflow Configuration`, yet it references it. If the workflow executes only from this webhook, `$('Workflow Configuration')` may be undefined, causing failures.
  - If multiple records match, downstream will run per item; verification might return multiple responses unless constrained (but Respond node returns once per execution).

#### Node: Verify on Blockchain
- **Type / Role:** `HTTP Request` — calls blockchain verification endpoint.
- **Configuration (interpreted):**
  - URL: `blockchainApiUrl + '/verify/' + $json.blockchainTransactionId`
  - Auth header: Bearer API key from `Workflow Configuration`
  - Response format: JSON
- **Connections:** → `Return Verification Result`
- **Version notes:** `typeVersion 4.3`.
- **Edge cases / failures:**
  - Same configuration dependency issue: relies on `Workflow Configuration` not executed in this branch.
  - If `blockchainTransactionId` is missing/blank, URL becomes invalid.

#### Node: Return Verification Result
- **Type / Role:** `Respond to Webhook` — returns structured JSON.
- **Configuration (interpreted):**
  - Respond with JSON object containing:
    - propertyId, ownerName, transactionType
    - blockchainVerified (boolean)
    - blockchainTransactionId, blockchainTimestamp
    - verificationTimestamp, status
- **Connections:** terminal response.
- **Version notes:** `typeVersion 1.5`.
- **Edge cases / failures:**
  - Values like `blockchainVerified`, `verificationTimestamp` must be provided by upstream nodes; otherwise response contains null/empty.
  - If multiple items, responding can be ambiguous; typically you’d aggregate to one response.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Property Registration Form | n8n-nodes-base.formTrigger | Primary intake form | — | Workflow Configuration | ## How It Works… (overall) |
| Workflow Configuration | n8n-nodes-base.set | Inject blockchain/table config | Property Registration Form | AI Fraud Detection Agent; Check Transaction Type | ## Prerequisites… / ## Setup Steps… |
| AI Fraud Detection Agent | @n8n/n8n-nodes-langchain.agent | Fraud scoring (structured) | Workflow Configuration | Check Fraud Risk Level | ## Assess Fraud Risk… |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for fraud agent | — (AI link) | AI Fraud Detection Agent (ai_languageModel) | ## Assess Fraud Risk… |
| Fraud Risk Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce fraud JSON schema | — (AI link) | AI Fraud Detection Agent (ai_outputParser) | ## Assess Fraud Risk… |
| Check Fraud Risk Level | n8n-nodes-base.if | Branch on high risk | AI Fraud Detection Agent | Flag High Risk Transaction; AI Compliance Validator | ## Assess Fraud Risk… |
| Flag High Risk Transaction | n8n-nodes-base.set | Tag manual review fields | Check Fraud Risk Level (true) | Check Transaction Type | ## Assess Fraud Risk… |
| AI Compliance Validator | @n8n/n8n-nodes-langchain.agent | Compliance assessment (structured) | Check Fraud Risk Level (false) | Check Transaction Type | ## Validate Transaction Type… |
| OpenAI Chat Model for Compliance | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for compliance agent | — (AI link) | AI Compliance Validator (ai_languageModel) | ## Validate Transaction Type… |
| Compliance Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce compliance structure | — (AI link) | AI Compliance Validator (ai_outputParser) | ## Validate Transaction Type… |
| Check Transaction Type | n8n-nodes-base.if | Route property vs lease hashing | Workflow Configuration; AI Compliance Validator; Flag High Risk Transaction | Generate Property Hash; Generate Lease Hash | ## Validate Transaction Type… |
| Generate Property Hash | n8n-nodes-base.code | SHA-256 hash for property/transfer | Check Transaction Type (true) | Register on Blockchain | ## Generate Cryptographic Hashes & Log Compliance Results… |
| Generate Lease Hash | n8n-nodes-base.code | SHA-256 hash for lease | Check Transaction Type (false) | Register on Blockchain | ## Generate Cryptographic Hashes & Log Compliance Results… |
| Register on Blockchain | n8n-nodes-base.httpRequest | Write hash to blockchain API | Generate Property Hash; Generate Lease Hash | Store Property Records | ## Generate Cryptographic Hashes & Log Compliance Results… |
| Store Property Records | n8n-nodes-base.dataTable | Persist ledger record | Register on Blockchain | Log to Audit Sheet | ## Generate Cryptographic Hashes & Log Compliance Results… |
| Log to Audit Sheet | n8n-nodes-base.googleSheets | Append/update audit log | Store Property Records | — | ## Generate Cryptographic Hashes & Log Compliance Results… |
| Verification API Webhook | n8n-nodes-base.webhook | Verification endpoint | — | Query Property Records | ## How It Works… (overall) |
| Query Property Records | n8n-nodes-base.dataTable | Lookup by propertyId | Verification API Webhook | Verify on Blockchain | ## How It Works… (overall) |
| Verify on Blockchain | n8n-nodes-base.httpRequest | Verify transactionId on chain | Query Property Records | Return Verification Result | ## How It Works… (overall) |
| Return Verification Result | n8n-nodes-base.respondToWebhook | Return verification JSON | Verify on Blockchain | — | ## How It Works… (overall) |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation (prereqs/use cases) | — | — | ## Prerequisites… |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation (setup steps) | — | — | ## Setup Steps… |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation (how it works) | — | — | ## How It Works… |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation (hashing/logging) | — | — | ## Generate Cryptographic Hashes & Log Compliance Results… |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation (transaction type validation) | — | — | ## Validate Transaction Type… |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation (fraud assessment) | — | — | ## Assess Fraud Risk… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** (keep it inactive until credentials/placeholders are set).

2) **Add “Property Registration Form” (Form Trigger)**
   - Title: *Blockchain Property Registration*
   - Description: *Register properties, leases, and transactions…*
   - Fields:
     - `transactionType` dropdown: Property Registration / Lease Agreement / Property Transfer
     - `propertyId`, `ownerName`, `ownerAddress`, `propertyValue` (number), `location`, `legalDescription`, `additionalDetails` (textarea)
   - Response text: confirmation message.
   - Connect → next node.

3) **Add “Workflow Configuration” (Set)**
   - Add fields:
     - `blockchainApiUrl` (your blockchain API base URL)
     - `blockchainApiKey` (your API key)
     - `dataTableName` = `PropertyLedger`
   - Enable “Include Other Fields”.
   - Connect from Form → Set.
   - Create two outgoing connections:
     - → `AI Fraud Detection Agent`
     - → `Check Transaction Type` (later it will also be reached via compliance paths)

4) **Fraud AI: add “OpenAI Chat Model” (OpenAI Chat)**
   - Model: `gpt-4.1-mini`
   - Configure **OpenAI credentials** in n8n (API key).
   - This node will be linked to the agent via AI connection (not main).

5) **Fraud AI: add “Fraud Risk Output Parser” (Structured Output Parser)**
   - Use a manual JSON schema with: `riskLevel`, `riskScore`, `riskFactors[]`, `recommendation`, `reasoning`.

6) **Add “AI Fraud Detection Agent” (LangChain Agent)**
   - Prompt text: include the property fields (propertyId, ownerName, ownerAddress, propertyValue, location, transactionType, legalDescription, additionalDetails).
   - System message: fraud criteria + structured output requirements.
   - Connect AI inputs:
     - `OpenAI Chat Model` → agent as **ai_languageModel**
     - `Fraud Risk Output Parser` → agent as **ai_outputParser**
   - Main connection: `Workflow Configuration` → `AI Fraud Detection Agent`.

7) **Add “Check Fraud Risk Level” (IF)**
   - Condition: `riskLevel == "high"` (read from the agent output).
   - Connect: `AI Fraud Detection Agent` → `Check Fraud Risk Level`.

8) **Add “Flag High Risk Transaction” (Set)**
   - Add fields:
     - `fraudRiskFlag` = `HIGH_RISK`
     - `requiresManualReview` = true
     - `fraudRiskScore` = `{{$json.riskScore}}`
     - `fraudRiskFactors` = `{{$json.riskFactors}}`
   - Include other fields.
   - Connect from IF **true** → this Set.
   - Connect this Set → `Check Transaction Type`.

9) **Compliance AI: add “OpenAI Chat Model for Compliance”**
   - Model: `gpt-4.1-mini`
   - Reuse same OpenAI credential.

10) **Compliance AI: add “Compliance Output Parser”**
   - Configure structured output expectation (compliant boolean, score, violations, missingFields, recommendation). (Using an example schema is acceptable, but a strict schema is safer.)

11) **Add “AI Compliance Validator” (LangChain Agent)**
   - Prompt: property fields (propertyId, ownerName, ownerAddress, propertyValue, location, transactionType, legalDescription).
   - System message: compliance checks + structured output requirements.
   - Connect AI inputs:
     - Compliance chat model → agent as **ai_languageModel**
     - Compliance parser → agent as **ai_outputParser**
   - Connect IF **false** (not high risk) → `AI Compliance Validator`
   - Connect `AI Compliance Validator` → `Check Transaction Type`

12) **Add “Check Transaction Type” (IF)**
   - Condition (OR):
     - transactionType == “Property Registration”
     - OR transactionType == “Property Transfer”
   - Preferred: reference the current item: `{{$json.transactionType}}` (safer than cross-node reference).
   - Connect:
     - True → `Generate Property Hash`
     - False → `Generate Lease Hash`

13) **Add “Generate Property Hash” (Code)**
   - Use Node.js crypto SHA-256.
   - Produce: `propertyHash` and `blockchainTimestamp` ISO string.
   - Connect True path → this node → `Register on Blockchain`.

14) **Add “Generate Lease Hash” (Code)**
   - Hash lease-specific concatenated string or (recommended) JSON.
   - Produce: `leaseHash` and `blockchainTimestamp`.
   - Connect False path → this node → `Register on Blockchain`.

15) **Add “Register on Blockchain” (HTTP Request)**
   - POST to `{{$json.blockchainApiUrl}}` (or from config node)
   - Headers:
     - Authorization: `Bearer <apiKey>`
     - Content-Type: application/json
   - JSON body:
     - `hash`: `{{$json.propertyHash || $json.leaseHash}}`
     - `transactionType`, `propertyId`, `ownerName`
     - **timestamp:** use `{{$json.blockchainTimestamp}}` (recommended fix)
   - Expect JSON response (transactionId, timestamp, status, verified, etc. depending on your API).
   - Connect → `Store Property Records`.

16) **Add “Store Property Records” (Data Table)**
   - Create Data Table named `PropertyLedger` in n8n (or match your chosen name).
   - Map columns to store:
     - identifiers + owner + location + values + hash
     - AI outputs if desired (fraud/compliance)
     - blockchain response values: `transactionId`, blockchain timestamp
   - Connect → `Log to Audit Sheet`.

17) **Add “Log to Audit Sheet” (Google Sheets)**
   - Create a Google Sheet with headers matching your mappings.
   - Configure credentials via OAuth2.
   - Operation: Append or Update; match on `Property ID`.
   - Map relevant columns (ensure field names actually exist; consider using `propertyHash` instead of `blockchainHash` unless your blockchain API returns that field).

18) **Add Verification Flow: “Verification API Webhook”**
   - Webhook path: `verify-property`, method POST.
   - Connect → `Query Property Records`.

19) **Add “Query Property Records” (Data Table get)**
   - Filter: `propertyId == {{$json.body.propertyId}}`
   - Return all (or limit to most recent record, recommended).
   - Connect → `Verify on Blockchain`.

20) **Add “Verify on Blockchain” (HTTP Request)**
   - GET (or default) to: `{blockchainApiUrl}/verify/{blockchainTransactionId}`
   - Add Authorization header.
   - Connect → `Return Verification Result`.

21) **Add “Return Verification Result” (Respond to Webhook)**
   - Respond with JSON including propertyId, ownerName, transactionType, blockchainVerified, blockchainTransactionId, blockchainTimestamp, verificationTimestamp, status.

22) **Hardening required for this workflow to work as intended (recommended)**
   - Ensure the verification branch has access to blockchainApiUrl/apiKey (either:
     - duplicate a config Set node in the verification branch, or
     - move config values to environment variables/credentials and reference them directly).
   - Align timestamp fields (`blockchainTimestamp` vs `timestamp`) consistently across nodes.
   - Align audit fields (`propertyHash` vs `blockchainHash`, `transactionId` location).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Prerequisites:** Property registration data source; OpenAI API key; blockchain network access | From sticky note “Prerequisites” |
| **Use Cases:** Real estate firms automating fraud checks on property transactions | From sticky note “Use Cases” |
| **Customization:** Adjust fraud detection criteria and risk thresholds, modify blockchain network selection | From sticky note “Customization” |
| **Benefits:** Eliminates manual fraud detection, prevents title fraud and forgery | From sticky note “Benefits” |
| **Setup Steps:** Configure data source + OpenAI; connect blockchain credentials; set Google Sheets audit logging; define thresholds/compliance criteria | From sticky note “Setup Steps” |
| **How It Works:** End-to-end description of ingestion → GPT analysis → hashing → blockchain logging → audit storage → verification | From sticky note “How It Works” |
| **Generate Cryptographic Hashes & Log Compliance Results:** Creates property/lease hashes for immutable records | From sticky note “Generate Cryptographic Hashes…” |
| **Validate Transaction Type:** Checks transactions against compliance/legitimacy criteria | From sticky note “Validate Transaction Type” |
| **Assess Fraud Risk:** GPT-based fraud scoring prior to processing | From sticky note “Assess Fraud Risk” |