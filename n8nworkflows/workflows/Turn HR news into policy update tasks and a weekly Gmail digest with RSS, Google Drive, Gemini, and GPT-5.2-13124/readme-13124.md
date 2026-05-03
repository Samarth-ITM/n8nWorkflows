Turn HR news into policy update tasks and a weekly Gmail digest with RSS, Google Drive, Gemini, and GPT-5.2

https://n8nworkflows.xyz/workflows/turn-hr-news-into-policy-update-tasks-and-a-weekly-gmail-digest-with-rss--google-drive--gemini--and-gpt-5-2-13124


# Turn HR news into policy update tasks and a weekly Gmail digest with RSS, Google Drive, Gemini, and GPT-5.2

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow runs weekly, reads HR-related news from an RSS feed, extracts and cleans article text, compares each article against existing HR policy/contract template filenames stored in a Google Drive folder, and generates a concise HTML “HR Weekly Radar” email sent via Gmail.

**Typical use cases:**
- HR/People Ops teams monitoring regulatory and practice changes (Germany-focused in prompting) and translating them into document review/update actions.
- Maintaining a weekly internal digest that stays scannable and low-noise while still pointing to concrete policy/template maintenance tasks.

### 1.1 Logical Blocks (by dependency)
1. **Scheduling & Configuration**: Weekly trigger + user-provided configuration values.
2. **RSS Intake & Filtering**: Fetch RSS items, keep last 7 days, cap total items for cost/runtime.
3. **Article Fetch + Text Extraction/Cleaning**: Download article HTML, extract main body via CSS selector, clean and truncate text, generate stable article IDs.
4. **Drive Inventory (Templates/Policies)**: List files in a configured Drive folder and aggregate them into a single inventory payload.
5. **Per-Article AI Impact Analysis (Gemini + Structured JSON)**: Agent compares each article to the Drive file list and emits structured JSON findings.
6. **Weekly Aggregation + Email Authoring (OpenAI)**: Aggregate results across articles, then generate calm HTML newsletter.
7. **Send Email (Gmail)**: Deliver the generated HTML to the configured recipient.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Configuration
**Overview:** Starts the workflow weekly and centralizes user-editable settings (RSS URL, recipient email, Drive folder ID, max articles).  
**Nodes involved:** `Weekly trigger`, `Config`

#### Node: Weekly trigger
- **Type / role:** Schedule Trigger; workflow entry point.
- **Configuration choices:**
  - Runs **weekly** (field: weeks), **day 1**, at **09:00** (n8n schedule semantics depend on instance timezone).
- **Inputs / outputs:**
  - **Input:** None (trigger).
  - **Output:** One item to `Config`.
- **Version notes:** `typeVersion 1.3` (standard schedule node behavior).
- **Edge cases / failures:**
  - Timezone mismatch (instance timezone vs. expectation).
  - Workflow inactive (`active: false`), so it won’t run until enabled.

#### Node: Config
- **Type / role:** Set node; defines runtime configuration variables.
- **Configuration choices (interpreted):**
  - Sets:
    - `newsUrl` (RSS feed URL placeholder)
    - `userEmail` (recipient placeholder)
    - `templatesFolderId` (Google Drive folder ID placeholder)
    - `maxArticles` (default 10)
  - `includeOtherFields: true` keeps incoming trigger fields (if any).
- **Key expressions / variables:**
  - Values are literal placeholders; downstream nodes reference them via `$node["Config"].json.*`.
- **Inputs / outputs:**
  - **Input:** `Weekly trigger`
  - **Output:** `Read RSS feed`
- **Edge cases / failures:**
  - If placeholders are not replaced, downstream nodes will fail (RSS read, Drive list, Gmail send).

---

### Block 2 — RSS Intake & Filtering
**Overview:** Reads the RSS feed, filters to only recent items (last 7 days), and caps the number of articles processed to control cost/runtime.  
**Nodes involved:** `Read RSS feed`, `Keep last 7 days`, `Cap articles`

#### Node: Read RSS feed
- **Type / role:** RSS Feed Read; pulls items from RSS/Atom.
- **Configuration choices:**
  - URL is taken from config: `{{$node["Config"].json.newsUrl}}`
  - `executeOnce: true` (reads feed once per execution, not per item).
- **Inputs / outputs:**
  - **Input:** `Config`
  - **Output:** `Keep last 7 days`
- **Edge cases / failures:**
  - Invalid URL / unreachable host / TLS issues.
  - Feed parsing errors or non-standard RSS fields.
  - Missing `date` field in items (affects next filter).

#### Node: Keep last 7 days
- **Type / role:** Filter; keeps only items newer than 7 days.
- **Configuration choices:**
  - Condition: `$json.date` **after** `$now.minus({ days: 7 }).toISO()`
  - Strict type validation enabled.
- **Inputs / outputs:**
  - **Input:** `Read RSS feed`
  - **Output:** `Cap articles`
- **Edge cases / failures:**
  - If `$json.date` is missing or not parseable as datetime, items may be dropped or the node may error depending on strictness and incoming types.
  - Publisher timezone/format inconsistencies.

#### Node: Cap articles
- **Type / role:** Code; limits number of items to process.
- **Configuration choices:**
  - Reads `maxArticles` from `Config` (defaults to 10).
  - Returns `items.slice(0, maxArticles)` with safety clamp.
- **Key expressions / variables:**
  - `Number($node["Config"].json.maxArticles ?? 10)`
- **Inputs / outputs:**
  - **Input:** `Keep last 7 days`
  - **Outputs:** **two parallel outputs**
    - To `Fetch article page` (per item)
    - To `Combine article data` (used as “RSS-side” data stream for merging)
- **Edge cases / failures:**
  - Non-numeric `maxArticles` leads to `NaN` → `Math.max(0, NaN)` becomes `NaN` and `slice(0, NaN)` returns empty array (effectively no processing).
  - If no items remain after filtering, downstream blocks produce empty aggregation; email writer may still run depending on n8n execution behavior and merges.

---

### Block 3 — Article Fetch + Text Extraction/Cleaning
**Overview:** Fetches each article URL, extracts the main body HTML using a CSS selector, removes noise, converts to plain text, truncates, and generates a stable article ID and excerpt.  
**Nodes involved:** `Fetch article page`, `Extract article body`, `Clean article text`, `Combine article data`, `Normalize article fields`, `Add id and excerpt`

#### Node: Fetch article page
- **Type / role:** HTTP Request; downloads full article HTML.
- **Configuration choices:**
  - URL: `{{$json.link}}` (from RSS item)
- **Inputs / outputs:**
  - **Input:** `Cap articles`
  - **Output:** `Extract article body`
- **Edge cases / failures:**
  - 403/paywall, bot protection, cookies required, redirects.
  - Large pages/timeouts (no explicit timeout configured here).
  - Some sites require headers (User-Agent); node currently uses defaults.

#### Node: Extract article body
- **Type / role:** HTML node; extracts a specific HTML fragment via CSS selector.
- **Configuration choices:**
  - Operation: extract HTML content
  - Extracts `body_html` using CSS selector: `.article-detail__body`
  - Returns HTML (not text).
- **Inputs / outputs:**
  - **Input:** `Fetch article page`
  - **Output:** `Clean article text`
- **Edge cases / failures:**
  - Selector mismatch yields empty `body_html`.
  - Publisher HTML structure changes frequently; this is a common break point.

#### Node: Clean article text
- **Type / role:** Code (run once per item); cleans HTML into plain text and trims length.
- **Configuration choices (interpreted):**
  - Removes:
    - Social bar: `<div class="horizontal-social-bar">…</div>`
    - Ads blocks with IDs like `Ads_...`
    - `<script>` and `<style>` blocks
    - Content from a German “related content” marker onward: “Das könnte Sie auch interessieren”
    - Tag list `.taglist--v2`
  - Converts headings/paragraphs/line breaks into newlines, strips remaining tags, decodes some entities.
  - Outputs:
    - `url: $json.url` (note: may not exist depending on upstream fields)
    - `body_text: cleaned text`
- **Inputs / outputs:**
  - **Input:** `Extract article body`
  - **Output:** `Combine article data` (into merge input index 1)
- **Edge cases / failures:**
  - If `body_html` is empty, output text becomes empty string.
  - `$json.url` might be undefined; downstream relies primarily on `link` from RSS, so this is usually non-fatal.
  - Regex cleaning assumes specific site patterns; for other publishers it may over-delete or under-delete.

#### Node: Combine article data
- **Type / role:** Merge; combines RSS item data with cleaned text.
- **Configuration choices:**
  - Mode: **combine**
  - Combine by: **position**
  - Expects two inputs:
    - Input 0: from `Cap articles` (RSS items)
    - Input 1: from `Clean article text` (cleaned content)
- **Inputs / outputs:**
  - **Inputs:** `Cap articles`, `Clean article text`
  - **Output:** `Normalize article fields`
- **Edge cases / failures:**
  - If one branch produces fewer items (e.g., failed fetch/extraction), position-based combine can misalign articles (wrong text attached to wrong RSS item).
  - If HTTP request fails for an item, that item may be missing on the cleaned-text branch; consider error handling or merging by `link`.

#### Node: Normalize article fields
- **Type / role:** Set; creates a consistent article schema for AI.
- **Configuration choices:**
  - Creates fields:
    - `article_id` (empty placeholder; filled later)
    - `title`, `link`, `date` from current JSON
    - `body_text` from merged cleaned text
    - `body_text_short` placeholder
- **Inputs / outputs:**
  - **Input:** `Combine article data`
  - **Output:** `Add id and excerpt`
- **Edge cases / failures:**
  - Missing `title/link/date` in RSS item results in empty fields; affects downstream ID generation and prompt quality.

#### Node: Add id and excerpt
- **Type / role:** Code (per item); truncates text and generates stable ID.
- **Configuration choices:**
  - `body_text_short` is truncated to **2500 chars** (adds `...`).
  - `article_id` generated from base64(link) truncated to 24 chars (stable, “cheap” ID).
- **Inputs / outputs:**
  - **Input:** `Normalize article fields`
  - **Outputs (parallel):**
    - To `List policy and template files`
    - To `Attach file inventory` (merge input index 0)
- **Edge cases / failures:**
  - If `link` is empty, base64 of empty string yields empty ID → collisions possible.
  - Very short/empty `body_text` reduces AI accuracy.

---

### Block 4 — Drive Inventory (Templates/Policies)
**Overview:** Lists files in a Google Drive folder and aggregates them into a single `drive_files` array used by the AI impact analyzer.  
**Nodes involved:** `List policy and template files`, `Aggregate file inventory`, `Attach file inventory`

#### Node: List policy and template files
- **Type / role:** Google Drive; lists files in a folder.
- **Configuration choices:**
  - Resource: file/folder listing
  - Folder ID from config: `{{$node["Config"].json.templatesFolderId}}`
  - Returns all files; fields limited to: `id`, `name`, `mimeType`, `modifiedTime`
  - `executeOnce: true` (list performed once, reused for all articles).
- **Inputs / outputs:**
  - **Input:** `Add id and excerpt`
  - **Output:** `Aggregate file inventory`
- **Credential requirements:**
  - Google Drive OAuth2 or service account (depending on n8n setup); must have access to the folder.
- **Edge cases / failures:**
  - Invalid folder ID / permissions.
  - Large folders: `returnAll: true` can be slow and memory-heavy; consider pagination or filtering.

#### Node: Aggregate file inventory
- **Type / role:** Code; collapses many file items into a single inventory object.
- **Configuration choices:**
  - Outputs one item: `{ drive_files: [ {id,name,mimeType,modifiedTime}, ... ] }`
- **Inputs / outputs:**
  - **Input:** `List policy and template files`
  - **Output:** `Attach file inventory` (merge input index 1)
- **Edge cases / failures:**
  - If Drive returns no files, `drive_files` becomes empty array; AI may propose missing docs more often.

#### Node: Attach file inventory
- **Type / role:** Merge; attaches the single file inventory object to each article item.
- **Configuration choices:**
  - Mode: **combine**
  - Combine by: **combineAll**
  - Intended effect: pair each article with the same `drive_files` list.
- **Inputs / outputs:**
  - **Inputs:**
    - Input 0: from `Add id and excerpt` (many article items)
    - Input 1: from `Aggregate file inventory` (single item)
  - **Output:** `Doc impact analyzer`
- **Edge cases / failures:**
  - If merge semantics don’t replicate as expected (version/behavior differences), articles may not receive `drive_files`.
  - If either input is empty, output may be empty, skipping AI analysis.

---

### Block 5 — Per-Article AI Impact Analysis (Gemini + Structured JSON)
**Overview:** For each article, an AI agent evaluates relevance and suggests which existing policy/template documents to review/update, or proposes missing documents. Output is forced into a JSON schema via a structured output parser.  
**Nodes involved:** `Gemini chat model`, `Structured output schema`, `Doc impact analyzer`

#### Node: Gemini chat model
- **Type / role:** LangChain Chat Model (Google Gemini); provides the LLM used by the analyzer agent.
- **Configuration choices:**
  - Default options (none specified).
- **Inputs / outputs:**
  - **Output (AI language model connection):** feeds into `Doc impact analyzer` as its model.
- **Credential requirements:**
  - Google Gemini / Google AI Studio credentials as supported by your n8n LangChain integration.
- **Edge cases / failures:**
  - Auth/quota errors.
  - Model availability/region restrictions.
  - Output variability; mitigated by structured parser + strict system message.

#### Node: Structured output schema
- **Type / role:** Structured Output Parser; constrains agent output to valid JSON matching a schema example.
- **Configuration choices:**
  - Schema example includes:
    - `article_id`, `title`, `link`, `date`
    - `relevance` (0–3 expected by prompt)
    - `summary`
    - `impacted_files` array (with `action: review|update`)
    - `missing_docs` array
- **Inputs / outputs:**
  - **Output (AI output parser connection):** into `Doc impact analyzer`.
- **Edge cases / failures:**
  - If the model returns invalid JSON, parser fails and the node errors.
  - Schema example is not a strict JSON Schema; it’s an example-based constraint—still helpful but not foolproof.

#### Node: Doc impact analyzer
- **Type / role:** LangChain Agent; analyzes each article against Drive file list.
- **Configuration choices (interpreted):**
  - Prompt includes:
    - Article header fields + `body_text_short`
    - `Existing documents: {{ JSON.stringify($json.drive_files) }}`
  - System message:
    - Role: “HR policy and contract analyst for Germany”
    - Must output **valid JSON only**, short, include relevance 0–3, impacted files, and missing docs proposals.
  - `hasOutputParser: true` to enforce structured JSON.
- **Inputs / outputs:**
  - **Main input:** from `Attach file inventory` (each item contains article + drive_files)
  - **Main output:** to `Build radar report`
  - **AI connections:** receives model from `Gemini chat model` and parser from `Structured output schema`
- **Edge cases / failures:**
  - Token limits if `drive_files` list is large (JSON stringified list can be long).
  - Empty `body_text_short` reduces relevance detection.
  - File-name-only matching can miss relevant documents if naming is inconsistent (agent can only infer from names).

---

### Block 6 — Weekly Aggregation + Email Authoring (OpenAI)
**Overview:** Aggregates all per-article JSON outputs into a single weekly report object (top 3, impacted files grouped, missing doc ideas), then uses OpenAI to write a scannable HTML email strictly based on that report.  
**Nodes involved:** `Build radar report`, `OpenAI chat model`, `Email writer`

#### Node: Build radar report
- **Type / role:** Code; converts AI outputs into one consolidated report.
- **Configuration choices (interpreted):**
  - Parses each item as JSON (accepts either `i.json.output` or `i.json`).
  - Sorts by `relevance` descending.
  - Builds:
    - `top3_articles` (title/link/date/relevance/summary)
    - `impacted_files` grouped by `file_id`, each with list of actions and source article metadata
    - `missing_doc_ideas` de-duplicated case-insensitively
    - `all_articles` raw normalized list
  - Returns **one item** report.
- **Inputs / outputs:**
  - **Input:** `Doc impact analyzer` (multiple items)
  - **Output:** `Email writer`
- **Edge cases / failures:**
  - If parser output shape differs (e.g., missing `output` field), parsing may drop items.
  - If no results, report fields become empty arrays; email writer must handle empties (prompt doesn’t explicitly cover “no news” weeks).

#### Node: OpenAI chat model
- **Type / role:** LangChain Chat Model (OpenAI); provides the LLM used by the email writer agent.
- **Configuration choices:**
  - Model: **gpt-5.2**
  - Timeout: **600000 ms** (10 minutes)
  - Built-in tools: none enabled.
- **Inputs / outputs:**
  - **Output (AI language model connection):** feeds into `Email writer`.
- **Credential requirements:**
  - OpenAI API key configured in n8n credentials.
- **Edge cases / failures:**
  - Auth/quota/model access issues.
  - Long generations or timeouts (mitigated by high timeout, but still possible).

#### Node: Email writer
- **Type / role:** LangChain Agent; generates HTML email body.
- **Configuration choices (interpreted):**
  - Input text: `{{ JSON.stringify($json) }}` (the weekly report)
  - System message:
    - Must output **ONLY valid HTML** (no markdown).
    - Must be calm/scannable; max width 600px; inline CSS; no emojis/icons.
    - Uses only provided info; no invented legal interpretation.
    - Mandatory structure: Header, Intro, Top 3, Docs to review/update, Missing topics, Suggested next steps.
    - “No subject line” in content (subject is handled by Gmail node).
- **Inputs / outputs:**
  - **Main input:** `Build radar report`
  - **Main output:** `Send radar email`
  - **AI connection:** uses `OpenAI chat model`
- **Edge cases / failures:**
  - Model may still output non-HTML or include extra text → downstream email may look broken.
  - If report is empty, the model might struggle to produce meaningful sections without “inventing”; prompt forbids invention, so output may be very thin.

---

### Block 7 — Send Email (Gmail)
**Overview:** Sends the HTML output to the configured recipient via Gmail.  
**Nodes involved:** `Send radar email`

#### Node: Send radar email
- **Type / role:** Gmail; sends outbound email.
- **Configuration choices:**
  - To: `{{$node["Config"].json.userEmail}}`
  - Subject: `"Turn HR news into policy update tasks with RSS, Google Drive, AI, and Gmail"`
  - Message body: `{{$json.output}}` (expects `Email writer` to set `output` field containing HTML)
- **Inputs / outputs:**
  - **Input:** `Email writer`
  - **Output:** none (terminal)
- **Credential requirements:**
  - Gmail OAuth2 credentials in n8n with permission to send mail.
- **Edge cases / failures:**
  - If `Email writer` output isn’t in `$json.output`, the email body will be empty.
  - Gmail sending limits, OAuth token expiration, restricted scopes.
  - HTML rendering differences across email clients.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly trigger | n8n-nodes-base.scheduleTrigger | Weekly entry point | — | Config | ## Collect items; Runs weekly on the schedule trigger; Reads the RSS feed you set in newsUrl; Keeps only items from the last 7 days; Caps processing with maxArticles to control runtime and cost |
| Config | n8n-nodes-base.set | Central configuration variables | Weekly trigger | Read RSS feed | ## Collect items; Runs weekly on the schedule trigger; Reads the RSS feed you set in newsUrl; Keeps only items from the last 7 days; Caps processing with maxArticles to control runtime and cost |
| Read RSS feed | n8n-nodes-base.rssFeedRead | Fetch RSS items | Config | Keep last 7 days | ## Collect items; Runs weekly on the schedule trigger; Reads the RSS feed you set in newsUrl; Keeps only items from the last 7 days; Caps processing with maxArticles to control runtime and cost |
| Keep last 7 days | n8n-nodes-base.filter | Filter to last 7 days | Read RSS feed | Cap articles | ## Collect items; Runs weekly on the schedule trigger; Reads the RSS feed you set in newsUrl; Keeps only items from the last 7 days; Caps processing with maxArticles to control runtime and cost |
| Cap articles | n8n-nodes-base.code | Limit number of processed articles | Keep last 7 days | Fetch article page; Combine article data | ## Collect items; Runs weekly on the schedule trigger; Reads the RSS feed you set in newsUrl; Keeps only items from the last 7 days; Caps processing with maxArticles to control runtime and cost |
| Fetch article page | n8n-nodes-base.httpRequest | Download article HTML | Cap articles | Extract article body | ## Extract article text; Fetches each article URL from the RSS item; Extracts the main body with a CSS selector; Removes common noise (ads, social blocks, scripts); Truncates the cleaned text to reduce tokens and cost; If the body is empty, adjust the CSS selector in the HTML extraction node for your source site |
| Extract article body | n8n-nodes-base.html | Extract main body HTML by CSS selector | Fetch article page | Clean article text | ## Extract article text; Fetches each article URL from the RSS item; Extracts the main body with a CSS selector; Removes common noise (ads, social blocks, scripts); Truncates the cleaned text to reduce tokens and cost; If the body is empty, adjust the CSS selector in the HTML extraction node for your source site |
| Clean article text | n8n-nodes-base.code | Clean/convert HTML to text | Extract article body | Combine article data | ## Extract article text; Fetches each article URL from the RSS item; Extracts the main body with a CSS selector; Removes common noise (ads, social blocks, scripts); Truncates the cleaned text to reduce tokens and cost; If the body is empty, adjust the CSS selector in the HTML extraction node for your source site |
| Combine article data | n8n-nodes-base.merge | Merge RSS fields + cleaned text | Cap articles; Clean article text | Normalize article fields | ## Analyze and send; Loads file names from your Drive folder (templatesFolderId); Compares each article to your document list and suggests review or updates; Aggregates results into a weekly report; Generates a calm, scannable HTML email and sends it to userEmail |
| Normalize article fields | n8n-nodes-base.set | Standardize article schema | Combine article data | Add id and excerpt | ## Analyze and send; Loads file names from your Drive folder (templatesFolderId); Compares each article to your document list and suggests review or updates; Aggregates results into a weekly report; Generates a calm, scannable HTML email and sends it to userEmail |
| Add id and excerpt | n8n-nodes-base.code | Create stable ID + truncate text | Normalize article fields | List policy and template files; Attach file inventory | ## Analyze and send; Loads file names from your Drive folder (templatesFolderId); Compares each article to your document list and suggests review or updates; Aggregates results into a weekly report; Generates a calm, scannable HTML email and sends it to userEmail |
| List policy and template files | n8n-nodes-base.googleDrive | List Drive files in folder | Add id and excerpt | Aggregate file inventory | ## Analyze and send; Loads file names from your Drive folder (templatesFolderId); Compares each article to your document list and suggests review or updates; Aggregates results into a weekly report; Generates a calm, scannable HTML email and sends it to userEmail |
| Aggregate file inventory | n8n-nodes-base.code | Aggregate file list into one item | List policy and template files | Attach file inventory | ## Analyze and send; Loads file names from your Drive folder (templatesFolderId); Compares each article to your document list and suggests review or updates; Aggregates results into a weekly report; Generates a calm, scannable HTML email and sends it to userEmail |
| Attach file inventory | n8n-nodes-base.merge | Attach Drive inventory to each article | Add id and excerpt; Aggregate file inventory | Doc impact analyzer | ## Analyze and send; Loads file names from your Drive folder (templatesFolderId); Compares each article to your document list and suggests review or updates; Aggregates results into a weekly report; Generates a calm, scannable HTML email and sends it to userEmail |
| Gemini chat model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM for per-article analysis | — | Doc impact analyzer (ai_languageModel) | ## Analyze and send; Loads file names from your Drive folder (templatesFolderId); Compares each article to your document list and suggests review or updates; Aggregates results into a weekly report; Generates a calm, scannable HTML email and sends it to userEmail |
| Structured output schema | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON output shape | — | Doc impact analyzer (ai_outputParser) | ## Analyze and send; Loads file names from your Drive folder (templatesFolderId); Compares each article to your document list and suggests review or updates; Aggregates results into a weekly report; Generates a calm, scannable HTML email and sends it to userEmail |
| Doc impact analyzer | @n8n/n8n-nodes-langchain.agent | Compare article vs Drive filenames; output JSON | Attach file inventory | Build radar report | ## Analyze and send; Loads file names from your Drive folder (templatesFolderId); Compares each article to your document list and suggests review or updates; Aggregates results into a weekly report; Generates a calm, scannable HTML email and sends it to userEmail |
| Build radar report | n8n-nodes-base.code | Aggregate all analyses into weekly report | Doc impact analyzer | Email writer | ## Analyze and send; Loads file names from your Drive folder (templatesFolderId); Compares each article to your document list and suggests review or updates; Aggregates results into a weekly report; Generates a calm, scannable HTML email and sends it to userEmail |
| OpenAI chat model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for email generation | — | Email writer (ai_languageModel) | ## Analyze and send; Loads file names from your Drive folder (templatesFolderId); Compares each article to your document list and suggests review or updates; Aggregates results into a weekly report; Generates a calm, scannable HTML email and sends it to userEmail |
| Email writer | @n8n/n8n-nodes-langchain.agent | Generate scannable HTML newsletter | Build radar report | Send radar email | ## Analyze and send; Loads file names from your Drive folder (templatesFolderId); Compares each article to your document list and suggests review or updates; Aggregates results into a weekly report; Generates a calm, scannable HTML email and sends it to userEmail |
| Send radar email | n8n-nodes-base.gmail | Send HTML email via Gmail | Email writer | — | ## Good to know; This sends article text and file names to LLMs; Respect the publisher’s terms and avoid redistributing full article text; If cost is too high, lower maxArticles or shorten body_text_short |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation (canvas note) | — | — | ## How it works; This workflow runs once per week and turns HR news into a short internal radar email.; It reads an RSS feed and keeps only articles from the last 7 days; It fetches each article page, extracts the main content, and cleans it into plain text; For each article, an AI agent compares the article against your policy and contract template file names from Google Drive; The agent outputs structured JSON with a relevance score, a short summary, and suggested documents to review or update; A second AI step aggregates everything into a calm, scannable HTML email and sends it to you; ## Setup steps; Set newsUrl to the RSS feed you want to monitor; Set templatesFolderId to the Google Drive folder that contains your policies and templates; Set userEmail to the recipient address for the weekly email; Tune maxArticles to cap cost and runtime; Run once manually to verify the HTML extractor CSS selector matches your publisher |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation (canvas note) | — | — | ## Good to know; This sends article text and file names to LLMs; Respect the publisher’s terms and avoid redistributing full article text; If cost is too high, lower maxArticles or shorten body_text_short |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation (canvas note) | — | — | ## Collect items; Runs weekly on the schedule trigger; Reads the RSS feed you set in newsUrl; Keeps only items from the last 7 days; Caps processing with maxArticles to control runtime and cost |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation (canvas note) | — | — | ## Extract article text; Fetches each article URL from the RSS item; Extracts the main body with a CSS selector; Removes common noise (ads, social blocks, scripts); Truncates the cleaned text to reduce tokens and cost; If the body is empty, adjust the CSS selector in the HTML extraction node for your source site |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation (canvas note) | — | — | ## Analyze and send; Loads file names from your Drive folder (templatesFolderId); Compares each article to your document list and suggests review or updates; Aggregates results into a weekly report; Generates a calm, scannable HTML email and sends it to userEmail |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Turn HR news into policy update tasks with RSS, Google Drive, AI, and Gmail* (or your preferred name).
   - Ensure the workflow is saved before configuring credentials.

2. **Add and configure the trigger**
   1) Add node: **Schedule Trigger**  
      - Set it to run **weekly**, on **day 1**, at **09:00**.

3. **Add configuration variables**
   1) Add node: **Set** and name it `Config`  
      - Add fields:
        - `newsUrl` (String): your RSS feed URL
        - `userEmail` (String): recipient email
        - `templatesFolderId` (String): Google Drive folder ID containing policies/templates
        - `maxArticles` (Number): e.g., 10  
      - Enable **Include Other Fields**.

4. **Read RSS feed**
   1) Add node: **RSS Feed Read** (`Read RSS feed`)
      - URL: expression referencing Config → `{{$node["Config"].json.newsUrl}}`
      - Enable “execute once” behavior if available (to avoid repeated reads).

5. **Filter to last 7 days**
   1) Add node: **Filter** (`Keep last 7 days`)
      - Condition: DateTime `{{$json.date}}` **is after** `{{$now.minus({ days: 7 }).toISO()}}`
      - Keep strict validation if you want early failure on bad dates.

6. **Cap number of items**
   1) Add node: **Code** (`Cap articles`)
      - Use code equivalent to:
        - Read `maxArticles` from Config
        - Return only the first N items  
      - Connect `Keep last 7 days` → `Cap articles`.

7. **Fetch and extract article text (per item)**
   1) Add node: **HTTP Request** (`Fetch article page`)
      - URL: `{{$json.link}}`
   2) Add node: **HTML** (`Extract article body`)
      - Operation: extract HTML content
      - CSS selector: `.article-detail__body`
      - Return value: HTML into `body_html`
   3) Add node: **Code** (`Clean article text`)
      - Clean HTML (remove scripts/styles/ads), convert to text, trim.
      - Output `body_text` (and optionally `url`).
   4) Connect: `Cap articles` → `Fetch article page` → `Extract article body` → `Clean article text`.

8. **Merge RSS metadata with cleaned text**
   1) Add node: **Merge** (`Combine article data`)
      - Mode: **Combine**
      - Combine by: **Position**
   2) Connect:
      - `Cap articles` → `Combine article data` (input 0)
      - `Clean article text` → `Combine article data` (input 1)

9. **Normalize article fields**
   1) Add node: **Set** (`Normalize article fields`)
      - Create fields:
        - `article_id` (empty)
        - `title` = `{{$json.title}}`
        - `link` = `{{$json.link}}`
        - `date` = `{{$json.date}}`
        - `body_text` = `{{$json.body_text}}`
        - `body_text_short` (empty)

10. **Add stable ID and excerpt**
   1) Add node: **Code** (`Add id and excerpt`)
      - Truncate `body_text` into `body_text_short` (e.g., 2500 chars).
      - Compute `article_id` from `link` (base64 truncated is acceptable).
   2) Connect: `Normalize article fields` → `Add id and excerpt`.

11. **List template/policy files from Google Drive**
   1) Add node: **Google Drive** (`List policy and template files`)
      - Operation: list files in folder
      - Folder ID: `{{$node["Config"].json.templatesFolderId}}`
      - Return all files
      - Restrict returned fields to `id,name,mimeType,modifiedTime`
      - Enable “execute once” if available.
   2) **Credentials:** configure Google Drive OAuth2 (must have read access to the folder).
   3) Connect: `Add id and excerpt` → `List policy and template files`.

12. **Aggregate file inventory into one object**
   1) Add node: **Code** (`Aggregate file inventory`)
      - Convert the multiple Drive items into one item containing `drive_files: [...]`.
   2) Connect: `List policy and template files` → `Aggregate file inventory`.

13. **Attach file inventory to each article**
   1) Add node: **Merge** (`Attach file inventory`)
      - Mode: **Combine**
      - Combine by: **Combine All**
   2) Connect:
      - `Add id and excerpt` → `Attach file inventory` (input 0)
      - `Aggregate file inventory` → `Attach file inventory` (input 1)

14. **Configure per-article AI impact analyzer (Gemini)**
   1) Add node: **LangChain Chat Model (Google Gemini)** (`Gemini chat model`)
      - Configure credentials for Gemini.
   2) Add node: **Structured Output Parser** (`Structured output schema`)
      - Provide the example structure used in the workflow (article fields, relevance, summary, impacted_files, missing_docs).
   3) Add node: **LangChain Agent** (`Doc impact analyzer`)
      - Prompt text should include the article fields and `JSON.stringify(drive_files)`.
      - System message should enforce:
        - Germany HR policy analyst role
        - relevance 0–3
        - JSON only, short
      - Attach:
        - `Gemini chat model` to the agent’s **Language Model** input
        - `Structured output schema` to the agent’s **Output Parser** input
   4) Connect: `Attach file inventory` → `Doc impact analyzer`.

15. **Aggregate into weekly report**
   1) Add node: **Code** (`Build radar report`)
      - Parse each agent output
      - Sort by relevance
      - Build `top3_articles`, grouped `impacted_files`, `missing_doc_ideas`, `all_articles`
   2) Connect: `Doc impact analyzer` → `Build radar report`.

16. **Generate the HTML email (OpenAI)**
   1) Add node: **LangChain Chat Model (OpenAI)** (`OpenAI chat model`)
      - Model: `gpt-5.2`
      - Timeout: 600000 ms
      - Configure OpenAI credentials.
   2) Add node: **LangChain Agent** (`Email writer`)
      - Input text: `{{ JSON.stringify($json) }}` (weekly report)
      - System message: enforce “HTML only”, scannable structure, no invention, inline CSS, max width 600px.
      - Attach `OpenAI chat model` as the agent’s **Language Model**.
   3) Connect: `Build radar report` → `Email writer`.

17. **Send the email via Gmail**
   1) Add node: **Gmail** (`Send radar email`)
      - To: `{{$node["Config"].json.userEmail}}`
      - Subject: your chosen static subject (the provided workflow uses a descriptive subject)
      - Message body: `{{$json.output}}` (must match where the agent outputs HTML)
   2) Configure Gmail OAuth2 credentials with send permission.
   3) Connect: `Email writer` → `Send radar email`.

18. **Validation run**
   - Run manually once:
     - Confirm RSS items contain a `date` field compatible with the Filter node.
     - Confirm the CSS selector returns `body_html` for your publisher.
     - Confirm AI outputs valid JSON for the analyzer and valid HTML for the writer.
     - Confirm Gmail receives a well-rendered email.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| How it works: Runs weekly; reads RSS; keeps last 7 days; fetches and cleans article text; AI compares articles against Drive policy/template filenames; outputs structured JSON; aggregates into calm HTML email; sends to you. | From canvas note “How it works” |
| Setup steps: set `newsUrl`, `templatesFolderId`, `userEmail`, tune `maxArticles`, run manually to verify the HTML extractor CSS selector matches your publisher. | From canvas note “Setup steps” |
| Good to know: This sends article text and file names to LLMs; respect publisher terms and avoid redistributing full article text; reduce cost by lowering `maxArticles` or shortening `body_text_short`. | From canvas note “Good to know” |