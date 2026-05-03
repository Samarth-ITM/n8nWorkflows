Auto-sync and qualify HubSpot CRM contacts with Lusha bulk enrichment

https://n8nworkflows.xyz/workflows/auto-sync-and-qualify-hubspot-crm-contacts-with-lusha-bulk-enrichment-13350


# Auto-sync and qualify HubSpot CRM contacts with Lusha bulk enrichment

## 1. Workflow Overview

**Title:** Auto-sync and qualify HubSpot CRM contacts with Lusha bulk enrichment  
**Workflow name (internal):** Automated CRM Signal Sync & Workflow Activation with Lusha

**Purpose:**  
This workflow runs on a schedule to pull HubSpot contacts that need a data refresh, enrich them in bulk via Lusha (community node), applies ICP qualification scoring, updates HubSpot contact properties with enriched fields, and sends a Slack alert for “fast-track” qualified leads.

**Target use cases:**
- RevOps / Sales Ops data hygiene (refresh missing phone, job title, company data)
- Bulk enrichment for efficiency (single bulk API call vs per-contact calls)
- Automated lead qualification + routing signals to sales teams (Slack fast-track channel)

### 1.1 Input Reception / Orchestration
Scheduled trigger starts the run every 6 hours.

### 1.2 CRM Extraction (HubSpot)
Pulls up to 200 HubSpot contacts (as configured) to enrich.

### 1.3 Bulk Preparation + Lusha Enrichment
Transforms HubSpot contacts into a single Lusha Bulk Enrichment payload (JSON) and calls Lusha Bulk Enrich.

### 1.4 Qualification, CRM Update, and Alerting
Scores each enriched result against ICP criteria, updates HubSpot contact fields, and posts a Slack alert if “fast-track”.

---

## 2. Block-by-Block Analysis

### Block 0 — Documentation / Operator Notes (Sticky Notes)
**Overview:** Provides human guidance on what the workflow does, intended audience, and setup pointers (including a link to Lusha docs).  
**Nodes involved:**  
- 📋 CRM Signal Sync & Workflow Activation  
- 📅 1. Scheduled CRM Pull  
- 🔄 2. Bulk Enrich with Lusha  
- 📤 3. Qualify, Update & Route  

#### Node details
1) **📋 CRM Signal Sync & Workflow Activation**  
- **Type / role:** Sticky Note; top-level description and setup checklist.  
- **Key content:** install Lusha node, configure CRM + Lusha credentials, customize ICP criteria and routing tiers.  
- **Failure modes:** None (non-executing).

2) **📅 1. Scheduled CRM Pull**  
- **Type / role:** Sticky Note; explains schedule + HubSpot pull.  
- **Failure modes:** None.

3) **🔄 2. Bulk Enrich with Lusha**  
- **Type / role:** Sticky Note; explains bulk payload formatting + Lusha call; includes link: https://www.lusha.com/docs/  
- **Failure modes:** None.

4) **📤 3. Qualify, Update & Route**  
- **Type / role:** Sticky Note; explains scoring, updating CRM, and Slack alert logic.  
- **Failure modes:** None.

---

### Block 1 — Scheduled CRM Pull
**Overview:** Runs every 6 hours and pulls a batch of HubSpot contacts to refresh/enrich.  
**Nodes involved:**  
- Scheduled Sync (Every 6 Hours)  
- Get Contacts Needing Refresh  

#### Node details
1) **Scheduled Sync (Every 6 Hours)**  
- **Type / role:** Schedule Trigger; workflow entry point.  
- **Configuration:** Runs every **6 hours** (interval rule).  
- **Outputs:** Sends one execution into HubSpot fetch.  
- **Edge cases / failures:**  
  - n8n instance downtime can skip runs unless you implement catch-up logic externally.
  - Timezone considerations depend on n8n instance settings (schedule is interval-based here, not a specific local time).

2) **Get Contacts Needing Refresh**  
- **Type / role:** HubSpot node; retrieves contacts to enrich.  
- **Operation:** `contact.getAll` (not returnAll).  
- **Key config choices:**
  - **Limit:** 200 contacts per run.
  - **Filters:** `formSubmissionMode: none` (as configured; note this does not by itself guarantee “needing refresh” logic—see edge cases below).  
- **Credentials:** HubSpot OAuth2 (`HubSpot OAuth2`).  
- **Inputs:** Trigger output.  
- **Outputs:** List of HubSpot contact records (each item includes `json.properties.*`, `json.id`, etc.).  
- **Edge cases / failures:**
  - OAuth token expiration / revoked access.
  - Rate limits (HubSpot API throttling) on frequent schedules.
  - The current configuration does **not** explicitly filter “missing fields” (email/phone/etc.). If you truly want “needing refresh,” add filter groups (e.g., missing phone OR missing jobtitle OR last_enriched_at older than X).
  - Contacts without `properties.email` will still be returned; later nodes filter these out for Lusha payload.

---

### Block 2 — Bulk Enrichment with Lusha
**Overview:** Converts the HubSpot contact list into a single Lusha Bulk Enrichment JSON payload and submits one bulk enrichment request.  
**Nodes involved:**  
- Format for Bulk Enrichment  
- Enrich All Contacts in Bulk  

#### Node details
1) **Format for Bulk Enrichment**  
- **Type / role:** Code node (JavaScript); builds one JSON string payload for bulk enrichment.  
- **Key logic / variables:**
  - Reads all incoming items: `const items = $input.all();`
  - Filters out contacts with no email: `.filter(item => item.json.properties?.email)`
  - Assigns sequential string `contactId` starting at `"1"` (local counter, not HubSpot ID).
  - Builds: `{ contacts: [{contactId, email}, ...], metadata: {} }`
  - Outputs a single item: `{ contactsPayload: JSON.stringify(payload) }`
- **Inputs:** From “Get Contacts Needing Refresh”.  
- **Outputs:** One item containing `contactsPayload` (stringified JSON).  
- **Edge cases / failures:**
  - If **no contacts have email**, payload will contain `contacts: []`; Lusha may reject empty requests (depends on API behavior).
  - Duplicated emails in HubSpot: later matching uses email as a key; duplicates will overwrite in lookup (see qualification block).
  - If HubSpot returns malformed contact objects (missing `properties`), filter prevents most issues.

2) **Enrich All Contacts in Bulk**  
- **Type / role:** Lusha community node (`@lusha-org/n8n-nodes-lusha.lusha`); submits bulk enrichment.  
- **Operation:** `enrichBulk`  
- **Key config choices:**
  - `bulkType: json`
  - `contactsPayloadJson: ={{ $json.contactsPayload }}`
- **Credentials:** Lusha API (`Lusha API`).  
- **Inputs:** Single payload item from previous Code node.  
- **Outputs:** Bulk enrichment results (one or multiple items depending on node implementation; this workflow assumes items emitted represent enriched contacts).  
- **Version-specific requirements:**
  - Requires installing the **Lusha community node** in n8n (per sticky note).
- **Edge cases / failures:**
  - Authentication failure (invalid API key/token).
  - Payload schema mismatch (Lusha API expects a specific structure; ensure fields match Lusha docs).
  - API limits (bulk size limits, daily quotas) and timeouts for large lists.
  - Partial enrich results: some contacts may return with missing fields (no phone, no seniority, etc.).

---

### Block 3 — Qualification, Update & Route
**Overview:** Matches Lusha results back to HubSpot contacts (by email), computes an ICP score, updates HubSpot fields, then alerts Slack if the lead is “fast-track.”  
**Nodes involved:**  
- Qualify & Route Each  
- Update CRM with Enriched Data  
- Fast-Track Qualified?  
- Fast-Track Alert  

#### Node details
1) **Qualify & Route Each**  
- **Type / role:** Code node (JavaScript); scoring + routing decision and normalized output for downstream updates/alerts.  
- **Key logic / variables:**
  - Reads all enrichment results: `const items = $input.all();`
  - Reads original CRM contacts directly by node name: `$('Get Contacts Needing Refresh').all();`
  - Creates lookup: `crmByEmail[email] = c.json;` using lowercased email.
  - Defines ICP criteria:
    - `targetSeniority`: `['vp','director','c-suite','founder','head']`
    - `minCompanySize`: `50`
    - `targetIndustries`: `['technology','software','saas','fintech']`
  - Score contributions:
    - Seniority match: +30
    - Company size max (array second element) ≥ 50: +25
    - Industry match: +20
    - Has phone: +10
    - Has email: +10
  - Routing thresholds:
    - `>= 70`: `fast-track`
    - `>= 40`: `standard-outreach`
    - else: `nurture`
  - Output per contact includes:
    - `contactId: crm.id || ''` (HubSpot contact ID if matched)
    - enriched fields (`jobTitle`, `seniority`, `phone`, `company`, `industry`, `companySize`)
    - `qualificationScore`, `signals[]`, `targetWorkflow`, `syncedAt`
- **Inputs:** Lusha node output.  
- **Outputs:** One item per enriched contact result (intended).  
- **Edge cases / failures:**
  - **Email matching risk:** Matching uses `lusha.primaryEmail`; if Lusha returns a different email than HubSpot’s `properties.email`, you may fail to find `crm.id`, resulting in `contactId=''`.
  - **Duplicate emails in HubSpot:** `crmByEmail` map keeps only the last seen contact for that email.
  - **Company size parsing:** assumes `lusha.companySize` can be an array `[min,max]`. If it’s a string/object, `empMax` becomes `0`, lowering score.
  - **Node-name dependency:** `$('Get Contacts Needing Refresh')` hard-codes the node name; renaming that node breaks lookup.
  - If Lusha returns one “bulk response object” rather than one per contact, this code may not iterate as intended (depends on community node’s output structure).

2) **Update CRM with Enriched Data**  
- **Type / role:** HubSpot node; updates contact properties with enriched fields.  
- **Operation:** `contact.update`  
- **Key expressions:**
  - `contactId: ={{ $json.contactId }}`
  - Updates:
    - `phone = {{ $json.phone }}`
    - `company = {{ $json.company }}`
    - `industry = {{ $json.industry }}`
    - `jobtitle = {{ $json.jobTitle }}`
    - `numberofemployees = {{ $json.companySize }}`
- **Credentials:** HubSpot OAuth2 (`HubSpot OAuth2`).  
- **Inputs:** Items from “Qualify & Route Each”.  
- **Outputs:** HubSpot update result per item.  
- **Edge cases / failures:**
  - If `contactId` is empty/invalid, HubSpot update will fail (likely 404 / validation error).
  - Property names must exist in HubSpot:
    - `jobtitle` and `numberofemployees` are common but confirm your portal’s schema.
    - `industry` and `company` may map to default properties, but verify exact internal names.
  - Rate limits when updating many contacts per run.
  - Overwriting with blanks: if Lusha didn’t return a value, you may set HubSpot fields to empty. Consider conditional updates (only set if non-empty).

3) **Fast-Track Qualified?**  
- **Type / role:** IF node; routes items based on `targetWorkflow`.  
- **Condition:** `{{ $json.targetWorkflow }} equals "fast-track"`  
- **Inputs:** From “Update CRM with Enriched Data”.  
- **Outputs:**  
  - **True branch:** to Slack alert  
  - **False branch:** ends (no further action)  
- **Edge cases / failures:**
  - If prior node output doesn’t carry `targetWorkflow` forward (some nodes replace output data), the condition may fail. If HubSpot node outputs only HubSpot response, you might lose the computed fields unless n8n is set to “Keep input data” (depends on node behavior/version). To be safe, consider enabling “Include Input Data” / using Merge node pattern.

4) **Fast-Track Alert**  
- **Type / role:** Slack node; posts a formatted message to a channel for fast-track leads.  
- **Channel:** `#fast-track-leads`  
- **Message text:** Uses expressions including:
  - Name, title, company
  - `qualificationScore`
  - `signals.join(', ')`
  - email and phone fallback: `{{ $json.phone || 'N/A' }}`
- **Credentials:** Slack OAuth2 (`Slack OAuth2`).  
- **Inputs:** True branch from IF node.  
- **Edge cases / failures:**
  - Slack auth/token issues or missing scopes (chat:write).
  - Channel not found / bot not invited to channel.
  - If `signals` is not an array (lost/overwritten upstream), `join` can error.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 CRM Signal Sync & Workflow Activation | Sticky Note | High-level description + setup checklist | — | — | ## Automated CRM Signal Sync with Lusha; Who it’s for; What it does; How it works; Setup steps |
| 📅 1. Scheduled CRM Pull | Sticky Note | Explains schedule + HubSpot pull | — | — | Runs every 6 hours to pull contacts… Adjust interval and limit. |
| 🔄 2. Bulk Enrich with Lusha | Sticky Note | Explains bulk formatting + Lusha bulk API | — | — | Bulk payload + Lusha Bulk API. Link: https://www.lusha.com/docs/ |
| 📤 3. Qualify, Update & Route | Sticky Note | Explains ICP scoring, update, Slack fast-track | — | — | Customize ICP weights/thresholds in Code node. |
| Scheduled Sync (Every 6 Hours) | Schedule Trigger | Starts workflow every 6 hours | — | Get Contacts Needing Refresh | Runs every 6 hours to pull contacts… Adjust interval and limit. |
| Get Contacts Needing Refresh | HubSpot | Fetch contacts batch from HubSpot | Scheduled Sync (Every 6 Hours) | Format for Bulk Enrichment | Runs every 6 hours to pull contacts… Adjust interval and limit. |
| Format for Bulk Enrichment | Code | Build single JSON payload for Lusha bulk enrich | Get Contacts Needing Refresh | Enrich All Contacts in Bulk | All contacts formatted into a single payload… Link: https://www.lusha.com/docs/ |
| Enrich All Contacts in Bulk | Lusha (community) | Bulk enrichment API call | Format for Bulk Enrichment | Qualify & Route Each | All contacts formatted into a single payload… Link: https://www.lusha.com/docs/ |
| Qualify & Route Each | Code | ICP scoring + routing decision per contact | Enrich All Contacts in Bulk | Update CRM with Enriched Data | Each enriched contact is scored… Customize ICP weights/thresholds in Code node. |
| Update CRM with Enriched Data | HubSpot | Update HubSpot contact properties | Qualify & Route Each | Fast-Track Qualified? | Each enriched contact is scored… Customize ICP weights/thresholds in Code node. |
| Fast-Track Qualified? | IF | Branch: fast-track vs other | Update CRM with Enriched Data | (true) Fast-Track Alert; (false) — | Each enriched contact is scored… Customize ICP weights/thresholds in Code node. |
| Fast-Track Alert | Slack | Post fast-track lead alert to Slack channel | Fast-Track Qualified? (true) | — | Each enriched contact is scored… Customize ICP weights/thresholds in Code node. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: **Automated CRM Signal Sync & Workflow Activation with Lusha** (or your preferred name).

2) **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Configure: **Interval** → every **6 hours**
   - Name: **Scheduled Sync (Every 6 Hours)**

3) **Add HubSpot: Get Contacts**
   - Node: **HubSpot**
   - Credentials: create/select **HubSpot OAuth2** (connect your HubSpot portal)
   - Resource: **Contact**
   - Operation: **Get All**
   - Return All: **false**
   - Limit: **200**
   - (Optional) adjust filters to truly represent “needing refresh” (e.g., missing phone/jobtitle or last_enriched_at older than X)
   - Name: **Get Contacts Needing Refresh**
   - Connect: **Scheduled Sync (Every 6 Hours)** → **Get Contacts Needing Refresh**

4) **Add Code node: format bulk payload**
   - Node: **Code**
   - Name: **Format for Bulk Enrichment**
   - Paste logic (adapt as needed):  
     - Build `contacts` array from HubSpot items where `properties.email` exists  
     - Output one item with `contactsPayload` as a JSON string
   - Connect: **Get Contacts Needing Refresh** → **Format for Bulk Enrichment**

5) **Install and add Lusha community node**
   - In n8n, install community package: `@lusha-org/n8n-nodes-lusha` (method depends on your n8n hosting).
   - Add node: **Lusha**
   - Credentials: create/select **Lusha API** (API key/token from Lusha)
   - Operation: **enrichBulk**
   - Set bulk type to **JSON**
   - Map payload field:
     - `contactsPayloadJson` = `{{ $json.contactsPayload }}`
   - Name: **Enrich All Contacts in Bulk**
   - Connect: **Format for Bulk Enrichment** → **Enrich All Contacts in Bulk**

6) **Add Code node: qualification + routing**
   - Node: **Code**
   - Name: **Qualify & Route Each**
   - Implement:
     - Read Lusha results from `$input.all()`
     - Read original HubSpot contacts via `$('Get Contacts Needing Refresh').all()`
     - Match by lowercased email
     - Compute `qualificationScore`, `signals`, and `targetWorkflow` using your ICP thresholds
     - Output one item per contact with `contactId` (HubSpot id) + enriched fields
   - Connect: **Enrich All Contacts in Bulk** → **Qualify & Route Each**

7) **Add HubSpot: Update Contact**
   - Node: **HubSpot**
   - Credentials: **HubSpot OAuth2**
   - Resource: **Contact**
   - Operation: **Update**
   - Contact ID: `{{ $json.contactId }}`
   - Map update fields (align to your HubSpot property internal names):
     - phone → `{{ $json.phone }}`
     - company → `{{ $json.company }}`
     - industry → `{{ $json.industry }}`
     - jobtitle → `{{ $json.jobTitle }}`
     - numberofemployees → `{{ $json.companySize }}`
   - Name: **Update CRM with Enriched Data**
   - Connect: **Qualify & Route Each** → **Update CRM with Enriched Data**

8) **Add IF node: fast-track check**
   - Node: **IF**
   - Name: **Fast-Track Qualified?**
   - Condition (String):  
     - Left: `{{ $json.targetWorkflow }}`
     - Operation: **equals**
     - Right: `fast-track`
   - Connect: **Update CRM with Enriched Data** → **Fast-Track Qualified?**
   - Note: if HubSpot update output does not include `targetWorkflow`, enable keeping input data (or restructure with a Merge node) so the IF can still see `targetWorkflow`.

9) **Add Slack node: post alert**
   - Node: **Slack**
   - Credentials: **Slack OAuth2** (ensure `chat:write` scope; invite bot to channel)
   - Operation: **Post Message** (or equivalent in your Slack node version)
   - Channel: `#fast-track-leads`
   - Text: include key fields (name, company, score, signals, email, phone)
   - Name: **Fast-Track Alert**
   - Connect: **Fast-Track Qualified? (true)** → **Fast-Track Alert**

10) **(Optional) Add sticky notes**
   - Add sticky notes matching the provided descriptions for operator context and handoff.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install the Lusha community node before using “Enrich All Contacts in Bulk”. | Mentioned in workflow notes |
| Lusha API documentation | https://www.lusha.com/docs/ |
| Customize ICP criteria (seniority, company size, industry) and scoring thresholds in the qualification Code node. | “Qualify & Route Each” logic |
| Ensure HubSpot property internal names match (jobtitle, numberofemployees, industry, company, phone). | HubSpot update mapping |
| Slack bot must have permission to post and be present in `#fast-track-leads`. | Slack node configuration |

**Disclaimer (as provided):**  
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.