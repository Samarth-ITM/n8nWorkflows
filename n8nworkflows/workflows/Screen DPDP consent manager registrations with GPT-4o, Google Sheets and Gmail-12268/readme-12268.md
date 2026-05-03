Screen DPDP consent manager registrations with GPT-4o, Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/screen-dpdp-consent-manager-registrations-with-gpt-4o--google-sheets-and-gmail-12268


# Screen DPDP consent manager registrations with GPT-4o, Google Sheets and Gmail

## 1. Workflow Overview

**Workflow name:** Automated Consent Manager Registration Screening & Eligibility Evaluation Pipeline (DPDP-Aligned)  
**Provided title:** Screen DPDP consent manager registrations with GPT-4o, Google Sheets and Gmail

**Purpose:**  
This workflow ingests Consent Manager registration submissions (via webhook), normalizes and validates the payload, logs entries to Google Sheets, uses **Azure OpenAI GPT‑4o** to evaluate DPDP-aligned eligibility, then routes outcomes:
- **Eligible** → email the compliance team + update status in Google Sheets
- **Ineligible** → email the applicant with rejection details
- **Invalid payload** → log to a separate Google Sheet for audit/follow-up

### 1.1 Input Reception & Normalization
Receives POST submissions and transforms the raw webhook body into a consistent schema.

### 1.2 Payload Validation & Intake Logging
Checks that required structure exists (currently only `action` is validated), logs valid submissions to the main sheet; invalid ones to an audit sheet.

### 1.3 AI Eligibility Evaluation (DPDP-Aligned)
Runs a GPT‑4o agent with strict JSON output requirements and specific logic: **missing documentation must not cause rejection** if the rest is strong.

### 1.4 Decision Routing (Eligible vs Ineligible)
Parses AI output JSON, branches based on `eligible`.

### 1.5 Notifications & Status Update
Eligible path: send compliance email, set `Status=passed`, update the existing sheet row using `contactEmail` as the key.  
Ineligible path: send applicant rejection email (but note: current email template expects fields not produced by the AI parser as configured).

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Normalization

**Overview:**  
Accepts incoming registration events via webhook and extracts a clean, normalized registration object to ensure downstream nodes work with stable keys.

**Nodes involved:**  
- Receive Consent Registration Event  
- Extract & Normalize Registration Payload

#### Node: Receive Consent Registration Event
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — workflow trigger / HTTP intake.
- **Configuration (interpreted):**
  - Method: `POST`
  - Path: `14ad13dc-0916-4ea3-ac2c-943413207ad4` (also used as webhookId)
- **Input/Output:**
  - Entry point (no input)
  - Outputs to: **Extract & Normalize Registration Payload**
- **Version-specific:** typeVersion `2.1`
- **Failure modes / edge cases:**
  - Incorrect HTTP method (GET) → no trigger
  - Missing/invalid JSON body → downstream normalization defaults to empty strings/null
  - If the sender uses a different structure than `incoming.body`, fields may be empty

#### Node: Extract & Normalize Registration Payload
- **Type / role:** `Code` — transforms raw webhook payload into a cleaned schema.
- **Configuration choices:**
  - Reads `$json.body || {}`
  - Outputs a single item with keys:
    - `action`, `organizationName`, `applicationType`, `contactEmail`, `netWorth`,
      `technicalCapacity`, `operationalCapacity`, `documentAttached`, `submittedAt`
  - Defaults to empty string for text fields, `null` for `documentAttached`
- **Key expressions/variables:**
  - Uses `$json` from the webhook node
- **Input/Output:**
  - Input: webhook event JSON
  - Output to: **Validate Registration Payload Structure**
- **Version-specific:** typeVersion `2`
- **Failure modes / edge cases:**
  - If webhook payload is not shaped as `{ body: {...} }`, all fields will default to empty, making validation likely fail (or pass incorrectly depending on validation rules).
  - `documentAttached` is passed through as-is; if it’s a complex object, Sheets/email formatting may not match expectations later.

---

### Block 2 — Payload Validation & Intake Logging

**Overview:**  
Determines whether the request has minimal structure and then logs it: valid payloads go to the main registration sheet; invalid payloads go to an audit sheet.

**Nodes involved:**  
- Validate Registration Payload Structure  
- Write Initial Registration Entry to Sheet  
- Log Invalid Registration Requests to Sheet

#### Node: Validate Registration Payload Structure
- **Type / role:** `IF` — minimal structural validation gate.
- **Configuration choices:**
  - Condition checks: `action` **is not empty**
  - Uses strict type validation
  - Combinator set to `or` (currently only one condition)
- **Key expressions/variables:**
  - `={{ $json.action }}`
- **Input/Output:**
  - Input: cleaned registration JSON from normalization
  - **True output** → AI Eligibility Evaluator + Write Initial Registration Entry to Sheet (two parallel branches)
  - **False output** → Log Invalid Registration Requests to Sheet
- **Version-specific:** typeVersion `2.2`
- **Failure modes / edge cases:**
  - This validation is extremely permissive: payloads with empty `organizationName`, invalid `contactEmail`, etc. still pass if `action` is present.
  - If `action` exists but is whitespace, “notEmpty” still typically counts as not empty; consider trimming in normalization.

#### Node: Write Initial Registration Entry to Sheet
- **Type / role:** `Google Sheets` — append intake record to primary sheet.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `sample_leads_50` (ID `17rcNd_ZpUQLm0uWEVbD-NY6GyFUkrD4BglvawlyBygM`)
  - Sheet/tab: `consent_manager_registration` (gid `616260608`)
  - Maps columns from the cleaned payload via expressions (e.g., `={{ $json.organizationName }}`)
- **Credentials:** Google Sheets OAuth2 `automations@techdome.ai`
- **Input/Output:**
  - Input: output of IF (valid payload)
  - No downstream connection in this workflow (logging only)
- **Version-specific:** typeVersion `4.6`
- **Failure modes / edge cases:**
  - Sheet/Doc permissions or expired OAuth token → auth errors
  - Column names in sheet must match mapping schema; otherwise values may not land as intended
  - `documentAttached` might be `null`; depending on Sheets node behavior, it may store blank or the string “null” (varies with conversion settings)

#### Node: Log Invalid Registration Requests to Sheet
- **Type / role:** `Google Sheets` — audit logging for malformed/incomplete requests.
- **Configuration choices:**
  - Operation: `append`
  - **DocumentId and sheetName are empty** in the node configuration (both are blank in the JSON)
- **Credentials:** Google Sheets OAuth2 `automations@techdome.ai`
- **Input/Output:**
  - Input: output of IF (invalid payload)
  - No downstream connection
- **Version-specific:** typeVersion `4.6`
- **Failure modes / edge cases:**
  - As configured, this node will fail at runtime due to missing `documentId` and `sheetName`.
  - Even after setting them, ensure a schema is defined or use auto-mapping; otherwise the logged data may be incomplete.

---

### Block 3 — AI Eligibility Evaluation (DPDP-Aligned)

**Overview:**  
Uses an Azure OpenAI GPT‑4o chat model connected to a LangChain Agent. The agent is instructed to output *strict JSON* with eligibility decision, risk level, missing items, and next steps, and to treat missing documentation as a minor issue.

**Nodes involved:**  
- Configure GPT-4o — Eligibility Evaluation Model  
- AI Eligibility Evaluator (DPDP Compliance)  
- Parse AI Eligibility JSON Output

#### Node: Configure GPT-4o — Eligibility Evaluation Model
- **Type / role:** `Azure OpenAI Chat Model` (LangChain) — provides the LLM to the agent.
- **Configuration choices:**
  - Model: `gpt-4o`
  - No special options configured
- **Credentials:** Azure OpenAI account
- **Connections:**
  - Connected via **ai_languageModel** output to the agent node
- **Version-specific:** typeVersion `1`
- **Failure modes / edge cases:**
  - Azure deployment/model name mismatch (Azure often requires deployment names) → model not found
  - Rate limits / quota / content filtering → errors or truncated output
  - Network timeouts on large prompts

#### Node: AI Eligibility Evaluator (DPDP Compliance)
- **Type / role:** `LangChain Agent` — prompts the model to evaluate eligibility.
- **Configuration choices (interpreted):**
  - PromptType: `define`
  - User prompt includes:
    - DPDP eligibility instruction
    - Explicit “logic changes” (missing doc does not reject)
    - Injects full registration JSON: `{{ JSON.stringify($json, null, 2) }}`
    - Requires output in exact JSON structure:
      ```json
      {
        "eligible": true/false,
        "decisionReason": "...",
        "riskLevel": "Low / Medium / High",
        "missingItems": ["..."],
        "recommendedNextSteps": "..."
      }
      ```
  - System message enforces:
    - Missing docs are minor
    - Prioritize financial/technical/operational capacity + email validity
    - Output must ALWAYS follow JSON format exactly
- **Input/Output:**
  - Input: validated registration payload
  - Output to: Parse AI Eligibility JSON Output
  - Receives LLM via `ai_languageModel` from the Azure chat model node
- **Version-specific:** typeVersion `2.1`
- **Failure modes / edge cases:**
  - The model may still wrap JSON in markdown fences; downstream parser tries to remove them.
  - Model might output invalid JSON (trailing commas, comments, unescaped quotes) → parser node fails.
  - “Email validity” is requested but no deterministic validation exists; LLM may hallucinate validity.

#### Node: Parse AI Eligibility JSON Output
- **Type / role:** `Code` — converts AI output string into structured JSON.
- **Configuration choices:**
  - Reads `const raw = $json.output;`
  - Removes markdown fences via regex: `/```json|```/g`
  - `JSON.parse(cleaned)`
  - Returns `parsed` directly
- **Input/Output:**
  - Input: agent output (must contain `.output`)
  - Output to: Validate Eligibility Status
- **Version-specific:** typeVersion `2`
- **Failure modes / edge cases:**
  - If `$json.output` is undefined (agent node output shape differs), this throws.
  - If AI returns JSON but with leading text, or code fences like ```JSON (uppercase), regex won’t remove it.
  - `return parsed;` in n8n Code nodes is risky: n8n typically expects `return [{ json: parsed }]`. Returning a raw object/array may break execution depending on n8n version/runtime behavior.

---

### Block 4 — Decision Routing (Eligible vs Ineligible)

**Overview:**  
Branches on the parsed AI decision. Eligible applicants are summarized and forwarded to compliance; ineligible applicants receive a rejection email.

**Nodes involved:**  
- Validate Eligibility Status  
- Merge Registration + Eligibility Summary  
- Send Rejection Email to Applicant

#### Node: Validate Eligibility Status
- **Type / role:** `IF` — routing decision.
- **Configuration choices:**
  - Condition: `eligible` (boolean) equals `true`
  - Strict type validation
- **Key expressions:**
  - `={{ $json.eligible }}`
- **Input/Output:**
  - Input: parsed AI JSON
  - True → Merge Registration + Eligibility Summary
  - False → Send Rejection Email to Applicant
- **Version-specific:** typeVersion `2.2`
- **Failure modes / edge cases:**
  - If `eligible` is `"true"` (string) rather than boolean true, strict validation may fail or branch unexpectedly.

#### Node: Merge Registration + Eligibility Summary
- **Type / role:** `Code` — combines original registration data + AI decision into one package for internal email and sheet update.
- **Configuration choices:**
  - Pulls form data from: `$node["Validate Registration Payload Structure"].json`
  - Pulls eligibility from current input: `items[0].json`
  - Outputs:
    - `registrationPackage: { ... }`
    - `eligibility: { ... }`
- **Input/Output:**
  - Input: AI parsed result (eligible path)
  - Output to: Send Approval Email to Compliance Team
- **Version-specific:** typeVersion `2`
- **Failure modes / edge cases:**
  - `$node["Validate Registration Payload Structure"].json` references the IF node, not the normalization node. IF node’s `.json` usually contains the item passed through, but relying on IF node output shape can be brittle.
  - If multiple items are processed, using only `items[0]` and `.item` references may mismatch items.

#### Node: Send Rejection Email to Applicant
- **Type / role:** `Gmail` — sends rejection notification.
- **Configuration choices:**
  - To: hard-coded expression `=gargsaurabh1804april@gmail.com` (not the applicant email)
  - Subject: “Consent Manager Registration – Rejected (Missing Documents)”
  - Body uses:
    - organization name from `Validate Registration Payload Structure`
    - `{{ $json.issues.join(", ") }}`
    - `{{ $json.recommendedNextSteps }}`
- **Credentials:** Gmail OAuth2 “Gmail account”
- **Input/Output:**
  - Input: ineligible branch from Validate Eligibility Status
  - No downstream
- **Version-specific:** typeVersion `2.1`
- **Failure modes / edge cases (important):**
  - **Data mismatch:** The AI schema returns `missingItems` not `issues`. So `$json.issues` will be `undefined` → `.join()` will throw and the node may fail.
  - Recipient is not the applicant; likely should be `={{ $node["Validate Registration Payload Structure"].json.contactEmail }}`.
  - Subject implies rejection for missing documents, but AI logic rejects only for major compliance failure; subject/body may be misleading.

---

### Block 5 — Eligible Notifications & Sheet Status Update

**Overview:**  
For eligible applicants, the workflow emails the compliance team with a structured summary, then updates the applicant row in Google Sheets by matching on `contactEmail`.

**Nodes involved:**  
- Send Approval Email to Compliance Team  
- Prepare Status Update Fields  
- Update Registration Status in Sheet

#### Node: Send Approval Email to Compliance Team
- **Type / role:** `Gmail` — internal notification.
- **Configuration choices:**
  - To: hard-coded `gargsaurabh1804april@gmail.com`
  - Subject: “Consent Manager Registration – Eligibility Approved (Verification Stage)”
  - HTML body includes details from:
    - `$json.registrationPackage.*`
    - `$json.eligibility.*`
  - “Missing Items” section has an empty `<ul>`; it does not render the `missingItems` array.
- **Credentials:** Gmail OAuth2 “Gmail account”
- **Input/Output:**
  - Input: merged summary
  - Output to: Prepare Status Update Fields
- **Version-specific:** typeVersion `2.1`
- **Failure modes / edge cases:**
  - If merged object keys differ, email templates will produce blanks.
  - Consider injecting missingItems into `<ul>` via an expression loop alternative (n8n doesn’t support loops in templates directly; often done with a Code node to render HTML list).

#### Node: Prepare Status Update Fields
- **Type / role:** `Set` — creates fields needed for the sheet update.
- **Configuration choices:**
  - Sets `Status` to string `"passed "` (note trailing space)
- **Input/Output:**
  - Input: after approval email
  - Output to: Update Registration Status in Sheet
- **Version-specific:** typeVersion `3.4`
- **Failure modes / edge cases:**
  - Trailing whitespace in status can complicate filtering/analytics in Sheets.

#### Node: Update Registration Status in Sheet
- **Type / role:** `Google Sheets` — updates or appends row based on key.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Spreadsheet: same as intake sheet
  - Matching column: `contactEmail`
  - Writes:
    - `Status` from current item: `={{ $json.Status }}`
    - `contactEmail` from merged summary: `={{ $('Merge Registration + Eligibility Summary').item.json.registrationPackage.contactEmail }}`
- **Credentials:** Google Sheets OAuth2 `automations@techdome.ai`
- **Input/Output:** terminal node
- **Version-specific:** typeVersion `4.6`
- **Failure modes / edge cases:**
  - If `contactEmail` is blank/invalid, appendOrUpdate may append duplicates or fail to match.
  - Using `.item` reference can mismatch if multiple items are processed simultaneously.
  - If the sheet column is named differently (case/spacing), matching will not work.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Consent Registration Event | Webhook | Entry point: receives registration POST | — | Extract & Normalize Registration Payload | ## Receive Consent Registration Event  \nWebhook collects new registration submissions from the UI.  \nTriggers workflow execution immediately and passes the raw payload forward. |
| Extract & Normalize Registration Payload | Code | Normalize incoming body to stable schema | Receive Consent Registration Event | Validate Registration Payload Structure | ## Extract & Normalize Registration Payload  \nCleans the incoming JSON body and extracts essential registration fields.  \nNormalizes values into a consistent structure for downstream validation. |
| Validate Registration Payload Structure | IF | Minimal structural validation gate | Extract & Normalize Registration Payload | (true) AI Eligibility Evaluator (DPDP Compliance); Write Initial Registration Entry to Sheet; (false) Log Invalid Registration Requests to Sheet | ## Validate Registration Payload Structure  \nChecks whether core fields (like action) are present and properly structured.  \nValid entries move to evaluation → invalid entries go to the logging sheet. |
| Log Invalid Registration Requests to Sheet | Google Sheets | Audit log invalid submissions | Validate Registration Payload Structure (false) | — | ## Log Invalid Registration Requests to Sheet  \nRecords malformed or incomplete submissions in a dedicated audit sheet.  \nSupports retry handling and compliance tracking for improper requests. |
| Write Initial Registration Entry to Sheet | Google Sheets | Append intake record to main sheet | Validate Registration Payload Structure (true) | — | ## Write Initial Registration Entry to Sheet  \nStores the raw registration data into the main registration sheet.  \nCreates the intake record before AI eligibility processing begins. |
| Configure GPT-4o — Eligibility Evaluation Model | Azure OpenAI Chat Model (LangChain) | Provides GPT‑4o model to agent | — | AI Eligibility Evaluator (DPDP Compliance) (ai_languageModel) | ## Configure GPT-4o — Eligibility Evaluation Model  \nPrepares the AI model used to evaluate DPDP compliance eligibility.  \nServes as the language model input for the eligibility agent. |
| AI Eligibility Evaluator (DPDP Compliance) | LangChain Agent | Evaluate DPDP eligibility; produce strict JSON | Validate Registration Payload Structure (true) + Configure GPT-4o (ai_languageModel) | Parse AI Eligibility JSON Output | ## AI Eligibility Evaluator (DPDP Compliance)  \nAnalyzes DPDP eligibility based on financial, technical, and operational criteria.  \nProduces a structured JSON result with eligibility, risk level, and next steps. |
| Parse AI Eligibility JSON Output | Code | Strip fences + JSON.parse AI output | AI Eligibility Evaluator (DPDP Compliance) | Validate Eligibility Status | ## Parse AI Eligibility JSON Output  \nCleans markdown artifacts and safely converts AI output into valid JSON.  \nEnsures standardized data for all branching decisions ahead. |
| Validate Eligibility Status | IF | Branch on eligible true/false | Parse AI Eligibility JSON Output | (true) Merge Registration + Eligibility Summary; (false) Send Rejection Email to Applicant | ## Validate Eligibility Status  \nChecks whether the AI-determined eligibility is true or false.  \nRoutes eligible applicants to approval flow → ineligible ones to rejection. |
| Merge Registration + Eligibility Summary | Code | Combine registration + AI decision into one object | Validate Eligibility Status (true) | Send Approval Email to Compliance Team | ## Merge Registration + Eligibility Summary  \nCombines cleaned registration data with AI evaluation output.  \nCreates a single structured summary for internal compliance review. |
| Send Approval Email to Compliance Team | Gmail | Notify compliance for verification stage | Merge Registration + Eligibility Summary | Prepare Status Update Fields | ## Send Approval Email to Compliance Team  \nNotifies compliance that an applicant passed eligibility checks.  \nIncludes applicant details, capacity overview, and risk assessment. |
| Prepare Status Update Fields | Set | Set Status field for sheet update | Send Approval Email to Compliance Team | Update Registration Status in Sheet | ## Prepare Status Update Fields  \nSets the final status field (e.g., “passed”) for database update.  \nPrepares the output required for updating the registration sheet. |
| Update Registration Status in Sheet | Google Sheets | Append/update Status by contactEmail | Prepare Status Update Fields | — | ## Update Registration Status in Sheet  \nUpdates the existing registration record using contactEmail as the key.  \nStores final eligibility status, forming a complete audit trail. |
| Send Rejection Email to Applicant | Gmail | Send rejection notice | Validate Eligibility Status (false) | — | ## Send Rejection Email to Applicant  \nSends a structured rejection email explaining missing/invalid elements.  \nGuides applicants with clear next steps for re-submission. |
| Sticky Note | Sticky Note | Canvas documentation | — | — | ## 📝 Automated Consent Manager Registration Screening & Eligibility Evaluation Pipeline (DPDP-Aligned)\n\nAutomates intake, validation, eligibility evaluation, and routing of Consent Manager Registration applications under DPDP guidelines.  \nEvery incoming registration request is normalized, validated, evaluated using AI-based compliance logic, logged to Sheets, and then routed to either Approval or Rejection flow.  \nThe workflow ensures correct handling of missing documents, validates operational/technical capacity, and updates master records automatically.\n\n🔹 Workflow Overview\n1️⃣ Webhook receives new registration request  \n2️⃣ Extracts & cleans core form fields  \n3️⃣ Validates structure (action + mandatory values)  \n4️⃣ Writes initial registration entry to Google Sheets  \n5️⃣ AI performs eligibility evaluation (DPDP criteria)  \n6️⃣ Parses JSON & validates eligibility result  \n7️⃣ If **eligible** → build summary + email compliance + update Sheet status  \n8️⃣ If **ineligible** → send rejection email + log for follow-up  \n9️⃣ All invalid requests (missing core payload) are logged separately  \n\n🔹 Tools & Integrations  \nWebhook → Intake  \nCode Nodes → Payload extraction, merging, JSON parsing  \nGoogle Sheets → Registration recordkeeping + status updates  \nAzure GPT-4o → Eligibility evaluation engine  \nGmail → Rejection & Approval notifications  \nSet Node → Status field creation for final update |
| Sticky Note1 | Sticky Note | Canvas documentation | — | — | ## Receive Consent Registration Event  \nWebhook collects new registration submissions from the UI.  \nTriggers workflow execution immediately and passes the raw payload forward. |
| Sticky Note2 | Sticky Note | Canvas documentation | — | — | ## Extract & Normalize Registration Payload  \nCleans the incoming JSON body and extracts essential registration fields.  \nNormalizes values into a consistent structure for downstream validation. |
| Sticky Note3 | Sticky Note | Canvas documentation | — | — | ## Validate Registration Payload Structure  \nChecks whether core fields (like action) are present and properly structured.  \nValid entries move to evaluation → invalid entries go to the logging sheet. |
| Sticky Note4 | Sticky Note | Canvas documentation | — | — | ## Log Invalid Registration Requests to Sheet  \nRecords malformed or incomplete submissions in a dedicated audit sheet.  \nSupports retry handling and compliance tracking for improper requests. |
| Sticky Note5 | Sticky Note | Canvas documentation | — | — | ## Write Initial Registration Entry to Sheet  \nStores the raw registration data into the main registration sheet.  \nCreates the intake record before AI eligibility processing begins. |
| Sticky Note6 | Sticky Note | Canvas documentation | — | — | ## Configure GPT-4o — Eligibility Evaluation Model  \nPrepares the AI model used to evaluate DPDP compliance eligibility.  \nServes as the language model input for the eligibility agent. |
| Sticky Note7 | Sticky Note | Canvas documentation | — | — | ## AI Eligibility Evaluator (DPDP Compliance)  \nAnalyzes DPDP eligibility based on financial, technical, and operational criteria.  \nProduces a structured JSON result with eligibility, risk level, and next steps. |
| Sticky Note8 | Sticky Note | Canvas documentation | — | — | ## Parse AI Eligibility JSON Output  \nCleans markdown artifacts and safely converts AI output into valid JSON.  \nEnsures standardized data for all branching decisions ahead. |
| Sticky Note9 | Sticky Note | Canvas documentation | — | — | ## Validate Eligibility Status  \nChecks whether the AI-determined eligibility is true or false.  \nRoutes eligible applicants to approval flow → ineligible ones to rejection. |
| Sticky Note10 | Sticky Note | Canvas documentation | — | — | ## Send Rejection Email to Applicant  \nSends a structured rejection email explaining missing/invalid elements.  \nGuides applicants with clear next steps for re-submission. |
| Sticky Note11 | Sticky Note | Canvas documentation | — | — | ## Merge Registration + Eligibility Summary  \nCombines cleaned registration data with AI evaluation output.  \nCreates a single structured summary for internal compliance review. |
| Sticky Note12 | Sticky Note | Canvas documentation | — | — | ## Send Approval Email to Compliance Team  \nNotifies compliance that an applicant passed eligibility checks.  \nIncludes applicant details, capacity overview, and risk assessment. |
| Sticky Note13 | Sticky Note | Canvas documentation | — | — | ## Prepare Status Update Fields  \nSets the final status field (e.g., “passed”) for database update.  \nPrepares the output required for updating the registration sheet. |
| Sticky Note14 | Sticky Note | Canvas documentation | — | — | ## Update Registration Status in Sheet  \nUpdates the existing registration record using contactEmail as the key.  \nStores final eligibility status, forming a complete audit trail. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it (e.g.) “Automated Consent Manager Registration Screening & Eligibility Evaluation Pipeline (DPDP-Aligned)”.
   - (Optional) Add sticky notes to document blocks.

2) **Add Webhook trigger**
   - Node: **Webhook**
   - Method: `POST`
   - Path: generate or set a fixed path (store it for the UI/backend sender).
   - Connect to the next node.

3) **Add “Extract & Normalize Registration Payload” (Code)**
   - Node: **Code**
   - Logic: read `incoming.body` and output a normalized JSON with:
     - `action`, `organizationName`, `applicationType`, `contactEmail`, `netWorth`,
       `technicalCapacity`, `operationalCapacity`, `documentAttached`, `submittedAt`
   - Connect to the IF node.

4) **Add “Validate Registration Payload Structure” (IF)**
   - Condition: `{{$json.action}}` **not empty**
   - True branch should go to:
     - Google Sheets append (initial intake)
     - AI agent (eligibility)
   - False branch should go to:
     - Google Sheets append (invalid audit)

5) **Add Google Sheets node: “Write Initial Registration Entry to Sheet”**
   - Node: **Google Sheets**
   - Credentials: configure **Google Sheets OAuth2** (select the account).
   - Operation: `Append`
   - Document: select your spreadsheet
   - Sheet: select your tab (e.g., `consent_manager_registration`)
   - Map columns for each normalized field using expressions like `{{$json.organizationName}}`.

6) **Add Google Sheets node: “Log Invalid Registration Requests to Sheet”**
   - Node: **Google Sheets**
   - Credentials: same Google account or another
   - Operation: `Append`
   - **Important:** set **Document** and **Sheet** (this is missing in the provided workflow).
   - Choose whether to:
     - store raw `$json` in a single “payload” column, or
     - store the normalized fields plus an “errorReason”.

7) **Add Azure OpenAI model node: “Configure GPT‑4o — Eligibility Evaluation Model”**
   - Node: **Azure OpenAI Chat Model** (LangChain)
   - Credentials: configure **Azure OpenAI** (endpoint, key, deployment/model mapping as required by your n8n setup)
   - Model: `gpt-4o` (or your Azure deployment name if required)
   - Connect its **ai_languageModel** output to the Agent node’s language model input.

8) **Add LangChain Agent: “AI Eligibility Evaluator (DPDP Compliance)”**
   - Node: **AI Agent** (LangChain)
   - PromptType: “Define”
   - System message: DPDP evaluation rules; enforce strict JSON output.
   - User message: include the registration JSON via `{{ JSON.stringify($json, null, 2) }}`
   - Connect from IF (true) → agent main input
   - Connect model node → agent `ai_languageModel`

9) **Add “Parse AI Eligibility JSON Output” (Code)**
   - Node: **Code**
   - Input: agent output (typically in `$json.output`)
   - Steps:
     - strip markdown fences
     - `JSON.parse`
     - return as an n8n item (recommended): `return [{ json: parsed }];`
   - Connect to the eligibility IF node.

10) **Add “Validate Eligibility Status” (IF)**
   - Condition: `{{$json.eligible}}` equals `true` (boolean)
   - True → eligible flow; False → rejection flow

11) **Eligible flow: add “Merge Registration + Eligibility Summary” (Code)**
   - Node: **Code**
   - Combine:
     - Original registration fields (from earlier node output)
     - Parsed AI result
   - Output structure:
     - `registrationPackage` object
     - `eligibility` object
   - Connect to approval email.

12) **Eligible flow: add Gmail “Send Approval Email to Compliance Team”**
   - Node: **Gmail**
   - Credentials: configure Gmail OAuth2
   - To: compliance team address (currently hard-coded in the workflow)
   - Subject/body: use `{{$json.registrationPackage...}}` and `{{$json.eligibility...}}`
   - (Optional) Render missingItems by pre-building HTML in a Code node.

13) **Eligible flow: add Set “Prepare Status Update Fields”**
   - Node: **Set**
   - Set `Status` = `passed` (avoid trailing space)

14) **Eligible flow: add Google Sheets “Update Registration Status in Sheet”**
   - Node: **Google Sheets**
   - Operation: `Append or Update`
   - Document & Sheet: same as the main sheet
   - Matching column: `contactEmail`
   - Values:
     - `contactEmail` from merged package
     - `Status` from Set node

15) **Ineligible flow: add Gmail “Send Rejection Email to Applicant”**
   - Node: **Gmail**
   - To: applicant email from normalized payload (recommended: `{{$node["Extract & Normalize Registration Payload"].json.contactEmail}}`)
   - Body: reference AI fields (`missingItems`, `decisionReason`, `recommendedNextSteps`)
   - Connect from eligibility IF (false) to this node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow’s invalid-submission logging node is not fully configured (missing Google Sheet document/tab). | Fix required before production use. |
| Rejection email template expects `$json.issues`, but AI output provides `missingItems`; update the email template or adjust parsing/merge to provide `issues`. | Prevents runtime failure on `.join()`. |
| Approval email “Missing Items” section does not render `missingItems` array (empty `<ul>`). | Consider generating HTML list in a Code node before sending. |
| Status is set to `"passed "` with a trailing space. | Recommend using `"passed"` consistently. |
| Disclaimer (provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Compliance/context statement. |