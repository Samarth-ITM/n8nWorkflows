News-to-Blog Automation with GPT, Leonardo AI & WordPress Publishing

https://n8nworkflows.xyz/workflows/news-to-blog-automation-with-gpt--leonardo-ai---wordpress-publishing-7790


# News-to-Blog Automation with GPT, Leonardo AI & WordPress Publishing

---
### 1. Workflow Overview

This workflow automates the creation and publication of SEO-optimized blog posts based on curated news about startups, venture capital, AI startups, and technology. It integrates multiple data sources and AI services to research, generate, enhance, and publish articles with editorial images on a WordPress site.

The workflow is logically structured into the following blocks:

- **1.1 News Aggregation & Deduplication:** Collects news headlines from Google News RSS and GDELT Docs API, merges, deduplicates, sorts, and classifies them by topic.
- **1.2 Article Generation & Enhancement:** Uses OpenAI GPT models to generate a detailed, SEO-friendly article draft based on the top headlines and a defined writing style. It applies tone and format revision and enforces a minimum word count with an expansion step if needed.
- **1.3 Editorial Image Creation:** Crafts a prompt for Leonardo AI to generate a blog header image reflecting the article content and uploads it to WordPress with appropriate alt text.
- **1.4 WordPress Publishing:** Publishes the finalized article along with the uploaded image as a featured media on a WordPress site.

The workflow runs daily at noon via a schedule trigger, ensuring fresh content is created automatically.

---

### 2. Block-by-Block Analysis

#### 2.1 News Aggregation & Deduplication

**Overview:**  
Aggregates news items from two external sources, merges and deduplicates them, selects top headlines, and classifies them by topic and angle.

**Nodes Involved:**  
- Schedule Trigger  
- Google News RSS  
- GDELT Docs API  
- Merge News  
- Within dedupe (Code)  
- Top Headlines (Code)  
- Classify Headlines (Code)  
- Writing Style (Code)  
- Sticky Notes: "Pulls from the Google news RSS to extract headlines", "Removes duplicates, pulls the top headlines for the day and classifies."

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts workflow daily at 12:00 PM.  
  - Config: Trigger at hour 12, every day.  
  - Inputs: None  
  - Outputs: Google News RSS, GDELT Docs API  
  - Edge cases: Timezone issues could affect trigger timing.

- **Google News RSS**  
  - Type: RSS Feed Read  
  - Role: Fetches news headlines matching keywords ("startups", "venture capital", "AI startup", "technology") from Google News RSS feed.  
  - Config: RSS URL with search query and Australian English locale.  
  - Inputs: Schedule Trigger  
  - Outputs: Merge News  
  - Edge cases: RSS feed downtime, malformed XML.

- **GDELT Docs API**  
  - Type: HTTP Request  
  - Role: Fetches news articles list from GDELT API with matching keywords, sorted by date descending.  
  - Config: HTTP GET with query parameters for max 50 records, JSON format.  
  - Inputs: Schedule Trigger  
  - Outputs: Merge News  
  - Edge cases: API rate limits, network errors.

- **Merge News**  
  - Type: Merge  
  - Role: Combines input items from Google News RSS and GDELT Docs API into a single stream.  
  - Inputs: Google News RSS, GDELT Docs API  
  - Outputs: Within dedupe  
  - Edge cases: Unequal input sizes, empty inputs.

- **Within dedupe**  
  - Type: Code  
  - Role: Deduplicates news items by canonicalized URLs (stripping UTM parameters and URL fragments).  
  - Logic: Uses a Set to track seen normalized URLs; outputs unique items.  
  - Inputs: Merge News  
  - Outputs: Top Headlines  
  - Edge cases: Missing or malformed URLs; items without URLs are skipped.

- **Top Headlines**  
  - Type: Code  
  - Role: Sorts merged and deduplicated items by most recent date and selects top 3 headlines. Formats headlines and news bullets for downstream use.  
  - Inputs: Within dedupe  
  - Outputs: Classify Headlines  
  - Edge cases: Items missing date fields default to timestamp 0; empty input throws no error but results in empty headlines.

- **Classify Headlines**  
  - Type: Code  
  - Role: Classifies the overall topic and angle of the news batch based on keyword matching in headlines and links. Assigns trusted domain list per topic.  
  - Inputs: Top Headlines  
  - Outputs: Writing Style  
  - Edge cases: If no matches, defaults to 'startups-tech' topic and generic trusted domains.

- **Writing Style**  
  - Type: Code  
  - Role: Provides a fixed writing style guide, do’s and don’ts checklist to shape article tone and structure.  
  - Inputs: Classify Headlines  
  - Outputs: Build GPT body  
  - Edge cases: Static content, no failure expected.

---

#### 2.2 Article Generation & Enhancement

**Overview:**  
Generates a detailed article draft using GPT-4 based on news headlines and writing style, polishes the draft, enforces a minimum word count, and expands the draft if necessary.

**Nodes Involved:**  
- Build GPT body (Code)  
- Research Topic – GPT (OpenAI HTTP Request)  
- Build Tone Request (Code)  
- Tone & Format Revision – GPT (OpenAI HTTP Request)  
- Word Count Guard (Code)  
- If1 (If)  
- Expand Draft (OpenAI LangChain Node)  
- Get Title, Content, and Image FileName (Code)  
- Sticky Notes: "Builds your GPT Body ready for research", "Researches, builds the narrative tone and formats the article. Then it passes through a word count guard to ensure articles are over 1600 words, if not it will pass through the expand draft."

**Node Details:**

- **Build GPT body**  
  - Type: Code  
  - Role: Constructs the GPT request payload to generate a news-driven article with citations, minimum 1600 words, using the top headlines and style guide.  
  - Inputs: Writing Style  
  - Outputs: Research Topic – GPT  
  - Edge cases: Handles missing headlines by throwing error.

- **Research Topic – GPT**  
  - Type: HTTP Request (OpenAI chat completion)  
  - Role: Sends GPT request to generate initial article draft JSON object (title, content).  
  - Config: Uses OpenAI API with predefined credentials, sends constructed JSON payload.  
  - Inputs: Build GPT body  
  - Outputs: Build Tone Request  
  - Edge cases: API errors, authentication failures, JSON parsing errors.

- **Build Tone Request**  
  - Type: Code  
  - Role: Prepares a GPT request to polish and improve the article's clarity and flow without changing facts or structure, preserving HTML formatting.  
  - Inputs: Research Topic – GPT  
  - Outputs: Tone & Format Revision – GPT  
  - Edge cases: Handles missing style guide gracefully.

- **Tone & Format Revision – GPT**  
  - Type: HTTP Request (OpenAI chat completion)  
  - Role: Calls GPT to refine the article tone and formatting per instructions.  
  - Inputs: Build Tone Request  
  - Outputs: Word Count Guard  
  - Edge cases: API errors, output JSON parsing errors.

- **Word Count Guard**  
  - Type: Code  
  - Role: Measures the word count of the article content (stripping HTML). If below 1600 words, marks need_expand true.  
  - Inputs: Tone & Format Revision – GPT  
  - Outputs: If1  
  - Edge cases: Empty or malformed content.

- **If1**  
  - Type: If  
  - Role: Branches workflow based on whether article expansion is needed.  
  - Condition: Checks if need_expand === true.  
  - Inputs: Word Count Guard  
  - Outputs:  
    - True: Expand Draft  
    - False: Get Title, Content, and Image FileName  

- **Expand Draft**  
  - Type: OpenAI LangChain Node  
  - Role: Expands the article to exceed 1800 words by deepening key sections while preserving voice and links.  
  - Inputs: If1 (true branch)  
  - Outputs: Get Title, Content, and Image FileName  
  - Edge cases: API failures, token limits.

- **Get Title, Content, and Image FileName**  
  - Type: Code  
  - Role: Extracts and cleans title, content, and constructs an image filename slug for subsequent image generation and uploading. Also appends a "Sources" section with extracted URLs.  
  - Inputs: If1 (false branch) or Expand Draft  
  - Outputs: Leonardo Prompt Creator  
  - Edge cases: JSON parsing errors, malformed content.

---

#### 2.3 Editorial Image Creation

**Overview:**  
Generates a descriptive prompt for Leonardo AI to create an editorial blog post image, triggers image generation, waits for completion, retrieves image URL, downloads image binary, uploads it to WordPress, and adds alt text.

**Nodes Involved:**  
- Leonardo Prompt Creator (OpenAI LangChain Node)  
- Leonardo: Create Post Image (HTTP Request)  
- Code (Keep Generation Id)  
- Wait  
- Get Leonardo Image Status (HTTP Request)  
- Get Leonardo Image (HTTP Request)  
- Upload Image to Wordpress (HTTP Request)  
- Add ALT to Image (HTTP Request)  
- Sticky Notes: "Extracts information from the article and writes a prompt for leonardo to create a header picture.", "Creates the image and uploads to wordpress, change to your site/ blog location"

**Node Details:**

- **Leonardo Prompt Creator**  
  - Type: OpenAI LangChain Node  
  - Role: Generates a concise English prompt describing an editorial-style image for the article.  
  - Inputs: Get Title, Content, and Image FileName  
  - Outputs: Leonardo: Create Post Image  
  - Edge cases: API errors, prompt generation failures.

- **Leonardo: Create Post Image**  
  - Type: HTTP Request  
  - Role: Sends image generation request to Leonardo AI with parameters including model ID, size, prompt, and settings.  
  - Inputs: Leonardo Prompt Creator  
  - Outputs: Code (Keep Generation Id)  
  - Edge cases: API rate limits, generation failures.

- **Code (Keep Generation Id)**  
  - Type: Code  
  - Role: Extracts and retains the generation ID from the image creation response for polling.  
  - Inputs: Leonardo: Create Post Image  
  - Outputs: Wait  
  - Edge cases: Missing generation ID throws error.

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow for 30 seconds to allow image generation to complete.  
  - Inputs: Code  
  - Outputs: Get Leonardo Image Status  
  - Edge cases: Fixed delay may not adapt to longer generation times.

- **Get Leonardo Image Status**  
  - Type: HTTP Request  
  - Role: Queries Leonardo API for the status of the image generation job using generation ID.  
  - Inputs: Wait  
  - Outputs: Get Leonardo Image  
  - Edge cases: Job not ready, network errors, retries enabled.

- **Get Leonardo Image**  
  - Type: HTTP Request  
  - Role: Retrieves the generated image URL from Leonardo API response.  
  - Inputs: Get Leonardo Image Status  
  - Outputs: Upload Image to Wordpress  
  - Edge cases: Missing or invalid image URL.

- **Upload Image to Wordpress**  
  - Type: HTTP Request  
  - Role: Uploads the binary image to WordPress media library using OAuth2 authentication, setting proper headers for filename and content type.  
  - Inputs: Get Leonardo Image  
  - Outputs: Add ALT to Image  
  - Edge cases: Authentication errors, upload failures.

- **Add ALT to Image**  
  - Type: HTTP Request  
  - Role: Updates WordPress media item to add alt text derived from Leonardo prompt to improve SEO and accessibility.  
  - Inputs: Upload Image to Wordpress  
  - Outputs: Create WordPress Post  
  - Edge cases: Media ID missing, permission errors.

---

#### 2.4 WordPress Publishing

**Overview:**  
Publishes the finalized blog post with title, content, category, and featured image on WordPress.

**Nodes Involved:**  
- Create WordPress Post (HTTP Request)

**Node Details:**

- **Create WordPress Post**  
  - Type: HTTP Request  
  - Role: Creates and publishes a new WordPress post using REST API, associating the post with a specific category and featured media.  
  - Config: POST to WordPress site posts endpoint, JSON body includes title, content, status "publish", category ID 916, featured_media ID from uploaded image.  
  - Inputs: Add ALT to Image  
  - Outputs: None (end node)  
  - Edge cases: Authentication errors, invalid category or media IDs, API failures.  
  - Notes: Requires OAuth2 and Basic Auth credentials configured.

---

### 3. Summary Table

| Node Name                      | Node Type                      | Functional Role                                | Input Node(s)                             | Output Node(s)                          | Sticky Note                                                                                      |
|--------------------------------|--------------------------------|-----------------------------------------------|------------------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger               | Starts workflow daily at noon                  | None                                     | Google News RSS, GDELT Docs API       |                                                                                                |
| Google News RSS                | RSS Feed Read                 | Fetches news headlines from Google News RSS   | Schedule Trigger                         | Merge News                           | Pulls from the Google news RSS to extract headlines                                          |
| GDELT Docs API                | HTTP Request                  | Fetches news articles list from GDELT API     | Schedule Trigger                         | Merge News                           |                                                                                                |
| Merge News                    | Merge                        | Combines news items from RSS and GDELT        | Google News RSS, GDELT Docs API          | Within dedupe                       | Removes duplicates, pulls the top headlines for the day and classifies.                       |
| Within dedupe                 | Code                         | Deduplicates news by canonical URL             | Merge News                              | Top Headlines                       |                                                                                                |
| Top Headlines                 | Code                         | Sorts and selects top 3 headlines              | Within dedupe                           | Classify Headlines                  |                                                                                                |
| Classify Headlines            | Code                         | Classifies topic and angle of news batch       | Top Headlines                          | Writing Style                      |                                                                                                |
| Writing Style                | Code                         | Provides writing style guide and do's/don'ts   | Classify Headlines                     | Build GPT body                    | Change the writing style to what you would prefer,                                          |
| Build GPT body              | Code                         | Builds GPT request payload for article draft   | Writing Style                         | Research Topic - GPT               | Builds your GPT Body ready for research                                                     |
| Research Topic – GPT         | HTTP Request (OpenAI)         | Generates initial article draft from GPT       | Build GPT body                        | Build Tone Request                | Researches, builds the narrative tone and formats the article. Then it passes through a word count guard to ensure articles are over 1600 words, if not it will pass through the expand draft. |
| Build Tone Request           | Code                         | Prepares GPT request to polish article          | Research Topic - GPT                  | Tone & Format Revision - GPT      |                                                                                                |
| Tone & Format Revision – GPT | HTTP Request (OpenAI)         | Refines article clarity and flow                | Build Tone Request                   | Word Count Guard                 |                                                                                                |
| Word Count Guard             | Code                         | Checks word count, flags if expansion needed   | Tone & Format Revision - GPT         | If1                            |                                                                                                |
| If1                         | If                           | Branches on need to expand article              | Word Count Guard                    | Expand Draft / Get Title, Content, and Image FileName |                                                                                                |
| Expand Draft                | OpenAI LangChain Node         | Expands article to exceed 1800 words            | If1 (true branch)                   | Get Title, Content, and Image FileName |                                                                                                |
| Get Title, Content, and Image FileName | Code                         | Extracts title, content, and image filename     | If1 (false branch), Expand Draft    | Leonardo Prompt Creator           | Extracts information from the article and writes a prompt for leonardo to create a header picture. |
| Leonardo Prompt Creator     | OpenAI LangChain Node         | Generates prompt for Leonardo AI image          | Get Title, Content, and Image FileName | Leonardo: Create Post Image       |                                                                                                |
| Leonardo: Create Post Image | HTTP Request                 | Requests image generation from Leonardo AI      | Leonardo Prompt Creator             | Code (Keep Generation Id)          | Creates the image and uploads to wordpress, change to your site/ blog location              |
| Code (Keep Generation Id)   | Code                         | Extracts generation ID from image creation response | Leonardo: Create Post Image         | Wait                           |                                                                                                |
| Wait                        | Wait                         | Waits 30 seconds for image generation           | Code (Keep Generation Id)            | Get Leonardo Image Status         |                                                                                                |
| Get Leonardo Image Status   | HTTP Request                 | Checks image generation status                   | Wait                              | Get Leonardo Image               |                                                                                                |
| Get Leonardo Image          | HTTP Request                 | Retrieves generated image URL                     | Get Leonardo Image Status           | Upload Image to Wordpress         |                                                                                                |
| Upload Image to Wordpress   | HTTP Request                 | Uploads binary image to WordPress media library | Get Leonardo Image                 | Add ALT to Image                 |                                                                                                |
| Add ALT to Image            | HTTP Request                 | Adds alt text to uploaded WordPress image       | Upload Image to Wordpress           | Create WordPress Post             |                                                                                                |
| Create WordPress Post       | HTTP Request                 | Publishes blog post with content and image      | Add ALT to Image                   | None                           |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 12:00 PM.

2. **Create News Sources Nodes**  
   - Create **Google News RSS** node:  
     - Type: RSS Feed Read  
     - URL: `https://news.google.com/rss/search?q=startups%20OR%20%22venture%20capital%22%20OR%20%22AI%20startup%22%20OR%20technology&hl=en-AU&gl=AU&ceid=AU:en`  
   - Create **GDELT Docs API** node:  
     - Type: HTTP Request (GET)  
     - URL: `https://api.gdeltproject.org/api/v2/doc/doc`  
     - Query parameters:  
       - `query`: `startups OR "venture capital" OR "AI startup" OR technology`  
       - `mode`: `artlist`  
       - `maxrecords`: `50`  
       - `format`: `json`  
       - `sort`: `DateDesc`  
     - Enable retry on fail.

3. **Create Merge Node**  
   - Type: Merge  
   - Merge inputs from Google News RSS and GDELT Docs API.

4. **Create Deduplication Code Node**  
   - Type: Code  
   - Paste JavaScript code to deduplicate items by canonical URL (strip UTM and hash).

5. **Create Top Headlines Code Node**  
   - Type: Code  
   - Sort merged items newest first, pick top 3 headlines, format headlines and news bullets.

6. **Create Classify Headlines Code Node**  
   - Type: Code  
   - Classify topic and angle based on keywords in headlines and links; assign trusted domains accordingly.

7. **Create Writing Style Code Node**  
   - Type: Code  
   - Define writing style guide, do’s and don’ts for article tone and structure.

8. **Create Build GPT Body Code Node**  
   - Type: Code  
   - Construct GPT chat completion request payload with system and user messages embedding headlines and style guide.

9. **Create Research Topic – GPT Node**  
   - Type: HTTP Request (OpenAI Chat Completion)  
   - URL: `https://api.openai.com/v1/chat/completions`  
   - Method: POST  
   - Authentication: OpenAI API credentials  
   - Body: JSON from Build GPT Body node  

10. **Create Build Tone Request Code Node**  
    - Type: Code  
    - Prepare GPT request to polish article (clarity and flow) preserving HTML and facts.

11. **Create Tone & Format Revision – GPT Node**  
    - Type: HTTP Request (OpenAI Chat Completion)  
    - Method: POST  
    - Authentication: OpenAI API credentials  
    - Body: JSON from Build Tone Request node  
    - Enable retry on fail.

12. **Create Word Count Guard Code Node**  
    - Type: Code  
    - Count words in article content, flag if below 1600 words.

13. **Create If Node**  
    - Type: If  
    - Condition: Check if `need_expand` is true.

14. **Create Expand Draft Node**  
    - Type: OpenAI LangChain Node  
    - Model: GPT-4.1  
    - Prompt: Expand article to exceed 1800 words by deepening key sections, keep voice and links.  
    - Authentication: OpenAI API credentials.

15. **Create Get Title, Content, and Image FileName Code Node**  
    - Type: Code  
    - Extract title, content, slugify title for image filename, append sources section.

16. **Create Leonardo Prompt Creator Node**  
    - Type: OpenAI LangChain Node  
    - Model: GPT-4.1-mini  
    - Prompt: Generate a single-line English editorial image prompt based on article title and content.  
    - Authentication: OpenAI API credentials.

17. **Create Leonardo: Create Post Image Node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Leonardo AI generation API endpoint  
    - Body: JSON with prompt, model ID, size, settings  
    - Authentication: Bearer Token for Leonardo AI  
    - Enable retry on fail.

18. **Create Code Node to Keep Generation Id**  
    - Type: Code  
    - Extract generation ID from previous response, throw error if missing.

19. **Create Wait Node**  
    - Type: Wait  
    - Duration: 30 seconds.

20. **Create Get Leonardo Image Status Node**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: Leonardo API generation status endpoint with generation ID  
    - Authentication: Bearer Token  
    - Enable retry on fail.

21. **Create Get Leonardo Image Node**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: Use URL from generation status response to fetch generated image  
    - Enable retry on fail.

22. **Create Upload Image to Wordpress Node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: WordPress media endpoint (your site)  
    - Content type: binary  
    - Headers: Content-Disposition with filename, Content-Type image/jpeg  
    - Authentication: OAuth2 and Basic Auth credentials for WordPress  
    - Input data field name: binary data  
    - Enable retry on fail.

23. **Create Add ALT to Image Node**  
    - Type: HTTP Request  
    - Method: PUT  
    - URL: WordPress media endpoint with media ID  
    - Body parameter: alt_text from Leonardo prompt content  
    - Authentication: OAuth2 and Basic Auth  
    - Enable retry on fail.

24. **Create Create WordPress Post Node**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: WordPress posts endpoint (your site)  
    - Body: JSON with title, content, status "publish", categories [916], featured_media ID  
    - Authentication: OAuth2 and Basic Auth  
    - Enable retry on fail.

25. **Connect Nodes**  
    - Schedule Trigger → Google News RSS and GDELT Docs API  
    - Google News RSS + GDELT Docs API → Merge News  
    - Merge News → Within dedupe  
    - Within dedupe → Top Headlines  
    - Top Headlines → Classify Headlines  
    - Classify Headlines → Writing Style  
    - Writing Style → Build GPT body  
    - Build GPT body → Research Topic – GPT  
    - Research Topic – GPT → Build Tone Request  
    - Build Tone Request → Tone & Format Revision – GPT  
    - Tone & Format Revision – GPT → Word Count Guard  
    - Word Count Guard → If1  
    - If1 (true) → Expand Draft → Get Title, Content, and Image FileName  
    - If1 (false) → Get Title, Content, and Image FileName  
    - Get Title, Content, and Image FileName → Leonardo Prompt Creator  
    - Leonardo Prompt Creator → Leonardo: Create Post Image  
    - Leonardo: Create Post Image → Code (Keep Generation Id) → Wait → Get Leonardo Image Status → Get Leonardo Image → Upload Image to Wordpress → Add ALT to Image → Create WordPress Post  

26. **Credential Setup**  
    - OpenAI API with appropriate API key for GPT models.  
    - Leonardo AI Bearer Token for image generation API access.  
    - WordPress OAuth2 and Basic Auth credentials for media upload and post creation.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Change the writing style to what you would prefer                                                        | Sticky Note near Writing Style node                                                             |
| Pulls from the Google news RSS to extract headlines                                                      | Sticky Note near Google News RSS node                                                           |
| Removes duplicates, pulls the top headlines for the day and classifies                                   | Sticky Note covering Merge News, Within dedupe, Top Headlines, Classify Headlines nodes          |
| Builds your GPT Body ready for research                                                                  | Sticky Note near Build GPT body node                                                            |
| Researches, builds the narrative tone and formats the article. Then it passes through a word count guard to ensure articles are over 1600 words, if not it will pass through the expand draft. | Sticky Note near Research Topic – GPT and Tone & Format Revision – GPT nodes                      |
| Extracts information from the article and writes a prompt for Leonardo to create a header picture        | Sticky Note near Get Title, Content, and Image FileName and Leonardo Prompt Creator nodes         |
| Creates the image and uploads to WordPress, change to your site/blog location                            | Sticky Note near Leonardo: Create Post Image and Upload Image to Wordpress nodes                  |

---

**Disclaimer**: The above documentation is generated from an n8n workflow automation, respecting all content policies and handling only public and legal data sources.