Generate a weekly content calendar with OpenAI GPT-5-mini, RSS feeds, and Notion

https://n8nworkflows.xyz/workflows/generate-a-weekly-content-calendar-with-openai-gpt-5-mini--rss-feeds--and-notion-13103


# Generate a weekly content calendar with OpenAI GPT-5-mini, RSS feeds, and Notion

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate weekly content calendar with AI from RSS feeds  
**Internal workflow name:** Generate weekly content calendar with AI from RSS feeds  
**Purpose:** Every Monday at 9:00 AM, this workflow pulls recent articles from multiple RSS feeds, filters them to the last 7 days, prompts OpenAI (GPT‑5‑mini) to generate exactly 7 structured content ideas (strict JSON), then creates one Notion database page per idea. If no recent articles are found, it stops gracefully.

### 1.1 Scheduled Trigger (Entry Point)
Runs weekly on Monday at 9 AM and fans out to multiple RSS sources in parallel.

### 1.2 RSS Ingestion (Multi-source)
Fetches articles from TechCrunch, Ars Technica, and MIT News RSS feeds.

### 1.3 Aggregation + Filtering + Prompt Formatting
Merges items from all RSS feeds, filters to recent posts, sorts newest-first, caps the list, and formats a numbered list for LLM consumption.

### 1.4 Conditional Gate (No articles vs. Continue)
If at least one recent article exists, proceed to AI generation; otherwise, end via a NoOp node.

### 1.5 AI Generation (Structured JSON) + Itemization
Uses the OpenAI node with structured JSON output constraints, then splits the 7 returned ideas into individual items.

### 1.6 Notion Output
Creates a Notion database page for each generated idea, mapping fields from the model output to Notion properties.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Trigger (Entry Point)

**Overview:** Kicks off the workflow automatically every week, Monday at 09:00. It triggers three RSS reads in parallel.  
**Nodes involved:** Weekly Monday 9AM

#### Node: Weekly Monday 9AM
- **Type / role:** `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`) — time-based entry point.
- **Configuration (interpreted):**
  - Weekly interval.
  - **Day:** Monday (`triggerAtDay: 1`).
  - **Hour:** 9 (server/workflow timezone as configured in n8n instance).
- **Inputs:** None (trigger node).
- **Outputs / connections:**
  - Sends execution to:
    - `RSS Feed - Tech News`
    - `RSS Feed - Biz & IT`
    - `RSS Feed - MIT Science & Tech`
- **Version notes:** TypeVersion 1.2.
- **Edge cases / failures:**
  - Timezone surprises if instance timezone differs from expected business timezone.
  - Missed executions if n8n instance is down at scheduled time (depends on n8n schedule handling and hosting).

---

### Block 2 — RSS Ingestion (Multi-source)

**Overview:** Reads articles from three RSS feeds. Each node outputs a list of items (articles) which are later merged.  
**Nodes involved:** RSS Feed - Tech News, RSS Feed - Biz & IT, RSS Feed - MIT Science & Tech

#### Node: RSS Feed - Tech News
- **Type / role:** `RSS Feed Read` (`n8n-nodes-base.rssFeedRead`) — fetch and parse RSS.
- **Configuration:**
  - URL: `https://techcrunch.com/feed/`
  - Default parsing options.
- **Input:** From `Weekly Monday 9AM`.
- **Output / connections:** To `Combine All Articles` (input index 0).
- **Version notes:** TypeVersion 1.
- **Edge cases / failures:**
  - Feed unreachable (DNS/timeout/HTTP errors).
  - RSS format changes causing parse issues.
  - Rate limiting / bot protection by the feed host.

#### Node: RSS Feed - Biz & IT
- **Type / role:** `RSS Feed Read`
- **Configuration:**
  - URL: `https://feeds.arstechnica.com/arstechnica/technology-lab`
- **Input:** From `Weekly Monday 9AM`.
- **Output / connections:** To `Combine All Articles` (input index 1).
- **Version notes:** TypeVersion 1.
- **Edge cases / failures:** same as above.

#### Node: RSS Feed - MIT Science & Tech
- **Type / role:** `RSS Feed Read`
- **Configuration:**
  - URL: `https://news.mit.edu/rss/topic/science-technology-and-society`
- **Input:** From `Weekly Monday 9AM`.
- **Output / connections:** To `Combine All Articles` (input index 2).
- **Version notes:** TypeVersion 1.
- **Edge cases / failures:** same as above.

---

### Block 3 — Aggregation + Filtering + Prompt Formatting

**Overview:** Combines the three RSS streams, then filters for the last 7 days, sorts newest-first, limits to 20 items, and produces a numbered list string for the AI prompt plus debug data.  
**Nodes involved:** Combine All Articles, Filter and Format Articles

#### Node: Combine All Articles
- **Type / role:** `Merge` (`n8n-nodes-base.merge`) — multi-input aggregation.
- **Configuration:**
  - Mode: **Combine** with `numberInputs: 3` (expects three incoming connections).
  - Purpose: output a single combined stream of items from all feeds.
- **Inputs:** Three inputs from the three RSS nodes (indexes 0/1/2).
- **Outputs / connections:** To `Filter and Format Articles`.
- **Version notes:** TypeVersion 3.
- **Edge cases / failures:**
  - If one feed fails and produces no items, behavior depends on upstream failure handling (an RSS node error typically stops execution unless “Continue On Fail” is configured—here it is not).
  - If any feed returns unusual item shapes (missing `pubDate`, `title`, etc.), downstream code may filter them out or format with fallbacks.

#### Node: Filter and Format Articles
- **Type / role:** `Code` (`n8n-nodes-base.code`) — transforms merged items into AI-ready prompt data.
- **Key configuration choices:**
  - Looks back **7 days** (`DAYS_BACK = 7`).
  - Sends at most **20 articles** to the AI (`MAX_ARTICLES = 20`).
  - Truncates description/snippet to **150 chars** (`SNIPPET_LENGTH = 150`).
- **Key logic / variables:**
  - `cutoffDate` = now minus `DAYS_BACK`.
  - `allArticles = $input.all().map(item => item.json)`
  - Filter: requires `article.pubDate` and `new Date(article.pubDate) >= cutoffDate`
  - Sort descending by pubDate
  - Slice to max articles
  - Format lines like: `1. "Title" - snippet...`
- **Output (one item):**
  - `articleCount`: number of articles after filtering
  - `articlesList`: formatted numbered list string for prompt
  - `rawArticles`: the filtered article objects (debug/reference)
- **Input:** From `Combine All Articles`.
- **Output / connections:** To `Has Articles?`
- **Version notes:** TypeVersion 2.
- **Edge cases / failures:**
  - Invalid `pubDate` strings could become `Invalid Date` and fail comparisons (those items will likely be filtered out implicitly or behave unexpectedly).
  - If `title` is missing, the formatted line will contain `"undefined"`.
  - Large feeds: merge could produce many items; code caps to 20 but still loads all items in memory via `$input.all()`.

---

### Block 4 — Conditional Gate (No articles vs. Continue)

**Overview:** Prevents the OpenAI call (and Notion writes) if there are no recent articles in the defined timeframe.  
**Nodes involved:** Has Articles?, No Recent Articles

#### Node: Has Articles?
- **Type / role:** `IF` (`n8n-nodes-base.if`) — conditional branching.
- **Condition (interpreted):**
  - Check whether `{{$json.articleCount}} > 0`
- **Inputs:** From `Filter and Format Articles`.
- **Outputs / connections:**
  - **True branch:** to `Generate Content Calendar (Structured)`
  - **False branch:** to `No Recent Articles`
- **Version notes:** TypeVersion 2.
- **Edge cases / failures:**
  - If `articleCount` is missing or not numeric, strict validation may cause unexpected branch behavior (here it is produced reliably by the previous Code node).

#### Node: No Recent Articles
- **Type / role:** `No Operation` (`n8n-nodes-base.noOp`) — graceful stop.
- **Configuration:** none.
- **Inputs:** From `Has Articles?` (false branch).
- **Outputs:** None.
- **Version notes:** TypeVersion 1.
- **Edge cases / failures:** none (used specifically to end cleanly).

---

### Block 5 — AI Generation (Structured JSON) + Itemization

**Overview:** Sends the weekly article list to GPT‑5‑mini with strict “JSON only” constraints to generate 7 content ideas, then splits the returned collection into individual items for downstream Notion insertion.  
**Nodes involved:** Generate Content Calendar (Structured), Split Into Individual Ideas

#### Node: Generate Content Calendar (Structured)
- **Type / role:** OpenAI via LangChain (`@n8n/n8n-nodes-langchain.openAi`) — LLM call with structured output expectation.
- **Configuration choices (interpreted):**
  - **Model:** `gpt-5-mini`
  - **Max tokens:** 4000
  - **JSON output:** enabled (`jsonOutput: true`)
  - **Messages:**
    - **System message:** defines role, strict JSON-only output, exact object schema, field rules, exactly 7 ideas, publish day constraints (MONDAY–FRIDAY), and content mix constraints.
    - **User message:** injects the formatted list via `{{ $json.articlesList }}` and requests exactly 7 content ideas.
- **Inputs:** From `Has Articles?` (true branch), receiving `articlesList`.
- **Outputs / connections:** To `Split Into Individual Ideas`.
- **Version notes:** TypeVersion 1.8.
- **Edge cases / failures:**
  - Model may still return invalid JSON or a different shape than expected; despite “jsonOutput”, strict schema is enforced only by prompt unless the node performs parsing/validation—this can break the next node.
  - Token overflow if article list becomes too large (mitigated by `MAX_ARTICLES` and `SNIPPET_LENGTH`, but still possible with long titles/snippets).
  - Credential/auth errors (OpenAI API key missing/invalid), rate limits, or provider outages.

#### Node: Split Into Individual Ideas
- **Type / role:** `Split Out` (`n8n-nodes-base.splitOut`) — converts an array field into one item per element.
- **Configuration:**
  - Field to split out: `message.content.ideas`
- **Inputs:** From `Generate Content Calendar (Structured)`.
- **Outputs / connections:** To `Create Database Page in Notion` (runs once per idea).
- **Version notes:** TypeVersion 1.
- **Edge cases / failures:**
  - If the OpenAI node output does not contain `message.content.ideas` (different key name or parsing failure), this node will error or produce zero items.
  - If `ideas` is not an array, split will fail.

---

### Block 6 — Notion Output

**Overview:** Creates one Notion database page per generated idea, mapping AI fields into Notion properties.  
**Nodes involved:** Create Database Page in Notion

#### Node: Create Database Page in Notion
- **Type / role:** `Notion` (`n8n-nodes-base.notion`) — writes a page to a Notion database.
- **Configuration (interpreted):**
  - Resource: **Database Page**
  - Operation: **Create** (implied by “Create Database Page” behavior)
  - Database: **not set in JSON** (databaseId value is empty; must be configured before use)
  - Title field: `{{ $json.Title }}`
  - Property mappings:
    - `Content Type` ← `{{ $json['Content type'] }}`
    - `Target Audience` ← `{{ $json['Target Audience'] }}`
    - `Publish Day` ← `{{ $json['Publish Day'] }}`
    - `Status` ← `{{ $json.Status }}`
    - `Key Points` ← `{{ $json['Key Point'] }}`
- **Inputs:** From `Split Into Individual Ideas` (each item is one idea).
- **Outputs:** None downstream.
- **Version notes:** TypeVersion 2.2.
- **Edge cases / failures:**
  - Notion credentials missing/invalid, or workspace access not granted to the integration.
  - DatabaseId not set (currently blank) → node will fail at runtime.
  - Property type mismatches: the node uses “rich text” mappings for several fields; if your Notion properties are Select/Title types, you must map accordingly.
  - If AI output keys differ in capitalization/spelling (`Content type` vs `Content Type`), expressions will resolve to empty values.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Monday 9AM | Schedule Trigger | Weekly entry point | — | RSS Feed - Tech News; RSS Feed - Biz & IT; RSS Feed - MIT Science & Tech | ## Generate weekly content calendar with AI from RSS feeds\n\n### How it works\nThis workflow automatically generates a weekly content calendar by analyzing trending articles from your industry RSS feeds. Every Monday at 9 AM, it collects articles from TechCrunch, Ars Technica, and MIT News, filters those published in the last 7 days, and uses OpenAI's structured output to generate exactly 7 content ideas in validated JSON format. Each idea includes a title, content type, target audience, and key talking points—then automatically adds them to your Notion database.\n\n### Setup steps\n1. **Configure RSS feeds**: Replace the example URLs with feeds relevant to your industry\n2. **Add OpenAI credentials**: Connect your OpenAI API key (gpt-5-mini recommended)\n3. **Create Notion database** with these properties:\n   - Title (title)\n   - Content Type (select: blog, linkedin, twitter, video, newsletter)\n   - Target Audience (rich text)\n   - Publish Day (select: Monday-Friday)\n   - Status (select: Draft, In Progress, Done)\n   - Key Points (rich text)\n4. **Connect Notion**: Add your Notion credentials and select your database\n5. **Map fields**: In the Notion node, map the AI output fields to your database properties\n6. **Test manually** before enabling the schedule\n\n**Tip**: Duplicate RSS nodes to add more sources. The AI automatically adjusts to more input. |
| RSS Feed - Tech News | RSS Feed Read | Fetch RSS articles | Weekly Monday 9AM | Combine All Articles | ### 📥 RSS Sources\nAdd your industry feeds here.\nDuplicate nodes for more sources. |
| RSS Feed - Biz & IT | RSS Feed Read | Fetch RSS articles | Weekly Monday 9AM | Combine All Articles | ### 📥 RSS Sources\nAdd your industry feeds here.\nDuplicate nodes for more sources. |
| RSS Feed - MIT Science & Tech | RSS Feed Read | Fetch RSS articles | Weekly Monday 9AM | Combine All Articles | ### 📥 RSS Sources\nAdd your industry feeds here.\nDuplicate nodes for more sources. |
| Combine All Articles | Merge | Combine 3 RSS streams | RSS Feed - Tech News; RSS Feed - Biz & IT; RSS Feed - MIT Science & Tech | Filter and Format Articles | ### 🔄 Processing\nAggregate and filter articles from last 7 days.\n\n**Customize in code:**\n- `DAYS_BACK` → time window\n- `MAX_ARTICLES` → limit for AI |
| Filter and Format Articles | Code | Filter recent items + format prompt list | Combine All Articles | Has Articles? | ### 🔄 Processing\nAggregate and filter articles from last 7 days.\n\n**Customize in code:**\n- `DAYS_BACK` → time window\n- `MAX_ARTICLES` → limit for AI |
| Has Articles? | IF | Branch based on articleCount | Filter and Format Articles | Generate Content Calendar (Structured); No Recent Articles |  |
| Generate Content Calendar (Structured) | OpenAI (LangChain) | Generate 7 structured content ideas | Has Articles? (true) | Split Into Individual Ideas | ### 🤖 AI Generation\nGPT creates 7 content ideas with strict JSON output.\n\nDistributes across Monday-Friday. |
| Split Into Individual Ideas | Split Out | One item per content idea | Generate Content Calendar (Structured) | Create Database Page in Notion | ### 🤖 AI Generation\nGPT creates 7 content ideas with strict JSON output.\n\nDistributes across Monday-Friday. |
| Create Database Page in Notion | Notion | Create a Notion DB page per idea | Split Into Individual Ideas | — | ### 📝 Notion Output\nEach idea becomes a database page.\n\nReady for your content workflow. |
| No Recent Articles | NoOp | Stop gracefully when no items | Has Articles? (false) | — | **No articles found in timeframe**\n\nStop execution gracefully. |
| Sticky Note | Sticky Note | Documentation / canvas note | — | — | ## Generate weekly content calendar with AI from RSS feeds\n\n### How it works\nThis workflow automatically generates a weekly content calendar by analyzing trending articles from your industry RSS feeds. Every Monday at 9 AM, it collects articles from TechCrunch, Ars Technica, and MIT News, filters those published in the last 7 days, and uses OpenAI's structured output to generate exactly 7 content ideas in validated JSON format. Each idea includes a title, content type, target audience, and key talking points—then automatically adds them to your Notion database.\n\n### Setup steps\n1. **Configure RSS feeds**: Replace the example URLs with feeds relevant to your industry\n2. **Add OpenAI credentials**: Connect your OpenAI API key (gpt-5-mini recommended)\n3. **Create Notion database** with these properties:\n   - Title (title)\n   - Content Type (select: blog, linkedin, twitter, video, newsletter)\n   - Target Audience (rich text)\n   - Publish Day (select: Monday-Friday)\n   - Status (select: Draft, In Progress, Done)\n   - Key Points (rich text)\n4. **Connect Notion**: Add your Notion credentials and select your database\n5. **Map fields**: In the Notion node, map the AI output fields to your database properties\n6. **Test manually** before enabling the schedule\n\n**Tip**: Duplicate RSS nodes to add more sources. The AI automatically adjusts to more input. |
| Sticky Note1 | Sticky Note | Documentation / canvas note | — | — | ### 📥 RSS Sources\nAdd your industry feeds here.\nDuplicate nodes for more sources. |
| Sticky Note2 | Sticky Note | Documentation / canvas note | — | — | ### 🔄 Processing\nAggregate and filter articles from last 7 days.\n\n**Customize in code:**\n- `DAYS_BACK` → time window\n- `MAX_ARTICLES` → limit for AI |
| Sticky Note3 | Sticky Note | Documentation / canvas note | — | — | ### 🤖 AI Generation\nGPT creates 7 content ideas with strict JSON output.\n\nDistributes across Monday-Friday. |
| Sticky Note4 | Sticky Note | Documentation / canvas note | — | — | ### 📝 Notion Output\nEach idea becomes a database page.\n\nReady for your content workflow. |
| Sticky Note5 | Sticky Note | Documentation / canvas note | — | — | **No articles found in timeframe**\n\nStop execution gracefully. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name it: *Generate weekly content calendar with AI from RSS feeds* (or your preferred name).

2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Configure:
     - Interval: **Weeks**
     - Day of week: **Monday**
     - Time: **09:00**
   - This is the entry node.

3. **Add three RSS Feed Read nodes**
   - Node A: **RSS Feed Read**
     - URL: `https://techcrunch.com/feed/`
   - Node B: **RSS Feed Read**
     - URL: `https://feeds.arstechnica.com/arstechnica/technology-lab`
   - Node C: **RSS Feed Read**
     - URL: `https://news.mit.edu/rss/topic/science-technology-and-society`
   - Connect **Schedule Trigger → each RSS node** (three parallel connections).
   - Optional: replace URLs with your industry feeds.

4. **Add a Merge node to combine all feeds**
   - Node: **Merge**
   - Configure:
     - Number of inputs: **3**
   - Connect:
     - RSS A → Merge input 1
     - RSS B → Merge input 2
     - RSS C → Merge input 3

5. **Add a Code node to filter/format**
   - Node: **Code**
   - Paste logic equivalent to:
     - Filter where `pubDate` exists and is within last **7** days
     - Sort newest-first
     - Limit to **20**
     - Build `articlesList` as numbered lines with truncated snippets (~150 chars)
     - Output a single item with `articleCount`, `articlesList`, `rawArticles`
   - Connect: **Merge → Code**

6. **Add an IF node to guard empty results**
   - Node: **IF**
   - Condition:
     - Left value: expression `{{$json.articleCount}}`
     - Operator: **greater than**
     - Right value: `0`
   - Connect: **Code → IF**

7. **Add a NoOp node for the “no articles” branch**
   - Node: **No Operation**
   - Connect: **IF (false) → NoOp**
   - (This ends the workflow gracefully.)

8. **Add the OpenAI (LangChain) node for structured generation**
   - Node: **OpenAI** (LangChain OpenAI node)
   - Credentials:
     - Create/choose **OpenAI API credential** in n8n and attach it.
   - Configure:
     - Model: **gpt-5-mini**
     - Max tokens: **4000**
     - Enable **JSON output**
     - System message: include strict “JSON only” rules and require exactly 7 items with fields:
       - `Title`, `Content type`, `Target Audience`, `Publish Day`, `Status`, `Key Point`
       - Publish Day restricted to `MONDAY..FRIDAY`, Status always `Draft`
     - User message: inject `{{$json.articlesList}}`
   - Connect: **IF (true) → OpenAI**

   **Important:** Ensure your OpenAI node returns a structure that contains an array of ideas at a known path (the next step expects `message.content.ideas`).

9. **Add Split Out node**
   - Node: **Split Out**
   - Field to split out: `message.content.ideas`
   - Connect: **OpenAI → Split Out**

10. **Create your Notion database**
    - In Notion, create a database with properties matching your mapping strategy. This workflow expects:
      - **Title** (title)
      - **Content Type**
      - **Target Audience**
      - **Publish Day**
      - **Status**
      - **Key Points**
    - Choose property types carefully (Select vs Rich text). The provided node mapping uses rich text for most fields.

11. **Add Notion node to create database pages**
    - Node: **Notion**
    - Credentials:
      - Create/choose **Notion OAuth2 / integration credential** in n8n.
      - Share the target database with the Notion integration.
    - Resource: **Database Page**
    - Operation: **Create**
    - Select your **Database ID**
    - Map fields:
      - Title: `{{$json.Title}}`
      - Content Type: `{{$json['Content type']}}`
      - Target Audience: `{{$json['Target Audience']}}`
      - Publish Day: `{{$json['Publish Day']}}`
      - Status: `{{$json.Status}}`
      - Key Points: `{{$json['Key Point']}}`
    - Connect: **Split Out → Notion**

12. **Test manually**
    - Run the workflow manually once.
    - Validate:
      - `articleCount > 0`
      - OpenAI output parses and contains `message.content.ideas` as an array of 7
      - Notion pages are created with correct property mapping

13. **Activate the workflow**
    - Turn the workflow **Active** to enable the weekly schedule.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Configure RSS feeds by replacing the example URLs; duplicate RSS nodes to add more sources. | Sticky note: “📥 RSS Sources” |
| Customize filtering window and AI load via `DAYS_BACK` and `MAX_ARTICLES` in the Code node. | Sticky note: “🔄 Processing” |
| AI node enforces strict JSON-only output and generates exactly 7 ideas distributed across Monday–Friday. | Sticky note: “🤖 AI Generation” |
| Notion output expects one database page per idea; ensure DB properties exist and are correctly typed/mapped. | Sticky note: “📝 Notion Output” |
| If no articles are found in timeframe, workflow stops via NoOp node. | Sticky note: “No articles found in timeframe” |