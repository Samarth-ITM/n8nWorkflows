Generate News-based AI Videos with HeyGen and Publish to Instagram & TikTok

https://n8nworkflows.xyz/workflows/generate-news-based-ai-videos-with-heygen-and-publish-to-instagram---tiktok-6084


# Generate News-based AI Videos with HeyGen and Publish to Instagram & TikTok

---

## 1. Workflow Overview

This workflow automates the process of generating AI-driven short news videos and publishing them on social media platforms like Instagram and TikTok. It uses AI tools to find trending news, generate video scripts, create engaging captions, produce AI avatar videos with dynamic background content, and finally publish these videos to social media via an intermediary service.

The workflow is logically divided into the following functional blocks:

- **1.1 Scheduled News Discovery & Script Generation:** Automatically triggers daily to find trending news articles, then generates a detailed video script.
- **1.2 URL Extraction & Screenshot Video Creation:** Extracts the news article URL from AI responses and generates a short scrolling video screenshot of the article.
- **1.3 Video Assembly via HeyGen AI Avatar:** Combines the video script, caption, and background screenshot video to create an AI talking avatar video.
- **1.4 Video Upload, Tracking, and Status Management:** Uploads generated videos to cloud storage and manages metadata, including status updates in Google Sheets.
- **1.5 Social Media Publishing:** Publishes the completed videos with captions to Instagram and TikTok through the Blotato API.
- **1.6 HeyGen Webhook Callback Processing:** Receives asynchronous status updates from HeyGen about video generation success or failure and triggers subsequent social media publishing steps.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduled News Discovery & Script Generation

**Overview:**  
This block runs daily at 6 AM to find trending news articles using Perplexity AI, then generates a concise, engaging 30-second video script based on the article via OpenAI GPT-4.

**Nodes Involved:**  
- Schedule Trigger  
- Perplexity Search  
- Write Video Script  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow daily at 6:00 AM.  
  - Config: Triggers once daily at hour 6 (6 AM).  
  - Inputs: None  
  - Outputs: Perplexity Search  
  - Edge Cases: Missed trigger if n8n server is down; timezone considerations.  

- **Perplexity Search**  
  - Type: Perplexity AI node  
  - Role: Searches for trending news articles emphasizing controversial or surprising viral content.  
  - Config: Uses the 'sonar-reasoning' model with a prompt requesting exact article URLs in a strict format.  
  - Key Expression: System message with instructions to include article URL in “Article URL: [url]” format.  
  - Inputs: Schedule Trigger output  
  - Outputs: Write Video Script  
  - Edge Cases: API rate limits, no relevant news found, malformed responses.  

- **Write Video Script**  
  - Type: OpenAI (Chat Completion)  
  - Role: Generates a 30-second video script in JSON format with hook, main point, CTA, and source URL fields.  
  - Config: Uses GPT-4.1 with system prompt defining style, tone, and output format.  
  - Key Expression: Input prompt includes Perplexity Search content dynamically.  
  - Inputs: Perplexity Search output  
  - Outputs: Extract URL  
  - Edge Cases: API errors, invalid JSON output, missing URL in response.  

---

### 2.2 URL Extraction & Screenshot Video Creation

**Overview:**  
Extracts the news article URL from AI-generated text and creates a short, scrolling video screenshot of the article on a mobile viewport.

**Nodes Involved:**  
- Extract URL  
- Get Screenshot  
- Upload to tmpfiles  
- Extract tmpfiles  

**Node Details:**

- **Extract URL**  
  - Type: Code node  
  - Role: Parses Perplexity and script outputs to extract the article URL robustly and parses script JSON.  
  - Configuration: Uses regex patterns to find URLs; fallback to default URL if none found. Cleans trailing punctuation. Parses script JSON from OpenAI output.  
  - Inputs: Write Video Script output  
  - Outputs: Get Screenshot  
  - Edge Cases: Malformed AI responses; missing URL; JSON parsing errors.  

- **Get Screenshot**  
  - Type: HTTP Request  
  - Role: Calls ScreenshotOne API to generate a 20-second scrolling MP4 video of the article on an iPhone 15 Pro Max viewport.  
  - Config: Uses credentials for ScreenshotOne access key; parameters include scroll duration, blocking ads and trackers.  
  - Inputs: Extract URL output (uses encoded URL)  
  - Outputs: Upload to tmpfiles  
  - Edge Cases: API timeouts, invalid URL, video generation failures.  

- **Upload to tmpfiles**  
  - Type: HTTP Request  
  - Role: Uploads the ScreenshotOne video file to Cloudinary for reliable hosting.  
  - Config: Multipart form-data with preset upload configuration and Cloudinary credentials.  
  - Inputs: Get Screenshot output (binary video file)  
  - Outputs: Extract tmpfiles  
  - Edge Cases: Upload failures, credential issues, file size/timeouts.  

- **Extract tmpfiles**  
  - Type: Code node  
  - Role: Extracts and validates the Cloudinary upload response URL for downstream use.  
  - Config: Checks for secure_url or url field; throws error if missing.  
  - Inputs: Upload to tmpfiles output  
  - Outputs: Write Video Caption  
  - Edge Cases: Missing URL, invalid response format.  

---

### 2.3 Video Caption Creation & Avatar Data Preparation

**Overview:**  
Generates social media video captions from the script and prepares all data for the HeyGen avatar video creation.

**Nodes Involved:**  
- Write Video Caption  
- Prepare Avatar Data  
- Create Avatar Video (Optimized)  

**Node Details:**

- **Write Video Caption**  
  - Type: OpenAI Chat Completion  
  - Role: Writes engaging captions optimized for TikTok, YouTube Shorts, and Instagram Reels based on the video script.  
  - Config: GPT-4.1 with prompt specifying tone and character limits (150-200 chars).  
  - Inputs: Extract tmpfiles output (script data)  
  - Outputs: Prepare Avatar Data  
  - Edge Cases: API rate limits, irrelevant captions, missing script data.  

- **Prepare Avatar Data**  
  - Type: Code node  
  - Role: Aggregates caption, script, background video URL (Cloudinary link), and estimates video duration for HeyGen API.  
  - Config: Validates presence of background video URL and script text. Logs info for debugging.  
  - Inputs: Write Video Caption, Extract URL (script data), Extract tmpfiles (Cloudinary URL)  
  - Outputs: Create Avatar Video (Optimized)  
  - Edge Cases: Missing data, invalid URLs, non-public video URLs (warning).  

- **Create Avatar Video (Optimized)**  
  - Type: HTTP Request  
  - Role: Calls HeyGen API to generate an AI avatar video with talking photo, voice, and video background.  
  - Config: POST to HeyGen v2 video generation endpoint with API key header; JSON body includes character settings, voice, background video URL, dimensions, and webhook callback URL.  
  - Inputs: Prepare Avatar Data output  
  - Outputs: Prepare Sheets Data  
  - Edge Cases: API key/auth errors, network issues, invalid payload, callback URL misconfiguration.  

---

### 2.4 Video Metadata Storage & Status Tracking

**Overview:**  
Stores initial video creation metadata in Google Sheets and updates status as the video progresses.

**Nodes Involved:**  
- Prepare Sheets Data  
- Store to Google Sheets  

**Node Details:**

- **Prepare Sheets Data**  
  - Type: Code node  
  - Role: Extracts video ID from HeyGen response and prepares data object with caption, timestamp, and status='processing' for storage.  
  - Inputs: Create Avatar Video (Optimized) output, Write Video Caption output  
  - Outputs: Store to Google Sheets  
  - Edge Cases: Missing video ID, mismatched data, date formatting issues.  

- **Store to Google Sheets**  
  - Type: Google Sheets node  
  - Role: Appends or updates a row in the specified Google Sheet with video metadata (video_id, caption, timestamp, status).  
  - Config: Requires Google Sheets credentials; sheet must have headers: video_id, caption, timestamp, status.  
  - Inputs: Prepare Sheets Data output  
  - Outputs: None  
  - Edge Cases: API permission errors, sheet ID misconfiguration, rate limits.  

---

### 2.5 HeyGen Webhook Callback Processing & Social Media Publishing

**Overview:**  
Handles asynchronous callbacks from HeyGen about video generation status, retrieves captions from Google Sheets, uploads video to Blotato, and posts to Instagram and TikTok.

**Nodes Involved:**  
- HeyGen Webhook  
- Process HeyGen Callback  
- Get Caption from Sheets  
- Combine Data  
- Upload to Blotato  
- Extract Blotato URL  
- Assign Social Media IDs  
- INSTAGRAM  
- TIKTOK  

**Node Details:**

- **HeyGen Webhook**  
  - Type: Webhook (POST)  
  - Role: Receives HeyGen video generation status callbacks at path /heygen-webhook.  
  - Inputs: External HTTP POST from HeyGen  
  - Outputs: Process HeyGen Callback  
  - Edge Cases: Webhook security (not shown), malformed payloads, network downtime.  

- **Process HeyGen Callback**  
  - Type: Code node  
  - Role: Parses webhook JSON body, logs event, returns video data if success, throws error on failure or unknown event type.  
  - Inputs: HeyGen Webhook output  
  - Outputs: Get Caption from Sheets  
  - Edge Cases: Unexpected event types, missing fields, error handling.  

- **Get Caption from Sheets**  
  - Type: Google Sheets node  
  - Role: Retrieves the stored caption by looking up video_id from Google Sheets.  
  - Config: Filters by video_id column equal to callback video_id.  
  - Inputs: Process HeyGen Callback output  
  - Outputs: Combine Data  
  - Edge Cases: Not found entries, permission errors.  

- **Combine Data**  
  - Type: Code node  
  - Role: Merges video URLs from HeyGen with caption from Sheets, marks video as ready for upload.  
  - Inputs: Get Caption from Sheets output, Process HeyGen Callback output  
  - Outputs: Upload to Blotato  
  - Edge Cases: Missing caption or video URL.  

- **Upload to Blotato**  
  - Type: HTTP Request  
  - Role: Uploads video URL to Blotato API for hosting and further social media integration.  
  - Config: POST with Blotato API key header and JSON body containing video URL.  
  - Inputs: Combine Data output  
  - Outputs: Extract Blotato URL  
  - Edge Cases: API rate limits, invalid URL, authentication errors.  

- **Extract Blotato URL**  
  - Type: Code node  
  - Role: Extracts the hosted video URL from Blotato response, merges it with existing data.  
  - Inputs: Upload to Blotato output, Combine Data output  
  - Outputs: Assign Social Media IDs  
  - Edge Cases: Missing URL in response.  

- **Assign Social Media IDs**  
  - Type: Set node  
  - Role: Sets Instagram and TikTok account IDs statically for downstream publishing.  
  - Config: Hardcoded account IDs to be replaced with actual ones.  
  - Inputs: Extract Blotato URL output  
  - Outputs: INSTAGRAM, TIKTOK  
  - Edge Cases: Missing or incorrect account IDs will cause failure downstream.  

- **INSTAGRAM**  
  - Type: HTTP Request  
  - Role: Posts the video with caption to Instagram via Blotato backend API.  
  - Config: POST request with Blotato API key and JSON body containing account ID, target platform, caption text, and media URL.  
  - Inputs: Assign Social Media IDs output  
  - Outputs: None  
  - Edge Cases: API errors, invalid credentials, content policy violations.  

- **TIKTOK**  
  - Type: HTTP Request  
  - Role: Posts the video with caption to TikTok via Blotato backend API with additional parameters for privacy and AI-generated flags.  
  - Config: Similar to Instagram node but with TikTok-specific platform flags.  
  - Inputs: Assign Social Media IDs output  
  - Outputs: None  
  - Edge Cases: Same as Instagram node.  

---

## 3. Summary Table

| Node Name                | Node Type                 | Functional Role                                  | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                          |
|--------------------------|---------------------------|-------------------------------------------------|----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger          | Starts workflow daily at 6 AM                    | None                       | Perplexity Search            |                                                                                                    |
| Perplexity Search        | Perplexity AI             | Finds trending news articles                      | Schedule Trigger            | Write Video Script           |                                                                                                    |
| Write Video Script       | OpenAI Chat Completion    | Generates 30-second video script in JSON format  | Perplexity Search           | Extract URL                 |                                                                                                    |
| Extract URL              | Code                      | Extracts article URL and parses script JSON       | Write Video Script          | Get Screenshot              |                                                                                                    |
| Get Screenshot           | HTTP Request              | Generates scrolling video screenshot of article  | Extract URL                 | Upload to tmpfiles          |                                                                                                    |
| Upload to tmpfiles       | HTTP Request              | Uploads video file to Cloudinary                   | Get Screenshot              | Extract tmpfiles            |                                                                                                    |
| Extract tmpfiles         | Code                      | Extracts Cloudinary video URL                      | Upload to tmpfiles          | Write Video Caption         |                                                                                                    |
| Write Video Caption      | OpenAI Chat Completion    | Creates engaging social media caption              | Extract tmpfiles            | Prepare Avatar Data         |                                                                                                    |
| Prepare Avatar Data      | Code                      | Prepares and validates data for HeyGen video      | Write Video Caption, Extract URL, Extract tmpfiles | Create Avatar Video (Optimized) |                                                                                                    |
| Create Avatar Video (Optimized) | HTTP Request              | Calls HeyGen API to generate AI avatar video      | Prepare Avatar Data         | Prepare Sheets Data         |                                                                                                    |
| Prepare Sheets Data      | Code                      | Prepares metadata for Google Sheets                | Create Avatar Video (Optimized), Write Video Caption | Store to Google Sheets      |                                                                                                    |
| Store to Google Sheets   | Google Sheets             | Stores video metadata with status                  | Prepare Sheets Data         | None                       | "You need to:\n1. Create a Google Sheet\n2. Add headers: video_id, caption, timestamp, status\n3. Replace YOUR_GOOGLE_SHEET_ID with actual ID\n4. Connect your Google account" |
| HeyGen Webhook           | Webhook                   | Receives HeyGen video generation callbacks         | External HTTP POST          | Process HeyGen Callback     |                                                                                                    |
| Process HeyGen Callback  | Code                      | Parses callback and returns video status           | HeyGen Webhook              | Get Caption from Sheets     |                                                                                                    |
| Get Caption from Sheets  | Google Sheets             | Retrieves caption by video_id                        | Process HeyGen Callback     | Combine Data               | "This looks up the video_id in Google Sheets to get the caption"                                   |
| Combine Data            | Code                      | Combines video URLs and captions                     | Get Caption from Sheets, Process HeyGen Callback | Upload to Blotato           |                                                                                                    |
| Upload to Blotato       | HTTP Request              | Uploads video URL to Blotato                         | Combine Data                | Extract Blotato URL         |                                                                                                    |
| Extract Blotato URL     | Code                      | Extracts hosted video URL from Blotato response     | Upload to Blotato           | Assign Social Media IDs     |                                                                                                    |
| Assign Social Media IDs | Set                       | Sets Instagram and TikTok account IDs               | Extract Blotato URL         | INSTAGRAM, TIKTOK          |                                                                                                    |
| INSTAGRAM               | HTTP Request              | Posts video with caption to Instagram via Blotato  | Assign Social Media IDs     | None                       |                                                                                                    |
| TIKTOK                  | HTTP Request              | Posts video with caption to TikTok via Blotato     | Assign Social Media IDs     | None                       |                                                                                                    |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 6:00 AM (hour 6).  

2. **Add Perplexity Search Node**  
   - Type: Perplexity AI  
   - Connect input from Schedule Trigger.  
   - Use the 'sonar-reasoning' model.  
   - Set prompt to find trending news with instructions to include exact article URL in format: `Article URL: [URL]`.  

3. **Add Write Video Script Node (OpenAI Chat Completion)**  
   - Connect input from Perplexity Search.  
   - Use GPT-4.1 model with system prompt specifying a 30-second video script with fields: script, sourceURL, hook, mainPoint, cta (JSON output).  
   - Set user message dynamically to include Perplexity Search content.  

4. **Add Extract URL Code Node**  
   - Connect input from Write Video Script.  
   - Implement JavaScript code that extracts the article URL using regex patterns from Perplexity response, cleans URLs, and parses the script JSON from the AI output.  
   - Output extracted URL, encoded URL, script data, and perplexity raw content.  

5. **Add Get Screenshot HTTP Request Node**  
   - Connect input from Extract URL.  
   - Configure to call ScreenshotOne API with parameters: access key from credentials, URL as encodedURL, scrolling scenario, iPhone 15 Pro Max viewport, MP4 format, 20 seconds duration.  
   - Set timeout to 60 seconds and expect file response.  

6. **Add Upload to tmpfiles HTTP Request Node**  
   - Connect input from Get Screenshot (binary video file).  
   - Configure POST to Cloudinary video upload endpoint with multipart form-data: file (from binary data), upload_preset.  
   - Replace `YOUR_CLOUD_NAME` and `YOUR_UPLOAD_PRESET` with your Cloudinary settings.  

7. **Add Extract tmpfiles Code Node**  
   - Connect input from Upload to tmpfiles.  
   - Implement code to extract secure_url or url from Cloudinary response; throw error if missing.  

8. **Add Write Video Caption Node (OpenAI Chat Completion)**  
   - Connect input from Extract tmpfiles.  
   - Use GPT-4.1 with prompt to generate a 150-200 character engaging caption based on the script text.  
   - Temperature set to 0.7 for conversational tone.  

9. **Add Prepare Avatar Data Code Node**  
   - Connect input from Write Video Caption, Extract URL (for script data), and Extract tmpfiles (for background video URL).  
   - Code should aggregate caption, script, encoded URL, background video URL, and estimate video duration; validate required data presence.  

10. **Add Create Avatar Video (Optimized) HTTP Request Node**  
    - Connect input from Prepare Avatar Data.  
    - POST to HeyGen API endpoint `/v2/video/generate` with JSON body including:  
      - Talking photo ID (replace `YOUR_TALKING_PHOTO_ID`), voice ID (`YOUR_VOICE_ID`), background video URL, video dimension 720x1280, callback URL (`YOUR_WEBHOOK_URL`).  
    - Set headers with HeyGen API key from credentials and Content-Type application/json.  
    - Timeout set to 45s.  

11. **Add Prepare Sheets Data Code Node**  
    - Connect input from Create Avatar Video (Optimized) and Write Video Caption.  
    - Extract video_id from HeyGen response, prepare JSON with video_id, caption, current timestamp, and status 'processing'.  

12. **Add Store to Google Sheets Node**  
    - Connect input from Prepare Sheets Data.  
    - Configure to append or update rows in your Google Sheet (sheet name "Sheet1") with columns: video_id, caption, timestamp, status.  
    - Replace `YOUR_GOOGLE_SHEET_ID` with actual ID and connect Google credentials.  

13. **Add HeyGen Webhook Node**  
    - Type: Webhook (POST)  
    - Path: `heygen-webhook`  
    - This node awaits callbacks from HeyGen video generation API.  

14. **Add Process HeyGen Callback Code Node**  
    - Connect input from HeyGen Webhook.  
    - Parse webhook JSON body, log event type, check for `avatar_video.success` or `avatar_video.fail`.  
    - Return video data on success with status 'completed' and flag ready_for_upload.  
    - Throw error on failure or unknown event.  

15. **Add Get Caption from Sheets Node**  
    - Connect input from Process HeyGen Callback.  
    - Lookup Google Sheet by video_id to retrieve caption.  

16. **Add Combine Data Code Node**  
    - Connect input from Get Caption from Sheets and Process HeyGen Callback.  
    - Merge video URLs and captions into one JSON object with status 'completed' and ready_for_upload flag.  

17. **Add Upload to Blotato HTTP Request Node**  
    - Connect input from Combine Data.  
    - POST to Blotato `/v2/media` API with video URL to upload video for social publishing.  
    - Use Blotato API key credentials.  

18. **Add Extract Blotato URL Code Node**  
    - Connect input from Upload to Blotato and Combine Data.  
    - Extract hosted Blotato video URL from response and merge with existing data.  

19. **Add Assign Social Media IDs Set Node**  
    - Connect input from Extract Blotato URL.  
    - Set static JSON with your Instagram and TikTok account IDs (replace placeholders).  

20. **Add INSTAGRAM HTTP Request Node**  
    - Connect input from Assign Social Media IDs.  
    - POST to Blotato `/v2/posts` with Instagram account ID, caption, and media URL.  
    - Include Blotato API key header.  

21. **Add TIKTOK HTTP Request Node**  
    - Connect input from Assign Social Media IDs.  
    - POST to Blotato `/v2/posts` with TikTok account ID and similar parameters, including flags for AI-generated content and privacy.  
    - Include Blotato API key header.  

---

## 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Please replace placeholder values such as `YOUR_TALKING_PHOTO_ID`, `YOUR_VOICE_ID`, `YOUR_WEBHOOK_URL`, `YOUR_CLOUD_NAME`, `YOUR_UPLOAD_PRESET`, `YOUR_GOOGLE_SHEET_ID`, and social media account IDs with your actual credentials and IDs. | Configuration details for HeyGen, Cloudinary, Google Sheets, and social media publishing APIs.  |
| Google Sheet for status tracking must have headers: video_id, caption, timestamp, status.                                          | Essential setup for metadata management.                                                       |
| HeyGen Webhook security is not detailed; ensure webhook endpoint is secured appropriately.                                         | Security best practice for webhook endpoints.                                                  |
| Blotato API is used for social media content upload and posting. Visit https://blotato.com for API documentation and accounts.   | External API service for managing social media posts.                                         |
| Video dimension is set to 720x1280 for vertical mobile video formats suitable for TikTok and Instagram Reels.                     | Video format recommendation.                                                                   |
| ScreenshotOne API is used for dynamic scrolling video screenshots of news articles.                                                | ScreenshotOne API docs: https://screenshotone.com/docs                                          |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---