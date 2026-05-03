Find qualified leads in Reddit posts with Firecrawl and GPT-4.1-mini

https://n8nworkflows.xyz/workflows/find-qualified-leads-in-reddit-posts-with-firecrawl-and-gpt-4-1-mini-12882


# Find qualified leads in Reddit posts with Firecrawl and GPT-4.1-mini

## 1. Workflow Overview

**Purpose:** This workflow takes a product URL, scrapes the landing page with Firecrawl, uses GPT‑4.1‑mini to generate **10 Reddit-search keywords**, searches Reddit for posts matching each keyword, and then uses GPT‑4.1‑mini again to **classify each post as relevant/irrelevant**. Relevant posts are collected as “qualified leads” (threads where you can engage and promote your product).

**Target use cases:**
- Finding early adopters and potential customers discussing a problem your product solves
- Building a list of high-intent Reddit conversations for outreach/commenting
- Automating lead discovery across multiple problem-focused keywords

### 1.1 Input Reception (Product URL)
User enters the product URL via an n8n Form trigger.

### 1.2 Landing Page Scrape (Firecrawl)
The workflow scrapes the product page and retrieves markdown + metadata (title/description), used later for keyword generation and relevance checking.

### 1.3 Keyword Generation (GPT‑4.1‑mini + Structured JSON)
An AI Agent analyzes the scraped markdown and outputs a strict JSON object containing 10 keyword phrases.

### 1.4 Keyword Iteration (Split + Batch Loop)
The 10 keywords are split into items and iterated via a batch loop.

### 1.5 Reddit Search per Keyword
For each keyword item, the Reddit node searches “All Reddit” and returns up to 10 hot results.

### 1.6 Post Sanitization + AI Relevance Classification
Returned Reddit fields are normalized, then each post is sent to an AI Agent that returns `{"assessment":"relevant/irrelevant"}`.

### 1.7 Aggregation, Merge, and Filtering to Qualified Posts
Assessments and post fields are aggregated into arrays, merged, then filtered in code to return only relevant posts. The loop continues for the next keyword.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Product URL)
**Overview:** Collects a single required URL from the user to start the workflow.

**Nodes involved:**
- **Enter Product URL** (Form Trigger)

#### Node: Enter Product URL
- **Type / role:** `n8n-nodes-base.formTrigger` — entry point that creates a form and starts execution when submitted.
- **Configuration choices:**
  - Form title: “Domain URL”
  - One required field: `URL`
  - Form description: “Enter You Product URL”
- **Key variables / outputs:**
  - Produces `{{$json.URL}}` which is used by Firecrawl.
- **Connections:**
  - Output → **Scrape Product URL and get its content**
- **Failure modes / edge cases:**
  - Empty/invalid URL (user input). Consider adding validation or a “Set/IF” guard node.
  - URL pointing to content blocked by bot protections may later fail in Firecrawl.

---

### Block 2 — Landing Page Scrape (Firecrawl)
**Overview:** Scrapes the provided URL and returns page content (markdown) and metadata for downstream AI prompts.

**Nodes involved:**
- **Scrape Product URL and get its content** (Firecrawl)

#### Node: Scrape Product URL and get its content
- **Type / role:** `@mendable/n8n-nodes-firecrawl.firecrawl` — web scraping to extract content and metadata.
- **Configuration choices (interpreted):**
  - Operation: **scrape**
  - URL: `={{ $json.URL }}`
  - Parsers include `pdf` (enabled)
  - Output format settings are left mostly default (formats array present but not explicitly set in a meaningful way)
- **Key variables / outputs:**
  - Downstream nodes reference:
    - `{{$json.data.markdown}}` (for keyword generation)
    - `{{$json.data.metadata.ogTitle}}`
    - `{{$json.data.metadata['og:description'][0]}}` (note: assumes array)
- **Connections:**
  - Output → **Reddit Posts Keywords Generator1**
- **Credentials:**
  - Firecrawl API credential (“Firecrawl 20k”)
- **Failure modes / edge cases:**
  - Firecrawl auth error / quota exceeded.
  - Scrape returns missing `data.markdown` or missing OG metadata.
  - `og:description` may be a string (not array) depending on Firecrawl output; current expression uses `[0]` and can error or return undefined.

---

### Block 3 — Keyword Generation (GPT‑4.1‑mini + Structured Output)
**Overview:** Uses an AI Agent to derive 10 problem-focused Reddit keywords from the scraped landing page markdown, enforced via a structured JSON output parser.

**Nodes involved:**
- **OpenAI Chat Model1**
- **Structured Output Parser2**
- **Reddit Posts Keywords Generator1**
- **Split Out**

#### Node: OpenAI Chat Model1
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the chat model for the agent.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
- **Connections:**
  - AI Language Model → **Reddit Posts Keywords Generator1**
- **Credentials:**
  - OpenAI API credential (“My Open AI Account”)
- **Failure modes / edge cases:**
  - Auth / billing / rate limit errors.
  - Model availability changes.

#### Node: Structured Output Parser2
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces JSON schema-like output.
- **Configuration choices:**
  - Example schema includes keys `keyword1` … `keyword10`
- **Connections:**
  - AI Output Parser → **Reddit Posts Keywords Generator1**
- **Failure modes / edge cases:**
  - If the agent returns non-JSON or missing keys, parsing fails.

#### Node: Reddit Posts Keywords Generator1
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — generates the 10 keywords.
- **Configuration choices:**
  - Text input: `Product landing Page Description: {{ $json.data.markdown }}`
  - System message: detailed instructions to output **only JSON** with 10 keyword fields, avoid product name, keep 2–6 words, etc.
  - Output parser enabled (Structured Output Parser2)
- **Key expressions / variables:**
  - Uses the scrape node output `{{$json.data.markdown}}`
- **Connections:**
  - Main output → **Split Out**
  - AI model: **OpenAI Chat Model1**
  - Output parser: **Structured Output Parser2**
- **Failure modes / edge cases:**
  - Large markdown may exceed token limits.
  - If landing page content is thin, keywords may be generic/low quality.

#### Node: Split Out
- **Type / role:** `n8n-nodes-base.splitOut` — converts an array/object field into separate items.
- **Configuration choices:**
  - Field to split out: `output`
- **Important behavior note:**
  - The keyword agent’s structured output is expected to appear under a field named `output`. This node assumes `output` is splittable.
- **Connections:**
  - Output → **Loop Over Items**
- **Failure modes / edge cases:**
  - If the agent output isn’t stored under `output`, or is not split-compatible, this node yields nothing or errors.
  - If `output` is an object (keyword1..keyword10) rather than an array, Split Out behavior depends on n8n’s handling (it may split into key/value items). Downstream nodes expect `{{$json.output}}` to be a keyword string.

---

### Block 4 — Keyword Iteration (Batch Loop)
**Overview:** Iterates through each keyword item. For each iteration it triggers Reddit search, collects results, and loops until all keywords are processed.

**Nodes involved:**
- **Loop Over Items** (SplitInBatches)

#### Node: Loop Over Items
- **Type / role:** `n8n-nodes-base.splitInBatches` — implements looping/batching over incoming items.
- **Configuration choices:**
  - Batch options left default (batch size defaults are determined by node defaults; commonly 1 unless changed).
- **Connections:**
  - Output (current batch item) → **Search for Posts per Keyword/Phrase**
  - “No items left / next” output → **Aggregate All Qualified Conversations**
  - Receives loop-back input from **Parse Qualified Posts Data**
- **Failure modes / edge cases:**
  - Misconfigured batch size can overwhelm Reddit/OpenAI or slow execution.
  - If loop-back is incorrect, it may not iterate correctly.

---

### Block 5 — Reddit Search per Keyword
**Overview:** Searches Reddit for each keyword phrase and returns up to 10 posts, sorted by “hot”.

**Nodes involved:**
- **Search for Posts per Keyword/Phrase** (Reddit)

#### Node: Search for Posts per Keyword/Phrase
- **Type / role:** `n8n-nodes-base.reddit` — queries Reddit search API.
- **Configuration choices:**
  - Operation: `search`
  - Location: `allReddit`
  - Keyword: `={{ $json.output }}`
  - Limit: 10
  - Sort: hot
- **Connections:**
  - Output → **Sanitize Results**
- **Credentials:**
  - Reddit OAuth2 credential (“Reddit account”)
- **Failure modes / edge cases:**
  - Reddit OAuth token expiration / permission issues.
  - Rate limiting from Reddit.
  - Keyword item not a string (if Split Out produced objects/key/value pairs).

---

### Block 6 — Post Sanitization + Relevance AI Classification
**Overview:** Normalizes Reddit fields and classifies each post as relevant/irrelevant using GPT‑4.1‑mini and a structured output parser.

**Nodes involved:**
- **Sanitize Results** (Code)
- **OpenAI Chat Model**
- **Structured Output Parser**
- **Posts Relevance Processing AI Agent**
- **Aggregate**
- **Aggregate1**

#### Node: Sanitize Results
- **Type / role:** `n8n-nodes-base.code` — maps Reddit API response fields to a simplified schema.
- **Configuration choices:**
  - Extracts: `subreddit`, `title`, `selftext` → `body`, `url` → `postUrl`
- **Code behavior (interpreted):**
  - For each incoming Reddit post item, returns:
    - `subreddit: item.json.subreddit`
    - `title: item.json.title`
    - `body: item.json.selftext`
    - `postUrl: item.json.url`
- **Connections:**
  - Output → **Aggregate1**
  - Output → **Posts Relevance Processing AI Agent**
- **Failure modes / edge cases:**
  - Some Reddit results can have missing `selftext` (e.g., link posts); `body` becomes `undefined`.
  - If Reddit node returns a different shape, mappings break.

#### Node: OpenAI Chat Model
- **Type / role:** OpenAI chat model provider for relevance agent.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
- **Connections:**
  - AI Language Model → **Posts Relevance Processing AI Agent**
- **Credentials:** OpenAI (“My Open AI Account”)
- **Failure modes / edge cases:** auth/rate limits/model changes.

#### Node: Structured Output Parser
- **Type / role:** Enforces JSON output for relevance classification.
- **Configuration choices:**
  - Example schema: `{"assessment":"relevant/irrelevant"}`
- **Connections:**
  - AI Output Parser → **Posts Relevance Processing AI Agent**
- **Failure modes / edge cases:** agent returns invalid JSON, additional text, or different field name.

#### Node: Posts Relevance Processing AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — classifies each post.
- **Configuration choices:**
  - Text passed to the agent:
    - `title: {{ $('Sanitize Results').item.json.title }}`
    - `Body: {{ $('Sanitize Results').item.json.body }}`
  - System message includes:
    - The keyword reference: `{{ $('Reddit Posts Keywords Generator1').item.json.output.keyword1 }}`
      - **Important:** This is hard-coded to `keyword1`, even though the workflow loops over many keywords.
    - Product title & description from Firecrawl OG metadata
    - Instruction to output JSON with `assessment`
  - Output parser enabled (Structured Output Parser)
- **Connections:**
  - Main output → **Aggregate**
  - AI model: **OpenAI Chat Model**
  - Output parser: **Structured Output Parser**
- **Failure modes / edge cases:**
  - **Keyword mismatch bug:** It references only `keyword1` in the system message, so later iterations may classify relevance with the wrong keyword context.
  - Missing OG metadata fields can break expressions (especially `og:description[0]`).
  - Empty `body` can reduce classification accuracy.

#### Node: Aggregate
- **Type / role:** `n8n-nodes-base.aggregate` — collects assessments into an array aligned with posts.
- **Configuration choices:**
  - Aggregates field: `output.assessment`
- **Connections:**
  - Output → **Merge1** (input 0)
- **Failure modes / edge cases:**
  - If the relevance agent output structure differs, `output.assessment` path may not exist.

#### Node: Aggregate1
- **Type / role:** Aggregates post fields into arrays.
- **Configuration choices:**
  - Aggregates: `subreddit`, `title`, `body`, `postUrl`
- **Connections:**
  - Output → **Merge1** (input 1)
- **Failure modes / edge cases:**
  - If any field missing, aggregated arrays may contain null/undefined entries.

---

### Block 7 — Merge, Filter Qualified Posts, Loop Continuation, Final Aggregation
**Overview:** Merges the aggregated assessments and post arrays, filters to only relevant posts, loops back for the next keyword, and finally aggregates results after the loop ends.

**Nodes involved:**
- **Merge1**
- **Parse Qualified Posts Data**
- **Aggregate All Qualified Conversations**

#### Node: Merge1
- **Type / role:** `n8n-nodes-base.merge` — combines data streams.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `combineAll`
  - Effectively produces one item containing both aggregates (assessments + fields).
- **Connections:**
  - Output → **Parse Qualified Posts Data**
- **Failure modes / edge cases:**
  - If either input is empty (e.g., AI failed or Reddit returned no posts), merge behavior may produce incomplete data.

#### Node: Parse Qualified Posts Data
- **Type / role:** `n8n-nodes-base.code` — filters only relevant posts and prepares list for collection.
- **Configuration choices / behavior:**
  - Reads first merged item: `const data = $input.first().json;`
  - Expects arrays: `data.assessment`, `data.subreddit`, `data.title`, `data.body`, `data.postUrl`
  - Builds `passed_post[]` where `assessment === "relevant"`
  - Returns `{ passed_post }`
- **Connections:**
  - Output → **Loop Over Items** (loop-back to continue with next keyword)
- **Failure modes / edge cases:**
  - If `data.assessment` is not an array (or named differently), the loop throws.
  - Array length mismatches (e.g., assessments array shorter than titles) can misalign posts.
  - Case sensitivity: only `"relevant"` passes; `"Relevant"` would be dropped.

#### Node: Aggregate All Qualified Conversations
- **Type / role:** `n8n-nodes-base.aggregate` — final collector after looping completes.
- **Configuration choices:**
  - Aggregate mode: `aggregateAllItemData` (collects all items that reach it)
- **Connections:**
  - Receives from **Loop Over Items** when no items left.
  - No downstream node is connected (this is the workflow end).
- **Failure modes / edge cases:**
  - Depending on loop behavior, this may aggregate intermediate loop items; ensure the loop emits only desired records.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Enter Product URL | Form Trigger | Collect product URL (entry point) | — | Scrape Product URL and get its content | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Scrape Product URL and get its content | Firecrawl | Scrape landing page markdown + metadata | Enter Product URL | Reddit Posts Keywords Generator1 | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| OpenAI Chat Model1 | OpenAI Chat Model (LangChain) | LLM provider for keyword agent | — | Reddit Posts Keywords Generator1 (ai_languageModel) | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Structured Output Parser2 | Structured Output Parser | Enforce JSON for 10 keywords | — | Reddit Posts Keywords Generator1 (ai_outputParser) | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Reddit Posts Keywords Generator1 | LangChain Agent | Generate 10 Reddit search keywords from markdown | Scrape Product URL and get its content | Split Out | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Split Out | Split Out | Split keyword output into items | Reddit Posts Keywords Generator1 | Loop Over Items | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Loop Over Items | Split In Batches | Iterate over keyword items (loop) | Split Out; Parse Qualified Posts Data | Search for Posts per Keyword/Phrase; Aggregate All Qualified Conversations | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Search for Posts per Keyword/Phrase | Reddit | Search Reddit posts per keyword | Loop Over Items | Sanitize Results | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Sanitize Results | Code | Normalize Reddit results fields | Search for Posts per Keyword/Phrase | Aggregate1; Posts Relevance Processing AI Agent | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM provider for relevance agent | — | Posts Relevance Processing AI Agent (ai_languageModel) | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Structured Output Parser | Structured Output Parser | Enforce JSON for relevance assessment | — | Posts Relevance Processing AI Agent (ai_outputParser) | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Posts Relevance Processing AI Agent | LangChain Agent | Classify each post relevant/irrelevant | Sanitize Results | Aggregate | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Aggregate | Aggregate | Collect assessments into an array | Posts Relevance Processing AI Agent | Merge1 | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Aggregate1 | Aggregate | Collect post fields into arrays | Sanitize Results | Merge1 | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Merge1 | Merge | Combine post arrays + assessment array | Aggregate; Aggregate1 | Parse Qualified Posts Data | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Parse Qualified Posts Data | Code | Filter to only relevant posts and prepare output | Merge1 | Loop Over Items | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Aggregate All Qualified Conversations | Aggregate | Final aggregation of all qualified results | Loop Over Items | — | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Sticky Note1 | Sticky Note | Comment block label | — | — | ## Find Your First Reddit Users (Loop Over All 10 keywords) |
| Sticky Note6 | Sticky Note | Setup guidance + image | — | — | ![](https://res.cloudinary.com/dd6vlwblr/image/upload/v1769022321/1_cps1u9.png)\n## Basic Setup Guide\nYou can follow the n8n credential setup docs for reddit, openai to set them up. Then the workflow ends with an aggregate node, you can connect your desired data collection (google sheets, airtable) or notification (telegram, email) node\n\nYou can check the Medium article in the resources card below, or watch the Youtube tutorial to get more details. |
| Sticky Note | Sticky Note | Resources + contacts | — | — | ## Resources\n1. [Youtube Tutorial](https://bit.ly/youtubetutorialredditworkflow)\n2. [Medium Article Guide](https://bit.ly/mediumarticleredditworkflow)\n3. [Frontend Integrated Version](https://bit.ly/frontendversionredditworkflow)\n\n## Contacts for Questions and Work\n**Website**: [Leadly Solutions](https://leadlysolutionns.com)\n**Email**: joseph@leadlysolutions.com\n**X/Twitter**: [@juppfy](https://x.com/juppfy) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add “Enter Product URL”**
   - Node: **Form Trigger**
   - Form title: `Domain URL`
   - Description: `Enter You Product URL`
   - Add one required field: Label `URL`
3. **Add “Scrape Product URL and get its content”**
   - Node: **Firecrawl** (community node `@mendable/n8n-nodes-firecrawl.firecrawl`)
   - Operation: `scrape`
   - URL: `={{ $json.URL }}`
   - Credentials: configure a **Firecrawl API** credential and select it.
4. **Add “OpenAI Chat Model1”**
   - Node: **OpenAI Chat Model (LangChain)**
   - Model: `gpt-4.1-mini`
   - Credentials: configure **OpenAI API** credential and select it.
5. **Add “Structured Output Parser2”**
   - Node: **Structured Output Parser (LangChain)**
   - Set schema example with `keyword1` … `keyword10` (string values).
6. **Add “Reddit Posts Keywords Generator1”**
   - Node: **AI Agent (LangChain)**
   - Prompt type: “define”
   - Text: `=Product landing Page Description:  {{ $json.data.markdown }}`
   - System message: include instructions to output only JSON with 10 keyword fields (as in the workflow).
   - Enable output parser and connect **Structured Output Parser2** to its `ai_outputParser`.
   - Connect **OpenAI Chat Model1** to its `ai_languageModel`.
7. **Connect**: Form Trigger → Firecrawl → Keyword Agent.
8. **Add “Split Out”**
   - Node: **Split Out**
   - Field to split out: `output`
   - Connect Keyword Agent → Split Out
9. **Add “Loop Over Items”**
   - Node: **Split In Batches**
   - Keep defaults (or set Batch Size = 1 for safer API usage)
   - Connect Split Out → Loop Over Items
10. **Add “Search for Posts per Keyword/Phrase”**
   - Node: **Reddit**
   - Operation: `search`
   - Location: `allReddit`
   - Keyword: `={{ $json.output }}`
   - Limit: `10`
   - Additional field: Sort = `hot`
   - Credentials: configure **Reddit OAuth2** and select it.
   - Connect Loop Over Items → Reddit Search
11. **Add “Sanitize Results”**
   - Node: **Code**
   - Paste logic to map to:
     - `subreddit: item.json.subreddit`
     - `title: item.json.title`
     - `body: item.json.selftext`
     - `postUrl: item.json.url`
   - Connect Reddit Search → Sanitize Results
12. **Add “OpenAI Chat Model”**
   - Node: **OpenAI Chat Model (LangChain)**
   - Model: `gpt-4.1-mini`
   - Credentials: same OpenAI credential as before
13. **Add “Structured Output Parser”**
   - Node: **Structured Output Parser (LangChain)**
   - Schema example: `{"assessment":"relevant/irrelevant"}`
14. **Add “Posts Relevance Processing AI Agent”**
   - Node: **AI Agent (LangChain)**
   - Text:  
     `=title: {{ $('Sanitize Results').item.json.title }}\nBody: {{ $('Sanitize Results').item.json.body }}`
   - System message: instruct to assess relevance to your product using scraped metadata; output only JSON `{ "assessment": "relevant/irrelevant" }`
   - Connect:
     - **OpenAI Chat Model** → agent `ai_languageModel`
     - **Structured Output Parser** → agent `ai_outputParser`
   - Connect Sanitize Results → Relevance Agent
15. **Add “Aggregate”**
   - Node: **Aggregate**
   - Aggregate field: `output.assessment`
   - Connect Relevance Agent → Aggregate
16. **Add “Aggregate1”**
   - Node: **Aggregate**
   - Aggregate fields: `subreddit`, `title`, `body`, `postUrl`
   - Connect Sanitize Results → Aggregate1
17. **Add “Merge1”**
   - Node: **Merge**
   - Mode: `combine`
   - Combine by: `combineAll`
   - Connect Aggregate → Merge1 (Input 1)
   - Connect Aggregate1 → Merge1 (Input 2)
18. **Add “Parse Qualified Posts Data”**
   - Node: **Code**
   - Implement logic to:
     - iterate over `assessment[]`
     - keep indices where `"relevant"`
     - output `{ passed_post: [...] }`
   - Connect Merge1 → Parse Qualified Posts Data
19. **Loop back**
   - Connect Parse Qualified Posts Data → Loop Over Items (so the loop proceeds to next keyword)
20. **Add final collector “Aggregate All Qualified Conversations”**
   - Node: **Aggregate**
   - Mode: `aggregateAllItemData`
   - Connect Loop Over Items “done/no more items” output → Aggregate All Qualified Conversations
21. (Optional) **Add a destination**
   - Connect the final aggregate node to Google Sheets/Airtable/Notion/Email/Telegram to store or notify with `passed_post`.

**Credential setup requirements:**
- **OpenAI API**: set API key, ensure access to `gpt-4.1-mini`.
- **Reddit OAuth2**: create Reddit app, set redirect URL to n8n callback, grant required scopes for search.
- **Firecrawl API**: set API key; confirm plan limits.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Basic setup note: configure n8n credentials (Reddit, OpenAI), then connect the final aggregate node to your preferred storage/notification (Google Sheets, Airtable, Telegram, email). Includes image. | https://res.cloudinary.com/dd6vlwblr/image/upload/v1769022321/1_cps1u9.png |
| Youtube Tutorial | https://bit.ly/youtubetutorialredditworkflow |
| Medium Article Guide | https://bit.ly/mediumarticleredditworkflow |
| Frontend Integrated Version | https://bit.ly/frontendversionredditworkflow |
| Leadly Solutions website | https://leadlysolutionns.com |
| Contact email | joseph@leadlysolutions.com |
| X/Twitter @juppfy | https://x.com/juppfy |