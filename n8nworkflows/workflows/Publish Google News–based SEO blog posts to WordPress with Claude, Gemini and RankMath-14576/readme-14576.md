Publish Google News–based SEO blog posts to WordPress with Claude, Gemini and RankMath

https://n8nworkflows.xyz/workflows/publish-google-news-based-seo-blog-posts-to-wordpress-with-claude--gemini-and-rankmath-14576


# Publish Google News–based SEO blog posts to WordPress with Claude, Gemini and RankMath

# Workflow Documentation: Publish Google News–based SEO blog posts to WordPress

## 1. Workflow Overview
This workflow automates the process of content curation and creation by monitoring Google News for SEO and digital marketing trends, rewriting the content using AI to ensure originality and SEO optimization, and publishing the final result to a WordPress site.

The logic is divided into the following functional blocks:
- **1.1 Input Reception & News Fetching:** Triggers the process and retrieves current news and historical logs.
- **1.2 Filtering & Deduplication:** Ensures only new, relevant, and recent articles are processed.
- **1.3 Article Selection & Scraping:** Uses AI to pick the most "worthy" article and extracts its raw text.
- **1.4 AI Content Generation:** Transforms the scraped data into a unique, high-quality SEO article and an image prompt.
- **1.5 Media Production & Upload:** Generates a featured image and uploads it to the WordPress Media Library.
- **1.6 Publishing & Logging:** Formats the content, creates a WordPress post (including RankMath SEO metadata), and logs the activity in Google Sheets.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & News Fetching
**Overview:** Initiates the workflow on a schedule and gathers raw data from Google News and a history log.
- **Nodes Involved:** `⏰ Run Every 12 Hours`, `Google_news search`, `Get Previous Published News`.
- **Node Details:**
    - **⏰ Run Every 12 Hours** (Schedule Trigger): Executes the workflow every 12 hours.
    - **Google_news search** (SerpAPI): Performs a Google News search with the query `"seo"`.
    - **Get Previous Published News** (Google Sheets): Reads a specific sheet (`Google News to Wordpress`) to retrieve a list of already processed news links.

### 2.2 Filtering & Deduplication
**Overview:** Cleans the incoming news list to prevent duplicate posts and focuses on the most recent items.
- **Nodes Involved:** `Split into Items`, `Remove Already Used News`, `Sort By Time`, `Limit`.
- **Node Details:**
    - **Split into Items** (Split Out): Breaks the `news_results` array from SerpAPI into individual items.
    - **Remove Already Used News** (Merge): Uses "Keep Non-Matches" mode to compare `news_link` from the search with `link` from the Google Sheet, removing any that already exist.
    - **Sort By Time** (Sort): Orders the remaining articles by `iso_date` in descending order.
    - **Limit** (Limit): Keeps only the top 10 most recent articles to avoid overloading the AI.

### 2.3 Article Selection & Scraping
**Overview:** Uses an LLM to select one high-value article from the filtered list and retrieves the full text.
- **Nodes Involved:** `Merge into One Output`, `Slect one worthy article`, `Scrape Source Page`, `Extract Article Content`.
- **Node Details:**
    - **Merge into One Output** (Set): Maps the list of 10 titles and links into a single string for AI analysis.
    - **Slect one worthy article** (AI Agent): Uses **Claude 3.5 Sonnet** (via OpenRouter) to pick one article based on specific rules (SEO focus, actionable insights). It uses a **Structured Output Parser** to return exactly `title` and `link`.
    - **Scrape Source Page** (HTTP Request): Performs a GET request to the URL selected by the AI.
    - **Extract Article Content** (HTML): Extracts text from `h1, h2, h3, p` tags to gather the main body of the article.

### 2.4 AI Content Generation
**Overview:** Turns raw scraped data into a professional, original blog post.
- **Nodes Involved:** `Aggregate`, `AI Content Generator`.
- **Node Details:**
    - **Aggregate** (Aggregate): Combines the extracted HTML content fragments into a single block of text.
    - **AI Content Generator** (AI Agent): Uses **Claude 3.5 Sonnet** to generate a completely original article.
        - **Key Configuration:** The system prompt explicitly forbids copying or paraphrasing; it demands a new structure, a professional "Designer Style" image prompt, and SEO-optimized meta tags.
        - **Output Parser:** Uses a **Structured Output Parser** to extract `title`, `meta title`, `meta description`, `content`, and `prompt`.

### 2.5 Media Production & Upload
**Overview:** Creates a visual asset and integrates it into the WordPress ecosystem.
- **Nodes Involved:** `Generate Featured Image`, `Uploading Image to WP`.
- **Node Details:**
    - **Generate Featured Image** (Google Gemini): Uses the `gemini-3-pro-image-preview` model to create a 16:9 image based on the AI-generated prompt.
    - **Uploading Image to WP** (HTTP Request): Sends the binary image data to the WordPress REST API (`/wp-json/wp/v2/media`).

### 2.6 Publishing & Logging
**Overview:** Finalizes the post and maintains a record of the automation.
- **Nodes Involved:** `Markdown to Html`, `Data Arrange`, `Create Post in WP`, `Add Image, Meta Title, Description to Post`, `Add Details In Sheet`.
- **Node Details:**
    - **Markdown to Html** (Markdown): Converts the AI's Markdown output into HTML for WordPress compatibility.
    - **Data Arrange** (Set): Consolidates the image ID, final title, and meta-data into a single object.
    - **Create Post in WP** (WordPress): Creates a post in `draft` status within category 24.
    - **Add Image, Meta Title, Description to Post** (HTTP Request): A PUT request to update the post with the `featured_media` ID and **RankMath** custom meta fields (`rank_math_title`, `rank_math_description`).
    - **Add Details In Sheet** (Google Sheets): Appends the original news link and the new WordPress post link to the log.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ⏰ Run Every 12 Hours | Schedule Trigger | Workflow Initiation | - | Google_news search, Get Previous Published News | Trigger and fetch news |
| Google_news search | SerpAPI | News Retrieval | ⏰ Run Every 12 Hours | Split into Items | Trigger and fetch news |
| Get Previous Published News | Google Sheets | History Retrieval | ⏰ Run Every 12 Hours | Remove Already Used News | Trigger and fetch news |
| Split into Items | Split Out | Data Normalization | Google_news search | Remove Already Used News | Process and filter news |
| Remove Already Used News | Merge | Deduplication | Split into Items, Get Previous Published News | Sort By Time | Process and filter news |
| Sort By Time | Sort | Recency Ordering | Remove Already Used News | Limit | Process and filter news |
| Limit | Limit | Volume Control | Sort By Time | Merge into One Output | Process and filter news |
| Merge into One Output | Set | Data Aggregation | Limit | Slect one worthy article | Select article and prepare content |
| Slect one worthy article | AI Agent | Strategic Selection | Merge into One Output | Scrape Source Page | Select article and prepare content |
| Scrape Source Page | HTTP Request | Content Acquisition | Slect one worthy article | Extract Article Content | Select article and prepare content |
| Extract Article Content | HTML | Text Extraction | Scrape Source Page | Aggregate | Select article and prepare content |
| Aggregate | Aggregate | Text Consolidation | Extract Article Content | AI Content Generator | Aggregate and generate new content |
| AI Content Generator | AI Agent | Content Creation | Aggregate | Generate Featured Image | Aggregate and generate new content |
| Generate Featured Image | Google Gemini | Image Generation | AI Content Generator | Uploading Image to WP | Generate and upload image |
| Uploading Image to WP | HTTP Request | Media Upload | Generate Featured Image | Markdown to Html | Generate and upload image |
| Markdown to Html | Markdown | Format Conversion | Uploading Image to WP | Data Arrange | Format and post content |
| Data Arrange | Set | Object Mapping | Markdown to Html | Create Post in WP | Format and post content |
| Create Post in WP | WordPress | Draft Creation | Data Arrange | Add Image, Meta Title, Description to Post | Format and post content |
| Add Image, Meta Title, Description to Post | HTTP Request | SEO/Media Finalization | Create Post in WP | Add Details In Sheet | Finalize post and update logs |
| Add Details In Sheet | Google Sheets | Activity Logging | Add Image, Meta Title, Description to Post | - | Finalize post and update logs |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Triggers and Data Retrieval
1. Create a **Schedule Trigger** set to "Every 12 Hours".
2. Add a **SerpAPI** node: Operation "Google News", Query "seo".
3. Add a **Google Sheets** node: Operation "Get Many", select your log spreadsheet.

### Step 2: Filtering Logic
1. Add a **Split Out** node to target the `news_results` field from SerpAPI.
2. Add a **Merge** node: Mode "Combine", Join Mode "Keep Non-Matches". Map `news_link` (from search) to `link` (from Sheets).
3. Add a **Sort** node: Sort by `iso_date` (Descending).
4. Add a **Limit** node: Max items = 10.

### Step 3: AI Selection and Scraping
1. Add a **Set** node to combine the 10 items into a single string list.
2. Add an **AI Agent** node:
    - **Model:** OpenRouter (Claude 3.5 Sonnet).
    - **Output Parser:** Structured Parser (`title`, `link`).
    - **System Prompt:** Instructions to pick one SEO-worthy article.
3. Add an **HTTP Request** node: Method GET, URL = `{{ $json.output.link }}`.
4. Add an **HTML** node: Operation "Extract HTML Content", CSS Selectors `h1, h2, h3, p`.

### Step 4: Content Generation
1. Add an **Aggregate** node to merge HTML content into one string.
2. Add an **AI Agent** node:
    - **Model:** OpenRouter (Claude 3.5 Sonnet).
    - **Output Parser:** Structured Parser (`title`, `meta title`, `meta description`, `content`, `prompt`).
    - **System Prompt:** Comprehensive SEO strategy guidelines (provided in the workflow JSON).

### Step 5: Media and WordPress Integration
1. Add a **Google Gemini** node: Resource "Image", Model `gemini-3-pro-image-preview`. Prompt = `{{ $json.output.prompt }} image should be 16:9 ratio`.
2. Add an **HTTP Request** node: POST to `/wp-json/wp/v2/media`, Binary data, Header `Content-Disposition: attachment; filename=image.jpg`.
3. Add a **Markdown** node: Mode "Markdown to HTML", targeting the AI-generated content.
4. Add a **Set** node to arrange `image_id`, `title`, and `meta` tags.
5. Add a **WordPress** node: Operation "Create Post", Status "Draft", Category 24.
6. Add an **HTTP Request** node: PUT to `/wp-json/wp/v2/posts/{{ $json.id }}`. Body should contain `featured_media` and `meta` (RankMath fields).

### Step 6: Final Log
1. Add a **Google Sheets** node: Operation "Append", map `news_link` and `wp_post_link`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Requires SerpAPI account for Google News | [SerpAPI](https://serpapi.com/) |
| Requires OpenRouter account for Claude 3.5 Sonnet | [OpenRouter](https://openrouter.ai/) |
| Requires Google Gemini API for image generation | [Google AI Studio](https://aistudio.google.com/) |
| Target website must have RankMath SEO installed | WordPress Plugin |
| WordPress API permissions must be granted for Media and Posts | WP-JSON API |