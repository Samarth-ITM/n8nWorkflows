Enrich new form leads with Lusha, HubSpot, and Slack alerts

https://n8nworkflows.xyz/workflows/enrich-new-form-leads-with-lusha--hubspot--and-slack-alerts-13342


# Enrich new form leads with Lusha, HubSpot, and Slack alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *Form Enrichment with Lusha*  
**Purpose:** Accept inbound web-form submissions, validate the lead email, check for an existing HubSpot contact, enrich the lead via Lusha (contact + firmographics), upsert the contact in HubSpot, alert SDRs in Slack, and return the enriched payload back to the form submitter via webhook response.

### 1.1 Input Reception & Validation
Receives a POST request from a web form, normalizes/validates the email and standardizes the incoming fields.

### 1.2 Duplicate Check in HubSpot
Searches HubSpot Contacts by email to determine whether the record already exists.

### 1.3 Enrichment via Lusha
Uses Lusha enrichment by email to retrieve phone, role, seniority, LinkedIn, and company details.

### 1.4 Merge + CRM Sync + Slack + Webhook Response
Combines form + Lusha + HubSpot search context into a single enriched object, upserts the contact into HubSpot, posts a formatted Slack message to SDRs, and returns the enriched JSON to the caller.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception & Validation
**Overview:** This block is the entry point. It receives the form payload, validates email presence/format, normalizes casing/whitespace, and outputs a cleaned canonical lead object for downstream steps.

**Nodes involved:**
- `Receive Form Submission` (Webhook)
- `Validate Email` (Code)

#### Node: Receive Form Submission
- **Type / role:** `n8n-nodes-base.webhook` — HTTP entry point for form submissions.
- **Configuration (interpreted):**
  - Method: **POST**
  - Path: **/form-enrichment**
  - Response mode: **Respond via “Respond to Webhook” node** (response is not returned automatically by this node).
- **Inputs / outputs:**
  - **Output →** `Validate Email`
- **Version notes:** Node typeVersion **2**.
- **Edge cases / failures:**
  - If the caller sends unexpected content-type/body shape, downstream code may not find `body.email`.
  - Since response is handled by a response node, failing before the response node typically yields an error response (depending on n8n error handling) and the client may see a non-200.

#### Node: Validate Email
- **Type / role:** `n8n-nodes-base.code` — Validate/normalize inputs and create a consistent schema.
- **Key logic/config:**
  - Reads payload from `const body = $json.body || $json;` to support either raw JSON payloads or webhook-wrapped body objects.
  - Normalizes email: `trim().toLowerCase()`.
  - Validates minimal email structure:
    - Must exist, must contain `@` and `.`
    - On failure: `throw new Error('Invalid or missing email address');`
  - Outputs canonical fields:
    - `email, firstName, lastName, company, source (default 'web-form'), message, submittedAt (ISO timestamp)`
- **Inputs / outputs:**
  - **Input ←** `Receive Form Submission`
  - **Output →** `Check HubSpot Duplicate`
- **Version notes:** typeVersion **2**.
- **Edge cases / failures:**
  - Overly simplistic email validation (e.g., accepts `a@b.` or rejects valid uncommon formats). Consider regex validation if needed.
  - If upstream form uses different field names (e.g., `first_name`), fields will be empty unless mapped.

---

### Block 1.2 — Duplicate Check in HubSpot
**Overview:** This block searches HubSpot contacts by email to detect duplicates and provide context for later messaging/merge logic.

**Nodes involved:**
- `Check HubSpot Duplicate` (HubSpot)

#### Node: Check HubSpot Duplicate
- **Type / role:** `n8n-nodes-base.hubspot` — Query HubSpot CRM for existing contact(s).
- **Configuration (interpreted):**
  - Resource: **Contact**
  - Operation: **Search**
  - Filter: `email EQ {{$json.email}}`
  - Credentials: **HubSpot OAuth2**
- **Key expressions/variables:**
  - Uses the validated output email: `={{ $json.email }}`
- **Inputs / outputs:**
  - **Input ←** `Validate Email`
  - **Output →** `Enrich with Lusha`
- **Version notes:** typeVersion **2**.
- **Edge cases / failures:**
  - OAuth2 token expired/insufficient scopes → authentication errors.
  - HubSpot search API rate limits.
  - If HubSpot returns unexpected structure, downstream merge assumes:
    - `hubspotResults.total`
    - `hubspotResults.results[0].id`

---

### Block 1.3 — Enrichment via Lusha
**Overview:** This block enriches the lead using Lusha by email, returning contact and company attributes.

**Nodes involved:**
- `Enrich with Lusha` (Lusha community node)

#### Node: Enrich with Lusha
- **Type / role:** `@lusha-org/n8n-nodes-lusha.lusha` — Lusha enrichment request.
- **Configuration (interpreted):**
  - Operation: **enrichSingle**
  - Search by: **email**
  - Email: pulled from validated form data: `$('Validate Email').first().json.email`
  - Credentials: **Lusha API**
- **Key expressions/variables:**
  - `={{ $('Validate Email').first().json.email }}`
- **Inputs / outputs:**
  - **Input ←** `Check HubSpot Duplicate`
  - **Output →** `Merge Form + Lusha Data`
- **Version notes:** typeVersion **1** (community node; ensure installed).
- **Edge cases / failures:**
  - Community node not installed or not allowed on n8n environment.
  - Lusha API quota exhausted / 402-like credit issues (varies by plan).
  - No enrichment found: may return partial/empty fields; merge node must tolerate missing properties.

---

### Block 1.4 — Merge + CRM Sync + Slack + Webhook Response
**Overview:** This block merges all data into one enriched lead object, upserts it to HubSpot, alerts SDRs in Slack, and responds to the original webhook request with enriched JSON.

**Nodes involved:**
- `Merge Form + Lusha Data` (Code)
- `Create/Update HubSpot Contact` (HubSpot)
- `Alert SDR on Slack` (Slack)
- `Return Enriched Lead` (Respond to Webhook)

#### Node: Merge Form + Lusha Data
- **Type / role:** `n8n-nodes-base.code` — Build the final enriched object and compute “existing vs new” flags.
- **Key logic/config:**
  - Loads:
    - `form` from `Validate Email`
    - `lusha` from current node input (`$json`)
    - `hubspotResults` from `Check HubSpot Duplicate`
  - Determines existing contact:
    - `isExisting = hubspotResults.total > 0`
    - `existingId = hubspotResults.results?.[0]?.id || null`
  - Produces merged output with sections:
    - Identity: email, firstName, lastName (prefers form over Lusha)
    - Enrichment: jobTitle, seniority, primaryPhone, primaryEmail, LinkedIn
    - Company: companyName/domain/size/industry/revenue/location
    - Metadata: isExistingContact, existingContactId, source, message, enrichedAt, enrichmentSource='lusha'
- **Inputs / outputs:**
  - **Input ←** `Enrich with Lusha`
  - **Outputs →** (three parallel)
    - `Create/Update HubSpot Contact`
    - `Alert SDR on Slack`
    - `Return Enriched Lead`
- **Version notes:** typeVersion **2**.
- **Edge cases / failures:**
  - If HubSpot search output does not include `total`, `hubspotResults.total > 0` may throw; consider defensive defaults.
  - If Lusha returns nested structures rather than flat (depends on node implementation/version), property paths may not match.

#### Node: Create/Update HubSpot Contact
- **Type / role:** `n8n-nodes-base.hubspot` — Upsert contact in HubSpot using email as key.
- **Configuration (interpreted):**
  - Resource: **Contact**
  - Operation: **Upsert**
  - Upsert key: **Email** `={{ $json.email }}`
  - Additional fields mapped:
    - `phone, company, website (companyDomain), industry, jobtitle, firstName, lastName`
  - Credentials: **HubSpot OAuth2**
- **Key expressions/variables:**
  - Uses merged object fields, e.g. `={{ $json.phone }}`
- **Inputs / outputs:**
  - **Input ←** `Merge Form + Lusha Data`
  - **No downstream node connected** (side-effect operation).
- **Version notes:** typeVersion **2**.
- **Edge cases / failures:**
  - Missing scopes for contact write.
  - HubSpot property name mismatches:
    - `jobtitle` is a standard property, but custom portals may differ.
    - `website` expects a URL; here it receives a domain string.
  - Validation errors if HubSpot enforces formats (phone, website).

#### Node: Alert SDR on Slack
- **Type / role:** `n8n-nodes-base.slack` — Send an SDR notification message.
- **Configuration (interpreted):**
  - Posts a formatted text message to channel `#inbound-leads`
  - Message includes:
    - Name, title, seniority, company
    - Best email: `verifiedEmail || email`
    - Phone (or N/A), LinkedIn (or N/A)
    - Existing vs new indicator based on `isExistingContact`
  - Credentials: **Slack OAuth2**
- **Key expressions/variables:**
  - References merged node item explicitly:
    - `{{ $('Merge Form + Lusha Data').item.json.firstName }}`
  - Conditional in-text:
    - `{{ condition ? '♻️ ...' : '✨ ...' }}`
- **Inputs / outputs:**
  - **Input ←** `Merge Form + Lusha Data`
  - **No downstream node connected** (notification side-effect).
- **Version notes:** typeVersion **2**.
- **Edge cases / failures:**
  - Slack OAuth scopes missing (e.g., `chat:write`).
  - Channel not found or bot not invited to channel.
  - Message rendering: if fields are empty, output may look incomplete (consider fallback text for job title/company).

#### Node: Return Enriched Lead
- **Type / role:** `n8n-nodes-base.respondToWebhook` — Returns HTTP response to the original webhook request.
- **Configuration (interpreted):**
  - Response code: **200**
  - Respond with: **JSON**
  - Body: the merged JSON object: `$('Merge Form + Lusha Data').item.json`
- **Inputs / outputs:**
  - **Input ←** `Merge Form + Lusha Data`
  - **Terminates the webhook response**
- **Version notes:** typeVersion **1**.
- **Edge cases / failures:**
  - If execution errors occur before this node runs, the client won’t receive the enriched JSON.
  - If multiple items are processed, `.item` semantics matter; current design assumes a single lead per request.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Form Lead Enrichment + CRM Sync | Sticky Note | Documentation / grouping |  |  | ## Enrich New Form Leads with Lusha  \n\n**Who it's for:** Marketing & Growth teams capturing inbound leads  \n\n**What it does:** Enriches web form submissions in real time with Lusha, checks for duplicates in HubSpot, creates or updates the CRM record, sends an SDR alert to Slack, and returns the enriched data to your form.\n\n### How it works\n1. Receives form submission via webhook\n2. Validates the email and checks for duplicates in HubSpot\n3. Enriches the contact with Lusha (phone, title, seniority, company data)\n4. Merges form + Lusha data and creates/updates the HubSpot contact\n5. Sends an SDR notification to Slack with the enriched lead summary\n6. Returns the enriched record to the form as a JSON response\n\n### Setup\n1. Install the [Lusha community node](https://www.npmjs.com/package/@lusha-org/n8n-nodes-lusha)\n2. Add your Lusha API, HubSpot OAuth2, and Slack credentials\n3. Point your web form's action URL to the webhook endpoint\n4. Customize the Slack channel and CRM field mapping |
| ⚡ 1. Receive & Validate | Sticky Note | Documentation / block label |  |  | Webhook receives a POST with the form submission. A code node validates the email format and checks for duplicates in HubSpot before enrichment.\n\n**Nodes:** Webhook → Validate Email → Check HubSpot Duplicate\n\n💡 Customize the duplicate-check field to match your CRM setup. |
| 🔍 2. Enrich with Lusha | Sticky Note | Documentation / block label |  |  | Lusha enriches the contact by email, returning verified phone, job title, seniority, LinkedIn, and full company firmographics.\n\n**Nodes:** Lusha Contact Enrich → Merge Form + Lusha Data\n\n📖 [Lusha API docs](https://www.lusha.com/docs/) |
| 💾 3. CRM + Slack + Response | Sticky Note | Documentation / block label |  |  | Creates or updates the HubSpot contact record, sends an SDR alert to Slack with the full enriched lead summary, and returns the enriched data to the form.\n\n**Nodes:** Upsert HubSpot Contact → Slack SDR Alert → Webhook Response\n\n💡 Customize the Slack channel (#inbound-leads) and message format. |
| Receive Form Submission | Webhook | Receive form lead POST | (Entry point) | Validate Email | (Covered by: 📋 Form Lead Enrichment + CRM Sync) \n(Covered by: ⚡ 1. Receive & Validate) |
| Validate Email | Code | Normalize payload + validate email | Receive Form Submission | Check HubSpot Duplicate | (Covered by: 📋 Form Lead Enrichment + CRM Sync) \n(Covered by: ⚡ 1. Receive & Validate) |
| Check HubSpot Duplicate | HubSpot | Search existing contact by email | Validate Email | Enrich with Lusha | (Covered by: 📋 Form Lead Enrichment + CRM Sync) \n(Covered by: ⚡ 1. Receive & Validate) |
| Enrich with Lusha | Lusha (community) | Enrich lead by email | Check HubSpot Duplicate | Merge Form + Lusha Data | (Covered by: 📋 Form Lead Enrichment + CRM Sync) \n(Covered by: 🔍 2. Enrich with Lusha) |
| Merge Form + Lusha Data | Code | Merge form + enrichment + duplicate context | Enrich with Lusha | Create/Update HubSpot Contact; Alert SDR on Slack; Return Enriched Lead | (Covered by: 📋 Form Lead Enrichment + CRM Sync) \n(Covered by: 🔍 2. Enrich with Lusha) |
| Create/Update HubSpot Contact | HubSpot | Upsert contact in HubSpot | Merge Form + Lusha Data |  | (Covered by: 📋 Form Lead Enrichment + CRM Sync) \n(Covered by: 💾 3. CRM + Slack + Response) |
| Alert SDR on Slack | Slack | Notify SDR channel with enriched summary | Merge Form + Lusha Data |  | (Covered by: 📋 Form Lead Enrichment + CRM Sync) \n(Covered by: 💾 3. CRM + Slack + Response) |
| Return Enriched Lead | Respond to Webhook | Return enriched JSON payload | Merge Form + Lusha Data |  | (Covered by: 📋 Form Lead Enrichment + CRM Sync) \n(Covered by: 💾 3. CRM + Slack + Response) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Form Enrichment with Lusha**

2. **Add Webhook node**
   - Node type: **Webhook**
   - Name: `Receive Form Submission`
   - HTTP Method: **POST**
   - Path: `form-enrichment`
   - Response: set to **Respond with “Respond to Webhook” node** (responseMode = responseNode)

3. **Add Code node for validation**
   - Node type: **Code**
   - Name: `Validate Email`
   - Paste logic to:
     - Read `body = $json.body || $json`
     - Normalize `email = (body.email || '').trim().toLowerCase()`
     - Throw error if invalid/missing
     - Output normalized fields (`firstName`, `lastName`, `company`, `source`, `message`, `submittedAt`)
   - Connect: `Receive Form Submission` → `Validate Email`

4. **Add HubSpot node to search duplicates**
   - Node type: **HubSpot**
   - Name: `Check HubSpot Duplicate`
   - Resource: **Contact**
   - Operation: **Search**
   - Filter group:
     - Property: `email`
     - Operator: `EQ`
     - Value: `{{$json.email}}`
   - Credentials:
     - Create/select **HubSpot OAuth2** credential
     - Ensure scopes allow CRM read (and later write for upsert)
   - Connect: `Validate Email` → `Check HubSpot Duplicate`

5. **Install and add Lusha community node**
   - Install on your n8n instance: `@lusha-org/n8n-nodes-lusha`
     - Link: https://www.npmjs.com/package/@lusha-org/n8n-nodes-lusha
   - Node type: **Lusha**
   - Name: `Enrich with Lusha`
   - Operation: **enrichSingle**
   - Search by: **email**
   - Email expression: `{{$('Validate Email').first().json.email}}`
   - Credentials:
     - Create/select **Lusha API** credential (API key/token as required by the node)
   - Connect: `Check HubSpot Duplicate` → `Enrich with Lusha`

6. **Add Code node to merge outputs**
   - Node type: **Code**
   - Name: `Merge Form + Lusha Data`
   - Implement merge logic:
     - Read `form` from `Validate Email`
     - Read `hubspotResults` from `Check HubSpot Duplicate`
     - Read `lusha` from `$json`
     - Compute:
       - `isExistingContact = hubspotResults.total > 0`
       - `existingContactId = hubspotResults.results?.[0]?.id || null`
     - Output a unified JSON object with identity, enrichment, company fields, and metadata (`enrichedAt`, `enrichmentSource`)
   - Connect: `Enrich with Lusha` → `Merge Form + Lusha Data`

7. **Add HubSpot upsert node**
   - Node type: **HubSpot**
   - Name: `Create/Update HubSpot Contact`
   - Resource: **Contact**
   - Operation: **Upsert**
   - Email: `{{$json.email}}`
   - Additional fields mapping (adjust to your CRM properties):
     - phone `{{$json.phone}}`
     - company `{{$json.company}}`
     - website `{{$json.companyDomain}}`
     - industry `{{$json.industry}}`
     - jobtitle `{{$json.jobTitle}}`
     - firstname `{{$json.firstName}}`
     - lastname `{{$json.lastName}}`
   - Credentials: same **HubSpot OAuth2** (must include write scopes)
   - Connect: `Merge Form + Lusha Data` → `Create/Update HubSpot Contact`

8. **Add Slack alert node**
   - Node type: **Slack**
   - Name: `Alert SDR on Slack`
   - Operation: **Post message** (text-based)
   - Channel: `#inbound-leads` (change as needed)
   - Text: build message using expressions referencing `Merge Form + Lusha Data` fields (name, title, company, email, phone, LinkedIn, existing/new flag).
   - Credentials:
     - Create/select **Slack OAuth2**
     - Ensure app has `chat:write` and is in the target channel
   - Connect: `Merge Form + Lusha Data` → `Alert SDR on Slack`

9. **Add Respond to Webhook node**
   - Node type: **Respond to Webhook**
   - Name: `Return Enriched Lead`
   - Respond with: **JSON**
   - Response code: **200**
   - Response body expression: `{{$('Merge Form + Lusha Data').item.json}}`
   - Connect: `Merge Form + Lusha Data` → `Return Enriched Lead`

10. **Test end-to-end**
   - Execute the webhook test URL with a JSON payload including at least `email` (and optionally `firstName`, `lastName`, `company`, `source`, `message`).
   - Confirm:
     - HubSpot contact is created/updated
     - Slack message posts successfully
     - Webhook response returns enriched JSON

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install the Lusha community node package | https://www.npmjs.com/package/@lusha-org/n8n-nodes-lusha |
| Lusha API documentation | https://www.lusha.com/docs/ |
| Suggested customization points: duplicate-check field (HubSpot), Slack channel `#inbound-leads`, HubSpot property mapping | From workflow sticky notes |
| Web form must POST to the webhook endpoint path `/form-enrichment` | From workflow sticky notes and Webhook configuration |