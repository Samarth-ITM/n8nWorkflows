Detect fake product reviews with OpenAI and send alerts to Slack via Airtable

https://n8nworkflows.xyz/workflows/detect-fake-product-reviews-with-openai-and-send-alerts-to-slack-via-airtable-12129


# Detect fake product reviews with OpenAI and send alerts to Slack via Airtable

## 1. Workflow Overview

**Purpose:** This workflow receives a payload of products (each with a scraper URL), fetches the latest reviews for each product, deduplicates reviews using a generated hash stored in Airtable, analyzes new reviews with OpenAI for “fake/manipulated” likelihood, writes the AI results back to Airtable, and sends a Slack alert when the fake score is above a threshold.

**Target use cases**
- Trust & Safety / moderation teams monitoring marketplaces for suspicious reviews
- Automated enrichment of review datasets with AI-based risk classification
- Near-real-time alerting when suspicious reviews appear

### 1.1 Input Reception & Product Fan-out
Receives a webhook payload, splits the `products` array into individual product items, and iterates products one-by-one.

### 1.2 Review Fetching & Normalization to Per-Review Items
For each product, calls an external review-scraper URL, verifies reviews exist, and expands `reviews[]` into one item per review.

### 1.3 Review Field Cleanup, Hashing & Batch Iteration
Standardizes review fields, then generates a deterministic hash from review text + reviewer + date to support deduplication. Reviews are processed sequentially via batching.

### 1.4 Deduplication Against Airtable
Searches Airtable for an existing record with the same hash, normalizes “empty result” edge cases, and branches based on whether the review is new.

### 1.5 Storage, AI Analysis, Persistence of AI Output
Creates an Airtable record for a new review, sends it to OpenAI, parses the AI JSON response, and updates the Airtable record with `fake_score`, `risk_level`, and `reason`.

### 1.6 Thresholding & Slack Alerting
If `fake_score >= 60`, sends a detailed Slack moderation alert; otherwise continues without alerting.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception & Product Fan-out
**Overview:** Accepts inbound product payloads via webhook and converts the payload’s product list into individual product items for downstream processing.

**Nodes involved**
- Webhook – Receive Product Payload
- Extract products
- Process Each Product

#### Node: Webhook – Receive Product Payload
- **Type / role:** `Webhook` (entry point) to receive product payloads.
- **Config choices:**
  - Method: `POST`
  - Path: `product_data` (endpoint becomes `/webhook/.../product_data` depending on n8n mode)
- **Key data expectations:** Incoming JSON should include `body.products` (array).
- **Outputs:** Sends the full webhook payload forward.
- **Connections:** → `Extract products`
- **Edge cases / failures:**
  - Missing `body.products` causes `Extract products` to produce no items (or error depending on node behavior).
  - If webhook receives non-JSON or wrong structure, downstream expressions may fail.

#### Node: Extract products
- **Type / role:** `Split Out` to fan out `body.products` into separate items.
- **Config choices:**
  - Field to split out: `body.products`
- **Input:** Webhook payload with `body.products`.
- **Output:** One item per product, each item becomes one product object from the array.
- **Connections:** → `Process Each Product`
- **Edge cases / failures:**
  - If `body.products` isn’t an array, split may error or yield 0 items.

#### Node: Process Each Product
- **Type / role:** `Split In Batches` (batch size 1) to process products sequentially.
- **Config choices:**
  - `batchSize: 1`
  - `reset: false` (does not reset automatically between executions; typical for looping patterns)
- **Connections:** → `Fetch Product Reviews`
- **Edge cases / failures:**
  - If upstream yields 0 products, nothing runs.
  - Misuse of batch looping can cause unexpected behavior; here it’s used as a simple sequential iterator.

**Sticky note coverage**
- “## Product Input & Setup …” applies to: Webhook – Receive Product Payload, Extract products, Process Each Product

---

### Block 1.2 — Review Fetching & Normalization to Per-Review Items
**Overview:** Fetches reviews from a provided URL per product, checks if reviews exist, then expands reviews into individual items.

**Nodes involved**
- Fetch Product Reviews
- IF – Has Reviews?
- Expand reviews[] to items
- Split review in batches

#### Node: Fetch Product Reviews
- **Type / role:** `HTTP Request` to call the scraper API.
- **Config choices:**
  - URL expression: `={{ $json.url }}`
- **Input requirement:** Each product item must have `url` (scraper endpoint returning review JSON).
- **Connections:** → `IF – Has Reviews?`
- **Edge cases / failures:**
  - 4xx/5xx from scraper, network timeouts.
  - Response shape mismatch (no `reviews` array).
  - If scraper returns non-JSON, n8n may fail to parse depending on defaults.

#### Node: IF – Has Reviews?
- **Type / role:** `IF` guard to ensure `reviews` exists and is non-empty.
- **Condition (boolean):**
  - `={{!!$json["reviews"] && $json["reviews"].length > 0}}` equals `true`
- **Connections:** (true) → `Expand reviews[] to items`
- **Edge cases / failures:**
  - If `reviews` is not an array, `.length` may throw; the expression tries to guard with `!!$json["reviews"]`, but a non-array object could still have `.length` undefined (safe) or behave unexpectedly.

#### Node: Expand reviews[] to items
- **Type / role:** `Code` node to transform one product-with-reviews item into many per-review items.
- **Key logic:**
  - `const reviews = $json.reviews || [];`
  - Returns `reviews.map(...)` and merges product fields:
    - Adds `product_id`, `marketplace`, and spreads each review object `...r`
- **Input:** Item containing `reviews` plus `product_id` / `marketplace`.
- **Output:** One item per review, with combined fields.
- **Connections:** → `Split review in batches`
- **Edge cases / failures:**
  - If `reviews` is huge, can create many items (memory/time).
  - If reviews contain keys that overwrite `product_id`/`marketplace`, spread order matters (here `product_id` and `marketplace` come before `...r`, so `r` can overwrite them).

#### Node: Split review in batches
- **Type / role:** `Split In Batches` for sequential review processing and loopback control.
- **Config choices:** TypeVersion 3, default options.
- **Connections:**
  - Output **1** (batch items) → (in this workflow) used implicitly via connections:
    - Goes to `Prepare Review Fields` on **output index 1** (see connections)
  - Output **0** loops to next steps (used by multiple nodes to continue iteration):
    - From `Check Suspicious Score Threshold` false branch → this node
    - From `Send Moderation Alert` → this node
    - From `Is New Review?` false branch → this node
- **Edge cases / failures:**
  - If loop connections are miswired, you can create infinite loops or stop early.
  - If an item errors mid-loop, remaining items may not process depending on error settings.

**Sticky note coverage**
- “## Fetch Reviews …” applies to: Fetch Product Reviews, IF – Has Reviews?, Expand reviews[] to items
- “## Clean & Hash Review …” applies to: Split review in batches, Prepare Review Fields, Generate Review Hash1 (next block)

---

### Block 1.3 — Review Field Cleanup, Hashing & Batch Iteration
**Overview:** Standardizes field names used later, then generates a stable hash used for deduplication in Airtable.

**Nodes involved**
- Prepare Review Fields
- Generate Review Hash1

#### Node: Prepare Review Fields
- **Type / role:** `Set` to normalize/rename fields into a consistent schema for hashing + Airtable.
- **Key mappings:**
  - `review_id = {{$json.review_id}}`
  - `text = {{$json.title}}` (note: uses title, not full review text)
  - `rating = {{$json["rating"]}}`
  - `date = {{$json.review_date}}`
  - `reviewer_id = {{$json.review_id}}` (likely a mistake: reviewer_id set to review_id)
  - `reviewer_profile = {{$json["reviewer"]?.profile_url || $json["reviewer_profile"]}}`
  - `marketplace = {{$json["marketplace"]}}`
  - `product_id = {{$json["product_id"] || $json["productId"]}}`
- **Input:** Per-review items from the review expansion.
- **Output:** Same item plus normalized fields (only string values configured; other incoming fields still present unless “Keep Only Set” is enabled—here it is not shown as enabled).
- **Connections:** → `Generate Review Hash1`
- **Edge cases / failures:**
  - If `title` is missing, `text` becomes empty → weaker hash uniqueness.
  - `reviewer_id` mapping appears incorrect; this reduces dedupe quality (different reviewers could collide if other fields similar).
  - `reviewer` object may not exist; optional chaining handles it.

#### Node: Generate Review Hash1
- **Type / role:** `Function` to compute a deterministic hash.
- **Key logic:**
  - Builds input string: `${text}||${reviewer}||${date}`
  - Hash function: djb2-like (seed 5381, multiply 33, XOR char)
  - Outputs `hash` as hex string
- **Inputs used:** `$json["text"]`, `$json["reviewer_id"]`, `$json["date"]`
- **Output:** Same item plus `hash`
- **Connections:** → `List Bases` (which then leads into Airtable search)
- **Edge cases / failures:**
  - Empty/missing components cause hash collisions (e.g., missing date).
  - Hash is not cryptographic; collisions are possible at scale.
  - Any change in normalization (e.g., whitespace) changes hash, impacting dedupe consistency.

---

### Block 1.4 — Deduplication Against Airtable
**Overview:** Looks up the hash in Airtable to decide whether to process/store/analyze the review again.

**Nodes involved**
- List Bases
- Search Records by Hash
- Normalize Airtable Result
- Is New Review?

#### Node: List Bases
- **Type / role:** `Airtable` (resource: base). Used here as a connectivity/credential check and to chain execution.
- **Config choices:** `resource: base` with default listing.
- **Credentials:** Airtable Personal Access Token (airtableTokenApi).
- **Connections:** → `Search Records by Hash`
- **Edge cases / failures:**
  - If Airtable credentials invalid/scopes missing, the workflow fails here.
  - This node is not strictly required for dedupe; it adds an extra API call per review.

#### Node: Search Records by Hash
- **Type / role:** `Airtable` search operation in the `reviews` table.
- **Config choices:**
  - Base: “Fake Review Detector” (`appipUxLpxPEKRXrJ`)
  - Table: “reviews” (`tbl14GPDbEqhOfC4f`)
  - Filter formula:
    - `={hash} = "{{ $('Generate Review Hash1').item.json.hash }}"`
- **Important expression behavior:**
  - Pulls hash specifically from the `Generate Review Hash1` node item context.
- **alwaysOutputData:** `true` (ensures output even when no match; may output empty items)
- **Connections:** → `Normalize Airtable Result`
- **Edge cases / failures:**
  - If `{hash}` field name in Airtable differs (case/spacing), filter returns nothing.
  - If formula quoting breaks (hash contains `"`), formula can become invalid (unlikely with hex hash).
  - Airtable rate limits if volume is high.

#### Node: Normalize Airtable Result
- **Type / role:** `Code` node to standardize “record exists?” output.
- **Key logic:**
  - Pulls the original review from `Generate Review Hash1`: `const review = $node["Generate Review Hash1"].json;`
  - Reads incoming Airtable search items via `$items()`
  - Outputs one item:
    - `{ exists: 0, ...review }` if no records or empty object result
    - `{ exists: 1, ...review }` if record exists
- **Connections:** → `Is New Review?`
- **Edge cases / failures:**
  - If Airtable returns multiple items, still treated as “exists: 1” (OK).
  - Relies on `$items()` in Code node; behavior differs if executed in different modes, but generally stable in n8n.

#### Node: Is New Review?
- **Type / role:** `IF` to branch on existence.
- **Condition:** `$json.exists == 0`
- **Connections:**
  - True (new) → `Create Review Record`
  - False (duplicate) → `Split review in batches` (continue loop)
- **Edge cases / failures:**
  - If `exists` missing or non-numeric, strict type validation can cause condition mis-evaluation.

**Sticky note coverage**
- “## Check for Duplicate …” applies to: List Bases, Search Records by Hash, Normalize Airtable Result, Is New Review?

---

### Block 1.5 — Storage, AI Analysis, Persistence of AI Output
**Overview:** Stores a new review record in Airtable, submits it to OpenAI for analysis, parses AI JSON, then updates the Airtable record with the AI results.

**Nodes involved**
- Create Review Record
- AI Fake Review Analysis
- Parse AI Respons
- Update Review Record

#### Node: Create Review Record
- **Type / role:** `Airtable` create operation (insert a new review).
- **Config choices:**
  - Base/Table: same as above
  - Field mappings (selected):
    - `Date = {{$json.review_date}}`
    - `hash = {{$json.hash}}`
    - `rating = {{$json.rating}}`
    - `review_id = {{$json.review_id}}`
    - `product_id = {{$json.product_id}}`
    - `marketplace = {{$json.marketplace}}`
    - `profile_url = {{$json.reviewer_profile}}`
    - `review_text = {{$json.review_text}}`
    - `review_title = {{$json.title}}`
    - `reviewer_name = {{$json.reviewer_name}}`
    - `reviewer_profile = {{$json.reviewer_profile}}`
    - `verified_purchase = {{$json.verified_purchase}}`
- **Connections:** → `AI Fake Review Analysis`
- **Edge cases / failures:**
  - Field type mismatch (e.g., `Date` not valid datetime string).
  - Missing required fields in Airtable schema (if you configured them as required).
  - If upstream didn’t carry `review_text`/`title` etc. (depends on scraper response), Airtable will store blanks.

#### Node: AI Fake Review Analysis
- **Type / role:** `OpenAI` Chat to classify review authenticity.
- **Model:** `o4-mini`
- **Prompt structure:**
  - System-style instruction: “You are a Trust & Safety analyst…”
  - User message includes templated review details from Airtable-created record:
    - `rating: {{$json.fields.rating}}`, `review_title`, `review_text`, `reviewer_name`, `verified_purchase`, `marketplace`, `review_date: {{$json.fields.Date}}`
  - Requires **JSON-only** response:
    ```json
    { "fake_score": <0-100>, "risk_level": "<low|medium|high>", "reason": "..." }
    ```
- **Simplify output:** `false` (so output includes `choices[0].message.content`)
- **Connections:** → `Parse AI Respons`
- **Edge cases / failures:**
  - OpenAI auth errors, quota, rate limits.
  - Model returns non-JSON or extra text despite instruction.
  - Prompt references `$json.fields...` which depends on Airtable node output shape (Create returns `fields`).

#### Node: Parse AI Respons
- **Type / role:** `Code` node to parse the AI response content into structured JSON.
- **Key logic:**
  - `const content = $json.choices[0].message.content;`
  - `JSON.parse(content)` with fallback:
    - `{ fake_score: null, risk_level: "error", reason: "Invalid JSON returned by AI" }`
- **Output:** A single item containing `fake_score`, `risk_level`, `reason`.
- **Connections:** → `Update Review Record`
- **Edge cases / failures:**
  - If OpenAI node output format changes (or simplifyOutput true), `choices[0]...` path breaks.
  - If the model returns JSON with trailing commas/comments, parse fails.

#### Node: Update Review Record
- **Type / role:** `Airtable` update operation to enrich the stored review with AI results.
- **Config choices:**
  - Matching/update keyed by `hash` (`matchingColumns: ["hash"]`)
  - Fields updated:
    - `hash = {{ $('Create Review Record').item.json.fields.hash }}`
    - `reason = {{$json.reason}}`
    - `fake_score = {{$json.fake_score}}`
    - `risk_level = {{$json.risk_level}}`
- **Important behavior:** Uses hash from the **Create Review Record** node, while AI parsed fields come from current item.
- **Connections:** → `Check Suspicious Score Threshold`
- **Edge cases / failures:**
  - If Airtable table has multiple records with same hash (shouldn’t happen), update could affect unexpected record(s) depending on Airtable node behavior.
  - If `fake_score` is `null` (parse failed), later numeric threshold check may fail strict validation.

**Sticky note coverage**
- “## Store & Analyze Review …” applies to: Create Review Record, AI Fake Review Analysis, Parse AI Respons

---

### Block 1.6 — Thresholding & Slack Alerting
**Overview:** Evaluates AI fake score against a threshold and sends an alert to Slack for high-risk reviews; in all cases, continues the batch loop.

**Nodes involved**
- Check Suspicious Score Threshold
- Send Moderation Alert

#### Node: Check Suspicious Score Threshold
- **Type / role:** `IF` to escalate only suspicious reviews.
- **Condition:** `{{$json.fields.fake_score}} >= 60`
  - Note: This node expects the incoming item to include `fields.fake_score` (Airtable update output usually includes `fields`).
- **Connections:**
  - True → `Send Moderation Alert`
  - False → `Split review in batches` (continue processing next review)
- **Edge cases / failures:**
  - If `fake_score` missing/null/string, strict type validation can cause errors or false negatives.
  - If Update node output doesn’t include `fields` (depending on node settings/version), the expression fails.

#### Node: Send Moderation Alert
- **Type / role:** `Slack` node to post a message to a channel.
- **Config choices:**
  - Target: Channel (ID `C09S57E2JQ2`, cached name “n8n”)
  - Message template includes product, reviewer, rating, fake score, risk level, review text, AI reason, hash, and review ID.
- **Connections:** → `Split review in batches` (continue loop)
- **Edge cases / failures:**
  - Slack auth/scopes missing (`chat:write`).
  - Message length limits (very long reviews may be truncated or fail).
  - Uses fields like `$json.fields.reason`—requires Slack node input to be Airtable-shaped output.

**Sticky note coverage**
- “## Update Review Status …” applies to: Update Review Record, Check Suspicious Score Threshold
- “## Send Moderation Alert …” applies to: Send Moderation Alert

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook – Receive Product Payload | Webhook | Entry point: receive product payload | — | Extract products | ## Product Input & Setup<br>Receives product data from the webhook, extracts individual product entries, and prepares them for processing. This ensures each product request is handled independently and structured correctly before moving into the review-fetching stage. |
| Extract products | Split Out | Split `body.products` array into items | Webhook – Receive Product Payload | Process Each Product | ## Product Input & Setup<br>Receives product data from the webhook, extracts individual product entries, and prepares them for processing. This ensures each product request is handled independently and structured correctly before moving into the review-fetching stage. |
| Process Each Product | Split In Batches | Iterate products sequentially | Extract products | Fetch Product Reviews | ## Product Input & Setup<br>Receives product data from the webhook, extracts individual product entries, and prepares them for processing. This ensures each product request is handled independently and structured correctly before moving into the review-fetching stage. |
| Fetch Product Reviews | HTTP Request | Call scraper URL to retrieve reviews | Process Each Product | IF – Has Reviews? | ## Fetch Reviews<br>Fetches reviews from the external scraper API, checks whether valid review data exists, and expands the results into individual review items. This provides a clean, per-review format ready for analysis and normalization. |
| IF – Has Reviews? | IF | Guard: ensure `reviews[]` exists | Fetch Product Reviews | Expand reviews[] to items | ## Fetch Reviews<br>Fetches reviews from the external scraper API, checks whether valid review data exists, and expands the results into individual review items. This provides a clean, per-review format ready for analysis and normalization. |
| Expand reviews[] to items | Code | Convert reviews array to per-review items | IF – Has Reviews? | Split review in batches | ## Fetch Reviews<br>Fetches reviews from the external scraper API, checks whether valid review data exists, and expands the results into individual review items. This provides a clean, per-review format ready for analysis and normalization. |
| Split review in batches | Split In Batches | Iterate reviews sequentially / loop control | Expand reviews[] to items; Is New Review? (false); Send Moderation Alert; Check Suspicious Score Threshold (false) | Prepare Review Fields (output 1) | ## Clean & Hash Review<br>Breaks reviews into single units, standardizes fields like reviewer, rating, and date, and generates a unique hash by combining key attributes. The hash supports consistent deduplication and prevents repeated processing of the same review. |
| Prepare Review Fields | Set | Normalize review fields for hashing/dedupe | Split review in batches (output 1) | Generate Review Hash1 | ## Clean & Hash Review<br>Breaks reviews into single units, standardizes fields like reviewer, rating, and date, and generates a unique hash by combining key attributes. The hash supports consistent deduplication and prevents repeated processing of the same review. |
| Generate Review Hash1 | Function | Compute deterministic review hash | Prepare Review Fields | List Bases | ## Clean & Hash Review<br>Breaks reviews into single units, standardizes fields like reviewer, rating, and date, and generates a unique hash by combining key attributes. The hash supports consistent deduplication and prevents repeated processing of the same review. |
| List Bases | Airtable | Credential/connectivity step; list bases | Generate Review Hash1 | Search Records by Hash | ## Check for Duplicate<br>Searches Airtable for existing records using the generated hash and normalizes the response to avoid empty-object issues. Determines whether the review already exists or must proceed as a new entry. |
| Search Records by Hash | Airtable (search) | Deduplication lookup by hash | List Bases | Normalize Airtable Result | ## Check for Duplicate<br>Searches Airtable for existing records using the generated hash and normalizes the response to avoid empty-object issues. Determines whether the review already exists or must proceed as a new entry. |
| Normalize Airtable Result | Code | Normalize “no match”/empty object to `exists=0` | Search Records by Hash | Is New Review? | ## Check for Duplicate<br>Searches Airtable for existing records using the generated hash and normalizes the response to avoid empty-object issues. Determines whether the review already exists or must proceed as a new entry. |
| Is New Review? | IF | Branch new vs duplicate | Normalize Airtable Result | Create Review Record (true); Split review in batches (false) | ## Check for Duplicate<br>Searches Airtable for existing records using the generated hash and normalizes the response to avoid empty-object issues. Determines whether the review already exists or must proceed as a new entry. |
| Create Review Record | Airtable (create) | Store new review row | Is New Review? (true) | AI Fake Review Analysis | ## Store & Analyze Review<br>Stores new reviews in Airtable and submits them to the AI model for fake-review analysis. The AI response is cleaned and structured into usable fields like fake score, risk level, and explanation. |
| AI Fake Review Analysis | OpenAI (chat) | AI scoring/classification | Create Review Record | Parse AI Respons | ## Store & Analyze Review<br>Stores new reviews in Airtable and submits them to the AI model for fake-review analysis. The AI response is cleaned and structured into usable fields like fake score, risk level, and explanation. |
| Parse AI Respons | Code | Parse JSON-only output from AI | AI Fake Review Analysis | Update Review Record | ## Store & Analyze Review<br>Stores new reviews in Airtable and submits them to the AI model for fake-review analysis. The AI response is cleaned and structured into usable fields like fake score, risk level, and explanation. |
| Update Review Record | Airtable (update) | Write AI results into Airtable | Parse AI Respons | Check Suspicious Score Threshold | ## Update Review Status<br>Updates the Airtable record with AI analysis, computes the risk threshold, and determines whether the review qualifies as suspicious. Only high-risk reviews are escalated for further moderation. |
| Check Suspicious Score Threshold | IF | Threshold fake_score >= 60 | Update Review Record | Send Moderation Alert (true); Split review in batches (false) | ## Update Review Status<br>Updates the Airtable record with AI analysis, computes the risk threshold, and determines whether the review qualifies as suspicious. Only high-risk reviews are escalated for further moderation. |
| Send Moderation Alert | Slack | Send Slack alert with details | Check Suspicious Score Threshold (true) | Split review in batches | ## Send Moderation Alert<br>Sends a detailed Slack alert when a review is flagged as suspicious, including product details, fake score, reviewer information, and reasoning. This enables moderators to quickly identify and act on potentially fake or harmful reviews. |
| Sticky Note | Sticky Note | Documentation | — | — | ## Product Input & Setup<br>Receives product data from the webhook, extracts individual product entries, and prepares them for processing. This ensures each product request is handled independently and structured correctly before moving into the review-fetching stage. |
| Sticky Note1 | Sticky Note | Documentation | — | — | ## Fetch Reviews<br>Fetches reviews from the external scraper API, checks whether valid review data exists, and expands the results into individual review items. This provides a clean, per-review format ready for analysis and normalization. |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## Clean & Hash Review<br>Breaks reviews into single units, standardizes fields like reviewer, rating, and date, and generates a unique hash by combining key attributes. The hash supports consistent deduplication and prevents repeated processing of the same review. |
| Sticky Note3 | Sticky Note | Documentation | — | — | ## Check for Duplicate<br>Searches Airtable for existing records using the generated hash and normalizes the response to avoid empty-object issues. Determines whether the review already exists or must proceed as a new entry. |
| Sticky Note4 | Sticky Note | Documentation | — | — | ## Store & Analyze Review<br>Stores new reviews in Airtable and submits them to the AI model for fake-review analysis. The AI response is cleaned and structured into usable fields like fake score, risk level, and explanation. |
| Sticky Note5 | Sticky Note | Documentation | — | — | ## Update Review Status<br>Updates the Airtable record with AI analysis, computes the risk threshold, and determines whether the review qualifies as suspicious. Only high-risk reviews are escalated for further moderation. |
| Sticky Note6 | Sticky Note | Documentation | — | — | ## Send Moderation Alert<br>Sends a detailed Slack alert when a review is flagged as suspicious, including product details, fake score, reviewer information, and reasoning. This enables moderators to quickly identify and act on potentially fake or harmful reviews. |
| Sticky Note7 | Sticky Note | Documentation | — | — | ## How It Works<br>This workflow helps you automatically check whether product reviews might be fake… (includes setup steps and operational description). |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Webhook node** named **“Webhook – Receive Product Payload”**
   - Method: `POST`
   - Path: `product_data`
   - Save the node to generate a production/test webhook URL.
3. **Add “Split Out” node** named **“Extract products”**
   - Field to split out: `body.products`
   - Connect: Webhook → Extract products
4. **Add “Split In Batches” node** named **“Process Each Product”**
   - Batch size: `1`
   - Options: `reset = false`
   - Connect: Extract products → Process Each Product
5. **Add “HTTP Request” node** named **“Fetch Product Reviews”**
   - URL: `={{ $json.url }}`
   - Connect: Process Each Product → Fetch Product Reviews
   - Ensure your upstream product items contain a `url` field.
6. **Add “IF” node** named **“IF – Has Reviews?”**
   - Condition (boolean): `={{!!$json["reviews"] && $json["reviews"].length > 0}}` equals `true`
   - Connect: Fetch Product Reviews → IF – Has Reviews? (main)
7. **Add “Code” node** named **“Expand reviews[] to items”**
   - Paste code:
     - Map `$json.reviews` to items and include `product_id`, `marketplace`.
   - Connect: IF (true) → Expand reviews[] to items
8. **Add “Split In Batches” node** named **“Split review in batches”**
   - Batch size: `1` (default in v3; set explicitly if needed)
   - Connect: Expand reviews[] to items → Split review in batches
   - You will later connect loop-backs into this node.
9. **Add “Set” node** named **“Prepare Review Fields”**
   - Add fields (as strings) matching:
     - `review_id`, `text`, `rating`, `date`, `reviewer_id`, `reviewer_profile`, `marketplace`, `product_id`
   - Use the same expressions as in the workflow (notably `reviewer_profile` with optional chaining).
   - Connect: Split review in batches **(output 1)** → Prepare Review Fields
10. **Add “Function” node** named **“Generate Review Hash1”**
    - Paste the hash function (djb2-like) and generate `hash` from `text`, `reviewer_id`, `date`.
    - Connect: Prepare Review Fields → Generate Review Hash1
11. **Configure Airtable credentials**
    - Create Airtable Personal Access Token credential in n8n (with read/write access to the base/table).
12. **Add Airtable node** named **“List Bases”**
    - Resource: `base` (list)
    - Select Airtable credentials
    - Connect: Generate Review Hash1 → List Bases
13. **Add Airtable node** named **“Search Records by Hash”**
    - Operation: `search`
    - Base: select your base (e.g., “Fake Review Detector”)
    - Table: `reviews`
    - Filter formula:
      - `={hash} = "{{ $('Generate Review Hash1').item.json.hash }}"`
    - Enable **Always Output Data** (so “no results” still flows downstream)
    - Connect: List Bases → Search Records by Hash
14. **Add “Code” node** named **“Normalize Airtable Result”**
    - Paste code that outputs `{exists:0|1, ...review}` and handles empty object case.
    - Connect: Search Records by Hash → Normalize Airtable Result
15. **Add “IF” node** named **“Is New Review?”**
    - Condition: `exists` equals `0` (number compare)
    - Connect: Normalize Airtable Result → Is New Review?
16. **Add Airtable node** named **“Create Review Record”**
    - Operation: `create`
    - Base/Table: same reviews table
    - Map fields (Date, hash, rating, review_id, product_id, marketplace, review_text, review_title, reviewer_name, reviewer_profile, verified_purchase, etc.) from the incoming JSON.
    - Connect: Is New Review? (true) → Create Review Record
17. **Configure OpenAI credentials**
    - Add OpenAI API credential in n8n.
18. **Add OpenAI node** named **“AI Fake Review Analysis”**
    - Resource: `chat`
    - Model: `o4-mini`
    - Prompt messages:
      - Instruction message defining role (Trust & Safety analyst)
      - Message including `{{$json.fields...}}` values from Airtable create output and requiring JSON-only response with `fake_score`, `risk_level`, `reason`
    - Set **Simplify Output** = `false` (to match parsing logic)
    - Connect: Create Review Record → AI Fake Review Analysis
19. **Add “Code” node** named **“Parse AI Respons”**
    - Parse `choices[0].message.content` as JSON with fallback to `{risk_level:"error"...}`
    - Connect: AI Fake Review Analysis → Parse AI Respons
20. **Add Airtable node** named **“Update Review Record”**
    - Operation: `update`
    - Matching column: `hash`
    - Set:
      - `hash = {{ $('Create Review Record').item.json.fields.hash }}`
      - `fake_score = {{$json.fake_score}}`
      - `risk_level = {{$json.risk_level}}`
      - `reason = {{$json.reason}}`
    - Connect: Parse AI Respons → Update Review Record
21. **Add “IF” node** named **“Check Suspicious Score Threshold”**
    - Condition: `{{$json.fields.fake_score}} >= 60`
    - Connect: Update Review Record → Check Suspicious Score Threshold
22. **Configure Slack credentials**
    - Add Slack API credential in n8n with permission to post messages.
23. **Add Slack node** named **“Send Moderation Alert”**
    - Operation: post message to channel
    - Choose channel (e.g., `#n8n`)
    - Paste the message template using `$json.fields.*`
    - Connect: Check Suspicious Score Threshold (true) → Send Moderation Alert
24. **Close the batch loop**
    - Connect **Send Moderation Alert → Split review in batches** (to continue to next review)
    - Connect **Check Suspicious Score Threshold (false) → Split review in batches**
    - Connect **Is New Review? (false) → Split review in batches**
25. **Test end-to-end**
    - Send a POST to the webhook with `{"products":[{"url":"https://...","product_id":"...","marketplace":"..."}]}` and ensure the scraper returns `reviews: []`.
    - Verify Airtable creates/updates records and Slack posts only for high-risk scores.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided disclaimer |
| “## How It Works … ## Setup Steps … Adjust the suspicious score threshold … Activate the workflow …” | Sticky Note “How It Works” (overview + operational setup guidance embedded in workflow canvas) |

