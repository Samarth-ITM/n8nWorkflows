Summarize Japanese AI engineering articles from Qiita and note RSS to Slack

https://n8nworkflows.xyz/workflows/summarize-japanese-ai-engineering-articles-from-qiita-and-note-rss-to-slack-14980


# Summarize Japanese AI engineering articles from Qiita and note RSS to Slack

# Technical Reference: Japanese AI Engineering Article Summarizer

### 1. Workflow Overview
This workflow automates the discovery, filtering, and summarization of technical articles from Japanese platforms (Qiita and note). It transforms raw RSS feeds into a high-quality daily digest delivered to Slack, ensuring that only the most practical and high-value engineering content is surfaced.

The logic is organized into six functional blocks:
- **1.1 Input Reception:** Scheduled triggering and multi-source RSS data collection.
- **1.2 Data Filtering & Structuring:** Time-based filtering (last 24 hours) and data normalization.
- **1.3 AI-Based Scoring:** Use of a LLM to rank articles based on technical utility and select the top 10.
- **1.4 Content Retrieval:** Fetching the full HTML of selected articles and cleaning the text.
- **1.5 AI Summarization:** Detailed extraction of merits, demerits, and use cases using a second LLM.
- **1.6 Notification:** Formatting the final structured data into a human-readable Slack message.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
**Overview:** Initiates the process daily and gathers raw article data from multiple Japanese technical sources.
- **Nodes Involved:** `Trigger: Schedule Workflow`, `Fetch RSS: Qiita Popular Feed`, `Fetch RSS: note Engineer Hashtag Feed`, `Merge Feeds`.
- **Node Details:**
    - **Trigger: Schedule Workflow:** (Schedule Trigger) Set to trigger every day at 07:00.
    - **Fetch RSS: Qiita Popular Feed:** (RSS Feed Read) Pulls from the Qiita popular items Atom feed.
    - **Fetch RSS: note Engineer Hashtag Feed:** (RSS Feed Read) Pulls from the note.com "Engineer" hashtag RSS.
    - **Merge Feeds:** (Merge) Combines streams from both RSS sources into a single list for downstream processing.

#### 2.2 Data Filtering & Structuring
**Overview:** Removes outdated content and standardizes the data schema.
- **Nodes Involved:** `Filter: Last 24 Hours`, `Transform: Format RSS Fields`, `Aggregate: Build Article List`.
- **Node Details:**
    - **Filter: Last 24 Hours:** (Code) A JavaScript filter that compares `isoDate` or `pubDate` against the current time. Only items published within the last 24 hours are kept.
    - **Transform: Format RSS Fields:** (Set) Maps diverse RSS fields into a uniform structure: `title`, `summary` (from content), and `link`.
    - **Aggregate: Build Article List:** (Aggregate) Collects all individual filtered articles into a single array named `articles` to be processed as a batch by the AI.

#### 2.3 AI-Based Article Scoring and Selection
**Overview:** Evaluates the "quality" of articles to avoid noise and select only high-value technical content.
- **Nodes Involved:** `AI: Score Articles (Top 10)`, `Model: Gemini`, `Parser: Structured JSON`, `Split: Individual Articles`.
- **Node Details:**
    - **AI: Score Articles (Top 10):** (Chain LLM) Uses a detailed prompt to act as an "AI Evaluator for Junior Engineers." It scores articles 0-100 based on practical utility, reproducibility, and technical value.
    - **Model: Gemini:** (Google Gemini Chat Model) The underlying LLM providing the reasoning and scoring.
    - **Parser: Structured JSON:** (Structured Output Parser) Forces the AI to return a JSON array containing `title`, `link`, `score`, and `reason`.
    - **Split: Individual Articles:** (Split Out) Breaks the AI's top 10 list back into individual items for per-article processing.

#### 2.4 Content Retrieval & Preparation
**Overview:** Retrieves the actual content of the selected articles for deep analysis.
- **Nodes Involved:** `Prepare: Extract Article Link for HTTP`, `Fetch: Article HTML`, `Extract: Clean Text from HTML`.
- **Node Details:**
    - **Prepare: Extract Article Link for HTTP:** (Set) Isolates the `link` variable for the HTTP request.
    - **Fetch: Article HTML:** (HTTP Request) Performs a GET request to the article URL.
    - **Extract: Clean Text from HTML:** (Code) A JS script that removes `<script>`, `<style>`, and HTML tags. It filters for lines longer than 20 characters and truncates the final text to 3,000 characters to fit LLM context windows.

#### 2.5 AI Article Summarization
**Overview:** Generates a structured analysis of the cleaned article text.
- **Nodes Involved:** `AI: Generate Summary`, `Model: OpenAI`.
- **Node Details:**
    - **AI: Generate Summary:** (AI Agent) Prompted to be a "Data Formatting AI." It extracts: `summary`, `target` (audience), `use_case`, `merit` (200-400 chars), and `demerit` (200-400 chars).
    - **Model: OpenAI:** (OpenAI Chat Model) Uses the `gpt-4.1-nano` model to ensure high-quality synthesis.

#### 2.6 Notification
**Overview:** Converts the JSON analysis into a formatted Slack post.
- **Nodes Involved:** `Format: Slack Message`, `Notify: Send to Slack`.
- **Node Details:**
    - **Format: Slack Message:** (Code) Maps the AI's JSON output into a structured string template using markers (■, 【要約】, etc.).
    - **Notify: Send to Slack:** (Slack) Sends the formatted text to a specific channel (`gmailアラート2`) via OAuth2.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Trigger: Schedule Workflow | Schedule Trigger | Entry Point | - | RSS Fetchers | STEP1: Data Collection |
| Fetch RSS: Qiita Popular Feed | RSS Feed Read | Data Source | Trigger | Merge Feeds | STEP1: Data Collection |
| Fetch RSS: note Engineer Hashtag Feed | RSS Feed Read | Data Source | Trigger | Merge Feeds | STEP1: Data Collection |
| Merge Feeds | Merge | Stream Union | RSS Fetchers | Filter: 24h | STEP1: Data Collection |
| Filter: Last 24 Hours | Code | Time Filter | Merge Feeds | Transform: RSS | STEP2: Data Filtering & Structuring |
| Transform: Format RSS Fields | Set | Normalization | Filter: 24h | Aggregate | STEP2: Data Filtering & Structuring |
| Aggregate: Build Article List | Aggregate | Batching | Transform: RSS | AI: Score | STEP2: Data Filtering & Structuring |
| AI: Score Articles (Top 10) | Chain LLM | Quality Ranking | Aggregate | Split: Individual | STEP3: AI-Based Article Scoring... |
| Model: Gemini | Chat Model | Intelligence | - | AI: Score | STEP3: AI-Based Article Scoring... |
| Parser: Structured JSON | Output Parser | Schema Enforcement | - | AI: Score | STEP3: AI-Based Article Scoring... |
| Split: Individual Articles | Split Out | De-batching | AI: Score | Prepare: Link | STEP3: AI-Based Article Scoring... |
| Prepare: Extract Article Link | Set | Variable Prep | Split: Indiv. | Fetch: HTML | STEP4: Content Retrieval... |
| Fetch: Article HTML | HTTP Request | Content Fetching | Prepare: Link | Extract: Clean | STEP4: Content Retrieval... |
| Extract: Clean Text from HTML | Code | HTML Stripping | Fetch: HTML | AI: Summary | STEP4: Content Retrieval... |
| AI: Generate Summary | AI Agent | Synthesis | Extract: Clean | Format: Slack | STEP5: AI Article Summarization |
| Model: OpenAI | Chat Model | Intelligence | - | AI: Summary | STEP5: AI Article Summarization |
| Format: Slack Message | Code | Text Templating | AI: Summary | Notify: Slack | STEP6: Notification |
| Notify: Send to Slack | Slack | Delivery | Format: Slack | - | STEP6: Notification |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Data Collection & Filtering
1. Create a **Schedule Trigger** set to daily at 7 AM.
2. Add two **RSS Feed Read** nodes:
   - URL 1: `https://qiita.com/popular-items/feed.atom`
   - URL 2: `https://note.com/hashtag/%E3%82%A8%E3%83%B3%E3%82%B8%E3%83%8B%E3%82%A2/rss`
3. Connect both to a **Merge** node.
4. Add a **Code** node to filter items where the difference between `now` and `isoDate`/`pubDate` is $\le 24$ hours.
5. Add a **Set** node to map `title`, `content` $\rightarrow$ `summary`, and `link`.
6. Add an **Aggregate** node to combine all items into a single field called `articles`.

#### Step 2: AI Scoring (The Ranker)
1. Create a **Chain LLM** node.
2. Attach a **Google Gemini Chat Model** (requires Google Palm API credentials).
3. Attach a **Structured Output Parser** with the following JSON schema: `[{ "title": "string", "link": "string", "score": number, "reason": "string" }]`.
4. Configure the prompt to evaluate articles for junior engineers, focusing on practicality and specific code examples, returning only the top 10.
5. Add a **Split Out** node to target the `output` field of the AI response.

#### Step 3: Content Extraction
1. Add a **Set** node to ensure the `link` is available.
2. Add an **HTTP Request** node to fetch the URL from the `link` field.
3. Add a **Code** node to clean the HTML:
   - Remove `<script>` and `<style>` tags.
   - Strip all remaining HTML tags.
   - Remove characters not in the range of Japanese/English/Numbers.
   - Filter lines $< 20$ characters.
   - Slice text to 3,000 characters.

#### Step 4: AI Summarization & Delivery
1. Create an **AI Agent** node.
2. Attach an **OpenAI Chat Model** (requires OpenAI API credentials, model `gpt-4.1-nano`).
3. Configure the prompt to return a JSON object with: `summary`, `target`, `use_case`, `merit` (200-400 chars), and `demerit` (200-400 chars).
4. Add a **Code** node to map this JSON into a string: `■ [Index]\n[Link]\n【要約】[Summary]...`
5. Add a **Slack** node using OAuth2 credentials, selecting the target channel and passing the formatted text.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Target Audience | Japanese engineers and learners |
| Core Value | Filters noise and forces AI to provide detailed merits/demerits |
| Required Credentials | Google Gemini API, OpenAI API, Slack OAuth2 |
| Data Source 1 | [Qiita Popular Feed](https://qiita.com/popular-items/feed.atom) |
| Data Source 2 | [note Engineer Hashtag](https://note.com/hashtag/%E3%82%A8%E3%83%B3%E3%82%B8%E3%83%8B%E3%82%A2/rss) |