Enrich B2B Google Maps leads with Apify, Jina AI, GPT-5.2 and Google Sheets

https://n8nworkflows.xyz/workflows/enrich-b2b-google-maps-leads-with-apify--jina-ai--gpt-5-2-and-google-sheets-13582


# Enrich B2B Google Maps leads with Apify, Jina AI, GPT-5.2 and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *Enrich Google Maps business leads using Apify, GPT-5.2 and Google Sheets*  
**Purpose:** Triggered from Telegram, this workflow scrapes Google Maps places via Apify, deduplicates them, generates a concise company summary with an OpenAI model, upserts results into Google Sheets, and (when a website exists) fetches the website content via Jina AI and extracts a primary contact email with OpenAI to update the same sheet. It rate-limits per item and sends a final “DONE” message back to Telegram.

### 1.1 Input Reception & Parameter Parsing
Receives a Telegram message containing `sector; limit; mapsUrl`, parses it into structured parameters used by the scraper.

### 1.2 Google Maps Scrape (Apify) + Dataset Retrieval
Runs the Apify Google Maps scraper actor using parsed parameters, then fetches the actor’s dataset items.

### 1.3 Deduplication & Per-Place Loop
Removes duplicate places by title and iterates through places one by one.

### 1.4 AI Summary Enrichment + Google Sheets Upsert
Generates a natural-language company summary from each place’s Maps data, then appends or updates a row in Google Sheets keyed by the place title.

### 1.5 Website Email Enrichment (Conditional)
If a website URL exists, fetches the website content via Jina AI, asks OpenAI to extract a single best email, and updates the corresponding Google Sheets row.

### 1.6 Loop Control & Completion Notification
Waits between items to respect rate limits and ends by notifying the original Telegram chat with “DONE”.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Parameter Parsing
**Overview:** Starts the workflow from Telegram and converts the message text into parameters used throughout the run.  
**Nodes involved:** Receive Telegram Input → Parse Input Parameters

#### Node: Receive Telegram Input
- **Type / role:** `telegramTrigger` — workflow entry point; listens for Telegram messages.
- **Configuration (interpreted):**
  - Listens to `message` updates.
  - Uses Telegram bot credentials `Piranusa_assistantbot`.
- **Inputs / outputs:** No input; outputs Telegram update JSON (includes `message.text`, `chat.id`, etc.) to **Parse Input Parameters**.
- **Edge cases / failures:**
  - Bot not authorized or webhook misconfigured → trigger never fires.
  - Telegram message without `text` (photo, sticker) → downstream parsing may fail.

#### Node: Parse Input Parameters
- **Type / role:** `code` — parses the Telegram message.
- **Configuration (interpreted):**
  - Reads: `$input.first().json.message.text`
  - Splits by semicolon `;` (note: comment says comma-separated, but code uses `;`).
  - Outputs JSON:
    - `sector` = parts[0] or `''`
    - `limit` = `parseInt(parts[1])` or `0`
    - `mapsUrl` = parts[2] or `''`
- **Key variables/expressions:**
  - `const parts = messageText.split(';').map(part => part.trim());`
- **Outputs to:** **Run Maps Scraper**
- **Edge cases / failures:**
  - Missing semicolons or fewer than 3 parts → empty values; scraper may run with empty `startUrls`/search string.
  - `limit` non-numeric → becomes `0`; may yield 0 results depending on actor behavior.
  - If `message.text` is undefined → runtime error in code node.

---

### Block 2 — Google Maps Scrape (Apify) + Dataset Retrieval
**Overview:** Launches an Apify actor for Google Maps scraping and downloads the resulting dataset items.  
**Nodes involved:** Run Maps Scraper → Fetch Dataset Items

#### Node: Run Maps Scraper
- **Type / role:** `@apify/n8n-nodes-apify.apify` — runs an Apify Actor.
- **Configuration (interpreted):**
  - Actor ID: `nwua9Gu5YrADL7ZDj` (Google Maps scraper).
  - Custom actor input body includes:
    - `language: "fr"`
    - `maxCrawledPlacesPerSearch`: from `Parse Input Parameters.limit`
    - `searchStringsArray`: contains `Parse Input Parameters.sector`
    - `startUrls[0].url`: from `Parse Input Parameters.mapsUrl`
    - Many scrape/enrichment flags disabled (contacts, directories, place detail page off; reviews personal data on).
- **Key expressions/variables:**
  - `{{ $('Parse Input Parameters').item.json.limit }}`
  - `{{ $('Parse Input Parameters').item.json.sector }}`
  - `{{ $('Parse Input Parameters').item.json.mapsUrl }}`
- **Outputs to:** **Fetch Dataset Items**
- **Edge cases / failures:**
  - Apify auth failure or quota exhaustion.
  - Invalid `mapsUrl` → actor may error or return empty dataset.
  - `maxCrawledPlacesPerSearch = 0` might produce no items.
  - Actor input schema changes (Apify actor updates) can break expected behavior.
- **Version requirements:** Node typeVersion `1` (Apify node); ensure the Apify community node package is installed/enabled.

#### Node: Fetch Dataset Items
- **Type / role:** `@apify/n8n-nodes-apify.apify` — reads items from an Apify dataset.
- **Configuration (interpreted):**
  - Resource: `Datasets`
  - Dataset ID: `={{ $json.defaultDatasetId }}` (from actor run output)
- **Outputs to:** **Deduplicate Places**
- **Edge cases / failures:**
  - If the actor run output does not include `defaultDatasetId`, this expression resolves empty → dataset fetch fails.
  - Large datasets may require paging; here `limit`/`offset` are blank, so defaults apply (may not fetch everything depending on node defaults).

---

### Block 3 — Deduplication & Per-Place Loop
**Overview:** Cleans the dataset by removing duplicates and then processes items iteratively to control rate limits and downstream writes.  
**Nodes involved:** Deduplicate Places → Process Each Place

#### Node: Deduplicate Places
- **Type / role:** `removeDuplicates` — removes duplicate place entries.
- **Configuration (interpreted):**
  - Compare mode: selected fields
  - Field compared: `title`
- **Outputs to:** **Process Each Place**
- **Edge cases / failures:**
  - Places with same `title` but different locations will be collapsed incorrectly.
  - Places missing `title` could all be treated as duplicates or pass through depending on node behavior.

#### Node: Process Each Place
- **Type / role:** `splitInBatches` — loop controller.
- **Configuration (interpreted):**
  - No explicit batch size shown (defaults apply; typically 1 item per batch unless configured).
- **Connections:**
  - Output 1 → **Send Done Notification** (this is unusual; see edge case)
  - Output 2 → **Generate Company Summary**
  - Input from **Deduplicate Places**
  - Receives loop continuation from **Wait Rate Limit**
- **Edge cases / failures:**
  - In n8n, `SplitInBatches` commonly sends each batch on output 1 and “no items left” on output 2. This workflow appears to treat output 2 as the per-item path (to Generate Company Summary), which may invert expected behavior depending on exact node version/config defaults.
  - If the outputs are reversed, you may send “DONE” for every item or never generate summaries.
  - Ensure the node is configured/behaves as intended in your n8n version (typeVersion `3`).

---

### Block 4 — AI Summary Enrichment + Google Sheets Upsert
**Overview:** Uses OpenAI to create a clean summary paragraph for each place, then writes core place data + summary into Google Sheets (append or update).  
**Nodes involved:** Generate Company Summary → Upsert Places to Sheet

#### Node: Generate Company Summary
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — LLM call to generate a company summary.
- **Configuration (interpreted):**
  - Model: `gpt-5.1`
  - System prompt instructs: professional, one paragraph, omit missing fields, output only final text.
  - User prompt injects place fields: `title`, `categoryName`, `address`, `city`, `countryCode`, `phones`, `url`.
  - `jsonOutput: true` and prompt requests: “Output the result in the JSON format companySummary”.
- **Key expressions:**
  - Uses `$json.title`, `$json.categoryName`, etc. from the place item.
- **Outputs to:** **Upsert Places to Sheet**
- **Edge cases / failures:**
  - Source field mismatch: prompt uses `{{ $json.phones }}` but upstream data appears to have `phone` / `phoneUnformatted`. If `phones` doesn’t exist, phone will be omitted.
  - JSON output enforcement may fail if model responds with invalid JSON → downstream extraction may break.
  - Model availability / OpenAI quota issues.

#### Node: Upsert Places to Sheet
- **Type / role:** `googleSheets` — upsert (append or update) a lead row.
- **Configuration (interpreted):**
  - Operation: `appendOrUpdate`
  - Spreadsheet: “Google Maps Leads Enrichment” (`documentId` set)
  - Sheet tab: “Leads” (`gid=0`)
  - Matching columns: `title` (used as unique key)
  - Writes many columns from `Process Each Place` item fields, plus summary from the AI output:
    - `website` is normalized with `.split('?')[0]`
    - `scrapedAt` formatted with substring and `T` replacement
    - `categories` takes first 3 joined by comma
    - `companySummaryIn` uses:
      - `($json.message.content.companySummary || $json.message.content)` if present
- **Key expressions / variables:**
  - Place data references use: `$('Process Each Place').item.json.<field>`
  - Summary reference uses current node input `$json.message.content...` (from OpenAI node output)
- **Outputs to:** **Select Website URL**
- **Edge cases / failures:**
  - Matching on `title` may overwrite distinct businesses with same name.
  - If `website` is empty or not a string, `.split('?')[0]` can throw; current expression assumes string exists. (It does not guard like `(website || '')`.)
  - Schema mismatch between sheet headers and defined columns causes write failures.
  - Google Sheets API rate limits; large runs may error without backoff.

---

### Block 5 — Website Email Enrichment (Conditional)
**Overview:** If a lead has a website, fetches website content with Jina AI, asks OpenAI to extract one best email, then updates the existing row in Sheets.  
**Nodes involved:** Select Website URL → Website URL Exists? → (true) Fetch Website Content → Extract Website Email → Update Email in Sheet

#### Node: Select Website URL
- **Type / role:** `set` — creates a normalized “Site internet” field.
- **Configuration (interpreted):**
  - Sets `Site internet` to trimmed string of the place website:  
    `={{ (($('Process Each Place').item.json.website || '') + '').trim() }}`
- **Outputs to:** **Website URL Exists?**
- **Edge cases / failures:**
  - None major; this is defensive (forces string).

#### Node: Website URL Exists?
- **Type / role:** `if` — conditional branch.
- **Configuration (interpreted):**
  - Condition: `Site internet` is **not empty**.
  - True path → **Fetch Website Content**
  - False path → **Wait Rate Limit** (skip enrichment, continue loop)
- **Edge cases / failures:**
  - URLs with whitespace only are trimmed upstream; good.
  - Non-HTTP URLs still pass “not empty” and may fail fetch.

#### Node: Fetch Website Content
- **Type / role:** `jinaAi` — fetches page content as markdown/text.
- **Configuration (interpreted):**
  - URL: `={{ $json['Site internet'] }}`
  - Uses Jina AI credentials `aiformotu`
- **Outputs to:** **Extract Website Email**
- **Edge cases / failures:**
  - Website blocks bots, returns 403/429, or heavy pages time out.
  - Some sites may return little content; email extraction will yield `Null`.
  - Jina AI API limits/quotas.

#### Node: Extract Website Email
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — LLM email extraction.
- **Configuration (interpreted):**
  - Model: `gpt-5.2`
  - Prompt: Extract exactly one best email from provided markdown content; if none return exactly `Null`; output only the email or `Null`.
  - Injects: `{{ $json.content }}` (from Jina node output)
- **Outputs to:** **Update Email in Sheet**
- **Edge cases / failures:**
  - Model may output `Null` vs `null` or extra whitespace; downstream writes as-is.
  - If Jina output field is not `content`, the prompt will be empty → likely `Null`.

#### Node: Update Email in Sheet
- **Type / role:** `googleSheets` — updates the existing row’s Email.
- **Configuration (interpreted):**
  - Operation: `update`
  - Matching columns: `title`
  - Sets:
    - `Email` from OpenAI output: `={{ $json.message.content }}`
    - `title` from current place: `={{ $('Process Each Place').item.json.title }}`
- **Outputs to:** **Wait Rate Limit**
- **Edge cases / failures:**
  - If the “upsert” step didn’t create/identify the row correctly, update may fail or update the wrong row (again due to matching only on title).
  - If the model returns `Null`, you will store literal text “Null” in the sheet.

---

### Block 6 — Loop Control & Completion Notification
**Overview:** Rate-limits processing and sends a completion message to Telegram once the loop ends.  
**Nodes involved:** Wait Rate Limit → Process Each Place (loop) and Send Done Notification (final)

#### Node: Wait Rate Limit
- **Type / role:** `wait` — delays before continuing loop.
- **Configuration (interpreted):**
  - Wait amount: `2` (seconds by default in this node family/version)
- **Outputs to:** **Process Each Place** (to request the next item/batch)
- **Edge cases / failures:**
  - Too short for external API quotas (Sheets/OpenAI/Jina/Apify) → intermittent 429 errors.
  - Wait node resumes via internal execution; if n8n restarts, waits may be interrupted depending on execution mode/settings.

#### Node: Send Done Notification
- **Type / role:** `telegram` — sends completion notification.
- **Configuration (interpreted):**
  - Text: `DONE`
  - chatId: `={{ $('Receive Telegram Input').first().json.message.chat.id }}`
- **Inputs / outputs:** Receives from **Process Each Place** output 1.
- **Edge cases / failures:**
  - As noted earlier, if `SplitInBatches` outputs are misinterpreted, this may send “DONE” prematurely or repeatedly.
  - Telegram send can fail if bot blocked by user/chat, or credentials invalid.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Telegram Input | telegramTrigger | Entry point: receive `sector; limit; mapsUrl` | — | Parse Input Parameters | ## Input & Scrape  \n  Receive Telegram input (sector; limit; mapsUrl), parse parameters, run Apify Maps Scraper, fetch dataset items, and deduplicate places. |
| Parse Input Parameters | code | Parse Telegram message into structured params | Receive Telegram Input | Run Maps Scraper | ## Input & Scrape  \n  Receive Telegram input (sector; limit; mapsUrl), parse parameters, run Apify Maps Scraper, fetch dataset items, and deduplicate places. |
| Run Maps Scraper | apify | Run Apify Google Maps scraper actor | Parse Input Parameters | Fetch Dataset Items | ## Input & Scrape  \n  Receive Telegram input (sector; limit; mapsUrl), parse parameters, run Apify Maps Scraper, fetch dataset items, and deduplicate places. |
| Fetch Dataset Items | apify | Download results dataset from Apify run | Run Maps Scraper | Deduplicate Places | ## Input & Scrape  \n  Receive Telegram input (sector; limit; mapsUrl), parse parameters, run Apify Maps Scraper, fetch dataset items, and deduplicate places. |
| Deduplicate Places | removeDuplicates | Remove duplicate places by title | Fetch Dataset Items | Process Each Place | ## Input & Scrape  \n  Receive Telegram input (sector; limit; mapsUrl), parse parameters, run Apify Maps Scraper, fetch dataset items, and deduplicate places. |
| Process Each Place | splitInBatches | Iterate through places one by one / loop control | Deduplicate Places; Wait Rate Limit | Send Done Notification; Generate Company Summary | ## Enrich & Save  \n  Process each place, generate a company summary with GPT, then upsert core place data into Google Sheets. |
| Generate Company Summary | openAi (LangChain) | Create 1-paragraph summary from Maps data | Process Each Place | Upsert Places to Sheet | ## Enrich & Save  \n  Process each place, generate a company summary with GPT, then upsert core place data into Google Sheets. |
| Upsert Places to Sheet | googleSheets | Append-or-update core lead data + summary | Generate Company Summary | Select Website URL | ## Enrich & Save  \n  Process each place, generate a company summary with GPT, then upsert core place data into Google Sheets. |
| Select Website URL | set | Normalize website URL into `Site internet` | Upsert Places to Sheet | Website URL Exists? | ## Website Email Enrichment  \n  Select website URL, check if it exists, fetch page content via Jina, extract email with GPT, then update Email in Google Sheets. |
| Website URL Exists? | if | Branch: only enrich email if website exists | Select Website URL | Fetch Website Content (true); Wait Rate Limit (false) | ## Website Email Enrichment  \n  Select website URL, check if it exists, fetch page content via Jina, extract email with GPT, then update Email in Google Sheets. |
| Fetch Website Content | jinaAi | Fetch website page content for extraction | Website URL Exists? (true) | Extract Website Email | ## Website Email Enrichment  \n  Select website URL, check if it exists, fetch page content via Jina, extract email with GPT, then update Email in Google Sheets. |
| Extract Website Email | openAi (LangChain) | Extract a single best email from website content | Fetch Website Content | Update Email in Sheet | ## Website Email Enrichment  \n  Select website URL, check if it exists, fetch page content via Jina, extract email with GPT, then update Email in Google Sheets. |
| Update Email in Sheet | googleSheets | Update Email column for matching title | Extract Website Email | Wait Rate Limit | ## Website Email Enrichment  \n  Select website URL, check if it exists, fetch page content via Jina, extract email with GPT, then update Email in Google Sheets. |
| Wait Rate Limit | wait | Delay between items; continue loop | Website URL Exists? (false); Update Email in Sheet | Process Each Place | ## Loop Control & Notify  \n  Apply rate-limit wait between items and send final DONE notification when processing completes. |
| Send Done Notification | telegram | Notify Telegram chat on completion | Process Each Place | — | ## Loop Control & Notify  \n  Apply rate-limit wait between items and send final DONE notification when processing completes. |
| Sticky Note | stickyNote | Documentation / on-canvas notes | — | — | ## Google Maps Leads Enrichment via Telegram\n\n### How it works\n1. Receive a Telegram message with three values (sector; limit; mapsUrl) to trigger a run.\n2. Run a Google Maps scraper (Apify) using the provided URL and sector, fetch results and remove duplicate places.\n3. Process each place: generate a human-ready company summary with the AI model and append or update rows in your Google Sheet.\n4. If a place has a website, fetch its HTML, call the AI to extract a primary contact email, and update the sheet row.\n5. Respect rate limits between requests and send a DONE message back to the original Telegram chat when processing finishes.\n\n### Setup\n- [ ] Connect the Telegram bot and confirm it can send messages to the workflow.\n- [ ] Add Apify API credentials and configure the Maps scraper actor.\n- [ ] Authorize Google Sheets and set the spreadsheet ID and sheet name.\n- [ ] Add OpenAI API key for summaries and email extraction.\n- [ ] Enable website fetch (JinaAI or similar) and verify access to target sites.\n- [ ] Set rate-limit wait seconds and run a small test. |
| Sticky Note1 | stickyNote | Documentation / on-canvas notes | — | — | ## Input & Scrape  \n  Receive Telegram input (sector; limit; mapsUrl), parse parameters, run Apify Maps Scraper, fetch dataset items, and deduplicate places. |
| Sticky Note2 | stickyNote | Documentation / on-canvas notes | — | — | ## Enrich & Save  \n  Process each place, generate a company summary with GPT, then upsert core place data into Google Sheets. |
| Sticky Note3 | stickyNote | Documentation / on-canvas notes | — | — | ## Website Email Enrichment  \n  Select website URL, check if it exists, fetch page content via Jina, extract email with GPT, then update Email in Google Sheets. |
| Sticky Note4 | stickyNote | Documentation / on-canvas notes | — | — | ## Loop Control & Notify  \n  Apply rate-limit wait between items and send final DONE notification when processing completes. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Enrich Google Maps business leads using Apify, GPT-5.2 and Google Sheets*.

2. **Add Telegram Trigger**
   - Node: **Telegram Trigger** (`Receive Telegram Input`)
   - Updates: `message`
   - Credentials: connect your Telegram bot token (Telegram API credential).

3. **Add Code node to parse message**
   - Node: **Code** (`Parse Input Parameters`)
   - Paste logic (adapted): split incoming `message.text` by `;` into:
     - `sector` (string)
     - `limit` (int)
     - `mapsUrl` (string)
   - Ensure it returns one item with `{ sector, limit, mapsUrl }`.
   - Connect: `Receive Telegram Input` → `Parse Input Parameters`.

4. **Add Apify node to run the Maps scraper**
   - Node: **Apify** (`Run Maps Scraper`) using the Apify community node.
   - Operation: Run Actor
   - Actor: select the Google Maps scraper actor (ID `nwua9Gu5YrADL7ZDj` in this workflow).
   - Build actor input JSON using expressions:
     - `maxCrawledPlacesPerSearch` = `Parse Input Parameters.limit`
     - `searchStringsArray` includes `Parse Input Parameters.sector`
     - `startUrls[0].url` = `Parse Input Parameters.mapsUrl`
     - Keep other flags as desired (contacts/reviews/detail page).
   - Credentials: Apify API token.
   - Connect: `Parse Input Parameters` → `Run Maps Scraper`.

5. **Add Apify node to fetch dataset items**
   - Node: **Apify** (`Fetch Dataset Items`)
   - Resource: Datasets → Fetch items (or equivalent “Get dataset items”)
   - Dataset ID expression: `{{ $json.defaultDatasetId }}`
   - Connect: `Run Maps Scraper` → `Fetch Dataset Items`.

6. **Add Remove Duplicates**
   - Node: **Remove Duplicates** (`Deduplicate Places`)
   - Compare: Selected Fields
   - Fields: `title`
   - Connect: `Fetch Dataset Items` → `Deduplicate Places`.

7. **Add Split In Batches for iteration**
   - Node: **Split In Batches** (`Process Each Place`)
   - Set batch size explicitly to **1** (recommended to avoid ambiguity).
   - Connect: `Deduplicate Places` → `Process Each Place`.

8. **Add OpenAI node for company summary**
   - Node: **OpenAI (LangChain)** (`Generate Company Summary`)
   - Model: `gpt-5.1` (or equivalent available model)
   - Provide:
     - System instructions: one-paragraph professional summary, omit missing fields.
     - User message with place fields (title, categoryName, address, city, countryCode, phone, url).
   - Enable JSON output if you want structured output, and ensure prompt matches (e.g., return `{"companySummary": "..."}`).
   - Credentials: OpenAI API key.
   - Connect: `Process Each Place` (per-item output) → `Generate Company Summary`.

9. **Add Google Sheets upsert**
   - Node: **Google Sheets** (`Upsert Places to Sheet`)
   - Operation: **Append or Update**
   - Select Spreadsheet + Sheet (your “Leads” tab).
   - Set matching column(s): `title`
   - Map columns from the place item (url, city, rank, phone, title, address, etc.) and map summary from the OpenAI output.
   - Credentials: Google Sheets OAuth2.
   - Connect: `Generate Company Summary` → `Upsert Places to Sheet`.

10. **Add Set node to select website**
    - Node: **Set** (`Select Website URL`)
    - Create field `Site internet` = trimmed website string (guard with `|| ''`).
    - Connect: `Upsert Places to Sheet` → `Select Website URL`.

11. **Add IF node to check website presence**
    - Node: **IF** (`Website URL Exists?`)
    - Condition: `Site internet` is not empty.
    - Connect: `Select Website URL` → `Website URL Exists?`.

12. **Add Jina AI “Fetch Website Content”**
    - Node: **Jina AI** (`Fetch Website Content`)
    - URL: `{{ $json["Site internet"] }}`
    - Credentials: Jina AI API key.
    - Connect: IF **true** → `Fetch Website Content`.

13. **Add OpenAI node to extract email**
    - Node: **OpenAI (LangChain)** (`Extract Website Email`)
    - Model: `gpt-5.2`
    - Prompt: “Return only one email or exactly `Null`.”
    - Provide website content variable from Jina output (commonly `$json.content`).
    - Connect: `Fetch Website Content` → `Extract Website Email`.

14. **Add Google Sheets update (Email)**
    - Node: **Google Sheets** (`Update Email in Sheet`)
    - Operation: **Update**
    - Match column(s): `title`
    - Set `title` from the current place item, and `Email` from OpenAI output.
    - Connect: `Extract Website Email` → `Update Email in Sheet`.

15. **Add Wait node for rate limiting**
    - Node: **Wait** (`Wait Rate Limit`)
    - Wait: 2 seconds (adjust as needed).
    - Connect:
      - IF **false** → `Wait Rate Limit`
      - `Update Email in Sheet` → `Wait Rate Limit`

16. **Close the loop**
    - Connect: `Wait Rate Limit` → `Process Each Place` (to continue to next item).

17. **Add Telegram “DONE” notification**
    - Node: **Telegram** (`Send Done Notification`)
    - Chat ID: `{{ $("Receive Telegram Input").first().json.message.chat.id }}`
    - Text: `DONE`
    - Connect this node to the **“no items left”** output of `Split In Batches` (ensure you wire the correct output for your n8n version).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Telegram message format required: `sector; limit; mapsUrl` | Used by **Parse Input Parameters** and described in sticky note. |
| Workflow logic summary + setup checklist (Telegram, Apify, Sheets, OpenAI, Jina, rate limit) | From the canvas note “Google Maps Leads Enrichment via Telegram”. |
| Deduplication and row matching use `title` as the unique key | Can overwrite different businesses with identical names; consider matching on `url` (Maps URL) instead. |
| Potential loop wiring ambiguity with SplitInBatches outputs | Verify “per-item” vs “done” outputs in your n8n version; wire “DONE” only on completion. |