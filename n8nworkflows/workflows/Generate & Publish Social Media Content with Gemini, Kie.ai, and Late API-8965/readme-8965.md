Generate & Publish Social Media Content with Gemini, Kie.ai, and Late API

https://n8nworkflows.xyz/workflows/generate---publish-social-media-content-with-gemini--kie-ai--and-late-api-8965


# Generate & Publish Social Media Content with Gemini, Kie.ai, and Late API

### 1. Workflow Overview

This workflow automates the generation and publishing of social media content by leveraging AI-powered text and image generation services, and then publishing posts via the Late API. It is designed for businesses or marketers who want to create customized, platform-optimized social media posts automatically and publish them on multiple platforms such as Facebook, Instagram, TikTok, and LinkedIn.

The workflow is logically divided into these blocks:

- **1.1 Initialization & Settings:** Sets default variables for business type, content topic, enabled platforms, and account IDs.
- **1.2 Platform Enablement Check:** Determines which platforms are enabled for posting.
- **1.3 AI Content Generation:** Uses Google Gemini API to generate platform-specific post text.
- **1.4 Content Parsing & Filtering:** Parses Gemini's JSON output and filters content for enabled platforms.
- **1.5 AI Image Generation:** Uses Kie.ai’s Seedream v4 model to create a branded social media image.
- **1.6 Image Retrieval & Preparation:** Polls Kie.ai API to obtain the generated image URL and prepares metadata.
- **1.7 Media File Size Handling & Upload:** Checks image size to choose upload method; uploads to Late API either directly or via multipart flow for large files.
- **1.8 Post Creation per Platform:** Conditional branches publish posts to enabled social platforms via Late API.
- **1.9 Result Merging & Notifications:** Merges results from all platforms, composes structured success notifications for Slack, Discord, Email, and Webhook, and returns the final response.
- **1.10 Manual Trigger Entry Point:** Starts the workflow manually for testing or scheduled runs.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Settings

- **Overview:** Defines the workflow’s initial variables such as business type, content topic, enabled platforms toggles, and social media account IDs.
- **Nodes Involved:**
  - Default Settings
- **Node Details:**
  - **Default Settings (Set Node)**
    - Assigns string variables for BUSINESS_TYPE, CONTENT_TOPIC, and platform toggles (ENABLE_FACEBOOK, ENABLE_INSTAGRAM, ENABLE_LINKEDIN, ENABLE_TIKTOK).
    - Includes account ID variables for each platform.
    - Input: Manual Trigger node.
    - Output: Check Enabled Platforms.
    - Edge cases: Missing or incorrect account IDs can cause post failures downstream.
    - Version: 3.4

---

#### 2.2 Platform Enablement Check

- **Overview:** Checks which social media platforms are enabled to determine whether to proceed with content generation.
- **Nodes Involved:**
  - Check Enabled Platforms
- **Node Details:**
  - **Check Enabled Platforms (If Node)**
    - Combines multiple OR conditions to check if any ENABLE_* variable equals "true".
    - Input: Default Settings.
    - Output: Gemini - Generate Content (if true).
    - Edge cases: If all platform toggles are false or missing, workflow halts here.
    - Version: 2.1

---

#### 2.3 AI Content Generation

- **Overview:** Calls Google Gemini API to generate social media text content tailored per platform.
- **Nodes Involved:**
  - Gemini - Generate Content
- **Node Details:**
  - **Gemini - Generate Content (HTTP Request Node)**
    - POST request to Google Gemini v1beta API (model gemini-2.5-flash).
    - Request body includes a prompt instructing content creation optimized for Facebook, Instagram, TikTok, LinkedIn with character limits and style.
    - Uses OAuth2 credentials for authentication via Google Palm API credentials.
    - Input: Check Enabled Platforms.
    - Output: Parse & Filter Platforms.
    - Edge cases: API rate limits, auth failures, malformed responses.
    - Version: 4.2

---

#### 2.4 Content Parsing & Filtering

- **Overview:** Parses the JSON text returned by Gemini and filters content for only enabled platforms.
- **Nodes Involved:**
  - Parse & Filter Platforms
- **Node Details:**
  - **Parse & Filter Platforms (Code Node)**
    - Extracts text from Gemini API response, parses JSON.
    - Provides fallback content if parsing fails.
    - Builds an array of enabled platforms with their content.
    - Reads environment variables to determine enabled platforms.
    - Input: Gemini - Generate Content.
    - Output: Kie.ai - Generate Image.
    - Edge cases: JSON parse errors, missing keys, empty content.
    - Version: 2

---

#### 2.5 AI Image Generation

- **Overview:** Requests Kie.ai’s Seedream v4 text-to-image model to generate a social media image based on the content topic.
- **Nodes Involved:**
  - Kie.ai - Generate Image
  - Wait 30s
  - Kie.ai - Obtain Image
- **Node Details:**
  - **Kie.ai - Generate Image (HTTP Request Node)**
    - POST to Kie.ai API creating a task with prompt describing image style and topic.
    - Uses Bearer token for Kie.ai authentication.
    - Input: Parse & Filter Platforms.
    - Output: Wait 30s.
    - Edge cases: API limits, auth errors, invalid prompt.
    - Version: 4.2

  - **Wait 30s (Wait Node)**
    - Pauses workflow for 30 seconds to allow Kie.ai task processing.
    - Input: Kie.ai - Generate Image.
    - Output: Kie.ai - Obtain Image.
    - Version: 1.1

  - **Kie.ai - Obtain Image (HTTP Request Node)**
    - GET request polling Kie.ai for task results using taskId.
    - Retrieves image URL from response.
    - Input: Wait 30s.
    - Output: Prepare data.
    - Edge cases: Timeout if image not ready, API error.
    - Version: 4.2

---

#### 2.6 Image Retrieval & Preparation

- **Overview:** Extracts image URL and metadata, determines file type, and prepares for upload.
- **Nodes Involved:**
  - Prepare data
- **Node Details:**
  - **Prepare data (Code Node)**
    - Parses Kie.ai response to extract first image URL.
    - Determines filename, extension, and MIME content type.
    - Input: Kie.ai - Obtain Image.
    - Output: HTTP Request for content-length.
    - Edge cases: Missing or malformed URLs, unsupported file extensions.
    - Version: 2

---

#### 2.7 Media File Size Handling & Upload

- **Overview:** Checks image file size and uploads it to Late API using different flows depending on size.
- **Nodes Involved:**
  - HTTP Request for content-length
  - IF – „Is Large?”
  - LATE – Init Upload files > 4MB
  - Download Large (GET)
  - Upload to Storage (PUT)
  - HTTP Request for files ≤ ~4 MB
  - Late - Upload Media files <=4MB
- **Node Details:**

  - **HTTP Request for content-length (HTTP Request Node)**
    - Requests the image URL with full response headers to read content-length.
    - Input: Prepare data.
    - Output: IF – „Is Large?”.
    - Edge cases: Missing content-length header, network errors.
    - Version: 4.2

  - **IF – „Is Large?” (If Node)**
    - Checks if content-length > 4MB (4194304 bytes).
    - Input: HTTP Request for content-length.
    - Output: Two branches:
      - True: LATE – Init Upload files > 4MB.
      - False: HTTP Request for files ≤ ~4 MB.
    - Version: 2.2

  - **LATE – Init Upload files > 4MB (HTTP Request Node)**
    - Initiates multipart upload at Late API for large files.
    - Sends filename, content type, and multipart flag.
    - Input: IF – „Is Large?” (true branch).
    - Output: Download Large (GET).
    - Edge cases: API errors, incorrect headers.
    - Version: 4.2

  - **Download Large (GET) (HTTP Request Node)**
    - Downloads large image file as binary data.
    - Input: LATE – Init Upload files > 4MB.
    - Output: Upload to Storage (PUT).
    - Edge cases: Network timeouts, large file memory handling.
    - Version: 4.2

  - **Upload to Storage (PUT) (HTTP Request Node)**
    - Uploads binary image data to storage URL provided by Late API.
    - Uses PUT method with content-type header.
    - Input: Download Large (GET).
    - Output: Normalize Media Output.
    - Edge cases: Upload failures, incorrect URLs.
    - Version: 4.2

  - **HTTP Request for files ≤ ~4 MB (HTTP Request Node)**
    - Downloads small image file as binary data.
    - Input: IF – „Is Large?” (false branch).
    - Output: Late - Upload Media files <=4MB.
    - Version: 4.2

  - **Late - Upload Media files <=4MB (HTTP Request Node)**
    - Uploads small image file directly as multipart form-data to Late API.
    - Input: HTTP Request for files ≤ ~4 MB.
    - Output: Normalize Media Output.
    - Edge cases: Multipart upload errors.
    - Version: 4.2

---

#### 2.8 Post Creation per Platform

- **Overview:** Combines content with media URLs and posts to each enabled platform via Late API.
- **Nodes Involved:**
  - Normalize Media Output
  - Combine Content with Media
  - IF Facebook
  - IF Instagram
  - IF LinkedIn
  - IF TikTok
  - Late Create Post Facebook
  - Late Create Post Instagram
  - Late Create Post Linkedin
  - Late Create Post TikTok
  - Merge
- **Node Details:**

  - **Normalize Media Output (Code Node)**
    - Normalizes media API response to extract a single public image URL.
    - Input: Upload to Storage (PUT) or Late - Upload Media files <=4MB.
    - Output: Combine Content with Media.
    - Version: 2

  - **Combine Content with Media (Code Node)**
    - Merges the text content and media URL into a single JSON object for posting.
    - Input: Normalize Media Output and Parse & Filter Platforms.
    - Output: IF Facebook, IF Instagram, IF LinkedIn, IF TikTok.
    - Version: 2

  - **IF Facebook / IF Instagram / IF LinkedIn / IF TikTok (If Nodes)**
    - Checks if each platform is enabled.
    - Input: Combine Content with Media.
    - Output: Corresponding Late Create Post node if true.
    - Version: 2.2

  - **Late Create Post Facebook / Instagram / LinkedIn / TikTok (HTTP Request Nodes)**
    - POST requests to Late API /api/v1/posts endpoint.
    - Payload includes content text plus hashtags, publishNow flag, target platform and accountId, and mediaItems with image URL.
    - Uses Bearer auth for Late API.
    - Input: IF Facebook/Instagram/LinkedIn/TikTok.
    - Output: Merge.
    - Edge cases: Missing accountId, auth errors, invalid media URLs.
    - Version: 4.2

  - **Merge (Merge Node)**
    - Merges the outputs of all platform post nodes into a single stream.
    - Input: All Late Create Post nodes.
    - Output: Success Notification & Logging.
    - Version: 3.2

---

#### 2.9 Result Merging & Notifications

- **Overview:** Consolidates all post results and generates structured notifications for downstream systems.
- **Nodes Involved:**
  - Success Notification & Logging
  - Respond to Webhook
- **Node Details:**
  - **Success Notification & Logging (Code Node)**
    - Collects all posts, formats platform names, IDs, and previews.
    - Builds messages for Slack, Discord, Email, and webhook notifications.
    - Returns a consolidated JSON with posts and notification details.
    - Input: Merge.
    - Output: Respond to Webhook.
    - Edge cases: Missing post details.
    - Version: 2

  - **Respond to Webhook (Respond to Webhook Node)**
    - Sends final JSON response back to the calling client or system.
    - Input: Success Notification & Logging.
    - Version: 1.4

---

#### 2.10 Manual Trigger Entry Point

- **Overview:** Allows manual execution of the workflow for testing or on-demand runs.
- **Nodes Involved:**
  - When clicking ‘Execute workflow’
- **Node Details:**
  - **When clicking ‘Execute workflow’ (Manual Trigger Node)**
    - Manual start point.
    - Output: Default Settings.
    - Version: 1

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                     | Input Node(s)                        | Output Node(s)                        | Sticky Note                                                                                                          |
|-------------------------------|------------------------|-----------------------------------|------------------------------------|-------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger          | Entry point                       | -                                  | Default Settings                    |                                                                                                                     |
| Default Settings               | Set                    | Define variables & credentials    | When clicking ‘Execute workflow’    | Check Enabled Platforms             | Defines default workflow variables and toggles (BUSINESS_TYPE, CONTENT_TOPIC, ENABLE_*, *_ACCOUNT_ID).               |
| Check Enabled Platforms        | If                     | Check enabled social platforms    | Default Settings                   | Gemini - Generate Content           | Checks if any platform is enabled to proceed with generation.                                                       |
| Gemini - Generate Content      | HTTP Request           | Generate social media text via Gemini API | Check Enabled Platforms            | Parse & Filter Platforms            | Calls Google Gemini to create platform-specific post content.                                                       |
| Parse & Filter Platforms       | Code                   | Parse Gemini output, filter platforms | Gemini - Generate Content           | Kie.ai - Generate Image             | Parses JSON from Gemini response, applies platform filters.                                                         |
| Kie.ai - Generate Image        | HTTP Request           | Request image generation from Kie.ai | Parse & Filter Platforms            | Wait 30s                           | Creates Seedream image generation task with prompt.                                                                 |
| Wait 30s                      | Wait                   | Pause for image generation        | Kie.ai - Generate Image             | Kie.ai - Obtain Image               | Waits 30 seconds for Kie.ai image generation completion.                                                             |
| Kie.ai - Obtain Image          | HTTP Request           | Poll Kie.ai for generated image   | Wait 30s                          | Prepare data                       | Retrieves generated image URL from Kie.ai.                                                                           |
| Prepare data                  | Code                   | Extract image URL and metadata    | Kie.ai - Obtain Image              | HTTP Request for content-length    | Determines filename, extension, and content type.                                                                    |
| HTTP Request for content-length | HTTP Request           | Get image file size via headers   | Prepare data                      | IF – „Is Large?”                   | Retrieves content-length header to decide upload method.                                                            |
| IF – „Is Large?”              | If                     | Branch by file size > 4MB         | HTTP Request for content-length    | LATE – Init Upload files > 4MB / HTTP Request for files ≤ ~4 MB | Branches upload logic based on image size threshold.                                                                |
| LATE – Init Upload files > 4MB | HTTP Request           | Initiate multipart upload for large files | IF – „Is Large?” (true branch)     | Download Large (GET)               | Starts multipart upload session at Late API.                                                                         |
| Download Large (GET)          | HTTP Request           | Download large image file          | LATE – Init Upload files > 4MB     | Upload to Storage (PUT)            | Downloads the large image as binary.                                                                                  |
| Upload to Storage (PUT)       | HTTP Request           | Upload large image binary to storage | Download Large (GET)               | Normalize Media Output             | Completes multipart upload to Late's storage URL.                                                                    |
| HTTP Request for files ≤ ~4 MB | HTTP Request           | Download small image file          | IF – „Is Large?” (false branch)    | Late - Upload Media files <=4MB    | Downloads image file if ≤ 4MB.                                                                                        |
| Late - Upload Media files <=4MB | HTTP Request           | Upload small image directly       | HTTP Request for files ≤ ~4 MB      | Normalize Media Output             | Uploads image as multipart form-data to Late API.                                                                    |
| Normalize Media Output        | Code                   | Normalize media upload output     | Upload to Storage (PUT), Late - Upload Media files <=4MB | Combine Content with Media        | Extracts public URL for image from Late API response.                                                                |
| Combine Content with Media    | Code                   | Merge text content with media URL | Normalize Media Output, Parse & Filter Platforms | IF Facebook, IF Instagram, IF LinkedIn, IF TikTok | Combines platform texts and media URLs into a unified object.                                                        |
| IF Facebook                  | If                     | Check Facebook enabled             | Combine Content with Media          | Late Create Post Facebook          |                                                                                                                     |
| IF Instagram                 | If                     | Check Instagram enabled            | Combine Content with Media          | Late Create Post Instagram         |                                                                                                                     |
| IF LinkedIn                 | If                     | Check LinkedIn enabled             | Combine Content with Media          | Late Create Post Linkedin          |                                                                                                                     |
| IF TikTok                   | If                     | Check TikTok enabled               | Combine Content with Media          | Late Create Post TikTok            |                                                                                                                     |
| Late Create Post Facebook     | HTTP Request           | Publish post to Facebook via Late | IF Facebook                      | Merge                            | Posts content + hashtags + media to Facebook account on Late API.                                                   |
| Late Create Post Instagram    | HTTP Request           | Publish post to Instagram via Late | IF Instagram                     | Merge                            | Posts content + hashtags + media to Instagram account on Late API.                                                  |
| Late Create Post Linkedin     | HTTP Request           | Publish post to LinkedIn via Late  | IF LinkedIn                      | Merge                            | Posts content + hashtags + media to LinkedIn account on Late API.                                                   |
| Late Create Post TikTok       | HTTP Request           | Publish post to TikTok via Late    | IF TikTok                        | Merge                            | Posts content + hashtags + media to TikTok account on Late API.                                                     |
| Merge                        | Merge                  | Combine all platform post outputs | Late Create Post Facebook/Instagram/Linkedin/TikTok | Success Notification & Logging  | Aggregates all post responses.                                                                                       |
| Success Notification & Logging | Code                   | Compile notifications & logs      | Merge                            | Respond to Webhook                | Creates notifications for Slack, Discord, Email, Webhook with post summaries.                                        |
| Respond to Webhook           | Respond to Webhook      | Return final response             | Success Notification & Logging    | -                               | Sends final workflow output to caller.                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Name: "When clicking ‘Execute workflow’"
   - Type: Manual Trigger

2. **Create Set Node for Defaults**
   - Name: "Default Settings"
   - Create string variables for:
     - BUSINESS_TYPE (e.g., "YOUR_BUSINESS_TYPE")
     - CONTENT_TOPIC (e.g., "YOUR_CONTENT_TOPIC")
     - ENABLE_FACEBOOK (true/false as string)
     - FACEBOOK_ACCOUNT_ID (string)
     - ENABLE_INSTAGRAM (true/false)
     - INSTAGRAM_ACCOUNT_ID (string)
     - ENABLE_LINKEDIN (true/false)
     - LINKEDIN_ACCOUNT_ID (string)
     - ENABLE_TIKTOK (true/false)
     - TIKTOK_ACCOUNT_ID (string)
   - Connect Manual Trigger → Default Settings

3. **Create If Node to Check Enabled Platforms**
   - Name: "Check Enabled Platforms"
   - Condition: OR of ENABLE_FACEBOOK=="true", ENABLE_INSTAGRAM=="true", ENABLE_TIKTOK=="true", ENABLE_LINKEDIN=="true"
   - Connect Default Settings → Check Enabled Platforms

4. **Create HTTP Request Node for Gemini Content Generation**
   - Name: "Gemini - Generate Content"
   - Method: POST
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent`
   - Authentication: OAuth2 (Google Palm API with valid credentials)
   - Body (JSON):
     ```json
     {
       "contents": [
         {
           "parts": [
             {
               "text": "Create social media content for {{ $json.BUSINESS_TYPE }}. Generate content optimized for:\n- Facebook: Professional but friendly, 100-150 chars\n- Instagram: Visual focus, storytelling, include emojis, 125 chars\n- TikTok: Trendy, Gen-Z friendly, viral potential, 80 chars\n- LinkedIn: Professional, thought leadership, 120 chars\n\nTopic: {{ $json.CONTENT_TOPIC }}\n\nReturn as JSON with keys: topic, facebook, instagram, tiktok, linkedin, hashtags, bestTime"
             }
           ]
         }
       ],
       "generationConfig": {
         "temperature": 0.8,
         "maxOutputTokens": 1024,
         "responseMimeType": "application/json"
       }
     }
     ```
   - Connect Check Enabled Platforms (true) → Gemini - Generate Content

5. **Create Code Node to Parse Gemini Output and Filter Platforms**
   - Name: "Parse & Filter Platforms"
   - JavaScript code to parse Gemini response JSON and filter enabled platforms based on workflow variables (ENABLE_FACEBOOK, etc.)
   - Connect Gemini - Generate Content → Parse & Filter Platforms

6. **Create HTTP Request Node to Generate Image via Kie.ai**
   - Name: "Kie.ai - Generate Image"
   - Method: POST
   - URL: `https://api.kie.ai/api/v1/jobs/createTask`
   - Authentication: HTTP Bearer with Kie.ai API key
   - Headers: Content-Type: application/json
   - Body (JSON):
     ```json
     {
       "model": "bytedance/seedream-v4-text-to-image",
       "input": {
         "prompt": "Create a professional social media image about: {{ $json.topic }}. Modern tech company style, clean design, blue and white color scheme, suitable for Facebook, Instagram, LinkedIn. Square format 1080x1080.",
         "image_size": "square_hd",
         "image_resolution": "1K",
         "max_images": 1
       }
     }
     ```
   - Connect Parse & Filter Platforms → Kie.ai - Generate Image

7. **Add Wait Node (30 seconds)**
   - Name: "Wait 30s"
   - Amount: 30 seconds
   - Connect Kie.ai - Generate Image → Wait 30s

8. **Create HTTP Request Node to Poll Kie.ai for Image**
   - Name: "Kie.ai - Obtain Image"
   - Method: GET
   - URL: `https://api.kie.ai/api/v1/jobs/recordInfo?taskId={{ $json["data"]["taskId"] }}`
   - Authentication: HTTP Bearer with Kie.ai API key
   - Connect Wait 30s → Kie.ai - Obtain Image

9. **Create Code Node to Prepare Image Data**
   - Name: "Prepare data"
   - JavaScript code to extract image URL, filename, extension, and content type from Kie.ai response
   - Connect Kie.ai - Obtain Image → Prepare data

10. **Create HTTP Request Node to Get Image Content-Length**
    - Name: "HTTP Request for content-length"
    - Method: GET
    - URL: `={{ $json.imageUrl }}`
    - Options: Full response headers requested
    - Connect Prepare data → HTTP Request for content-length

11. **Create If Node to Check Image Size > 4MB**
    - Name: "IF – „Is Large?”"
    - Condition: content-length header > 4194304 (4 MB)
    - Connect HTTP Request for content-length → IF – „Is Large?”

12. **Create HTTP Request Node to Initiate Large File Upload to Late API**
    - Name: "LATE – Init Upload files > 4MB"
    - Method: POST
    - URL: `https://getlate.dev/api/v1/media`
    - Authentication: HTTP Bearer with Late API key
    - Body (JSON): filename, contentType, multipart: true
    - Connect IF – „Is Large?” (true) → LATE – Init Upload files > 4MB

13. **Create HTTP Request Node to Download Large Image**
    - Name: "Download Large (GET)"
    - Method: GET
    - URL: `={{ $('Prepare data').item.json.imageUrl }}`
    - Response Format: File (binary)
    - Connect LATE – Init Upload files > 4MB → Download Large (GET)

14. **Create HTTP Request Node to Upload Large Image Binary to Late Storage**
    - Name: "Upload to Storage (PUT)"
    - Method: PUT
    - URL: uploadUrl or url from LATE – Init Upload files > 4MB response
    - Content-Type: from Prepare data node
    - Input Data Field: binaryData
    - Connect Download Large (GET) → Upload to Storage (PUT)

15. **Create HTTP Request Node to Download Small Image**
    - Name: "HTTP Request for files ≤ ~4 MB"
    - Method: GET
    - URL: `={{ $('Prepare data').item.json.imageUrl }}`
    - Response Format: File (binary)
    - Connect IF – „Is Large?” (false) → HTTP Request for files ≤ ~4 MB

16. **Create HTTP Request Node to Upload Small Image Multipart to Late API**
    - Name: "Late - Upload Media files <=4MB"
    - Method: POST
    - URL: `https://getlate.dev/api/v1/media`
    - Authentication: HTTP Bearer with Late API key
    - Content-Type: multipart-form-data
    - Body: binary file from previous node
    - Connect HTTP Request for files ≤ ~4 MB → Late - Upload Media files <=4MB

17. **Create Code Node to Normalize Media Output**
    - Name: "Normalize Media Output"
    - JavaScript code to extract publicUrl or fallback URLs from Late upload response
    - Connect Upload to Storage (PUT) and Late - Upload Media files <=4MB → Normalize Media Output

18. **Create Code Node to Combine Content with Media**
    - Name: "Combine Content with Media"
    - JavaScript code merges platform texts and hashtags with media URL
    - Connect Normalize Media Output and Parse & Filter Platforms → Combine Content with Media

19. **Create If Nodes for Each Platform Enabled Check**
    - Names: "IF Facebook", "IF Instagram", "IF LinkedIn", "IF TikTok"
    - Check respective ENABLE_* variables == true
    - Connect Combine Content with Media → each IF node

20. **Create HTTP Request Nodes to Create Posts on Late API per Platform**
    - Names: "Late Create Post Facebook", "Late Create Post Instagram", "Late Create Post Linkedin", "Late Create Post TikTok"
    - Method: POST
    - URL: `https://getlate.dev/api/v1/posts`
    - Authentication: HTTP Bearer with Late API key
    - JSON Body includes content, publishNow=true, platform and accountId, mediaItems with image URL
    - Connect each IF node (true branch) → corresponding Late Create Post node

21. **Create Merge Node**
    - Name: "Merge"
    - Number of inputs: 4 (one for each platform)
    - Connect all Late Create Post nodes → Merge

22. **Create Code Node for Success Notification & Logging**
    - Name: "Success Notification & Logging"
    - JavaScript code to aggregate posts, build notification messages for Slack, Discord, Email, webhook
    - Connect Merge → Success Notification & Logging

23. **Create Respond to Webhook Node**
    - Name: "Respond to Webhook"
    - Connect Success Notification & Logging → Respond to Webhook

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                              | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| AI-Generated Social Media Posting with Late API                                                                                                                                                                                                            | Workflow title sticky note                                                                                     |
| Workflow automates generating and publishing social media posts with Google Gemini, Kie.ai, and Late API.                                                                                                                                                   | Workflow overview sticky note                                                                                  |
| Prerequisites: Google Gemini API key, Kie.ai API key, Late API key, and social account IDs from Late dashboard. Each post requires platform and accountId pair.                                                                                              | Prerequisites sticky note                                                                                       |
| Default Settings node controls business type, content topic, enabled platforms, and account IDs. Change here to enable/disable platforms.                                                                                                                  | Default Settings sticky note                                                                                   |
| Gemini API generates platform-optimized post text; response is parsed and filtered for enabled platforms.                                                                                                                                                  | Content Generation sticky note                                                                                  |
| Kie.ai Seedream v4 model generates professional social media images with modern branding style. Flow includes task creation and polling for results.                                                                                                       | Image Generation sticky note                                                                                   |
| Media images uploaded to Late API using two flows: direct multipart upload for ≤ 4MB files, or multipart init + PUT for > 4MB files.                                                                                                                       | Media Upload sticky note                                                                                        |
| Posts published to each enabled platform with content, hashtags, and media image via Late API /posts endpoint.                                                                                                                                              | Publishing Logic sticky note                                                                                    |
| Notifications generated for Slack, Discord, Email, Webhook with post details like IDs, platform, content preview, timestamp.                                                                                                                               | Notifications sticky note                                                                                       |
| Common error cases: missing accountId, large file upload failures, empty content fallback, invalid credentials. Errors only halt failing branches; others continue.                                                                                         | Error Handling sticky note                                                                                      |
| Gemini API documentation and API key setup instructions: https://ai.google.dev/gemini-api/docs/get-started                                                                                                                                                 | Sticky Note10                                                                                                  |
| Kie.ai Seedream API usage and credential setup: https://kie.ai/seedream-api                                                                                                                                                                                | Sticky Note11                                                                                                  |
| Late API documentation and usage for media upload and posting: https://getlate.dev/docs                                                                                                                                                                     | Sticky Note12                                                                                                  |
| Cost estimates for this workflow: Gemini text generation (free tier or paid tiers), Kie.ai image generation (~$0.04–$0.07/image), Late API posting (subscription-based with free tiers). Quick estimate ~ $0.05 per run excluding Late subscription.             | Run Cost Cheat-Sheet sticky note                                                                               |

---

**Disclaimer:** The provided content is based entirely on an automated n8n workflow. It complies with all applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.