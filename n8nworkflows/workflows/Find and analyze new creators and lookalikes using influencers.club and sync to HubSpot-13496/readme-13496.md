Find and analyze new creators and lookalikes using influencers.club and sync to HubSpot

https://n8nworkflows.xyz/workflows/find-and-analyze-new-creators-and-lookalikes-using-influencers-club-and-sync-to-hubspot-13496


# Find and analyze new creators and lookalikes using influencers.club and sync to HubSpot

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow discovers new Instagram creators via the influencers.club **Discovery API**, enriches them with full creator/audience/monetization data, runs a structured **AI analysis** on each, and then **creates/updates HubSpot contacts**. After analyzing the discovery batch, it selects the **top creator** (highest estimated reach per post) and uses that handle as a seed to find **lookalike creators**, which then go through the same enrichment → normalization → AI scoring → HubSpot sync pipeline.

**Target use cases:**
- Automating creator prospecting for influencer marketing
- Building a creator CRM in HubSpot with enriched + AI-scored fields
- Finding “more creators like this” automatically from the strongest discovery result

### 1.1 Scheduling / Entry
A schedule trigger launches the run.

### 1.2 Branch A — Discovery → Enrich → Normalize → AI Score → HubSpot
Fetch discovery results, split to one creator per item, enrich each, normalize to a consistent schema, analyze with an AI Agent (structured JSON), and upsert to HubSpot.

### 1.3 Post-Discovery Aggregation → Pick Top Seed
After all discovery creators are processed, aggregate results and pick the creator with highest `estimated_reach_per_post.high`.

### 1.4 Branch B — Lookalike → Enrich → Normalize → AI Score → HubSpot
Call lookalike API using the top seed handle, then process lookalike creators identically and sync them to HubSpot.

---

## 2. Block-by-Block Analysis

### Block 1 — Documentation / On-canvas notes
**Overview:** Sticky notes describe the pipeline and required credentials. They don’t affect execution.

**Nodes involved:**
- Pipeline Overview
- Note: Schedule
- Note: Discovery
- Note: Enrich Discovery
- Note: Normalize Discovery
- Note: Loop Discovery
- Note: Agent Discovery
- Note: HubSpot Discovery
- Note: Aggregate
- Note: Pick Top
- Note: Lookalike
- Note: Enrich Lookalike
- Note: Loop Lookalike
- Note: Agent Lookalike
- Note: HubSpot Lookalike
- Sticky Note1

**Node details (applies to all sticky notes):**
- **Type/role:** `stickyNote` nodes are purely explanatory UI elements.
- **Connections:** none.
- **Edge cases:** none (non-executing).

---

### Block 2 — Trigger & Discovery API call (Branch A start)
**Overview:** Runs on a schedule, queries influencers.club Discovery API for Instagram creators matching filters, returns an array under `accounts`.

**Nodes involved:**
- Schedule Trigger1
- Discovery API Call1

#### Node: Schedule Trigger1
- **Type/role:** `scheduleTrigger` — workflow entry point.
- **Configuration (interpreted):** Interval rule is present but not specifically defined in the exported JSON (`interval: [{}]`). In n8n UI this typically means you must set an actual cadence (e.g., every day / every hour).
- **Output:** Emits a single item at each scheduled execution.
- **Edge cases:** Misconfigured schedule (never fires) or timezone expectations.

#### Node: Discovery API Call1
- **Type/role:** `httpRequest` — POST to influencers.club Discovery endpoint.
- **Endpoint:** `POST https://api-dashboard.influencers.club/public/v1/discovery/`
- **Authentication:** Header-based credential (`httpHeaderAuth`) named **Lookalike** (despite name, used here for Discovery).
- **Headers:** `Accept: application/json`
- **Body:** Static JSON with filters, key field:
  - `filters.ai_search = "sport"` (main targeting control)
  - `paging.limit = 5`
  - Many filters set to empty strings/arrays or null (effectively “no constraint”).
- **Connections:**
  - Input: Schedule Trigger1
  - Output: Split Out Discovery
- **Failure modes / edge cases:**
  - 401/403 if API key header missing/invalid
  - 429 if rate-limited / credits exhausted
  - Response shape mismatch (missing `accounts`) breaks downstream split
  - `continueOnFail: true` means the node can output failure items; downstream nodes may then see unexpected structures.

---

### Block 3 — Split discovery results → Enrich each creator
**Overview:** Converts `accounts[]` array into individual items, then enriches each creator with full data.

**Nodes involved:**
- Split Out Discovery
- Enrichment API (Discovery)

#### Node: Split Out Discovery
- **Type/role:** `splitOut` — splits array field into separate items.
- **Config:** `fieldToSplitOut = "accounts"`
- **Input:** From Discovery API Call1 response.
- **Output:** One item per creator account object.
- **Edge cases:**
  - If `accounts` missing or not an array, produces 0 items (or error depending on n8n behavior/version).

#### Node: Enrichment API (Discovery)
- **Type/role:** `httpRequest` — POST to enrichment endpoint per creator.
- **Endpoint:** `POST https://api-dashboard.influencers.club/public/v1/enrichment/creators/enrich/`
- **Authentication:** Header auth credential **Enrichment API**
- **Headers:** `Content-Type: application/json`
- **Body (expression-based):**
  - `media_url`: `https://www.instagram.com/{{ $json.profile.username }}`
  - `platform`: `instagram`
  - Includes post, connected platforms, audience, income data
  - `email_required: "preferred"`
- **Error handling:** `onError = continueErrorOutput` (emits error output for failed requests)
- **Connections:**
  - Input: Split Out Discovery
  - Output: Normalize Profile (Discovery)
- **Edge cases / failure types:**
  - `$json.profile.username` may be missing depending on Discovery response shape (sometimes it might be `$json.username`), producing invalid URLs.
  - API may return partial data when email is not available even with “preferred”.
  - 429 / credit issues.
  - Error output items will reach normalization with different structure unless handled.

---

### Block 4 — Normalize enriched profile (Discovery)
**Overview:** Converts raw enrichment payload into a single `normalized_profile` object with consistent fields used by AI and HubSpot.

**Nodes involved:**
- Normalize Profile (Discovery)

#### Node: Normalize Profile (Discovery)
- **Type/role:** `code` — per-item transformation.
- **Mode:** `runOnceForEachItem`
- **Key transformations:**
  - Handle normalization: removes `@`, lowercases.
  - Strips query params from profile images (CloudFront-style).
  - Builds cross-platform links from `creator_has` flags or mappings.
  - Computes `comment_to_like_ratio_pct`.
  - Determines follower growth trend (`accelerating` vs `decelerating`) from historical percentages.
  - Deduplicates collaborators from tagged data (top 8).
  - Builds `dedup_key` preferring `instagram:<handle>` else `instagram:<user_id>`.
  - Flags:
    - `low_engagement` if engagement% < 1.0 and followers > 10k
    - `tiktok_id_only` if TikTok link looks like numeric-only path
- **Output structure:** `{$json: { normalized_profile: {...} } }`
- **Connections:**
  - Input: Enrichment API (Discovery)
  - Output: Loop Over Items (Discovery)
- **Edge cases:**
  - If enrichment returns an error payload, the code may not find expected fields; most accesses are guarded with `?.` and fall back to null, but the end structure still exists.
  - Email may be null → later HubSpot upsert by email can fail or create unusable records.

---

### Block 5 — Loop + AI analysis + HubSpot sync (Discovery)
**Overview:** Processes creators one-by-one using a batch loop. Each creator is analyzed by an AI Agent producing structured JSON, then the creator is upserted into HubSpot. When loop completes, results are aggregated.

**Nodes involved:**
- Loop Over Items (Discovery)
- AI Agent (Discovery)
- OpenAI Model (Discovery)
- Structured Output Parser (Discovery)
- HubSpot — Discovery Creator
- Aggregate Discovery Results

#### Node: Loop Over Items (Discovery)
- **Type/role:** `splitInBatches` — iteration controller.
- **Configuration:** batch options empty (defaults; typically batch size defaults to 1 unless changed in UI).
- **Connections:**
  - Input: Normalize Profile (Discovery)
  - Output port 0 (“done”): Aggregate Discovery Results
  - Output port 1 (“next item”): AI Agent (Discovery)
- **Edge cases:**
  - If batch size > 1, AI/HB nodes receive multiple items per pass; the workflow appears designed for per-creator processing.
  - Very large result sets could cause long runtimes.

#### Node: AI Agent (Discovery)
- **Type/role:** LangChain `agent` — LLM-based analysis with enforced JSON schema.
- **Prompt:** Injects `JSON.stringify($json.normalized_profile, null, 2)` and demands output in a specific JSON object including:
  - summary, strengths, concerns, audience/growth/content analyses, brand_fit, flags
  - `recommended_action` and reason
  - `estimated_reach_per_post {low, high, basis}`
  - `handle`
- **System message:** Forces “valid JSON only”, no markdown.
- **Output parser:** Uses Structured Output Parser (Discovery) via `ai_outputParser` connection.
- **Language model:** OpenAI Model (Discovery) via `ai_languageModel`.
- **Connections (main):**
  - Outputs to:
    - Loop Over Items (Discovery) (to continue loop)
    - HubSpot — Discovery Creator (parallel sync)
- **Edge cases:**
  - Model may still produce invalid JSON; parser will throw.
  - Missing `normalized_profile` can reduce analysis quality.
  - Token size: extremely large enriched payloads could exceed context limits, but normalization keeps it moderate.

#### Node: OpenAI Model (Discovery)
- **Type/role:** `lmChatOpenAi` — LLM provider.
- **Model:** `gpt-4o-mini`
- **Credential:** “N8N open AI”
- **Connections:** to AI Agent (Discovery) as `ai_languageModel`.
- **Failure modes:** auth errors, rate limits, model deprecation, network timeouts.

#### Node: Structured Output Parser (Discovery)
- **Type/role:** structured parser enforcing a manual JSON Schema.
- **Schema:** Requires `handle` and all major analysis fields; enforces enums for engagement_quality, trajectory, niche_clarity, recommended_action.
- **Connections:** to AI Agent (Discovery) as `ai_outputParser`.
- **Failure modes:** any schema mismatch causes node/agent failure.

#### Node: HubSpot — Discovery Creator
- **Type/role:** `hubspot` — contact create/update by email.
- **Authentication:** HubSpot App Token credential “gorgi hubspot”
- **Key field:** `email = {{$json.normalized_profile.identity.email}}`
- **Additional fields:** none configured (so it likely only upserts the contact by email without setting properties).
- **Connections:** Input from AI Agent (Discovery)
- **Edge cases:**
  - If email is null/empty: HubSpot node commonly fails because email is required for upsert by email.
  - Duplicate emails across creators will overwrite/update same contact.

#### Node: Aggregate Discovery Results
- **Type/role:** `aggregate` — collects all items after loop completes.
- **Mode:** `aggregateAllItemData` (stores all item JSON into `data` array on a single output item).
- **Connections:**
  - Input: Loop Over Items (Discovery) port 0 (done)
  - Output: Pick Top Creator (Highest Reach)
- **Edge cases:**
  - If no creators processed, `data` may be empty; next node has fallback logic.

---

### Block 6 — Pick top creator as lookalike seed
**Overview:** Selects the discovery creator with the highest `estimated_reach_per_post.high` and outputs `lookalike_seed_handle`.

**Nodes involved:**
- Pick Top Creator (Highest Reach)

#### Node: Pick Top Creator (Highest Reach)
- **Type/role:** `code` — reduces aggregated array to one seed handle.
- **Logic:**
  - Reads `items[0].json.data` (the aggregated array)
  - For each element: `reach = item?.output?.estimated_reach_per_post?.high`
  - Picks max reach with a valid `item?.output?.handle`
  - Fallback: first creator’s handle or `'unknown'`
- **Output:**  
  - `lookalike_seed_handle`
  - `seed_reach_high`
- **Connections:** Output → Lookalike API Call1
- **Edge cases:**
  - This code assumes AI Agent output is under `item.output`. Depending on LangChain node output shape/version, results may be under a different key (e.g., `json` directly). If mismatched, it will always fall back.
  - If handle is missing, can become `'unknown'`, leading to poor lookalike results or API error.

---

### Block 7 — Lookalike API → Enrich → Normalize → AI → HubSpot (Branch B)
**Overview:** Finds similar creators to the chosen seed, enriches and normalizes each, runs the same AI scoring, and upserts to HubSpot.

**Nodes involved:**
- Lookalike API Call1
- Split Out Lookalike
- Enrichment API (Lookalike)
- Normalize Profile (Lookalike)
- Loop Over Items (Lookalike)
- AI Agent (Lookalike)
- OpenAI Model (Lookalike)
- Structured Output Parser (Lookalike)
- HubSpot — Lookalike Creator

#### Node: Lookalike API Call1
- **Type/role:** `httpRequest` — POST to similar creators endpoint.
- **Endpoint:** `POST https://api-dashboard.influencers.club/public/v1/discovery/creators/similar/`
- **Authentication:** Header auth credential **Enrichment API** (note: naming suggests enrichment, but used here for lookalike too).
- **Body (expression-based):**
  - `filter_key = "username"`
  - `filter_value = "{{ $json.lookalike_seed_handle }}"`
  - Filters include `location: ["United States"]`, `profile_language: ["en"]`, `time_range_months: 3`, `paging.limit: 5`
- **Connections:** Input from Pick Top Creator → Output to Split Out Lookalike
- **Failure modes:**
  - Invalid seed handle (unknown/private) → empty results
  - Auth/credits/rate limits
  - `continueOnFail: true` can pass error-shaped items forward

#### Node: Split Out Lookalike
- **Type/role:** `splitOut`
- **Config:** `fieldToSplitOut = "accounts"`
- **Connections:** Lookalike API Call1 → Enrichment API (Lookalike)
- **Edge cases:** same as discovery split (missing/invalid `accounts`)

#### Node: Enrichment API (Lookalike)
- **Type/role:** `httpRequest` — same endpoint and body pattern as discovery enrichment.
- **Body dependency:** `media_url` uses `{{ $json.profile.username }}`
- **Error handling:** `onError=continueErrorOutput`
- **Connections:** → Normalize Profile (Lookalike)
- **Edge cases:** same as discovery enrichment

#### Node: Normalize Profile (Lookalike)
- **Type/role:** `code`
- **Logic:** Essentially identical to Normalize Profile (Discovery), producing `normalized_profile`.
- **Connections:** → Loop Over Items (Lookalike)

#### Node: Loop Over Items (Lookalike)
- **Type/role:** `splitInBatches`
- **Connections:**
  - Input: Normalize Profile (Lookalike)
  - Output port 1 (“next”): AI Agent (Lookalike)
  - Output port 0 (“done”): **not connected** (ends branch)
- **Edge cases:** Without an aggregation/end node, the branch simply completes after processing; that’s fine if HubSpot is the final sink.

#### Node: AI Agent (Lookalike)
- **Type/role:** LangChain `agent`
- **Prompt:** Same structure as discovery but **does not request `handle` field** in the response JSON block (unlike discovery). This is a functional inconsistency: discovery requires `handle`, lookalike parser does not.
- **Connections (main):**
  - Loop Over Items (Lookalike) (continue)
  - HubSpot — Lookalike Creator
- **Failure modes:** same as discovery agent (JSON validity/schema/token limits)

#### Node: OpenAI Model (Lookalike)
- **Type/role:** `lmChatOpenAi`
- **Model:** `gpt-4o-mini`
- **Connections:** to AI Agent (Lookalike) as `ai_languageModel`

#### Node: Structured Output Parser (Lookalike)
- **Type/role:** structured JSON schema validator/parser.
- **Schema:** Similar to discovery but **no `handle` required**.
- **Connections:** to AI Agent (Lookalike) as `ai_outputParser`

#### Node: HubSpot — Lookalike Creator
- **Type/role:** `hubspot` contact upsert by email.
- **Email field:** `{{$json.normalized_profile.identity.email}}`
- **Edge cases:** same as discovery HubSpot node (null email, duplicates, API limits)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Pipeline Overview | stickyNote | On-canvas overview & credential notes |  |  | ## 🚀 Creator Discovery — Dual Branch Pipeline; Credentials to configure listed |
| Note: Schedule | stickyNote | Explains trigger cost considerations |  |  | ⏰ **Schedule Trigger** Fires pipeline; adjust frequency based on credit budget |
| Note: Discovery | stickyNote | Explains discovery API purpose |  |  | 🔍 **Discovery API** Search creators; edit `ai_search` |
| Note: Enrich Discovery | stickyNote | Explains enrichment API |  |  | 🔬 **Enrichment API** Fetches full profile data |
| Note: Normalize Discovery | stickyNote | Explains normalization |  |  | 🧹 **Normalize Profile** Cleans/standardizes fields |
| Note: Loop Discovery | stickyNote | Explains loop ports |  |  | 🔁 **Loop Over Items** Port 0 done → Aggregate; Port 1 next → AI |
| Note: Agent Discovery | stickyNote | Explains AI analysis |  |  | 🤖 **AI Agent (Discovery)** Structured scoring |
| Note: HubSpot Discovery | stickyNote | Explains HubSpot upsert |  |  | 🟠 **HubSpot (Discovery)** Upsert contact using email |
| Note: Aggregate | stickyNote | Explains aggregation for comparison |  |  | 📦 **Aggregate** Collects all scored discovery creators |
| Note: Pick Top | stickyNote | Explains seed selection |  |  | 🏆 **Pick Top Creator** Chooses highest reach seed |
| Note: Lookalike | stickyNote | Explains lookalike search |  |  | 👯 **Lookalike API** Finds similar creators |
| Note: Enrich Lookalike | stickyNote | Explains enrichment in branch B |  |  | 🔬 **Enrichment API (Lookalike)** Same enrichment call |
| Note: Loop Lookalike | stickyNote | Explains lookalike loop |  |  | 🔁 **Loop (Lookalike)** Port 0 done; Port 1 next |
| Note: Agent Lookalike | stickyNote | Explains AI scoring in branch B |  |  | 🤖 **AI Agent (Lookalike)** Identical analysis intent |
| Note: HubSpot Lookalike | stickyNote | Explains HubSpot upsert in branch B |  |  | 🟠 **HubSpot (Lookalike)** Same node config; converge to CRM |
| Split Out Discovery | splitOut | Split discovery `accounts[]` into items | Discovery API Call1 | Enrichment API (Discovery) | 🔍 **Discovery API** Searches… |
| Enrichment API (Discovery) | httpRequest | Enrich each discovery creator | Split Out Discovery | Normalize Profile (Discovery) | 🔬 **Enrichment API** Fetches full profile… |
| Normalize Profile (Discovery) | code | Normalize enriched payload to `normalized_profile` | Enrichment API (Discovery) | Loop Over Items (Discovery) | 🧹 **Normalize Profile** Standardizes raw API data… |
| Loop Over Items (Discovery) | splitInBatches | Iterate creators; route done vs next | Normalize Profile (Discovery); AI Agent (Discovery) | (0) Aggregate Discovery Results; (1) AI Agent (Discovery) | 🔁 **Loop Over Items** Port 0 done → Aggregate; Port 1 next → AI |
| AI Agent (Discovery) | langchain agent | LLM analysis → structured JSON | Loop Over Items (Discovery) | Loop Over Items (Discovery); HubSpot — Discovery Creator | 🤖 **AI Agent (Discovery)** Analyzes each creator… |
| OpenAI Model (Discovery) | OpenAI chat model | Provides LLM to agent |  | AI Agent (Discovery) (ai_languageModel) | 🤖 **AI Agent (Discovery)** Analyzes each creator… |
| Structured Output Parser (Discovery) | structured output parser | Enforces JSON schema for agent output |  | AI Agent (Discovery) (ai_outputParser) | 🤖 **AI Agent (Discovery)** Analyzes each creator… |
| HubSpot — Discovery Creator | hubspot | Upsert contact by email | AI Agent (Discovery) |  | 🟠 **HubSpot (Discovery)** Creates/updates contact… |
| Aggregate Discovery Results | aggregate | Aggregate all discovery processed items | Loop Over Items (Discovery) (done) | Pick Top Creator (Highest Reach) | 📦 **Aggregate** Collects ALL scored… |
| Pick Top Creator (Highest Reach) | code | Choose seed handle by highest reach | Aggregate Discovery Results | Lookalike API Call1 | 🏆 **Pick Top Creator** Finds highest reach… |
| Lookalike API Call1 | httpRequest | Fetch similar creators to seed | Pick Top Creator (Highest Reach) | Split Out Lookalike | 👯 **Lookalike API** Finds creators similar… |
| Split Out Lookalike | splitOut | Split lookalike `accounts[]` | Lookalike API Call1 | Enrichment API (Lookalike) | 👯 **Lookalike API** Finds creators similar… |
| Enrichment API (Lookalike) | httpRequest | Enrich each lookalike creator | Split Out Lookalike | Normalize Profile (Lookalike) | 🔬 **Enrichment API (Lookalike)** Same enrichment call… |
| Normalize Profile (Lookalike) | code | Normalize lookalike enrichment payload | Enrichment API (Lookalike) | Loop Over Items (Lookalike) | (same intent as normalize note) 🧹 **Normalize Profile** Standardizes… |
| Loop Over Items (Lookalike) | splitInBatches | Iterate lookalikes | Normalize Profile (Lookalike); AI Agent (Lookalike) | (1) AI Agent (Lookalike) | 🔁 **Loop (Lookalike)** Processes each individually… |
| AI Agent (Lookalike) | langchain agent | LLM analysis → structured JSON | Loop Over Items (Lookalike) | Loop Over Items (Lookalike); HubSpot — Lookalike Creator | 🤖 **AI Agent (Lookalike)** Scores each lookalike… |
| OpenAI Model (Lookalike) | OpenAI chat model | Provides LLM to agent |  | AI Agent (Lookalike) (ai_languageModel) | 🤖 **AI Agent (Lookalike)** Scores each lookalike… |
| Structured Output Parser (Lookalike) | structured output parser | Enforces JSON schema for lookalike output |  | AI Agent (Lookalike) (ai_outputParser) | 🤖 **AI Agent (Lookalike)** Scores each lookalike… |
| HubSpot — Lookalike Creator | hubspot | Upsert contact by email | AI Agent (Lookalike) |  | 🟠 **HubSpot (Lookalike)** Creates/updates contact… |
| Schedule Trigger1 | scheduleTrigger | Workflow entry point |  | Discovery API Call1 | ⏰ **Schedule Trigger** Fires pipeline… |
| Discovery API Call1 | httpRequest | Discovery search | Schedule Trigger1 | Split Out Discovery | 🔍 **Discovery API** Searches… |
| Sticky Note1 | stickyNote | High-level description + link |  |  | ## Find new creators and lookalikes and add them to your CRM — [Full explanation](https://influencers.club/creatorbook/find-creators-for-influencer-marketing-platform/) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name it: *Find new creators and lookalikes using influencers.club and add them to Hubspot* (or your preferred name).

2) **Add Schedule Trigger**
- Node: **Schedule Trigger**
- Configure the interval (e.g., daily, hourly). Ensure it’s not left blank.

3) **Add Discovery API request**
- Node: **HTTP Request**
- Method: **POST**
- URL: `https://api-dashboard.influencers.club/public/v1/discovery/`
- Authentication: **Header Auth** credential (create one; name can be “Lookalike” as in the workflow)
  - Add the required API key header per influencers.club docs (commonly `Authorization: Bearer <token>` or an API-key header—match your account requirements).
- Headers: `Accept: application/json`
- Body Content Type: **JSON**
- Body: include:
  - `platform: "instagram"`
  - `paging.limit` (e.g., 5)
  - `filters.ai_search` (e.g., `"sport"`)
  - Other filters as needed
- Connect: **Schedule Trigger → Discovery API**

4) **Split discovery accounts**
- Node: **Split Out**
- Field to split out: `accounts`
- Connect: **Discovery API → Split Out Discovery**

5) **Enrich each discovery creator**
- Node: **HTTP Request**
- Method: POST
- URL: `https://api-dashboard.influencers.club/public/v1/enrichment/creators/enrich/`
- Auth: Header Auth credential (create; name “Enrichment API”)
- Headers: `Content-Type: application/json`
- Body (JSON with expression for username):
  - `media_url = https://www.instagram.com/{{ $json.profile.username }}`
  - `platform = instagram`
  - `include_post_data/include_connected_platforms/include_audience_data/include_income_data = true`
  - `email_required = "preferred"`
- Error handling: set **On Error → Continue (output error)** (or equivalent).
- Connect: **Split Out Discovery → Enrichment API (Discovery)**

6) **Normalize discovery profiles**
- Node: **Code**
- Mode: **Run once for each item**
- Paste normalization logic equivalent to the workflow (handle cleanup, cross-platform links, metrics, flags, `dedup_key`) and output `normalized_profile`.
- Connect: **Enrichment API (Discovery) → Normalize Profile (Discovery)**

7) **Loop over discovery items**
- Node: **Split in Batches**
- Batch size: **1** (recommended for predictable per-creator AI/HubSpot processing)
- Connect: **Normalize Profile (Discovery) → Loop Over Items (Discovery)**

8) **Add AI components (Discovery)**
- Node: **OpenAI Chat Model** (LangChain)
  - Model: `gpt-4o-mini`
  - Credential: configure **OpenAI API** credential in n8n.
- Node: **Structured Output Parser**
  - Schema type: Manual
  - Paste the JSON schema requiring `handle`, analysis fields, enums, and `estimated_reach_per_post`.
- Node: **AI Agent**
  - Prompt type: Define
  - System message: “expert influencer marketing analyst… JSON only…”
  - User prompt: inject `{{$json.normalized_profile}}` stringified and request the exact JSON.
  - Connect model to agent via **ai_languageModel**
  - Connect parser to agent via **ai_outputParser**
- Connect loop: **Loop Over Items (Discovery) (port 1) → AI Agent (Discovery)**

9) **HubSpot upsert (Discovery)**
- Node: **HubSpot**
- Authentication: **App Token**
  - Create credential with your HubSpot Private App token.
- Operation: Create/Update contact by email (as supported by the node)
- Email field: `{{$json.normalized_profile.identity.email}}`
- Connect: **AI Agent (Discovery) → HubSpot — Discovery Creator**
- Note: Consider adding additional HubSpot properties mapping (not present in the JSON) if you want to store AI results.

10) **Close the discovery loop**
- Connect: **AI Agent (Discovery) → Loop Over Items (Discovery)** (so it fetches the next batch)

11) **Aggregate discovery results**
- Node: **Aggregate**
- Mode: **Aggregate All Item Data**
- Connect: **Loop Over Items (Discovery) (port 0 done) → Aggregate Discovery Results**

12) **Pick top creator seed**
- Node: **Code**
- Logic: iterate aggregated `data[]` and choose max `estimated_reach_per_post.high`; output `lookalike_seed_handle`.
- Ensure the path matches your AI Agent output structure (you may need to reference where the agent stores parsed JSON).
- Connect: **Aggregate → Pick Top Creator**

13) **Lookalike API call**
- Node: **HTTP Request**
- Method: POST
- URL: `https://api-dashboard.influencers.club/public/v1/discovery/creators/similar/`
- Auth: header credential (in the JSON it uses “Enrichment API”)
- Body (JSON):
  - `filter_key: "username"`
  - `filter_value: {{$json.lookalike_seed_handle}}`
  - `platform: "instagram"`
  - `paging.limit: 5`
  - Filters as desired (location/language/etc.)
- Connect: **Pick Top Creator → Lookalike API Call**

14) **Split lookalike accounts**
- Node: **Split Out**
- Field: `accounts`
- Connect: **Lookalike API → Split Out Lookalike**

15) **Enrich lookalikes**
- Node: **HTTP Request** (same as discovery enrichment)
- Connect: **Split Out Lookalike → Enrichment API (Lookalike)**

16) **Normalize lookalikes**
- Node: **Code** (same normalization code)
- Connect: **Enrichment API (Lookalike) → Normalize Profile (Lookalike)**

17) **Loop lookalikes**
- Node: **Split in Batches** (batch size 1)
- Connect: **Normalize Profile (Lookalike) → Loop Over Items (Lookalike)**

18) **AI analysis (Lookalike)**
- Add **OpenAI Chat Model**, **Structured Output Parser**, **AI Agent** similar to discovery.
- Important: align the schema and prompt (either include `handle` consistently in both branches or omit consistently).
- Connect: **Loop Over Items (Lookalike) (port 1) → AI Agent (Lookalike)**  
- Connect AI Agent back to the loop: **AI Agent (Lookalike) → Loop Over Items (Lookalike)**

19) **HubSpot upsert (Lookalike)**
- Node: **HubSpot** (same credential)
- Email: `{{$json.normalized_profile.identity.email}}`
- Connect: **AI Agent (Lookalike) → HubSpot — Lookalike Creator**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Find new creators and lookalikes and add them to your CRM… Full explanation” | https://influencers.club/creatorbook/find-creators-for-influencer-marketing-platform/ |
| Configure credentials: Discovery API header auth; Enrichment + Lookalike header auth; OpenAI; HubSpot App Token | Mentioned in “Pipeline Overview” sticky note |
| Cost control: schedule frequency impacts enrichment credit usage | Mentioned in “Note: Schedule” sticky note |

