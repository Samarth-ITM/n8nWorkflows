Create Viral Social Media Videos with FalAI Flux/Kling and GPT-4 Automation

https://n8nworkflows.xyz/workflows/create-viral-social-media-videos-with-falai-flux-kling-and-gpt-4-automation-6918


# Create Viral Social Media Videos with FalAI Flux/Kling and GPT-4 Automation

### 1. Workflow Overview

This workflow automates the creation of a 60-second viral social media video featuring a narrated, cinematic story based on a scientific or cosmic theme. It employs AI models and external APIs to generate ideas, scripts, images, animations, voiceovers, and finally assembles and uploads a polished video.

The workflow is organized into three main logical blocks:

**1.1 Ideation & Scripting**  
- Generate a viral video idea with title, hashtags, and setting using GPT-4.1.  
- Create a detailed 12-segment scripted narration matching a 60-second runtime.  
- Combine narration segments into a full voiceover paragraph for audio generation.

**1.2 Visual Generation Loop**  
- For each of the 12 script segments:  
  - Generate a photorealistic still image from the segment prompt using Fal.ai Flux.  
  - Analyze the image through OpenAI Vision to produce an animation prompt.  
  - Animate the image into a 5-second video clip using Fal.ai Kling.

**1.3 Assembly & Finalization**  
- Merge the 12 generated video clips sequentially into a single video using Fal.ai FFMPEG API.  
- Generate the voiceover audio from the combined script text via ElevenLabs TTS API.  
- Combine the voiceover audio with the merged video to produce the final video.  
- Upload the completed video file to Google Drive.  
- Record metadata and status updates in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Ideation & Scripting

**Overview:**  
Generates a viral video idea and a 12-segment documentary-style narration script, then combines the segments into a full script paragraph suitable for voiceover generation.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Create New Idea1 (OpenAI GPT-4.1)  
- Organise idea, caption etc1 (Google Sheets)  
- Generate Timed Script (OpenAI GPT-4.1)  
- Generating scenes1 (OpenAI GPT-4.1)  
- Unbundle prompts1 (Code)  
- Get Full Voiceover Prompt (Code)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Start the workflow manually.  
  - Outputs: Starts the idea generation.

- **Create New Idea1**  
  - Type: OpenAI GPT-4.1 (Langchain node)  
  - Role: Generates one immersive, viral video idea in JSON format with title, hashtags, brief setting, and status.  
  - Key Config: Custom prompt enforcing strict rules on output format, length, style, and hashtags.  
  - Potential Failures: OpenAI API limit exceeded, response formatting errors.

- **Organise idea, caption etc1**  
  - Type: Google Sheets (Append)  
  - Role: Saves the generated idea, title, hashtags, and environment prompt to a Google Sheet for tracking and later reference.  
  - Inputs: JSON from Create New Idea1.  
  - Edge cases: Google Sheets API quota limits, permission errors.

- **Generate Timed Script**  
  - Type: OpenAI GPT-4.1 (Langchain)  
  - Role: Creates exactly 12 short narration segments (5 seconds each) matching the generated idea, formatted as JSON array with start, end, and text fields.  
  - Key Config: Prompt enforces documentary style, engaging hook, and call to action.  
  - Potential Failures: API timeouts, malformed JSON output.

- **Generating scenes1**  
  - Type: OpenAI GPT-4.1 (Langchain)  
  - Role: For each segment's text, generates a cinematic image prompt in a specified style and format for visual generation.  
  - Key Config: Comprehensive prompt rules to avoid fantasy, movement verbs, and enforce scientific plausibility.  
  - Outputs: Array of 12 scene prompt objects.

- **Unbundle prompts1**  
  - Type: Code  
  - Role: Splits the array of scene prompts into individual items for processing each scene separately.  
  - Input: JSON array of scenes from Generating scenes1.  
  - Edge cases: Missing or malformed prompts array.

- **Get Full Voiceover Prompt**  
  - Type: Code  
  - Role: Combines the 12 narration segments into one continuous paragraph for voiceover synthesis.  
  - Input: Segments array from Generate Timed Script node.  
  - Edge cases: Missing segments, non-string text fields.

---

#### 2.2 Visual Generation Loop

**Overview:**  
Processes each of the 12 segments to generate an image, create an animation prompt, and produce a 5-second video clip from the image and prompt.

**Nodes Involved:**  
- Create Images1 (HTTP Request to Fal.ai Flux)  
- Get Images1 (HTTP Request to Fal.ai Flux for result)  
- Video Prompts1 (OpenAI Vision)  
- Create Video1 (HTTP Request to Fal.ai Kling Video)  
- Get Video1 (HTTP Request to Fal.ai Kling Video for result)  
- Wait1, Wait11 (Wait nodes for timing)

**Node Details:**

- **Create Images1**  
  - Type: HTTP Request  
  - Role: Sends text prompt to Fal.ai Flux for fast text-to-image generation.  
  - Key Config: POST to `fal-ai/flux/schnell` with prompt and image size.  
  - Edge cases: API key missing, rate limits, malformed prompt.

- **Wait1**  
  - Type: Wait  
  - Role: Pause to allow image generation to complete before fetching results. Duration: 240 seconds.  
  - Edge cases: Insufficient wait time may cause missing results.

- **Get Images1**  
  - Type: HTTP Request  
  - Role: Retrieve generated images from Fal.ai Flux using request ID.  
  - Edge cases: Timeout if image isn't ready, API auth errors.

- **Video Prompts1**  
  - Type: OpenAI Vision (Langchain)  
  - Role: Analyzes generated image to produce a cinematic animation prompt matching the scene style.  
  - Key Config: Uses image URL as input, GPT-4 model with high detail.  
  - Edge cases: Vision API quota, invalid image URLs.

- **Create Video1**  
  - Type: HTTP Request  
  - Role: Sends the image and animation prompt to Fal.ai Kling Video API to create a 5-second animated clip.  
  - Key Config: POST with JSON body including prompt, image_url, duration, aspect ratio, negative prompt, and CFG scale.  
  - Edge cases: API limits, invalid image URL or prompt.

- **Wait11**  
  - Type: Wait  
  - Role: Wait 700 seconds for video rendering to complete before fetching result.  
  - Edge cases: Too short wait time causes missing or partial videos.

- **Get Video1**  
  - Type: HTTP Request  
  - Role: Fetches the rendered video clip from Fal.ai Kling Video API by request ID.  
  - Edge cases: Network errors, missing video, auth errors.

- **Merge1**  
  - Type: Merge  
  - Role: Combines outputs of Get Video1 and Get Images1 nodes by position for downstream processing.

- **List Elements1**  
  - Type: Code  
  - Role: Extracts an array of video URLs from merged video clips for final assembly.

---

#### 2.3 Assembly & Finalization

**Overview:**  
Combines the 12 video clips into one video, generates voiceover audio from the full script, merges audio and video, uploads the final video to Google Drive, and updates Google Sheets with final data.

**Nodes Involved:**  
- Create Final Video (HTTP Request to Fal.ai FFMPEG)  
- Wait3 (Wait)  
- Get Final video (HTTP Request)  
- Get Full Voiceover Prompt (Code) [already described in Ideation block]  
- Create Voiceover (HTTP Request to ElevenLabs TTS)  
- Wait9, Wait10 (Wait nodes)  
- Get Voiceover, Get Voice and Video (HTTP Request to ElevenLabs and Fal.ai)  
- Combine Voice and Video (HTTP Request to Fal.ai FFMPEG Compose)  
- URL to file (HTTP Request)  
- Upload file to drive (Google Drive node)  
- Final Video (Longest) (Google Sheets update)

**Node Details:**

- **List Elements1** (described above) outputs video URLs needed for final composition.

- **Create Final Video**  
  - Type: HTTP Request  
  - Role: Sends all 12 video URLs with timestamps and durations to Fal.ai FFMPEG compose endpoint to create one seamless video.  
  - Edge cases: API errors, missing URLs, incomplete video clips.

- **Wait3**  
  - Type: Wait  
  - Role: Pause to allow final video assembly to complete (200 seconds).  
  - Edge cases: Insufficient wait time.

- **Get Final video**  
  - Type: HTTP Request  
  - Role: Retrieves the final assembled video metadata including URL from Fal.ai.  
  - Edge cases: API errors, video not ready.

- **Get Full Voiceover Prompt** (from Ideation block) prepares the full narration text.

- **Create Voiceover**  
  - Type: HTTP Request  
  - Role: Sends the full narration paragraph to ElevenLabs TTS turbo-v2.5 endpoint to generate a voiceover audio request.  
  - Key Config: Voice ID specified, stability, similarity boost, speed parameters.  
  - Edge cases: API rate limits, invalid parameters.

- **Wait9**  
  - Type: Wait  
  - Role: Pause 280 seconds allowing voiceover audio generation.  
  - Edge cases: Too short wait.

- **Get Voiceover**  
  - Type: HTTP Request  
  - Role: Fetch the generated voiceover audio from ElevenLabs API using request ID.  
  - Edge cases: Audio not ready, API errors.

- **Combine Voice and Video**  
  - Type: HTTP Request  
  - Role: Sends video URL and audio URL to Fal.ai FFMPEG compose endpoint to merge audio and video into one file.  
  - Edge cases: Sync issues, API errors.

- **Wait10**  
  - Type: Wait  
  - Role: Pause 280 seconds to allow combined video/audio processing.

- **Get Voice and Video**  
  - Type: HTTP Request  
  - Role: Fetches the final video with audio combined from Fal.ai.  
  - Edge cases: Missing file, API errors.

- **URL to file**  
  - Type: HTTP Request  
  - Role: Downloads the final video file for upload to Google Drive.  
  - Edge cases: Network issues.

- **Upload file to drive**  
  - Type: Google Drive  
  - Role: Uploads the final video file to Google Drive root folder using the video title as filename.  
  - Edge cases: Drive quota exceeded, permission denied.

- **Final Video (Longest)**  
  - Type: Google Sheets (Update)  
  - Role: Updates the Google Sheet row for the idea with the final video URL and marks production as done.  
  - Edge cases: Sheet row not found, API limits.

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                            | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                          |
|---------------------------|----------------------------------|--------------------------------------------|------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Starts workflow execution                   |                              | Create New Idea1              |                                                                                                                      |
| Create New Idea1           | OpenAI GPT-4.1 (Langchain)       | Generate viral video idea JSON              | When clicking ‘Execute workflow’ | Organise idea, caption etc1    |                                                                                                                      |
| Organise idea, caption etc1 | Google Sheets Append             | Save idea and metadata                       | Create New Idea1             | Generate Timed Script          |                                                                                                                      |
| Generate Timed Script      | OpenAI GPT-4.1 (Langchain)       | Generate 12 narration segments               | Organise idea, caption etc1  | Generating scenes1             |                                                                                                                      |
| Generating scenes1          | OpenAI GPT-4.1 (Langchain)       | Generate cinematic image prompts             | Generate Timed Script        | Unbundle prompts1              |                                                                                                                      |
| Unbundle prompts1          | Code                             | Split scene prompts into individual items   | Generating scenes1           | Create Images1                |                                                                                                                      |
| Create Images1             | HTTP Request (Fal.ai Flux)       | Generate photorealistic images               | Unbundle prompts1            | Wait1                        |                                                                                                                      |
| Wait1                     | Wait                             | Pause to allow image generation              | Create Images1               | Get Images1                  |                                                                                                                      |
| Get Images1                | HTTP Request                     | Retrieve generated images                     | Wait1                       | Merge1, Video Prompts1        |                                                                                                                      |
| Video Prompts1             | OpenAI Vision                    | Generate animation prompt from image         | Get Images1                 | Create Video1                |                                                                                                                      |
| Create Video1              | HTTP Request (Fal.ai Kling Video) | Animate image into 5s video clip              | Video Prompts1              | Wait11                       |                                                                                                                      |
| Wait11                    | Wait                             | Pause to allow video rendering                | Create Video1               | Get Video1                   |                                                                                                                      |
| Get Video1                 | HTTP Request                     | Fetch animated video clip                      | Wait11                      | Merge1                       |                                                                                                                      |
| Merge1                    | Merge (combine by position)      | Combine video and image data                  | Get Video1, Get Images1      | List Elements1               |                                                                                                                      |
| List Elements1             | Code                             | Extract array of video URLs                    | Merge1                      | Create Final Video           |                                                                                                                      |
| Create Final Video         | HTTP Request (Fal.ai FFMPEG)     | Stitch 12 clips into one final video          | List Elements1              | Wait3                        |                                                                                                                      |
| Wait3                     | Wait                             | Pause for final video assembly                 | Create Final Video          | Get Final video              |                                                                                                                      |
| Get Final video            | HTTP Request                     | Retrieve final video metadata                  | Wait3                       | Get Full Voiceover Prompt    |                                                                                                                      |
| Get Full Voiceover Prompt  | Code                             | Combine narration segments into paragraph     | Get Final video             | Create Voiceover             |                                                                                                                      |
| Create Voiceover           | HTTP Request (ElevenLabs TTS)    | Generate voiceover audio                        | Get Full Voiceover Prompt   | Wait9                        |                                                                                                                      |
| Wait9                     | Wait                             | Pause for voiceover generation                  | Create Voiceover            | Get Voiceover                |                                                                                                                      |
| Get Voiceover              | HTTP Request                     | Fetch voiceover audio                            | Wait9                       | Combine Voice and Video      |                                                                                                                      |
| Combine Voice and Video    | HTTP Request (Fal.ai FFMPEG)     | Merge final video and audio                      | Get Voiceover               | Wait10                       |                                                                                                                      |
| Wait10                    | Wait                             | Pause for combined video/audio processing       | Combine Voice and Video     | Get Voice and Video          |                                                                                                                      |
| Get Voice and Video        | HTTP Request                     | Retrieve combined video/audio file               | Wait10                      | URL to file, Final Video (Longest) |                                                                                                                      |
| URL to file               | HTTP Request                     | Download final video file                         | Get Voice and Video         | Upload file to drive         |                                                                                                                      |
| Upload file to drive       | Google Drive                    | Upload final video to Google Drive                | URL to file                 |                               |                                                                                                                      |
| Final Video (Longest)      | Google Sheets Update             | Update final video URL and status in Sheet        | Get Voice and Video         |                               |                                                                                                                      |
| Sticky Note (multiple)     | Sticky Note                     | Provide visual guidance and explanations         |                              |                               | See individual sticky notes in workflow for context (see section 5).                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Create OpenAI GPT-4 Node: Create New Idea1**  
   - Model: GPT-4.1  
   - Role: Generate viral video idea JSON with title, hashtags, environment prompt.  
   - Prompt: Use detailed instructions to produce one idea under 13 words, with hashtags and status "for production".  
   - Ensure JSON output is enabled.

3. **Google Sheets Append Node: Organise idea, caption etc1**  
   - Connect from Create New Idea1.  
   - Append idea details (id, idea text, caption, production status, environment prompt) to Sheet1.  
   - Configure Google Sheets OAuth2 credentials.

4. **OpenAI GPT-4 Node: Generate Timed Script**  
   - Connect from Organise idea, caption etc1.  
   - Prompt to generate exactly 12 narration segments (5s each) as JSON array with start, end, text.  
   - Use the idea title and idea text in the prompt.  
   - Enable JSON output.

5. **OpenAI GPT-4 Node: Generating scenes1**  
   - Connect from Generate Timed Script.  
   - For each narration segment’s text, generate a cinematic image prompt following strict rules for style and content.  
   - Output JSON array of 12 scene prompts.

6. **Code Node: Unbundle prompts1**  
   - Connect from Generating scenes1.  
   - Split the array of scene prompts into individual items for parallel processing.

7. **HTTP Request Node: Create Images1**  
   - Connect from Unbundle prompts1.  
   - POST to `https://queue.fal.run/fal-ai/flux/schnell` with JSON body containing prompt and image size "portrait_16_9".  
   - Set header Authorization with your Fal.ai API Key.

8. **Wait Node: Wait1**  
   - Connect from Create Images1.  
   - Wait 240 seconds for image generation.

9. **HTTP Request Node: Get Images1**  
   - Connect from Wait1.  
   - GET request to `https://queue.fal.run/fal-ai/flux/requests/{{request_id}}` with Authorization header.  
   - Retrieve generated images.

10. **OpenAI Vision Node: Video Prompts1**  
    - Connect from Get Images1.  
    - Analyze fetched image URL to generate cinematic animation prompt.  
    - Use GPT-4 model with high detail.

11. **HTTP Request Node: Create Video1**  
    - Connect from Video Prompts1.  
    - POST to `https://queue.fal.run/fal-ai/kling-video/v1.6/standard/image-to-video` with JSON body including prompt, image URL, duration 5s, aspect ratio 9:16, negative prompt, and CFG scale.  
    - Include Authorization header.

12. **Wait Node: Wait11**  
    - Connect from Create Video1.  
    - Wait 700 seconds for video rendering.

13. **HTTP Request Node: Get Video1**  
    - Connect from Wait11.  
    - GET request to `https://queue.fal.run/fal-ai/kling-video/requests/{{request_id}}` with Authorization header.  
    - Retrieve rendered video clip.

14. **Merge Node: Merge1**  
    - Combine outputs of Get Video1 and Get Images1 by position for further processing.

15. **Code Node: List Elements1**  
    - Extract an array of all video URLs from merged items.

16. **HTTP Request Node: Create Final Video**  
    - Connect from List Elements1.  
    - POST to `https://queue.fal.run/fal-ai/ffmpeg-api/compose` with JSON body listing the 12 video URLs with timestamps and durations for concatenation.  
    - Authorization header required.

17. **Wait Node: Wait3**  
    - Wait 200 seconds for final video assembly.

18. **HTTP Request Node: Get Final video**  
    - Fetch the final merged video information.

19. **Code Node: Get Full Voiceover Prompt**  
    - Combine the 12 narration segments into one paragraph for voiceover.

20. **HTTP Request Node: Create Voiceover**  
    - POST to `https://queue.fal.run/fal-ai/elevenlabs/tts/turbo-v2.5` with JSON body including text (full paragraph), voice ID, stability, similarity boost, speed.  
    - Includes Authorization header.

21. **Wait Node: Wait9**  
    - Wait 280 seconds for voiceover generation.

22. **HTTP Request Node: Get Voiceover**  
    - Retrieve voiceover audio file using request ID.

23. **HTTP Request Node: Combine Voice and Video**  
    - POST to Fal.ai FFMPEG compose endpoint with video and audio URLs to merge them.

24. **Wait Node: Wait10**  
    - Wait 280 seconds for combined processing.

25. **HTTP Request Node: Get Voice and Video**  
    - Retrieve combined final video with audio.

26. **HTTP Request Node: URL to file**  
    - Download final video file for upload.

27. **Google Drive Node: Upload file to drive**  
    - Upload final video to Google Drive root folder with filename from idea title.

28. **Google Sheets Update Node: Final Video (Longest)**  
    - Update the idea row with final video URL and mark production status as done.

**Credential Setup:**  
- **OpenAI API Key** configured for OpenAI nodes.  
- **Fal.ai API Key** manually inserted in all HTTP Request nodes calling fal.run endpoints in the Authorization header.  
- **Google OAuth2 Credentials** for Sheets and Drive nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                                                                    |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| ⚠️ CRITICAL SETUP: Must add OpenAI, Google, and Fal.ai credentials; manually update Fal.ai HTTP nodes Authorization headers with your key to avoid workflow failure.                                                                 | Sticky Note titled "Critical Setup" in workflow                                                                                                  |
| 🚀 AI Viral Video Factory Overview: Automates viral 60-second narrated videos from a single idea using GPT-4 and Fal.ai models.                                                                                                      | Sticky Note "Workflow Overview"                                                                                                                  |
| 🎬 Scene Generation Loop: Runs 12 times to generate images, animation prompts, and 5-second video clips using Fal.ai Flux and Kling with OpenAI Vision.                                                                             | Sticky Note "Scene Generation Loop"                                                                                                             |
| 🔧 How to Customize & Run: Edit the Create New Idea1 prompt to change video topic; execute workflow to generate one full video.                                                                                                     | Sticky Note "How to Customize"                                                                                                                  |
| Fal.ai API Reference: Uses `fal-ai/flux/schnell` (text-to-image), `fal-ai/kling-video` (image-to-video animation), and `fal-ai/ffmpeg-api` (video/audio manipulation).                                                               | Sticky Note "Fal.ai API Reference"                                                                                                              |
| The video script segments are designed for a documentary style, with calls to action at the end. The narration is split into 12 segments of 5 seconds each, ensuring smooth pacing and coherent storytelling.                          | From OpenAI prompts and Code nodes                                                                                                              |
| The workflow relies on multiple wait nodes to accommodate asynchronous processing delays of external APIs; adjusting wait times may affect reliability and completeness of generated media.                                         | Based on wait node parameters                                                                                                                   |
| Video aspect ratio is set to 9:16 (portrait) suitable for social media platforms like TikTok, Instagram Reels, and YouTube Shorts.                                                                                                 | From Create Video1 node parameters                                                                                                              |
| The voiceover uses ElevenLabs' TTS turbo-v2.5 voice with parameters tuned for stability and similarity boost to achieve natural narration.                                                                                        | From Create Voiceover node parameters                                                                                                           |

---

This document provides a detailed, structured reference for understanding, reproducing, and modifying the "Create Viral Social Media Videos with FalAI Flux/Kling and GPT-4 Automation" workflow in n8n.