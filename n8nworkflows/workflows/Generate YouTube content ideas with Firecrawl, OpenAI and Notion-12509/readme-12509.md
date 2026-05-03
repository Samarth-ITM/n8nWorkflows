Generate YouTube content ideas with Firecrawl, OpenAI and Notion

https://n8nworkflows.xyz/workflows/generate-youtube-content-ideas-with-firecrawl--openai-and-notion-12509


# Generate YouTube content ideas with Firecrawl, OpenAI and Notion

## 1. Workflow Overview

**Purpose:** This n8n workflow performs lightweight YouTube market research for a given niche keyword, scrapes competitor video pages (description + top comments) via **Firecrawl**, uses **OpenAI (LangChain Agent)** to generate **strategy + 5 content ideas** in a strict JSON structure, then saves both competitor insights and generated ideas into **Notion** databases.

**Target use cases:**
- Content strategy teams needing data-backed YouTube ideas for a niche
- Agencies analyzing competitor comment signals to derive audience pains and topic gaps
- Building a repeatable monthly research pipeline into Notion

### 1.1 Input & Scheduling
Monthly trigger + manual configuration of niche keyword and result limit.

### 1.2 Fetch YouTube Search Results (Firecrawl) + Parse/Filter
Scrape the YouTube search results page for the keyword, parse the resulting markdown to extract video entries, and limit how many videos to process.

### 1.3 Per-Video Deep Scrape + Comment Extraction
For each selected video, scrape the video page markdown and extract description + top 20 comments (ranked by likes).

### 1.4 AI Strategy Generation (OpenAI Agent + Structured Output)
Feed extracted content into an AI strategist agent that outputs a strict JSON object containing audience insights and 5 content ideas.

### 1.5 Save to Notion (Competitor record + Ideas records)
Store competitor insights into one Notion database and store each generated content idea into another Notion database.

---

## 2. Block-by-Block Analysis

### Block 1 — Input & Scheduling
**Overview:** Starts the workflow monthly and defines the niche keyword + how many search results to process.

**Nodes involved:**
- Monthly Trigger
- Set Niche

#### Node: Monthly Trigger
- **Type / role:** `Schedule Trigger` — workflow entry point (time-based).
- **Configuration (interpreted):** Runs every month (interval rule: months).
- **Outputs to:** Set Niche
- **Edge cases/failures:** Timezone considerations depend on n8n instance settings; missed executions if instance is down.

#### Node: Set Niche
- **Type / role:** `Set` — defines parameters used downstream.
- **Key fields set:**
  - `query` (string): `"<YOUR_NICHE_OR_KEYWORD>"`
  - `limit` (number): `10`
- **Expressions used downstream:**
  - Search query formatting in the Firecrawl request
  - `limit` used by Limit Search Results and Parse Video List
- **Outputs to:** Search YouTube Videos
- **Edge cases/failures:** Leaving `<YOUR_NICHE_OR_KEYWORD>` unchanged yields irrelevant/empty results.

---

### Block 2 — Step 1: Fetch & Filter Videos
**Overview:** Scrapes YouTube search results via Firecrawl, validates response success, parses the markdown into structured video items, and limits how many are processed.

**Nodes involved:**
- Search YouTube Videos
- Check Search Status
- Parse Video List
- Split Video Items
- Limit Search Results

#### Node: Search YouTube Videos
- **Type / role:** `HTTP Request` — calls Firecrawl scrape endpoint for a YouTube search URL.
- **Request:**
  - **POST** `https://api.firecrawl.dev/v2/scrape`
  - Body parameter `url` = `https://www.youtube.com/results?search_query={{ $json.query.split(' ').join('+') }}`
- **Auth:** `genericCredentialType` → `httpHeaderAuth` (Firecrawl API key in headers).
- **Outputs to:** Check Search Status
- **Edge cases/failures:**
  - Firecrawl auth failure (401/403)
  - Firecrawl rate limits / quota exhaustion
  - YouTube page layout changes causing markdown format shifts (affects parsing later)

#### Node: Check Search Status
- **Type / role:** `IF` — validates Firecrawl response.
- **Conditions:**
  - `$json.success === true`
  - `$json.data.metadata.statusCode === 200`
- **Outputs to:** Parse Video List (only the “true” path is wired)
- **Edge cases/failures:**
  - If condition fails, workflow stops silently (no “false” branch handling is connected).
  - `data.metadata.statusCode` missing → expression resolves to null and condition fails.

#### Node: Parse Video List
- **Type / role:** `Code` — converts Firecrawl markdown into a ranked list of videos.
- **Key logic (interpreted):**
  - Reads `limit` from `$('Set Niche').first().json.limit`
  - Extracts: title, watch URL, channel name, channel handle, views, uploaded time-ago
  - Derives numeric scoring fields for sorting:
    - `_viewsNumeric` from views like `722K`
    - `_subscriberNumeric` from “K/M/B subscribers” found in markdown
    - `_daysAgo` from “X day/week/month/year ago”
  - Sort priority:
    1) Newer videos first (smaller `_daysAgo`)
    2) Higher views
    3) Higher subscribers
  - Returns `{ videos: [...] }` limited to top N.
- **Outputs to:** Split Video Items
- **Edge cases/failures:**
  - If `inputItem.data.markdown` missing → returns `{ error: "Data Not Found." }` (downstream split will fail because `videos` missing).
  - Parsing depends heavily on YouTube/Firecrawl markdown patterns.
  - Bug: `b` multiplier uses `1+1234567890` (should likely be `1_000_000_000`), which will distort subscriber/view ranking when “B” appears.

#### Node: Split Video Items
- **Type / role:** `Split Out` — turns `videos[]` array into one item per video.
- **Field:** `videos`
- **Outputs to:** Limit Search Results
- **Edge cases/failures:** If `videos` absent/not an array (e.g., parse error), node errors.

#### Node: Limit Search Results
- **Type / role:** `Limit` — caps how many video items proceed.
- **Max items:** `={{ $('Set Niche').item.json.limit }}`
- **Outputs to:** Loop Over Items
- **Edge cases/failures:** If `limit` is non-numeric/null → may behave unexpectedly.

---

### Block 3 — Step 2: Scrape & Extract Insights (Per Video)
**Overview:** Iterates through each selected video, scrapes the video page with Firecrawl, and extracts description + top comments.

**Nodes involved:**
- Loop Over Items
- Scrape Video Details
- Extract Audience Comments

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — provides loop/batching behavior.
- **Configuration:** defaults (batch size not explicitly set).
- **Connections:**
  - **Output 1 (main index 1)** → Scrape Video Details (this is the “batch items” output)
  - **Output 0** unused (typically “done”)
- **Outputs to:** Scrape Video Details
- **Edge cases/failures:**
  - Without explicit batch size, behavior depends on n8n defaults.
  - If you expect a “continue” connection back into this node, it is not present; looping relies on SplitInBatches internal behavior and downstream node execution completion (commonly this still works, but it’s fragile to edits).

#### Node: Scrape Video Details
- **Type / role:** `HTTP Request` — Firecrawl scrape of each video URL.
- **Request:**
  - **POST** `https://api.firecrawl.dev/v2/scrape`
  - Body `url` = `={{ $json.url }}`
- **Auth:** same Firecrawl header auth.
- **Outputs to:** Extract Audience Comments
- **Edge cases/failures:** Same as Firecrawl issues above; also YouTube pages can be restricted, causing incomplete markdown.

#### Node: Extract Audience Comments
- **Type / role:** `Code` — parses Firecrawl markdown to extract:
  - `channelName`
  - `views`
  - `totalLikes`
  - `description` (trimmed to 500 chars)
  - `url` (normalized watch URL)
  - `top20Comments` sorted by like count
  - `totalCommentsExtracted`
- **Important parsing details:**
  - Uses regex patterns tailored to Firecrawl’s markdown representation of YouTube.
  - Comments are extracted via a regex that looks for an avatar image markdown `![](...)` + comment text + “Like … Dislike”.
- **Outputs to:** Identify Strategic Gaps
- **Edge cases/failures:**
  - If markdown missing → outputs `{ error: "Markdown data not found in input." }` which will break the AI prompt expectations.
  - Likes parsing uses `parseInt` and simplistic K/M handling; “1.2K” becomes `1` not `1200` (lossy).
  - Regex sensitivity: slight formatting changes may yield zero comments.

---

### Block 4 — Step 3: AI Strategy Generation (LangChain)
**Overview:** Uses an OpenAI chat model via a LangChain Agent to produce a strict JSON strategy object, validated by a structured output parser.

**Nodes involved:**
- OpenAI Chat Model
- Structured Output Parser
- Identify Strategic Gaps

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` — provides the language model to the agent.
- **Model:** `gpt-4.1`
- **Credentials:** OpenAI API credential “Sumopod AI”
- **Connected to:** Identify Strategic Gaps via `ai_languageModel`
- **Edge cases/failures:** API quota, model availability, latency/timeouts, or policy refusals (less likely with this prompt, but possible if input contains unexpected content).

#### Node: Structured Output Parser
- **Type / role:** `outputParserStructured` — enforces the agent output structure.
- **Schema:** Provided as a JSON schema example containing:
  - `analysis` (audienceProfile, keyInsights, topCommentAnalysis)
  - `contentIdeas[]` (with id/title/brief/angle/why/format/keyPoints/engagementPotential)
  - `strategy` (overallTheme/publishingOrder/seriesOpportunity)
- **Connected to:** Identify Strategic Gaps via `ai_outputParser`
- **Edge cases/failures:** If model returns invalid JSON or deviates from schema, parsing fails and the run errors.

#### Node: Identify Strategic Gaps
- **Type / role:** `LangChain Agent` — orchestrates prompt + LLM + output parser.
- **Prompt setup:**
  - `text`: “make content Idea” (minimal; real instructions are in system message)
  - **System message:** Large strategist prompt including:
    - brand archetype constraints
    - content pillars
    - required JSON output format
    - injects variables:
      - `videoTitle`: `{{ $('Limit Search Results').item.json.title }}`
      - `channelName`: `{{ $('Limit Search Results').item.json.channelName }}`
      - `description/likes/views/comments`: from `Extract Audience Comments`
- **Has output parser:** enabled (`hasOutputParser: true`)
- **Outputs to:** Store Competitor Data
- **Edge cases/failures:**
  - Referencing `$('Limit Search Results').item...` inside a per-video loop is risky if item linkage is lost; generally OK in n8n but can break if nodes are rearranged or if multiple branches are merged.
  - If `Extract Audience Comments` returns `error`, the prompt will embed wrong fields and output may become invalid.

---

### Block 5 — Step 4: Save to Notion
**Overview:** Stores competitor-level insights in one Notion database, then stores each generated content idea as a separate Notion page in another database.

**Nodes involved:**
- Store Competitor Data
- ContentIdeas
- Split Content Ideas
- Set Video Analysis Limit
- Store Content Strategy
- Aggregate

#### Node: Store Competitor Data
- **Type / role:** `Notion` — creates a database page (competitor video insight record).
- **Database:** “Competitor Content Insigth” (ID provided in node)
- **Title:** `={{ $('Split Video Items').item.json.title }}`
- **Properties written:**
  - Description (rich text) from extracted description
  - URL (url) from extracted url
  - Likes, Views (rich text)
  - AudienceInsigths (rich text) built from AI output:
    - `Goals/Needs/Pain Points/Challenges` from `$json.output.analysis.audienceProfile...`
- **Inputs:** The agent output item (`$json.output...`) plus cross-node references to Extract Audience Comments and Split Video Items.
- **Outputs to:** ContentIdeas
- **Edge cases/failures:**
  - Notion auth issues, database permission issues, property type mismatch (e.g., trying to write rich_text into a non-rich_text field).
  - If `$json.output` missing (AI failure), expression errors.

#### Node: ContentIdeas
- **Type / role:** `Set` — prepares an array `contentIdeas` for splitting.
- **Field set:**
  - `contentIdeas` = `={{ $('Identify Strategic Gaps').item.json.output.contentIdeas }}`
- **Outputs to:** Split Content Ideas
- **Edge cases/failures:** If `output.contentIdeas` missing/not an array → Split Out will fail.

#### Node: Split Content Ideas
- **Type / role:** `Split Out` — one item per content idea.
- **Field:** `contentIdeas`
- **Outputs to:** Set Video Analysis Limit
- **Edge cases/failures:** Same as above; fails if field is not an array.

#### Node: Set Video Analysis Limit
- **Type / role:** `Limit` — caps number of stored ideas.
- **Max items:** 100
- **Outputs to:** Store Content Strategy
- **Edge cases/failures:** None significant; just truncation.

#### Node: Store Content Strategy
- **Type / role:** `Notion` — creates a database page per content idea.
- **Database:** “Content Idea” (ID provided in node)
- **Title:** `={{ $json.title }}`
- **Properties written:**
  - Angle, Brief, Format (rich text)
  - Idea Number (number) = `={{ $json.id }}`
  - Reference (url) = `={{ $('Store Competitor Data').item.json.url }}`
    - Note: this expects Store Competitor Data output to contain `url` (Notion’s response usually includes properties; relying on `item.json.url` may be fragile depending on Notion node output format/version).
- **Outputs to:** Aggregate
- **Edge cases/failures:**
  - Notion property schema mismatch (e.g., “Idea Number” must be a number property).
  - Reference URL may not resolve from `Store Competitor Data` output as expected; safer would be to reference `Extract Audience Comments`.  

#### Node: Aggregate
- **Type / role:** `Aggregate` — aggregates the `id` field from processed ideas.
- **Field aggregated:** `id`
- **Outputs to:** Loop Over Items (main index 0)
- **Purpose in flow:** Acts like an end-of-idea-processing step before returning to the batch loop.
- **Edge cases/failures:**
  - If `id` missing, aggregate may output empty values.
  - Looping design is somewhat non-standard; changes may break iteration.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Monthly Trigger | Schedule Trigger | Monthly workflow entry point | — | Set Niche | ## Step 1: Fetch & Filter Videos |
| Set Niche | Set | Define keyword + limit | Monthly Trigger | Search YouTube Videos | ## Step 1: Fetch & Filter Videos |
| Search YouTube Videos | HTTP Request | Firecrawl scrape of YouTube search results | Set Niche | Check Search Status | ## Step 1: Fetch & Filter Videos |
| Check Search Status | IF | Validate Firecrawl response success/status | Search YouTube Videos | Parse Video List | ## Step 1: Fetch & Filter Videos |
| Parse Video List | Code | Parse markdown into sorted video list | Check Search Status | Split Video Items | ## Step 1: Fetch & Filter Videos |
| Split Video Items | Split Out | One item per video | Parse Video List | Limit Search Results | ## Step 1: Fetch & Filter Videos |
| Limit Search Results | Limit | Cap number of videos processed | Split Video Items | Loop Over Items | ## Step 1: Fetch & Filter Videos |
| Loop Over Items | Split In Batches | Batch/loop over videos | Limit Search Results; Aggregate | Scrape Video Details | ## Step 2: Scrape & Extract Insights |
| Scrape Video Details | HTTP Request | Firecrawl scrape per video page | Loop Over Items | Extract Audience Comments | ## Step 2: Scrape & Extract Insights |
| Extract Audience Comments | Code | Extract description + top comments | Scrape Video Details | Identify Strategic Gaps | ## Step 2: Scrape & Extract Insights |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM provider for agent | — | Identify Strategic Gaps (AI input) | ## Step 3: AI Strategy Generation |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforce JSON output format | — | Identify Strategic Gaps (parser input) | ## Step 3: AI Strategy Generation |
| Identify Strategic Gaps | LangChain Agent | Generate insights + 5 ideas JSON | Extract Audience Comments; OpenAI Chat Model; Structured Output Parser | Store Competitor Data | ## Step 3: AI Strategy Generation |
| Store Competitor Data | Notion | Save competitor insights to Notion DB | Identify Strategic Gaps | ContentIdeas | ## Step 4: Save to Notion |
| ContentIdeas | Set | Extract `contentIdeas` array from agent output | Store Competitor Data | Split Content Ideas | ## Step 4: Save to Notion |
| Split Content Ideas | Split Out | One item per idea | ContentIdeas | Set Video Analysis Limit | ## Step 4: Save to Notion |
| Set Video Analysis Limit | Limit | Cap number of ideas saved | Split Content Ideas | Store Content Strategy | ## Step 4: Save to Notion |
| Store Content Strategy | Notion | Save each content idea to Notion DB | Set Video Analysis Limit | Aggregate | ## Step 4: Save to Notion |
| Aggregate | Aggregate | Collect processed idea ids; loop control | Store Content Strategy | Loop Over Items | ## Step 4: Save to Notion |

**Additional Sticky Note content (covers the workflow generally):**
- **Sticky Note (How it works / Setup steps)**  
  Content:  
  “This workflow automates market research…” plus setup steps for Firecrawl/OpenAI/Notion and running the workflow.

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add **Schedule Trigger** node named **Monthly Trigger**.
   2. Set rule interval to **every 1 month** (field: months).

2) **Add configuration node**
   1. Add **Set** node named **Set Niche**.
   2. Add fields:
      - `query` (String) = your niche keyword (e.g., `AI automation for SMEs`)
      - `limit` (Number) = how many videos to analyze (e.g., `10`)
   3. Connect **Monthly Trigger → Set Niche**.

3) **Scrape YouTube search results with Firecrawl**
   1. Add **HTTP Request** node named **Search YouTube Videos**.
   2. Method: **POST**
   3. URL: `https://api.firecrawl.dev/v2/scrape`
   4. Body: add parameter `url` with expression:  
      `=https://www.youtube.com/results?search_query={{ $json.query.split(' ').join('+') }}`
   5. Authentication: **Header Auth** (generic credential type).
      - Create **HTTP Header Auth** credential with Firecrawl API key (per Firecrawl docs, typically `Authorization: Bearer <key>`).
   6. Connect **Set Niche → Search YouTube Videos**.

4) **Validate Firecrawl response**
   1. Add **IF** node named **Check Search Status**.
   2. Conditions:
      - Boolean equals: `{{$json.success}}` equals `true`
      - Number equals: `{{$json.data.metadata.statusCode}}` equals `200`
   3. Connect **Search YouTube Videos → Check Search Status** (true output onward).

5) **Parse markdown into a structured video list**
   1. Add **Code** node named **Parse Video List**.
   2. Paste logic equivalent to the workflow’s parsing (extract title/url/channel/views/time-ago; sort; return `{videos: [...]}`).
   3. Ensure it reads `limit` from `Set Niche`.
   4. Connect **Check Search Status (true) → Parse Video List**.

6) **Split videos into items and limit count**
   1. Add **Split Out** node named **Split Video Items**.
      - Field to split: `videos`
   2. Add **Limit** node named **Limit Search Results**.
      - Max items: `={{ $('Set Niche').item.json.limit }}`
   3. Connect **Parse Video List → Split Video Items → Limit Search Results**.

7) **Loop/batch over videos**
   1. Add **Split In Batches** node named **Loop Over Items**.
   2. Keep default options (or set a batch size if desired).
   3. Connect **Limit Search Results → Loop Over Items**.
   4. Use the node output that represents “items to process” to continue the flow (in the provided workflow that is output index 1).

8) **Scrape each video page**
   1. Add **HTTP Request** node named **Scrape Video Details**.
      - POST `https://api.firecrawl.dev/v2/scrape`
      - Body parameter `url` = `={{ $json.url }}`
      - Same Firecrawl Header Auth credential
   2. Connect **Loop Over Items → Scrape Video Details**.

9) **Extract description + comments**
   1. Add **Code** node named **Extract Audience Comments**.
   2. Implement regex extraction for:
      - channelName, views, totalLikes
      - description (trim to ~500 chars)
      - top 20 comments sorted by likes
   3. Connect **Scrape Video Details → Extract Audience Comments**.

10) **Configure OpenAI + structured output**
   1. Add **OpenAI Chat Model (LangChain)** node named **OpenAI Chat Model**.
      - Model: `gpt-4.1` (or your preferred)
      - Credentials: OpenAI API key credential
   2. Add **Structured Output Parser** node named **Structured Output Parser**.
      - Provide the JSON schema example matching the expected output (analysis + contentIdeas + strategy).
   3. Add **AI Agent (LangChain Agent)** node named **Identify Strategic Gaps**.
      - Enable output parser.
      - Put the strategist instructions in **System Message**.
      - Reference input fields from **Extract Audience Comments** and video metadata from upstream nodes (title/channel).
   4. Connect:
      - **OpenAI Chat Model → Identify Strategic Gaps** via **ai_languageModel**
      - **Structured Output Parser → Identify Strategic Gaps** via **ai_outputParser**
      - **Extract Audience Comments → Identify Strategic Gaps** via main connection

11) **Save competitor insight record to Notion**
   1. Add **Notion** node named **Store Competitor Data** with resource **Database Page**.
   2. Configure Notion credentials (OAuth/token) and share the target databases with the integration.
   3. Choose competitor database (e.g., “Competitor Content Insight”).
   4. Map:
      - Title: video title
      - Description, URL, Likes, Views from Extract Audience Comments
      - Audience insights from agent output (`output.analysis.audienceProfile...`)
   5. Connect **Identify Strategic Gaps → Store Competitor Data**.

12) **Split and store each generated content idea**
   1. Add **Set** node named **ContentIdeas**:
      - Set `contentIdeas` (Array) = agent output `output.contentIdeas`
   2. Add **Split Out** node named **Split Content Ideas**:
      - Field: `contentIdeas`
   3. Add **Limit** node named **Set Video Analysis Limit**:
      - Max items: 100
   4. Add **Notion** node named **Store Content Strategy**:
      - Database: “Content Idea”
      - Title: `{{$json.title}}`
      - Properties: Angle, Brief, Format, Idea Number (`{{$json.id}}`)
      - Reference URL: link back to competitor video
   5. Connect:  
      **Store Competitor Data → ContentIdeas → Split Content Ideas → Set Video Analysis Limit → Store Content Strategy**

13) **Aggregate and loop control**
   1. Add **Aggregate** node named **Aggregate**:
      - Aggregate field: `id`
   2. Connect **Store Content Strategy → Aggregate**.
   3. Connect **Aggregate → Loop Over Items** to continue batching (matching the original design).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automates market research by identifying what your audience actually wants…” plus setup steps for Firecrawl/OpenAI/Notion and running the workflow. | Sticky note “How it works / Setup steps” embedded in the canvas. |
| Firecrawl API key is required and is configured via Header Auth on the HTTP Request nodes. | https://firecrawl.dev |
| Notion requires two databases: one for competitor insights and one for content ideas; both must be shared with the Notion integration used by n8n. | Notion integration setup |

