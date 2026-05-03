Find and add lookalike influencers to HubSpot with influencers.club and GPT-4o-mini

https://n8nworkflows.xyz/workflows/find-and-add-lookalike-influencers-to-hubspot-with-influencers-club-and-gpt-4o-mini-13514


# Find and add lookalike influencers to HubSpot with influencers.club and GPT-4o-mini

## 1. Workflow Overview

This workflow listens for new HubSpot contacts, checks whether the new contact is a creator, finds similar creators through the influencers.club API, enriches each lookalike profile, analyzes them with GPT-4o-mini, and then creates or updates those lookalikes in HubSpot.

Its main use case is creator discovery for influencer marketing teams: starting from an existing high-performing creator in HubSpot, the workflow identifies comparable creators and pushes enriched, AI-qualified prospects back into the CRM.

### 1.1 Trigger and HubSpot Contact Retrieval
The workflow starts from a HubSpot trigger on contact creation, then fetches the full contact record by ID.

### 1.2 Seed Creator Preparation
It extracts contact fields, enriches the person by email through influencers.club, normalizes the returned creator data, and determines whether the contact is actually a creator.

### 1.3 Lookalike Discovery
If the contact is a creator, the workflow calls the influencers.club lookalike endpoint using the detected main platform and account handle.

### 1.4 Lookalike Enrichment and Normalization
Each returned lookalike is processed one at a time, enriched again for full profile data, and transformed into a normalized profile object.

### 1.5 AI Qualification and HubSpot Upsert
GPT-4o-mini produces a structured evaluation of the lookalike creator, and the workflow then sends the analyzed result into HubSpot for create/update handling.

### 1.6 Error Handling and Silent Drops
If the original enrichment fails, the workflow stops with an explicit error. Non-creators are dropped silently. Failed lookalike enrichments are also configured to continue without breaking the loop.

---

## 2. Block-by-Block Analysis

## 2.1 Workflow Documentation and Context Notes

**Overview:**  
This block contains all sticky notes used to describe the workflow, setup requirements, and per-stage behavior. These notes are not executable but are essential for maintainers and automation agents interpreting the workflow.

**Nodes Involved:**  
- 📋 OVERVIEW
- Note: Trigger
- Note: Get Contact
- Note: Extract Email
- Note: Enrich by Email
- Note: Stop and Error
- Note: Normalize Seed
- Note: Filter
- Note: Lookalike API
- Note: Loop
- Note: Enrich Lookalike1
- Note: AI Agent
- Note: HubSpot Out
- Sticky Note1

### Node Details

#### 📋 OVERVIEW
- **Type and role:** Sticky Note; high-level workflow description.
- **Configuration choices:** Documents purpose, flow, platform detection behavior, and required credentials.
- **Key expressions or variables used:** Mentions `main_platform`.
- **Input/output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or failure types:** None; documentation only.
- **Sub-workflow reference:** None.

#### Note: Trigger
- **Type and role:** Sticky Note; describes the HubSpot trigger behavior.
- **Configuration choices:** Notes that the trigger fires on `contact.creation` and can be extended with `contact.propertyChange`.
- **Input/output connections:** None.
- **Version-specific requirements:** Depends on HubSpot app event support.
- **Edge cases:** If the HubSpot app is not configured for the intended event, nothing will trigger.
- **Sub-workflow reference:** None.

#### Note: Get Contact
- **Type and role:** Sticky Note; explains HubSpot contact retrieval.
- **Configuration choices:** Mentions retrieved properties `firstname`, `company`, `hs_full_name_or_email`.
- **Edge cases:** Warns that more properties may need to be added for custom CRM setups.

#### Note: Extract Email
- **Type and role:** Sticky Note; describes field mapping into `Email` and `Company Name`.

#### Note: Enrich by Email
- **Type and role:** Sticky Note; explains the email-based enrichment call.
- **Edge cases:** Indicates failures go to Stop and Error.

#### Note: Stop and Error
- **Type and role:** Sticky Note; documents the hard-stop failure path.

#### Note: Normalize Seed
- **Type and role:** Sticky Note; explains the JavaScript normalization of the seed creator.
- **Configuration choices:** Notes platform auto-detection, follower/engagement tiering, growth trend detection, and URL cleanup.

#### Note: Filter
- **Type and role:** Sticky Note; describes creator-only filtering.
- **Configuration choices:** Documents true branch vs silent drop branch.

#### Note: Lookalike API
- **Type and role:** Sticky Note; explains the lookalike search and possible filters.
- **Configuration choices:** Notes `paging.limit` and optional filtering dimensions like engagement and brand deals.

#### Note: Loop
- **Type and role:** Sticky Note; describes sequential item processing using looping.

#### Note: Enrich Lookalike1
- **Type and role:** Sticky Note; explains full enrichment per lookalike.
- **Configuration choices:** Notes that errors are suppressed so the loop can continue.

#### Note: AI Agent
- **Type and role:** Sticky Note; explains GPT-4o-mini structured analysis output.
- **Configuration choices:** Suggests adjusting the system prompt for campaign context.

#### Note: HubSpot Out
- **Type and role:** Sticky Note; describes final HubSpot upsert.
- **Configuration choices:** Recommends mapping AI outputs into custom HubSpot properties.

#### Sticky Note1
- **Type and role:** Sticky Note; marketing/context note with external link.
- **Configuration choices:** Includes link to influencers.club content.
- **Context or link:** `https://influencers.club/creatorbook/find-similar-creators-to-top-performers/`

---

## 2.2 Trigger and Contact Retrieval

**Overview:**  
This block starts the workflow when a HubSpot contact is created and retrieves the full contact record using the emitted `contactId`.

**Nodes Involved:**  
- HubSpot Trigger
- Get Contact by ID

### Node Details

#### HubSpot Trigger
- **Type and technical role:** `n8n-nodes-base.hubspotTrigger`; event-driven entry point.
- **Configuration choices:**  
  - Uses a HubSpot Developer API credential.
  - Configured with one event in `eventsUi`; based on the note, this is intended for `contact.creation`.
  - `maxConcurrentRequests` is set to `5`.
- **Key expressions or variables used:** Outputs `contactId`.
- **Input and output connections:**  
  - No input node; entry point.
  - Outputs to **Get Contact by ID**.
- **Version-specific requirements:** `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - HubSpot webhook subscription not configured correctly.
  - Developer app missing required scopes.
  - Event mismatch if the app is not subscribed to contact creation.
  - Concurrency issues if HubSpot bursts exceed allowed processing rate.
- **Sub-workflow reference:** None.

#### Get Contact by ID
- **Type and technical role:** `n8n-nodes-base.hubspot`; retrieves a HubSpot contact by ID.
- **Configuration choices:**  
  - Operation: `get`
  - Authentication: HubSpot app token
  - Contact ID comes from `={{ $json.contactId }}`
  - Requests properties: `firstname`, `company`, `hs_full_name_or_email`
- **Key expressions or variables used:**  
  - `{{ $json.contactId }}`
- **Input and output connections:**  
  - Input from **HubSpot Trigger**
  - Output to **Extract Email**
- **Version-specific requirements:** `typeVersion: 2.2`
- **Edge cases or potential failure types:**  
  - Invalid or missing `contactId`
  - HubSpot auth/token failure
  - Property path differences depending on portal configuration
  - The later Set node expects fields that may not actually be included here, especially email and associated company structures
- **Sub-workflow reference:** None.

---

## 2.3 Seed Contact Extraction and Enrichment

**Overview:**  
This block maps HubSpot contact fields into simpler variables and sends the contact email to influencers.club for creator enrichment.

**Nodes Involved:**  
- Extract Email
- Enrich by Email
- Stop and Error

### Node Details

#### Extract Email
- **Type and technical role:** `n8n-nodes-base.set`; creates a simplified payload.
- **Configuration choices:**  
  - Assigns `Email` from `={{ $json.properties.email.value }}`
  - Assigns `Company Name` from `={{ $json["associated-company"].properties.name.value }}`
- **Key expressions or variables used:**  
  - `{{ $json.properties.email.value }}`
  - `{{ $json["associated-company"].properties.name.value }}`
- **Input and output connections:**  
  - Input from **Get Contact by ID**
  - Output to **Enrich by Email**
- **Version-specific requirements:** `typeVersion: 3.4`
- **Edge cases or potential failure types:**  
  - Likely field mismatch: the previous HubSpot node does not explicitly request `email`.
  - `associated-company` may not exist.
  - Expression evaluation failure if nested properties are absent.
- **Sub-workflow reference:** None.

#### Enrich by Email
- **Type and technical role:** `n8n-nodes-influencersclub.influencersClub`; creator enrichment by email.
- **Configuration choices:**  
  - Uses `email = {{ $json.Email }}`
  - `onError` is set to `continueErrorOutput`, which allows the error output branch to be used
- **Key expressions or variables used:**  
  - `{{ $json.Email }}`
- **Input and output connections:**  
  - Input from **Extract Email**
  - Main output 0 to **Normalize**
  - Error output to **Stop and Error**
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - Missing/invalid email
  - influencers.club auth errors
  - No creator found for the given email
  - Rate limits or transient API failures
- **Sub-workflow reference:** None.

#### Stop and Error
- **Type and technical role:** `n8n-nodes-base.stopAndError`; explicit execution termination.
- **Configuration choices:**  
  - Error message: `API enrichment failed - contact is not a creator or API error`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from **Enrich by Email** error branch
  - No outputs
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - This intentionally stops execution; useful for observability but may be noisy if many contacts are non-creators.
- **Sub-workflow reference:** None.

---

## 2.4 Seed Creator Normalization and Filtering

**Overview:**  
This block transforms the enrichment payload into a flat creator object, auto-detects the main platform, and filters out non-creators before lookalike search.

**Nodes Involved:**  
- Normalize
- Filter - Is Creator?

### Node Details

#### Normalize
- **Type and technical role:** `n8n-nodes-base.code`; JavaScript normalization for seed creator data.
- **Configuration choices:**  
  - Accepts either array-wrapped or direct payloads.
  - Extracts `result` if present.
  - Builds a platform map for TikTok, Instagram, YouTube, and Twitter.
  - Detects `main_platform` by highest follower/subscriber count.
  - Computes:
    - `followers_fmt`
    - `follower_tier`
    - `engagement_tier`
    - `growth_trend`
  - Normalizes handle and strips URL query params from images.
  - Outputs a flattened object suitable for filtering and lookalike lookup.
- **Key expressions or variables used:**  
  - Uses `$input.first().json[0] || $input.first().json`
  - Produces keys such as:
    - `email`
    - `full_name`
    - `handle`
    - `main_platform`
    - `profile_url`
    - `followers`
    - `engagement_percent`
    - `is_creator`
    - `platforms_available`
- **Input and output connections:**  
  - Input from **Enrich by Email**
  - Output to **Filter - Is Creator?**
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Unknown or empty platform data leads to `main_platform = 'unknown'`
  - Missing usernames cause `handle = null`
  - Assumes certain enrichment fields exist for each platform
  - YouTube URL construction depends on `custom_url`
- **Sub-workflow reference:** None.

#### Filter - Is Creator?
- **Type and technical role:** `n8n-nodes-base.if`; boolean gate.
- **Configuration choices:**  
  - Condition: `{{ $json.is_creator }}` equals `true`
  - Only the true branch is connected
- **Key expressions or variables used:**  
  - `{{ $json.is_creator }}`
- **Input and output connections:**  
  - Input from **Normalize**
  - True output to **Find Similar Creators**
  - False output unconnected, so non-creators are dropped silently
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - If `is_creator` is null/undefined/false, the item disappears without error
  - Strict boolean validation may reject non-boolean variants
- **Sub-workflow reference:** None.

---

## 2.5 Lookalike Discovery

**Overview:**  
This block queries influencers.club for similar creators based on the seed creator’s detected platform and handle, then splits the returned accounts into individual items.

**Nodes Involved:**  
- Find Similar Creators
- Split Out
- Loop Over Items (Lookalike)

### Node Details

#### Find Similar Creators
- **Type and technical role:** `n8n-nodes-influencersclub.influencersClub`; lookalike search.
- **Configuration choices:**  
  - Resource: `creator`
  - Operation: `findLookalikes`
  - Platform: `={{ $json.main_platform }}`
  - Filter key: `username`
  - Filter value: `={{ $json.handle }}`
  - Filter groups for TikTok, Twitch, Twitter, YouTube, OnlyFans, Instagram, and advanced filters are present but empty
- **Key expressions or variables used:**  
  - `{{ $json.main_platform }}`
  - `{{ $json.handle }}`
- **Input and output connections:**  
  - Input from **Filter - Is Creator?**
  - Output to **Split Out**
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - If `main_platform` is `unknown`, the API may fail or return no useful results
  - Missing handle causes poor or failed matching
  - API rate limit/auth failure
  - Empty result set
- **Sub-workflow reference:** None.

#### Split Out
- **Type and technical role:** `n8n-nodes-base.splitOut`; splits array field into separate items.
- **Configuration choices:**  
  - Field to split: `accounts`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from **Find Similar Creators**
  - Output to **Loop Over Items (Lookalike)**
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - Missing `accounts` field
  - `accounts` not being an array
  - Empty array leading to no downstream processing
- **Sub-workflow reference:** None.

#### Loop Over Items (Lookalike)
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; sequential loop controller.
- **Configuration choices:**  
  - Default batching behavior; effectively processes one item at a time in this pattern
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from **Split Out**
  - Output 1 to **Enrich by Handle (Full)** for the next item
  - Output 0 is loop completion
  - Receives a return connection from **AI Agent (Lookalike)** to continue the loop
- **Version-specific requirements:** `typeVersion: 3`
- **Edge cases or potential failure types:**  
  - Improper loop-back connection could stall the workflow
  - Large result sets may still create long-running executions
- **Sub-workflow reference:** None.

---

## 2.6 Lookalike Enrichment and Profile Normalization

**Overview:**  
This block takes each lookalike candidate, fetches the full creator profile, and converts potentially different response formats into one standardized structure for AI analysis.

**Nodes Involved:**  
- Enrich by Handle (Full)
- Normalize Profile (Lookalike)

### Node Details

#### Enrich by Handle (Full)
- **Type and technical role:** `n8n-nodes-influencersclub.influencersClub`; full creator enrichment by handle.
- **Configuration choices:**  
  - Resource: `creator`
  - Operation: `enrichByHandle`
  - Handle: `={{ $json.profile.username }}`
  - Platform: `={{ $('Normalize').item.json.main_platform }}`
  - `onError` is `continueErrorOutput`
- **Key expressions or variables used:**  
  - `{{ $json.profile.username }}`
  - `{{ $('Normalize').item.json.main_platform }}`
- **Input and output connections:**  
  - Input from **Loop Over Items (Lookalike)**
  - Output to **Normalize Profile (Lookalike)**
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - Missing `profile.username` in the lookalike payload
  - Platform mismatch if lookalike results are not actually on the same platform as the seed
  - API enrichment failure; configured not to abort execution
- **Sub-workflow reference:** None.

#### Normalize Profile (Lookalike)
- **Type and technical role:** `n8n-nodes-base.code`; transforms lookalike profile data into a consistent schema.
- **Configuration choices:**  
  - Runs once for each item.
  - Supports multiple source formats:
    - Format A: enriched result object
    - Format B: lightweight lookalike object with `profile`
    - Format C: already-flat enrichment
  - Creates nested `normalized_profile` with sections:
    - `identity`
    - `cross_platform_links`
    - `contact`
    - `audience`
    - `follower_growth`
    - `engagement`
    - `content`
    - `monetization`
    - `network`
    - `flags`
    - `dedup_key`
  - Includes helper logic for:
    - URL cleanup
    - handle normalization
    - follower formatting
    - growth trend calculation
    - cross-platform link inference
    - collaborator deduplication
    - hashtag extraction
- **Key expressions or variables used:**  
  - Uses `item.json`
  - Outputs `normalized_profile`
- **Input and output connections:**  
  - Input from **Enrich by Handle (Full)**
  - Output to **AI Agent (Lookalike)**
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Assumes Instagram-centric normalization in several branches
  - Some fields may be null if enrichment is partial
  - If input is an array unexpectedly, it maps over entries, which may change item cardinality
  - Deduplication key may be null when both handle and user ID are absent
- **Sub-workflow reference:** None.

---

## 2.7 AI Analysis

**Overview:**  
This block sends each normalized lookalike profile to a LangChain AI agent backed by GPT-4o-mini and forces the output into a strict JSON schema.

**Nodes Involved:**  
- AI Agent (Lookalike)
- OpenAI Model (Lookalike)1
- Structured Output Parser (Lookalike)1

### Node Details

#### OpenAI Model (Lookalike)1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; language model provider.
- **Configuration choices:**  
  - Model: `gpt-4o-mini`
  - No extra options set
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Connected as AI language model to **AI Agent (Lookalike)**
- **Version-specific requirements:** `typeVersion: 1.3`
- **Edge cases or potential failure types:**  
  - Invalid OpenAI credentials
  - Token/rate limit issues
  - Model availability changes
- **Sub-workflow reference:** None.

#### Structured Output Parser (Lookalike)1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; validates/parses model output against a manual JSON schema.
- **Configuration choices:**  
  - Manual schema with required fields:
    - `summary`
    - `strengths`
    - `concerns`
    - `audience_analysis`
    - `growth_analysis`
    - `content_analysis`
    - `brand_fit`
    - `flags`
    - `recommended_action`
    - `recommended_action_reason`
    - `estimated_reach_per_post`
  - Includes enums for engagement quality, trajectory, niche clarity, and recommended action
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Connected as AI output parser to **AI Agent (Lookalike)**
- **Version-specific requirements:** `typeVersion: 1.3`
- **Edge cases or potential failure types:**  
  - Model output may fail schema validation
  - Numeric fields may be returned as strings by the model
- **Sub-workflow reference:** None.

#### AI Agent (Lookalike)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; prompts the model and produces structured analysis.
- **Configuration choices:**  
  - Prompt type: defined manually
  - Uses `{{ JSON.stringify($json.normalized_profile, null, 2) }}`
  - Strong system instruction to return valid JSON only
  - Output parser enabled
- **Key expressions or variables used:**  
  - `{{ JSON.stringify($json.normalized_profile, null, 2) }}`
- **Input and output connections:**  
  - Main input from **Normalize Profile (Lookalike)**
  - AI model input from **OpenAI Model (Lookalike)1**
  - AI output parser input from **Structured Output Parser (Lookalike)1**
  - Main outputs to:
    - **HubSpot — Lookalike Creator**
    - **Loop Over Items (Lookalike)** to continue iteration
- **Version-specific requirements:** `typeVersion: 3.1`
- **Edge cases or potential failure types:**  
  - Schema mismatch despite parser
  - Hallucinated values if profile data is sparse
  - Long prompts if enriched profile payload becomes large
  - Depending on agent behavior, output may contain only parsed JSON and not the original profile fields, which matters for the next HubSpot node
- **Sub-workflow reference:** None.

---

## 2.8 HubSpot Output

**Overview:**  
This block is intended to create or update the analyzed lookalike in HubSpot.

**Nodes Involved:**  
- HubSpot — Lookalike Creator

### Node Details

#### HubSpot — Lookalike Creator
- **Type and technical role:** `n8n-nodes-base.hubspot`; intended CRM write-back node.
- **Configuration choices:**  
  - Authentication: HubSpot app token
  - `email` parameter is currently set to `=`, which is incomplete
  - No clear operation or property mapping is configured in the visible parameters
- **Key expressions or variables used:**  
  - Intended to use enriched email, but no valid expression is present
- **Input and output connections:**  
  - Input from **AI Agent (Lookalike)**
  - No downstream nodes
- **Version-specific requirements:** `typeVersion: 2.2`
- **Edge cases or potential failure types:**  
  - Incomplete configuration will likely fail at runtime
  - AI output may not include the email or normalized profile fields needed for upsert unless merged beforehand
  - Custom properties like `recommended_action` and `brand_fit` must exist in HubSpot before mapping
- **Sub-workflow reference:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| HubSpot Trigger | n8n-nodes-base.hubspotTrigger | Entry point on HubSpot contact creation |  | Get Contact by ID | ## HubSpot Trigger<br>Fires on `contact.creation`. Outputs `contactId` for the next node.<br>Add `contact.propertyChange` in HubSpot app settings to also catch updates. |
| Get Contact by ID | n8n-nodes-base.hubspot | Fetch HubSpot contact fields from contactId | HubSpot Trigger | Extract Email | ## Get Contact by ID<br>Fetches `firstname`, `company`, `hs_full_name_or_email` from HubSpot using `contactId`.<br>Add more properties to `propertiesCollection` if your CRM has custom creator fields. |
| Extract Email | n8n-nodes-base.set | Map HubSpot data to simplified fields | Get Contact by ID | Enrich by Email | ## Extract Email<br>Maps HubSpot fields into clean `Email` and `Company Name` variables. |
| Enrich by Email | n8n-nodes-influencersclub.influencersClub | Enrich seed contact through influencers.club email lookup | Extract Email | Normalize; Stop and Error | ## Enrich by Email<br>POSTs email to influencers.club and returns a full main profile info.<br>Error → Stop and Error node. |
| Stop and Error | n8n-nodes-base.stopAndError | Stop execution on failed seed enrichment | Enrich by Email |  | ## Stop and Error<br>Triggered when enrichment fails or returns no creator. Halts execution with a logged error message. |
| Normalize | n8n-nodes-base.code | Normalize seed creator profile and detect main platform | Enrich by Email | Filter - Is Creator? | ## Normalize (Code in JS)<br>Detects `main_platform` by comparing follower counts across all platform blocks. Computes `follower_tier`, `engagement_tier`, `growth_trend`. Strips CloudFront URL signing params.<br>Outputs a flat clean object for the filter and lookalike nodes. |
| Filter - Is Creator? | n8n-nodes-base.if | Keep only creator profiles | Normalize | Find Similar Creators | ## Filter — Is Creator?<br>Checks `is_creator === true`. Only real creators proceed.<br>**Port 0** → Lookalike API<br>**Port 1** → dead end, dropped silently |
| Find Similar Creators | n8n-nodes-influencersclub.influencersClub | Search for similar creators | Filter - Is Creator? | Split Out | ## Lookalike API<br>Finds similar creators using `main_platform` + `handle`. Returns up to 5 accounts.<br>Adjust `paging.limit` (max ~50) or add filters: `engagement_percent.min`, `number_of_followers`, `has_done_brand_deals: true`. |
| Split Out | n8n-nodes-base.splitOut | Split lookalike accounts array into individual items | Find Similar Creators | Loop Over Items (Lookalike) | ## Lookalike API<br>Finds similar creators using `main_platform` + `handle`. Returns up to 5 accounts.<br>Adjust `paging.limit` (max ~50) or add filters: `engagement_percent.min`, `number_of_followers`, `has_done_brand_deals: true`. |
| Loop Over Items (Lookalike) | n8n-nodes-base.splitInBatches | Process lookalikes sequentially | Split Out; AI Agent (Lookalike) | Enrich by Handle (Full) | ## Loop Over Items<br>Processes one lookalike at a time to avoid API rate limits.<br>**Port 0** → all done, exits loop<br>**Port 1** → next item → Enrichment API |
| Enrich by Handle (Full) | n8n-nodes-influencersclub.influencersClub | Fetch full lookalike creator profile | Loop Over Items (Lookalike) | Normalize Profile (Lookalike) | ## Enrich Lookalike<br>Fetches full profile per lookalike: post data, audience, income, connected platforms.<br>Error output suppressed — failed enrichments skip silently so the loop continues. |
| Normalize Profile (Lookalike) | n8n-nodes-base.code | Convert lookalike data into standardized nested schema | Enrich by Handle (Full) | AI Agent (Lookalike) | ## Enrich Lookalike<br>Fetches full profile per lookalike: post data, audience, income, connected platforms.<br>Error output suppressed — failed enrichments skip silently so the loop continues. |
| AI Agent (Lookalike) | @n8n/n8n-nodes-langchain.agent | Produce structured creator evaluation | Normalize Profile (Lookalike); OpenAI Model (Lookalike)1; Structured Output Parser (Lookalike)1 | HubSpot — Lookalike Creator; Loop Over Items (Lookalike) | ## AI Agent<br>GPT-4o-mini analyzes the profile and returns structured JSON: summary, strengths, concerns, audience authenticity, growth trajectory, niche clarity, brand fit, recommended action, and estimated reach per post.<br>Edit the system prompt to add campaign context or scoring bias. |
| OpenAI Model (Lookalike)1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide GPT-4o-mini model to agent |  | AI Agent (Lookalike) | ## AI Agent<br>GPT-4o-mini analyzes the profile and returns structured JSON: summary, strengths, concerns, audience authenticity, growth trajectory, niche clarity, brand fit, recommended action, and estimated reach per post.<br>Edit the system prompt to add campaign context or scoring bias. |
| Structured Output Parser (Lookalike)1 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema on AI output |  | AI Agent (Lookalike) | ## AI Agent<br>GPT-4o-mini analyzes the profile and returns structured JSON: summary, strengths, concerns, audience authenticity, growth trajectory, niche clarity, brand fit, recommended action, and estimated reach per post.<br>Edit the system prompt to add campaign context or scoring bias. |
| HubSpot — Lookalike Creator | n8n-nodes-base.hubspot | Create or update lookalike in HubSpot | AI Agent (Lookalike) |  | ## HubSpot Upsert<br>Creates or updates the lookalike as a HubSpot contact using their enriched email.<br>Map `recommended_action`, `brand_fit`, and `estimated_reach_per_post` to custom HubSpot properties for outreach sequencing. |
| 📋 OVERVIEW | n8n-nodes-base.stickyNote | Workflow documentation note |  |  | # HubSpot → Lookalike Creator Discovery<br>**How it works** When a contact is created in HubSpot, this pipeline checks if they're a creator, finds similar creators via the influencers.club API, enriches and AI-analyzes each one, then pushes qualified lookalikes back into HubSpot.<br>**Flow:** Trigger → Fetch contact → Extract email → Enrich → Detect platform → Filter creators → Lookalike API → Loop each result → Enrich → Normalize → AI analysis → HubSpot upsert<br>**Platform detection:** `main_platform` is auto-detected by comparing follower counts across TikTok / Instagram / YouTube / Twitter — the highest audience wins. Non-creators are silently dropped at the Filter node.<br>**Set up required**<br>- HubSpot Developer App (trigger)<br>- HubSpot App Token (read + write)<br>- influencers.club API key<br>- OpenAI API key |
| Note: Trigger | n8n-nodes-base.stickyNote | Documentation note |  |  | ## HubSpot Trigger<br>Fires on `contact.creation`. Outputs `contactId` for the next node.<br>Add `contact.propertyChange` in HubSpot app settings to also catch updates. |
| Note: Get Contact | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Get Contact by ID<br>Fetches `firstname`, `company`, `hs_full_name_or_email` from HubSpot using `contactId`.<br>Add more properties to `propertiesCollection` if your CRM has custom creator fields. |
| Note: Extract Email | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Extract Email<br>Maps HubSpot fields into clean `Email` and `Company Name` variables. |
| Note: Enrich by Email | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Enrich by Email<br>POSTs email to influencers.club and returns a full main profile info.<br>Error → Stop and Error node. |
| Note: Stop and Error | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Stop and Error<br>Triggered when enrichment fails or returns no creator. Halts execution with a logged error message. |
| Note: Normalize Seed | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Normalize (Code in JS)<br>Detects `main_platform` by comparing follower counts across all platform blocks. Computes `follower_tier`, `engagement_tier`, `growth_trend`. Strips CloudFront URL signing params.<br>Outputs a flat clean object for the filter and lookalike nodes. |
| Note: Filter | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Filter — Is Creator?<br>Checks `is_creator === true`. Only real creators proceed.<br>**Port 0** → Lookalike API<br>**Port 1** → dead end, dropped silently |
| Note: Lookalike API | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Lookalike API<br>Finds similar creators using `main_platform` + `handle`. Returns up to 5 accounts.<br>Adjust `paging.limit` (max ~50) or add filters: `engagement_percent.min`, `number_of_followers`, `has_done_brand_deals: true`. |
| Note: Loop | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Loop Over Items<br>Processes one lookalike at a time to avoid API rate limits.<br>**Port 0** → all done, exits loop<br>**Port 1** → next item → Enrichment API |
| Note: Enrich Lookalike1 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Enrich Lookalike<br>Fetches full profile per lookalike: post data, audience, income, connected platforms.<br>Error output suppressed — failed enrichments skip silently so the loop continues. |
| Note: AI Agent | n8n-nodes-base.stickyNote | Documentation note |  |  | ## AI Agent<br>GPT-4o-mini analyzes the profile and returns structured JSON: summary, strengths, concerns, audience authenticity, growth trajectory, niche clarity, brand fit, recommended action, and estimated reach per post.<br>Edit the system prompt to add campaign context or scoring bias. |
| Note: HubSpot Out | n8n-nodes-base.stickyNote | Documentation note |  |  | ## HubSpot Upsert<br>Creates or updates the lookalike as a HubSpot contact using their enriched email.<br>Map `recommended_action`, `brand_fit`, and `estimated_reach_per_post` to custom HubSpot properties for outreach sequencing. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note |  |  | ## Find similar creators to your top performers and boost your brand visibility<br>**Step by step workflow to discover lookalike creators with multi social (Instagram, Tiktok, Youtube, Twitter, Onlyfans, Twitch and more) data using the influencer.club API and add them to Hubspot**.<br>https://influencers.club/creatorbook/find-similar-creators-to-top-performers/ |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   **Find and add similar creators to HubSpot based on top performers**.

2. **Add a HubSpot Trigger node**.
   - Node type: **HubSpot Trigger**
   - Credentials: **HubSpot Developer API**
   - Configure it for the contact creation event.
   - Optionally add contact property change events if you also want updates.
   - Set `maxConcurrentRequests` to `5`.

3. **Add a HubSpot node named “Get Contact by ID”**.
   - Node type: **HubSpot**
   - Operation: **Get Contact**
   - Authentication: **App Token**
   - Credentials: HubSpot app token with contact read access
   - Set Contact ID to:
     `{{ $json.contactId }}`
   - Request the needed properties. At minimum, include:
     - `email`
     - `firstname`
     - `company`
     - `hs_full_name_or_email`
   - If you depend on associated company information, also ensure your retrieval approach actually returns associated company data, or add an additional HubSpot association/company node.

4. **Connect HubSpot Trigger → Get Contact by ID**.

5. **Add a Set node named “Extract Email”**.
   - Create fields:
     - `Email` = `{{ $json.properties.email.value }}`
     - `Company Name` = if available from the payload, otherwise map from a valid company source
   - Important: the provided workflow assumes `associated-company.properties.name.value`, which may not exist unless explicitly fetched elsewhere. If needed, replace with a safer expression.

6. **Connect Get Contact by ID → Extract Email**.

7. **Add an influencers.club node named “Enrich by Email”**.
   - Node type: **influencersClub**
   - Credentials: influencers.club API key
   - Operation: enrich by email
   - Set email to:
     `{{ $json.Email }}`
   - Enable **Continue On Fail / Error Output** behavior so failed enrichments can be routed.

8. **Connect Extract Email → Enrich by Email**.

9. **Add a Stop and Error node**.
   - Node type: **Stop and Error**
   - Error message:
     `API enrichment failed - contact is not a creator or API error`

10. **Connect the error output of Enrich by Email → Stop and Error**.

11. **Add a Code node named “Normalize”**.
   - Paste the seed normalization JavaScript from the workflow.
   - This code:
     - unwraps the enrichment response
     - compares follower counts across TikTok, Instagram, YouTube, and Twitter
     - sets `main_platform`
     - computes formatted follower count, tiers, growth trend
     - creates a flattened creator object

12. **Connect Enrich by Email main output → Normalize**.

13. **Add an IF node named “Filter - Is Creator?”**.
   - Condition:
     - left value: `{{ $json.is_creator }}`
     - operation: boolean equals
     - right value: `true`

14. **Connect Normalize → Filter - Is Creator?**.

15. **Leave the false output unconnected** if you want silent dropping of non-creators.

16. **Add an influencers.club node named “Find Similar Creators”**.
   - Resource: `creator`
   - Operation: `findLookalikes`
   - Platform:
     `{{ $json.main_platform }}`
   - Filter key: `username`
   - Filter value:
     `{{ $json.handle }}`
   - Credentials: influencers.club API key
   - Optionally add advanced filters such as:
     - follower thresholds
     - engagement minimums
     - brand deal requirement
     - paging limit

17. **Connect the true branch of Filter - Is Creator? → Find Similar Creators**.

18. **Add a Split Out node named “Split Out”**.
   - Field to split: `accounts`

19. **Connect Find Similar Creators → Split Out**.

20. **Add a Split In Batches node named “Loop Over Items (Lookalike)”**.
   - Use default options for sequential processing.

21. **Connect Split Out → Loop Over Items (Lookalike)**.

22. **Add another influencers.club node named “Enrich by Handle (Full)”**.
   - Resource: `creator`
   - Operation: `enrichByHandle`
   - Handle:
     `{{ $json.profile.username }}`
   - Platform:
     `{{ $('Normalize').item.json.main_platform }}`
   - Enable continue-on-error behavior so one failed lookalike does not stop the whole run.

23. **Connect output 1 of Loop Over Items (Lookalike) → Enrich by Handle (Full)**.
   - In this pattern, output 1 is used for each batch item.

24. **Add a Code node named “Normalize Profile (Lookalike)”**.
   - Set mode to **Run Once for Each Item**.
   - Paste the provided normalization script.
   - This code standardizes multiple possible influencers.club response formats into `normalized_profile`.

25. **Connect Enrich by Handle (Full) → Normalize Profile (Lookalike)**.

26. **Add an OpenAI Chat Model node named “OpenAI Model (Lookalike)1”**.
   - Node type: LangChain OpenAI Chat Model
   - Credentials: OpenAI API key
   - Model: `gpt-4o-mini`

27. **Add a Structured Output Parser node named “Structured Output Parser (Lookalike)1”**.
   - Use manual schema mode.
   - Paste the full JSON schema from the workflow.
   - Ensure all required fields and enums match exactly.

28. **Add an AI Agent node named “AI Agent (Lookalike)”**.
   - Prompt type: define manually
   - Main prompt should inject:
     `{{ JSON.stringify($json.normalized_profile, null, 2) }}`
   - Add the system message instructing the model to return valid JSON only.
   - Enable output parser usage.

29. **Connect Normalize Profile (Lookalike) → AI Agent (Lookalike)**.

30. **Connect OpenAI Model (Lookalike)1 to the AI Agent’s language model input**.

31. **Connect Structured Output Parser (Lookalike)1 to the AI Agent’s output parser input**.

32. **Add a HubSpot node named “HubSpot — Lookalike Creator”**.
   - Authentication: HubSpot app token
   - Configure it as create or update contact, depending on your CRM design.
   - Recommended matching key: email, if enrichment returns one.
   - Map fields such as:
     - email
     - firstname/full name
     - social handle
     - platform
     - follower count
     - recommended action
     - summary
     - brand fit
     - estimated reach per post
   - Create corresponding custom HubSpot properties beforehand.

33. **Connect AI Agent (Lookalike) → HubSpot — Lookalike Creator**.

34. **Connect AI Agent (Lookalike) back to Loop Over Items (Lookalike)**.
   - This closes the loop so the next lookalike item is processed.

35. **Add sticky notes** for documentation if desired, mirroring:
   - workflow overview
   - trigger notes
   - seed enrichment notes
   - creator filtering
   - lookalike discovery
   - sequential processing
   - AI analysis
   - HubSpot output mapping

36. **Validate the workflow carefully before activation**.
   - Confirm `email` is really available from HubSpot.
   - Confirm `associated-company` references are valid or replaced.
   - Confirm the final HubSpot node has a real email mapping instead of an empty expression.
   - Confirm the AI output still contains the fields you need, or add a Merge node if you must preserve both normalized profile data and AI analysis together.

37. **Recommended hardening adjustments before production**:
   - Add a Merge node after AI analysis if you need both source profile data and AI assessment in HubSpot.
   - Add deduplication before HubSpot create/update using `dedup_key`.
   - Add an IF node to skip lookalikes with no email if HubSpot requires email-based upsert.
   - Add retry logic or throttling for API calls if you expect volume.

### Required credentials
- **HubSpot Developer API** for the trigger
- **HubSpot App Token** for contact get/create/update
- **influencers.club API credential**
- **OpenAI API credential**

### Input/output expectations
- **Trigger input:** HubSpot contact creation event
- **Seed enrichment input:** email address from HubSpot contact
- **Lookalike discovery input:** normalized `main_platform` and `handle`
- **AI input:** `normalized_profile`
- **HubSpot output input:** ideally merged enriched profile + AI analysis + valid email

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Find similar creators to your top performers and boost your brand visibility | influencers.club article |
| Step by step workflow to discover lookalike creators with multi social (Instagram, Tiktok, Youtube, Twitter, Onlyfans, Twitch and more) data using the influencer.club API and add them to Hubspot | https://influencers.club/creatorbook/find-similar-creators-to-top-performers/ |
| Required setup includes HubSpot Developer App, HubSpot App Token, influencers.club API key, and OpenAI API key | Workflow overview note |
| Platform detection is automatic based on the highest follower count among TikTok, Instagram, YouTube, and Twitter | Workflow overview note |
| Non-creators are dropped silently at the filter step | Workflow overview note |

### Important implementation note
The final HubSpot node appears incomplete in the provided workflow JSON. To make the workflow operational, you will likely need to finish the HubSpot create/update configuration and ensure the node receives both a valid contact identifier (usually email) and the enriched/AI-analyzed fields you want to store.