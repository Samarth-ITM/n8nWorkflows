Send daily Malaysian weather alerts with Perplexity AI, Firecrawl and Telegram

https://n8nworkflows.xyz/workflows/send-daily-malaysian-weather-alerts-with-perplexity-ai--firecrawl-and-telegram-12232


# Send daily Malaysian weather alerts with Perplexity AI, Firecrawl and Telegram

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Send a daily English weather alert report for Malaysia to a Telegram channel/group. The workflow pulls official warnings from **data.gov.my**, uses an AI search agent (Perplexity Sonar Pro via OpenRouter) to find recent related news links, scrapes/summarizes those URLs via **Firecrawl**, refines the final report with **OpenAI**, then posts it to **Telegram**.

**Target use cases:**
- Daily operational weather/flood awareness briefings
- Automated monitoring of official warnings plus corroborating media coverage
- Telegram-based alerting for communities/teams

### 1.1 Scheduled Input & Official Warning Fetch
Runs every day at 9:00 and fetches Malaysia weather warnings from the government API.

### 1.2 Warning Parsing → Search Query Construction
Extracts warning type, severity, and affected locations; builds up to 3 deduplicated search queries.

### 1.3 AI Web Search (Perplexity via OpenRouter) → URL Extraction
Uses an AI Agent connected to Perplexity Sonar Pro Search to return only links to relevant, recent (≤3 days) news.

### 1.4 URL Looping + Firecrawl Scraping (Rate-Limited)
Splits URLs and scrapes them one by one, pausing between requests to respect Firecrawl free-tier limits.

### 1.5 Aggregation + Final Report Generation (OpenAI)
Aggregates all scraped summaries and source URLs, then asks OpenAI to produce a categorized Telegram-ready report.

### 1.6 Telegram Delivery
Posts the AI-generated report to a configured Telegram chat/channel.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Input & Official Warning Fetch
**Overview:** Triggers daily and retrieves current official weather warning data from data.gov.my.

**Nodes involved:**
- Schedule Trigger
- Get Weather Warnings

#### Node: Schedule Trigger
- **Type / role:** `scheduleTrigger` — workflow entry point based on time schedule.
- **Configuration:** Runs daily at **triggerAtHour = 9** (server time of the n8n instance).
- **Connections:**  
  - **Output →** Get Weather Warnings
- **Edge cases / failures:**
  - Timezone confusion: “9 AM” is relative to n8n server timezone unless configured otherwise at instance/workflow level.
  - If n8n is down at trigger time, execution may be missed (depends on n8n scheduling behavior/version).

#### Node: Get Weather Warnings
- **Type / role:** `httpRequest` — fetches official warning feed.
- **Configuration choices:**
  - **GET** `https://api.data.gov.my/weather/warning`
  - Default options (no custom headers/auth indicated).
- **Input/Output:**
  - **Input:** Trigger signal from Schedule Trigger
  - **Output:** JSON response items (each warning record becomes an item depending on API response format).
- **Connections:**  
  - **Output →** Separate Search Query, Warning Type and other information
- **Edge cases / failures:**
  - API downtime, throttling, or schema changes.
  - Non-200 responses causing node failure unless “Continue On Fail” is enabled (not shown here).
  - Unexpected response shape could break downstream parsing in the Code node.

---

### Block 2 — Warning Parsing → Search Query Construction
**Overview:** Filters out “No Advisory”, extracts locations from English warning text, derives severity, and emits up to 3 unique search queries.

**Nodes involved:**
- Separate Search Query, Warning Type and other information
- Combine Search and Location information

#### Node: Separate Search Query, Warning Type and other information
- **Type / role:** `code` — transforms and filters the warning data.
- **Key logic (interpreted):**
  - Iterates through all input items (`$input.all()`).
  - Skips items where `heading_en === "No Advisory"`.
  - Uses regex to extract location phrases from `text_en`, looking for patterns like:
    - “states of … until”
    - “over the waters of … until”
    - “waters of … until”
  - Extracts Title-Case tokens as “states/locations”.
  - Derives `severity`:
    - contains “Severe” → `severe`
    - contains “Alert” → `alert`
    - otherwise empty string
  - Builds `searchQuery` like: `"<warningType base> Malaysia news"`
  - Deduplicates queries by `searchQuery`, returns **top 3**.
- **Output fields produced per item:**
  - `searchQuery`, `warningType`, `severity`, `locations` (comma-separated string), `dateRange` (“past 24 hours”)
- **Connections:**  
  - **Output →** Combine Search and Location information
- **Edge cases / failures:**
  - If API uses different wording (regex mismatch), `locations` may end up empty.
  - Title-case extraction may pick up non-location words or miss uppercase acronyms.
  - `locations` is built from a shared Set across items; this means later queries may include locations from earlier warnings (global accumulation), which may or may not be intended.
  - If `text_en` is null/undefined, `.match(...)` will throw (expression runtime error).

#### Node: Combine Search and Location information
- **Type / role:** `aggregate` — combines fields across items.
- **Configuration choices:**
  - Aggregates: `searchQuery` and `locations`
  - Produces a single combined item (typical Aggregate behavior) containing arrays of aggregated values.
- **Connections:**  
  - **Output →** AI Agent
- **Edge cases / failures:**
  - If there are zero items (all warnings were “No Advisory”), downstream AI Agent may run with empty/undefined values.
  - Aggregated structure (arrays) may not match what AI Agent expects (see next block).

---

### Block 3 — AI Web Search (Perplexity via OpenRouter) → URL Extraction
**Overview:** Uses an n8n LangChain AI Agent powered by Perplexity Sonar Pro Search to return only recent news links relevant to the warnings/locations.

**Nodes involved:**
- Perplexity Sonar Pro Model
- AI Agent
- Clean the URLs

#### Node: Perplexity Sonar Pro Model
- **Type / role:** `lmChatOpenRouter` — LLM connector for OpenRouter.
- **Configuration choices:**
  - Model: `perplexity/sonar-pro-search`
  - Temperature: `0.3`
  - Credentials: **OpenRouter API** (must contain an OpenRouter key that can access Perplexity model).
- **Connections:**
  - **AI language model output →** AI Agent (as its `ai_languageModel`)
- **Edge cases / failures:**
  - OpenRouter auth/key missing, quota exceeded, or model not available in account.
  - Model may return non-URL text despite instructions, impacting URL extraction.

#### Node: AI Agent
- **Type / role:** `agent` — prompts the model to search and output only links.
- **Configuration choices:**
  - Prompt includes:
    - `Find latest news about {{ $json.searchQuery }}`
    - Location: `{{ $json.locations }}`
    - Sources: Utusan, Harian Metro, Berita Harian, Kosmo, plus YouTube/Dailymotion
    - Must be within 3 days of `{{ $now }}`
    - “Just give the link of the news without explanation.”
  - Prompt type: `define`
- **Inputs:**
  - Main input from Aggregate node (combined search/locations).
  - LLM input via `ai_languageModel` from Perplexity Sonar Pro Model.
- **Outputs:**
  - Agent output placed in `item.json.output` (as implied by downstream code).
- **Connections:**  
  - **Output →** Clean the URLs
- **Edge cases / failures:**
  - If `searchQuery`/`locations` are arrays (from Aggregate), the rendered prompt may be odd (comma-separated array stringification), which can degrade results.
  - The agent can return duplicates, tracking URLs, or non-HTTP links; downstream cleaning is basic.

#### Node: Clean the URLs
- **Type / role:** `code` — extracts and de-duplicates URLs from AI output.
- **Key logic:**
  - Reads `item.json.output` and matches `https?://...`
  - Strips trailing punctuation `, . )`
  - De-duplicates list
  - Outputs items shaped as `{ json: { url } }`
- **Connections:**  
  - **Output →** Loop URL One by One
- **Edge cases / failures:**
  - If AI Agent output is not in `json.output` (different schema), no URLs will be found.
  - Excludes URLs without http/https (e.g., “www.example.com”).
  - Some legitimate URLs can end with `)` in query strings; stripping may break edge cases.

---

### Block 4 — URL Looping + Firecrawl Scraping (Rate-Limited)
**Overview:** Processes URLs one at a time, scrapes each with Firecrawl for a summary, and waits between calls to avoid free-tier rate limits.

**Nodes involved:**
- Loop URL One by One
- Scrape Website with Firecrawl
- Replace Me
- Interval to avoid Firecrawl Free API Limit

#### Node: Loop URL One by One
- **Type / role:** `splitInBatches` — batching/loop control for sequential processing.
- **Configuration choices:** Default options (batch size not explicitly set; n8n default is typically 1 for this node unless configured).
- **Connections:**
  - **Output 1 →** Combine the Summary and Sources (this is unusual; see edge cases)
  - **Output 1 →** Scrape Website with Firecrawl
- **Behavior notes:**
  - In n8n, Split In Batches usually has a looping pattern where after processing a batch, you connect downstream back to Split In Batches to fetch next batch.
  - Here, the loop-back is implemented via: Firecrawl → Replace Me → Wait → Split In Batches.
- **Edge cases / failures:**
  - Dual outgoing connections from the same output path can cause sequencing/aggregation confusion; “Combine the Summary and Sources” may receive items before they are scraped, depending on execution order and node behavior.
  - If batch size > 1 unintentionally, Firecrawl may be hit in parallel (undesirable for rate limiting).

#### Node: Scrape Website with Firecrawl
- **Type / role:** `httpRequest` — calls Firecrawl scrape endpoint to produce a summary.
- **Configuration choices:**
  - **POST** `https://api.firecrawl.dev/v1/scrape`
  - Timeout: 30s
  - JSON body:
    - `url`: `{{ $json.url }}`
    - `formats`: `["summary"]`
    - `onlyMainContent`: `true`
  - Headers:
    - `Authorization: Bearer fc-YOUR-KEY` (placeholder to replace)
    - `Content-Type: application/json`
- **Output shape (expected):**
  - `data.summary`
  - `data.metadata.sourceURL`
- **Connections:**  
  - **Output →** Replace Me
- **Edge cases / failures:**
  - Invalid/expired Firecrawl key; free-tier quota exceeded.
  - Some sites block scraping, require JS rendering, or return short/empty summaries.
  - Response schema differences (missing `data.summary`) will break later aggregation.

#### Node: Replace Me
- **Type / role:** `noOp` — pass-through placeholder node.
- **Likely intent:** A convenient hook to insert extra processing/logging between Firecrawl and the Wait node.
- **Connections:**  
  - **Output →** Interval to avoid Firecrawl Free API Limit
- **Edge cases:** None functionally (unless removed and connections break).

#### Node: Interval to avoid Firecrawl Free API Limit
- **Type / role:** `wait` — throttling.
- **Configuration:** Waits **amount = 3** (unit not specified in JSON snippet; in n8n Wait node it is typically seconds/minutes depending on UI setting; this workflow shows only `amount`).
- **Connections:**  
  - **Output →** Loop URL One by One (loop-back)
- **Edge cases / failures:**
  - If unit is minutes unintentionally, workflow becomes slow.
  - Wait nodes can hold executions; high frequency usage can increase n8n execution storage/queue load.

---

### Block 5 — Aggregation + Final Report Generation (OpenAI)
**Overview:** Aggregates Firecrawl summaries/URLs and asks OpenAI to write a categorized, Telegram-ready report in English.

**Nodes involved:**
- Combine the Summary and Sources
- Make a summary

#### Node: Combine the Summary and Sources
- **Type / role:** `aggregate` — consolidates scraped outputs across URLs.
- **Configuration choices:**
  - Aggregates:
    - `data.summary`
    - `data.metadata.sourceURL`
- **Expected output fields:** likely arrays; referenced downstream as `$json.summary` and `$json.sourceURL` (so node may be outputting flattened/renamed fields depending on Aggregate behavior/version).
- **Connections:**  
  - **Output →** Make a summary
- **Edge cases / failures:**
  - If Firecrawl returns errors or missing fields, aggregated arrays may contain null/undefined.
  - If earlier wiring sends non-Firecrawl items here (see Split In Batches dual output), aggregation may include unrelated items or fail to resolve `data.summary`.

#### Node: Make a summary
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI chat generation to create final report.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Temperature: `0.3`
  - System prompt instructs:
    - Act as expert weather/flood forecaster for Malaysia
    - Use provided `Summary: {{ $json.summary }}` and `URL {{ $json.sourceURL }}`
    - Categorize each item and produce a Telegram message
    - “Greet and just give the result”
    - Include source URLs
    - English output
- **Output usage:** Telegram node reads `{{ $json.output[0].content[0].text }}`
  - This implies the OpenAI node outputs an `output` array with message content blocks.
- **Connections:**  
  - **Output →** Send to Telegram
- **Edge cases / failures:**
  - If Aggregate output fields don’t match exactly (`summary` vs `data.summary`), prompt will contain empty variables.
  - Model availability depends on OpenAI account and n8n node version compatibility.
  - Token limits: many URLs/summaries can exceed context length; consider truncation or limiting URLs.

---

### Block 6 — Telegram Delivery
**Overview:** Sends the final AI-written report into a Telegram channel/group.

**Nodes involved:**
- Send to Telegram

#### Node: Send to Telegram
- **Type / role:** `telegram` — sends a message.
- **Configuration choices:**
  - Text: `{{ $json.output[0].content[0].text }}`
  - Chat ID: placeholder `INSERT YOUR TELEGRAM GROUP OR CHANNEL ID HERE`
  - Append attribution: false
  - Notifications enabled (`disable_notification: false`)
  - Credentials: Telegram API (bot token)
- **Connections:** End of workflow.
- **Edge cases / failures:**
  - Wrong chat ID format (channel IDs often start with `-100...`).
  - Bot not added to group/channel or lacking admin/post permissions.
  - If OpenAI output schema differs, `output[0].content[0].text` may be undefined and message send fails.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Daily trigger at 9 AM | — | Get Weather Warnings | ## Automated Malaysian Weather Alerts with Perplexity AI and Telegram… (full note applies) |
| Get Weather Warnings | n8n-nodes-base.httpRequest | Fetch official warnings from data.gov.my | Schedule Trigger | Separate Search Query, Warning Type and other information | ## 1. Fetch weather warning from the official government API |
| Separate Search Query, Warning Type and other information | n8n-nodes-base.code | Parse warnings, extract locations, build queries | Get Weather Warnings | Combine Search and Location information | ## 2. Combine the Search Query and Location |
| Combine Search and Location information | n8n-nodes-base.aggregate | Aggregate query/location info before AI search | Separate Search Query, Warning Type and other information | AI Agent | ## 2. Combine the Search Query and Location |
| Perplexity Sonar Pro Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM provider (Perplexity via OpenRouter) | — (credential/model provider) | AI Agent (ai_languageModel) | ## 3.Crawl the internet to search all possible related weather news |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Search for recent links based on warnings/locations | Combine Search and Location information + Perplexity Sonar Pro Model | Clean the URLs | ## 3.Crawl the internet to search all possible related weather news |
| Clean the URLs | n8n-nodes-base.code | Extract and deduplicate URLs from AI output | AI Agent | Loop URL One by One | ## 3.Crawl the internet to search all possible related weather news |
| Loop URL One by One | n8n-nodes-base.splitInBatches | Process URLs sequentially | Clean the URLs + loop-back from Wait | Scrape Website with Firecrawl; Combine the Summary and Sources | ## 4. Process the news and get the summary of the articles |
| Scrape Website with Firecrawl | n8n-nodes-base.httpRequest | Scrape & summarize each URL | Loop URL One by One | Replace Me | ## 4. Process the news and get the summary of the articles |
| Replace Me | n8n-nodes-base.noOp | Placeholder/pass-through after scrape | Scrape Website with Firecrawl | Interval to avoid Firecrawl Free API Limit | ## 4. Process the news and get the summary of the articles |
| Interval to avoid Firecrawl Free API Limit | n8n-nodes-base.wait | Throttle between Firecrawl calls | Replace Me | Loop URL One by One | ## 4. Process the news and get the summary of the articles |
| Combine the Summary and Sources | n8n-nodes-base.aggregate | Aggregate summaries and source URLs | Loop URL One by One | Make a summary | ## 5. Refine the weather report and arrange the URL for the spesific report |
| Make a summary | @n8n/n8n-nodes-langchain.openAi | Generate final categorized English report | Combine the Summary and Sources | Send to Telegram | ## 5. Refine the weather report and arrange the URL for the spesific report |
| Send to Telegram | n8n-nodes-base.telegram | Deliver report to Telegram | Make a summary | — | ## 6.Send the report to the Telegram channel |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## Automated Malaysian Weather Alerts with Perplexity AI and Telegram (contains setup/customize steps and links) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## 1. Fetch weather warning from the official government API |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## 2. Combine the Search Query and Location |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## 3.Crawl the internet to search all possible related weather news |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## 4. Process the news and get the summary of the articles |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## 5. Refine the weather report and arrange the URL for the spesific report |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation/comment | — | — | ## 6.Send the report to the Telegram channel |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name: *Automated Malaysian Weather Alerts with Perplexity AI, Firecrawl and Telegram* (or your preferred name).

2. **Add “Schedule Trigger” (Schedule Trigger node)**
   - Set it to run **daily at 09:00** (confirm your instance timezone).
   - Connect **Schedule Trigger → Get Weather Warnings**.

3. **Add “Get Weather Warnings” (HTTP Request node)**
   - Method: **GET**
   - URL: `https://api.data.gov.my/weather/warning`
   - Leave auth empty.
   - Connect **Get Weather Warnings → Separate Search Query, Warning Type and other information**.

4. **Add “Separate Search Query, Warning Type and other information” (Code node)**
   - Paste the JS logic that:
     - filters `heading_en === "No Advisory"`
     - extracts locations from `text_en`
     - builds `searchQuery`, `warningType`, `severity`, `locations`, etc.
     - deduplicates and returns up to 3 queries
   - Connect **Code → Combine Search and Location information**.

5. **Add “Combine Search and Location information” (Aggregate node)**
   - Configure fields to aggregate:
     - `searchQuery`
     - `locations`
   - Connect **Aggregate → AI Agent**.

6. **Add “Perplexity Sonar Pro Model” (OpenRouter Chat Model node)**
   - Node type: **LangChain OpenRouter Chat Model** (`lmChatOpenRouter`)
   - Model: `perplexity/sonar-pro-search`
   - Temperature: `0.3`
   - **Credentials:** create **OpenRouter API** credential with your OpenRouter key (must support the chosen model).

7. **Add “AI Agent” (LangChain Agent node)**
   - Prompt (text) should request:
     - latest news for `{{ $json.searchQuery }}` with location `{{ $json.locations }}`
     - sources from Malaysian news outlets
     - only links, no explanation
     - published within last 3 days (`{{ $now }}`)
   - Connect the model:
     - **Perplexity Sonar Pro Model → AI Agent** via the **ai_languageModel** connection.
   - Connect **AI Agent → Clean the URLs**.

8. **Add “Clean the URLs” (Code node)**
   - Implement URL extraction from `item.json.output`, de-duplicate, and output `{ url }` items.
   - Connect **Clean the URLs → Loop URL One by One**.

9. **Add “Loop URL One by One” (Split In Batches node)**
   - Set batch size to **1** (recommended) to truly process one-by-one.
   - Connect **Split In Batches → Scrape Website with Firecrawl**.

10. **Add “Scrape Website with Firecrawl” (HTTP Request node)**
   - Method: **POST**
   - URL: `https://api.firecrawl.dev/v1/scrape`
   - Headers:
     - `Authorization: Bearer fc-<YOUR_FIRECRAWL_KEY>`
     - `Content-Type: application/json`
   - Body (JSON):
     - `url`: `{{ $json.url }}`
     - `formats`: `["summary"]`
     - `onlyMainContent`: `true`
   - Timeout: ~30 seconds
   - Connect **Firecrawl → Replace Me**.

11. **Add “Replace Me” (NoOp node)**
   - No config needed.
   - Connect **Replace Me → Interval to avoid Firecrawl Free API Limit**.

12. **Add “Interval to avoid Firecrawl Free API Limit” (Wait node)**
   - Set wait duration to **3 seconds** (or appropriate for your Firecrawl plan).
   - Connect **Wait → Loop URL One by One** (this creates the loop for next batch).

13. **Add “Combine the Summary and Sources” (Aggregate node)**
   - Aggregate fields:
     - `data.summary`
     - `data.metadata.sourceURL`
   - Important: connect this node so it receives the **scrape results** (recommended wiring is from **Scrape Website with Firecrawl → Combine the Summary and Sources**, not from Split In Batches directly).
   - Then connect **Combine the Summary and Sources → Make a summary**.

14. **Add “Make a summary” (OpenAI / LangChain OpenAI node)**
   - Model: `gpt-4.1-mini`
   - Temperature: `0.3`
   - System message should:
     - treat input as Malaysia weather news summaries + URLs
     - categorize items
     - greet and output Telegram-ready English report with sources
   - **Credentials:** create an **OpenAI API** credential with your OpenAI API key.
   - Connect **Make a summary → Send to Telegram**.

15. **Add “Send to Telegram” (Telegram node)**
   - Set **Chat ID** to your target group/channel (often `-100xxxxxxxxxx` for channels).
   - Message text expression: `{{ $json.output[0].content[0].text }}`
   - **Credentials:** Telegram bot credential (create bot via **@BotFather** and paste token into n8n credential).
   - Ensure bot has posting permissions in the target chat.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Get an OpenAI API key | https://platform.openai.com |
| Get a Perplexity API key via OpenRouter | https://openrouter.ai |
| Get a free Firecrawl API key (free tier limited requests/hour) | https://firecrawl.dev |
| Create a Telegram bot and get token from BotFather | https://t.me/botfather |
| Customization ideas: change schedule time, language, filter severity/states, expand sources, add storage (Supabase/Sheets), multi-channel delivery (email/WhatsApp/SMS) | From the workflow sticky note content |

