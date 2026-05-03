Discover SaaS creators from HubSpot with influencers.club, OpenAI and Gmail/SendGrid

https://n8nworkflows.xyz/workflows/discover-saas-creators-from-hubspot-with-influencers-club--openai-and-gmail-sendgrid-13336


# Discover SaaS creators from HubSpot with influencers.club, OpenAI and Gmail/SendGrid

## 1. Workflow Overview

**Purpose:**  
This workflow detects which HubSpot contacts are also social-media creators by enriching their email with **influencers.club**, classifies them by **tier and niche**, generates **AI-personalized outreach**, writes creator attributes back into **HubSpot**, then routes and sends the outreach via **Founder Gmail (high priority)** or **SendGrid (normal/medium)**.

**Primary use cases:**
- Turning SaaS customers (signups / paid users) into potential creator partners
- Enriching CRM records with creator metrics for segmentation and outreach automation
- Scaling personalized outbound while preserving “human” messaging for top-tier creators

### 1.1 Input Reception (HubSpot event → contact details)
Receives a HubSpot event (contact-based), fetches the contact record, and extracts the email (and company name if available).

### 1.2 Creator Enrichment (influencers.club)
Posts the email to influencers.club enrichment endpoint and handles failure/non-creator outcomes.

### 1.3 Filtering + Classification (creator check → tier & niche)
Checks whether the enrichment indicates the person is a creator, then classifies tier (nano/micro/mid/macro) and niche (developer/founder/etc.) based on follower count and bio keywords.

### 1.4 AI Outreach Generation (OpenAI agent with structured JSON output)
Uses an OpenAI chat model + structured output parser to produce a JSON object containing: subject, message, priority, program, and talking points.

### 1.5 CRM Enrichment (write results back to HubSpot)
Updates HubSpot (intended) with creator fields and outreach attributes.

### 1.6 Priority Routing + Email Sending (Gmail vs SendGrid)
Routes “high” priority to Founder Gmail; otherwise sends via SendGrid.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (HubSpot → contact → email/company)
**Overview:**  
Triggers on a HubSpot contact event, retrieves the contact record, and normalizes key fields (email, company name) for downstream enrichment.

**Nodes involved:**
- HubSpot Trigger
- Get Contact by ID
- Extract Email

#### Node: **HubSpot Trigger**
- **Type / role:** `hubspotTrigger` — webhook trigger for HubSpot events.
- **Configuration (interpreted):**
  - Uses HubSpot Developer credentials (developer API).
  - `maxConcurrentRequests: 5` limits concurrency.
  - `eventsUi.eventValues` is effectively empty/unspecified in the exported JSON; in practice you must select which HubSpot events (e.g., contact created/updated) trigger this workflow.
- **Inputs/outputs:**
  - **Output:** Event payload including `contactId` (used by next node).
- **Edge cases / failures:**
  - Misconfigured/no events selected → never triggers.
  - HubSpot webhook delivery/auth issues.
  - Contact event may not include associated company data; later node assumes it exists.
- **Version notes:** Node typeVersion `1`.

#### Node: **Get Contact by ID**
- **Type / role:** `hubspot` (operation: get contact) — fetches full contact details.
- **Configuration (interpreted):**
  - Authentication: **HubSpot App Token**.
  - Contact ID: `={{ $json.contactId }}`
  - Requested properties: `firstname`, `company`, `hs_full_name_or_email` (note: the workflow later reads `properties.email.value`, which is **not explicitly requested** here).
- **Inputs/outputs:**
  - **Input:** Trigger payload with `contactId`.
  - **Output:** HubSpot contact object including `properties`.
- **Edge cases / failures:**
  - If the `email` property is not returned by default, `Extract Email` may fail or produce empty email.
  - Token scopes/permissions can block contact retrieval.
- **Version notes:** typeVersion `2.2`.

#### Node: **Extract Email**
- **Type / role:** `set` — maps HubSpot response into simplified fields.
- **Configuration (interpreted):**
  - Sets:
    - `Email = {{ $json.properties.email.value }}`
    - `Company Name = {{ $json["associated-company"].properties.name.value }}`
- **Inputs/outputs:**
  - **Input:** HubSpot contact JSON.
  - **Output:** JSON containing `Email` and `Company Name` used by enrichment.
- **Edge cases / failures:**
  - If `properties.email.value` is missing → Email becomes null/undefined.
  - If no associated company exists or the field path differs → expression fails or returns undefined (depends on n8n expression handling).
  - HubSpot association data is not guaranteed to be present in “Get contact” responses unless specifically requested via associations or separate API call.
- **Version notes:** typeVersion `3.4`.

---

### Block 2 — Creator Enrichment (influencers.club)
**Overview:**  
Sends the email to influencers.club enrichment API. If the HTTP request errors, workflow continues via the error output; otherwise passes results to creator filtering.

**Nodes involved:**
- Enrich by Email
- Stop and Error1

#### Node: **Enrich by Email**
- **Type / role:** `httpRequest` — calls influencers.club enrichment endpoint.
- **Configuration (interpreted):**
  - POST `https://api-dashboard.influencers.club/public/v1/creators/enrich/email/`
  - Body parameter: `email = {{ $json.Email }}`
  - Authentication: **Generic Credential Type → HTTP Header Auth**
  - Header: `Accept: application/json`
  - **onError:** `continueErrorOutput` (important)  
    This creates two outputs:
    - **Main output (success)**
    - **Error output (failure path)**
- **Inputs/outputs:**
  - **Input:** `Email` from **Extract Email**.
  - **Success output:** Expected to include `result` object (used later).
  - **Error output:** Routed to Stop and Error1.
- **Edge cases / failures:**
  - Missing/invalid API key in header auth credential.
  - Rate limits / 429 responses.
  - API returns “not a creator” as null/empty `result` (as sticky note states this is normal). This is *not* an HTTP error; it will still be on the success branch and must be handled by filtering.
- **Version notes:** typeVersion `4.2`.
- **Sticky note / comment:** Node has a node “notes” field: “⚙️ CONFIGURE: Add your influencers.club API key in credentials”.

#### Node: **Stop and Error1**
- **Type / role:** `stopAndError` — terminates execution with an error.
- **Configuration (interpreted):**
  - Error message: “API enrichment failed - contact is not a creator or API error”
- **Inputs/outputs:**
  - **Input:** Error output from Enrich by Email.
  - **Output:** None (stops workflow).
- **Edge cases / failures:**
  - Message conflates two cases: true HTTP failure vs “not a creator”. In this workflow, “not a creator” is intended to be handled by the filter (success path), so this stop node mostly represents HTTP/transport/auth failures.

---

### Block 3 — Filtering + Classification (creator check → tier/niche)
**Overview:**  
Filters to proceed only if enrichment indicates the contact is a creator, then computes tier and niche from platform data and bio keywords.

**Nodes involved:**
- Filter - Is Creator?
- Classify Tier & Niche

#### Node: **Filter - Is Creator?**
- **Type / role:** `if` — boolean gate.
- **Configuration (interpreted):**
  - Condition: `{{ $json.result?.is_creator }} == true`
- **Inputs/outputs:**
  - **Input:** influencers.club response.
  - **True output:** goes to **Classify Tier & Niche**
  - **False output:** not connected (silently ends for non-creators)
- **Edge cases / failures:**
  - If `result` is missing or null → `result?.is_creator` becomes undefined and condition fails (treated as not a creator).
  - If influencers.club returns a different schema (e.g., `is_creator` not present), everyone is treated as not a creator.
- **Version notes:** typeVersion `2`.

#### Node: **Classify Tier & Niche**
- **Type / role:** `code` — custom JS classification logic.
- **Configuration (interpreted):**
  - Chooses a “primary platform” in order: TikTok → Instagram → YouTube → Twitter (based on available follower/subscriber count).
  - Extracts:
    - `followers`
    - `engagement_rate`
    - `username`
    - `bio` (lowercased)
    - `primary_platform`
  - Determines **tier**:
    - `macro` ≥ 250,000
    - `mid` ≥ 50,000
    - `micro` ≥ 5,000
    - else `nano`
  - Determines **niche** via regex keyword matching in bio:
    - developer/engineer, founder/CEO, product/design, marketing/growth, tech leadership, AI/data, business/entrepreneurship, lifestyle; else general
  - Outputs merged object: original JSON + `classification` object.
- **Inputs/outputs:**
  - **Input:** Only creator items from IF node.
  - **Output:** JSON with `classification` used by AI agent and downstream steps.
- **Edge cases / failures:**
  - If API returns counts in unexpected fields/types → follower count becomes 0; tier defaults to nano.
  - Regex may misclassify (false positives, multilingual bios, emojis, etc.).
  - Only one platform is selected; large following on a lower-priority platform could be ignored.
- **Version notes:** typeVersion `2`.

---

### Block 4 — AI Outreach Generation (OpenAI + structured output)
**Overview:**  
Generates a personalized outreach email in a strict JSON format. The routing priority and recommended program come from the AI output.

**Nodes involved:**
- OpenAI Chat Model1
- Structured Output Parser1
- Influencer Outreach Agent

#### Node: **OpenAI Chat Model1**
- **Type / role:** `lmChatOpenAi` (LangChain) — provides the LLM used by the agent.
- **Configuration (interpreted):**
  - Model: `gpt-4o`
  - Temperature: `0.8` (more creative variability)
- **Inputs/outputs:**
  - Connected to the agent via the **ai_languageModel** connection.
- **Edge cases / failures:**
  - OpenAI credential missing/invalid.
  - Model availability changes or org policy restrictions.
  - Higher temperature can increase schema violations (mitigated by output parser).
- **Version notes:** typeVersion `1.3`.

#### Node: **Structured Output Parser1**
- **Type / role:** Structured JSON schema output parser — enforces a manual JSON Schema.
- **Configuration (interpreted):**
  - Expects output matching schema `InfluencerOutreach`:
    - `outreach.subject` (string)
    - `outreach.message` (string)
    - `outreach.priority` enum: normal|medium|high
    - `outreach.program` enum: community|content_collaboration|strategic_partnership
    - `outreach.talking_points` array of strings
- **Inputs/outputs:**
  - Connected to agent via **ai_outputParser**.
- **Edge cases / failures:**
  - If the model output can’t be parsed/validated, the agent node may error.
- **Version notes:** typeVersion `1.2`.

#### Node: **Influencer Outreach Agent**
- **Type / role:** LangChain Agent — crafts outreach text with strict rules and structured output.
- **Configuration (interpreted):**
  - Input text: `={{ JSON.stringify($json) }}` (sends the entire enriched+classified JSON as a string)
  - System message: extensive instruction set controlling:
    - JSON-only output
    - when follower count can be mentioned
    - tone and structure by tier
    - subject-line constraints
    - output schema to match the structured parser
  - `hasOutputParser: true` (enforces parser)
- **Inputs/outputs:**
  - **Main input:** classified creator JSON from **Classify Tier & Niche**
  - **Main output:** agent output object; later referenced as `$('Influencer Outreach Agent').first().json.output.outreach...`
- **Edge cases / failures:**
  - If upstream JSON is large/noisy, token usage can grow.
  - If `first_name` / `result.first_name` isn’t present in the influencers.club result, instruction says fallback to username or “there”; depends on actual input fields available.
  - Any schema mismatch → parser failure → node error.
- **Version notes:** typeVersion `3.1`.

---

### Block 5 — CRM Enrichment (write creator/outreach data back)
**Overview:**  
Intended to save creator fields and outreach metadata back into HubSpot.

**Nodes involved:**
- Enrich CRM with Creator Data

#### Node: **Enrich CRM with Creator Data**
- **Type / role:** `hubspot` — updates HubSpot contact by email (intended enrichment step).
- **Configuration (interpreted):**
  - Authentication: HubSpot App Token.
  - Email: `={{ $('Extract Email').first().json.Email }}`
  - **Important:** `additionalFields` is empty in the exported JSON, so this node currently does **not** write any properties unless configured in the UI.
- **Inputs/outputs:**
  - **Input:** AI output from **Influencer Outreach Agent** (via main connection).
  - **Output:** HubSpot update response; routed to **Route by Priority**.
- **Edge cases / failures:**
  - If HubSpot contact lookup by email fails (no match / duplicates), update may fail.
  - Missing custom properties in HubSpot (as required by sticky note) will prevent writing or cause errors.
  - As currently configured, it may do nothing—ensure “operation” and “properties to update” are set.
- **Version notes:** typeVersion `2.2`.

---

### Block 6 — Priority Routing + Email Sending
**Overview:**  
Splits the flow based on AI-selected `priority`. High priority uses Gmail (founder). Normal/medium uses SendGrid.

**Nodes involved:**
- Route by Priority
- Send from Founder (High Priority)
- Send an email

#### Node: **Route by Priority**
- **Type / role:** `if` — string comparison routing.
- **Configuration (interpreted):**
  - Condition: `{{ $json.output.outreach.priority }} == "high"`
- **Inputs/outputs:**
  - **Input:** Output from “Enrich CRM with Creator Data” (which passes through the agent output context).
  - **True branch:** Founder Gmail node
  - **False branch:** SendGrid node
- **Edge cases / failures:**
  - If `$json.output.outreach.priority` is missing (e.g., CRM node output overwrote structure), routing may misbehave.  
    In practice, be careful: HubSpot nodes often output their own response shape; you may need to use expressions referencing the agent node directly (as the email nodes do) or “Merge” data.
- **Version notes:** typeVersion `1`.

#### Node: **Send from Founder (High Priority)**
- **Type / role:** `gmail` — sends email from a connected Gmail account.
- **Configuration (interpreted):**
  - To: `={{ $('Extract Email').first().json.Email }}`
  - Subject: `={{ $('Influencer Outreach Agent').first().json.output.outreach.subject }}`
  - Message: `={{ $('Influencer Outreach Agent').first().json.output.outreach.message }}`
- **Inputs/outputs:**
  - **Input:** High-priority branch.
  - **Output:** none connected.
- **Edge cases / failures:**
  - Gmail OAuth token expiration or insufficient scopes.
  - Gmail sending limits / throttling.
- **Version notes:** typeVersion `2.1`.

#### Node: **Send an email** (SendGrid)
- **Type / role:** `sendGrid` — sends scalable outreach via SendGrid.
- **Configuration (interpreted):**
  - To: `={{ $('Classify Tier & Niche').first().json.result.email }}`
    - Note: this uses `result.email` from influencers.club payload, not the HubSpot email field; confirm it matches the intended recipient.
  - Subject/message sourced from **Influencer Outreach Agent** output.
- **Inputs/outputs:**
  - **Input:** Non-high branch.
  - **Output:** none connected.
- **Edge cases / failures:**
  - SendGrid API key invalid or sender not authenticated.
  - From/sender identity requirements (Sender Authentication).
  - Recipient email mismatch if `result.email` differs from HubSpot email.
- **Version notes:** typeVersion `1`.

---

### Block 7 — Documentation / Annotations (Sticky Notes)
**Overview:**  
Sticky notes document intended steps, setup requirements, and external resources. They don’t execute but contain important configuration requirements (HubSpot custom properties, SendGrid setup, etc.).

**Nodes involved:**
- Sticky Note6
- Sticky Note10
- Sticky Note11
- Sticky Note13
- Sticky Note - AI Outreach1
- Sticky Note - CRM Enrichment
- Sticky Note - Priority Routing
- Sticky Note - Email Sending
- Sticky Note (SendGrid Setup)
- Sticky Note1

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note6 | Sticky Note | High-level workflow description & prerequisites |  |  | ## 🎯 SaaS Customer → Tech Influencer Discovery… (setup required list) |
| Sticky Note10 | Sticky Note | Step 1 documentation |  |  | ## 📋 STEP 1: Get Customer Data… |
| HubSpot Trigger | HubSpot Trigger | Entry point from HubSpot events |  | Get Contact by ID | ## 📋 STEP 1: Get Customer Data… |
| Get Contact by ID | HubSpot | Fetch contact record | HubSpot Trigger | Extract Email | ## 📋 STEP 1: Get Customer Data… |
| Extract Email | Set | Normalize email / company name | Get Contact by ID | Enrich by Email | ## 📋 STEP 1: Get Customer Data… |
| Sticky Note11 | Sticky Note | Step 2 documentation (influencers.club) |  |  | ## 🔍 STEP 2: Enrich with influencers.club… |
| Enrich by Email | HTTP Request | Enrich email via influencers.club | Extract Email | Filter - Is Creator?; Stop and Error1 (error output) | ## 🔍 STEP 2: Enrich with influencers.club… |
| Stop and Error1 | Stop and Error | Stop on enrichment HTTP failure | Enrich by Email (error output) |  | ## 🔍 STEP 2: Enrich with influencers.club… |
| Sticky Note13 | Sticky Note | Step 3 documentation (tier/niche) |  |  | ## 🏷️ STEP 3: Classify Influencer… |
| Filter - Is Creator? | IF | Proceed only if is_creator | Enrich by Email | Classify Tier & Niche (true) | ## 🏷️ STEP 3: Classify Influencer… |
| Classify Tier & Niche | Code | Compute tier/niche/platform metrics | Filter - Is Creator? | Influencer Outreach Agent | ## 🏷️ STEP 3: Classify Influencer… |
| Sticky Note - AI Outreach1 | Sticky Note | Step 4 documentation (AI outreach) |  |  | ## ✉️ STEP 4: AI-Powered Personalized Outreach… |
| OpenAI Chat Model1 | OpenAI Chat Model (LangChain) | LLM for agent |  | Influencer Outreach Agent (ai_languageModel) | ## ✉️ STEP 4: AI-Powered Personalized Outreach… |
| Structured Output Parser1 | Structured Output Parser | Enforce JSON schema output |  | Influencer Outreach Agent (ai_outputParser) | ## ✉️ STEP 4: AI-Powered Personalized Outreach… |
| Influencer Outreach Agent | LangChain Agent | Generate outreach JSON | Classify Tier & Niche | Enrich CRM with Creator Data | ## ✉️ STEP 4: AI-Powered Personalized Outreach… |
| Sticky Note - CRM Enrichment | Sticky Note | Step 5 documentation (HubSpot properties) |  |  | ## 💾 STEP 5: ENRICH CRM WITH INFLUENCER DATA… (create 10 custom properties) |
| Enrich CRM with Creator Data | HubSpot | Update HubSpot with creator/outreach fields | Influencer Outreach Agent | Route by Priority | ## 💾 STEP 5: ENRICH CRM WITH INFLUENCER DATA… |
| Sticky Note - Priority Routing | Sticky Note | Step 6 documentation (routing) |  |  | ## 🚦 STEP 6: Route by Priority… |
| Route by Priority | IF | Route high vs normal/medium | Enrich CRM with Creator Data | Send from Founder (High Priority); Send an email | ## 🚦 STEP 6: Route by Priority… |
| Sticky Note - Email Sending | Sticky Note | Step 7 documentation (sending options) |  |  | ## 📧 STEP 7: Send Outreach Emails… |
| Send from Founder (High Priority) | Gmail | Send high-priority outreach | Route by Priority (true) |  | ## 📧 STEP 7: Send Outreach Emails… |
| Sticky Note | Sticky Note | SendGrid credential/setup notes |  |  | ## 📧 SendGrid Setup (3 min)… |
| Send an email | SendGrid | Send normal/medium outreach | Route by Priority (false) |  | ## 📧 SendGrid Setup (3 min)… |
| Sticky Note1 | Sticky Note | External explanation link |  |  | ## Discover tech influencers… [Full explanation](https://influencers.club/creatorbook/discover-influencers-among-saas-clients/) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name it (example): *Find creators among saas customers on Hubspot and tailor outreach with Gmail and SendGrid*.

2. **Add “HubSpot Trigger”**
   - Node type: **HubSpot Trigger**
   - Credentials: HubSpot Developer (or appropriate HubSpot OAuth / Private App credentials depending on your setup)
   - Configure the **event type(s)** you want (e.g., contact created, contact property changed).
   - Set concurrency (optional): `maxConcurrentRequests = 5`.

3. **Add “Get Contact by ID”**
   - Node type: **HubSpot**
   - Operation: **Contact → Get**
   - Authentication: **App Token**
   - Contact ID expression: `{{ $json.contactId }}`
   - Properties to request: include at least **email** (add it explicitly), plus any others you need (firstname, company, etc.).
   - Connect: **HubSpot Trigger → Get Contact by ID**

4. **Add “Extract Email”**
   - Node type: **Set**
   - Add fields:
     - `Email` = `{{ $json.properties.email.value }}`
     - (Optional) `Company Name`  
       If you truly need associated company name, you will likely need an additional HubSpot API call to fetch associations; otherwise remove this mapping to avoid undefined paths.
   - Connect: **Get Contact by ID → Extract Email**

5. **Add “Enrich by Email”**
   - Node type: **HTTP Request**
   - Method: **POST**
   - URL: `https://api-dashboard.influencers.club/public/v1/creators/enrich/email/`
   - Authentication: **Generic Credential Type → HTTP Header Auth**
     - Create credential (example name: *Enrichment API*)
     - Add header for API key as required by influencers.club (typically `Authorization: Bearer <key>` or vendor-specific header—match their docs).
   - Send Body: true, Body Parameters:
     - `email` = `{{ $json.Email }}`
   - Add header: `Accept: application/json`
   - Error handling: set **On Error → Continue (error output)** (so you get a second “error” output).
   - Connect: **Extract Email → Enrich by Email**

6. **Add “Stop and Error”**
   - Node type: **Stop and Error**
   - Message: “API enrichment failed - contact is not a creator or API error”
   - Connect: **Enrich by Email (error output) → Stop and Error**

7. **Add “Filter - Is Creator?”**
   - Node type: **IF**
   - Condition (boolean equals true):
     - Left: `{{ $json.result?.is_creator }}`
     - Right: `true`
   - Connect: **Enrich by Email (success output) → Filter - Is Creator?**
   - Leave the **false** output unconnected (or connect to a “No-op”/logging path if you want).

8. **Add “Classify Tier & Niche”**
   - Node type: **Code**
   - Paste the classification JS (logic: pick platform, compute followers/engagement, tier thresholds, niche regex matching).
   - Connect: **Filter - Is Creator? (true) → Classify Tier & Niche**

9. **Add AI components**
   1) **OpenAI Chat Model**
   - Node type: **OpenAI Chat Model (LangChain)**
   - Credentials: OpenAI API key
   - Model: `gpt-4o`
   - Temperature: `0.8`

   2) **Structured Output Parser**
   - Node type: **Structured Output Parser**
   - Schema: manual JSON schema for:
     - `outreach.subject`, `outreach.message`, `outreach.priority`, `outreach.program`, `outreach.talking_points`

   3) **Influencer Outreach Agent**
   - Node type: **AI Agent (LangChain)**
   - Prompt type: “Define”
   - Input text: `{{ JSON.stringify($json) }}`
   - System message: use the provided rules (JSON-only, tier-based messaging, constraints, etc.)
   - Enable output parsing and attach the structured output parser.

   - Connections:
     - **Classify Tier & Niche → Influencer Outreach Agent**
     - **OpenAI Chat Model → Influencer Outreach Agent** (as AI Language Model connection)
     - **Structured Output Parser → Influencer Outreach Agent** (as AI Output Parser connection)

10. **Add “Enrich CRM with Creator Data” (HubSpot update)**
   - Node type: **HubSpot**
   - Authentication: App Token
   - Choose an update operation that supports updating a contact by email or first search then update by ID (recommended):
     - Option A (common): **Search contact by email → Update contact by ID**
     - Option B (if node supports): update by email directly
   - Map HubSpot custom properties (create them first in HubSpot as described in the sticky note):
     - is_creator
     - influencer_tier
     - influencer_niche
     - follower_count
     - engagement_rate
     - twitter_handle (or primary handle)
     - creator_bio
     - outreach_program
     - outreach_priority
     - outreach_sent_date
   - Values should come from:
     - `classification.*` (tier, niche, followers, engagement_rate, username, bio, platform)
     - `Influencer Outreach Agent` output: `output.outreach.program`, `output.outreach.priority`
     - timestamp: `{{ $now }}`
   - Connect: **Influencer Outreach Agent → Enrich CRM with Creator Data**

11. **Add “Route by Priority”**
   - Node type: **IF**
   - Condition (string equals):
     - `{{ $json.output.outreach.priority }}` equals `high`
   - Connect: **Enrich CRM with Creator Data → Route by Priority**
   - If HubSpot node output doesn’t include `output.outreach`, instead reference the agent directly in the IF node:
     - `{{ $('Influencer Outreach Agent').first().json.output.outreach.priority }}`

12. **Add Gmail sender for high priority**
   - Node type: **Gmail**
   - Operation: **Send**
   - Credentials: Gmail OAuth2 (founder mailbox)
   - To: `{{ $('Extract Email').first().json.Email }}`
   - Subject: `{{ $('Influencer Outreach Agent').first().json.output.outreach.subject }}`
   - Message: `{{ $('Influencer Outreach Agent').first().json.output.outreach.message }}`
   - Connect: **Route by Priority (true/high) → Send from Founder**

13. **Add SendGrid sender for normal/medium**
   - Node type: **SendGrid**
   - Resource: **Mail**
   - Credentials: SendGrid API key
     - Ensure Sender Authentication is configured in SendGrid.
   - To: pick a consistent email source; recommended:
     - `{{ $('Extract Email').first().json.Email }}`
   - Subject/message: from agent output as above
   - Connect: **Route by Priority (false) → SendGrid node**

14. **(Optional but recommended) Add logging**
   - The sticky notes mention Google Sheets + Slack, but they are not present in this JSON. Add them if you want tracking and notifications.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Discover tech influencers among your SaaS clients and enrich the data for personalized partnership and content outreach. Full explanation link preserved. | https://influencers.club/creatorbook/discover-influencers-among-saas-clients/ |
| “Returns null if not a creator - this is normal!” | influencers.club enrichment behavior note (Sticky Note11) |
| HubSpot requires 10 custom properties to store enrichment & outreach metadata | Listed in “STEP 5: ENRICH CRM WITH INFLUENCER DATA” sticky note |
| SendGrid setup: Sender Authentication + API Key + add to n8n credentials; free tier limit noted | “SendGrid Setup (3 min)” sticky note |
| Priority logic: “high” sent from Founder Gmail; normal/medium sent via email service | “STEP 6/7” sticky notes |

---