Generate & Auto-Post Social Videos to Multiple Platforms with GPT-4 and Kling AI

https://n8nworkflows.xyz/workflows/generate---auto-post-social-videos-to-multiple-platforms-with-gpt-4-and-kling-ai-3501


# Generate & Auto-Post Social Videos to Multiple Platforms with GPT-4 and Kling AI

### 1. Workflow Overview

This workflow automates the entire process of generating, enhancing, and publishing short-form social videos across multiple platforms using AI technologies. It is designed for content creators, marketers, and social media managers who want to streamline video content production and distribution without manual intervention.

The workflow is logically divided into five main blocks:

- **1.1 Input Reception and Prompt Preparation**  
  Receives a Telegram message containing a video prompt and optional caption idea, then extracts and refines this input for AI video generation.

- **1.2 AI Video Generation**  
  Uses GPT-4 to transform the prompt into a cinematic video description optimized for Kling AI, then triggers Kling’s API to generate the video.

- **1.3 Voice-Over Creation and Audio-Video Merging**  
  Generates a concise voice-over script with GPT-4, converts it to speech audio, uploads the audio, and merges it with the AI-generated video.

- **1.4 Captioning, Metadata Generation, and Notification**  
  Adds styled subtitles to the video, creates social captions and YouTube-style titles, saves metadata to Google Sheets, and sends a preview via Telegram.

- **1.5 Multi-Platform Auto-Publishing**  
  Uploads the final video to Blotato and auto-posts it to nine social media platforms (Instagram, TikTok, YouTube, Facebook, LinkedIn, Threads, Twitter (X), Pinterest, Bluesky) using Blotato’s API.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Prompt Preparation

**Overview:**  
This block listens for Telegram messages, extracts the video prompt and caption idea, and prepares the prompt for AI video generation.

**Nodes Involved:**  
- Trigger: Telegram Prompt  
- Extract Prompt & Caption  
- Transform Prompt for Kling (GPT-4)  
- OpenAI Model Bridge  

**Node Details:**

- **Trigger: Telegram Prompt**  
  - Type: Telegram Trigger  
  - Role: Entry point; listens for incoming Telegram messages with video generation requests.  
  - Configuration: Listens for "message" updates; connected to a Telegram bot credential.  
  - Inputs: Telegram messages.  
  - Outputs: Raw Telegram message JSON.  
  - Edge Cases: Telegram API downtime, invalid message formats, or missing bot permissions.

- **Extract Prompt & Caption**  
  - Type: Code (JavaScript)  
  - Role: Parses the Telegram message text to remove the prefix "generate video", splits it into a video prompt and a caption idea.  
  - Key Expressions: Uses regex to remove prefix and split on the first comma.  
  - Inputs: Telegram message JSON.  
  - Outputs: JSON with `videoPrompt` and `captionIdea`.  
  - Edge Cases: Messages without a comma, empty prompt, or malformed input.

- **Transform Prompt for Kling (GPT-4)**  
  - Type: LangChain Agent (GPT-4)  
  - Role: Expands the short prompt into a detailed cinematic video description optimized for Kling AI.  
  - Configuration: Uses a system message with detailed instructions and examples to generate vivid, concise prompts.  
  - Inputs: `videoPrompt` from previous node.  
  - Outputs: Refined video prompt string.  
  - Edge Cases: API rate limits, unexpected AI output, or empty response.

- **OpenAI Model Bridge**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Acts as a bridge for language model processing, connected to the Transform Prompt node.  
  - Configuration: Uses GPT-4o-mini model.  
  - Inputs/Outputs: Connected downstream to Transform Prompt for Kling.  
  - Edge Cases: OpenAI API errors, credential issues.

---

#### 1.2 AI Video Generation

**Overview:**  
This block sends the refined prompt to Kling’s API to generate a cinematic video and waits for the video generation to complete.

**Nodes Involved:**  
- Generate Video via Kling API  
- Wait for Video Generation  
- Get Generated Video URL  

**Node Details:**

- **Generate Video via Kling API**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Kling’s video generation API with the cinematic prompt and video parameters (duration, aspect ratio, camera control).  
  - Configuration: Uses HTTP header authentication with API key; 10-second video, 9:16 aspect ratio, version 1.6.  
  - Inputs: Refined prompt from GPT-4.  
  - Outputs: Task ID for video generation.  
  - Edge Cases: API authentication failure, network errors, invalid prompt causing generation failure.

- **Wait for Video Generation**  
  - Type: Wait  
  - Role: Pauses workflow for 7 minutes to allow Kling to process video generation asynchronously.  
  - Inputs: Task ID from previous node.  
  - Outputs: Passes control after wait.  
  - Edge Cases: Insufficient wait time causing premature requests.

- **Get Generated Video URL**  
  - Type: HTTP Request  
  - Role: Polls Kling API to retrieve the generated video URL using the task ID.  
  - Configuration: Uses same HTTP header authentication.  
  - Inputs: Task ID.  
  - Outputs: JSON containing video URL.  
  - Edge Cases: Task not completed, API errors, invalid task ID.

---

#### 1.3 Voice-Over Creation and Audio-Video Merging

**Overview:**  
Generates a short voice-over script, converts it to audio, uploads the audio to Cloudinary, and merges it with the generated video.

**Nodes Involved:**  
- Generate Voice-Over Script  
- Convert Script to Audio (TTS)  
- Upload Audio to Cloudinary  
- Merge Audio + Video  
- Wait for Audio/Video Fusion  
- Get Video URL with Audio  

**Node Details:**

- **Generate Voice-Over Script**  
  - Type: LangChain OpenAI  
  - Role: Creates a concise 7-second voice-over script based on the caption idea.  
  - Configuration: GPT-4o model; prompts specify word count (~18-22 words), natural tone, no headers.  
  - Inputs: Caption idea from Extract Prompt & Caption.  
  - Outputs: Plain text voice-over script.  
  - Edge Cases: API errors, script too long/short.

- **Convert Script to Audio (TTS)**  
  - Type: LangChain OpenAI (Audio resource)  
  - Role: Converts the voice-over script text to speech audio.  
  - Inputs: Voice-over script text.  
  - Outputs: Audio file content.  
  - Edge Cases: TTS conversion failures, API limits.

- **Upload Audio to Cloudinary**  
  - Type: HTTP Request  
  - Role: Uploads the generated audio file to Cloudinary for hosting.  
  - Configuration: Multipart form-data upload with preset `n8n_video`.  
  - Inputs: Audio binary data.  
  - Outputs: Cloudinary URL of uploaded audio.  
  - Edge Cases: Upload failures, credential issues.

- **Merge Audio + Video**  
  - Type: HTTP Request  
  - Role: Calls JSON2Video API to merge the Kling video URL and Cloudinary audio URL into a single video with audio.  
  - Configuration: Custom resolution 720x1280; video resized to cover; uses HTTP header and basic auth.  
  - Inputs: Video URL from Kling, audio URL from Cloudinary.  
  - Outputs: Project ID for merged video.  
  - Edge Cases: API errors, invalid URLs.

- **Wait for Audio/Video Fusion**  
  - Type: Wait  
  - Role: Waits 1 minute for JSON2Video to process the merge.  
  - Inputs: Project ID.  
  - Outputs: Passes control after wait.  
  - Edge Cases: Insufficient wait time.

- **Get Video URL with Audio**  
  - Type: HTTP Request  
  - Role: Retrieves the merged video URL from JSON2Video using the project ID.  
  - Inputs: Project ID.  
  - Outputs: Final video URL with audio.  
  - Edge Cases: API errors, project not ready.

---

#### 1.4 Captioning, Metadata Generation, and Notification

**Overview:**  
Adds styled captions to the video, generates social captions and titles, saves metadata to Google Sheets, and sends the final video and caption to Telegram.

**Nodes Involved:**  
- Wait Before Captioning  
- Add Captions/Subtitles to Video  
- Wait for Caption Render  
- Get Final Video URL (Audio + Captions)  
- Generate Social Caption from Voiceover  
- Generate YouTube-Style Title  
- Save Video Metadata to Google Sheets  
- Send Final Video to Telegram  
- Send Caption Link via Telegram  

**Node Details:**

- **Wait Before Captioning**  
  - Type: Wait  
  - Role: Waits 30 seconds before starting captioning to ensure video availability.  
  - Inputs: Final video URL with audio.  
  - Outputs: Passes control.  
  - Edge Cases: Insufficient wait time.

- **Add Captions/Subtitles to Video**  
  - Type: HTTP Request  
  - Role: Calls JSON2Video API to overlay styled subtitles on the video.  
  - Configuration: Uses a "classic-progressive" subtitle style with specific fonts, colors, and positioning.  
  - Inputs: Video URL with audio.  
  - Outputs: Project ID for captioned video.  
  - Edge Cases: API errors, invalid video URL.

- **Wait for Caption Render**  
  - Type: Wait  
  - Role: Waits 1 minute for caption rendering to complete.  
  - Inputs: Captioning project ID.  
  - Outputs: Passes control.  
  - Edge Cases: Insufficient wait time.

- **Get Final Video URL (Audio + Captions)**  
  - Type: HTTP Request  
  - Role: Retrieves the final video URL including audio and captions from JSON2Video.  
  - Inputs: Captioning project ID.  
  - Outputs: Final video URL.  
  - Edge Cases: API errors, project not ready.

- **Generate Social Caption from Voiceover**  
  - Type: LangChain OpenAI  
  - Role: Creates a concise, engaging social media caption based on the voice-over script.  
  - Configuration: GPT-4o model; instructions to avoid generic fluff and special characters.  
  - Inputs: Voice-over script text.  
  - Outputs: Social caption text.  
  - Edge Cases: API errors, inappropriate content.

- **Generate YouTube-Style Title**  
  - Type: LangChain OpenAI  
  - Role: Generates a punchy, curiosity-driven YouTube video title from the voice-over script.  
  - Configuration: GPT-4o model; max 70 characters, no special characters.  
  - Inputs: Voice-over script text.  
  - Outputs: Video title text.  
  - Edge Cases: API errors, title too long or generic.

- **Save Video Metadata to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends a new row with video title, original prompt, final video URL, and description to a Google Sheet for tracking.  
  - Configuration: Uses OAuth2 credentials; dynamic document and sheet IDs.  
  - Inputs: Title, prompt, video URL, description.  
  - Outputs: Confirmation of append operation.  
  - Edge Cases: Google API quota limits, permission errors.

- **Send Final Video to Telegram**  
  - Type: Telegram  
  - Role: Sends the final video file to the Telegram chat that initiated the request.  
  - Inputs: Final video URL, chat ID.  
  - Outputs: Message confirmation.  
  - Edge Cases: Telegram API errors, file size limits.

- **Send Caption Link via Telegram**  
  - Type: Telegram  
  - Role: Sends the social caption text and video link as a message to the Telegram chat for validation or reuse.  
  - Inputs: Caption text, video URL, chat ID.  
  - Outputs: Message confirmation.  
  - Edge Cases: Telegram API errors.

---

#### 1.5 Multi-Platform Auto-Publishing

**Overview:**  
Uploads the final video to Blotato and posts it automatically to nine social media platforms using their respective IDs.

**Nodes Involved:**  
- Assign Social Media IDs  
- Upload Video to Blotato  
- Post to Instagram  
- Post to YouTube  
- Post to TikTok  
- Post to Facebook Page  
- Post to Threads  
- Post to Twitter (X)  
- Post to LinkedIn  
- Post to Bluesky  
- Post to Pinterest  

**Node Details:**

- **Assign Social Media IDs**  
  - Type: Set  
  - Role: Defines platform-specific account and board IDs required by Blotato API for posting.  
  - Configuration: Hardcoded JSON with IDs for Instagram, YouTube, TikTok, Facebook, Threads, Twitter, LinkedIn, Pinterest (including board ID), Bluesky, and Facebook page ID.  
  - Inputs: None (manual assignment).  
  - Outputs: JSON with all platform IDs.  
  - Edge Cases: Incorrect or outdated IDs will cause posting failures.

- **Upload Video to Blotato**  
  - Type: HTTP Request  
  - Role: Uploads the final video URL to Blotato’s media endpoint to prepare for posting.  
  - Configuration: POST with Blotato API key in headers; sends video URL in body.  
  - Inputs: Final video URL from Google Sheets node.  
  - Outputs: Media upload confirmation with media URL.  
  - Edge Cases: API key invalid, network errors.

- **Post to Instagram, YouTube, TikTok, Facebook Page, Threads, Twitter (X), LinkedIn, Bluesky, Pinterest**  
  - Type: HTTP Request (one per platform)  
  - Role: Posts the uploaded video with the generated social caption to each platform via Blotato API.  
  - Configuration: Each node uses platform-specific `accountId` and `targetType`, includes text caption and media URLs, and sends Blotato API key in headers.  
  - Inputs: Media URL from Blotato upload, social caption text, platform IDs.  
  - Outputs: Posting confirmation.  
  - Edge Cases: Platform-specific API limits, invalid credentials, content restrictions, or network failures.

---

### 3. Summary Table

| Node Name                     | Node Type                        | Functional Role                                  | Input Node(s)                     | Output Node(s)                          | Sticky Note                                                                                  |
|-------------------------------|---------------------------------|-------------------------------------------------|----------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------|
| Trigger: Telegram Prompt       | Telegram Trigger                | Receives Telegram messages to start workflow    | -                                | Extract Prompt & Caption               | # 🟫 STEP 1 — Create Video Using AI                                                         |
| Extract Prompt & Caption       | Code                           | Parses prompt and caption from Telegram message | Trigger: Telegram Prompt          | Transform Prompt for Kling (GPT-4)    | # 🟫 STEP 1 — Create Video Using AI                                                         |
| Transform Prompt for Kling (GPT-4) | LangChain Agent (GPT-4)         | Refines prompt for Kling video generation        | Extract Prompt & Caption          | Generate Video via Kling API           | # 🟫 STEP 1 — Create Video Using AI                                                         |
| OpenAI Model Bridge            | LangChain LM Chat OpenAI        | Language model bridge for prompt transformation | -                                | Transform Prompt for Kling (GPT-4)    | # 🟫 STEP 1 — Create Video Using AI                                                         |
| Generate Video via Kling API   | HTTP Request                   | Sends prompt to Kling API to generate video     | Transform Prompt for Kling (GPT-4)| Wait for Video Generation              | # 🟫 STEP 1 — Create Video Using AI                                                         |
| Wait for Video Generation      | Wait                           | Waits for Kling video generation to complete    | Generate Video via Kling API      | Get Generated Video URL                | # 🟫 STEP 1 — Create Video Using AI                                                         |
| Get Generated Video URL        | HTTP Request                   | Retrieves generated video URL from Kling         | Wait for Video Generation         | Generate Voice-Over Script             | # 🟫 STEP 1 — Create Video Using AI                                                         |
| Generate Voice-Over Script     | LangChain OpenAI               | Creates short voice-over script                   | Get Generated Video URL           | Convert Script to Audio (TTS)          | # 🟫 STEP 2 — Add Voice-Over to Video                                                       |
| Convert Script to Audio (TTS)  | LangChain OpenAI (Audio)       | Converts script text to speech audio              | Generate Voice-Over Script        | Upload Audio to Cloudinary             | # 🟫 STEP 2 — Add Voice-Over to Video                                                       |
| Upload Audio to Cloudinary     | HTTP Request                   | Uploads audio file to Cloudinary                   | Convert Script to Audio (TTS)     | Merge Audio + Video                    | # 🟫 STEP 2 — Add Voice-Over to Video                                                       |
| Merge Audio + Video            | HTTP Request                   | Merges Kling video and audio into one video       | Upload Audio to Cloudinary        | Wait for Audio/Video Fusion            | # 🟫 STEP 2 — Add Voice-Over to Video                                                       |
| Wait for Audio/Video Fusion    | Wait                           | Waits for merged video processing                  | Merge Audio + Video               | Get Video URL with Audio               | # 🟫 STEP 2 — Add Voice-Over to Video                                                       |
| Get Video URL with Audio       | HTTP Request                   | Retrieves merged video URL with audio              | Wait for Audio/Video Fusion       | Wait Before Captioning                 | # 🟫 STEP 2 — Add Voice-Over to Video                                                       |
| Wait Before Captioning         | Wait                           | Waits before starting captioning                    | Get Video URL with Audio          | Add Captions/Subtitles to Video       | # 🟫 STEP 3 — Add Captions to Enhance Engagement                                           |
| Add Captions/Subtitles to Video| HTTP Request                   | Adds styled subtitles to video                      | Wait Before Captioning            | Wait for Caption Render                | # 🟫 STEP 3 — Add Captions to Enhance Engagement                                           |
| Wait for Caption Render        | Wait                           | Waits for caption rendering to complete             | Add Captions/Subtitles to Video   | Get Final Video URL (Audio + Captions)| # 🟫 STEP 3 — Add Captions to Enhance Engagement                                           |
| Get Final Video URL (Audio + Captions) | HTTP Request                   | Retrieves final video URL with captions             | Wait for Caption Render           | Generate Social Caption from Voiceover| # 🟫 STEP 4 — Save Video & Notify via Telegram                                            |
| Generate Social Caption from Voiceover | LangChain OpenAI               | Creates social media caption from voice-over script| Get Final Video URL (Audio + Captions) | Generate YouTube-Style Title           | # 🟫 STEP 4 — Save Video & Notify via Telegram                                            |
| Generate YouTube-Style Title  | LangChain OpenAI               | Generates YouTube-style video title                  | Generate Social Caption from Voiceover | Save Video Metadata to Google Sheets   | # 🟫 STEP 4 — Save Video & Notify via Telegram                                            |
| Save Video Metadata to Google Sheets | Google Sheets                  | Saves video metadata for tracking                    | Generate YouTube-Style Title      | Send Final Video to Telegram           | # 🟫 STEP 4 — Save Video & Notify via Telegram                                            |
| Send Final Video to Telegram  | Telegram                      | Sends final video to Telegram chat                    | Save Video Metadata to Google Sheets | Send Caption Link via Telegram         | # 🟫 STEP 4 — Save Video & Notify via Telegram                                            |
| Send Caption Link via Telegram| Telegram                      | Sends caption and video link to Telegram chat         | Send Final Video to Telegram      | Assign Social Media IDs                | # 🟫 STEP 4 — Save Video & Notify via Telegram                                            |
| Assign Social Media IDs       | Set                           | Defines platform account IDs for auto-posting         | Send Caption Link via Telegram    | Upload Video to Blotato                | # 🟥 STEP 5 — Auto-Publish to 9 Social Platforms                                         |
| Upload Video to Blotato       | HTTP Request                  | Uploads final video to Blotato media library            | Assign Social Media IDs           | Post to Instagram, YouTube, TikTok, Facebook Page, Threads, Twitter (X), LinkedIn, Bluesky, Pinterest | # 🟥 STEP 5 — Auto-Publish to 9 Social Platforms                                         |
| Post to Instagram             | HTTP Request                  | Posts video and caption to Instagram via Blotato API   | Upload Video to Blotato           | -                                     | # 🟥 STEP 5 — Auto-Publish to 9 Social Platforms                                         |
| Post to YouTube               | HTTP Request                  | Posts video and caption to YouTube via Blotato API     | Upload Video to Blotato           | -                                     | # 🟥 STEP 5 — Auto-Publish to 9 Social Platforms                                         |
| Post to TikTok                | HTTP Request                  | Posts video and caption to TikTok via Blotato API      | Upload Video to Blotato           | -                                     | # 🟥 STEP 5 — Auto-Publish to 9 Social Platforms                                         |
| Post to Facebook Page         | HTTP Request                  | Posts video and caption to Facebook Page via Blotato API| Upload Video to Blotato           | -                                     | # 🟥 STEP 5 — Auto-Publish to 9 Social Platforms                                         |
| Post to Threads               | HTTP Request                  | Posts video and caption to Threads via Blotato API     | Upload Video to Blotato           | -                                     | # 🟥 STEP 5 — Auto-Publish to 9 Social Platforms                                         |
| Post to Twitter (X)           | HTTP Request                  | Posts video and caption to Twitter (X) via Blotato API | Upload Video to Blotato           | -                                     | # 🟥 STEP 5 — Auto-Publish to 9 Social Platforms                                         |
| Post to LinkedIn              | HTTP Request                  | Posts video and caption to LinkedIn via Blotato API    | Upload Video to Blotato           | -                                     | # 🟥 STEP 5 — Auto-Publish to 9 Social Platforms                                         |
| Post to Bluesky               | HTTP Request                  | Posts video and caption to Bluesky via Blotato API     | Upload Video to Blotato           | -                                     | # 🟥 STEP 5 — Auto-Publish to 9 Social Platforms                                         |
| Post to Pinterest             | HTTP Request                  | Posts video and caption to Pinterest via Blotato API   | Upload Video to Blotato           | -                                     | # 🟥 STEP 5 — Auto-Publish to 9 Social Platforms                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram bot credentials.  
   - Set to listen for "message" updates.

2. **Add Code Node "Extract Prompt & Caption"**  
   - Type: Code (JavaScript)  
   - Paste code to remove "generate video" prefix and split input into `videoPrompt` and `captionIdea`.  
   - Connect output of Telegram Trigger to this node.

3. **Add LangChain Agent Node "Transform Prompt for Kling (GPT-4)"**  
   - Use GPT-4 model with a system prompt instructing to expand the prompt into cinematic video description.  
   - Input: `videoPrompt` from previous node.  
   - Connect output of "Extract Prompt & Caption" to this node.

4. **Add LangChain LM Chat OpenAI Node "OpenAI Model Bridge"**  
   - Use GPT-4o-mini model.  
   - Connect output to "Transform Prompt for Kling (GPT-4)" node.

5. **Add HTTP Request Node "Generate Video via Kling API"**  
   - POST to Kling API endpoint with JSON body containing the cinematic prompt and video parameters (duration 10s, aspect ratio 9:16, version 1.6).  
   - Use HTTP header authentication with Kling API key.  
   - Connect output of "Transform Prompt for Kling (GPT-4)" to this node.

6. **Add Wait Node "Wait for Video Generation"**  
   - Wait 7 minutes to allow video generation.  
   - Connect output of Kling API node to this node.

7. **Add HTTP Request Node "Get Generated Video URL"**  
   - GET request to Kling API with task ID from previous node to fetch video URL.  
   - Use same authentication as Kling API node.  
   - Connect output of wait node to this node.

8. **Add LangChain OpenAI Node "Generate Voice-Over Script"**  
   - Use GPT-4o model.  
   - Prompt: Generate a 7-second voice-over script based on `captionIdea`.  
   - Connect output of "Get Generated Video URL" to this node.

9. **Add LangChain OpenAI Node "Convert Script to Audio (TTS)"**  
   - Resource: Audio (TTS).  
   - Input: Voice-over script text from previous node.  
   - Connect output of "Generate Voice-Over Script" to this node.

10. **Add HTTP Request Node "Upload Audio to Cloudinary"**  
    - POST multipart/form-data to Cloudinary upload endpoint.  
    - Use Cloudinary credentials and upload preset `n8n_video`.  
    - Input: Audio binary data from TTS node.  
    - Connect output of TTS node to this node.

11. **Add HTTP Request Node "Merge Audio + Video"**  
    - POST to JSON2Video API to merge video URL (from Kling) and audio URL (from Cloudinary).  
    - Use HTTP header and basic auth credentials.  
    - Connect output of Cloudinary upload node to this node.

12. **Add Wait Node "Wait for Audio/Video Fusion"**  
    - Wait 1 minute for merging process.  
    - Connect output of merge node to this node.

13. **Add HTTP Request Node "Get Video URL with Audio"**  
    - GET request to JSON2Video API to retrieve merged video URL using project ID.  
    - Use HTTP header auth.  
    - Connect output of wait node to this node.

14. **Add Wait Node "Wait Before Captioning"**  
    - Wait 30 seconds before captioning.  
    - Connect output of merged video URL node to this node.

15. **Add HTTP Request Node "Add Captions/Subtitles to Video"**  
    - POST to JSON2Video API to add styled subtitles with specified font, size, colors, and position.  
    - Use HTTP header auth.  
    - Input: Video URL with audio.  
    - Connect output of wait node to this node.

16. **Add Wait Node "Wait for Caption Render"**  
    - Wait 1 minute for caption rendering.  
    - Connect output of captioning node to this node.

17. **Add HTTP Request Node "Get Final Video URL (Audio + Captions)"**  
    - GET request to JSON2Video API to retrieve final video URL with captions.  
    - Use HTTP header auth.  
    - Connect output of wait node to this node.

18. **Add LangChain OpenAI Node "Generate Social Caption from Voiceover"**  
    - Use GPT-4o model.  
    - Input: Voice-over script text.  
    - Prompt: Generate concise, engaging social media caption.  
    - Connect output of final video URL node to this node.

19. **Add LangChain OpenAI Node "Generate YouTube-Style Title"**  
    - Use GPT-4o model.  
    - Input: Voice-over script text.  
    - Prompt: Generate punchy YouTube video title under 70 characters.  
    - Connect output of social caption node to this node.

20. **Add Google Sheets Node "Save Video Metadata to Google Sheets"**  
    - Append row with columns: Title, Prompt, Video URL, Description.  
    - Use OAuth2 credentials.  
    - Connect output of title generation node to this node.

21. **Add Telegram Node "Send Final Video to Telegram"**  
    - Send video file to Telegram chat ID from original message.  
    - Connect output of Google Sheets node to this node.

22. **Add Telegram Node "Send Caption Link via Telegram"**  
    - Send social caption and video link to Telegram chat.  
    - Connect output of previous Telegram node to this node.

23. **Add Set Node "Assign Social Media IDs"**  
    - Define JSON with platform account IDs and board/page IDs for Blotato.  
    - Connect output of Telegram caption node to this node.

24. **Add HTTP Request Node "Upload Video to Blotato"**  
    - POST video URL to Blotato media endpoint.  
    - Use Blotato API key in headers.  
    - Connect output of Set node to this node.

25. **Add HTTP Request Nodes for Each Platform Posting**  
    - Instagram, YouTube, TikTok, Facebook Page, Threads, Twitter (X), LinkedIn, Bluesky, Pinterest.  
    - POST to Blotato posts endpoint with platform-specific account IDs, captions, and media URLs.  
    - Use Blotato API key in headers.  
    - Connect all platform posting nodes to output of Blotato upload node.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow uses Community Nodes, which require a self-hosted n8n instance.                             | Workflow Setup Instructions                                                                                  |
| Documentation and detailed setup guide available on Notion.                                              | [Notion Guide](https://automatisation.notion.site/AI-powered-social-video-creation-and-auto-posting-to-Instagram-TikTok-LinkedIn-more-1cf3d6550fd98038a91edbdd3022d37f?pvs=4) |
| Full video tutorial demonstrating the workflow is available on YouTube.                                  | [YouTube Demo](https://youtu.be/lpfe0joyNL4)                                                                |
| The workflow supports customization such as changing visual style, voice style, captions, and platform targets. | Customization Instructions                                                                                   |
| Telegram messages must start with "generate video" followed by prompt and optional caption separated by a comma. | Input Format                                                                                                 |
| The workflow integrates multiple external services: Telegram, OpenAI GPT-4, Kling AI, Cloudinary, JSON2Video, Google Sheets, and Blotato. | Integration Overview                                                                                          |
| Potential failure points include API rate limits, authentication errors, network issues, and timing mismatches in wait nodes. | Error Handling Considerations                                                                                 |

---

This structured documentation provides a comprehensive understanding of the workflow’s architecture, node configurations, and operational flow, enabling advanced users and automation agents to reproduce, modify, and troubleshoot the workflow effectively.