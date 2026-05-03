Track certification requirements with ScrapeGraphAI, GitLab and Rocket.Chat

https://n8nworkflows.xyz/workflows/track-certification-requirements-with-scrapegraphai--gitlab-and-rocket-chat-12234


# Track certification requirements with ScrapeGraphAI, GitLab and Rocket.Chat

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Track certification requirements with ScrapeGraphAI, GitLab and Rocket.Chat  
**Workflow name (in JSON):** Certification Requirement Tracker with Rocket.Chat and GitLab

**Purpose:**  
Monitors certification renewal/requirement pages (one or more URLs), extracts the latest requirement text and update date using **ScrapeGraphAI**, compares results with a **baseline JSON file stored in GitLab**, and when changes are detected it **alerts a Rocket.Chat channel** and **commits the updated baseline** back to GitLab. The workflow is triggered via an **HTTP POST webhook** and always returns an HTTP response via a Respond to Webhook node.

**Typical use cases:**
- Compliance/certification teams tracking renewal rules across multiple certification bodies
- Automated “watchers” for policy/requirements pages
- Audit-friendly change history using Git version control

### 1.1 Input Reception & URL List Preparation
Webhook trigger receives a POST, then a Code node outputs one item per monitored URL.

### 1.2 AI Scraping & Normalization
ScrapeGraphAI extracts structured data from each URL. A Code node normalizes fields, ISO-formats the date, and adds a hash.

### 1.3 Aggregation & Baseline Retrieval
Aggregates all results into a single array object, fetches the prior baseline file from GitLab, and merges both for side-by-side comparison.

### 1.4 Change Detection & Routing
Compares the newly scraped array to the baseline content and sets `changed` plus a `diff`. An IF node routes to either “alert + update” or “respond only”.

### 1.5 Notification, Storage Update & Webhook Response
If changed: send Rocket.Chat message and update GitLab file, then respond. If not changed: respond immediately.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Config (Webhook + URL items)
**Overview:** Receives an external trigger and prepares the list of certification URLs to scrape as separate n8n items (parallel-friendly fan-out).  
**Nodes involved:**  
- Incoming Webhook  
- Prepare Certification URLs

#### Node: “Incoming Webhook”
- **Type / role:** `n8n-nodes-base.webhook` — entry point trigger (HTTP).
- **Configuration (interpreted):**
  - **Method:** POST
  - **Path:** `certification-requirements`
  - **Response mode:** “Respond via Respond to Webhook node” (responseMode = `responseNode`)
- **Key expressions/variables:** none
- **Connections:**
  - **Outputs:** → Prepare Certification URLs
- **Version-specific notes:** Node `typeVersion: 1`.
- **Edge cases / failures:**
  - If a caller expects an immediate response but the workflow takes too long (scraping + GitLab), the request may time out at the client/reverse proxy level.
  - If the workflow errors before reaching “Respond to Webhook”, the caller receives an error/timeout depending on n8n configuration.

#### Node: “Prepare Certification URLs”
- **Type / role:** `n8n-nodes-base.code` — produces one item per monitored site.
- **Configuration choices:**
  - Returns a static array of items shaped as `{ json: { url: '...' } }`.
  - Currently includes sample URLs:
    - `https://example-certbody1.org/requirements`
    - `https://example-association2.com/certification-updates`
- **Key expressions/variables:** none; pure JS return.
- **Connections:**
  - **Input:** from Incoming Webhook
  - **Output:** → Scrape Certification Requirements (each item independently)
- **Version-specific notes:** `typeVersion: 2` (Code node uses modern runtime conventions).
- **Edge cases / failures:**
  - Invalid/blocked URLs will fail downstream scraping.
  - If you later need per-site headers/cookies, they are not modeled here (would require adding fields and using them downstream).

**Sticky note covering this block:** “Section – Trigger & Config” (explains the decoupled webhook and URL-list approach).

---

### Block 2 — Scraping (ScrapeGraphAI + normalization)
**Overview:** Uses ScrapeGraphAI to extract structured requirement data from each URL, then normalizes it into a consistent internal schema and computes a hash for change detection.  
**Nodes involved:**  
- Scrape Certification Requirements  
- Normalize Requirement Data

#### Node: “Scrape Certification Requirements”
- **Type / role:** `n8n-nodes-scrapegraphai.scrapegraphAi` — AI-powered web extraction per URL item.
- **Configuration choices:**
  - **websiteUrl:** set via expression `{{ $json.url }}` (each incoming item provides its own URL).
  - **userPrompt:** instructs extraction and forces strict JSON schema:
    ```text
    {"certification_name":"string","requirement_text":"string","last_updated":"ISO8601"}
    ```
- **Credentials:**
  - Uses `ScrapegraphAI account` (ScrapeGraphAI API credential).
- **Connections:**
  - **Input:** from Prepare Certification URLs
  - **Output:** → Normalize Requirement Data
- **Version-specific notes:** `typeVersion: 1`.
- **Edge cases / failures:**
  - ScrapeGraphAI may return non-parseable JSON if the model fails to follow the schema (despite prompt).
  - Target websites may block scraping (403/CAPTCHA), causing errors or empty outputs.
  - Latency/timeouts when scraping multiple sites.
  - “last_updated” may be missing or not truly ISO8601 despite prompt (causes downstream date parsing issues).

#### Node: “Normalize Requirement Data”
- **Type / role:** `n8n-nodes-base.code` — field normalization + hashing.
- **Configuration choices:**
  - Uses Node.js `crypto` to compute SHA-256 hash over:
    - `certification_name + requirement_text`
  - Renames fields to internal camelCase:
    - `certificationName`
    - `requirementText`
    - `lastUpdated` (forced ISO string via `new Date(...).toISOString()`)
    - `hash`
  - Processes **all incoming items** with `$input.all().map(...)`.
- **Key expressions/variables:**
  - `item.json` as `data`
  - `crypto.createHash('sha256')...`
- **Connections:**
  - **Input:** from Scrape Certification Requirements
  - **Output:** → Aggregate Results
- **Version-specific notes:** `typeVersion: 2`.
- **Edge cases / failures:**
  - If `data.last_updated` is invalid or missing, `new Date(...).toISOString()` may throw `RangeError: Invalid time value`.
  - If scraped output keys differ (e.g., `certificationName` instead of `certification_name`), fields become `undefined`.
  - Hash detects even whitespace changes in text (as intended by the sticky note), but can also trigger on trivial formatting changes.

**Sticky note covering this block:** “Section – Scraping”.

---

### Block 3 — Aggregation & Baseline Retrieval (GitLab + Merge)
**Overview:** Consolidates all normalized items into a single array object, retrieves the prior baseline JSON file from GitLab, and merges both streams into one item for comparison.  
**Nodes involved:**  
- Aggregate Results  
- Get Baseline From GitLab  
- Merge Current & Baseline

#### Node: “Aggregate Results”
- **Type / role:** `n8n-nodes-base.code` — fan-in aggregator.
- **Configuration choices:**
  - Builds a single output item:
    - `currentRequirements: items.map(i => i.json)`
- **Connections:**
  - **Input:** from Normalize Requirement Data (multiple items)
  - **Outputs:**
    - → Get Baseline From GitLab
    - → Merge Current & Baseline (connected to Merge input **index 1**)
- **Version-specific notes:** `typeVersion: 2`.
- **Edge cases / failures:**
  - Ordering: `items.map(...)` preserves incoming order, which can influence change detection because later comparison uses `JSON.stringify`. If scrape order changes between runs, you may get false positives.

#### Node: “Get Baseline From GitLab”
- **Type / role:** `n8n-nodes-base.gitlab` — fetches baseline file from repository.
- **Configuration choices (interpreted from minimal JSON):**
  - **Resource:** `repositoryFile` (intended for “Get file” operation)
  - **Missing in JSON:** project/repo identification, file path, branch, and operation details are not shown and must be configured in n8n UI.
- **Connections:**
  - **Input:** from Aggregate Results
  - **Output:** → Merge Current & Baseline (connected to Merge input **index 0**)
- **Version-specific notes:** `typeVersion: 1`.
- **Edge cases / failures:**
  - Auth errors (invalid token/OAuth, insufficient repo permissions).
  - File not found (first run) unless you handle it—currently not handled here; the node may fail depending on operation settings.
  - GitLab API rate limits/timeouts.

#### Node: “Merge Current & Baseline”
- **Type / role:** `n8n-nodes-base.merge` — combines current and baseline into one merged item context.
- **Configuration choices:**
  - **Mode:** `combine`
  - Uses two inputs:
    - Input 0: baseline from GitLab
    - Input 1: current aggregated data
- **Connections:**
  - **Inputs:** from Get Baseline From GitLab (0) and Aggregate Results (1)
  - **Output:** → Detect Changes
- **Version-specific notes:** `typeVersion: 2`.
- **Edge cases / failures:**
  - If GitLab node errors and produces no item, merge may stall/fail.
  - Combine behavior depends on how many items arrive on each input; here each is expected to be **single-item**.

**Sticky note covering this block:** “Section – Aggregation”.

---

### Block 4 — Decision & Alerting (diff + IF)
**Overview:** Parses the baseline file, compares it with the current scraped array, and routes execution depending on whether any change is detected.  
**Nodes involved:**  
- Detect Changes  
- Requirements Changed?

#### Node: “Detect Changes”
- **Type / role:** `n8n-nodes-base.code` — compute `changed` and a diff payload.
- **Configuration choices:**
  - Assumes two inputs are present (comment says):
    - `Input[0] = current data, Input[1] = baseline file data`
  - Implementation actually does:
    - `const [currentItem, baselineItem] = $input.all();`
    - `currentReq = currentItem.json.currentRequirements;`
  - Baseline parsing:
    - Expects `baselineItem.json.content` to be **base64** of JSON text.
    - Decodes using `Buffer.from(..., 'base64').toString()` then `JSON.parse(...)`.
    - On parse/decode error: fallback to empty array.
  - Change detection:
    - `changed = JSON.stringify(currentReq) !== JSON.stringify(baselineReq)`
  - Output:
    - `changed`
    - `diff`: if changed, returns full `currentReq` (not a minimal diff)
    - `currentRequirements`: current array
- **Connections:**
  - **Input:** from Merge Current & Baseline
  - **Output:** → Requirements Changed?
- **Version-specific notes:** `typeVersion: 2`.
- **Edge cases / failures:**
  - **Input order risk:** Merge output ordering may not match the assumption. If swapped, code may try to read `currentRequirements` from the GitLab item and break (e.g., `Cannot read properties of undefined`). The comment and connection wiring suggest baseline is input 0 and current is input 1 at Merge, but the code assumes the opposite.
  - Base64 expectation: GitLab node must return file content in `content` base64 form; if the GitLab node returns raw text or a different field name, parsing fails and baseline becomes `[]`, causing “changed = true” every time.
  - `JSON.stringify` comparison is order-sensitive; reordering items can produce false changes.

#### Node: “Requirements Changed?”
- **Type / role:** `n8n-nodes-base.if` — conditional routing.
- **Configuration choices:**
  - Boolean condition: `{{ $json.changed }}` equals `true`.
- **Connections:**
  - **True output (index 0):** → Send Rocket.Chat Alert AND → Update GitLab Record (parallel)
  - **False output (index 1):** → Respond to Webhook
- **Version-specific notes:** `typeVersion: 2`.
- **Edge cases / failures:**
  - If `changed` is missing/not boolean, condition may not behave as intended.

**Sticky note covering this block:** “Section – Decision & Alerting”.

---

### Block 5 — Storage & Notification (Rocket.Chat + GitLab write + response)
**Overview:** When changes are detected, posts an alert to Rocket.Chat and commits the new baseline to GitLab; regardless of path, responds to the webhook caller.  
**Nodes involved:**  
- Send Rocket.Chat Alert  
- Update GitLab Record  
- Respond to Webhook

#### Node: “Send Rocket.Chat Alert”
- **Type / role:** `n8n-nodes-base.rocketchat` — sends a chat message to a channel.
- **Configuration choices:**
  - **Channel:** `#cert-alerts`
  - **Text expression:**
    - Concatenates a fixed message with pretty-printed JSON of `$json.diff`:
      - `Certification requirements updated... Changed items: { ... }`
- **Credentials:** Rocket.Chat API credentials must be configured in n8n (not shown in JSON).
- **Connections:**
  - **Input:** from Requirements Changed? (true branch)
  - **Output:** → Respond to Webhook
- **Version-specific notes:** `typeVersion: 1`.
- **Edge cases / failures:**
  - Message size limits: dumping full JSON (possibly large) can exceed Rocket.Chat message limits.
  - Auth/permissions/channel not found errors.
  - If `diff` is large, sending may be slow or rejected.

#### Node: “Update GitLab Record”
- **Type / role:** `n8n-nodes-base.gitlab` — writes/updates the baseline file in the repo.
- **Configuration choices (interpreted from minimal JSON):**
  - **Resource:** `repositoryFile` (intended for “Create/Update file” operation with commit message)
  - Must be configured to write `requirements.json` (as described by sticky notes), on a chosen branch and path.
  - Must map content from `currentRequirements` (likely JSON string) into the file content field.
- **Connections:**
  - **Input:** from Requirements Changed? (true branch)
  - **Output:** → Respond to Webhook
- **Version-specific notes:** `typeVersion: 1`.
- **Edge cases / failures:**
  - Merge conflicts or branch protection rules can block commits.
  - Incorrect encoding (GitLab expects content string; some operations expect base64).
  - Missing permissions for repository write.

#### Node: “Respond to Webhook”
- **Type / role:** `n8n-nodes-base.respondToWebhook` — completes the HTTP request.
- **Configuration choices:**
  - Options empty in JSON; by default it responds with the last input data unless configured otherwise.
- **Connections:**
  - **Inputs:** can be reached from:
    - Requirements Changed? (false branch)
    - Send Rocket.Chat Alert
    - Update GitLab Record
- **Version-specific notes:** `typeVersion: 1`.
- **Edge cases / failures:**
  - Multiple branches converging: depending on execution timing, you may have more than one item reaching this node (from Rocket.Chat and GitLab). Respond-to-webhook expects a single response; n8n generally handles this by responding once, but you should ensure deterministic response behavior (e.g., merge/wait patterns if needed).

**Sticky note covering this block:** “Section – Storage & Notification”.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Documentation / operator guidance |  |  | ## How it works … (full note content describing end-to-end logic and setup steps) |
| Section – Trigger & Config | Sticky Note | Documentation for trigger/config block |  |  | ## Trigger & URL Configuration … |
| Section – Scraping | Sticky Note | Documentation for scraping block |  |  | ## AI-Powered Scraping … |
| Section – Aggregation | Sticky Note | Documentation for aggregation/baseline block |  |  | ## Aggregation & Baseline Retrieval … |
| Section – Decision & Alerting | Sticky Note | Documentation for decision block |  |  | ## Change Detection & Routing … |
| Section – Storage & Notification | Sticky Note | Documentation for notification/storage block |  |  | ## Storage & Notifications … |
| Incoming Webhook | Webhook | Entry point (POST trigger) |  | Prepare Certification URLs | Section – Trigger & Config: Trigger & URL Configuration… |
| Prepare Certification URLs | Code | Emit one item per monitored URL | Incoming Webhook | Scrape Certification Requirements | Section – Trigger & Config: Trigger & URL Configuration… |
| Scrape Certification Requirements | ScrapeGraphAI | Extract structured requirement data from each URL | Prepare Certification URLs | Normalize Requirement Data | Section – Scraping: AI-Powered Scraping… |
| Normalize Requirement Data | Code | Normalize fields, ISO date, compute hash | Scrape Certification Requirements | Aggregate Results | Section – Scraping: AI-Powered Scraping… |
| Aggregate Results | Code | Aggregate all items to `currentRequirements` array | Normalize Requirement Data | Get Baseline From GitLab; Merge Current & Baseline | Section – Aggregation: Aggregation & Baseline Retrieval… |
| Get Baseline From GitLab | GitLab | Read `requirements.json` baseline from repo | Aggregate Results | Merge Current & Baseline | Section – Aggregation: Aggregation & Baseline Retrieval… |
| Merge Current & Baseline | Merge | Combine baseline + current into one flow for comparison | Get Baseline From GitLab; Aggregate Results | Detect Changes | Section – Aggregation: Aggregation & Baseline Retrieval… |
| Detect Changes | Code | Parse baseline, compare, output `changed` + `diff` | Merge Current & Baseline | Requirements Changed? | Section – Decision & Alerting: Change Detection & Routing… |
| Requirements Changed? | IF | Branching based on `changed` boolean | Detect Changes | (true) Send Rocket.Chat Alert, Update GitLab Record; (false) Respond to Webhook | Section – Decision & Alerting: Change Detection & Routing… |
| Send Rocket.Chat Alert | Rocket.Chat | Notify channel with diff payload | Requirements Changed? (true) | Respond to Webhook | Section – Storage & Notification: Storage & Notifications… |
| Update GitLab Record | GitLab | Commit updated baseline file | Requirements Changed? (true) | Respond to Webhook | Section – Storage & Notification: Storage & Notifications… |
| Respond to Webhook | Respond to Webhook | Return HTTP response to the original webhook call | Requirements Changed? (false); Send Rocket.Chat Alert; Update GitLab Record |  | Section – Storage & Notification: Storage & Notifications… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add trigger node: Webhook**
   - Node type: **Webhook**
   - **HTTP Method:** POST  
   - **Path:** `certification-requirements`
   - **Response mode:** “Using ‘Respond to Webhook’ node” (responseMode = responseNode)
3. **Add Code node: “Prepare Certification URLs”**
   - Node type: **Code**
   - Paste/implement code that returns one item per URL:
     - Each item must be `{ json: { url: 'https://...' } }`
   - Connect: **Incoming Webhook → Prepare Certification URLs**
4. **Add ScrapeGraphAI node: “Scrape Certification Requirements”**
   - Node type: **ScrapeGraphAI**
   - **Credentials:** create/select ScrapeGraphAI API credentials in n8n.
   - **Website URL:** expression `{{ $json.url }}`
   - **Prompt:** instruct extraction and strict JSON schema:
     - Must return `certification_name`, `requirement_text`, `last_updated`
   - Connect: **Prepare Certification URLs → Scrape Certification Requirements**
5. **Add Code node: “Normalize Requirement Data”**
   - Node type: **Code**
   - Implement:
     - Rename fields to `certificationName`, `requirementText`, `lastUpdated`
     - Convert date to ISO with `new Date(...).toISOString()`
     - Add SHA-256 hash using Node `crypto`
   - Connect: **Scrape Certification Requirements → Normalize Requirement Data**
6. **Add Code node: “Aggregate Results”**
   - Node type: **Code**
   - Create one output item:
     - `currentRequirements: items.map(i => i.json)`
   - Connect: **Normalize Requirement Data → Aggregate Results**
7. **Add GitLab node: “Get Baseline From GitLab”**
   - Node type: **GitLab**
   - **Credentials:** configure GitLab access (token/OAuth) with repo read permissions.
   - **Resource:** Repository File
   - Configure to **Get** a file (baseline), typically:
     - **Project:** your GitLab project
     - **Branch:** e.g. `main`
     - **File path:** e.g. `requirements.json`
   - Connect: **Aggregate Results → Get Baseline From GitLab**
8. **Add Merge node: “Merge Current & Baseline”**
   - Node type: **Merge**
   - **Mode:** Combine
   - Connect:
     - **Get Baseline From GitLab → Merge** (Input 0)
     - **Aggregate Results → Merge** (Input 1)
9. **Add Code node: “Detect Changes”**
   - Node type: **Code**
   - Implement logic to:
     - Read `currentRequirements` from the aggregated item
     - Decode and parse GitLab file content (often base64 in `content`)
     - Compare current vs baseline
     - Output `{ changed, diff, currentRequirements }`
   - Connect: **Merge Current & Baseline → Detect Changes**
   - Important when rebuilding: ensure your code matches the **actual input ordering** you produced from the Merge node.
10. **Add IF node: “Requirements Changed?”**
    - Node type: **IF**
    - Condition:
      - Boolean: `{{ $json.changed }}` **equals** `true`
    - Connect: **Detect Changes → Requirements Changed?**
11. **Add Rocket.Chat node: “Send Rocket.Chat Alert”**
    - Node type: **Rocket.Chat**
    - **Credentials:** configure Rocket.Chat API access in n8n.
    - **Channel:** `#cert-alerts` (or your target)
    - **Message text:** include `{{ JSON.stringify($json.diff, null, 2) }}` or a summarized diff.
    - Connect: **Requirements Changed? (true) → Send Rocket.Chat Alert**
12. **Add GitLab node: “Update GitLab Record”**
    - Node type: **GitLab**
    - **Credentials:** same or another GitLab credential with **write** permission.
    - **Resource:** Repository File
    - Configure to **Create/Update** file:
      - Same **Project**, **Branch**, **File path** (`requirements.json`)
      - **Content:** JSON string of `currentRequirements` (e.g. `{{ JSON.stringify($json.currentRequirements, null, 2) }}`)
      - **Commit message:** e.g. “Update certification requirements baseline”
    - Connect: **Requirements Changed? (true) → Update GitLab Record**
13. **Add Respond to Webhook node: “Respond to Webhook”**
    - Node type: **Respond to Webhook**
    - Configure response (optional but recommended):
      - Status code 200
      - Body could include `{ changed, count, timestamp }` etc.
    - Connect:
      - **Requirements Changed? (false) → Respond to Webhook**
      - **Send Rocket.Chat Alert → Respond to Webhook**
      - **Update GitLab Record → Respond to Webhook**
14. **Add sticky notes (optional)** to document each section (as in the provided workflow).
15. **Test**
    - POST to `/webhook/certification-requirements`
    - Verify scraping output, baseline retrieval, and that:
      - First run typically triggers “changed” (baseline missing/empty)
      - Subsequent run triggers only when content differs
16. **Enable the workflow** once credentials, GitLab paths/branch, and Rocket.Chat channel are correct.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow lets certification-holding professionals subscribe to live updates… compares to the last stored copy in a GitLab repo… posts the diff… commits… returns 200.” | Sticky note: **Workflow Overview** (high-level behavior) |
| Setup steps: add ScrapeGraphAI creds; fill URLs in Code node; configure GitLab project/path/branch; Rocket.Chat creds + channel; deploy webhook; enable workflow | Sticky note: **Workflow Overview** |
| Trigger is decoupled: any POST can fire it; Code node emits one item per authority URL | Sticky note: **Section – Trigger & Config** |
| ScrapeGraphAI prompt forces strict JSON schema to reduce breakage when HTML changes | Sticky note: **Section – Scraping** |
| GitLab baseline provides audit history and rollbacks | Sticky note: **Section – Aggregation** |
| IF prevents unnecessary commits and chat noise | Sticky note: **Section – Decision & Alerting** |
| Alert + GitLab update run in parallel, then respond | Sticky note: **Section – Storage & Notification** |