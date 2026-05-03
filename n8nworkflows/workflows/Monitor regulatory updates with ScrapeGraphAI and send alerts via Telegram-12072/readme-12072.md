Monitor regulatory updates with ScrapeGraphAI and send alerts via Telegram

https://n8nworkflows.xyz/workflows/monitor-regulatory-updates-with-scrapegraphai-and-send-alerts-via-telegram-12072


# Monitor regulatory updates with ScrapeGraphAI and send alerts via Telegram

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Monitor official regulatory news pages (SEC, UK FCA, ESMA), extract recent items using ScrapeGraphAI, flag potentially important updates via keyword matching, **alert via Telegram**, and **store all scraped items in Redis** with a 7‑day TTL for lightweight retention and deduplication.

**Typical use cases**
- Compliance / regulatory intelligence daily sweeps
- Early warning alerts for rule/directive-related announcements
- Short-term archive of all scraped notices for audit, dashboards, or later analytics

### 1.1 Input Reception & Source Definition
Manual execution starts the run; a Code node defines a list of regulator URLs and labels, then items are iterated one-by-one.

### 1.2 AI Scraping (ScrapeGraphAI)
Each source page is scraped independently using an LLM prompt that returns structured JSON (title/date/summary/url/sourceName).

### 1.3 Aggregation, Normalization, Deduplication & Classification
Results are merged, normalized into a consistent schema, assigned a stable dedup key, and classified as “important” based on keywords.

### 1.4 Alerts (Telegram) and Storage (Redis)
Important items produce a Markdown Telegram message. All items are stored into Redis for 7 days under a deterministic key.

### 1.5 Error Handling
Scrape failures are routed to a Code-based error formatter so failures are captured without terminating the overall design intent.

---

## 2. Block-by-Block Analysis

### Block 1 — Source & Split (Execution entry + list of sources + batching)

**Overview:** Starts the workflow manually, generates the list of regulatory sources, then iterates through them one at a time to keep scraping isolated per website.

**Nodes involved:**
- Start Workflow
- Define Sources
- Split Sources

#### Node: **Start Workflow**
- **Type / role:** Manual Trigger (entry point)
- **Configuration (interpreted):** No parameters; execution is started manually from the editor.
- **Inputs / outputs:**
  - Input: none
  - Output: to **Define Sources**
- **Edge cases / failures:** None (manual start only).
- **Version notes:** `manualTrigger` v1.

#### Node: **Define Sources**
- **Type / role:** Code node (creates the source list)
- **Configuration:**
  - JavaScript builds an array of objects like `{ url, source }` for:
    - `https://www.sec.gov/news/pressreleases` (SEC Press Releases)
    - `https://www.fca.org.uk/news` (UK FCA News)
    - `https://www.esma.europa.eu/press-news/esma-news` (ESMA News)
  - Returns one n8n item per source: `return urls.map(item => ({ json: item }));`
- **Key variables/logic:**
  - `urls` array controls what gets scraped.
- **Inputs / outputs:**
  - Input: from **Start Workflow**
  - Output: to **Split Sources**
- **Edge cases / failures:**
  - Invalid URL strings lead to downstream scraping errors.
  - Forgetting to include `source` will degrade labeling downstream (fallback exists but may become `unknown`).
- **Version notes:** Code node v2.

#### Node: **Split Sources**
- **Type / role:** Split In Batches (iteration controller)
- **Configuration:**
  - Batch options default (batch size effectively defaults to 1 in typical UI usage; JSON shows only `options: {}`).
  - Feeds each source one-by-one into the scraper.
- **Inputs / outputs:**
  - Input: from **Define Sources**
  - Output: to **Scrape Regulatory Data**
- **Edge cases / failures:**
  - If zero items are produced by Define Sources, nothing is scraped.
  - Very large source lists increase total runtime; consider adding scheduling and/or concurrency patterns.
- **Version notes:** splitInBatches v3.

---

### Block 2 — Scraping Layer (ScrapeGraphAI + error path)

**Overview:** Calls ScrapeGraphAI per source URL using an extraction prompt. Failures are routed to an error formatter node.

**Nodes involved:**
- Scrape Regulatory Data
- Error Handler

#### Node: **Scrape Regulatory Data**
- **Type / role:** ScrapeGraphAI node (LLM-based web extraction)
- **Configuration:**
  - **websiteUrl:** expression `={{ $json.url }}` (from Define Sources item)
  - **userPrompt:** instructs extraction of up to 10 items with fields:
    - `title`, `date`, `summary`, `url`
    - and `sourceName = "{{$json.source}}"`
- **Key expressions/variables:**
  - `{{$json.url}}` to target current source
  - `{{$json.source}}` embedded into the prompt to stamp source name
- **Inputs / outputs:**
  - Input: from **Split Sources**
  - Output: connected to **Merge Results** (two inputs) and to **Error Handler**
- **Important connection detail:**
  - The node is wired into **Merge Results** on *two inputs* (`index 0` and `index 1`). In “combine” mode this can have implications (see Merge node notes).
  - It is also wired to **Error Handler**, presumably using the node’s error output in the UI (the JSON connections don’t explicitly label error vs main; in n8n, error output wiring is a special channel).
- **Credentials required:**
  - ScrapeGraphAI API credential (as referenced in sticky note setup steps).
- **Edge cases / failures:**
  - Auth/credential failure (401/403).
  - Target sites blocking bots, rate limiting, or requiring JS rendering.
  - Model returns non-JSON or unexpected schema (downstream Code node partly normalizes but may still miss fields).
  - Timeouts on slow pages.
- **Version notes:** `n8n-nodes-scrapegraphai.scrapegraphAi` v1 (requires that community/custom node installed and compatible with your n8n version).

#### Node: **Error Handler**
- **Type / role:** Code node (normalizes error shape)
- **Configuration:**
  - Reads first incoming item: `const err = $input.all()[0].json;`
  - Outputs: `{ error: true, message: err.message || err, time: ISOString }`
- **Inputs / outputs:**
  - Input: from **Scrape Regulatory Data** (error path)
  - Output: none connected (as provided)
- **Edge cases / failures:**
  - If the incoming error object is not shaped as expected (e.g., `json` missing), `err.message` may be undefined; code falls back to `err`.
  - Because it is not connected onward, errors are only visible in execution logs unless you add a sink (Telegram/email/DB).
- **Version notes:** Code node v2.

---

### Block 3 — Aggregation & Processing (merge + normalize + classify)

**Overview:** Waits for scraped results, then flattens/normalizes items, creates a deterministic dedup key, flags “important” items, and prepares data for branching.

**Nodes involved:**
- Merge Results
- Format & Deduplicate
- New Important Update?

#### Node: **Merge Results**
- **Type / role:** Merge node (aggregates scraping outputs)
- **Configuration:**
  - **Mode:** `combine`
  - Merge-by-fields structure present but effectively empty (`mergeByFields.values: [{}]`), implying no explicit key-based merge.
- **Inputs / outputs:**
  - Input: from **Scrape Regulatory Data** (wired into both input 0 and input 1)
  - Output: to **Format & Deduplicate**
- **Edge cases / failures / design notes:**
  - **Potential misconfiguration:** In `combine` mode, Merge typically expects two distinct input streams; wiring the *same* node into both inputs can cause:
    - duplicated data, unexpected pairing behavior, or waiting behavior depending on how n8n handles arrivals per input.
  - If the intent is “collect all results”, common alternatives are:
    - Merge mode “Append”
    - Or no merge at all, if downstream can process per-source
    - Or use “Wait for All” patterns
  - As-is, verify executions to ensure the merged output is what you expect.
- **Version notes:** Merge v2.

#### Node: **Format & Deduplicate**
- **Type / role:** Code node (normalization + dedup id + importance tag)
- **Configuration/logic (interpreted):**
  - Reads all incoming items: `const items = $input.all();`
  - Defines keyword regex: `/(rule|regulation|directive|act)/i`
  - For each incoming item:
    - If `item.json` is an array, it iterates it; else wraps into array.
    - Normalizes:
      - `title` from `d.title || d.headline || ''`
      - `summary` from `d.summary || d.description || ''`
    - Creates `dedupId`:
      - base64 of `(d.url || title)` truncated to 24 chars
    - Sets:
      - `isImportant` if regex matches title or summary
      - `scrapedAt` ISO timestamp
      - `source` from `d.sourceName || item.json.source || 'unknown'`
  - Emits one n8n item per news notice.
- **Inputs / outputs:**
  - Input: from **Merge Results**
  - Output 1: to **New Important Update?**
  - Output 2: to **Save to Redis** (parallel path)
- **Edge cases / failures:**
  - If ScrapeGraphAI returns a nested structure not matching `item.json` as array/object of notices, flattening may not work.
  - If `d.url` is missing and title empty, dedupId will be based on empty string → identical dedup keys → overwrite in Redis.
  - Base64 truncation collisions are possible (rare but possible); consider hashing (SHA-256) if collisions matter.
- **Version notes:** Code node v2.

#### Node: **New Important Update?**
- **Type / role:** IF node (branches on importance)
- **Configuration:**
  - Boolean condition: `value1 = {{ $json.isImportant }}` operation `true`
- **Inputs / outputs:**
  - Input: from **Format & Deduplicate**
  - Output (true branch): to **Prepare Telegram Message**
  - False branch: not connected (important: non-important items will not be alerted, but they still go to Redis because Redis is connected earlier in parallel)
- **Edge cases / failures:**
  - If `isImportant` is missing or not boolean, expression coercion may yield unexpected results.
- **Version notes:** IF node v2.

---

### Block 4 — Storage & Alerts (Redis cache + Telegram alerting)

**Overview:** Stores every normalized item in Redis with a 7-day TTL. For important items, formats a Telegram Markdown message and sends it.

**Nodes involved:**
- Save to Redis
- Prepare Telegram Message
- Send Telegram Alert

#### Node: **Save to Redis**
- **Type / role:** Redis node (key-value storage)
- **Configuration:**
  - **Operation:** set
  - **Key:** `reg_update:{{ $json.dedupId }}`
  - **Value:** `JSON.stringify($json)`
  - **TTL:** 604800 seconds (7 days)
  - **Expire:** enabled
- **Inputs / outputs:**
  - Input: from **Format & Deduplicate**
  - Output: none connected
- **Credentials required:**
  - Redis credential with write access (host/port/password/TLS depending on your setup).
- **Edge cases / failures:**
  - Redis auth/connection errors (network, TLS, wrong password).
  - If `dedupId` missing, key becomes `reg_update:undefined` leading to overwrites.
  - Storing full JSON may exceed size limits if scraper returns very large fields (rare for 10 items, but possible).
- **Version notes:** Redis node v1.

#### Node: **Prepare Telegram Message**
- **Type / role:** Set node (builds message payload)
- **Configuration (current JSON):**
  - No fields are defined in the node parameters shown; however downstream Telegram expects `$json.text`.
- **Inputs / outputs:**
  - Input: from **New Important Update?** (true branch)
  - Output: to **Send Telegram Alert**
- **Edge cases / failures (important):**
  - **As configured in the provided JSON, this node does not actually create `text`.**  
    This will likely cause Telegram to send an empty message or fail validation depending on Telegram node behavior.
- **Recommended intended behavior (inferred from sticky note):**
  - Create a Markdown string including `source`, `title`, `date`, and `url`.
- **Version notes:** Set node v3.

#### Node: **Send Telegram Alert**
- **Type / role:** Telegram node (message send)
- **Configuration:**
  - **Chat ID:** `{{ $env.TELEGRAM_CHAT_ID || 'YOUR_CHAT_ID' }}`
    - Uses an environment variable if present; otherwise a placeholder string (will fail unless replaced).
  - **Text:** `{{ $json.text }}`
  - **Parse mode:** Markdown
  - Notifications: `disable_notification: false`
- **Inputs / outputs:**
  - Input: from **Prepare Telegram Message**
  - Output: none connected
- **Credentials required:**
  - Telegram Bot credential (bot token).
- **Edge cases / failures:**
  - Missing/incorrect bot token.
  - Invalid `chatId` (placeholder not replaced) or bot not allowed to message that chat.
  - Markdown parse errors if message contains unescaped characters; consider MarkdownV2 or escaping.
  - Rate limiting if many “important” items in one run.
- **Version notes:** Telegram node v1.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start Workflow | Manual Trigger | Manual entry point | — | Define Sources | ## Source & Split<br><br>This section contains the Manual Trigger, **Define Sources** Code node and **Split Sources** node. The trigger is kept manual to simplify testing; you can connect a Schedule Trigger later if you want unattended runs. The code node returns an array of JavaScript objects where each object holds the URL of an official regulator press-release page and a human-readable label. Because the URLs live in code it is trivial to add or remove sources without touching any downstream logic. The **Split in Batches** node then iterates through that array with a batch size of one. n8n processes each batch independently, so slow or failing pages will not block the entire job. Running sources individually also gives ScrapeGraphAI a predictable, single-URL payload which keeps memory overhead low and makes troubleshooting easier. Together these nodes prepare a clean, one-by-one stream of inputs for the AI scraper that follows. |
| Define Sources | Code | Define regulator URLs and labels | Start Workflow | Split Sources | (same as above) |
| Split Sources | Split In Batches | Iterate through sources one-by-one | Define Sources | Scrape Regulatory Data | (same as above) |
| Scrape Regulatory Data | ScrapeGraphAI | Extract recent regulatory items from each source page | Split Sources | Merge Results, Error Handler | ## Scraping Layer<br><br>The **Scrape Regulatory Data** node is the heart of the workflow. For every incoming URL it calls ScrapeGraphAI with a concise yet specific prompt instructing the model to extract headline, publication date, summary paragraph and canonical link. Because the node receives only one URL at a time, it can devote the maximum timeout to a single site without starving others. The LLM approach means you are not writing fragile CSS selectors; minor layout tweaks on government portals rarely break the extraction. Error handling is wired through the node’s built-in error output and funnels directly to an **Error Handler** Code node, ensuring that individual failures are recorded without halting the job. Successful responses pass to the Merge node where they will be queued until all parallel scrapers complete. Keeping the scraping concerns isolated in this dedicated layer makes the rest of the workflow easier to maintain and extend. |
| Merge Results | Merge | Aggregate scraped results for downstream processing | Scrape Regulatory Data | Format & Deduplicate | ## Aggregation & Processing<br><br>Once each source has finished scraping, their results converge in the **Merge Results** node which waits until all input streams arrive. This guarantees that downstream logic always receives a complete picture of the regulatory landscape for that execution. The subsequent **Format & Deduplicate** Code node does four important things: it standardises field names so every item has `title`, `date`, `summary` and `url`; it attaches an ISO time stamp; it builds a 24-character base64 hash that acts as a deduplication key; and it tags the record as important if keywords like “rule” or “directive” appear. The **New Important Update?** IF node then branches the flow. Items flagged important head toward Telegram for immediate alerting, while all items—regardless of importance—continue to storage. This separation lets you fine-tune what gets pushed to noisy channels without losing the full history in your database. |
| Format & Deduplicate | Code | Normalize items, create dedupId, tag importance | Merge Results | New Important Update?, Save to Redis | (same as above) |
| New Important Update? | IF | Route important items to alerts | Format & Deduplicate | Prepare Telegram Message | (same as above) |
| Prepare Telegram Message | Set | Build Markdown message text for Telegram | New Important Update? | Send Telegram Alert | ## Storage & Alerts<br><br>The last cluster includes **Save to Redis**, **Prepare Telegram Message** and **Send Telegram Alert**. Redis is chosen because it is lightning fast and well-suited for simple key-value caching. Each update is stored under the key `reg_update:<hash>` with a seven-day TTL so your datastore remains compact while still offering a rolling archive for audit purposes. Storing all records—even ones that are not flagged important—means you can build dashboards or run later analytics without scraping again. For urgent items, a Set node assembles a concise Markdown message that gives the compliance team everything they need at a glance: source, headline, date and link. The Telegram node posts this straight into your chosen chat, leveraging Telegram’s real-time push notifications and ubiquitous mobile apps. Because storage and alerting are separate branches, you can extend either one—such as adding a database sink or alternate alert channel—without affecting the other. |
| Send Telegram Alert | Telegram | Send alert to Telegram chat | Prepare Telegram Message | — | (same as above) |
| Save to Redis | Redis | Store each scraped item with TTL for dedup/history | Format & Deduplicate | — | (same as above) |
| Error Handler | Code | Normalize scrape errors for logging/forwarding | Scrape Regulatory Data | — | ## Scraping Layer<br><br>(same content as “Scraping Layer” note above; it explicitly references the Error Handler in this layer.) |
| Workflow Overview | Sticky Note | Documentation / inline overview | — | — | ## How it works<br><br>This workflow lets compliance analysts manually launch a daily sweep across several official regulatory news feeds. When you click ‘Execute’, a Code node produces a list of URLs for the SEC, FCA and ESMA press-release pages (add more if you monitor other jurisdictions). The list is split so each page is scraped in its own run of ScrapeGraphAI. The AI extracts headline, date, summary and link for every notice it finds. A Merge node waits until all sources finish, then a follow-up Code node normalises the data, flags posts that look like new rules or directives and assigns a unique hash. All items are stored in Redis for seven days so you keep a lightweight archive, while anything flagged as important triggers an instant Telegram message.<br><br>## Setup steps<br><br>1. Add a ScrapeGraphAI API credential<br>2. Add a Redis credential (ensure write access)<br>3. Add a Telegram Bot credential and set your chat ID<br>4. Open the “Define Sources” Code node and list all URLs you need<br>5. Optional: tweak keyword list in “Format & Deduplicate”<br>6. Click Execute to test, then enable scheduling externally if desired |
| Section – Source & Split | Sticky Note | Documentation / section header | — | — | (content is the sticky note itself) |
| Section – Scraping Layer | Sticky Note | Documentation / section header | — | — | (content is the sticky note itself) |
| Section – Aggregation & Processing | Sticky Note | Documentation / section header | — | — | (content is the sticky note itself) |
| Section – Storage & Alerts | Sticky Note | Documentation / section header | — | — | (content is the sticky note itself) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it as desired (JSON name is “Medical Research Tracker with Email and Pipedrive”, but your stated title is “Monitor regulatory updates with ScrapeGraphAI and send alerts via Telegram”).

2. **Add node: Manual Trigger**
   - Node type: **Manual Trigger**
   - Name: **Start Workflow**

3. **Add node: Code**
   - Node type: **Code**
   - Name: **Define Sources**
   - Paste logic that returns one item per source, e.g.:
     - Maintain an array of `{ url, source }`
     - Return `urls.map(item => ({ json: item }))`

4. **Connect:** Start Workflow → Define Sources

5. **Add node: Split In Batches**
   - Node type: **Split In Batches**
   - Name: **Split Sources**
   - Keep defaults (or explicitly set batch size to 1 in the UI if available)

6. **Connect:** Define Sources → Split Sources

7. **Add node: ScrapeGraphAI**
   - Node type: **ScrapeGraphAI** (community/custom node must be installed)
   - Name: **Scrape Regulatory Data**
   - **Website URL:** expression `{{$json.url}}`
   - **Prompt:** instruct extraction of items returning JSON fields `title, date, summary, url` and set `sourceName` to the incoming `source` (via expression in prompt).
   - **Credentials:** configure **ScrapeGraphAI API** credential in n8n Credentials.

8. **Connect:** Split Sources → Scrape Regulatory Data

9. **Add node: Merge**
   - Node type: **Merge**
   - Name: **Merge Results**
   - Mode: **Combine**
   - (Recommended for correctness: consider using **Append** instead if your goal is simply to collect items from all sources. If you keep “Combine”, ensure you truly have two distinct inputs.)
   - Connect Scrape Regulatory Data output to Merge Results (input 0).  
     - If you replicate the JSON exactly, also connect to input 1—but validate behavior.

10. **Add node: Code**
    - Node type: **Code**
    - Name: **Format & Deduplicate**
    - Implement:
      - Flatten array/object responses into one item per notice
      - Normalize `title/summary`
      - Generate `dedupId`
      - Set `isImportant` via regex keywords
      - Add `scrapedAt` and normalize `source`

11. **Connect:** Merge Results → Format & Deduplicate

12. **Add node: IF**
    - Node type: **IF**
    - Name: **New Important Update?**
    - Condition:
      - Boolean → `{{$json.isImportant}}` is true

13. **Connect:** Format & Deduplicate → New Important Update?

14. **Add node: Redis**
    - Node type: **Redis**
    - Name: **Save to Redis**
    - Operation: **Set**
    - Key: `reg_update:{{$json.dedupId}}`
    - Value: `{{JSON.stringify($json)}}`
    - Enable expire + set TTL to **604800** seconds
    - **Credentials:** configure Redis connection credential in n8n.

15. **Connect:** Format & Deduplicate → Save to Redis  
    - (This ensures *all* items are stored, regardless of importance.)

16. **Add node: Set**
    - Node type: **Set**
    - Name: **Prepare Telegram Message**
    - Add a field:
      - `text` (string), e.g. Markdown:
        - `*{{$json.source}}*\n{{$json.title}}\n_{{$json.date}}_\n{{$json.url}}`
    - Ensure the output item contains `$json.text`.

17. **Connect:** New Important Update? (true output) → Prepare Telegram Message

18. **Add node: Telegram**
    - Node type: **Telegram**
    - Name: **Send Telegram Alert**
    - Operation: send message
    - Chat ID:
      - Use `{{$env.TELEGRAM_CHAT_ID}}` if you run n8n with that env var set, otherwise hardcode your chat id.
    - Text: `{{$json.text}}`
    - Parse mode: Markdown
    - **Credentials:** create Telegram Bot credential (bot token from BotFather).

19. **Connect:** Prepare Telegram Message → Send Telegram Alert

20. **Add node: Code** (optional but included in the provided workflow)
    - Node type: **Code**
    - Name: **Error Handler**
    - Convert error payload into `{ error: true, message, time }`
    - Connect ScrapeGraphAI node **error output** to this node (in the UI: drag from ScrapeGraphAI error connector).

21. **Add Sticky Notes (optional)**
    - Create sticky notes to document sections and setup steps (content can be copied from the provided sticky notes).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ScrapeGraphAI extraction reduces reliance on fragile CSS selectors and often survives minor layout changes on regulator portals. | Scraping Layer note |
| Setup requires: ScrapeGraphAI credential, Redis credential (write access), Telegram bot credential + chat ID, and optional keyword tuning. | Workflow Overview note |
| Consider changing Merge mode if “Combine” with the same upstream wired to both inputs produces unexpected results; “Append” is typically safer for aggregation. | Merge Results design consideration |
| Prepare Telegram Message node must set `$json.text` (current JSON does not), otherwise Telegram message text may be empty/invalid. | Alerting reliability consideration |