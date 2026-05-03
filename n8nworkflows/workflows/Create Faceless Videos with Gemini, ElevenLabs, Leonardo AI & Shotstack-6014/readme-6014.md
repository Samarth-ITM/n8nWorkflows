Create Faceless Videos with Gemini, ElevenLabs, Leonardo AI & Shotstack

https://n8nworkflows.xyz/workflows/create-faceless-videos-with-gemini--elevenlabs--leonardo-ai---shotstack-6014


# Create Faceless Videos with Gemini, ElevenLabs, Leonardo AI & Shotstack

---

## 1. Workflow Overview

This workflow automates the creation of faceless videos from a simple topic idea. It is designed for content creators, marketers, and agencies who want to produce engaging videos without manual video editing or recording their own voice. The workflow leverages multiple AI services and cloud tools to generate scripts, voiceovers, images, videos, and then compiles everything into a final polished video.

### Logical Blocks:

- **1.1 Input Reception:** Receive and set the video topic idea to start the process.
- **1.2 Script Generation:** Generate a concise 60-second video script based on the idea using Google Gemini.
- **1.3 Audio Generation:** Convert the generated script into voiceover audio with ElevenLabs, upload and share it on Google Drive, then transcribe it using OpenAI Whisper.
- **1.4 Timestamps Generation:** Merge script and transcription data, generate timestamped image prompts with Google Gemini, and parse the output into structured JSON.
- **1.5 Images Generation:** Split prompts and generate corresponding images with Leonardo AI, including a wait to ensure rendering is complete.
- **1.6 Images to Video Conversion:** Convert generated images into short video scenes using Leonardo AI, wait for rendering, download all scenes, and aggregate them.
- **1.7 Video Editing and Downloading:** Use Shotstack to edit and combine all video scenes with audio, wait for processing, then download the final video for local use.

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception

**Overview:**  
Starts the workflow with a manual trigger and sets the video idea/topic to be processed.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Fields - Set Idea  
- Sticky Note31 (Instructional)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates the entire workflow manually.  
  - Inputs: None  
  - Outputs: Connects to Fields - Set Idea  
  - Failure Modes: None expected.

- **Fields - Set Idea**  
  - Type: Set  
  - Role: Defines the video idea text; default value "What is AI Agents".  
  - Configuration: Sets a string field `Idea` with the video topic.  
  - Inputs: From Manual Trigger  
  - Outputs: To 60 Second Script Writer  
  - Failure Modes: If empty or invalid topic, subsequent nodes may generate irrelevant scripts.  
  - Notes: User should replace the default idea with their desired topic.

- **Sticky Note31**  
  - Type: Sticky Note  
  - Role: Documents the importance of providing the topic input.

---

### 1.2 Script Generation

**Overview:**  
Generates a concise, engaging 60-second script using Google Gemini chat model and formats it for voice and visuals.

**Nodes Involved:**  
- 60 Second Script Writer  
- Google Gemini Chat Model 1  
- Fields - Script Format  
- Sticky Note29

**Node Details:**

- **Google Gemini Chat Model 1**  
  - Type: Langchain Google Gemini chat LLM node  
  - Role: Receives the video idea and generates a faceless video script.  
  - Configuration: Uses Gemini 2.0 flash model, default options.  
  - Inputs: From Fields - Set Idea (via 60 Second Script Writer)  
  - Outputs: To 60 Second Script Writer  
  - Credentials: Google Gemini API  
  - Failure Modes: API errors, rate limits, or malformed input can cause failure.

- **60 Second Script Writer**  
  - Type: Langchain chain LLM  
  - Role: Sends prompt to Google Gemini to create an engaging 60-second script based on the idea.  
  - Configuration: Custom prompt emphasizing storytelling, emotion, and engagement; outputs script text only.  
  - Inputs: From Fields - Set Idea  
  - Outputs: To Fields - Script Format  
  - Failure Modes: LLM errors, prompt formatting issues.

- **Fields - Script Format**  
  - Type: Set  
  - Role: Cleans and formats the script text by removing all newline characters for smooth downstream processing.  
  - Inputs: From 60 Second Script Writer  
  - Outputs: To Generate Voice  
  - Failure Modes: Expression errors if input text is missing.

- **Sticky Note29**  
  - Documentation about script generation process.

---

### 1.3 Audio Generation

**Overview:**  
Converts the formatted script into voiceover audio via ElevenLabs, uploads audio to Google Drive and makes it public, then transcribes it with OpenAI Whisper.

**Nodes Involved:**  
- Generate Voice  
- Upload Audio to Drive  
- Make Audio File Public  
- Transcribe Audio with OpenAI Whisper  
- Sticky Note30

**Node Details:**

- **Generate Voice**  
  - Type: HTTP Request  
  - Role: Sends script text to ElevenLabs Text-to-Speech API to generate audio.  
  - Configuration: POST to ElevenLabs TTS endpoint with text in JSON body, uses API key authentication in header.  
  - Inputs: From Fields - Script Format  
  - Outputs: To Upload Audio to Drive and Transcribe Audio with OpenAI Whisper (two parallel outputs)  
  - Failure Modes: API key issues, network errors, invalid text input, rate limits.

- **Upload Audio to Drive**  
  - Type: Google Drive node  
  - Role: Uploads generated audio file to Google Drive under "faceless-video-audio-<timestamp>" in root folder.  
  - Configuration: Uses OAuth2 credentials for Google Drive, uploads to root folder.  
  - Inputs: From Generate Voice  
  - Outputs: To Make Audio File Public  
  - Failure Modes: OAuth token expiration, quota exceeded, file size limits.

- **Make Audio File Public**  
  - Type: Google Drive node  
  - Role: Shares uploaded audio file publicly with reader permission for anyone.  
  - Configuration: Uses file ID from upload, sets permission role "reader", type "anyone".  
  - Inputs: From Upload Audio to Drive  
  - Outputs: To Merge node (for sync later)  
  - Failure Modes: Permission errors, API limits.

- **Transcribe Audio with OpenAI Whisper**  
  - Type: HTTP Request  
  - Role: Sends audio file binary to OpenAI Whisper API to generate detailed transcription with word-level timestamps.  
  - Configuration: POST multipart-form data with Whisper model "whisper-1", verbose JSON output.  
  - Inputs: From Generate Voice (audio binary)  
  - Outputs: To Merge node  
  - Failure Modes: API key invalid, large file size, network timeout.

- **Sticky Note30**  
  - Explains audio generation, upload, and transcription process.

---

### 1.4 Timestamps Generation

**Overview:**  
Merges audio transcription and script data, generates timestamped image prompts via Google Gemini, and parses output into structured JSON for further processing.

**Nodes Involved:**  
- Merge  
- Google Gemini Chat Model 2  
- Structured Output Parser1  
- Auto-fixing Output Parse  
- Generate Image Prompts  
- Sticky Note23

**Node Details:**

- **Merge**  
  - Type: Merge  
  - Role: Combines outputs from Make Audio File Public and Transcribe Audio with OpenAI Whisper to synchronize data.  
  - Configuration: Combine mode "combineByPosition", includes unpaired data.  
  - Inputs: From Make Audio File Public (audio info) and Transcribe Audio (transcription)  
  - Outputs: To Google Gemini Chat Model 2  
  - Failure Modes: Mismatched data arrays, missing inputs.

- **Google Gemini Chat Model 2**  
  - Type: Langchain Google Gemini chat LLM node  
  - Role: Receives merged transcription and script data, generates detailed image prompts with start/end times for each scene.  
  - Configuration: Gemini 2.0 flash model, default options.  
  - Inputs: From Merge  
  - Outputs: To Structured Output Parser1 and Generate Image Prompts  
  - Failure Modes: API errors, unexpected input format.

- **Structured Output Parser1**  
  - Type: Langchain output parser structured  
  - Role: Parses Gemini’s raw output into strict JSON schema with start_time, end_time, duration, and prompt fields.  
  - Configuration: Manual JSON schema with required fields, ensures data validity.  
  - Inputs: From Google Gemini Chat Model 2  
  - Outputs: To Auto-fixing Output Parse  
  - Failure Modes: Schema mismatch, invalid JSON.

- **Auto-fixing Output Parse**  
  - Type: Langchain output parser autofixing  
  - Role: Attempts to correct parsing errors and sanitize the output for robust downstream processing.  
  - Inputs: From Structured Output Parser1  
  - Outputs: To Generate Image Prompts  
  - Failure Modes: Unfixable parsing errors.

- **Generate Image Prompts**  
  - Type: Langchain chain LLM  
  - Role: Processes cleaned output to produce final image prompts with timestamps for video scenes.  
  - Configuration: Custom prompt instructing to create cinematic, vivid image prompts covering entire video duration, with precise timing.  
  - Inputs: From Auto-fixing Output Parse  
  - Outputs: To Split Prompts  
  - Failure Modes: Prompt or output errors.

- **Sticky Note23**  
  - Explains the process of merging script and transcription, generating prompts with timestamps.

---

### 1.5 Images Generation

**Overview:**  
Splits generated prompts into individual entries, sends each to Leonardo AI for image generation, waits 30 seconds for rendering completion, then retrieves images.

**Nodes Involved:**  
- Split Prompts  
- Generate Images  
- Wait 30s  
- Get Images  
- Sticky Note24

**Node Details:**

- **Split Prompts**  
  - Type: Split Out  
  - Role: Splits the array of image prompts into individual items for separate processing.  
  - Inputs: From Generate Image Prompts  
  - Outputs: To Generate Images  
  - Failure Modes: Empty arrays.

- **Generate Images**  
  - Type: HTTP Request  
  - Role: Calls Leonardo AI’s image generation API with prompt text, fixed width 720, height 1280, model ID specified.  
  - Inputs: From Split Prompts  
  - Outputs: To Wait 30s  
  - Credentials: Leonardo API via header auth  
  - Failure Modes: API key errors, rate limits, prompt rejection.

- **Wait 30s**  
  - Type: Wait  
  - Role: Pauses workflow for 30 seconds to allow Leonardo AI to complete image rendering.  
  - Inputs: From Generate Images  
  - Outputs: To Get Images  
  - Failure Modes: None.

- **Get Images**  
  - Type: HTTP Request  
  - Role: Retrieves generated images from Leonardo AI using generation job IDs.  
  - Inputs: From Wait 30s  
  - Outputs: To Generate Videos/Scenes  
  - Credentials: Leonardo API  
  - Failure Modes: API errors, timeout.

- **Sticky Note24**  
  - Describes the image generation process and timing.

---

### 1.6 Images to Video Conversion

**Overview:**  
Converts each generated image into a short video scene via Leonardo AI, waits 5 minutes for rendering, downloads all scenes, and aggregates them for final editing.

**Nodes Involved:**  
- Generate Videos/Scenes  
- Wait 5 mins  
- Get Videos/Scenes  
- Download Generated Videos/Scenes  
- Aggregate  
- Sticky Note25

**Node Details:**

- **Generate Videos/Scenes**  
  - Type: HTTP Request  
  - Role: Sends image ID to Leonardo AI motion generation API to create short video clips with motion strength 3.  
  - Inputs: From Get Images  
  - Outputs: To Wait 5 mins  
  - Credentials: Leonardo API  
  - Failure Modes: API errors, invalid image IDs.

- **Wait 5 mins**  
  - Type: Wait  
  - Role: Pauses workflow 5 minutes to allow video scene rendering.  
  - Inputs: From Generate Videos/Scenes  
  - Outputs: To Get Videos/Scenes  
  - Failure Modes: None.

- **Get Videos/Scenes**  
  - Type: HTTP Request  
  - Role: Retrieves video scenes by generation job ID from Leonardo AI API.  
  - Inputs: From Wait 5 mins  
  - Outputs: To Download Generated Videos/Scenes  
  - Credentials: Leonardo API  
  - Failure Modes: API errors, incomplete rendering.

- **Download Generated Videos/Scenes**  
  - Type: HTTP Request  
  - Role: Downloads video scene files (motionMP4URL) for each generated video.  
  - Inputs: From Get Videos/Scenes  
  - Outputs: To Aggregate  
  - Failure Modes: Network errors, unavailable URLs.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Collects all downloaded video scene data into a single list for final processing.  
  - Inputs: From Download Generated Videos/Scenes  
  - Outputs: To Edit with Shotstack  
  - Failure Modes: Empty input.

- **Sticky Note25**  
  - Details images-to-video conversion and aggregation.

---

### 1.7 Video Editing and Downloading

**Overview:**  
Combines audio and video scenes using Shotstack API, waits for rendering, retrieves the final video, and provides download.

**Nodes Involved:**  
- Edit with Shotstack  
- Wait 1 min  
- Render Final Video with Shotstack  
- Download Final Video  
- Sticky Note22

**Node Details:**

- **Edit with Shotstack**  
  - Type: HTTP Request  
  - Role: Sends timeline data including audio soundtrack URL and clips (video scenes) to Shotstack API to start rendering a combined video.  
  - Configuration: POST with JSON body specifying soundtrack (Google Drive audio public link), video clips with start time and length, output format mp4 at 720x1280.  
  - Inputs: From Aggregate  
  - Outputs: To Wait 1 min  
  - Credentials: Shotstack API header auth  
  - Failure Modes: API errors, invalid URLs, authorization failure.

- **Wait 1 min**  
  - Type: Wait  
  - Role: Waits 1 minute for Shotstack to process the video rendering.  
  - Inputs: From Edit with Shotstack  
  - Outputs: To Render Final Video with Shotstack  
  - Failure Modes: None.

- **Render Final Video with Shotstack**  
  - Type: HTTP Request  
  - Role: Checks rendering status and retrieves final video metadata from Shotstack using render ID.  
  - Inputs: From Wait 1 min  
  - Outputs: To Download Final Video  
  - Credentials: Shotstack API  
  - Failure Modes: Video not ready yet, API errors.

- **Download Final Video**  
  - Type: HTTP Request  
  - Role: Downloads the final polished video file from Shotstack’s URL for local storage.  
  - Inputs: From Render Final Video with Shotstack  
  - Outputs: None (end node)  
  - Failure Modes: Network errors, unavailable URL.

- **Sticky Note22**  
  - Explains video editing and downloading process.

---

## 3. Summary Table

| Node Name                       | Node Type                                | Functional Role                           | Input Node(s)                             | Output Node(s)                          | Sticky Note                                                                                                            |
|--------------------------------|-----------------------------------------|-----------------------------------------|------------------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger                          | Start workflow manually                  | None                                     | Fields - Set Idea                      |                                                                                                                        |
| Fields - Set Idea               | Set                                    | Set video topic idea                     | When clicking ‘Test workflow’             | 60 Second Script Writer                | ## 1. Provide Topic Input For Your Video - A short topic and idea should be entered here before triggering the process. |
| 60 Second Script Writer         | Langchain Chain LLM                     | Generate 60-second video script          | Fields - Set Idea                        | Fields - Script Format                 | ## 2. Script Generation - Create concise faceless video script with Google Gemini                                        |
| Google Gemini Chat Model 1      | Langchain Google Gemini Chat LLM       | Backend LLM model for script generation  | 60 Second Script Writer (ai_languageModel) | 60 Second Script Writer (ai_languageModel) |                                                                                                                        |
| Fields - Script Format          | Set                                    | Format script text for voice generation  | 60 Second Script Writer                  | Generate Voice                        |                                                                                                                        |
| Generate Voice                 | HTTP Request                           | Convert script to voiceover with ElevenLabs | Fields - Script Format                   | Upload Audio to Drive, Transcribe Audio with OpenAI Whisper | ## 3. Audio Generation - Generate voice, upload and transcribe audio                                                   |
| Upload Audio to Drive           | Google Drive                           | Upload audio file to Google Drive        | Generate Voice                          | Make Audio File Public                |                                                                                                                        |
| Make Audio File Public          | Google Drive                           | Share audio file publicly                 | Upload Audio to Drive                   | Merge                                |                                                                                                                        |
| Transcribe Audio with OpenAI Whisper | HTTP Request                           | Transcribe audio to text using Whisper   | Generate Voice                          | Merge                                |                                                                                                                        |
| Merge                          | Merge                                  | Combine audio file info and transcription | Make Audio File Public, Transcribe Audio with OpenAI Whisper | Google Gemini Chat Model 2             | ## 4. Timestamps Generation - Merge script and transcription to generate timestamp prompts                            |
| Google Gemini Chat Model 2      | Langchain Google Gemini Chat LLM       | Generate timestamped image prompts       | Merge                                   | Structured Output Parser1, Generate Image Prompts |                                                                                                                        |
| Structured Output Parser1       | Langchain Structured Output Parser     | Parse Gemini output into structured JSON | Google Gemini Chat Model 2               | Auto-fixing Output Parse              |                                                                                                                        |
| Auto-fixing Output Parse        | Langchain Autofixing Output Parser     | Fix parsing errors in output              | Structured Output Parser1                | Generate Image Prompts                |                                                                                                                        |
| Generate Image Prompts          | Langchain Chain LLM                     | Generate cinematic image prompts          | Auto-fixing Output Parse                 | Split Prompts                       |                                                                                                                        |
| Split Prompts                  | Split Out                              | Split prompts array into individual items | Generate Image Prompts                   | Generate Images                      | ## 5. Images Generation - Generate images based on prompts                                                            |
| Generate Images                | HTTP Request                           | Call Leonardo AI to generate images       | Split Prompts                          | Wait 30s                           |                                                                                                                        |
| Wait 30s                      | Wait                                   | Pause for image rendering                  | Generate Images                        | Get Images                         |                                                                                                                        |
| Get Images                    | HTTP Request                           | Retrieve generated images                   | Wait 30s                              | Generate Videos/Scenes              |                                                                                                                        |
| Generate Videos/Scenes         | HTTP Request                           | Create video scenes from images            | Get Images                            | Wait 5 mins                       | ## 6. Images to Video Conversion - Convert images into short videos                                                  |
| Wait 5 mins                   | Wait                                   | Wait for scene rendering                    | Generate Videos/Scenes                 | Get Videos/Scenes                  |                                                                                                                        |
| Get Videos/Scenes             | HTTP Request                           | Retrieve rendered video scenes              | Wait 5 mins                          | Download Generated Videos/Scenes   |                                                                                                                        |
| Download Generated Videos/Scenes | HTTP Request                           | Download each video scene                    | Get Videos/Scenes                    | Aggregate                        |                                                                                                                        |
| Aggregate                    | Aggregate                              | Collect all video scenes into one list      | Download Generated Videos/Scenes      | Edit with Shotstack               |                                                                                                                        |
| Edit with Shotstack           | HTTP Request                           | Send combined video and audio data for editing | Aggregate                          | Wait 1 min                      | ## 7. Video Editing And Downloading - Final editing with Shotstack                                                  |
| Wait 1 min                   | Wait                                   | Wait for Shotstack processing                | Edit with Shotstack                   | Render Final Video with Shotstack |                                                                                                                        |
| Render Final Video with Shotstack | HTTP Request                           | Check rendering status and get video URL     | Wait 1 min                          | Download Final Video             |                                                                                                                        |
| Download Final Video          | HTTP Request                           | Download final polished video                 | Render Final Video with Shotstack    | None                             |                                                                                                                        |
| Sticky Note22                | Sticky Note                           | Explains video editing and downloading block | None                               | None                             | ## 7. Video Editing And Downloading - Shotstack process explained                                                    |
| Sticky Note23                | Sticky Note                           | Explains timestamps generation block         | None                               | None                             | ## 4. Timestamps Generation - Merging script and transcription                                                      |
| Sticky Note24                | Sticky Note                           | Explains images generation block              | None                               | None                             | ## 5. Images Generation - Leonardo AI image generation explained                                                   |
| Sticky Note25                | Sticky Note                           | Explains images to video conversion block     | None                               | None                             | ## 6. Images to Video Conversion - Leonardo AI video scenes explained                                               |
| Sticky Note29                | Sticky Note                           | Explains script generation block               | None                               | None                             | ## 2. Script Generation - Google Gemini script generation explained                                                |
| Sticky Note30                | Sticky Note                           | Explains audio generation block                 | None                               | None                             | ## 3. Audio Generation - Voice and transcription explained                                                        |
| Sticky Note31                | Sticky Note                           | Explains input reception block                  | None                               | None                             | ## 1. Provide Topic Input For Your Video - Input instructions                                                      |
| Fields - Script Format       | Set                                    | Format script text for voice generation        | 60 Second Script Writer              | Generate Voice                   |                                                                                                                        |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: When clicking ‘Test workflow’  
   - Purpose: To manually start the workflow.

2. **Create Set Node for Video Idea:**  
   - Name: Fields - Set Idea  
   - Set field `Idea` (string) with a default value or user input (e.g., "What is AI Agents").  
   - Connect manual trigger output to this node.

3. **Create Langchain Google Gemini Chat Node (Script Generation):**  
   - Name: Google Gemini Chat Model 1  
   - Model: "models/gemini-2.0-flash"  
   - Credentials: Connect Google Gemini API credentials.  
   - Input: Connect from Fields - Set Idea (via a chain LLM node).  
   - Purpose: Generate a 60-second faceless video script.

4. **Create Chain LLM Node for 60 Second Script Writer:**  
   - Name: 60 Second Script Writer  
   - Prompt: “Act as a YouTube video scriptwriter… (custom prompt as per original).”  
   - Input: Connect Fields - Set Idea output.  
   - Output: Connect to Google Gemini Chat Model 1 (ai_languageModel input).  
   - Output of Gemini goes back to this node (ai_languageModel output).  
   - Then connect its output to Fields - Script Format node.

5. **Create Set Node to Format Script Text:**  
   - Name: Fields - Script Format  
   - Expression: Remove all newline characters from script text using regex.  
   - Input: From 60 Second Script Writer.  
   - Output: To HTTP Request node Generate Voice.

6. **Create HTTP Request Node to Generate Voice:**  
   - Name: Generate Voice  
   - POST to ElevenLabs TTS API endpoint with script text in JSON body.  
   - Authentication: HTTP Header Auth with ElevenLabs API key.  
   - Output: Connect to both Upload Audio to Drive and Transcribe Audio with OpenAI Whisper nodes (parallel).

7. **Create Google Drive Node to Upload Audio:**  
   - Name: Upload Audio to Drive  
   - Operation: Upload file, name with timestamp prefix, upload to root folder.  
   - Credentials: Google Drive OAuth2.  
   - Input: From Generate Voice.  
   - Output: To Make Audio File Public.

8. **Create Google Drive Node to Make Audio Public:**  
   - Name: Make Audio File Public  
   - Operation: Share file with permission role "reader", type "anyone".  
   - Input: From Upload Audio to Drive.  
   - Output: To Merge node.

9. **Create HTTP Request Node to Transcribe Audio:**  
   - Name: Transcribe Audio with OpenAI Whisper  
   - POST multipart-form data to OpenAI Whisper API with audio binary.  
   - Model: whisper-1, response format verbose_json, timestamp granularity word.  
   - Credentials: OpenAI API key.  
   - Input: From Generate Voice.  
   - Output: To Merge node.

10. **Create Merge Node:**  
    - Name: Merge  
    - Mode: combineByPosition, include unpaired.  
    - Inputs: From Make Audio File Public and Transcribe Audio.  
    - Output: To Google Gemini Chat Model 2.

11. **Create Google Gemini Chat Model 2 Node:**  
    - Name: Google Gemini Chat Model 2  
    - Model: "models/gemini-2.0-flash"  
    - Credentials: Google Gemini API.  
    - Input: From Merge.  
    - Output: To Structured Output Parser1 and Generate Image Prompts.

12. **Create Structured Output Parser Node:**  
    - Name: Structured Output Parser1  
    - Schema: JSON schema with array of objects containing start_time, end_time, duration, prompt.  
    - Input: From Google Gemini Chat Model 2.  
    - Output: To Auto-fixing Output Parse.

13. **Create Auto-fixing Output Parser Node:**  
    - Name: Auto-fixing Output Parse  
    - Input: From Structured Output Parser1.  
    - Output: To Generate Image Prompts.

14. **Create Chain LLM Node to Generate Image Prompts:**  
    - Name: Generate Image Prompts  
    - Prompt: Expert prompt creator to divide video into scenes with cinematic image prompts and timings.  
    - Input: From Auto-fixing Output Parse.  
    - Output: To Split Prompts.

15. **Create Split Out Node:**  
    - Name: Split Prompts  
    - Field to split: "output" (array of prompts).  
    - Input: From Generate Image Prompts.  
    - Output: To Generate Images.

16. **Create HTTP Request Node for Image Generation:**  
    - Name: Generate Images  
    - POST to Leonardo AI generations endpoint with prompt, width=720, height=1280, modelId as per original.  
    - Authentication: Header Auth with Leonardo API key.  
    - Input: From Split Prompts.  
    - Output: To Wait 30s.

17. **Create Wait Node:**  
    - Name: Wait 30s  
    - Time: 30 seconds.  
    - Input: From Generate Images.  
    - Output: To Get Images.

18. **Create HTTP Request Node to Retrieve Images:**  
    - Name: Get Images  
    - GET request to Leonardo AI generations endpoint by generation ID.  
    - Authentication: Header Auth with Leonardo API key.  
    - Input: From Wait 30s.  
    - Output: To Generate Videos/Scenes.

19. **Create HTTP Request Node to Generate Videos/Scenes:**  
    - Name: Generate Videos/Scenes  
    - POST to Leonardo AI motion generation endpoint with image ID, motionStrength=3, isPublic=true.  
    - Authentication: Header Auth with Leonardo API key.  
    - Input: From Get Images.  
    - Output: To Wait 5 mins.

20. **Create Wait Node:**  
    - Name: Wait 5 mins  
    - Time: 5 minutes.  
    - Input: From Generate Videos/Scenes.  
    - Output: To Get Videos/Scenes.

21. **Create HTTP Request Node to Retrieve Videos/Scenes:**  
    - Name: Get Videos/Scenes  
    - GET Leonardo AI generation motion endpoint with generation ID.  
    - Authentication: Header Auth with Leonardo API key.  
    - Input: From Wait 5 mins.  
    - Output: To Download Generated Videos/Scenes.

22. **Create HTTP Request Node to Download Videos/Scenes:**  
    - Name: Download Generated Videos/Scenes  
    - GET request to video URL (motionMP4URL).  
    - Input: From Get Videos/Scenes.  
    - Output: To Aggregate.

23. **Create Aggregate Node:**  
    - Name: Aggregate  
    - Aggregate all items into a list under "list" field.  
    - Input: From Download Generated Videos/Scenes.  
    - Output: To Edit with Shotstack.

24. **Create HTTP Request Node to Edit with Shotstack:**  
    - Name: Edit with Shotstack  
    - POST to Shotstack API /edit/stage/render endpoint.  
    - Body: JSON with soundtrack (Google Drive public audio URL) and video clips with start times and lengths.  
    - Output format: mp4, size 720x1280.  
    - Authentication: HTTP Header Auth with Shotstack API key.  
    - Input: From Aggregate.  
    - Output: To Wait 1 min.

25. **Create Wait Node:**  
    - Name: Wait 1 min  
    - Time: 1 minute.  
    - Input: From Edit with Shotstack.  
    - Output: To Render Final Video with Shotstack.

26. **Create HTTP Request Node to Render Final Video:**  
    - Name: Render Final Video with Shotstack  
    - GET request to Shotstack render status endpoint with render ID.  
    - Authentication: Header Auth with Shotstack API key.  
    - Input: From Wait 1 min.  
    - Output: To Download Final Video.

27. **Create HTTP Request Node to Download Final Video:**  
    - Name: Download Final Video  
    - GET request to final video URL from Shotstack response.  
    - Input: From Render Final Video with Shotstack.  
    - Output: None (end).

28. **Add Sticky Notes:**  
    - Add all sticky notes as visual documentation in relevant workflow positions with content provided.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| This n8n template walks you through a fully automated faceless video creation process from script to final download, ideal for Youtube Shorts, TikTok, marketers, solopreneurs, and agencies.                                                                                                                                                                                                                                                                                                                                                                                               | Sticky Note1 (workflow introduction and usage)                 |
| Requires Google Cloud Console setup with Drive API enabled, Google Gemini API access, ElevenLabs API, OpenAI Whisper API, Leonardo AI API, and Shotstack API for smooth execution.                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note1 (requirements section)                            |
| To customize: change video topic in Fields - Set Idea, adjust script length in 60 Second Script Writer prompt, swap AI providers (Google Gemini, OpenAI ChatGPT, Claude, ElevenLabs, Whisper, Leonardo, Shotstack) as needed.                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note1 (customization section)                           |
| For support and community, visit Agent Circle’s website and social platforms listed in Sticky Note1.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | https://www.agentcircle.ai/ and linked social media platforms |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, respecting all applicable content policies. It contains no illegal, offensive, or protected elements. All data processed are legal and public.

---