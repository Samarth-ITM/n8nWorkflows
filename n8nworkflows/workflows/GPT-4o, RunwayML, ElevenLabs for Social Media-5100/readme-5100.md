GPT-4o, RunwayML, ElevenLabs for Social Media

https://n8nworkflows.xyz/workflows/gpt-4o--runwayml--elevenlabs-for-social-media-5100


# GPT-4o, RunwayML, ElevenLabs for Social Media

---

## 1. Workflow Overview

This workflow automates the generation of short social media videos featuring touristic destinations, leveraging advanced AI models and services to create text, images, videos, and audio. It is designed for content creators and social media managers who want to produce engaging multimedia posts from simple text prompts with minimal manual intervention.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Trigger & Data Retrieval:** Initiates workflow manually or on schedule and fetches pending video ideas from a Google Sheets document.
- **1.2 Text Content Preparation:** Processes the idea content and style, preparing prompts for AI models.
- **1.3 Image Prompt Generation & Image Creation:** Uses GPT-4o to generate detailed image prompts, then creates images via Flux API.
- **1.4 Video Generation:** Converts generated images into short videos using RunwayML API.
- **1.5 Audio Generation:** Crafts sound prompts and produces matching audio clips with ElevenLabs.
- **1.6 Video Rendering, Download, and Upload:** Combines videos and audio into a final video using Creatomate API, downloads it, uploads to Google Drive, and shares the file.
- **1.7 Google Sheets Update:** Updates the status and links in Google Sheets to reflect the workflow progress.

---

## 2. Block-by-Block Analysis

### 2.1 Input Trigger & Data Retrieval

**Overview:**  
Starts the workflow either manually or via a schedule trigger and retrieves a pending video idea from a Google Sheet where video status is "ToDo".

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Schedule Trigger  
- Grab Idea (Google Sheets)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow on demand for testing.  
  - *Configuration:* No parameters.  
  - *Input/Output:* No input; outputs a trigger event.  
  - *Failure Modes:* None typical; manual start.  

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Periodically triggers workflow based on configured interval (default unspecified).  
  - *Configuration:* Interval set with empty object (default to run every minute?).  
  - *Input/Output:* No input; outputs a trigger event.  
  - *Failure Modes:* Scheduling misconfiguration or n8n downtime.

- **Grab Idea**  
  - *Type:* Google Sheets (Read)  
  - *Role:* Fetches the first matching row where `video status` == "ToDo".  
  - *Configuration:*  
    - Sheet: "Tour Agent Video Prompts" (documentId: 16MeId2XFWgj2JwS3h83QUcK_ygNy_3rwRj8XymhosLE)  
    - Sheet tab: gid=0 ("Sheet1")  
    - Filter: `video status` equals "ToDo"  
    - Return first match only.  
  - *Credential:* Google Sheets OAuth2 (named "Google Sheets account 5").  
  - *Input/Output:* No input; outputs matched row JSON including columns like `content1` to `content4`, `style`, `row_number`.  
  - *Failure Modes:* OAuth token expiration, API limits, no matching rows found (workflow might proceed but with empty data).

---

### 2.2 Text Content Preparation

**Overview:**  
Consolidates multiple content fields into an array and prepares style information for downstream AI prompt generation.

**Nodes Involved:**  
- Set Content  
- Split Out

**Node Details:**

- **Set Content**  
  - *Type:* Set  
  - *Role:* Combines content columns (`content1` to `content4`) into a single `content` array and sets `style` as a string.  
  - *Configuration:*  
    - Sets `content` as an array of four content fields.  
    - Sets `style` from the sheet's `style` column.  
  - *Input:* From "Grab Idea" node.  
  - *Output:* JSON object with `content` array and `style` string.  
  - *Failure Modes:* Missing content fields or style might cause downstream prompt issues.

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits the `content` array into individual items for parallel processing.  
  - *Configuration:* Field to split out: "content".  
  - *Input:* From "Set Content" node.  
  - *Output:* Multiple output items, each with one content element.  
  - *Failure Modes:* Empty or malformed `content` array.

---

### 2.3 Image Prompt Generation & Image Creation

**Overview:**  
Generates detailed image prompts for each content piece using GPT-4o, cleans prompt outputs, then creates images using Flux API.

**Nodes Involved:**  
- GPT 4o  
- Image Prompt Agent  
- Remove \n (Code)  
- Set Prompts  
- Generate Image  
- 90 seconds (Wait)  
- Get Images

**Node Details:**

- **GPT 4o**  
  - *Type:* LangChain OpenAI Chat Model node (OpenAI GPT-4o)  
  - *Role:* Generates natural language image prompts from content.  
  - *Configuration:* Model set to "gpt-4o", no special options.  
  - *Credentials:* OpenAI API (named "OpenAi account 5").  
  - *Input:* Content text from "Split Out".  
  - *Output:* Generated prompt text.  
  - *Failure Modes:* API rate limiting, auth errors, input formatting issues.

- **Image Prompt Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Refines and structures image prompts with style context for advanced image models.  
  - *Configuration:*  
    - Text input: combines content and style variables.  
    - System message instructs detailed, family friendly image prompt generation with examples.  
    - Prompt type: define.  
  - *Input:* Output from GPT 4o and style context from "Set Content".  
  - *Output:* Refined image prompt text.  
  - *Failure Modes:* Expression evaluation errors, unexpected input format.

- **Remove \n**  
  - *Type:* Code (JavaScript)  
  - *Role:* Removes all newline characters from the prompt output to produce clean prompt strings.  
  - *Key Code:* Replaces `\n` and actual newlines globally in `json.output`.  
  - *Input:* From "Image Prompt Agent".  
  - *Output:* Cleaned prompt strings.  
  - *Failure Modes:* Missing `output` field causes no changes; otherwise robust.

- **Set Prompts**  
  - *Type:* Set  
  - *Role:* Assigns the cleaned prompt string to a field named `prompts` for the next image generation step.  
  - *Input:* From "Remove \n".  
  - *Output:* JSON with `prompts` string.  
  - *Failure Modes:* Missing input `output` property.

- **Generate Image**  
  - *Type:* HTTP Request  
  - *Role:* Calls Flux API to generate images based on the prompt.  
  - *Configuration:*  
    - URL: https://api.piapi.ai/api/v1/task  
    - Method: POST  
    - Body: JSON with model "Qubico/flux1-dev", task_type "txt2img", input prompt, width 540, height 960.  
    - Authentication: HTTP Header Auth (Flux API key).  
  - *Input:* `prompts` string.  
  - *Output:* Task ID for generated images.  
  - *Failure Modes:* API limits, auth failure, invalid prompts.

- **90 seconds**  
  - *Type:* Wait  
  - *Role:* Waits 90 seconds to allow image generation to complete asynchronously.  
  - *Input:* From "Generate Image".  
  - *Output:* Passes data forward after delay.  
  - *Failure Modes:* Timeout too short or too long affects workflow timing.

- **Get Images**  
  - *Type:* HTTP Request  
  - *Role:* Polls Flux API to retrieve generated image URLs using task ID.  
  - *Configuration:*  
    - URL template: https://api.piapi.ai/api/v1/task/{{task_id}}  
    - Auth: HTTP Header Auth (Flux API key).  
  - *Input:* Task ID from "Generate Image".  
  - *Output:* JSON with image URLs.  
  - *Failure Modes:* Task not ready, API errors.

---

### 2.4 Video Generation

**Overview:**  
Generates short videos from images and text prompts using RunwayML API, waits for rendering completion, and merges results.

**Nodes Involved:**  
- Generate Videos  
- 2 minutes (Wait)  
- Get Videos  
- Merge  
- Limit  
- Split Out Parts

**Node Details:**

- **Generate Videos**  
  - *Type:* HTTP Request  
  - *Role:* Sends image URL and prompt text to RunwayML's image-to-video API.  
  - *Configuration:*  
    - URL: https://api.dev.runwayml.com/v1/image_to_video  
    - Method: POST  
    - Body includes promptImage (image URL), model "gen3a_turbo", ratio "768:1280", duration 5 seconds, promptText.  
    - Headers for authorization and API version.  
  - *Input:* Image URLs from "Get Images".  
  - *Output:* Task ID or video generation response.  
  - *Failure Modes:* API auth errors, invalid parameters, rate limits.

- **2 minutes**  
  - *Type:* Wait  
  - *Role:* Waits 2 minutes for video generation to complete asynchronously.  
  - *Input:* From "Generate Videos".  
  - *Output:* Passes data forward.  
  - *Failure Modes:* Timeout mismatch with actual video generation time.

- **Get Videos**  
  - *Type:* HTTP Request  
  - *Role:* Polls RunwayML for the status and URLs of generated videos using task ID.  
  - *Configuration:*  
    - URL template: https://api.dev.runwayml.com/v1/tasks/{{id}}  
    - Headers as above.  
  - *Input:* Task ID from "Generate Videos".  
  - *Output:* Video URLs and metadata.  
  - *Failure Modes:* Task not ready, API errors.

- **Merge**  
  - *Type:* Merge  
  - *Role:* Combines multiple streams of video URLs (possibly from several images/videos) into one item.  
  - *Configuration:* Mode: combine all inputs.  
  - *Input:* From "Get Videos" and "Share File".  
  - *Output:* Single combined item with all URLs.  
  - *Failure Modes:* Missing inputs, empty arrays.

- **Limit**  
  - *Type:* Limit  
  - *Role:* Limits the number of items passed downstream (default 1).  
  - *Input:* From "Merge".  
  - *Output:* Limited number of items.  
  - *Failure Modes:* None typical.

- **Split Out Parts**  
  - *Type:* Code (JavaScript)  
  - *Role:* Extracts URLs from the merged output arrays and creates a single array with metadata for rendering.  
  - *Logic:* Iterates over multiple items and their `output` arrays, collecting URLs with sourceId and createdAt.  
  - *Input:* From "Merge".  
  - *Output:* Single JSON item with array `urls`.  
  - *Failure Modes:* Unexpected input structure causes error.

---

### 2.5 Audio Generation

**Overview:**  
Generates a sound prompt based on the style, creates a matching audio clip via ElevenLabs, and uploads the audio file to Google Drive.

**Nodes Involved:**  
- Video Status (Google Sheets Update)  
- Sound Agent  
- Set Audio  
- Generate Audio  
- Upload to Drive  
- Share File

**Node Details:**

- **Video Status**  
  - *Type:* Google Sheets (Update)  
  - *Role:* Updates the `video status` column to "Created" for the current row to mark workflow progress.  
  - *Configuration:* Matches row by `row_number` from "Grab Idea".  
  - *Input:* From "Limit".  
  - *Output:* Confirmation of update.  
  - *Failure Modes:* API auth issues, row mismatch.

- **Sound Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Generates a 1-2 sentence sound prompt describing ambiance and mood based on the style.  
  - *Configuration:*  
    - Input text: Style from "Grab Idea".  
    - System message guides detailed sound prompt creation.  
  - *Input:* From "Video Status".  
  - *Output:* Sound prompt text.  
  - *Failure Modes:* API errors, prompt generation failures.

- **Set Audio**  
  - *Type:* Set  
  - *Role:* Trims and sets the sound prompt text for audio generation.  
  - *Input:* From "Sound Agent".  
  - *Output:* JSON with `audio` string.  
  - *Failure Modes:* Missing input output field.

- **Generate Audio**  
  - *Type:* HTTP Request  
  - *Role:* Calls ElevenLabs API to generate audio from the sound prompt.  
  - *Configuration:*  
    - URL: https://api.elevenlabs.io/v1/sound-generation  
    - Method: POST  
    - Body parameters: text (sound prompt), duration 20 seconds.  
    - Auth: HTTP Header Auth (ElevenLabs).  
  - *Input:* `audio` prompt string.  
  - *Output:* Audio file metadata including IDs.  
  - *Failure Modes:* API limits, invalid auth, malformed prompt.

- **Upload to Drive**  
  - *Type:* Google Drive (Upload)  
  - *Role:* Uploads the generated audio file to a specified Google Drive folder.  
  - *Configuration:*  
    - File name: `style`.mp3  
    - Folder ID: AudioFile folder (1oGmtM9LXXKfblOQgyJiLCAr3DphJuAFY)  
    - Drive: My Drive  
  - *Input:* Audio file from "Generate Audio".  
  - *Output:* Google Drive file metadata including shareable link.  
  - *Failure Modes:* OAuth errors, folder access denied.

- **Share File**  
  - *Type:* Google Drive (Share)  
  - *Role:* Sets sharing permissions on the uploaded audio file to "anyone with link can view".  
  - *Configuration:* Role: reader, type: anyone, allow file discovery.  
  - *Input:* File ID from "Upload to Drive".  
  - *Output:* Sharing confirmation.  
  - *Failure Modes:* Permission errors, invalid file ID.

---

### 2.6 Video Rendering, Download, and Upload

**Overview:**  
Renders the final video by combining generated videos and audio, waits for processing, downloads the rendered video, and updates Google Sheets with the video link.

**Nodes Involved:**  
- Render Video  
- 25 Seconds (Wait)  
- Download Video  
- Update Sheet

**Node Details:**

- **Render Video**  
  - *Type:* HTTP Request  
  - *Role:* Calls Creatomate API to render a video template with combined video clips, audio track, and text overlays.  
  - *Configuration:*  
    - Template ID: fixed UUID "62cb0638-8d36-4602-900a-0e331e45de27"  
    - Modifications include:  
      - Video sources: URLs from the merged video URLs array (indexes 0 to 3).  
      - Audio source: Google Drive webContentLink from uploaded audio.  
      - Text overlays: content1 to content4 from "Grab Idea".  
    - Auth: Bearer token in Authorization header (placeholder "Bearer credentials").  
  - *Input:* Merged video URLs and audio file link.  
  - *Output:* Render task response containing video URL.  
  - *Failure Modes:* API authorization, invalid URLs, template errors.

- **25 Seconds**  
  - *Type:* Wait  
  - *Role:* Waits 25 seconds for video rendering to complete.  
  - *Input:* From "Render Video".  
  - *Output:* Passes data forward.  
  - *Failure Modes:* Inadequate waiting time.

- **Download Video**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the rendered video file from the URL provided by Creatomate.  
  - *Configuration:* URL taken from previous node's output.  
  - *Input:* From "25 Seconds" node.  
  - *Output:* Video binary data or URL reference.  
  - *Failure Modes:* Download failures, broken links.

- **Update Sheet**  
  - *Type:* Google Sheets (Update)  
  - *Role:* Updates the Google Sheet row with the new video link and status fields (`video status` = "Created", `publish status` = "Processed").  
  - *Configuration:* Matches by `video status` (possibly should be `row_number`?), updates `videoLink`.  
  - *Input:* From "Download Video".  
  - *Output:* Confirmation.  
  - *Failure Modes:* Incorrect matching criteria, auth errors.

---

## 3. Summary Table

| Node Name                 | Node Type                       | Functional Role                                  | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                          |
|---------------------------|--------------------------------|-------------------------------------------------|----------------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                 | Manual start of workflow                         | —                          | Grab Idea                   |                                                                                                    |
| Schedule Trigger          | Schedule Trigger                | Scheduled start of workflow                      | —                          | Grab Idea                   |                                                                                                    |
| Grab Idea                | Google Sheets                  | Fetches next video idea from sheet              | When clicking ‘Test workflow’, Schedule Trigger | Set Content                 |                                                                                                    |
| Set Content              | Set                            | Prepares content array and style string         | Grab Idea                  | Split Out                   |                                                                                                    |
| Split Out                | Split Out                      | Splits content array into items                  | Set Content                | GPT 4o                      |                                                                                                    |
| GPT 4o                   | LangChain OpenAI Chat Model    | Generates initial image prompts                   | Split Out                  | Image Prompt Agent          |                                                                                                    |
| Image Prompt Agent       | LangChain Agent                | Refines image prompts with style context         | GPT 4o                     | Remove \n                   | # Generate Image Prompts                                                                            |
| Remove \n                | Code                           | Cleans newlines in prompts                        | Image Prompt Agent          | Set Prompts                 |                                                                                                    |
| Set Prompts              | Set                            | Sets prompt text for image generation             | Remove \n                  | Generate Image              |                                                                                                    |
| Generate Image           | HTTP Request                   | Calls Flux API to generate images                 | Set Prompts                | 90 seconds                 | # Generate Images                                                                                   |
| 90 seconds               | Wait                           | Waits for image generation completion             | Generate Image             | Get Images                  |                                                                                                    |
| Get Images               | HTTP Request                   | Retrieves generated image URLs                     | 90 seconds                 | Generate Videos             |                                                                                                    |
| Generate Videos          | HTTP Request                   | Sends images and prompts to RunwayML for video    | Get Images                 | 2 minutes                   | # Generate Videos                                                                                   |
| 2 minutes                | Wait                           | Waits for video generation to complete            | Generate Videos            | Get Videos                  |                                                                                                    |
| Get Videos               | HTTP Request                   | Polls for generated video URLs                     | 2 minutes                  | Merge, Limit                |                                                                                                    |
| Merge                    | Merge                          | Combines multiple video URL arrays                  | Get Videos, Share File     | Split Out Parts, Limit      |                                                                                                    |
| Limit                    | Limit                          | Limits processing to one item                       | Merge                      | Video Status                |                                                                                                    |
| Video Status             | Google Sheets                  | Updates video status to "Created"                   | Limit                      | Sound Agent                 |                                                                                                    |
| Sound Agent              | LangChain Agent                | Generates sound prompts from style                  | Video Status               | Set Audio                   | # Generate Audio                                                                                   |
| Set Audio                | Set                            | Sets cleaned audio prompt text                       | Sound Agent                | Generate Audio              |                                                                                                    |
| Generate Audio           | HTTP Request                   | Calls ElevenLabs to generate audio                   | Set Audio                  | Upload to Drive             |                                                                                                    |
| Upload to Drive          | Google Drive                   | Uploads generated audio file                         | Generate Audio             | Share File                  |                                                                                                    |
| Share File               | Google Drive                   | Shares uploaded audio with public link              | Upload to Drive            | Merge                      |                                                                                                    |
| Split Out Parts          | Code                           | Extracts and combines video URLs for rendering      | Merge                      | Render Video                |                                                                                                    |
| Render Video             | HTTP Request                   | Calls Creatomate to render final video              | Split Out Parts            | 25 Seconds                  | # Render & Upload                                                                                   |
| 25 Seconds               | Wait                           | Waits for video rendering completion                 | Render Video               | Download Video              |                                                                                                    |
| Download Video           | HTTP Request                   | Downloads the rendered video file                     | 25 Seconds                 | Update Sheet                |                                                                                                    |
| Update Sheet             | Google Sheets                  | Updates the sheet with video link and statuses       | Download Video             | —                           |                                                                                                    |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’" with no parameters.  
   - Add a **Schedule Trigger** node named "Schedule Trigger" with default interval (modify as needed).  

2. **Fetch Video Idea from Google Sheets**  
   - Add a **Google Sheets** node named "Grab Idea".  
   - Configure credentials with your Google Sheets OAuth2.  
   - Set operation to read rows, select sheet by document ID: `16MeId2XFWgj2JwS3h83QUcK_ygNy_3rwRj8XymhosLE`.  
   - Sheet tab: gid=0 (Sheet1).  
   - Filter rows where `video status` equals "ToDo".  
   - Return only the first matched row.  
   - Connect both trigger nodes to "Grab Idea".  

3. **Prepare Content and Style**  
   - Add a **Set** node "Set Content".  
   - Assign fields:  
     - `content` as array of `content1` to `content4` from sheet data.  
     - `style` as string from sheet data.  
   - Connect "Grab Idea" to "Set Content".  
   - Add a **Split Out** node "Split Out" set to split on `content`.  
   - Connect "Set Content" to "Split Out".  

4. **Generate Image Prompts**  
   - Add a **LangChain OpenAI Chat** node named "GPT 4o".  
   - Use OpenAI API credentials for GPT-4o model.  
   - Connect "Split Out" to "GPT 4o".  
   - Add a **LangChain Agent** node "Image Prompt Agent".  
     - Configure system message with prompt instructions (as summarized in 2.3).  
     - Input text: combine content and style variables.  
   - Connect "GPT 4o" to "Image Prompt Agent".  
   - Add a **Code** node "Remove \n" to strip newline characters from `output`.  
   - Connect "Image Prompt Agent" to "Remove \n".  
   - Add a **Set** node "Set Prompts" assigning `prompts` to the cleaned output from previous node.  
   - Connect "Remove \n" to "Set Prompts".  

5. **Generate Images with Flux API**  
   - Add an **HTTP Request** node "Generate Image".  
   - Configure POST to `https://api.piapi.ai/api/v1/task` with JSON body specifying model "Qubico/flux1-dev", task_type "txt2img", prompt from `prompts`, width 540, height 960.  
   - Add HTTP Header Auth credentials for Flux API.  
   - Connect "Set Prompts" to "Generate Image".  
   - Add a **Wait** node "90 seconds" for 90 seconds.  
   - Connect "Generate Image" to "90 seconds".  
   - Add an **HTTP Request** node "Get Images" to poll image generation status with URL `https://api.piapi.ai/api/v1/task/{{ $json.data.task_id }}`.  
   - Use same Flux API credentials.  
   - Connect "90 seconds" to "Get Images".  

6. **Generate Videos with RunwayML**  
   - Add an **HTTP Request** node "Generate Videos".  
   - Configure POST to `https://api.dev.runwayml.com/v1/image_to_video` with parameters including `promptImage` (image URL), model "gen3a_turbo", ratio "768:1280", duration 5, promptText.  
   - Add HTTP Header Auth with RunwayML credentials, set header `X-Runway-Version` to `2024-11-06`.  
   - Connect "Get Images" to "Generate Videos".  
   - Add a **Wait** node "2 minutes" for 2 minutes.  
   - Connect "Generate Videos" to "2 minutes".  
   - Add an **HTTP Request** node "Get Videos" to poll video task status with URL `https://api.dev.runwayml.com/v1/tasks/{{ $json.id }}`.  
   - Use RunwayML credentials and headers as above.  
   - Connect "2 minutes" to "Get Videos".  

7. **Merge and Process Video URLs**  
   - Add a **Google Sheets** node "Video Status" to update `video status` to "Created" in the sheet row matching `row_number` from "Grab Idea".  
   - Connect "Get Videos" to "Merge" node.  
   - Add a **Google Drive** node "Share File" to share uploaded audio file (to be created later).  
   - Connect "Upload to Drive" to "Share File".  
   - Connect "Share File" to "Merge".  
   - Add a **Merge** node "Merge" (combine mode) to combine video and audio data.  
   - Connect "Get Videos" and "Share File" to "Merge".  
   - Add a **Limit** node "Limit" to limit to 1 item.  
   - Connect "Merge" to "Limit".  
   - Connect "Limit" to "Video Status".  
   - Add a **Code** node "Split Out Parts" to extract URLs from merged outputs and produce an array for rendering.  
   - Connect "Merge" to "Split Out Parts".  

8. **Generate Audio with ElevenLabs**  
   - Add a **LangChain Agent** node "Sound Agent" to generate sound prompt text based on style from "Grab Idea".  
   - Connect "Video Status" to "Sound Agent".  
   - Add a **Set** node "Set Audio" to clean and assign the output prompt as `audio`.  
   - Connect "Sound Agent" to "Set Audio".  
   - Add an **HTTP Request** node "Generate Audio".  
   - Configure POST to `https://api.elevenlabs.io/v1/sound-generation` with body parameters `text` (audio prompt) and `duration_seconds` 20.  
   - Use HTTP Header Auth with ElevenLabs credentials.  
   - Connect "Set Audio" to "Generate Audio".  
   - Add a **Google Drive** node "Upload to Drive" to upload audio file to a specific folder (folder ID: `1oGmtM9LXXKfblOQgyJiLCAr3DphJuAFY`).  
   - Name file as `style`.mp3.  
   - Connect "Generate Audio" to "Upload to Drive".  

9. **Render Final Video with Creatomate**  
   - Add an **HTTP Request** node "Render Video".  
   - Configure POST to `https://api.creatomate.com/v1/renders` with JSON body:  
     - Template ID: `62cb0638-8d36-4602-900a-0e331e45de27`  
     - Modifications: Video sources from video URLs (indexes 0-3), audio source from Google Drive link, text overlays from content1-4.  
   - Add Authorization header with Bearer token.  
   - Connect "Split Out Parts" to "Render Video".  
   - Add a **Wait** node "25 Seconds".  
   - Connect "Render Video" to "25 Seconds".  

10. **Download Rendered Video and Update Sheet**  
    - Add an **HTTP Request** node "Download Video" to download final video from URL returned by Creatomate.  
    - Connect "25 Seconds" to "Download Video".  
    - Add a **Google Sheets** node "Update Sheet" to update the row with `videoLink`, and statuses: `video status` = "Created", `publish status` = "Processed".  
    - Match the row using the appropriate column (preferably `row_number`).  
    - Connect "Download Video" to "Update Sheet".  

---

## 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| "Tour Agent's Social media video post generated via text prompt: A simple 20 second video and audio generated by just giving simple text prompt." | Sticky Note at workflow start (node "Sticky Note6")                                                    |
| "# Generate Image Prompts"                                                                                     | Sticky Note near "GPT 4o" and "Image Prompt Agent" nodes                                                |
| "# Generate Images"                                                                                            | Sticky Note near "Generate Image" node                                                                 |
| "# Generate Videos"                                                                                            | Sticky Note near "Generate Videos" node                                                                |
| "# Generate Audio"                                                                                             | Sticky Note near "Sound Agent" node                                                                    |
| "# Render & Upload"                                                                                            | Sticky Note near "Render Video" node                                                                   |
| "Connect to Social media and post the video automatically."                                                   | Sticky Note near "Update Sheet" and "Download Video" nodes                                             |
| OpenAI GPT-4o model usage with credentials "OpenAi account 5".                                                | Credential reference                                                                                     |
| Google Sheets document ID: `16MeId2XFWgj2JwS3h83QUcK_ygNy_3rwRj8XymhosLE` used for ideas and status updates | Source for input and output data                                                                        |
| Flux API for image generation: `https://api.piapi.ai/api/v1/task` with model "Qubico/flux1-dev".               | Image generation API                                                                                    |
| RunwayML API for video generation: `https://api.dev.runwayml.com/v1/` with model "gen3a_turbo".                | Video generation API                                                                                     |
| ElevenLabs API for audio synthesis: `https://api.elevenlabs.io/v1/sound-generation`.                           | Audio generation API                                                                                    |
| Creatomate API for final video rendering: `https://api.creatomate.com/v1/renders`.                             | Video rendering API with template ID `62cb0638-8d36-4602-900a-0e331e45de27`                            |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created using n8n, an integration and automation tool. This process strictly complies with content policies and contains no illegal or offensive elements. All processed data is legal and publicly accessible.

---