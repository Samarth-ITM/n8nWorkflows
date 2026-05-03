Write daily tech news with OpenAI using RSS feeds and WordPress

https://n8nworkflows.xyz/workflows/write-daily-tech-news-with-openai-using-rss-feeds-and-wordpress-13924


# Write daily tech news with OpenAI using RSS feeds and WordPress

# 1. Workflow Overview

This workflow generates a daily tech news post automatically from RSS feeds, uses an OpenAI-powered agent to select and synthesize the most relevant story, and then saves the result as a draft in WordPress.

Its main use case is automated editorial content production for tech blogs, newsletters, or media sites that want one curated article per day with minimal manual work.

The workflow is organized into four logical blocks:

## 1.1 Scheduled Input Reception
A daily schedule trigger starts the workflow at 8:00 AM and launches four RSS feed readers in parallel.

## 1.2 Feed Consolidation and Article Selection Prep
The workflow merges all RSS items, normalizes inconsistent feed fields, filters to articles published in the last 24 hours, sorts by recency, and limits the candidate pool.

## 1.3 AI-Based Story Research and Writing
The selected RSS digest is compiled into a single prompt input. An AI agent reviews the digest, optionally fetches full article text through a custom tool, and writes an original HTML-formatted news article.

## 1.4 Output Parsing and WordPress Publishing
The AI output is split into title and content, then posted to WordPress as a draft.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Input Reception

### Overview
This block starts the workflow every day at 8:00 AM and retrieves articles from four tech news RSS feeds in parallel. It acts as the ingestion layer for the whole automation.

### Nodes Involved
- Daily 8AM Trigger
- RSS BBC Tech
- RSS The Verge
- RSS Ars Technica
- RSS TechCrunch

### Node Details

#### Daily 8AM Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point.
- **Configuration choices:** Configured with an interval rule that triggers daily at hour 8.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to all four RSS nodes in parallel.
- **Version-specific requirements:** Uses type version `1.2`.
- **Edge cases or potential failure types:**
  - Server timezone affects actual execution time.
  - If the n8n instance is down at trigger time, execution may be missed depending on deployment behavior.
- **Sub-workflow reference:** None.

#### RSS BBC Tech
- **Type and technical role:** `n8n-nodes-base.rssFeedRead`; fetches and parses an RSS feed.
- **Configuration choices:** Reads `https://feeds.bbci.co.uk/news/technology/rss.xml`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from Daily 8AM Trigger; output to Merge All Feeds input 0.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Feed URL unavailable or rate-limited.
  - Feed schema may differ from expected fields.
  - Network/DNS/SSL errors.
- **Sub-workflow reference:** None.

#### RSS The Verge
- **Type and technical role:** `n8n-nodes-base.rssFeedRead`; fetches and parses an RSS feed.
- **Configuration choices:** Reads `https://www.theverge.com/rss/index.xml`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from Daily 8AM Trigger; output to Merge All Feeds input 1.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - XML feed changes.
  - Temporary upstream outage.
- **Sub-workflow reference:** None.

#### RSS Ars Technica
- **Type and technical role:** `n8n-nodes-base.rssFeedRead`; fetches and parses an RSS feed.
- **Configuration choices:** Reads `https://feeds.arstechnica.com/arstechnica/index`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from Daily 8AM Trigger; output to Merge All Feeds input 2.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Feed entries may use different metadata naming conventions.
  - Temporary HTTP failure.
- **Sub-workflow reference:** None.

#### RSS TechCrunch
- **Type and technical role:** `n8n-nodes-base.rssFeedRead`; fetches and parses an RSS feed.
- **Configuration choices:** Reads `https://techcrunch.com/feed/`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from Daily 8AM Trigger; output to Merge All Feeds input 3.
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Feed access restrictions or timeout.
  - Some articles may have incomplete content fields.
- **Sub-workflow reference:** None.

---

## 2.2 Feed Consolidation and Article Selection Prep

### Overview
This block unifies the four incoming RSS streams into one dataset and prepares it for AI consumption. It standardizes key fields, keeps only recent content, sorts by recency, and reduces the number of candidates.

### Nodes Involved
- Merge All Feeds
- Normalize Fields
- Filter Last 24 Hours
- Sort by Date
- Limit to 15

### Node Details

#### Merge All Feeds
- **Type and technical role:** `n8n-nodes-base.merge`; combines multiple data inputs into one stream.
- **Configuration choices:** Configured for `numberInputs: 4`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Inputs from all four RSS nodes; output to Normalize Fields.
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - If one upstream feed returns no items, the merged output may still continue depending on node behavior, but article volume drops.
  - If one feed node errors hard, the workflow may stop unless error handling is added.
- **Sub-workflow reference:** None.

#### Normalize Fields
- **Type and technical role:** `n8n-nodes-base.set`; remaps fields into a consistent schema.
- **Configuration choices:** Creates these normalized fields:
  - `title` from `$json.title`
  - `snippet` from `$json.contentSnippet || $json.content || ''`
  - `link` from `$json.link`
  - `isoDate` as Unix timestamp from `new Date($json.isoDate || $json.pubDate || 0).getTime()`
  - `source` from `$json.creator || $json['dc:creator'] || ''`
- **Key expressions or variables used:**
  - `={{ $json.title }}`
  - `={{ $json.contentSnippet || $json.content || '' }}`
  - `={{ $json.link }}`
  - `={{ new Date($json.isoDate || $json.pubDate || 0).getTime() }}`
  - `={{ $json.creator || $json['dc:creator'] || '' }}`
- **Input and output connections:** Input from Merge All Feeds; output to Filter Last 24 Hours.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Invalid or missing publication dates become `0`, which will later be filtered out.
  - HTML-heavy `content` fields may pass through as raw text/html fragments.
  - `source` here refers to creator metadata, not the publication name.
- **Sub-workflow reference:** None.

#### Filter Last 24 Hours
- **Type and technical role:** `n8n-nodes-base.filter`; excludes stale items.
- **Configuration choices:** Keeps only items where `isoDate` is greater than `Date.now() - 24 * 60 * 60 * 1000`.
- **Key expressions or variables used:**
  - Left value: `={{ $json.isoDate }}`
  - Right value: `={{ Date.now() - 24 * 60 * 60 * 1000 }}`
- **Input and output connections:** Input from Normalize Fields; output to Sort by Date.
- **Version-specific requirements:** Type version `2.2`, condition system version `2`.
- **Edge cases or potential failure types:**
  - If feed dates are in unexpected formats, valid content may be excluded.
  - If no articles match the last 24 hours, all downstream nodes receive zero items.
- **Sub-workflow reference:** None.

#### Sort by Date
- **Type and technical role:** `n8n-nodes-base.sort`; orders items.
- **Configuration choices:** Sorts descending by `isoDate`.
- **Key expressions or variables used:** Field name `isoDate`.
- **Input and output connections:** Input from Filter Last 24 Hours; output to Limit to 15.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - If `isoDate` is missing or malformed, ordering may be inconsistent.
- **Sub-workflow reference:** None.

#### Limit to 15
- **Type and technical role:** `n8n-nodes-base.limit`; caps the number of items forwarded.
- **Configuration choices:** Despite the node name, it is configured with `maxItems: 10`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from Sort by Date; output to Aggregate Articles.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Naming mismatch can confuse maintenance: the node says “15” but actually limits to 10.
  - If fewer than 10 items exist, all pass through.
- **Sub-workflow reference:** None.

---

## 2.3 AI-Based Story Research and Writing

### Overview
This block compresses the selected RSS items into a digest, passes them to an AI agent, and allows the agent to fetch full article text from URLs before writing an original tech news post.

### Nodes Involved
- Aggregate Articles
- OpenAI Chat Model
- Fetch Article
- AI Write News Article

### Node Details

#### Aggregate Articles
- **Type and technical role:** `n8n-nodes-base.code`; aggregates multiple RSS items into one prompt-ready text payload.
- **Configuration choices:** Uses JavaScript to collect all input items and build a formatted digest string named `articlesSummary`. Also outputs `articleCount`.
- **Key expressions or variables used:**
  - `$input.all()`
  - Per-item access through `articles[i].json`
- **Output structure:**
  - `articlesSummary`: textual digest listing title, URL, and truncated snippet
  - `articleCount`: number of included articles
- **Input and output connections:** Input from Limit to 15; output to AI Write News Article.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If zero items arrive, it still outputs a digest saying `0 articles`, which may produce weak or invalid AI output.
  - Snippets are truncated to 250 characters in code.
- **Sub-workflow reference:** None.

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; language model backend for the AI agent.
- **Configuration choices:** Uses model `gpt-5.1`. No extra options or built-in tools are configured.
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected to AI Write News Article via `ai_languageModel`.
- **Version-specific requirements:** Type version `1.3`; requires compatible n8n LangChain/OpenAI node support and valid OpenAI credentials.
- **Edge cases or potential failure types:**
  - Invalid API key or insufficient quota.
  - Model availability differences by account or region.
  - Latency or token-limit issues depending on fetched article length.
- **Sub-workflow reference:** None.

#### Fetch Article
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolCode`; custom AI tool that fetches and strips article web pages into readable text.
- **Configuration choices:**
  - Tool name exposed internally as `my_tool`
  - Accepts a URL via `query`
  - Performs HTTP GET with 15-second timeout
  - Removes scripts, styles, nav, footer, header, aside, noscript, iframe, comments
  - Converts some block-level tags to line breaks
  - Strips remaining HTML
  - Decodes common HTML entities
  - Truncates text at 4000 characters
  - Returns explicit error text instead of throwing when fetch/parsing fails
- **Key expressions or variables used:**
  - `const url = query.trim();`
  - `await this.helpers.httpRequest(...)`
- **Input and output connections:** Connected to AI Write News Article via `ai_tool`.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Paywalled or JavaScript-rendered pages may return low-quality text.
  - Some publishers block scraping or return anti-bot responses.
  - Relative URLs are rejected.
  - Text extraction may include noise because it is regex-based, not DOM-aware.
  - Returns human-readable `"ERROR: ..."` strings; the agent must interpret them correctly.
- **Sub-workflow reference:** None.

#### AI Write News Article
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; AI agent that interprets the digest, uses tools if needed, and generates the final article.
- **Configuration choices:**
  - Prompt type: defined manually
  - Receives `{{ $json.articlesSummary }}`
  - Instructed to:
    - pick one major story or combine 2–3 related ones
    - use the Fetch Article tool
    - fetch no more than 3 URLs
    - skip failed fetches
    - write a 600–1000 word original article
    - format output as:
      - `TITLE: ...`
      - `---CONTENT---`
      - HTML article body
  - Requires use of `<h2>`, `<p>`, `<strong>`, `<em>`
  - Must not include full HTML document tags
- **Key expressions or variables used:**
  - `{{ $json.articlesSummary }}`
- **Input and output connections:**
  - Main input from Aggregate Articles
  - AI language model input from OpenAI Chat Model
  - AI tool input from Fetch Article
  - Main output to Parse AI Output
- **Version-specific requirements:** Type version `1.7`.
- **Edge cases or potential failure types:**
  - The model may not always respect the exact output format.
  - If no articles are provided, it may hallucinate or produce generic content.
  - Tool calls may fail or be skipped.
  - HTML output may contain malformed tags.
  - Prompt compliance varies by model behavior.
- **Sub-workflow reference:** None.

---

## 2.4 Output Parsing and WordPress Publishing

### Overview
This block extracts the headline and HTML body from the AI response and creates a draft post in WordPress. It is the publishing layer of the workflow.

### Nodes Involved
- Parse AI Output
- Create WordPress Draft

### Node Details

#### Parse AI Output
- **Type and technical role:** `n8n-nodes-base.code`; converts the agent’s raw text response into structured fields.
- **Configuration choices:**
  - Reads `output` from the incoming item
  - Looks for `TITLE:`
  - Looks for `---CONTENT---`
  - Defaults title to `Untitled` if missing
  - Defaults content to full raw output if marker not found
- **Key expressions or variables used:**
  - `$input.item.json.output`
  - String parsing with `indexOf`, `substring`, and `trim`
- **Output structure:**
  - `title`
  - `content`
- **Input and output connections:** Input from AI Write News Article; output to Create WordPress Draft.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If the AI output format changes, parsing may degrade silently.
  - If the model emits extra text before `TITLE:`, title extraction still works only if marker exists.
  - If content marker is absent, all output becomes post content.
- **Sub-workflow reference:** None.

#### Create WordPress Draft
- **Type and technical role:** `n8n-nodes-base.wordpress`; creates a WordPress post.
- **Configuration choices:**
  - Title from `={{ $json.title }}`
  - Content from `={{ $json.content }}`
  - Post status set to `draft`
- **Key expressions or variables used:**
  - `={{ $json.title }}`
  - `={{ $json.content }}`
- **Input and output connections:** Input from Parse AI Output; no downstream node.
- **Version-specific requirements:** Type version `1`; requires WordPress credentials configured in n8n.
- **Edge cases or potential failure types:**
  - Authentication failure due to wrong application password or site URL.
  - WordPress REST API disabled or blocked.
  - HTML may be sanitized or rejected depending on WordPress/site security plugins.
  - Empty title/content may still create unusable drafts or trigger API validation errors.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily 8AM Trigger | n8n-nodes-base.scheduleTrigger | Starts the workflow every day at 8 AM |  | RSS BBC Tech, RSS The Verge, RSS Ars Technica, RSS TechCrunch | ## 1. Fetch RSS feeds<br>Pulls the latest articles from BBC Tech, The Verge, Ars Technica, and TechCrunch in parallel, triggered daily at 8 AM. |
| RSS BBC Tech | n8n-nodes-base.rssFeedRead | Reads BBC Technology RSS feed | Daily 8AM Trigger | Merge All Feeds | ## 1. Fetch RSS feeds<br>Pulls the latest articles from BBC Tech, The Verge, Ars Technica, and TechCrunch in parallel, triggered daily at 8 AM. |
| RSS The Verge | n8n-nodes-base.rssFeedRead | Reads The Verge RSS feed | Daily 8AM Trigger | Merge All Feeds | ## 1. Fetch RSS feeds<br>Pulls the latest articles from BBC Tech, The Verge, Ars Technica, and TechCrunch in parallel, triggered daily at 8 AM. |
| RSS Ars Technica | n8n-nodes-base.rssFeedRead | Reads Ars Technica RSS feed | Daily 8AM Trigger | Merge All Feeds | ## 1. Fetch RSS feeds<br>Pulls the latest articles from BBC Tech, The Verge, Ars Technica, and TechCrunch in parallel, triggered daily at 8 AM. |
| RSS TechCrunch | n8n-nodes-base.rssFeedRead | Reads TechCrunch RSS feed | Daily 8AM Trigger | Merge All Feeds | ## 1. Fetch RSS feeds<br>Pulls the latest articles from BBC Tech, The Verge, Ars Technica, and TechCrunch in parallel, triggered daily at 8 AM. |
| Merge All Feeds | n8n-nodes-base.merge | Combines the four RSS streams | RSS BBC Tech, RSS The Verge, RSS Ars Technica, RSS TechCrunch | Normalize Fields |  |
| Normalize Fields | n8n-nodes-base.set | Standardizes article fields across feeds | Merge All Feeds | Filter Last 24 Hours | ## 2. Normalize and filter<br>Standardizes fields across feeds, drops articles older than 24 hours, sorts by date, and caps the list at 10 items. |
| Filter Last 24 Hours | n8n-nodes-base.filter | Keeps only recent articles | Normalize Fields | Sort by Date | ## 2. Normalize and filter<br>Standardizes fields across feeds, drops articles older than 24 hours, sorts by date, and caps the list at 10 items. |
| Sort by Date | n8n-nodes-base.sort | Orders articles by newest first | Filter Last 24 Hours | Limit to 15 | ## 2. Normalize and filter<br>Standardizes fields across feeds, drops articles older than 24 hours, sorts by date, and caps the list at 10 items. |
| Limit to 15 | n8n-nodes-base.limit | Restricts candidate articles sent to the AI | Sort by Date | Aggregate Articles | ## 2. Normalize and filter<br>Standardizes fields across feeds, drops articles older than 24 hours, sorts by date, and caps the list at 10 items. |
| Aggregate Articles | n8n-nodes-base.code | Builds a digest summary for the AI | Limit to 15 | AI Write News Article | ## 3. AI writes the article<br>Compiles headlines into a digest, then an OpenAI agent picks the top story, fetches full text, and writes an original news article. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supplies the LLM used by the agent |  | AI Write News Article | ## 3. AI writes the article<br>Compiles headlines into a digest, then an OpenAI agent picks the top story, fetches full text, and writes an original news article. |
| Fetch Article | @n8n/n8n-nodes-langchain.toolCode | Tool used by the agent to retrieve article text from URLs |  | AI Write News Article | ## 3. AI writes the article<br>Compiles headlines into a digest, then an OpenAI agent picks the top story, fetches full text, and writes an original news article. |
| AI Write News Article | @n8n/n8n-nodes-langchain.agent | Selects story candidates and writes the final article | Aggregate Articles, OpenAI Chat Model, Fetch Article | Parse AI Output | ## 3. AI writes the article<br>Compiles headlines into a digest, then an OpenAI agent picks the top story, fetches full text, and writes an original news article. |
| Parse AI Output | n8n-nodes-base.code | Extracts title and HTML body from AI output | AI Write News Article | Create WordPress Draft | ## 4. Publish to WordPress<br>Extracts the headline and HTML body from the AI output and creates a WordPress draft post. |
| Create WordPress Draft | n8n-nodes-base.wordpress | Creates a draft post in WordPress | Parse AI Output |  | ## 4. Publish to WordPress<br>Extracts the headline and HTML body from the AI output and creates a WordPress draft post. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation / usage guidance |  |  | ## Try It Out!<br>### Automatically write and publish an original tech news article to WordPress every morning — completely hands-free.<br><br>Great for tech bloggers, content creators, or anyone who wants a daily news column without the daily effort.<br><br>### How it works<br>* A schedule trigger fires daily at 8 AM and pulls articles from four RSS feeds (BBC Tech, The Verge, Ars Technica, TechCrunch).<br>* Articles are merged, normalized, filtered to the last 24 hours, sorted, and capped at 10.<br>* An OpenAI-powered AI agent picks the most newsworthy story, fetches full article text, and writes an original 600–1000 word article.<br>* The output is parsed into a title and HTML body, then posted as a draft to WordPress.<br><br>### How to use<br>* Add your OpenAI API key under Credentials.<br>* Add your WordPress API credentials (site URL + application password).<br>* Optionally swap RSS feeds, adjust the article limit, or change the post status.<br>* Activate the workflow — it runs daily at 8 AM.<br><br>### Requirements<br>* OpenAI account and API key<br>* WordPress site with application password enabled<br><br>### Need Help?<br>Join the [Discord](https://discord.com/invite/XPKeKXeB7d) or ask in the [Forum](https://community.n8n.io/)!<br><br>Happy Hacking! |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas label for RSS ingestion block |  |  | ## 1. Fetch RSS feeds<br>Pulls the latest articles from BBC Tech, The Verge, Ars Technica, and TechCrunch in parallel, triggered daily at 8 AM. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas label for normalization/filter block |  |  | ## 2. Normalize and filter<br>Standardizes fields across feeds, drops articles older than 24 hours, sorts by date, and caps the list at 10 items. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas label for AI writing block |  |  | ## 3. AI writes the article<br>Compiles headlines into a digest, then an OpenAI agent picks the top story, fetches full text, and writes an original news article. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas label for WordPress publishing block |  |  | ## 4. Publish to WordPress<br>Extracts the headline and HTML body from the AI output and creates a WordPress draft post. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Name it: **Write daily tech news with AI using RSS and WordPress**.

2. **Add a Schedule Trigger node**.
   - Node type: `Schedule Trigger`
   - Name: `Daily 8AM Trigger`
   - Set it to run **daily at 8:00 AM**.
   - Confirm the instance timezone is the one you expect.

3. **Add four RSS Feed Read nodes** and connect all of them from the trigger.
   - Node type: `RSS Feed Read`

   Configure them as follows:
   1. `RSS BBC Tech`
      - URL: `https://feeds.bbci.co.uk/news/technology/rss.xml`
   2. `RSS The Verge`
      - URL: `https://www.theverge.com/rss/index.xml`
   3. `RSS Ars Technica`
      - URL: `https://feeds.arstechnica.com/arstechnica/index`
   4. `RSS TechCrunch`
      - URL: `https://techcrunch.com/feed/`

   Connection pattern:
   - `Daily 8AM Trigger` → all four RSS nodes

4. **Add a Merge node**.
   - Name: `Merge All Feeds`
   - Node type: `Merge`
   - Set **number of inputs = 4**
   - Connect:
     - `RSS BBC Tech` → input 1
     - `RSS The Verge` → input 2
     - `RSS Ars Technica` → input 3
     - `RSS TechCrunch` → input 4

5. **Add a Set node** for normalization.
   - Name: `Normalize Fields`
   - Node type: `Set`
   - Create these fields:

   - `title` as String:
     - `{{ $json.title }}`
   - `snippet` as String:
     - `{{ $json.contentSnippet || $json.content || '' }}`
   - `link` as String:
     - `{{ $json.link }}`
   - `isoDate` as Number:
     - `{{ new Date($json.isoDate || $json.pubDate || 0).getTime() }}`
   - `source` as String:
     - `{{ $json.creator || $json['dc:creator'] || '' }}`

   Connect:
   - `Merge All Feeds` → `Normalize Fields`

6. **Add a Filter node** to keep only recent articles.
   - Name: `Filter Last 24 Hours`
   - Node type: `Filter`
   - Create one condition:
     - Left value: `{{ $json.isoDate }}`
     - Operator: **Number >**
     - Right value: `{{ Date.now() - 24 * 60 * 60 * 1000 }}`
   - Use strict type validation if available.

   Connect:
   - `Normalize Fields` → `Filter Last 24 Hours`

7. **Add a Sort node**.
   - Name: `Sort by Date`
   - Node type: `Sort`
   - Sort field:
     - Field name: `isoDate`
     - Order: `Descending`

   Connect:
   - `Filter Last 24 Hours` → `Sort by Date`

8. **Add a Limit node**.
   - Name it either:
     - `Limit to 15` to match the original, or preferably
     - `Limit to 10` to match actual behavior
   - Node type: `Limit`
   - Set **max items = 10**

   Connect:
   - `Sort by Date` → `Limit to 15`

9. **Add a Code node** to aggregate the articles.
   - Name: `Aggregate Articles`
   - Node type: `Code`
   - Language: JavaScript
   - Paste this logic conceptually:
     - Read all incoming items
     - Build a text digest listing article count, title, URL, and first ~250 chars of snippet
     - Return one single item with:
       - `articlesSummary`
       - `articleCount`

   Equivalent code behavior:
   - Loop over `$input.all()`
   - Build:
     - `TODAY'S TOP NEWS ARTICLES (X articles from the last 24 hours):`
   - Return one JSON object

   Connect:
   - `Limit to 15` → `Aggregate Articles`

10. **Add an OpenAI Chat Model node**.
    - Name: `OpenAI Chat Model`
    - Node type: `OpenAI Chat Model`
    - Model: `gpt-5.1`
    - Credentials: add your **OpenAI API key**
    - Leave other options at default unless you need temperature or token controls.

11. **Add a Tool Code node** for article retrieval.
    - Name: `Fetch Article`
    - Node type: `Tool Code`
    - This node will be connected as a tool to the AI agent.
    - Set tool code to:
      - accept a URL from `query`
      - validate that it starts with `http://` or `https://`
      - request the page with a 15-second timeout
      - strip tags and noisy sections
      - truncate output at 4000 chars
      - return readable text or an `ERROR:` string

    Core implementation requirements:
    - Use `await this.helpers.httpRequest(...)`
    - Remove scripts/styles/navigation blocks
    - Strip HTML
    - Normalize whitespace
    - Return fallback errors rather than throwing

12. **Add an AI Agent node**.
    - Name: `AI Write News Article`
    - Node type: `AI Agent`
    - Prompt mode: define prompt manually
    - Main input should come from `Aggregate Articles`
    - Use the digest variable:
      - `{{ $json.articlesSummary }}`

    Configure the prompt so the agent:
    - acts as a professional tech journalist
    - reviews the digest
    - picks the most important story, or combines 2–3 related stories
    - uses the Fetch Article tool for full text
    - fetches no more than 3 URLs
    - skips failed fetches
    - writes an original 600–1000 word article
    - outputs exactly:

      `TITLE: [headline]`

      `---CONTENT---`

      `[HTML article body]`

    Include formatting instructions:
    - use `<h2>`, `<p>`, `<strong>`, `<em>`
    - do not output `<html>`, `<head>`, or `<body>`

13. **Wire the AI components together**.
    - Main connection:
      - `Aggregate Articles` → `AI Write News Article`
    - AI model connection:
      - `OpenAI Chat Model` → `AI Write News Article` using the **language model** connector
    - Tool connection:
      - `Fetch Article` → `AI Write News Article` using the **tool** connector

14. **Add a Code node** to parse the AI response.
    - Name: `Parse AI Output`
    - Node type: `Code`
    - Logic:
      - Read `output` from the agent node
      - Extract the first line after `TITLE:`
      - Extract everything after `---CONTENT---`
      - If no title exists, default to `Untitled`
      - If no content marker exists, use the entire output as content

    Expected output fields:
    - `title`
    - `content`

    Connect:
    - `AI Write News Article` → `Parse AI Output`

15. **Add a WordPress node**.
    - Name: `Create WordPress Draft`
    - Node type: `WordPress`
    - Operation: create post
    - Credentials:
      - WordPress site URL
      - WordPress username if needed by your credential type
      - Application password / API auth as supported by n8n
    - Title:
      - `{{ $json.title }}`
    - Content:
      - `{{ $json.content }}`
    - Status:
      - `draft`

    Connect:
    - `Parse AI Output` → `Create WordPress Draft`

16. **Configure credentials**.
    - **OpenAI credentials**
      - Add API key in n8n credentials.
      - Ensure the selected model is available on your account.
    - **WordPress credentials**
      - Use the correct site URL.
      - Ensure the WordPress REST API is enabled.
      - Generate and use an application password if required.

17. **Optionally add visual sticky notes** for documentation.
    - A general project note with setup guidance.
    - A note for RSS fetching.
    - A note for normalization/filtering.
    - A note for AI writing.
    - A note for WordPress publishing.

18. **Test the workflow manually**.
    - Run the trigger manually or execute from a later node.
    - Validate:
      - RSS feeds return items
      - `isoDate` is numeric
      - the filter leaves some items
      - the aggregate node outputs one item
      - the AI agent respects title/content format
      - WordPress draft is created correctly

19. **Activate the workflow** once validated.
    - It will then run automatically every day at 8:00 AM.

### Reproduction Notes and Constraints
- There is **no sub-workflow** in this design.
- The custom tool is internal to the AI agent, not a separate workflow.
- The node named `Limit to 15` actually limits to **10** items; if rebuilding, decide whether to preserve that mismatch or rename it for clarity.
- If you want publication instead of draft creation, change the WordPress status from `draft` to `publish`, but this removes editorial review.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically write and publish an original tech news article to WordPress every morning — completely hands-free. | General workflow purpose |
| Great for tech bloggers, content creators, or anyone who wants a daily news column without the daily effort. | Use case |
| Add your OpenAI API key under Credentials. | Setup |
| Add your WordPress API credentials (site URL + application password). | Setup |
| Optionally swap RSS feeds, adjust the article limit, or change the post status. | Customization |
| Activate the workflow — it runs daily at 8 AM. | Operations |
| OpenAI account and API key required. | Requirement |
| WordPress site with application password enabled. | Requirement |
| Join the Discord | https://discord.com/invite/XPKeKXeB7d |
| Ask in the Forum | https://community.n8n.io/ |

## Additional Implementation Notes
- The workflow has a **single entry point**: `Daily 8AM Trigger`.
- There are **no sub-workflows**.
- The AI agent depends on both a connected model node and a connected tool node.
- The workflow is currently **inactive** in the provided export (`active: false`).
- Execution order is configured as **v1**.
- Binary data mode is set to **separate**, though binary handling is not used by this workflow.
- Because the article fetch tool uses simple HTML stripping rather than a browser renderer, some modern websites may yield incomplete text.