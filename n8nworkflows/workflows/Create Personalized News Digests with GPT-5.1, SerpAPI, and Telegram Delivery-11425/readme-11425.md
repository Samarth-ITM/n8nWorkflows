Create Personalized News Digests with GPT-5.1, SerpAPI, and Telegram Delivery

https://n8nworkflows.xyz/workflows/create-personalized-news-digests-with-gpt-5-1--serpapi--and-telegram-delivery-11425


# Create Personalized News Digests with GPT-5.1, SerpAPI, and Telegram Delivery

---

### 1. Workflow Overview

This automated workflow creates personalized news digests by leveraging AI (GPT-5.1), SerpAPI news data, and Telegram for delivery. It is designed to fetch recent news articles on specified topics, filter and deduplicate them based on past newsletters, let an AI editor decide whether to send a newsletter and select top articles, enrich the articles with additional research, and finally deliver the curated newsletter to a Telegram chat.

The workflow is logically divided into the following blocks:

- **1.1 Schedule & Configuration:** Defines the schedule and global parameters such as topics, language, and frequency constraints.
- **1.2 News Fetching & Storage:** Builds search queries, fetches news articles from SerpAPI, processes and stores them in a data table with deduplication.
- **1.3 News Filtering & Frequency Guardrails:** Retrieves past newsletters, filters out old or redundant articles, and enforces minimum frequency between newsletters.
- **1.4 Newsletter Selection & AI Editorial Decision:** Aggregates candidate articles and past newsletters for context, then uses GPT-5.1 AI to decide whether to send a newsletter and which articles to include.
- **1.5 Newsletter Writing & Delivery:** Enriches selected articles with AI and Tavily web search, composes the newsletter content, archives it, and sends it via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Configuration

- **Overview:** Initializes the workflow run with scheduled triggers and sets global parameters including topics, language, and minimum/maximum days between newsletters.
- **Nodes Involved:** Schedule Trigger, Set topics and language, Sticky Notes (for documentation)
- **Node Details:**

  - **Schedule Trigger**
    - Type: ScheduleTrigger
    - Role: Triggers the workflow twice daily at 9 AM and 5 PM (Europe/Berlin timezone)
    - Configuration: Cron expression `0 0 9,17 * * *`
    - Inputs: None (trigger node)
    - Outputs: Connects to "Set topics and language"
    - Edge cases: Cron misconfiguration or timezone issues can delay execution.
  
  - **Set topics and language**
    - Type: Set
    - Role: Defines key workflow parameters:
      - `topics`: comma-separated list, default "AI, n8n"
      - `language`: output newsletter language, default "English"
      - `minDaysBetween`: minimum days between newsletters (default 2)
      - `maxDaysBetween`: maximum days without newsletter before forced send (default 5)
    - Inputs: Trigger from Schedule Trigger
    - Outputs: Feeds into topic query builder and newsletter history retrieval
    - Edge cases: Empty or malformed topics string; invalid number values.

---

#### 2.2 News Fetching & Storage

- **Overview:** Converts configured topics into individual queries, fetches news articles via SerpAPI’s DuckDuckGo News engine, splits results into articles, and stores them with deduplication into a News data table.
- **Nodes Involved:** Build topic queries, Fetch news from SerpAPI (DuckDuckGo News), Split SerpAPI results into articles, Upsert articles into News table, Sticky Notes
- **Node Details:**

  - **Build topic queries**
    - Type: Code
    - Role: Splits the comma-separated `topics` string into an array of individual topic objects for querying
    - Key expression: Splits `$input.first().json.topics` by comma and trims spaces
    - Inputs: From Set topics and language
    - Outputs: An array of objects with single `topic` property each
    - Edge cases: Empty topics input or trailing commas resulting in empty queries.

  - **Fetch news from SerpAPI (DuckDuckGo News)**
    - Type: HTTP Request
    - Role: Fetches news articles for each topic using SerpAPI’s DuckDuckGo News engine
    - Query parameters: `engine=duckduckgo_news`, `q` = current topic, `df=d` (last day)
    - Authentication: HTTP Query Auth with SerpAPI credentials
    - Inputs: From Build topic queries
    - Outputs: JSON response with news results
    - Edge cases: API quota limits, network timeouts, invalid API key, unexpected response format.

  - **Split SerpAPI results into articles**
    - Type: SplitOut
    - Role: Splits the `results` array from SerpAPI response into individual article items
    - Field to split: `results`
    - Inputs: From Fetch news node
    - Outputs: Individual article JSON objects
    - Edge cases: Empty or missing `results` field, malformed JSON.

  - **Upsert articles into News table**
    - Type: DataTable (upsert)
    - Role: Inserts or updates articles in the News data table with deduplication by `title` and `url`
    - Fields: `url`, `date` (converted from UNIX timestamp), `title`, `source`, `excerpt`
    - Inputs: From Split SerpAPI results into articles
    - Outputs: Passes combined article data downstream
    - Edge cases: Data table access errors, date conversion failures, duplicate handling logic errors.

---

#### 2.3 News Filtering & Frequency Guardrails

- **Overview:** Retrieves past newsletters, sorts them by newest first, limits to recent entries, filters out articles older than the last newsletter, and applies frequency constraints to stop if newsletters are too recent.
- **Nodes Involved:** Get previous newsletters, Sort newsletters by newest first, Get most recent newsletter, Limit previous newsletters to last 5, Aggregate previous newsletters into list, Combine articles with last newsletter metadata, Filter articles newer than last newsletter, Stop if last newsletter is too recent, Sticky Notes
- **Node Details:**

  - **Get previous newsletters**
    - Type: DataTable (get)
    - Role: Fetches all past newsletters from the Newsletters data table
    - Inputs: From Set topics and language
    - Outputs: Newsletter items
    - Edge cases: Data table connectivity, large data volume.

  - **Sort newsletters by newest first**
    - Type: Sort
    - Role: Orders past newsletters descending by creation date
    - Inputs: From Get previous newsletters
    - Outputs: Sorted newsletters
    - Edge cases: Sorting failures if date fields missing or malformed.

  - **Get most recent newsletter**
    - Type: Limit
    - Role: Limits to the single most recent newsletter
    - Inputs: From Sort newsletters
    - Outputs: One newsletter item
    - Edge cases: Empty newsletter history.

  - **Limit previous newsletters to last 5**
    - Type: Limit
    - Role: Restricts to last 5 newsletters for context in AI evaluation
    - Inputs: From Sort newsletters
    - Outputs: 5 newest newsletters
    - Edge cases: Less than 5 newsletters existing.

  - **Aggregate previous newsletters into list**
    - Type: Aggregate
    - Role: Aggregates the last 5 newsletters into a single array field named `newsletters`
    - Inputs: From Limit previous newsletters to last 5
    - Outputs: Aggregated newsletters data
    - Edge cases: Empty input.

  - **Combine articles with last newsletter metadata**
    - Type: Merge (combine)
    - Role: Combines new articles with metadata from the most recent newsletter for filtering
    - Inputs: From Get most recent newsletter and Upsert articles into News table
    - Outputs: Combined dataset
    - Edge cases: Merging errors if data shapes mismatch.

  - **Filter articles newer than last newsletter**
    - Type: Filter
    - Role: Filters articles to keep only those published after the last newsletter’s creation date or a default date if none exists
    - Condition: Article date > last newsletter createdAt or fallback date (Jan 1, 2024)
    - Inputs: From Combine articles with last newsletter metadata
    - Outputs: Filtered article list
    - Edge cases: Date parsing errors, missing dates.

  - **Stop if last newsletter is too recent**
    - Type: If
    - Role: Stops workflow if the last newsletter was sent within the configured `minDaysBetween` days to avoid excessive frequency
    - Condition: Last newsletter date is before current time minus `minDaysBetween`
    - Inputs: From Filter articles newer than last newsletter
    - Outputs: Continues only if sufficient time elapsed
    - Edge cases: Missing last newsletter date, time zone discrepancies.

---

#### 2.4 Newsletter Selection & AI Editorial Decision

- **Overview:** Aggregates candidate articles and recent newsletters for AI context, then uses GPT-5.1 to decide if the newsletter should be sent and selects the best articles. Applies fallback rules based on maximum days between newsletters.
- **Nodes Involved:** Aggregate candidate articles for AI, Combine candidate articles with past newsletters, AI: decide send + select articles, If AI decided to send newsletter, Sticky Notes
- **Node Details:**

  - **Aggregate candidate articles for AI**
    - Type: Aggregate
    - Role: Aggregates candidate articles selecting fields: `title`, `excerpt`, `source`, and `url` into a `results` array for AI processing
    - Inputs: From Stop if last newsletter is too recent (filtered articles)
    - Outputs: Aggregated article data
    - Edge cases: Empty article list.

  - **Combine candidate articles with past newsletters**
    - Type: Merge (combine)
    - Role: Combines aggregated candidate articles with aggregated past newsletters to provide AI with full context
    - Inputs: From Aggregate candidate articles and Aggregate previous newsletters into list
    - Outputs: Combined input dataset for AI decision
    - Edge cases: Data format mismatch.

  - **AI: decide send + select articles**
    - Type: LangChain OpenAI (GPT-5.1)
    - Role: AI Newsletter Editor evaluates novelty, relevance, and value of articles against past newsletters and user topics, decides "YES" or "NO" for sending newsletter, and selects top 3-5 articles with rewritten titles and summaries.
    - Key parameters:
      - Model: GPT-5.1
      - Input: JSON containing article list and past newsletters content
      - Output: JSON with `decision` ("YES"/"NO") and array of selected articles
    - Inputs: From Combine candidate articles with past newsletters
    - Outputs: AI decision and selected articles
    - Edge cases: AI API failures, output parsing errors, incorrect JSON format.

  - **If AI decided to send newsletter**
    - Type: If
    - Role: Routes workflow based on AI decision; continues only if decision is "YES"
    - Condition: `$json.message.content.decision == "YES"`
    - Inputs: From AI decision node
    - Outputs: Continues only if sending newsletter
    - Edge cases: Missing AI decision, malformed data.

---

#### 2.5 Newsletter Writing & Delivery

- **Overview:** Splits selected articles for individual enrichment, uses GPT-5.1 with Tavily web search to produce concise, factual article summaries, aggregates enriched articles, inserts the newsletter content into the database, and sends the formatted newsletter to Telegram.
- **Nodes Involved:** Split selected articles for enrichment, AI: enrich & write article, Tavily web search tool, GPT-5.1 (AI language model), Parse enriched article JSON, Aggregate enriched articles, Insert newsletter content into Newsletters table, Send newsletter to Telegram, Sticky Notes
- **Node Details:**

  - **Split selected articles for enrichment**
    - Type: SplitOut
    - Role: Splits selected articles array into individual article JSON objects for enrichment
    - Field to split: `message.content.articles`
    - Inputs: From If AI decided to send newsletter
    - Outputs: Individual article items for enrichment
    - Edge cases: Empty articles array.

  - **AI: enrich & write article**
    - Type: LangChain Agent (GPT-5.1)
    - Role: Enriches each article using Tavily web search tool and GPT-5.1 to write concise, factual 1–3 sentence articles with updated titles and translated to desired language.
    - Input Template:
      - Includes title, summary (excerpt), source, URL, and target language
      - Instructions to search recent sources, rewrite, and output structured JSON with title, content, source, url
    - Uses Tavily web search tool as an AI tool plugin for research
    - Output parsed by Parse enriched article JSON node
    - Inputs: From Split selected articles for enrichment and Tavily web search tool, GPT-5.1 language model
    - Outputs: Structured enriched article JSON output
    - Edge cases: AI or web search API errors, parsing failures.

  - **Tavily web search tool**
    - Type: External AI Tool (Tavily)
    - Role: Provides recent web search results to AI agent for enrichment
    - Inputs: AI agent prompt query
    - Outputs: Search results to AI agent
    - Edge cases: API limits, network failures.

  - **GPT-5.1 (AI language model)**
    - Type: LangChain LM Chat OpenAI
    - Role: Provides GPT-5.1 language model capabilities for article enrichment
    - Inputs: From AI agent
    - Outputs: Language model completions
    - Edge cases: API quota, latency.

  - **Parse enriched article JSON**
    - Type: LangChain Output Parser Structured
    - Role: Parses AI agent output to structured JSON with defined schema: title, content, source, url
    - Inputs: From AI agent output
    - Outputs: Parsed JSON
    - Edge cases: Parsing errors from malformed AI output.

  - **Aggregate enriched articles**
    - Type: Aggregate
    - Role: Collects all enriched articles into a single array under field `output`
    - Inputs: From AI: enrich & write article
    - Outputs: Aggregated enriched articles
    - Edge cases: Missing enriched articles.

  - **Insert newsletter content into Newsletters table**
    - Type: DataTable (insert)
    - Role: Saves the final newsletter content (concatenated article summaries) into the Newsletters data table for archival
    - Content formatted with bold titles, article content, and source links in markdown
    - Inputs: From Aggregate enriched articles
    - Outputs: Passes to Telegram send node
    - Edge cases: Data table write errors.

  - **Send newsletter to Telegram**
    - Type: Telegram
    - Role: Sends the formatted newsletter as a message to a configured Telegram chat ID
    - Chat ID: Configured as `89066090`
    - Text: Markdown formatted concatenation of enriched articles with bold titles and source hyperlinks
    - Inputs: From Insert newsletter content into Newsletters table
    - Outputs: Final output node (no further connected nodes)
    - Edge cases: Telegram API errors, invalid chat ID, message length limits.

---

### 3. Summary Table

| Node Name                        | Node Type                          | Functional Role                            | Input Node(s)                           | Output Node(s)                            | Sticky Note                                                                                         |
|---------------------------------|----------------------------------|--------------------------------------------|---------------------------------------|------------------------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger                | ScheduleTrigger                   | Initiates workflow on schedule             | None                                  | Set topics and language                   | Triggers the workflow and sets global parameters. • Schedule Trigger: Runs at 9am and 5pm daily per default • Set topics and language: Configure your topics (comma-separated), output language, and frequency guardrails |
| Set topics and language         | Set                              | Sets topics, language, frequency parameters | Schedule Trigger                     | Build topic queries, Get previous newsletters | See above                                                                                         |
| Build topic queries             | Code                             | Splits topics string into individual queries | Set topics and language               | Fetch news from SerpAPI (DuckDuckGo News) | Fetch News • Build topic queries: Splits comma-separated topics into individual search queries • Fetch news from SerpAPI: Queries DuckDuckGo News for each topic (last 24h) • Split results: Extracts individual articles from API response • Upsert into News table: Stores articles with deduplication |
| Fetch news from SerpAPI (DuckDuckGo News) | HTTP Request                    | Fetches news articles for each topic       | Build topic queries                   | Split SerpAPI results into articles       | See above                                                                                         |
| Split SerpAPI results into articles | SplitOut                        | Splits API response array into individual articles | Fetch news from SerpAPI              | Upsert articles into News table           | See above                                                                                         |
| Upsert articles into News table | DataTable (upsert)               | Stores articles with deduplication          | Split SerpAPI results into articles  | Combine articles with last newsletter metadata | See above                                                                                         |
| Get previous newsletters        | DataTable (get)                  | Retrieves all past newsletters              | Set topics and language               | Sort newsletters by newest first           | News Filtering & Frequency Guardrails • Get previous newsletters: Retrieves all past editions from the Newsletters table • Filter by date: Keeps only articles newer than the last newsletter • Frequency check: Stops execution if last newsletter was sent within minDaysBetween |
| Sort newsletters by newest first | Sort                            | Sorts newsletters descending by creation date | Get previous newsletters             | Get most recent newsletter, Limit previous newsletters to last 5 | See above                                                                                         |
| Get most recent newsletter      | Limit                           | Limits to the most recent newsletter        | Sort newsletters by newest first      | Combine articles with last newsletter metadata | See above                                                                                         |
| Limit previous newsletters to last 5 | Limit                        | Limits to last 5 newsletters for context    | Sort newsletters by newest first      | Aggregate previous newsletters into list   | See above                                                                                         |
| Aggregate previous newsletters into list | Aggregate                  | Aggregates last 5 newsletters into list     | Limit previous newsletters to last 5 | Combine candidate articles with past newsletters | See above                                                                                         |
| Combine articles with last newsletter metadata | Merge (combine)            | Combines new articles with last newsletter metadata | Get most recent newsletter, Upsert articles into News table | Filter articles newer than last newsletter | See above                                                                                         |
| Filter articles newer than last newsletter | Filter                      | Filters articles newer than last newsletter | Combine articles with last newsletter metadata | Stop if last newsletter is too recent      | See above                                                                                         |
| Stop if last newsletter is too recent | If                             | Stops process if last newsletter is recent  | Filter articles newer than last newsletter | Aggregate candidate articles for AI        | See above                                                                                         |
| Aggregate candidate articles for AI | Aggregate                    | Aggregates candidate articles for AI input  | Stop if last newsletter is too recent | Combine candidate articles with past newsletters | Newsletter Selection & Editorial AI • Aggregate articles: Bundles candidate articles with last 5 newsletters for context • AI: decide send + select articles: GPT-5.1 evaluates novelty, relevance, and value • Returns YES/NO decision • Selects top 3-5 articles when YES • Fallback rule applies if maxDaysBetween exceeded |
| Combine candidate articles with past newsletters | Merge (combine)            | Combines candidate articles with past newsletters | Aggregate candidate articles for AI, Aggregate previous newsletters into list | AI: decide send + select articles          | See above                                                                                         |
| AI: decide send + select articles | LangChain OpenAI (GPT-5.1)    | AI decides newsletter send and selects articles | Combine candidate articles with past newsletters | If AI decided to send newsletter           | See above                                                                                         |
| If AI decided to send newsletter | If                             | Routes flow based on AI decision to send/skip | AI: decide send + select articles    | Split selected articles for enrichment      | See above                                                                                         |
| Split selected articles for enrichment | SplitOut                    | Splits selected articles for individual enrichment | If AI decided to send newsletter     | AI: enrich & write article                  | Newsletter Writing & Delivery • Split selected articles: Processes each article individually • AI: enrich & write article: GPT-5.1 + Tavily research enhance each article • Insert into Newsletters table: Archives the edition • Send to Telegram: Delivers formatted newsletter |
| AI: enrich & write article      | LangChain Agent (GPT-5.1)       | Enriches and rewrites each article using AI and Tavily web search | Split selected articles for enrichment, Tavily web search, GPT-5.1 LM | Aggregate enriched articles, Parse enriched article JSON | See above                                                                                         |
| Tavily web search tool          | External AI Tool (Tavily)        | Provides web search results for AI enrichment | AI: enrich & write article          | AI: enrich & write article (tool input)    | See above                                                                                         |
| GPT-5.1 (AI language model)     | LangChain LM Chat OpenAI         | Provides GPT-5.1 language model support    | AI: enrich & write article           | AI: enrich & write article (language model) | See above                                                                                         |
| Parse enriched article JSON     | LangChain Output Parser Structured | Parses AI enriched article output into structured JSON | AI: enrich & write article           | AI: enrich & write article (parsed output) | See above                                                                                         |
| Aggregate enriched articles     | Aggregate                       | Aggregates all enriched articles into an array | AI: enrich & write article           | Insert newsletter content into Newsletters table | See above                                                                                         |
| Insert newsletter content into Newsletters table | DataTable (insert)          | Archives the composed newsletter content   | Aggregate enriched articles          | Send newsletter to Telegram                  | See above                                                                                         |
| Send newsletter to Telegram     | Telegram                        | Sends the final newsletter message          | Insert newsletter content into Newsletters table | None                                     | See above                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**
   - Type: ScheduleTrigger
   - Cron expression: `0 0 9,17 * * *` (runs daily at 9 AM and 5 PM)
   - Timezone: Europe/Berlin (UTC+01:00)
   - Connect output to "Set topics and language"

2. **Create Set node "Set topics and language":**
   - Define variables:
     - `topics` (string): e.g. `"AI, n8n"`
     - `language` (string): e.g. `"English"`
     - `minDaysBetween` (number): e.g. 2
     - `maxDaysBetween` (number): e.g. 5
   - Connect output to "Build topic queries" and "Get previous newsletters"

3. **Create Code node "Build topic queries":**
   - JavaScript code:
     ```js
     return $input.first().json.topics.split(',').map(topic => ({
       json: { topic: topic.trim() }
     }));
     ```
   - Connect output to "Fetch news from SerpAPI (DuckDuckGo News)"

4. **Create HTTP Request node "Fetch news from SerpAPI (DuckDuckGo News)":**
   - URL: `https://serpapi.com/search.html`
   - Method: GET
   - Query parameters:
     - `engine=duckduckgo_news`
     - `q` = `={{ $json.topic }}`
     - `df=d`
   - Authentication: HTTP Query Auth with SerpAPI credentials (add credentials)
   - Connect output to "Split SerpAPI results into articles"

5. **Create SplitOut node "Split SerpAPI results into articles":**
   - Field to split out: `results`
   - Connect output to "Upsert articles into News table"

6. **Create DataTable node "Upsert articles into News table":**
   - Operation: Upsert
   - Deduplicate by `title` and `url`
   - Columns mapping:
     - `url`: `$json.url`
     - `date`: convert UNIX timestamp `$json.date` to ISO string
     - `title`: `$json.title`
     - `source`: `$json.source`
     - `excerpt`: `$json.excerpt`
   - Connect output to "Combine articles with last newsletter metadata"

7. **Create DataTable node "Get previous newsletters":**
   - Operation: Get
   - Return all items
   - Data table: Newsletters table
   - Connect output to "Sort newsletters by newest first"

8. **Create Sort node "Sort newsletters by newest first":**
   - Sort by `createdAt` descending
   - Connect outputs to "Get most recent newsletter" and "Limit previous newsletters to last 5"

9. **Create Limit node "Get most recent newsletter":**
   - Limit to 1
   - Connect output to "Combine articles with last newsletter metadata"

10. **Create Limit node "Limit previous newsletters to last 5":**
    - Limit to 5
    - Connect output to "Aggregate previous newsletters into list"

11. **Create Aggregate node "Aggregate previous newsletters into list":**
    - Aggregate all item data into field `newsletters`
    - Connect output to "Combine candidate articles with past newsletters"

12. **Create Merge node "Combine articles with last newsletter metadata":**
    - Mode: Combine all
    - Clash handling: Add suffix
    - Connect inputs from "Get most recent newsletter" and "Upsert articles into News table"
    - Connect output to "Filter articles newer than last newsletter"

13. **Create Filter node "Filter articles newer than last newsletter":**
    - Condition: article date > last newsletter createdAt (or fallback date)
    - Connect output to "Stop if last newsletter is too recent"

14. **Create If node "Stop if last newsletter is too recent":**
    - Condition: last newsletter createdAt < now - `minDaysBetween`
    - If true, continue to "Aggregate candidate articles for AI"
    - If false, end workflow (stop)

15. **Create Aggregate node "Aggregate candidate articles for AI":**
    - Aggregate specified fields: `title`, `excerpt`, `source`, `url` into `results`
    - Connect output to "Combine candidate articles with past newsletters"

16. **Create Merge node "Combine candidate articles with past newsletters":**
    - Mode: Combine all
    - Connect inputs from "Aggregate candidate articles for AI" and "Aggregate previous newsletters into list"
    - Connect output to "AI: decide send + select articles"

17. **Create LangChain OpenAI node "AI: decide send + select articles":**
    - Model: GPT-5.1
    - Prompt: AI Newsletter Editor with detailed instructions and input template including articles and past newsletters
    - Output: JSON with `decision` and `articles`
    - Connect output to "If AI decided to send newsletter"

18. **Create If node "If AI decided to send newsletter":**
    - Condition: decision == "YES"
    - If true, continue to "Split selected articles for enrichment"
    - If false, end workflow

19. **Create SplitOut node "Split selected articles for enrichment":**
    - Field to split: `message.content.articles`
    - Connect output to "AI: enrich & write article"

20. **Create LangChain Agent node "AI: enrich & write article":**
    - Role: Research writer that enriches and rewrites each article
    - Uses:
      - Tavily web search tool (add as AI tool)
      - GPT-5.1 model (add as AI language model)
    - Input: article JSON with title, summary, source, url, language
    - Output: structured JSON parsed by "Parse enriched article JSON"
    - Connect output to "Aggregate enriched articles"

21. **Create Tavily web search tool node:**
    - Used by AI agent for web research
    - Connect to AI agent node as AI tool

22. **Create GPT-5.1 LangChain LM Chat OpenAI node:**
    - Used by AI agent for language model completions
    - Connect to AI agent node as AI language model

23. **Create LangChain Output Parser Structured node "Parse enriched article JSON":**
    - Schema: object with `title`, `content`, `source`, `url` (all strings)
    - Connect output to "AI: enrich & write article" (parsed output)

24. **Create Aggregate node "Aggregate enriched articles":**
    - Aggregate field: `output` (enriched article JSONs)
    - Connect output to "Insert newsletter content into Newsletters table"

25. **Create DataTable node "Insert newsletter content into Newsletters table":**
    - Insert operation
    - Content: Markdown formatted concatenation of articles (`*title*\ncontent\nSource: [source](url)`)
    - Connect output to "Send newsletter to Telegram"

26. **Create Telegram node "Send newsletter to Telegram":**
    - Chat ID: configured chat ID (e.g., `89066090`)
    - Text: Markdown formatted newsletter from aggregated articles
    - Credentials: Telegram Bot credentials
    - Final node; no outputs

27. **Add Sticky Notes at appropriate sections** for documentation and clarity (optional)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| This workflow uses SerpAPI's DuckDuckGo News engine for fetching recent news articles filtered to last 24 hours.                                                                                                                                                                                                                                                                                                                                                                | https://serpapi.com                                                 |
| AI editorial decisions and article enrichment rely on GPT-5.1 via OpenAI API, requiring appropriate API key setup.                                                                                                                                                                                                                                                                                                                                                              | https://platform.openai.com/docs/models/gpt-5                      |
| Tavily web search tool is integrated to provide grounded, recent web search results to the AI agent during article enrichment.                                                                                                                                                                                                                                                                                                                                                   | https://tavily.com                                                  |
| Telegram Bot credentials and the target chat ID must be configured for delivering newsletters. The chat ID should be a valid Telegram chat where the bot has permissions to post messages.                                                                                                                                                                                                                                                                                        | https://core.telegram.org/bots/api                                 |
| Frequency guardrails (`minDaysBetween` and `maxDaysBetween`) help avoid spamming users with newsletters and ensure relevant cadence. The AI fallback rule forces sending if no newsletter in maxDaysBetween despite other criteria.                                                                                                                                                                                                                                              | Configured in "Set topics and language" node                       |
| The workflow stores articles and newsletters in n8n internal Data Tables for deduplication and archival. Proper permissions and capacity should be ensured for these data tables.                                                                                                                                                                                                                                                                                                 | n8n Data Tables documentation                                      |
| The workflow includes comprehensive error handling via "If" and "Filter" nodes to avoid unnecessary runs and redundant newsletters.                                                                                                                                                                                                                                                                                                                                            | See nodes "Stop if last newsletter is too recent" and filters     |
| Sticky notes are added throughout the workflow for user guidance and documentation.                                                                                                                                                                                                                                                                                                                                                                                             | Visible in n8n editor                                              |

---

**Disclaimer:**  
This document is based exclusively on the provided n8n workflow JSON. It adheres to content policies and contains no illegal or protected material. All data handled is legal and public.

---