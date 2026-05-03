Score and enrich inbound leads with Lusha, Slack, and HubSpot

https://n8nworkflows.xyz/workflows/score-and-enrich-inbound-leads-with-lusha--slack--and-hubspot-13348


# Score and enrich inbound leads with Lusha, Slack, and HubSpot

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Score and enrich inbound leads with Lusha, Slack, and HubSpot

**Purpose:**  
This workflow receives inbound leads via a webhook, enriches them using Lusha (contact + company data), computes a lead score/tier (hot/warm/cold), then routes the lead to Slack notifications and HubSpot upserts based on the tier. It also returns a webhook response with the computed score and tier.

**Target use cases:**
- Growth / RevOps teams prioritizing inbound leads automatically
- Routing “hot” leads to SDRs immediately while still capturing all leads in HubSpot
- Adding lightweight ICP scoring based on seniority, company size, industry, and data completeness

### 1.1 Input Reception & Validation
Receives a lead payload (expects an email) and validates/normalizes it before enrichment.

### 1.2 Enrichment (Lusha) & Scoring (Code)
Calls Lusha to enrich the lead and calculates a numeric score plus a tier (hot/warm/cold).

### 1.3 Tier Routing + Notifications + CRM Upsert + Webhook Response
Branches based on tier:
- Hot: Slack alert + HubSpot upsert
- Warm: HubSpot upsert + Slack notification
- Cold: HubSpot upsert only  
Separately, an immediate webhook response returns `tier` and `score`.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Validation
**Overview:**  
Accepts inbound lead submissions through a POST webhook, then validates and normalizes the email field. Invalid email stops the workflow with an error.

**Nodes Involved:**
- New Lead Webhook
- Validate Email

#### Node: New Lead Webhook
- **Type / Role:** `Webhook` (n8n-nodes-base.webhook) — entry point that receives inbound lead data.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `lead-scoring`
  - Exposes an endpoint like: `https://<n8n-host>/webhook/lead-scoring` (or `/webhook-test/lead-scoring` in test mode).
- **Inputs / Outputs:**
  - **Input:** External HTTP request body (typically JSON).
  - **Output:** Passes the webhook payload to **Validate Email**.
- **Potential failure / edge cases:**
  - If caller sends non-JSON or unexpected shape, downstream code may not find `body.email`.
  - If webhook is called without the required field, **Validate Email** throws.
  - Consider authentication/secret validation if used publicly (not implemented here).

#### Node: Validate Email
- **Type / Role:** `Code` (n8n-nodes-base.code) — validates and normalizes the email address.
- **Configuration choices (interpreted):**
  - Reads from `$json.body` if present, otherwise uses `$json` as the body.
  - Extracts `email`, trims, lowercases.
  - Basic validity check: must contain `@` and `.`.
  - Throws an error if invalid/missing.
  - Returns a normalized JSON object: `{ ...body, email }`.
- **Key expressions / variables:**
  - `const body = $json.body || $json;`
  - `const email = (body.email || '').trim().toLowerCase();`
- **Inputs / Outputs:**
  - **Input:** Webhook payload
  - **Output:** Cleaned payload to **Enrich with Lusha**
- **Potential failure / edge cases:**
  - Very basic validation (e.g., `a@b.c` passes; complex valid emails might fail if missing a dot in expected place).
  - If inbound payload uses a different field name (e.g., `Email`), enrichment will fail unless mapped.

---

### Block 2 — Enrichment (Lusha) & Scoring
**Overview:**  
Enriches the validated lead using Lusha and calculates a score and tier using weighted rules (seniority, company size, industry match, phone/email availability).

**Nodes Involved:**
- Enrich with Lusha
- Calculate Lead Score

#### Node: Enrich with Lusha
- **Type / Role:** `Lusha` community node (`@lusha-org/n8n-nodes-lusha.lusha`) — performs contact enrichment.
- **Configuration choices (interpreted):**
  - **Search by:** email
  - **Email input:** `={{ $json.email }}`
  - Additional options left default.
- **Inputs / Outputs:**
  - **Input:** Normalized lead (must include `email`)
  - **Output:** Lusha enrichment result into **Calculate Lead Score**
- **Credentials / requirements:**
  - Requires Lusha API credentials configured in n8n for this node.
  - Community node must be installed on the n8n instance (`@lusha-org/n8n-nodes-lusha`).
- **Potential failure / edge cases:**
  - Auth errors (invalid API key).
  - Rate limiting by Lusha.
  - Email not found: output may be partial/empty; downstream scoring assumes some fields might be missing (handled mostly safely).

#### Node: Calculate Lead Score
- **Type / Role:** `Code` — computes lead score, tier, and returns a normalized enriched lead record for downstream routing.
- **Configuration choices (interpreted):**
  - Reads Lusha data from current `$json`.
  - Reads original form data from `$('Validate Email').first().json` to prefer form-provided first/last name.
  - **Scoring logic:**
    - **Seniority:** C-suite/founder +30, VP +25, Director +20, Manager +10
    - **Company size:** max employees extracted from `companySize[1]`
      - 1000+ +25, 200+ +20, 50+ +10
    - **Industry match:** +15 if company industry contains one of: `technology, software, saas, fintech, financial services`
    - **Data quality:** direct phone +10; verified email +5
  - **Tiering:**
    - `hot` if score >= 60
    - `warm` if score >= 35
    - else `cold`
  - Produces a single consolidated output object including `leadScore`, `leadTier`, `scoreReasons`, and enriched fields.
- **Key expressions / variables:**
  - `const form = $('Validate Email').first().json;`
  - `const empMax = Array.isArray(lusha.companySize) ? lusha.companySize[1] : 0;`
  - `scoreReasons: reasons` (array)
- **Inputs / Outputs:**
  - **Input:** Lusha enrichment payload
  - **Output:** Sent to:
    - **Is Hot Lead?** (for routing)
    - **Respond to Webhook** (immediate response)
- **Potential failure / edge cases:**
  - If Lusha returns unexpected types (e.g., `companySize` not array), defaults to `0` safely.
  - `scoreReasons.join('\n')` later assumes `scoreReasons` is an array (it is).
  - If Validate Email node returns multiple items (unlikely here), `.first()` only takes the first.

---

### Block 3 — Tier Routing + Slack/HubSpot Actions + Webhook Response
**Overview:**  
Routes scored leads into hot/warm/cold paths. Hot leads alert the SDR Slack channel and upsert to HubSpot. Warm leads upsert to HubSpot and notify Slack. Cold leads only upsert to HubSpot. In parallel, the workflow responds to the webhook caller with score/tier.

**Nodes Involved:**
- Respond to Webhook
- Is Hot Lead?
- Alert SDR Team (Hot Lead)
- CRM Upsert (Hot Lead)
- Is Warm Lead?
- CRM Upsert (Warm — Outreach)
- Notify Warm Lead
- CRM Upsert (Cold — Nurture)

#### Node: Respond to Webhook
- **Type / Role:** `Respond to Webhook` — returns a JSON response to the original HTTP request.
- **Configuration choices (interpreted):**
  - Responds with JSON.
  - Response body is constructed via expression and stringified:
    - `status: 'scored'`
    - `tier` and `score` pulled from `$('Calculate Lead Score').first().json`
- **Key expression:**
  - `={{ JSON.stringify({ status: 'scored', tier: $('Calculate Lead Score').first().json.leadTier, score: $('Calculate Lead Score').first().json.leadScore }) }}`
- **Inputs / Outputs:**
  - **Input:** From **Calculate Lead Score** (runs in parallel with routing).
  - **Output:** HTTP response back to the webhook caller.
- **Potential failure / edge cases:**
  - If scoring node fails, this node never runs.
  - The response is a JSON *string* (because of `JSON.stringify`) while “Respond with: json” is selected—often n8n accepts it, but it can lead to double-encoding depending on node behavior/version. Prefer returning an object directly when possible.

#### Node: Is Hot Lead?
- **Type / Role:** `IF` — checks if `leadTier` equals `hot`.
- **Configuration choices:**
  - Condition: `={{ $json.leadTier }}` equals `hot`
- **Routing:**
  - **True path:** to **Alert SDR Team (Hot Lead)** and **CRM Upsert (Hot Lead)** (both in parallel).
  - **False path:** to **Is Warm Lead?**
- **Potential failure / edge cases:**
  - If `leadTier` missing, condition evaluates false → treated as not hot.

#### Node: Alert SDR Team (Hot Lead)
- **Type / Role:** `Slack` — posts a formatted hot-lead message.
- **Configuration choices (interpreted):**
  - Message includes score, name, title, company, seniority, company size, industry, email/phone/linkedin, and a score breakdown (newline-joined).
  - Uses fallback values like `'N/A'` for phone/linkedin.
- **Key expressions:**
  - `{{ $json.scoreReasons.join('\n') }}`
  - `{{ $json.verifiedEmail || $json.email }}`
- **Inputs / Outputs:**
  - **Input:** Hot-lead enriched record
  - **Output:** none downstream
- **Credentials / requirements:**
  - Slack credential must be configured (OAuth2/bot token depending on n8n Slack node setup).
- **Potential failure / edge cases:**
  - Slack auth errors / missing scopes (e.g., chat:write).
  - Channel not found or blocked posting.
  - Message formatting issues if fields are unexpectedly non-strings (rare here).

#### Node: CRM Upsert (Hot Lead)
- **Type / Role:** `HubSpot` — upserts a contact by email with basic fields.
- **Configuration choices (interpreted):**
  - Uses `email` as identifier.
  - Additional fields set: `firstName`, `lastName`, `industry`.
- **Key expressions:**
  - `email: ={{ $json.email }}`
  - `firstName/lastName/industry` mapped from scored payload
- **Inputs / Outputs:**
  - **Input:** Hot-lead record
  - **Output:** none downstream
- **Credentials / requirements:**
  - HubSpot credential (Private App token or OAuth, depending on node/instance setup).
- **Potential failure / edge cases:**
  - HubSpot API validation: property names must match portal schema (e.g., `industry` must exist as a contact property).
  - Rate limiting.
  - Email missing would break upsert (but validated earlier).

#### Node: Is Warm Lead?
- **Type / Role:** `IF` — checks if `leadTier` equals `warm`.
- **Configuration choices:**
  - Condition: `={{ $json.leadTier }}` equals `warm`
- **Routing:**
  - **True path:** to **CRM Upsert (Warm — Outreach)** and **Notify Warm Lead** (parallel).
  - **False path:** to **CRM Upsert (Cold — Nurture)**
- **Potential failure / edge cases:**
  - Missing tier defaults to cold path behavior.

#### Node: CRM Upsert (Warm — Outreach)
- **Type / Role:** `HubSpot` — upserts contact for warm leads.
- **Configuration:** Same mappings as hot upsert: email + first/last/industry.
- **Potential failure / edge cases:** Same as other HubSpot nodes.

#### Node: Notify Warm Lead
- **Type / Role:** `Slack` — posts warm-lead notification.
- **Configuration choices (interpreted):**
  - Message includes score, name, title/company, email/phone, and short reason list joined by commas.
- **Key expressions:**
  - `{{ $json.scoreReasons.join(', ') }}`
- **Potential failure / edge cases:** Same as Slack hot alert.

#### Node: CRM Upsert (Cold — Nurture)
- **Type / Role:** `HubSpot` — upserts cold leads (minimal fields).
- **Configuration choices:**
  - Sets email, firstName, lastName (no industry in this node).
- **Potential failure / edge cases:** Same as other HubSpot nodes.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Lead Scoring + Enrichment | Sticky Note | Documentation / grouping | — | — | ## Lead Scoring + Enrichment with Lusha<br><br>**Who it's for:** Growth & RevOps teams who want to prioritize inbound leads automatically<br><br>**What it does:** Combines Lusha enrichment with scoring logic (job seniority, company size, tech stack) to automatically prioritize the most valuable leads.<br><br>### How it works<br>1. Receives new lead via webhook<br>2. Enriches with Lusha for contact + company data<br>3. Applies scoring logic based on seniority, company size, industry<br>4. Routes high-score leads to SDRs immediately<br>5. Low-score leads go to nurture campaigns<br><br>### Scoring Criteria (customize these!)<br>- VP/C-Suite seniority: +30 pts<br>- Director: +20 pts<br>- Manager: +10 pts<br>- Company 200+ employees: +20 pts<br>- Target industry match: +15 pts<br>- Has direct phone: +10 pts<br>- Has verified email: +5 pts |
| ⚡ 1. Receive Lead | Sticky Note | Documentation / grouping | — | — | Webhook receives a new lead with an email address. The lead is immediately sent to Lusha for enrichment.<br><br>**Nodes:** Webhook → Lusha Contact Enrich<br><br>💡 Point your lead capture form, landing page, or marketing automation to this webhook. |
| 🔍 2. Enrich & Score | Sticky Note | Documentation / grouping | — | — | Lusha enriches the contact with seniority, company size, industry, and phone data. A code node calculates a lead score based on ICP fit criteria (customize the weights!).<br><br>**Nodes:** Lusha Enrich → Calculate Lead Score<br><br>📖 [Lusha API docs](https://www.lusha.com/docs/) |
| 📤 3. Route by Score Tier | Sticky Note | Documentation / grouping | — | — | Three-tier routing based on lead score:<br><br>**Hot (60+ pts):** Slack #hot-leads alert + CRM upsert with full data<br>**Warm (35-59 pts):** CRM upsert + Slack #warm-leads notification<br>**Cold (<35 pts):** CRM upsert for long-term nurture<br><br>**Nodes:** Is Hot? → [Slack Alert + CRM Upsert] / Is Warm? → [CRM + Slack] / CRM Nurture<br><br>A webhook response returns the score and tier to the calling system. |
| New Lead Webhook | Webhook | Receive inbound lead payload | — | Validate Email | |
| Validate Email | Code | Validate/normalize email | New Lead Webhook | Enrich with Lusha | |
| Enrich with Lusha | Lusha | Contact/company enrichment by email | Validate Email | Calculate Lead Score | |
| Calculate Lead Score | Code | Compute score, tier, and normalized enriched output | Enrich with Lusha | Is Hot Lead?; Respond to Webhook | |
| Respond to Webhook | Respond to Webhook | Return score/tier to caller | Calculate Lead Score | — | |
| Is Hot Lead? | IF | Branch hot vs non-hot | Calculate Lead Score | (true) Alert SDR Team (Hot Lead), CRM Upsert (Hot Lead); (false) Is Warm Lead? | |
| Alert SDR Team (Hot Lead) | Slack | Post hot-lead alert | Is Hot Lead? (true) | — | |
| CRM Upsert (Hot Lead) | HubSpot | Upsert hot lead to HubSpot | Is Hot Lead? (true) | — | |
| Is Warm Lead? | IF | Branch warm vs cold | Is Hot Lead? (false) | (true) CRM Upsert (Warm — Outreach), Notify Warm Lead; (false) CRM Upsert (Cold — Nurture) | |
| CRM Upsert (Warm — Outreach) | HubSpot | Upsert warm lead to HubSpot | Is Warm Lead? (true) | — | |
| Notify Warm Lead | Slack | Post warm-lead notification | Is Warm Lead? (true) | — | |
| CRM Upsert (Cold — Nurture) | HubSpot | Upsert cold lead to HubSpot | Is Warm Lead? (false) | — | |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes (optional but recommended for clarity)**
   1) Add a Sticky Note named **“📋 Lead Scoring + Enrichment”** and paste the overview/scoring criteria text.  
   2) Add a Sticky Note named **“⚡ 1. Receive Lead”** describing the webhook → enrichment flow.  
   3) Add a Sticky Note named **“🔍 2. Enrich & Score”** including the link to Lusha docs: https://www.lusha.com/docs/  
   4) Add a Sticky Note named **“📤 3. Route by Score Tier”** describing hot/warm/cold routing.

2. **Add the Webhook Trigger**
   - Create node: **Webhook** named **“New Lead Webhook”**
   - Set:
     - **HTTP Method:** POST  
     - **Path:** `lead-scoring`
   - Save the node and copy the test/production URL to your form/marketing system.

3. **Add Email Validation / Normalization**
   - Create node: **Code** named **“Validate Email”**
   - Paste logic:
     - Read `const body = $json.body || $json`
     - Normalize email: `trim().toLowerCase()`
     - Throw error if missing/invalid
     - Return `{ ...body, email }`
   - Connect: **New Lead Webhook → Validate Email**

4. **Install and Configure Lusha Node**
   - Ensure the community node package is installed on your n8n instance: `@lusha-org/n8n-nodes-lusha`
   - Create node: **Lusha** named **“Enrich with Lusha”**
   - Configure:
     - **Search By:** Email
     - **Email:** `={{ $json.email }}`
   - Set Lusha credentials (API key) in n8n credentials and select them in the node.
   - Connect: **Validate Email → Enrich with Lusha**

5. **Add Scoring Code**
   - Create node: **Code** named **“Calculate Lead Score”**
   - Implement:
     - Pull original form data: `const form = $('Validate Email').first().json;`
     - Score seniority, company size, industry, phone/email availability
     - Compute `leadTier` with thresholds (>=60 hot, >=35 warm, else cold)
     - Output a consolidated object with:
       - Identity: firstName, lastName, email
       - Enriched: verifiedEmail, phone, jobTitle, seniority, company, domain, size, industry, linkedinUrl
       - Scoring: leadScore, leadTier, scoreReasons
       - Timestamp: enrichedAt
   - Connect: **Enrich with Lusha → Calculate Lead Score**

6. **Add Webhook Response (parallel)**
   - Create node: **Respond to Webhook** named **“Respond to Webhook”**
   - Set:
     - **Respond With:** JSON
     - **Response Body:** build a response containing status, tier, score (sourced from Calculate Lead Score output)
   - Connect: **Calculate Lead Score → Respond to Webhook**
   - Note: Prefer returning an object directly (avoid double-encoding). If you keep `JSON.stringify`, verify the actual response your caller receives.

7. **Add Hot Lead Branch**
   - Create node: **IF** named **“Is Hot Lead?”**
   - Condition:
     - Left: `={{ $json.leadTier }}`
     - Operation: equals
     - Right: `hot`
   - Connect: **Calculate Lead Score → Is Hot Lead?**

8. **Hot Lead Actions**
   1) Create node: **Slack** named **“Alert SDR Team (Hot Lead)”**
      - Configure Slack credentials (bot/OAuth) and target channel in node settings as appropriate.
      - Set message text using expressions for lead details and `scoreReasons.join('\n')`.
   2) Create node: **HubSpot** named **“CRM Upsert (Hot Lead)”**
      - Configure HubSpot credentials (Private App token or OAuth).
      - Operation: upsert/create contact by **Email**.
      - Map fields: `firstName`, `lastName`, `industry` (ensure these HubSpot properties exist).
   - Connect **Is Hot Lead? (true)** → **Alert SDR Team (Hot Lead)**
   - Connect **Is Hot Lead? (true)** → **CRM Upsert (Hot Lead)**

9. **Warm vs Cold Branch**
   - Create node: **IF** named **“Is Warm Lead?”**
   - Condition:
     - `={{ $json.leadTier }}` equals `warm`
   - Connect **Is Hot Lead? (false)** → **Is Warm Lead?**

10. **Warm Lead Actions**
   1) Create node: **HubSpot** named **“CRM Upsert (Warm — Outreach)”**
      - Same upsert-by-email mapping (firstName, lastName, industry).
   2) Create node: **Slack** named **“Notify Warm Lead”**
      - Post a shorter message, include `scoreReasons.join(', ')`.
   - Connect **Is Warm Lead? (true)** → **CRM Upsert (Warm — Outreach)**
   - Connect **Is Warm Lead? (true)** → **Notify Warm Lead**

11. **Cold Lead Action**
   - Create node: **HubSpot** named **“CRM Upsert (Cold — Nurture)”**
   - Upsert by email; map at least firstName/lastName.
   - Connect **Is Warm Lead? (false)** → **CRM Upsert (Cold — Nurture)**

12. **Test End-to-End**
   - POST to the webhook with JSON like:
     - `{ "email": "person@company.com", "firstName": "A", "lastName": "B" }`
   - Confirm:
     - Webhook response includes `tier` and `score`
     - Slack posts appear for hot/warm
     - HubSpot contact is created/updated

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Lusha API documentation | https://www.lusha.com/docs/ |
| Scoring weights and target industries are meant to be customized | Adjust inside **Calculate Lead Score** code node (seniority keywords, employee thresholds, industries, tier cutoffs). |
| Consider adding webhook authentication (shared secret, signature validation, IP allowlist) | Not present in current workflow; recommended if endpoint is public. |
| HubSpot properties must exist and match expected internal names | Ensure `industry`, `firstname`, `lastname` mappings align with your HubSpot portal schema. |
| Webhook response body is stringified | Verify caller receives proper JSON; consider returning an object instead of `JSON.stringify(...)` depending on n8n behavior/version. |