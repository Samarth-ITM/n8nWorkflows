Design scalable sync workflows with Data Tables, ProspectPro and HubSpot

https://n8nworkflows.xyz/workflows/design-scalable-sync-workflows-with-data-tables--prospectpro-and-hubspot-13536


# Design scalable sync workflows with Data Tables, ProspectPro and HubSpot

## 1. Workflow Overview

**Workflow name (given):** Design scalable sync workflows with Data Tables, ProspectPro and HubSpot  
**Workflow name (in JSON):** Workflow Patterns & Boilerplate for Scaling up  
**Status:** Inactive (`active: false`)  
**Purpose:** A pattern library demonstrating scalable n8n designs for:
- Trigger workflows that control *cost (executions)* and *rate limits*
- Custom polling using **n8n Data Tables** as persistent checkpoints
- Manual backfills (manual triggers) with throttling and optional pagination
- “Manager / Function / Utility” workflow architecture for maintainable sync automation
- A minimal error-notification workflow using Error Trigger + Data Table logging + Telegram

The workflow is largely educational: many nodes are “templates” and some are disabled. The patterns are organized into blocks (Part 0–2 + Bonus), each with sticky notes explaining best practices.

### 1.1 Part 0 – Introduction & Modularity Examples (documentation-only)
Explains why to modularize into Trigger / Manager / Function / Utility workflows and how that helps scaling, rate limits, and maintenance.

### 1.2 Part 1 – Trigger Workflows (A, B, C)
Patterns to receive or poll items, throttle processing, and call a sub-workflow per item.

- **A:** Trigger basics (webhook + ProspectPro trigger) and per-item processing; optional batching/delays
- **B1/B2/B3:** Custom polling (Schedule Trigger) + Data Table checkpointing + optional pagination/looping
- **C1/C2/C3:** Manual backfills (Manual Trigger) with throttling and optional pagination

### 1.3 Part 2 – Workflow Types (D/E/F)
Patterns that represent how to structure downstream workflows:
- **D / D1:** Manager workflow patterns (state indicators/tags to avoid duplicates & provide user feedback)
- **E1:** Function workflow example (sync a ProspectPro “prospect” to HubSpot company, with success/error tags)
- **F1:** Utility workflow example (find HubSpot company and return a consistent `{result, hsCompany}` structure)

### 1.4 Bonus – Error Handling Pattern
A minimal “Error workflow” that logs executions to a Data Table and notifies via Telegram.

---

## 2. Block-by-Block Analysis

> Notes:
> - Many nodes use **“Execute Workflow”** pointing to a placeholder workflow **“Subflow - TEMPLATE”** (`workflowId: IdtvYdzQ2uGp7quU`). This workflow is not included; treat it as a required dependency.
> - Several triggers are **disabled**: this file is a pattern canvas more than a single runnable automation.
> - Data persistence for polling is done via **n8n Data Tables** (e.g., “Timestamps”, “Errors”) and **workflow static data** (pagination store).

---

### Block A – Trigger basics (A1 simple; A2 rate-limited batching)

**Overview:** Demonstrates two top-level trigger patterns: (A1) simple per-item processing and (A2) batch+delay throttling to prevent API rate limit issues.

**Nodes involved (functional):**
- Webhook
- New Website Visitors (ProspectPro Trigger)
- Process one item at a time - (1)
- Create Batches
- Process one item at a time (2)
- Delay after each batch

**Sticky notes (context):**
- “A. Triggers basics”
- “A1 - Simple, individual processing.”
- “A2 - Rate-limited individual processing”
- “Choose: A1 or A2?”

#### Node: **Webhook**
- **Type / role:** `Webhook` trigger; receives external events (A1-style).
- **Configuration:** Webhook path is a UUID-like string; default options.
- **Outputs:** Directly to “Process one item at a time - (1)”.
- **Edge cases:** Public endpoint security (auth/signature), bursts (hard to throttle in n8n), malformed payloads.

#### Node: **New Website Visitors**
- **Type / role:** `@bedrijfsdatanl/n8n-nodes-prospectpro.prospectproTrigger` (ProspectPro trigger) polling every minute.
- **Credentials:** `prospectproApi` (“TESTACCOUNT”).
- **Outputs:** Sends items both to:
  - A1: “Process one item at a time - (1)”
  - A2: “Create Batches”
- **Edge cases:** API auth errors, polling duplication, rate limits if too frequent.

#### Node: **Process one item at a time - (1)**
- **Type / role:** `Execute Workflow` to process each incoming item individually (sub-workflow pattern).
- **Configuration:**
  - `mode: each` (“Run once for each item”)
  - `waitForSubWorkflow: true`
  - Calls **Subflow - TEMPLATE** (`IdtvYdzQ2uGp7quU`)
  - Input schema expects `id` (string), but mapping is empty in the node (template state).
- **Connections:** Trigger → this node; no further connection shown in graph (acts as terminal).
- **Edge cases:** Sub-workflow missing, input mismatch (missing `id`), sub-workflow errors (propagate depending on `onError` of caller).

#### Node: **Create Batches**
- **Type / role:** `Split In Batches`; throttling pattern.
- **Configuration:** `batchSize: 10`.
- **Connections:** Input from “New Website Visitors”; output (loop) to “Delay after each batch” and per-item path to “Process one item at a time (2)” (via the node’s second output behavior as wired here).
- **Edge cases:** Must be looped correctly; otherwise it processes only first batch.

#### Node: **Process one item at a time (2)**
- **Type / role:** `Execute Workflow` per item within a batch.
- **Configuration:** same sub-workflow call as (1), `mode: each`, wait enabled.
- **Connections:** From “Create Batches” → this → “Delay after each batch”.
- **Edge cases:** same as (1) plus throughput/backpressure if sub-workflow is slow.

#### Node: **Delay after each batch**
- **Type / role:** `Wait` node used as delay between batches (rate limit smoothing).
- **Configuration:** parameters empty (template). In practice you’d set “Wait for” a fixed duration.
- **Connections:** After delay → loops back to “Create Batches”.
- **Edge cases:** Wait nodes increase execution time; if trigger overlaps (new executions start before old ends), you can create concurrency issues.

---

### Block B1 – Custom polling (simple) with Data Table checkpoint

**Overview:** A cost-optimized polling pattern: schedule runs periodically, read last sync timestamp from Data Table, fetch items since then, then update timestamp (but updates “now” before the fetch completes → risk of skipping items).

**Nodes involved:**
- B1 - Schedule Trigger
- B1 - Set variables
- B1 - Get timestamp
- B1 - Set lastSync
- B1 - Has timestamp / not first run?
- B1 - Update timestamp
- B1 - Create timestamp
- B1 - No Operation, do nothing
- B1 - Get Items
- B1 - Has Items?
- B1 - Items
- B1 - Filter Items

**Sticky notes (context):**
- “B1. Custom polling 1: simple…”
- “B1 - Start sync”
- “B1 - Get/set lastSync”
- “B1 - Create/update database entry”
- “B1 - Fetch new items”
- “B1 - Validate, prepare & filter items”

#### Node: **B1 - Schedule Trigger**
- **Type / role:** `Schedule Trigger` for periodic polling.
- **Configuration:** interval rule is present but empty `{}` (template); set your desired cadence.
- **Connections:** → “B1 - Set variables”.
- **Edge cases:** Overlapping runs if schedule is too frequent and processing is slow.

#### Node: **B1 - Set variables**
- **Type / role:** `Set` node; defines `timestamp_key`.
- **Config:** `timestamp_key = "demo_timestamp"`.
- **Connections:** → “B1 - Get timestamp”.
- **Edge cases:** Key collision if reused across workflows; choose a unique key per sync stream.

#### Node: **B1 - Get timestamp**
- **Type / role:** `Data Table` (operation: get) from Data Table **“Timestamps”**.
- **Config:** filter where `key == {{$('B1 - Set variables').first().json.timestamp_key}}`; `limit: 1`.
- **Error handling:** `onError: continueErrorOutput`, `alwaysOutputData: true` (ensures downstream gets an item even when not found).
- **Connections:** outputs to “B1 - Set lastSync” and “B1 - Has timestamp / not first run?”.
- **Edge cases:** Data Table missing, permissions, schema mismatch, empty results.

#### Node: **B1 - Set lastSync**
- **Type / role:** `Code` node; converts stored ISO datetime to unix seconds.
- **Code:** `lastSync = floor(new Date($json.value || 0).getTime()/1000)`
- **Connections:** → “B1 - Get Items”.
- **Edge cases:** `$json.value` not parseable; timezone assumptions; value may already be a unix timestamp.

#### Node: **B1 - Has timestamp / not first run?**
- **Type / role:** `IF` to decide update vs create.
- **Condition:** `!!$('B1 - Get timestamp').first().json.id`
- **True:** “B1 - Update timestamp”; **False:** “B1 - Create timestamp”
- **Edge cases:** because “Get timestamp” uses `alwaysOutputData`, you must rely on `id` existence as done here.

#### Node: **B1 - Update timestamp**
- **Type / role:** `Data Table` update row in “Timestamps”.
- **Config:** sets `value = now().toISOString()`, filters by row `id` and `key`.
- **Connections:** → “B1 - No Operation, do nothing”
- **Edge cases:** Updating timestamp before fetch can skip late-arriving items (documented in sticky note).

#### Node: **B1 - Create timestamp**
- **Type / role:** `Data Table` insert into “Timestamps”.
- **Config:** columns `key = timestamp_key`, `value = now().toISOString()`.
- **Connections:** → “B1 - No Operation, do nothing”

#### Node: **B1 - No Operation, do nothing**
- **Type / role:** `NoOp` terminal placeholder (keeps graph clean).

#### Node: **B1 - Get Items**
- **Type / role:** `HTTP Request` to ProspectPro endpoint.
- **Config:**
  - URL: `https://api.prospectpro.nl/v1.2/prospects`
  - Query: `from_last_visit={{$json.lastSync}}`, `sort_by=last_visit`, `sort_order=asc`
  - `onError: continueErrorOutput`
- **Connections:** → “B1 - Has Items?”
- **Credentials:** none set here (unlike B2/B3). In practice you need auth (header/token) or use credential-based auth.
- **Edge cases:** Missing auth, 429 rate limits, 5xx, response schema changes.

#### Node: **B1 - Has Items?**
- **Type / role:** `IF` checks response.
- **Condition:** `$json.found > 0`
- **True:** → “B1 - Items”
- **Edge cases:** If API returns different fields (e.g., `total` or array length only), condition fails.

#### Node: **B1 - Items**
- **Type / role:** `Split Out` field `items` into one item per prospect.
- **Connections:** → “B1 - Filter Items”
- **Edge cases:** `items` missing/non-array.

#### Node: **B1 - Filter Items**
- **Type / role:** `Filter` to ensure updated_time is newer than lastSync.
- **Condition:** `!!$json.updated_time && $json.updated_time > $('B1 - Set lastSync').first().json.lastSync`
- **Output:** `alwaysOutputData: true` (keeps flow consistent).
- **Edge cases:** `updated_time` format/units (seconds vs ms), missing field.

---

### Block B2 – Custom polling (optimal) with stable checkpointing and optional recursion

**Overview:** Similar to B1 but safer: after fetching sorted limited results, it updates `lastSync` based on the most recent returned item, reducing missed items. Includes optional self-calling loop (disabled) to continue when `found < total`.

**Nodes involved:**
- B2 - Schedule Trigger
- B2 - Set variables
- B2 - Get timestamp
- B2 - Set lastSync
- B2 - Get Items
- B2 - Has Items?
- B2 - Items
- B2 - Filter Items
- B2 - Update lastSync
- B2 - Has timestamp / not first run?
- B2 - Update timestamp
- B2 - Create timestamp
- B2 - No Operation, do nothing
- B2 - Needs loop? (disabled)
- B2 - Call [this workflow] (disabled)

#### Node: **B2 - Set variables**
- Sets:
  - `timestamp_key = "demo_timestamp"`
  - `limit = 20`

#### Node: **B2 - Get Items**
- **HTTP Request** with **predefined credential** `prospectproApi`.
- Query includes `limit={{ $('B2 - Set variables').first().json?.limit || undefined }}`.

#### Node: **B2 - Update lastSync**
- **Code** computes `lastSync` based on `companies[].changed_time` maximum; fallback to now.
- **Important mismatch risk:** This is querying `/prospects` but code refers to `$json.companies`. Template likely needs adaptation to ProspectPro response structure (`items`, maybe `prospects`, etc.).

#### Node: **B2 - Needs loop?** (disabled)
- Checks if more results exist: `found < total`.
- If true, would call **this same workflow** (recursive pattern).

#### Node: **B2 - Call [this workflow]** (disabled)
- Execute Workflow referencing current workflow id `w0NDnlGr_-uqHUHVrgPM_`.

**Edge cases (B2 overall):**
- Requires API supports: sorted by date ascending + limiting results.
- Recursion can create infinite loops if stop condition is wrong or data keeps arriving.

---

### Block B3 – Custom polling (complete) with manual pagination + page-wise processing

**Overview:** Full-control polling: schedule, read timestamp, reset pagination counters, fetch pages with limit/page, process items in throttled batches, then decide whether to fetch next page. Uses **workflow static data** for pagination state across node runs.

**Nodes involved:**
- B3 - Schedule Trigger
- B3 - Set variables
- B3 - Get timestamp
- B3 - Set lastSync
- B3 - Initiate pagination
- Wait (pagination delay)
- B3 - Set pagination
- B3 - Done looping?
- B3 - Get Items
- B3 - Update pagination
- B3 - Has Items?
- B3 - Items
- B3 - Filter Items
- B3 - Create Batches
- B3 - Process one item at a time
- B3 - Delay after each batch
- B3 - Update lastSync
- B3 - Has timestamp / not first run?
- B3 - Update timestamp
- B3 - Create timestamp
- B3 - No Operation, do nothing

#### Node: **B3 - Initiate pagination**
- **Code** resets global static data:
  - `store.page = 0`, `store.processed = 0`, `store.total_pages = 0`
- **Purpose:** avoids stale pagination state and supports “reset before start”.

#### Node: **Wait** (pagination delay)
- Wait node between pages, connected from “B3 - Create Batches” back to “B3 - Set pagination”.
- **Template:** duration not set; must be configured.

#### Node: **B3 - Set pagination**
- **Code** increments `store.page`, tracks `processed`, calculates `offset`, forwards `lastSync`.
- Output includes: `page`, `processed`, `total_pages`, `offset`, `lastSync`.

#### Node: **B3 - Done looping?**
- Condition: `$json.page > $json.total_pages`
- If **false** (second output), goes to “B3 - Get Items”.

#### Node: **B3 - Get Items**
- **HTTP Request** to ProspectPro with credential `prospectproApi`
- Query includes `limit`, `page`, `from_last_visit={{$json.lastSync}}`, sorting parameters.

#### Node: **B3 - Update pagination**
- **Code** attempts to infer total pages from various API styles:
  - WordPress header `x-wp-totalpages`
  - `last`, `total`, `next` fields
  - Computes `store.total_pages`
- **Customization required:** ProspectPro response likely differs; you must adjust logic.

#### Node: **B3 - Create Batches / Process / Delay**
- Same throttling structure as Block A2, but within each page.

**Edge cases (B3 overall):**
- Uses global static data: if executions overlap, state can corrupt.
- Pagination inference must match the target API.
- Long executions may hit timeouts depending on n8n settings and environment.

---

### Block C – Manual triggers (backfills)

**Overview:** Patterns for retroactive processing and gap filling: manual trigger → get items → optional pagination → throttled per-item sub-workflow calls.

**Nodes involved:**
- When clicking ‘Execute workflow’ (disabled)
- C1 - Process one item at a time
- C2 - Get Items
- C2 - Has Items?
- C2 - Items
- C2 - Filter Items
- C2 - Create Batches
- C2 - Process one item at a time
- C2 - Delay after each batch
- C3 - Set variables
- C3 - Set pagination
- C3 - Get Items
- C3 - Has Items?
- C3 - Items
- C3 - Filter Items
- C3 - Create Batches
- C3 - Process one item at a time
- C3 - Delay after each batch
- C3 - Wait

#### Node: **When clicking ‘Execute workflow’** (disabled)
- Manual trigger entry point for C1/C2/C3.
- Connected to:
  - C1 - Process one item at a time
  - C2 - Get Items
  - C3 - Set variables

#### Node: **C2 - Get Items**
- HTTP Request to ProspectPro (no auth configured in this node).
- Template query parameters empty.

#### Node: **C3 - Set pagination / C3 - Wait**
- Uses global static data to increment `page` and `processed`.
- Loops via “C3 - Wait” → “C3 - Set pagination”.

**Edge cases:**
- Manual runs can be stopped mid-way; without checkpointing, restart behavior may duplicate work.
- C2/C3 Filter Items reference B1/B3 lastSync nodes (cross-block dependency). If you run C blocks standalone, those references can break.

---

### Block D – Manager workflow example (simple) + D1 (full “indicators” pattern)

**Overview:** Shows how a manager orchestrates multi-step processing on one item and prevents:
1) duplicate concurrent runs (“in progress” tag),  
2) user collisions (visible state),  
3) endless re-runs after error (error tag).

**Nodes involved (D simple):**
- When Executed by [trigger workflow] (disabled)
- D - Qualify Prospect
- D - Qualification Successful?
- D - Identify DMU
- D - DMU identification successful?
- D - Sync Hubspot
- D - Hubspot sync successful?
- D - Set output
- D - Switch (var b) and its variant branch nodes (var b)

**Nodes involved (D1 full):**
- D1 - Set variables
- D1 - Get prospect
- D1 - Continue?
- D1 - Add "tag_in_progress"
- D1 - Update prospect
- D1 - Automatically Qualify Prospect
- D1 - Qualification Successful?
- D1 - Get Contacts
- D1 - Contacts search successful?
- D1 - Sync Hubspot
- D1 - Hubspot sync successful?
- D1 - Get prospect for final update: Success
- D1 - Add "tag_success", remove "tag_in_processing"
- D1 - Update Tags: Success
- D1 - Get prospect for final update: Fail
- D1 - Add "tag_error", remove "tag_in_progress"
- D1 - Update Tags: Fail
- D1 - Stop and Error

#### Node: **When Executed by [trigger workflow]** (disabled)
- `Execute Workflow Trigger` with `inputSource: passthrough`.
- This is the canonical start node for manager/function/utility workflows, but disabled here because this is a pattern board.

#### Node: **D - Set output**
- Set node returning `error_code = 200 or 500` depending on `!!$json.error`.
- **Key requirement:** always return exactly one item so upstream batching/wait-for-completion can proceed.

#### Node: **D1 - Set variables**
- Defines:
  - `tag_error = "EnrichmentFailed"`
  - `tag_success = "Enriched"`
- **Important:** Other nodes reference `tag_in_progress`, but it is not set here in the JSON. You must add `tag_in_progress` (e.g., `"EnrichmentInProgress"`).

#### Node: **D1 - Get prospect**
- ProspectPro node `operation: get` by `id={{$json.id}}`.
- Credentials: ProspectPro account “Bedrijfsdata.nl - XXX003”.
- `onError: continueErrorOutput`.
- **Edge cases:** prospect not found, auth errors, transient API issues.

#### Node: **D1 - Continue?**
- IF: continue only if tags do **not** include `tag_error` or `tag_in_progress`.
- Uses expression:
  - `!!$json.tags?.includes(tag_error) || !!$json.tags?.includes(tag_in_progress)` negated via `operation: false`
- **Edge case:** `tags` type mismatch (string vs array). ProspectPro tags sometimes appear as pipe-delimited string; later code assumes string.

#### Node: **D1 - Add "tag_in_progress"**
- Code: splits `prospect.tags` by `|` into an array, appends `tag_in_progress`, returns merged object.
- **Risk:** if ProspectPro expects tags as pipe-delimited string, patching with an array may fail unless the node converts.

#### Node: **D1 - Update prospect**
- ProspectPro node `operation: patch` with `tags={{$json.tags}}`.

#### Node: **D1 - Stop and Error**
- StopAndError returns a structured error object `{code:404, message:"Prospect not found"}`.
- Used as guard failure signal in this template.

---

### Block E1 – Function workflow example (sync ProspectPro → HubSpot company)

**Overview:** Demonstrates a function workflow: refresh item, guard conditions, try to find company (via utility F1), then update/create company in HubSpot and tag success/failure back in ProspectPro.

**Nodes involved:**
- E1 - Set variables
- E1 - Get prospect
- E1 - No Existing Errors?
- E1 - Continue?
- E1 - Find Company in Hubspot (Execute Workflow → template utility)
- E1 - Found?
- E1 - Update a company
- E1 - Update Successful?
- E1 - Create a company
- E1 - Creation Successful?
- E1 - Add "tag_success"
- E1 - Add "tag_error"
- E1 - Update Tags: Success
- E1 - Update Tags: Fail
- E1 - return { result: true };
- E1 - return { result: false };
- E1 - Stop and Error

#### Node: **E1 - Continue?**
- Guards: requires `domain` and `id` and includes an extra empty-string equals condition (template artifact).
- False branch stops with error.

#### Node: **E1 - Find Company in Hubspot**
- ExecuteWorkflow calling **Subflow - TEMPLATE** (intended to be utility F1).
- **Issue:** workflow input mapping uses `id={{ $('D1 - Add "tag_in_progress"').item.json.id }}` which is a cross-block reference and likely wrong for E1. Should be `id={{$json.id}}` (from E1 context) or prospect’s id.

#### Node: **E1 - Update a company / Create a company**
- HubSpot node (`n8n-nodes-base.hubspot`) using OAuth2.
- Update sets domain and custom properties `prospectpro_id` and `kvk`.
- Create sets many company fields from ProspectPro (city, address, url, revenue, employees, etc.).
- **Edge cases:** custom properties must exist in HubSpot; validation errors; rate limits; domain formatting.

#### Node: **E1 - Add "tag_success" / Add "tag_error"**
- Code nodes: take `prospect.tags` string, split by `|`, add desired tag, ensure uniqueness.
- Patch back into ProspectPro via “E1 - Update Tags:*”.

---

### Block F1 – Utility workflow example (find HubSpot company, return normalized result)

**Overview:** Searches HubSpot companies by `prospectpro_id`, falls back to domain if needed, then returns a consistent output structure for callers.

**Nodes involved:**
- Search Companies by Bedrijfsdata ID
- Found by ID?
- Has Domain?
- Search Companies by Domain
- Found by Domain?
- F1 - return Hubspot Company object
- F1 - return { result: true, hsCompany: Object };
- F1 - return { result: false };

#### Node: **Search Companies by Bedrijfsdata ID**
- HubSpot CRM Search API POST `/crm/v3/objects/companies/search`
- Body filters `prospectpro_id == {{$json.id}}`.
- Credentials: `hubspotOAuth2Api` (“HubSpot account 2”).
- **Edge cases:** `prospectpro_id` property must exist; permissions/scopes for CRM search.

#### Node: **Search Companies by Domain**
- Same endpoint, filters `domain == {{ $('E1 - Get prospect').item.json.domain }}`
- **Risk:** cross-block dependency again: a utility should not depend on E1 node names; should use its own input (`$json.domain`).

#### Node: **F1 - return Hubspot Company object**
- Returns first company from `results[0]`.

#### Node: **F1 - return { result: true, hsCompany: Object };**
- Normalizes output: `{ result: true, hsCompany: $json }`.

#### Node: **F1 - return { result: false };**
- Normalizes “not found”.

---

### Block Bonus – Error handling workflow

**Overview:** Uses `Error Trigger` to capture any workflow failure, logs it to a Data Table, and sends a Telegram message.

**Nodes involved:**
- 1 - Errors - Error Trigger
- 1 - Errors - Insert row
- 1 - Errors - Send a text message

#### Node: **1 - Errors - Error Trigger**
- Triggers on workflow errors in the n8n instance.

#### Node: **1 - Errors - Insert row**
- Data Table insert into **“Errors”** with `workflow_id`, `execution_id`, `url`.
- Retries enabled (`retryOnFail: true`, `maxTries: 2`).
- **Edge cases:** Data Table unavailable → logging fails, but `onError: continueRegularOutput` prevents total break.

#### Node: **1 - Errors - Send a text message**
- Telegram node with retries (`maxTries: 5`, `waitBetweenTries: 5000`).
- Text includes workflow id, execution id, and URL.
- **Edge cases:** Telegram downtime; chatId placeholder “000” must be replaced.

---

## 3. Summary Table

> “Sticky Note” column duplicates the sticky content for nodes covered by that note region. Large background color blocks (empty content) are omitted (blank).

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation |  |  | ## A. Triggers basics … (full note content) |
| Sticky Note7 | Sticky Note | Documentation |  |  | ### A2 - Rate-limited individual processing… |
| Create Batches | Split In Batches | Batch throttling (A2) | New Website Visitors; Delay after each batch | Process one item at a time (2) | ### A2 - Rate-limited individual processing… |
| Process one item at a time - (1) | Execute Workflow | Per-item sub-workflow call (A1) | Webhook; New Website Visitors |  | ### A1 - Simple, individual processing… |
| Process one item at a time (2) | Execute Workflow | Per-item call inside batches (A2) | Create Batches | Delay after each batch | ### A2 - Rate-limited individual processing… |
| Delay after each batch | Wait | Delay between batches | Process one item at a time (2) | Create Batches | ### A2 - Rate-limited individual processing… |
| Sticky Note9 | Sticky Note | Background |  |  |  |
| Sticky Note10 | Sticky Note | Documentation |  |  | ### A1 - Simple, individual processing… |
| New Website Visitors | ProspectPro Trigger | Poll ProspectPro visitors |  | Process one item at a time - (1); Create Batches | ### A1 - Simple… / ### A2 - Rate-limited… (choose pattern) |
| Sticky Note6 | Sticky Note | Documentation |  |  | ## B1. Custom polling 1… |
| Sticky Note11 | Sticky Note | Documentation |  |  | ### B1 - Start sync… |
| Sticky Note12 | Sticky Note | Documentation |  |  | ### B1 - Get/set lastSync… |
| Sticky Note13 | Sticky Note | Documentation |  |  | ### B1 - Create/update database entry… |
| Sticky Note14 | Sticky Note | Documentation |  |  | ### B1 - Fetch new items… |
| Sticky Note15 | Sticky Note | Documentation |  |  | ### B1 - Validate, prepare & filter items… |
| Sticky Note16 | Sticky Note | Background |  |  |  |
| B1 - Schedule Trigger | Schedule Trigger | Start B1 polling |  | B1 - Set variables | ### B1 - Start sync… |
| B1 - Set variables | Set | Define timestamp key | B1 - Schedule Trigger | B1 - Get timestamp | ### B1 - Start sync… |
| B1 - Get timestamp | Data Table | Read last checkpoint | B1 - Set variables | B1 - Set lastSync; B1 - Has timestamp / not first run? | ### B1 - Get/set lastSync… |
| B1 - Set lastSync | Code | Convert stored timestamp → unix | B1 - Get timestamp | B1 - Get Items | ### B1 - Get/set lastSync… |
| B1 - Get Items | HTTP Request | Fetch items since lastSync | B1 - Set lastSync | B1 - Has Items? | ### B1 - Fetch new items… |
| B1 - Has Items? | IF | Check if results exist | B1 - Get Items | B1 - Items | ### B1 - Validate, prepare & filter items… |
| B1 - Items | Split Out | Expand items array | B1 - Has Items? | B1 - Filter Items | ### B1 - Validate, prepare & filter items… |
| B1 - Filter Items | Filter | Ensure updated_time > lastSync | B1 - Items |  | ### B1 - Validate, prepare & filter items… |
| B1 - Has timestamp / not first run? | IF | Choose update vs create timestamp | B1 - Get timestamp | B1 - Update timestamp; B1 - Create timestamp | ### B1 - Create/update database entry… |
| B1 - Update timestamp | Data Table | Update checkpoint | B1 - Has timestamp / not first run? | B1 - No Operation, do nothing | ### B1 - Create/update database entry… |
| B1 - Create timestamp | Data Table | Create checkpoint | B1 - Has timestamp / not first run? | B1 - No Operation, do nothing | ### B1 - Create/update database entry… |
| B1 - No Operation, do nothing | NoOp | Terminal placeholder | B1 - Update timestamp; B1 - Create timestamp |  |  |
| Sticky Note17 | Sticky Note | Documentation |  |  | ### B2 - Start sync… |
| Sticky Note18 | Sticky Note | Documentation |  |  | ### B2 - Get/set lastSync… |
| Sticky Note19 | Sticky Note | Documentation |  |  | ### B2 - Create/update database entry… |
| Sticky Note20 | Sticky Note | Documentation |  |  | ### B2 - Fetch new items… |
| Sticky Note21 | Sticky Note | Documentation |  |  | ### B2 - Validate, prepare & filter items… |
| B2 - Schedule Trigger | Schedule Trigger | Start B2 polling |  | B2 - Set variables | ### B2 - Start sync… |
| B2 - Set variables | Set | Define timestamp key + limit | B2 - Schedule Trigger | B2 - Get timestamp | ### B2 - Start sync… |
| B2 - Get timestamp | Data Table | Read checkpoint | B2 - Set variables | B2 - Set lastSync | ### B2 - Get/set lastSync… |
| B2 - Set lastSync | Code | Timestamp → unix | B2 - Get timestamp | B2 - Get Items | ### B2 - Get/set lastSync… |
| B2 - Get Items | HTTP Request | Fetch items since lastSync (limited) | B2 - Set lastSync | B2 - Has Items?; B2 - Update lastSync | ### B2 - Fetch new items… |
| B2 - Has Items? | IF | Check results | B2 - Get Items | B2 - Items | ### B2 - Validate, prepare & filter items… |
| B2 - Items | Split Out | Expand items | B2 - Has Items? | B2 - Filter Items | ### B2 - Validate, prepare & filter items… |
| B2 - Filter Items | Filter | Ensure updated_time > lastSync | B2 - Items | B2 - Needs loop? | ### B2 - Validate, prepare & filter items… |
| B2 - Update lastSync | Code | Compute next checkpoint from results | B2 - Get Items | B2 - Has timestamp / not first run? | ### B2 - Create/update database entry… |
| B2 - Has timestamp / not first run? | IF | Update vs create checkpoint | B2 - Update lastSync | B2 - Update timestamp; B2 - Create timestamp | ### B2 - Create/update database entry… |
| B2 - Update timestamp | Data Table | Update checkpoint row | B2 - Has timestamp / not first run? | B2 - No Operation, do nothing | ### B2 - Create/update database entry… |
| B2 - Create timestamp | Data Table | Create checkpoint row | B2 - Has timestamp / not first run? | B2 - No Operation, do nothing | ### B2 - Create/update database entry… |
| B2 - No Operation, do nothing | NoOp | Terminal placeholder | B2 - Update timestamp; B2 - Create timestamp |  |  |
| B2 - Needs loop? | IF (disabled) | Decide recursion | B2 - Filter Items | B2 - Call [this workflow] | ### DANGER ZONE (optional): repeat… |
| B2 - Call [this workflow] | Execute Workflow (disabled) | Recursive call | B2 - Needs loop? |  | ### DANGER ZONE (optional): repeat… |
| Sticky Note8 | Sticky Note | Documentation |  |  | ## B2. Custom polling 2… |
| Sticky Note24 | Sticky Note | Documentation |  |  | ### B2 - Enable looping (optional)… |
| Sticky Note25 | Sticky Note | Documentation |  |  | ### DANGER ZONE (optional): repeat… |
| Sticky Note31 | Sticky Note | Documentation |  |  | ## B3 - Custom polling 3… |
| B3 - Schedule Trigger | Schedule Trigger | Start B3 polling |  | B3 - Set variables | ### B3 - Start sync… |
| B3 - Set variables | Set | timestamp key + limit | B3 - Schedule Trigger | B3 - Get timestamp | ### B3 - Start sync… |
| B3 - Get timestamp | Data Table | Read checkpoint | B3 - Set variables | B3 - Set lastSync | ### B3 - Get/set lastSync & setup pagination… |
| B3 - Set lastSync | Code | Timestamp → unix | B3 - Get timestamp | B3 - Initiate pagination | ### B3 - Get/set lastSync & setup pagination… |
| B3 - Initiate pagination | Code | Reset global pagination state | B3 - Set lastSync | B3 - Set pagination | ### B3 - Pagination… |
| B3 - Set pagination | Code | Increment page/offset counters | Wait | B3 - Done looping? | ### B3 - Pagination… |
| B3 - Done looping? | IF | Decide whether to fetch page | B3 - Set pagination | B3 - Get Items | ### DANGER ZONE (optional): repeat… |
| B3 - Get Items | HTTP Request | Fetch page of items | B3 - Done looping? | B3 - Update lastSync; B3 - Update pagination | ### B3 - Fetch new items… |
| B3 - Update pagination | Code | Compute total pages (customize) | B3 - Get Items | B3 - Has Items? | ### B3 - Update pagination… |
| B3 - Has Items? | IF | Check results | B3 - Update pagination | B3 - Items | ### B3 - Validate, prepare & filter items… |
| B3 - Items | Split Out | Expand items | B3 - Has Items? | B3 - Filter Items | ### B3 - Validate, prepare & filter items… |
| B3 - Filter Items | Filter | updated_time > lastSync | B3 - Items | B3 - Create Batches | ### B3 - Validate, prepare & filter items… |
| B3 - Create Batches | Split In Batches | Throttle per-page processing | B3 - Filter Items; B3 - Delay after each batch | Wait; B3 - Process one item at a time | ### A2 - Rate-limited individual processing… |
| B3 - Process one item at a time | Execute Workflow | Subflow per item | B3 - Create Batches | B3 - Delay after each batch | ### A2 - Rate-limited individual processing… |
| B3 - Delay after each batch | Wait | Delay between batches | B3 - Process one item at a time | B3 - Create Batches | ### A2 - Rate-limited individual processing… |
| Wait | Wait | Delay after each processed page | B3 - Create Batches | B3 - Set pagination | ### B3 - Pagination delay… |
| B3 - Update lastSync | Code | Compute next checkpoint | B3 - Get Items | B3 - Has timestamp / not first run? | ### B3 - Create/update database entry… |
| B3 - Has timestamp / not first run? | IF | Update/create checkpoint | B3 - Update lastSync | B3 - Update timestamp; B3 - Create timestamp | ### B3 - Create/update database entry… |
| B3 - Update timestamp | Data Table | Update checkpoint | B3 - Has timestamp / not first run? | B3 - No Operation, do nothing | ### B3 - Create/update database entry… |
| B3 - Create timestamp | Data Table | Create checkpoint | B3 - Has timestamp / not first run? | B3 - No Operation, do nothing | ### B3 - Create/update database entry… |
| B3 - No Operation, do nothing | NoOp | Terminal | B3 - Update timestamp; B3 - Create timestamp |  |  |
| When clicking ‘Execute workflow’ | Manual Trigger (disabled) | Start manual backfills |  | C1/C2/C3 branches | ## C. Manual triggers… |
| C1 - Process one item at a time | Execute Workflow | Manual single-item subflow | When clicking ‘Execute workflow’ |  | ### C1 - Simple example. |
| C2 - Get Items | HTTP Request | Manual fetch all | When clicking ‘Execute workflow’ | C2 - Has Items? | ### C2 - All items… |
| C2 - Has Items? | IF | Check results | C2 - Get Items | C2 - Items | ### C2 - Validate, prepare & filter items… |
| C2 - Items | Split Out | Expand items | C2 - Has Items? | C2 - Filter Items | ### C2 - Validate, prepare & filter items… |
| C2 - Filter Items | Filter | Filter updated_time | C2 - Items | C2 - Create Batches | ### C2 - Validate, prepare & filter items… |
| C2 - Create Batches | Split In Batches | Throttle | C2 - Filter Items; C2 - Delay after each batch | C2 - Process one item at a time | ### C2 - Rate-limited individual processing… |
| C2 - Process one item at a time | Execute Workflow | Subflow per item | C2 - Create Batches | C2 - Delay after each batch | ### C2 - Rate-limited individual processing… |
| C2 - Delay after each batch | Wait | Delay between batches | C2 - Process one item at a time | C2 - Create Batches | ### C2 - Rate-limited individual processing… |
| C3 - Set variables | Set | Manual pagination config | When clicking ‘Execute workflow’ | C3 - Set pagination | ### C3 - Setup… |
| C3 - Set pagination | Code | Increment page counters | C3 - Set variables; C3 - Wait | C3 - Get Items | ### C3 - Loop… |
| C3 - Get Items | HTTP Request | Fetch paged results | C3 - Set pagination | C3 - Has Items? | ### C3 - Get Items… |
| C3 - Has Items? | IF | Stop loop if empty | C3 - Get Items | C3 - Items | ### C3 - Validate, prepare & filter items… |
| C3 - Items | Split Out | Expand items | C3 - Has Items? | C3 - Filter Items | ### C3 - Validate, prepare & filter items… |
| C3 - Filter Items | Filter | Filter updated_time | C3 - Items | C3 - Create Batches | ### C3 - Validate, prepare & filter items… |
| C3 - Create Batches | Split In Batches | Throttle per page | C3 - Filter Items; C3 - Delay after each batch | C3 - Wait; C3 - Process one item at a time | ### C3 - Rate-limited individual processing… |
| C3 - Process one item at a time | Execute Workflow | Subflow per item | C3 - Create Batches | C3 - Delay after each batch | ### C3 - Rate-limited individual processing… |
| C3 - Delay after each batch | Wait | Delay between batches | C3 - Process one item at a time | C3 - Create Batches | ### C3 - Rate-limited individual processing… |
| C3 - Wait | Wait | Delay after each page | C3 - Create Batches | C3 - Set pagination | ### C3 - Pagination delay… |
| D1 - Set variables | Set | Define indicator tags | When Executed by [trigger workflow] | D1 - Get prospect | ### D1 - Set variables… |
| D1 - Get prospect | ProspectPro | Refresh item | D1 - Set variables | D1 - Continue?; D1 - Stop and Error | ### D1 - Guard Error… |
| D1 - Continue? | IF | Skip if already processed/in-progress | D1 - Get prospect | D1 - Add "tag_in_progress" | ### D1 - Set "in progress"-indicator… |
| D1 - Add "tag_in_progress" | Code | Build updated tags list | D1 - Continue? | D1 - Update prospect | ### D1 - Set "in progress"-indicator… |
| D1 - Update prospect | ProspectPro | Persist tags | D1 - Add "tag_in_progress" | D1 - Automatically Qualify Prospect | ### D1 - Set "in progress"-indicator… |
| D1 - Automatically Qualify Prospect | Execute Workflow | Step 1 (function) | D1 - Update prospect | D1 - Qualification Successful?; D1 - Get prospect for final update: Fail | ### D1 - Your process… |
| D1 - Qualification Successful? | IF | Step 1 ok? | D1 - Automatically Qualify Prospect | D1 - Get Contacts | ### D1 - Your process… |
| D1 - Get Contacts | Execute Workflow | Step 2 (function) | D1 - Qualification Successful? | D1 - Contacts search successful? | ### D1 - Your process… |
| D1 - Contacts search successful? | IF | Step 2 ok? | D1 - Get Contacts | D1 - Sync Hubspot; D1 - Get prospect for final update: Fail | ### D1 - Your process… |
| D1 - Sync Hubspot | Execute Workflow | Step 3 (function) | D1 - Contacts search successful? | D1 - Hubspot sync successful?; D1 - Get prospect for final update: Fail | ### D1 - Your process… |
| D1 - Hubspot sync successful? | IF | Step 3 ok? | D1 - Sync Hubspot | D1 - Get prospect for final update: Success | ### D1 - Completed successfully… / Completed with errors… |
| D1 - Get prospect for final update: Success | ProspectPro | Refresh before final tag set | D1 - Hubspot sync successful? | D1 - Add "tag_success"... | ### D1 - Completed successfully… |
| D1 - Add "tag_success", remove "tag_in_processing" | Code | Remove in-progress, add success | D1 - Get prospect for final update: Success | D1 - Update Tags: Success | ### D1 - Completed successfully… |
| D1 - Update Tags: Success | ProspectPro | Persist success tags | D1 - Add "tag_success"... |  | ### D1 - Completed successfully… |
| D1 - Get prospect for final update: Fail | ProspectPro | Refresh before error tag set | D1 failures | D1 - Add "tag_error"... | ### D1 - Completed with errors… |
| D1 - Add "tag_error", remove "tag_in_progress" | Code | Remove in-progress, add error | D1 - Get prospect for final update: Fail | D1 - Update Tags: Fail | ### D1 - Completed with errors… |
| D1 - Update Tags: Fail | ProspectPro | Persist error tags | D1 - Add "tag_error"... |  | ### D1 - Completed with errors… |
| D1 - Stop and Error | StopAndError | Guard termination | D1 - Get prospect; D1 - Update prospect |  | ### D1 - Guard Error… |
| E1 - Set variables | Set | Define success/error tags | When Executed by [trigger workflow] | E1 - Get prospect | ### E1 - Set variables… |
| E1 - Get prospect | ProspectPro | Refresh item | E1 - Set variables | E1 - No Existing Errors?; E1 - Stop and Error | ### E1 - Refresh item… |
| E1 - No Existing Errors? | IF | Guard: skip if prior failure tag | E1 - Get prospect | E1 - Continue? | ### E1 - Guard your workflow… |
| E1 - Continue? | IF | Guard required fields | E1 - No Existing Errors? | E1 - Find Company in Hubspot; E1 - Stop and Error | ### E1 - Guard your workflow… |
| E1 - Find Company in Hubspot | Execute Workflow | Call utility (find company) | E1 - Continue? | E1 - Found?; E1 - Create a company | ### E1 - Your workflow… |
| E1 - Found? | IF | Branch update vs create | E1 - Find Company in Hubspot | E1 - Update a company; E1 - Create a company | ### E1 - Your workflow… |
| E1 - Update a company | HubSpot | Update company properties | E1 - Found? | E1 - Update Successful?; E1 - Add "tag_error" | ### E1 - Your workflow… |
| E1 - Update Successful? | IF | Update ok? | E1 - Update a company | E1 - Add "tag_success"; E1 - Add "tag_error" | ### E1 - Your workflow… |
| E1 - Create a company | HubSpot | Create company | E1 - Found? | E1 - Creation Successful?; E1 - Add "tag_error" | ### E1 - Your workflow… |
| E1 - Creation Successful? | IF | Create ok? | E1 - Create a company | E1 - Add "tag_success"; E1 - Add "tag_error" | ### E1 - Your workflow… |
| E1 - Add "tag_success" | Code | Add success tag | E1 success | E1 - Update Tags: Success | ### E1 - Add a success indicator (optional)… |
| E1 - Update Tags: Success | ProspectPro | Persist tags | E1 - Add "tag_success" | E1 - return { result: true }; | ### E1 - Add a success indicator (optional)… |
| E1 - return { result: true }; | Code | Return normalized result | E1 - Update Tags: Success |  | ### E1 - Set output… |
| E1 - Add "tag_error" | Code | Add error tag | E1 failures | E1 - Update Tags: Fail | ### E1 - Add a error indicator (optional)… |
| E1 - Update Tags: Fail | ProspectPro | Persist tags | E1 - Add "tag_error" | E1 - return { result: false }; | ### E1 - Add a error indicator (optional)… |
| E1 - return { result: false }; | Code | Return normalized result | E1 - Update Tags: Fail |  | ### E1 - Set output… |
| E1 - Stop and Error | StopAndError | Guard termination | E1 guards |  | ### E1 - Guard Error… |
| Search Companies by Bedrijfsdata ID | HTTP Request | HubSpot search by prospectpro_id | When Executed by [trigger workflow] | Found by ID?; Has Domain? | ### F1 - Your workflow… |
| Found by ID? | IF | Found? | Search Companies by Bedrijfsdata ID | F1 - return Hubspot Company object; Has Domain? | ### F1 - Your workflow… |
| Has Domain? | IF | Guard for domain fallback | Found by ID? | Search Companies by Domain; F1 - return { result: false }; | ### F1 - Your workflow… |
| Search Companies by Domain | HTTP Request | HubSpot search by domain | Has Domain? | Found by Domain?; F1 - return { result: false }; | ### F1 - Your workflow… |
| Found by Domain? | IF | Found? | Search Companies by Domain | F1 - return Hubspot Company object; F1 - return { result: false }; | ### F1 - Your workflow… |
| F1 - return Hubspot Company object | Code | Extract first company result | Found by ID? / Found by Domain? | F1 - return { result: true, hsCompany: Object }; | ### F1 - Set output… |
| F1 - return { result: true, hsCompany: Object }; | Code | Utility success output | F1 - return Hubspot Company object |  | ### F1 - Set output… |
| F1 - return { result: false }; | Code | Utility not-found output | Has Domain? / Found by Domain? / Search Companies by Domain |  | ### F1 - Set output… |
| 1 - Errors - Error Trigger | Error Trigger | Catch workflow errors |  | 1 - Errors - Insert row; 1 - Errors - Send a text message | ## 1 - Errors… |
| 1 - Errors - Insert row | Data Table | Log error execution | 1 - Errors - Error Trigger |  | ### 1 - Errors - Log the issue… |
| 1 - Errors - Send a text message | Telegram | Notify via Telegram | 1 - Errors - Error Trigger |  | ### 1 - Errors - Notify yourself… |

(Sticky-note-only nodes not listed exhaustively here are documentation/background elements with no runtime role.)

---

## 4. Reproducing the Workflow from Scratch

Because this workflow is a **pattern board**, you typically reproduce only the patterns you need. The steps below rebuild the full structure as provided.

1) **Create a new workflow**
- Name it: “Workflow Patterns & Boilerplate for Scaling up” (or your title).
- Set workflow to **Inactive** during build.

2) **Create required credentials**
- **ProspectPro API credential** (custom node package `@bedrijfsdatanl/n8n-nodes-prospectpro`):
  - Create credential(s) matching your account(s).
- **HubSpot OAuth2**:
  - Add HubSpot OAuth2 credential with scopes to read/write Companies and use Search API.
- **Telegram API**:
  - Bot token + chat ID.

3) **Create required Data Tables**
- Data Table **“Timestamps”** with columns:
  - `key` (string)
  - `value` (dateTime)
- Data Table **“Errors”** with columns:
  - `workflow_id` (string)
  - `execution_id` (string)
  - `url` (string)

4) **Create (or import) the dependency sub-workflow**
- Create a workflow named **“Subflow - TEMPLATE”** (or replace all ExecuteWorkflow nodes with your actual workflows).
- Ensure it accepts an `id` input and returns exactly **one item**.

---

### Build A: Trigger basics

5) Add **Webhook** node
- Set a path (unique).
- Connect to “Process one item at a time - (1)”.

6) Add **ProspectPro Trigger** node “New Website Visitors”
- Polling: every minute (or adjust).
- Set ProspectPro credential.
- Connect it to:
  - “Process one item at a time - (1)” (A1)
  - “Create Batches” (A2)

7) Add **Execute Workflow** node “Process one item at a time - (1)”
- Mode: **Run once for each item**
- Wait for completion: **On**
- Workflow: your manager workflow (or Subflow - TEMPLATE)
- Map input: `id = {{$json.id}}` (recommended)

8) Add **Split In Batches** “Create Batches”
- Batch size: 10
- Connect to “Process one item at a time (2)” and loop via “Delay after each batch”.

9) Add **Execute Workflow** “Process one item at a time (2)”
- Same settings as step 7.
- Connect to “Delay after each batch”.

10) Add **Wait** “Delay after each batch”
- Configure wait duration (e.g., 2–10 seconds).
- Connect back to “Create Batches”.

---

### Build B1: Simple polling with Data Tables

11) Add **Schedule Trigger** “B1 - Schedule Trigger”
- Set interval (e.g., every 15 minutes).

12) Add **Set** “B1 - Set variables”
- Add string field `timestamp_key = demo_timestamp`.

13) Add **Data Table** “B1 - Get timestamp”
- Operation: Get
- Table: “Timestamps”
- Filter: `key == {{$('B1 - Set variables').first().json.timestamp_key}}`
- Limit: 1
- On error: “Continue”
- Always output data: On

14) Add **Code** “B1 - Set lastSync”
- Return unix seconds from stored timestamp value.

15) Add **IF** “B1 - Has timestamp / not first run?”
- Condition: `!!$('B1 - Get timestamp').first().json.id`
- True → Update; False → Create.

16) Add **Data Table** “B1 - Update timestamp”
- Operation: Update
- Set `value = {{new Date().toISOString()}}`
- Filter by row `id` and `key`.

17) Add **Data Table** “B1 - Create timestamp”
- Operation: Insert
- Columns: `key`, `value`.

18) Add **HTTP Request** “B1 - Get Items”
- URL: `https://api.prospectpro.nl/v1.2/prospects`
- Add auth (either predefined credential or headers)
- Query:
  - `from_last_visit={{$json.lastSync}}`
  - `sort_by=last_visit`
  - `sort_order=asc`

19) Add **IF** “B1 - Has Items?” → **Split Out** “B1 - Items” (field `items`) → **Filter** “B1 - Filter Items”
- Filter condition: `updated_time > lastSync`.

---

### Build B2/B3/C/D/E/F
- Follow the same node types and configurations shown in the sections above.
- **Important adjustments to make these runnable:**
  - Add `tag_in_progress` to D1/E1 variable sets.
  - Remove cross-block node references (e.g., utilities must use `$json.domain`, not `$('E1 - Get prospect')...`).
  - Configure all Wait nodes with explicit durations.
  - Configure missing ProspectPro auth in HTTP Request nodes where credentials are not set.

---

### Build Bonus: Error workflow

20) Add **Error Trigger**
21) Add **Data Table** insert row into “Errors”
- Map:
  - `workflow_id = {{$json.workflow.id}}`
  - `execution_id = {{$json.execution.id}}`
  - `url = {{$json.execution.url}}`

22) Add **Telegram** send message
- Replace chatId “000” with real chat id.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Subflow settings: Mode → Run once for each item; Wait For Sub-Workflow Completion → On” | Part A sticky notes; critical for batching triggers |
| “Subflow-executions don’t count as executions in n8n billing scheme (31-01-2026)” | Part A sticky note (billing behavior may change; verify with n8n plan) |
| B1 limitation: timestamp updated before fetch → can skip items | B1 sticky note (“not perfect”) |
| B2 preference over B1 when API supports sorted+limited results | B2 sticky note |
| B3 uses static workflow data for pagination; prevent overlapping executions | B3 sticky note (“executions don’t overlap”) |
| Error notifications should have a fallback channel | Bonus “1 - Errors” sticky note |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.