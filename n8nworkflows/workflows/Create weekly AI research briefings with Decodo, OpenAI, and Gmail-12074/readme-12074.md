Create weekly AI research briefings with Decodo, OpenAI, and Gmail

https://n8nworkflows.xyz/workflows/create-weekly-ai-research-briefings-with-decodo--openai--and-gmail-12074


# Create weekly AI research briefings with Decodo, OpenAI, and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Create weekly AI research briefings with Decodo, OpenAI, and Gmail  
**Workflow name (in JSON):** Automate Research with Decodo, OpenAI, and Gmail  
**Purpose:** On a weekly schedule, the workflow searches for recent AI/LLM news via Decodo (Google Search), scrapes each resulting page via Decodo, cleans the text, summarizes each page using an OpenAI chat model through a LangChain summarization chain, then uses an AI Agent to rank and compile a single intelligence report which is emailed via Gmail. If nothing is found, it emails an “empty” update. If scraping yields poor/empty content, it alerts via Telegram.

### Logical Blocks
**1.1 Scheduling & Search Configuration**  
Weekly trigger → define search parameters.

**1.2 Search (Decodo) + URL Filtering/Parsing**  
Run Decodo Google search → normalize/parse results → filter out non-articles (YouTube).

**1.3 Batch Loop + Scrape + Quality Gate**  
Iterate over URLs → scrape HTML via Decodo → validate content length → on failure send Telegram alert.

**1.4 Sanitize + Summarize per URL**  
Strip scripts/styles/tags → summarize cleaned text with OpenAI (LangChain summarization chain).

**1.5 Aggregate + Analyze + Email**  
Aggregate all summaries → if none, send empty email → otherwise run agent to rank and produce report → email report.

---

## 2. Block-by-Block Analysis

### 1.1 Scheduling & Search Configuration

**Overview:** Starts the workflow weekly and sets a configurable JSON “search config” used downstream (query, thresholds, etc.).

**Nodes Involved:**  
- Weekly Trigger  
- Set Search Config

#### Node: Weekly Trigger
- **Type / role:** `Schedule Trigger` — entry point; runs automatically.
- **Configuration (interpreted):** Runs every **7 days** at **06:00** (instance timezone).
- **Outputs:** To **Set Search Config**.
- **Potential failures / edge cases:**
  - Timezone mismatch (n8n instance settings vs expectation).
  - Workflow is **inactive** (`active: false`), so it won’t run until activated.

#### Node: Set Search Config
- **Type / role:** `Set` — creates a structured JSON object used by search.
- **Configuration (interpreted):** Raw JSON output:
  - `search_query`: `"latest AI LLM news"`
  - `max_results`: `10` *(note: not actually enforced downstream in this JSON)*
  - `min_relevance_score`: `50` *(agent prompt uses 50% cutoff)*
  - `summary_language`: `"en"`
  - `delivery_channel`: `"email"`
  - `debug_mode`: `false`
- **Key variables:** Values stored in `$json.*` for downstream expressions.
- **Outputs:** To **Search API (Decodo)**.
- **Edge cases:**
  - `max_results`, `min_relevance_score`, etc. are not referenced by all downstream nodes; changing them may have no effect unless you also update corresponding nodes.

---

### 1.2 Search (Decodo) + URL Filtering/Parsing

**Overview:** Performs a Google search through Decodo and transforms the results into a clean list of article-like URLs.

**Nodes Involved:**  
- Search API (Decodo)  
- Filter & Parse URLs

#### Node: Search API (Decodo)
- **Type / role:** `@decodo/n8n-nodes-decodo.decodo` — calls Decodo’s Google Search operation.
- **Configuration (interpreted):**
  - **Operation:** `google_search`
  - **Query:** from config: `={{ $json.search_query }}`
  - **Results limit:** `1` *(important: this may limit the returned result pages/blocks depending on Decodo API semantics; it does not match `max_results: 10` in config)*
- **Credentials:** Decodo API key (configured in n8n Credentials).
- **Outputs:** To **Filter & Parse URLs**.
- **Potential failures:**
  - Auth/credential errors (401/403).
  - Quota/rate limits.
  - Response schema changes (breaks parsing in next node).

#### Node: Filter & Parse URLs
- **Type / role:** `Code` — normalizes Decodo response shape, filters and maps to a list of items.
- **Configuration (interpreted):**
  - Reads organic results from potentially different shapes:
    - `const root = $json.results?.[0]?.content?.results;`
    - `const organic = root?.results?.organic || root?.organic || [];`
  - Returns **one item per URL** with fields:
    - `url`, `title`, `desc`, `pos`, `url_shown`, `favicon_text`, plus passthrough `region`, `platform`
  - Filters out YouTube:
    - excludes if URL contains `youtube.com` or `youtu.be`
- **Inputs:** From **Search API (Decodo)**.
- **Outputs:** To **Batch Loop**.
- **Edge cases / failures:**
  - If Decodo returns no `organic`, node returns `[]` → downstream receives nothing (ultimately may lead to empty email).
  - Items with missing `url` become `""` and may cause scrape failures later.
  - If Decodo response structure changes, parsing may yield empty set.

---

### 1.3 Batch Loop + Scrape + Quality Gate

**Overview:** Iterates through each URL, scrapes HTML content, and checks that the content looks usable. If not, sends a Telegram warning.

**Nodes Involved:**  
- Batch Loop  
- Scrape Content (Decodo)  
- Check Content Quality  
- Telegram Error Alert

#### Node: Batch Loop
- **Type / role:** `Split In Batches` — controls iteration over multiple URL items.
- **Configuration (interpreted):** Uses default options (batch size default in node; not explicitly set here).
- **Connections (important behavior):**
  - Has two outgoing connections:
    1. **Output 0 → Scrape Content (Decodo)** (main scraping loop)
    2. **Output 1 → Summarize Chain** (the “no items left / done” path in SplitInBatches semantics)
- **Inputs:** From **Filter & Parse URLs** and later from **Sanitize HTML** (to continue loop).
- **Outputs:** To **Scrape Content (Decodo)** and to **Summarize Chain**.
- **Edge cases:**
  - If there are **0 URLs**, the node may immediately take the “done” output (depends on SplitInBatches behavior), potentially triggering summarization with no items.
  - Batch size too large can increase time/memory usage.

#### Node: Scrape Content (Decodo)
- **Type / role:** `@decodo/n8n-nodes-decodo.decodo` — fetches page content from a URL.
- **Configuration (interpreted):**
  - URL: `={{$json.url}}`
  - Operation not explicitly shown in parameters; node likely defaults to a scraping/fetch operation for Decodo when `url` is provided (per Decodo node design).
- **Inputs:** From **Batch Loop**.
- **Outputs:** To **Check Content Quality**.
- **Potential failures:**
  - Invalid/empty URL.
  - Target site blocks scraping (403/429), captchas, bot protection.
  - Timeouts/large pages.
  - Decodo response shape differences.

#### Node: Check Content Quality
- **Type / role:** `IF` — validates scraped content exists and is sufficiently long.
- **Configuration (interpreted):** “True” branch if:
  1. `{{$json.results?.[0]?.content}}` is **not empty**
  2. `{{$json.results?.[0]?.content.length}}` is **> 300**
- **True output:** To **Sanitize HTML**  
- **False output:** To **Telegram Error Alert**
- **Edge cases:**
  - If `results[0].content` is not a string (e.g., object/array), `.length` may be unexpected.
  - Some pages are short but still meaningful; they will be rejected (<300 chars).

#### Node: Telegram Error Alert
- **Type / role:** `Telegram` — sends a warning when scraping quality check fails.
- **Configuration (interpreted):**
  - Chat ID: `"YOUR_CHAT_ID"` (must be replaced)
  - Text: `"Warning ... Error in scraping "`
- **Inputs:** From **Check Content Quality** (false path).
- **Outputs:** None downstream.
- **Potential failures:**
  - Missing/invalid Telegram credentials/bot token.
  - Wrong chat ID (message not delivered).

---

### 1.4 Sanitize + Summarize per URL

**Overview:** Converts HTML into cleaned text, then uses an LLM summarization chain to produce a short technical summary for each page.

**Nodes Involved:**  
- Sanitize HTML  
- OpenAI Model (Summary)  
- Summarize Chain

#### Node: Sanitize HTML
- **Type / role:** `Code` (run once per item) — strips HTML/JS/CSS and normalizes whitespace.
- **Configuration (interpreted):**
  - Reads: `item.results[0].content` into `html`
  - Removes:
    - `<script>…</script>`, `<style>…</style>`, `<noscript>…</noscript>`
    - all tags via `/<\/?[^>]+>/g`
  - Decodes a few HTML entities (`&nbsp;`, `&amp;`, etc.)
  - Collapses whitespace and trims.
  - Outputs: `{ text_clean: cleaned }`
- **Inputs:** From **Check Content Quality** (true path).
- **Outputs:** To **Batch Loop** (this is critical to how the loop is wired).
- **Edge cases:**
  - If Decodo returns content in a different location, `html` becomes empty and summarization will be poor.
  - Removing all tags may remove meaningful formatting (tables/code). Summaries may miss details.

#### Node: OpenAI Model (Summary)
- **Type / role:** `LangChain Chat Model` — provides the LLM for the summarization chain.
- **Configuration (interpreted):**
  - Model: `gpt-4.1-mini`
  - No special options/tools enabled.
- **Connections:** Linked to **Summarize Chain** via `ai_languageModel`.
- **Potential failures:**
  - OpenAI credential missing/invalid.
  - Model not available in the account/region.
  - Rate limits / timeouts with many URLs.

#### Node: Summarize Chain
- **Type / role:** `@n8n/n8n-nodes-langchain.chainSummarization` — generates one summary per input item.
- **Configuration (interpreted):**
  - Method: **“stuff”** (single-pass summarization prompt over the provided text).
  - Prompt uses: `{{ $json.text_clean }}`
  - Output requested: short human-readable text (not JSON), 6–8 lines, no fluff, extracts topic/entity/type/key points/why it matters when present.
- **Inputs (as connected):** From **Batch Loop (output 1)**. In this workflow wiring, summarization appears to be triggered when the batch loop finishes.
- **Outputs:** To **Aggregate Summaries**.
- **Edge cases / important note on wiring:**
  - The current connections suggest the chain runs on the “done” output of SplitInBatches, which can be valid if the intent is to accumulate items then summarize; however, **Sanitize HTML currently feeds back into Batch Loop**, not into Summarize Chain directly. If your intention is “scrape → sanitize → summarize each URL”, verify the loop wiring in n8n UI to ensure summaries are actually created per URL.
  - With “stuff” method, very long `text_clean` may exceed token limits; consider chunking/map-reduce if needed.

---

### 1.5 Aggregate + Analyze + Email

**Overview:** Collects all summaries into a single payload, checks if any exist, then either emails “no updates” or generates a ranked report via an AI agent and emails it.

**Nodes Involved:**  
- Aggregate Summaries  
- Check Results Exist  
- OpenAI Model (Agent)  
- Research Analyst Agent  
- Send Email (Report)  
- Send Email (Empty)

#### Node: Aggregate Summaries
- **Type / role:** `Code` — aggregates multiple items into one JSON document.
- **Configuration (interpreted):**
  - `const items = $input.all();`
  - `const summaries = items.map(item => item.json);`
  - Outputs single item: `{ summaries: [...] }`
- **Inputs:** From **Summarize Chain**.
- **Outputs:** To **Check Results Exist**.
- **Edge cases:**
  - If Summarize Chain outputs unexpected structure, you may aggregate wrong fields.
  - If there are zero items, `summaries` becomes `[]`.

#### Node: Check Results Exist
- **Type / role:** `IF` — determines whether to proceed with report generation.
- **Condition:** `{{ $json.summaries.length }} > 0`
- **True output:** To **Research Analyst Agent**  
- **False output:** To **Send Email (Empty)**
- **Edge cases:**
  - If `$json.summaries` is missing/null, expression may error; current flow assumes it always exists from Aggregate node.

#### Node: OpenAI Model (Agent)
- **Type / role:** `LangChain Chat Model` — LLM backing the agent.
- **Configuration:** Model `gpt-4.1-mini`.
- **Connections:** Provides `ai_languageModel` to **Research Analyst Agent**.
- **Failures:** Same as other OpenAI node (auth, rate limits, model access).

#### Node: Research Analyst Agent
- **Type / role:** `LangChain Agent` — transforms a set of summaries into a consolidated ranked report.
- **Configuration (interpreted):**
  - Prompt type: “define”
  - Instructions:
    - Score each summary 0–100%
    - Exclude below 50%
    - Provide overview, ranked updates, and patterns/signals
    - Plain text only; no markdown; do not invent facts
- **Input expectation:** Receives the aggregated `summaries` item from **Check Results Exist** (true path).
- **Output:** Produces `output` text used by email node: `{{$json.output}}`
- **Potential failures / edge cases:**
  - If the input summaries are not actually included in the agent context (depending on agent node input mapping defaults), output may be generic. Ensure the agent is configured to incorporate the incoming JSON (n8n agent nodes typically include input by default, but verify).
  - Long summary lists may hit context/token limits.

#### Node: Send Email (Report)
- **Type / role:** `Gmail` — sends the final intelligence report.
- **Configuration (interpreted):**
  - To: `user@example.com` (must be replaced)
  - Subject: `Weekly AI / LLM Intelligence Report – <local date>`
  - Message body: `={{ $json.output }}`
  - Email type: `text`
  - `appendAttribution: false`
  - Auth: `serviceAccount` (requires Google service account credential setup in n8n)
- **Inputs:** From **Research Analyst Agent**.
- **Failures:**
  - Service account misconfigured (domain-wide delegation issues, missing scopes).
  - Recipient blocked, rate limits, Gmail API errors.

#### Node: Send Email (Empty)
- **Type / role:** `Gmail` — sends a fallback email if no summaries exist.
- **Configuration:**
  - To: `user@example.com`
  - Subject: same date-based subject
  - Message: `No Updates Found...`
  - Email type: `text`
  - Auth: `serviceAccount`
- **Inputs:** From **Check Results Exist** (false path).
- **Failures:** Same as report email.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Overview | Sticky Note | Documentation / operator notes | — | — | ### How it works; Setup steps; Customization tips (as written in node) |
| Section 1 | Sticky Note | Section label | — | — | ## 1. Search & Filter Configure the search parameters, fetch results from Google via Decodo, and filter out non-article URLs (e.g., YouTube). |
| Section 2 | Sticky Note | Section label | — | — | ## 2. Scrape & Sanitize Loop through each URL, download the raw HTML, and clean it to remove scripts/styles before passing to the LLM. |
| Section 3 | Sticky Note | Section label | — | — | ## 3. Summarize Content Generate concise, dense summaries for each individual article using an LLM Chain. |
| Section 4 | Sticky Note | Section label | — | — | ## 4. Analyze & Report An AI Agent reviews all summaries, ranks them by relevance, and sends the final email report. |
| Weekly Trigger | Schedule Trigger | Weekly entry point | — | Set Search Config | ## 1. Search & Filter Configure the search parameters, fetch results from Google via Decodo, and filter out non-article URLs (e.g., YouTube). |
| Set Search Config | Set | Defines search parameters | Weekly Trigger | Search API (Decodo) | ## 1. Search & Filter Configure the search parameters, fetch results from Google via Decodo, and filter out non-article URLs (e.g., YouTube). |
| Search API (Decodo) | Decodo | Google search via Decodo | Set Search Config | Filter & Parse URLs | ## 1. Search & Filter Configure the search parameters, fetch results from Google via Decodo, and filter out non-article URLs (e.g., YouTube). |
| Filter & Parse URLs | Code | Normalize results, filter URLs | Search API (Decodo) | Batch Loop | ## 1. Search & Filter Configure the search parameters, fetch results from Google via Decodo, and filter out non-article URLs (e.g., YouTube). |
| Batch Loop | Split In Batches | Iteration control over URLs | Filter & Parse URLs; Sanitize HTML | Scrape Content (Decodo); Summarize Chain | ## 2. Scrape & Sanitize Loop through each URL, download the raw HTML, and clean it to remove scripts/styles before passing to the LLM. |
| Scrape Content (Decodo) | Decodo | Fetch raw page HTML/content | Batch Loop | Check Content Quality | ## 2. Scrape & Sanitize Loop through each URL, download the raw HTML, and clean it to remove scripts/styles before passing to the LLM. |
| Check Content Quality | IF | Gate: content exists and >300 chars | Scrape Content (Decodo) | Sanitize HTML (true); Telegram Error Alert (false) | ## 2. Scrape & Sanitize Loop through each URL, download the raw HTML, and clean it to remove scripts/styles before passing to the LLM. |
| Telegram Error Alert | Telegram | Alert on scrape failure/low content | Check Content Quality (false) | — | ## 2. Scrape & Sanitize Loop through each URL, download the raw HTML, and clean it to remove scripts/styles before passing to the LLM. |
| Sanitize HTML | Code | Strip scripts/styles/tags; produce clean text | Check Content Quality (true) | Batch Loop | ## 2. Scrape & Sanitize Loop through each URL, download the raw HTML, and clean it to remove scripts/styles before passing to the LLM. |
| OpenAI Model (Summary) | LangChain ChatOpenAI | LLM for summarization | — | Summarize Chain (ai_languageModel) | ## 3. Summarize Content Generate concise, dense summaries for each individual article using an LLM Chain. |
| Summarize Chain | LangChain Summarization Chain | Summarize cleaned content | Batch Loop | Aggregate Summaries | ## 3. Summarize Content Generate concise, dense summaries for each individual article using an LLM Chain. |
| Aggregate Summaries | Code | Combine per-URL summaries into one array | Summarize Chain | Check Results Exist | ## 4. Analyze & Report An AI Agent reviews all summaries, ranks them by relevance, and sends the final email report. |
| Check Results Exist | IF | Branch: summaries exist vs empty | Aggregate Summaries | Research Analyst Agent (true); Send Email (Empty) (false) | ## 4. Analyze & Report An AI Agent reviews all summaries, ranks them by relevance, and sends the final email report. |
| OpenAI Model (Agent) | LangChain ChatOpenAI | LLM for agent report | — | Research Analyst Agent (ai_languageModel) | ## 4. Analyze & Report An AI Agent reviews all summaries, ranks them by relevance, and sends the final email report. |
| Research Analyst Agent | LangChain Agent | Rank/filter summaries, produce report | Check Results Exist (true) | Send Email (Report) | ## 4. Analyze & Report An AI Agent reviews all summaries, ranks them by relevance, and sends the final email report. |
| Send Email (Report) | Gmail | Deliver final report | Research Analyst Agent | — | ## 4. Analyze & Report An AI Agent reviews all summaries, ranks them by relevance, and sends the final email report. |
| Send Email (Empty) | Gmail | Notify no updates found | Check Results Exist (false) | — | ## 4. Analyze & Report An AI Agent reviews all summaries, ranks them by relevance, and sends the final email report. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: “Automate Research with Decodo, OpenAI, and Gmail” (or your preferred name).

2) **Add Trigger**
- Add node: **Schedule Trigger**
- Set: every **7 days** at **06:00** (or adjust as desired).

3) **Add config node**
- Add node: **Set** → set **Mode: Raw (JSON)**
- Paste/create JSON keys:
  - `search_query` (string)
  - `max_results` (number; optional unless you wire it later)
  - `min_relevance_score` (number; used conceptually by the agent prompt)
  - `summary_language`, `delivery_channel`, `debug_mode`
- Connect: **Schedule Trigger → Set Search Config**

4) **Add Decodo search**
- Add node: **Decodo** (package `@decodo/n8n-nodes-decodo`)
- Operation: **google_search**
- Query expression: `{{ $json.search_query }}`
- Results limit: `1` (or adjust to match your needs)
- Configure **Decodo credentials** (API key) in n8n.
- Connect: **Set Search Config → Search API (Decodo)**

5) **Parse/filter URLs**
- Add node: **Code**
- Paste logic to:
  - locate organic results in Decodo output
  - filter out YouTube URLs
  - map to items with `url`, `title`, etc.
- Connect: **Search API (Decodo) → Filter & Parse URLs**

6) **Add batch loop**
- Add node: **Split In Batches**
- Keep defaults unless you want to set a batch size.
- Connect: **Filter & Parse URLs → Batch Loop**

7) **Scrape each URL**
- Add node: **Decodo**
- Configure to fetch by URL (set `url` field to `{{ $json.url }}`).
- Connect: **Batch Loop (output 0) → Scrape Content (Decodo)**

8) **Quality gate**
- Add node: **IF**
- Conditions:
  - String “not empty”: `{{ $json.results?.[0]?.content }}`
  - Number “greater than”: `{{ $json.results?.[0]?.content.length }}` > `300`
- Connect: **Scrape Content (Decodo) → Check Content Quality**

9) **Telegram alert on bad scrape**
- Add node: **Telegram**
- Set Chat ID to your target chat.
- Set message text (e.g., “Warning ... Error in scraping”).
- Configure **Telegram credentials** (bot token).
- Connect: **Check Content Quality (false) → Telegram Error Alert**

10) **Sanitize HTML**
- Add node: **Code** with “Run once for each item”
- Implement HTML cleanup and output `{ text_clean }`.
- Connect: **Check Content Quality (true) → Sanitize HTML**

11) **Loop continuation wiring**
- Connect: **Sanitize HTML → Batch Loop** (to continue iterating).
- Also connect: **Batch Loop (output 1 / done) → Summarize Chain** (to proceed after loop completes).

12) **Summarize chain**
- Add node: **Summarization Chain** (`@n8n/n8n-nodes-langchain.chainSummarization`)
- Set method to **stuff**
- Provide prompt referencing: `{{ $json.text_clean }}`
- Add node: **OpenAI Chat Model** (`lmChatOpenAi`)
  - Model: `gpt-4.1-mini`
  - Configure **OpenAI credentials** (API key).
- Connect OpenAI model to chain via **ai_languageModel** connection.

13) **Aggregate summaries**
- Add node: **Code** to collect `$input.all()` and output `{ summaries: [...] }` as a single item.
- Connect: **Summarize Chain → Aggregate Summaries**

14) **Check summaries exist**
- Add node: **IF**
- Condition: `{{ $json.summaries.length }}` > `0`
- Connect: **Aggregate Summaries → Check Results Exist**

15) **Empty email path**
- Add node: **Gmail** (send)
- Auth: **Service Account** (as in the workflow) or OAuth2 if preferred.
- To: replace `user@example.com`
- Subject: `Weekly AI / LLM Intelligence Report – {{ new Date().toLocaleDateString() }}`
- Message: `No Updates Found...`
- Connect: **Check Results Exist (false) → Send Email (Empty)**

16) **Agent report path**
- Add node: **AI Agent** (`@n8n/n8n-nodes-langchain.agent`)
- Paste the analyst instructions (scoring, filtering <50%, structured plain text report).
- Add node: **OpenAI Chat Model** for the agent (`gpt-4.1-mini`) and connect via **ai_languageModel**.
- Connect: **Check Results Exist (true) → Research Analyst Agent**

17) **Send report email**
- Add node: **Gmail** (send)
- To: replace `user@example.com`
- Subject: same as above
- Message: `{{ $json.output }}`
- Email type: text
- Connect: **Research Analyst Agent → Send Email (Report)**

18) **Activate workflow**
- Toggle workflow to **Active**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Credentials: Add your Decodo and OpenAI API keys in the workflow credentials.” | From “Main Overview” sticky note |
| “Schedule: Adjust the Weekly Trigger node to your preferred day and time.” | From “Main Overview” sticky note |
| “Search Config: Update the search_query in the Set Search Config node if you want to track a different topic.” | From “Main Overview” sticky note |
| “Email: Change the recipient email address in both Send Email nodes.” | From “Main Overview” sticky note |
| “Customization tips: Adjust the ‘Relevance Score’ prompt in the Research Analyst Agent node to make the filtering stricter or looser.” | From “Main Overview” sticky note |