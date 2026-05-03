Publish blog posts from YouTube transcripts to WordPress and Telegram with Gemini

https://n8nworkflows.xyz/workflows/publish-blog-posts-from-youtube-transcripts-to-wordpress-and-telegram-with-gemini-14758


# Publish blog posts from YouTube transcripts to WordPress and Telegram with Gemini

# Technical Analysis: YouTube to WordPress & Telegram Blog Automation

This document provides a comprehensive technical breakdown of the n8n workflow designed to automatically convert YouTube video transcripts into professional blog posts, publishing them to WordPress and announcing them via Telegram.

---

### 1. Workflow Overview

The purpose of this workflow is to create a fully automated content pipeline that transforms video content into written articles using AI. It eliminates the manual effort of transcribing, writing, image sourcing, and cross-platform publishing.

The logic is divided into three primary autonomous pipelines:

*   **1.1 Transcript Acquisition:** Monitors a YouTube playlist, detects new videos, fetches their transcripts via API, and archives them in Google Docs and Sheets.
*   **1.2 AI Content Generation:** Periodically scans for unprocessed transcripts and uses a Gemini-powered AI agent to write structured blog posts (in Persian), including internal linking and image prompts.
*   **1.3 Multi-Channel Publishing:** Selects pending posts based on priority, publishes them to WordPress, generates a custom AI featured image, and shares the result on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Transcript Acquisition Pipeline
**Overview:** This block manages the ingestion of YouTube data. It ensures only new videos are processed and that their text content is safely stored.

*   **Nodes Involved:** `When Fetch Playlist Scheduled`, `Fetch Playlist Items`, `Split Playlist Items`, `Set Video ID`, `Read Existing Videos from Sheets`, `Filter New Videos Only`, `Append New Videos to Sheets`, `Fetch Transcript via RapidAPI`, `Split Transcript Lines`, `Join Transcript Lines`, `Clean and Aggregate Transcript`, `Create Transcript Google Doc`, `Write to Transcript Google Doc`, `Update Doc Link in Sheets`, `Notify Transcript Saved`.
*   **Node Details:**
    *   **Fetch Playlist Items (HTTP Request):** Calls YouTube v3 API. Uses `playlistId` and `key` as query parameters.
    *   **Filter New Videos Only (Merge):** Performs a "Keep Non-Matches" join between the current API results and the Google Sheet history to prevent duplicate processing.
    *   **Fetch Transcript via RapidAPI (HTTP Request):** Uses a third-party API (`youtube-transcript3`) to retrieve the text of the video.
    *   **Clean and Aggregate Transcript (Code):** A JavaScript node that cleans HTML entities (e.g., `&amp;#39;` $\rightarrow$ `'`) and removes excessive whitespace to prepare text for AI consumption.
    *   **Create/Write Transcript Google Doc (Google Docs):** Creates a new document named after the video title and inserts the cleaned transcript.
    *   **Update Doc Link in Sheets (Google Sheets):** Updates the `youtubeVideos` sheet with the URL of the newly created Google Doc.
    *   **Edge Cases:** API quota limits on YouTube or RapidAPI; videos without available transcripts.

#### 2.2 AI Content Generation Pipeline
**Overview:** This block converts raw transcripts into high-quality articles using an LLM Agent capable of using tools to gather context.

*   **Nodes Involved:** `When Blog Generation Scheduled`, `Fetch All Videos from Sheets`, `Filter Unprocessed Videos`, `Update Video as Processing`, `Blog Post Generator Agent`, `Google Gemini Model`, `Fetch Internal Links Tool`, `Fetch Transcript from Google Docs`, `Parse AI Output`, `Append Blog Post to Sheets`, `Update Video as Completed`, `Notify Content Generation`.
*   **Node Details:**
    *   **Blog Post Generator Agent (AI Agent):** The core engine. It uses a system prompt to act as a Persian media writer. It is configured to output **strictly JSON**.
    *   **Google Gemini Model (LLM):** Uses `gemini-2.5-flash` for high-speed, large-context processing.
    *   **AI Tools:** 
        *   *Fetch Transcript Tool:* Retrieves the specific Google Doc content.
        *   *Internal Links Tool:* Reads the `blogsAndNewsUploaded` sheet to find previous articles for internal linking.
    *   **Parse AI Output (Code):** A robust JavaScript parser that extracts a JSON array from the AI's response, handling potential markdown formatting (like ```json blocks) and repairing common JSON syntax errors.
    *   **Append Blog Post to Sheets (Google Sheets):** Saves the generated title, body, tags, and image prompt to the `blogsAndNewsUploaded` sheet with a `pending` status.
    *   **Edge Cases:** AI failing to follow JSON format; transcript being too long for the model's context window.

#### 2.3 Multi-Channel Publishing Pipeline
**Overview:** This block handles the final delivery of the content, including visual assets and social distribution.

*   **Nodes Involved:** `When Publish Scheduled`, `Fetch Pending Posts from Sheets`, `Choose Priority Post`, `Publish to WordPress`, `Create AI Featured Image`, `Download AI Image`, `Transform Image to WebP`, `Upload Image to WordPress`, `Associate Image with Post`, `Send to Telegram Channel`, `Update Post as Published`.
*   **Node Details:**
    *   **Choose Priority Post (Code):** Logic that sorts pending posts by priority (`urgent` $\rightarrow$ `normal` $\rightarrow$ `evergreen`) and selects the oldest high-priority item.
    *   **Publish to WordPress (WordPress):** Creates a post with the AI-generated body and title.
    *   **Create AI Featured Image (HTTP Request):** Sends the `image_prompt` generated by the AI to an image generation API.
    *   **Transform Image to WebP (Edit Image):** Converts the downloaded AI image to WebP format for web optimization.
    *   **Upload Image to WordPress (HTTP Request):** Uses the WP REST API to upload the binary image data.
    *   **Associate Image with Post (HTTP Request):** Links the uploaded media ID to the specific post ID via the `featured_media` field.
    *   **Send to Telegram Channel (Telegram):** Sends the final image and a formatted caption (including the WP link and channel handle).
    *   **Edge Cases:** WordPress API authentication failure; image generation API timeout; Telegram file size limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Fetch Playlist Scheduled | Schedule Trigger | Trigger pipeline | - | Fetch Playlist Items | Schedule and fetch YouTube |
| Fetch Playlist Items | HTTP Request | Get YT data | When Fetch... | Split Playlist Items, Read Existing... | Schedule and fetch YouTube |
| Split Playlist Items | Split Out | Itemization | Fetch Playlist... | Set Video ID | Process YouTube videos |
| Set Video ID | Set | Variable definition | Split Playlist... | Filter New Videos Only | Process YouTube videos |
| Read Existing Videos... | Google Sheets | History check | Fetch Playlist... | Filter New Videos Only | Process YouTube videos |
| Filter New Videos Only | Merge | Deduplication | Set Video ID, Read Existing... | Append New Videos... | Process YouTube videos |
| Append New Videos... | Google Sheets | Database log | Filter New Videos... | Fetch Transcript... | Log new videos |
| Fetch Transcript via... | HTTP Request | Text retrieval | Append New Videos... | Split Transcript Lines | Fetch and aggregate transcript |
| Split Transcript Lines | Split Out | Line break | Fetch Transcript... | Join Transcript Lines | Fetch and aggregate transcript |
| Join Transcript Lines | Aggregate | Text merging | Split Transcript... | Clean and Aggregate... | Fetch and aggregate transcript |
| Clean and Aggregate... | Code | Text sanitization | Join Transcript... | Create Transcript... | Process transcript text |
| Create Transcript... | Google Docs | File creation | Clean and Aggregate... | Write to Transcript... | Process transcript text |
| Write to Transcript... | Google Docs | File population | Create Transcript... | Update Doc Link... | Process transcript text |
| Update Doc Link... | Google Sheets | Database update | Write to Transcript... | Notify Transcript... | Save and notify transcript |
| Notify Transcript Saved | Telegram | Alert | Update Doc Link... | - | Save and notify transcript |
| When Blog Generation... | Schedule Trigger | Trigger pipeline | - | Fetch All Videos... | Schedule and get videos |
| Fetch All Videos... | Google Sheets | Data retrieval | When Blog Gen... | Filter Unprocessed... | Schedule and get videos |
| Filter Unprocessed... | Filter | Status check | Fetch All Videos... | Update Video... | Filter and update video state |
| Update Video as Proc. | Google Sheets | State locking | Filter Unprocessed... | Blog Post Gen. Agent | Filter and update video state |
| Blog Post Generator Agent | AI Agent | Content writing | Update Video... | Parse AI Output | AI content generation |
| Google Gemini Model | Gemini LLM | Language brain | - | Blog Post Gen. Agent | AI content generation |
| Fetch Internal Links... | GSheet Tool | Context retrieval | - | Blog Post Gen. Agent | AI content generation |
| Fetch Transcript... | GDoc Tool | Context retrieval | - | Blog Post Gen. Agent | AI content generation |
| Parse AI Output | Code | JSON extraction | Blog Post Gen. Agent | Append Blog Post... | Process AI output |
| Append Blog Post... | Google Sheets | Content staging | Parse AI Output | Update Video... | Process AI output |
| Update Video as Compl. | Google Sheets | State update | Append Blog Post... | Notify Content Gen. | Process AI output |
| Notify Content Gen. | Telegram | Alert | Update Video... | - | Notify content generation |
| When Publish Scheduled | Schedule Trigger | Trigger pipeline | - | Fetch Pending Posts... | Schedule and get posts |
| Fetch Pending Posts... | Google Sheets | Queue retrieval | When Publish... | Choose Priority Post | Schedule and get posts |
| Choose Priority Post | Code | Queue management | Fetch Pending... | Publish to WordPress | Select and publish post |
| Publish to WordPress | WordPress | CMS Upload | Choose Priority Post | Create AI Featured... | Select and publish post |
| Create AI Featured Img | HTTP Request | Visual generation | Publish to WordPress | Download AI Image | Generate and process image |
| Download AI Image | HTTP Request | File retrieval | Create AI Featured... | Transform Image... | Generate and process image |
| Transform Image... | Edit Image | Optimization | Download AI Image | Upload Image... | Generate and process image |
| Upload Image to WP | HTTP Request | Media upload | Transform Image... | Associate Image... | Upload and attach image |
| Associate Image... | HTTP Request | Media linking | Upload Image... | Send to Telegram... | Upload and attach image |
| Send to Telegram... | Telegram | Social push | Associate Image... | Update Post as Pub. | Publish and notify post |
| Update Post as Pub. | Google Sheets | Final state update | Send to Telegram... | - | Publish and notify post |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Data Infrastructure (Google Sheets)
Create a Google Spreadsheet with two tabs:
1.  `youtubeVideos`: Columns: `videoID`, `title`, `description`, `videoPublishedAt`, `videoOwnerChannelTitle`, `status`, `transcriptDOCLink`.
2.  `blogsAndNewsUploaded`: Columns: `post_id`, `title`, `summary`, `url`, `tags`, `category`, `status`, `body`, `telegram_body`, `image_prompt`, `publish_priority`, `publish_date`, `row_number`.

#### Step 2: Acquisition Pipeline
1.  **Trigger:** `Schedule Trigger` $\rightarrow$ Set your desired frequency.
2.  **Fetch:** `HTTP Request` $\rightarrow$ GET `googleapis.com/youtube/v3/playlistItems` with API Key and Playlist ID.
3.  **Filter:** `Split Out` the items $\rightarrow$ `Set` the `videoID` $\rightarrow$ `Merge` with a `Google Sheets` (Read) node using `videoID` as the key (Mode: Keep Non-Matches).
4.  **Storage:** `Google Sheets` (Append) the new video details $\rightarrow$ `HTTP Request` to RapidAPI for transcript $\rightarrow$ `Split Out` $\rightarrow$ `Aggregate` $\rightarrow$ `Code` (clean text).
5.  **Docs:** `Google Docs` (Create) $\rightarrow$ `Google Docs` (Update/Insert text) $\rightarrow$ `Google Sheets` (Update) to save the Doc link.

#### Step 3: AI Generation Pipeline
1.  **Trigger:** `Schedule Trigger` (e.g., every 5 mins).
2.  **Queue:** `Google Sheets` (Read) $\rightarrow$ `Filter` where `status == 'N'`.
3.  **Lock:** `Google Sheets` (Update) status to `processing`.
4.  **Agent Setup:** 
    *   `AI Agent` node.
    *   Connect `Google Gemini Model` (gemini-2.5-flash).
    *   Connect `Google Docs Tool` (to read the transcript).
    *   Connect `Google Sheets Tool` (to read `blogsAndNewsUploaded` for internal links).
    *   **Prompt:** Set system message to "Professional Persian AI writer... output ONLY valid JSON array."
5.  **Output:** `Code` node to parse the JSON string into individual items $\rightarrow$ `Google Sheets` (Append) to the blog tab with status `pending`.
6.  **Completion:** `Google Sheets` (Update) original video status to `Y`.

#### Step 4: Publishing Pipeline
1.  **Trigger:** `Schedule Trigger` (e.g., every 3 hours).
2.  **Selection:** `Google Sheets` (Read) $\rightarrow$ `Code` node to sort by `publish_priority` and pick the top 1.
3.  **CMS:** `WordPress` node $\rightarrow$ Create post (status: publish).
4.  **Visuals:** `HTTP Request` (AI Image API) $\rightarrow$ `HTTP Request` (Download) $\rightarrow$ `Edit Image` (Convert to WebP) $\rightarrow$ `HTTP Request` (WP Media API POST).
5.  **Linking:** `HTTP Request` (WP Post API POST) to update `featured_media`.
6.  **Social:** `Telegram` node $\rightarrow$ `sendPhoto` with caption and WP link.
7.  **Finalize:** `Google Sheets` (Update) post status to `published`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| YouTube API Key | Required for playlist fetching |
| RapidAPI Key | Required for transcript extraction |
| Gemini API Key | Required for the AI Agent |
| WordPress Application Password | Required for REST API authentication |
| Telegram Bot Token & Chat ID | Required for notifications and channel posting |
| Persian Language Support | The AI agent is specifically tuned for Persian media output |