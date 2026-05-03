Enrich newsletter subscribers with cross-platform social data and tag in Mailchimp

https://n8nworkflows.xyz/workflows/enrich-newsletter-subscribers-with-cross-platform-social-data-and-tag-in-mailchimp-13293


# Enrich newsletter subscribers with cross-platform social data and tag in Mailchimp

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow listens for **new Mailchimp newsletter subscribers**, enriches them with **cross-platform social data** using the **influencers.club enrichment API**, uses **GPT-4o-mini** to classify creator attributes into a strict JSON schema, determines a **routing/onboarding flow** via business rules, then prepares **Mailchimp tags** for activation.

**Target use cases:**
- Segmenting subscribers into creator-related cohorts (non-creator, aspiring, active).
- Detecting high-value/dormant creators and prioritizing outreach.
- Automatically applying structured tags to drive downstream email/CRM flows.

**Logical blocks (by dependency chain):**
1.1 **Trigger & Intake (Mailchimp)** → receives subscribe event.  
1.2 **Normalization / Validation** → extracts email/name/list/subscriber id, validates email.  
1.3 **API Enrichment (influencers.club)** → POST by email, continues on error.  
1.4 **AI Classification (GPT + JSON schema)** → forces structured classification fields.  
1.5 **Routing Logic (Code rules)** → determines flow/priority/slack_alert, etc.  
1.6 **Tag Preparation & Mailchimp Action** → formats tags and calls Mailchimp member-tag endpoint (note: configured as *delete*, see edge cases).

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Intake (Mailchimp)

**Overview:**  
Receives a Mailchimp “subscribe” webhook event for a specific audience/list and starts the workflow with the raw subscriber payload.

**Nodes involved:**
- **Mailchimp On-Subscriber Trigger**

#### Node: Mailchimp On-Subscriber Trigger
- **Type / role:** `n8n-nodes-base.mailchimpTrigger` — webhook trigger for Mailchimp events.
- **Configuration (interpreted):**
  - Audience/List: `94509e241e`
  - Events: `subscribe`
  - Sources: `user` (only user-originated subscribes)
  - Auth: OAuth2 (Mailchimp)
- **Inputs/Outputs:**
  - Entry node (no inputs)
  - Output → **Extract Subscriber Present Data**
- **Version specifics:** node typeVersion `1`
- **Edge cases / failures:**
  - OAuth2 token expiration/revocation → trigger stops receiving events.
  - Event payload fields can vary depending on Mailchimp settings (merge fields, marketing permissions, etc.).

---

### 2.2 Normalization / Validation

**Overview:**  
Normalizes subscriber data into a consistent internal structure (email, names, list id, subscriber id, source) and rejects items without an email.

**Nodes involved:**
- **Extract Subscriber Present Data**

#### Node: Extract Subscriber Present Data
- **Type / role:** `n8n-nodes-base.code` — data normalization and validation.
- **Configuration choices:**
  - Mode: “run once for each item”
  - Detects three formats:
    1) Mailchimp subscribe: `item.type === 'subscribe' && item.data`
    2) ActiveCampaign-like: `item.contact`
    3) Direct: `item.email`
  - Validates email presence; throws error if missing.
  - Normalizes email to `lowercase().trim()`.
- **Key variables produced (output JSON):**
  - `email`, `first_name`, `last_name`, `tags`, `source`
  - `list_id`, `subscriber_id`
  - `timestamp`
  - `original_data` (raw payload)
- **Inputs/Outputs:**
  - Input ← **Mailchimp On-Subscriber Trigger**
  - Output → **Influencers.club - Enrichment API by Email**
- **Version specifics:** typeVersion `2`
- **Edge cases / failures:**
  - Missing `item.data.email` or unexpected webhook payload shape → throws `No email found in trigger data`.
  - If used with non-Mailchimp sources later, ensure the event structure matches one of the supported branches.

---

### 2.3 API Enrichment (influencers.club)

**Overview:**  
Calls influencers.club enrichment endpoint using the subscriber email. The node is configured to continue even if the API call fails, allowing downstream logic to run with missing enrichment.

**Nodes involved:**
- **Influencers.club - Enrichment API by Email**

#### Node: Influencers.club - Enrichment API by Email
- **Type / role:** `n8n-nodes-base.httpRequest` — HTTP POST enrichment request.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api-dashboard.influencers.club/public/v1/creators/enrich/email/`
  - Body: `{ email: {{$json.email}} }`
  - Headers: `Accept: application/json`
  - Authentication: Generic credential type using **HTTP Header Auth** (API key in headers via credential)
  - **onError:** `continueErrorOutput` (important: does not hard-fail workflow)
- **Inputs/Outputs:**
  - Input ← **Extract Subscriber Present Data**
  - Output (success) → **Classificator**
  - Output (error branch index 1) → not connected (error results are currently dropped)
- **Version specifics:** typeVersion `4.2`
- **Edge cases / failures:**
  - Credential missing/invalid API key → 401/403; workflow continues but downstream classification may be based on an error payload or incomplete data.
  - Rate limits / timeouts → partial enrichment; because error output is not connected, you lose structured error handling unless you wire that branch.
  - Response shape changes → could break AI prompt assumptions if fields like `instagram.follower_count` are missing or renamed.

---

### 2.4 AI Classification (GPT + enforced schema)

**Overview:**  
Uses GPT-4o-mini to deterministically classify creator attributes from the enrichment JSON and enforces a strict JSON schema with an output parser.

**Nodes involved:**
- **OpenAI Chat Model**
- **Structured Output Parser**
- **Classificator**

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — language model provider for the agent.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
- **Inputs/Outputs:**
  - Output (AI languageModel) → **Classificator**
- **Credentials:** OpenAI API credential (“N8N open AI”)
- **Version specifics:** typeVersion `1.3`
- **Edge cases / failures:**
  - OpenAI credential issues, quota exceeded, model unavailable.
  - Latency/timeouts on large payloads (the agent is fed `JSON.stringify($json)`).

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces JSON schema output.
- **Configuration choices:**
  - Schema type: manual JSON schema
  - Required keys:
    - `creator_status` ∈ {non_creator, aspiring_creator, active_creator}
    - `creator_tier` ∈ {nano, micro, mid, macro}
    - `creator_type` array of {written, videos, shorts, podcasts}
    - `primary_niche` ∈ {lifestyle, business, fitness, entertainment, education, other}
    - `primary_platform` ∈ {instagram, tiktok, youtube, unknown}
    - `intent_signals` array of {affiliate_links, has_brand_deals, sells_products}
  - Optional: `primary_platform_why`
  - `additionalProperties: false` (strict)
- **Inputs/Outputs:**
  - Output (AI outputParser) → **Classificator**
- **Version specifics:** typeVersion `1.3`
- **Edge cases / failures:**
  - If the model returns extra keys or invalid enums, parsing fails.
  - If enrichment data is missing, model must still output valid values (e.g., `unknown`, `[]`), otherwise parser fails.

#### Node: Classificator
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — agent that produces schema-compliant classification.
- **Configuration choices:**
  - Prompt input text: `={{ JSON.stringify($json) }}` (entire enrichment response)
  - System message includes explicit deterministic rules:
    - Status based on `is_creator` and follower thresholds
    - Tier based on max audience across IG/TikTok/YouTube
    - Primary platform tie-break rules (audience → posting frequency → fixed preference)
    - Intent signal detection (flags + URL heuristics)
  - `hasOutputParser: true` (uses **Structured Output Parser**)
- **Inputs/Outputs:**
  - Input ← **Influencers.club - Enrichment API by Email**
  - AI model input ← **OpenAI Chat Model**
  - Output parser input ← **Structured Output Parser**
  - Main output → **Determine Routing**
- **Version specifics:** typeVersion `3.1`
- **Edge cases / failures:**
  - If upstream HTTP node returns an error object (because onError continues), the agent may classify garbage input; consider guarding against `$json.error`.
  - Large JSON may exceed model context or increase cost/latency.

---

### 2.5 Routing Logic (Business rules)

**Overview:**  
Applies rule-based routing based on classification + activity recency + follower thresholds to produce a `routing` object (flow, priority, slack_alert).

**Nodes involved:**
- **Determine Routing**

#### Node: Determine Routing
- **Type / role:** `n8n-nodes-base.code` — deterministic routing decision engine.
- **Configuration choices / logic highlights:**
  - Computes dormancy by checking last post dates across platforms; dormant if **most recent post ≥ 90 days**.
    - Instagram: `most_recent_post_date`
    - TikTok: `last_post_date`
    - YouTube: `last_uploaded`
  - Computes `maxFollowers` across IG/TikTok/YouTube/Twitter.
  - “High value” if `maxFollowers >= 50,000`.
  - Routing outcomes (examples):
    - Dormant + high value → `dormant_reactivation`, priority `medium-high`, `slack_alert: true`
    - non_creator → `standard_newsletter`, priority `low`
    - aspiring_creator → `education_and_growth`, priority `medium`
    - active_creator + mid/macro → `partnership_team`, priority `high`, slack alert
    - active_creator + micro + has_brand_deals → `fast_track_ambassador`, high + slack
    - active_creator + micro default → `affiliate_partnership`, `medium-high`
    - active_creator + nano default → `ambassador_program`, `medium`
  - Output includes `classification` and `routing`.
- **Key expressions/variables:**
  - `const classification = inputData.classification || inputData.output || {}`
  - `const creatorData = inputData.creator_data || inputData.creatorData || {}`
  - `email`/`list_id`/`subscriber_id` sourced from `subscriber` or top-level.
- **Inputs/Outputs:**
  - Input ← **Classificator**
  - Output → **Prepare Tags for Mailchimp**
- **Version specifics:** typeVersion `2`
- **Edge cases / failures:**
  - If the enrichment response does not place platform data where expected (e.g., no `instagram` object), dormancy check yields `false` and maxFollowers `0`.
  - If classification is missing (parser failure), the node may route to fallback `standard_newsletter` only if it runs; but parser failures usually stop execution earlier.

---

### 2.6 Tag Preparation & Mailchimp Action

**Overview:**  
Builds a list of tags from classification/routing and calls Mailchimp’s member tag endpoint. The Mailchimp node is currently configured to **delete** selected tags, which is inconsistent with the stated goal of tagging.

**Nodes involved:**
- **Prepare Tags for Mailchimp**
- **Apply Tags to Mailchimp Member**

#### Node: Prepare Tags for Mailchimp
- **Type / role:** `n8n-nodes-base.code` — converts classification/routing into Mailchimp tag strings.
- **Configuration choices:**
  - Mode: run once per item
  - Tag conventions:
    - `status:<creator_status>`
    - `tier:<creator_tier>`
    - `platform:<primary_platform>` (skips if `unknown`)
    - `niche:<primary_niche>`
    - `flow:<routing.flow>`
    - `type:<each creator_type>`
    - `signal:<each intent_signal>`
    - `priority:<routing.priority>`
  - Outputs: `{ email, list_id, subscriber_id, tags, classification, routing }`
- **Inputs/Outputs:**
  - Input ← **Determine Routing**
  - Output → **Apply Tags to Mailchimp Member**
- **Version specifics:** typeVersion `2`
- **Edge cases / failures:**
  - If arrays are empty, no type/signal tags are added (fine).
  - If `classification.primary_platform` is missing, it won’t add a platform tag; schema enforcement should prevent this.

#### Node: Apply Tags to Mailchimp Member
- **Type / role:** `n8n-nodes-base.mailchimp` — modifies tags for a Mailchimp list member.
- **Configuration choices (as configured):**
  - Resource: `memberTag`
  - Operation: **`delete`** (removes tags)
  - List: `94509e241e`
  - Email: `={{ $('Mailchimp On-Subscriber Trigger').first().json.data.email }}`
  - Tags passed: only **three** elements by index:
    - `={{ $json.tags[1] }}`
    - `={{ $json.tags[2] }}`
    - `={{ $json.tags[7] }}`
  - Auth: OAuth2
- **Inputs/Outputs:**
  - Input ← **Prepare Tags for Mailchimp**
  - Output → none
- **Version specifics:** typeVersion `1`
- **Edge cases / failures (important):**
  - **Likely misconfiguration:** operation is `delete`, but the workflow intention (and sticky note) says “applies tags”. In Mailchimp node, you likely want `add` (or “update/replace” depending on node capabilities).
  - Using `tags[1]`, `tags[2]`, `tags[7]` is fragile:
    - If fewer tags exist, expressions evaluate to `undefined`, possibly causing API errors.
    - It ignores most tags produced by the previous node.
  - Email source is pulled from the trigger node, not from the current item (`$json.email`); this matters if you later expand to batch processing or different triggers.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Mailchimp On-Subscriber Trigger | mailchimpTrigger | Entry point: listen to subscribe events | — | Extract Subscriber Present Data | ## 📥 STEP 1: TRIGGER\n\n**Listens for new subscribers**\n\nSources:\n• Mailchimp: subscribe events\n• ActiveCampaign: new contacts\n• Drift\n\nExtracts:\n• Email (required)\n• First/Last name\n• Tags\n• Source platform |
| Extract Subscriber Present Data | code | Normalize/validate subscriber fields | Mailchimp On-Subscriber Trigger | Influencers.club - Enrichment API by Email | ## 🔍 STEP 2: EXTRACT DATA\n\n**Normalizes different input formats**\n\nHandles 3 sources:\n1. Mailchimp: `data.email`, `data.merge_fields.FNAME`\n2. ActiveCampaign: `contact.email`, `contact.first_name`\n3. Direct: `email` field\n\nValidation:\n✅ Email is required - throws error if missing\n✅ Email cleaned (lowercase, trimmed)\n✅ Source detected automatically |
| Influencers.club - Enrichment API by Email | httpRequest | Enrich subscriber by email via influencers.club | Extract Subscriber Present Data | Classificator | ## 🔍 STEP 3: API ENRICHMENT\n\n**Calls influencers.club API**\n\nEndpoint: `/public/v1/creators/enrich/email/`\n\nResponse includes:\n• Main platform data\n• Follower counts & engagement\n• Niche classifications\n• Brand deal signals\n• Audience demographics |
| OpenAI Chat Model | lmChatOpenAi | Provides GPT-4o-mini model to agent | — | Classificator | ## 🤖 STEP 4: AI CLASSIFICATION\n\n**GPT-4 determines creator attributes**\n\nClassifies:\n• Creator Status: non_creator \| aspiring_creator \| active_creator\n• Creator Tier: nano \| micro \| mid \| macro\n• Primary Platform: Instagram, TikTok, YouTube\n• Creator Type: written, videos, shorts, podcasts\n• Niche: lifestyle, business, fitness, etc.\n• Intent Signals: affiliate_links, has_brand_deals, sells_products\n\n⚙️ MODEL: gpt-4o-mini\n🔒 OUTPUT: Enforced JSON schema |
| Structured Output Parser | outputParserStructured | Enforces strict JSON schema output | — | Classificator | ## 🤖 STEP 4: AI CLASSIFICATION\n\n**GPT-4 determines creator attributes**\n\nClassifies:\n• Creator Status: non_creator \| aspiring_creator \| active_creator\n• Creator Tier: nano \| micro \| mid \| macro\n• Primary Platform: Instagram, TikTok, YouTube\n• Creator Type: written, videos, shorts, podcasts\n• Niche: lifestyle, business, fitness, etc.\n• Intent Signals: affiliate_links, has_brand_deals, sells_products\n\n⚙️ MODEL: gpt-4o-mini\n🔒 OUTPUT: Enforced JSON schema |
| Classificator | langchain.agent | AI-driven classification with deterministic rules | Influencers.club - Enrichment API by Email; OpenAI Chat Model; Structured Output Parser | Determine Routing | ## 🤖 STEP 4: AI CLASSIFICATION\n\n**GPT-4 determines creator attributes**\n\nClassifies:\n• Creator Status: non_creator \| aspiring_creator \| active_creator\n• Creator Tier: nano \| micro \| mid \| macro\n• Primary Platform: Instagram, TikTok, YouTube\n• Creator Type: written, videos, shorts, podcasts\n• Niche: lifestyle, business, fitness, etc.\n• Intent Signals: affiliate_links, has_brand_deals, sells_products\n\n⚙️ MODEL: gpt-4o-mini\n🔒 OUTPUT: Enforced JSON schema |
| Determine Routing | code | Rule-based flow/priority selection | Classificator | Prepare Tags for Mailchimp | ## 🛤️ STEP 5: ROUTING LOGIC\n\n**Business rules determine onboarding flow**\n\nDecision factors:\n✓ Creator tier (50K+ = high value)\n✓ Activity status (90+ days = dormant)\n✓ Intent signals (brand deals = fast track)\n✓ Platform presence |
| Prepare Tags for Mailchimp | code | Build Mailchimp tag list from classification/routing | Determine Routing | Apply Tags to Mailchimp Member | ## 🏷️ STEP 6: MAILCHIMP TAGGING\n\n**Applies tags based on classification**\n\nTags applied:\n• Creator Status tag\n• Creator Tier tag\n• Primary Platform tag\n• Primary Niche tag\n• Routing Flow tag\n• Intent Signal tags (multiple)\n\nExample:\n• `creator:active_creator`\n• `tier:micro`\n• `platform:instagram`\n• `niche:fitness`\n• `flow:affiliate_partnership`\n• `signal:has_brand_deals` |
| Apply Tags to Mailchimp Member | mailchimp | Modify Mailchimp member tags (configured as delete) | Prepare Tags for Mailchimp | — | ## 🏷️ STEP 6: MAILCHIMP TAGGING\n\n**Applies tags based on classification**\n\nTags applied:\n• Creator Status tag\n• Creator Tier tag\n• Primary Platform tag\n• Primary Niche tag\n• Routing Flow tag\n• Intent Signal tags (multiple)\n\nExample:\n• `creator:active_creator`\n• `tier:micro`\n• `platform:instagram`\n• `niche:fitness`\n• `flow:affiliate_partnership`\n• `signal:has_brand_deals` |
| STEP 1: TRIGGER1 | stickyNote | Documentation block label | — | — |  |
| STEP 2: EXTRACT DATA1 | stickyNote | Documentation block label | — | — |  |
| STEP 3: API ENRICHMENT1 | stickyNote | Documentation block label | — | — |  |
| STEP 4: AI CLASSIFICATION | stickyNote | Documentation block label | — | — |  |
| STEP 5: ROUTING LOGIC | stickyNote | Documentation block label | — | — |  |
| STEP 6: MAILCHIMP TAGGING | stickyNote | Documentation block label | — | — |  |
| Sticky Note | stickyNote | Global description + link | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**  
   - Name: “Enrich newsletter subscribers with cross-platform social data from influencers.club and activate them with Mailchimp”

2) **Add Trigger: “Mailchimp On-Subscriber Trigger”**  
   - Node type: *Mailchimp Trigger*  
   - Authentication: **OAuth2** (connect your Mailchimp account)  
   - List/Audience: `94509e241e`  
   - Events: `subscribe`  
   - Sources: `user`

3) **Add Code node: “Extract Subscriber Present Data”**  
   - Node type: *Code*  
   - Mode: “Run Once for Each Item”  
   - Paste logic that:
     - Extracts `email`, `merge_fields.FNAME/LNAME`, `list_id`, `id` from Mailchimp subscribe payload  
     - Normalizes email to lowercase/trim  
     - Throws an error if email is missing  
     - Outputs: `email, first_name, last_name, tags, source, list_id, subscriber_id, timestamp, original_data`

4) **Connect**: Trigger → Extract Subscriber Present Data

5) **Add HTTP Request node: “Influencers.club - Enrichment API by Email”**  
   - Node type: *HTTP Request*  
   - Method: `POST`  
   - URL: `https://api-dashboard.influencers.club/public/v1/creators/enrich/email/`  
   - Send body: yes (body parameter `email` = `{{$json.email}}`)  
   - Headers: `Accept: application/json`  
   - Authentication: **Header Auth** via credential (store your influencers.club API key in the credential)  
   - Error handling: set node to **Continue on Fail** (equivalent to `continueErrorOutput`)
   - (Recommended) Connect error output to a handler if you want observability.

6) **Connect**: Extract Subscriber Present Data → Influencers.club - Enrichment API by Email

7) **Add OpenAI model node: “OpenAI Chat Model”**  
   - Node type: *OpenAI Chat Model (LangChain)*  
   - Credentials: OpenAI API key  
   - Model: `gpt-4o-mini`

8) **Add Output Parser node: “Structured Output Parser”**  
   - Node type: *Structured Output Parser (LangChain)*  
   - Schema: create a manual JSON schema with:
     - required: `creator_status, creator_tier, creator_type, primary_niche, primary_platform, intent_signals`
     - enums exactly as in the workflow
     - `additionalProperties: false`

9) **Add Agent node: “Classificator”**  
   - Node type: *AI Agent (LangChain)*  
   - Prompt type: “Define”  
   - Text/Input: `{{ JSON.stringify($json) }}`  
   - System message: paste the classification rules (status/tier/platform tie-breaks/type/niche/intent rules).  
   - Enable “Use Output Parser” and select **Structured Output Parser**.

10) **Wire AI nodes**  
   - Connect **OpenAI Chat Model** → (AI languageModel connection) → **Classificator**  
   - Connect **Structured Output Parser** → (AI outputParser connection) → **Classificator**  
   - Connect **Influencers.club - Enrichment API by Email** → **Classificator** (main)

11) **Add Code node: “Determine Routing”**  
   - Node type: *Code*  
   - Mode: “Run Once for Each Item”  
   - Implement:
     - Dormancy check (≥90 days since most recent post)
     - maxFollowers across platforms
     - rule table to output `routing.flow`, `routing.priority`, `routing.slack_alert`, etc.
   - Output should include: `email, list_id, subscriber_id, classification, routing, timestamp`

12) **Connect**: Classificator → Determine Routing

13) **Add Code node: “Prepare Tags for Mailchimp”**  
   - Node type: *Code*  
   - Build `tags[]` from classification/routing (status/tier/platform/niche/flow/type/signal/priority)
   - Output should include `email`, `list_id`, `subscriber_id`, `tags`

14) **Connect**: Determine Routing → Prepare Tags for Mailchimp

15) **Add Mailchimp node: “Apply Tags to Mailchimp Member”**  
   - Node type: *Mailchimp*  
   - Credentials: Mailchimp OAuth2  
   - Resource: “Member Tag” (or equivalent)  
   - List: `94509e241e`  
   - Email: **use** `{{$json.email}}` (recommended)  
   - **Operation:** to *apply* tags, select **Add** (or “Update”) rather than Delete.
   - Tags: ideally pass the whole list (how you do this depends on the node UI; if it only accepts fixed fields, you may need to map multiple tag entries or loop over tags with an item list).

16) **Connect**: Prepare Tags for Mailchimp → Apply Tags to Mailchimp Member

17) **Credentials checklist**
   - Mailchimp OAuth2: connected, list accessible.
   - influencers.club: HTTP Header Auth credential with API key.
   - OpenAI: valid API key with access to `gpt-4o-mini`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Enrich newsletter subscribers with social media data and direct them to personalized partner and ambassador email flows… Full explanation” | https://influencers.club/creatorbook/turn-newsletter-subscribers-into-creator-ambassadors/ |
| HTTP node note: “⚙️ CONFIGURE: Add your influencers.club API key in credentials” | Attached to the influencers.club HTTP Request node |
| Important configuration mismatch: Mailchimp tag node is set to **delete** and only references `tags[1]`, `tags[2]`, `tags[7]` | Review before production to ensure tags are applied as intended |

