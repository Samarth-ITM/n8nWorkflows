Curate tech news from RSS with OpenAI, Google Sheets and Slack

https://n8nworkflows.xyz/workflows/curate-tech-news-from-rss-with-openai--google-sheets-and-slack-14702


# Curate tech news from RSS with OpenAI, Google Sheets and Slack

# 1. Workflow Overview

## Purpose
This workflow curates recent technology news from an RSS feed, filters for emerging-tech topics, removes duplicates using Google Sheets, asks OpenAI to score each article and generate a tweet, keeps only the strongest items, posts them to Slack, and logs them back to Google Sheets.

## Target use cases
- Daily monitoring of tech-news sources such as TechCrunch
- Automated social-content generation for internal teams or social managers
- Low-noise news curation focused on high-innovation topics
- Deduplicated article tracking across repeated scheduled runs

## Logical blocks

### 1.1 Scheduled Input Reception
The workflow starts on a schedule and fetches articles from the TechCrunch RSS feed.

### 1.2 Recency Filtering and Link Normalization
Only articles published in the last 72 hours are kept, and article links are normalized to make deduplication more reliable.

### 1.3 Deduplication Against Google Sheets
The workflow checks previously logged links in Google Sheets and removes items that were already processed.

### 1.4 Topic Filtering and Prioritization
Remaining articles are filtered to only include selected emerging-tech keywords, then sorted by publish date and reduced to the three most recent items.

### 1.5 AI Scoring and Tweet Generation
Each shortlisted article is transformed into a compact AI input structure and sent to OpenAI, which returns a JSON object containing an innovation score and tweet draft.

### 1.6 AI/RSS Merge and Output Normalization
The workflow merges AI output with the preserved RSS metadata, validates the AI response shape, and produces a clean final data structure.

### 1.7 Quality Threshold and Ranking
Only items with score 8 or higher are kept, then sorted by score and limited to the top two posts.

### 1.8 Distribution and Tracking
The final posts are sent to a Slack channel and appended to Google Sheets for future deduplication and auditability.

### 1.9 Explicit Drop Paths
Several no-op nodes document discarded branches: old articles, non-tech-related content, and low-scoring articles.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Input Reception

### Overview
This block triggers the workflow once per day and retrieves the RSS entries from TechCrunch. It is the only entry point in the workflow.

### Nodes Involved
- Schedule Trigger
- Fetch RSS - TechCrunch

### Node Details

#### Schedule Trigger
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; starts the workflow automatically on a recurring schedule.
- **Configuration choices:** Configured with an interval rule that triggers at hour `7`.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Fetch RSS - TechCrunch**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or failure types:**
  - Timezone-sensitive execution; workflow timezone is `Asia/Manila`, so 7 means 7 AM Manila time.
  - If the workflow is inactive, no scheduled execution occurs.
- **Sub-workflow reference:** None.

#### Fetch RSS - TechCrunch
- **Type and role:** `n8n-nodes-base.rssFeedRead`; reads article items from the RSS feed.
- **Configuration choices:** Reads `https://techcrunch.com/feed/`; SSL validation is not ignored.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Schedule Trigger**; output to **Filter - Published Within 72h**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or failure types:**
  - Feed unavailable or temporary network failure
  - Invalid XML or feed parsing issues
  - Site-side throttling or access restrictions
- **Sub-workflow reference:** None.

---

## 2.2 Recency Filtering and Link Normalization

### Overview
This block excludes stale articles and standardizes URLs by removing trailing slashes. That normalization is important because the workflow later compares links against Google Sheets.

### Nodes Involved
- Filter - Published Within 72h
- Normalize Link
- Old Article

### Node Details

#### Filter - Published Within 72h
- **Type and role:** `n8n-nodes-base.if`; filters feed items based on recency.
- **Configuration choices:** Uses a boolean condition that evaluates whether `isoDate` is newer than `Date.now() - 72 hours`.
- **Key expressions or variables used:**
  - `={{ new Date($json.isoDate) > new Date(Date.now() - 72 * 60 * 60 * 1000) }}`
- **Input and output connections:** Input from **Fetch RSS - TechCrunch**; true output to **Normalize Link**; false output to **Old Article**.
- **Version-specific requirements:** Type version `2.3`, conditions version `3`.
- **Edge cases or failure types:**
  - Missing or malformed `isoDate`
  - Timezone interpretation differences in source feed
  - Strict type validation can expose unexpected field formats
- **Sub-workflow reference:** None.

#### Normalize Link
- **Type and role:** `n8n-nodes-base.code`; removes a trailing slash from each article link.
- **Configuration choices:** Runs JavaScript over all items and rewrites `item.json.link`.
- **Key expressions or variables used:**
  - `item.json.link = item.json.link.replace(/\/$/, '')`
- **Input and output connections:** Input from **Filter - Published Within 72h** true branch; output to **Check Google sheet**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - Fails if `link` is missing or not a string
  - Only removes one final slash; does not normalize tracking parameters or casing
- **Sub-workflow reference:** None.

#### Old Article
- **Type and role:** `n8n-nodes-base.noOp`; terminal branch for outdated items.
- **Configuration choices:** No parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Filter - Published Within 72h** false branch; no downstream nodes.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None operational; used for visual clarity only.
- **Sub-workflow reference:** None.

---

## 2.3 Deduplication Against Google Sheets

### Overview
This block checks whether normalized links already exist in the tracking sheet and removes any duplicates from the current batch. It prevents reposting previously processed articles.

### Nodes Involved
- Check Google sheet
- Remove Already Posted Links

### Node Details

#### Check Google sheet
- **Type and role:** `n8n-nodes-base.googleSheets`; looks up matching rows in Google Sheets using the current article link.
- **Configuration choices:**
  - Uses Google Sheets OAuth2 credentials
  - Targets spreadsheet `techcrunch_rss_feed`
  - Targets sheet/tab `techcrunch`
  - Applies a filter where column `link` equals `{{$json.link}}`
  - `alwaysOutputData` is enabled, so execution continues even when no row matches
- **Key expressions or variables used:**
  - Lookup value: `={{ $json.link }}`
- **Input and output connections:** Input from **Normalize Link**; output to **Remove Already Posted Links**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or failure types:**
  - OAuth permission or expired token issues
  - Spreadsheet or sheet ID changed
  - Missing `link` column in target sheet
  - Rate limits for high-volume usage
- **Sub-workflow reference:** None.

#### Remove Already Posted Links
- **Type and role:** `n8n-nodes-base.code`; compares current normalized links to links returned by the Google Sheets lookup and removes duplicates.
- **Configuration choices:** Reads items from both **Check Google sheet** and **Normalize Link** using `$items(...)`.
- **Key expressions or variables used:**
  - `const existingLinks = $items("Check Google sheet").map(i => i.json.link?.replace(/\/$/, '')).filter(Boolean);`
  - `return $items("Normalize Link").filter(item => !existingLinks.includes(cleanLink));`
- **Input and output connections:** Triggered by **Check Google sheet**; output to **Filter - Tech Keywords**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - Depends on cross-node item access by node name; renaming nodes can break logic
  - If Google Sheets lookup returns partial or unexpected data, duplicates may pass through
  - Assumes links are present in both source and sheet rows
- **Sub-workflow reference:** None.

---

## 2.4 Topic Filtering and Prioritization

### Overview
This block keeps only articles whose titles contain selected emerging-tech keywords, then prioritizes the most recent ones and narrows the batch before sending items to AI.

### Nodes Involved
- Filter - Tech Keywords
- Sort by Publish Date
- Preserve RSS Data
- Not Tech Related

### Node Details

#### Filter - Tech Keywords
- **Type and role:** `n8n-nodes-base.if`; keeps only articles with target keywords in the title.
- **Configuration choices:** Evaluates one boolean expression against the lowercase title. Keywords include:
  - ai
  - artificial intelligence
  - cybersecurity
  - robotics
  - quantum
  - blockchain
  - machine learning
- **Key expressions or variables used:**
  - `={{$json.title.toLowerCase().includes("ai") || ... }}`
- **Input and output connections:** Input from **Remove Already Posted Links**; true output to **Sort by Publish Date**; false output to **Not Tech Related**.
- **Version-specific requirements:** Type version `2.3`, conditions version `3`.
- **Edge cases or failure types:**
  - Fails if `title` is missing
  - Keyword matching is simplistic and may create false positives, especially `"ai"` inside unrelated words
  - Title-only filtering may miss relevant articles whose keywords are in summary/body only
- **Sub-workflow reference:** None.

#### Sort by Publish Date
- **Type and role:** `n8n-nodes-base.code`; sorts matching articles by descending publish date and keeps only the top three.
- **Configuration choices:** Uses JavaScript sort on `pubDate`.
- **Key expressions or variables used:**
  - `items.sort((a, b) => new Date(b.json.pubDate) - new Date(a.json.pubDate));`
  - `return items.slice(0, 3);`
- **Input and output connections:** Input from **Filter - Tech Keywords** true branch; output to **Preserve RSS Data**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - Invalid or missing `pubDate`
  - If fewer than three items exist, all are returned
- **Sub-workflow reference:** None.

#### Preserve RSS Data
- **Type and role:** `n8n-nodes-base.code`; reshapes each item so full RSS data is preserved under `rss`, while exposing a concise AI prompt payload.
- **Configuration choices:** Stores:
  - `rss`: full original item JSON
  - `title`
  - `summary`: from `contentSnippet`
  - `categories`
- **Key expressions or variables used:**
  - `rss: item.json`
  - `summary: item.json.contentSnippet`
- **Input and output connections:** Input from **Sort by Publish Date**; outputs in parallel to **AI - Score & Generate Tweet** and **Merge AI + RSS (By Position)** input 1.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - If `contentSnippet` or `categories` is absent, AI prompt quality may decrease
  - This node is essential for positional merging later; item order must remain consistent
- **Sub-workflow reference:** None.

#### Not Tech Related
- **Type and role:** `n8n-nodes-base.noOp`; terminal branch for articles rejected by topic filter.
- **Configuration choices:** No parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Filter - Tech Keywords** false branch; no downstream nodes.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None operational.
- **Sub-workflow reference:** None.

---

## 2.5 AI Scoring and Tweet Generation

### Overview
This block sends the shortlisted article data to OpenAI, asking it to score innovation and create a tweet in a strict JSON format.

### Nodes Involved
- AI - Score & Generate Tweet

### Node Details

#### AI - Score & Generate Tweet
- **Type and role:** `@n8n/n8n-nodes-langchain.openAi`; invokes an OpenAI chat model.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Uses a system instruction positioning the model as a professional tech social media editor
  - Requests:
    1. Innovation score from 1–10
    2. Tweet under 240 characters without the link
    3. 1–2 relevant tech hashtags
  - Enforces structured output via JSON schema with required fields:
    - `score` as number
    - `tweet` as string
- **Key expressions or variables used:**
  - System prompt includes strict JSON return instructions
  - User content:
    - `Title: {{$json.title}}`
    - `Summary: {{ $json.summary }}`
    - `Categories: {{($json.categories || []).join(", ")}}`
- **Input and output connections:** Input from **Preserve RSS Data**; output to **Merge AI + RSS (By Position)** input 0.
- **Version-specific requirements:** Type version `2.1`; requires the LangChain OpenAI node package available in the n8n instance.
- **Edge cases or failure types:**
  - Invalid or expired OpenAI credentials
  - Model access restrictions
  - Token/context issues if summary content becomes too large
  - Even with JSON schema, malformed or unexpected response nesting is possible
- **Sub-workflow reference:** None.

---

## 2.6 AI/RSS Merge and Output Normalization

### Overview
This block recombines the AI response with the original RSS data using positional alignment, validates whether AI output exists, then emits a normalized object for downstream filtering and delivery.

### Nodes Involved
- Merge AI + RSS (By Position)
- Alignment Validator
- Normalize Final Structure

### Node Details

#### Merge AI + RSS (By Position)
- **Type and role:** `n8n-nodes-base.merge`; combines AI results and preserved RSS records by item position.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `combineByPosition`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input 0 from **AI - Score & Generate Tweet**
  - Input 1 from **Preserve RSS Data**
  - Output to **Alignment Validator**
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases or failure types:**
  - Positional merge assumes both branches preserve the same item count and order
  - If AI node drops or reorders items, outputs can become mismatched
- **Sub-workflow reference:** None.

#### Alignment Validator
- **Type and role:** `n8n-nodes-base.code`; checks whether AI output exists and extracts preliminary `ai_score` and `tweet_text`.
- **Configuration choices:** Pulls AI content from nested path `item.json.output?.[0]?.content?.[0]?.text`.
- **Key expressions or variables used:**
  - `const ai = item.json.output?.[0]?.content?.[0]?.text;`
  - Sets:
    - `item.json.ai_score`
    - `item.json.tweet_text`
    - `item.json.valid`
- **Input and output connections:** Input from **Merge AI + RSS (By Position)**; output to **Normalize Final Structure**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - Response structure may differ between node versions or model formats
  - `valid` is computed but not used later as a control branch
  - A missing AI response does not fail here; it marks the item invalid and continues
- **Sub-workflow reference:** None.

#### Normalize Final Structure
- **Type and role:** `n8n-nodes-base.code`; creates the final payload used by the remaining workflow.
- **Configuration choices:** Extracts AI object and RSS object, then returns:
  - `score`
  - `tweet`
  - `link`
  - `guid`
  - `creator`
  - `pub_date`
- **Key expressions or variables used:**
  - `const ai = item.json.output?.[0]?.content?.[0]?.text;`
  - `const rss = item.json.rss;`
  - Throws error if either is missing
- **Input and output connections:** Input from **Alignment Validator**; output to **Filter - Score ≥ 8**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - Throws `"Missing AI or RSS data"` if either branch data is absent
  - Assumes AI text payload is already parsed as an object with `score` and `tweet`
  - Uses `rss.pubDate` but renames it to `pub_date`
- **Sub-workflow reference:** None.

---

## 2.7 Quality Threshold and Ranking

### Overview
This block keeps only strong AI-rated items, sorts them by descending score, and limits delivery to the two best candidates.

### Nodes Involved
- Filter - Score ≥ 8
- Sort by AI Score
- Limit - Top 2 Posts
- Low Score

### Node Details

#### Filter - Score ≥ 8
- **Type and role:** `n8n-nodes-base.if`; keeps only items with score greater than or equal to 8.
- **Configuration choices:** Numeric `gte` condition against `$json.score`.
- **Key expressions or variables used:**
  - `={{ $json.score }}`
- **Input and output connections:** Input from **Normalize Final Structure**; true output to **Sort by AI Score**; false output to **Low Score**.
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or failure types:**
  - Non-numeric score values can fail strict validation or compare incorrectly
  - Threshold is hard-coded; changing editorial criteria requires node update
- **Sub-workflow reference:** None.

#### Sort by AI Score
- **Type and role:** `n8n-nodes-base.code`; sorts qualifying items by descending AI score.
- **Configuration choices:** Standard numeric sort.
- **Key expressions or variables used:**
  - `items.sort((a, b) => b.json.score - a.json.score);`
- **Input and output connections:** Input from **Filter - Score ≥ 8** true branch; output to **Limit - Top 2 Posts**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or failure types:**
  - Missing scores will produce unstable sorting results
- **Sub-workflow reference:** None.

#### Limit - Top 2 Posts
- **Type and role:** `n8n-nodes-base.limit`; caps output to the first two items.
- **Configuration choices:** `maxItems = 2`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Sort by AI Score**; outputs in parallel to **Log to Google Sheet** and **Send to Slack Channel**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:**
  - If fewer than two items remain, only existing items are passed on
- **Sub-workflow reference:** None.

#### Low Score
- **Type and role:** `n8n-nodes-base.noOp`; terminal branch for lower-quality content.
- **Configuration choices:** No parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Filter - Score ≥ 8** false branch; no downstream nodes.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None operational.
- **Sub-workflow reference:** None.

---

## 2.8 Distribution and Tracking

### Overview
This block publishes the final tweet text to Slack and logs processed content to Google Sheets so future runs can avoid duplicates.

### Nodes Involved
- Send to Slack Channel
- Log to Google Sheet

### Node Details

#### Send to Slack Channel
- **Type and role:** `n8n-nodes-base.slack`; posts the generated tweet text to a Slack channel.
- **Configuration choices:**
  - Sends `{{$json.tweet}}` as the message text
  - Uses a selected channel destination
  - Channel configured as `all-team-sawi`
  - Workflow link inclusion disabled
- **Key expressions or variables used:**
  - `={{ $json.tweet }}`
- **Input and output connections:** Input from **Limit - Top 2 Posts**; no downstream nodes.
- **Version-specific requirements:** Type version `2.4`.
- **Edge cases or failure types:**
  - Slack auth or bot permission errors
  - Bot not invited to target channel
  - Tweet text may omit the original article link, which may or may not be desired operationally
- **Sub-workflow reference:** None.

#### Log to Google Sheet
- **Type and role:** `n8n-nodes-base.googleSheets`; appends processed items to the tracking spreadsheet.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `techcrunch_rss_feed`
  - Sheet: `techcrunch`
  - Maps fields explicitly:
    - `guid`
    - `link`
    - `score`
    - `tweet`
    - `creator`
    - `pub_date`
    - `date_posted` = current workflow time
  - Matching columns include `link`, but because operation is append, this mainly reflects available schema/mapping context rather than upsert behavior
- **Key expressions or variables used:**
  - `={{ $json.guid }}`
  - `={{ $json.link }}`
  - `={{ $json.score }}`
  - `={{ $json.tweet }}`
  - `={{ $json.creator }}`
  - `={{ $json.pub_date }}`
  - `={{ $now }}`
- **Input and output connections:** Input from **Limit - Top 2 Posts**; no downstream nodes.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or failure types:**
  - OAuth or sheet access errors
  - Schema mismatch if columns are renamed or removed
  - Duplicate rows can still be appended if deduplication earlier fails
- **Sub-workflow reference:** None.

---

## 2.9 Visual Documentation / Sticky Notes

### Overview
These nodes do not affect execution but explain purpose, setup, and customization. They are important for human operators and should be preserved when rebuilding the workflow.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6

### Node Details

#### Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`; high-level workflow documentation.
- **Configuration choices:** Large note describing workflow purpose, setup, and customization.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and role:** Sticky note for the ingestion area.
- **Configuration choices:** Labels the ingestion block.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and role:** Sticky note for filtering and deduplication.
- **Configuration choices:** Labels the filtering block.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and role:** Sticky note for AI scoring section.
- **Configuration choices:** Labels AI generation block.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and role:** Sticky note for merge and normalization.
- **Configuration choices:** Labels merge block.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and role:** Sticky note for quality filtering.
- **Configuration choices:** Labels score-threshold block.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note6
- **Type and role:** Sticky note for distribution and tracking.
- **Configuration choices:** Labels posting/logging block.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Daily workflow entry point |  | Fetch RSS - TechCrunch | ## Data Ingestion\nFetch latest tech articles on a schedule from RSS sources. |
| Fetch RSS - TechCrunch | RSS Feed Read | Retrieve RSS articles from TechCrunch | Schedule Trigger | Filter - Published Within 72h | ## Data Ingestion\nFetch latest tech articles on a schedule from RSS sources. |
| Filter - Published Within 72h | If | Keep only articles from last 72 hours | Fetch RSS - TechCrunch | Normalize Link; Old Article | ## Filtering & Deduplication\nFilter relevant topics and remove already processed articles. |
| Old Article | No Op | Terminal branch for outdated items | Filter - Published Within 72h |  | ## Filtering & Deduplication\nFilter relevant topics and remove already processed articles. |
| Normalize Link | Code | Standardize links for deduplication | Filter - Published Within 72h | Check Google sheet | ## Filtering & Deduplication\nFilter relevant topics and remove already processed articles. |
| Check Google sheet | Google Sheets | Lookup existing links in tracking sheet | Normalize Link | Remove Already Posted Links | ## Filtering & Deduplication\nFilter relevant topics and remove already processed articles. |
| Remove Already Posted Links | Code | Remove items already logged previously | Check Google sheet | Filter - Tech Keywords | ## Filtering & Deduplication\nFilter relevant topics and remove already processed articles. |
| Filter - Tech Keywords | If | Keep only selected tech-topic articles | Remove Already Posted Links | Sort by Publish Date; Not Tech Related | ## Filtering & Deduplication\nFilter relevant topics and remove already processed articles. |
| Not Tech Related | No Op | Terminal branch for irrelevant topics | Filter - Tech Keywords |  | ## Filtering & Deduplication\nFilter relevant topics and remove already processed articles. |
| Sort by Publish Date | Code | Sort recent matching articles and keep top 3 | Filter - Tech Keywords | Preserve RSS Data | ## AI Scoring & Content Generation\nScore innovation and generate tweet + image concept using AI. |
| Preserve RSS Data | Code | Preserve RSS payload and prepare AI input fields | Sort by Publish Date | AI - Score & Generate Tweet; Merge AI + RSS (By Position) | ## AI Scoring & Content Generation\nScore innovation and generate tweet + image concept using AI. |
| AI - Score & Generate Tweet | OpenAI (LangChain) | Score innovation and generate tweet text | Preserve RSS Data | Merge AI + RSS (By Position) | ## AI Scoring & Content Generation\nScore innovation and generate tweet + image concept using AI. |
| Merge AI + RSS (By Position) | Merge | Recombine AI output with RSS data by item order | AI - Score & Generate Tweet; Preserve RSS Data | Alignment Validator | ## Merge & Normalize\nCombine AI output with RSS data and standardize structure. |
| Alignment Validator | Code | Validate AI presence and extract temporary fields | Merge AI + RSS (By Position) | Normalize Final Structure | ## Merge & Normalize\nCombine AI output with RSS data and standardize structure. |
| Normalize Final Structure | Code | Produce final normalized output object | Alignment Validator | Filter - Score ≥ 8 | ## Merge & Normalize\nCombine AI output with RSS data and standardize structure. |
| Filter - Score ≥ 8 | If | Keep only high-scoring items | Normalize Final Structure | Sort by AI Score; Low Score | ## Quality Filter\nKeep only high-impact articles based on AI score threshold. |
| Low Score | No Op | Terminal branch for low-scoring items | Filter - Score ≥ 8 |  | ## Quality Filter\nKeep only high-impact articles based on AI score threshold. |
| Sort by AI Score | Code | Rank items by AI score descending | Filter - Score ≥ 8 | Limit - Top 2 Posts | ## Quality Filter\nKeep only high-impact articles based on AI score threshold. |
| Limit - Top 2 Posts | Limit | Restrict output to top two items | Sort by AI Score | Log to Google Sheet; Send to Slack Channel | ## Quality Filter\nKeep only high-impact articles based on AI score threshold. |
| Send to Slack Channel | Slack | Publish generated tweet text to Slack | Limit - Top 2 Posts |  | ## Distribution & tracking\nSend curated tech insights to Slack or other platforms.\nStore processed items for tracking and deduplication. |
| Log to Google Sheet | Google Sheets | Append processed items for tracking | Limit - Top 2 Posts |  | ## Distribution & tracking\nSend curated tech insights to Slack or other platforms.\nStore processed items for tracking and deduplication. |
| Sticky Note | Sticky Note | Overall workflow documentation |  |  |  |
| Sticky Note1 | Sticky Note | Visual label for ingestion block |  |  |  |
| Sticky Note2 | Sticky Note | Visual label for filtering block |  |  |  |
| Sticky Note3 | Sticky Note | Visual label for AI block |  |  |  |
| Sticky Note4 | Sticky Note | Visual label for merge block |  |  |  |
| Sticky Note5 | Sticky Note | Visual label for quality block |  |  |  |
| Sticky Note6 | Sticky Note | Visual label for distribution block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it `AI Tech Insight Engine`.
   - Set workflow timezone to `Asia/Manila`.
   - Leave execution order as default compatible with current n8n behavior (`v1` in this export).

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Configure it to run daily at hour `7`.
   - This becomes the single entry point.

3. **Add an RSS Feed Read node**
   - Node type: **RSS Feed Read**
   - Name it `Fetch RSS - TechCrunch`
   - URL: `https://techcrunch.com/feed/`
   - Keep SSL validation enabled.
   - Connect **Schedule Trigger → Fetch RSS - TechCrunch**.

4. **Add a recency filter**
   - Node type: **If**
   - Name it `Filter - Published Within 72h`
   - Add one boolean condition using an expression:
     - `new Date($json.isoDate) > new Date(Date.now() - 72 * 60 * 60 * 1000)`
   - Connect **Fetch RSS - TechCrunch → Filter - Published Within 72h**.

5. **Add a no-op node for stale content**
   - Node type: **No Op**
   - Name it `Old Article`
   - Connect the **false** output of `Filter - Published Within 72h` to `Old Article`.

6. **Add a code node to normalize article links**
   - Node type: **Code**
   - Name it `Normalize Link`
   - Use JavaScript:
     ```javascript
     return items.map(item => {
       item.json.link = item.json.link.replace(/\/$/, '');
       return item;
     });
     ```
   - Connect the **true** output of `Filter - Published Within 72h` to `Normalize Link`.

7. **Prepare Google Sheets for deduplication and logging**
   - Create a Google Sheet with a tab such as `techcrunch`.
   - Add these columns:
     - `link`
     - `guid`
     - `creator`
     - `pub_date`
     - `tweet`
     - `score`
     - `date_posted`
   - Configure **Google Sheets OAuth2** credentials in n8n.

8. **Add a Google Sheets lookup node**
   - Node type: **Google Sheets**
   - Name it `Check Google sheet`
   - Select your spreadsheet and target tab.
   - Configure a filter:
     - Lookup column: `link`
     - Lookup value: `{{$json.link}}`
   - Enable **Always Output Data**.
   - Connect **Normalize Link → Check Google sheet**.

9. **Add a code node to remove already logged links**
   - Node type: **Code**
   - Name it `Remove Already Posted Links`
   - Use:
     ```javascript
     const existingLinks = $items("Check Google sheet")
       .map(i => i.json.link?.replace(/\/$/, ''))
       .filter(Boolean);

     return $items("Normalize Link")
       .filter(item => {
         const cleanLink = item.json.link.replace(/\/$/, '');
         return !existingLinks.includes(cleanLink);
       });
     ```
   - Connect **Check Google sheet → Remove Already Posted Links**.
   - Important: keep the node names exactly as referenced, or update the code accordingly.

10. **Add a keyword filter**
    - Node type: **If**
    - Name it `Filter - Tech Keywords`
    - Add one boolean expression:
      ```javascript
      $json.title.toLowerCase().includes("ai") ||
      $json.title.toLowerCase().includes("artificial intelligence") ||
      $json.title.toLowerCase().includes("cybersecurity") ||
      $json.title.toLowerCase().includes("robotics") ||
      $json.title.toLowerCase().includes("quantum") ||
      $json.title.toLowerCase().includes("blockchain") ||
      $json.title.toLowerCase().includes("machine learning")
      ```
    - Connect **Remove Already Posted Links → Filter - Tech Keywords**.

11. **Add a no-op node for irrelevant items**
    - Node type: **No Op**
    - Name it `Not Tech Related`
    - Connect the **false** output of `Filter - Tech Keywords` to it.

12. **Add a code node to sort by publish date**
    - Node type: **Code**
    - Name it `Sort by Publish Date`
    - Use:
      ```javascript
      items.sort((a, b) => new Date(b.json.pubDate) - new Date(a.json.pubDate));
      return items.slice(0, 3);
      ```
    - Connect the **true** output of `Filter - Tech Keywords` to it.

13. **Add a code node to preserve RSS data for later merge**
    - Node type: **Code**
    - Name it `Preserve RSS Data`
    - Use:
      ```javascript
      return items.map(item => {
        return {
          json: {
            rss: item.json,
            title: item.json.title,
            summary: item.json.contentSnippet,
            categories: item.json.categories
          }
        };
      });
      ```
    - Connect **Sort by Publish Date → Preserve RSS Data**.

14. **Configure OpenAI credentials**
    - Create an **OpenAI API** credential in n8n.
    - Ensure the account has access to `gpt-4o-mini` or replace with another supported model.

15. **Add the OpenAI node**
    - Node type: **OpenAI** from the LangChain/OpenAI integration
    - Name it `AI - Score & Generate Tweet`
    - Model: `gpt-4o-mini`
    - Add a system message:
      ```text
      You are a professional tech social media editor.

      1. Rate the article from 1-10 based on innovation in emerging technology.
      2. Write a concise, engaging tweet under 240 characters (without the link).
      3. Add 1-2 relevant tech hashtags.

      Return your answer strictly in this JSON format:

      {
      "score": number,
      "tweet": "text"
      }
      ```
    - Add a user message:
      ```text
      Title: {{$json.title}}
      Summary: {{ $json.summary }}
      Categories: {{($json.categories || []).join(", ")}}
      ```
    - Enable structured output / text format with JSON schema:
      ```json
      {
        "type": "object",
        "properties": {
          "score": { "type": "number" },
          "tweet": { "type": "string" }
        },
        "required": ["score", "tweet"],
        "additionalProperties": false
      }
      ```
    - Connect **Preserve RSS Data → AI - Score & Generate Tweet**.

16. **Add a Merge node**
    - Node type: **Merge**
    - Name it `Merge AI + RSS (By Position)`
    - Mode: **Combine**
    - Combine by: **Position**
    - Connect:
      - **AI - Score & Generate Tweet → Merge AI + RSS (By Position)** input 1
      - **Preserve RSS Data → Merge AI + RSS (By Position)** input 2

17. **Add an alignment validator code node**
    - Node type: **Code**
    - Name it `Alignment Validator`
    - Use:
      ```javascript
      return items.map(item => {

        const ai = item.json.output?.[0]?.content?.[0]?.text;

        if (!ai) {
          item.json.ai_score = 0;
          item.json.tweet_text = null;
          item.json.valid = false;
          return item;
        }

        item.json.ai_score = ai.score ?? 0;
        item.json.tweet_text = ai.tweet ?? null;
        item.json.valid = item.json.ai_score >= 8;

        return item;
      });
      ```
    - Connect **Merge AI + RSS (By Position) → Alignment Validator**.

18. **Add a normalization code node**
    - Node type: **Code**
    - Name it `Normalize Final Structure`
    - Use:
      ```javascript
      return items.map(item => {
        const ai = item.json.output?.[0]?.content?.[0]?.text;
        const rss = item.json.rss;

        if (!ai || !rss) {
          throw new Error("Missing AI or RSS data");
        }

        return {
          json: {
            score: ai.score,
            tweet: ai.tweet,
            link: rss.link,
            guid: rss.guid,
            creator: rss.creator,
            pub_date: rss.pubDate
          }
        };
      });
      ```
    - Connect **Alignment Validator → Normalize Final Structure**.

19. **Add a score threshold filter**
    - Node type: **If**
    - Name it `Filter - Score ≥ 8`
    - Configure a numeric condition:
      - Left value: `{{$json.score}}`
      - Operation: `greater than or equal`
      - Right value: `8`
    - Connect **Normalize Final Structure → Filter - Score ≥ 8**.

20. **Add a no-op node for low scores**
    - Node type: **No Op**
    - Name it `Low Score`
    - Connect the **false** output of `Filter - Score ≥ 8` to it.

21. **Add a code node to sort by AI score**
    - Node type: **Code**
    - Name it `Sort by AI Score`
    - Use:
      ```javascript
      items.sort((a, b) => b.json.score - a.json.score);
      return items;
      ```
    - Connect the **true** output of `Filter - Score ≥ 8` to it.

22. **Add a Limit node**
    - Node type: **Limit**
    - Name it `Limit - Top 2 Posts`
    - Max items: `2`
    - Connect **Sort by AI Score → Limit - Top 2 Posts**.

23. **Add the Slack node**
    - Node type: **Slack**
    - Name it `Send to Slack Channel`
    - Configure Slack credentials.
    - Choose operation to send a message to a channel.
    - Set message text to:
      - `{{$json.tweet}}`
    - Select your target channel.
    - Disable “include link to workflow” if you want parity with the exported workflow.
    - Connect **Limit - Top 2 Posts → Send to Slack Channel**.

24. **Add the Google Sheets logging node**
    - Node type: **Google Sheets**
    - Name it `Log to Google Sheet`
    - Operation: **Append**
    - Use the same spreadsheet and tab as deduplication.
    - Map fields:
      - `guid` → `{{$json.guid}}`
      - `link` → `{{$json.link}}`
      - `score` → `{{$json.score}}`
      - `tweet` → `{{$json.tweet}}`
      - `creator` → `{{$json.creator}}`
      - `pub_date` → `{{$json.pub_date}}`
      - `date_posted` → `{{$now}}`
    - Connect **Limit - Top 2 Posts → Log to Google Sheet**.

25. **Add sticky notes for maintainability**
    - Add one general workflow note explaining:
      - workflow purpose
      - setup requirements
      - customization options
    - Add block labels:
      - `Data Ingestion`
      - `Filtering & Deduplication`
      - `AI Scoring & Content Generation`
      - `Merge & Normalize`
      - `Quality Filter`
      - `Distribution & tracking`

26. **Test the workflow**
    - Run manually.
    - Confirm:
      - RSS items are returned
      - old items go to `Old Article`
      - duplicate links are excluded
      - AI node returns structured JSON
      - merge keeps proper article/AI alignment
      - only score `>= 8` items continue
      - top two are posted and logged

27. **Activate the workflow**
    - Once credentials and mappings are verified, activate it so the schedule executes automatically.

## Credential setup summary
- **OpenAI API**
  - Required for `AI - Score & Generate Tweet`
  - Must support selected model
- **Google Sheets OAuth2**
  - Required for `Check Google sheet` and `Log to Google Sheet`
  - Spreadsheet must be accessible by the authenticated Google account
- **Slack API**
  - Required for `Send to Slack Channel`
  - Bot/user must have permission to post to selected channel

## Input/output expectations
- **RSS input fields expected:** `title`, `link`, `guid`, `creator`, `pubDate`, `isoDate`, `contentSnippet`, optional `categories`
- **AI output expected:** object with `score` number and `tweet` string
- **Final output structure before publishing/logging:**
  - `score`
  - `tweet`
  - `link`
  - `guid`
  - `creator`
  - `pub_date`

## Important rebuild cautions
- The code in `Remove Already Posted Links` depends on exact node names.
- The positional merge depends on both branches keeping identical order and item count.
- The OpenAI response extraction path may need adjustment if your installed OpenAI/LangChain node version returns a different structure.
- The sticky note says “tweet + image concept,” but the actual AI node only returns `score` and `tweet`; there is no image-generation step in this workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI Tech Insight Engine: automatically collects tech news, filters emerging-tech topics, scores with AI, and generates social-ready content while reducing noise. | Overall workflow purpose |
| Setup guidance: add RSS feed URL, configure OpenAI credentials, connect Google Sheets, and set up Slack notifications. | Deployment notes |
| Customization options: adjust score threshold, modify prompt tone or niche, and change output destination to X, Telegram, or Email. | Maintenance and extension |
| The descriptive note mentions “tweet + image concept,” but the implemented AI schema only returns `score` and `tweet`. | Implementation consistency note |
| No sub-workflows are used in this workflow. | Architecture note |