Curate and generate an AI newsletter from RSS feeds with OpenAI and Slack

https://n8nworkflows.xyz/workflows/curate-and-generate-an-ai-newsletter-from-rss-feeds-with-openai-and-slack-12533


# Curate and generate an AI newsletter from RSS feeds with OpenAI and Slack

## 1. Workflow Overview

**Title:** Curate and generate an AI newsletter from RSS feeds with OpenAI and Slack

**Purpose:**  
This workflow continuously ingests AI/tech news from multiple RSS-style sources (newsletters, Google News, Hacker News, official AI-company blogs, and several subreddits via rss.app JSON feeds). It scrapes each item’s linked page, uses an LLM to validate relevance, logs relevant items to Google Sheets, then (on a daily schedule) composes a blog/newsletter edition (“The Recap”) including:
- selected top stories (human-in-the-loop approval in Slack with iterative revisions),
- a subject line + pre-header (also Slack approval + iterative revisions),
- full story segments written in a consistent style,
- an intro section,
- a “Shortlist/Other Top Stories” quick-hits section,
- and 3 viral short-form video ideas.

The final newsletter and video ideas are uploaded to Slack as files, with permalink messages.

### 1.1 Ingestion: RSS/JSON triggers (multi-source)
- Multiple entry points: several RSS Feed Read Triggers and Schedule Triggers fetching rss.app JSON feeds.
- Output is normalized into a common schema: `sourceName`, `title`, `creator`, `link`, `pubDate`, `isoDate`, `feedType`, `feedUrl`.

### 1.2 Scraping & relevance classification
- Each normalized item is passed to a separate “Scrape Url” sub-workflow to fetch full page content/metadata.
- Scrape errors are filtered out; then an LLM evaluates if content is relevant to an AI newsletter.
- Relevant stories are appended/updated in Google Sheets as a tracker.

### 1.3 Daily edition generation (“Blog Trigger”)
- At a scheduled time, the workflow reads collected stories from Google Sheets, aggregates them, and prompts an LLM to select the top stories.
- A Slack approval loop allows a human editor to approve or provide feedback; feedback triggers revision via another LLM step.

### 1.4 Subject line + pre-header generation (with approval loop)
- After top stories approval, an LLM writes a subject line and pre-header.
- Another Slack approval loop allows iterative refinement.

### 1.5 Draft writing: story segments, intro, shortlist
- For each selected story, the workflow pulls all associated raw content by identifier, optionally scrapes “external source links”, and writes a formatted story segment.
- After all segments are generated, it writes the intro and “Other Top Stories” section.

### 1.6 Packaging + Slack delivery + video ideas
- The full newsletter is assembled into markdown, converted to a file, and uploaded to Slack.
- Viral video ideas are generated from the final content, converted to a file, and uploaded to Slack.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Source ingestion: RSS/JSON feeds and normalization
**Overview:**  
This block collects items from multiple sources at different schedules. Each source’s output is normalized into a shared schema for downstream scraping and processing.

**Nodes involved:**
- RSS Feed triggers (newsletter-style):
  - `the_rundown_ai_trigger`, `bens_bites_trigger`, `the_neuron_trigger`, `futurepedia_trigger`, `superhuman_trigger`, `taaft_trigger`
- Schedule + HTTP JSON feeds (news sites and communities):
  - `google_news_trigger` → `fetch_google_news_feed` → `split_google_news_items` → `normalize_google_news_articles`
  - `hacker_news_trigger` → `fetch_hacker_news_feed` → `split_hacker_news_items` → `normalize_hacker_news_articles`
  - `reddit_artificial_inteligence_trigger` → `fetch_reddit_artificial_inteligence_feed` → `split_reddit_artificial_inteligence_items` → `get_reddit_artificial_inteligence_items` (disabled) → `filter_reddit_artificial_inteligence_items` → `normalize_reddit_artificial_inteligence_items`
  - `reddit_open_ai_trigger` → `fetch_reddit_open_ai_feed` → `split_reddit_open_ai_items` → `get_reddit_open_ai_items` (disabled) → `filter_reddit_open_ai_items` → `normalize_reddit_open_ai_items`
  - `reddit_artificial_trigger` → `fetch_reddit_artificial_feed` → `split_reddit_artificial_items` → `get_reddit_artificial_items` (disabled) → `filter_reddit_artificial_items` → `normalize_reddit_artificial_items`
- Official blog feeds:
  - `blog_meta_ai_trigger` → `fetch_blog_meta_ai_feed` → `split_blog_meta_ai_items` → `normalize_blog_meta_ai_articles`
  - `blog_cloudflare_ai_trigger` → `fetch_blog_cloudflare_ai_feed` → `split_blog_cloudflare_ai_items` → `normalize_blog_cloudflare_ai_articles`
  - `blog_anthropic_ai_trigger` → `fetch_blog_anthropic_ai_feed` → `split_blog_anthropic_ai_items` → `normalize_blog_anthropic_ai_articles`
  - `blog_google_ai_trigger` → `fetch_blog_google_ai_feed` → `split_blog_google_ai_items` → `normalize_blog_google_ai_articles`
  - `blog_open_ai_trigger` → `fetch_blog_open_ai_feed` → `split_blog_open_ai_items` → `normalize_blog_open_ai_articles`
  - `blog_nvidia_ai_trigger` → `fetch_blog_nvidia_ai_feed` → `split_blog_nvidia_ai_items` → `normalize_blog_nvidia_ai_articles`
- Events feed:
  - `google_cloud_events_trigger` → `fetch_google_cloud_events_feed` → `split_google_cloud_events_items` → `normalize_google_cloud_events`
- Newsletter normalization Set nodes:
  - `normalize_the_rundown_ai`, `normalize_bens_bites`, `normalize_neuron`, `normalize_futurepedia`, `normalize_superhuman`, `normalize_taaft`
- Sticky notes affecting this area:
  - “RSS feeds”, “RSS Feed”

**Key node details (representative patterns):**

1) **RSS triggers (e.g., `the_neuron_trigger`)**
- **Type/role:** `rssFeedReadTrigger` – polls RSS feed URL every 4 minutes (“everyX value 4”).
- **Output:** Typical RSS item fields (title, link, pubDate, etc.), then normalized.
- **Failure modes:** feed downtime, malformed RSS, high-frequency polling rate limits.

2) **Schedule + HTTP JSON feed (e.g., `google_news_trigger` + `fetch_google_news_feed`)**
- **Type/role:** `scheduleTrigger` (every 3 hours) → `httpRequest` to rss.app JSON feed.
- **Split:** `splitOut` splits `.items[]` into individual items.
- **Normalize (`normalize_google_news_articles`)**:
  - **Type:** Code node (“runOnceForEachItem”).
  - **Logic:** Extract hostname via regex from the item URL; map known domains to friendly `sourceName`; fallback transforms domain into dashed string.
  - **Output schema:** `{ sourceName, title, creator, link, pubDate, isoDate, feedType: "article", feedUrl }`
  - **Failure modes:** URL missing/invalid (regex failure), unexpected rss.app schema changes.
  - **Edge case:** domain map includes many explicit domains; unknown domains still handled by fallback.

3) **Reddit via rss.app feeds**
- **Approach used:** HTTP JSON feed (rss.app) instead of direct Reddit API.
- **Disabled nodes:** `get_reddit_*_items` nodes are disabled but still connected; they are configured to fetch a post by ID.
  - **Risk:** If enabled, it depends on Reddit credentials and postId extraction; also the rss item schema might not match.
- **Filtering (`filter_reddit_*_items`):**
  - Excludes items with errors.
  - Requires `url_overridden_by_dest` exists and is not empty.
  - Excludes destinations containing reddit/youtube/x/github/redd.it media.
- **Normalization:** Code nodes compute `sourceName` from URL domain and convert unix timestamps to ISO.

4) **Newsletter normalization (`normalize_*` Set nodes)**
- **Type/role:** Set node that adds:
  - `sourceName` (e.g., “the-neuron”),
  - `feedType: "newsletter"`,
  - `feedUrl` to the archive homepage.
- **Edge case:** RSS item link may be different from archive; downstream scraping uses `link` (later mapped to `url`).

5) **Events normalization bug**
- `normalize_google_cloud_events` returns `sourceName: "blog-nvidia-ai"` but should likely be something like `"google-cloud-events"` or `"blog-google-cloud"`.
- **Impact:** misleading source attribution and file naming; may complicate limiting/dedup.

**Sub-workflow references:** none in this block.

---

### Block 2.2 — Identity shaping, limiting, scraping, relevance classification, and logging
**Overview:**  
Normalized feed items are mapped into a consistent “identity” payload, optionally limited per source, delayed, scraped via a sub-workflow, checked for errors, evaluated by an LLM for relevance, then logged to Google Sheets if relevant.

**Nodes involved:**
- `Get Identity`
- `Limit Number of Items from Each Source`
- `Delay 10s`
- `Scrape URL` (Execute Workflow)
- `Filter Scrape Errors`
- `evaluate_content` (LLM chain)
- `o3-mini` (OpenAI chat model used by evaluate_content)
- `is_revelant_content_parser` (structured output parser)
- `Merge Every Source`
- `Ensure Relevant`
- `Log to Google Sheets`
- Sticky notes:
  - “Scraping Content”
  - “Limiting Content”

**Node details:**

1) **Get Identity**
- **Type/role:** Set – normalizes incoming item fields into canonical fields.
- **Key fields created:**
  - `title = $json.title`
  - `url = $json.link`
  - `authors = $json.creator`
  - `date = $json.pubDate`
  - `publishedTimestamp = $json.isoDate`
  - `sourceName`, `feedType`, `feedUrl`
  - `uploadFileName`: constructs a path-like slug: `YYYY-MM-DD/<slug>.<sourceName>`
- **Edge cases:**
  - `$json.isoDate` missing → substring call fails.
  - Titles with unusual characters are stripped; could collapse to empty slug.
- **Connections:** output → `Limit Number of Items from Each Source`

2) **Limit Number of Items from Each Source**
- **Type/role:** Limit – intended to cap number of items per feed.
- **Config:** parameters are empty in JSON (so it may default to 1 item). In n8n, the Limit node typically needs an explicit limit; otherwise behavior depends on defaults/version.
- **Risk:** accidental under-collection or over-collection if not set.
- **Connections:** → `Delay 10s`

3) **Delay 10s**
- **Type/role:** Wait – throttling to reduce scrape bursts / rate limiting.
- **Config:** waits 10 seconds per item.
- **Connections:** → `Scrape URL`

4) **Scrape URL**
- **Type/role:** Execute Workflow – calls sub-workflow **“AI News Aggregator - Scrape Url”**.
- **Input mapping:** `{ url: {{$node["Get Identity"].json.url}} }`
- **Retry:** enabled with 5s wait between tries.
- **Failure modes:** sub-workflow failure, timeouts, blocked by paywalls/bot protection, invalid HTML, missing content extraction.

5) **Filter Scrape Errors**
- **Type/role:** Filter – drops items where `$json.error` exists.
- **Important connection detail:** This node outputs to:
  - `evaluate_content` (for relevance check),
  - and also to `Merge Every Source` (index 0 path) to merge metadata even before LLM evaluation.

6) **evaluate_content** + **o3-mini** + **is_revelant_content_parser**
- **Type/role:** LangChain LLM chain node that prompts: “Given content fetched from a web page, analyze… determine if it’s full and relevant…”
- **Model:** `o3-mini` via OpenAI credentials.
- **Structured output:** `is_revelant_content_parser` expects:
  - `chainOfThought` (string)
  - `is_revelant_content` (boolean)
- **Connections:** evaluation output is merged with source data in `Merge Every Source` (as second input).
- **Failure modes:**
  - LLM returning invalid JSON → parser errors (no autofix parser here).
  - Large scraped text → token limit / truncation.
  - Model availability / auth issues.

7) **Merge Every Source**
- **Type/role:** Merge (combine by position).
- **Inputs:**
  - Input 0: from `Filter Scrape Errors` (scraped payload)
  - Input 1: from `evaluate_content` (LLM output)
- **Risk:** Combine-by-position assumes both streams have identical item order and counts. Any mismatch (e.g., evaluation fails for one item) can misalign relevance with the wrong story.

8) **Ensure Relevant**
- **Type/role:** Filter – keeps only items where `$json.output.is_revelant_content` is true.
- **Failure mode:** if `output` or `is_revelant_content` missing due to upstream merge mismatch, filter may drop all or error depending on strictness.

9) **Log to Google Sheets**
- **Type/role:** Google Sheets “appendOrUpdate” to tracker.
- **Matching:** matches on `Title` column.
- **Mapped columns:**
  - URL from `$json.data.metadata.url`
  - Title from `$json.data.metadata['og:title']`
  - Source from `$json.data.metadata.ogSiteName`
  - Content from `$json.data.json.content`
  - Published from `$json.data.metadata.publishedTime`
  - External Sources from `$json.data.metadata.externalSources || ''`
- **Edge cases/failures:**
  - Missing metadata keys → blank cells or expression errors.
  - Matching only by title can overwrite different stories sharing a title.
  - Google OAuth token expiry; sheet permission errors.

**Sub-workflow reference:**
- `Scrape URL` and `Scrape Segment External Source URL` invoke **“AI News Aggregator - Scrape Url”** and require that workflow to accept an input `url` and return structured content used downstream (`data.json.content`, `data.metadata`, possibly `markdown`).

---

### Block 2.3 — Daily run: retrieve stored stories and pick top stories (human-in-the-loop)
**Overview:**  
On a daily schedule, the workflow reads the tracker sheet, aggregates all items, prompts the LLM to pick top stories and produce structured JSON, posts selections + reasoning to Slack, and waits for approval/feedback. If feedback is provided, the workflow revises selections.

**Nodes involved:**
- `Blog Trigger`
- `Get Stories`
- `Aggregate Stories`
- `Stories Prompt`
- `pick_top_stories`
- `top_stories_parser`
- `top_stories_auto_parser`
- `Set Current Stories`
- `Share Selected Stories`
- `Share Stories Reasoning`
- `Share Stories Approval Feedback` (sendAndWait)
- `extract_stories_approval_feedback`
- `Check Stories Feedback` (IF)
- `edit_top_stories`
- Sticky notes:
  - “Retrieve Blog Content”
  - “Pick Top Stories”
  - “Select top 3-5 stories”

**Node details:**

1) **Blog Trigger**
- **Type/role:** Schedule trigger.
- **Config:** triggers daily at hour 9 (`triggerAtHour: 9`).
- **Edge case:** timezone is n8n instance timezone; may not be local time.

2) **Get Stories**
- **Type/role:** Google Sheets read (operation not explicitly shown; default is typically “read/getAll” depending on node UI).
- **Output:** rows from “AI News Tracker” document, Sheet1.

3) **Aggregate Stories**
- **Type/role:** Aggregate – “aggregateAllItemData”.
- **Purpose:** creates one item with `.data` containing all rows, used to prompt LLM.

4) **Stories Prompt**
- **Type/role:** Set – builds `select_top_stories_prompt` using `Aggregate Stories` data.
- **Important constraints embedded:** choose 4 stories, group sources, exclude politics/ethics/training-course ads, copy identifiers/URLs exactly.

5) **pick_top_stories**
- **Type/role:** LangChain LLM chain using OpenAI Chat model (from “OpenAI Chat Model” connection).
- **Output:** JSON-only response; enforced by structured parser chain.
- **Retry:** enabled.
- **Parsers:** `top_stories_parser` → `top_stories_auto_parser` (auto-fixing) → back into the chain node output.
- **Failure modes:** schema mismatch (notably the schema requires `reason_for_selecting` but the schema’s `required` list includes it while the properties list omits it—this can cause strict validation issues depending on parser behavior).

6) **Set Current Stories**
- **Type/role:** Set – stores `$json.output` into `current_stories`.

7) **Slack posting + approval**
- `Share Selected Stories`: posts the selected stories list with identifiers and external links.
- `Share Stories Reasoning`: posts chain-of-thought reasoning in a thread under the message timestamp.
- `Share Stories Approval Feedback`: interactive Slack node `sendAndWait` to collect free-text feedback or approval.
- **Failure modes:** Slack permissions, wrong channelId, interactive messages require proper Slack app setup.

8) **extract_stories_approval_feedback** + **Check Stories Feedback**
- **Type/role:** Information extractor using LLM.
- **Extracted fields:** `approved` (boolean), `feedback` (string).
- **IF:** if approved is true → proceed; else trigger edit path.

9) **edit_top_stories**
- **Type/role:** LLM chain to apply editor feedback strictly and preserve identifiers/external links for retained stories.
- **Input:** feedback from Slack + original prompt + current selection.
- **Output:** new `top_selected_stories` JSON; fed back into `Set Current Stories`, then re-shared (via same downstream share nodes on next runIndex).

**Edge cases:**
- Approval classifier may misinterpret casual messages (“looks good but…”)—it’s instructed to set `approved=false` if feedback exists, but natural language is messy.
- Revision loop is linear but relies on `$runIndex` messaging; ensure Slack threads and run indexing behave as expected.

---

### Block 2.4 — Subject line + pre-header generation (human-in-the-loop)
**Overview:**  
Once stories are approved, the workflow generates subject line + pre-header, shares them to Slack with reasoning, waits for approval/feedback, and optionally revises.

**Nodes involved:**
- `Subject Examples`
- `Set Subject Line Prompt`
- `write_subject_line`
- `subject_line_parser`
- `subject_line_auto_parser`
- `Set Current Subject Line`
- `Share Subject Line`
- `Share Subject Line Reasoning`
- `Share Subject Line Approval Feedback` (sendAndWait)
- `extract_subject_line_approval_feedback`
- `Check Subject Line Feedback` (IF)
- `edit_subject_line`
- Sticky note:
  - “Write Title”

**Node details:**

1) **Subject Examples**
- **Type/role:** Set – stores a multi-line list of example subject lines. (Field name in JSON includes `=subject_line_examples` which is unusual but n8n allows arbitrary keys.)
- **Connection:** → `Set Subject Line Prompt` (but the prompt does not clearly reference the examples field; may be unused unless referenced in the prompt string elsewhere).

2) **Set Subject Line Prompt**
- **Type/role:** Set – constructs `write_subject_line_prompt` with strict instructions:
  - Title must focus on lead story.
  - 7–9 words max.
  - Provide reasoning first.
- **Potential issue:** The prompt appears to start a fenced JSON block (` ```json `) but does not show closing fences in the provided snippet; may be truncated in JSON export or could confuse the model.

3) **write_subject_line**
- **Type/role:** LLM chain (OpenAI model) producing JSON with subject line, alternatives, pre-header, and reasoning fields.
- **Parsers:** `subject_line_parser` → `subject_line_auto_parser` (auto-fixing).
- **Retry:** enabled.
- **Failure modes:** JSON schema mismatch; too-long title vs constraint; LLM may violate word count.

4) **Slack approval loop**
- `Share Subject Line` + `Share Subject Line Reasoning` + `Share Subject Line Approval Feedback`.
- `extract_subject_line_approval_feedback` extracts `approved` + `feedback`.
- `Check Subject Line Feedback`: approved → proceed; otherwise → `edit_subject_line`.

5) **edit_subject_line**
- **Type/role:** LLM chain that edits only what feedback requests; must preserve unchanged fields exactly where not mentioned.
- **Output:** re-fed into `Set Current Subject Line`.

---

### Block 2.5 — Write story segments (iterate selected stories, enrich sources)
**Overview:**  
For each selected story, the workflow gathers all related raw content by identifier, optionally scrapes additional “external sources”, then writes a markdown story segment via LLM, shares it to Slack, and stores it for later assembly.

**Nodes involved:**
- `Set Selected Stories`
- `Split Stories`
- `Iterate Stories` (SplitInBatches)
- `Set Current Segement`
- `Split Content IDs`
- `Aggregate Segment Text Content`
- `Check External URLs` (IF)
- External-source scraping path:
  - `Set Segment External Source Links` → `Split Segment External Source URLs` → `Scrape Segment External Source URL` → `Filter Segment External Source Errors` → `Aggregate Segment External Source Content`
- Segment writing + parsing:
  - `write_segment_content` → `story_segment_output_parser` → `story_segment_auto_parser` → `share_segment_msg` → `set_story_segment`
- Sticky note:
  - “Important Segement” (covers the segment-writing area)

**Node details:**

1) **Set Selected Stories**
- **Type/role:** Set – combines approved stories + approved subject line into a single object:
  - `subject_line`
  - `pre_header_text`
  - `top_selected_stories` array
- **Dependency:** requires `Set Current Stories` and `Set Current Subject Line` to have been finalized.

2) **Split Stories** / **Iterate Stories**
- **SplitOut:** creates one item per story.
- **SplitInBatches (`Iterate Stories`):** iterates serially through stories; also used as a loop anchor (the segment output is fed back into `Iterate Stories` via `set_story_segment`).

3) **Set Current Segement**
- **Type/role:** Set – stores current story object as `current_story`.

4) **Split Content IDs** → **Aggregate Segment Text Content**
- **Purpose:** gather the raw content snippets for all identifiers belonging to the current story.
- **Important assumption:** the incoming items already contain `content_item` corresponding to each identifier. However, in the provided workflow, there is no explicit step shown that maps identifiers to sheet rows and extracts `content_item`. If `content_item` is not present, the aggregate will be empty.
- **Risk:** The segment-writing prompt references `$node["Aggregate Segment Text Content"].json.content_item.join("\n\n")`. If `content_item` is missing or not an array, this expression can fail.

5) **Check External URLs**
- **IF condition:** checks if `current_story.external_source_links` exists and is not empty.
- **Outputs:**
  - True branch: goes to external-source scraping path.
  - Also routes to `write_segment_content` directly (so segment writing proceeds regardless, but the prompt conditionally includes external source content only if the aggregate node executed).

6) **Scrape Segment External Source URL** (Execute Workflow)
- Calls the same sub-workflow “AI News Aggregator - Scrape Url” for each external source link.
- `onError: continueRegularOutput` so failures won’t crash the run.
- Then filtered (`Filter Segment External Source Errors`) and aggregated.

7) **write_segment_content**
- **Type/role:** LLM chain that produces JSON with `newsletter_section_content` (markdown) for this story.
- **Model:** OpenAI Chat model (chatgpt-4o-latest from “OpenAI Chat Model” connection).
- **Prompt:** strict formatting, link integrity rules (“source material only”, “copy URLs character-for-character”), and JSON-only output requirement.
- **Parser chain:** structured parser `story_segment_output_parser` + autofixing parser `story_segment_auto_parser`.
- **Notable inconsistency:** prompt schema shows `blog_post_section_content` but later “IMPORTANT” demands `newsletter_section_content`; the parser expects `newsletter_section_content`. This mismatch can cause occasional model confusion and reliance on auto-fixer.

8) **share_segment_msg**
- Posts the generated segment content to Slack as a code block for review.
- **Edge case:** Slack message length limits—long segments may be truncated.

9) **set_story_segment**
- Stores the segment output object as `story_segment` and loops back to `Iterate Stories` for the next story.

---

### Block 2.6 — Combine segments, write intro, write shortlist (“Other Top Stories”)
**Overview:**  
Once all story segments are created, this block concatenates them, generates an intro section based on the combined body, and generates a “Quick Hits / Other Top Stories” section from remaining aggregated stories (excluding already covered topics).

**Nodes involved:**
- `Set Story Segments`
- `Aggregate Story Sections`
- `Set Combined Sections Content`
- `write_intro` → `intro_parser` → `intro_auto_parser`
- `write_other_top_stories` → `other_top_stories_parser` → `other_top_stories_auto_parser`
- Sticky notes:
  - “Write Intro Section”
  - “Write "The Shortlist" Section”

**Node details:**

1) **Set Story Segments → Aggregate Story Sections → Set Combined Sections Content**
- **Role:** collect each `newsletter_section_content` into a single markdown body separated by `---`.
- **Potential issue:** `Set Story Segments` pulls from `$node["Iterate Stories"].json.story_segment.newsletter_section_content`. This implies `Iterate Stories` items contain `story_segment`; depends on loop behavior and may fail if the loop state is not consistent.

2) **write_intro**
- **Type/role:** LLM chain writing intro section in a specific format including the bolded phrase **“In today’s AI recap:”** and a bullet list of topics.
- **Parser chain:** `intro_parser` → `intro_auto_parser`.
- **Mismatch:** The prompt says output JSON key `"intro_content"` but the parser expects `newsletter_intro_section_content`. Auto-fixer may reconcile, but this is a structural risk.

3) **write_other_top_stories**
- **Type/role:** LLM chain creates a “Other Top AI Stories” section with strict deep-link copying rules.
- **Input:** `Aggregate Stories` full dataset + `Set Combined Sections Content` for exclusion of duplicates.
- **Parser chain:** `other_top_stories_parser` → `other_top_stories_auto_parser`.
- **Mismatch:** prompt says `"other_stories_content"` but parser expects `newsletter_other_top_stories_section_content`. Again relies on autofix.

---

### Block 2.7 — Assemble final newsletter + generate and deliver video ideas
**Overview:**  
Assembles the full newsletter markdown, converts it to a text file, uploads it to Slack, generates 3 viral video concepts, uploads them as a second file, and posts permalink messages.

**Nodes involved:**
- `Set Full Newsletter`
- `Create Newsletter File`
- `Upload Newsletter File`
- `Share Newsletter Message`
- `Generate Viral Video Ideas`
- `Create Video Ideas File`
- `Upload Video Ideas File`
- `Share Video Ideas Message`
- Sticky notes:
  - “Format Full Blog Post & Generate Video Ideas”
  - “Final Part”

**Node details:**

1) **Set Full Newsletter**
- **Type/role:** Set – constructs `full_newsletter_content` markdown:
  - H1 title = subject line
  - pre-header text
  - intro section
  - story sections
  - “The Shortlist” heading + other stories content

2) **Create Newsletter File** → **Upload Newsletter File**
- **ConvertToFile:** `toText` from `full_newsletter_content`, filename = `YYYY-MM-DD`.
- **Slack file upload:** uploads file; output includes `permalink`.

3) **Share Newsletter Message**
- Posts permalink to Slack channel.

4) **Generate Viral Video Ideas**
- **Type/role:** LLM chain that creates 3 video concept outlines from the blog content.
- **Model:** connected to “Anthropic Chat Model” in this workflow.
- **Output usage:** downstream expects a property named `text` (see `Create Video Ideas File` uses `sourceProperty: =text`). If the chain outputs under `output` (typical for chain nodes), this may fail unless n8n maps the result to `text`.
- **Risk:** misconfigured source property leading to empty file.

5) **Create Video Ideas File** → **Upload Video Ideas File** → **Share Video Ideas Message**
- Converts the `text` field to file and uploads it to Slack.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| the_rundown_ai_trigger | rssFeedReadTrigger | Poll newsletter RSS | — | normalize_the_rundown_ai | RSS feeds; RSS Feed (https://rss.app/) |
| bens_bites_trigger | rssFeedReadTrigger | Poll newsletter RSS | — | normalize_bens_bites | RSS feeds; RSS Feed (https://rss.app/) |
| the_neuron_trigger | rssFeedReadTrigger | Poll newsletter RSS | — | normalize_neuron | RSS feeds; RSS Feed (https://rss.app/) |
| futurepedia_trigger | rssFeedReadTrigger | Poll newsletter RSS | — | normalize_futurepedia | RSS feeds; RSS Feed (https://rss.app/) |
| superhuman_trigger | rssFeedReadTrigger | Poll newsletter RSS | — | normalize_superhuman | RSS feeds; RSS Feed (https://rss.app/) |
| taaft_trigger | rssFeedReadTrigger | Poll newsletter RSS | — | normalize_taaft | RSS feeds; RSS Feed (https://rss.app/) |
| google_news_trigger | scheduleTrigger | Schedule Google News feed pull | — | fetch_google_news_feed | RSS feeds |
| fetch_google_news_feed | httpRequest | Fetch rss.app JSON | google_news_trigger | split_google_news_items | RSS feeds |
| split_google_news_items | splitOut | Split items[] | fetch_google_news_feed | normalize_google_news_articles | RSS feeds |
| normalize_google_news_articles | code | Normalize Google News items | split_google_news_items | Get Identity | RSS feeds |
| hacker_news_trigger | scheduleTrigger | Schedule HN feed pull | — | fetch_hacker_news_feed | RSS feeds |
| fetch_hacker_news_feed | httpRequest | Fetch rss.app JSON | hacker_news_trigger | split_hacker_news_items | RSS feeds |
| split_hacker_news_items | splitOut | Split items[] | fetch_hacker_news_feed | normalize_hacker_news_articles | RSS feeds |
| normalize_hacker_news_articles | code | Normalize HN items | split_hacker_news_items | Get Identity | RSS feeds |
| reddit_artificial_inteligence_trigger | scheduleTrigger | Schedule subreddit feed pull | — | fetch_reddit_artificial_inteligence_feed | RSS feeds |
| fetch_reddit_artificial_inteligence_feed | httpRequest | Fetch rss.app JSON | reddit_artificial_inteligence_trigger | split_reddit_artificial_inteligence_items | RSS feeds |
| split_reddit_artificial_inteligence_items | splitOut | Split items[] | fetch_reddit_artificial_inteligence_feed | get_reddit_artificial_inteligence_items | RSS feeds |
| get_reddit_artificial_inteligence_items | reddit (disabled) | (Optional) fetch full post | split_reddit_artificial_inteligence_items | filter_reddit_artificial_inteligence_items | RSS feeds |
| filter_reddit_artificial_inteligence_items | filter | Exclude non-article links | get_reddit_artificial_inteligence_items | normalize_reddit_artificial_inteligence_items | RSS feeds |
| normalize_reddit_artificial_inteligence_items | code | Normalize subreddit items | filter_reddit_artificial_inteligence_items | Get Identity | RSS feeds |
| reddit_open_ai_trigger | scheduleTrigger | Schedule subreddit feed pull | — | fetch_reddit_open_ai_feed | RSS feeds |
| fetch_reddit_open_ai_feed | httpRequest | Fetch rss.app JSON | reddit_open_ai_trigger | split_reddit_open_ai_items | RSS feeds |
| split_reddit_open_ai_items | splitOut | Split items[] | fetch_reddit_open_ai_feed | get_reddit_open_ai_items | RSS feeds |
| get_reddit_open_ai_items | reddit (disabled) | (Optional) fetch full post | split_reddit_open_ai_items | filter_reddit_open_ai_items | RSS feeds |
| filter_reddit_open_ai_items | filter | Exclude non-article links | get_reddit_open_ai_items | normalize_reddit_open_ai_items | RSS feeds |
| normalize_reddit_open_ai_items | code | Normalize subreddit items | filter_reddit_open_ai_items | Get Identity | RSS feeds |
| reddit_artificial_trigger | scheduleTrigger | Schedule subreddit feed pull | — | fetch_reddit_artificial_feed | RSS feeds |
| fetch_reddit_artificial_feed | httpRequest | Fetch rss.app JSON | reddit_artificial_trigger | split_reddit_artificial_items | RSS feeds |
| split_reddit_artificial_items | splitOut | Split items[] | fetch_reddit_artificial_feed | get_reddit_artificial_items | RSS feeds |
| get_reddit_artificial_items | reddit (disabled) | (Optional) fetch full post | split_reddit_artificial_items | filter_reddit_artificial_items | RSS feeds |
| filter_reddit_artificial_items | filter | Exclude non-article links | get_reddit_artificial_items | normalize_reddit_artificial_items | RSS feeds |
| normalize_reddit_artificial_items | code | Normalize subreddit items | filter_reddit_artificial_items | Get Identity | RSS feeds |
| blog_meta_ai_trigger | scheduleTrigger | Schedule blog feed pull | — | fetch_blog_meta_ai_feed | RSS feeds |
| fetch_blog_meta_ai_feed | httpRequest | Fetch rss.app JSON | blog_meta_ai_trigger | split_blog_meta_ai_items | RSS feeds |
| split_blog_meta_ai_items | splitOut | Split items[] | fetch_blog_meta_ai_feed | normalize_blog_meta_ai_articles | RSS feeds |
| normalize_blog_meta_ai_articles | code | Normalize blog items | split_blog_meta_ai_items | Get Identity | RSS feeds |
| blog_cloudflare_ai_trigger | scheduleTrigger | Schedule blog feed pull | — | fetch_blog_cloudflare_ai_feed | RSS feeds |
| fetch_blog_cloudflare_ai_feed | httpRequest | Fetch rss.app JSON | blog_cloudflare_ai_trigger | split_blog_cloudflare_ai_items | RSS feeds |
| split_blog_cloudflare_ai_items | splitOut | Split items[] | fetch_blog_cloudflare_ai_feed | normalize_blog_cloudflare_ai_articles | RSS feeds |
| normalize_blog_cloudflare_ai_articles | code | Normalize blog items | split_blog_cloudflare_ai_items | Get Identity | RSS feeds |
| blog_anthropic_ai_trigger | scheduleTrigger | Schedule blog feed pull | — | fetch_blog_anthropic_ai_feed | RSS feeds |
| fetch_blog_anthropic_ai_feed | httpRequest | Fetch rss.app JSON | blog_anthropic_ai_trigger | split_blog_anthropic_ai_items | RSS feeds |
| split_blog_anthropic_ai_items | splitOut | Split items[] | fetch_blog_anthropic_ai_feed | normalize_blog_anthropic_ai_articles | RSS feeds |
| normalize_blog_anthropic_ai_articles | code | Normalize blog items | split_blog_anthropic_ai_items | Get Identity | RSS feeds |
| blog_google_ai_trigger | scheduleTrigger | Schedule blog feed pull | — | fetch_blog_google_ai_feed | RSS feeds |
| fetch_blog_google_ai_feed | httpRequest | Fetch rss.app JSON | blog_google_ai_trigger | split_blog_google_ai_items | RSS feeds |
| split_blog_google_ai_items | splitOut | Split items[] | fetch_blog_google_ai_feed | normalize_blog_google_ai_articles | RSS feeds |
| normalize_blog_google_ai_articles | code | Normalize blog items | split_blog_google_ai_items | Get Identity | RSS feeds |
| blog_open_ai_trigger | scheduleTrigger | Schedule blog feed pull | — | fetch_blog_open_ai_feed | RSS feeds |
| fetch_blog_open_ai_feed | httpRequest | Fetch rss.app JSON | blog_open_ai_trigger | split_blog_open_ai_items | RSS feeds |
| split_blog_open_ai_items | splitOut | Split items[] | fetch_blog_open_ai_feed | normalize_blog_open_ai_articles | RSS feeds |
| normalize_blog_open_ai_articles | code | Normalize blog items | split_blog_open_ai_items | Get Identity | RSS feeds |
| blog_nvidia_ai_trigger | scheduleTrigger | Schedule blog feed pull | — | fetch_blog_nvidia_ai_feed | RSS feeds |
| fetch_blog_nvidia_ai_feed | httpRequest | Fetch rss.app JSON | blog_nvidia_ai_trigger | split_blog_nvidia_ai_items | RSS feeds |
| split_blog_nvidia_ai_items | splitOut | Split items[] | fetch_blog_nvidia_ai_feed | normalize_blog_nvidia_ai_articles | RSS feeds |
| normalize_blog_nvidia_ai_articles | code | Normalize blog items | split_blog_nvidia_ai_items | Get Identity | RSS feeds |
| google_cloud_events_trigger | scheduleTrigger | Schedule events feed pull | — | fetch_google_cloud_events_feed | RSS feeds |
| fetch_google_cloud_events_feed | httpRequest | Fetch rss.app JSON | google_cloud_events_trigger | split_google_cloud_events_items | RSS feeds |
| split_google_cloud_events_items | splitOut | Split items[] | fetch_google_cloud_events_feed | normalize_google_cloud_events | RSS feeds |
| normalize_google_cloud_events | code | Normalize events items | split_google_cloud_events_items | Get Identity | RSS feeds |
| normalize_the_rundown_ai | set | Add source metadata | the_rundown_ai_trigger | Get Identity | RSS feeds |
| normalize_bens_bites | set | Add source metadata | bens_bites_trigger | Get Identity | RSS feeds |
| normalize_neuron | set | Add source metadata | the_neuron_trigger | Get Identity | RSS feeds |
| normalize_futurepedia | set | Add source metadata | futurepedia_trigger | Get Identity | RSS feeds |
| normalize_superhuman | set | Add source metadata | superhuman_trigger | Get Identity | RSS feeds |
| normalize_taaft | set | Add source metadata | taaft_trigger | Get Identity | RSS feeds |
| Get Identity | set | Canonical fields + upload filename | all normalize_* | Limit Number of Items from Each Source | Limiting Content |
| Limit Number of Items from Each Source | limit | Throttle volume per source | Get Identity | Delay 10s | Limiting Content |
| Delay 10s | wait | Rate limiting | Limit Number of Items... | Scrape URL | Limiting Content |
| Scrape URL | executeWorkflow | Scrape article content (sub-workflow) | Delay 10s | Filter Scrape Errors | Scraping Content |
| Filter Scrape Errors | filter | Drop scrape failures | Scrape URL | evaluate_content; Merge Every Source | Scraping Content |
| o3-mini | lmChatOpenAi | LLM model for relevance eval | — | evaluate_content | Scraping Content |
| is_revelant_content_parser | outputParserStructured | Parse relevance JSON | — | evaluate_content | Scraping Content |
| evaluate_content | chainLlm | LLM relevance check | Filter Scrape Errors | Merge Every Source | Scraping Content |
| Merge Every Source | merge | Merge scrape + LLM relevance | Filter Scrape Errors; evaluate_content | Ensure Relevant | Scraping Content |
| Ensure Relevant | filter | Keep AI-relevant items | Merge Every Source | Log to Google Sheets | Scraping Content |
| Log to Google Sheets | googleSheets | Append/update tracker | Ensure Relevant | — | Scraping Content |
| Blog Trigger | scheduleTrigger | Daily edition start | — | Get Stories | Blog Generator Workflow |
| Get Stories | googleSheets | Read tracker rows | Blog Trigger | Aggregate Stories | Blog Generator Workflow |
| Aggregate Stories | aggregate | Combine rows into one payload | Get Stories | Stories Prompt | Blog Generator Workflow |
| Stories Prompt | set | Build top-stories prompt | Aggregate Stories | pick_top_stories | Pick Top Stories |
| pick_top_stories | chainLlm | Select top stories (JSON) | Stories Prompt | Set Current Stories | Pick Top Stories |
| top_stories_parser | outputParserStructured | Validate top stories schema | — | top_stories_auto_parser | Pick Top Stories |
| top_stories_auto_parser | outputParserAutofixing | Fix invalid JSON | top_stories_parser | pick_top_stories; edit_top_stories | Pick Top Stories |
| Set Current Stories | set | Store selection as current_stories | pick_top_stories / edit_top_stories | Share Selected Stories | Pick Top Stories |
| Share Selected Stories | slack | Post selected stories to Slack | Set Current Stories | Share Stories Reasoning | Select top 3-5 stories |
| Share Stories Reasoning | slack | Post reasoning in thread | Share Selected Stories | Share Stories Approval Feedback | Select top 3-5 stories |
| Share Stories Approval Feedback | slack (sendAndWait) | Collect approval/feedback | Share Stories Reasoning | extract_stories_approval_feedback | Select top 3-5 stories |
| extract_stories_approval_feedback | informationExtractor | Classify approval vs feedback | Share Stories Approval Feedback | Check Stories Feedback | Select top 3-5 stories |
| Check Stories Feedback | if | Branch on approved | extract_stories_approval_feedback | Subject Examples; edit_top_stories | Select top 3-5 stories |
| edit_top_stories | chainLlm | Apply feedback to selections | Check Stories Feedback | Set Current Stories | Select top 3-5 stories |
| Subject Examples | set | Store subject line examples | Check Stories Feedback | Set Subject Line Prompt | Write Title |
| Set Subject Line Prompt | set | Build subject/preheader prompt | Subject Examples | write_subject_line | Write Title |
| write_subject_line | chainLlm | Generate subject + preheader | Set Subject Line Prompt | Set Current Subject Line | Write Title |
| subject_line_parser | outputParserStructured | Validate subject schema | — | subject_line_auto_parser | Write Title |
| subject_line_auto_parser | outputParserAutofixing | Fix invalid JSON | subject_line_parser | write_subject_line; edit_subject_line | Write Title |
| Set Current Subject Line | set | Store subject output | write_subject_line / edit_subject_line | Share Subject Line | Write Title |
| Share Subject Line | slack | Post subject/preheader | Set Current Subject Line | Share Subject Line Reasoning | Write Title |
| Share Subject Line Reasoning | slack | Post reasoning in thread | Share Subject Line | Share Subject Line Approval Feedback | Write Title |
| Share Subject Line Approval Feedback | slack (sendAndWait) | Collect approval/feedback | Share Subject Line Reasoning | extract_subject_line_approval_feedback | Write Title |
| extract_subject_line_approval_feedback | informationExtractor | Classify approval vs feedback | Share Subject Line Approval Feedback | Check Subject Line Feedback | Write Title |
| Check Subject Line Feedback | if | Branch on approved | extract_subject_line_approval_feedback | Set Selected Stories; edit_subject_line | Write Title |
| edit_subject_line | chainLlm | Apply feedback to title/preheader | Check Subject Line Feedback | Set Current Subject Line | Write Title |
| Set Selected Stories | set | Combine approved subject + stories | Check Subject Line Feedback | Split Stories | Important Segement |
| Split Stories | splitOut | One item per story | Set Selected Stories | Iterate Stories | Important Segement |
| Iterate Stories | splitInBatches | Loop through stories | Split Stories; set_story_segment | Set Story Segments; Set Current Segement | Important Segement |
| Set Current Segement | set | Store current_story | Iterate Stories | Split Content IDs | Important Segement |
| Split Content IDs | splitOut | Split identifiers[] | Set Current Segement | Aggregate Segment Text Content | Important Segement |
| Aggregate Segment Text Content | aggregate | Collect text per identifier | Split Content IDs | Check External URLs | Important Segement |
| Check External URLs | if | Branch if external links exist | Aggregate Segment Text Content | Set Segment External Source Links; write_segment_content | Important Segement |
| Set Segment External Source Links | set | Copy external links array | Check External URLs | Split Segment External Source URLs | Important Segement |
| Split Segment External Source URLs | splitOut | One item per external URL | Set Segment External Source Links | Scrape Segment External Source URL | Important Segement |
| Scrape Segment External Source URL | executeWorkflow | Scrape external source (sub-workflow) | Split Segment External Source URLs | Filter Segment External Source Errors | Important Segement |
| Filter Segment External Source Errors | filter | Keep only successful scrapes | Scrape Segment External Source URL | Aggregate Segment External Source Content | Important Segement |
| Aggregate Segment External Source Content | aggregate | Collect external source content | Filter Segment External Source Errors | write_segment_content | Important Segement |
| write_segment_content | chainLlm | Write story segment (JSON) | Check External URLs; Aggregate Segment External Source Content | share_segment_msg | Important Segement |
| story_segment_output_parser | outputParserStructured | Validate segment schema | — | story_segment_auto_parser | Important Segement |
| story_segment_auto_parser | outputParserAutofixing | Fix invalid JSON | story_segment_output_parser | write_segment_content | Important Segement |
| share_segment_msg | slack | Post segment to Slack | write_segment_content | set_story_segment | Important Segement |
| set_story_segment | set | Store segment for loop | share_segment_msg | Iterate Stories | Important Segement |
| Set Story Segments | set | Extract section content string | Iterate Stories | Aggregate Story Sections | Write Intro Section |
| Aggregate Story Sections | aggregate | Collect all sections | Set Story Segments | Set Combined Sections Content | Write Intro Section |
| Set Combined Sections Content | set | Join sections with separators | Aggregate Story Sections | write_intro | Write Intro Section |
| write_intro | chainLlm | Write intro section (JSON) | Set Combined Sections Content | write_other_top_stories | Write Intro Section |
| intro_parser | outputParserStructured | Validate intro schema | — | intro_auto_parser | Write Intro Section |
| intro_auto_parser | outputParserAutofixing | Fix invalid JSON | intro_parser | write_intro | Write Intro Section |
| write_other_top_stories | chainLlm | Write shortlist/quick hits | write_intro | Set Full Newsletter | Write "The Shortlist" Section |
| other_top_stories_parser | outputParserStructured | Validate shortlist schema | — | other_top_stories_auto_parser | Write "The Shortlist" Section |
| other_top_stories_auto_parser | outputParserAutofixing | Fix invalid JSON | other_top_stories_parser | write_other_top_stories | Write "The Shortlist" Section |
| Set Full Newsletter | set | Assemble final markdown | write_other_top_stories | Create Newsletter File; Generate Viral Video Ideas | Final Part |
| Create Newsletter File | convertToFile | Convert markdown to file | Set Full Newsletter | Upload Newsletter File | Final Part |
| Upload Newsletter File | slack (file) | Upload newsletter file | Create Newsletter File | Share Newsletter Message | Final Part |
| Share Newsletter Message | slack | Post permalink | Upload Newsletter File | — | Final Part |
| Generate Viral Video Ideas | chainLlm | Generate 3 video concepts | Set Full Newsletter | Create Video Ideas File | Format Full Blog Post & Generate Video Ideas |
| Create Video Ideas File | convertToFile | Convert ideas to file | Generate Viral Video Ideas | Upload Video Ideas File | Format Full Blog Post & Generate Video Ideas |
| Upload Video Ideas File | slack (file) | Upload video ideas file | Create Video Ideas File | Share Video Ideas Message | Format Full Blog Post & Generate Video Ideas |
| Share Video Ideas Message | slack | Post permalink | Upload Video Ideas File | — | Format Full Blog Post & Generate Video Ideas |
| OpenAI Chat Model | lmChatOpenAi | Shared OpenAI model for many LLM nodes | — | multiple chain nodes |  |
| Anthropic Chat Model | lmChatAnthropic | Shared Anthropic model for parsers + video ideas | — | multiple parser/LLM nodes |  |
| Sticky Note | stickyNote | Comment | — | — | ## RSS feeds… |
| Sticky Note1 | stickyNote | Comment | — | — | ## Scraping Content |
| Sticky Note2 | stickyNote | Comment | — | — | ## Write Intro Section |
| Sticky Note3 (disabled) | stickyNote | Comment | — | — | ## Iterate Over & Write Each Selected Story |
| Sticky Note4 | stickyNote | Comment | — | — | ## Write "The Shortlist" Section |
| Sticky Note5 | stickyNote | Comment | — | — | ## Format Full Blog Post & Generate Video Ideas |
| Sticky Note6 | stickyNote | Comment | — | — | ## Pick Top Stories |
| Sticky Note7 | stickyNote | Comment | — | — | ## Write Title |
| Sticky Note8 | stickyNote | Comment | — | — | ## 1. Retrieve Blog Content |
| Sticky Note9 | stickyNote | Comment | — | — | ### How it works… |
| Sticky Note10 | stickyNote | Comment | — | — | RSS Feed… https://rss.app/ |
| Sticky Note11 | stickyNote | Comment | — | — | ## Limiting Content… |
| Sticky Note12 | stickyNote | Comment | — | — | ## Blog Generator Workflow… |
| Sticky Note13 | stickyNote | Comment | — | — | ## Select top 3-5 stories… |
| Sticky Note14 | stickyNote | Comment | — | — | ## Important Segement… |
| Sticky Note15 | stickyNote | Comment | — | — | ## Final Part… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create all credentials**
   1. OpenAI API credential (used by “OpenAI Chat Model” and `o3-mini`).
   2. Anthropic API credential (used by “Anthropic Chat Model” and some auto-parsers / video ideas).
   3. Google Sheets OAuth2 credential (for `Log to Google Sheets`, `Get Stories`).
   4. Slack OAuth credential (for all Slack nodes including “send and wait” interactive steps).

2) **Create the ingestion triggers and normalizers**
   1. Add RSS Feed Read Trigger nodes for each newsletter feed URL:
      - the-rundown-ai, Ben’s Bites, The Neuron, Futurepedia, Superhuman, TAAFT.
      - Set polling to every 4 minutes.
   2. After each RSS trigger, add a **Set** node to add:
      - `sourceName`, `feedType: "newsletter"`, `feedUrl` (archive URL).
   3. Add schedule triggers (every 3h) for Google News and Hacker News.
      - Add HTTP Request nodes to fetch rss.app JSON URLs.
      - Add Split Out node splitting field `items`.
      - Add Code node to normalize each item into:
        `sourceName,title,creator,link,pubDate,isoDate,feedType,feedUrl`.
   4. Add schedule triggers (every 3h) + HTTP Request + Split Out + Filter + Code for each subreddit feed.
      - Keep the “Reddit API get” node optional/disabled.
      - Filter out unwanted destinations (reddit/youtube/x/github/redd.it).
   5. Add schedule triggers (every 4h) + HTTP Request + Split Out + Code for each AI-company blog feed.
   6. Add schedule triggers (every 4h) + HTTP Request + Split Out + Code for events feed.
      - Fix `sourceName` to an events-appropriate name.

3) **Unify all sources into a single scraping pipeline**
   1. Connect every normalizer (Set/Code outputs) into a single downstream **Get Identity** Set node.
   2. Configure **Get Identity** to map:
      - `title` from item title
      - `url` from item link
      - `authors`, `date`, `publishedTimestamp`, `sourceName`, `feedType`, `feedUrl`
      - `uploadFileName` slug formula (date/title/sourceName).
   3. Add a **Limit** node (configure explicit limit, e.g., 5–20) after Get Identity.
   4. Add a **Wait** node (10 seconds).
   5. Add an **Execute Workflow** node named “Scrape URL” that calls a separate workflow:
      - Sub-workflow name: “AI News Aggregator - Scrape Url”
      - Input: `url` (string).
      - Expected output: object containing `data.json.content` and `data.metadata` (and optionally `markdown`).
   6. Add a **Filter** node to remove scrape errors (`$json.error` must not exist).

4) **Add relevance evaluation and logging**
   1. Add an OpenAI model node `o3-mini`.
   2. Add a LangChain **Chain LLM** node `evaluate_content`:
      - Prompt uses scraped content (`$json.data.json.content`) and asks for relevance to AI newsletter.
      - Attach a **Structured Output Parser** expecting `{ chainOfThought: string, is_revelant_content: boolean }`.
   3. Add a **Merge** node “combine by position” to merge scraped item + LLM output.
      - (Recommended improvement: merge by key like URL instead of by position to avoid misalignment.)
   4. Add a **Filter** node `Ensure Relevant` where `$json.output.is_revelant_content` is true.
   5. Add **Google Sheets** node `Log to Google Sheets`:
      - Operation: Append or Update
      - Match column: Title
      - Map URL/Title/Source/Content/Published/External Sources from the scrape metadata/content.

5) **Create the daily blog/newsletter generator chain**
   1. Add **Schedule Trigger** “Blog Trigger” to run daily at your target hour.
   2. Add Google Sheets node `Get Stories` to read the tracker sheet.
   3. Add Aggregate node `Aggregate Stories` to aggregate all item data.
   4. Add Set node `Stories Prompt` building `select_top_stories_prompt` using the aggregated data.
   5. Add LLM chain `pick_top_stories` (OpenAI model) with JSON-only instructions and structured parsing + autofix parser.
   6. Add Set `Set Current Stories` storing `$json.output` as `current_stories`.
   7. Add Slack nodes:
      - `Share Selected Stories` (posts list)
      - `Share Stories Reasoning` (thread reply)
      - `Share Stories Approval Feedback` (sendAndWait, free text)
   8. Add LLM Information Extractor node to extract `{approved, feedback}` from Slack reply.
   9. Add IF node: if approved true → proceed; else route to `edit_top_stories` then back to `Set Current Stories` (and re-share).

6) **Subject line approval loop**
   1. Add Set node `Subject Examples` (optional).
   2. Add Set node `Set Subject Line Prompt`.
   3. Add LLM chain `write_subject_line` with structured parser + autofix parser.
   4. Add Set node `Set Current Subject Line`.
   5. Add Slack nodes:
      - `Share Subject Line`
      - `Share Subject Line Reasoning` (thread)
      - `Share Subject Line Approval Feedback` (sendAndWait)
   6. Add Information Extractor + IF branching:
      - Approved → proceed to drafting
      - Feedback → `edit_subject_line` → back to `Set Current Subject Line` and re-share.

7) **Draft story segments**
   1. Add Set node `Set Selected Stories` combining approved subject line + approved stories.
   2. Add SplitOut node splitting `top_selected_stories`.
   3. Add SplitInBatches node `Iterate Stories`.
   4. For each story:
      - Set `Set Current Segement`
      - SplitOut identifiers
      - Aggregate text content for those identifiers (ensure your dataset actually includes `content_item` per identifier)
      - IF external links exist:
        - Split external URLs
        - Execute Workflow scrape sub-workflow per URL
        - Filter errors and aggregate external content
      - LLM chain `write_segment_content` with parsers
      - Slack `share_segment_msg`
      - Set `set_story_segment` and loop to next batch item

8) **Combine sections + intro + shortlist**
   1. Set `Set Story Segments` to extract the segment markdown.
   2. Aggregate all segments then join them with separators in `Set Combined Sections Content`.
   3. LLM chain `write_intro` with parsers.
   4. LLM chain `write_other_top_stories` with parsers (must exclude already covered topics).

9) **Assemble and deliver**
   1. Set `Set Full Newsletter` to assemble markdown.
   2. Convert to file → Slack upload → Slack permalink message.
   3. Generate video ideas → convert to file → Slack upload → Slack permalink message.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “You can select your favorite feeds using this site: https://rss.app/” | Source discovery and RSS-to-JSON feed generation |
| How it works: ingestion → filtering → drafting → approval → revisions | Included as a sticky note explaining overall flow |
| Setup steps: connect OpenAI/Anthropic/Sheets/Slack; update Sheet ID, Slack channel ID; customize prompts | Operational setup guidance |

---

**Disclaimer (as provided):**  
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.