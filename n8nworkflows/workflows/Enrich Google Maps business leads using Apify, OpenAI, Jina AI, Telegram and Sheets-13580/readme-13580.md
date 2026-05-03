Enrich Google Maps business leads using Apify, OpenAI, Jina AI, Telegram and Sheets

https://n8nworkflows.xyz/workflows/enrich-google-maps-business-leads-using-apify--openai--jina-ai--telegram-and-sheets-13580


# Enrich Google Maps business leads using Apify, OpenAI, Jina AI, Telegram and Sheets

## 1. Workflow Overview

**Purpose:**  
This workflow receives a Telegram message containing search parameters, runs an Apify Google Maps scraping actor, deduplicates the resulting places, enriches each lead with an AI-generated company summary (OpenAI), stores/updates the lead in Google Sheets, optionally fetches the business website content (Jina AI) to extract a primary contact email (OpenAI), updates the email back into the same sheet, rate-limits between items, and finally sends a “DONE” message to Telegram.

**Typical use cases:**
- Building/enriching a Google Maps prospecting list (SMBs, local businesses) from a specific Maps URL and sector keyword.
- Maintaining a deduplicated lead database in Google Sheets with summaries + extracted email contacts.

### 1.1 Input Reception (Telegram)
Receives a Telegram message and parses it into `sector`, `limit`, `mapsUrl`.

### 1.2 Google Maps Scrape via Apify
Runs the Apify Maps Scraper actor using the parsed parameters, then fetches dataset items from Apify.

### 1.3 Normalize & Loop Through Places
Removes duplicates and iterates through places one by one.

### 1.4 AI Summary + Save Core Lead Data (Sheets)
Generates a concise company summary (OpenAI) and upserts the main place fields into Google Sheets.

### 1.5 Website Email Enrichment (Jina + OpenAI) + Sheet Update
If a website exists, fetches website content via Jina AI, asks OpenAI to extract one best email, then updates the Email column for that lead.

### 1.6 Rate Limiting + Final Notification
Waits between items, continues the batch loop, and sends a final DONE notification to Telegram.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Telegram → Parse)
**Overview:** Ingests a Telegram chat message and extracts three parameters separated by semicolons to drive the scrape.  
**Nodes involved:** Receive Telegram Input → Parse Input Parameters

#### Node: Receive Telegram Input
- **Type / role:** `telegramTrigger` — entry point, triggers workflow on Telegram messages.
- **Configuration:**
  - Updates: `message`
  - Uses Telegram credentials (bot).
- **Input/Output:**
  - **Output:** Telegram update payload, notably `message.text` and `message.chat.id`.
- **Failure modes / edge cases:**
  - Bot not authorized / wrong token.
  - Webhook not reachable from Telegram (n8n URL/firewall).
  - Unexpected message type (non-text) may cause parsing issues downstream.

#### Node: Parse Input Parameters
- **Type / role:** `code` — parses message text into structured parameters.
- **Configuration choices (interpreted):**
  - Expects message format: `sector; limit; mapsUrl`
  - Splits on `;` and trims.
  - Converts `limit` via `parseInt`, defaults to `0`.
- **Key variables:**
  - `sector`, `limit`, `mapsUrl` returned as:
    - `json.sector`
    - `json.limit`
    - `json.mapsUrl`
- **Input/Output:**
  - **Input:** `Receive Telegram Input` output.
  - **Output:** single item with parsed JSON.
- **Failure modes / edge cases:**
  - If the user uses commas instead of semicolons, parsing will fail silently (sector becomes whole string; `mapsUrl` empty).
  - `limit=0` will propagate to Apify setting `maxCrawledPlacesPerSearch` (could yield no results or actor-specific default behavior).
  - Missing URL or malformed URL may cause Apify actor failure.

---

### Block 2 — Google Maps Scrape via Apify (Run Actor → Fetch Dataset)
**Overview:** Runs an Apify Google Maps scraping actor, then reads back the resulting dataset items using the returned `defaultDatasetId`.  
**Nodes involved:** Run Maps Scraper → Fetch Dataset Items

#### Node: Run Maps Scraper
- **Type / role:** `@apify/n8n-nodes-apify.apify` — runs an Apify Actor.
- **Configuration choices:**
  - Actor ID: `nwua9Gu5YrADL7ZDj` (selected from Apify actor list).
  - Custom actor input body includes:
    - `language: "fr"`
    - `maxCrawledPlacesPerSearch`: from `Parse Input Parameters` → `limit`
    - `searchStringsArray`: uses `sector`
    - `startUrls[0].url`: uses `mapsUrl`
    - Various scrape toggles set to `false`; `scrapeReviewsPersonalData: true` (note: this may increase sensitivity/collection scope depending on actor behavior).
- **Key expressions:**
  - `{{ $('Parse Input Parameters').item.json.limit }}`
  - `{{ $('Parse Input Parameters').item.json.sector }}`
  - `{{ $('Parse Input Parameters').item.json.mapsUrl }}`
- **Input/Output:**
  - **Input:** parsed parameters.
  - **Output:** Apify run result including `defaultDatasetId` (used next).
- **Failure modes / edge cases:**
  - Apify auth errors (invalid token, plan limits).
  - Actor run failures due to invalid `startUrls` or Google blocking/anti-bot.
  - If `limit` is very high, execution time/cost may spike; dataset can be large.

#### Node: Fetch Dataset Items
- **Type / role:** `@apify/n8n-nodes-apify.apify` — reads items from an Apify Dataset.
- **Configuration choices:**
  - Resource: `Datasets`
  - `datasetId`: `={{ $json.defaultDatasetId }}`
  - Limit/offset left empty (actor-dependent; may fetch default page size or all depending on node behavior/version).
- **Input/Output:**
  - **Input:** output of `Run Maps Scraper` (must contain `defaultDatasetId`).
  - **Output:** list of place items from dataset.
- **Failure modes / edge cases:**
  - Missing/undefined `defaultDatasetId` if actor fails or returns unexpected payload.
  - Large datasets may hit n8n memory/time constraints if fetched all at once.

---

### Block 3 — Normalize & Loop Through Places (Dedup → Split in Batches)
**Overview:** Removes duplicate places by title and processes places iteratively using batch looping.  
**Nodes involved:** Deduplicate Places → Process Each Place

#### Node: Deduplicate Places
- **Type / role:** `removeDuplicates` — reduces repeated entries.
- **Configuration choices:**
  - Compare mode: selected fields
  - Field to compare: `title`
- **Input/Output:**
  - **Input:** dataset items.
  - **Output:** dataset items with duplicates removed.
- **Failure modes / edge cases:**
  - Different businesses can share the same `title` → false deduplication (data loss).
  - Missing `title` field in some items could cause incorrect dedupe behavior.

#### Node: Process Each Place
- **Type / role:** `splitInBatches` — iterates through items, enabling a loop.
- **Configuration choices:**
  - Batch size not explicitly set (node default applies; often 1 unless configured otherwise).
- **Connections:**
  - Output 0 goes to two branches:
    1) `Send Done Notification` (this is unusual; see edge case below)
    2) `Generate Company Summary`
  - Receives loop-back from `Wait Rate Limit`.
- **Failure modes / edge cases:**
  - **Important:** With the current connections, `Send Done Notification` is triggered from `Process Each Place` output. Depending on batch behavior, it may send “DONE” at the start or for each batch rather than only once at the end. In many designs, DONE is sent after the final batch completes; here it is not gated by an “no items left” condition.
  - If batch size > 1, downstream nodes may receive multiple items, affecting rate limiting and sheet updates.

---

### Block 4 — AI Summary + Save Core Lead Data (OpenAI → Sheets)
**Overview:** Creates a professional one-paragraph company summary from Google Maps fields, then upserts the lead into Google Sheets (append or update by title).  
**Nodes involved:** Generate Company Summary → Upsert Places to Sheet

#### Node: Generate Company Summary
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — calls OpenAI chat model with JSON output enabled.
- **Configuration choices:**
  - Model: `gpt-5.1`
  - `jsonOutput: true`
  - System prompt: mandates one-paragraph professional summary; omit missing fields; no extra output.
  - User prompt: references fields like `title`, `categoryName`, `address`, `city`, `countryCode`, `phones`, `url`.
  - Final instruction: “Output the result in the JSON format companySummary”.
- **Key expressions/fields:**
  - Uses `$json.title`, `$json.categoryName`, `$json.address`, `$json.city`, `$json.countryCode`, `$json.phones`, `$json.url`.
- **Input/Output:**
  - **Input:** current place item from `Process Each Place`.
  - **Output:** AI response stored as `message.content` (and because JSON output is enabled, expected structure like `{ "companySummary": "..." }`).
- **Failure modes / edge cases:**
  - If the place schema doesn’t contain `phones` (often it’s `phone`), the summary may omit phone.
  - JSON output can fail if the model returns invalid JSON; downstream expressions attempt to handle both object and string.
  - OpenAI rate limits/timeouts.

#### Node: Upsert Places to Sheet
- **Type / role:** `googleSheets` — append or update lead row.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Document: “Google Maps Leads Enrichment” (spreadsheet ID set)
  - Sheet: “Leads” (gid=0)
  - Matching column: `title` (used as unique key)
  - Maps multiple fields from the place item, plus summary:
    - `companySummaryIn`: `{{ $json.message && $json.message.content && ($json.message.content.companySummary || $json.message.content) || '' }}`
    - `website`: `{{ $('Process Each Place').item.json.website.split('?')[0] }}`
    - `scrapedAt`: formatted substring of ISO datetime
    - `categories`: first 3 joined
- **Input/Output:**
  - **Input:** output of `Generate Company Summary` (AI response item).
  - **Output:** Google Sheets operation result; then proceeds to website enrichment.
- **Failure modes / edge cases:**
  - `website.split('?')[0]` will error if `website` is `null/undefined` (no null guard). This can break runs for places without a website.
  - Matching only on `title` risks collisions (same business names in different cities).
  - Google Sheets auth/permissions errors; schema mismatch if sheet headers differ.

---

### Block 5 — Website Email Enrichment (Set → IF → Jina → OpenAI → Sheets)
**Overview:** Extracts the website URL, checks it is non-empty, fetches website content via Jina AI, asks OpenAI to extract a single best email, then updates the Email column for the matching title row.  
**Nodes involved:** Select Website URL → Website URL Exists? → (Fetch Website Content → Extract Website Email → Update Email in Sheet) OR (Wait Rate Limit)

#### Node: Select Website URL
- **Type / role:** `set` — creates a normalized field for the website URL.
- **Configuration choices:**
  - Adds field `Site internet` = trimmed string version of website:
    - `={{ (($('Process Each Place').item.json.website || '') + '').trim() }}`
- **Input/Output:**
  - **Input:** output of `Upsert Places to Sheet`.
  - **Output:** same item with added `Site internet`.
- **Failure modes / edge cases:**
  - If earlier node failed due to missing website, this node never runs (but it is safer itself).

#### Node: Website URL Exists?
- **Type / role:** `if` — gate to skip website enrichment when URL missing.
- **Condition:**
  - String `notEmpty` on `{{ $json['Site internet'] }}`
- **Routing:**
  - **True:** Fetch Website Content
  - **False:** Wait Rate Limit (skip enrichment but continue loop)
- **Failure modes / edge cases:**
  - URL may be non-empty but invalid (will fail in fetch).
  - Some websites block bots; fetch may return error/empty content.

#### Node: Fetch Website Content
- **Type / role:** `jinaAi` — fetches and returns page content (often as Markdown/text).
- **Configuration choices:**
  - URL: `={{ $json['Site internet'] }}`
- **Input/Output:**
  - **Input:** item with `Site internet`.
  - **Output:** content payload, typically including a `content` field used by the next node.
- **Failure modes / edge cases:**
  - Jina credentials invalid or quota exceeded.
  - Timeout / 403 / bot protection; content may be empty or not contain emails.
  - Very large pages could be truncated.

#### Node: Extract Website Email
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — extracts one email address from fetched content.
- **Configuration choices:**
  - Model: `gpt-5.2`
  - Prompt instructs:
    - Return **only one** email, prefer authoritative (avoid info/support unless only choice)
    - If none: return exactly `Null`
    - Output must be only the email (no JSON)
  - Injects `{{ $json.content }}` (from Jina output).
- **Input/Output:**
  - **Input:** Jina fetch output.
  - **Output:** AI response as `message.content` (string).
- **Failure modes / edge cases:**
  - If Jina output field is not exactly `content`, prompt will contain blank → model returns `Null`.
  - “Null” string is written to sheet as text (not an actual null).
  - OpenAI rate limits/timeouts.

#### Node: Update Email in Sheet
- **Type / role:** `googleSheets` — updates existing row’s Email column using title as key.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `title`
  - Updates:
    - `Email` = `{{ $json.message && $json.message.content ? $json.message.content : '' }}`
    - `title` = `{{ $('Process Each Place').item.json.title }}`
- **Connections:**
  - After update → `Wait Rate Limit`
- **Failure modes / edge cases:**
  - If multiple rows share same title, update may affect unexpected row (depends on Sheets node behavior).
  - If the row does not exist (upsert failed earlier), update may fail or do nothing.
  - Writing “Null” will populate Email with literal “Null”.

---

### Block 6 — Rate Limiting + Loop + Notification
**Overview:** Adds a fixed delay between processed items and continues the batch loop; separately sends a Telegram DONE message (currently connected directly to the batch output).  
**Nodes involved:** Wait Rate Limit → Process Each Place, and Send Done Notification

#### Node: Wait Rate Limit
- **Type / role:** `wait` — throttles processing.
- **Configuration choices:**
  - Wait amount: `2` (seconds by default in this node)
- **Connections:**
  - After wait → loops back into `Process Each Place` input to fetch next batch/item.
- **Failure modes / edge cases:**
  - If execution is restarted/resumed, wait node behavior depends on n8n execution mode.
  - Rate limit may be insufficient for OpenAI/Sheets/Jina quotas in high volume.

#### Node: Send Done Notification
- **Type / role:** `telegram` — sends message back to the originating chat.
- **Configuration choices:**
  - Text: `DONE`
  - Chat ID: `={{ $('Receive Telegram Input').first().json.message.chat.id }}`
- **Input/Output:**
  - **Input:** from `Process Each Place` (current wiring).
  - **Output:** message send result.
- **Failure modes / edge cases:**
  - As wired, may send DONE prematurely or multiple times (per batch/item), because it is not conditioned on “no more items”.
  - Telegram API errors if bot can’t message that chat (privacy settings, bot removed).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Telegram Input | telegramTrigger | Entry point: receive Telegram message | — | Parse Input Parameters | ## Input & Scrape  \nReceive Telegram input (sector; limit; mapsUrl), parse parameters, run Apify Maps Scraper, fetch dataset items, and deduplicate places. |
| Parse Input Parameters | code | Parse `sector;limit;mapsUrl` into JSON | Receive Telegram Input | Run Maps Scraper | ## Input & Scrape  \nReceive Telegram input (sector; limit; mapsUrl), parse parameters, run Apify Maps Scraper, fetch dataset items, and deduplicate places. |
| Run Maps Scraper | apify | Run Apify Google Maps scraping actor | Parse Input Parameters | Fetch Dataset Items | ## Input & Scrape  \nReceive Telegram input (sector; limit; mapsUrl), parse parameters, run Apify Maps Scraper, fetch dataset items, and deduplicate places. |
| Fetch Dataset Items | apify | Fetch items from Apify dataset | Run Maps Scraper | Deduplicate Places | ## Input & Scrape  \nReceive Telegram input (sector; limit; mapsUrl), parse parameters, run Apify Maps Scraper, fetch dataset items, and deduplicate places. |
| Deduplicate Places | removeDuplicates | Remove duplicate places by title | Fetch Dataset Items | Process Each Place | ## Input & Scrape  \nReceive Telegram input (sector; limit; mapsUrl), parse parameters, run Apify Maps Scraper, fetch dataset items, and deduplicate places. |
| Process Each Place | splitInBatches | Iterate through places (loop control) | Deduplicate Places, Wait Rate Limit | Send Done Notification; Generate Company Summary | ## Enrich & Save  \nProcess each place, generate a company summary with GPT, then upsert core place data into Google Sheets. |
| Generate Company Summary | openAi (LangChain) | Create 1-paragraph company summary (JSON output) | Process Each Place | Upsert Places to Sheet | ## Enrich & Save  \nProcess each place, generate a company summary with GPT, then upsert core place data into Google Sheets. |
| Upsert Places to Sheet | googleSheets | Append or update lead record (match by title) | Generate Company Summary | Select Website URL | ## Enrich & Save  \nProcess each place, generate a company summary with GPT, then upsert core place data into Google Sheets. |
| Select Website URL | set | Normalize website field to `Site internet` | Upsert Places to Sheet | Website URL Exists? | ## Website Email Enrichment  \nSelect website URL, check if it exists, fetch page content via Jina, extract email with GPT, then update Email in Google Sheets. |
| Website URL Exists? | if | Branch: website present? | Select Website URL | Fetch Website Content (true); Wait Rate Limit (false) | ## Website Email Enrichment  \nSelect website URL, check if it exists, fetch page content via Jina, extract email with GPT, then update Email in Google Sheets. |
| Fetch Website Content | jinaAi | Retrieve website content (Markdown/text) | Website URL Exists? (true) | Extract Website Email | ## Website Email Enrichment  \nSelect website URL, check if it exists, fetch page content via Jina, extract email with GPT, then update Email in Google Sheets. |
| Extract Website Email | openAi (LangChain) | Extract best contact email from website content | Fetch Website Content | Update Email in Sheet | ## Website Email Enrichment  \nSelect website URL, check if it exists, fetch page content via Jina, extract email with GPT, then update Email in Google Sheets. |
| Update Email in Sheet | googleSheets | Update Email column for matching title | Extract Website Email | Wait Rate Limit | ## Website Email Enrichment  \nSelect website URL, check if it exists, fetch page content via Jina, extract email with GPT, then update Email in Google Sheets. |
| Wait Rate Limit | wait | Delay between items; loop back | Website URL Exists? (false), Update Email in Sheet | Process Each Place | ## Loop Control & Notify  \nApply rate-limit wait between items and send final DONE notification when processing completes. |
| Send Done Notification | telegram | Send DONE message to Telegram chat | Process Each Place | — | ## Loop Control & Notify  \nApply rate-limit wait between items and send final DONE notification when processing completes. |
| Sticky Note | stickyNote | Documentation block (no execution) | — | — | ## Google Maps Leads Enrichment via Telegram\n\n### How it works\n1. Receive a Telegram message with three values (sector; limit; mapsUrl) to trigger a run.\n2. Run a Google Maps scraper (Apify) using the provided URL and sector, fetch results and remove duplicate places.\n3. Process each place: generate a human-ready company summary with the AI model and append or update rows in your Google Sheet.\n4. If a place has a website, fetch its HTML, call the AI to extract a primary contact email, and update the sheet row.\n5. Respect rate limits between requests and send a DONE message back to the original Telegram chat when processing finishes.\n\n### Setup\n- [ ] Connect the Telegram bot and confirm it can send messages to the workflow.\n- [ ] Add Apify API credentials and configure the Maps scraper actor.\n- [ ] Authorize Google Sheets and set the spreadsheet ID and sheet name.\n- [ ] Add OpenAI API key for summaries and email extraction.\n- [ ] Enable website fetch (JinaAI or similar) and verify access to target sites.\n- [ ] Set rate-limit wait seconds and run a small test. |
| Sticky Note1 | stickyNote | Section label (no execution) | — | — | ## Input & Scrape  \nReceive Telegram input (sector; limit; mapsUrl), parse parameters, run Apify Maps Scraper, fetch dataset items, and deduplicate places. |
| Sticky Note2 | stickyNote | Section label (no execution) | — | — | ## Enrich & Save  \nProcess each place, generate a company summary with GPT, then upsert core place data into Google Sheets. |
| Sticky Note3 | stickyNote | Section label (no execution) | — | — | ## Website Email Enrichment  \nSelect website URL, check if it exists, fetch page content via Jina, extract email with GPT, then update Email in Google Sheets. |
| Sticky Note4 | stickyNote | Section label (no execution) | — | — | ## Loop Control & Notify  \nApply rate-limit wait between items and send final DONE notification when processing completes. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger**
   - Add node: **Telegram Trigger**
   - Updates: `message`
   - Create/connect Telegram credentials (bot token).
   - Note your bot must be able to receive messages (webhook reachable).

2. **Add “Parse Input Parameters” (Code)**
   - Add node: **Code**
   - Paste logic to split `message.text` by `;` into:
     - `sector` (string)
     - `limit` (integer)
     - `mapsUrl` (string)
   - Connect: Telegram Trigger → Code.

3. **Add Apify “Run Maps Scraper”**
   - Add node: **Apify** (resource: Actor run)
   - Select Actor ID: `nwua9Gu5YrADL7ZDj` (the Google Maps scraper used in the workflow).
   - Build actor input (custom body) with:
     - `language: "fr"`
     - `maxCrawledPlacesPerSearch: {{$('Parse Input Parameters').item.json.limit}}`
     - `searchStringsArray: ["{{$('Parse Input Parameters').item.json.sector}}"]`
     - `startUrls: [{ url: "{{$('Parse Input Parameters').item.json.mapsUrl}}" }]`
     - Other toggles as in your desired scope.
   - Connect: Code → Apify (Run).

4. **Add Apify “Fetch Dataset Items”**
   - Add node: **Apify** (resource: Datasets → Get items / fetch items)
   - Set Dataset ID to: `{{$json.defaultDatasetId}}`
   - Connect: Run Maps Scraper → Fetch Dataset Items.

5. **Add Deduplication**
   - Add node: **Remove Duplicates**
   - Compare: `selectedFields`
   - Fields to compare: `title`
   - Connect: Fetch Dataset Items → Deduplicate Places.

6. **Add Looping**
   - Add node: **Split in Batches**
   - Keep default options (or set batch size explicitly if desired).
   - Connect: Deduplicate Places → Split in Batches.

7. **Add OpenAI “Generate Company Summary”**
   - Add node: **OpenAI (LangChain)** / “Chat Model” node as available in your n8n version.
   - Model: `gpt-5.1`
   - Enable JSON output.
   - Prompt: system + user prompt using fields from the incoming place item (title/category/address/etc.), and instruct output as JSON with `companySummary`.
   - Connect: Split in Batches → Generate Company Summary.

8. **Add Google Sheets “Upsert Places to Sheet”**
   - Add node: **Google Sheets**
   - Credentials: OAuth2 (Google) with access to the target spreadsheet.
   - Operation: **Append or Update**
   - Spreadsheet ID: your target spreadsheet
   - Sheet: “Leads”
   - Matching column(s): `title`
   - Map columns (examples):
     - `title`, `phone`, `address`, `city`, `url`, etc. from the place item
     - `companySummaryIn` from the OpenAI response:
       - Prefer `message.content.companySummary`, fallback to `message.content`
   - Connect: Generate Company Summary → Upsert Places to Sheet.

9. **Add “Select Website URL” (Set)**
   - Add node: **Set**
   - Create field `Site internet` = trimmed website string from the original place item (use a null-safe expression).
   - Connect: Upsert Places to Sheet → Select Website URL.

10. **Add IF “Website URL Exists?”**
    - Add node: **IF**
    - Condition: `Site internet` is not empty
    - Connect: Select Website URL → IF.

11. **Add Jina AI “Fetch Website Content”**
    - Add node: **Jina AI**
    - Credentials: Jina AI API key
    - URL: `{{$json["Site internet"]}}`
    - Connect: IF (true) → Fetch Website Content.

12. **Add OpenAI “Extract Website Email”**
    - Add node: **OpenAI (LangChain)**
    - Model: `gpt-5.2`
    - Prompt includes `{{$json.content}}` from Jina output and strict output rules (single email or `Null`).
    - Connect: Fetch Website Content → Extract Website Email.

13. **Add Google Sheets “Update Email in Sheet”**
    - Add node: **Google Sheets**
    - Operation: **Update**
    - Match by `title`
    - Update `Email` with the model output (`message.content`)
    - Connect: Extract Website Email → Update Email in Sheet.

14. **Add Wait “Wait Rate Limit”**
    - Add node: **Wait**
    - Amount: `2` seconds (or your preferred throttle)
    - Connect:
      - IF (false) → Wait
      - Update Email in Sheet → Wait

15. **Close the loop**
    - Connect: Wait → Split in Batches (so it processes the next item).

16. **Add Telegram “Send Done Notification”**
    - Add node: **Telegram**
    - Text: `DONE`
    - Chat ID: `{{$('Receive Telegram Input').first().json.message.chat.id}}`
    - **Important:** To send DONE only once after the last item, add logic to detect end-of-batch (commonly: use Split in Batches second output or an IF checking “no items left”). The provided workflow connects DONE directly from the batch output, which can cause repeated/premature DONE messages.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google Maps Leads Enrichment via Telegram — How it works + Setup checklist (Telegram → Apify → AI summary → Sheets → Website email extraction → Rate limit → DONE) | Embedded sticky note text in the workflow canvas |
| Input format expected: `sector; limit; mapsUrl` | Telegram message parsing logic and sticky notes |
| Risk: upsert/update matching only on `title` can collide across cities | Google Sheets matching configuration |
| Potential runtime bug: `website.split('?')[0]` without null guard | Upsert Places to Sheet mapping expression |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.