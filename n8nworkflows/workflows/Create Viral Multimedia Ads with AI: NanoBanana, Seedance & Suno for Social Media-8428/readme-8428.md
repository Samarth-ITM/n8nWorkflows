Create Viral Multimedia Ads with AI: NanoBanana, Seedance & Suno for Social Media

https://n8nworkflows.xyz/workflows/create-viral-multimedia-ads-with-ai--nanobanana--seedance---suno-for-social-media-8428


# Create Viral Multimedia Ads with AI: NanoBanana, Seedance & Suno for Social Media

### 1. Workflow Overview

This workflow automates the creation and publication of viral multimedia advertisements for social media platforms by combining AI-powered image generation, video creation, and music synthesis. It is designed for marketers and content creators who want to generate engaging, authentic-looking user-generated content (UGC) ads with minimal manual effort. The workflow integrates Telegram for input reception, Google Drive for media storage, NanoBanana and Seedance AI services for image and video creation, Suno AI for background music generation, and Upload-Post for publishing on multiple social networks.

Logical blocks:

- **1.1 Input Reception:** Accepts creative ideas via Telegram messages including images and text prompts.
- **1.2 Image Generation & Editing:** Parses text prompts, generates AI-enhanced product images, and uploads them.
- **1.3 Social Image Publishing:** Posts the generated images with optimized captions on Instagram and X (Twitter).
- **1.4 Video Generation:** Uses Seedance AI to generate a product video from the created image.
- **1.5 Background Music Generation:** Creates background music with Suno AI based on user prompts.
- **1.6 Video & Audio Merging:** Combines the generated video and music into a final advertisement.
- **1.7 Final Publishing & Logging:** Uploads the final ad video to Google Drive, posts it on Facebook, TikTok, and YouTube, sends notifications via Telegram, and logs data to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Captures user-submitted creative ideas through Telegram, including a photo and a caption with semicolon-separated prompts for image, text overlay, video, and music.
- **Nodes Involved:** 
  - Trigger: Receive Idea via Telegram
  - Telegram: Get Image File
  - Google Drive: Upload Image
  - Parse Idea Into Prompts

##### Node Details

- **Trigger: Receive Idea via Telegram**
  - Type: Telegram Trigger
  - Role: Starts workflow on new Telegram message update.
  - Configuration: Listens to incoming messages.
  - Inputs: Telegram webhook
  - Outputs: Message JSON with photo and caption.
  - Failures: Connectivity, webhook misconfiguration.

- **Telegram: Get Image File**
  - Type: Telegram node (get file)
  - Role: Downloads highest resolution image from Telegram message.
  - Configuration: Extracts file_id from photo array index 2.
  - Inputs: Trigger node output
  - Outputs: Image file data.
  - Failures: File not found, Telegram API errors, file size limits.

- **Google Drive: Upload Image**
  - Type: Google Drive node
  - Role: Uploads the Telegram image to Google Drive for storage/access.
  - Configuration: Uses unique file ID as filename; stores to specified folder and drive.
  - Inputs: Telegram image file
  - Outputs: Google Drive file metadata with accessible URLs.
  - Failures: Google API auth errors, quota limits.

- **Parse Idea Into Prompts**
  - Type: Code node (JavaScript)
  - Role: Parses caption text into structured prompts: imagePrompt, textOverlay, videoPrompt, musicPrompt.
  - Configuration: Splits caption by semicolons, trims parts.
  - Inputs: Caption text from Telegram message.
  - Outputs: JSON object with four prompt strings.
  - Failures: Insufficient parts in caption, malformed input.

---

#### 1.2 Image Generation & Editing

- **Overview:** Generates an AI image prompt, creates an image using NanoBanana API, waits for editing completion, downloads the edited image, and uploads it to Google Drive.
- **Nodes Involved:**
  - Generate Image Prompt (Langchain Agent)
  - LLM: OpenAI Chat (Chat completion for prompt generation)
  - LLM: Structured Output Parser
  - NanoBanana: Create Image (HTTP Request)
  - Wait for Image Edit
  - Download Edited Image
  - Google Drive: Upload Image (already covered in 1.1 to upload original Telegram image)

##### Node Details

- **LLM: OpenAI Chat**
  - Type: Langchain OpenAI Chat
  - Role: Generates initial image prompt text using GPT-5-mini model.
  - Configuration: No extra options; uses conversational AI to generate prompt.
  - Inputs: None directly, connected internally.
  - Outputs: Text prompt for image generation.

- **LLM: Structured Output Parser**
  - Type: Langchain Output Parser
  - Role: Ensures output from LLM is structured JSON with key 'image_prompt'.
  - Configuration: JSON schema example provided.
  - Inputs: LLM Chat output.
  - Outputs: Parsed JSON for image prompt.

- **Generate Image Prompt**
  - Type: Langchain Agent Node
  - Role: Creates an image prompt based on user inputs (imagePrompt and textOverlay) following strict style and safety guidelines.
  - Configuration: System prompt instructs to produce a concise JSON object describing the image with camera style, text overlay, etc.
  - Inputs: Parsed prompts from previous node.
  - Outputs: JSON with final 'image_prompt'.
  - Edge Cases: LLM output may fail to conform; parser ensures JSON only output.

- **NanoBanana: Create Image**
  - Type: HTTP Request
  - Role: Sends image prompt plus reference image URL to NanoBanana API for AI image generation/editing.
  - Configuration: POST JSON body includes escaped prompt and image URL.
  - Inputs: JSON image prompt, Google Drive image URL.
  - Outputs: API response with image edit info.
  - Failures: API auth errors, rate limits, malformed prompt.

- **Wait for Image Edit**
  - Type: Wait node
  - Role: Pauses workflow 2 seconds to allow NanoBanana processing.
  - Inputs: NanoBanana request completion.
  - Outputs: Triggers download node after delay.

- **Download Edited Image**
  - Type: HTTP Request
  - Role: Downloads the edited image from NanoBanana using response URL.
  - Inputs: NanoBanana response URL.
  - Outputs: JSON with edited image URL array.
  - Failures: URL invalid, download timeout.

---

#### 1.3 Social Image Publishing

- **Overview:** Rewrites the original caption for TikTok/Instagram using OpenAI, posts the edited image with caption on Instagram and X (Twitter).
- **Nodes Involved:**
  - Rewrite Caption (TikTok/Instagram)
  - Post Image to Instagram + X

##### Node Details

- **Rewrite Caption (TikTok/Instagram)**
  - Type: OpenAI node (GPT-4o)
  - Role: Generates a concise caption under 200 characters based on image prompt and text overlay.
  - Configuration: Prompt instructs strict formatting and length rules.
  - Inputs: Image prompt and text overlay.
  - Outputs: Clean caption text.
  - Failures: API errors, over-length output.

- **Post Image to Instagram + X**
  - Type: Upload-Post node
  - Role: Publishes image and caption to Instagram and X platforms.
  - Configuration: Uses user account "DRFIRAS", passes photo URL and caption.
  - Inputs: Edited image URL and caption text.
  - Outputs: Posting response.
  - Failures: API auth errors, platform limits.

---

#### 1.4 Video Generation

- **Overview:** Creates a short promotional video from the generated image using Seedance AI, waits for rendering, downloads the video, and extracts the video URL.
- **Nodes Involved:**
  - Seedance: Generate Video from Image
  - Wait for Rendering
  - Download Video from Seedance
  - Set: Video URL

##### Node Details

- **Seedance: Generate Video from Image**
  - Type: HTTP Request
  - Role: Initiates video generation task on Seedance API with specified prompts, resolution, duration, aspect ratio.
  - Configuration: POST JSON body with video prompt and image URL.
  - Inputs: Video prompt and edited image URL.
  - Outputs: Task ID and status.
  - Failures: API errors, invalid parameters.

- **Wait for Rendering**
  - Type: Wait
  - Role: Delays workflow 20 seconds to allow Seedance video rendering.
  - Inputs: Task creation confirmation.
  - Outputs: Triggers video download.

- **Download Video from Seedance**
  - Type: HTTP Request
  - Role: Queries Seedance API for video task status and video URL.
  - Inputs: Task ID.
  - Outputs: Video metadata including URL.
  - Failures: Task failure, timeout.

- **Set: Video URL**
  - Type: Set node
  - Role: Extracts video URL from Seedance response JSON, normalizing possible fields.
  - Inputs: Video metadata JSON.
  - Outputs: Structured JSON with video URL.
  - Failures: JSON parse errors, missing URLs.

---

#### 1.5 Background Music Generation

- **Overview:** Generates background music for the ad using Suno AI based on the user's music prompt, waits for rendering, downloads the audio file, and sets the audio URL.
- **Nodes Involved:**
  - Suno: Generate Music
  - Wait: Music Rendering
  - Download Music File
  - Set: Audio URL

##### Node Details

- **Suno: Generate Music**
  - Type: HTTP Request
  - Role: Calls Suno AI API to create instrumental music with classical style.
  - Configuration: POST JSON body with music prompt, style parameters, callback URL placeholder.
  - Inputs: Music prompt.
  - Outputs: Task ID and status.
  - Failures: API auth errors, invalid prompt.

- **Wait: Music Rendering**
  - Type: Wait
  - Role: Pauses 120 seconds for music generation to complete.
  - Inputs: Task creation.
  - Outputs: Triggers music download.

- **Download Music File**
  - Type: HTTP Request
  - Role: Queries Suno API for music task status and audio file URL.
  - Inputs: Task ID.
  - Outputs: Audio metadata with URL.
  - Failures: Task failure, timeout.

- **Set: Audio URL**
  - Type: Set node
  - Role: Extracts audio URL from Suno response JSON.
  - Inputs: Audio metadata.
  - Outputs: JSON with 'url_audio'.
  - Failures: Missing audio URL.

---

#### 1.6 Video & Audio Merging

- **Overview:** Combines the video and audio URLs into a single final ad video using Fal.ai's FFMPEG merge API, waits for completion, checks status, and downloads the final video.
- **Nodes Involved:**
  - Merge Audio + Video
  - Wait: Merge Process
  - Check Merge Status
  - If Merge Completed
  - Check Merge Status II
  - Download Final Video

##### Node Details

- **Merge Audio + Video**
  - Type: HTTP Request
  - Role: Sends video and audio URLs to Fal.ai API for merging.
  - Configuration: POST JSON with URLs and start offset.
  - Inputs: Video URL and Audio URL.
  - Outputs: Merge task status URL.
  - Failures: API auth errors, invalid URLs.

- **Wait: Merge Process**
  - Type: Wait node
  - Role: Waits 2 seconds before polling merge status.
  - Inputs: Merge task initiation.
  - Outputs: Triggers status check.

- **Check Merge Status**
  - Type: HTTP Request
  - Role: Polls Fal.ai API for merge task status.
  - Inputs: Status URL from merge response.
  - Outputs: JSON status.
  - Failures: Network issues, server errors.

- **If Merge Completed**
  - Type: If node
  - Role: Branches workflow depending on whether status equals "COMPLETED".
  - Inputs: Status JSON.
  - Outputs: Proceeds to final download or waits again.
  - Failures: Condition mismatch.

- **Check Merge Status II**
  - Type: HTTP Request
  - Role: Rechecks merge status after waiting.
  - Inputs: Status URL.
  - Outputs: Video download URL.
  - Failures: As above.

- **Download Final Video**
  - Type: HTTP Request
  - Role: Downloads the final merged video file.
  - Inputs: Final video URL.
  - Outputs: Video binary or metadata.
  - Failures: Download issues.

---

#### 1.7 Final Publishing & Logging

- **Overview:** Uploads final video to Google Drive, reads brand settings from Google Sheets, generates ad copy with AI, saves ad data and publishing status to Google Sheets, posts final video on Facebook, TikTok, YouTube, and sends notifications via Telegram.
- **Nodes Involved:**
  - Upload Final Video to Google Drive
  - Read Brand Settings
  - Extract Brand Info
  - Ads Copywriter Generator (AI)
  - Save Ad Data to Google Sheets
  - Send Video URL via Telegram
  - Send Final Video Preview
  - Post Video on Social Media (FB, TikTok, YT)
  - Save Publishing Status to Google Sheets
  - Telegram: Send notification

##### Node Details

- **Upload Final Video to Google Drive**
  - Type: Google Drive node
  - Role: Uploads final merged video to specified Google Drive folder.
  - Inputs: Final video file.
  - Outputs: Google Drive metadata and shareable URL.
  - Failures: API quota, file size limits.

- **Read Brand Settings**
  - Type: Google Sheets node
  - Role: Reads brand and product info from a Google Sheet document.
  - Inputs: None (triggered internally).
  - Outputs: Rows with productName, category, offer, features, website.
  - Failures: Sheet access errors.

- **Extract Brand Info**
  - Type: Code node
  - Role: Extracts specific fields from multiple rows into one JSON object.
  - Inputs: Rows from Google Sheets.
  - Outputs: Structured brand info JSON.
  - Failures: Missing data.

- **Ads Copywriter Generator (AI)**
  - Type: OpenAI node
  - Role: Creates compelling social ad copy with a strict emoji and format structure.
  - Inputs: Brand info JSON.
  - Outputs: Ad copy text.
  - Failures: API errors.

- **Save Ad Data to Google Sheets**
  - Type: Google Sheets node
  - Role: Appends new ad data (status, copy, media URLs) to log sheet.
  - Inputs: Ad copy, image/video URLs.
  - Outputs: Confirmation.
  - Failures: API write errors.

- **Send Video URL via Telegram**
  - Type: Telegram node
  - Role: Sends the final video link to the original Telegram chat.
  - Inputs: Final video URL, chat ID.
  - Outputs: Message sent confirmation.
  - Failures: Telegram errors.

- **Send Final Video Preview**
  - Type: Telegram node
  - Role: Sends the final video file preview to Telegram chat.
  - Inputs: Video file.
  - Outputs: Message sent confirmation.
  - Failures: File size limits.

- **Post Video on Social Media (FB, TikTok, YT)**
  - Type: Upload-Post node
  - Role: Publishes final video with ad copy on Facebook, TikTok, and YouTube.
  - Inputs: Video URL, ad copy text.
  - Outputs: Posting confirmation.
  - Failures: API errors, platform restrictions.

- **Save Publishing Status to Google Sheets**
  - Type: Google Sheets node
  - Role: Updates the ad record status to "Published".
  - Inputs: Ad ID.
  - Outputs: Confirmation.
  - Failures: Sheet access errors.

- **Telegram: Send notification**
  - Type: Telegram node
  - Role: Sends a "Published" notification message to Telegram user.
  - Inputs: Chat ID.
  - Outputs: Message sent.
  - Failures: Telegram API errors.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                           | Input Node(s)                          | Output Node(s)                             | Sticky Note                                                                                         |
|--------------------------------|----------------------------------|-----------------------------------------|--------------------------------------|-------------------------------------------|---------------------------------------------------------------------------------------------------|
| Trigger: Receive Idea via Telegram | Telegram Trigger                 | Start workflow on new Telegram message  | -                                    | Telegram: Get Image File                    | # 🟢 Step 1 — Input                                                                               |
| Telegram: Get Image File        | Telegram                         | Download image file from Telegram        | Trigger: Receive Idea via Telegram   | Google Drive: Upload Image                  | # 🟢 Step 1 — Input                                                                               |
| Google Drive: Upload Image      | Google Drive                    | Upload Telegram image to Drive           | Telegram: Get Image File              | Parse Idea Into Prompts                      | # 🟢 Step 1 — Input                                                                               |
| Parse Idea Into Prompts         | Code                            | Parse caption into structured prompts    | Google Drive: Upload Image            | Generate Image Prompt                        | # 🟢 Step 1 — Input                                                                               |
| Generate Image Prompt           | Langchain Agent                 | Create AI image prompt JSON               | Parse Idea Into Prompts               | NanoBanana: Create Image                     | # 🟢 Step 2 — Generate Product Image                                                             |
| LLM: OpenAI Chat               | Langchain OpenAI Chat            | Generate initial image prompt text       | Think                              | Generate Image Prompt                        | # 🟢 Step 2 — Generate Product Image                                                             |
| LLM: Structured Output Parser  | Langchain Output Parser          | Parse LLM text to JSON                    | LLM: OpenAI Chat                    | Generate Image Prompt                        | # 🟢 Step 2 — Generate Product Image                                                             |
| NanoBanana: Create Image        | HTTP Request                    | Send prompt and image to NanoBanana API  | Generate Image Prompt                | Wait for Image Edit                          | # 🟢 Step 2 — Generate Product Image                                                             |
| Wait for Image Edit             | Wait                           | Wait for NanoBanana image processing     | NanoBanana: Create Image             | Download Edited Image                        | # 🟢 Step 2 — Generate Product Image                                                             |
| Download Edited Image           | HTTP Request                   | Download edited image from NanoBanana    | Wait for Image Edit                  | Post Image to Instagram + X, Rewrite Caption | # 🟢 Step 2 — Generate Product Image                                                             |
| Rewrite Caption (TikTok/Instagram) | OpenAI                         | Rewrite caption text under 200 chars     | Download Edited Image                | Post Image to Instagram + X                  | # 🟢 Step 3 — Publish Product Image on Instagram & X                                              |
| Post Image to Instagram + X     | Upload-Post                    | Publish image and caption on socials     | Rewrite Caption (TikTok/Instagram)  | -                                           | # 🟢 Step 3 — Publish Product Image on Instagram & X                                              |
| Seedance: Generate Video from Image | HTTP Request                 | Create video from image via Seedance      | Download Edited Image                | Wait for Rendering                           | # 🟢 Step 4 — Generate Product Video with Seedance                                               |
| Wait for Rendering             | Wait                           | Wait for Seedance video rendering        | Seedance: Generate Video from Image | Download Video from Seedance                 | # 🟢 Step 4 — Generate Product Video with Seedance                                               |
| Download Video from Seedance    | HTTP Request                   | Download Seedance generated video         | Wait for Rendering                  | Set: Video URL                              | # 🟢 Step 4 — Generate Product Video with Seedance                                               |
| Set: Video URL                 | Set                            | Extract and set video URL                  | Download Video from Seedance        | Suno: Generate Music                         | # 🟢 Step 4 — Generate Product Video with Seedance                                               |
| Suno: Generate Music           | HTTP Request                   | Generate background music with Suno AI    | Set: Video URL                     | Wait: Music Rendering                        | # 🟢 Step 5 — Generate Background Music with Suno                                                |
| Wait: Music Rendering           | Wait                           | Wait for Suno music generation            | Suno: Generate Music                | Download Music File                          | # 🟢 Step 5 — Generate Background Music with Suno                                                |
| Download Music File            | HTTP Request                   | Download generated music file              | Wait: Music Rendering              | Set: Audio URL                              | # 🟢 Step 5 — Generate Background Music with Suno                                                |
| Set: Audio URL                 | Set                            | Extract and set audio URL                   | Download Music File                | Merge Audio + Video                          | # 🟢 Step 5 — Generate Background Music with Suno                                                |
| Merge Audio + Video            | HTTP Request                   | Merge video and audio into final ad        | Set: Audio URL, Set: Video URL     | Wait: Merge Process                          | # 🟢 Step 6 — Merge Video & Audio into Final Ad                                                  |
| Wait: Merge Process             | Wait                           | Wait before checking merge status          | Merge Audio + Video                | Check Merge Status                           | # 🟢 Step 6 — Merge Video & Audio into Final Ad                                                  |
| Check Merge Status             | HTTP Request                   | Poll merge status from API                  | Wait: Merge Process               | If Merge Completed                           | # 🟢 Step 6 — Merge Video & Audio into Final Ad                                                  |
| If Merge Completed             | If                             | Branch on merge status completion           | Check Merge Status               | Check Merge Status II / Wait: Merge Process | # 🟢 Step 6 — Merge Video & Audio into Final Ad                                                  |
| Check Merge Status II          | HTTP Request                   | Second check for merge completion           | If Merge Completed                | Download Final Video                         | # 🟢 Step 6 — Merge Video & Audio into Final Ad                                                  |
| Download Final Video           | HTTP Request                   | Download finalized merged video             | Check Merge Status II             | Upload Final Video to Google Drive           | # 🟢 Step 6 — Merge Video & Audio into Final Ad                                                  |
| Upload Final Video to Google Drive | Google Drive                | Upload final video to Drive                  | Download Final Video              | Read Brand Settings                          | # 🟢 Step 7 — Publish Final Ad on Social Media                                                  |
| Read Brand Settings           | Google Sheets                  | Read product and brand info for ads         | Upload Final Video to Google Drive | Extract Brand Info                           | # 🟢 Step 7 — Publish Final Ad on Social Media                                                  |
| Extract Brand Info            | Code                          | Extract structured brand data from sheet    | Read Brand Settings              | Ads Copywriter Generator (AI)                | # 🟢 Step 7 — Publish Final Ad on Social Media                                                  |
| Ads Copywriter Generator (AI) | OpenAI                        | Create compelling ad copy text               | Extract Brand Info               | Save Ad Data to Google Sheets                 | # 🟢 Step 7 — Publish Final Ad on Social Media                                                  |
| Save Ad Data to Google Sheets | Google Sheets                  | Append ad data and media URLs to sheet      | Ads Copywriter Generator (AI)    | Send Video URL via Telegram                   | # 🟢 Step 7 — Publish Final Ad on Social Media                                                  |
| Send Video URL via Telegram   | Telegram                      | Send final video URL to Telegram chat        | Save Ad Data to Google Sheets    | Send Final Video Preview                      | # 🟢 Step 7 — Publish Final Ad on Social Media                                                  |
| Send Final Video Preview      | Telegram                      | Send video preview file to Telegram chat     | Send Video URL via Telegram      | Post Video on Social Media (FB, TikTok, YT)  | # 🟢 Step 7 — Publish Final Ad on Social Media                                                  |
| Post Video on Social Media (FB, TikTok, YT) | Upload-Post           | Post final video and ad copy to social media | Send Final Video Preview         | Save Publishing Status to Google Sheets       | # 🟢 Step 7 — Publish Final Ad on Social Media                                                  |
| Save Publishing Status to Google Sheets | Google Sheets          | Mark ad as Published                          | Post Video on Social Media        | Telegram: Send notification                    | # 🟢 Step 7 — Publish Final Ad on Social Media                                                  |
| Telegram: Send notification   | Telegram                      | Notify user of published status via Telegram | Save Publishing Status to Google Sheets | -                                           | # 🟢 Step 7 — Publish Final Ad on Social Media                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger: Receive Idea via Telegram**
   - Node: Telegram Trigger
   - Listen to new message updates.
   - Connect Telegram API credentials.
   - Save webhook URL.

2. **Add Telegram: Get Image File**
   - Node: Telegram
   - Configure to download the highest resolution photo from incoming message (photo[2].file_id).
   - Connect output of Telegram Trigger.

3. **Add Google Drive: Upload Image**
   - Node: Google Drive
   - Upload file from Telegram node.
   - Use unique photo file ID for filename.
   - Set target Drive ID and folder ID.
   - Connect output of Telegram: Get Image File.

4. **Add Code Node: Parse Idea Into Prompts**
   - Node: Code (JavaScript)
   - Parse caption text from Telegram message, splitting by semicolon into:
     - imagePrompt, textOverlay, videoPrompt, musicPrompt.
   - Connect output of Google Drive upload.

5. **Add Langchain OpenAI Chat Node: LLM: OpenAI Chat**
   - Use GPT-5-mini model.
   - No additional options.
   - Connect to a "Think" node if needed.

6. **Add Langchain Output Parser Node: LLM: Structured Output Parser**
   - Configure with JSON schema expecting {"image_prompt": "string"}.
   - Connect output of OpenAI Chat.

7. **Add Langchain Agent Node: Generate Image Prompt**
   - Set system message per detailed instructions (prompt formatting, style, safety).
   - Use parsed prompts (imagePrompt, textOverlay).
   - Connect output of Structured Output Parser.

8. **Add HTTP Request Node: NanoBanana: Create Image**
   - POST to NanoBanana API endpoint.
   - JSON body includes escaped image_prompt and Google Drive image URL.
   - Use HTTP Header Authentication with Fal.ai credentials.
   - Connect output of Generate Image Prompt.

9. **Add Wait Node: Wait for Image Edit**
   - Pause for 2 seconds after NanoBanana request.
   - Connect output of NanoBanana HTTP Request.

10. **Add HTTP Request Node: Download Edited Image**
    - GET request to NanoBanana response URL to fetch edited image.
    - Use Fal.ai HTTP Header Auth.
    - Connect output of Wait node.

11. **Add OpenAI Node: Rewrite Caption (TikTok/Instagram)**
    - Use GPT-4o model.
    - Prompt to rewrite caption under 200 characters based on image prompt and text overlay.
    - Connect output of Download Edited Image.

12. **Add Upload-Post Node: Post Image to Instagram + X**
    - Configure user and platforms Instagram & X.
    - Use edited image URL and rewritten caption.
    - Provide Upload-Post API credentials.
    - Connect output of Rewrite Caption node.

13. **Add HTTP Request Node: Seedance: Generate Video from Image**
    - POST to Seedance API to create video from image.
    - Body includes video prompt, image URL, resolution, duration, aspect ratio, safety check.
    - Use Kie AI HTTP Header credentials.
    - Connect output of Download Edited Image.

14. **Add Wait Node: Wait for Rendering**
    - Delay 20 seconds for video rendering.
    - Connect output of Seedance video creation request.

15. **Add HTTP Request Node: Download Video from Seedance**
    - GET request to Seedance API with task ID to get video info.
    - Use Kie AI credentials.
    - Connect output of Wait for Rendering.

16. **Add Set Node: Set Video URL**
    - Extract video URL from Seedance response JSON.
    - Connect output of Download Video node.

17. **Add HTTP Request Node: Suno: Generate Music**
    - POST to Suno API with music prompt, style, weights.
    - Use Kie AI credentials.
    - Connect output of Set Video URL.

18. **Add Wait Node: Wait Music Rendering**
    - Delay 120 seconds for music generation.
    - Connect output of Suno generate music request.

19. **Add HTTP Request Node: Download Music File**
    - GET Suno API music task status and audio URL.
    - Use Kie AI credentials.
    - Connect output of Wait Music Rendering.

20. **Add Set Node: Set Audio URL**
    - Extract audio URL from response.
    - Connect output of Download Music File.

21. **Add HTTP Request Node: Merge Audio + Video**
    - POST to Fal.ai FFMPEG merge API with video and audio URLs.
    - Use Fal.ai HTTP Header credentials.
    - Connect outputs of Set Audio URL and Set Video URL.

22. **Add Wait Node: Wait Merge Process**
    - Wait 2 seconds before polling merge status.
    - Connect output of Merge Audio + Video.

23. **Add HTTP Request Node: Check Merge Status**
    - GET request to merge status URL.
    - Use Fal.ai credentials.
    - Connect output of Wait Merge Process.

24. **Add If Node: If Merge Completed**
    - Condition: $json.status == "COMPLETED"
    - True branch connects to Check Merge Status II.
    - False branch connects back to Wait Merge Process (loop).

25. **Add HTTP Request Node: Check Merge Status II**
    - Second poll for merge status.
    - Connect output of If Node true branch.

26. **Add HTTP Request Node: Download Final Video**
    - Download video from final merge URL.
    - Connect output of Check Merge Status II.

27. **Add Google Drive Node: Upload Final Video to Google Drive**
    - Upload final video file to Drive folder.
    - Connect output of Download Final Video.

28. **Add Google Sheets Node: Read Brand Settings**
    - Read product data from Google Sheets document and sheet.
    - Connect output of Upload Final Video to Google Drive.

29. **Add Code Node: Extract Brand Info**
    - Extract specific columns into JSON: productName, productCategory, mainOffer, keyFeature1, keyFeature2, websiteURL.
    - Connect output of Read Brand Settings.

30. **Add OpenAI Node: Ads Copywriter Generator (AI)**
    - Use GPT-4o model.
    - Prompt to generate structured ad copy with emoji and formatting.
    - Connect output of Extract Brand Info.

31. **Add Google Sheets Node: Save Ad Data to Google Sheets**
    - Append new row with ad status, text, image/video URLs.
    - Connect output of Ads Copywriter Generator.

32. **Add Telegram Node: Send Video URL via Telegram**
    - Send final video URL message to original chat.
    - Connect output of Save Ad Data to Google Sheets.

33. **Add Telegram Node: Send Final Video Preview**
    - Send final video file preview to Telegram.
    - Connect output of Send Video URL via Telegram.

34. **Add Upload-Post Node: Post Video on Social Media (FB, TikTok, YT)**
    - Upload final video and ad copy to Facebook, TikTok, YouTube.
    - Use Upload-Post API credentials.
    - Connect output of Send Final Video Preview.

35. **Add Google Sheets Node: Save Publishing Status to Google Sheets**
    - Update ad record status to "Published".
    - Connect output of Post Video on Social Media.

36. **Add Telegram Node: Telegram: Send notification**
    - Notify user with "Published" message.
    - Connect output of Save Publishing Status.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Workflow created by Dr. Firas to automate viral ad creation combining NanoBanana image AI, Seedance video AI, and Suno music AI, publishing via Upload-Post integration.                                                                                                                                                                                                                                                                                                                                 | Branding note on workflow header sticky note.                                                                                       |
| Full video tutorial demonstrating workflow use: https://youtu.be/4ec9WDCz9CY                                                                                                                                                                                                                                                                                                                                                                                                                              | YouTube Video Tutorial link.                                                                                                        |
| Comprehensive documentation available on Notion: https://automatisation.notion.site/Create-viral-Ads-with-NanoBanana-Seedance-publish-on-socials-via-upload-post-2683d6550fd980ffa23ee340fdb3285e?source=copy_link                                                                                                                                                                                                                                                                                                 | Official detailed instructions, API guides, platform integrations.                                                                  |
| Requirements include Upload-Post Pro account with API key, latest n8n version, Upload-Post node installed, and Google Sheets for ad logging.                                                                                                                                                                                                                                                                                                                                                           | Requirement checklist from sticky note.                                                                                            |
| Upload-Post API documentation: https://www.upload-post.com/                                                                                                                                                                                                                                                                                                                                                                                                                                              | For correct upload-post node setup and parameters.                                                                                  |
| Google Sheets ad tracking template: https://docs.google.com/spreadsheets/d/1TCebBvfgvVJyxYXeOiFVpg0RJGP38CpLKNxcOiRwbfQ/edit?usp=sharing                                                                                                                                                                                                                                                                                                                                                                 | Google Sheets template to replicate ad logging data.                                                                               |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, respecting all content policies and containing only legal and public data.