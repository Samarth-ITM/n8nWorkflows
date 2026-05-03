Generate AI YouTube Shorts with Flux, Runway, Eleven Labs and Creatomate

https://n8nworkflows.xyz/workflows/generate-ai-youtube-shorts-with-flux--runway--eleven-labs-and-creatomate-3416


# Generate AI YouTube Shorts with Flux, Runway, Eleven Labs and Creatomate

### 1. Workflow Overview

This workflow automates the end-to-end creation and publishing of AI-generated YouTube Shorts videos based on ideas stored in a Google Sheet. It sequentially processes one video idea at a time, generating image prompts, AI images, animations, audio narration and sound effects, merging all media into a final video, and uploading it to YouTube. The workflow also updates the Google Sheet to track progress and prevent duplicate processing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Idea Extraction**: Triggering the workflow and retrieving a video idea from Google Sheets.
- **1.2 AI Prompt Generation**: Creating text prompts for image generation and audio narration using AI language models.
- **1.3 Image Generation and Animation**: Generating AI images and animating them using external APIs.
- **1.4 Audio Generation and Upload**: Producing voiceover and sound effects, then uploading audio files.
- **1.5 Video Rendering and Download**: Merging video and audio, rendering the final video, and downloading it.
- **1.6 Video Upload and Status Update**: Uploading the final video to YouTube and updating Google Sheets statuses.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Idea Extraction

**Overview:**  
This block initiates the workflow manually and extracts the first video idea marked as "To Do" from the Google Sheet. It sets the idea for further processing.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Grab Idea (Google Sheets)  
- Set Idea (Set)  

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on manual user action.  
  - Configuration: Default manual trigger with no parameters.  
  - Inputs: None  
  - Outputs: Connected to "Grab Idea" node.  
  - Edge Cases: None, but requires manual start.  

- **Grab Idea**  
  - Type: Google Sheets  
  - Role: Retrieves the first row from the Google Sheet where `videoStatus` is "To Do".  
  - Configuration: Reads from a configured Google Sheet with filters applied to select unprocessed ideas.  
  - Inputs: Trigger from manual node.  
  - Outputs: Passes data to "Set Idea".  
  - Edge Cases: Empty or no matching rows; should handle gracefully to avoid errors.  
  - Credential: Google Sheets OAuth2.  

- **Set Idea**  
  - Type: Set  
  - Role: Prepares and formats the extracted idea data for downstream nodes.  
  - Configuration: Maps fields like title, bibleverse, idea, style, caption, videoStatus, publishStatus.  
  - Inputs: Data from "Grab Idea".  
  - Outputs: Passes to "Image Prompt Agent".  
  - Edge Cases: Missing or malformed data fields could cause downstream issues.  

---

#### 2.2 AI Prompt Generation

**Overview:**  
Generates text prompts for image creation and audio narration using AI language models Anthropic Claude or Google Gemini. It processes the idea text to create detailed prompts.

**Nodes Involved:**  
- Anthropic Chat Model (AI Language Model)  
- Image Prompt Agent (LangChain Agent)  
- Remove \n (Code)  
- Set Prompts (Set)  
- Audio Prompt Agent (LangChain Agent)  
- Flash 2.0 (Google Gemini LM Chat)  

**Node Details:**  

- **Anthropic Chat Model**  
  - Type: LangChain Anthropic Chat Model  
  - Role: Generates initial text prompt based on the idea for image generation.  
  - Configuration: Uses Anthropic Claude API with prompt templates.  
  - Inputs: Receives idea data from "Set Idea".  
  - Outputs: Feeds into "Image Prompt Agent".  
  - Edge Cases: API rate limits, authentication errors.  
  - Credential: Anthropic API key.  

- **Image Prompt Agent**  
  - Type: LangChain Agent  
  - Role: Refines and expands the prompt for image generation.  
  - Configuration: Uses LangChain agent with prompt templates and AI model.  
  - Inputs: Output from "Anthropic Chat Model".  
  - Outputs: Passes to "Remove \n".  
  - Edge Cases: Expression evaluation errors, API failures.  

- **Remove \n**  
  - Type: Code (JavaScript)  
  - Role: Cleans the prompt text by removing newline characters for API compatibility.  
  - Configuration: Custom JavaScript code to replace `\n` with spaces or empty strings.  
  - Inputs: Prompt text from "Image Prompt Agent".  
  - Outputs: Passes cleaned prompt to "Set Prompts".  
  - Edge Cases: Unexpected input formats causing code errors.  

- **Set Prompts**  
  - Type: Set  
  - Role: Stores the cleaned prompt and prepares it for image generation API calls.  
  - Configuration: Sets variables for image prompt and audio prompt.  
  - Inputs: Cleaned prompt from "Remove \n".  
  - Outputs: Passes to "Image Generation".  
  - Edge Cases: Missing prompt data.  

- **Audio Prompt Agent**  
  - Type: LangChain Agent  
  - Role: Generates text prompts for audio narration and sound effects.  
  - Configuration: Uses LangChain agent with Google Gemini model ("Flash 2.0").  
  - Inputs: Receives data from "Update Video Status" node (see below).  
  - Outputs: Passes to "Set Audio".  
  - Edge Cases: API errors, malformed prompt generation.  

- **Flash 2.0**  
  - Type: LangChain LM Chat Google Gemini  
  - Role: Provides AI chat completions for audio prompt generation.  
  - Configuration: Uses Google Gemini API with prompt templates.  
  - Inputs: Feeds into "Audio Prompt Agent".  
  - Outputs: Passes generated audio prompt text.  
  - Edge Cases: API limits, auth failures.  

---

#### 2.3 Image Generation and Animation

**Overview:**  
Generates AI images from prompts using Flux AI and animates them using RunwayML APIs.

**Nodes Involved:**  
- Image Generation (HTTP Request)  
- Get Images (HTTP Request)  
- Generate Videos (HTTP Request)  
- Get Videos (HTTP Request)  
- Merge (Merge)  

**Node Details:**  

- **Image Generation**  
  - Type: HTTP Request  
  - Role: Sends the image prompt to Flux AI API to generate an image.  
  - Configuration: POST request with prompt and style parameters.  
  - Inputs: Prompt from "Set Prompts".  
  - Outputs: Passes image metadata to "20 seconds" wait node.  
  - Edge Cases: API timeouts, invalid prompt errors, quota exceeded.  
  - Credential: Flux AI API key via RapidAPI.  

- **Get Images**  
  - Type: HTTP Request  
  - Role: Retrieves generated images from Flux AI after generation request.  
  - Configuration: GET request polling for image readiness.  
  - Inputs: Triggered after "20 seconds" wait node.  
  - Outputs: Passes image URLs to "Generate Videos".  
  - Edge Cases: Image not ready, API errors.  

- **Generate Videos**  
  - Type: HTTP Request  
  - Role: Sends image to RunwayML API to create animated video clips.  
  - Configuration: POST request with image URL and animation style parameters.  
  - Inputs: Image data from "Get Images".  
  - Outputs: Passes video generation job info to "1 minute" wait node.  
  - Edge Cases: API failures, invalid image data.  
  - Credential: RunwayML API key.  

- **Get Videos**  
  - Type: HTTP Request  
  - Role: Polls RunwayML API to check if video animation is ready.  
  - Configuration: GET request to retrieve video URL.  
  - Inputs: Triggered after "1 minute" wait node.  
  - Outputs: Passes video URL to "Merge" node.  
  - Edge Cases: Video not ready, API errors.  

- **Merge**  
  - Type: Merge  
  - Role: Combines video and audio streams for final rendering.  
  - Configuration: Merges video data from "Get Videos" and audio data from "Share File".  
  - Inputs: Video from "Get Videos", audio from "Share File".  
  - Outputs: Passes merged media to "Render Video1".  
  - Edge Cases: Missing inputs, merge conflicts.  

---

#### 2.4 Audio Generation and Upload

**Overview:**  
Generates audio narration and sound effects using ElevenLabs, uploads audio files to Google Drive, and prepares audio for merging.

**Nodes Involved:**  
- Set Audio (Set)  
- Generate Audio (HTTP Request)  
- Upload to Drive (Google Drive)  
- Share File (Google Drive)  

**Node Details:**  

- **Set Audio**  
  - Type: Set  
  - Role: Prepares audio prompt and metadata for audio generation.  
  - Configuration: Sets variables for audio text, voice parameters, and file naming.  
  - Inputs: Output from "Audio Prompt Agent".  
  - Outputs: Passes to "Generate Audio".  
  - Edge Cases: Missing audio prompt text.  

- **Generate Audio**  
  - Type: HTTP Request  
  - Role: Calls ElevenLabs API to generate voiceover and sound effects audio files.  
  - Configuration: POST request with audio prompt and voice parameters.  
  - Inputs: Audio prompt from "Set Audio".  
  - Outputs: Passes audio file data to "Upload to Drive".  
  - Edge Cases: API rate limits, audio generation failures.  
  - Credential: ElevenLabs API key.  

- **Upload to Drive**  
  - Type: Google Drive  
  - Role: Uploads generated audio files to Google Drive for storage and sharing.  
  - Configuration: Uploads audio file with appropriate metadata and folder path.  
  - Inputs: Audio file from "Generate Audio".  
  - Outputs: Passes file metadata to "Share File".  
  - Edge Cases: Drive quota exceeded, permission errors.  
  - Credential: Google Drive OAuth2.  

- **Share File**  
  - Type: Google Drive  
  - Role: Sets sharing permissions on uploaded audio files to allow access during merging.  
  - Configuration: Modifies file permissions to public or specific users.  
  - Inputs: File metadata from "Upload to Drive".  
  - Outputs: Passes shared file info to "Merge".  
  - Edge Cases: Permission errors, sharing failures.  

---

#### 2.5 Video Rendering and Download

**Overview:**  
Renders the final video by merging audio and animation, waits for rendering completion, then downloads the video file.

**Nodes Involved:**  
- Render Video1 (HTTP Request)  
- 25 Seconds (Wait)  
- Download Video (HTTP Request)  

**Node Details:**  

- **Render Video1**  
  - Type: HTTP Request  
  - Role: Sends merged media to Creatomate API to render the final video.  
  - Configuration: POST request with merged video and audio assets.  
  - Inputs: Merged media from "Merge".  
  - Outputs: Passes rendering job info to "25 Seconds" wait node.  
  - Edge Cases: API errors, invalid media data.  
  - Credential: Creatomate API key.  

- **25 Seconds**  
  - Type: Wait  
  - Role: Waits 25 seconds to allow video rendering to complete.  
  - Configuration: Fixed delay of 25 seconds.  
  - Inputs: Triggered after "Render Video1".  
  - Outputs: Passes control to "Download Video".  
  - Edge Cases: Rendering may take longer; fixed wait may cause premature download attempts.  

- **Download Video**  
  - Type: HTTP Request  
  - Role: Downloads the rendered video file from Creatomate.  
  - Configuration: GET request to video URL provided by rendering API.  
  - Inputs: Triggered after wait node.  
  - Outputs: Passes video file data to "Upload Video".  
  - Edge Cases: Download failures, network errors.  

---

#### 2.6 Video Upload and Status Update

**Overview:**  
Uploads the final video to YouTube and updates the Google Sheet to mark the video as created and published.

**Nodes Involved:**  
- Upload Video (YouTube)  
- Update Sheet (Google Sheets)  
- Update Video Status (Google Sheets)  
- Limit (Limit)  

**Node Details:**  

- **Upload Video**  
  - Type: YouTube  
  - Role: Uploads the final video file to YouTube Shorts channel.  
  - Configuration: Uses YouTube API with OAuth2 credentials; sets video metadata like title, description, tags.  
  - Inputs: Video file from "Download Video".  
  - Outputs: Passes upload confirmation to "Update Sheet".  
  - Edge Cases: Authentication errors, upload failures, quota limits.  
  - Credential: YouTube OAuth2.  

- **Update Sheet**  
  - Type: Google Sheets  
  - Role: Updates the `publishStatus` column to "Processed" in the Google Sheet for the uploaded video.  
  - Configuration: Writes back to the same row identified by video ID or row number.  
  - Inputs: Triggered after successful upload.  
  - Outputs: None.  
  - Edge Cases: Write permission errors, concurrency conflicts.  

- **Update Video Status**  
  - Type: Google Sheets  
  - Role: Updates the `videoStatus` column to "Created" to mark the video as processed.  
  - Configuration: Similar to "Update Sheet", but for video creation status.  
  - Inputs: Triggered after "Get Videos" node.  
  - Outputs: Passes data to "Audio Prompt Agent".  
  - Edge Cases: Same as above.  

- **Limit**  
  - Type: Limit  
  - Role: Controls workflow execution concurrency or limits number of processed items.  
  - Configuration: Limits to one item at a time to prevent duplicate processing.  
  - Inputs: From "Get Videos".  
  - Outputs: Passes to "Update Video Status".  
  - Edge Cases: Blocking or throttling issues if misconfigured.  

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                              | Input Node(s)                | Output Node(s)               | Sticky Note                      |
|-------------------------|----------------------------------|----------------------------------------------|------------------------------|------------------------------|---------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Starts workflow manually                      | None                         | Grab Idea                    |                                 |
| Grab Idea               | Google Sheets                    | Retrieves next video idea from sheet          | When clicking ‘Test workflow’ | Set Idea                     |                                 |
| Set Idea                | Set                             | Prepares idea data for prompt generation      | Grab Idea                    | Image Prompt Agent           |                                 |
| Anthropic Chat Model    | LangChain Anthropic Chat Model  | Generates initial image prompt                 | Set Idea                     | Image Prompt Agent           |                                 |
| Image Prompt Agent      | LangChain Agent                 | Refines prompt for image generation            | Anthropic Chat Model         | Remove \n                   |                                 |
| Remove \n               | Code                            | Cleans prompt text                             | Image Prompt Agent           | Set Prompts                 |                                 |
| Set Prompts             | Set                             | Sets cleaned prompts for image generation     | Remove \n                    | Image Generation            |                                 |
| Image Generation        | HTTP Request                    | Calls Flux AI to generate image                | Set Prompts                  | 20 seconds (Wait)           |                                 |
| 20 seconds              | Wait                            | Waits for image generation                      | Image Generation             | Get Images                  |                                 |
| Get Images              | HTTP Request                    | Retrieves generated image                       | 20 seconds                  | Generate Videos             |                                 |
| Generate Videos         | HTTP Request                    | Sends image to RunwayML for animation          | Get Images                   | 1 minute (Wait)             |                                 |
| 1 minute                | Wait                            | Waits for video animation completion           | Generate Videos              | Get Videos                  |                                 |
| Get Videos              | HTTP Request                    | Retrieves animated video                        | 1 minute                    | Merge, Limit                |                                 |
| Limit                   | Limit                           | Controls concurrency                            | Get Videos                   | Update Video Status         |                                 |
| Update Video Status     | Google Sheets                   | Marks video as created                          | Limit                        | Audio Prompt Agent          |                                 |
| Audio Prompt Agent      | LangChain Agent                 | Generates audio narration prompt                | Update Video Status          | Set Audio                   |                                 |
| Flash 2.0               | LangChain Google Gemini LM Chat | AI model for audio prompt generation            | Audio Prompt Agent           | Set Audio                   |                                 |
| Set Audio               | Set                             | Prepares audio generation parameters            | Audio Prompt Agent           | Generate Audio              |                                 |
| Generate Audio          | HTTP Request                    | Calls ElevenLabs to generate audio              | Set Audio                    | Upload to Drive             |                                 |
| Upload to Drive         | Google Drive                    | Uploads audio files                             | Generate Audio               | Share File                  |                                 |
| Share File              | Google Drive                    | Shares audio files for merging                   | Upload to Drive              | Merge                       |                                 |
| Merge                   | Merge                           | Combines video and audio streams                 | Get Videos, Share File       | Render Video1               |                                 |
| Render Video1           | HTTP Request                    | Sends merged media to Creatomate for rendering  | Merge                        | 25 Seconds (Wait)           |                                 |
| 25 Seconds              | Wait                            | Waits for video rendering completion             | Render Video1                | Download Video              |                                 |
| Download Video          | HTTP Request                    | Downloads rendered video                          | 25 Seconds                  | Upload Video                |                                 |
| Upload Video            | YouTube                        | Uploads final video to YouTube                    | Download Video               | Update Sheet                |                                 |
| Update Sheet            | Google Sheets                   | Marks video as published                          | Upload Video                 | None                       |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.  

2. **Add Google Sheets Node ("Grab Idea")**  
   - Configure to read rows where `videoStatus` = "To Do" from your Google Sheet.  
   - Use OAuth2 credentials for Google Sheets.  
   - Connect Manual Trigger → Grab Idea.  

3. **Add Set Node ("Set Idea")**  
   - Map fields from Google Sheets row: title, bibleverse, idea, style, caption, videoStatus, publishStatus.  
   - Connect Grab Idea → Set Idea.  

4. **Add LangChain Anthropic Chat Model Node ("Anthropic Chat Model")**  
   - Configure with Anthropic API key.  
   - Use prompt templates to generate image prompt from idea.  
   - Connect Set Idea → Anthropic Chat Model.  

5. **Add LangChain Agent Node ("Image Prompt Agent")**  
   - Configure with LangChain agent using Anthropic output.  
   - Connect Anthropic Chat Model → Image Prompt Agent.  

6. **Add Code Node ("Remove \n")**  
   - JavaScript code to remove newline characters from prompt text.  
   - Connect Image Prompt Agent → Remove \n.  

7. **Add Set Node ("Set Prompts")**  
   - Store cleaned prompt text for image generation.  
   - Connect Remove \n → Set Prompts.  

8. **Add HTTP Request Node ("Image Generation")**  
   - POST request to Flux AI API with prompt and style parameters.  
   - Use Flux AI API key via RapidAPI credentials.  
   - Connect Set Prompts → Image Generation.  

9. **Add Wait Node ("20 seconds")**  
   - Fixed wait of 20 seconds for image generation.  
   - Connect Image Generation → 20 seconds.  

10. **Add HTTP Request Node ("Get Images")**  
    - GET request to Flux AI to retrieve generated image URL.  
    - Connect 20 seconds → Get Images.  

11. **Add HTTP Request Node ("Generate Videos")**  
    - POST request to RunwayML API to animate image.  
    - Use RunwayML API key credentials.  
    - Connect Get Images → Generate Videos.  

12. **Add Wait Node ("1 minute")**  
    - Fixed wait of 1 minute for video animation.  
    - Connect Generate Videos → 1 minute.  

13. **Add HTTP Request Node ("Get Videos")**  
    - GET request to RunwayML to retrieve animated video URL.  
    - Connect 1 minute → Get Videos.  

14. **Add Limit Node ("Limit")**  
    - Configure to process one item at a time.  
    - Connect Get Videos → Limit.  

15. **Add Google Sheets Node ("Update Video Status")**  
    - Update `videoStatus` to "Created" for processed row.  
    - Connect Limit → Update Video Status.  

16. **Add LangChain Google Gemini LM Chat Node ("Flash 2.0")**  
    - Configure with Google Gemini API key.  
    - Connect Update Video Status → Flash 2.0 (ai_languageModel input).  

17. **Add LangChain Agent Node ("Audio Prompt Agent")**  
    - Configure to generate audio narration prompts.  
    - Connect Flash 2.0 → Audio Prompt Agent.  

18. **Add Set Node ("Set Audio")**  
    - Prepare audio prompt and voice parameters.  
    - Connect Audio Prompt Agent → Set Audio.  

19. **Add HTTP Request Node ("Generate Audio")**  
    - POST request to ElevenLabs API to generate audio files.  
    - Use ElevenLabs API key credentials.  
    - Connect Set Audio → Generate Audio.  

20. **Add Google Drive Node ("Upload to Drive")**  
    - Upload audio files to Google Drive.  
    - Use Google Drive OAuth2 credentials.  
    - Connect Generate Audio → Upload to Drive.  

21. **Add Google Drive Node ("Share File")**  
    - Set sharing permissions on uploaded audio files.  
    - Connect Upload to Drive → Share File.  

22. **Add Merge Node ("Merge")**  
    - Merge video from "Get Videos" and audio from "Share File".  
    - Connect Get Videos → Merge (input 1), Share File → Merge (input 2).  

23. **Add HTTP Request Node ("Render Video1")**  
    - POST request to Creatomate API to render final video.  
    - Use Creatomate API key credentials.  
    - Connect Merge → Render Video1.  

24. **Add Wait Node ("25 Seconds")**  
    - Fixed wait for rendering completion.  
    - Connect Render Video1 → 25 Seconds.  

25. **Add HTTP Request Node ("Download Video")**  
    - GET request to download rendered video.  
    - Connect 25 Seconds → Download Video.  

26. **Add YouTube Node ("Upload Video")**  
    - Upload final video to YouTube Shorts.  
    - Use YouTube OAuth2 credentials.  
    - Connect Download Video → Upload Video.  

27. **Add Google Sheets Node ("Update Sheet")**  
    - Update `publishStatus` to "Processed" in Google Sheet.  
    - Connect Upload Video → Update Sheet.  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Flux AI API documentation and playground available at RapidAPI.                                               | https://rapidapi.com/poorav925/api/ai-text-to-image-generator-flux-free-api/playground/apiendpoint_b28cd8ef-63fe-4242-98e4-908a332770d3 |
| RunwayML API documentation for video animation.                                                              | https://dev.runwayml.com/                                                                           |
| Creatomate website for video/audio merging and rendering.                                                     | https://creatomate.com                                                                              |
| Google Sheets and Google Drive credentials require OAuth2 setup in n8n.                                       | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets                      |
| YouTube API OAuth2 credentials needed for video upload.                                                       | https://developers.google.com/youtube/v3/guides/authentication                                       |
| Workflow requires API keys for Anthropic Claude or Google Gemini for prompt generation, ElevenLabs for audio. | API keys must be securely stored in n8n credentials.                                               |
| Fixed wait nodes (20 seconds, 1 minute, 25 seconds) are used to handle asynchronous API processing delays.    | Consider adjusting wait times based on API response times to optimize workflow speed and reliability.|

---

This document provides a detailed, structured reference for understanding, reproducing, and modifying the "Generate AI YouTube Shorts with Flux, Runway, Eleven Labs and Creatomate" workflow in n8n. It covers all nodes, their roles, configurations, and integration points, enabling advanced users and AI agents to work effectively with this automation pipeline.