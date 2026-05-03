Draft daily crypto news tweets with CryptoPanic, Google, GPT‑4o and Gmail

https://n8nworkflows.xyz/workflows/draft-daily-crypto-news-tweets-with-cryptopanic--google--gpt-4o-and-gmail-11960


# Draft daily crypto news tweets with CryptoPanic, Google, GPT‑4o and Gmail

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow (“Twitter Agent”) runs on a schedule, pulls the latest crypto news from CryptoPanic, filters it with an AI “Narrative Analyst” to keep only stories matching specific narratives (stablecoins, regulation, RWA, influential people/institutions), performs Google search-based context lookup, scrapes the best matching page, purifies page content, and uses an AI “Tweet Architect” to draft punchy daily-news tweets. It then archives tweets to Google Sheets and emails drafts for human review via Gmail.

**Target use cases:**
- Daily crypto news monitoring for a defined niche (stablecoins/regulation/RWA/institutions/personalities)
- Producing beginner-friendly tweet drafts with a “human-in-the-loop” review step
- Archiving draft outputs for later publishing

### Logical blocks
1.1 **Triggering** (Scheduled runs)  
1.2 **News sourcing + failover** (CryptoPanic primary + backup token)  
1.3 **AI curation / filtering** (LLM + structured output parsing into a list of selected items)  
1.4 **Per-item processing** (Batch loop over curated items)  
1.5 **Context research + scraping** (Google Custom Search → fetch page HTML)  
1.6 **Content cleaning + data assembly** (Purify HTML → merge curated data + search URL + cleaned content)  
1.7 **Tweet generation** (LLM agent generates a single tweet per item)  
1.8 **Archiving + notifications** (Merge recap → save to Sheets → email for review)

---

## 2. Block-by-Block Analysis

### 2.1 Triggering

**Overview:** Starts the workflow every 6 hours (as configured).  
**Nodes involved:** `Daily Check`

#### Node: Daily Check
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — entry point.
- **Configuration (interpreted):** Runs every **6 hours** at **minute 59**.
- **Connections:**  
  - **Output →** `Fetch Market News`
- **Edge cases / failures:**  
  - Timezone/instance time drift affects firing times.
  - High-frequency schedules can overlap if executions take longer than the interval (depending on n8n concurrency settings).

---

### 2.2 News sourcing + failover (CryptoPanic)

**Overview:** Calls CryptoPanic API to fetch posts; if the primary request returns an error object, it falls back to a backup CryptoPanic credential.  
**Nodes involved:** `Fetch Market News`, `Check API Status`, `Fetch Market News Backup`

#### Node: Fetch Market News
- **Type / role:** HTTP Request — primary CryptoPanic fetch.
- **Configuration:**
  - URL: `https://cryptopanic.com/api/developer/v2/posts/`
  - Auth: `httpQueryAuth` via “generic credential type” (query-string token style).
  - **On Error:** “continueRegularOutput” (workflow continues with an `error` field present in output).
- **Connections:**  
  - **Output →** `Check API Status`
- **Failure modes:**
  - Invalid/expired token → HTTP 401/403.
  - Rate limiting (likely 429) → triggers failover path if it produces an `error` field.
  - Response structure changes could break downstream assumptions (`results` array).

#### Node: Check API Status
- **Type / role:** IF node — chooses primary vs backup path.
- **Configuration (interpreted):**
  - Condition checks whether `$json.error` is **empty**.
  - **If TRUE (no error)** → proceed with `Narrative Analyst`.
  - **If FALSE (error present)** → call `Fetch Market News Backup`.
- **Connections:**  
  - **True output →** `Narrative Analyst`  
  - **False output →** `Fetch Market News Backup`
- **Edge cases:**
  - Some HTTP failures may not set `$json.error` in the way expected; the IF condition may misroute.
  - If CryptoPanic returns a valid 200 response but embeds error info differently, this check may not detect it.

#### Node: Fetch Market News Backup
- **Type / role:** HTTP Request — backup CryptoPanic fetch using a second credential.
- **Configuration:** Same endpoint as primary; separate credential `CryptoPanic2`. Also “continueRegularOutput”.
- **Connections:**  
  - **Output →** `Narrative Analyst`
- **Failure modes:** Same as primary; if both keys are rate-limited, pipeline still continues but curation may have no input.

---

### 2.3 AI curation / filtering (Narrative Analyst + structured parsing)

**Overview:** Uses an LLM-powered agent to select relevant news from CryptoPanic `results` based on defined themes, and enforces a strict JSON array format via a structured output parser.  
**Nodes involved:** `Curation Intelligence`, `Parser AI`, `Structured Output Parser`, `Narrative Analyst`, `Format output`

#### Node: Curation Intelligence
- **Type / role:** OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — language model for the Narrative Analyst agent.
- **Configuration:**
  - Model: `gpt-4o-mini`
  - Credentials: OpenAI account
- **Connections:**  
  - Provides **AI languageModel** input to `Narrative Analyst`.
- **Failure modes:** Missing OpenAI credentials, model name not available, rate limits, timeouts.

#### Node: Parser AI
- **Type / role:** OpenAI Chat Model — model used by the structured output parser auto-fix mechanism.
- **Configuration:**
  - Model: `gpt-4.1-mini`
  - Credentials: OpenAI account
- **Connections:**  
  - Provides **AI languageModel** input to `Structured Output Parser`.
- **Failure modes:** Same as above; if auto-fix fails, parsing may still error or output malformed JSON.

#### Node: Structured Output Parser
- **Type / role:** LangChain Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — forces/repairs JSON output.
- **Configuration:**
  - `autoFix: true` (attempts to fix malformed JSON using the connected model)
  - Schema example: array of objects with keys like `title`, `summary`, `category`, `published_at`, `slug`, `id` (example shows `slug`; agent instructions require `source`—see edge case below).
- **Connections:**  
  - Provides **AI outputParser** to `Narrative Analyst`.
- **Edge cases:**
  - **Schema mismatch risk:** The Narrative Analyst instructions say output must include `source` (and not `slug`), but the schema example includes `slug`. This can yield inconsistent field names and break downstream nodes expecting `slug`.
  - If CryptoPanic input is empty or non-JSON, the parser may produce `[]` or fail.

#### Node: Narrative Analyst
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — selects and summarizes relevant news items.
- **Configuration choices:**
  - PromptType: “define”
  - Prompt includes:
    - Strict selection criteria (stablecoins, regulation, RWA, influential figures, institutions).
    - Output must be **JSON array only** with keys:
      - `title`, `summary`, `category`, `published_at` (YYYY-MM-DD), `source` (from slug), `id`
    - Input is injected as: `{{ JSON.stringify($json["results"], null, 2) }}`
  - `hasOutputParser: true` → connected to `Structured Output Parser` for enforcement.
- **Connections:**  
  - **Input:** from `Check API Status` true path or `Fetch Market News Backup`
  - **Output →** `Format output`
  - **AI languageModel:** from `Curation Intelligence`
  - **AI outputParser:** from `Structured Output Parser`
- **Failure modes / edge cases:**
  - If the upstream CryptoPanic response doesn’t contain `results`, expression resolves to `undefined` and the agent may output nothing or invalid JSON.
  - Field naming inconsistency: instructions say `source`, but later nodes use `slug` (e.g., Google search query uses `$json['slug']`). You should align this (recommended: output `slug` consistently, or map `source → slug` after parsing).

#### Node: Format output
- **Type / role:** Split Out (`n8n-nodes-base.splitOut`) — converts the array into one item per news entry.
- **Configuration:**
  - `fieldToSplitOut: "output"`
- **Connections:**  
  - **Output →** `Process Selected News`
- **Edge cases:**
  - This node expects the Narrative Analyst result to be in `$json.output`. Depending on how the agent node returns data in your n8n version, it may instead return the array at the root (e.g., `$json` itself) or in another field. If `output` is missing, Split Out yields zero items or errors.
  - If the LLM returns `[]`, no downstream per-item processing occurs.

---

### 2.4 Per-item processing (batch loop)

**Overview:** Iterates through curated news items, enabling per-item research, tweet drafting, saving, and emailing.  
**Nodes involved:** `Process Selected News`

#### Node: Process Selected News
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — batch iterator.
- **Configuration:** Default options (batch size not explicitly set; n8n defaults apply).
- **Connections:**  
  - **Output 1 →** `Send Review Notification` (email per item; depends on data availability)  
  - **Output 1 →** `Daily Recap Aggregator` (input 0)  
  - **Output 1 →** `Google Deep Context Research`  
  - **Output 1 →** `Data Assembler` (input 0)
- **Edge cases:**
  - If items are large or numerous, execution time increases; may overlap schedule runs.
  - If you intended “send email only after tweet generation”, current wiring sends email directly from this node, before tweet creation (see also the email node content mismatch later).

---

### 2.5 Context research + scraping

**Overview:** Uses Google Custom Search API to find the best URL for each selected news slug, then downloads that page’s HTML.  
**Nodes involved:** `Google Deep Context Research`, `Content Scraper`

#### Node: Google Deep Context Research
- **Type / role:** HTTP Request — Google Custom Search API.
- **Configuration:**
  - URL: `https://www.googleapis.com/customsearch/v1`
  - Auth: `httpQueryAuth` (API key in query params via credential)
  - Query parameters:
    - `cx` hardcoded: `015006539695750909647:jf5zbnim0fy`
    - `q` = `{{$json['slug']}}`
    - `num` = `1`
  - Notes indicate credentials must be set.
- **Connections:**  
  - **Output →** `Data Assembler` (input 1) and `Content Scraper`
- **Failure modes / edge cases:**
  - If the curated item doesn’t contain `slug` (because the agent outputs `source` instead), `q` becomes empty → poor results.
  - Google API quota/rate limits, invalid key/CX, or search engine configuration restrictions → 4xx errors.
  - If response has no `items`, downstream nodes accessing `items[0].link` will fail.

#### Node: Content Scraper
- **Type / role:** HTTP Request — fetches the HTML content of the first Google search result.
- **Configuration:**
  - URL expression: `{{ $json['items'][0]['link'] }}`
  - On Error: `continueRegularOutput` (continues even if request fails)
- **Connections:**  
  - **Output →** `Content Purifier`
- **Failure modes / edge cases:**
  - If `items[0]` missing: expression error (or undefined URL) → request fails.
  - Many sites block automated requests (403), require JS rendering, or return paywalls.
  - Large pages can exceed memory/time limits.

---

### 2.6 Content cleaning + data assembly

**Overview:** Strips HTML noise, limits content size, and merges curated metadata + Google URL + cleaned page content into one object for tweet drafting.  
**Nodes involved:** `Content Purifier`, `Data Assembler`, `Data Purifier`

#### Node: Content Purifier
- **Type / role:** Code (`n8n-nodes-base.code`) — cleans HTML and truncates text.
- **Configuration (interpreted):**
  - Reads `const html = $input.first().json.data ?? ""`
    - Assumes the HTTP node output puts raw response body in `data`.
  - Removes `<script>`, `<style>`, `<footer>`, SVG, and many tags via regex.
  - Removes various self-closing tags.
  - Truncates to `MAX = 12000` characters.
  - Outputs `{ content: snippet }`.
- **Connections:**  
  - **Output →** `Data Assembler` (input 2)
- **Failure modes / edge cases:**
  - If the HTTP Request node returns body in a different field (commonly `body` in some configurations), `data` will be undefined → empty content.
  - Regex-based stripping can leave artifacts; non-HTML (PDF, JSON, etc.) will not be handled well.
  - 12k characters may still be large for some LLM token limits depending on model and other fields.

#### Node: Data Assembler
- **Type / role:** Merge (`n8n-nodes-base.merge`) — combines 3 inputs by position.
- **Configuration:**
  - Mode: `combine`
  - Combine by: `combineByPosition`
  - Inputs: 3
    - Input 0: from `Process Selected News` (curated item)
    - Input 1: from `Google Deep Context Research` (search result JSON)
    - Input 2: from `Content Purifier` (cleaned `content`)
- **Connections:**  
  - **Output →** `Data Purifier`
- **Failure modes / edge cases:**
  - Combine-by-position is fragile: if any branch yields fewer items (e.g., Google search fails for some items), data can misalign across items.
  - If Content Scraper fails and Content Purifier outputs empty, you may still proceed but tweet quality degrades.

#### Node: Data Purifier
- **Type / role:** Code — removes noisy Google fields and extracts the first URL.
- **Configuration (interpreted):**
  - Takes merged object, extracts `first = item.items?.[0]?.link || null`
  - Removes: `queries`, `kind`, `searchInformation`, `context`, `items` from the object
  - Outputs cleaned fields + `url: first`
- **Connections:**  
  - **Output →** `Tweet Architect`
- **Failure modes / edge cases:**
  - If `items` missing, `url` becomes null; Tweet Architect prompt requires a link at the end, so output may violate requirements.
  - The merge may overwrite similarly named fields; ensure curated item fields (`title`, `summary`, `category`, `published_at`, `id`, `slug/source`) survive.

---

### 2.7 Tweet generation

**Overview:** Drafts one tweet per selected item using the cleaned article content and metadata; tweet is intended to be under 280 characters and end with the source URL.  
**Nodes involved:** `Brain Engine`, `Tweet Architect`

#### Node: Brain Engine
- **Type / role:** OpenAI Chat Model — model backing the Tweet Architect agent.
- **Configuration:**
  - Model: `gpt-3.5-turbo`
- **Connections:**  
  - Provides **AI languageModel →** `Tweet Architect`
- **Failure modes:** model availability, rate limits; tweet quality may be weaker than newer models.

#### Node: Tweet Architect
- **Type / role:** LangChain Agent — generates the final tweet text.
- **Configuration choices:**
  - PromptType: “define”
  - Key rules:
    - Must start with: `🚨 DAILY NEWS`
    - Double line breaks between blocks
    - End with `{{ $json["url"] }}`
    - Exactly 2 hashtags based on category
    - No external tools; use provided `content`
    - Output must be only the tweet (no preamble)
  - Inputs referenced: `title`, `summary`, `category`, `published_at`, `slug`, `id`, `url`, `content`
- **Connections:**  
  - **Input:** from `Data Purifier`
  - **Output →** `Daily Recap Aggregator` (input 1)
  - **AI languageModel:** from `Brain Engine`
- **Edge cases / failures:**
  - If `category` values don’t match expected mapping (note prompt typo: “personnalite” vs earlier “personality”), hashtags may default incorrectly.
  - If `url` is null, tweet violates “must end with link”.
  - 280-char constraint is LLM-enforced; may occasionally exceed limit.

---

### 2.8 Archiving + notifications

**Overview:** Aggregates data for saving and sends review notifications. Saves tweet drafts to Google Sheets and emails a digest-like message for approval.  
**Nodes involved:** `Daily Recap Aggregator`, `Save tweet`, `Send Review Notification`

#### Node: Daily Recap Aggregator
- **Type / role:** Merge — combines all items for archival.
- **Configuration:**
  - Mode: `combine`
  - CombineBy: `combineAll` (collects inputs)
  - Input 0: from `Process Selected News`
  - Input 1: from `Tweet Architect`
- **Connections:**  
  - **Output →** `Save tweet`
- **Edge cases:**
  - Combining “all” can create nested structures depending on n8n merge semantics; ensure `Save tweet` receives the fields it maps (id/date/slug/title/category/output).
  - Field naming mismatch: Tweet Architect output is typically in a field like `output` (agent) or `text`; your Sheets node uses `$json['output']`.

#### Node: Save tweet
- **Type / role:** Google Sheets — append or update a row.
- **Configuration:**
  - Operation: `appendOrUpdate`
  - Matching column: `id`
  - Sheet: document “X news”, tab “tweets”
  - Columns mapped:
    - `id` = `{{$json['id']}}`
    - `date` = `{{$json['published_at']}}`
    - `slug` = `{{$json['slug']}}`
    - `title` = `{{$json['title']}}`
    - `category` = `{{$json['category']}}`
    - `tweet` = `{{$json['output']}}`
- **Connections:**  
  - **Input:** from `Daily Recap Aggregator`
  - **Output →** `Process Selected News` (this creates a loop back into the batch node)
- **Important edge case (logic risk):**
  - **Feedback loop:** `Save tweet` outputs back into `Process Selected News`. This can cause unintended re-processing or looping behavior depending on how SplitInBatches interprets incoming items. In most designs, Sheets save would terminate the per-item branch or go to a separate “next batch” trigger, not back into the batch node.
- **Failure modes:**
  - OAuth issues (expired token, missing scopes)
  - Sheet/tab not found, column mismatch, permission errors
  - Data type mismatch (less likely since values are strings)

#### Node: Send Review Notification
- **Type / role:** Gmail — sends an email for review.
- **Configuration:**
  - To: `user@example.com` (placeholder)
  - Subject: `{{$json['category']}} : {{ $json['title'] }}`
  - Body includes:
    - `Categorie: {{ $json['category'] }}`
    - `Date: {{ $json['date'] }}`
    - `Titre: {{ $json['title'] }}`
    - `{{ $json['tweet'] }}`
  - `appendAttribution: false`
- **Connections:**  
  - **Input:** from `Process Selected News`
- **Edge cases / data mismatch:**
  - At this point in the flow, the item likely does **not** yet have `date` or `tweet` fields (those are created/mapped in the Sheets node). The curated item has `published_at`, not `date`. Tweet text is produced later by Tweet Architect, likely in `output`. Result: email may be missing tweet content and date unless you rewire it to trigger after tweet generation (or map fields correctly).
- **Failure modes:** Gmail OAuth scope issues, sending limits, invalid recipient.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Project intro / setup guidance | — | — | # Try It Out!\n\nLaunch X-Ray—your personal crypto intelligence agent that tracks market narratives, performs deep research, and drafts viral tweets.\n\n**To get started:**\n\n1. **Connect all credentials** (OpenAI, CryptoPanic, Google Search, Sheets & Gmail).\n2. **Setup your API Keys**:\n   • **CryptoPanic**: Get your token [here](https://cryptopanic.com/developers/api/). *Pro Tip: Create a second account/key if you hit rate limits.*\n   • **Google Search**: Enable the [Custom Search API](https://console.cloud.google.com/) and create a [Programmable Search Engine](https://programmablesearchengine.google.com/) for your CX ID.\n3. **Customize your niche**: Adjust the keywords in the **Narrative Analyst** agent (e.g., RWA, DeFi, L2).\n4. **Activate the workflow**: Receive daily tweet drafts in your inbox and Google Sheets for final review.\n\n## ⚙️ How it works:\n• 🔍 **Monitor**: Fetches real-time news from CryptoPanic.\n• 🧠 **Analyze**: AI filters for high-impact stories based on your niche.\n• ✍️ **Draft**: Generates structured, viral tweet drafts with deep context.\n• 📬 **Review**: \"Human-in-the-loop\" approval via Gmail & Sheets.\n\n## Questions or Need Help?\n\nFor setup assistance, customization, or workflow support 🌐 [Contact me here](https://www.cadaero.ovh)\n\nHappy automating! -- Cyrille d'Urbal |
| Sticky Note1 | Sticky Note | Section header: Data sourcing | — | — | ## 1. Data Sourcing\n\nFetches the latest market news from CryptoPanic API\n\n### 💡 API Configuration\n**CryptoPanic Setup:**\n1. Log in to [CryptoPanic Developers](https://cryptopanic.com/developers/api/) to get your **API Token**.\n2. **Pro Tip**: If you hit rate limits, create a second account to generate an additional API Key as a fallback. |
| Daily Check | Schedule Trigger | Periodic execution | — | Fetch Market News | ## 1. Data Sourcing\n\nFetches the latest market news from CryptoPanic API\n\n### 💡 API Configuration\n**CryptoPanic Setup:**\n1. Log in to [CryptoPanic Developers](https://cryptopanic.com/developers/api/) to get your **API Token**.\n2. **Pro Tip**: If you hit rate limits, create a second account to generate an additional API Key as a fallback. |
| Fetch Market News | HTTP Request | Fetch CryptoPanic posts (primary key) | Daily Check | Check API Status | ## 1. Data Sourcing\n\nFetches the latest market news from CryptoPanic API\n\n### 💡 API Configuration\n**CryptoPanic Setup:**\n1. Log in to [CryptoPanic Developers](https://cryptopanic.com/developers/api/) to get your **API Token**.\n2. **Pro Tip**: If you hit rate limits, create a second account to generate an additional API Key as a fallback. |
| Check API Status | IF | Route to backup key if API call errored | Fetch Market News | Narrative Analyst; Fetch Market News Backup | ## 1. Data Sourcing\n\nFetches the latest market news from CryptoPanic API\n\n### 💡 API Configuration\n**CryptoPanic Setup:**\n1. Log in to [CryptoPanic Developers](https://cryptopanic.com/developers/api/) to get your **API Token**.\n2. **Pro Tip**: If you hit rate limits, create a second account to generate an additional API Key as a fallback. |
| Fetch Market News Backup | HTTP Request | Fetch CryptoPanic posts (backup key) | Check API Status (false path) | Narrative Analyst | ## 1. Data Sourcing\n\nFetches the latest market news from CryptoPanic API\n\n### 💡 API Configuration\n**CryptoPanic Setup:**\n1. Log in to [CryptoPanic Developers](https://cryptopanic.com/developers/api/) to get your **API Token**.\n2. **Pro Tip**: If you hit rate limits, create a second account to generate an additional API Key as a fallback. |
| Sticky Note2 | Sticky Note | Section header: AI filtering & research | — | — | ## 2. AI Filtering & Research\n\nIdentifies relevant news based on your keywords and performs deep research for context.\n\n### 💡 API Configuration\n**Google Custom Search Setup:**\n1. Enable the **Custom Search API** in the [Google Cloud Console](https://console.cloud.google.com/) to get your **API Key**.\n2. Create a search engine in the [Programmable Search Engine](https://programmablesearchengine.google.com/) to get your **Search Engine ID (CX)**. |
| Curation Intelligence | OpenAI Chat Model (LangChain) | LLM for curation agent | — | Narrative Analyst (ai_languageModel) | ## 2. AI Filtering & Research\n\nIdentifies relevant news based on your keywords and performs deep research for context.\n\n### 💡 API Configuration\n**Google Custom Search Setup:**\n1. Enable the **Custom Search API** in the [Google Cloud Console](https://console.cloud.google.com/) to get your **API Key**.\n2. Create a search engine in the [Programmable Search Engine](https://programmablesearchengine.google.com/) to get your **Search Engine ID (CX)**. |
| Parser AI | OpenAI Chat Model (LangChain) | LLM used for parser auto-fix | — | Structured Output Parser (ai_languageModel) | ## 2. AI Filtering & Research\n\nIdentifies relevant news based on your keywords and performs deep research for context.\n\n### 💡 API Configuration\n**Google Custom Search Setup:**\n1. Enable the **Custom Search API** in the [Google Cloud Console](https://console.cloud.google.com/) to get your **API Key**.\n2. Create a search engine in the [Programmable Search Engine](https://programmablesearchengine.google.com/) to get your **Search Engine ID (CX)**. |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforce/repair JSON array output | Parser AI (ai_languageModel) | Narrative Analyst (ai_outputParser) | ## 2. AI Filtering & Research\n\nIdentifies relevant news based on your keywords and performs deep research for context.\n\n### 💡 API Configuration\n**Google Custom Search Setup:**\n1. Enable the **Custom Search API** in the [Google Cloud Console](https://console.cloud.google.com/) to get your **API Key**.\n2. Create a search engine in the [Programmable Search Engine](https://programmablesearchengine.google.com/) to get your **Search Engine ID (CX)**. |
| Narrative Analyst | Agent (LangChain) | Filter + summarize relevant news into structured JSON | Check API Status (true) / Fetch Market News Backup | Format output | ## 2. AI Filtering & Research\n\nIdentifies relevant news based on your keywords and performs deep research for context.\n\n### 💡 API Configuration\n**Google Custom Search Setup:**\n1. Enable the **Custom Search API** in the [Google Cloud Console](https://console.cloud.google.com/) to get your **API Key**.\n2. Create a search engine in the [Programmable Search Engine](https://programmablesearchengine.google.com/) to get your **Search Engine ID (CX)**. |
| Format output | Split Out | Convert curated array into per-item records | Narrative Analyst | Process Selected News | ## 2. AI Filtering & Research\n\nIdentifies relevant news based on your keywords and performs deep research for context.\n\n### 💡 API Configuration\n**Google Custom Search Setup:**\n1. Enable the **Custom Search API** in the [Google Cloud Console](https://console.cloud.google.com/) to get your **API Key**.\n2. Create a search engine in the [Programmable Search Engine](https://programmablesearchengine.google.com/) to get your **Search Engine ID (CX)**. |
| Process Selected News | Split In Batches | Iterate over selected stories | Format output; Save tweet | Send Review Notification; Daily Recap Aggregator; Google Deep Context Research; Data Assembler |  |
| Sticky Note3 | Sticky Note | Section header: Content creation | — | — | ## 3. Content Creation\n\nExtracts full article content and generates engaging tweet drafts with a structured format. |
| Google Deep Context Research | HTTP Request | Google Custom Search query per story | Process Selected News | Data Assembler; Content Scraper | ## 3. Content Creation\n\nExtracts full article content and generates engaging tweet drafts with a structured format. |
| Content Scraper | HTTP Request | Fetch HTML of best matching result | Google Deep Context Research | Content Purifier | ## 3. Content Creation\n\nExtracts full article content and generates engaging tweet drafts with a structured format. |
| Content Purifier | Code | Strip HTML + truncate to 12k chars | Content Scraper | Data Assembler | ## 3. Content Creation\n\nExtracts full article content and generates engaging tweet drafts with a structured format. |
| Data Assembler | Merge | Combine curated + search + content | Process Selected News; Google Deep Context Research; Content Purifier | Data Purifier | ## 3. Content Creation\n\nExtracts full article content and generates engaging tweet drafts with a structured format. |
| Data Purifier | Code | Keep only needed fields + extract URL | Data Assembler | Tweet Architect | ## 3. Content Creation\n\nExtracts full article content and generates engaging tweet drafts with a structured format. |
| Brain Engine | OpenAI Chat Model (LangChain) | LLM for tweet drafting agent | — | Tweet Architect (ai_languageModel) | ## 3. Content Creation\n\nExtracts full article content and generates engaging tweet drafts with a structured format. |
| Tweet Architect | Agent (LangChain) | Generate final tweet text | Data Purifier | Daily Recap Aggregator | ## 3. Content Creation\n\nExtracts full article content and generates engaging tweet drafts with a structured format. |
| Sticky Note4 | Sticky Note | Section header: Archiving & notification | — | — | ## 4. Archiving & Notification\n\nSaves everything to your database and sends a daily digest for your final approval. |
| Daily Recap Aggregator | Merge | Combine item + tweet for persistence | Process Selected News; Tweet Architect | Save tweet | ## 4. Archiving & Notification\n\nSaves everything to your database and sends a daily digest for your final approval. |
| Save tweet | Google Sheets | Append/update tweet drafts archive | Daily Recap Aggregator | Process Selected News | ## 4. Archiving & Notification\n\nSaves everything to your database and sends a daily digest for your final approval. |
| Send Review Notification | Gmail | Email draft(s) for human review | Process Selected News | — | ## 4. Archiving & Notification\n\nSaves everything to your database and sends a daily digest for your final approval. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
   - Name: `Twitter Agent`
   - Activate later after credentials are set.

2) **Add Sticky Notes (optional but recommended)**
   - Add 5 sticky notes with the exact content from the JSON:
     - “Try It Out!” note (includes links to CryptoPanic, Google Cloud, Programmable Search Engine, contact link)
     - “1. Data Sourcing”
     - “2. AI Filtering & Research”
     - “3. Content Creation”
     - “4. Archiving & Notification”

3) **Trigger**
   - Add **Schedule Trigger** node: `Daily Check`
   - Configure: every `6` hours, at minute `59` (or your preferred cadence)

4) **CryptoPanic primary fetch**
   - Add **HTTP Request** node: `Fetch Market News`
   - Method: GET
   - URL: `https://cryptopanic.com/api/developer/v2/posts/`
   - Authentication: **Generic Credential Type → HTTP Query Auth**
   - Create credential `CryptoPanic`:
     - Add query param token per your CryptoPanic API requirements (commonly `auth_token=...`)
   - Set **On Error**: Continue (so errors appear in output)

5) **IF failover routing**
   - Add **IF** node: `Check API Status`
   - Condition: `$json.error` **is empty**
   - True output = primary success path
   - False output = failover path

6) **CryptoPanic backup fetch**
   - Add **HTTP Request** node: `Fetch Market News Backup`
   - Same URL and settings as primary
   - Credential: `CryptoPanic2` (second token)
   - On Error: Continue
   - Wire:
     - `Daily Check` → `Fetch Market News` → `Check API Status`
     - `Check API Status (true)` → `Narrative Analyst`
     - `Check API Status (false)` → `Fetch Market News Backup` → `Narrative Analyst`

7) **LLM model for curation**
   - Add **OpenAI Chat Model** node: `Curation Intelligence`
   - Model: `gpt-4o-mini`
   - Set OpenAI credential.

8) **Structured output enforcement**
   - Add **OpenAI Chat Model** node: `Parser AI`
   - Model: `gpt-4.1-mini`
   - Set OpenAI credential.
   - Add **Structured Output Parser** node: `Structured Output Parser`
     - Enable **Auto-fix**
     - Paste the schema example (array of objects) as in the workflow
   - Connect:
     - `Parser AI` → `Structured Output Parser` via **ai_languageModel**

9) **Narrative Analyst agent**
   - Add **AI Agent** node: `Narrative Analyst`
   - Prompt: paste the provided “You are a crypto news analysis assistant…” instruction set
   - Ensure it references `{{$json["results"]}}`
   - Enable “Has Output Parser”
   - Connect:
     - `Curation Intelligence` → `Narrative Analyst` via **ai_languageModel**
     - `Structured Output Parser` → `Narrative Analyst` via **ai_outputParser**

10) **Split curated list into items**
   - Add **Split Out** node: `Format output`
   - Field to split: `output`
   - Connect: `Narrative Analyst` → `Format output`

11) **Batch iterator**
   - Add **Split In Batches** node: `Process Selected News`
   - Default options (or set a batch size you prefer, e.g., 5)
   - Connect: `Format output` → `Process Selected News`

12) **Google Custom Search request**
   - Add **HTTP Request** node: `Google Deep Context Research`
   - URL: `https://www.googleapis.com/customsearch/v1`
   - Auth: **HTTP Query Auth** credential named `Google Search` (API key)
   - Enable “Send Query Parameters”
   - Params:
     - `cx` = your Programmable Search Engine CX (currently hardcoded in the JSON)
     - `q` = `{{$json['slug']}}` (or adjust if your curated field is `source`)
     - `num` = `1`
   - Connect: `Process Selected News` → `Google Deep Context Research`

13) **Scrape the first result**
   - Add **HTTP Request** node: `Content Scraper`
   - URL: `{{$json['items'][0]['link']}}`
   - On Error: Continue
   - Connect: `Google Deep Context Research` → `Content Scraper`

14) **Purify page content**
   - Add **Code** node: `Content Purifier`
   - Paste the JS cleaning code (removes HTML and truncates to 12k chars)
   - Connect: `Content Scraper` → `Content Purifier`

15) **Merge data (3-way)**
   - Add **Merge** node: `Data Assembler`
   - Mode: Combine
   - Combine By: Position
   - Number of inputs: 3
   - Connect:
     - `Process Selected News` → `Data Assembler` (input 0)
     - `Google Deep Context Research` → `Data Assembler` (input 1)
     - `Content Purifier` → `Data Assembler` (input 2)

16) **Clean merged object + extract URL**
   - Add **Code** node: `Data Purifier`
   - Paste the JS code that extracts `items[0].link` to `url` and removes Google noise fields
   - Connect: `Data Assembler` → `Data Purifier`

17) **LLM model for tweet drafting**
   - Add **OpenAI Chat Model** node: `Brain Engine`
   - Model: `gpt-3.5-turbo`
   - Set OpenAI credential.

18) **Tweet drafting agent**
   - Add **AI Agent** node: `Tweet Architect`
   - Paste the “You are a copywriter specializing in crypto news…” prompt
   - Ensure it references `{{$json["url"]}}` and `{{$json["content"]}}`
   - Connect:
     - `Brain Engine` → `Tweet Architect` via **ai_languageModel**
     - `Data Purifier` → `Tweet Architect` (main)

19) **Recap merge**
   - Add **Merge** node: `Daily Recap Aggregator`
   - Mode: Combine
   - Combine By: All
   - Connect:
     - `Process Selected News` → `Daily Recap Aggregator` (input 0)
     - `Tweet Architect` → `Daily Recap Aggregator` (input 1)

20) **Save to Google Sheets**
   - Add **Google Sheets** node: `Save tweet`
   - Credential: Google Sheets OAuth2
   - Operation: Append or Update
   - Spreadsheet: select your document
   - Sheet/tab: `tweets`
   - Create columns in the sheet: `date`, `slug`, `title`, `category`, `tweet`, plus `id` (used for matching)
   - Mapping:
     - `id` = `{{$json['id']}}`
     - `date` = `{{$json['published_at']}}`
     - `slug` = `{{$json['slug']}}`
     - `title` = `{{$json['title']}}`
     - `category` = `{{$json['category']}}`
     - `tweet` = `{{$json['output']}}`
   - Connect: `Daily Recap Aggregator` → `Save tweet`

21) **Send email for review**
   - Add **Gmail** node: `Send Review Notification`
   - Credential: Gmail OAuth2
   - To: set your email
   - Subject: `{{$json['category']}} : {{$json['title']}}`
   - Body: include category/date/title/tweet fields
   - Connect as desired (note: in the provided workflow it is connected directly from `Process Selected News`, but for correct tweet inclusion you typically connect it **after** `Tweet Architect` or after `Save tweet` and map the correct fields).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| CryptoPanic API token | https://cryptopanic.com/developers/api/ |
| Google Custom Search API enablement | https://console.cloud.google.com/ |
| Programmable Search Engine (CX) setup | https://programmablesearchengine.google.com/ |
| Support / contact | https://www.cadaero.ovh |
| Operational note: workflow includes primary + backup CryptoPanic credentials to handle rate limits | Mentioned in sticky note and implemented via `Check API Status` routing |
| Data consistency warning: curated output field names (`source` vs `slug`) and downstream dependencies (`q={{$json.slug}}`, Sheets uses `$json.slug`) must be aligned | Adjust Narrative Analyst output or add a mapping step |
| Logic warning: `Save tweet` loops back into `Process Selected News` and `Send Review Notification` triggers before tweet drafting | Consider rewiring to avoid loops and send emails after tweet generation |