Send AI-curated weekly news digests with RSS, Vector DB & GPT-4o

https://n8nworkflows.xyz/workflows/send-ai-curated-weekly-news-digests-with-rss--vector-db---gpt-4o-11951


# Send AI-curated weekly news digests with RSS, Vector DB & GPT-4o

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensé ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow ingests tech news articles daily from multiple RSS feeds, stores them in an **in-memory vector store** with **OpenAI embeddings**, then—once per week—uses an **AI agent (GPT‑4o)** to retrieve the most relevant articles from the vector store (based on user interests) and generate an email-friendly weekly digest, sent via Gmail.

**Target use cases:**
- Personal or team weekly tech newsletter automation
- Topic-focused news curation using semantic retrieval (vector search)
- Separating ingestion (daily) from delivery (weekly) for efficiency

### Logical blocks
**1.1 Daily News Collection (RSS ingestion)**  
Triggered daily; loads RSS feed URLs, iterates through them, reads RSS items, and normalizes fields.

**1.2 Vectorization & Storage (embeddings + vector store insert)**  
Splits article text into chunks, creates documents with metadata, generates embeddings, inserts into an in-memory vector store.

**1.3 Weekly Digest Generation (preferences → retrieval → GPT)**  
Triggered weekly; sets user preferences, provides the agent with a retrieval tool backed by the same vector store, and runs GPT‑4o to generate a curated digest.

**1.4 Formatting & Email Delivery**  
Converts AI output from Markdown to HTML and emails the result via Gmail.

---

## 2. Block-by-Block Analysis

### 2.1 Daily News Collection (RSS ingestion)

**Overview:**  
Runs on a daily schedule to fetch articles from configured RSS feeds. It iterates over feed URLs, reads items, and normalizes key fields (title/content/date/link) before sending them to the storage pipeline.

**Nodes involved:**
- Get Articles Daily
- Set Tech News RSS Feeds
- Split Out
- Read RSS News Feeds
- Set and Normalize Fields

#### Node: **Get Articles Daily**
- **Type / role:** `Schedule Trigger` — entry point for daily ingestion.
- **Configuration:** Runs on an interval schedule (default interval object present; exact cadence depends on n8n UI setting).
- **Outputs:** Main output to **Set Tech News RSS Feeds**.
- **Potential failures / edge cases:** If misconfigured schedule (timezone expectations); workflow won’t run as expected.

#### Node: **Set Tech News RSS Feeds**
- **Type / role:** `Set` — defines the array of RSS feed URLs.
- **Key config:** Sets field `rss` as an array:
  - `https://www.engadget.com/rss.xml`
  - `https://feeds.arstechnica.com/arstechnica/index`
  - `https://www.theverge.com/rss/index.xml`
  - `https://www.wired.com/feed/rss`
  - `https://www.technologyreview.com/topnews.rss`
  - `https://techcrunch.com/feed/`
- **Output:** To **Split Out**.
- **Edge cases:** Invalid feed URL, blocked/redirected feeds, rate limiting.

#### Node: **Split Out**
- **Type / role:** `Split Out` — iterates over items in an array field to process each feed URL separately.
- **Key config:** `fieldToSplitOut = rss`
- **Input/Output:** From **Set Tech News RSS Feeds** → to **Read RSS News Feeds** (one item per URL).
- **Edge cases:** `rss` missing/not an array → node error or no items.

#### Node: **Read RSS News Feeds**
- **Type / role:** `RSS Feed Read` — fetches and parses each RSS feed.
- **Key config:** `url = {{ $json.rss }}`, `ignoreSSL = false`
- **Output:** To **Set and Normalize Fields**
- **Failure modes:** Network timeouts, SSL errors, feed parsing errors, feed returns non-RSS content, HTTP 403/429.

#### Node: **Set and Normalize Fields**
- **Type / role:** `Set` — standardizes article fields for downstream embedding/document creation.
- **Key expressions:**
  - `title = {{ $json.title }}`
  - `content = {{ $json['content:encodedSnippet'] ?? $json.contentSnippet }}`
  - `date = {{ $json.isoDate }}`
  - `link = {{ $json.link }}`
- **Output:** To **Store News Articles**
- **Edge cases:**
  - Some RSS items may not have `isoDate` or `contentSnippet` → may produce `null`/empty content.
  - HTML/encoded snippets may be partial; summaries may be too short for good embeddings.

---

### 2.2 Vectorization & Storage (embeddings + vector store insert)

**Overview:**  
Transforms each RSS item into chunked documents, attaches metadata, generates OpenAI embeddings, and inserts them into an **in-memory** vector store under a shared key (`news_store_key`).

**Nodes involved:**
- Recursive Character Text Splitter
- Default Data Loader
- Embeddings OpenAI
- Store News Articles

#### Node: **Recursive Character Text Splitter**
- **Type / role:** LangChain text splitter — chunks long text for improved embedding quality and retrieval.
- **Key config:** `chunkSize = 3000` characters.
- **Connections:** Provides `ai_textSplitter` output to **Default Data Loader**.
- **Edge cases:** Very short content → single chunk; very long content → multiple chunks; chunking may split mid-sentence (acceptable but can reduce coherence).

#### Node: **Default Data Loader**
- **Type / role:** LangChain document loader — constructs documents from JSON with content + metadata.
- **Key config (interpreted):**
  - **Document text** (expression mode):  
    `{{ $json.title }}\n{{ $json.content }}`
  - **Metadata added:**
    - `title = {{ $json.title }}`
    - `createDate = {{ $now.toISO() }}`
    - `publishDate = {{ $json.date }}`
    - `link = {{ $json.link }}`
- **Input/Output:** Receives text-splitter output; emits `ai_document` to **Store News Articles**.
- **Edge cases:**
  - Missing fields (title/content/date/link) reduce metadata quality.
  - If `$json.content` is empty, embeddings will be low-signal.

#### Node: **Embeddings OpenAI**
- **Type / role:** OpenAI embeddings provider (LangChain).
- **Credentials:** `openAiApi`
- **Connections:** Provides `ai_embedding` to **Store News Articles**.
- **Failure modes:** Invalid API key, model/endpoint issues, rate limits, payload too large (if chunking fails), network timeouts.
- **Version notes:** Node is `typeVersion 1.2`; embedding model selection is not explicitly shown—defaults apply.

#### Node: **Store News Articles**
- **Type / role:** `Vector Store In Memory` (LangChain) — inserts documents + embeddings.
- **Key config:**
  - `mode = insert`
  - `memoryKey = news_store_key` (shared identifier used later for retrieval)
- **Inputs:** Main data from **Set and Normalize Fields**, plus `ai_document` from **Default Data Loader**, plus `ai_embedding` from **Embeddings OpenAI**.
- **Edge cases / limitations:**
  - **In-memory store is ephemeral**: data is lost on n8n restart, redeploy, or workflow reload (important for “weekly” behavior).
  - Potential duplicates: daily ingestion may reinsert the same articles unless deduplication is added.

---

### 2.3 Weekly Digest Generation (preferences → retrieval → GPT)

**Overview:**  
Once per week, the workflow sets user preferences (topics + number of items) and runs an AI Agent powered by GPT‑4o. The agent can call a retrieval tool to query the vector store for relevant recent articles.

**Nodes involved:**
- Send Weekly Summary
- Your topics of interest
- Embeddings OpenAI2
- Get News Articles
- OpenAI Chat Model
- News reader AI

#### Node: **Send Weekly Summary**
- **Type / role:** `Schedule Trigger` — weekly entry point.
- **Key config:** Weekly schedule: `weeks`, triggers **Day 1** at **05:00** (n8n timezone dependent).
- **Output:** To **Your topics of interest**
- **Edge cases:** Timezone mismatch; schedule not aligned with intended “Monday 5am” in your locale.

#### Node: **Your topics of interest**
- **Type / role:** `Set` — user preferences for curation.
- **Key config:**
  - `Interests = " games"` (note leading space as configured)
  - `Number of news items to include = "15"` (string)
- **Output:** To **News reader AI**
- **Edge cases:** Non-numeric string for count could confuse prompting; leading/trailing spaces.

#### Node: **Embeddings OpenAI2**
- **Type / role:** OpenAI embeddings provider used for retrieval queries.
- **Credentials:** `openAiApi`
- **Connections:** Supplies `ai_embedding` to **Get News Articles**
- **Failures:** Same as Embeddings OpenAI (auth, rate limits, timeouts).
- **Why separate:** Common pattern: one embedding node for insert pipeline, another for retrieval pipeline.

#### Node: **Get News Articles**
- **Type / role:** `Vector Store In Memory` in **retrieve-as-tool** mode — exposes vector search as an AI tool.
- **Key config:**
  - `mode = retrieve-as-tool`
  - `toolName = get_news`
  - `toolDescription = "Call this tool to get the latest news articles."`
  - `topK = 20`
  - `memoryKey = news_store_key` (must match insert node)
- **Connections:** Outputs an `ai_tool` connection into **News reader AI** (agent can call it).
- **Edge cases:**
  - If the in-memory store is empty (restart happened, ingestion hasn’t run, etc.), retrieval returns little/no results.
  - `topK=20` may return too many similar items unless deduplication is handled by the agent prompt.

#### Node: **OpenAI Chat Model**
- **Type / role:** LangChain chat model provider for the agent.
- **Key config:** Model set to `gpt-4o`.
- **Credentials:** `openAiApi`
- **Connections:** Provides `ai_languageModel` to **News reader AI**
- **Failure modes:** API auth errors, rate limiting, model availability, latency/timeouts.

#### Node: **News reader AI**
- **Type / role:** `AI Agent` — orchestrates prompt + tool usage to produce the weekly digest.
- **Prompt (user text):**  
  `Summarize the most relevant tech news published in the past 7 days. Today is {{ $now }}`
- **System message highlights (uses workflow fields):**
  - Write as a weekly tech newsletter in clear English.
  - Prioritize: `{{ $json.Interests }}`
  - Exactly `{{ $json['Number of news items to include'] }}` distinct news items.
  - Each item includes: title + direct link + 1–3 sentence summary.
  - Avoid duplicates.
- **Inputs:**
  - Main data from **Your topics of interest**
  - Tool from **Get News Articles**
  - Model from **OpenAI Chat Model**
- **Output:** To **Convert Response to an Email-Friendly Format**
- **Edge cases / behavior notes:**
  - “Past 7 days” constraint is not enforced at retrieval level (no metadata filtering); it relies on the agent reasoning.
  - If stored items lack `publishDate` or if ingestion is stale, the agent may include older items.
  - Agent may fail to call the tool if prompt doesn’t strongly require it; consider adding explicit instruction like “Use get_news tool”.

---

### 2.4 Formatting & Email Delivery

**Overview:**  
Converts the agent’s Markdown output into HTML and emails it to a specified recipient via Gmail OAuth2.

**Nodes involved:**
- Convert Response to an Email-Friendly Format
- Send Newsletter

#### Node: **Convert Response to an Email-Friendly Format**
- **Type / role:** `Markdown` — transforms Markdown → HTML.
- **Key config:** `mode = markdownToHtml`
- **Input expression:** `markdown = {{ $json.output }}`
- **Output:** To **Send Newsletter**
- **Edge cases:** If agent output is not in `output` field (depending on agent node version/behavior), expression may be empty. (Sometimes agent output is under `$json.output`, `$json.text`, or a nested structure—verify in executions.)

#### Node: **Send Newsletter**
- **Type / role:** `Gmail` — sends the weekly email.
- **Key config:**
  - `sendTo = "enter receiver mail here"`
  - `subject = "Weekly tech newsletter"`
  - `message = {{ $json.data }}` (this is the HTML produced by the Markdown node)
- **Credentials:** `gmailOAuth2`
- **Failure modes:** OAuth expired/invalid, Gmail API scope issues, sending limits, invalid recipient, HTML content blocked by policies.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Get Articles Daily | Schedule Trigger | Daily ingestion trigger | — | Set Tech News RSS Feeds | ## Daily News Collection (RSS)\nNews is scraped daily from multiple platforms\nusing their RSS feeds.\nEach article is cleaned, categorized,\nand prepared for vector storage. |
| Set Tech News RSS Feeds | Set | Define RSS feed URL list | Get Articles Daily | Split Out | ## Daily News Collection (RSS)\nNews is scraped daily from multiple platforms\nusing their RSS feeds.\nEach article is cleaned, categorized,\nand prepared for vector storage. |
| Split Out | Split Out | Iterate over RSS URLs | Set Tech News RSS Feeds | Read RSS News Feeds | ## Daily News Collection (RSS)\nNews is scraped daily from multiple platforms\nusing their RSS feeds.\nEach article is cleaned, categorized,\nand prepared for vector storage. |
| Read RSS News Feeds | RSS Feed Read | Fetch & parse RSS items | Split Out | Set and Normalize Fields | ## Daily News Collection (RSS)\nNews is scraped daily from multiple platforms\nusing their RSS feeds.\nEach article is cleaned, categorized,\nand prepared for vector storage. |
| Set and Normalize Fields | Set | Normalize article fields | Read RSS News Feeds | Store News Articles | ## Daily News Collection (RSS)\nNews is scraped daily from multiple platforms\nusing their RSS feeds.\nEach article is cleaned, categorized,\nand prepared for vector storage. |
| Recursive Character Text Splitter | LangChain Text Splitter | Chunk article text | — (used as AI input dependency) | Default Data Loader | ## Vector Store Storage\nAll news articles are stored in a vector database\nwith embeddings and category information.\nThis enables fast and relevant topic-based searches. |
| Default Data Loader | LangChain Document Loader | Build documents + metadata | Recursive Character Text Splitter | Store News Articles | ## Vector Store Storage\nAll news articles are stored in a vector database\nwith embeddings and category information.\nThis enables fast and relevant topic-based searches. |
| Embeddings OpenAI | OpenAI Embeddings (LangChain) | Create embeddings for stored docs | — (used as AI input dependency) | Store News Articles (ai_embedding) | ## Vector Store Storage\nAll news articles are stored in a vector database\nwith embeddings and category information.\nThis enables fast and relevant topic-based searches. |
| Store News Articles | Vector Store In Memory (insert) | Insert documents into vector store | Set and Normalize Fields + Default Data Loader + Embeddings OpenAI | — | ## Vector Store Storage\nAll news articles are stored in a vector database\nwith embeddings and category information.\nThis enables fast and relevant topic-based searches. |
| Send Weekly Summary | Schedule Trigger | Weekly digest trigger | — | Your topics of interest | ## Weekly Trigger & User Preferences\n\nOn a weekly basis, this workflow runs to prepare news emails.\nUser preferences include:\n• Area of interest (topics)\n• Number of news items\n• Email address |
| Your topics of interest | Set | User preference inputs | Send Weekly Summary | News reader AI | ## Weekly Trigger & User Preferences\n\nOn a weekly basis, this workflow runs to prepare news emails.\nUser preferences include:\n• Area of interest (topics)\n• Number of news items\n• Email address |
| Embeddings OpenAI2 | OpenAI Embeddings (LangChain) | Query embeddings for retrieval tool | — (used as AI input dependency) | Get News Articles (ai_embedding) | ## Weekly Trigger & User Preferences\n\nOn a weekly basis, this workflow runs to prepare news emails.\nUser preferences include:\n• Area of interest (topics)\n• Number of news items\n• Email address |
| Get News Articles | Vector Store In Memory (retrieve-as-tool) | Retrieval tool for agent | Embeddings OpenAI2 | News reader AI (ai_tool) | ## Weekly Trigger & User Preferences\n\nOn a weekly basis, this workflow runs to prepare news emails.\nUser preferences include:\n• Area of interest (topics)\n• Number of news items\n• Email address |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM powering the agent | — (used as AI input dependency) | News reader AI (ai_languageModel) | ## Weekly Trigger & User Preferences\n\nOn a weekly basis, this workflow runs to prepare news emails.\nUser preferences include:\n• Area of interest (topics)\n• Number of news items\n• Email address |
| News reader AI | AI Agent (LangChain) | Curate weekly digest using tool + GPT‑4o | Your topics of interest + Get News Articles + OpenAI Chat Model | Convert Response to an Email-Friendly Format | ## Weekly Trigger & User Preferences\n\nOn a weekly basis, this workflow runs to prepare news emails.\nUser preferences include:\n• Area of interest (topics)\n• Number of news items\n• Email address |
| Convert Response to an Email-Friendly Format | Markdown | Markdown → HTML conversion | News reader AI | Send Newsletter | ## Weekly Trigger & User Preferences\n\nOn a weekly basis, this workflow runs to prepare news emails.\nUser preferences include:\n• Area of interest (topics)\n• Number of news items\n• Email address |
| Send Newsletter | Gmail | Send email via Gmail | Convert Response to an Email-Friendly Format | — | ## Weekly Trigger & User Preferences\n\nOn a weekly basis, this workflow runs to prepare news emails.\nUser preferences include:\n• Area of interest (topics)\n• Number of news items\n• Email address |
| Sticky Note | Sticky Note | Documentation | — | — | ## Daily News Collection (RSS)\nNews is scraped daily from multiple platforms\nusing their RSS feeds.\nEach article is cleaned, categorized,\nand prepared for vector storage. |
| Sticky Note1 | Sticky Note | Documentation | — | — | ## Vector Store Storage\nAll news articles are stored in a vector database\nwith embeddings and category information.\nThis enables fast and relevant topic-based searches. |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## Weekly Trigger & User Preferences\n\nOn a weekly basis, this workflow runs to prepare news emails.\nUser preferences include:\n• Area of interest (topics)\n• Number of news items\n• Email address |
| Sticky Note3 | Sticky Note | Documentation / global explanation | — | — | ## How it works\n\nThis workflow sends weekly topic-based news emails using an AI agent.\nNews articles are first collected daily from multiple platforms via RSS feeds\nand stored in a vector database with category and embedding information.\n\nOn a weekly schedule, the workflow fetches relevant news from the vector store\nbased on user-defined interests and the number of articles required.\nThe selected news is then passed to an AI agent, which converts the raw data\ninto a clean, readable email format. Finally, the generated content is sent\nto the user via email.\n\nThe separation between daily ingestion and weekly delivery ensures\nefficient processing, better performance, and reusable news data.\n\n## Setup steps\n\n1. Add your RSS feed URLs in the RSS Feed nodes\n2. Configure vector database credentials\n3. Add your AI model credentials for the agent\n4. Configure email service credentials\n5. Adjust the weekly schedule if needed\n6. Customize the AI prompt or email format as required |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger (Daily)**
   1. Add node: **Schedule Trigger** → name **Get Articles Daily**  
   2. Configure it to run daily (choose preferred hour/timezone in UI).

2) **Define RSS feed list**
   1. Add node: **Set** → name **Set Tech News RSS Feeds**  
   2. Add an assignment:
      - Field: `rss` (Type: Array)
      - Value: list of RSS URLs (Engadget, Ars Technica, The Verge, Wired, MIT Tech Review, TechCrunch).

3) **Loop over feeds**
   1. Add node: **Split Out** → name **Split Out**  
   2. Set **Field to split out**: `rss`
   3. Connect: **Get Articles Daily → Set Tech News RSS Feeds → Split Out**

4) **Read RSS**
   1. Add node: **RSS Feed Read** → name **Read RSS News Feeds**
   2. Set **URL**: `{{ $json.rss }}`
   3. Connect: **Split Out → Read RSS News Feeds**

5) **Normalize article fields**
   1. Add node: **Set** → name **Set and Normalize Fields**
   2. Add fields:
      - `title` = `{{ $json.title }}`
      - `content` = `{{ $json['content:encodedSnippet'] ?? $json.contentSnippet }}`
      - `date` = `{{ $json.isoDate }}`
      - `link` = `{{ $json.link }}`
   3. Connect: **Read RSS News Feeds → Set and Normalize Fields**

6) **Create chunking + document building pipeline**
   1. Add node: **Recursive Character Text Splitter**  
      - `chunkSize = 3000`
   2. Add node: **Default Data Loader**
      - Configure **Text/content** in expression mode to:
        - `{{ $json.title }}\n{{ $json.content }}`
      - Configure **Metadata**:
        - `title` = `{{ $json.title }}`
        - `createDate` = `{{ $now.toISO() }}`
        - `publishDate` = `{{ $json.date }}`
        - `link` = `{{ $json.link }}`
   3. Connect Text Splitter → Default Data Loader via the LangChain/AI connector (`ai_textSplitter` to `ai_document` path as per node UI).

7) **Embeddings for storage**
   1. Add node: **Embeddings OpenAI** (LangChain)
   2. Create/configure **OpenAI API credentials** in n8n and select them in this node.

8) **Vector store insert**
   1. Add node: **Vector Store In Memory** → name **Store News Articles**
   2. Set:
      - `mode = insert`
      - `memoryKey = news_store_key`
   3. Connect:
      - Main: **Set and Normalize Fields → Store News Articles**
      - AI connections:
        - **Default Data Loader → Store News Articles** (document input)
        - **Embeddings OpenAI → Store News Articles** (embedding input)

9) **Create Trigger (Weekly)**
   1. Add node: **Schedule Trigger** → name **Send Weekly Summary**
   2. Configure weekly schedule (e.g., Monday 05:00).

10) **Set user preferences**
   1. Add node: **Set** → name **Your topics of interest**
   2. Add fields:
      - `Interests` (string) e.g. `games` (remove leading space if undesired)
      - `Number of news items to include` (string or number; keep consistent) e.g. `15`
   3. Connect: **Send Weekly Summary → Your topics of interest**

11) **Create retrieval tool from the same vector store**
   1. Add node: **Embeddings OpenAI** → name **Embeddings OpenAI2**  
      - Select the same **OpenAI API** credentials.
   2. Add node: **Vector Store In Memory** → name **Get News Articles**
      - `mode = retrieve-as-tool`
      - `memoryKey = news_store_key`
      - `toolName = get_news`
      - `toolDescription = Call this tool to get the latest news articles.`
      - `topK = 20`
   3. Connect: **Embeddings OpenAI2 → Get News Articles** via `ai_embedding`.

12) **Create the AI Agent**
   1. Add node: **OpenAI Chat Model** → name **OpenAI Chat Model**
      - Model: `gpt-4o`
      - Credentials: OpenAI API
   2. Add node: **AI Agent** → name **News reader AI**
      - User prompt text:  
        `Summarize the most relevant tech news published in the past 7 days. Today is {{ $now }}`
      - System message: include the preference variables:
        - `{{ $json.Interests }}`
        - `{{ $json['Number of news items to include'] }}`
      - Ensure the agent has:
        - **Language model connection** from **OpenAI Chat Model**
        - **Tool connection** from **Get News Articles**
   3. Connect main: **Your topics of interest → News reader AI**

13) **Convert to HTML**
   1. Add node: **Markdown** → name **Convert Response to an Email-Friendly Format**
   2. Set:
      - Mode: `markdownToHtml`
      - Markdown input: `{{ $json.output }}`
   3. Connect: **News reader AI → Convert Response to an Email-Friendly Format**

14) **Send email**
   1. Add node: **Gmail** → name **Send Newsletter**
   2. Configure **Gmail OAuth2 credentials** (Google Cloud project, OAuth consent, scopes as required by n8n Gmail node).
   3. Set:
      - To: your recipient email
      - Subject: `Weekly tech newsletter`
      - Message: `{{ $json.data }}` (HTML from Markdown node)
   4. Connect: **Convert Response to an Email-Friendly Format → Send Newsletter**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow separates daily ingestion from weekly delivery for efficiency and reusability of stored news data. | From “How it works” sticky note |
| Setup steps listed: add RSS URLs, configure vector DB credentials, configure AI model credentials, configure email credentials, adjust schedule, customize prompt/email format. | From “How it works” sticky note |
| Important limitation: Vector Store In Memory is ephemeral; weekly runs may have empty data if n8n restarts or ingestion hasn’t populated the store. Consider switching to a persistent vector store (e.g., Pinecone, Qdrant, Postgres pgvector) for production reliability. | Implementation consideration based on node choice |

