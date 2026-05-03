Content Farming - : AI-Powered Blog Automation for WordPress 

https://n8nworkflows.xyz/workflows/content-farming-----ai-powered-blog-automation-for-wordpress--5230


# Content Farming - : AI-Powered Blog Automation for WordPress 

### 1. Workflow Overview

This workflow titled **"Content Farming - AI-Powered Blog Automation for WordPress"** automates the end-to-end generation and publication of AI-driven blog content focused on technology and AI news. It is designed to ingest daily tech news from multiple RSS feeds, filter and classify articles for relevance, analyze and extract content insights, generate SEO-optimized blog post ideas and outlines, draft full articles, enrich SEO metadata, create related images, and finally publish posts on WordPress.

The workflow targets content marketers, bloggers, and digital media teams who want to automate content production using AI to save time and improve SEO effectiveness. It integrates various AI models (OpenAI GPT-4o-mini, Langchain agents), MongoDB for state and data storage, and WordPress for publishing.

**Logical blocks and their roles:**

- **1.1 RSS Feed Ingestion & Initial Processing:** Fetch daily news articles from curated technology RSS feeds, split and normalize article fields.
- **1.2 Article Filtering & Classification:** Filter recent articles (published last 24h) and classify for AI relevance.
- **1.3 AI Trend Analysis & Blog Title Generation:** Aggregate AI-related articles, then generate multiple SEO-friendly, unique blog title ideas.
- **1.4 Article Research & Summarization:** Fetch full article content, extract key sentences and summarize with named entities and quotes.
- **1.5 Blog Title Scoring & Selection:** Rank generated blog titles based on uniqueness, SEO, and clickbait quality to select the best.
- **1.6 Blog Outline Generation:** Create a SEO-optimized blog post outline in markdown format based on the chosen title and keywords.
- **1.7 Blog Content Drafting:** Generate full blog article content using the outline, keywords, tone, and citations.
- **1.8 SEO Metadata & Image Generation:** Create meta titles, descriptions, alt text, URL slug, generate a featured image prompt, produce AI image, and upload.
- **1.9 WordPress Publishing:** Publish the blog post draft with image and SEO metadata on WordPress, update post status to publish.
- **1.10 Logging & Workflow Management:** Store intermediate states and data in MongoDB to track progress and avoid duplication.

---

### 2. Block-by-Block Analysis

---

#### 1.1 RSS Feed Ingestion & Initial Processing

**Overview:**  
This block triggers daily at noon, sets a list of technology news RSS feeds, reads each feed, splits the feeds into individual articles, normalizes article fields, and filters out older articles.

**Nodes Involved:**  
- Get Articles Daily (Schedule Trigger)  
- Set Tech News RSS Feeds  
- Split Out  
- Read RSS News Feeds  
- Filter  
- Set and Normalize Fields

**Node Details:**

- **Get Articles Daily**  
  - Type: Schedule Trigger  
  - Config: Triggers daily at 12:00 PM  
  - Inputs: None (trigger node)  
  - Outputs: Trigger event  
  - Failure modes: Scheduler downtime or misconfiguration  

- **Set Tech News RSS Feeds**  
  - Type: Set  
  - Config: Sets an array variable `rss` with 7 curated technology RSS feed URLs (BBC Tech, Wired, TechCrunch, AI news, etc.)  
  - Inputs: Trigger event  
  - Outputs: Object with RSS feeds array  
  - Edge cases: RSS URLs outdated or inaccessible  

- **Split Out**  
  - Type: Split Out  
  - Config: Splits the array field `rss` into individual items for parallel processing  
  - Inputs: List of RSS URLs  
  - Outputs: One RSS feed URL per item  
  - Edge cases: Empty feed array  

- **Read RSS News Feeds**  
  - Type: RSS Feed Read  
  - Config: Reads RSS feed from dynamic URL `{{$json.rss}}`  
  - Inputs: Single RSS feed URL  
  - Outputs: List of articles from feed  
  - Edge cases: Feed downtime, invalid RSS format, SSL errors  

- **Filter**  
  - Type: Filter  
  - Config: Filters articles published after `now - 1 day` using `isoDate` field  
  - Inputs: Articles from RSS feed  
  - Outputs: Only recent articles  
  - Edge cases: Missing or malformed date fields  

- **Set and Normalize Fields**  
  - Type: Set  
  - Config: Normalizes fields: title, content snippet, date, link, categories  
  - Inputs: Filtered articles  
  - Outputs: Cleaned article objects  
  - Edge cases: Missing content snippet or categories fields  

---

#### 1.2 Article Filtering & Classification

**Overview:**  
Classifies normalized articles into categories (specifically "AI") using a Langchain text classifier node, then aggregates classified items for further AI processing.

**Nodes Involved:**  
- Text Classifier  
- Aggregate

**Node Details:**

- **Text Classifier**  
  - Type: Langchain Text Classifier  
  - Config: Input text composed of title, content, and categories fields; classifies into category "AI" with fallback to discard  
  - Inputs: Normalized article JSON  
  - Outputs: Classified articles tagged with "AI" or discarded  
  - Edge cases: Classification failures, fallback discard removes relevant articles  

- **Aggregate**  
  - Type: Aggregate  
  - Config: Aggregates all classified articles into a single array under `data`  
  - Inputs: Classified article items  
  - Outputs: Single aggregated item for AI Agent input  

---

#### 1.3 AI Trend Analysis & Blog Title Generation

**Overview:**  
Feeds aggregated AI articles to a Langchain agent which generates 10 unique, SEO-friendly, long-tail blog title ideas with keywords and tones, then parses structured output.

**Nodes Involved:**  
- AI Agent  
- Structured Output Parser  
- Split Out2  
- Code (for preparing MongoDB insert)  
- MongoDB (insert new blog document)

**Node Details:**

- **AI Agent**  
  - Type: Langchain Agent  
  - Config: Prompt includes JSON array of trending AI articles; requests titles with main and related keywords, tone, SEO focus  
  - Inputs: Aggregated data from previous block  
  - Outputs: JSON with blog title ideas and keywords  
  - Edge cases: API failures, malformed response  

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser  
  - Config: Parses JSON array of titles and metadata from AI Agent  
  - Inputs: AI Agent raw output  
  - Outputs: Structured JSON for further processing  

- **Split Out2**  
  - Type: Split Out  
  - Config: Splits the `output` array of titles into individual items for ranking  

- **Code**  
  - Type: Code (JS)  
  - Config: Prepares document for MongoDB insertion by adding `completed_step: 2` flag  

- **MongoDB**  
  - Type: MongoDB Insert  
  - Config: Inserts blog title ideas document into `blog` collection  

---

#### 1.4 Article Research & Summarization

**Overview:**  
Retrieves full article content from original links, extracts main body text, summarizes key points, names, quotes, and generates a 2-paragraph summary with key elements for each article.

**Nodes Involved:**  
- MongoDB1 (query step 2)  
- Split Out1  
- Loop Over Items  
- Code10  
- HTTP Request (fetch article)  
- HTML (extract body content)  
- Transform news to MD (strip HTML, convert to markdown)  
- OpenAI Chat Model2  
- AI Agent1  
- Structured Output Parser1  
- Aggregate1  
- Code4  
- MongoDB2 (update step 3)

**Node Details:**

- **MongoDB1**  
  - Type: MongoDB Find  
  - Config: Queries documents with `completed_step: 2` to get blog ideas needing article research  

- **Split Out1**  
  - Type: Split Out  
  - Config: Splits `link_articles` array to process each article URL separately  

- **Loop Over Items**  
  - Type: Split In Batches  
  - Config: Processes articles one by one  

- **HTTP Request**  
  - Type: HTTP Request  
  - Config: Fetches full article HTML from each `link_articles` URL  

- **HTML**  
  - Type: HTML Extract  
  - Config: Extracts main article content by selecting `<body>` content, excluding images  

- **Transform news to MD**  
  - Type: Code  
  - Config: Cleans extracted HTML into plain markdown text  

- **OpenAI Chat Model2**  
  - Type: Langchain LM Chat OpenAI  
  - Config: Uses GPT-4o-mini to assist AI Agent1  

- **AI Agent1**  
  - Type: Langchain Agent  
  - Config: Extracts 3 factual sentences, writes 2-paragraph summary, key concepts, names, quotes from article markdown  

- **Structured Output Parser1**  
  - Type: Langchain Structured Output Parser  
  - Config: Parses AI Agent1's structured summary output  

- **Aggregate1**  
  - Type: Aggregate  
  - Config: Aggregates summaries back into the blog document  

- **Code4**  
  - Type: Code  
  - Config: Sanitizes article content and updates document with `completed_step: 3`  

- **MongoDB2**  
  - Type: MongoDB Update  
  - Config: Updates blog document with article summaries and step completion  

---

#### 1.5 Blog Title Scoring & Selection

**Overview:**  
Generates 5 compelling blog post titles for the core topic, then scores each title on uniqueness, SEO clarity, and clickbait quality, ranks them, and selects the best title.

**Nodes Involved:**  
- MongoDB3 (query step 3)  
- AI Agent2 (generate blog titles)  
- Structured Output Parser2  
- AI Agent3 (rate titles)  
- Structured Output Parser3  
- Code7 (ranking and selection)  
- MongoDB7 (update step 4)

**Node Details:**

- **MongoDB3**  
  - Type: MongoDB Find  
  - Config: Finds blog document with `completed_step: 3`  

- **AI Agent2**  
  - Type: Langchain Agent  
  - Config: Generates 5 blog post titles with emotional/curiosity angles based on blog article concept  

- **Structured Output Parser2**  
  - Type: Structured Output Parser  
  - Config: Parses generated titles JSON  

- **AI Agent3**  
  - Type: Langchain Agent  
  - Config: Rates each title with scores (uniqueness, SEO clarity, clickbait quality) and provides justification  

- **Structured Output Parser3**  
  - Type: Structured Output Parser  
  - Config: Parses title rating scores  

- **Code7**  
  - Type: Code  
  - Config: Calculates average score, sorts titles descending, selects top title, marks step 4 completed  

- **MongoDB7**  
  - Type: MongoDB Update  
  - Config: Updates blog document with ranked titles, selected title, core topic, and step completion  

---

#### 1.6 Blog Outline Generation

**Overview:**  
Generates a detailed SEO-optimized blog outline in Markdown format, structured with H1 and H2 headings, using the selected blog title, keywords, and related articles.

**Nodes Involved:**  
- AI Agent5  
- Structured Output Parser6  
- MongoDB8 (query step 7)  
- Code3 (clean and store outline)  
- MongoDB4 (update step 5)

**Node Details:**

- **AI Agent5**  
  - Type: Langchain Agent  
  - Config: Creates blog outline with specified structure (Intro, Background, Trend, Insight, Forecast, CTA), optimized for featured snippet  

- **Structured Output Parser6**  
  - Type: Structured Output Parser  
  - Config: Parses markdown outline JSON  

- **MongoDB8**  
  - Type: MongoDB Find  
  - Config: Query blog document where `completed_step: 7` (used later for image generation and publishing)  

- **Code3**  
  - Type: Code  
  - Config: Cleans markdown (removes code fences, escapes quotes), prepares for MongoDB update  

- **MongoDB4**  
  - Type: MongoDB Update  
  - Config: Stores cleaned outline and marks step 5 completed  

---

#### 1.7 Blog Content Drafting

**Overview:**  
Generates full blog article content (approx. 300-500 words per section) in Markdown, incorporating keywords, citations, analogies, future implications, and tone. Converts Markdown to HTML for WordPress publishing.

**Nodes Involved:**  
- AI Agent6  
- Structured Output Parser4  
- Code5  
- MongoDB5 (update step 6)  
- Code8 (Markdown to HTML converter)

**Node Details:**

- **AI Agent6**  
  - Type: Langchain Agent  
  - Config: Writes blog content based on title, keywords, tone, outline, and related articles; outputs full article in Markdown  

- **Structured Output Parser4**  
  - Type: Structured Output Parser  
  - Config: Parses blog article content JSON  

- **Code5**  
  - Type: Code  
  - Config: Cleans Markdown (removes fences, escapes quotes), sets step 6 completed  

- **MongoDB5**  
  - Type: MongoDB Update  
  - Config: Stores blog article content  

- **Code8**  
  - Type: Code  
  - Config: Converts Markdown blog article content to HTML for CMS posting (handles headings, bold, italic, links)  

---

#### 1.8 SEO Metadata & Image Generation

**Overview:**  
Generates SEO meta title, description, image alt text, URL slug, and SEO report. Creates an emotionally compelling image generation prompt, sends it to Leonardo AI for image creation, uploads generated image to WordPress.

**Nodes Involved:**  
- AI Agent7 (SEO metadata generation)  
- Structured Output Parser4  
- MongoDB6 (update step 7)  
- MongoDB8 (query step 7)  
- Code9 (create enriched image prompt)  
- HTTP Request1 (Leonardo AI generate image)  
- Wait (delay for image generation)  
- HTTP Request2 (check image generation status)  
- HTTP Request3 (download generated image)  
- Upload image2 (upload image to WordPress)  
- Code12 (update blog with image URL)  
- MongoDB15 (update step 8)

**Node Details:**

- **AI Agent7**  
  - Type: Langchain Agent  
  - Config: Creates meta title (max 60 chars), meta description, alt text, slug, and SEO report (keyword density, passive voice count, Flesch score)  

- **MongoDB6**  
  - Type: MongoDB Update  
  - Config: Stores SEO metadata and marks step 7 completed  

- **Code9**  
  - Type: Code  
  - Config: Enhances image alt text into a cinematic, emotionally compelling image prompt  

- **HTTP Request1**  
  - Type: HTTP Request  
  - Config: Sends prompt to Leonardo AI API for image generation (modelId and style specified)  

- **Wait**  
  - Type: Wait  
  - Config: Waits 1 minute for image generation to complete  

- **HTTP Request2**  
  - Type: HTTP Request  
  - Config: Polls Leonardo AI API for generation status and results  

- **HTTP Request3**  
  - Type: HTTP Request  
  - Config: Downloads generated image from Leonardo AI URL  

- **Upload image2**  
  - Type: HTTP Request  
  - Config: Uploads image binary data to WordPress media library with filename based on blog slug  

- **Code12**  
  - Type: Code  
  - Config: Updates blog document with image URL and marks step 8 completed  

- **MongoDB15**  
  - Type: MongoDB Update  
  - Config: Stores image URL and step 8 complete  

---

#### 1.9 WordPress Publishing

**Overview:**  
Creates WordPress draft post with title, slug, author, category, and HTML content, sets featured image, adds Yoast SEO metatags, then publishes the post and logs published URL and status.

**Nodes Involved:**  
- Wordpress1 (create post)  
- Set Image (set featured image)  
- Set metatag1 (update Yoast SEO metadata)  
- Wordpress2 (publish post)  
- Code14 (log published URL and status)  
- MongoDB16 (update step 9)

**Node Details:**

- **Wordpress1**  
  - Type: WordPress node  
  - Config: Creates new draft post with title, slug, status draft, authorId=3, categoryId=8, post content HTML, comments closed  

- **Set Image**  
  - Type: HTTP Request  
  - Config: Updates WordPress post to set featured image by post ID and media ID  

- **Set metatag1**  
  - Type: HTTP Request  
  - Config: Updates post meta with Yoast SEO fields (meta title and description)  

- **Wordpress2**  
  - Type: WordPress node  
  - Config: Updates post status to "publish"  

- **Code14**  
  - Type: Code  
  - Config: Logs post URL and status, marks step 9 complete  

- **MongoDB16**  
  - Type: MongoDB Update  
  - Config: Updates blog document with URL, status, step completion  

---

#### 1.10 Logging & Workflow Management

**Overview:**  
Handles MongoDB queries and updates throughout steps to track progress, avoid duplicate processing, and store intermediate data such as titles, outlines, articles, SEO data, and publishing status.

**Nodes Involved:**  
- MongoDB nodes spread throughout all blocks (MongoDB1, MongoDB2, MongoDB3, MongoDB4, MongoDB5, MongoDB6, MongoDB7, MongoDB8, MongoDB15, MongoDB16)

**Node Details:**  
- MongoDB is used as a persistent store for blog documents with fields indicating processing stage (`completed_step`), content data, metadata, and publication details.  
- Query nodes fetch documents at specific step completions to continue processing.  
- Update nodes write back processed data and flags to enable incremental workflow progression.

---

### 3. Summary Table

| Node Name               | Node Type                            | Functional Role                              | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                                                           |
|-------------------------|------------------------------------|----------------------------------------------|----------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------|
| Get Articles Daily       | Schedule Trigger                   | Daily trigger to start RSS feed ingestion   | None                             | Set Tech News RSS Feeds           |                                                                                                     |
| Set Tech News RSS Feeds  | Set                               | Defines array of tech RSS feed URLs          | Get Articles Daily               | Split Out                       |                                                                                                     |
| Split Out               | Split Out                         | Splits RSS feed array to process individually | Set Tech News RSS Feeds          | Read RSS News Feeds              |                                                                                                     |
| Read RSS News Feeds      | RSS Feed Read                    | Reads articles from RSS feed URL              | Split Out                       | Filter                         |                                                                                                     |
| Filter                  | Filter                            | Filters articles published in last 24 hours  | Read RSS News Feeds              | Set and Normalize Fields         |                                                                                                     |
| Set and Normalize Fields | Set                               | Normalizes article fields                      | Filter                         | Text Classifier                 |                                                                                                     |
| Text Classifier          | Langchain Text Classifier          | Classifies articles for AI relevance          | Set and Normalize Fields         | Aggregate                      |                                                                                                     |
| Aggregate               | Aggregate                        | Aggregates classified articles into one array | Text Classifier                 | AI Agent                      |                                                                                                     |
| AI Agent                | Langchain Agent                   | Generates AI blog title ideas                  | Aggregate                      | Structured Output Parser          | Sticky Note: Step 2: Relevance scoring with suggestions for improvement (serp api, jina ai, etc.)   |
| Structured Output Parser | Langchain Structured Output Parser | Parses AI-generated blog title ideas          | AI Agent                      | Split Out2                    |                                                                                                     |
| Split Out2              | Split Out                         | Splits blog titles for ranking                 | Structured Output Parser         | Code                          |                                                                                                     |
| Code                    | Code                             | Prepares titles for MongoDB insertion          | Split Out2                    | MongoDB                      |                                                                                                     |
| MongoDB                 | MongoDB Insert                    | Inserts blog document with title ideas         | Code                          | None                         |                                                                                                     |
| MongoDB1                | MongoDB Find                     | Queries blog docs needing article research    | None                          | Split Out1                   |                                                                                                     |
| Split Out1              | Split Out                         | Splits article links for full content fetch   | MongoDB1                      | Loop Over Items              |                                                                                                     |
| Loop Over Items         | Split In Batches                 | Processes articles one by one                    | Split Out1                    | Aggregate1 / Code10            |                                                                                                     |
| Code10                  | Code                             | Prepares articles for aggregation               | Loop Over Items               | HTTP Request                 |                                                                                                     |
| HTTP Request            | HTTP Request                    | Fetches full article HTML                        | Code10                       | HTML                         |                                                                                                     |
| HTML                    | HTML Extract                    | Extracts main body content from HTML            | HTTP Request                 | Transform news to MD           |                                                                                                     |
| Transform news to MD    | Code                             | Cleans HTML to Markdown                          | HTML                         | AI Agent1                    |                                                                                                     |
| OpenAI Chat Model2      | Langchain LM Chat OpenAI          | Supports AI Agent1 for summarization            | Transform news to MD           | AI Agent1                    |                                                                                                     |
| AI Agent1               | Langchain Agent                   | Summarizes articles, extracts key info          | OpenAI Chat Model2           | Structured Output Parser1       |                                                                                                     |
| Structured Output Parser1| Langchain Structured Output Parser | Parses article summaries                          | AI Agent1                    | Aggregate1                   |                                                                                                     |
| Aggregate1              | Aggregate                       | Aggregates article summaries                      | Loop Over Items              | Code4                        |                                                                                                     |
| Code4                   | Code                             | Sanitizes articles and updates step              | Aggregate1                   | MongoDB2                     |                                                                                                     |
| MongoDB2                | MongoDB Update                   | Updates blog document with article summaries    | Code4                        | None                         |                                                                                                     |
| MongoDB3                | MongoDB Find                    | Queries blog docs ready for title scoring        | None                         | AI Agent2                   |                                                                                                     |
| AI Agent2               | Langchain Agent                  | Generates 5 blog titles with SEO and clickbait  | MongoDB3                    | Structured Output Parser2      |                                                                                                     |
| Structured Output Parser2| Langchain Structured Output Parser | Parses generated blog titles                      | AI Agent2                   | AI Agent3                   |                                                                                                     |
| AI Agent3               | Langchain Agent                  | Rates blog titles on uniqueness, SEO, clickbait | Structured Output Parser2    | Structured Output Parser3      |                                                                                                     |
| Structured Output Parser3| Langchain Structured Output Parser | Parses rating scores                               | AI Agent3                   | Code7                       |                                                                                                     |
| Code7                   | Code                             | Ranks titles, selects best, updates step          | Structured Output Parser3    | MongoDB7                    |                                                                                                     |
| MongoDB7                | MongoDB Update                  | Updates blog doc with ranked titles and status   | Code7                        | None                         |                                                                                                     |
| AI Agent5               | Langchain Agent                  | Generates SEO-optimized blog outline in Markdown | MongoDB7                    | Structured Output Parser6      | Sticky Note: Step 4,5,6,7: Write the article and prepare SEO (with guide link)                    |
| Structured Output Parser6| Langchain Structured Output Parser | Parses blog outline                                | AI Agent5                   | MongoDB4                    |                                                                                                     |
| MongoDB4                | MongoDB Update                  | Stores blog outline and marks step 5 completed   | Structured Output Parser6    | None                         |                                                                                                     |
| AI Agent6               | Langchain Agent                  | Writes full blog content with keywords and tone  | MongoDB4                    | Structured Output Parser4      |                                                                                                     |
| Structured Output Parser4| Langchain Structured Output Parser | Parses full blog article content                    | AI Agent6                   | Code5                       |                                                                                                     |
| Code5                   | Code                             | Cleans blog article Markdown, updates step         | Structured Output Parser4    | MongoDB5                    |                                                                                                     |
| MongoDB5                | MongoDB Update                  | Stores full blog article                            | Code5                       | None                         |                                                                                                     |
| Code8                   | Code                             | Converts Markdown blog content to HTML             | MongoDB5                    | Wordpress1                  |                                                                                                     |
| AI Agent7               | Langchain Agent                  | Generates SEO metadata and SEO report              | MongoDB5                    | Structured Output Parser4      |                                                                                                     |
| MongoDB6                | MongoDB Update                  | Stores SEO metadata and marks step 7 completed    | AI Agent7                   | None                         |                                                                                                     |
| MongoDB8                | MongoDB Find                    | Queries blogs for image generation and publishing  | None                         | Code9                       |                                                                                                     |
| Code9                   | Code                             | Creates enriched cinematic prompt for image gen    | MongoDB8                    | HTTP Request1               |                                                                                                     |
| HTTP Request1           | HTTP Request                    | Sends image generation request to Leonardo AI      | Code9                       | Wait                        |                                                                                                     |
| Wait                    | Wait                            | Waits 1 minute for image generation to complete    | HTTP Request1               | HTTP Request2               |                                                                                                     |
| HTTP Request2           | HTTP Request                    | Polls Leonardo AI for image generation status       | Wait                        | HTTP Request3               |                                                                                                     |
| HTTP Request3           | HTTP Request                    | Downloads generated image                             | HTTP Request2               | Upload image2               |                                                                                                     |
| Upload image2           | HTTP Request                    | Uploads image to WordPress media library              | HTTP Request3               | Code12                      |                                                                                                     |
| Code12                  | Code                             | Updates blog doc with image URL and step 8           | Upload image2               | MongoDB15                   |                                                                                                     |
| MongoDB15               | MongoDB Update                  | Stores image URL and marks step 8 completed           | Code12                      | None                         |                                                                                                     |
| Wordpress1              | WordPress node                 | Creates blog post draft with content and metadata    | Code8                       | Set Image                   |                                                                                                     |
| Set Image               | HTTP Request                    | Sets featured image for WordPress post                | Wordpress1                  | Set metatag1                |                                                                                                     |
| Set metatag1            | HTTP Request                    | Updates Yoast SEO metadata (title, description)       | Set Image                   | Wordpress2                  |                                                                                                     |
| Wordpress2              | WordPress node                 | Publishes the WordPress post                          | Set metatag1                | Code14                      |                                                                                                     |
| Code14                  | Code                             | Logs blog post URL and status, marks step 9           | Wordpress2                  | MongoDB16                   |                                                                                                     |
| MongoDB16               | MongoDB Update                  | Updates blog doc with published status and URL        | Code14                      | None                         |                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Setup Schedule Trigger for Daily Run**  
- Create a **Schedule Trigger** node named "Get Articles Daily"  
- Configure it to trigger daily at 12:00 PM  

**Step 2: Define RSS Feeds**  
- Add a **Set** node "Set Tech News RSS Feeds"  
- Assign an array field `rss` with technology news RSS URLs  

**Step 3: Split RSS Feeds**  
- Add a **Split Out** node "Split Out"  
- Configure to split on field `rss`  

**Step 4: Read RSS Feeds**  
- Add an **RSS Feed Read** node "Read RSS News Feeds"  
- Set URL to expression `{{$json.rss}}`  

**Step 5: Filter Recent Articles**  
- Add **Filter** node "Filter"  
- Condition: `isoDate` after `now.minus(1, "days")`  

**Step 6: Normalize Article Fields**  
- Add **Set** node "Set and Normalize Fields"  
- Map fields: title, content snippet or encoded snippet, date, link, categories  

**Step 7: Classify Articles for AI Relevance**  
- Add **Langchain Text Classifier** node "Text Classifier"  
- Input text: title, content, categories  
- Categories: "AI" with fallback "discard"  

**Step 8: Aggregate Classified Articles**  
- Add **Aggregate** node "Aggregate" set to aggregate all item data  

**Step 9: Generate Blog Title Ideas**  
- Add **Langchain Agent** node "AI Agent"  
- Input prompt with JSON stringified aggregated articles  
- Configure OpenAI GPT-4o-mini credentials  

**Step 10: Parse AI Output**  
- Add **Structured Output Parser** node "Structured Output Parser"  
- Provide JSON schema example for titles and keywords  

**Step 11: Split Title Ideas**  
- Add **Split Out** node "Split Out2" on `output` field  

**Step 12: Prepare MongoDB Document**  
- Add **Code** node with JS to mark `completed_step: 2` and structure MongoDB insert  

**Step 13: Insert Titles to MongoDB**  
- Add **MongoDB** node  
- Insert operation into `blog` collection  
- Use configured MongoDB credentials  

**Step 14: Query MongoDB for Step 2 Docs**  
- Add **MongoDB** node "MongoDB1"  
- Query: `{"completed_step": 2}` limit 1  

**Step 15: Split Article Links**  
- Add **Split Out** node "Split Out1" on `link_articles` field  

**Step 16: Loop Over Article Links**  
- Add **Split In Batches** node "Loop Over Items"  

**Step 17: Fetch Full Article HTML**  
- Add **HTTP Request** node with URL from item JSON link_articles  

**Step 18: Extract Article Body HTML**  
- Add **HTML Extract** node "HTML"  
- CSS selector: `body` skip images  

**Step 19: Convert to Markdown**  
- Add **Code** node "Transform news to MD"  
- Strip HTML tags and sanitize content  

**Step 20: Summarize Article**  
- Add **Langchain Agent** node "AI Agent1"  
- Prompt to extract 3 factual sentences, 2-paragraph summary, key ideas, names, quotes  
- Use GPT-4o-mini API credentials  

**Step 21: Parse Summary Output**  
- Add **Structured Output Parser1** node  

**Step 22: Aggregate Summaries**  
- Add **Aggregate** node "Aggregate1" to combine article summaries  

**Step 23: Sanitize and Update MongoDB**  
- Add **Code** node "Code4" to escape quotes and mark `completed_step: 3`  
- Add **MongoDB** update node "MongoDB2"  

**Step 24: Query MongoDB for Step 3 Docs**  
- Add **MongoDB** find node "MongoDB3"  

**Step 25: Generate Blog Titles with Emotional Angles**  
- Add **Langchain Agent** node "AI Agent2"  
- Prompt for 5 compelling blog titles with emotional, curiosity angles  

**Step 26: Parse Titles Output**  
- Add **Structured Output Parser2**  

**Step 27: Rate Blog Titles**  
- Add **Langchain Agent** node "AI Agent3"  
- Rate titles (uniqueness, SEO, clickbait)  

**Step 28: Parse Ratings**  
- Add **Structured Output Parser3**  

**Step 29: Rank Titles and Select Best**  
- Add **Code** node "Code7" to calculate average score, sort, and select top title  
- Add **MongoDB** update node "MongoDB7" to store results and mark `completed_step:4`  

**Step 30: Generate SEO-Optimized Blog Outline**  
- Add **Langchain Agent** node "AI Agent5"  
- Prompt to generate outline with H1 and H2, keywords, featured snippet optimization  

**Step 31: Parse Outline**  
- Add **Structured Output Parser6**  

**Step 32: Store Outline in MongoDB**  
- Add **Code** node "Code3" to clean markdown fences and escape quotes  
- Add **MongoDB** update node "MongoDB4" mark `completed_step: 5`  

**Step 33: Generate Full Blog Content**  
- Add **Langchain Agent** node "AI Agent6"  
- Prompt uses title, keywords, tone, outline, related articles, citations  
- Output full blog post in Markdown  

**Step 34: Parse Blog Content**  
- Add **Structured Output Parser4**  

**Step 35: Clean and Store Blog Article**  
- Add **Code** node "Code5" to clean fences and escape quotes  
- Add **MongoDB** update node "MongoDB5" mark `completed_step:6`  

**Step 36: Convert Markdown to HTML**  
- Add **Code** node "Code8" to convert markdown to HTML for CMS  

**Step 37: Generate SEO Metadata and Report**  
- Add **Langchain Agent** node "AI Agent7"  
- Prompt generates meta title, description, alt text, URL slug, SEO report  

**Step 38: Store SEO Data**  
- Add **MongoDB** update node "MongoDB6" mark `completed_step:7`  

**Step 39: Query MongoDB for Image Generation**  
- Add **MongoDB** find node "MongoDB8" for step 7 docs  

**Step 40: Create Enriched Image Prompt**  
- Add **Code** node "Code9" to enrich alt text into cinematic image prompt  

**Step 41: Request Image Generation (Leonardo AI)**  
- Add **HTTP Request** node "HTTP Request1" to trigger image gen with prompt, model ID, style UUID  

**Step 42: Wait for Image Generation**  
- Add **Wait** node for 1 minute  

**Step 43: Poll Image Generation Status**  
- Add **HTTP Request** node "HTTP Request2" to check generation status  

**Step 44: Download Generated Image**  
- Add **HTTP Request** node "HTTP Request3" to download image from URL  

**Step 45: Upload Image to WordPress**  
- Add **HTTP Request** node "Upload image2"  
- POST binary data with appropriate headers for WordPress media upload  

**Step 46: Update MongoDB with Image URL**  
- Add **Code** node "Code12" to update blog doc with image URL and step 8 completed  
- Add **MongoDB** update node "MongoDB15"  

**Step 47: Create WordPress Post Draft**  
- Add **WordPress** node "Wordpress1"  
- Create draft post with title, slug, author, category, content (HTML)  

**Step 48: Set Featured Image**  
- Add **HTTP Request** node "Set Image"  
- POST to WordPress API to assign featured image by media ID  

**Step 49: Update Yoast SEO Metadata**  
- Add **HTTP Request** node "Set metatag1"  
- PUT meta fields `_yoast_wpseo_title` and `_yoast_wpseo_metadesc`  

**Step 50: Publish WordPress Post**  
- Add **WordPress** node "Wordpress2"  
- Update post status to "publish"  

**Step 51: Log Published URL and Status**  
- Add **Code** node "Code14" to save post link, status in MongoDB  
- Add **MongoDB** update node "MongoDB16" mark `completed_step:9`  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Step 1: Save news in a vector store (runs daily)                                                                                                                                                                               | Sticky Note1 content                                                                              |
| Step 2: Relevance scoring - Consider improving with serp api, jina ai, perplexity ai, open ai web search                                                                                                                       | https://serpapi.com/dashboard, https://jina.ai/api-dashboard/reader, https://www.perplexity.ai/, https://www.youtube.com/watch?v=eeuLRpvQ3DY                   |
| Detailed workflow steps and explanation on content farming, filtering, summarization, SEO outline, drafting, enrichment, image generation, publishing, and logging                                                            | Sticky Note2 content                                                                              |
| Step 3: Research the news articles - placeholder to edit and add guidance                                                                                                                                                      | https://docs.n8n.io/workflows/sticky-notes/                                                     |
| Step 4,5,6,7: Write the article and prepare SEO - placeholder for editing and guidance                                                                                                                                         | https://docs.n8n.io/workflows/sticky-notes/                                                     |
| Step 8,9: Generate image and create blog article                                                                                                                                                                               | Sticky Note5 content                                                                              |

---

**Disclaimer:** The provided text and workflow details derive exclusively from an automated n8n workflow. All processing complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.