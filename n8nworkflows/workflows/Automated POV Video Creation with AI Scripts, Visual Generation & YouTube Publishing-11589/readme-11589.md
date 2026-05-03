Automated POV Video Creation with AI Scripts, Visual Generation & YouTube Publishing

https://n8nworkflows.xyz/workflows/automated-pov-video-creation-with-ai-scripts--visual-generation---youtube-publishing-11589


# Automated POV Video Creation with AI Scripts, Visual Generation & YouTube Publishing

### 1. Workflow Overview

This workflow automates the creation of immersive POV (Point of View) videos using AI-driven script generation, image and video synthesis, sound design, and YouTube publishing. It targets creators and marketers who want to produce cinematic, first-person style videos around given themes or ideas, automating the entire pipeline from idea intake to final video upload.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Idea Selection**: Triggers periodically and fetches the next video idea from a Google Sheets document marked for production.
- **1.2 Scene Title Generation**: Uses AI models to create 5 concise, action-driven POV scene titles representing a “day in the life” narrative sequence.
- **1.3 Detailed Prompt Expansion**: Expands each short scene title into detailed, hyper-realistic first-person image prompts for use in image generation models.
- **1.4 Image Generation**: Converts detailed prompts into portrait-style POV images via an external text-to-image API.
- **1.5 Video Generation**: Transforms generated images into 5-second cinematic POV video clips using a video generation API.
- **1.6 Sound Generation**: Creates ambient soundtracks tailored to each scene’s action using an AI sound generation service, then uploads audio files to Google Drive and sets public sharing.
- **1.7 Merging & Final Video Rendering**: Combines scene titles, video clips, and soundtracks into a structured object and renders a polished final video via a Creatomate template, updating the Google Sheet with the final video link.
- **1.8 Publishing Agent**: Runs on a separate schedule to pick finalized videos ready for publishing, downloads video files, and uploads them to YouTube as unlisted videos using metadata from the sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Idea Selection

- **Overview:**  
  Periodically triggered to fetch the next video idea from a Google Sheets document filtered by a “for production” status. This idea serves as the seed for the entire video creation pipeline.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Google Sheets

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: `Schedule Trigger`  
    - Configured to run at regular intervals (default every minute).  
    - Input: None (time-based trigger)  
    - Output: Triggers Google Sheets node.

  - **Google Sheets**  
    - Type: `Google Sheets`  
    - Configured to read the first row with `production` column set to "for production" in a specific Google Sheet identified by document ID.  
    - Returns video idea and environment prompt for subsequent nodes.  
    - Inputs: Trigger from Schedule Trigger node  
    - Outputs: JSON with video idea and environment prompt.  
    - Potential failures: OAuth errors, sheet access permissions, empty results if no rows match.  
    - Credentials: Google Sheets OAuth2 (specific account).  

---

#### 1.2 Scene Title Generation

- **Overview:**  
  Takes the raw video idea text and generates 5 short, cinematic, action-based POV scene titles that describe a sequential narrative of a day in the life related to the idea.

- **Nodes Involved:**  
  - Generate Titles (`@n8n/n8n-nodes-langchain.chainLlm`)  
  - OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)  
  - Item List Output Parser (`@n8n/n8n-nodes-langchain.outputParserItemList`)

- **Node Details:**  
  - **Generate Titles**  
    - Type: LangChain chain LLM node  
    - Uses a custom prompt to instruct the AI to generate concise 5-10 word action-driven POV prompts without double quotes, focusing on immersive first-person sequences.  
    - Input: Video idea from Google Sheets  
    - Output: Raw AI-generated text containing 5 scene titles.  
    - Potential issues: AI model rate limits, prompt parsing errors.

  - **OpenAI Chat Model**  
    - Type: OpenAI GPT-4 variant chat model  
    - Invoked via LangChain integration to generate the titles.  
    - Credentials: OpenAI API key.

  - **Item List Output Parser**  
    - Type: Output parser that extracts exactly 5 items (scene titles) from the AI response.  
    - Ensures the output is structured as a list for downstream processing.

---

#### 1.3 Detailed Prompt Expansion

- **Overview:**  
  Expands each short scene title into a detailed, cinematic, hyper-realistic first-person image prompt incorporating environment details from the sheet, optimized for use with image-generation models like Flux or MidJourney.

- **Nodes Involved:**  
  - OpenAI (`@n8n/n8n-nodes-langchain.openAi`)

- **Node Details:**  
  - **OpenAI**  
    - Type: OpenAI text generation node  
    - Uses a prompt template that includes: the overall video idea, the short scene title, and environment prompts from Google Sheets.  
    - Outputs a single detailed first-person POV prompt describing foreground (limbs/hands interaction) and background (scenery), emphasizing cinematic immersion and sensory details.  
    - Input: Each short scene title iteratively processed.  
    - Credentials: OpenAI API key.  
    - Potential failures: API errors, prompt formatting issues.

---

#### 1.4 Image Generation

- **Overview:**  
  Converts each detailed prompt into a portrait-style POV image through the PiAPI text-to-image endpoint.

- **Nodes Involved:**  
  - Text-to-Image (`HTTP Request`)  
  - Wait  
  - Get Image (`HTTP Request`)

- **Node Details:**  
  - **Text-to-Image**  
    - Type: HTTP Request  
    - POSTs JSON with prompt, image size (540x960), and model identifier to PiAPI’s txt2img endpoint.  
    - Requires an API key header (X-API-Key).  
    - Input: Detailed prompt from OpenAI node.  
    - Output: Task ID for generation.

  - **Wait**  
    - Type: Wait node  
    - Pauses workflow for 3 minutes, allowing image generation to complete.

  - **Get Image**  
    - Type: HTTP Request  
    - GETs the image generation result using the task ID from Text-to-Image node.  
    - Retrieves the generated image URL.  
    - Potential failures: API timeouts, invalid task ID, rate limits.

---

#### 1.5 Video Generation

- **Overview:**  
  Converts each generated image into a 5-second cinematic POV video clip using Kling video generation via PiAPI.

- **Nodes Involved:**  
  - Image-to-Video (`HTTP Request`)  
  - Wait1  
  - Get Video (`HTTP Request`)

- **Node Details:**  
  - **Image-to-Video**  
    - Type: HTTP Request  
    - POSTs a video generation task with parameters including the image URL, prompt, negative prompt ("bad quality"), duration (5s), and camera control settings to the PiAPI video_generation endpoint.  
    - Requires API key header.  
    - Input: Image URL and prompt.

  - **Wait1**  
    - Type: Wait node  
    - Waits 10 minutes for video processing.

  - **Get Video**  
    - Type: HTTP Request  
    - GETs video generation result via task ID.  
    - Outputs video URL for further processing.  
    - Potential failures: API errors, delayed processing, invalid task ID.

---

#### 1.6 Sound Generation

- **Overview:**  
  Generates ambient, scene-specific soundtracks based on the video idea and scene action using ElevenLabs audio generation API, then uploads the sounds to Google Drive and sets sharing permissions for public access.

- **Nodes Involved:**  
  - Text-to-Sound (`HTTP Request`)  
  - Upload MP3 (`Google Drive`)  
  - Update Access (`Google Drive`)

- **Node Details:**  
  - **Text-to-Sound**  
    - Type: HTTP Request  
    - POSTs text prompt combining the video idea and scene action to ElevenLabs sound-generation endpoint.  
    - Duration fixed at 5 seconds, prompt influence 0.75.  
    - Requires API key header (xi-api-key).  

  - **Upload MP3**  
    - Type: Google Drive node  
    - Uploads generated MP3 file with dynamic naming based on task ID to a specified folder on Google Drive.  
    - Credentials: Google Drive OAuth2.  
    - Potential failures: Upload errors, insufficient permissions.

  - **Update Access**  
    - Type: Google Drive node  
    - Shares the uploaded MP3 publicly (role: writer, type: anyone).  
    - Ensures the file is accessible for Creatomate rendering.

---

#### 1.7 Merging & Final Video Rendering

- **Overview:**  
  Combines all generated elements (scene titles, video URLs, sound URLs) into a structured JSON object and sends it to Creatomate API to produce a polished final video. Updates the Google Sheet with the final video link and marks production as done.

- **Nodes Involved:**  
  - Merge (`Merge`)  
  - List Elements (`Code`)  
  - Render Video (`HTTP Request`)  
  - Final Video Link (`Google Sheets`)

- **Node Details:**  
  - **Merge**  
    - Type: Merge node (combine mode)  
    - Combines 4 inputs: scene titles, 5 video URLs, 5 sound URLs, and others by position.  
    - Ensures all arrays align correctly.

  - **List Elements**  
    - Type: Code node  
    - Processes merged data to produce an object containing arrays of scene titles, video URLs, and sound URLs for rendering.

  - **Render Video**  
    - Type: HTTP Request  
    - POSTs to Creatomate render API with template ID and modifications injecting videos, audio, and text layers.  
    - Authenticated with a Bearer token in headers.  
    - Outputs a final video URL.

  - **Final Video Link**  
    - Type: Google Sheets  
    - Updates the original row with the final video URL, marks production as "done," and moves status to "for publishing".  
    - Potential failures: Sheet update errors, invalid URLs.

---

#### 1.8 Publishing Agent

- **Overview:**  
  Runs on a separate schedule to pick rows marked for publishing, download the final video from the stored URL, and upload it to YouTube with metadata from the sheet, setting the video to unlisted.

- **Nodes Involved:**  
  - Schedule Trigger2  
  - Get Video Link (`Google Sheets`)  
  - Get Video File (`HTTP Request`)  
  - YouTube (`YouTube` node)

- **Node Details:**  
  - **Schedule Trigger2**  
    - Type: Schedule Trigger  
    - Triggers regularly to check for videos marked "for publishing."

  - **Get Video Link**  
    - Type: Google Sheets  
    - Fetches the first row with `publishing` column set to "for publishing."

  - **Get Video File**  
    - Type: HTTP Request  
    - Downloads the video file from the final output URL stored in the sheet.  
    - May require large payload handling.

  - **YouTube**  
    - Type: YouTube node  
    - Uploads the video file with title and description from Google Sheets.  
    - Sets privacy status to unlisted, category ID to 1 (Film & Animation), region code US.  
    - Credentials: YouTube OAuth2.  
    - Potential failures: Upload errors, quota limits, OAuth refresh.

---

### 3. Summary Table

| Node Name           | Node Type                           | Functional Role                          | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                                                                                  |
|---------------------|-----------------------------------|----------------------------------------|------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger                  | Trigger periodic workflow start        | None                         | Google Sheets                  | ## INPUT: Video topic                                                                                                                                                        |
| Google Sheets       | Google Sheets                    | Fetch next idea for production          | Schedule Trigger             | Generate Titles                | ## INPUT – Fetch Video Topic: Gets the next row marked for production. Add ideas → set production = for production.                                                           |
| Generate Titles      | LangChain chain LLM               | Generate 5 short POV scene titles       | Google Sheets                | OpenAI, Merge                  | ## Generate Scene Titles: Creates 5 concise cinematic action prompts representing a day sequence.                                                                            |
| OpenAI Chat Model    | LangChain OpenAI Chat            | AI model for title generation            | Generate Titles (ai_languageModel) | Generate Titles (ai_outputParser) |                                                                                                                                                                              |
| Item List Output Parser | LangChain Output Parser         | Parse AI output into 5 items             | OpenAI Chat Model            | Generate Titles                |                                                                                                                                                                              |
| OpenAI               | OpenAI Text Generation           | Expand short titles into detailed prompts | Generate Titles              | Text-to-Image                 | ## Generate Prompts: Expands each prompt into detailed first-person cinematic descriptions including limbs and environment.                                                  |
| Text-to-Image        | HTTP Request                    | Generate POV images from prompts         | OpenAI                      | Wait                         | ## Generate Images: Uses PiAPI txt2img to create 5 portrait-style POV images.                                                                                                |
| Wait                 | Wait                            | Wait for image generation completion     | Text-to-Image               | Get Image                    |                                                                                                                                                                              |
| Get Image             | HTTP Request                    | Retrieve generated image URL             | Wait                        | Image-to-Video               |                                                                                                                                                                              |
| Image-to-Video       | HTTP Request                    | Generate 5-second POV videos from images | Get Image                   | Wait1                        | ## Generate Videos: Uses Kling via PiAPI to create video clips from images.                                                                                                  |
| Wait1                | Wait                            | Wait for video generation completion     | Image-to-Video              | Get Video                   |                                                                                                                                                                              |
| Get Video             | HTTP Request                    | Retrieve generated video URL             | Wait1                       | Text-to-Sound, Merge          |                                                                                                                                                                              |
| Text-to-Sound        | HTTP Request                    | Generate ambient soundtracks for scenes  | Get Video                   | Upload MP3                   | ## Generate Sounds: Creates ambient soundtracks using ElevenLabs AI based on idea and scene action.                                                                           |
| Upload MP3            | Google Drive                    | Upload generated audio to Google Drive   | Text-to-Sound               | Update Access                |                                                                                                                                                                              |
| Update Access         | Google Drive                    | Set public sharing for audio files       | Upload MP3                  | Merge                       |                                                                                                                                                                              |
| Merge                 | Merge                          | Combine scene titles, video & sound URLs | Generate Titles, Get Video, Update Access | List Elements               | ## Merge & Compile Elements: Combines 5 scene titles, video URLs, and sound URLs into a structured object.                                                                    |
| List Elements         | Code                           | Create structured JSON for rendering     | Merge                       | Render Video                |                                                                                                                                                                              |
| Render Video          | HTTP Request                   | Render final video using Creatomate      | List Elements               | Final Video Link             | ## Render Final Video (Creatomate): Creates final video from template with videos, sounds, and titles.                                                                       |
| Final Video Link      | Google Sheets                  | Update sheet with final video URL        | Render Video                | None                        | ## Update Sheet with Final Link: Marks production done, moves row to publishing.                                                                                            |
| Schedule Trigger2     | Schedule Trigger               | Trigger publishing workflow               | None                       | Get Video Link              | ## Publishing Agent: Runs separately to upload final videos to YouTube.                                                                                                      |
| Get Video Link        | Google Sheets                 | Fetch next video marked for publishing   | Schedule Trigger2           | Get Video File              |                                                                                                                                                                              |
| Get Video File        | HTTP Request                  | Download final video file from cloud     | Get Video Link              | YouTube                    |                                                                                                                                                                              |
| YouTube               | YouTube                      | Upload video to YouTube                   | Get Video File              | None                       |                                                                                                                                                                              |
| Sticky Note(s)        | Sticky Note                  | Various documentation and explanations   | N/A                        | N/A                        | Multiple sticky notes provide contextual explanations for each block, including purpose and configuration guidance. See detailed notes in sections 1 & 2.                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a `Schedule Trigger` node**  
   - Set to execute at your desired interval (default every minute).  
   - Connect output to Google Sheets node.

2. **Add a `Google Sheets` node**  
   - Operation: Read rows  
   - Document ID: Your Google Sheet ID containing video ideas.  
   - Sheet Name: Specific sheet or gid=0.  
   - Filter rows where `production` column = "for production".  
   - Credentials: Set up Google Sheets OAuth2 for access.  
   - Connect output to "Generate Titles" node.

3. **Add a `Generate Titles` node (`@n8n/n8n-nodes-langchain.chainLlm`)**  
   - Input text: Map from the idea field from Google Sheets.  
   - Prompt: Use the custom prompt included in the workflow to generate 5 short POV scene titles with specific instructions.  
   - Connect to OpenAI Chat Model node.

4. **Add `OpenAI Chat Model` node**  
   - Model: GPT-4o-mini or suitable GPT-4 variant.  
   - Credentials: Configure OpenAI API key.  
   - Connect output to `Item List Output Parser`.

5. **Add `Item List Output Parser` node**  
   - Configure to parse 5 items from AI output.  
   - Connect output back to "Generate Titles" node for continuation and to OpenAI expansion node.

6. **Add `OpenAI` node for prompt expansion**  
   - Model: O1-mini or similar.  
   - Prompt: Use detailed template to expand short titles into cinematic first-person POV prompts including environment details.  
   - Input from "Generate Titles" node outputs.  
   - Connect output to Text-to-Image node.

7. **Add `Text-to-Image` HTTP Request node**  
   - Method: POST  
   - URL: `https://api.piapi.ai/api/v1/task`  
   - Body: JSON containing model (`Qubico/flux1-dev`), task_type (`txt2img`), prompt text, width (540), height (960).  
   - Headers: Include `X-API-Key` with your PiAPI key.  
   - Connect output to `Wait` node.

8. **Add `Wait` node**  
   - Duration: 3 minutes.  
   - Connect output to `Get Image` HTTP Request node.

9. **Add `Get Image` HTTP Request node**  
   - Method: GET  
   - URL: `https://api.piapi.ai/api/v1/task/{{task_id}}` (task_id from Text-to-Image response).  
   - Headers: Include `X-API-Key`.  
   - Connect output to `Image-to-Video` node.

10. **Add `Image-to-Video` HTTP Request node**  
    - Method: POST  
    - URL: `https://api.piapi.ai/api/v1/task`  
    - Body: JSON containing model (`kling`), mode (`pro`), task_type (`video_generation`), prompt, negative_prompt, cfg_scale, duration (5 seconds), and the image_url from previous node.  
    - Headers: Include `X-API-Key`.  
    - Connect output to `Wait1` node.

11. **Add `Wait1` node**  
    - Duration: 10 minutes.  
    - Connect output to `Get Video` HTTP Request node.

12. **Add `Get Video` HTTP Request node**  
    - Method: GET  
    - URL: `https://api.piapi.ai/api/v1/task/{{task_id}}` (task_id from Image-to-Video).  
    - Headers: Include `X-API-Key`.  
    - Connect output to `Text-to-Sound` and `Merge` nodes.

13. **Add `Text-to-Sound` HTTP Request node**  
    - Method: POST  
    - URL: `https://api.elevenlabs.io/v1/sound-generation`  
    - Body: JSON with text description (combining idea and scene action), duration_seconds (5), prompt_influence (0.75).  
    - Headers: Include `xi-api-key` with ElevenLabs API key.  
    - Connect output to `Upload MP3`.

14. **Add `Upload MP3` Google Drive node**  
    - Operation: Upload  
    - Folder ID: Target Google Drive folder.  
    - File Name: Use task_id.mp3 or similar.  
    - Credentials: Google Drive OAuth2.  
    - Connect output to `Update Access`.

15. **Add `Update Access` Google Drive node**  
    - Operation: Share file  
    - File ID: From Upload MP3 output.  
    - Permissions: Role writer, type anyone, allow discovery true.  
    - Connect output to `Merge`.

16. **Add `Merge` node**  
    - Mode: Combine by position  
    - Inputs: From Generate Titles, Get Video, Update Access nodes.  
    - Connect output to `List Elements`.

17. **Add `List Elements` Code node**  
    - JavaScript code to aggregate arrays of scene titles, video URLs, and sound URLs into one object.  
    - Connect output to `Render Video`.

18. **Add `Render Video` HTTP Request node**  
    - Method: POST  
    - URL: `https://api.creatomate.com/v1/renders`  
    - Body: JSON with template ID and modifications injecting audio, video, and text layers using merged data and Google Sheets idea.  
    - Headers: Authorization bearer token and content-type.  
    - Connect output to `Final Video Link`.

19. **Add `Final Video Link` Google Sheets node**  
    - Operation: Update row  
    - Update columns: Set `production` to "done", `publishing` to "for publishing", `final_output` to video URL from Render Video.  
    - Credentials: Google Sheets OAuth2.  
    - No outputs.

20. **Add `Schedule Trigger2` node**  
    - Set to trigger on a separate schedule for publishing.  
    - Connect output to `Get Video Link`.

21. **Add `Get Video Link` Google Sheets node**  
    - Reads first row with `publishing` = "for publishing".  
    - Credentials: Google Sheets OAuth2.  
    - Connect output to `Get Video File`.

22. **Add `Get Video File` HTTP Request node**  
    - Method: GET  
    - URL: From the `final_output` field in the sheet.  
    - Connect output to `YouTube`.

23. **Add `YouTube` node**  
    - Operation: Upload video  
    - Title: From idea field  
    - Description: From caption field  
    - Privacy status: Unlisted  
    - Category: 1 (Film & Animation)  
    - Region: US  
    - Credentials: YouTube OAuth2.  
    - No outputs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                      | Context or Link                                                                                               |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| The workflow employs a multi-stage AI pipeline integrating OpenAI (GPT-4 variants) for prompt generation and expansion, PiAPI for image and video generation, and ElevenLabs for audio. | Ensure you have valid API keys and credentials for OpenAI, PiAPI, ElevenLabs, Google Drive, and YouTube APIs. |
| Creatomate template ID `5aafffa3-6adc-4a2f-90dc-c91a80d2136a` is used for final video rendering. Customize it in Creatomate as needed.                                           | Creatomate documentation: https://docs.creatomate.com/                                                       |
| Google Sheets acts as the central data hub for idea management, status tracking, and metadata storage.                                                                            | Structure your sheet with columns: id, idea, caption, production, environment_prompt, publishing, final_output. |
| The publishing agent runs independently to separate video generation from YouTube upload, allowing manual control over the publishing stage.                                     | YouTube API quota and OAuth management should be monitored for smooth operation.                             |
| Sticky notes in the workflow provide detailed instructions and explanations at each stage to aid maintenance and troubleshooting.                                                | Review sticky notes carefully within n8n editor for inline guidance.                                         |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated n8n workflow. All data processed is legal and publicly sourced. The workflow respects current content policies and contains no illegal or offensive material.