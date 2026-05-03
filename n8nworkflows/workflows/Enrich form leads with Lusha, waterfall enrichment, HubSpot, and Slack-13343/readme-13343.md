Enrich form leads with Lusha, waterfall enrichment, HubSpot, and Slack

https://n8nworkflows.xyz/workflows/enrich-form-leads-with-lusha--waterfall-enrichment--hubspot--and-slack-13343


# Enrich form leads with Lusha, waterfall enrichment, HubSpot, and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Form Enrichment + Waterfall Fallback  
**Purpose:** Enrich inbound form leads with **Lusha first** (primary provider). If Lusha returns incomplete contactability data (missing **email or phone**), the workflow calls a **fallback enrichment provider** via HTTP. The resulting data is merged into a single “clean” contact record, **upserted to HubSpot**, and an **SDR alert is posted to Slack**. Finally, the enriched record is returned to the submitting form via a webhook response.

**Primary use cases**
- Growth/RevOps teams maximizing enrichment coverage while controlling enrichment API spend.
- Waterfall enrichment strategy (paid provider first, fallback only when needed).
- Real-time form enrichment with immediate JSON response.

### Logical blocks
**1.1 Input Reception & Validation**  
Receives form data via webhook, normalizes/validates the email, and builds a baseline lead payload.

**1.2 Primary Enrichment (Lusha)**  
Calls Lusha enrichment using the validated email.

**1.3 Waterfall Decision + Fallback Enrichment**  
Checks whether Lusha returned a primary email or phone. If missing, calls a fallback HTTP provider.

**1.4 Data Merge (Clean Record Assembly)**  
Merges form + Lusha + fallback (if used) into one normalized record.

**1.5 CRM Sync + SDR Notification + Webhook Response**  
Upserts to HubSpot, posts a Slack message, and returns the enriched JSON to the caller.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Validation
**Overview:** Accepts a POST submission, extracts the payload, validates/normalizes the email, and outputs a clean baseline object used downstream.

**Nodes involved:**
- Receive Form Submission
- Validate Email

#### Node: Receive Form Submission
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — entry point for form POST submissions.
- **Configuration (interpreted):**
  - **HTTP method:** POST
  - **Path:** `form-enrichment-waterfall`
  - **Response mode:** “Response Node” (the workflow must end with a Respond to Webhook node)
- **Inputs / outputs:**
  - **Output →** Validate Email
- **Version notes:** Node typeVersion 2.
- **Edge cases / failures:**
  - If the caller expects an immediate response but the workflow errors before the response node, the request may time out.
  - Payload shape depends on the form tool; downstream code assumes `body` may exist.

#### Node: Validate Email
- **Type / role:** `Code` — validates required fields and standardizes input.
- **Configuration (interpreted):**
  - Reads `const body = $json.body || $json;`
  - Normalizes email: trim + lowercase.
  - Rejects if missing or does not contain `@` by throwing: `Invalid or missing email`.
  - Outputs a baseline JSON:
    - `email`, `firstName`, `lastName`, `company`, `source` (default `web-form`)
- **Key variables / expressions:**
  - Uses `$json.body || $json` to handle webhook payload differences.
- **Inputs / outputs:**
  - **Input ←** Receive Form Submission
  - **Output →** Enrich with Lusha
- **Version notes:** Node typeVersion 2.
- **Edge cases / failures:**
  - Simple validation only; accepts many invalid emails that contain `@` (e.g., `a@b`).
  - If the form sends nested fields with different keys, these fields will be empty.
  - Throwing an error will fail the entire webhook request (no response node executed unless you add error handling).

---

### 2.2 Primary Enrichment (Lusha)
**Overview:** Uses Lusha as the first enrichment source for the lead, queried by email.

**Nodes involved:**
- Enrich with Lusha
- Missing Email or Phone?

#### Node: Enrich with Lusha
- **Type / role:** `Lusha` community node (`@lusha-org/n8n-nodes-lusha.lusha`) — enriches a single contact.
- **Configuration (interpreted):**
  - **Operation:** Enrich Single
  - **Search by:** email
  - **Email input:** `{{ $json.email }}`
  - Additional options: present but empty (`contactEnrichAdditionalOptions: {}`)
- **Credentials:** Lusha API credential (named “Lusha API” in the workflow).
- **Inputs / outputs:**
  - **Input ←** Validate Email
  - **Output →** Missing Email or Phone?
- **Version notes:** Node typeVersion 1; requires installing the community node package.
- **Edge cases / failures:**
  - Lusha auth errors (invalid API key), quota/credit exhaustion, rate limiting.
  - Lusha response field names must match downstream expectations (`primaryEmail`, `primaryPhone`, etc.). If the node returns different field names (version changes), IF logic and merge code can break.
  - Network timeouts/5xx from Lusha will fail the execution unless handled.

#### Node: Missing Email or Phone?
- **Type / role:** `IF` — waterfall gate to decide whether to call the fallback provider.
- **Configuration (interpreted):**
  - **Condition combinator:** OR
  - True if **either** is empty:
    - `{{ $json.primaryEmail }}` is empty
    - `{{ $json.primaryPhone }}` is empty
- **Inputs / outputs:**
  - **Input ←** Enrich with Lusha
  - **True output →** Fallback Provider Enrichment (when missing email OR phone)
  - **False output →** Merge Data (Lusha Complete) (when both present)
- **Version notes:** Node typeVersion 2.
- **Edge cases / failures:**
  - “Empty” check behavior depends on whether values are `null`, `undefined`, or `''`. If Lusha returns `null`, it is typically treated as empty, but verify in your n8n version.
  - Waterfall logic currently triggers fallback even when only one field is missing (intended), but you may want stricter logic (e.g., fallback only if both missing).

---

### 2.3 Waterfall Fallback Enrichment
**Overview:** When Lusha is incomplete, calls a secondary enrichment provider via HTTP request.

**Nodes involved:**
- Fallback Provider Enrichment

#### Node: Fallback Provider Enrichment
- **Type / role:** `HTTP Request` — calls an external enrichment API.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `https://api.fallback-provider.com/enrich` (placeholder)
  - **Send headers:** enabled
  - **Send body:** enabled
  - **Body parameter:** `email = {{ $('Validate Email').first().json.email }}`
  - **Headers:**
    - `Authorization: Bearer YOUR_TOKEN_HERE` (placeholder)
    - `Content-Type: application/json`
- **Key expressions / variables:**
  - Uses `$('Validate Email').first().json.email` to reference the validated email regardless of current node input.
- **Inputs / outputs:**
  - **Input ←** Missing Email or Phone? (true branch)
  - **Output →** Merge All Data (Fallback)
- **Version notes:** Node typeVersion 4.
- **Edge cases / failures:**
  - As configured, the token is hardcoded; should be replaced with an n8n credential or environment variable to avoid secrets in workflow.
  - If the provider expects a raw JSON body rather than key/value “bodyParameters”, you may need to switch to “JSON” body mode depending on provider requirements.
  - Non-2xx responses, timeouts, invalid JSON responses can fail the node.
  - Fallback response schema is assumed by merge code (`fb.firstName`, `fb.phone`, `fb.domain`, etc.); mismatches will cause missing data.

---

### 2.4 Data Merge (Clean Record Assembly)
**Overview:** Normalizes and merges data from the form + Lusha (+ fallback if called) into one consistent contact record, preferring form values for identity fields and Lusha for enrichment.

**Nodes involved:**
- Merge All Data (Fallback)
- Merge Data (Lusha Complete)

#### Node: Merge All Data (Fallback)
- **Type / role:** `Code` — merges three sources: validated form, Lusha, and fallback response.
- **Configuration (interpreted):**
  - Reads:
    - `form = $('Validate Email').first().json`
    - `lusha = $('Enrich with Lusha').first().json`
    - `fb = $json` (fallback HTTP response)
  - Produces normalized output fields:
    - `email` (always form email)
    - `firstName`, `lastName`, `company` (prefers form, then Lusha, then fallback)
    - `jobTitle`, `seniority`, `phone`, `verifiedEmail`, `companyDomain`, `companySize`, `industry`
    - `revenueRange`, `linkedinUrl` (Lusha-only in this mapping)
    - `enrichmentSource`: `lusha+fallback` if `lusha.primaryEmail` exists, else `fallback`
    - `source` from form
    - `enrichedAt` ISO timestamp
- **Inputs / outputs:**
  - **Input ←** Fallback Provider Enrichment
  - **Output →** Upsert HubSpot Contact
- **Version notes:** Node typeVersion 2.
- **Edge cases / failures:**
  - If Lusha failed earlier but the workflow continued (not currently possible without custom error handling), `$('Enrich with Lusha').first()` would throw.
  - Assumes fallback returns top-level JSON fields; if the HTTP node returns `{ body: ... }` depending on settings, you may need to reference `fb.body`.
  - `enrichmentSource` logic checks only `lusha.primaryEmail`, not whether fallback actually supplied missing fields.

#### Node: Merge Data (Lusha Complete)
- **Type / role:** `Code` — merges validated form + Lusha only (no fallback).
- **Configuration (interpreted):**
  - Reads:
    - `form = $('Validate Email').first().json`
    - `lusha = $json` (Lusha output)
  - Outputs same normalized schema as the fallback merge, with:
    - `enrichmentSource: 'lusha'`
- **Inputs / outputs:**
  - **Input ←** Missing Email or Phone? (false branch)
  - **Output →** Upsert HubSpot Contact
- **Version notes:** Node typeVersion 2.
- **Edge cases / failures:**
  - If Lusha returns unexpected field names or nested structures, fields like `companyDomain`, `linkedinProfile` may be empty.
  - `verifiedEmail` is set to `lusha.primaryEmail`; if you want the validated form email as fallback, you’d need to add it explicitly.

---

### 2.5 CRM Sync + Slack + Webhook Response
**Overview:** Writes the enriched contact to HubSpot (upsert), alerts an SDR in Slack with the key details, then returns the merged record to the original webhook caller.

**Nodes involved:**
- Upsert HubSpot Contact
- SDR Slack Alert
- Return Enriched Lead

#### Node: Upsert HubSpot Contact
- **Type / role:** `HubSpot` — creates or updates a contact by email.
- **Configuration (interpreted):**
  - **Resource:** Contact
  - **Operation:** Upsert
  - **Email:** `{{ $json.email }}`
  - **Additional fields mapped:**
    - phone, company, website (= companyDomain), industry, jobtitle, lastName, firstName
- **Credentials:** HubSpot OAuth2 (named “HubSpot OAuth2”).
- **Inputs / outputs:**
  - **Input ←** Merge All Data (Fallback) OR Merge Data (Lusha Complete)
  - **Output →** SDR Slack Alert
  - **Output →** Return Enriched Lead (in parallel)
- **Version notes:** Node typeVersion 2.
- **Edge cases / failures:**
  - OAuth token expiry/permission issues.
  - HubSpot property names: n8n’s HubSpot node maps “jobtitle”, “website”, etc. Ensure your HubSpot portal uses standard properties or adapt mapping.
  - Upsert may fail if email is invalid per HubSpot rules (even if it passed basic `@` check).

#### Node: SDR Slack Alert
- **Type / role:** `Slack` — posts an alert message to a channel.
- **Configuration (interpreted):**
  - **Channel:** `#inbound-leads`
  - **Message text:** formatted message referencing `Upsert HubSpot Contact` output fields:
    - firstName, lastName, jobTitle, company, verifiedEmail, phone (or `N/A`), enrichmentSource
- **Credentials:** Slack OAuth2 (named “Slack OAuth2”).
- **Inputs / outputs:**
  - **Input ←** Upsert HubSpot Contact
  - No outgoing connections.
- **Version notes:** Node typeVersion 2.
- **Edge cases / failures:**
  - Channel not found or app not authorized to post.
  - The expression uses `$('Upsert HubSpot Contact').item.json...`; if HubSpot node output doesn’t include these fields (depends on node behavior), the Slack message may show blanks. You may prefer `$json` if the Slack node receives the merged record (it currently receives HubSpot output, not the merged record).

#### Node: Return Enriched Lead
- **Type / role:** `Respond to Webhook` — returns JSON to the original request.
- **Configuration (interpreted):**
  - **Respond with:** JSON
  - **Body:** `{{ $json }}` (whatever is on input)
  - **HTTP status:** 200
- **Inputs / outputs:**
  - **Input ←** Upsert HubSpot Contact
- **Version notes:** Node typeVersion 1.
- **Edge cases / failures:**
  - Because it’s connected to **HubSpot output**, the response will likely be the HubSpot upsert response, not strictly the merged enrichment record. If you want to return the merged record, connect this node from the merge node(s) instead, or pass merged data through HubSpot using “Keep input data” options (if available in your version).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Form Enrichment with Waterfall Fallback | Sticky Note | Documentation / grouping | — | — | ## Enrich Form Leads with Lusha and Waterfall Fallback Provider; Setup steps incl. install community node and credentials; Link: https://www.npmjs.com/package/@lusha-org/n8n-nodes-lusha |
| ⚡ 1. Receive & Validate | Sticky Note | Documentation / grouping | — | — | Webhook receives the form POST; code node validates email; suggestion: add more validation (e.g., block free domains). |
| 🥇 2. Lusha (Primary Enrichment) | Sticky Note | Documentation / grouping | — | — | Lusha is called first; fallback skipped if email+phone found; Link: https://www.lusha.com/docs/ |
| 🥈 3. Fallback + Merge | Sticky Note | Documentation / grouping | — | — | Fallback HTTP fills gaps; merge paths; replace fallback URL with secondary provider API. |
| 💾 4. CRM + Slack + Response | Sticky Note | Documentation / grouping | — | — | Sync to HubSpot; Slack alert; return JSON response to form. |
| Receive Form Submission | Webhook | Receive form POST; entry point | — | Validate Email | Webhook receives the form POST; code node validates email; suggestion: add more validation (e.g., block free domains). |
| Validate Email | Code | Normalize & validate email; build baseline lead object | Receive Form Submission | Enrich with Lusha | Webhook receives the form POST; code node validates email; suggestion: add more validation (e.g., block free domains). |
| Enrich with Lusha | Lusha (community) | Primary enrichment provider | Validate Email | Missing Email or Phone? | Lusha is called first; fallback skipped if email+phone found; Link: https://www.lusha.com/docs/ |
| Missing Email or Phone? | IF | Waterfall decision gate | Enrich with Lusha | Fallback Provider Enrichment (true); Merge Data (Lusha Complete) (false) | Lusha is called first; fallback skipped if email+phone found; Link: https://www.lusha.com/docs/ |
| Fallback Provider Enrichment | HTTP Request | Secondary enrichment provider call | Missing Email or Phone? (true) | Merge All Data (Fallback) | Fallback HTTP fills gaps; merge paths; replace fallback URL with secondary provider API. |
| Merge All Data (Fallback) | Code | Merge form + Lusha + fallback into normalized record | Fallback Provider Enrichment | Upsert HubSpot Contact | Fallback HTTP fills gaps; merge paths; replace fallback URL with secondary provider API. |
| Merge Data (Lusha Complete) | Code | Merge form + Lusha into normalized record | Missing Email or Phone? (false) | Upsert HubSpot Contact | Fallback HTTP fills gaps; merge paths; replace fallback URL with secondary provider API. |
| Upsert HubSpot Contact | HubSpot | Create/update contact in HubSpot | Merge All Data (Fallback); Merge Data (Lusha Complete) | SDR Slack Alert; Return Enriched Lead | Sync to HubSpot; Slack alert; return JSON response to form. |
| SDR Slack Alert | Slack | Notify SDR channel with lead summary | Upsert HubSpot Contact | — | Sync to HubSpot; Slack alert; return JSON response to form. |
| Return Enriched Lead | Respond to Webhook | Return JSON HTTP response | Upsert HubSpot Contact | — | Sync to HubSpot; Slack alert; return JSON response to form. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Form Enrichment + Waterfall Fallback**

2. **Add the Webhook trigger**
   - Add node: **Webhook** → rename to **Receive Form Submission**
   - Set:
     - **HTTP Method:** POST
     - **Path:** `form-enrichment-waterfall`
     - **Response Mode:** *Response Node*
   - Save once to generate a test URL.

3. **Add input validation / normalization**
   - Add node: **Code** → rename to **Validate Email**
   - Paste logic (adapt if needed):
     - Read body from `$json.body || $json`
     - Normalize `email` (trim, lowercase)
     - Throw error if invalid
     - Return `{ email, firstName, lastName, company, source }`
   - Connect: **Receive Form Submission → Validate Email**

4. **Install and configure the Lusha community node**
   - Install community node package on your n8n instance:  
     https://www.npmjs.com/package/@lusha-org/n8n-nodes-lusha
   - Restart n8n if required so the node appears.

5. **Add Lusha enrichment**
   - Add node: **Lusha** (community) → rename to **Enrich with Lusha**
   - Configure:
     - **Operation:** Enrich Single
     - **Search By:** Email
     - **Email:** `{{ $json.email }}`
   - Create/select **Lusha API credentials** (API key as required by the node).
   - Connect: **Validate Email → Enrich with Lusha**

6. **Add the waterfall decision**
   - Add node: **IF** → rename to **Missing Email or Phone?**
   - Condition group:
     - **Combinator:** OR
     - Condition 1: String → **Empty** → left value `{{ $json.primaryEmail }}`
     - Condition 2: String → **Empty** → left value `{{ $json.primaryPhone }}`
   - Connect: **Enrich with Lusha → Missing Email or Phone?**

7. **Add fallback enrichment provider (HTTP)**
   - Add node: **HTTP Request** → rename to **Fallback Provider Enrichment**
   - Configure:
     - **Method:** POST
     - **URL:** your provider endpoint (replace placeholder)
     - **Send Headers:** true
     - **Send Body:** true
     - **Body parameter:** `email = {{ $('Validate Email').first().json.email }}`
     - **Headers:** set provider authorization (prefer credentials/env vars over hardcoding)
   - Connect: **Missing Email or Phone? (true) → Fallback Provider Enrichment**

8. **Add merge code for fallback path**
   - Add node: **Code** → rename to **Merge All Data (Fallback)**
   - Implement merge logic:
     - Load `form` from **Validate Email**
     - Load `lusha` from **Enrich with Lusha**
     - Use `$json` as fallback response
     - Output normalized fields (email, phone, verifiedEmail, companyDomain, etc.)
   - Connect: **Fallback Provider Enrichment → Merge All Data (Fallback)**

9. **Add merge code for Lusha-complete path**
   - Add node: **Code** → rename to **Merge Data (Lusha Complete)**
   - Implement merge logic:
     - Load `form` from **Validate Email**
     - Use `$json` as Lusha data
     - Output the same normalized schema
   - Connect: **Missing Email or Phone? (false) → Merge Data (Lusha Complete)**

10. **Add HubSpot upsert**
    - Add node: **HubSpot** → rename to **Upsert HubSpot Contact**
    - Configure:
      - **Resource:** Contact
      - **Operation:** Upsert
      - **Email:** `{{ $json.email }}`
      - Map additional fields (as needed):
        - phone, company, website (= companyDomain), industry, jobtitle, firstName, lastName
    - Create/select **HubSpot OAuth2 credentials** (connected app / OAuth as required by n8n).
    - Connect:
      - **Merge All Data (Fallback) → Upsert HubSpot Contact**
      - **Merge Data (Lusha Complete) → Upsert HubSpot Contact**

11. **Add Slack alert**
    - Add node: **Slack** → rename to **SDR Slack Alert**
    - Configure:
      - **Channel:** `#inbound-leads` (or your channel)
      - **Text:** include lead summary fields (ensure expressions match the incoming JSON)
    - Create/select **Slack OAuth2 credentials** with permission to post in the channel.
    - Connect: **Upsert HubSpot Contact → SDR Slack Alert**

12. **Add webhook response**
    - Add node: **Respond to Webhook** → rename to **Return Enriched Lead**
    - Configure:
      - **Respond With:** JSON
      - **Response Body:** `{{ $json }}`
      - **Response Code:** 200
    - Connect: **Upsert HubSpot Contact → Return Enriched Lead**
    - (Optional but often recommended) If you want to return the **merged enrichment record** rather than HubSpot’s response, connect **Return Enriched Lead** from the merge node(s) or ensure HubSpot node passes through merged data.

13. **Test end-to-end**
    - Execute workflow in test mode and POST a sample payload to the webhook test URL:
      - include at least `email`, optionally `firstName`, `lastName`, `company`, `source`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install the Lusha community node package before the workflow can run. | https://www.npmjs.com/package/@lusha-org/n8n-nodes-lusha |
| Lusha API documentation reference. | https://www.lusha.com/docs/ |
| Fallback HTTP provider URL and token are placeholders and must be replaced with your secondary provider’s endpoint and secure auth method. | Applies to “Fallback Provider Enrichment” configuration |
| Consider enhancing validation (e.g., block free email domains, required names/company) in the Validate Email code node. | Applies to “Validate Email” node |