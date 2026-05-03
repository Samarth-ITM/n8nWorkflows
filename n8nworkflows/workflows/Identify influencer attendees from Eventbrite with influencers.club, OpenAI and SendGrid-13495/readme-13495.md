Identify influencer attendees from Eventbrite with influencers.club, OpenAI and SendGrid

https://n8nworkflows.xyz/workflows/identify-influencer-attendees-from-eventbrite-with-influencers-club--openai-and-sendgrid-13495


# Identify influencer attendees from Eventbrite with influencers.club, OpenAI and SendGrid

## 1. Workflow Overview

**Purpose:**  
This workflow listens for **Eventbrite attendee events** (registered / updated / checked in), enriches each attendee by email using **influencers.club**, uses **OpenAI** to (1) classify the attendee’s creator profile and VIP routing, then (2) generate a **personalized outreach email**, and finally sends the email through **SendGrid**.

**Target use cases:**
- Automatically identifying creators/influencers among event attendees
- Routing attendees into VIP experience tiers based on audience size and signals
- Sending personalized onboarding / VIP invite emails at registration time or check-in

### 1.1 Input Reception (Eventbrite)
Receives attendee payloads from Eventbrite and normalizes them.

### 1.2 Attendee Normalization
Extracts email + identity + event/order context + any custom answers (including self-reported social handles).

### 1.3 Enrichment (influencers.club)
Calls influencers.club enrichment-by-email endpoint to obtain cross-platform creator profile.

### 1.4 Creator Gate (IF)
Checks whether the enrichment result indicates the attendee is a creator. If not, stops.

### 1.5 AI Classification + VIP Routing (OpenAI)
Classifies creator status/tier, niche, platform, behavior, intent, and computes VIP access + experience package.

### 1.6 Merge + AI Email Generation (OpenAI)
Merges classification with enriched profile and generates a structured email (subject, preheader, body, CTA).

### 1.7 Delivery (SendGrid)
Sends the generated email to the attendee’s email.

---

## 2. Block-by-Block Analysis

### Block 1 — Eventbrite Intake
**Overview:** Captures Eventbrite attendee lifecycle events and starts the workflow.  
**Nodes involved:** `Eventbrite Trigger1`

#### Node: Eventbrite Trigger1
- **Type / role:** `n8n-nodes-base.eventbriteTrigger` — webhook trigger for Eventbrite events.
- **Configuration (interpreted):**
  - Primary event: `attendee.registered`
  - Additional actions: `attendee.updated`, `attendee.checked_in`
  - Auth: OAuth2 (Eventbrite)
  - Webhook identifier: `eventbrite-creator-vip`
- **Input / Output:**
  - **Input:** none (trigger)
  - **Output:** Eventbrite attendee payload (shape varies by action type)
- **Potential failures / edge cases:**
  - OAuth2 token expired/invalid → trigger stops receiving events
  - Organization parameter appears empty (`organization: "="`) → may require selecting an organization in UI
  - Payload shape differences across actions → handled downstream in code

---

### Block 2 — Extract + Normalize Attendee
**Overview:** Normalizes Eventbrite payload into a consistent `attendee_context` object and enforces presence of email.  
**Nodes involved:** `Extract Attendee`

#### Node: Extract Attendee
- **Type / role:** `n8n-nodes-base.code` — transforms raw Eventbrite payload into normalized fields.
- **Configuration choices:**
  - Mode: “Run once for each item”
  - Extracts attendee from multiple possible payload shapes:
    - `payload.attendees?.[0]` OR `payload.attendee` OR `payload`
  - Detects trigger event (`registered`, `updated`, `checked_in`) using:
    - `attendee.checked_in === true`
    - `payload.action === 'attendee.updated' || payload.changed`
- **Key output fields:**
  - `attendee_context.email` (required; throws if missing)
  - Identity: `first_name`, `last_name`, `full_name`
  - Event/order: `order_id`, `event_id`, `event_name` (fallback: `"the event"`), `ticket_class`
  - Timestamps: `registration_date` (fallback: now), `checked_in`, `checked_in_at`
  - `custom_answers` map derived from attendee answers
  - `self_reported_handles` (instagram/tiktok/youtube) extracted from custom answers when present
- **Connections:**
  - **Input:** `Eventbrite Trigger1`
  - **Output:** `Influencers.club - Enrichment API by Email`
- **Potential failures / edge cases:**
  - Missing email → throws `Attendee has no email address — cannot enrich` (hard stop for that execution)
  - Missing/partial name fields → safe fallbacks (empty strings; `full_name` becomes `null` if blank)
  - Custom answers not present or not array → handled by defaulting to `[]`

---

### Block 3 — Enrichment via influencers.club
**Overview:** Enriches attendee by email to retrieve creator profile across platforms; continues even if the request errors.  
**Nodes involved:** `Influencers.club - Enrichment API by Email`

#### Node: Influencers.club - Enrichment API by Email
- **Type / role:** `n8n-nodes-base.httpRequest` — POST request to enrichment endpoint.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api-dashboard.influencers.club/public/v1/creators/enrich/email/`
  - Sends body and headers
  - Body parameter: `email = {{$json.attendee_context.email}}`
  - Header: `Accept: application/json`
  - Auth: Generic credential type → `httpHeaderAuth` (API key in header)
  - **Error handling:** `onError: continueErrorOutput` (workflow can proceed on error path)
- **Connections:**
  - **Input:** `Extract Attendee`
  - **Outputs:**
    - Main output (success): to `IS - Creator?`
    - Error output: to `No Operation, do nothing`
- **Potential failures / edge cases:**
  - Invalid/missing API key → 401/403
  - Rate limiting / 429
  - Non-JSON response or API downtime
  - Response schema changes (e.g., missing `result.is_creator`) will affect the IF condition downstream

---

### Block 4 — Creator Gate / Routing
**Overview:** Only proceeds to AI classification if enrichment indicates the attendee is a creator.  
**Nodes involved:** `IS - Creator?`, `No Operation, do nothing`

#### Node: IS - Creator?
- **Type / role:** `n8n-nodes-base.if` — conditional branch.
- **Condition (interpreted):**
  - Checks if `{{$json.result.is_creator}}` equals string `"true"`
  - Loose type validation enabled (but still compares to a string)
- **Connections:**
  - **Input:** `Influencers.club - Enrichment API by Email` (success path)
  - **True output:** to `AI Classificator` and also to `Merge` input index 1 (profile side)
  - **False output:** to `No Operation, do nothing`
- **Potential failures / edge cases:**
  - If API returns boolean `true` (not string) and node compares to `"true"`, behavior depends on “loose” validation; safest is to compare to boolean `true` instead.
  - If `result` or `is_creator` missing → condition evaluates false → creator might be skipped.

#### Node: No Operation, do nothing
- **Type / role:** `n8n-nodes-base.noOp` — explicit stop/do-nothing sink.
- **Connections:**
  - Receives from:
    - `IS - Creator?` false branch
    - `Influencers.club - Enrichment API by Email` error output
- **Potential failures / edge cases:** none (used as a terminator)

---

### Block 5 — AI Classification + VIP Routing
**Overview:** Uses an LLM to classify the enriched creator profile and compute VIP routing outputs in a strict JSON schema.  
**Nodes involved:** `AI Classificator`, `OpenAI (Classificator)`, `Structured Output Parser (Classificator)`

#### Node: AI Classificator
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LangChain agent orchestrating model + structured parser.
- **Configuration choices:**
  - **Input text:** JSON stringified from the enrichment node item:
    - `={{ JSON.stringify($('Influencers.club - Enrichment API by Email').item.json, null, 2) }}`
  - **System message:** detailed rule set that:
    - Determines `creator_status`, `creator_tier` by `max_followers`
    - Chooses `primary_platform` by platform audience metrics and tie-breaks
    - Infers `creator_type`, `primary_niche`, `posting_behavior`, `intent_signals`
    - Computes `vip_access_level`, `experience_package`, `badge_marker`, `personalized_invite_note`
  - `hasOutputParser: true` (requires structured output parser connection)
- **Connections:**
  - **Main output:** to `Merge` (classification side)
  - **AI language model:** from `OpenAI (Classificator)`
  - **AI output parser:** from `Structured Output Parser (Classificator)`
- **Potential failures / edge cases:**
  - Model returns invalid JSON → parser fails; node errors
  - Missing expected fields in enrichment data (e.g., no follower counts) → prompt defines defaults (e.g., nano tier)
  - Prompt assumes specific enrichment field names (e.g., `max_followers`, `engagement_percent`)—if API uses different keys, classification quality degrades

#### Node: OpenAI (Classificator)
- **Type / role:** `lmChatOpenAi` — provides the chat model to the agent.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Temperature/default options (not customized)
  - Credential: OpenAI API credential (`N8N open AI`)
- **Connections:** feeds `AI Classificator` via `ai_languageModel`
- **Potential failures / edge cases:**
  - Invalid API key / quota exceeded
  - Model unavailable or renamed
  - Latency/timeouts on large inputs

#### Node: Structured Output Parser (Classificator)
- **Type / role:** `outputParserStructured` — enforces JSON schema.
- **Configuration choices:**
  - Manual JSON schema with `additionalProperties: false`
  - Required fields include:
    - `creator_status`, `creator_tier`, `creator_type`, `primary_niche`, `primary_platform`, `primary_platform_why`
    - `posting_behavior` object (frequency, engagement_quality, platform_mix)
    - `intent_signals`, `vip_access_level`, `experience_package`, `badge_marker`, `personalized_invite_note`
- **Connections:** feeds `AI Classificator` via `ai_outputParser`
- **Potential failures / edge cases:**
  - If the model includes any extra keys → parse failure
  - If enums mismatch (e.g., niche not in allowed list) → parse failure

---

### Block 6 — Merge Classification + Profile
**Overview:** Combines the enrichment profile and the AI classification into one object for email generation.  
**Nodes involved:** `Merge`, `Merge Classification + Profile`

#### Node: Merge
- **Type / role:** `n8n-nodes-base.merge` — combines two incoming item streams.
- **Configuration choices:** default merge behavior (no explicit mode shown).
- **Connections:**
  - **Input 0:** from `AI Classificator` (classification)
  - **Input 1:** from `IS - Creator?` true branch (enrichment profile)
  - **Output:** to `Merge Classification + Profile`
- **Potential failures / edge cases:**
  - If the Merge node pairs items incorrectly (e.g., multiple items / concurrency), classification could mismatch profile. With triggers usually 1 item, it’s typically safe.
  - If `AI Classificator` errors, merge won’t receive input 0.

#### Node: Merge Classification + Profile
- **Type / role:** `n8n-nodes-base.code` — constructs a single merged JSON.
- **Configuration choices / logic:**
  - Assumes:
    - `items[0].json.output` contains classification (agent output)
    - `items[1].json.result` contains normalized enrichment profile
  - Outputs:
    - `{ normalized_profile: <items[1].json.result>, output: <items[0].json.output> }`
- **Connections:**
  - **Input:** `Merge`
  - **Output:** `Email Personalization Agent`
- **Potential failures / edge cases:**
  - If `items[0].json.output` is not present (agent node output shape changes), output becomes `{}` and later nodes may fail (missing vip_access_level, etc.)
  - If enrichment API returns profile not in `result`, normalized_profile becomes `{}`

---

### Block 7 — AI Email Generation + Delivery
**Overview:** Generates a structured, brand-controlled email based on classification and profile, then sends it via SendGrid.  
**Nodes involved:** `Email Personalization Agent`, `OpenAI (Email Agent)1`, `Structured Output Parser (Email)1`, `Send an email`

#### Node: Email Personalization Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LLM agent to produce structured email JSON.
- **Configuration choices:**
  - Input text: `={{ JSON.stringify($json, null, 2) }}`
  - System message defines:
    - Name resolution rules (avoid handles starting with `@`)
    - Event name resolution
    - Hard rules: don’t invent facts; don’t mention followers under 10k; JSON-only output; body length 150–250 words
    - Messaging strategy by `vip_access_level`
    - Fixed CTA URLs per vip level (exact URLs)
    - Output schema: `{ email: { subject, preheader, body, cta: { text, url }}}`
  - `hasOutputParser: true`
- **Connections:**
  - **Input:** `Merge Classification + Profile`
  - **Main output:** `Send an email`
  - **AI language model:** from `OpenAI (Email Agent)1`
  - **AI output parser:** from `Structured Output Parser (Email)1`
- **Potential failures / edge cases:**
  - If upstream `output.vip_access_level` missing → prompt says treat as standard, but only if the model follows; better to set a default in code before the agent if needed.
  - If model emits URL not matching schema `format: uri` → parser fails.
  - If model returns extra keys or non-JSON → parser fails.

#### Node: OpenAI (Email Agent)1
- **Type / role:** `lmChatOpenAi` — chat model for email generation.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Temperature: `0.7` (more creative)
  - Credential: OpenAI API credential (`N8N open AI`)
- **Connections:** feeds `Email Personalization Agent` via `ai_languageModel`
- **Potential failures / edge cases:** auth/quota/model availability/timeouts

#### Node: Structured Output Parser (Email)1
- **Type / role:** `outputParserStructured` — validates generated email JSON.
- **Configuration choices:**
  - Manual JSON schema requiring:
    - `email.subject`, `email.preheader`, `email.body`
    - `email.cta.text`, `email.cta.url` (must be a valid URI)
- **Connections:** feeds `Email Personalization Agent` via `ai_outputParser`
- **Potential failures / edge cases:**
  - Any non-URI in cta.url → parse failure
  - Missing required keys → parse failure

#### Node: Send an email
- **Type / role:** `n8n-nodes-base.sendGrid` — sends the final email.
- **Configuration choices:**
  - To: `={{ $('Extract Attendee').first().json.attendee_context.email }}`
  - Subject: `={{ $json.output.email.subject }}`
  - Body: `={{ $json.output.email.body }}`
  - Resource: `mail`
  - Credential: SendGrid API credential (`SendGrid account`)
- **Connections:**
  - **Input:** `Email Personalization Agent`
  - **Output:** none
- **Potential failures / edge cases:**
  - SendGrid auth errors / sender identity not verified
  - The agent output shape: this node expects `$json.output.email...` but the Email Agent schema outputs `{ email: {...} }`. Unless n8n wraps agent output into `output`, this may **misreference fields**.
    - If Email Personalization Agent returns `{ email: ... }`, the expressions should likely be:
      - Subject: `={{ $json.email.subject }}`
      - Body: `={{ $json.email.body }}`
  - “From” address not set here (relies on SendGrid defaults / credential settings); may fail depending on SendGrid configuration.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Pipeline Overview | stickyNote | Visual overview & credential checklist |  |  | ## 🎟️ Eventbrite Creator VIP Experience Pipeline / Flow Overview + credentials to configure |
| Sticky Note1 | stickyNote | Link + context description |  |  | ## Spot creators among event attendees… [Full explanation](https://influencers.club/creatorbook/spot-creators-among-event-attendees/) |
| Note: Trigger | stickyNote | Explains trigger events |  |  | ⏰ **Eventbrite Trigger** … |
| Eventbrite Trigger1 | eventbriteTrigger | Entry point: receive attendee events |  | Extract Attendee | ⏰ **Eventbrite Trigger** … |
| Note: Extract | stickyNote | Explains attendee extraction |  |  | 📧 **Extract Attendee** … |
| Extract Attendee | code | Normalize payload; require email | Eventbrite Trigger1 | Influencers.club - Enrichment API by Email | 📧 **Extract Attendee** … |
| Note: Enrich | stickyNote | Explains enrichment |  |  | 🔬 **Enrichment API** … |
| Influencers.club - Enrichment API by Email | httpRequest | Enrich attendee profile by email | Extract Attendee | IS - Creator?; No Operation, do nothing | 🔬 **Enrichment API** … |
| IS - Creator? | if | Gate: proceed only if is_creator | Influencers.club - Enrichment API by Email | AI Classificator; Merge; No Operation, do nothing |  |
| No Operation, do nothing | noOp | Sink for non-creators / API errors | IS - Creator? (false); Influencers.club… (error) |  |  |
| Note: Classify | stickyNote | Explains AI classification outputs |  |  | 🤖 **AI Classificator** … |
| OpenAI (Classificator) | lmChatOpenAi | LLM for classification |  | AI Classificator | 🤖 **AI Classificator** … |
| Structured Output Parser (Classificator) | outputParserStructured | Enforce classification JSON schema |  | AI Classificator | 🤖 **AI Classificator** … |
| AI Classificator | langchain.agent | Classify + VIP routing | IS - Creator? (true) | Merge | 🤖 **AI Classificator** … |
| Merge | merge | Combine classification and profile streams | AI Classificator; IS - Creator? | Merge Classification + Profile |  |
| Note: VIP Routing | stickyNote | Explains VIP package logic |  |  | 🎯 **VIP Routing** … |
| Merge Classification + Profile | code | Build `{normalized_profile, output}` object | Merge | Email Personalization Agent | 🎯 **VIP Routing** … |
| Note: Outreach Strategy | stickyNote | Brand voice + CTA link guidance |  |  | ## ✉️ Outreach Strategy — Customize Here … |
| OpenAI (Email Agent)1 | lmChatOpenAi | LLM for email writing |  | Email Personalization Agent | ✉️ **AI Email Agent** … + Outreach Strategy note |
| Structured Output Parser (Email)1 | outputParserStructured | Enforce email JSON schema |  | Email Personalization Agent | ✉️ **AI Email Agent** … + Outreach Strategy note |
| Email Personalization Agent | langchain.agent | Generate structured personalized email | Merge Classification + Profile | Send an email | ✉️ **AI Email Agent** … + Outreach Strategy note |
| Note: Email | stickyNote | Explains email generation + SendGrid |  |  | ✉️ **AI Email Agent** … |
| Send an email | sendGrid | Send email via SendGrid | Email Personalization Agent |  | ✉️ **AI Email Agent** … |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it (example): *Spot influencers among event attendees on Eventbrite to onboard as creators*.
- Ensure you have credentials ready: Eventbrite OAuth2, influencers.club API key (header auth), OpenAI API key, SendGrid API key.

2) **Add trigger: Eventbrite Trigger**
- Node type: **Eventbrite Trigger**
- Configure:
  - Event: `attendee.registered`
  - Actions: enable `attendee.updated` and `attendee.checked_in`
  - Authentication: OAuth2
  - Select your Eventbrite organization (required in most setups)
- Save and (optionally) test webhook in Eventbrite environment.

3) **Add Code node: “Extract Attendee”**
- Node type: **Code**
- Mode: **Run once for each item**
- Paste logic to:
  - Normalize attendee payload into `attendee_context`
  - Extract: email, name, event/order IDs, event_name fallback, checked_in flags, custom answers
  - Throw an error if email is missing
- Connect: **Eventbrite Trigger → Extract Attendee**

4) **Add HTTP Request node: “Influencers.club - Enrichment API by Email”**
- Node type: **HTTP Request**
- Configure:
  - Method: `POST`
  - URL: `https://api-dashboard.influencers.club/public/v1/creators/enrich/email/`
  - Body: `email` = expression `{{$json.attendee_context.email}}`
  - Headers: `Accept: application/json`
  - Authentication: **Header Auth** (Generic Credential)
    - Create credential **HTTP Header Auth**:
      - Header name/value per influencers.club requirements (e.g., `Authorization: Bearer <KEY>` or their specified header)
- Error handling:
  - Set **On Error** to “Continue (error output)”
- Connect: **Extract Attendee → Enrichment API**

5) **Add IF node: “IS - Creator?”**
- Node type: **IF**
- Condition:
  - Check `{{$json.result.is_creator}}` is true
  - Recommended robust condition: boolean equals `true` (avoid string `"true"` if API returns boolean)
- Connect:
  - **Enrichment API (success) → IS - Creator?**
  - **Enrichment API (error output) → NoOp** (next step)

6) **Add NoOp node: “No Operation, do nothing”**
- Node type: **No Operation**
- Connect:
  - **IS - Creator? (false) → NoOp**
  - **Enrichment API (error output) → NoOp**

7) **Add AI classification agent: “AI Classificator”**
- Node type: **AI Agent (LangChain)**
- Set **Prompt type**: “Define”
- Input text:
  - Use an expression that passes the enrichment JSON (for the current item), e.g.:
    - `{{ JSON.stringify($json, null, 2) }}`
  - (If you want to reference the enrichment node explicitly, ensure item linkage is correct.)
- System message:
  - Include your full classification + VIP routing rules (creator_status/tier, platform selection, niche normalization, posting behavior, intent signals, vip_access_level, experience_package, badge_marker, personalized_invite_note).
- Attach model + parser (next steps).
- Connect:
  - **IS - Creator? (true) → AI Classificator**

8) **Add OpenAI chat model node for classification**
- Node type: **OpenAI Chat Model**
- Model: `gpt-4o-mini` (or your preferred)
- Credentials: configure OpenAI API key
- Connect:
  - **OpenAI Chat Model → AI Classificator** via **AI Language Model** connection

9) **Add Structured Output Parser for classification**
- Node type: **Structured Output Parser**
- Schema: create a manual JSON schema enforcing exactly the required keys (no additional properties), matching:
  - creator_status, creator_tier, creator_type, primary_niche, primary_platform, primary_platform_why,
  - posting_behavior {frequency, engagement_quality, platform_mix},
  - intent_signals, vip_access_level, experience_package, badge_marker, personalized_invite_note
- Connect:
  - **Structured Output Parser → AI Classificator** via **AI Output Parser** connection

10) **Add Merge node**
- Node type: **Merge**
- Use it to combine:
  - classification output (from AI Classificator)
  - enrichment profile (from IS - Creator? true path / or directly from enrichment)
- Connect:
  - **AI Classificator → Merge (Input 0)**
  - **IS - Creator? (true) → Merge (Input 1)**

11) **Add Code node: “Merge Classification + Profile”**
- Node type: **Code**
- Logic:
  - Read classification from classification item (often `items[0].json.output`)
  - Read enrichment from enrichment item (often `items[1].json.result`)
  - Output `{ normalized_profile: ..., output: ... }`
- Connect: **Merge → Merge Classification + Profile**

12) **Add AI email agent: “Email Personalization Agent”**
- Node type: **AI Agent (LangChain)**
- Input text: `{{ JSON.stringify($json, null, 2) }}`
- System message:
  - Include name resolution rules, “no invented facts”, follower mention threshold (10k), body length, vip_access_level mapping, and fixed CTA URLs.
- Connect: **Merge Classification + Profile → Email Personalization Agent**

13) **Add OpenAI chat model node for email**
- Node type: **OpenAI Chat Model**
- Model: `gpt-4o-mini`
- Temperature: `0.7`
- Credentials: OpenAI
- Connect:
  - **OpenAI Chat Model → Email Personalization Agent** via **AI Language Model**

14) **Add Structured Output Parser for email**
- Node type: **Structured Output Parser**
- Schema requires:
  - `email.subject`, `email.preheader`, `email.body`, `email.cta.text`, `email.cta.url` (URI)
- Connect:
  - **Structured Output Parser → Email Personalization Agent** via **AI Output Parser**

15) **Add SendGrid node: “Send an email”**
- Node type: **SendGrid**
- Resource: **Mail**
- Credentials: configure SendGrid API key and ensure sender identity is set/verified
- Set:
  - To: `{{ $('Extract Attendee').first().json.attendee_context.email }}`
  - Subject: map to the email agent output (ensure the correct path, commonly `{{$json.email.subject}}`)
  - Body/content: `{{$json.email.body}}`
- Connect: **Email Personalization Agent → Send an email**

16) **Validate expressions and data shape**
- Run a test execution with a sample Eventbrite payload.
- Confirm:
  - Enrichment returns `result.is_creator`
  - Classification output is accessible where Merge code expects it
  - Email agent output paths match what SendGrid node references

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Spot creators among event attendees to deliver VIP experiences & awareness… Full explanation” | https://influencers.club/creatorbook/spot-creators-among-event-attendees/ |
| Credentials to configure: Eventbrite OAuth2, influencers.club header auth, OpenAI, SendGrid | Mentioned in “Eventbrite Creator VIP Experience Pipeline” note |
| Outreach strategy can be customized in the Email agent system message (tone, CTA URLs, follower threshold) | Mentioned in “Outreach Strategy — Customize Here” note |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.