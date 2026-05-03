Create & Publish AI News Videos to Social Media with GPT-4o & HeyGen

https://n8nworkflows.xyz/workflows/create---publish-ai-news-videos-to-social-media-with-gpt-4o---heygen-6524


# Create & Publish AI News Videos to Social Media with GPT-4o & HeyGen

### 1. Workflow Overview

This workflow automates the creation and publishing of AI-generated news videos to social media platforms using GPT-4o-powered caption generation and HeyGen avatar video synthesis. It is designed for content creators, social media managers, and digital marketers who want to streamline news video production and cross-platform distribution.

The workflow is divided into the following logical blocks:

- **1.1 Trigger & News Fetching:** Manual trigger initiates the workflow which reads the latest news articles from the CNN RSS feed.
- **1.2 News Logging:** Logs fetched news data into Google Sheets for record keeping.
- **1.3 AI Caption Generation:** Uses an AI agent (LangChain with GPT-4o) to generate engaging social media captions based on the news articles.
- **1.4 HeyGen Video Preparation & Generation:** Prepares parameters and calls HeyGen API to generate avatar videos with the AI captions.
- **1.5 Video Handling & Storage:** Waits for video processing completion, downloads the video, then uploads it to Google Drive and Postiz platform, logging video details.
- **1.6 Postiz Integration & Social Publishing:** Retrieves Postiz social media integrations, routes the workflow to platform-specific branches (Instagram, Facebook, YouTube), cleans captions accordingly, and publishes videos via Postiz API.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & News Fetching

- **Overview:** Starts the workflow manually and fetches the latest news from the CNN RSS feed, limiting to one item for testing or control.
- **Nodes Involved:** 
  - When clicking 'Test workflow' (Manual Trigger)
  - RSS Read
  - Limit1
- **Node Details:**

  - **When clicking 'Test workflow'**
    - Type: Manual Trigger
    - Role: Initiates workflow execution manually.
    - Config: Default, no parameters.
    - Inputs: None (initiator).
    - Outputs: Triggers RSS Read.
    - Edge Cases: None significant; manual invocation.
  
  - **RSS Read**
    - Type: RSS Feed Read
    - Role: Fetches news articles from specified RSS feed URL.
    - Config: URL set to `http://rss.cnn.com/rss/edition.rss`; default options.
    - Inputs: Trigger from manual node.
    - Outputs: Raw news feed items.
    - Edge Cases: RSS feed downtime or malformed data; network errors.
  
  - **Limit1**
    - Type: Limit
    - Role: Limits number of items processed (set to 1) to control execution scope.
    - Config: `executeOnce` enabled.
    - Inputs: RSS Read output.
    - Outputs: Single news item.
    - Edge Cases: No items fetched; zero or empty input.

---

#### 2.2 News Logging

- **Overview:** Records the fetched news article data into Google Sheets for historical archiving and analysis.
- **Nodes Involved:** 
  - Log news to sheets
- **Node Details:**

  - **Log news to sheets**
    - Type: Google Sheets
    - Role: Appends news article details (title, link, guid, content, pubDate, isoDate) to a specific Google Sheet.
    - Config: 
      - `documentId`: Points to the "RSS FEEDS" spreadsheet.
      - `sheetName`: Sheet 1 (gid=0).
      - Operation: Append rows with defined columns mapped from RSS fields.
    - Inputs: From Limit1 node.
    - Outputs: Passes data to AI Agent node.
    - Edge Cases: Google Sheets API auth errors, quota exceeded, invalid spreadsheet ID.

---

#### 2.3 AI Caption Generation

- **Overview:** Uses an AI agent powered by GPT-4o to generate concise, engaging captions (30–60 words) based on news title and content.
- **Nodes Involved:** 
  - AI Agent (LangChain Agent)
  - write script (Chat OpenAI)
  - Parse Caption (Code)
- **Node Details:**

  - **AI Agent**
    - Type: LangChain Agent
    - Role: Runs an AI agent to generate a short social media caption.
    - Config: 
      - Text prompt includes news Title and Content.
      - Output: Single concise caption.
    - Inputs: From Log news to sheets.
    - Outputs: To Parse Caption.
    - Edge Cases: OpenAI API rate limiting, malformed prompt, empty news content.
  
  - **write script**
    - Type: LangChain LM Chat OpenAI
    - Role: Provides underlying language model for AI Agent.
    - Config: Defaults; connected as language model for AI Agent.
    - Inputs: AI Agent (ai_languageModel input).
    - Outputs: AI Agent receives model output.
    - Edge Cases: API key invalidation, model unavailable.
  
  - **Parse Caption**
    - Type: Code
    - Role: Extracts the generated caption text from the AI Agent output into simpler JSON structure.
    - Config: Maps output array to `scripts` array.
    - Inputs: AI Agent output.
    - Outputs: To Setup Heygen Parameters.
    - Edge Cases: Unexpected AI Agent output formats, empty responses.

---

#### 2.4 HeyGen Video Preparation & Generation

- **Overview:** Sets required parameters (API key, avatar ID, voice ID, caption, title) and calls HeyGen API to generate an avatar video from the AI caption.
- **Nodes Involved:** 
  - Setup Heygen Parameters
  - Create Avatar Video (HeyGen)
  - Wait for Video (HeyGen)
  - Get Avatar Video Status (HeyGen)
- **Node Details:**

  - **Setup Heygen Parameters**
    - Type: Set
    - Role: Defines HeyGen API parameters, including API key, avatar and voice IDs, caption text, and news title.
    - Config: Static or dynamic values; placeholders must be replaced with actual credentials and IDs.
    - Inputs: From Parse Caption.
    - Outputs: To Create Avatar Video.
    - Edge Cases: Missing or invalid API key, wrong avatar/voice IDs.
  
  - **Create Avatar Video (HeyGen)**
    - Type: HTTP Request
    - Role: Calls HeyGen API endpoint to generate video using avatar and voice with caption.
    - Config: 
      - POST to `https://api.heygen.com/v2/video/generate`
      - JSON body includes avatar_id, voice_id, input text (caption), video dimension.
      - Header includes `X-Api-Key`.
    - Inputs: From Setup Heygen Parameters.
    - Outputs: To Wait for Video.
    - Edge Cases: API rate limiting, invalid API key, malformed JSON, server errors.
  
  - **Wait for Video (HeyGen)**
    - Type: Wait
    - Role: Pauses workflow for 2 minutes to allow video generation.
    - Config: Wait 2 minutes (adjustable).
    - Inputs: From Create Avatar Video.
    - Outputs: To Get Avatar Video Status.
    - Edge Cases: Insufficient wait time leading to premature status checks.
  
  - **Get Avatar Video Status (HeyGen)**
    - Type: HTTP Request
    - Role: Polls HeyGen API for video generation status using video_id.
    - Config: 
      - GET `https://api.heygen.com/v1/video_status.get?video_id=...`
      - Header includes `X-Api-Key`.
    - Inputs: From Wait for Video.
    - Outputs: To Download Video.
    - Edge Cases: Video generation failure, API errors, video_id invalid or expired.

---

#### 2.5 Video Handling & Storage

- **Overview:** Downloads the generated video, uploads it to Google Drive for archival, also uploads to Postiz platform, and logs video details.
- **Nodes Involved:** 
  - Download Video
  - Upload to Google Drive
  - Upload Video to Postiz
  - Log Video Details to Sheets
- **Node Details:**

  - **Download Video**
    - Type: HTTP Request
    - Role: Downloads the video binary from HeyGen video URL.
    - Config: URL taken from HeyGen video status response.
    - Inputs: From Get Avatar Video Status.
    - Outputs: To Upload to Google Drive and Upload Video to Postiz.
    - Edge Cases: Download failures, URL expiration.
  
  - **Upload to Google Drive**
    - Type: Google Drive
    - Role: Uploads downloaded video to Google Drive root folder.
    - Config: Drive ID “My Drive”, folder “root”.
    - Inputs: From Download Video.
    - Outputs: To Log Video Details to Sheets.
    - Edge Cases: Google Drive quota, auth errors.
  
  - **Upload Video to Postiz**
    - Type: HTTP Request
    - Role: Uploads video binary to Postiz internal storage.
    - Config: 
      - POST to `https://postiz.yourdomain.com/api/public/v1/upload`
      - Content type: multipart-form-data with file binary field.
      - Authentication: Postiz HTTP Header Auth.
    - Inputs: From Download Video.
    - Outputs: To Get Postiz Integrations.
    - Edge Cases: API auth failure, upload size limits, network errors.
  
  - **Log Video Details to Sheets**
    - Type: Google Sheets
    - Role: Logs video metadata including title, caption, HeyGen video URL, Google Drive file ID.
    - Config:
      - Spreadsheet ID and gid=0 for the "Avatar video" sheet.
      - Appends data with defined columns.
    - Inputs: From Upload to Google Drive.
    - Outputs: None.
    - Edge Cases: Auth or quota issues.

---

#### 2.6 Postiz Integration & Social Publishing

- **Overview:** Fetches social media integrations from Postiz, routes to platform-specific publishing branches, cleans captions, and publishes videos with appropriate metadata.
- **Nodes Involved:** 
  - Get Postiz Integrations
  - Video Platform Router (Switch)
  - Clean Instagram Caption (Code)
  - Instagram Video Publisher (HTTP Request)
  - Clean Facebook Video Caption (Code)
  - Facebook Video Publisher (HTTP Request)
  - YouTube Video Publisher (HTTP Request)
- **Node Details:**

  - **Get Postiz Integrations**
    - Type: HTTP Request
    - Role: Fetches available social media integrations from Postiz.
    - Config: GET `https://postiz.yourdomain.com/api/public/v1/integrations`
    - Auth: Postiz HTTP Header Auth.
    - Inputs: From Upload Video to Postiz.
    - Outputs: To Video Platform Router.
    - Edge Cases: API auth failure, empty integration list.
  
  - **Video Platform Router**
    - Type: Switch
    - Role: Routes workflow based on integration identifier (`instagram`, `facebook`, `youtube`).
    - Config: Switch rules comparing `identifier` field.
    - Inputs: From Get Postiz Integrations.
    - Outputs: To platform-specific cleaning nodes.
    - Edge Cases: Unknown platform identifiers.
  
  - **Clean Instagram Caption**
    - Type: Code
    - Role: Sanitizes caption for Instagram video posts:
      - Replaces line breaks, tabs with spaces.
      - Replaces multiple spaces with single spaces.
      - Escapes quotes and backslashes.
      - Trims whitespace.
      - Enforces Instagram caption limit of 2200 chars.
    - Inputs: From Video Platform Router.
    - Outputs: To Instagram Video Publisher.
    - Edge Cases: Caption too long, invalid characters causing JSON errors.
  
  - **Instagram Video Publisher**
    - Type: HTTP Request
    - Role: Publishes video to Instagram via Postiz API.
    - Config: POST to `https://postiz.yourdomain.com/api/public/v1/posts`
      - Sends cleaned caption, video path.
      - Immediate posting, with tag "instagram".
    - Auth: Postiz HTTP Header Auth.
    - Inputs: From Clean Instagram Caption.
    - Outputs: None.
    - Edge Cases: API errors, auth failures.
  
  - **Clean Facebook Video Caption**
    - Type: Code
    - Role: Sanitizes caption for Facebook video posts similarly to Instagram but with a 63206 character limit.
    - Inputs: From Video Platform Router.
    - Outputs: To Facebook Video Publisher.
    - Edge Cases: Excessively long captions truncated.
  
  - **Facebook Video Publisher**
    - Type: HTTP Request
    - Role: Publishes video to Facebook via Postiz API.
    - Config: Similar to Instagram publisher but no tags.
    - Inputs: From Clean Facebook Video Caption.
    - Outputs: None.
  
  - **YouTube Video Publisher**
    - Type: HTTP Request
    - Role: Publishes video to YouTube via Postiz API.
    - Config:
      - Uses news title as video title.
      - Original AI caption as description.
      - Sets YouTube metadata (categoryId, tags, madeForKids).
      - Visibility is public.
    - Inputs: From Video Platform Router.
    - Outputs: None.
    - Edge Cases: YouTube API quota, metadata validation.

---

### 3. Summary Table

| Node Name                      | Node Type                 | Functional Role                          | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                               |
|--------------------------------|---------------------------|----------------------------------------|----------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------|
| When clicking 'Test workflow'   | Manual Trigger            | Workflow manual start trigger          | None                             | RSS Read                        | 🚀 **Workflow Trigger:** Manual start for testing and execution.                                         |
| RSS Read                       | RSS Feed Read             | Fetches news from CNN RSS feed          | When clicking 'Test workflow'    | Limit1                         | 📰 **News Feed Source:** Pulls latest news articles from CNN RSS feed.                                   |
| Limit1                        | Limit                     | Limits news items processed (1 item)   | RSS Read                        | Log news to sheets             | ⚡ **Data Limiter:** Limits items to 1 for testing and control.                                          |
| Log news to sheets             | Google Sheets             | Logs news article data                  | Limit1                         | AI Agent                      | 📊 **News Article Logger:** Appends news details to Google Sheets.                                      |
| AI Agent                      | LangChain Agent           | Generates social media caption using AI| Log news to sheets              | Parse Caption                 | ✍️ **Caption Generation AI:** Generates concise caption with GPT-4o.                                   |
| write script                  | LangChain LM Chat OpenAI  | Provides language model for AI Agent   | AI Agent (ai_languageModel)     | AI Agent                      |                                                                                                         |
| Parse Caption                 | Code                      | Extracts caption from AI Agent output  | AI Agent                       | Setup Heygen Parameters       | 🧹 **Caption Extractor:** Simplifies AI output for downstream use.                                      |
| Setup Heygen Parameters       | Set                       | Prepares parameters for HeyGen API     | Parse Caption                  | Create Avatar Video (HeyGen)  | 🔧 **HeyGen Configuration:** Sets API key, avatar/voice IDs, caption, and title.                         |
| Create Avatar Video (HeyGen)   | HTTP Request              | Calls HeyGen API to generate video     | Setup Heygen Parameters        | Wait for Video (HeyGen)       | 🎬 **Video Generation & Status Check:** Calls HeyGen API and waits for video processing.                 |
| Wait for Video (HeyGen)        | Wait                      | Waits fixed time for video processing  | Create Avatar Video (HeyGen)   | Get Avatar Video Status (HeyGen)|                                                                                                         |
| Get Avatar Video Status (HeyGen)| HTTP Request             | Polls HeyGen for video status           | Wait for Video (HeyGen)         | Download Video                |                                                                                                         |
| Download Video                | HTTP Request              | Downloads generated video binary       | Get Avatar Video Status (HeyGen)| Upload to Google Drive, Upload Video to Postiz |                                                                                                         |
| Upload to Google Drive         | Google Drive              | Stores video in Google Drive            | Download Video                 | Log Video Details to Sheets   | 📊 **Video Details Logger:** Logs video info to Google Sheets.                                          |
| Upload Video to Postiz         | HTTP Request              | Uploads video binary to Postiz          | Download Video                 | Get Postiz Integrations       | ⬆️ **Postiz Video Upload:** Uploads video for social publishing; requires proper API key credential.     |
| Log Video Details to Sheets    | Google Sheets             | Logs video metadata                     | Upload to Google Drive         | None                         |                                                                                                         |
| Get Postiz Integrations        | HTTP Request              | Fetches social media integrations       | Upload Video to Postiz         | Video Platform Router         |                                                                                                         |
| Video Platform Router          | Switch                    | Routes workflow by social platform      | Get Postiz Integrations         | Clean Instagram Caption, Clean Facebook Video Caption, YouTube Video Publisher | 🎯 **Dynamic Platform Routing:** Routes to platform-specific publishing branches.                        |
| Clean Instagram Caption        | Code                      | Cleans and truncates Instagram caption | Video Platform Router (Instagram)| Instagram Video Publisher    | 🧼 **Instagram Caption Cleaner:** Sanitizes caption for Instagram, escapes quotes and limits length.      |
| Instagram Video Publisher      | HTTP Request              | Publishes video to Instagram via Postiz| Clean Instagram Caption         | None                         | 📸 **Publish to Instagram:** Posts video with cleaned caption and path to Postiz.                        |
| Clean Facebook Video Caption   | Code                      | Cleans and truncates Facebook caption  | Video Platform Router (Facebook)| Facebook Video Publisher     |                                                                                                         |
| Facebook Video Publisher       | HTTP Request              | Publishes video to Facebook via Postiz | Clean Facebook Video Caption    | None                         |                                                                                                         |
| YouTube Video Publisher        | HTTP Request              | Publishes video to YouTube via Postiz  | Video Platform Router (YouTube) | None                         | 📺 **Publish to YouTube:** Posts video with title, description, tags, and category metadata.             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**
   - Type: Manual Trigger
   - Use default settings
   - Position: Start of workflow
   - Purpose: To manually start the workflow

2. **Add RSS Feed Read Node**
   - Connect Manual Trigger → RSS Feed Read
   - Set URL to `http://rss.cnn.com/rss/edition.rss`
   - Default options
   - Fetches latest news articles

3. **Add Limit Node**
   - Connect RSS Feed Read → Limit
   - Configure to limit to 1 item for controlled testing
   - Enable `executeOnce` to true

4. **Add Google Sheets Node for News Logging**
   - Connect Limit → Google Sheets
   - Operation: Append
   - Spreadsheet: "RSS FEEDS" (use document ID `1UdfAbMMkJssMVu2qJy2swscL-dETUbjkervC08TYgFo`)
   - Sheet Name: `Sheet1` (gid=0)
   - Map columns: Guid, Link, Title, Content, IsoDate, pubDate from RSS item fields
   - Credential: Google Sheets account with write access

5. **Add LangChain AI Agent Node**
   - Connect Google Sheets → AI Agent
   - Prompt: Generate a short, engaging caption (30–60 words) based on news title and content
   - Use news fields Title and Content in prompt expressions
   - Configure to output a single concise caption

6. **Add LangChain Chat OpenAI Node**
   - Connect LangChain Chat OpenAI → AI Agent (as language model)
   - Use OpenAI credentials with GPT-4o-mini or GPT-4o model
   - Default settings suffice

7. **Add Code Node to Parse Caption**
   - Connect AI Agent → Code Node
   - JavaScript: Extract `output` field from AI Agent results into `scripts` array
   - Output simplified JSON with caption text

8. **Add Set Node to Setup HeyGen Parameters**
   - Connect Parse Caption → Set Node
   - Define variables:
     - `heygen_api_key`: Your HeyGen API key (replace placeholder)
     - `avatar_id`: Your HeyGen avatar ID
     - `voice_id`: Your HeyGen voice ID
     - `caption`: Use parsed caption
     - `news_title`: Use news Title
   - Ensure all required parameters are set for HeyGen API call

9. **Add HTTP Request Node to Create Avatar Video (HeyGen)**
   - Connect Setup Heygen Parameters → HTTP Request
   - Method: POST
   - URL: `https://api.heygen.com/v2/video/generate`
   - Body (JSON): video_inputs with avatar, voice, input_text as caption, voice_id, speed=1.1, dimension 1280x720
   - Headers: Add `X-Api-Key` with HeyGen API key
   - Content-Type: application/json

10. **Add Wait Node**
    - Connect Create Avatar Video → Wait Node
    - Wait 2 minutes (adjust as needed for video processing time)

11. **Add HTTP Request Node to Get Avatar Video Status**
    - Connect Wait → HTTP Request
    - Method: GET
    - URL: `https://api.heygen.com/v1/video_status.get`
    - Query param: video_id from previous HeyGen response
    - Header: `X-Api-Key` with HeyGen API key

12. **Add HTTP Request Node to Download Video**
    - Connect Get Avatar Video Status → HTTP Request
    - URL: video_url from HeyGen status response
    - Method: GET
    - Download binary video content

13. **Add Google Drive Node to Upload Video**
    - Connect Download Video → Google Drive
    - Drive: "My Drive"
    - Folder: root (or desired folder)
    - Upload binary video file
    - Credential: Google Drive account with upload permission

14. **Add HTTP Request Node to Upload Video to Postiz**
    - Connect Download Video → HTTP Request
    - Method: POST
    - URL: `https://postiz.yourdomain.com/api/public/v1/upload`
    - Content-Type: multipart/form-data
    - Body Parameter: file (binary data from download)
    - Authentication: HTTP Header Auth with Postiz API key credential

15. **Add Google Sheets Node to Log Video Details**
    - Connect Google Drive → Google Sheets
    - Operation: Append
    - Spreadsheet ID and Sheet1 for "Avatar video" records
    - Map columns: Title, video caption, HeyGen video URL, Google Drive File ID

16. **Add HTTP Request Node to Get Postiz Integrations**
    - Connect Upload Video to Postiz → HTTP Request
    - Method: GET
    - URL: `https://postiz.yourdomain.com/api/public/v1/integrations`
    - Authentication: Postiz HTTP Header Auth

17. **Add Switch Node for Video Platform Router**
    - Connect Get Postiz Integrations → Switch
    - Route by `identifier` field:
      - instagram
      - facebook
      - youtube

18. **Add Code Node to Clean Instagram Caption**
    - Connect Switch (instagram) → Code Node
    - Logic:
      - Replace newlines, tabs with spaces
      - Replace multiple spaces with single space
      - Escape single/double quotes and backslashes
      - Trim whitespace
      - Truncate to 2200 characters with ellipsis if needed

19. **Add HTTP Request Node for Instagram Video Publisher**
    - Connect Clean Instagram Caption → HTTP Request
    - Method: POST
    - URL: `https://postiz.yourdomain.com/api/public/v1/posts`
    - Body JSON: includes cleaned caption, video path from Postiz upload, tag "instagram", immediate posting
    - Auth: Postiz HTTP Header Auth

20. **Add Code Node to Clean Facebook Caption**
    - Connect Switch (facebook) → Code Node
    - Similar cleaning as Instagram, but limit 63206 characters

21. **Add HTTP Request Node for Facebook Video Publisher**
    - Connect Clean Facebook Caption → HTTP Request
    - Method: POST
    - Similar body as Instagram but adjusted for Facebook

22. **Add HTTP Request Node for YouTube Video Publisher**
    - Connect Switch (youtube) → HTTP Request
    - Method: POST
    - Body JSON includes:
      - Title: news_title
      - Description: original AI caption
      - Tags: news, AI
      - Category ID: 22 (People & Blogs)
      - Visibility: public
      - Video path from Postiz upload
    - Auth: Postiz HTTP Header Auth

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow uses GPT-4o-mini via LangChain nodes for AI caption generation, offering concise and engaging text optimized for social media.                                                                                                                                                                                                                                                     | AI Captioning Block                                 |
| HeyGen API requires avatar and voice IDs, which must be configured in the 'Setup Heygen Parameters' node. Replace placeholders with your own credentials.                                                                                                                                                                                                                                         | HeyGen Video Generation                             |
| Postiz platform is used as a unified publishing API for Instagram, Facebook, and YouTube. Ensure HTTP Header Auth credentials are configured with your API keys.                                                                                                                                                                                                                                  | Social Media Publishing                             |
| Google Sheets are used for logging news and video metadata. Use separate sheets to avoid data clutter and to facilitate record keeping for analytics.                                                                                                                                                                                                                                           | Data Logging                                        |
| The workflow includes deliberate waiting and video status polling to accommodate HeyGen's asynchronous video generation process. Adjust waiting time based on video length and HeyGen's response times.                                                                                                                                                                                          | Video Generation Timing                             |
| Caption cleaning nodes escape problematic characters and enforce platform-specific character limits to prevent JSON parsing errors and API rejections.                                                                                                                                                                                                                                          | Caption Cleaning & Limits                           |
| Postiz API endpoint URLs use a placeholder domain (`postiz.yourdomain.com`), which must be replaced with your actual Postiz instance URL throughout the workflow.                                                                                                                                                                                                                                | Postiz API Configuration                            |
| For testing and development, the Limit node restricts processing to a single news item, conserving API credits and accelerating runs. Remove or increase this limit for production use.                                                                                                                                                                                                           | Development Control                                |
| Workflow trigger is manual for controlled execution; consider scheduling with a Cron node for automated periodic runs.                                                                                                                                                                                                                                                                            | Triggering                                         |
| For detailed HeyGen API documentation and credential management, refer to https://heygen.com/docs/api                                                                                                                                                                                                                                                                                            | HeyGen API Documentation                            |
| Postiz API documentation and credential setup details can be found at your Postiz instance or official Postiz docs.                                                                                                                                                                                                                                                                               | Postiz API Documentation                            |

---

This comprehensive reference document enables developers and automation agents to fully understand, replicate, and modify the "Create & Publish AI News Videos to Social Media with GPT-4o & HeyGen" workflow without requiring access to the original JSON. It highlights key configurations, potential error points, and integration specifics for reliable operation.