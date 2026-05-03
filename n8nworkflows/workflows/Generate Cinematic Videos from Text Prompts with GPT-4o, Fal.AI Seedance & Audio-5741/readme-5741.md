Generate Cinematic Videos from Text Prompts with GPT-4o, Fal.AI Seedance & Audio

https://n8nworkflows.xyz/workflows/generate-cinematic-videos-from-text-prompts-with-gpt-4o--fal-ai-seedance---audio-5741


# Generate Cinematic Videos from Text Prompts with GPT-4o, Fal.AI Seedance & Audio

### 1. Workflow Overview

This n8n workflow automates the generation of cinematic videos from text prompts using advanced AI models (OpenAI GPT-4o and Fal.AI Seedance) combined with automated video and audio processing. It targets creators and developers who want to transform textual story ideas into fully rendered video clips with synchronized audio, without manual video editing.

The workflow’s logic is structured into four main functional blocks, corresponding to distinct stages of the video generation pipeline:

- **1.1 Prompt Input & Story-to-Scenes:** Receive a text prompt, generate a detailed narrative, and break it into a fixed number of scenes.
- **1.2 Create Scene Prompts & Generate Video:** Generate detailed scene descriptions for each scene and call Fal.AI’s Seedance API to produce video clips.
- **1.3 Add Audio to Video:** Add audio tracks to each generated video clip using Fal AI’s audio service.
- **1.4 Merge Videos & Download Final Output:** Aggregate all video clips with audio and merge them into one final cinematic video ready for download or upload.

---

### 2. Block-by-Block Analysis

#### 2.1 Prompt Input & Story-to-Scenes

**Overview:**  
This block initiates the workflow by manually triggering execution, fetching input data from a Google Sheet, and transforming a raw story prompt into a structured long-form narrative. It then breaks the narrative into a user-defined number of scenes, verifying the scene count before proceeding.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Get Data (Google Sheets)  
- Generate Full Narrative from Prompt (OpenAI Agent)  
- Structured Output Parser  
- Break Narrative into {{n}} Scenes (OpenAI Agent)  
- Structured Output Parser1  
- Verify number of scene (Code)  
- Scene count (If)  

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually.  
  - Inputs: None  
  - Outputs: Triggers “Get Data” node.  
  - Failure edge cases: None (manual trigger).  

- **Get Data (Google Sheets)**  
  - Type: Google Sheets  
  - Role: Fetches input parameters and story prompt from a specific Google Sheet and sheet tab.  
  - Configuration: Reads entire sheet (gid=0), document ID set to a specific spreadsheet containing story and parameter data.  
  - Inputs: Trigger node  
  - Outputs: Supplies story text, number_of_scene, model, aspect_ratio, resolution, duration parameters.  
  - Potential failures: Authentication errors, missing or malformed sheet data.  

- **Generate Full Narrative from Prompt**  
  - Type: LangChain OpenAI Agent (GPT-4o mini)  
  - Role: Generate a vivid, cinematic, detailed story narrative from the short idea text.  
  - Configuration: Custom prompt instructing detailed story generation with rich visual and emotional content. Output parsed via Structured Output Parser with JSON schema containing "story" string.  
  - Inputs: Story prompt from “Get Data”  
  - Outputs: Long-form narrative text.  
  - Possible failures: API rate limits, prompt formatting errors, parsing errors.  

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses raw AI output into JSON object with “story” property for structured usage.  
  - Inputs: AI text output  
  - Outputs: JSON parsed narrative.  

- **Break Narrative into {{n}} Scenes**  
  - Type: LangChain OpenAI Agent (GPT-4o mini)  
  - Role: Break full narrative into exactly N scenes (N from Google Sheets).  
  - Configuration: Prompt includes instruction and the full story text, expects structured output with array of strings “scenes”. Output parsed by Structured Output Parser1.  
  - Inputs: Parsed story from previous step, number_of_scene parameter.  
  - Outputs: Array of scene descriptions.  
  - Error edges: Parsing errors, count mismatch, API failures.  

- **Structured Output Parser1**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses scene array from AI output.  

- **Verify number of scene**  
  - Type: Code (JavaScript)  
  - Role: Checks the length of the scenes array and outputs sceneCount and scenes array.  
  - Inputs: Parsed scenes array  
  - Outputs: sceneCount (integer), scenes (array)  
  - Edge cases: Empty scenes array, mismatch with desired scene count.  

- **Scene count (If)**  
  - Type: If Condition  
  - Role: Compares actual sceneCount with expected number_of_scene from Google Sheets to ensure equality or proceed accordingly.  
  - Inputs: sceneCount from code node and expected number_of_scene  
  - Outputs: Two branches - continue if sceneCount < expected, or proceed with splitting scenes.  
  - Edge cases: Scene count less than expected triggers alternative logic (not fully detailed here).  

---

#### 2.2 Create Scene Prompts & Generate Video

**Overview:**  
This block splits the scenes array into individual items, generates detailed visual descriptions for each scene using GPT-4o, and submits scene data to Fal.AI’s Seedance API to generate short video clips for each scene. It monitors each video job’s progress and retrieves the completed video.

**Nodes Involved:**  
- Split Out  
- Describe Each Scene for Video (OpenAI Agent)  
- Structured Output Parser2  
- Call Fal.ai API (Seedance) (HTTP Request)  
- Loop Over Items (SplitInBatches)  
- Wait for the video (Wait)  
- Get the video status (HTTP Request)  
- Video status (Switch)  
- Get the video (HTTP Request)  

**Node Details:**

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the scenes array into individual scene items for parallel processing.  
  - Inputs: Scenes array from previous block  
  - Outputs: One scene per execution branch.  

- **Describe Each Scene for Video**  
  - Type: LangChain OpenAI Agent (GPT-4o)  
  - Role: Generates detailed descriptions for each scene including characters, environment, camera movement, object movement, and sound effects.  
  - Configuration: Rich prompt template with instructions for detailed visual breakdown per 5-second clip.  
  - Inputs: Single scene text, full story context for continuity.  
  - Outputs: Structured JSON with multiple fields describing the scene visually and sonically.  
  - Edge cases: Parsing errors, model response inconsistencies.  

- **Structured Output Parser2**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses detailed scene description output into structured JSON with fields: characters, scene_description, camera_movement, object_movements, sound_effects.  

- **Call Fal.ai API (Seedance)**  
  - Type: HTTP Request  
  - Role: Sends scene description data to Fal.AI Seedance video generation API.  
  - Configuration: POST request with XML-like formatted body including characters, scene description, camera and object movements, plus video parameters from Google Sheets (aspect_ratio, resolution, duration). Authenticated with Fal AI API key.  
  - Inputs: Parsed scene description JSON  
  - Outputs: Job response containing status URLs for video processing.  
  - Potential failures: Network errors, authorization failures, malformed request body, API rate limits.  

- **Loop Over Items (SplitInBatches)**  
  - Type: SplitInBatches  
  - Role: Handles batch processing of video generation jobs, controlling concurrency and pacing of API calls.  
  - Inputs: Video generation job responses  
  - Outputs: Each video job processed stepwise.  

- **Wait for the video**  
  - Type: Wait  
  - Role: Pauses workflow to allow video processing time before status check. Webhook enabled for asynchronous wake-up.  

- **Get the video status**  
  - Type: HTTP Request  
  - Role: Requests the current processing status of the video job using status URL from job response.  

- **Video status (Switch)**  
  - Type: Switch  
  - Role: Routes workflow based on video job status: COMPLETED, IN_PROGRESS, IN_QUEUE.  
  - Edge cases: Unexpected status values, API timeouts.  

- **Get the video**  
  - Type: HTTP Request  
  - Role: Downloads or retrieves the completed video using response URL from job.  

---

#### 2.3 Add Audio to Video with Fal AI

**Overview:**  
This block adds audio tracks to each generated video clip by calling Fal AI’s audio API, then repeatedly checks audio processing status and retrieves the video with audio once completed.

**Nodes Involved:**  
- Start adding audio to the video (HTTP Request)  
- Loop Over Items1 (SplitInBatches)  
- Wait for adding the audio (Wait)  
- Get audio status (HTTP Request)  
- Audio status (Switch)  
- Get video with audio (HTTP Request)  

**Node Details:**

- **Start adding audio to the video**  
  - Type: HTTP Request  
  - Role: Sends a request to Fal AI’s MMAudio API to overlay audio on the video clip using the video URL and scene sound effect prompt extracted from detailed scene descriptions.  
  - Inputs: Video URL and sound effects prompt  
  - Outputs: Audio processing job status URL and response URL.  
  - Authentication: Fal AI API key.  
  - Edge cases: Request failures, malformed prompts, API limits.  

- **Loop Over Items1**  
  - Type: SplitInBatches  
  - Role: Processes multiple audio overlay jobs in batches.  

- **Wait for adding the audio**  
  - Type: Wait  
  - Role: Waits between status checks for audio processing completion, using webhook for asynchronous wake-up.  

- **Get audio status**  
  - Type: HTTP Request  
  - Role: Polls the status of audio processing job.  

- **Audio status (Switch)**  
  - Type: Switch  
  - Role: Routes workflow based on audio job status: COMPLETED, IN_PROGRESS, IN_QUEUE.  

- **Get video with audio**  
  - Type: HTTP Request  
  - Role: Retrieves the video clip with added audio once processing completes.  

---

#### 2.4 Merge Videos & Download Final Output

**Overview:**  
Aggregates all individual video clips with audio, sends a request to merge them into a single video using Fal AI’s ffmpeg API, monitors the merge status, retrieves the merged video, and uploads it to YouTube.

**Nodes Involved:**  
- Aggregate videos with audio (Aggregate)  
- Start merging videos (HTTP Request)  
- Wait for the merge to complete (Wait)  
- Get merge videos status (HTTP Request)  
- Merge videos status (Switch)  
- Get merged video (HTTP Request)  
- Get the video1 (HTTP Request)  
- YouTube (Upload)  

**Node Details:**

- **Aggregate videos with audio**  
  - Type: Aggregate  
  - Role: Combines all video-with-audio items into a single array for merge request.  
  - Inputs: Multiple video-with-audio outputs from previous block.  
  - Outputs: Aggregated array with video URLs and timestamps.  

- **Start merging videos**  
  - Type: HTTP Request  
  - Role: Sends POST request to Fal AI ffmpeg compose API with JSON body describing video tracks and keyframes to merge clips sequentially.  
  - Inputs: Aggregated video data with urls and timestamps.  
  - Outputs: Merge job status and response URLs.  

- **Wait for the merge to complete**  
  - Type: Wait  
  - Role: Waits asynchronously for video merge job completion.  

- **Get merge videos status**  
  - Type: HTTP Request  
  - Role: Polls the status of video merge job.  

- **Merge videos status (Switch)**  
  - Type: Switch  
  - Role: Routes workflow based on merge job status: COMPLETED, IN_PROGRESS, IN_QUEUE.  

- **Get merged video**  
  - Type: HTTP Request  
  - Role: Retrieves the merged final video URL once completed.  

- **Get the video1**  
  - Type: HTTP Request  
  - Role: Additional retrieval step for merged video (may support retry).  

- **YouTube**  
  - Type: YouTube node  
  - Role: Uploads the final merged video to YouTube using OAuth2 credentials with title from original story prompt.  
  - Inputs: Final video URL  
  - Outputs: Upload confirmation.  
  - Edge cases: Authentication errors, quota limits, upload failures.  

---

### 3. Summary Table

| Node Name                      | Node Type                        | Functional Role                                       | Input Node(s)               | Output Node(s)                          | Sticky Note                                                                                                                |
|-------------------------------|---------------------------------|------------------------------------------------------|-----------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                 | Starts workflow manually                             | None                        | Get Data                             | 🟨 Zone 1: Prompt Input & Story-to-Scenes                                                                                   |
| Get Data                      | Google Sheets                   | Fetch story and parameters from Sheet                | When clicking ‘Execute workflow’ | Generate Full Narrative from Prompt  | 🟨 Zone 1: Prompt Input & Story-to-Scenes                                                                                   |
| Generate Full Narrative from Prompt | LangChain OpenAI Agent (GPT-4o mini) | Generate detailed story narrative from prompt        | Get Data                    | Break Narrative into {{n}} Scenes     | 🟨 Zone 1: Prompt Input & Story-to-Scenes                                                                                   |
| Structured Output Parser       | LangChain Output Parser         | Parse story JSON output                               | Generate Full Narrative from Prompt | Generate Full Narrative from Prompt  |                                                                                                                            |
| Break Narrative into {{n}} Scenes | LangChain OpenAI Agent (GPT-4o mini) | Break story into scene array                          | Generate Full Narrative from Prompt | Verify number of scene               | 🟨 Zone 1: Prompt Input & Story-to-Scenes                                                                                   |
| Structured Output Parser1      | LangChain Output Parser         | Parse scenes array JSON                               | Break Narrative into {{n}} Scenes | Verify number of scene               |                                                                                                                            |
| Verify number of scene         | Code                           | Validate scene count                                  | Break Narrative into {{n}} Scenes | Scene count                        |                                                                                                                            |
| Scene count                   | If                             | Conditional branch based on scene count              | Verify number of scene       | Split Out / Break Narrative into {{n}} Scenes |                                                                                                                            |
| Split Out                    | Split Out                      | Split scenes array into individual scenes            | Scene count                 | Describe Each Scene for Video         | 🟫 Zone 2: Create Scene Prompts & Generate Video                                                                              |
| Describe Each Scene for Video  | LangChain OpenAI Agent (GPT-4o) | Generate detailed visual and audio descriptions per scene | Split Out                | Call Fal.ai API (Seedance)            | 🟫 Zone 2: Create Scene Prompts & Generate Video                                                                              |
| Structured Output Parser2      | LangChain Output Parser         | Parse detailed scene description JSON                | Describe Each Scene for Video | Call Fal.ai API (Seedance)            |                                                                                                                            |
| Call Fal.ai API (Seedance)     | HTTP Request                   | Send scene data to Fal AI Seedance video generator    | Describe Each Scene for Video | Loop Over Items                      | 🟫 Zone 2: Create Scene Prompts & Generate Video                                                                              |
| Loop Over Items               | SplitInBatches                 | Batch processing for video jobs                       | Get the video               | Start adding audio to the video       | 🟫 Zone 2: Create Scene Prompts & Generate Video                                                                              |
| Wait for the video            | Wait                          | Pause to allow video processing                       | Loop Over Items             | Get the video status                 | 🟫 Zone 2: Create Scene Prompts & Generate Video                                                                              |
| Get the video status          | HTTP Request                  | Poll video processing status                          | Wait for the video          | Video status                       | 🟫 Zone 2: Create Scene Prompts & Generate Video                                                                              |
| Video status                 | Switch                        | Route workflow based on video job status             | Get the video status        | Get the video / Wait for the video   | 🟫 Zone 2: Create Scene Prompts & Generate Video                                                                              |
| Get the video                | HTTP Request                  | Retrieve completed video                              | Video status                | Loop Over Items                     | 🟫 Zone 2: Create Scene Prompts & Generate Video                                                                              |
| Start adding audio to the video | HTTP Request                  | Request audio overlay on video                        | Loop Over Items             | Loop Over Items1                    | 🟥 Zone 3: Add Audio to Video with Fal AI                                                                                     |
| Loop Over Items1              | SplitInBatches                | Batch processing for audio jobs                       | Start adding audio to the video | Aggregate videos with audio / Wait for adding the audio | 🟥 Zone 3: Add Audio to Video with Fal AI                                                                                     |
| Wait for adding the audio     | Wait                         | Pause for audio processing                            | Audio status / Loop Over Items1 | Get audio status                    | 🟥 Zone 3: Add Audio to Video with Fal AI                                                                                     |
| Get audio status              | HTTP Request                 | Poll audio processing status                          | Wait for adding the audio   | Audio status                       | 🟥 Zone 3: Add Audio to Video with Fal AI                                                                                     |
| Audio status                 | Switch                       | Route workflow based on audio job status             | Get audio status            | Get video with audio / Wait for adding the audio | 🟥 Zone 3: Add Audio to Video with Fal AI                                                                                     |
| Get video with audio          | HTTP Request                 | Retrieve video with audio                             | Audio status                | Loop Over Items1                   | 🟥 Zone 3: Add Audio to Video with Fal AI                                                                                     |
| Aggregate videos with audio  | Aggregate                    | Combine all video with audio items                    | Loop Over Items1            | Start merging videos               | 🟩 Zone 4: Merge Videos & Download Final Output                                                                               |
| Start merging videos          | HTTP Request                 | Request video merge using ffmpeg API                  | Aggregate videos with audio | Wait for the merge to complete    | 🟩 Zone 4: Merge Videos & Download Final Output                                                                               |
| Wait for the merge to complete | Wait                        | Pause for merge processing                            | Start merging videos        | Get merge videos status           | 🟩 Zone 4: Merge Videos & Download Final Output                                                                               |
| Get merge videos status       | HTTP Request                | Poll merge job status                                | Wait for the merge to complete | Merge videos status              | 🟩 Zone 4: Merge Videos & Download Final Output                                                                               |
| Merge videos status          | Switch                      | Route workflow based on merge job status             | Get merge videos status     | Get merged video / Wait for the merge to complete | 🟩 Zone 4: Merge Videos & Download Final Output                                                                               |
| Get merged video             | HTTP Request                | Retrieve final merged video                           | Merge videos status         | Get the video1                    | 🟩 Zone 4: Merge Videos & Download Final Output                                                                               |
| Get the video1               | HTTP Request                | Additional video retrieval step                       | Get merged video            | YouTube                         | 🟩 Zone 4: Merge Videos & Download Final Output                                                                               |
| YouTube                     | YouTube Upload              | Upload final video to YouTube                         | Get the video1              | None                           | 🟩 Zone 4: Merge Videos & Download Final Output                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Add Google Sheets Node ("Get Data")**  
   - Configure OAuth2 credentials for Google Sheets.  
   - Set Document ID to your story+params spreadsheet.  
   - Set Sheet Name to use the main sheet tab.  
   - Output expected fields: story prompt, number_of_scene, model, aspect_ratio, resolution, duration.

3. **Add OpenAI Chat Model Node ("Generate Full Narrative from Prompt")**  
   - Type: LangChain OpenAI Agent  
   - Model: GPT-4o mini  
   - Credential: OpenAI API key  
   - Prompt: Instruction to create vivid, cinematic narrative from input story.  
   - Enable structured output parser with JSON schema expecting `{ "story": string }`.

4. **Add Structured Output Parser Node**  
   - Schema: `{ "story": string }`  
   - Connect output of OpenAI node here.

5. **Add OpenAI Chat Model Node ("Break Narrative into {{n}} Scenes")**  
   - Model: GPT-4o mini  
   - Credential: OpenAI API key  
   - Prompt: Break narrative into exact number of scenes (provided by Google Sheets).  
   - Enable structured output parser with JSON schema expecting `{ "scenes": string[] }`.

6. **Add Structured Output Parser1 Node**  
   - Schema: `{ "scenes": array of strings }`  
   - Connect output of previous node here.

7. **Add Code Node ("Verify number of scene")**  
   - JavaScript code to count scenes and emit `sceneCount` and `scenes`.

8. **Add If Node ("Scene count")**  
   - Condition: `sceneCount < number_of_scene` (both from previous nodes)  
   - True branch: handle scene count mismatch (optional)  
   - False branch: continue workflow.

9. **Add Split Out Node**  
   - Split the `scenes` array into individual scene items.

10. **Add OpenAI Chat Model Node ("Describe Each Scene for Video")**  
    - Model: GPT-4o  
    - Credential: OpenAI API key  
    - Prompt: Generate detailed scene breakdown including characters, environment, camera, movements, sound effects.  
    - Enable structured output parser with complex JSON schema for scene description.

11. **Add Structured Output Parser2 Node**  
    - Schema matching detailed scene description fields.

12. **Add HTTP Request Node ("Call Fal.ai API (Seedance)")**  
    - POST to `https://queue.fal.run/fal-ai/{model}` (model from Google Sheets).  
    - Body: XML-like formatted scene data (characters, scene_description, camera_movement, object_movements).  
    - Include aspect_ratio, resolution, duration from Google Sheets.  
    - Authentication: Fal AI HTTP Header Auth credentials.

13. **Add SplitInBatches Node ("Loop Over Items")**  
    - Batch process video generation jobs.

14. **Add Wait Node ("Wait for the video")**  
    - Use webhook to pause for video processing.

15. **Add HTTP Request Node ("Get the video status")**  
    - Poll status_url from video job response.

16. **Add Switch Node ("Video status")**  
    - Route on status: COMPLETED, IN_PROGRESS, IN_QUEUE.

17. **Add HTTP Request Node ("Get the video")**  
    - Retrieve completed video from response_url.

18. **Add HTTP Request Node ("Start adding audio to the video")**  
    - POST to Fal AI MMAudio API with video_url and sound_effects prompt.  
    - Authentication: Fal AI Header Auth.

19. **Add SplitInBatches Node ("Loop Over Items1")**  
    - Batch process audio overlay jobs.

20. **Add Wait Node ("Wait for adding the audio")**  
    - Pause for audio processing.

21. **Add HTTP Request Node ("Get audio status")**  
    - Poll audio status_url.

22. **Add Switch Node ("Audio status")**  
    - Route on audio job status.

23. **Add HTTP Request Node ("Get video with audio")**  
    - Retrieve video with audio.

24. **Add Aggregate Node ("Aggregate videos with audio")**  
    - Aggregate all video-with-audio items for merge.

25. **Add HTTP Request Node ("Start merging videos")**  
    - POST to Fal AI ffmpeg compose API with JSON describing tracks and timestamps.

26. **Add Wait Node ("Wait for the merge to complete")**  
    - Pause for merge job.

27. **Add HTTP Request Node ("Get merge videos status")**  
    - Poll merge job status.

28. **Add Switch Node ("Merge videos status")**  
    - Route on merge job status.

29. **Add HTTP Request Node ("Get merged video")**  
    - Retrieve final merged video.

30. **Add HTTP Request Node ("Get the video1")**  
    - Additional retrieval/retry of merged video.

31. **Add YouTube Node**  
    - OAuth2 credential for YouTube.  
    - Upload final merged video with original story as title.

**Credentials Required:**  
- Google Sheets OAuth2  
- OpenAI API key (GPT-4o and GPT-4o mini)  
- Fal AI HTTP Header Auth API key  
- YouTube OAuth2  

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The workflow uses Fal AI’s Seedance API for video generation and MMAudio API for audio overlay.                | Fal AI documentation and API: https://queue.fal.run/fal-ai                                           |
| OpenAI GPT-4o and GPT-4o mini models are used for text generation with structured JSON output parsing enabled. | OpenAI models: https://platform.openai.com/docs/models/gpt-4o                                        |
| YouTube upload requires OAuth2 credentials with upload permissions enabled.                                    | YouTube API: https://developers.google.com/youtube/v3/guides/uploading_a_video                        |
| The workflow uses multiple asynchronous wait nodes with webhooks to efficiently poll job statuses.            | n8n wait and webhook nodes: https://docs.n8n.io/nodes/n8n-nodes-base.wait/                           |
| Input parameters (story, number_of_scene, video model, aspect ratio, resolution, duration) are fetched from a Google Sheet for flexibility. | Google Sheets integration: https://docs.n8n.io/integrations/builtin/google-sheets/                   |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.