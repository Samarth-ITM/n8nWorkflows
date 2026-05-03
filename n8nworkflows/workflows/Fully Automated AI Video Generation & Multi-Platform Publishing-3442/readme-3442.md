Fully Automated AI Video Generation & Multi-Platform Publishing

https://n8nworkflows.xyz/workflows/fully-automated-ai-video-generation---multi-platform-publishing-3442


# Fully Automated AI Video Generation & Multi-Platform Publishing

### 1. Workflow Overview

This workflow automates the entire process of creating short-form, Point-of-View (POV) style videos from ideas stored in a Google Sheet and publishing them across multiple social media platforms. It leverages AI services for content generation, image and video creation, voiceover synthesis, video assembly, description generation, and multi-platform distribution. The workflow is designed for content creators, marketers, social media managers, and businesses seeking to scale video production and distribution efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Input Acquisition:** Scheduled trigger and loading video ideas from Google Sheets.
- **1.2 AI Content Generation:** Generating video captions, image prompts, images, video clips, and voiceover scripts/audio.
- **1.3 Video Assembly:** Combining video clips, captions, and voiceovers into a final video using Creatomate.
- **1.4 Description Generation:** Extracting audio from the final video and generating social media descriptions.
- **1.5 Multi-Platform Publishing:** Uploading the final video and descriptions to TikTok, Instagram, YouTube, Facebook, and LinkedIn.
- **1.6 Tracking & Notification:** Updating Google Sheets with results and sending Discord notifications.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Acquisition

**Overview:**  
This block triggers the workflow daily at a specified hour and loads new video ideas from a Google Sheet filtered for production readiness.

**Nodes Involved:**  
- Once Per Day  
- Set API Keys  
- Load Google Sheet

**Node Details:**

- **Once Per Day**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow daily at 7 AM.  
  - Configuration: Trigger set to hour 7 daily.  
  - Inputs: None  
  - Outputs: Triggers Set API Keys node.  
  - Edge Cases: Missed triggers if n8n is down; time zone considerations.

- **Set API Keys**  
  - Type: Set  
  - Role: Stores API keys and template IDs for downstream nodes.  
  - Configuration: Contains keys for PiAPI, ElevenLabs, Creatomate, and Creatomate Template ID.  
  - Inputs: Trigger from Once Per Day  
  - Outputs: Loads Google Sheet node  
  - Edge Cases: Missing or invalid keys cause failures downstream.

- **Load Google Sheet**  
  - Type: Google Sheets  
  - Role: Retrieves one video idea marked "for production" from the Google Sheet.  
  - Configuration: Filters rows where `production` column equals "for production".  
  - Inputs: Set API Keys  
  - Outputs: Generate Video Captions node  
  - Edge Cases: API quota limits, OAuth token expiration, empty or malformed sheet data.

---

#### 1.2 AI Content Generation

**Overview:**  
Generates video captions, image prompts, images, video clips, voiceover scripts, and voiceover audio using OpenAI, PiAPI (Flux and Kling models), and ElevenLabs.

**Nodes Involved:**  
- Generate Video Captions  
- Create List  
- Validate list formatting  
- Generate Image Prompts  
- Calculate Token Usage  
- Generate Image  
- Wait 3min  
- Get image  
- Check for failures  
- Wait 5min  
- Image-to-Video  
- Wait 10min  
- Get Video  
- Fail check  
- Wait to retry  
- Match captions with videos  
- List Elements  
- Generate Script  
- Generate voice  
- Upload Voice Audio  
- Set Access Permissions  
- List Elements1  
- Pair Videos with Audio

**Node Details:**

- **Generate Video Captions**  
  - Type: OpenAI (LangChain)  
  - Role: Generates 5 unhinged, entertaining TikTok captions based on the video idea.  
  - Configuration: Uses GPT-4o-mini with a detailed prompt emphasizing first-person POV, edgy tone, and job hunting themes.  
  - Inputs: Load Google Sheet (idea)  
  - Outputs: Create List  
  - Edge Cases: OpenAI API rate limits, prompt failures, unexpected output format.

- **Create List**  
  - Type: Code  
  - Role: Splits OpenAI caption response into an array of 5 caption items.  
  - Configuration: Splits text by newline, trims, filters empty lines.  
  - Inputs: Generate Video Captions  
  - Outputs: Validate list formatting  
  - Edge Cases: Empty or malformed text causing empty list.

- **Validate list formatting**  
  - Type: If  
  - Role: Checks if the list contains more than one item to ensure valid caption generation.  
  - Inputs: Create List  
  - Outputs:  
    - True: Generate Image Prompts, Match captions with videos, Generate Script  
    - False: Re-trigger Generate Video Captions (retry)  
  - Edge Cases: Infinite loop if OpenAI consistently fails.

- **Generate Image Prompts**  
  - Type: OpenAI (LangChain)  
  - Role: Expands each caption into a detailed, hyper-realistic image prompt for Flux model.  
  - Configuration: Uses GPT-3.5 o3-mini model with a strict prompt to avoid quotes/emojis and enforce first-person POV.  
  - Inputs: Validate list formatting (true branch)  
  - Outputs: Calculate Token Usage  
  - Edge Cases: API failures, prompt misinterpretation.

- **Calculate Token Usage**  
  - Type: Code  
  - Role: Totals prompt and completion tokens from multiple OpenAI responses for cost tracking.  
  - Inputs: Generate Image Prompts  
  - Outputs: Generate Image  
  - Edge Cases: Missing usage data.

- **Generate Image**  
  - Type: HTTP Request  
  - Role: Calls PiAPI Flux model to generate images from prompts.  
  - Configuration: POST to PiAPI with prompt, negative prompt, resolution 540x960, API key from Set API Keys node.  
  - Inputs: Calculate Token Usage  
  - Outputs: Wait 3min  
  - Edge Cases: API key invalid, rate limits, generation failures.

- **Wait 3min**  
  - Type: Wait  
  - Role: Pauses workflow to allow image generation to complete asynchronously.  
  - Inputs: Generate Image  
  - Outputs: Get image  
  - Edge Cases: Fixed wait time may be insufficient or excessive.

- **Get image**  
  - Type: HTTP Request  
  - Role: Polls PiAPI for image generation task status and result.  
  - Inputs: Wait 3min  
  - Outputs: Check for failures  
  - Edge Cases: Task failure, API errors.

- **Check for failures**  
  - Type: If  
  - Role: Checks if image generation task failed.  
  - Inputs: Get image  
  - Outputs:  
    - True: Wait 5min (retry)  
    - False: Image-to-Video  
  - Edge Cases: Failure detection accuracy.

- **Wait 5min**  
  - Type: Wait  
  - Role: Waits before retrying image generation.  
  - Inputs: Check for failures (true)  
  - Outputs: Generate Image  
  - Edge Cases: Potential infinite retry loop.

- **Image-to-Video**  
  - Type: HTTP Request  
  - Role: Calls PiAPI Kling model to generate 5-second video clips from images.  
  - Configuration: POST with prompt, negative prompt, cfg_scale 0.5, mode "pro", zoom 5, API key from Set API Keys.  
  - Inputs: Check for failures (false) or Wait to retry  
  - Outputs: Wait 10min  
  - Edge Cases: API failures, invalid inputs.

- **Wait 10min**  
  - Type: Wait  
  - Role: Pauses to allow video generation to complete.  
  - Inputs: Image-to-Video  
  - Outputs: Get Video  
  - Edge Cases: Fixed wait time may not match actual processing time.

- **Get Video**  
  - Type: HTTP Request  
  - Role: Polls PiAPI for video generation task status and result.  
  - Inputs: Wait 10min  
  - Outputs: Fail check  
  - Edge Cases: Task failure, API errors.

- **Fail check**  
  - Type: If  
  - Role: Checks if video generation failed.  
  - Inputs: Get Video  
  - Outputs:  
    - True: Wait to retry  
    - False: Match captions with videos  
  - Edge Cases: Failure detection accuracy.

- **Wait to retry**  
  - Type: Wait  
  - Role: Waits before retrying video generation.  
  - Inputs: Fail check (true)  
  - Outputs: Image-to-Video  
  - Edge Cases: Potential infinite retry loop.

- **Match captions with videos**  
  - Type: Merge (combine by position)  
  - Role: Combines captions and video URLs into a single data structure.  
  - Inputs: Fail check (false), Validate list formatting (true)  
  - Outputs: List Elements  
  - Edge Cases: Mismatched array lengths.

- **List Elements**  
  - Type: Code  
  - Role: Aggregates scene titles, video URLs, and token usage into one item for video rendering.  
  - Inputs: Match captions with videos  
  - Outputs: Pair Videos with Audio  
  - Edge Cases: Missing or inconsistent data.

- **Generate Script**  
  - Type: OpenAI (LangChain)  
  - Role: Generates a funny, edgy voiceover script narrating the 5 captions.  
  - Configuration: GPT-4o-mini model with prompt to emulate Andrew Tate/Charlie Sheen style, no emojis.  
  - Inputs: Validate list formatting (true)  
  - Outputs: Generate voice  
  - Edge Cases: API failures, inappropriate content.

- **Generate voice**  
  - Type: HTTP Request  
  - Role: Calls ElevenLabs API to convert script text to speech audio.  
  - Configuration: POST with text body, ElevenLabs API key from Set API Keys.  
  - Inputs: Generate Script  
  - Outputs: Upload Voice Audio  
  - Edge Cases: API key invalid, rate limits, audio generation failures.

- **Upload Voice Audio**  
  - Type: Google Drive  
  - Role: Uploads generated voiceover audio to Google Drive.  
  - Configuration: Uploads as `<Google Sheet ID>-voiceover.mp3` to a specified folder.  
  - Inputs: Generate voice  
  - Outputs: Set Access Permissions  
  - Edge Cases: Google Drive API quota, permission errors.

- **Set Access Permissions**  
  - Type: Google Drive  
  - Role: Sets sharing permissions on uploaded audio file to allow public access.  
  - Inputs: Upload Voice Audio  
  - Outputs: List Elements1  
  - Edge Cases: Permission setting failures.

- **List Elements1**  
  - Type: Code  
  - Role: Aggregates sound URLs from uploaded audio for video assembly.  
  - Inputs: Set Access Permissions  
  - Outputs: Pair Videos with Audio  
  - Edge Cases: Missing URLs.

- **Pair Videos with Audio**  
  - Type: Merge (combine by position)  
  - Role: Combines video URLs, captions, and audio URLs into a single item for rendering.  
  - Inputs: List Elements, List Elements1  
  - Outputs: Render Final Video  
  - Edge Cases: Mismatched array lengths.

---

#### 1.3 Video Assembly

**Overview:**  
Uses Creatomate API to render the final video by combining generated video clips, captions, and voiceover audio.

**Nodes Involved:**  
- Render Final Video  
- Wait1  
- Get Final Video  
- Get Raw File  
- Upload Final Video  
- Set Permissions  
- Update Google Sheet  
- Notify me on Discord

**Node Details:**

- **Render Final Video**  
  - Type: HTTP Request  
  - Role: Sends rendering request to Creatomate with template ID and modifications (video sources, audio source, captions).  
  - Inputs: Pair Videos with Audio  
  - Outputs: Wait1  
  - Edge Cases: API key invalid, rendering errors.

- **Wait1**  
  - Type: Wait  
  - Role: Pauses to allow Creatomate rendering to complete (3 minutes).  
  - Inputs: Render Final Video  
  - Outputs: Get Final Video  
  - Edge Cases: Fixed wait may be insufficient.

- **Get Final Video**  
  - Type: HTTP Request  
  - Role: Polls Creatomate for render status and final video URL.  
  - Inputs: Wait1  
  - Outputs: Get Raw File  
  - Edge Cases: API errors, render failures.

- **Get Raw File**  
  - Type: HTTP Request  
  - Role: Downloads the rendered video file from Creatomate URL.  
  - Inputs: Get Final Video  
  - Outputs: Upload Final Video, Write video  
  - Edge Cases: Network errors, file corruption.

- **Upload Final Video**  
  - Type: Google Drive  
  - Role: Uploads the final rendered video to Google Drive in a specified folder.  
  - Inputs: Get Raw File  
  - Outputs: Set Permissions  
  - Edge Cases: API quota, upload failures.

- **Set Permissions**  
  - Type: Google Drive  
  - Role: Sets sharing permissions on the uploaded video file to allow public access.  
  - Inputs: Upload Final Video  
  - Outputs: Update Google Sheet  
  - Edge Cases: Permission errors.

- **Update Google Sheet**  
  - Type: Google Sheets  
  - Role: Updates the original Google Sheet row with video metadata, token usage, costs, and marks production as done.  
  - Inputs: Set Permissions  
  - Outputs: Notify me on Discord  
  - Edge Cases: API quota, data mismatch.

- **Notify me on Discord**  
  - Type: Discord  
  - Role: Sends a notification message to a Discord channel via webhook indicating video creation completion with link.  
  - Inputs: Update Google Sheet  
  - Outputs: None  
  - Edge Cases: Webhook URL invalid, Discord API errors.

---

#### 1.4 Description Generation

**Overview:**  
Extracts audio from the final video and generates engaging social media descriptions using OpenAI.

**Nodes Involved:**  
- Write video  
- Get Audio from Video  
- Generate Description for Videos in Tiktok and Instagram  
- Read Video from Google Drive

**Node Details:**

- **Write video**  
  - Type: Write Binary File  
  - Role: Writes the downloaded video file to local disk for processing.  
  - Inputs: Get Raw File  
  - Outputs: Get Audio from Video  
  - Edge Cases: Disk space, file write errors.

- **Get Audio from Video**  
  - Type: OpenAI (LangChain)  
  - Role: Uses OpenAI Whisper to transcribe audio from the video file.  
  - Inputs: Write video  
  - Outputs: Generate Description for Videos in Tiktok and Instagram  
  - Edge Cases: Transcription errors, API limits.

- **Generate Description for Videos in Tiktok and Instagram**  
  - Type: OpenAI (LangChain)  
  - Role: Generates social media post descriptions based on the transcribed audio.  
  - Inputs: Get Audio from Video  
  - Outputs: Read Video from Google Drive  
  - Edge Cases: API failures, inappropriate content.

- **Read Video from Google Drive**  
  - Type: Read Binary File  
  - Role: Reads the video file from disk to prepare for upload to social platforms.  
  - Inputs: Generate Description for Videos in Tiktok and Instagram  
  - Outputs: Upload Video and Description nodes  
  - Edge Cases: File not found, read errors.

---

#### 1.5 Multi-Platform Publishing

**Overview:**  
Uploads the final video and generated description to TikTok, Instagram, YouTube, Facebook, and LinkedIn via upload-post.com API.

**Nodes Involved:**  
- Upload Video and Description to Tiktok  
- Upload Video and Description to Instagram  
- Upload Video and Description to Youtube  
- Upload Video and Description to Facebook  
- Upload Video and Description to Linkedin

**Node Details:**

- **Upload Video and Description to [Platform]** (TikTok, Instagram, YouTube, Facebook, LinkedIn)  
  - Type: HTTP Request  
  - Role: Uploads video and description to respective social media platform using upload-post.com API.  
  - Configuration:  
    - Method: POST  
    - Content-Type: multipart/form-data  
    - Authentication: API key in HTTP header  
    - Body: video file binary, description text, platform identifier, user ID, Facebook Page ID for Facebook node.  
  - Inputs: Read Video from Google Drive  
  - Outputs: None  
  - Edge Cases: API token invalid, upload failures, platform-specific restrictions.

---

#### 1.6 Tracking & Notification

**Overview:**  
Updates Google Sheet with final metadata and sends Discord notification upon workflow completion.

**Nodes Involved:**  
- Update Google Sheet  
- Notify me on Discord

**Node Details:**  
Covered in Block 1.3 Video Assembly.

---

### 3. Summary Table

| Node Name                             | Node Type                  | Functional Role                                | Input Node(s)                      | Output Node(s)                                | Sticky Note                                                                                              |
|-------------------------------------|----------------------------|-----------------------------------------------|----------------------------------|-----------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Once Per Day                        | Schedule Trigger           | Daily trigger to start workflow                | None                             | Set API Keys                                  |                                                                                                        |
| Set API Keys                       | Set                       | Stores API keys and template IDs               | Once Per Day                    | Load Google Sheet                             | SET BEFORE STARTING                                                                                    |
| Load Google Sheet                  | Google Sheets              | Loads video ideas filtered for production      | Set API Keys                   | Generate Video Captions                       | ## 1. 🗨️Generate video captions from ideas in a Google Sheet (see detailed instructions)               |
| Generate Video Captions            | OpenAI (LangChain)         | Generates 5 TikTok captions from idea          | Load Google Sheet              | Create List                                   |                                                                                                        |
| Create List                       | Code                      | Splits captions text into list items           | Generate Video Captions        | Validate list formatting                      |                                                                                                        |
| Validate list formatting          | If                        | Validates caption list length                   | Create List                   | Generate Image Prompts, Match captions with videos, Generate Script (true branch); Generate Video Captions (false branch) |                                                                                                        |
| Generate Image Prompts            | OpenAI (LangChain)         | Expands captions into detailed image prompts   | Validate list formatting       | Calculate Token Usage                         | ## 2. 🖼️Generate images with Flux using PiAPI (see Flux API docs)                                     |
| Calculate Token Usage             | Code                      | Totals OpenAI token usage for cost tracking    | Generate Image Prompts         | Generate Image                                |                                                                                                        |
| Generate Image                   | HTTP Request               | Calls PiAPI Flux model to generate images      | Calculate Token Usage          | Wait 3min                                     |                                                                                                        |
| Wait 3min                       | Wait                      | Waits for image generation completion          | Generate Image                | Get image                                     |                                                                                                        |
| Get image                       | HTTP Request               | Polls PiAPI for image generation result        | Wait 3min                    | Check for failures                            |                                                                                                        |
| Check for failures              | If                        | Checks if image generation failed               | Get image                    | Wait 5min (fail), Image-to-Video (success)   |                                                                                                        |
| Wait 5min                      | Wait                      | Waits before retrying image generation          | Check for failures (fail)     | Generate Image                                |                                                                                                        |
| Image-to-Video                 | HTTP Request               | Calls PiAPI Kling model to generate video clips| Check for failures (success), Wait to retry | Wait 10min                                    | ## 3. 🎬Generate videos with Kling using PiAPI (see Kling API docs)                                   |
| Wait 10min                    | Wait                      | Waits for video generation completion          | Image-to-Video               | Get Video                                     |                                                                                                        |
| Get Video                     | HTTP Request               | Polls PiAPI for video generation result        | Wait 10min                  | Fail check                                    |                                                                                                        |
| Fail check                   | If                        | Checks if video generation failed               | Get Video                   | Wait to retry (fail), Match captions with videos (success) |                                                                                                        |
| Wait to retry               | Wait                      | Waits before retrying video generation          | Fail check (fail)            | Image-to-Video                                |                                                                                                        |
| Match captions with videos  | Merge (combine by position)| Combines captions and video URLs                | Fail check (success), Validate list formatting (true) | List Elements                                  |                                                                                                        |
| List Elements               | Code                      | Aggregates scene titles, video URLs, token usage| Match captions with videos    | Pair Videos with Audio                        |                                                                                                        |
| Generate Script             | OpenAI (LangChain)         | Generates voiceover script from captions        | Validate list formatting (true) | Generate voice                                |                                                                                                        |
| Generate voice             | HTTP Request               | Converts script text to speech via ElevenLabs  | Generate Script              | Upload Voice Audio                            | ## 4. 🔉Generate voice overs with Eleven Labs (see Eleven Labs API docs)                              |
| Upload Voice Audio         | Google Drive               | Uploads voiceover audio to Google Drive         | Generate voice              | Set Access Permissions                        |                                                                                                        |
| Set Access Permissions     | Google Drive               | Sets public sharing permissions on audio file   | Upload Voice Audio          | List Elements1                                |                                                                                                        |
| List Elements1             | Code                      | Aggregates sound URLs                            | Set Access Permissions      | Pair Videos with Audio                        |                                                                                                        |
| Pair Videos with Audio     | Merge (combine by position)| Combines video URLs, captions, and audio URLs   | List Elements, List Elements1 | Render Final Video                            |                                                                                                        |
| Render Final Video         | HTTP Request               | Sends render request to Creatomate API           | Pair Videos with Audio       | Wait1                                         | ## 5. 📥Complete video with Creatomate (see Creatomate API docs)                                     |
| Wait1                     | Wait                      | Waits for video rendering completion             | Render Final Video           | Get Final Video                               |                                                                                                        |
| Get Final Video           | HTTP Request               | Polls Creatomate for rendered video URL          | Wait1                       | Get Raw File                                  |                                                                                                        |
| Get Raw File              | HTTP Request               | Downloads rendered video file                      | Get Final Video             | Upload Final Video, Write video               |                                                                                                        |
| Upload Final Video        | Google Drive               | Uploads final video to Google Drive                | Get Raw File                | Set Permissions                               |                                                                                                        |
| Set Permissions           | Google Drive               | Sets public sharing permissions on video file     | Upload Final Video          | Update Google Sheet                           |                                                                                                        |
| Update Google Sheet       | Google Sheets              | Updates sheet with metadata, costs, status        | Set Permissions             | Notify me on Discord                          |                                                                                                        |
| Notify me on Discord      | Discord                   | Sends completion notification via Discord webhook | Update Google Sheet         | None                                          |                                                                                                        |
| Write video               | Write Binary File          | Writes downloaded video file to disk               | Get Raw File                | Get Audio from Video                          |                                                                                                        |
| Get Audio from Video      | OpenAI (LangChain)         | Transcribes audio from video using Whisper         | Write video                 | Generate Description for Videos in Tiktok and Instagram |                                                                                                        |
| Generate Description for Videos in Tiktok and Instagram | OpenAI (LangChain) | Generates social media descriptions from audio transcription | Get Audio from Video         | Read Video from Google Drive                  |                                                                                                        |
| Read Video from Google Drive | Read Binary File          | Reads video file from disk for upload               | Generate Description for Videos in Tiktok and Instagram | Upload Video and Description to [Platforms] |                                                                                                        |
| Upload Video and Description to Tiktok | HTTP Request         | Uploads video and description to TikTok via upload-post.com | Read Video from Google Drive | None                                          | ## 6. Upload to all social networks with upload-post.com (see upload-post.com API docs)              |
| Upload Video and Description to Instagram | HTTP Request     | Uploads video and description to Instagram via upload-post.com | Read Video from Google Drive | None                                          |                                                                                                        |
| Upload Video and Description to Youtube | HTTP Request       | Uploads video and description to YouTube via upload-post.com | Read Video from Google Drive | None                                          |                                                                                                        |
| Upload Video and Description to Facebook | HTTP Request      | Uploads video and description to Facebook via upload-post.com | Read Video from Google Drive | None                                          |                                                                                                        |
| Upload Video and Description to Linkedin | HTTP Request      | Uploads video and description to LinkedIn via upload-post.com | Read Video from Google Drive | None                                          |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node** named `Once Per Day`:
   - Set to trigger daily at 7:00 AM.

2. **Add a Set node** named `Set API Keys`:
   - Define string parameters for:
     - PiAPI Key
     - ElevenLabs API Key
     - Creatomate API Key
     - Creatomate Template ID
   - Connect `Once Per Day` → `Set API Keys`.

3. **Add a Google Sheets node** named `Load Google Sheet`:
   - Connect to your Google Sheets account with OAuth2 credentials.
   - Set Document ID to your copied Google Sheet template.
   - Set Sheet Name to the first sheet (gid=0).
   - Add filter: `production` column equals "for production".
   - Connect `Set API Keys` → `Load Google Sheet`.

4. **Add an OpenAI (LangChain) node** named `Generate Video Captions`:
   - Use GPT-4o-mini model.
   - Configure prompt to generate 5 edgy TikTok captions based on the idea from Google Sheet.
   - Connect `Load Google Sheet` → `Generate Video Captions`.

5. **Add a Code node** named `Create List`:
   - JavaScript to split OpenAI response text by newline into separate items.
   - Connect `Generate Video Captions` → `Create List`.

6. **Add an If node** named `Validate list formatting`:
   - Condition: Check if input array length > 1.
   - True branch connects to:
     - `Generate Image Prompts`
     - `Match captions with videos`
     - `Generate Script`
   - False branch loops back to `Generate Video Captions` (retry).
   - Connect `Create List` → `Validate list formatting`.

7. **Add an OpenAI (LangChain) node** named `Generate Image Prompts`:
   - Use GPT-3.5 o3-mini model.
   - Prompt expands captions into detailed image prompts for Flux.
   - Connect `Validate list formatting` (true) → `Generate Image Prompts`.

8. **Add a Code node** named `Calculate Token Usage`:
   - JavaScript sums prompt and completion tokens from OpenAI responses.
   - Connect `Generate Image Prompts` → `Calculate Token Usage`.

9. **Add an HTTP Request node** named `Generate Image`:
   - POST to `https://api.piapi.ai/api/v1/task` with Flux model parameters.
   - Use PiAPI Key from `Set API Keys`.
   - Connect `Calculate Token Usage` → `Generate Image`.

10. **Add a Wait node** named `Wait 3min`:
    - Wait 3 minutes.
    - Connect `Generate Image` → `Wait 3min`.

11. **Add an HTTP Request node** named `Get image`:
    - GET request to PiAPI task status endpoint using task_id.
    - Use PiAPI Key.
    - Connect `Wait 3min` → `Get image`.

12. **Add an If node** named `Check for failures`:
    - Check if `data.status` equals "failed".
    - True branch → `Wait 5min` (retry).
    - False branch → `Image-to-Video`.
    - Connect `Get image` → `Check for failures`.

13. **Add a Wait node** named `Wait 5min`:
    - Wait 5 minutes before retry.
    - Connect `Check for failures` (true) → `Wait 5min`.

14. **Connect `Wait 5min` → `Generate Image`** to retry image generation.

15. **Add an HTTP Request node** named `Image-to-Video`:
    - POST to PiAPI with Kling model parameters to generate video clips.
    - Use PiAPI Key.
    - Connect `Check for failures` (false) → `Image-to-Video`.
    - Also connect `Wait to retry` → `Image-to-Video` (for video retry).

16. **Add a Wait node** named `Wait 10min`:
    - Wait 10 minutes for video generation.
    - Connect `Image-to-Video` → `Wait 10min`.

17. **Add an HTTP Request node** named `Get Video`:
    - GET request to PiAPI task status endpoint using task_id.
    - Use PiAPI Key.
    - Connect `Wait 10min` → `Get Video`.

18. **Add an If node** named `Fail check`:
    - Check if `data.status` equals "failed".
    - True branch → `Wait to retry`.
    - False branch → `Match captions with videos`.
    - Connect `Get Video` → `Fail check`.

19. **Add a Wait node** named `Wait to retry`:
    - Wait before retrying video generation.
    - Connect `Fail check` (true) → `Wait to retry`.

20. **Connect `Wait to retry` → `Image-to-Video`** to retry video generation.

21. **Add a Merge node** named `Match captions with videos`:
    - Combine captions and video URLs by position.
    - Connect `Fail check` (false) and `Validate list formatting` (true) → `Match captions with videos`.

22. **Add a Code node** named `List Elements`:
    - Aggregate scene titles, video URLs, and token usage into one item.
    - Connect `Match captions with videos` → `List Elements`.

23. **Add an OpenAI (LangChain) node** named `Generate Script`:
    - Use GPT-4o-mini to create a voiceover script narrating captions.
    - Connect `Validate list formatting` (true) → `Generate Script`.

24. **Add an HTTP Request node** named `Generate voice`:
    - POST to ElevenLabs text-to-speech API with script text.
    - Use ElevenLabs API Key.
    - Connect `Generate Script` → `Generate voice`.

25. **Add a Google Drive node** named `Upload Voice Audio`:
    - Upload voiceover audio to Google Drive folder.
    - Filename: `<Google Sheet ID>-voiceover.mp3`.
    - Connect `Generate voice` → `Upload Voice Audio`.

26. **Add a Google Drive node** named `Set Access Permissions`:
    - Set sharing permissions to "writer" and "anyone" for uploaded audio.
    - Connect `Upload Voice Audio` → `Set Access Permissions`.

27. **Add a Code node** named `List Elements1`:
    - Aggregate sound URLs from uploaded audio.
    - Connect `Set Access Permissions` → `List Elements1`.

28. **Add a Merge node** named `Pair Videos with Audio`:
    - Combine video URLs, captions, and audio URLs by position.
    - Connect `List Elements` and `List Elements1` → `Pair Videos with Audio`.

29. **Add an HTTP Request node** named `Render Final Video`:
    - POST to Creatomate API with template ID and modifications (videos, audio, captions).
    - Use Creatomate API Key.
    - Connect `Pair Videos with Audio` → `Render Final Video`.

30. **Add a Wait node** named `Wait1`:
    - Wait 3 minutes for rendering.
    - Connect `Render Final Video` → `Wait1`.

31. **Add an HTTP Request node** named `Get Final Video`:
    - GET request to Creatomate render status endpoint.
    - Use Creatomate API Key.
    - Connect `Wait1` → `Get Final Video`.

32. **Add an HTTP Request node** named `Get Raw File`:
    - Download rendered video file.
    - Connect `Get Final Video` → `Get Raw File`.

33. **Add a Google Drive node** named `Upload Final Video`:
    - Upload final video to Google Drive folder.
    - Connect `Get Raw File` → `Upload Final Video`.

34. **Add a Google Drive node** named `Set Permissions`:
    - Set sharing permissions on uploaded video file.
    - Connect `Upload Final Video` → `Set Permissions`.

35. **Add a Google Sheets node** named `Update Google Sheet`:
    - Update original row with video metadata, token usage, costs, and status.
    - Connect `Set Permissions` → `Update Google Sheet`.

36. **Add a Discord node** named `Notify me on Discord`:
    - Send notification with video link on completion.
    - Connect `Update Google Sheet` → `Notify me on Discord`.

37. **Add a Write Binary File node** named `Write video`:
    - Write downloaded video file to disk.
    - Connect `Get Raw File` → `Write video`.

38. **Add an OpenAI (LangChain) node** named `Get Audio from Video`:
    - Transcribe audio from video using Whisper.
    - Connect `Write video` → `Get Audio from Video`.

39. **Add an OpenAI (LangChain) node** named `Generate Description for Videos in Tiktok and Instagram`:
    - Generate social media descriptions from transcribed audio.
    - Connect `Get Audio from Video` → `Generate Description for Videos in Tiktok and Instagram`.

40. **Add a Read Binary File node** named `Read Video from Google Drive`:
    - Read video file from disk for upload.
    - Connect `Generate Description for Videos in Tiktok and Instagram` → `Read Video from Google Drive`.

41. **Add HTTP Request nodes** for uploading video and description to each platform (TikTok, Instagram, YouTube, Facebook, LinkedIn):
    - Configure each with upload-post.com API endpoint.
    - Use API key in HTTP header.
    - Connect `Read Video from Google Drive` → each upload node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                       | Context or Link                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Before starting, ensure you have accounts and API keys for all services: n8n, Google Cloud (Sheets & Drive), OpenAI, PiAPI, ElevenLabs, Creatomate, upload-post.com, Discord webhook.                                            | Setup instructions in Sticky Note6 and workflow description.                                                       |
| Use the provided Google Sheet template: [Google Sheet Template](https://docs.google.com/spreadsheets/d/1cjd8p_yx-M-3gWLEd5TargtoB35cW-3y66AOTNMQrrM/edit?usp=sharing) and Creatomate template JSON: [Creatomate Template JSON](https://pastebin.com/c7aMTeLK). |                                                                                                                     |
| Flux and Kling models pricing and API documentation: [Flux API Docs](https://piapi.ai/docs/flux-api/text-to-image?via=n8n), [Kling API Docs](https://piapi.ai/docs/kling-api/create-task?via=n8n)                                |                                                                                                                     |
| ElevenLabs API documentation for text-to-speech: [Eleven Labs API Docs](https://elevenlabs.io/docs/api-reference/text-to-speech/convert)                                                                                       |                                                                                                                     |
| upload-post.com API token generation required for multi-platform uploads.                                                                                                                | See Sticky Note7 for instructions.                                                                                  |
| Discord webhook for notifications: [Discord Webhooks](https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks)                                                                                                  |                                                                                                                     |
| Tested on n8n version 1.81.4.                                                                                                                                                                                                     | See Sticky Note4.                                                                                                   |

---

This document provides a detailed, structured reference to understand, reproduce, and maintain the "Fully Automated AI Video Generation & Multi-Platform Publishing" workflow in n8n. It covers all nodes, their roles, configurations, dependencies, and potential failure points, enabling advanced users and AI agents to work effectively with this automation.