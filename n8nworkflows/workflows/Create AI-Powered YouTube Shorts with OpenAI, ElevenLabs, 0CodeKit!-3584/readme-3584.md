Create AI-Powered YouTube Shorts with OpenAI, ElevenLabs, 0CodeKit!

https://n8nworkflows.xyz/workflows/create-ai-powered-youtube-shorts-with-openai--elevenlabs--0codekit--3584


# Create AI-Powered YouTube Shorts with OpenAI, ElevenLabs, 0CodeKit!

### 1. Workflow Overview

This workflow automates the creation of AI-powered YouTube Shorts by orchestrating multiple AI and media services. It generates a video script, converts it to speech, segments the audio, creates matching visuals, and compiles everything into a polished short video ready for upload. The workflow is designed for content creators, marketers, and hobbyists who want to produce engaging short-form videos quickly and with minimal manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Script Generation**: Starts the workflow manually and generates a video script with title and description using OpenAI.
- **1.2 Text-to-Speech and Audio Upload**: Converts the script to speech via ElevenLabs and uploads the audio to Cloudinary.
- **1.3 Audio Segmentation and Image Prompt Generation**: Segments the audio into 6-second clips and generates AI prompts for images corresponding to each segment.
- **1.4 Image and Video Generation**: Generates images and short video clips for each segment using Replicate models.
- **1.5 Video Compilation and Rendering**: Aggregates clips and audio, creates a JSON configuration for Creatomate, renders the final video, and retrieves the output.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Script Generation

- **Overview**: Initiates the workflow manually and uses OpenAI to generate a structured script, including intro, base content, CTA, title, and description with hashtags.
- **Nodes Involved**:  
  - When clicking ‘Test workflow’  
  - Ideator  
  - Script  
  - Script Generator  

- **Node Details**:

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Triggers "Ideator" node.  
    - Edge Cases: None typical; user must manually trigger.  

  - **Ideator**  
    - Type: OpenAI (Langchain)  
    - Role: Generates the video script, title, and description based on user input.  
    - Configuration: Uses GPT-4o-mini model with a prompt template to create intro, base, CTA, title, and description with hashtags.  
    - Inputs: Trigger output  
    - Outputs: JSON with script segments and metadata.  
    - Edge Cases: API key invalid or rate limits; prompt failures; incomplete script generation.  

  - **Script**  
    - Type: Set  
    - Role: Prepares or formats the script data for downstream nodes.  
    - Configuration: Likely sets variables or restructures data from Ideator output.  
    - Inputs: Output from Ideator  
    - Outputs: Feeds into Script Generator.  
    - Edge Cases: Expression errors if data structure changes.  

  - **Script Generator**  
    - Type: HTTP Request  
    - Role: Calls ElevenLabs API to convert the script text into speech audio (MP3) with timestamps.  
    - Configuration: HTTP POST with ElevenLabs TTS endpoint, sending script text, voice parameters, and receiving audio plus timestamps.  
    - Inputs: Script data from Script node  
    - Outputs: Audio file data and timestamps  
    - Edge Cases: API authentication errors, audio length limits, network timeouts.

---

#### 2.2 Text-to-Speech and Audio Upload

- **Overview**: Uploads the generated audio to Cloudinary for storage and accessibility by later nodes.
- **Nodes Involved**:  
  - Upload to Cloudinary  

- **Node Details**:

  - **Upload to Cloudinary**  
    - Type: HTTP Request  
    - Role: Uploads the MP3 audio file to Cloudinary cloud storage.  
    - Configuration: HTTP POST to Cloudinary upload API with audio file data and credentials.  
    - Inputs: Audio data from Script Generator  
    - Outputs: Cloudinary URL and metadata for the uploaded audio.  
    - Edge Cases: Upload failures, invalid credentials, file size limits.

---

#### 2.3 Audio Segmentation and Image Prompt Generation

- **Overview**: Segments the uploaded audio into 6-second clips with transcriptions, then generates AI prompts for images matching each segment.
- **Nodes Involved**:  
  - HTTP Request (audio segmentation Python script)  
  - Split Out  
  - Image Prompter  

- **Node Details**:

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Runs a Python script that segments the audio into 6-second chunks and produces transcription chunks with timestamps.  
    - Configuration: POST request with audio URL and segmentation parameters; returns segmented data.  
    - Inputs: Cloudinary audio URL from Upload to Cloudinary  
    - Outputs: Segmented audio chunks with text  
    - Edge Cases: Script errors, segmentation inaccuracies, timeout.  

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the segmented audio chunks into individual items for parallel processing.  
    - Configuration: Default splitting on array items.  
    - Inputs: Segmentation output array  
    - Outputs: Individual segment items  
    - Edge Cases: Empty or malformed arrays.  

  - **Image Prompter**  
    - Type: OpenAI (Langchain)  
    - Role: Generates a visual prompt for each audio segment, styled in CNSTLL style for simple, animatable images.  
    - Configuration: Uses GPT-4o-mini with prompt templates to create image descriptions.  
    - Inputs: Each segment text from Split Out  
    - Outputs: Image prompt strings  
    - Edge Cases: API rate limits, prompt failures, overly complex prompts causing generation errors.

---

#### 2.4 Image and Video Generation

- **Overview**: Generates images from prompts and creates short video clips from those images using Replicate models.
- **Nodes Involved**:  
  - Request Image  
  - Wait  
  - Get Image  
  - Request Video  
  - Wait for Video  
  - Get Video  

- **Node Details**:

  - **Request Image**  
    - Type: HTTP Request  
    - Role: Sends image prompt to Replicate’s Flux-Cinestill model to generate a 9:16 image.  
    - Configuration: POST request with prompt and model parameters.  
    - Inputs: Image prompt from Image Prompter  
    - Outputs: Job ID or immediate response for image generation  
    - Edge Cases: API key issues, prompt complexity, generation failures.  

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow to allow image generation to complete.  
    - Configuration: Waits for a webhook or fixed duration.  
    - Inputs: After Request Image  
    - Outputs: Triggers Get Image node.  
    - Edge Cases: Timeout if generation takes too long.  

  - **Get Image**  
    - Type: HTTP Request  
    - Role: Retrieves the generated image from Replicate using job ID.  
    - Configuration: GET request to Replicate API.  
    - Inputs: After Wait  
    - Outputs: Image URL or data  
    - Edge Cases: Job not ready, API errors.  

  - **Request Video**  
    - Type: HTTP Request  
    - Role: Sends the generated image to Replicate’s Minimax Video-01 model to create a 5-second video clip.  
    - Configuration: POST request with image URL and model parameters.  
    - Inputs: Image URL from Get Image  
    - Outputs: Job ID for video generation  
    - Edge Cases: API errors, invalid image URL.  

  - **Wait for Video**  
    - Type: Wait  
    - Role: Pauses workflow until video generation completes.  
    - Configuration: Waits for webhook or fixed delay.  
    - Inputs: After Request Video  
    - Outputs: Triggers Get Video node.  
    - Edge Cases: Timeout, webhook failure.  

  - **Get Video**  
    - Type: HTTP Request  
    - Role: Retrieves the generated video clip from Replicate.  
    - Configuration: GET request with job ID.  
    - Inputs: After Wait for Video  
    - Outputs: Video clip URL or data  
    - Edge Cases: Job incomplete, API errors.

---

#### 2.5 Video Compilation and Rendering

- **Overview**: Aggregates all video clips and audio, creates a JSON configuration for Creatomate, submits for rendering, waits for completion, and retrieves the final video.
- **Nodes Involved**:  
  - Aggregate  
  - Merge  
  - Create Editor JSON  
  - Set JSON Variable  
  - Editor  
  - Rendering  
  - Get Final Video  

- **Node Details**:

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Collects all video clips into a single dataset for compilation.  
    - Configuration: Aggregates items from Get Video node.  
    - Inputs: Multiple video clip items  
    - Outputs: Single aggregated array  
    - Edge Cases: Missing clips, partial data.  

  - **Merge**  
    - Type: Merge  
    - Role: Combines aggregated video clips with audio metadata from Cloudinary.  
    - Configuration: Merge mode set to combine by index or key.  
    - Inputs: Aggregate output and Upload to Cloudinary output  
    - Outputs: Combined data for video editing.  
    - Edge Cases: Mismatched array lengths, missing data.  

  - **Create Editor JSON**  
    - Type: HTTP Request  
    - Role: Calls a service or runs a script to generate Creatomate JSON configuration describing the video timeline, clips, transitions, and audio sync.  
    - Configuration: POST request with merged data, specifying fade durations, resolution, and clip timings.  
    - Inputs: Merge output  
    - Outputs: JSON config for Creatomate editor.  
    - Edge Cases: JSON formatting errors, API failures.  

  - **Set JSON Variable**  
    - Type: Set  
    - Role: Stores or formats the JSON config for the next node.  
    - Configuration: Sets JSON data in a variable.  
    - Inputs: Create Editor JSON output  
    - Outputs: Prepared JSON for submission.  
    - Edge Cases: Data corruption or formatting issues.  

  - **Editor**  
    - Type: HTTP Request  
    - Role: Submits the JSON config to Creatomate API to start rendering the video.  
    - Configuration: POST request with JSON payload and credentials.  
    - Inputs: Set JSON Variable output  
    - Outputs: Rendering job ID or status.  
    - Edge Cases: API key errors, invalid JSON, rate limits.  

  - **Rendering**  
    - Type: Wait  
    - Role: Waits for the rendering process to complete, typically up to 70 seconds.  
    - Configuration: Wait node with webhook or fixed delay.  
    - Inputs: Editor output  
    - Outputs: Triggers Get Final Video node.  
    - Edge Cases: Timeout, webhook failure.  

  - **Get Final Video**  
    - Type: HTTP Request  
    - Role: Retrieves the final rendered video URL from Creatomate.  
    - Configuration: GET request with rendering job ID.  
    - Inputs: Rendering output  
    - Outputs: Final video URL ready for download or upload.  
    - Edge Cases: Rendering failure, API errors.

---

### 3. Summary Table

| Node Name               | Node Type                   | Functional Role                              | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                         |
|-------------------------|-----------------------------|----------------------------------------------|-------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger             | Starts workflow manually                      | None                          | Ideator                        |                                                                                                   |
| Ideator                 | OpenAI (Langchain)           | Generates script, title, description          | When clicking ‘Test workflow’ | Script                        | Replace `<user-query>` with your video topic (e.g., "Fitness tips for beginners")                  |
| Script                  | Set                         | Prepares script data                           | Ideator                       | Script Generator               |                                                                                                   |
| Script Generator        | HTTP Request                | Converts script to speech via ElevenLabs      | Script                        | Upload to Cloudinary, HTTP Request | Configure ElevenLabs API key; handles text-to-speech conversion                                  |
| Upload to Cloudinary    | HTTP Request                | Uploads audio to Cloudinary                    | Script Generator              | Merge                         | Configure Cloudinary credentials                                                                 |
| HTTP Request            | HTTP Request                | Segments audio into 6-second clips            | Script Generator              | Split Out                     | Runs Python script for segmentation                                                               |
| Split Out               | Split Out                   | Splits audio segments for parallel processing | HTTP Request                  | Image Prompter                |                                                                                                   |
| Image Prompter          | OpenAI (Langchain)           | Generates image prompts per segment            | Split Out                    | Request Image                 | Use CNSTLL style for simple, animatable visuals                                                   |
| Request Image           | HTTP Request                | Requests image generation from Replicate      | Image Prompter                | Wait                         | Configure Replicate API key                                                                       |
| Wait                    | Wait                        | Waits for image generation completion         | Request Image                 | Get Image                    |                                                                                                   |
| Get Image               | HTTP Request                | Retrieves generated image                       | Wait                         | Request Video                |                                                                                                   |
| Request Video           | HTTP Request                | Requests video generation from image           | Get Image                    | Wait for Video               | Configure Replicate API key                                                                       |
| Wait for Video          | Wait                        | Waits for video generation completion          | Request Video                | Get Video                   |                                                                                                   |
| Get Video               | HTTP Request                | Retrieves generated video clip                  | Wait for Video               | Aggregate                   |                                                                                                   |
| Aggregate               | Aggregate                   | Aggregates all video clips                       | Get Video                    | Merge                       |                                                                                                   |
| Merge                   | Merge                       | Combines audio and video data                    | Aggregate, Upload to Cloudinary | Create Editor JSON           |                                                                                                   |
| Create Editor JSON      | HTTP Request                | Creates Creatomate JSON config for video editing | Merge                       | Set JSON Variable            | Adjust fade duration and resolution here                                                          |
| Set JSON Variable       | Set                         | Stores JSON config                               | Create Editor JSON            | Editor                      |                                                                                                   |
| Editor                  | HTTP Request                | Submits video editing job to Creatomate         | Set JSON Variable             | Rendering                   | Configure Creatomate API key                                                                      |
| Rendering               | Wait                        | Waits for video rendering completion             | Editor                       | Get Final Video             | Rendering can take up to 70 seconds                                                               |
| Get Final Video         | HTTP Request                | Retrieves final rendered video URL                | Rendering                    | None                       |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Position: Start of workflow  
   - No parameters needed.  

2. **Add OpenAI Node "Ideator"**  
   - Type: OpenAI (Langchain)  
   - Connect from Manual Trigger  
   - Configure with OpenAI API credentials (GPT-4o-mini model)  
   - Set prompt to generate: intro (40-70 chars), base (280-350 chars), CTA (55 chars), title, and description with hashtags.  
   - Output: JSON with script and metadata.  

3. **Add Set Node "Script"**  
   - Connect from Ideator  
   - Configure to format or extract script data as needed for TTS.  

4. **Add HTTP Request Node "Script Generator"**  
   - Connect from Script  
   - Configure POST request to ElevenLabs TTS API  
   - Use ElevenLabs API key credentials  
   - Send script text, voice parameters, request MP3 audio with timestamps  
   - Outputs audio data and timestamps.  

5. **Add HTTP Request Node "Upload to Cloudinary"**  
   - Connect from Script Generator  
   - Configure POST to Cloudinary upload API  
   - Use Cloudinary credentials  
   - Upload MP3 audio file  
   - Output: Cloudinary audio URL.  

6. **Add HTTP Request Node "HTTP Request" (Audio Segmentation)**  
   - Connect from Script Generator (parallel to Upload to Cloudinary)  
   - Configure POST request to Python segmentation script endpoint  
   - Send Cloudinary audio URL and segmentation parameters (default 6 seconds)  
   - Output: segmented audio chunks with transcriptions.  

7. **Add Split Out Node "Split Out"**  
   - Connect from HTTP Request (segmentation)  
   - Configure to split array of segments into individual items.  

8. **Add OpenAI Node "Image Prompter"**  
   - Connect from Split Out  
   - Configure with OpenAI API credentials (GPT-4o-mini)  
   - Prompt to generate simple, animatable image prompts in CNSTLL style for each segment.  

9. **Add HTTP Request Node "Request Image"**  
   - Connect from Image Prompter  
   - Configure POST to Replicate Flux-Cinestill model API  
   - Use Replicate API key  
   - Send image prompt, request 9:16 image generation.  

10. **Add Wait Node "Wait"**  
    - Connect from Request Image  
    - Configure to wait for image generation completion (webhook or fixed delay).  

11. **Add HTTP Request Node "Get Image"**  
    - Connect from Wait  
    - Configure GET request to Replicate API to retrieve image URL.  

12. **Add HTTP Request Node "Request Video"**  
    - Connect from Get Image  
    - Configure POST to Replicate Minimax Video-01 model API  
    - Use Replicate API key  
    - Send image URL to generate 5-second video clip.  

13. **Add Wait Node "Wait for Video"**  
    - Connect from Request Video  
    - Configure to wait for video generation completion (webhook or fixed delay).  

14. **Add HTTP Request Node "Get Video"**  
    - Connect from Wait for Video  
    - Configure GET request to Replicate API to retrieve video clip URL.  

15. **Add Aggregate Node "Aggregate"**  
    - Connect from Get Video  
    - Configure to aggregate all video clips into one array.  

16. **Add Merge Node "Merge"**  
    - Connect from Aggregate (main input) and Upload to Cloudinary (secondary input)  
    - Configure to merge video clips with audio metadata.  

17. **Add HTTP Request Node "Create Editor JSON"**  
    - Connect from Merge  
    - Configure POST request to service or script that creates Creatomate JSON config  
    - Include fade durations, video resolution, clip timings.  

18. **Add Set Node "Set JSON Variable"**  
    - Connect from Create Editor JSON  
    - Store the JSON configuration for submission.  

19. **Add HTTP Request Node "Editor"**  
    - Connect from Set JSON Variable  
    - Configure POST request to Creatomate API to start rendering  
    - Use Creatomate API key  

20. **Add Wait Node "Rendering"**  
    - Connect from Editor  
    - Configure wait for rendering completion (webhook or fixed delay, ~70 seconds).  

21. **Add HTTP Request Node "Get Final Video"**  
    - Connect from Rendering  
    - Configure GET request to Creatomate API to retrieve final video URL.  

22. **Save and test the workflow**  
    - Replace placeholder inputs (e.g., `<user-query>`) in Ideator node with your video topic.  
    - Verify outputs at each step, especially API responses and media URLs.  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Use specific, concise queries in the Ideator node for better script quality (e.g., "Minimalist home decor ideas"). | Prompt optimization tip for better AI output.                                                   |
| Video rendering in Creatomate may take up to 70 seconds; adjust wait node accordingly.                         | Performance consideration for video rendering.                                                  |
| Avoid complex or human-centric prompts in Image Prompter to reduce AI generation errors.                       | Best practice for stable image generation with Replicate.                                       |
| Add trending hashtags like #YouTubeShorts, #AI to enhance video description reach.                             | Social media optimization tip.                                                                   |
| For persistent issues, contact n8n Community Forum or API providers (OpenAI, ElevenLabs, Replicate, Creatomate). | Support and troubleshooting resources.                                                          |
| Workflow uses multiple APIs: OpenAI (GPT-4o-mini), ElevenLabs (Multilingual v2), Replicate (Flux-Cinestill, Minimax Video-01), Cloudinary, Creatomate. | External services integration overview.                                                         |
| Setup instructions and example outputs are included in the workflow description for quick start.              | Reference for initial configuration and expected results.                                       |

---

This document provides a comprehensive reference to understand, reproduce, and maintain the "Create AI-Powered YouTube Shorts with OpenAI, ElevenLabs, 0CodeKit!" workflow in n8n. It covers all nodes, their roles, configurations, dependencies, and potential failure points, enabling advanced users and automation agents to work effectively with the workflow.