Capture website leads to HubSpot or Google Sheets with Slack follow-up

https://n8nworkflows.xyz/workflows/capture-website-leads-to-hubspot-or-google-sheets-with-slack-follow-up-12374


# Capture website leads to HubSpot or Google Sheets with Slack follow-up

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Capture website leads to HubSpot or Google Sheets with Slack follow-up  
**Purpose:** Receive website leads via an HTTP webhook, normalize + validate the payload, optionally enrich the lead via an external HTTP service, route the lead to **Google Sheets** or **HubSpot** depending on a `destination` field, notify via **Slack**, and finally return an HTTP response to the original webhook caller.

### 1.1 Input Reception & Parsing
- Entry point: **Webhook Trigger (POST)**.
- Handles payloads where the body might be an object (JSON) or a JSON string.

### 1.2 Normalization & Validation
- Creates a consistent `lead.*` object (trim, lowercase email, defaults).
- Validates required fields and performs a basic anti-spam check.
- Invalid leads receive an immediate **400** response.

### 1.3 Optional Enrichment
- If `lead.enrich === true`, calls an enrichment endpoint (placeholder URL).
- Merges enrichment response back into the lead (because HTTP nodes may not preserve the original input context depending on configuration/branching).

### 1.4 Routing to Destination Storage
- Switch routes to:
  - **Google Sheets**: append-or-update by email (upsert).
  - **HubSpot**: create/update contact (upsert by email).

### 1.5 Notifications & Webhook Response
- On each destination branch:
  - Detects failure via presence of `$json.error`.
  - Sends Slack success/failure message.
  - Responds to webhook with **200** on success, **500** on storage failure.

---

## 2. Block-by-Block Analysis

### Block A — Webhook Intake & Payload Parsing
**Overview:** Receives the inbound lead payload via HTTP POST and ensures the body is a usable JSON object for downstream nodes.

**Nodes involved:**
- Webhook Trigger
- Parse Webhook body

#### Node: Webhook Trigger
- **Type / Role:** `Webhook` trigger; workflow entry point.
- **Key config:**
  - Method: **POST**
  - Path: `website-lead`
  - Response mode: **Respond via “Respond to Webhook” node** (`responseNode`)
- **Input/Output:**
  - No input (trigger).
  - Output goes to **Parse Webhook body**.
- **Edge cases / failures:**
  - Callers sending wrong HTTP method/path → never triggers.
  - Payload may arrive with `body` as string or object; addressed next node.
  - Because response is controlled by Respond nodes, failing to reach one can cause request to hang/time out.

#### Node: Parse Webhook body
- **Type / Role:** `Code` node; attempts to parse `body` if it is a JSON string.
- **Key logic:**
  - If `$json.body` is a string: `JSON.parse(body)` and merges fields into top-level JSON.
  - If parse fails: sets `parseError: "Invalid JSON in body"` and `rawBody`.
- **Input/Output:**
  - Input from Webhook Trigger.
  - Output to **Normalize leads**.
- **Edge cases / failures:**
  - If parse fails, workflow still continues; later normalization may fail if expected fields are absent.
  - If caller posts array/object formats inconsistent with expressions in Normalize node, may produce undefined values.

---

### Block B — Normalize & Validate Lead (and reject invalid)
**Overview:** Standardizes lead fields into a `lead` object and verifies required fields (email/message + at least one identity field). Invalid leads are returned immediately with HTTP 400.

**Nodes involved:**
- Normalize leads
- Code - Validate Lead
- IF node — Is Valid?
- Respond to Webhook — 400 Bad Request (invalid path)

#### Node: Normalize leads
- **Type / Role:** `Set` node; creates canonical fields under `lead.*`.
- **Key config choices (interpreted):**
  - Produces:
    - `lead.email`: lowercased + trimmed (tries `$json.email` else `$json.body[0].email`)
    - `lead.firstName`, `lead.lastName`, `lead.company`, `lead.message`
    - `lead.name`: constructed from first + last (multiple fallbacks)
    - `lead.source`: default `'unknown'`
    - `lead.destination`: default `'sheets'`
    - `lead.receivedAt`: `$now`
    - `lead.enrich`: boolean from `$json.enrich` or `$json.body[0].enrich`
- **Important expressions / pitfalls:**
  - Some fallbacks call `.trim()` without optional chaining (e.g. `$json.body[0].firstName.trim()`), which can throw if `body[0]` or `firstName` is missing.
  - `lead.name` expression uses concatenation with potentially undefined values; can yield `"undefined undefined"` in some cases.
- **Input/Output:**
  - Input from Parse Webhook body.
  - Output to Code - Validate Lead.
- **Edge cases / failures:**
  - Unexpected payload shapes (no `body`, or `body` not array) can break expressions.
  - Email normalization relies on presence of `email` or `body[0].email`.

#### Node: Code - Validate Lead
- **Type / Role:** `Code` node; validates required fields and flags spam-like messages.
- **Key logic:**
  - Missing checks:
    - `lead.email` required
    - `lead.message` required
    - At least one of `lead.firstName`, `lead.lastName`, or `lead.company`
  - Spam heuristic: message length < 5 → likely spam
  - Outputs:
    - `isValid` boolean
    - `missingFields` array
    - `isLikelySpam`
    - `errorMessage` string if invalid
- **Input/Output:**
  - Input from Normalize leads.
  - Output to IF node — Is Valid?
- **Edge cases / failures:**
  - If Normalize produced `lead` but with empty strings, validation treats empty string as falsy (good).
  - If Normalize failed earlier, `lead` may be `{}` and validation returns invalid (still safe).

#### Node: IF node — Is Valid?
- **Type / Role:** `IF` node; gates processing.
- **Condition:** `$json.isValid === true`
- **Routes:**
  - **True** → IF node — Enrichment enabled?
  - **False** → Respond to Webhook — 400 Bad Request
- **Edge cases / failures:**
  - If `isValid` is missing or not boolean, strict boolean comparison may route to false.

#### Node: Respond to Webhook — 400 Bad Request (invalid path)
- **Type / Role:** `Respond to Webhook`; returns error to caller.
- **Key config:**
  - Response code: **400**
  - JSON body includes:
    - `ok: false`
    - `error`: `$json.errorMessage`
    - `missingFields`: `$json.missingFields`
- **Input/Output:** Terminal response for invalid payloads.
- **Edge cases / failures:**
  - If `errorMessage` absent, caller sees empty/undefined error message.

---

### Block C — Optional Enrichment & Merge Back
**Overview:** If enrichment is enabled, calls a third-party enrichment endpoint and merges its response into the `lead` object while preserving the original lead data.

**Nodes involved:**
- IF node — Enrichment enabled?
- Set - Snapshot data
- HTTP Request - Enrich (Optional)
- Merge
- Set - Merge Enrichment into lead

#### Node: IF node — Enrichment enabled?
- **Type / Role:** `IF` node; decides whether to enrich.
- **Condition:** `{{ $json.lead.enrich }}` is true (loose validation enabled).
- **Routes / connections:**
  - **True branch** sends to **two nodes in parallel**:
    - Set - Snapshot data (index 0 of Merge)
    - HTTP Request - Enrich (Optional) (index 1 of Merge)
  - **False branch** goes directly to Set - Merge Enrichment into lead (no enrichment).
- **Edge cases / failures:**
  - If `lead.enrich` is undefined, condition false → no enrichment (expected).
  - If `lead.enrich` is a string `"true"`, loose validation may treat as true depending on n8n’s coercion behavior (can be surprising).

#### Node: Set - Snapshot data
- **Type / Role:** `Set` node; preserves the incoming lead data for later merge.
- **Key config:** `includeOtherFields: true` (effectively a pass-through snapshot).
- **Input/Output:**
  - Input from IF enrichment true branch.
  - Output to Merge (input 0).
- **Edge cases / failures:** Minimal; mainly used to ensure original context is present after HTTP call.

#### Node: HTTP Request - Enrich (Optional)
- **Type / Role:** `HTTP Request`; calls enrichment provider.
- **Key config:**
  - URL: `https://example.com` (placeholder)
  - Method: POST
  - Timeout: 20s
  - `onError: continueRegularOutput` and `alwaysOutputData: true` so workflow continues even on failure.
- **Input/Output:**
  - Input from IF enrichment true branch.
  - Output to Merge (input 1). The response typically appears under `$json.data` (depends on HTTP node response settings/version).
- **Edge cases / failures:**
  - DNS/timeout/4xx/5xx: continues, but response may contain error fields rather than `data`.
  - Provider auth headers not configured (common).
  - Response shape mismatch: Set node later expects `$json.data`.

#### Node: Merge
- **Type / Role:** `Merge` node; combines snapshot lead data with enrichment response.
- **Mode:** `combineByPosition` (pairs items by index).
- **Input/Output:**
  - Input 0: Set - Snapshot data (lead context)
  - Input 1: HTTP Request - Enrich (Optional) (enrichment result)
  - Output to Set - Merge Enrichment into lead
- **Edge cases / failures:**
  - If one branch returns no items (shouldn’t here), combine-by-position can produce empty output or mismatched pairs.

#### Node: Set - Merge Enrichment into lead
- **Type / Role:** `Set` node; attaches enrichment payload into `lead.enrichment`.
- **Key config:**
  - Sets `lead.enrichment = $json.data || null`
  - Excludes field `data` from the output, keeps other fields.
- **Input/Output:**
  - Input from Merge (enrichment path) OR directly from IF enrichment false branch.
  - Output to Switch - Choose destination (Sheets vs HubSpot).
- **Edge cases / failures:**
  - If HTTP node produced a different response field than `data`, enrichment will be null.
  - If false-branch bypasses HTTP, `$json.data` is undefined; it will set enrichment to null (fine).

---

### Block D — Route to Sheets vs HubSpot
**Overview:** Uses `lead.destination` to decide where to persist the lead.

**Nodes involved:**
- Switch - Choose destination (Sheets vs HubSpot)

#### Node: Switch - Choose destination (Sheets vs HubSpot)
- **Type / Role:** `Switch`; routes execution.
- **Rules:**
  - Output “Google Sheets” when `lower(lead.destination)` equals `sheets`
  - Output “HubSpot” when `lower(lead.destination)` equals `hubspot`
- **Input/Output:**
  - Input from Set - Merge Enrichment into lead
  - Output 0 → Google Sheets- Lookup by email
  - Output 1 → Hubspot - Create/Update contacts
- **Edge cases / failures:**
  - Any other destination value produces no matching output → workflow won’t reach a Respond node, causing webhook caller to time out.
  - Consider adding a default route (or third rule) to respond 400/422 for unsupported destination.

---

### Block E — Google Sheets Upsert + Slack + Webhook Response
**Overview:** Upserts the lead by email into a Google Sheet and notifies Slack; returns 200 or 500 to the webhook caller.

**Nodes involved:**
- Google Sheets- Lookup by email
- IF - Sheets update Failed?
- Slack - update failed
- Slack - Successfully updated
- Respond to Webhook - 500 - Sheets
- Respond to Webhook - 200 - Sheets

#### Node: Google Sheets- Lookup by email
- **Type / Role:** `Google Sheets` node; append or update row by matching email.
- **Operation:** `appendOrUpdate`
- **Matching:** `matchingColumns: ["email"]`
- **Mapping:** Defines columns:
  - `name, email, source, status="New", company, message, updatedAt=$now, receivedAt, destination`
  - `enrichment = JSON.stringify(lead.enrichment || null)`
- **Critical config gap:**
  - `documentId` is empty in the JSON (`value": ""`). Must be set to your spreadsheet.
- **Error handling:** `onError: continueRegularOutput` → downstream IF checks for `$json.error`.
- **Input/Output:**
  - Input from Switch (Sheets route).
  - Output to IF - Sheets update Failed?
- **Edge cases / failures:**
  - Missing credentials / OAuth scopes → error.
  - Spreadsheet/worksheet not selected → error.
  - Duplicate `email` column in schema definition (appears twice in node schema list); may confuse future edits but matching uses `matchingColumns`.
  - If `lead.email` empty, upsert match fails or writes blank email row.

#### Node: IF - Sheets update Failed?
- **Type / Role:** `IF`; detects Sheets write error.
- **Condition:** `$json.error?.message` is not empty.
- **Routes:**
  - **True (failed)** → Slack - update failed
  - **False (success)** → Slack - Successfully updated
- **Edge cases / failures:**
  - Some node errors may store error differently than `error.message`; condition could miss failures.

#### Node: Slack - update failed
- **Type / Role:** `Slack` node; sends failure alert.
- **Auth:** OAuth2 (Slack).
- **Message content:** Detailed multiline message including `$json.error` and lead fields (name/email/company/etc.) and truncated enrichment.
- **Input/Output:** On success posts message then goes to Respond to Webhook - 500 - Sheets.
- **Edge cases / failures:**
  - Slack OAuth token revoked / missing scopes (chat:write, etc.).
  - Message uses fields like `$json.name` (not `$json.lead.name`) because at this stage Google Sheets node output is likely flattened to columns; if not, values may appear as “-”.

#### Node: Respond to Webhook - 500 - Sheets
- **Type / Role:** Responds to caller with storage failure.
- **Response:** HTTP 500, JSON `{ ok:false, destination:"sheets", error:"sheets write failed" }`
- **Edge cases:** None; terminal.

#### Node: Slack - Successfully updated
- **Type / Role:** Slack success alert.
- **Auth:** OAuth2.
- **Input/Output:** Goes to Respond to Webhook - 200 - Sheets.
- **Edge cases:** Same field-shape dependency as failure message.

#### Node: Respond to Webhook - 200 - Sheets
- **Type / Role:** Success response.
- **Response:** HTTP 200, JSON `{ ok:true, destination:"sheets", message:"sheets write successful" }`

---

### Block F — HubSpot Upsert + Slack + Webhook Response
**Overview:** Upserts a HubSpot contact by email, notifies Slack, and returns 200 or 500.

**Nodes involved:**
- Hubspot - Create/Update contacts
- IF - HubSpot Failed?
- Slack - HubSpot update failed
- Slack - HubSpot Successfully updated
- Respond to Webhook - 500 - HubSpot
- Respond to Webhook - 200 - HubSpot

#### Node: Hubspot - Create/Update contacts
- **Type / Role:** `HubSpot` node; upsert contact.
- **Key config:**
  - Email: `{{$json.lead.email}}`
  - Additional fields: empty (so it likely only ensures contact exists; does not set firstname/lastname/company/message unless configured).
- **Error handling:** `onError: continueRegularOutput`
- **Input/Output:**
  - Input from Switch (HubSpot route).
  - Output to IF - HubSpot Failed?
- **Edge cases / failures:**
  - Missing HubSpot private app token / OAuth.
  - Rate limits or validation errors (invalid email).
  - Since additional fields are empty, expected CRM fields won’t be populated unless added.

#### Node: IF - HubSpot Failed?
- **Type / Role:** IF; detects HubSpot error.
- **Condition:** `$json.error?.message || $json.error` not empty.
- **Routes:**
  - **True (failed)** → Slack - HubSpot update failed → Respond 500
  - **False (success)** → Slack - HubSpot Successfully updated → Respond 200
- **Edge cases:** HubSpot node error shape might differ; condition attempts both `error.message` and `error`.

#### Node: Slack - HubSpot update failed
- **Type / Role:** Slack failure alert (simple format).
- **Message:** References `{{$json.lead.email}}` and `{{$json.error.message}}`.
- **Potential mismatch:** After HubSpot node, the lead object may or may not still exist depending on HubSpot node output. If HubSpot output doesn’t include `lead`, this message may show blank email.
- **Output:** Respond to Webhook - 500 - HubSpot.

#### Node: Respond to Webhook - 500 - HubSpot
- **Type / Role:** Respond failure.
- **Response:** HTTP 500, JSON `{ ok:false, destination:"hubspot", error:"HubSpot write failed" }`

#### Node: Slack - HubSpot Successfully updated
- **Type / Role:** Slack success alert (detailed format similar to Sheets success).
- **Field-shape risk:** Uses `$json.name`, `$json.email`, etc., which may not exist on HubSpot output unless mapped/merged.
- **Output:** Respond to Webhook - 200 - HubSpot.

#### Node: Respond to Webhook - 200 - HubSpot
- **Type / Role:** Respond success.
- **Response:** HTTP 200, JSON `{ ok:true, destination:"hubspot", message:"hubspot write successful" }`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger | Webhook | Entry point (POST form endpoint) | — | Parse Webhook body | ## Webhook Trigger node\nThis is your “form endpoint”. Your website (or Postman) will send POST JSON here.\n\nSend a POST request to the webhook URL with JSON body:\n```json\n{\n  \"firstName\": \"John\",\n  \"lastName\": \"Doe\",\n  \"email\": \"john@example.com\",\n  \"company\": \"johnscompany\",\n  \"message\": \"Need help...\",\n  \"source\": \"contact_page\",\n  \"enrich\": false,\n  \"destination\": \"sheets\"\n}\n``` |
| Sticky Note | Sticky Note | Comment / documentation | — | — |  |
| Parse Webhook body | Code | Parse `body` if string JSON | Webhook Trigger | Normalize leads | ## What this workflow does\n\n1. **Receives a lead** via **Webhook (POST)** ... (full diagram and explanation) |
| Normalize leads | Set | Build canonical `lead.*` object | Parse Webhook body | Code - Validate Lead | ## Normalize And Validate Lead\nReal data is messy. Trim spaces, lowercase email, create a consistent object once. |
| Sticky Note1 | Sticky Note | Comment / documentation | — | — |  |
| Code - Validate Lead | Code | Validate required fields + spam heuristic | Normalize leads | IF node — Is Valid? | ## Normalize And Validate Lead\nReal data is messy. Trim spaces, lowercase email, create a consistent object once. |
| IF node — Is Valid? | IF | Gate valid vs invalid | Code - Validate Lead | IF node — Enrichment enabled?, Respond to Webhook — 400 Bad Request (invalid path) | ## What this workflow does\n\n1. **Receives a lead** ... |
| Respond to Webhook — 400 Bad Request (invalid path) | Respond to Webhook | Return invalid payload error | IF node — Is Valid? (false) | — | ## What this workflow does\n\n1. **Receives a lead** ... |
| IF node — Enrichment enabled? | IF | Decide whether to enrich | IF node — Is Valid? (true) | Set - Snapshot data + HTTP Request - Enrich (Optional) (true), Set - Merge Enrichment into lead (false) | ## Enrichment check\nCheck if enrichment is enabled... Change `url: https://example.com/` to enrichment provider url |
| HTTP Request - Enrich (Optional) | HTTP Request | Call enrichment provider | IF node — Enrichment enabled? (true) | Merge | ## Enrichment check\nCheck if enrichment is enabled... Change `url: https://example.com/` to enrichment provider url |
| Set - Snapshot data | Set | Preserve lead context for merge | IF node — Enrichment enabled? (true) | Merge | ## Enrichment check\nCheck if enrichment is enabled... |
| Merge | Merge | Combine snapshot + enrichment response | Set - Snapshot data, HTTP Request - Enrich (Optional) | Set - Merge Enrichment into lead | Merge enrichment data along with the lead data, because HTTP node, looses the Input lead data. |
| Set - Merge Enrichment into lead | Set | Attach enrichment into `lead.enrichment` | Merge OR IF node — Enrichment enabled? (false) | Switch - Choose destination (Sheets vs HubSpot) | Merge enrichment data along with the lead data, because HTTP node, looses the Input lead data. |
| Switch - Choose destination (Sheets vs HubSpot) | Switch | Route to Sheets vs HubSpot | Set - Merge Enrichment into lead | Google Sheets- Lookup by email, Hubspot - Create/Update contacts | Switch cases, based on destination where to save the lead data, `Google Sheets` OR `HubSpot` |
| Google Sheets- Lookup by email | Google Sheets | Upsert lead row by email | Switch - Choose destination (Sheets) | IF - Sheets update Failed? | ## Example sheet pattern\n```\nreceivedAt → ={{$json.lead.receivedAt || $now}}\n...\n``` |
| IF - Sheets update Failed? | IF | Detect Sheets write errors | Google Sheets- Lookup by email | Slack - update failed, Slack - Successfully updated | ## What this workflow does\n\n1. **Receives a lead** ... |
| Slack - update failed | Slack | Alert on Sheets failure | IF - Sheets update Failed? (true) | Respond to Webhook - 500 - Sheets | ## What this workflow does\n\n1. **Receives a lead** ... |
| Respond to Webhook - 500 - Sheets | Respond to Webhook | Return 500 for Sheets failure | Slack - update failed | — | ## What this workflow does\n\n1. **Receives a lead** ... |
| Slack - Successfully updated | Slack | Alert on Sheets success | IF - Sheets update Failed? (false) | Respond to Webhook - 200 - Sheets | ## What this workflow does\n\n1. **Receives a lead** ... |
| Respond to Webhook - 200 - Sheets | Respond to Webhook | Return 200 for Sheets success | Slack - Successfully updated | — | ## What this workflow does\n\n1. **Receives a lead** ... |
| Hubspot - Create/Update contacts | HubSpot | Upsert contact by email | Switch - Choose destination (HubSpot) | IF - HubSpot Failed? | Switch cases, based on destination where to save the lead data... |
| IF - HubSpot Failed? | IF | Detect HubSpot write errors | Hubspot - Create/Update contacts | Slack - HubSpot update failed, Slack - HubSpot Successfully updated | ## What this workflow does\n\n1. **Receives a lead** ... |
| Slack - HubSpot update failed | Slack | Alert on HubSpot failure | IF - HubSpot Failed? (true) | Respond to Webhook - 500 - HubSpot | ## What this workflow does\n\n1. **Receives a lead** ... |
| Respond to Webhook - 500 - HubSpot | Respond to Webhook | Return 500 for HubSpot failure | Slack - HubSpot update failed | — | ## What this workflow does\n\n1. **Receives a lead** ... |
| Slack - HubSpot Successfully updated | Slack | Alert on HubSpot success | IF - HubSpot Failed? (false) | Respond to Webhook - 200 - HubSpot | ## What this workflow does\n\n1. **Receives a lead** ... |
| Respond to Webhook - 200 - HubSpot | Respond to Webhook | Return 200 for HubSpot success | Slack - HubSpot Successfully updated | — | ## What this workflow does\n\n1. **Receives a lead** ... |
| Sticky Note2 | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note3 | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note4 | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note5 | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note6 | Sticky Note | Comment / documentation | — | — |  |
| Sticky Note (all others) | Sticky Note | Visual documentation blocks | — | — | (See individual note contents above; duplicated on covered nodes) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Webhook Trigger”**
   - Node: **Webhook**
   - Method: `POST`
   - Path: `website-lead`
   - Response mode: **Using “Respond to Webhook” node**
   - Save, then copy the Production/Test webhook URL for your website form.

2) **Add “Parse Webhook body” (Code)**
   - Node: **Code**
   - Paste logic to parse `$json.body` when it’s a string, else pass through.

3) **Add “Normalize leads” (Set)**
   - Node: **Set**
   - Add fields under `lead.*`:
     - `lead.email` (lowercase + trim)
     - `lead.firstName`, `lead.lastName`, `lead.company`, `lead.message`
     - `lead.source` default `unknown`
     - `lead.destination` default `sheets`
     - `lead.receivedAt` = `$now`
     - `lead.enrich` boolean
   - Ensure your expressions use safe optional chaining if your inputs vary.

4) **Add “Code - Validate Lead”**
   - Node: **Code**
   - Validate: email, message, and at least one of first/last/company.
   - Add spam check (message length threshold).
   - Output `isValid`, `missingFields`, `errorMessage`.

5) **Add “IF node — Is Valid?”**
   - Node: **IF**
   - Condition: boolean `{{$json.isValid}}` is true.
   - False output → step 6
   - True output → step 7

6) **Add “Respond to Webhook — 400 Bad Request”**
   - Node: **Respond to Webhook**
   - Response code: `400`
   - Respond with: JSON including `ok:false`, `error`, `missingFields`
   - Connect from IF(false) to this node.

7) **Add “IF node — Enrichment enabled?”**
   - Node: **IF**
   - Condition: `{{$json.lead.enrich}}` is true
   - True output → two parallel branches (steps 8 and 9)
   - False output → directly to step 12

8) **Add “Set - Snapshot data”**
   - Node: **Set**
   - Enable “Keep Only Set” = false / or “Include other fields” = true (pass-through snapshot).
   - Connect from IF(enrich=true) to this node.

9) **Add “HTTP Request - Enrich (Optional)”**
   - Node: **HTTP Request**
   - Method: POST
   - URL: your provider (replace placeholder `https://example.com`)
   - Timeout: 20000ms
   - Enable:
     - **Continue on Fail** (onError continue)
     - **Always Output Data**
   - Add headers/auth per provider (API key, bearer token, etc.)
   - Connect from IF(enrich=true) to this node.

10) **Add “Merge”**
   - Node: **Merge**
   - Mode: **Combine by Position**
   - Input 1: connect from Set - Snapshot data
   - Input 2: connect from HTTP Request - Enrich (Optional)

11) **Add “Set - Merge Enrichment into lead”**
   - Node: **Set**
   - Set `lead.enrichment` to `{{$json.data || null}}` (adjust if your HTTP node returns a different structure)
   - Optionally remove `data` from output.
   - Connect from Merge → this Set node.
   - Also connect IF(enrich=false) → this Set node (so both paths converge).

12) **Add “Switch - Choose destination (Sheets vs HubSpot)”**
   - Node: **Switch**
   - Rule 1: `lower(lead.destination)` equals `sheets` → output “Google Sheets”
   - Rule 2: `lower(lead.destination)` equals `hubspot` → output “HubSpot”
   - (Recommended) Add a default/else route to a Respond node (422) to avoid webhook timeouts.

13) **Sheets branch: add “Google Sheets- Lookup by email”**
   - Node: **Google Sheets**
   - Credentials: Google (OAuth2) with Sheets scope.
   - Operation: **Append or Update**
   - Select Spreadsheet (Document ID) and Sheet/Tab.
   - Matching column: `email`
   - Map columns:
     - `receivedAt`, `name`, `email`, `company`, `message`, `source`, `destination`, `enrichment`, `updatedAt`, `status`
   - Connect Switch(sheets) → this node.

14) **Add “IF - Sheets update Failed?”**
   - Node: **IF**
   - Condition: `{{$json.error?.message}}` is not empty
   - Connect Sheets node → this IF.

15) **Add Slack nodes for Sheets**
   - Node: **Slack** (OAuth2 credential)
   - One for failure, one for success; configure target (user/channel) and message text.
   - Connect IF(true) → Slack failure → Respond 500 (Sheets)
   - Connect IF(false) → Slack success → Respond 200 (Sheets)

16) **Add “Respond to Webhook - 500 - Sheets” and “Respond to Webhook - 200 - Sheets”**
   - Node: **Respond to Webhook**
   - Response codes 500 and 200 with the JSON bodies as in the workflow.

17) **HubSpot branch: add “Hubspot - Create/Update contacts”**
   - Node: **HubSpot**
   - Credentials: HubSpot OAuth or Private App token credential in n8n.
   - Operation: create/update contact (by email).
   - Email: `{{$json.lead.email}}`
   - (Recommended) Map additional fields (firstname, lastname, company, lead source, message) if desired.
   - Connect Switch(hubspot) → this node.

18) **Add “IF - HubSpot Failed?”**
   - Node: **IF**
   - Condition: `{{$json.error?.message || $json.error}}` not empty
   - Connect HubSpot node → this IF.

19) **Add Slack nodes for HubSpot**
   - Slack failure + Slack success
   - Connect IF(true) → Slack failure → Respond 500 (HubSpot)
   - Connect IF(false) → Slack success → Respond 200 (HubSpot)

20) **Add “Respond to Webhook - 500 - HubSpot” and “Respond to Webhook - 200 - HubSpot”**
   - Node: **Respond to Webhook**
   - Response codes 500 and 200 with JSON bodies.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| In the HTTP enrichment node, change `url: https://example.com/` to your enrichment provider (Clearbit/Hunter/etc.). | Sticky note: “Enrichment check” |
| Merge is required because the enrichment HTTP call path may not preserve the original lead payload in the way you need for later nodes. | Sticky note: “Merge enrichment data…” |
| Example Google Sheets column mapping includes `enrichment = JSON.stringify(...)`, `status = New`, timestamps via `$now`. | Sticky note: “Example sheet pattern” |
| Add a default route for unsupported `destination` values, otherwise the webhook request may time out (no Respond node reached). | Operational reliability note |
| Consider hardening Normalize expressions (use optional chaining everywhere) to avoid `.trim()` on undefined values. | Data-shape robustness note |