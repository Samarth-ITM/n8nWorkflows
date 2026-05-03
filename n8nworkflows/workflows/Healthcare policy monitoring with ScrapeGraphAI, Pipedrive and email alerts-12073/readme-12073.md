Healthcare policy monitoring with ScrapeGraphAI, Pipedrive and email alerts

https://n8nworkflows.xyz/workflows/healthcare-policy-monitoring-with-scrapegraphai--pipedrive-and-email-alerts-12073


# Healthcare policy monitoring with ScrapeGraphAI, Pipedrive and email alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Monitor specific healthcare policy/proposal webpages, extract structured details via ScrapeGraphAI, enrich them with an external metadata API, filter to “recent” items (last 30 days), log them in Pipedrive, and send an email alert.

**Target use cases**
- Compliance / policy teams tracking new HHS/CMS policy announcements
- Internal “policy intake” pipeline: new policy pages become CRM items + notifications
- Lightweight deduplication using a stable hash per policy title + publication date

**Logical blocks**
1.1 **Input & URL List Definition**: manual start + define which pages to monitor  
1.2 **Iteration + AI Scraping**: loop through URLs and scrape structured policy info  
1.3 **Enrichment + Normalization**: call enrichment API, merge fields, add metadata  
1.4 **Deduplication Key + Freshness Filter**: compute `policy_id`, mark `is_recent`, filter  
1.5 **Storage (Pipedrive) + Notifications (Email)**: create a Pipedrive activity, then email alert

> Note: Sticky notes mention “Upsert Policy in Pipedrive as new deals”, but the actual node present is **Pipedrive → Create an activity** (not a Deal upsert). Also, the workflow name in n8n is “Breaking News Aggregator with SendGrid and PostgreSQL”, while the provided title is “Healthcare policy monitoring with ScrapeGraphAI, Pipedrive and email alerts”; the implemented logic matches the latter.

---

## 2. Block-by-Block Analysis

### 2.1 Input & URL List Definition

**Overview:** Starts the workflow manually and emits an array of target policy URLs (with a `source` label) to be processed downstream.

**Nodes involved**
- Run Workflow
- Define Target URLs

#### Node: Run Workflow
- **Type / role:** `Manual Trigger` — ad-hoc entry point for testing or on-demand sweeps.
- **Configuration choices:** No parameters; purely manual execution.
- **Inputs/outputs:**
  - **Output →** Define Target URLs
- **Failure/edge cases:** None (except workflow not executed).
- **Version notes:** TypeVersion 1; stable.

#### Node: Define Target URLs
- **Type / role:** `Code` — produces items containing `url` and `source`.
- **Configuration choices:**
  - Hard-coded list of objects:
    - `url`: policy/proposal page
    - `source`: e.g., “HHS”, “CMS”
- **Key variables/logic:**
  - Returns an array of `[{ json: { url, source } }, ...]`.
- **Inputs/outputs:**
  - **Input:** Manual Trigger (no data needed)
  - **Output →** Loop URLs (SplitInBatches)
- **Failure/edge cases:**
  - Invalid URLs, duplicates, removed pages, redirects.
  - If you later add many URLs, downstream rate limits become more likely.
- **Version notes:** TypeVersion 2 uses the newer n8n Code node runtime behavior.

---

### 2.2 Iteration + AI Scraping

**Overview:** Processes the URL list sequentially and uses ScrapeGraphAI to extract structured policy fields from each page.

**Nodes involved**
- Loop URLs
- Scrape Policy Page

#### Node: Loop URLs
- **Type / role:** `Split In Batches` — iterates items in batches (default behavior is effectively batch size 1 unless configured).
- **Configuration choices:** No batch size explicitly set; uses defaults.
- **Inputs/outputs:**
  - **Input:** Define Target URLs items
  - **Output →** Scrape Policy Page
- **Failure/edge cases:**
  - If batch size is large, can overload scrape/enrichment services.
  - If you expect to paginate through batches, you typically also need a loop-back connection; this workflow does not show a “continue” loop (n8n still processes the first batch; without loop-back, it may not iterate as intended depending on execution model). Consider validating execution behavior in your n8n version.
- **Version notes:** TypeVersion 3.

#### Node: Scrape Policy Page
- **Type / role:** `ScrapeGraphAI` — AI-driven extraction from a web page into JSON.
- **Configuration choices:**
  - `websiteUrl`: `={{ $json.url }}`
  - `userPrompt`: requests exact JSON keys: `title`, `summary` (≤300 chars), `publication_date`, `reference_number`, `source_url`.
- **Key expressions/variables:**
  - Uses incoming item’s `url`.
- **Inputs/outputs:**
  - **Input:** from Loop URLs (`url`, `source`)
  - **Output →** Enrich with Metadata
- **Failure/edge cases:**
  - Scraping blocked (robots/WAF), timeouts, rendering issues.
  - Model returns non-JSON or keys missing despite the prompt.
  - Publication date formats may vary or be absent.
- **Version notes:** TypeVersion 1; requires ScrapeGraphAI node package and credentials configured.

---

### 2.3 Enrichment + Normalization

**Overview:** Calls an external enrichment API based on the canonical `source_url`, then merges/normalizes fields into a single clean policy object.

**Nodes involved**
- Enrich with Metadata
- Transform & Clean Data

#### Node: Enrich with Metadata
- **Type / role:** `HTTP Request` — calls a third-party enrichment service.
- **Configuration choices:**
  - URL expression:  
    `={{ 'https://example-policy-enrichment-api.com/v1/enrich?url=' + encodeURIComponent($json.source_url) }}`
  - No explicit method specified (defaults to GET in n8n HTTP Request).
  - No headers/auth shown (likely expected to be configured via credentials or options, per sticky note).
- **Inputs/outputs:**
  - **Input:** scraped JSON (must include `source_url`)
  - **Output →** Transform & Clean Data
- **Failure/edge cases:**
  - If `$json.source_url` is missing/empty, the request URL becomes invalid.
  - 401/403 (missing API key), 429 rate limits, 5xx errors.
  - Response shape mismatches what downstream code expects.
- **Version notes:** TypeVersion 4.

#### Node: Transform & Clean Data
- **Type / role:** `Code` — merges scraped data with enrichment metadata and injects original URL/source.
- **Configuration choices / logic (interpreted):**
  - Reads `items[0].json` into `scraped`.
  - Attempts to derive enrichment metadata from `scraped.metadata` (important caveat below).
  - Outputs a unified object:
    - `title`, `summary`, `publication_date`, `reference_number`, `source_url`
    - `topics` default `[]`
    - `sentiment` default `'neutral'`
    - `source` and `original_url` from the incoming `$json`
- **Key expressions/variables:**
  - `source: $json.source`
  - `original_url: $json.url`
- **Inputs/outputs:**
  - **Input:** output of Enrich with Metadata
  - **Output →** Generate Unique ID
- **Critical implementation caveat (likely bug):**
  - The code treats the HTTP response as if it were embedded in `scraped.metadata`, but **after the HTTP Request node, `items[0].json` is typically the HTTP response body**, not the previous ScrapeGraphAI result.
  - As written, `scraped` is probably the enrichment response, not the scraped policy JSON; fields like `scraped.title` may be undefined unless the enrichment API echoes them.
  - If you intended to merge two separate node outputs, you typically need a **Merge** node (or store scraped fields via `$items()` references).
- **Failure/edge cases:**
  - `publication_date` missing/invalid → later date math can produce `Invalid Date`.
  - `source`/`url` may be missing at this point because the HTTP response replaced prior item structure (depending on HTTP node settings).
- **Version notes:** TypeVersion 2.

---

### 2.4 Deduplication Key + Freshness Filter

**Overview:** Creates a stable hash `policy_id` and marks whether the policy is within the last 30 days, then filters to only recent items.

**Nodes involved**
- Generate Unique ID
- Is Recent Policy?

#### Node: Generate Unique ID
- **Type / role:** `Code` — creates `policy_id` and computes `is_recent`.
- **Configuration choices / logic:**
  - Uses Node.js `crypto` to MD5 hash: `title-publication_date`.
  - Computes recency: `(now - publication_date) < 30 days`.
- **Key variables:**
  - `idString = `${item.title}-${item.publication_date}``
  - `item.policy_id = md5(idString)`
  - `item.is_recent = (new Date() - new Date(item.publication_date)) < days30`
- **Inputs/outputs:**
  - **Input:** unified policy object
  - **Output →** Is Recent Policy?
- **Failure/edge cases:**
  - If `title` or `publication_date` undefined → hash still generated but unstable/useless; may collapse to same hash for many items.
  - If `publication_date` invalid → `new Date(item.publication_date)` becomes Invalid Date; subtraction yields `NaN`; condition becomes `false` (so everything drops as “not recent”).
  - Timezone ambiguities (dates without timezone).
- **Version notes:** TypeVersion 2.

#### Node: Is Recent Policy?
- **Type / role:** `IF` — gating node.
- **Configuration choices:**
  - Boolean condition: `={{ $json.is_recent }}` **is true**
- **Inputs/outputs:**
  - **Input:** item with `is_recent`
  - **True output (main index 0) →** Create an activity (Pipedrive)
  - **False output:** not connected (items end here silently)
- **Failure/edge cases:**
  - Missing `is_recent` evaluates to falsey; item will be dropped.
- **Version notes:** TypeVersion 2.

---

### 2.5 Storage (Pipedrive) + Notifications (Email)

**Overview:** For recent policies, creates a Pipedrive Activity (not a Deal), then prepares and sends an email alert.

**Nodes involved**
- Create an activity
- Prepare Email Content
- Send Policy Alert

#### Node: Create an activity
- **Type / role:** `Pipedrive` — creates an Activity record.
- **Configuration choices:**
  - `resource: activity`
  - Operation is not explicitly shown in JSON, but with `resource=activity` and node name “Create an activity”, it is intended to create.
  - `additionalFields`: empty (so it likely creates a minimal activity unless required fields are mapped elsewhere).
- **Inputs/outputs:**
  - **Input:** recent policy item
  - **Output →** Prepare Email Content
- **Failure/edge cases:**
  - Missing required Pipedrive fields (subject, type, due date, etc.) depending on Pipedrive API requirements and node defaults.
  - Auth/permission errors with Pipedrive credentials.
  - Rate limiting if many policies pass through.
- **Version notes:** TypeVersion 1.

#### Node: Prepare Email Content
- **Type / role:** `Set` — intended to craft `emailSubject` and likely an email body from policy fields.
- **Configuration choices:** Not actually configured (no fields defined in parameters).
- **Inputs/outputs:**
  - **Input:** output of Pipedrive node (may include Pipedrive activity response rather than original policy fields)
  - **Output →** Send Policy Alert
- **Critical caveat (missing configuration):**
  - Email node uses `{{$json.emailSubject}}`, but this Set node does not define it. As-is, subject will be empty/undefined.
  - Also, after creating an activity, the JSON payload may be replaced by Pipedrive’s response; policy fields might not be available unless “Keep input data” is enabled or fields are merged.
- **Failure/edge cases:**
  - Expressions referencing fields that are not present cause empty strings / undefined content.
- **Version notes:** TypeVersion 3.4.

#### Node: Send Policy Alert
- **Type / role:** `Email Send` — sends the alert email via SMTP.
- **Configuration choices:**
  - `toEmail`: `user@example.com` (placeholder)
  - `fromEmail`: `user@example.com` (placeholder)
  - `subject`: `={{ $json.emailSubject }}`
  - No body text is set in parameters (so it may send empty body unless defaults exist).
- **Inputs/outputs:**
  - **Input:** prepared email content
  - **Output:** none
- **Failure/edge cases:**
  - Missing SMTP credentials/config in n8n.
  - Invalid from/to addresses, blocked relay, SPF/DKIM issues.
  - Subject/body missing due to upstream Set node not populating fields.
- **Version notes:** TypeVersion 2.1.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Workflow | Manual Trigger | Manual entry point | — | Define Target URLs | **Data Collection:** Starts manual polling while tweaking/testing. |
| Define Target URLs | Code | Defines monitored policy URLs | Run Workflow | Loop URLs | **Data Collection:** Single source of truth for URLs to watch. |
| Loop URLs | Split In Batches | Iterates through URL list | Define Target URLs | Scrape Policy Page | **Data Collection:** Batching to avoid overloading sites/services. |
| Scrape Policy Page | ScrapeGraphAI | Extracts structured fields from page | Loop URLs | Enrich with Metadata | **Data Collection:** AI extraction into clean JSON fields. |
| Enrich with Metadata | HTTP Request | Calls external enrichment API | Scrape Policy Page | Transform & Clean Data | **Enrichment & Transformation:** Tags topics and sentiment via external API. |
| Transform & Clean Data | Code | Merges/normalizes scraped + enriched fields | Enrich with Metadata | Generate Unique ID | **Enrichment & Transformation:** Merge payloads, normalize, remove empties (intended). |
| Generate Unique ID | Code | Dedup hash + recency flag | Transform & Clean Data | Is Recent Policy? | **Enrichment & Transformation:** Hash identifier and 30-day freshness flag. |
| Is Recent Policy? | IF | Filters only recent policies | Generate Unique ID | Create an activity (true path) | **Storage & Deduplication:** Gatekeeper to prevent old items cluttering CRM. |
| Create an activity | Pipedrive | Stores item in Pipedrive (activity) | Is Recent Policy? | Prepare Email Content | **Storage & Deduplication:** Sticky note claims deal upsert, but node actually creates an activity. |
| Prepare Email Content | Set | Builds email subject/body | Create an activity | Send Policy Alert | **Notifications & Alerts:** Intended to craft readable email content via expressions. |
| Send Policy Alert | Email Send | Sends email alert via SMTP | Prepare Email Content | — | **Notifications & Alerts:** Dispatches alert email; easy to branch to Slack/Teams later. |
| Workflow Overview | Sticky Note | Documentation | — | — | ## How it works … (contains setup steps and overall description) |
| Section – Data Collection | Sticky Note | Block comment | — | — | ## Data Collection … |
| Section – Enrichment & Transformation | Sticky Note | Block comment | — | — | ## Enrichment & Transformation … |
| Section – Storage | Sticky Note | Block comment | — | — | ## Storage & Deduplication … |
| Section – Notifications | Sticky Note | Block comment | — | — | ## Notifications & Alerts … |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Manual Trigger**
   - Node: **Manual Trigger**
   - Name: `Run Workflow`

3. **Add Code node to define URLs**
   - Node: **Code**
   - Name: `Define Target URLs`
   - Paste logic to return items like:
     - `json.url` (policy page URL)
     - `json.source` (label like HHS/CMS)
   - Connect: `Run Workflow` → `Define Target URLs`

4. **Add Split In Batches**
   - Node: **Split In Batches**
   - Name: `Loop URLs`
   - Keep defaults (or set Batch Size = 1 explicitly for safety)
   - Connect: `Define Target URLs` → `Loop URLs`

5. **Add ScrapeGraphAI node**
   - Node: **ScrapeGraphAI**
   - Name: `Scrape Policy Page`
   - Set **Website URL** to expression: `{{$json.url}}`
   - Set **User Prompt** to request exact JSON keys:
     - `title`, `summary` (≤300 chars), `publication_date`, `reference_number`, `source_url`
   - **Credentials:** add/configure ScrapeGraphAI credentials in n8n
   - Connect: `Loop URLs` → `Scrape Policy Page`

6. **Add HTTP Request enrichment call**
   - Node: **HTTP Request**
   - Name: `Enrich with Metadata`
   - Method: **GET**
   - URL expression:  
     `https://example-policy-enrichment-api.com/v1/enrich?url={{encodeURIComponent($json.source_url)}}`
   - **Credentials/Auth:**
     - If API uses header token, configure in node (e.g., `Authorization: Bearer ...`) or via HTTP Request credentials.
   - Connect: `Scrape Policy Page` → `Enrich with Metadata`

7. **Add Code node to transform/merge**
   - Node: **Code**
   - Name: `Transform & Clean Data`
   - Implement a merge strategy. To truly merge scrape + enrichment, do one of the following:
     - **Option A (recommended):** Insert a **Merge** node (Mode: “Combine”) to combine ScrapeGraphAI output and HTTP output, then transform.
     - **Option B:** In Code, reference prior node items using `$items('Scrape Policy Page')` while processing HTTP response.
   - Ensure the final output includes:
     - `title`, `summary`, `publication_date`, `reference_number`, `source_url`
     - `topics` (array), `sentiment`
     - `source` and `original_url`
   - Connect: `Enrich with Metadata` → `Transform & Clean Data`

8. **Add Code node for unique id + recency**
   - Node: **Code**
   - Name: `Generate Unique ID`
   - Use MD5 hash of `title-publication_date` into `policy_id`
   - Add boolean `is_recent` if published within last 30 days
   - Connect: `Transform & Clean Data` → `Generate Unique ID`

9. **Add IF node**
   - Node: **IF**
   - Name: `Is Recent Policy?`
   - Condition: Boolean → `{{$json.is_recent}}` “is true”
   - Connect: `Generate Unique ID` → `Is Recent Policy?`

10. **Add Pipedrive node (activity)**
    - Node: **Pipedrive**
    - Name: `Create an activity`
    - Resource: **Activity**
    - Operation: **Create**
    - Map key fields (recommended):
      - Subject/title to policy title
      - Notes/description to summary + URL
      - Optional: store `policy_id`, `reference_number`, `publication_date` in custom fields if supported for activities
    - **Credentials:** connect Pipedrive account / API token in n8n
    - Connect: `Is Recent Policy?` (true output) → `Create an activity`

11. **Add Set node to prepare email**
    - Node: **Set**
    - Name: `Prepare Email Content`
    - Add fields (example):
      - `emailSubject`: `New healthcare policy: {{$json.title}} ({{$json.publication_date}})`
      - `emailBody`: include title, summary, reference number, and `source_url`
    - Consider enabling “Keep Only Set” = false (so you retain original fields for the Email Send node).
    - Connect: `Create an activity` → `Prepare Email Content`

12. **Add Email Send node**
    - Node: **Email Send**
    - Name: `Send Policy Alert`
    - Configure:
      - To: your recipient(s)
      - From: your sender
      - Subject: `{{$json.emailSubject}}`
      - Body/Text: `{{$json.emailBody}}` (add this explicitly)
    - **Credentials:** configure SMTP credentials in n8n (host, port, user, pass, TLS)
    - Connect: `Prepare Email Content` → `Send Policy Alert`

13. **(Optional but important) Ensure the URL loop fully iterates**
    - If your n8n version requires it, connect a node after `Send Policy Alert` back to `Loop URLs` “Continue” input to process the next batch.
    - Verify by executing with 2+ URLs and confirming multiple iterations.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow lets healthcare administrators spot new policy proposals the moment they appear…” plus setup checklist (ScrapeGraphAI creds, enrichment API key, Pipedrive custom fields, SMTP, replace target URLs). | From sticky note **Workflow Overview** (embedded in the workflow). |
| Sticky notes describe “Upsert Policy in Pipedrive” and creating “deals”, but the implemented node is **Pipedrive Activity create**. | Consistency check between documentation and actual nodes. |
| Enrichment API endpoint is a placeholder domain (`example-policy-enrichment-api.com`). | Must be replaced with a real enrichment provider and authentication method. |
| Email settings use placeholders (`user@example.com`) and Set node does not currently populate `emailSubject`. | Must configure `Prepare Email Content` and Email body/subject mappings before production use. |