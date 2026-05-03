Enrich chatbot and demo request leads with Lusha, HubSpot, and Slack

https://n8nworkflows.xyz/workflows/enrich-chatbot-and-demo-request-leads-with-lusha--hubspot--and-slack-13347


# Enrich chatbot and demo request leads with Lusha, HubSpot, and Slack

## 1. Workflow Overview

**Purpose:** Receive inbound chatbot messages or demo requests, validate the sender’s email, enrich the lead with **Lusha**, compute an **ICP-style priority score**, upsert the contact into **HubSpot**, then route notifications to **Slack** with different urgency based on priority.

**Primary use cases:**
- SDR teams handling inbound leads from chat widgets and demo request forms
- Automated enrichment + immediate routing to the right Slack channels
- Consistent HubSpot contact creation/update using enriched data

### 1.1 Input Reception & Validation
Receives a POST payload via webhook, validates/normalizes email, and extracts form fields into a clean internal schema.

### 1.2 Enrichment & Lead Scoring
Uses Lusha to enrich the lead by email, then calculates a priority score based on seniority, company size, request type, and phone availability.

### 1.3 CRM Upsert & Priority-Based Slack Routing
Upserts the contact in HubSpot and sends either an urgent or standard Slack message depending on `isHighPriority`.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Validation

**Overview:** Accepts inbound HTTP requests and ensures the workflow only continues when an email exists and looks valid. Produces a normalized lead object for downstream steps.

**Nodes involved:**
- `Chatbot/Demo Request Webhook`
- `Validate & Clean Input`

#### Node: Chatbot/Demo Request Webhook
- **Type / role:** `Webhook` — entry point trigger for inbound leads
- **Configuration (interpreted):**
  - Method: **POST**
  - Path: **/chat-demo-enrichment**
  - Expects chatbot/demo form data in request body
- **Key fields/variables:** Outputs request data (commonly under `$json.body` depending on n8n/webhook configuration and caller)
- **Connections:**
  - Output → `Validate & Clean Input`
- **Version notes:** Node `typeVersion: 2`
- **Edge cases / failures:**
  - Callers sending non-JSON bodies or unexpected structure (may cause the next Code node to miss fields)
  - Missing `Content-Type: application/json` from the sender (depending on sender platform)
  - Public webhook exposure: consider authentication, secret path, IP allowlist, or verification token

#### Node: Validate & Clean Input
- **Type / role:** `Code` — input normalization and basic validation gate
- **Configuration (interpreted):**
  - Reads payload from `const body = $json.body || $json;`
  - Normalizes email: trim + lowercase
  - Validates email with a simple `includes('@')` check
  - Builds a canonical internal object with defaults:
    - `requestType` defaults to `"demo"`
    - `source` defaults to `"chatbot"`
    - `receivedAt` set to current ISO timestamp
- **Key expressions/variables used:**
  - `body.email`, `body.firstName`, `body.lastName`, `body.company`, `body.requestType`, `body.message`, `body.source`
- **Connections:**
  - Input ← `Chatbot/Demo Request Webhook`
  - Output → `Enrich with Lusha`
- **Version notes:** Node `typeVersion: 2`
- **Edge cases / failures:**
  - Throws hard error: `Invalid or missing email in demo request` if email is missing/invalid → workflow run fails
  - Email validation is minimal (allows many invalid addresses); consider stricter regex or additional checks
  - If chatbot sends nested structures (e.g., `body.user.email`), mapping must be adjusted

---

### Block 2 — Enrichment & Lead Scoring

**Overview:** Enriches the lead using Lusha by email, then computes a numeric priority score and a boolean `isHighPriority` used for routing and message content.

**Nodes involved:**
- `Enrich with Lusha`
- `Build Summary & Score`

#### Node: Enrich with Lusha
- **Type / role:** `@lusha-org/n8n-nodes-lusha.lusha` — third-party community node for Lusha enrichment
- **Configuration (interpreted):**
  - Operation: **enrichSingle**
  - Search by: **email**
  - Email value: `{{$json.email}}` (from validated input)
  - Additional options: none set
- **Credentials:**
  - Uses `Lusha API` credential (API key–based, as provided by the community node)
- **Connections:**
  - Input ← `Validate & Clean Input`
  - Output → `Build Summary & Score`
- **Version notes:** Node `typeVersion: 1` (community node versioning may differ from core n8n nodes)
- **Edge cases / failures:**
  - Credential/auth errors (invalid API key)
  - Rate limits / quota exhaustion from Lusha
  - No enrichment found for email → returned fields may be empty/undefined; downstream code must tolerate missing properties
  - Community node installation required on the n8n instance (won’t run on n8n Cloud unless community nodes are supported/enabled for that environment)

#### Node: Build Summary & Score
- **Type / role:** `Code` — merges form + enrichment, computes priority, prepares canonical enriched lead object
- **Configuration (interpreted):**
  - Pulls original validated form data explicitly:
    - `const form = $('Validate & Clean Input').first().json;`
  - Treats the current node input (`$json`) as the Lusha enrichment payload:
    - `const lusha = $json;`
  - **Scoring logic:**
    - Seniority (string match, lowercased):
      - C-level/founder/owner/cxo → +40
      - VP/director → +30
      - manager → +15
    - Company size:
      - Uses `empMax = Array.isArray(lusha.companySize) ? lusha.companySize[1] : 0;`
      - `>= 500` → +25; `>=100` → +15; `>=20` → +5
    - Request type:
      - if `form.requestType === 'demo'` → +15
    - Phone availability:
      - if `lusha.primaryPhone` present → +10
    - `isHighPriority = priority >= 50`
  - Output fields include merged data for HubSpot + Slack:
    - Contact: `firstName`, `lastName`, `email`, `phone`, `jobTitle`, `seniority`, `linkedinUrl`
    - Company: `company`, `companyDomain`, `industry`, `companySize`, `revenueRange`
    - Meta: `requestType`, `message`, `source`, `priorityScore`, `isHighPriority`, `enrichedAt`
- **Key expressions/variables used:**
  - Node reference: `$('Validate & Clean Input').first().json`
  - Lusha fields referenced: `seniority`, `companySize`, `primaryPhone`, `primaryEmail`, `firstName`, `lastName`, `jobTitle`, `linkedinProfile`, `companyName`, `companyDomain`, `companyMainIndustry`, `revenueRange`
- **Connections:**
  - Input ← `Enrich with Lusha`
  - Output → `Create/Update HubSpot Contact`
- **Version notes:** Node `typeVersion: 2`
- **Edge cases / failures:**
  - If Lusha returns `companySize` not as an array `[min,max]`, scoring defaults to `0` and may under-score
  - Seniority matching depends on Lusha’s exact text values; mismatches reduce scoring
  - `$('Validate & Clean Input').first()` assumes at least one item exists; if upstream fails or returns no items, this throws

---

### Block 3 — CRM Upsert & Priority-Based Slack Routing

**Overview:** Writes enriched lead data to HubSpot (upsert by email), then routes to Slack with different message templates/channels based on `isHighPriority`.

**Nodes involved:**
- `Create/Update HubSpot Contact`
- `Is High Priority?`
- `Urgent SDR Alert`
- `Standard SDR Notification`

#### Node: Create/Update HubSpot Contact
- **Type / role:** `HubSpot` — CRM write (create or update contact)
- **Configuration (interpreted):**
  - Resource: **contact**
  - Operation: **upsert** (idempotent by email)
  - Email: `{{$json.email}}`
  - Additional fields mapped:
    - `phone` ← `{{$json.phone}}`
    - `company` ← `{{$json.company}}`
    - `website` ← `{{$json.companyDomain}}`
    - `industry` ← `{{$json.industry}}`
    - `jobtitle` ← `{{$json.jobTitle}}`
    - `firstname` ← `{{$json.firstName}}`
    - `lastname` ← `{{$json.lastName}}`
- **Credentials:**
  - `HubSpot OAuth2` credential
- **Connections:**
  - Input ← `Build Summary & Score`
  - Output → `Is High Priority?`
- **Version notes:** Node `typeVersion: 2`
- **Edge cases / failures:**
  - OAuth token expiry/revocation
  - HubSpot property name mismatches (e.g., `jobtitle` is standard, but custom properties would require exact internal names)
  - If HubSpot rate limits or API errors occur, Slack routing won’t run (because it’s downstream)

#### Node: Is High Priority?
- **Type / role:** `IF` — branching decision for Slack routing
- **Configuration (interpreted):**
  - Condition checks boolean “true”:
    - Left value: `={{ $('Build Summary & Score').item.json.isHighPriority }}`
    - Operation: boolean is true
- **Connections:**
  - Input ← `Create/Update HubSpot Contact`
  - True output → `Urgent SDR Alert`
  - False output → `Standard SDR Notification`
- **Version notes:** Node `typeVersion: 2`
- **Edge cases / failures:**
  - Uses `$('Build Summary & Score').item.json...` instead of `$json.isHighPriority`; if multiple items are processed, `.item` resolution can be tricky depending on pairing
  - If HubSpot node changes item structure, referencing a prior node by `.item` usually still works, but `$json` would be safer for the current item

#### Node: Urgent SDR Alert
- **Type / role:** `Slack` — sends high-priority alert
- **Configuration (interpreted):**
  - Channel: `#urgent-leads`
  - Message template includes:
    - Priority score, name, title/seniority
    - Company + employee count
    - Verified email fallback to submitted email
    - Phone fallback to “N/A”
    - Message text and source
  - Pulls all values from `Build Summary & Score` using expressions like:
    - `{{ $('Build Summary & Score').item.json.priorityScore }}`
- **Credentials:** `Slack OAuth2`
- **Connections:**
  - Input ← `Is High Priority?` (true branch)
- **Version notes:** Node `typeVersion: 2`
- **Edge cases / failures:**
  - Slack channel not found / app not permitted to post
  - Rate limits
  - Template relies on `Build Summary & Score` fields; missing enrichment yields blanks but should still send

#### Node: Standard SDR Notification
- **Type / role:** `Slack` — sends normal alert for non-urgent leads
- **Configuration (interpreted):**
  - Channel: `#inbound-leads`
  - Message includes:
    - Request type, name, title/company
    - Verified email fallback, phone fallback
    - Priority score
- **Credentials:** `Slack OAuth2`
- **Connections:**
  - Input ← `Is High Priority?` (false branch)
- **Version notes:** Node `typeVersion: 2`
- **Edge cases / failures:**
  - Same Slack permission/rate-limit considerations as above

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Chatbot & Demo Request Enrichment | Sticky Note | Documentation / canvas annotation | — | — | ## Enrich Chatbot and Demo Request Leads with Lusha … Setup includes installing Lusha community node and adding credentials. Link: https://www.npmjs.com/package/@lusha-org/n8n-nodes-lusha |
| ⚡ 1. Receive & Validate | Sticky Note | Documentation / block label | — | — | Webhook receives the chatbot/demo form POST… Customize expected payload fields to match your chatbot. |
| 🔍 2. Enrich & Score | Sticky Note | Documentation / block label | — | — | Lusha enriches… calculates a priority score… Link: https://www.lusha.com/docs/ |
| 💾 3. CRM + Priority Routing | Sticky Note | Documentation / block label | — | — | HubSpot contact is created/updated… Customize Slack channels and priority thresholds. |
| Chatbot/Demo Request Webhook | Webhook | Receive inbound chatbot/demo requests | — | Validate & Clean Input | Webhook receives the chatbot/demo form POST… Customize expected payload fields to match your chatbot. |
| Validate & Clean Input | Code | Validate email + normalize inbound payload | Chatbot/Demo Request Webhook | Enrich with Lusha | Webhook receives the chatbot/demo form POST… Customize expected payload fields to match your chatbot. |
| Enrich with Lusha | Lusha (community) | Enrich contact/company data by email | Validate & Clean Input | Build Summary & Score | Lusha enriches… calculates a priority score… Link: https://www.lusha.com/docs/ |
| Build Summary & Score | Code | Merge form + Lusha data; compute priority | Enrich with Lusha | Create/Update HubSpot Contact | Lusha enriches… calculates a priority score… Link: https://www.lusha.com/docs/ |
| Create/Update HubSpot Contact | HubSpot | Upsert contact record in HubSpot | Build Summary & Score | Is High Priority? | HubSpot contact is created/updated… Customize Slack channels and priority thresholds. |
| Is High Priority? | IF | Route to urgent vs standard Slack path | Create/Update HubSpot Contact | Urgent SDR Alert; Standard SDR Notification | HubSpot contact is created/updated… Customize Slack channels and priority thresholds. |
| Urgent SDR Alert | Slack | Post urgent notification to SDR channel | Is High Priority? (true) | — | HubSpot contact is created/updated… Customize Slack channels and priority thresholds. |
| Standard SDR Notification | Slack | Post standard inbound lead notification | Is High Priority? (false) | — | HubSpot contact is created/updated… Customize Slack channels and priority thresholds. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named: `Chat & Demo Request Enrichment with Lusha`.

2. **Add Webhook node**
   - Node type: **Webhook**
   - Name: `Chatbot/Demo Request Webhook`
   - Method: **POST**
   - Path: `chat-demo-enrichment`
   - Keep defaults for response unless your chatbot requires a specific response payload.

3. **Add Code node for validation**
   - Node type: **Code**
   - Name: `Validate & Clean Input`
   - Paste logic to:
     - Read `const body = $json.body || $json;`
     - Normalize `email`
     - Throw error if missing/invalid
     - Return fields: `email, firstName, lastName, company, requestType, message, source, receivedAt`
   - Connect: `Chatbot/Demo Request Webhook` → `Validate & Clean Input`.

4. **Install and enable the Lusha community node**
   - Install package on the n8n host: `@lusha-org/n8n-nodes-lusha`
   - Restart n8n (required for community node availability).

5. **Add Lusha enrichment node**
   - Node type: **Lusha** (from the community nodes list)
   - Name: `Enrich with Lusha`
   - Operation: **Enrich Single**
   - Search by: **Email**
   - Email expression: `{{$json.email}}`
   - Credentials: create/select **Lusha API** credential (API key from Lusha)
   - Connect: `Validate & Clean Input` → `Enrich with Lusha`.

6. **Add scoring/summary Code node**
   - Node type: **Code**
   - Name: `Build Summary & Score`
   - Implement logic to:
     - Load form data from `$('Validate & Clean Input').first().json`
     - Use current `$json` as Lusha payload
     - Compute `priorityScore` and `isHighPriority` (`priority >= 50`)
     - Output merged fields needed for HubSpot and Slack
   - Connect: `Enrich with Lusha` → `Build Summary & Score`.

7. **Add HubSpot Upsert node**
   - Node type: **HubSpot**
   - Name: `Create/Update HubSpot Contact`
   - Resource: **Contact**
   - Operation: **Upsert**
   - Email: `{{$json.email}}`
   - Map additional fields:
     - Phone, Company, Website (domain), Industry, Job Title, First Name, Last Name
   - Credentials: create/select **HubSpot OAuth2** credential and complete OAuth authorization
   - Connect: `Build Summary & Score` → `Create/Update HubSpot Contact`.

8. **Add IF node for priority routing**
   - Node type: **IF**
   - Name: `Is High Priority?`
   - Condition:
     - Boolean “is true”
     - Left value: `{{ $('Build Summary & Score').item.json.isHighPriority }}`
   - Connect: `Create/Update HubSpot Contact` → `Is High Priority?`.

9. **Add Slack node (urgent path)**
   - Node type: **Slack**
   - Name: `Urgent SDR Alert`
   - Operation: **Post message** (send text)
   - Channel: `#urgent-leads`
   - Message: include priority score and enriched lead details using expressions from `Build Summary & Score`
   - Credentials: create/select **Slack OAuth2** credential (ensure scopes to post in channels)
   - Connect: `Is High Priority?` (true output) → `Urgent SDR Alert`.

10. **Add Slack node (standard path)**
    - Node type: **Slack**
    - Name: `Standard SDR Notification`
    - Channel: `#inbound-leads`
    - Message: include request type, identity, email/phone, score
    - Use the same Slack OAuth2 credential
    - Connect: `Is High Priority?` (false output) → `Standard SDR Notification`.

11. **(Optional but recommended) Harden production behavior**
    - Add webhook authentication/verification (secret token in header, HMAC signature, etc.)
    - Add “Continue On Fail” or error-handling branch where appropriate (e.g., Lusha failure → still upsert minimal HubSpot contact and notify Slack)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install the Lusha community node: `@lusha-org/n8n-nodes-lusha` | https://www.npmjs.com/package/@lusha-org/n8n-nodes-lusha |
| Lusha API documentation | https://www.lusha.com/docs/ |
| Workflow intent: enrich inbound chatbot/demo leads, score by ICP fit, upsert to HubSpot, alert Slack with urgency routing | From the main canvas note “Enrich Chatbot and Demo Request Leads with Lusha” |