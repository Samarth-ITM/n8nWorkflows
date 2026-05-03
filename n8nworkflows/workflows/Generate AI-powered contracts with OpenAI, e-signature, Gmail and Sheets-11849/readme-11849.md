Generate AI-powered contracts with OpenAI, e-signature, Gmail and Sheets

https://n8nworkflows.xyz/workflows/generate-ai-powered-contracts-with-openai--e-signature--gmail-and-sheets-11849


# Generate AI-powered contracts with OpenAI, e-signature, Gmail and Sheets

## 1. Workflow Overview

**Title:** Contract Template Generator with E-Signature Integration  
**Purpose:** Receive a contract request via webhook, validate it, generate a contract draft using an OpenAI-powered agent (Markdown), convert it to HTML, send it to an e-signature provider, wait for signature completion via webhook resume, then log and notify parties via Google Sheets and Gmail.

**Target use cases**
- Automated NDA / Service Agreement / Employment Contract generation from structured request data
- Multi-party signature collection (sequential order supported via signer “order” field)
- Automated status tracking (SIGNED vs EXPIRED/PENDING), logging, and email notifications

### 1.1 Input Reception & Validation
Receives the request, checks required fields, prepares normalized data and a unique contract ID.

### 1.2 Template Routing & AI Draft Generation
Selects a base template by `contractType`, merges base terms, and uses an AI agent + OpenAI chat model to generate a Markdown contract and metadata (suggested clauses, risk assessment).

### 1.3 Document Formatting & E-Signature Initiation
Converts Markdown → HTML, builds signer payload, sends an envelope to an e-signature API, emails signers, then waits (up to 7 days) for signature completion via resume webhook.

### 1.4 Completion Handling (Logging + Notifications) & Webhook Response
Interprets the resume webhook result, branches on signature status, logs to Google Sheets, sends appropriate emails, merges final branch outputs, and responds to the original webhook request.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Validation
**Overview:** Accepts contract requests through an HTTP webhook, validates the presence of required fields, and normalizes input into a consistent internal schema including a generated `contractId`.

**Nodes involved:**
- Contract Request Webhook
- Validate Contract Request
- Prepare Contract Data
- Invalid Request Response
- Respond to Webhook (also used later)

#### Node: Contract Request Webhook
- **Type/role:** `Webhook` (entry point), receives POST requests.
- **Key config:**
  - **Path:** `contract-request`
  - **HTTP Method:** `POST`
  - **Response mode:** `responseNode` (the workflow must end with **Respond to Webhook** to return a response).
- **Inputs/Outputs:** Entry node → outputs to **Validate Contract Request**.
- **Edge cases/failures:**
  - If callers don’t send JSON body or expected structure (e.g., missing `body`), downstream expressions like `$json.body.contractType` may resolve to `undefined`.
  - Webhook requests can time out if you wait too long before responding (note: this workflow waits for signature, which is typically longer than a webhook client will hold—see “Edge cases” under Wait node).

#### Node: Validate Contract Request
- **Type/role:** `If` node, validates required fields.
- **Key config (interpreted):**
  - Condition 1: `contractType` exists: `{{ $json.body.contractType }}`
  - Condition 2: `parties` array is not empty: `{{ $json.body.parties }}`
  - Loose type validation.
- **Routing:**
  - **True** → **Prepare Contract Data**
  - **False** → **Invalid Request Response**
- **Edge cases/failures:**
  - If `body` is missing entirely, both checks fail and the workflow returns the “invalid request” response (good default behavior).

#### Node: Prepare Contract Data
- **Type/role:** `Set` node, normalizes request payload and generates IDs/timestamps.
- **Key config (fields created):**
  - `contractId`: `CTR-<yyyyMMdd>-<random 6 chars>` via `$now.format(...)` + `Math.random()...`
  - `contractType`, `parties` from request body
  - Defaults:
    - `customTerms`: `''`
    - `effectiveDate`: today (`yyyy-MM-dd`)
    - `expirationDate`: `''`
    - `value`: `0`
    - `currency`: `USD`
  - `requestedAt`: `$now.toISO()`
- **Outputs:** to **Route by Contract Type**
- **Edge cases/failures:**
  - `parties` is stored as **object** type in the Set node, but downstream expects an array (it is used as array later in “Prepare Signature Request”). If `parties` isn’t an array, signer generation will fail or produce wrong results.
  - Random ID collisions are unlikely but possible; not checked against any datastore.

#### Node: Invalid Request Response
- **Type/role:** `Set` node, constructs an error response payload.
- **Key output fields:** `error: true`, `message: "Missing required fields..."`.
- **Outputs:** to **Respond to Webhook**
- **Edge cases/failures:** None significant.

---

### Block 2 — Template Routing & AI Draft Generation
**Overview:** Chooses a base set of terms based on `contractType`, merges that with the prepared contract data, then calls an AI agent to generate a Markdown contract and additional structured metadata.

**Nodes involved:**
- Route by Contract Type
- NDA Template Base
- Service Agreement Base
- Employment Contract Base
- Generic Contract Base
- Merge Template Data
- AI Contract Advisor
- OpenAI Chat Model
- Parse AI Contract Response

#### Node: Route by Contract Type
- **Type/role:** `Switch` node, routes by contract type.
- **Key config:**
  - Rules:
    - `contractType == "NDA"` → NDA Template Base
    - `contractType == "SERVICE"` → Service Agreement Base
    - `contractType == "EMPLOYMENT"` → Employment Contract Base
  - **Fallback output:** `extra` → Generic Contract Base
- **Inputs/Outputs:** From **Prepare Contract Data** → one of 4 template nodes.
- **Edge cases/failures:**
  - Case sensitivity matters (`NDA` vs `nda`), so unexpected values will fall back to generic.
  - If `contractType` is missing, it likely falls back to generic (depending on Switch evaluation).

#### Node: NDA Template Base / Service Agreement Base / Employment Contract Base / Generic Contract Base
- **Type/role:** `Set` nodes, define template metadata.
- **Key config:**
  - Each sets:
    - `templateType` (human-friendly)
    - `baseTerms` (string summary)
- **Outputs:** All route into **Merge Template Data**
- **Edge cases/failures:** None.

#### Node: Merge Template Data
- **Type/role:** `Merge` node in “chooseBranch” mode (passes through data from the chosen branch).
- **Inputs:** One of the template base nodes.
- **Outputs:** to **AI Contract Advisor**
- **Important behavior:** In this design, the merge node is effectively a pass-through from the selected template branch; it does not combine two streams here.
- **Edge cases:** If multiple inputs arrived (unexpected), “chooseBranch” behavior may be non-intuitive; but Switch ensures only one branch typically runs.

#### Node: AI Contract Advisor
- **Type/role:** `LangChain Agent` node (`@n8n/n8n-nodes-langchain.agent`) orchestrating LLM call and returning an `output`.
- **Key config:**
  - Prompt includes:
    - Contract Type, Template, Base Terms from current branch
    - `customTerms`, `parties`, dates, value/currency from **Prepare Contract Data** via node referencing:
      - `$('Prepare Contract Data').first().json...`
    - Requires **ONLY valid JSON** with keys: `contractMarkdown`, `suggestedClauses[]`, `riskAssessment`.
  - System message enforces JSON-only output.
- **Model connection:** Uses **OpenAI Chat Model** via `ai_languageModel` input.
- **Outputs:** to **Parse AI Contract Response**
- **Version-specific requirements:** Requires n8n’s LangChain nodes (`typeVersion` 1.7).
- **Edge cases/failures:**
  - Model may still output non-JSON or wrap JSON in text; downstream parsing attempts to recover.
  - Large parties arrays or extensive custom terms could exceed token limits (model-dependent).
  - Legal content generation may require jurisdiction-specific constraints not supplied (governing law is requested but not parameterized).

#### Node: OpenAI Chat Model
- **Type/role:** `lmChatOpenAi` providing the LLM backend.
- **Key config:**
  - Model: `gpt-4o-mini`
  - Temperature: `0.3` (more deterministic)
- **Credentials:** OpenAI API credential required in n8n.
- **Outputs:** Feeds the agent node (not a standard main connection).
- **Edge cases/failures:**
  - Auth/quotas/rate limits.
  - Model availability changes.

#### Node: Parse AI Contract Response
- **Type/role:** `Code` node, extracts JSON from agent output and merges with contract data.
- **Key logic:**
  - Reads:
    - `contractData` from **Prepare Contract Data**
    - `aiResponse` from incoming item
  - Attempts:
    - `aiResponse.output` → regex match for `{ ... }` → `JSON.parse`
  - Fallback default content on parse error:
    - Basic contract markdown + 3 suggested clauses + medium risk assessment.
  - Outputs a combined object including:
    - `contractMarkdown`, `suggestedClauses`, `riskAssessment`, `generatedAt`
- **Outputs:** to **Convert Markdown to HTML**
- **Edge cases/failures:**
  - Regex `{[\s\S]*}` is greedy; if multiple JSON objects appear, parsing may fail.
  - If agent output is already structured JSON (not as string), this logic may ignore it.

---

### Block 3 — Document Formatting & E-Signature Initiation
**Overview:** Converts the contract to HTML, prepares signer and envelope payload, submits it to an e-signature API endpoint, sends email notifications, and pauses execution awaiting a signature callback.

**Nodes involved:**
- Convert Markdown to HTML
- Prepare Signature Request
- Send to E-Signature Service
- Send Signing Request Email
- Wait for Signature Webhook

#### Node: Convert Markdown to HTML
- **Type/role:** `Markdown` node, transforms Markdown into HTML.
- **Key config:**
  - Mode: Markdown → HTML
  - Options: tables, strikethrough, simple line breaks enabled
  - Input markdown: `{{ $json.contractMarkdown }}`
  - Output field: `contractHtml`
- **Outputs:** to **Prepare Signature Request**
- **Edge cases/failures:**
  - If `contractMarkdown` is empty or missing, HTML may be empty or minimal.

#### Node: Prepare Signature Request
- **Type/role:** `Code` node, constructs signers list and envelope metadata.
- **Key logic/output:**
  - `parties` expected as array; maps to `signers[]`:
    - `signerId`, `name`, `email`, `role`, `order`
  - `documentTitle`: `<templateType> - <contractId>`
  - `signingDeadline`: now + 7 days
  - `webhookUrl`: `$execution.resumeUrl` (n8n resume URL for the Wait node to continue execution)
- **Outputs:** to **Send to E-Signature Service**
- **Edge cases/failures:**
  - If any party lacks `email` or `name`, downstream email/e-sign will break or send invalid payload.
  - `$execution.resumeUrl` is only meaningful for wait/resume patterns; ensure the e-sign service can call it.

#### Node: Send to E-Signature Service
- **Type/role:** `HTTP Request` node, posts envelope to e-sign provider.
- **Key config:**
  - POST `https://api.esignature-service.example.com/envelopes` (placeholder domain)
  - Body includes:
    - `envelope_id` = `contractId`
    - `document_title`
    - `document_html` (stringified HTML)
    - `signers` (stringified array)
    - `webhook_url` = resume URL
    - `deadline`
  - Authentication: `predefinedCredentialType` with `httpHeaderAuth`
- **Outputs:** two parallel main connections:
  - → **Send Signing Request Email**
  - → **Wait for Signature Webhook**
- **Edge cases/failures:**
  - Placeholder URL must be replaced with real provider endpoint.
  - Provider may require additional fields (files/PDFs, signature tabs/anchors, redirect URLs, HMAC validation).
  - Credential/header misconfiguration causes 401/403.
  - Large HTML may exceed provider limits.

#### Node: Send Signing Request Email
- **Type/role:** `Gmail` node, notifies signers by email.
- **Key config:**
  - To: `{{ $json.signers.map(s => s.email).join(', ') }}`
  - Subject/body include `contractId`, `documentTitle`, `signingDeadline`
- **Credentials:** Gmail OAuth2 credential.
- **Inputs:** From **Send to E-Signature Service** output.
- **Outputs:** (none downstream besides parallel flow; its output is not merged later—only used for side effect)
- **Edge cases/failures:**
  - Gmail API scopes/consent issues, rate limits.
  - Invalid recipient emails.

#### Node: Wait for Signature Webhook
- **Type/role:** `Wait` node, pauses execution until resume webhook is called or timeout reached.
- **Key config:**
  - Resume mode: webhook
  - HTTP Method: POST
  - Limit wait time: 7 days
- **Inputs:** From **Send to E-Signature Service**
- **Outputs:** to **Process Signature Result**
- **Critical integration note:** The original webhook response is only sent at workflow end. Waiting days will usually exceed the client’s HTTP timeout. In practice, you typically:
  - respond immediately to the initial webhook (202 Accepted), then handle signature completion as a separate workflow, **or**
  - store state externally and use the Wait node without blocking the original webhook response.
- **Edge cases/failures:**
  - If the e-sign provider never calls the resume URL, the workflow times out and resumes with no body (handled as expired in later logic).
  - Resume URLs must be reachable from the e-sign provider (public n8n URL, correct network/firewall).

---

### Block 4 — Completion Handling (Logging + Notifications) & Webhook Response
**Overview:** Processes the signature callback payload, derives final status, branches on SIGNED vs not, logs outcome to Google Sheets, emails parties, merges branch results, and responds to the initial webhook.

**Nodes involved:**
- Process Signature Result
- Check Signature Status
- Log Completed Contract
- Send Completion Email
- Log Expired or Pending
- Send Reminder or Expiry Email
- Merge Final Results
- Respond to Webhook

#### Node: Process Signature Result
- **Type/role:** `Code` node, interprets resume webhook data and creates a normalized status object.
- **Key logic:**
  - Reads `webhookData.body.status` or `webhookData.body.event`
    - Complete if status `completed` or event `envelope_completed`
    - Expired if status `expired` OR no webhook body at all
  - Uses original request context from **Prepare Signature Request**
  - Outputs:
    - `status`: `SIGNED` / `EXPIRED` / `PENDING`
    - `signedAt` only when signed
    - `signatureDetails` = webhook body or `{}`
- **Outputs:** to **Check Signature Status**
- **Edge cases/failures:**
  - Provider webhook schema may differ; status detection may never mark SIGNED.
  - If multiple signers complete individually, you may need additional event handling (partial completion).

#### Node: Check Signature Status
- **Type/role:** `If` node, routes based on final status.
- **Condition:** `{{ $json.status }} == "SIGNED"` (strict validation).
- **True →** Log Completed Contract  
- **False →** Log Expired or Pending
- **Edge cases:** Any non-“SIGNED” value (including provider-specific statuses) will go to the non-signed path.

#### Node: Log Completed Contract
- **Type/role:** `Google Sheets` append row.
- **Key config:** Operation `append`, but **documentId** and **sheetName** are not set (placeholders).
- **Outputs:** to **Send Completion Email**
- **Credentials:** Google Sheets OAuth2.
- **Edge cases/failures:**
  - Missing document/sheet configuration will cause runtime errors.
  - Column mapping isn’t shown; you must define which fields map to which columns.

#### Node: Send Completion Email
- **Type/role:** `Gmail` node, emails all signers that contract is signed.
- **To:** all signer emails, subject/body includes signedAt/effectiveDate.
- **Outputs:** to **Merge Final Results**
- **Edge cases:** Same Gmail issues as earlier.

#### Node: Log Expired or Pending
- **Type/role:** `Google Sheets` append row for non-signed cases.
- **Key config:** Same placeholders for documentId/sheetName.
- **Outputs:** to **Send Reminder or Expiry Email**
- **Edge cases:** Same Sheets configuration issues.

#### Node: Send Reminder or Expiry Email
- **Type/role:** `Gmail` node, notifies signers that request expired or pending.
- **Message uses conditional expression:**  
  - If EXPIRED → “expired” messaging, else “not been completed”
- **Outputs:** to **Merge Final Results** (index 1)
- **Edge cases:** None beyond Gmail.

#### Node: Merge Final Results
- **Type/role:** `Merge` node in “chooseBranch” mode (collects whichever branch executed).
- **Inputs:** from completion email branch and reminder/expiry branch.
- **Outputs:** to **Respond to Webhook**
- **Edge cases:** As with the earlier merge, chooseBranch simply forwards one branch; if both run unexpectedly, behavior may not match expectations.

#### Node: Respond to Webhook
- **Type/role:** `Respond to Webhook` node, returns JSON to the original webhook request.
- **Key config:**
  - Respond with JSON:
    - `success: true`
    - `contractId` from **Prepare Contract Data**
    - message: “Contract has been generated and sent for signatures”
- **Inputs:** from **Merge Final Results** and also from **Invalid Request Response**
- **Edge cases (important):**
  - If the workflow waits for signature (up to 7 days), the original requester will almost certainly time out before receiving this response.
  - If the invalid branch executes, `success` is still returned as `true` (because response body is hard-coded). The invalid branch sets `error: true` in data, but the Respond node doesn’t use it—this is a logic mismatch.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Workflow documentation panel |  |  | ## Contract Template Generator with E-Signature Integration / Overview + Key Features + Required Credentials |
| Sticky Note1 | Sticky Note | Step label: intake |  |  | ### Step 1: Contract Request Intake (webhook, validation, unique ID) |
| Sticky Note2 | Sticky Note | Step label: AI generation |  |  | ### Step 2: AI Contract Generation (route template, AI agent, Markdown draft) |
| Sticky Note3 | Sticky Note | Step label: signature request |  |  | ### Step 3: Document Processing & Signature Request (MD→HTML, send to e-sign, wait) |
| Sticky Note4 | Sticky Note | Step label: completion |  |  | ### Step 4: Completion & Notification (log to Sheets, email parties, handle expired/pending) |
| Contract Request Webhook | Webhook | Receives contract requests | — | Validate Contract Request | ### Step 1: Contract Request Intake (webhook, validation, unique ID) |
| Validate Contract Request | If | Validates required input fields | Contract Request Webhook | Prepare Contract Data; Invalid Request Response | ### Step 1: Contract Request Intake (webhook, validation, unique ID) |
| Prepare Contract Data | Set | Normalizes request + generates contractId | Validate Contract Request | Route by Contract Type | ### Step 1: Contract Request Intake (webhook, validation, unique ID) |
| Invalid Request Response | Set | Builds error payload | Validate Contract Request | Respond to Webhook | ### Step 1: Contract Request Intake (webhook, validation, unique ID) |
| Route by Contract Type | Switch | Selects base template branch | Prepare Contract Data | NDA Template Base; Service Agreement Base; Employment Contract Base; Generic Contract Base | ### Step 2: AI Contract Generation (route template, AI agent, Markdown draft) |
| NDA Template Base | Set | Defines NDA base terms | Route by Contract Type | Merge Template Data | ### Step 2: AI Contract Generation (route template, AI agent, Markdown draft) |
| Service Agreement Base | Set | Defines Service Agreement base terms | Route by Contract Type | Merge Template Data | ### Step 2: AI Contract Generation (route template, AI agent, Markdown draft) |
| Employment Contract Base | Set | Defines Employment base terms | Route by Contract Type | Merge Template Data | ### Step 2: AI Contract Generation (route template, AI agent, Markdown draft) |
| Generic Contract Base | Set | Defines fallback base terms | Route by Contract Type | Merge Template Data | ### Step 2: AI Contract Generation (route template, AI agent, Markdown draft) |
| Merge Template Data | Merge | Passes chosen template data forward | Template Base nodes | AI Contract Advisor | ### Step 2: AI Contract Generation (route template, AI agent, Markdown draft) |
| AI Contract Advisor | LangChain Agent | Generates contract JSON via LLM | Merge Template Data (+ OpenAI Chat Model) | Parse AI Contract Response | ### Step 2: AI Contract Generation (route template, AI agent, Markdown draft) |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM backend | — | AI Contract Advisor | ### Step 2: AI Contract Generation (route template, AI agent, Markdown draft) |
| Parse AI Contract Response | Code | Extracts/parses JSON; merges with request | AI Contract Advisor | Convert Markdown to HTML | ### Step 2: AI Contract Generation (route template, AI agent, Markdown draft) |
| Convert Markdown to HTML | Markdown | Formats contract for signing | Parse AI Contract Response | Prepare Signature Request | ### Step 3: Document Processing & Signature Request (MD→HTML, send to e-sign, wait) |
| Prepare Signature Request | Code | Builds signers list + resume webhook URL | Convert Markdown to HTML | Send to E-Signature Service | ### Step 3: Document Processing & Signature Request (MD→HTML, send to e-sign, wait) |
| Send to E-Signature Service | HTTP Request | Creates envelope in e-sign provider | Prepare Signature Request | Send Signing Request Email; Wait for Signature Webhook | ### Step 3: Document Processing & Signature Request (MD→HTML, send to e-sign, wait) |
| Send Signing Request Email | Gmail | Notifies signers to sign | Send to E-Signature Service | — | ### Step 3: Document Processing & Signature Request (MD→HTML, send to e-sign, wait) |
| Wait for Signature Webhook | Wait | Pauses until signature callback/timeout | Send to E-Signature Service | Process Signature Result | ### Step 3: Document Processing & Signature Request (MD→HTML, send to e-sign, wait) |
| Process Signature Result | Code | Interprets webhook; computes status | Wait for Signature Webhook | Check Signature Status | ### Step 4: Completion & Notification (log to Sheets, email parties, handle expired/pending) |
| Check Signature Status | If | Branches SIGNED vs other | Process Signature Result | Log Completed Contract; Log Expired or Pending | ### Step 4: Completion & Notification (log to Sheets, email parties, handle expired/pending) |
| Log Completed Contract | Google Sheets | Append SIGNED row | Check Signature Status | Send Completion Email | ### Step 4: Completion & Notification (log to Sheets, email parties, handle expired/pending) |
| Send Completion Email | Gmail | Confirms completion to parties | Log Completed Contract | Merge Final Results | ### Step 4: Completion & Notification (log to Sheets, email parties, handle expired/pending) |
| Log Expired or Pending | Google Sheets | Append non-signed row | Check Signature Status | Send Reminder or Expiry Email | ### Step 4: Completion & Notification (log to Sheets, email parties, handle expired/pending) |
| Send Reminder or Expiry Email | Gmail | Reminds/alerts signers | Log Expired or Pending | Merge Final Results | ### Step 4: Completion & Notification (log to Sheets, email parties, handle expired/pending) |
| Merge Final Results | Merge | Passes executed branch to responder | Send Completion Email; Send Reminder or Expiry Email | Respond to Webhook | ### Step 4: Completion & Notification (log to Sheets, email parties, handle expired/pending) |
| Respond to Webhook | Respond to Webhook | Returns JSON response | Invalid Request Response; Merge Final Results | — | ### Step 1: Contract Request Intake (webhook, validation, unique ID) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (keep execution order default; this one uses `v1`).
2. **Add Webhook node** named **Contract Request Webhook**:
   - Method: **POST**
   - Path: `contract-request`
   - Response: **Using “Respond to Webhook” node**
3. **Add If node** named **Validate Contract Request** and connect:
   - Contract Request Webhook → Validate Contract Request
   - Conditions (AND):
     - String **exists**: `{{$json.body.contractType}}`
     - Array **not empty**: `{{$json.body.parties}}`
4. **Add Set node** **Prepare Contract Data** (connect from If = true):
   - Create fields:
     - `contractId` (string): `{{ 'CTR-' + $now.format('yyyyMMdd') + '-' + Math.random().toString(36).substring(2, 8).toUpperCase() }}`
     - `contractType`: `{{$json.body.contractType}}`
     - `parties`: `{{$json.body.parties}}`
     - `customTerms`: `{{$json.body.customTerms || ''}}`
     - `effectiveDate`: `{{$json.body.effectiveDate || $now.format('yyyy-MM-dd')}}`
     - `expirationDate`: `{{$json.body.expirationDate || ''}}`
     - `value` (number): `{{$json.body.contractValue || 0}}`
     - `currency`: `{{$json.body.currency || 'USD'}}`
     - `requestedAt`: `{{$now.toISO()}}`
5. **Add Set node** **Invalid Request Response** (connect from If = false):
   - Fields:
     - `error` = true
     - `message` = “Missing required fields: contractType and parties array are required”
6. **Add Switch node** **Route by Contract Type**:
   - Input: `{{$json.contractType}}`
   - Rules:
     - equals `NDA`
     - equals `SERVICE`
     - equals `EMPLOYMENT`
   - Fallback output enabled (to a 4th branch)
   - Connect Prepare Contract Data → Route by Contract Type
7. **Add 4 Set nodes** for templates and connect each switch output:
   - **NDA Template Base**: `templateType = "Non-Disclosure Agreement"`, `baseTerms = "..."`
   - **Service Agreement Base**: `templateType = "Service Agreement"`, `baseTerms = "..."`
   - **Employment Contract Base**: `templateType = "Employment Contract"`, `baseTerms = "..."`
   - **Generic Contract Base** (fallback): `templateType = "General Agreement"`, `baseTerms = "Standard commercial terms apply"`
8. **Add Merge node** **Merge Template Data**:
   - Mode: **Choose Branch**
   - Connect all 4 template nodes → Merge Template Data
9. **Add OpenAI Chat Model node** **OpenAI Chat Model**:
   - Model: `gpt-4o-mini`
   - Temperature: `0.3`
   - Configure **OpenAI credentials** in n8n (API key).
10. **Add LangChain Agent node** **AI Contract Advisor**:
   - Prompt type: “Define”
   - System message: “You are an expert legal contract advisor… Always output valid JSON.”
   - User prompt: include the same interpolations referencing **Prepare Contract Data** and current template fields; require JSON keys (`contractMarkdown`, `suggestedClauses`, `riskAssessment`).
   - Connect:
     - Merge Template Data → AI Contract Advisor (main)
     - OpenAI Chat Model → AI Contract Advisor (AI language model connection)
11. **Add Code node** **Parse AI Contract Response**:
   - Paste logic that extracts JSON from `aiResponse.output`, merges with original contract data, and sets fallbacks.
   - Connect AI Contract Advisor → Parse AI Contract Response
12. **Add Markdown node** **Convert Markdown to HTML**:
   - Mode: Markdown → HTML
   - Markdown: `{{$json.contractMarkdown}}`
   - Destination key: `contractHtml`
   - Connect Parse AI Contract Response → Convert Markdown to HTML
13. **Add Code node** **Prepare Signature Request**:
   - Create `signers[]` from `parties`, set `documentTitle`, `signingDeadline`, and `webhookUrl = $execution.resumeUrl`.
   - Connect Convert Markdown to HTML → Prepare Signature Request
14. **Add HTTP Request node** **Send to E-Signature Service**:
   - Method: POST
   - URL: replace placeholder with your provider endpoint
   - Authentication: Header Auth (configure credentials, e.g. `Authorization: Bearer ...`)
   - JSON body containing envelope ID, title, HTML, signers, webhook URL, deadline.
   - Connect Prepare Signature Request → Send to E-Signature Service
15. **Add Gmail node** **Send Signing Request Email**:
   - To: `{{$json.signers.map(s => s.email).join(', ')}}`
   - Subject/body as in workflow
   - Configure **Gmail OAuth2** credentials in n8n
   - Connect Send to E-Signature Service → Send Signing Request Email
16. **Add Wait node** **Wait for Signature Webhook**:
   - Resume: webhook (POST)
   - Limit wait: enabled, 7 days
   - Connect Send to E-Signature Service → Wait for Signature Webhook
17. **Add Code node** **Process Signature Result**:
   - Determine `SIGNED` / `EXPIRED` / `PENDING` from webhook body and merge with original data from **Prepare Signature Request**
   - Connect Wait for Signature Webhook → Process Signature Result
18. **Add If node** **Check Signature Status**:
   - Condition: `{{$json.status}} equals "SIGNED"`
   - Connect Process Signature Result → Check Signature Status
19. **Add Google Sheets node** **Log Completed Contract** (true branch):
   - Operation: Append
   - Set Document ID and Sheet name (create/select your spreadsheet)
   - Map fields (contractId, title, status, signedAt, etc.) to columns
   - Connect Check Signature Status (true) → Log Completed Contract
20. **Add Gmail node** **Send Completion Email**:
   - To: all signer emails
   - Connect Log Completed Contract → Send Completion Email
21. **Add Google Sheets node** **Log Expired or Pending** (false branch):
   - Operation: Append
   - Same spreadsheet/sheet or a different tab
   - Connect Check Signature Status (false) → Log Expired or Pending
22. **Add Gmail node** **Send Reminder or Expiry Email**:
   - Message uses conditional expression based on `status`
   - Connect Log Expired or Pending → Send Reminder or Expiry Email
23. **Add Merge node** **Merge Final Results**:
   - Mode: Choose Branch
   - Connect Send Completion Email → Merge Final Results (input 1)
   - Connect Send Reminder or Expiry Email → Merge Final Results (input 2)
24. **Add Respond to Webhook node** **Respond to Webhook**:
   - Respond with JSON and include `contractId` from **Prepare Contract Data**
   - Connect:
     - Merge Final Results → Respond to Webhook
     - Invalid Request Response → Respond to Webhook

**Credential checklist**
- OpenAI API credential for the OpenAI Chat Model node
- Gmail OAuth2 credential for Gmail nodes
- Google Sheets OAuth2 credential for Sheets nodes
- HTTP Header Auth credential (or provider-specific auth) for the e-sign HTTP request

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow uses a Wait-for-webhook resume pattern and attempts to respond to the original webhook after signature completion; most clients will time out long before that. Consider responding immediately and handling signature completion asynchronously in a separate workflow. | Architecture / runtime behavior note |
| The e-signature endpoint is a placeholder (`api.esignature-service.example.com`) and must be replaced with a real provider integration (DocuSign, Dropbox Sign, Adobe Sign, etc.), including required payload schema and webhook verification. | E-sign provider integration requirement |
| Google Sheets nodes have empty `documentId` and `sheetName` placeholders; they must be configured and field-to-column mapping must be set. | Deployment requirement |
| Invalid-request branch sets `error: true`, but Respond to Webhook always returns `success: true` with a generic message. Adjust Respond to Webhook to reflect validation failures. | Logic consistency note |