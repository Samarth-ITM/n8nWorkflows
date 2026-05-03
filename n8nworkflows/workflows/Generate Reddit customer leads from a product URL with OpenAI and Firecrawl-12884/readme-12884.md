Generate Reddit customer leads from a product URL with OpenAI and Firecrawl

https://n8nworkflows.xyz/workflows/generate-reddit-customer-leads-from-a-product-url-with-openai-and-firecrawl-12884


# Generate Reddit customer leads from a product URL with OpenAI and Firecrawl

## 1. Workflow Overview

**Title:** Generate Reddit customer leads from a product URL with OpenAI and Firecrawl

**Purpose / use case:**  
This workflow receives a **product landing page URL** (plus a `searchId`) via an authenticated webhook, scrapes the page with **Firecrawl**, uses **OpenAI (LangChain nodes)** to (1) analyze the product and target market, (2) generate **10 Reddit search keywords**, then searches Reddit for posts matching those keywords and uses AI to **assess relevance**. It finally sends “conversations” (qualified Reddit posts) to an external backend in stages.

### 1.1 Input Reception & Environment Setup
Receives `productUrl` + `searchId`, loads backend URL + API key used by subsequent HTTP requests.

### 1.2 Scrape & Normalize Landing Page Data
Scrapes the product page, extracts metadata/markdown, and normalizes missing fields.

### 1.3 AI Website Analysis → Send to Server
Generates `website_summary` + `target_market` in structured JSON, sanitizes values for safe transport, then sends to backend.

### 1.4 AI Keyword Generation → Send to Server
Generates 10 Reddit-oriented keywords (structured JSON) and sends them to backend.

### 1.5 Reddit Leads: Keyword #1 Pipeline (Search → Clean → AI relevance → Aggregate → Format → Send)
Searches Reddit for keyword1, cleans fields, AI-assesses each post, aggregates results, formats “relevant” posts into markdown, sends to backend.

### 1.6 Reddit Leads: Keyword #2 Pipeline (Search → Clean → AI relevance → Aggregate → Format → Send)
Same as #1 but for keyword2.

### 1.7 Reddit Leads: Keywords #3–#10 Pipeline (Batch loop)
Builds an array of keywords 3–10, iterates in batches, searches Reddit for each keyword, AI-assesses relevance, aggregates/filters to a final payload, formats combined markdown, sends final result to backend.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Environment Setup

**Overview:**  
Starts workflow via an authenticated webhook, then sets two configuration values used by all downstream HTTP calls: backend base URL and a shared API key header.

**Nodes involved:** Webhook, Set Environment Variables

#### Node: **Webhook**
- **Type / role:** `n8n-nodes-base.webhook` — entry point (HTTP POST trigger).
- **Config choices:**
  - **Method:** POST
  - **Path:** `b2d7fb9a-f92c-4174-998e-58e902d0ce81`
  - **Auth:** Header auth (credential `leadsgen`)
- **Key expected input fields (from `body`):**
  - `productUrl` (used later by Firecrawl)
  - `searchId` (passed to backend)
- **Outputs:** to **Set Environment Variables**
- **Failure modes / edge cases:**
  - Missing/invalid auth header → webhook rejects.
  - Missing `body.productUrl` or `body.searchId` → later expressions fail or send incomplete payloads.
- **Version notes:** typeVersion 2.1 (ensure matching n8n version supports `headerAuth`).

#### Node: **Set Environment Variables**
- **Type / role:** `n8n-nodes-base.set` — stores workflow config values in item JSON.
- **Config choices:** Sets:
  - `N8N_WEBHOOK_API_KEY` = `your-secret-key/api-key`
  - `BACKEND_API_URL` = `your-ngrok-url/railway-server-base-url`
- **Outputs:** to **Scrape Product URL and get its content**
- **Failure modes / edge cases:**
  - If `BACKEND_API_URL` doesn’t end with `/`, subsequent nodes concatenate `...URL }}api/...` and may create invalid URLs.
  - Using placeholder values will cause HTTP request failures.

---

### Block 2 — Scrape & Normalize Landing Page Data

**Overview:**  
Scrapes the landing page content via Firecrawl and normalizes the scrape output into a predictable schema for AI prompting and backend transmission.

**Nodes involved:** Scrape Product URL and get its content, Sanitize Results

#### Node: **Scrape Product URL and get its content**
- **Type / role:** `@mendable/n8n-nodes-firecrawl.firecrawl` — website scraping.
- **Config choices:**
  - **Operation:** Scrape
  - **URL:** `{{ $('Webhook').item.json.body.productUrl }}`
  - **Parsers:** includes `pdf` (in addition to default formats)
- **Credentials:** Firecrawl API credential `Firecrawl 20k`
- **Outputs:** to **Sanitize Results**
- **Failure modes / edge cases:**
  - Invalid URL / blocked site / bot protection → scrape fails or returns partial data.
  - Firecrawl quota/auth errors.
  - Large pages may truncate markdown or exceed downstream token limits.

#### Node: **Sanitize Results**
- **Type / role:** `n8n-nodes-base.code` — extracts and defaults key scrape fields.
- **Config choices (interpreted):**
  - Produces a single item with:
    - `markdown` from `data.markdown` (or `not_applicable`)
    - `title` from `metadata['og:title'][0]` (or `not_applicable`)
    - `image` from `metadata['og:image'][0]`
    - `description` from `metadata['og:description']` (note: may be array or string)
    - `favicon` from `metadata.favicon`
- **Outputs:** to **Analyze Product URL Scrape**
- **Failure modes / edge cases:**
  - If Firecrawl changes metadata shape (array vs string), later expressions may behave unexpectedly.
  - `description` is not consistently indexed (`[0]` here is not used, but later nodes sometimes assume `[0]`).

---

### Block 3 — AI Website Analysis → Send to Server

**Overview:**  
Uses OpenAI to generate a structured summary and target market analysis, then sanitizes values for safe JSON/transport and sends the result to the backend.

**Nodes involved:** OpenAI Chat Model1, Structured Output Parser4, Analyze Product URL Scrape, Edit Fields, Sanitize JSON Values, Send Website Analysis Data to Server

#### Node: **OpenAI Chat Model1**
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — language model provider for website analysis.
- **Config choices:** model `gpt-4.1-mini`
- **Credentials:** `My Open AI Account`
- **Connections:** feeds **Analyze Product URL Scrape** via `ai_languageModel`.
- **Failure modes / edge cases:**
  - OpenAI auth/quotas.
  - Token limit if markdown is large.

#### Node: **Structured Output Parser4**
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces JSON schema for website analysis.
- **Expected schema:**
  - `website_summary` (exactly 2 sentences per prompt)
  - `target_market` (single paragraph)
- **Connections:** attached to **Analyze Product URL Scrape** via `ai_outputParser`.
- **Failure modes:** model returns invalid JSON → parsing error.

#### Node: **Analyze Product URL Scrape**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — prompts model to analyze landing page.
- **Key inputs / expressions:**
  - `site title: {{$json.title}}`
  - `markdown content: {{$json.markdown}}`
  - `cta: {{$json.description}}`
- **Config choices:**
  - System message instructs strict JSON-only output matching parser schema.
  - `hasOutputParser: true`
- **Outputs:** to **Edit Fields**
- **Failure modes / edge cases:**
  - If `cta` is `"not_applicable"` prompt says disregard “not_available” (mismatch), but still works semantically.
  - If markdown is huge, model may truncate or violate JSON constraints.

#### Node: **Edit Fields**
- **Type / role:** `n8n-nodes-base.set` — prepares a combined “website analysis” payload.
- **Key assignments (interpreted):**
  - `title` ← `Sanitize Results.title`
  - `cta` ← `Sanitize Results.description`
  - `website_summary` ← `$json.output.website_summary` (from agent output)
  - `target_market_analysis` ← `$json.output.target_market`
  - `preview_image`, `favicon` from `Sanitize Results`
  - `searchId` ← `Webhook.body.searchId`
  - `stage` = `website_analysis`
- **Outputs:** to **Sanitize JSON Values**
- **Failure modes:** expression errors if upstream nodes are missing or renamed.

#### Node: **Sanitize JSON Values**
- **Type / role:** `n8n-nodes-base.code` — “ultra-safe” sanitizer to prevent JSON breaking / control chars.
- **Config choices:**
  - Sanitizes: `title`, `cta`, `website_summary`, `target_market_analysis`
  - Normalizes unicode, escapes backslashes and quotes, removes control chars, trims, removes wrapping `[...]`.
- **Outputs:** to **Send Website Analysis Data to Server**
- **Failure modes / edge cases:**
  - Over-escaping may reduce readability (e.g., quotes become `\"`).
  - Converts objects/arrays to JSON strings; if backend expects structured arrays, it will receive strings.

#### Node: **Send Website Analysis Data to Server**
- **Type / role:** `n8n-nodes-base.httpRequest` — sends website analysis to backend.
- **Config choices:**
  - **URL:** `{{ BACKEND_API_URL }}api/webhook/n8n ` (note trailing space in both website/keywords nodes)
  - **Method:** POST
  - **Body:** JSON with:
    - `searchId`
    - `stage`: `"website_analysis"`
    - `websiteData`: array with title/cta/summary/target_market plus preview image + favicon
  - **Header:** `x-api-key: {{N8N_WEBHOOK_API_KEY}}`
- **Outputs:** to **Reddit Posts Keywords Generator**
- **Failure modes / edge cases:**
  - Trailing space in URL can cause request failures depending on HTTP client behavior.
  - Backend may require `Content-Type: application/json` (n8n sets automatically for JSON body).
  - Wrong API key → 401/403.

---

### Block 4 — AI Keyword Generation → Send to Server

**Overview:**  
Uses OpenAI to generate 10 Reddit search keywords from the scraped markdown and sends them to the backend.

**Nodes involved:** OpenAI Chat Model, Structured Output Parser, Reddit Posts Keywords Generator, Send keywords to Server

#### Node: **OpenAI Chat Model**
- **Type / role:** OpenAI chat model for keyword generation.
- **Model:** `gpt-4.1-mini`
- **Connections:** provides `ai_languageModel` to **Reddit Posts Keywords Generator**

#### Node: **Structured Output Parser**
- **Type / role:** structured JSON schema for 10 keyword fields (`keyword1`…`keyword10`).
- **Connections:** provides `ai_outputParser` to **Reddit Posts Keywords Generator**
- **Failure mode:** invalid JSON → parser error.

#### Node: **Reddit Posts Keywords Generator**
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — extracts Reddit-oriented keywords.
- **Key input:** uses Firecrawl markdown directly:
  - `{{ $('Scrape Product URL and get its content').item.json.data.markdown }}`
- **Config choices:**
  - Very strict system message: JSON only, 2–6 word phrases, no product name, problem-focused.
  - `hasOutputParser: true`
- **Outputs:** to **Send keywords to Server**
- **Failure modes / edge cases:**
  - If markdown is empty/blocked, keywords may be generic.
  - Model might still include product name; no enforcement besides prompt.

#### Node: **Send keywords to Server**
- **Type / role:** `n8n-nodes-base.httpRequest` — posts generated keywords to backend.
- **Config choices:**
  - **URL:** `{{ BACKEND_API_URL }}api/webhook/n8n ` (also has trailing space)
  - **Method:** POST, JSON body:
    - `searchId`
    - `stage`: `"keywords_generated"`
    - `keywords`: object keyword1..keyword10 from `$json.output.*`
  - **Header:** `x-api-key`
- **Outputs:** to **Search for Posts (Keyword/Phrase)** (keyword #1 pipeline start)
- **Failure modes:** same as other backend calls + schema mismatch if backend expects array not object.

---

### Block 5 — Reddit Leads for Keyword #1 (Search → Relevance → Send)

**Overview:**  
Searches Reddit for keyword1, normalizes post fields, asks AI if each post is relevant, aggregates results, formats relevant posts into markdown, and sends them to backend as `conversations_partial1`.

**Nodes involved:** Search for Posts (Keyword/Phrase), Clean results, OpenAI Chat Model2, Structured Output Parser1, Posts Relevance AI Agent, Aggregate3, Aggregate, Merge, Parse Qualifying Posts, Send Conversations for keyword #1 to Server

#### Node: **Search for Posts (Keyword/Phrase)**
- **Type / role:** `n8n-nodes-base.reddit` — searches Reddit.
- **Config choices:**
  - Operation: `search`
  - Location: `allReddit`
  - Keyword: `{{ $('Reddit Posts Keywords Generator').item.json.output.keyword1 }}`
  - Limit: 10
  - Sort: `hot`
- **Credentials:** Reddit OAuth2 (`Reddit account`)
- **Outputs:** to **Clean results**
- **Failure modes / edge cases:**
  - Reddit API rate limits, OAuth expiration.
  - Search results may include NSFW/low-signal content (no filtering here).
  - Some posts have empty `selftext`.

#### Node: **Clean results**
- **Type / role:** `n8n-nodes-base.code` — standardizes Reddit post fields.
- **Outputs per item:**
  - `subreddit`, `title`, `body` (from `selftext`), `postUrl` (from `url`), `createdAt` (ISO)
- **Outputs:** to **Posts Relevance AI Agent** and **Aggregate**
- **Failure modes:** if Reddit changes field names.

#### Node: **OpenAI Chat Model2**
- **Type / role:** LLM provider for relevance classification.
- **Model:** `gpt-4.1-mini`
- **Connections:** to **Posts Relevance AI Agent** via `ai_languageModel`

#### Node: **Structured Output Parser1**
- **Type / role:** ensures output `{ "assessment": "relevant/irrelevant" }`
- **Connections:** to **Posts Relevance AI Agent** via `ai_outputParser`

#### Node: **Posts Relevance AI Agent**
- **Type / role:** AI classifier for each Reddit post.
- **Key prompt inputs:**
  - Title/body from `Clean results`
  - References product title and description from Firecrawl metadata
  - Mentions keyword, but **hardcodes keyword1 reference** in system message:
    - `{{ $('Reddit Posts Keywords Generator').item.json.output.keyword1 }}`
- **Outputs:** to **Aggregate3**
- **Failure modes / edge cases:**
  - If `og:description` is not an array, expression `['og:description'][0]` can fail (it’s used in system message).
  - Misclassification; no confidence score.

#### Node: **Aggregate3**
- **Type / role:** `n8n-nodes-base.aggregate` — collects all AI assessments into an array.
- **Aggregates:** `output.assessment`
- **Outputs:** to **Merge** (input index 0)

#### Node: **Aggregate**
- **Type / role:** aggregates the cleaned post fields into arrays.
- **Aggregates:** `subreddit`, `title`, `body`, `postUrl`, `createdAt`
- **Outputs:** to **Merge** (input index 1)

#### Node: **Merge**
- **Type / role:** `n8n-nodes-base.merge` — combines both aggregates (post data arrays + assessment array).
- **Mode:** combineAll
- **Outputs:** to **Parse Qualifying Posts**
- **Failure modes:** if one side produces no items, merge behavior can yield empty output.

#### Node: **Parse Qualifying Posts**
- **Type / role:** `n8n-nodes-base.code` — builds a markdown “conversations” document containing only relevant posts.
- **Logic:** loops through `data.assessment[]`; when `relevant`, appends a markdown section with subreddit/title/date/body/link.
- **Output:** `{ content: <markdown> }` or `no_conversations_found`
- **Outputs:** to **Send Conversations for keyword #1 to Server**

#### Node: **Send Conversations for keyword #1 to Server**
- **Type / role:** `n8n-nodes-base.httpRequest` — sends partial conversations to backend.
- **Config choices:**
  - URL: `{{ BACKEND_API_URL }}api/webhook/n8n` (no trailing space here)
  - Method: POST
  - Content-Type: form-urlencoded
  - Body parameters:
    - `searchId` (from webhook)
    - `stage` = `conversations_partial1`
    - `keyword` = keyword1
    - `passedPosts` = `{{$json.content}}`
  - Header: `x-api-key`
- **Outputs:** to **Search for Posts (Keyword/Phrase)2**
- **Failure modes:**
  - Backend must accept `application/x-www-form-urlencoded`.
  - Large markdown may exceed backend limits.

---

### Block 6 — Reddit Leads for Keyword #2 (Search → Relevance → Send)

**Overview:**  
Same pattern as keyword #1, but searches keyword2 and sends result as `conversations_partial2`.

**Nodes involved:** Search for Posts (Keyword/Phrase)2, Clean results2, OpenAI Chat Model3, Structured Output Parser5, Posts Relevance AI Agent2, Aggregate6, Aggregate5, Merge3, Parse Qualifying Posts 2, Send Conversations for Keyword #2 to Server

Key differences:
- Search keyword: `keyword2`
- Stage: `conversations_partial2`

Notable issue:
- **Posts Relevance AI Agent2 system message still references keyword1** (copy/paste), even though search is for keyword2. That can reduce relevance accuracy.

---

### Block 7 — Keywords #3–#10 Batch Processing (Final Conversations)

**Overview:**  
Creates an array of keywords 3–10, splits them into items, loops over each keyword, searches Reddit and classifies relevance, aggregates relevant posts across iterations, formats into a final markdown payload, sends to backend as `conversations_final`.

**Nodes involved:** get keyword 3 - 10 For Final Processing, Split Out1, Loop Over Items1, Search for Posts (Keyword/Phrase)4, Clean results4, OpenAI Chat Model5, Structured Output Parser7, Posts Relevance AI Agent4, Aggregate10, Aggregate9, Merge5, Parse Qualifying Posts Final, Aggregate11, Sanitize & Parse Final Payload, Send Final Conversations For Keyword #3 to #10 to Server

#### Node: **get keyword 3 - 10 For Final Processing**
- **Type / role:** code node building array of keywords 3..10.
- **Logic:** reads `$('Reddit Posts Keywords Generator').item.json.output`, pushes keyword3..keyword10 into `keywords[]`.
- **Outputs:** `{ keywords: [ ... ] }` to **Split Out1**
- **Failure modes:** if keywords missing/null → shorter list (safe).

#### Node: **Split Out1**
- **Type / role:** `n8n-nodes-base.splitOut` — turns `keywords[]` into individual items.
- **Field:** `keywords`
- **Outputs:** to **Loop Over Items1**

#### Node: **Loop Over Items1**
- **Type / role:** `n8n-nodes-base.splitInBatches` — batch iterator.
- **Config:** defaults (batch size not explicitly set; n8n default is typically 1 unless configured).
- **Connections:**
  - Main output 0 → **Search for Posts (Keyword/Phrase)4**
  - Main output 1 → **Aggregate11** (this is the “done/continue” output pattern)
- **Failure modes / edge cases:**
  - If batch size is not set, behavior might not match intended throughput.
  - Incorrect loop wiring can cause only partial processing; here it relies on downstream node returning to the splitInBatches node (see below).

#### Node: **Search for Posts (Keyword/Phrase)4**
- **Type / role:** Reddit search for the current keyword.
- **Keyword expression:** `{{ $json.keywords }}` (after splitOut, `keywords` is the current string)
- **Outputs:** to **Clean results4**

#### Node: **Clean results4**
- Same standardization as other “Clean results” nodes.
- **Outputs:** to **Posts Relevance AI Agent4** and **Aggregate9**

#### Node: **OpenAI Chat Model5** + **Structured Output Parser7**
- Provide model + `{assessment}` schema to **Posts Relevance AI Agent4**

#### Node: **Posts Relevance AI Agent4**
- **Key difference:** system message references `{{ $('Split Out1').item.json.keywords }}` for keyword.
- **Outputs:** to **Aggregate10**
- **Edge case:** `og:description[0]` again assumed array.

#### Node: **Aggregate10 / Aggregate9 / Merge5**
- Aggregate10: assessments array (`output.assessment`)
- Aggregate9: post fields arrays
- Merge5: combines both aggregates
- Outputs to **Parse Qualifying Posts Final**

#### Node: **Parse Qualifying Posts Final**
- **Type / role:** code node that filters only “relevant” posts into an array `passed_post`.
- **Output:** `{ passed_post: [ {subreddit,title,body,postUrl,assessment,createdAt}, ... ] }`
- **Outputs:** back to **Loop Over Items1** (this is the loop “return” to get next keyword)

#### Node: **Aggregate11**
- **Type / role:** aggregates across loop iterations.
- **Aggregates:** `passed_post` (result becomes an array of arrays; one per keyword iteration)
- **Outputs:** to **Sanitize & Parse Final Payload**

#### Node: **Sanitize & Parse Final Payload**
- **Type / role:** formats final aggregated data into markdown.
- **Important mismatch / risk:**
  - The code expects each keyword group item to include `keyword` on each conversation (`keywordGroup[0]?.keyword`), but **Parse Qualifying Posts Final does not add `keyword`** to each conversation. This means headings will fall back to `Keyword Group X` rather than actual keywords.
  - It also assumes `createdAt` is parseable by `new Date(createdAt)`; when it is `'not_applicable'`, it becomes “Invalid Date”.
- **Output:** `{ conversations_final: <markdown> }`
- **Outputs:** to **Send Final Conversations For Keyword #3 to #10 to Server**

#### Node: **Send Final Conversations For Keyword #3 to #10 to Server**
- **Type / role:** HTTP POST to backend (form-urlencoded).
- **Body parameters:**
  - `searchId`
  - `stage` = `conversations_final`
  - `passedPosts` = `{{ $json.conversations_final }}`
- **Failure modes:** payload size; backend acceptance of large form fields.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | n8n-nodes-base.webhook | Entry point (authenticated POST trigger) | — | Set Environment Variables | ## Process Product URL, Send Website Analysis and Keywords to Server/Frontend |
| Set Environment Variables | n8n-nodes-base.set | Stores backend URL + API key in item JSON | Webhook | Scrape Product URL and get its content | ## Process Product URL, Send Website Analysis and Keywords to Server/Frontend |
| Scrape Product URL and get its content | @mendable/n8n-nodes-firecrawl.firecrawl | Scrape landing page markdown + metadata | Set Environment Variables | Sanitize Results | ## Process Product URL, Send Website Analysis and Keywords to Server/Frontend |
| Sanitize Results | n8n-nodes-base.code | Normalize scrape output into stable fields | Scrape Product URL and get its content | Analyze Product URL Scrape | ## Process Product URL, Send Website Analysis and Keywords to Server/Frontend |
| OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for website analysis | — | Analyze Product URL Scrape (ai_languageModel) | ## Process Product URL, Send Website Analysis and Keywords to Server/Frontend |
| Structured Output Parser4 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce website analysis JSON schema | — | Analyze Product URL Scrape (ai_outputParser) | ## Process Product URL, Send Website Analysis and Keywords to Server/Frontend |
| Analyze Product URL Scrape | @n8n/n8n-nodes-langchain.agent | Generate website summary + target market | Sanitize Results | Edit Fields | ## Process Product URL, Send Website Analysis and Keywords to Server/Frontend |
| Edit Fields | n8n-nodes-base.set | Build unified website analysis payload | Analyze Product URL Scrape | Sanitize JSON Values | ## Process Product URL, Send Website Analysis and Keywords to Server/Frontend |
| Sanitize JSON Values | n8n-nodes-base.code | Escape/clean strings for safe JSON transport | Edit Fields | Send Website Analysis Data to Server | ## Process Product URL, Send Website Analysis and Keywords to Server/Frontend |
| Send Website Analysis Data to Server | n8n-nodes-base.httpRequest | POST website analysis to backend | Sanitize JSON Values | Reddit Posts Keywords Generator | ## Process Product URL, Send Website Analysis and Keywords to Server/Frontend |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for keyword generation | — | Reddit Posts Keywords Generator (ai_languageModel) | ## Process Product URL, Send Website Analysis and Keywords to Server/Frontend |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce 10-keyword JSON schema | — | Reddit Posts Keywords Generator (ai_outputParser) | ## Process Product URL, Send Website Analysis and Keywords to Server/Frontend |
| Reddit Posts Keywords Generator | @n8n/n8n-nodes-langchain.agent | Generate 10 Reddit search keywords | Send Website Analysis Data to Server | Send keywords to Server | ## Process Product URL, Send Website Analysis and Keywords to Server/Frontend |
| Send keywords to Server | n8n-nodes-base.httpRequest | POST keywords to backend | Reddit Posts Keywords Generator | Search for Posts (Keyword/Phrase) | ## Process Product URL, Send Website Analysis and Keywords to Server/Frontend |
| Search for Posts (Keyword/Phrase) | n8n-nodes-base.reddit | Reddit search for keyword1 | Send keywords to Server | Clean results | ## Process and send conversations for Keyword #1 |
| Clean results | n8n-nodes-base.code | Normalize Reddit fields for keyword1 | Search for Posts (Keyword/Phrase) | Aggregate; Posts Relevance AI Agent | ## Process and send conversations for Keyword #1 |
| OpenAI Chat Model2 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for relevance classification (#1) | — | Posts Relevance AI Agent (ai_languageModel) | ## Process and send conversations for Keyword #1 |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce relevance JSON schema (#1) | — | Posts Relevance AI Agent (ai_outputParser) | ## Process and send conversations for Keyword #1 |
| Posts Relevance AI Agent | @n8n/n8n-nodes-langchain.agent | Classify each post relevant/irrelevant (#1) | Clean results | Aggregate3 | ## Process and send conversations for Keyword #1 |
| Aggregate3 | n8n-nodes-base.aggregate | Collect assessments array (#1) | Posts Relevance AI Agent | Merge | ## Process and send conversations for Keyword #1 |
| Aggregate | n8n-nodes-base.aggregate | Collect post fields arrays (#1) | Clean results | Merge | ## Process and send conversations for Keyword #1 |
| Merge | n8n-nodes-base.merge | Combine post arrays + assessment array (#1) | Aggregate3; Aggregate | Parse Qualifying Posts | ## Process and send conversations for Keyword #1 |
| Parse Qualifying Posts | n8n-nodes-base.code | Build markdown of relevant posts (#1) | Merge | Send Conversations for keyword #1 to Server | ## Process and send conversations for Keyword #1 |
| Send Conversations for keyword #1 to Server | n8n-nodes-base.httpRequest | Send partial conversations (#1) | Parse Qualifying Posts | Search for Posts (Keyword/Phrase)2 | ## Process and send conversations for Keyword #1 |
| Search for Posts (Keyword/Phrase)2 | n8n-nodes-base.reddit | Reddit search for keyword2 | Send Conversations for keyword #1 to Server | Clean results2 | ## Process and send conversations for Keyword #2 |
| Clean results2 | n8n-nodes-base.code | Normalize Reddit fields for keyword2 | Search for Posts (Keyword/Phrase)2 | Aggregate5; Posts Relevance AI Agent2 | ## Process and send conversations for Keyword #2 |
| OpenAI Chat Model3 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for relevance classification (#2) | — | Posts Relevance AI Agent2 (ai_languageModel) | ## Process and send conversations for Keyword #2 |
| Structured Output Parser5 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce relevance JSON schema (#2) | — | Posts Relevance AI Agent2 (ai_outputParser) | ## Process and send conversations for Keyword #2 |
| Posts Relevance AI Agent2 | @n8n/n8n-nodes-langchain.agent | Classify each post relevant/irrelevant (#2) | Clean results2 | Aggregate6 | ## Process and send conversations for Keyword #2 |
| Aggregate6 | n8n-nodes-base.aggregate | Collect assessments array (#2) | Posts Relevance AI Agent2 | Merge3 | ## Process and send conversations for Keyword #2 |
| Aggregate5 | n8n-nodes-base.aggregate | Collect post fields arrays (#2) | Clean results2 | Merge3 | ## Process and send conversations for Keyword #2 |
| Merge3 | n8n-nodes-base.merge | Combine post arrays + assessment array (#2) | Aggregate6; Aggregate5 | Parse Qualifying Posts 2 | ## Process and send conversations for Keyword #2 |
| Parse Qualifying Posts 2 | n8n-nodes-base.code | Build markdown of relevant posts (#2) | Merge3 | Send Conversations for Keyword #2 to Server | ## Process and send conversations for Keyword #2 |
| Send Conversations for Keyword #2 to Server | n8n-nodes-base.httpRequest | Send partial conversations (#2) | Parse Qualifying Posts 2 | get keyword 3 - 10 For Final Processing | ## Process and send conversations for Keyword #2 |
| get keyword 3 - 10 For Final Processing | n8n-nodes-base.code | Build keywords[3..10] array | Send Conversations for Keyword #2 to Server | Split Out1 | ## Process and send conversations for Keyword #3 - #10 |
| Split Out1 | n8n-nodes-base.splitOut | Split keywords array into items | get keyword 3 - 10 For Final Processing | Loop Over Items1 | ## Process and send conversations for Keyword #3 - #10 |
| Loop Over Items1 | n8n-nodes-base.splitInBatches | Loop over keywords 3..10 | Split Out1; Parse Qualifying Posts Final | Search for Posts (Keyword/Phrase)4; Aggregate11 | ## Process and send conversations for Keyword #3 - #10 |
| Search for Posts (Keyword/Phrase)4 | n8n-nodes-base.reddit | Reddit search for current keyword (3..10) | Loop Over Items1 | Clean results4 | ## Process and send conversations for Keyword #3 - #10 |
| Clean results4 | n8n-nodes-base.code | Normalize Reddit fields (3..10 loop) | Search for Posts (Keyword/Phrase)4 | Aggregate9; Posts Relevance AI Agent4 | ## Process and send conversations for Keyword #3 - #10 |
| OpenAI Chat Model5 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM for relevance classification (3..10) | — | Posts Relevance AI Agent4 (ai_languageModel) | ## Process and send conversations for Keyword #3 - #10 |
| Structured Output Parser7 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce relevance JSON schema (3..10) | — | Posts Relevance AI Agent4 (ai_outputParser) | ## Process and send conversations for Keyword #3 - #10 |
| Posts Relevance AI Agent4 | @n8n/n8n-nodes-langchain.agent | Classify post relevance (3..10) | Clean results4 | Aggregate10 | ## Process and send conversations for Keyword #3 - #10 |
| Aggregate10 | n8n-nodes-base.aggregate | Collect assessments array (3..10) | Posts Relevance AI Agent4 | Merge5 | ## Process and send conversations for Keyword #3 - #10 |
| Aggregate9 | n8n-nodes-base.aggregate | Collect post fields arrays (3..10) | Clean results4 | Merge5 | ## Process and send conversations for Keyword #3 - #10 |
| Merge5 | n8n-nodes-base.merge | Combine arrays + assessments (3..10) | Aggregate10; Aggregate9 | Parse Qualifying Posts Final | ## Process and send conversations for Keyword #3 - #10 |
| Parse Qualifying Posts Final | n8n-nodes-base.code | Filter relevant posts into `passed_post[]` | Merge5 | Loop Over Items1 | ## Process and send conversations for Keyword #3 - #10 |
| Aggregate11 | n8n-nodes-base.aggregate | Aggregate `passed_post` across loop iterations | Loop Over Items1 | Sanitize & Parse Final Payload | ## Process and send conversations for Keyword #3 - #10 |
| Sanitize & Parse Final Payload | n8n-nodes-base.code | Format final aggregated results as markdown | Aggregate11 | Send Final Conversations For Keyword #3 to #10 to Server | ## Process and send conversations for Keyword #3 - #10 |
| Send Final Conversations For Keyword #3 to #10 to Server | n8n-nodes-base.httpRequest | Send final conversations markdown to backend | Sanitize & Parse Final Payload | — | ## Process and send conversations for Keyword #3 - #10 |
| Sticky Note | n8n-nodes-base.stickyNote | Comment block header | — | — | ## Process Product URL, Send Website Analysis and Keywords to Server/Frontend |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment block header | — | — | ## Process and send conversations for Keyword #2 |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment block header | — | — | ## Process and send conversations for Keyword #1 |
| Sticky Note4 | n8n-nodes-base.stickyNote | Comment block header | — | — | ## Process and send conversations for Keyword #3 - #10 |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment banner | — | — | ## Reddit Leads Finder N8N Automation |
| Sticky Note1 | n8n-nodes-base.stickyNote | Resources + contacts | — | — | ## Resources<br>1. [Youtube Tutorial]()<br>2. [Medium Article Guide]()<br>3. [Open Source Github Repo]()<br><br>## Contacts for Questions and Work<br>**Website**: [Leadly Solutions](https://leadlysolutionns.com)<br>**Email**: joseph@leadlysolutions.com<br>**X/Twitter**: [@juppfy](https://x.com/juppfy) |
| Sticky Note6 | n8n-nodes-base.stickyNote | Setup guide + links | — | — | ![](https://res.cloudinary.com/dd6vlwblr/image/upload/v1769022321/0_lgiwxy.png)<br>## Basic Setup Guide<br>… [keygen.leadlysolutions.com](https://keygen.leadlysolutions.com) … [Guide](https://docs.n8n.io/workflows/sticky-notes/) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named: *Generate Reddit customer leads from a product URL with OpenAI and Firecrawl*.

2. **Add Webhook (POST)**
   - Node: **Webhook**
   - Method: POST
   - Authentication: **Header Auth**
   - Path: set to a unique value (your choice)
   - Create/select credentials: **HTTP Header Auth** (e.g., name `leadsgen`)
     - Configure the expected header/key according to your security scheme (must match client).

3. **Add Set node for environment-like values**
   - Node: **Set Environment Variables** (Set node)
   - Add string fields:
     - `N8N_WEBHOOK_API_KEY` = your shared secret
     - `BACKEND_API_URL` = your backend base URL **ending with `/`**

4. **Add Firecrawl scrape node**
   - Node: **Scrape Product URL and get its content**
   - Operation: Scrape
   - URL expression: `{{ $('Webhook').item.json.body.productUrl }}`
   - Credentials: create **Firecrawl API** credentials and select them
   - (Optional) enable parsers like PDF if needed.

5. **Add Code node to normalize scrape output**
   - Node: **Sanitize Results**
   - Implement logic to output: `markdown`, `title`, `image`, `description`, `favicon` with fallbacks.

6. **Add OpenAI Chat Model for website analysis**
   - Node: **OpenAI Chat Model1**
   - Model: `gpt-4.1-mini` (or your preferred)
   - Credentials: create/select **OpenAI API** credential.

7. **Add Structured Output Parser for website analysis**
   - Node: **Structured Output Parser4**
   - Schema fields:
     - `website_summary`
     - `target_market`

8. **Add LangChain Agent for website analysis**
   - Node: **Analyze Product URL Scrape**
   - Input text uses the sanitized fields (`title`, `markdown`, `description/cta`)
   - System message instructs JSON-only output matching parser schema
   - Connect:
     - **Sanitize Results → Analyze Product URL Scrape (main)**
     - **OpenAI Chat Model1 → Analyze Product URL Scrape (ai_languageModel)**
     - **Structured Output Parser4 → Analyze Product URL Scrape (ai_outputParser)**

9. **Add Set node to build final website analysis payload**
   - Node: **Edit Fields**
   - Map:
     - `title`, `cta` from Sanitize Results
     - `website_summary`, `target_market_analysis` from agent `output`
     - `preview_image`, `favicon`
     - `searchId` from webhook body
     - `stage` = `website_analysis`

10. **Add Code node to sanitize strings**
    - Node: **Sanitize JSON Values**
    - Sanitize: `title`, `cta`, `website_summary`, `target_market_analysis`

11. **Add HTTP Request to send website analysis**
    - Node: **Send Website Analysis Data to Server**
    - Method: POST
    - URL: `{{ $('Set Environment Variables').item.json.BACKEND_API_URL }}api/webhook/n8n`
    - Body: JSON with `searchId`, `stage`, and `websiteData[]`
    - Header: `x-api-key: {{N8N_WEBHOOK_API_KEY}}`

12. **Add OpenAI Chat Model for keyword generation**
    - Node: **OpenAI Chat Model**
    - Model: `gpt-4.1-mini`

13. **Add Structured Output Parser for keywords**
    - Node: **Structured Output Parser**
    - Schema: `keyword1` … `keyword10`

14. **Add LangChain Agent for keyword generation**
    - Node: **Reddit Posts Keywords Generator**
    - Prompt uses scraped markdown (from Firecrawl node output)
    - Connect:
      - **Send Website Analysis Data to Server → Reddit Posts Keywords Generator (main)**
      - **OpenAI Chat Model → Reddit Posts Keywords Generator (ai_languageModel)**
      - **Structured Output Parser → Reddit Posts Keywords Generator (ai_outputParser)**

15. **Add HTTP Request to send keywords**
    - Node: **Send keywords to Server**
    - Method: POST (JSON)
    - URL: `{{ BACKEND_API_URL }}api/webhook/n8n`
    - Header: `x-api-key`
    - Body: `searchId`, `stage=keywords_generated`, and keywords object.

16. **Keyword #1 Reddit pipeline**
    1. **Reddit node**: **Search for Posts (Keyword/Phrase)**
       - Operation: Search, `allReddit`, sort `hot`, limit 10
       - Keyword: `{{ keyword1 }}`
       - Set Reddit OAuth2 credentials.
    2. **Code node**: **Clean results** to map subreddit/title/body/url/createdAt.
    3. **OpenAI model + parser + agent**:
       - Add **OpenAI Chat Model2**
       - Add **Structured Output Parser1** (`assessment`)
       - Add **Posts Relevance AI Agent**
       - Connect Clean results → agent; model/parser to agent.
    4. **Aggregate** fields + assessments:
       - **Aggregate** (post fields arrays)
       - **Aggregate3** (assessment array)
    5. **Merge** aggregates (combineAll)
    6. **Code**: **Parse Qualifying Posts** to markdown
    7. **HTTP**: **Send Conversations for keyword #1 to Server**
       - form-urlencoded; stage `conversations_partial1`; include keyword + markdown.

17. **Keyword #2 Reddit pipeline**
    - Duplicate keyword #1 pipeline and change:
      - Reddit keyword to `keyword2`
      - stage to `conversations_partial2`
      - (Recommended) ensure the AI agent’s system message references keyword2, not keyword1.

18. **Keywords #3–#10 pipeline**
    1. **Code**: **get keyword 3 - 10 For Final Processing** → returns `{keywords:[...]}`
    2. **Split Out**: **Split Out1** on field `keywords`
    3. **SplitInBatches**: **Loop Over Items1**
    4. Inside loop:
       - Reddit search node (**Search for Posts (Keyword/Phrase)4**) uses `{{$json.keywords}}`
       - Clean node (**Clean results4**)
       - AI relevance agent (**Posts Relevance AI Agent4**) with model (**OpenAI Chat Model5**) and parser (**Structured Output Parser7**)
       - Aggregate post fields (**Aggregate9**) + assessments (**Aggregate10**)
       - Merge (**Merge5**)
       - Filter relevant posts to array (**Parse Qualifying Posts Final**)
       - Connect **Parse Qualifying Posts Final → Loop Over Items1** to continue loop.
    5. After loop completion:
       - **Aggregate11** to collect `passed_post` across iterations
       - **Sanitize & Parse Final Payload** to format final markdown
       - **Send Final Conversations For Keyword #3 to #10 to Server** with stage `conversations_final`

19. **Add Sticky Notes (optional but recommended)**
    - Add the provided comment headings to group sections visually (as in the original).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Disclaimer (provided) |
| **Website**: [Leadly Solutions](https://leadlysolutionns.com) | Contacts |
| **Email**: joseph@leadlysolutions.com | Contacts |
| **X/Twitter**: [@juppfy](https://x.com/juppfy) | Contacts |
| Setup: get secret key from [keygen.leadlysolutions.com](https://keygen.leadlysolutions.com). Ensure backend URL ends with `/`. If localhost, use ngrok; if Railway, generate a public domain. | Sticky note “Basic Setup Guide” |
| “Guide” link included in sticky note: [Guide](https://docs.n8n.io/workflows/sticky-notes/) | Referenced resource |
| Image reference in setup note: `https://res.cloudinary.com/dd6vlwblr/image/upload/v1769022321/0_lgiwxy.png` | Sticky note asset |
| Resources placeholders: [Youtube Tutorial](), [Medium Article Guide](), [Open Source Github Repo]() | Sticky note “Resources” (links not provided) |

