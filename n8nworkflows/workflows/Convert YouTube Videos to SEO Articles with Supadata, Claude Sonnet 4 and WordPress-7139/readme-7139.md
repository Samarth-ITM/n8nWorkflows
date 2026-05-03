Convert YouTube Videos to SEO Articles with Supadata, Claude Sonnet 4 and WordPress

https://n8nworkflows.xyz/workflows/convert-youtube-videos-to-seo-articles-with-supadata--claude-sonnet-4-and-wordpress-7139


# Convert YouTube Videos to SEO Articles with Supadata, Claude Sonnet 4 and WordPress

### 1. Workflow Overview

This n8n workflow automates the conversion of YouTube videos into SEO-optimized WordPress draft articles leveraging Supadata transcription, Anthropic Claude Sonnet 4 AI content generation, and YouTube API video discovery. It is designed for content creators and marketers aiming to generate high-quality, SEO-friendly articles based on trending or viral YouTube videos from specific channels, with minimal manual intervention.

The workflow logic is divided into the following blocks:

- **1.1 Trigger & Initialization**  
  Starts the workflow automatically every 6 hours or manually via trigger. Initializes the list of YouTube channels to monitor.

- **1.2 YouTube Channel & Video Discovery**  
  Searches YouTube channels by name, retrieves recent videos, removes duplicates, and filters videos by publication date.

- **1.3 Viral Video Selection and Transcription**  
  From filtered videos, scores and selects the most viral video based on views, likes, and comments, then obtains a transcription via Supadata API.

- **1.4 AI Article Composition**  
  Uses Anthropic Claude Sonnet 4 model to convert the transcription and video metadata into an SEO-optimized HTML article, with brand-specific customization placeholders.

- **1.5 Article Processing and WordPress Posting**  
  Splits generated article HTML into title and body, posts the article as a draft to WordPress via HTTP API.

- **1.6 Utility and Control Nodes**  
  Includes batch processing loops, wait nodes for API rate limiting, and sticky notes with instructions and configuration guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Initialization

**Overview:**  
Defines how and when the workflow starts and sets the YouTube channels to monitor.

**Nodes Involved:**  
- Schedule Trigger (Start every 6 hours)  
- When clicking ‘Test workflow’ (Manual Trigger)  
- SET YOUTUBE CHANNELS  
- Divide items into Title and Search phrases  
- Loop Over Items1  

**Node Details:**  

- **Schedule Trigger (Start every 6 hours):**  
  - Type: scheduleTrigger  
  - Role: Automatically starts the workflow every 6 hours using a cron expression.  
  - Inputs: None  
  - Outputs: Triggers the next node.  
  - Edge cases: Cron misconfiguration can cause missed or extra runs.

- **When clicking ‘Test workflow’:**  
  - Type: manualTrigger  
  - Role: Allows manual execution for testing or immediate runs.  
  - Inputs: None  
  - Outputs: Triggers downstream nodes.  

- **SET YOUTUBE CHANNELS:**  
  - Type: set  
  - Role: Holds user-defined YouTube channel names as variables (e.g., "Youtube channel 1").  
  - Configuration: Three string assignments representing channel names.  
  - Outputs: JSON object with channel names to be processed.  
  - Edge cases: Empty or incorrect channel names will produce no results.

- **Divide items into Title and Search phrases:**  
  - Type: code (JavaScript)  
  - Role: Converts the object of YouTube channel names into an array of items with `title` and `search` fields, preparing for batch processing.  
  - Inputs: JSON object from SET YOUTUBE CHANNELS.  
  - Outputs: Array of objects each with title and search strings.

- **Loop Over Items1:**  
  - Type: splitInBatches  
  - Role: Processes each channel in batches to handle API calls sequentially, preventing overload.  
  - Inputs: Array of channels from previous node.  
  - Outputs: Single channel items fed downstream.  

---

#### 2.2 YouTube Channel & Video Discovery

**Overview:**  
Searches for YouTube channels matching the names, retrieves recent videos, filters duplicates and recent uploads.

**Nodes Involved:**  
- Search Youtube Channel  
- Wait for 5 seconds (avoid API overload)  
- Get YouTube Channel Videos  
- Remove Duplicates  
- Filter by Date  
- Loop Over Items2  

**Node Details:**  

- **Search Youtube Channel:**  
  - Type: httpRequest  
  - Role: Uses YouTube Data API v3 to search for channels by name (parameterized by `search`).  
  - Key parameters: `part=snippet`, `type=channel`, `regionCode=US`, `maxResults=10`, API key required.  
  - Outputs: JSON with channel search results.  
  - Edge cases: API quota exceeded, invalid API key, no channels found.

- **Wait for 5 seconds (avoid API overload):**  
  - Type: wait  
  - Role: Adds a 5-second delay between API requests to respect rate limits.  
  - Inputs: From Search Youtube Channel.  
  - Outputs: Delayed output for next node.

- **Get YouTube Channel Videos:**  
  - Type: youTube  
  - Role: Retrieves up to 10 videos from the channel ID obtained from previous search, ordered by date.  
  - Inputs: Channel ID from search results.  
  - Outputs: Video list JSON.  
  - Edge cases: Empty channel, API limits.

- **Remove Duplicates:**  
  - Type: removeDuplicates  
  - Role: Filters out videos with duplicate titles to avoid repeated content.  
  - Configuration: Compares on `snippet.title`.  
  - Inputs: Video list.  
  - Outputs: Filtered video list.

- **Filter by Date:**  
  - Type: filter  
  - Role: Keeps videos published within the last 60 hours (216,000 seconds).  
  - Condition: Video publish timestamp greater than (now - 216000 seconds).  
  - Inputs: Deduplicated video list.  
  - Outputs: Recent videos only.

- **Loop Over Items2:**  
  - Type: splitInBatches  
  - Role: Processes each filtered video individually in batches to avoid API overload downstream.  
  - Inputs: Filtered videos.  
  - Outputs: Single video items.

---

#### 2.3 Viral Video Selection and Transcription

**Overview:**  
Scores videos by popularity metrics, picks the highest scoring viral video, then retrieves its transcription using Supadata.

**Nodes Involved:**  
- Get the most Viral Video  
- Get YouTube Video  
- Create Transcription  

**Node Details:**  

- **Get the most Viral Video:**  
  - Type: code  
  - Role: Custom scoring function ranks videos by weighted sum: views*20 + likes*10 + comments*1.  
  - Outputs: Single video with highest score.  
  - Inputs: Video batch from Loop Over Items2.  
  - Edge cases: Missing stats fields treated as zero, ties resolved by sort order.

- **Get YouTube Video:**  
  - Type: youTube  
  - Role: Retrieves detailed metadata for the selected video by videoId.  
  - Inputs: videoId from viral video selection.  
  - Outputs: Detailed video JSON.  
  - Edge cases: Invalid video ID, quota issues.

- **Create Transcription:**  
  - Type: httpRequest  
  - Role: Calls Supadata API to generate or fetch the transcript for the video ID.  
  - Configuration: Requires Supadata API key in header `X-API-KEY`. Query params include `videoId` and `text=true`.  
  - Inputs: video ID JSON.  
  - Outputs: Transcript text JSON.  
  - Edge cases: API key missing or invalid, network errors, API limits.

---

#### 2.4 AI Article Composition

**Overview:**  
Transforms the transcript and video metadata into a fully SEO-optimized, brand-customized article in valid HTML using Anthropic Claude Sonnet 4.

**Nodes Involved:**  
- Anthropic Chat Model1  
- Compose Article  

**Node Details:**  

- **Anthropic Chat Model1:**  
  - Type: lmChatAnthropic (Langchain integration)  
  - Role: Connects to Anthropic Claude Sonnet 4 model for chat completion AI tasks.  
  - Configuration: Model set to "claude-sonnet-4-20250514", with thinking enabled.  
  - Inputs: Prompt and input data from Compose Article node.  
  - Outputs: AI-generated text completion.  
  - Edge cases: API key or rate limits, model errors.

- **Compose Article:**  
  - Type: langchain.agent  
  - Role: Defines the prompt for AI to generate an SEO article from transcript and metadata.  
  - Configuration: Large detailed prompt including instructions for keyword usage, SEO checklist, brand tone, output format in HTML, and placeholders for brand customization.  
  - Inputs: Transcript content, video description, title, and placeholders for keywords, author name, tone, etc.  
  - Outputs: Generated article HTML string.  
  - Edge cases: Missing placeholders, incomplete input data.

---

#### 2.5 Article Processing and WordPress Posting

**Overview:**  
Splits the HTML article into title and body content, then creates a WordPress draft post via HTTP API.

**Nodes Involved:**  
- Split title and body  
- Create WordPress Post  

**Node Details:**  

- **Split title and body:**  
  - Type: code  
  - Role: Extracts the article’s `<h1>` as title, removes it from the body content.  
  - Inputs: Full HTML article string from Compose Article.  
  - Outputs: JSON with separate `title` and `body`.  
  - Edge cases: Missing `<h1>` tag leads to empty title.

- **Create WordPress Post:**  
  - Type: httpRequest  
  - Role: Makes a POST request to WordPress REST API to create a new post in draft status.  
  - Configuration: URL must be set to your WordPress site endpoint, body includes `title`, `content` (body), and `status` = draft. Uses multipart/form-data.  
  - Inputs: title and body from previous node.  
  - Outputs: API response.  
  - Edge cases: Authentication failure, invalid URL, API errors.

---

#### 2.6 Utility and Instructional Nodes

**Overview:**  
Sticky notes provide configuration and operational guidance for users.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note7  

**Details:**  
- Sticky Note: Marks "Search New Videos" section.  
- Sticky Note1: Marks "Get the Most Viral Video" section.  
- Sticky Note2: Marks "Write the Article and Post it to WordPress drafts" section.  
- Sticky Note7: Detailed instructions for setup, including API integrations for Google, Supadata, Anthropic, WordPress, and customization of placeholders in AI prompt. Also includes tutorial video links.

---

### 3. Summary Table

| Node Name                        | Node Type                         | Functional Role                                      | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                                           |
|---------------------------------|----------------------------------|-----------------------------------------------------|--------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger (Start every 6 hours) | scheduleTrigger                  | Start workflow automatically every 6 hours         |                                | SET YOUTUBE CHANNELS            |                                                                                                                                      |
| When clicking ‘Test workflow’    | manualTrigger                    | Manual start trigger for testing                     |                                | SET YOUTUBE CHANNELS            |                                                                                                                                      |
| SET YOUTUBE CHANNELS             | set                              | Define YouTube channels to monitor                   | Schedule Trigger, Manual Trigger | Divide items into Title and Search phrases |                                                                                                                                      |
| Divide items into Title and Search phrases | code                             | Convert channels object to array for batch processing | SET YOUTUBE CHANNELS           | Loop Over Items1                |                                                                                                                                      |
| Loop Over Items1                 | splitInBatches                   | Process channels in batches to avoid API overload    | Divide items into Title and Search phrases | Search Youtube Channel, Get YouTube Channel Videos |                                                                                                                                      |
| Search Youtube Channel           | httpRequest                     | Search YouTube API for channels by name              | Loop Over Items1               | Wait for 5 seconds (avoid API overload) | Sticky Note: "## Search New Videos"                                                                                                  |
| Wait for 5 seconds (avoid API overload) | wait                             | Delay to respect API rate limits                      | Search Youtube Channel          | Loop Over Items1               |                                                                                                                                      |
| Get YouTube Channel Videos       | youTube                         | Get latest videos from a YouTube channel              | Loop Over Items1               | Remove Duplicates               | Sticky Note1: "## Get the Most Viral Video (based on number of likes, views and comments)"                                            |
| Remove Duplicates                | removeDuplicates                | Remove videos with duplicate titles                   | Get YouTube Channel Videos      | Filter by Date                 |                                                                                                                                      |
| Filter by Date                  | filter                          | Filter videos published within the last 60 hours     | Remove Duplicates              | Loop Over Items2               |                                                                                                                                      |
| Loop Over Items2                 | splitInBatches                  | Batch processing of filtered videos                   | Filter by Date                | Get the most Viral Video, Get YouTube Video |                                                                                                                                      |
| Get the most Viral Video         | code                            | Score and select the most viral video                 | Loop Over Items2              | Create Transcription           | Sticky Note2: "## Write the Article and Post it to WordPress drafts"                                                                  |
| Get YouTube Video               | youTube                         | Retrieve detailed metadata for viral video            | Get the most Viral Video       | Loop Over Items2               |                                                                                                                                      |
| Create Transcription            | httpRequest                     | Get video transcript via Supadata API                 | Get the most Viral Video       | Compose Article               | Sticky Note7: Detailed configuration and setup instructions, includes tutorial links                                                  |
| Anthropic Chat Model1           | lmChatAnthropic                 | AI model for text generation                           | Compose Article               | Compose Article               |                                                                                                                                      |
| Compose Article                | langchain.agent                 | Generate SEO article HTML from transcript and metadata | Create Transcription          | Split title and body          |                                                                                                                                      |
| Split title and body            | code                            | Extract title and body from HTML article               | Compose Article               | Create WordPress Post          |                                                                                                                                      |
| Create WordPress Post           | httpRequest                     | Post article as draft to WordPress via API            | Split title and body          |                             |                                                                                                                                      |
| Sticky Note                    | stickyNote                     | Visual label: "Search New Videos"                      |                              |                              |                                                                                                                                      |
| Sticky Note1                   | stickyNote                     | Visual label: "Get the Most Viral Video ..."           |                              |                              |                                                                                                                                      |
| Sticky Note2                   | stickyNote                     | Visual label: "Write the Article and Post it to WordPress drafts" |                              |                              |                                                                                                                                      |
| Sticky Note7                   | stickyNote                     | Workflow configuration instructions with tutorial links |                              |                              | Contains setup instructions & links: https://www.youtube.com/watch?v=BfW1JpJ39Ek and https://www.youtube.com/watch?v=1jl_vBoVvq0 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Type: scheduleTrigger  
   - Set cron expression to `0 */6 * * *` to run every 6 hours.

2. **Create Manual Trigger**  
   - Type: manualTrigger  
   - For manual workflow testing.

3. **Create ‘SET YOUTUBE CHANNELS’ Node**  
   - Type: set  
   - Add string fields: "Youtube channel 1", "Youtube channel 2", "Youtube channel 3 (add more if needed)" with your target channel names.

4. **Create ‘Divide items into Title and Search phrases’ Node**  
   - Type: code (JavaScript)  
   - Code: Convert the `SET YOUTUBE CHANNELS` JSON object into an array of items with `title` and `search` fields.

5. **Create ‘Loop Over Items1’ Node**  
   - Type: splitInBatches  
   - No special options needed (batch size default).

6. **Create ‘Search Youtube Channel’ Node**  
   - Type: httpRequest  
   - Method: GET  
   - URL: `https://www.googleapis.com/youtube/v3/search`  
   - Query parameters:  
     - `part=snippet`  
     - `q={{ $json.search }}`  
     - `type=channel`  
     - `key=YOUR_YOUTUBE_API_KEY` (replace with your API key)  
     - `regionCode=US`  
     - `maxResults=10`

7. **Create ‘Wait for 5 seconds (avoid API overload)’ Node**  
   - Type: wait  
   - Duration: 5 seconds.

8. **Create ‘Get YouTube Channel Videos’ Node**  
   - Type: youTube  
   - Operation: Get videos  
   - Channel ID: `{{ $json.items[0].id.channelId }}`  
   - Limit: 10  
   - Order: date

9. **Create ‘Remove Duplicates’ Node**  
   - Type: removeDuplicates  
   - Fields to compare: `snippet.title`

10. **Create ‘Filter by Date’ Node**  
    - Type: filter  
    - Condition:  
      - Left value: `{{ $json.snippet.publishTime.toDateTime().toMillis() / 1000 }}`  
      - Operator: greater than (gt)  
      - Right value: current time minus 216000 seconds (60 hours).

11. **Create ‘Loop Over Items2’ Node**  
    - Type: splitInBatches

12. **Create ‘Get the most Viral Video’ Node**  
    - Type: code  
    - JavaScript: Sort videos by weighted popularity score (views*20 + likes*10 + comments), output top video.

13. **Create ‘Get YouTube Video’ Node**  
    - Type: youTube  
    - Operation: get  
    - Video ID: `{{ $json.id.videoId }}`

14. **Create ‘Create Transcription’ Node**  
    - Type: httpRequest  
    - Method: GET  
    - URL: `https://api.supadata.ai/v1/youtube/transcript`  
    - Query parameters: `videoId={{ $json.id }}`, `text=true`  
    - Header: `X-API-KEY: YOUR_SUPADATA_API_KEY`

15. **Create ‘Anthropic Chat Model1’ Node**  
    - Type: lmChatAnthropic (Langchain)  
    - Model: “claude-sonnet-4-20250514”  
    - Enable thinking option.

16. **Create ‘Compose Article’ Node**  
    - Type: langchain.agent  
    - Text prompt: Insert the detailed prompt provided, replacing placeholders `[TARGET_KEYWORD_LIST_PLACEHOLDER]`, `[AUTHOR_NAME_PLACEHOLDER]`, `[BRAND_TONE_ADJECTIVES]`, etc. with your brand values and SEO data.

17. **Connect ‘Anthropic Chat Model1’ output to ‘Compose Article’ input** (ai_languageModel connection).

18. **Create ‘Split title and body’ Node**  
    - Type: code  
    - JavaScript: Extract `<h1>` content as title, remove it from article body.

19. **Create ‘Create WordPress Post’ Node**  
    - Type: httpRequest  
    - Method: POST  
    - URL: Your WordPress REST API endpoint (e.g., `https://yourdomain.com/wp-json/wp/v2/posts`)  
    - Body parameters:  
      - `title`: `{{ $json.title }}`  
      - `content`: `{{ $json.body }}`  
      - `status`: `draft`  
    - Content type: multipart-form-data  
    - Configure authentication as needed (e.g., OAuth2 or Basic Auth).

20. **Add Sticky Notes** for documentation and user guidance at appropriate places:  
    - "Search New Videos" near YouTube search nodes  
    - "Get the Most Viral Video" near viral video selection  
    - "Write the Article and Post it to WordPress drafts" near AI and posting nodes  
    - Detailed configuration instructions including API keys and tutorials.

21. **Connect nodes as per the workflow connections:**  
    - Schedule Trigger & Manual Trigger → SET YOUTUBE CHANNELS → Divide items → Loop Over Items1 → Search Youtube Channel → Wait 5s → Loop Over Items1 → Get YouTube Channel Videos → Remove Duplicates → Filter by Date → Loop Over Items2 → Get the most Viral Video → Create Transcription → Anthropic Chat Model1 → Compose Article → Split title and body → Create WordPress Post.

22. **Test the workflow** after configuring API keys and placeholders.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                     | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| How to configure the workflow: Requires Google account for YouTube nodes, Supadata API key for transcription, Anthropic API key for AI article generation, WordPress credentials | Sticky Note7 content in workflow; includes video tutorials for setup                                |
| Recommended AI model: Claude Sonnet 4 for article generation; GPT models also supported                                                                                         | Sticky Note7 and Anthropic Chat Model1 node                                                        |
| YouTube API quota limitations and authentication are critical for smooth operation                                                                                              | Applies to Search Youtube Channel and YouTube nodes                                                |
| SEO and content prompt placeholders must be customized to fit brand tone, keyword strategy, and internal linking                                                                | Compose Article node prompt                                                                        |
| Workflow incorporates batch processing and delays to mitigate API rate limits                                                                                                   | Loop Over Items nodes and Wait node                                                                |
| Video scoring formula prioritizes views and likes over comments for viral content selection                                                                                      | Get the most Viral Video node                                                                       |
| WordPress posting requires REST API endpoint and valid authentication, supports draft posts                                                                                      | Create WordPress Post node                                                                          |
| Tutorial videos for YouTube setup: https://www.youtube.com/watch?v=BfW1JpJ39Ek and Anthropic setup: https://www.youtube.com/watch?v=1jl_vBoVvq0                              | Sticky Note7                                                                                        |

---

This detailed reference document captures the entire n8n workflow’s logic, configuration, and integration points, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.

---

*Disclaimer: The provided text is extracted exclusively from an automated n8n workflow. It complies strictly with applicable content policies, contains no illegal or offensive material, and handles only legal and public data.*