Enrich creator handles with cross-platform social data from influencers.club

https://n8nworkflows.xyz/workflows/enrich-creator-handles-with-cross-platform-social-data-from-influencers-club-13227


# Enrich creator handles with cross-platform social data from influencers.club

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Enrich creator handles with cross-platform social data from influencers.club  
**Workflow name (in JSON):** Enrich creators with social platform data  
**Status:** Inactive (must be activated to run on schedule)

**Purpose:**  
This workflow runs daily to enrich creator/influencer records stored in Supabase (`leads3`) using the influencers.club enrichment API. It targets only creators that have not yet been enriched (where `raw_response` is NULL), normalizes the API response into a consistent schema, then updates the database in a safe, idempotent way (only filling missing fields).

**Target use cases:**
- Influencer marketing platforms maintaining a lead/creator database
- Daily enrichment of social profiles across platforms using one handle (IG/TikTok/YouTube/etc.)
- Backfilling missing creator metadata and analytics without overwriting existing data

### Logical Blocks
**1.1 Scheduling & Run Trigger**  
Daily cron trigger starts the pipeline.

**1.2 Select Unenriched Creators (Supabase)**  
Query Supabase for rows missing enrichment payload (`raw_response IS NULL`).

**1.3 Batch Processing & Throttling Loop**  
Process creators in batches of 10 and insert a 5-second wait between batches to reduce API/DB pressure.

**1.4 Enrichment API Call (influencers.club)**  
POST request using platform + handle to retrieve enrichment data.

**1.5 Normalize Response (Code)**  
Convert varying platform payload shapes into a stable “normalized” structure and compute engagement metrics.

**1.6 Persist Enrichment Back to Supabase (Safe RPC / Optional Direct Update)**  
Preferred path: call a Supabase SQL RPC function to update only NULL fields.  
Alternative (disabled): directly update the row, which may overwrite existing data.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & Run Trigger
**Overview:** Triggers the workflow once per day at midnight (server time) to run the enrichment refresh cycle.

**Nodes involved:**
- **Daily Refresh Schedule** (Schedule Trigger)
- Sticky Note: **Sticky Note1**

#### Node: Daily Refresh Schedule
- **Type / role:** `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`) — entrypoint.
- **Configuration choices:**
  - Cron expression: `0 0 * * *` (daily at 00:00).
- **Inputs / outputs:**
  - **Output →** `List Influencers Without Enrichment`
- **Version:** typeVersion **1.1**
- **Edge cases / failures:**
  - Timezone ambiguity: schedule uses n8n instance timezone; may not match business timezone.
  - If workflow is inactive, it will not run.

**Sticky note content (applies to this block):**
- “Triggers the creator enrichment workflow once per day. The workflow is safe to re-run and will only process creators that have not been enriched yet.”

---

### 2.2 Select Unenriched Creators (Supabase)
**Overview:** Pulls all creator rows where enrichment has not yet been stored (NULL `raw_response`).

**Nodes involved:**
- **List Influencers Without Enrichment** (Supabase)
- Sticky Note: **Sticky Note2**

#### Node: List Influencers Without Enrichment
- **Type / role:** `Supabase` (`n8n-nodes-base.supabase`) — reads from DB table.
- **Configuration choices:**
  - **Operation:** Get All
  - **Table:** `leads3`
  - **Filter:** `raw_response is Null`
    - Implemented via condition `is` with keyValue “Null” (n8n UI representation).
- **Inputs / outputs:**
  - **Input ←** `Daily Refresh Schedule`
  - **Output →** `Process in Batches`
- **Version:** typeVersion **1**
- **Credentials:** `supabaseApi` (“new one”)
- **Edge cases / failures:**
  - Auth/permission errors (Supabase key lacks read access).
  - Large datasets: `getAll` can return many rows; may be slow or exceed memory/time depending on n8n settings.
  - NULL handling depends on Supabase node behavior; ensure the column is truly NULL (not `{}` or empty JSON).

**Sticky note content:**
- “Fetches creators that have not yet been enriched by checking for an empty enrichment payload. Uses the “Is Empty” filter to correctly match NULL jsonb fields.”
  - Note: the actual filter here is `raw_response is Null`. If your DB uses empty `{}` instead of NULL, adjust filtering accordingly.

---

### 2.3 Batch Processing & Throttling Loop
**Overview:** Splits the unenriched creator list into batches of 10 and loops with a delay between batches to control throughput.

**Nodes involved:**
- **Process in Batches** (Split in Batches)
- **Wait 5 Second** (Wait)

#### Node: Process in Batches
- **Type / role:** `Split In Batches` (`n8n-nodes-base.splitInBatches`) — batch iterator/loop controller.
- **Configuration choices:**
  - Batch size: **10**
- **Inputs / outputs:**
  - **Input ←** `List Influencers Without Enrichment`
  - **Output (batch items) →** `Influencers.club Enrichment API By Handle`
  - **Loop continuation input ←** `Wait 5 Second` (feeds back to request next batch)
- **Version:** typeVersion **3**
- **Key expressions / variables:**
  - Upstream items must contain at least: `creator_handle`, `platform`, and ideally `id`.
- **Edge cases / failures:**
  - If `platform` or `creator_handle` are missing/empty, downstream API call may fail or return unusable data.
  - If no items returned from Supabase, the node produces no batches (workflow ends quietly).

#### Node: Wait 5 Second
- **Type / role:** `Wait` (`n8n-nodes-base.wait`) — throttling between batches.
- **Configuration choices:**
  - Default wait configuration (no explicit parameters shown). In many n8n versions this implies a fixed wait when configured in UI; here it appears intended as ~5 seconds per node name.
- **Inputs / outputs:**
  - **Input ←** `Update Null Values Only` (preferred DB update) and also `Update a row` (disabled alternative)
  - **Output →** `Process in Batches` (continue loop)
- **Version:** typeVersion **1.1**
- **Edge cases / failures:**
  - If configured as “Wait for webhook” instead of “Wait amount of time”, execution may stall. Validate Wait node’s actual UI settings.
  - Long runs: workflows with many batches will take time (5 seconds per batch loop).

---

### 2.4 Enrichment API Call (influencers.club)
**Overview:** Calls influencers.club to enrich a creator based on platform + handle, requesting post data, connected platforms, audience data, and income data.

**Nodes involved:**
- **Influencers.club Enrichment API By Handle** (HTTP Request)
- Sticky Note: **Sticky Note3**

#### Node: Influencers.club Enrichment API By Handle
- **Type / role:** `HTTP Request` (`n8n-nodes-base.httpRequest`) — external API call.
- **Configuration choices:**
  - **Method:** POST
  - **URL:** `https://api-dashboard.influencers.club/public/v1/enrichment/creators/enrich/`
  - **Auth:** Generic credential type → HTTP Header Auth (likely `Authorization: Bearer …` or API key header, depending on credential setup)
  - **Body:** JSON (constructed via expression), key fields:
    - `media_url`: `https://www.{{ $json.platform }}.com/{{ $json.creator_handle }}`
    - `platform`: `{{ $json.platform }}`
    - `include_post_data`: true
    - `include_connected_platforms`: true
    - `include_audience_data`: true
    - `include_income_data`: true
    - `email_required`: "preferred"
  - **Headers:** `Content-Type: application/json`
  - **On error:** `continueErrorOutput` (workflow continues even if request fails)
- **Inputs / outputs:**
  - **Input ←** `Process in Batches` (each item is one creator row)
  - **Output →** `Normalize Creator Enrichment Payload`
- **Version:** typeVersion **4.2**
- **Key expressions / variables used:**
  - `$json.platform`, `$json.creator_handle` (must exist in Supabase rows)
- **Edge cases / failures:**
  - Invalid `platform` → malformed `media_url` and/or API rejection.
  - Rate limiting (429), temporary failures (5xx), timeouts.
  - Because errors continue, downstream may receive an error-shaped payload; the normalizer must handle missing keys (it mostly does).
  - If the API returns a payload without any known platform keys, `platform` becomes null in normalization.

**Sticky note content:**
- “Enriches a creator profile using their platform and handle… platform-agnostic… returns structured enrichment data (audience, content, and monetization insights).”

---

### 2.5 Normalize Response (Code)
**Overview:** Normalizes the potentially platform-specific API response into a consistent structure aligned with the Supabase schema, calculates engagement metrics from posts, and adds metadata (timestamp, raw response, credits).

**Nodes involved:**
- **Normalize Creator Enrichment Payload** (Code)

#### Node: Normalize Creator Enrichment Payload
- **Type / role:** `Code` (`n8n-nodes-base.code`) — transforms data per item.
- **Configuration choices:**
  - Mode: `runOnceForEachItem`
  - JavaScript normalizer that:
    - Detects platform key from `apiResponse` among: instagram, twitter, tiktok, youtube, linkedin, facebook, twitch
    - Pulls `platformData = apiResponse[platform]`
    - Retrieves matching batch item via:
      - `$items("Process in Batches", 0)[$itemIndex]?.json`
    - Calculates `avg_likes`, `avg_comments`, and `engagement_rate` from `post_data` when possible
    - Returns a normalized object containing identity, flags, metrics, arrays, and raw response
- **Inputs / outputs:**
  - **Input ←** `Influencers.club Enrichment API By Handle`
  - **Outputs →**
    - `Update Null Values Only` (preferred)
    - `Update a row` (disabled alternative)
- **Version:** typeVersion **2**
- **Key variables / expressions:**
  - `item.json` as API response for current item
  - `$items("Process in Batches", 0)[$itemIndex]` to align enrichment response with original lead row
- **Edge cases / failures:**
  - **Item index misalignment risk:** Using `$itemIndex` assumes a 1:1 mapping and stable ordering between the HTTP node output and the original batch items. This is usually true, but can break if nodes change item counts or if errors produce different output structures.
  - If API returns an error object, `apiResponse[k]` may not exist; platform becomes `null`, and most fields become null/empty arrays. This is safe, but could lead to “empty enrichment” being written unless the update function rejects it.
  - `platformData.follower_count` missing → engagementRate may remain null.
  - Posts schema differences: code supports likes/comments fields (`likes` vs `like_count`, `comments` vs `reply_count`) but may miss other platforms’ naming conventions.

---

### 2.6 Persist Enrichment Back to Supabase (Safe RPC / Optional Direct Update)
**Overview:** Writes normalized enrichment results back to Supabase. Preferred method uses an RPC SQL function to update only NULL fields, preventing overwrites. An alternative direct table update exists but is disabled.

**Nodes involved:**
- **Update Null Values Only** (HTTP Request to Supabase RPC)
- **Update a row** (Supabase Update) — disabled
- Sticky Notes: **Sticky Note4**, **Sticky Note5**, **Sticky Note6**

#### Node: Update Null Values Only
- **Type / role:** `HTTP Request` (`n8n-nodes-base.httpRequest`) — calls Supabase REST RPC endpoint.
- **Configuration choices:**
  - **Method:** POST
  - **URL:** `https://yphzlsruptlrnehidveq.supabase.co/rest/v1/rpc/enrich_lead`
  - **Auth:** Predefined credential type: `supabaseApi` (“new one”)
  - **Headers:** `Content-Type: application/json`
  - **Body parameters:**
    - `p_creator_handle` = `{{ $('Process in Batches').item.json.creator_handle }}`
    - `p_platform` = `{{ $('Process in Batches').item.json.platform }}`
    - `p_payload` = `{{$json}}` (the normalized payload from the Code node)
- **Inputs / outputs:**
  - **Input ←** `Normalize Creator Enrichment Payload`
  - **Output →** `Wait 5 Second` (to continue batch loop)
- **Version:** typeVersion **4.3**
- **Key expressions / variables used:**
  - Pulls creator identity from the batch item via `$('Process in Batches').item.json...`
  - Sends normalized payload as `p_payload`
- **Edge cases / failures:**
  - RPC function missing or misnamed (`enrich_lead`) → 404/400.
  - Supabase RLS policies may block function execution.
  - Payload type mismatch: if function expects `jsonb` and request sends incompatible structure.
  - If `creator_handle` or `platform` are NULL, function may not match any record.
  - If errors from enrichment API produce “empty” normalized payloads, this could still mark rows as enriched depending on SQL logic.

#### Node: Update a row (disabled)
- **Type / role:** `Supabase` (`n8n-nodes-base.supabase`) — direct table update.
- **Status:** **Disabled** (not executed).
- **Configuration choices:**
  - **Operation:** update
  - **Table:** `leads3`
  - **Filter:** `creator_handle eq {{$json.username}}` (note: this is likely incorrect mapping; handle ≠ username in all cases)
  - **Fields:** maps many normalized fields to columns, including `raw_response`, `enriched_at`, etc.
- **Inputs / outputs:**
  - **Input ←** `Normalize Creator Enrichment Payload`
  - **Output →** `Wait 5 Second`
- **Version:** typeVersion **1**
- **Edge cases / failures:**
  - Overwrites existing fields on re-run (explicitly warned).
  - Potential bug: filter uses `creator_handle = $json.username`; if DB stores handle separately, updates may not match or may update wrong row.
  - Schema mismatch if columns differ (e.g., `follower_count` mapping uses `$json.following_count` in the JSON provided—likely a mistake).

**Sticky note content (production-safe approach):**
- “Updates the creator data using a SQL database function that only fills missing fields… Safe to re-run… Prevents overwriting…”

**Sticky note content (alternative method warning):**
- “Alternative update method that writes data directly to the table… may overwrite existing data…”

**Sticky note link (SQL function):**
- “Click and view the full SQL function on GitHub: https://github.com/GjPetrovski-IC/N8N-Public-Templates”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Refresh Schedule | Schedule Trigger | Daily entrypoint trigger | — | List Influencers Without Enrichment | Triggers the creator enrichment workflow once per day. The workflow is safe to re-run and will only process creators that have not been enriched yet. |
| List Influencers Without Enrichment | Supabase | Fetch rows missing enrichment (`raw_response` is NULL) | Daily Refresh Schedule | Process in Batches | Fetches creators that have not yet been enriched by checking for an empty enrichment payload. Uses the “Is Empty” filter to correctly match NULL jsonb fields. |
| Process in Batches | Split In Batches | Batch iteration (size 10) + looping | List Influencers Without Enrichment; Wait 5 Second | Influencers.club Enrichment API By Handle |  |
| Influencers.club Enrichment API By Handle | HTTP Request | Call influencers.club enrichment endpoint | Process in Batches | Normalize Creator Enrichment Payload | Enriches a creator profile using their platform and handle. The request is designed to be platform-agnostic and returns structured enrichment data (audience, content, and monetization insights) using the influencers.club API. |
| Normalize Creator Enrichment Payload | Code | Normalize payload + compute metrics | Influencers.club Enrichment API By Handle | Update Null Values Only; Update a row | Enriches a creator profile using their platform and handle. The request is designed to be platform-agnostic and returns structured enrichment data (audience, content, and monetization insights) using the influencers.club API. |
| Update Null Values Only | HTTP Request | Supabase RPC update (only fill nulls) | Normalize Creator Enrichment Payload | Wait 5 Second | Updates the creator data using a SQL database function that only fills missing fields. Safe to re-run; prevents overwriting; handles partial enrichment responses. Recommended for production use. |
| Update a row | Supabase | Direct table update (alternative; disabled) | Normalize Creator Enrichment Payload | Wait 5 Second | Alternative update method that writes data directly to the table. May overwrite existing data if re-run; recommended only for simple or non-production setups. |
| Wait 5 Second | Wait | Throttle + continue batch loop | Update Null Values Only; Update a row | Process in Batches | Click and view the full SQL function on GitHub: https://github.com/GjPetrovski-IC/N8N-Public-Templates |
| Sticky Note | Sticky Note | Documentation / link | — | — | Get multi social platform data for creators from one social handle… Full explanation: https://influencers.club/creatorbook/cross-platform-data-for-influencer-marketing-platforms/ |
| Sticky Note1 | Sticky Note | Documentation | — | — | Triggers the creator enrichment workflow once per day… |
| Sticky Note2 | Sticky Note | Documentation | — | — | Fetches creators that have not yet been enriched… |
| Sticky Note3 | Sticky Note | Documentation | — | — | Enriches a creator profile using their platform and handle… |
| Sticky Note4 | Sticky Note | Documentation | — | — | Updates the creator data using a SQL database function that only fills missing fields… |
| Sticky Note5 | Sticky Note | Documentation | — | — | Alternative update method… may overwrite existing data… |
| Sticky Note6 | Sticky Note | Documentation / link | — | — | https://github.com/GjPetrovski-IC/N8N-Public-Templates |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: Schedule Trigger**
   - Name: `Daily Refresh Schedule`
   - Set **Cron**: `0 0 * * *`
3. **Add node: Supabase**
   - Name: `List Influencers Without Enrichment`
   - Credentials: create/select **Supabase API** credential (URL + service role key or suitable key with access)
   - Operation: **Get All**
   - Table: `leads3`
   - Filter: `raw_response` **is** `Null`
4. **Add node: Split In Batches**
   - Name: `Process in Batches`
   - Batch size: **10**
5. **Add node: HTTP Request** (influencers.club)
   - Name: `Influencers.club Enrichment API By Handle`
   - Method: **POST**
   - URL: `https://api-dashboard.influencers.club/public/v1/enrichment/creators/enrich/`
   - Authentication: **Header Auth** (create credential containing the required header, typically `Authorization`)
   - Headers: `Content-Type: application/json`
   - Body type: **JSON**
   - Body (expressions):
     - `media_url`: `https://www.{{ $json.platform }}.com/{{ $json.creator_handle }}`
     - `platform`: `{{ $json.platform }}`
     - `include_post_data`: `true`
     - `include_connected_platforms`: `true`
     - `include_audience_data`: `true`
     - `include_income_data`: `true`
     - `email_required`: `"preferred"`
   - Error handling: set **On Error** to “Continue (error output)” to avoid stopping the whole batch run.
6. **Add node: Code**
   - Name: `Normalize Creator Enrichment Payload`
   - Mode: **Run Once for Each Item**
   - Paste the normalizer script (the logic must: detect platform key, extract platformData, compute averages/engagement, return normalized object, include `raw_response` and `enriched_at`).
7. **Add node: HTTP Request** (Supabase RPC)
   - Name: `Update Null Values Only`
   - Method: **POST**
   - URL: `https://<your-project-ref>.supabase.co/rest/v1/rpc/enrich_lead`
   - Authentication: **Predefined credential type** → Supabase credential
   - Headers: `Content-Type: application/json`
   - Body parameters:
     - `p_creator_handle` = `{{ $('Process in Batches').item.json.creator_handle }}`
     - `p_platform` = `{{ $('Process in Batches').item.json.platform }}`
     - `p_payload` = `{{ $json }}`
   - **Supabase-side requirement:** create an SQL function `enrich_lead(p_creator_handle text, p_platform text, p_payload jsonb)` (exact signature may vary) that updates only NULL columns.
8. **Add node: Wait**
   - Name: `Wait 5 Second`
   - Configure wait time to **5 seconds** (ensure it is “Wait amount of time”, not webhook wait).
9. **(Optional) Add node: Supabase Update** (alternative, not recommended)
   - Name: `Update a row`
   - Disable it by default.
   - Operation: **Update**
   - Table: `leads3`
   - Filter: choose a reliable key (prefer `id` from batch item) instead of `creator_handle = username`.
   - Map normalized fields to columns carefully (verify follower_count mapping, etc.).
10. **Connect nodes in this order:**
    - `Daily Refresh Schedule` → `List Influencers Without Enrichment` → `Process in Batches`
    - `Process in Batches (main output)` → `Influencers.club Enrichment API By Handle` → `Normalize Creator Enrichment Payload`
    - `Normalize Creator Enrichment Payload` → `Update Null Values Only` → `Wait 5 Second` → back to `Process in Batches` (to request next batch)
    - (Optional parallel) `Normalize Creator Enrichment Payload` → `Update a row` → `Wait 5 Second`
11. **Activate** the workflow once credentials and the Supabase RPC function are confirmed.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Get multi social platform data for creators from one social handle – Built for influencer marketing platforms. Full explanation | https://influencers.club/creatorbook/cross-platform-data-for-influencer-marketing-platforms/ |
| SQL function reference used by the “Update Null Values Only” approach | https://github.com/GjPetrovski-IC/N8N-Public-Templates |