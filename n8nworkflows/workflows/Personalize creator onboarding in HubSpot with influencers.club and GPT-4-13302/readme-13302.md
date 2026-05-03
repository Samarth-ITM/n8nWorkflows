Personalize creator onboarding in HubSpot with influencers.club and GPT-4

https://n8nworkflows.xyz/workflows/personalize-creator-onboarding-in-hubspot-with-influencers-club-and-gpt-4-13302


# Personalize creator onboarding in HubSpot with influencers.club and GPT-4

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Personalize creator onboarding by enriching a newly created creator/contact using their email (via influencers.club), then computing (a) an AI-based primary niche + module recommendations and (b) a rule-based tier classification. Finally, the workflow aggregates everything and updates the creator record in HubSpot (or another CRM).

**Target use cases**
- Creator onboarding flows where you want automatic segmentation (niche + tier) to drive modules/features, messaging, and activation steps.
- CRM enrichment for sales/CS teams with social profile links and audience metrics.
- Multi-source triggering (HubSpot trigger by default; webhook / sheets / DB alternatives documented via sticky note).

**Logical blocks**
1.1 **Trigger & Contact Data Retrieval (HubSpot)** → Trigger on new contact, fetch contact, extract email + ID.  
1.2 **Profile Enrichment (influencers.club)** → POST email to enrichment endpoint; route errors to a stop node.  
1.3 **Personalization (Parallel: AI niche + Rule tier)** → GPT-based niche selection (strict structured JSON) and Python tier classification.  
1.4 **Merge, Aggregate & Persist (HubSpot update)** → Combine AI + rules, aggregate into one object, update CRM record.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Contact Data Retrieval (HubSpot)

**Overview:** Starts when a HubSpot event fires, retrieves the full contact record, and extracts the creator email and contact ID for downstream enrichment and CRM updates.

**Nodes involved**
- `HubSpot Trigger1`
- `Get Contact by ID`
- `Extract Email`

#### Node: HubSpot Trigger1
- **Type / role:** HubSpot Trigger (`n8n-nodes-base.hubspotTrigger`) — entry point; listens to HubSpot events.
- **Configuration (interpreted):**
  - Uses HubSpot Developer API credentials (developer app) to register a webhook subscription.
  - `maxConcurrentRequests: 5` to limit parallel processing.
  - `eventsUi.eventValues` is effectively unspecified in this export (shows an empty object), so in practice you must select the intended event(s) in n8n (commonly “Contact creation” / “Contact property change”).
- **Key data produced:** Expects `contactId` in the trigger payload (used immediately in next node).
- **Connections:** Outputs to `Get Contact by ID`.
- **Failure modes / edge cases:**
  - Webhook subscription misconfigured or missing event selection → trigger won’t fire or payload may not contain `contactId`.
  - HubSpot app permissions missing → webhook registration or event delivery failures.
  - High-volume spikes → may exceed HubSpot webhook delivery limits; concurrency helps but does not solve upstream throttling.

#### Node: Get Contact by ID
- **Type / role:** HubSpot node (`n8n-nodes-base.hubspot`) — fetches a contact record.
- **Configuration:**
  - Operation: **get contact**
  - Authentication: **App Token**
  - `contactId` is taken from expression: `{{ $json.contactId }}`
- **Key outputs used later:**
  - `properties.email.value`
  - `vid` (used as `contact_ID`)
- **Connections:** Outputs to `Extract Email`.
- **Failure modes / edge cases:**
  - `contactId` missing/undefined → expression resolves to empty → API call fails.
  - Contact deleted or inaccessible → 404/permission error.
  - HubSpot API rate limiting (429).

#### Node: Extract Email
- **Type / role:** Set node (`n8n-nodes-base.set`) — normalizes fields for the enrichment request.
- **Configuration:**
  - Creates:
    - `Email` (string): `{{ $json.properties.email.value }}`
    - `contact_ID` (number): `{{ $json.vid }}`
- **Connections:** Outputs to `Enrich by Email`.
- **Failure modes / edge cases:**
  - Contact missing email property → `Email` becomes `null/undefined`, causing enrichment to fail or return empty results.
  - `vid` may differ depending on HubSpot account/API changes; ensure it exists in your environment.

---

### 2.2 Profile Enrichment (influencers.club)

**Overview:** Calls influencers.club enrichment endpoint with the creator’s email to retrieve cross-platform data (Instagram/TikTok/YouTube/X/etc.). If the HTTP request errors, the workflow routes to a stop node for manual attention.

**Nodes involved**
- `Enrich by Email`
- `Stop and Error`

#### Node: Enrich by Email
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls influencers.club API.
- **Configuration:**
  - Method: **POST**
  - URL: `https://api-dashboard.influencers.club/public/v1/creators/enrich/email/`
  - Body parameter: `email = {{ $json.Email }}`
  - Headers: `Accept: application/json`
  - Authentication: **Generic Credential Type → HTTP Header Auth** (expects API key in headers via credential)
  - **Error handling:** `onError: continueErrorOutput`
    - This enables a second output path for failures.
- **Connections:**
  - **Success output (main index 0):** feeds both `AI Niche Analyzer` and `Tier Classifier` in parallel.
  - **Error output (main index 1):** routes to `Stop and Error`.
- **Failure modes / edge cases:**
  - Missing/invalid API key → 401/403.
  - Email not found or no matching creator profile → may return empty/partial `result`.
  - Timeout or transient API failures → triggers error output; consider retries/backoff if needed.
  - Response schema changes (e.g., platform field names) → can break tier classifier assumptions.

#### Node: Stop and Error
- **Type / role:** Stop and Error (`n8n-nodes-base.stopAndError`) — fails the execution with a fixed message.
- **Configuration:**
  - Error message: `"Attention needed!"`
- **Connections:** None (terminal).
- **Failure modes / edge cases:** This is intentional termination; consider replacing with a notification path (Slack/Email) for production operations.

---

### 2.3 Personalization (Parallel: AI niche + Rule tier)

**Overview:** Two analyses run in parallel off the enrichment output:
- An LLM chain that selects a single “primary niche” and recommended modules, producing **strict structured JSON**.
- A Python-based tier classifier that uses follower/subscriber counts to assign tier, main platform, and feature configuration.

**Nodes involved**
- `OpenAI GPT-4` (language model provider)
- `AI Niche Analyzer` (LLM chain)
- `Structured Output Parser` (schema enforcement)
- `Tier Classifier` (Python code)

#### Node: OpenAI GPT-4
- **Type / role:** OpenAI Chat Model node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides the language model to the chain.
- **Configuration:**
  - Model: `gpt-4o-mini`
  - Temperature: `0.3` (more deterministic, better for structured outputs)
  - Credential: OpenAI API key (configured in node credential)
- **Connections:**
  - Connected as `ai_languageModel` input into `AI Niche Analyzer` (LangChain wiring).
- **Failure modes / edge cases:**
  - Missing/invalid OpenAI credentials → auth failures.
  - Model availability changes or org policy restrictions.
  - Rate limits / token limits if enrichment payload is very large (the chain stringifies the whole `$json`).

#### Node: Structured Output Parser
- **Type / role:** Structured output parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — validates and parses LLM output into JSON using a JSON Schema.
- **Configuration:**
  - Manual schema requiring:
    - `primary_niche` enum (beauty/fashion/…/other)
    - `enabled_modules` object with booleans: `benchmarks`, `brand_marketplace`, `templates`, `playbooks`
    - `reasoning` string
  - `additionalProperties: false` ensures no extra keys.
- **Connections:**
  - Feeds `AI Niche Analyzer` as its `ai_outputParser`.
- **Failure modes / edge cases:**
  - If the model outputs non-JSON or violates schema → parsing fails and the chain errors.
  - Enum mismatch (typos/casing) → parsing fails.

#### Node: AI Niche Analyzer
- **Type / role:** LLM Chain (`@n8n/n8n-nodes-langchain.chainLlm`) — prompts GPT to choose one niche and module toggles based on enriched creator profile data.
- **Configuration:**
  - PromptType: “define”
  - Prompt embeds full creator profile: `{{ JSON.stringify($json, null, 2) }}`
  - Strict instruction: “Return ONLY raw JSON”
  - Has output parser enabled (wired to `Structured Output Parser`)
  - Uses the OpenAI model via `OpenAI GPT-4` connection
- **Inputs/outputs:**
  - **Input:** Enrichment response JSON from `Enrich by Email` (likely includes a `result` object with platform details and niche signals like `content.ai_niches`, `content.primary_niches` if present).
  - **Output:** Parsed JSON object with `primary_niche`, `enabled_modules`, `reasoning`.
- **Connections:** Outputs to `Merge AI + Rules` (input 0).
- **Failure modes / edge cases:**
  - Oversized enrichment payload → LLM token overflow; mitigation: pre-filter JSON before stringify.
  - Missing expected niche signals → model may pick less accurate niche; still constrained by enum.
  - Output parser failures stop downstream processing unless you add an error branch.

#### Node: Tier Classifier
- **Type / role:** Code node (`n8n-nodes-base.code`) using **pythonNative** — deterministic tier and feature configuration.
- **Configuration (logic summary):**
  - Expects enrichment payload under `item.json.result`.
  - Safely reads platform follower/subscriber counts:
    - Instagram: `follower_count`
    - TikTok: `follower_count`
    - YouTube: `subscriber_count`
    - Twitter: `follower_count`
  - Determines:
    - `max_followers`: max across platforms
    - `main_platform`: platform with max audience
    - `is_multi_platform`: at least 2 platforms with >= 1000 audience
    - Tier:
      - `nano` < 5k
      - `micro` < 50k
      - `mid` < 250k
      - `macro` >= 250k
  - Adds tier-specific UI/modules (`show_first`, `enable_tools`, `show_modules`)
  - Bonuses:
    - If `has_brand_deals`: adds deal tools and inserts “Brand Deal Manager”
    - If multi-platform: adds cross-platform tools and “Cross-Platform Manager”
- **Outputs:** A normalized `tier_config` JSON object (tier + modules/tools).
- **Connections:** Outputs to `Merge AI + Rules` (input 1).
- **Failure modes / edge cases:**
  - Enrichment response shape differs (no `result`) → all counts default to 0 → tier becomes nano; main_platform selection still returns a key but may be misleading.
  - Non-numeric follower values (strings/null) → Python `max()` could behave unexpectedly; current code assumes numbers.
  - Platforms beyond these four (e.g., Twitch/OnlyFans) are ignored unless you extend the code.

---

### 2.4 Merge, Aggregate & Persist (HubSpot update)

**Overview:** Combines AI niche output with tier config, aggregates all data into a single object, then updates the CRM contact record using the email.

**Nodes involved**
- `Merge AI + Rules`
- `Aggregate Final Data`
- `Update CRM Record`

#### Node: Merge AI + Rules
- **Type / role:** Merge (`n8n-nodes-base.merge`) — combines the two parallel branches.
- **Configuration:**
  - Mode: `combine`
  - Combine by: `combineAll` (pairs/combines all incoming items)
- **Inputs/outputs:**
  - Input 0: from `AI Niche Analyzer`
  - Input 1: from `Tier Classifier`
  - Output: a combined item containing both datasets.
- **Connections:** Outputs to `Aggregate Final Data`.
- **Failure modes / edge cases:**
  - If one branch returns 0 items (e.g., AI chain fails) merge may output nothing or error depending on execution; consider adding error handling for AI failures.
  - If either branch returns multiple items unexpectedly, combineAll can create cross-product-like combinations.

#### Node: Aggregate Final Data
- **Type / role:** Aggregate (`n8n-nodes-base.aggregate`) — collapses item data into a single object.
- **Configuration:**
  - Aggregate mode: `aggregateAllItemData` (combines all item JSON into one structure)
- **Output:** One item representing final combined personalization payload.
- **Connections:** Outputs to `Update CRM Record`.
- **Failure modes / edge cases:**
  - Key collisions when aggregating (same field names from AI and tier config) can overwrite; ensure namespaces if needed (e.g., `ai.*` vs `tier.*`).

#### Node: Update CRM Record
- **Type / role:** HubSpot node (`n8n-nodes-base.hubspot`) — intended to update the contact in HubSpot with enriched/personalized properties.
- **Configuration (as exported):**
  - Authentication: **App Token**
  - Identifies contact by email: `{{ $('Extract Email').item.json.Email }}`
  - `options.resolveData: true`
  - **Important:** The node’s specific operation and property mapping are not configured in the export (it shows only email and additionalFields empty). In practice, you must choose an update operation (e.g., “Upsert contact” / “Update contact”) and map fields to HubSpot properties.
- **Connections:** Terminal (no downstream nodes).
- **Failure modes / edge cases:**
  - If no operation/fields configured → node may do nothing or error depending on n8n version/UI state.
  - Email not found in HubSpot → update may fail unless using an upsert operation.
  - HubSpot custom properties not created → property update fails.
  - Rate limiting / validation errors for property types (numbers vs strings).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Workflow Overview | Sticky Note | Documentation / overview | — | — | ## 🎯 Creator Onboarding Personalization Workflow; Requirements; Template Variables |
| Sticky Note | Sticky Note | Documentation / external link | — | — | ## Personalize creator onboarding flows to increase activation rates; [Full explanation](https://influencers.club/creatorbook/personalize-onboarding-for-creators/) |
| Alternative Triggers | Sticky Note | Documentation / alternative entry points | — | — | ## 🎯 ALTERNATIVE TRIGGERS; Webhook / Google Sheets / Database options |
| Step 1 Header1 | Sticky Note | Documentation for step 1 | — | — | ## 📥 STEP 1: Get New Signups; extract email/contact ID |
| HubSpot Trigger1 | HubSpot Trigger | Entry trigger (new signup/contact event) | — | Get Contact by ID | ## 📥 STEP 1: Get New Signups; extract email/contact ID |
| Get Contact by ID | HubSpot | Fetch contact details | HubSpot Trigger1 | Extract Email | ## 📥 STEP 1: Get New Signups; extract email/contact ID |
| Extract Email | Set | Normalize email + contact ID | Get Contact by ID | Enrich by Email | ## 📥 STEP 1: Get New Signups; extract email/contact ID |
| Step 2 Header1 | Sticky Note | Documentation for enrichment step | — | — | ## 🔍 STEP 2: Enrich Creator Profile; configure API key; fallback flow |
| Enrich by Email | HTTP Request | Enrich creator profile from influencers.club | Extract Email | AI Niche Analyzer; Tier Classifier; Stop and Error (error path) | ## 🔍 STEP 2: Enrich Creator Profile; configure API key; fallback flow |
| Stop and Error | Stop and Error | Hard-stop on enrichment error | Enrich by Email (error output) | — | ## 🔍 STEP 2: Enrich Creator Profile; configure API key; fallback flow |
| Step 4 Header1 | Sticky Note | Documentation for AI + rules step | — | — | ## 🎯 STEP 3: AI-Powered Personalization; parallel analyses; merge |
| OpenAI GPT-4 | OpenAI Chat Model (LangChain) | LLM provider for niche analysis | — | AI Niche Analyzer (as ai_languageModel) | ## 🎯 STEP 3: AI-Powered Personalization; parallel analyses; merge |
| Structured Output Parser | LangChain Output Parser | Enforce JSON schema for LLM output | — | AI Niche Analyzer (as ai_outputParser) | ## 🎯 STEP 3: AI-Powered Personalization; parallel analyses; merge |
| AI Niche Analyzer | LangChain LLM Chain | Determine primary niche + modules | Enrich by Email; OpenAI GPT-4; Structured Output Parser | Merge AI + Rules | ## 🎯 STEP 3: AI-Powered Personalization; parallel analyses; merge |
| Tier Classifier | Code (Python) | Tier classification + feature config | Enrich by Email | Merge AI + Rules | ## 🎯 STEP 3: AI-Powered Personalization; parallel analyses; merge |
| Merge AI + Rules | Merge | Combine AI niche + tier outputs | AI Niche Analyzer; Tier Classifier | Aggregate Final Data | ## 🎯 STEP 3: AI-Powered Personalization; parallel analyses; merge |
| Aggregate Final Data | Aggregate | Collapse to single final object | Merge AI + Rules | Update CRM Record | ## 💾 STEP 4: Save to Your CRM; map enriched data to properties |
| Step 5 Header1 | Sticky Note | Documentation for CRM update step | — | — | ## 💾 STEP 4: Save to Your CRM; mapping guidance |
| Update CRM Record | HubSpot | Persist enrichment + personalization in CRM | Aggregate Final Data | — | ## 💾 STEP 4: Save to Your CRM; mapping guidance |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it similar to: *Personalize creator onboardings with social platforms data*.
- (Optional) Add sticky notes with the provided explanatory content to mirror the original layout.

2) **Add the HubSpot trigger**
- Add node: **HubSpot Trigger** → name: `HubSpot Trigger1`.
- Configure HubSpot Developer/App credentials so n8n can register a webhook subscription.
- Select the event(s) you want (commonly “Contact created”).
- Ensure the trigger payload includes a `contactId` (or adjust downstream expressions to match your payload shape).

3) **Fetch the contact by ID**
- Add node: **HubSpot** → name: `Get Contact by ID`.
- Operation: **Get Contact**.
- Authentication: **App Token** (or OAuth if preferred).
- Contact ID field: expression `{{ $json.contactId }}`.
- Connect: `HubSpot Trigger1` → `Get Contact by ID`.

4) **Extract the creator email and contact ID**
- Add node: **Set** → name: `Extract Email`.
- Add fields:
  - `Email` (String) = `{{ $json.properties.email.value }}`
  - `contact_ID` (Number) = `{{ $json.vid }}`
- Connect: `Get Contact by ID` → `Extract Email`.

5) **Configure influencers.club enrichment request**
- Add node: **HTTP Request** → name: `Enrich by Email`.
- Method: **POST**
- URL: `https://api-dashboard.influencers.club/public/v1/creators/enrich/email/`
- Send body: enabled → body parameter `email` = `{{ $json.Email }}`
- Send headers: enabled → header `Accept: application/json`
- Authentication:
  - Choose **Generic Credential Type**
  - Select **HTTP Header Auth**
  - Create credential (e.g., `Enrichment API`) that adds the required API key header for influencers.club (exact header name/value per your influencers.club account).
- Set **Error Handling**: “Continue (error output)” (equivalent to `onError: continueErrorOutput`).
- Connect: `Extract Email` → `Enrich by Email`.

6) **Add an error stop (fallback path)**
- Add node: **Stop and Error** → name: `Stop and Error`.
- Error message: `Attention needed!`
- Connect: `Enrich by Email` **error output** → `Stop and Error`.

7) **Add the OpenAI model node**
- Add node: **OpenAI Chat Model** (LangChain) → name: `OpenAI GPT-4`.
- Credentials: configure your **OpenAI API key** in n8n credentials.
- Model: `gpt-4o-mini` (or equivalent available model).
- Temperature: `0.3`.

8) **Add the Structured Output Parser**
- Add node: **Structured Output Parser** → name: `Structured Output Parser`.
- Schema type: manual.
- Paste/configure a JSON Schema matching:
  - `primary_niche` enum (beauty/fashion/fitness/food/lifestyle/gaming/finance/education/parenting/travel/tech/music/business/entertainment/other)
  - `enabled_modules` booleans: benchmarks, brand_marketplace, templates, playbooks
  - `reasoning` string
  - Disallow additional properties if you want strictness.

9) **Add the AI niche analyzer chain**
- Add node: **LLM Chain** → name: `AI Niche Analyzer`.
- Set prompt to include:
  - Role/instructions
  - Embedded creator profile: `{{ JSON.stringify($json, null, 2) }}`
  - Explicit “STRICT JSON ONLY” output format
  - Niche enum list and module decision rules (as in the workflow).
- Connect LangChain ports:
  - `OpenAI GPT-4` → `AI Niche Analyzer` as **ai_languageModel**
  - `Structured Output Parser` → `AI Niche Analyzer` as **ai_outputParser**
- Connect main data:
  - `Enrich by Email` **success output** → `AI Niche Analyzer` (main)

10) **Add the tier classifier**
- Add node: **Code** → name: `Tier Classifier`.
- Language: **Python (Native)**.
- Paste the tiering logic:
  - Read from `item.json.result`
  - Compute max audience, main platform, multi-platform flag
  - Tier thresholds (<5k nano, <50k micro, <250k mid, else macro)
  - Add brand-deal and multi-platform bonuses
  - Output `tier_config` JSON
- Connect: `Enrich by Email` **success output** → `Tier Classifier` (main)

11) **Merge AI + rules outputs**
- Add node: **Merge** → name: `Merge AI + Rules`.
- Mode: **Combine**
- Combine By: **Combine All**
- Connect:
  - `AI Niche Analyzer` → `Merge AI + Rules` (Input 0)
  - `Tier Classifier` → `Merge AI + Rules` (Input 1)

12) **Aggregate to a single final payload**
- Add node: **Aggregate** → name: `Aggregate Final Data`.
- Aggregation mode: **Aggregate All Item Data**
- Connect: `Merge AI + Rules` → `Aggregate Final Data`.

13) **Update the CRM record (HubSpot)**
- Add node: **HubSpot** → name: `Update CRM Record`.
- Authentication: **App Token**.
- Identify the contact:
  - Use email: `{{ $('Extract Email').item.json.Email }}`
  - (Alternative: use contact ID if you prefer consistent identity.)
- Choose the appropriate HubSpot operation (commonly **Update** or **Upsert** contact).
- Map fields from `Aggregate Final Data` into HubSpot properties, for example:
  - Primary niche → `primary_niche`
  - Tier code/label/follower_count → tier properties
  - Main platform + multi-platform flag
  - Enabled modules booleans
  - Any platform links and follower metrics from enrichment `result`
- Connect: `Aggregate Final Data` → `Update CRM Record`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Personalize creator onboarding flows to increase activation rates; enrich creator data with multi social intelligence and analytics using the influencer.club API | https://influencers.club/creatorbook/personalize-onboarding-for-creators/ |
| The workflow is designed to accept alternative triggers (Webhook, Google Sheets, DB triggers) as long as an `email` field is produced for the enrichment call. | Covered in the “ALTERNATIVE TRIGGERS” sticky note |
| Consider adding retry/backoff and alerting for influencers.club and OpenAI calls to reduce manual intervention. | Operational recommendation (not tied to a specific node) |
| If LLM payload size is large, pre-filter enrichment JSON before passing to GPT to avoid token limits. | AI Niche Analyzer prompt currently stringifies entire input JSON |