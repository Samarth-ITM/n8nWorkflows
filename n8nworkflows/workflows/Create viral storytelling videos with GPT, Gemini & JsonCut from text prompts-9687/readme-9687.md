Create viral storytelling videos with GPT, Gemini & JsonCut from text prompts

https://n8nworkflows.xyz/workflows/create-viral-storytelling-videos-with-gpt--gemini---jsoncut-from-text-prompts-9687


# Create viral storytelling videos with GPT, Gemini & JsonCut from text prompts

### 1. Workflow Overview

This workflow automates the creation of viral short-form storytelling videos for social media platforms such as Instagram, TikTok, and YouTube Shorts. It takes a text prompt defining a video theme, setting, and optional image style, then generates a cohesive video combining AI-generated narration, segmented subtitles, background images, and audio. The workflow is designed for content creators and marketers aiming to produce emotionally engaging videos with minimal manual effort.

**Logical Blocks:**

- **1.1 Input Reception:** Receive user input via a form containing the video theme, setting, and image style.
- **1.2 Content & Media Generation:** Generate video script and image prompts using GPT, obtain background audio from Openverse, download watermark, generate AI images with Gemini, and synthesize narration audio with OpenAI.
- **1.3 Media Upload:** Upload generated images, narration audio, watermark, and background music to JsonCut storage.
- **1.4 Video Job Creation & Monitoring:** Create a video rendering job on JsonCut API using uploaded assets; poll for job status until completion or failure.
- **1.5 Output Handling:** Download the completed video file and save metadata and the video to a NocoDB table for storage and later retrieval.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures the core video parameters from the user through a web form.
- **Nodes Involved:**  
  - `On form submission`  
- **Node Details:**

  - **On form submission**  
    - Type: `formTrigger`  
    - Role: Starts the workflow upon form submission.  
    - Configuration:  
      - Form fields:  
        - Video Theme (required textarea)  
        - Video Setting (required textarea)  
        - Image Style (optional textarea)  
      - Form description guides users to provide core video ideas.  
    - Input: None (triggered externally)  
    - Output: JSON with user input fields.  
    - Edge cases: Missing required fields; malformed input.  
    - Notes: This node is the entry point of the workflow.

#### 2.2 Content & Media Generation

- **Overview:** Uses AI APIs to generate video script, segmented narration text, image prompts, and background audio; downloads a watermark image.
- **Nodes Involved:**  
  - `Generate content and Image Prompt`  
  - `Split Image Prompts`  
  - `Generate an image`  
  - `Generate audio`  
  - `Get List of background Audio`  
  - `Download MP3`  
  - `Download Watermark`  
- **Node Details:**

  - **Generate content and Image Prompt**  
    - Type: `OpenAI (LangChain)`  
    - Role: Generates JSON with 5 image prompts, full narration text, 4 text segments, and a caption, based on user inputs.  
    - Configuration: Uses GPT-5-MINI model with system prompt guiding it as a creative video director/scriptwriter. Outputs JSON only.  
    - Key expressions: Uses form input fields via expressions (`$json['Video Theme']`, `$json['Video Setting']`, `$json['Image Style']`).  
    - Output: JSON structured with image_prompts[], full_text, text_segments[], caption.  
    - Edge cases: API rate limits, malformed JSON output, empty or inconsistent input.  

  - **Split Image Prompts**  
    - Type: `splitOut`  
    - Role: Separates the array of 5 image prompts into individual items for separate image generation.  
    - Configuration: Splits field `message.content.image_prompts`.  
    - Input: JSON from previous node.  
    - Output: Each image prompt as a separate item.  

  - **Generate an image**  
    - Type: `Google Gemini (PaLM)`  
    - Role: Generates one background image per image prompt, formatted 9:16, no text in image.  
    - Configuration: Model `imagen-4.0-generate-001`, prompt built from split image prompt.  
    - Credentials: Google Palm API.  
    - Edge cases: API quota limits, generation failures, prompt errors.  

  - **Generate audio**  
    - Type: `OpenAI (LangChain)`  
    - Role: Creates narration audio from the full_text output using OpenAI’s TTS capabilities.  
    - Configuration: Voice set as "echo", input is `message.content.full_text` from previous node.  
    - Credentials: OpenAI API.  
    - Edge cases: API errors, audio format issues.  

  - **Get List of background Audio**  
    - Type: `httpRequest`  
    - Role: Queries Openverse API for ambient background music files under CC licenses.  
    - Configuration: Query params: q=ambiente, license=cc0,by,by-sa, format=json.  
    - Edge cases: API downtime, empty results.  

  - **Download MP3**  
    - Type: `httpRequest`  
    - Role: Downloads a randomly selected background audio MP3 from the Openverse results.  
    - Configuration: URL chosen randomly from `$json.results`.  
    - Edge cases: Broken URLs, download failures.  

  - **Download Watermark**  
    - Type: `httpRequest`  
    - Role: Downloads a static watermark PNG from a fixed URL.  
    - Edge cases: URL unreachable, slow response.

#### 2.3 Media Upload

- **Overview:** Uploads all generated and downloaded media (images, narration audio, background music, watermark) to JsonCut file storage.
- **Nodes Involved:**  
  - `Upload logo`  
  - `Upload Background Images`  
  - `Upload Voice Audio`  
  - `Upload Background Music`  
  - `Background Image Urls` (aggregate uploads)  
  - `Merge all uploads`  
- **Node Details:**

  - **Upload logo**  
    - Type: `httpRequest`  
    - Role: Uploads the watermark PNG binary to JsonCut API.  
    - Configuration: POST to `https://api.jsoncut.com/api/v1/files/upload`, multipart form data with binary from `Download Watermark`.  
    - Credentials: JsonCut API Key (HTTP Header Auth).  
    - Output: JSON with `storageUrl`.  

  - **Upload Background Images**  
    - Type: `httpRequest`  
    - Role: Uploads each generated background image binary to JsonCut storage.  
    - Input: Binary data from `Generate an image`.  
    - Same API and auth as above.  

  - **Upload Voice Audio**  
    - Type: `httpRequest`  
    - Role: Uploads narration audio binary from `calculate audio duration` node.  
    - Same API and auth.  

  - **Upload Background Music**  
    - Type: `httpRequest`  
    - Role: Uploads the downloaded background MP3 binary.  
    - Same API and auth.  

  - **Background Image Urls**  
    - Type: `aggregate`  
    - Role: Aggregates uploaded image URLs into an array for job config.  
    - Input: Upload response JSONs.  

  - **Merge all uploads**  
    - Type: `merge`  
    - Role: Combines all upload results (logo, background images, voice audio, background music) into one JSON object for job creation.  
    - Inputs: Four inputs (logo, images, voice audio, background music).

#### 2.4 Video Job Creation & Monitoring

- **Overview:** Creates a JsonCut video rendering job using uploaded assets and configured parameters; polls job status until completion or failure.
- **Nodes Involved:**  
  - `Aggregate`  
  - `Create JsonCut Job`  
  - `Wait`  
  - `Check JsonCut job Status`  
  - `If Success`  
  - `If Error`  
  - `Error Stop`  
- **Node Details:**

  - **Aggregate**  
    - Type: `aggregate`  
    - Role: Aggregates all upload results from `Merge all uploads` to prepare job payload.  

  - **Create JsonCut Job**  
    - Type: `httpRequest`  
    - Role: Sends POST request to JsonCut API's `/jobs` endpoint to create a video job.  
    - Configuration:  
      - Video config: 1080x1920, 25 fps, MP4 format.  
      - Audio tracks: narration audio and background music with mixing volumes and timing.  
      - Clips: multiple image overlays with zoom effects, titles with text from script segments, watermark overlay.  
      - Duration dynamically calculated based on audio duration from `calculate audio duration` node.  
      - Uses expressions to retrieve upload URLs and narration text segments.  
    - Credentials: JsonCut API Key.  
    - Edge cases: API errors, invalid payload, missing uploads.  

  - **Wait**  
    - Type: `wait`  
    - Role: Pauses workflow 3 seconds before polling job status again (used for polling).  

  - **Check JsonCut job Status**  
    - Type: `httpRequest`  
    - Role: GET request to JsonCut API job status endpoint using job ID from creation response.  
    - Credentials: JsonCut API Key.  
    - Edge cases: Network errors, job ID invalid, timeout.  

  - **If Success**  
    - Type: `if`  
    - Role: Checks if job status is "COMPLETED". If yes, proceeds to download video; else loops or triggers error.  

  - **If Error**  
    - Type: `if`  
    - Role: Checks if job status is "FAILED" or "CANCELLED". If yes, stops workflow with error; else loops wait.  

  - **Error Stop**  
    - Type: `stopAndError`  
    - Role: Terminates workflow with error message "Failed to generate image".  

#### 2.5 Output Handling

- **Overview:** Downloads the completed video file from JsonCut and saves it along with metadata in a NocoDB database table.
- **Nodes Involved:**  
  - `Download Image` (actually downloads video file)  
  - `Save Video in NocoDB`  
- **Node Details:**

  - **Download Image**  
    - Type: `httpRequest`  
    - Role: Downloads the final video file from JsonCut using `outputFileId` from job completion data.  
    - Configuration: Accepts binary response (file).  
    - Credentials: JsonCut API Key.  
    - Output: Binary video file.  

  - **Save Video in NocoDB**  
    - Type: `nocoDb`  
    - Role: Creates a new row in a NocoDB table storing the video theme, setting, style, and binary video file.  
    - Configuration: Table and project IDs set; fields mapped from form input and video binary.  
    - Credentials: NocoDB API Token.  
    - Edge cases: API auth errors, size limits.

---

### 3. Summary Table

| Node Name                 | Node Type                     | Functional Role                          | Input Node(s)                     | Output Node(s)                     | Sticky Note                                                                                          |
|---------------------------|-------------------------------|----------------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------|
| On form submission         | formTrigger                   | Entry point; captures user video input | None                             | Generate content and Image Prompt, Get List of background Audio, Download Watermark | Provides form to input video theme, setting, image style                                            |
| Generate content and Image Prompt | OpenAI (LangChain)          | Generates video script & image prompts | On form submission               | Split Image Prompts, Generate audio | Generate Script and Background Image Prompts                                                        |
| Split Image Prompts        | splitOut                      | Splits image prompts array into single items | Generate content and Image Prompt | Generate an image                |                                                                                                    |
| Generate an image          | Google Gemini (PaLM)          | Generates background images from prompts | Split Image Prompts              | Upload Background Images          |                                                                                                    |
| Generate audio             | OpenAI (LangChain)            | Creates narration audio from script    | Generate content and Image Prompt | calculate audio duration          |                                                                                                    |
| Get List of background Audio | httpRequest                  | Fetches background music list          | On form submission               | Download MP3                     | download Random Background music                                                                   |
| Download MP3              | httpRequest                   | Downloads a random background music MP3 | Get List of background Audio     | Upload Background Music           |                                                                                                    |
| Download Watermark         | httpRequest                   | Downloads watermark image               | On form submission               | Upload logo                     | Download external ressources                                                                       |
| Upload logo                | httpRequest                   | Uploads watermark to JsonCut            | Download Watermark               | Merge all uploads                | Upload Files to JsonCut API                                                                         |
| Upload Background Images   | httpRequest                   | Uploads generated images to JsonCut     | Generate an image                | Background Image Urls             |                                                                                                    |
| Upload Voice Audio         | httpRequest                   | Uploads narration audio to JsonCut      | calculate audio duration         | Merge all uploads                |                                                                                                    |
| Upload Background Music    | httpRequest                   | Uploads background music to JsonCut     | Download MP3                    | Merge all uploads                |                                                                                                    |
| Background Image Urls      | aggregate                    | Aggregates uploaded image URLs          | Upload Background Images         | Merge all uploads                |                                                                                                    |
| Merge all uploads          | merge                        | Combines all uploaded file info         | Upload logo, Upload Background Images, Upload Voice Audio, Upload Background Music | Aggregate                       |                                                                                                    |
| Aggregate                 | aggregate                    | Aggregates merged upload results        | Merge all uploads               | Create JsonCut Job              | Create Job with Jsoncut API and wait for the result; alternative: JsonCut Community node            |
| Create JsonCut Job         | httpRequest                   | Creates video generation job on JsonCut | Aggregate                      | Wait                           |                                                                                                    |
| Wait                      | wait                         | Pauses workflow before rechecking job status | Create JsonCut Job             | Check JsonCut job Status          |                                                                                                    |
| Check JsonCut job Status   | httpRequest                   | Polls JsonCut job status                 | Wait                          | If Success                      |                                                                                                    |
| If Success                | if                            | Checks if job status is COMPLETED       | Check JsonCut job Status        | Download Image, If Error          |                                                                                                    |
| If Error                  | if                            | Checks if job status is FAILED or CANCELLED | If Success                    | Error Stop, Wait                 |                                                                                                    |
| Error Stop                | stopAndError                  | Stops workflow with error message       | If Error                      | None                          |                                                                                                    |
| Download Image             | httpRequest                   | Downloads completed video file           | If Success                    | Save Video in NocoDB             |                                                                                                    |
| Save Video in NocoDB       | nocoDb                       | Saves video metadata and binary in NocoDB | Download Image                | None                          | Stores the final output in a new Table Row with Theme, Setting, Style, Output Video File             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Trigger Node: `On form submission`**  
   - Type: `formTrigger`  
   - Configure form fields:  
     - Video Theme (textarea, required)  
     - Video Setting (textarea, required)  
     - Image Style (textarea, optional)  
   - Set a webhook ID for external form submissions.  

2. **Add `Generate content and Image Prompt` Node**  
   - Type: `OpenAI (LangChain)`  
   - Configure model: GPT-5-MINI  
   - Set prompt as per instructions to create JSON output with 5 image prompts, full narration text, 4 text segments, and caption.  
   - Use expression references to form fields for theme, setting, style.  
   - Connect output from `On form submission`.  
   - Provide OpenAI API credentials.

3. **Add `Split Image Prompts` Node**  
   - Type: `splitOut`  
   - Configure to split on `message.content.image_prompts`.  
   - Connect from `Generate content and Image Prompt`.  

4. **Add `Generate an image` Node**  
   - Type: `Google Gemini (PaLM)`  
   - Set prompt to generate 9:16 images with no text, using image prompt from split output.  
   - Use model `imagen-4.0-generate-001`.  
   - Connect from `Split Image Prompts`.  
   - Provide Google Palm API credentials.

5. **Add `Generate audio` Node**  
   - Type: `OpenAI (LangChain)`  
   - Input: `message.content.full_text` from `Generate content and Image Prompt`.  
   - Voice: "echo" (or configured voice)  
   - Provide OpenAI API credentials.  
   - Connect from `Generate content and Image Prompt`.  

6. **Add `Get List of background Audio` Node**  
   - Type: `httpRequest`  
   - GET `https://api.openverse.engineering/v1/audio/`  
   - Query parameters: q=ambiente, license=cc0,by,by-sa, format=json  
   - Connect from `On form submission`.  

7. **Add `Download MP3` Node**  
   - Type: `httpRequest`  
   - URL expression: randomly select one from `$json.results[].url` from previous node.  
   - Connect from `Get List of background Audio`.  

8. **Add `Download Watermark` Node**  
   - Type: `httpRequest`  
   - URL: `https://img.icons8.com/?size=100&id=532&format=png&color=000000`  
   - Connect from `On form submission`.  

9. **Add Upload Nodes:**  
   - Create four `httpRequest` nodes for uploading files to JsonCut:  
     - Upload logo: upload watermark binary from `Download Watermark`.  
     - Upload Background Images: upload binaries from `Generate an image`.  
     - Upload Voice Audio: upload binary from `Generate audio` after calculating duration.  
     - Upload Background Music: upload binary from `Download MP3`.  
   - Configure all for POST `https://api.jsoncut.com/api/v1/files/upload` with multipart form-data, header auth with JsonCut API Key.  

10. **Add `Background Image Urls` Node**  
    - Type: `aggregate`  
    - Aggregate `data.storageUrl` fields from uploaded background images.  
    - Connect from `Upload Background Images`.  

11. **Add `Merge all uploads` Node**  
    - Type: `merge` with 4 inputs (logo, background images aggregate, voice audio, background music).  
    - Connect from respective upload nodes.  

12. **Add `Aggregate` Node**  
    - Type: `aggregate`  
    - Aggregate all upload results from `Merge all uploads` into `upload_results`.  
    - Connect from `Merge all uploads`.  

13. **Add `Create JsonCut Job` Node**  
    - Type: `httpRequest` POST to `https://api.jsoncut.com/api/v1/jobs`  
    - JSON body: configure video job with parameters from aggregated uploads and calculated audio duration; includes clips with image overlays, titles, watermark, audio tracks.  
    - Use expressions to pull URLs and text segments.  
    - Connect from `Aggregate`.  

14. **Add `Wait` Node**  
    - Type: `wait` with 3 seconds delay.  
    - Connect from `Create JsonCut Job`.  

15. **Add `Check JsonCut job Status` Node**  
    - Type: `httpRequest` GET job status by jobId from `Create JsonCut Job`.  
    - Connect from `Wait`.  

16. **Add `If Success` Node**  
    - Type: `if` node checking if `status == "COMPLETED"`.  
    - Connect from `Check JsonCut job Status`.  

17. **Add `If Error` Node**  
    - Type: `if` node checking if `status == "FAILED"` or `status == "CANCELLED"`.  
    - Connect from `If Success` (on false branch).  

18. **Add `Error Stop` Node**  
    - Type: `stopAndError` with message "Failed to generate image".  
    - Connect from `If Error` (on true branch).  

19. **Connect `If Error` false branch back to `Wait`** for polling loop.  

20. **Add `Download Image` Node**  
    - Type: `httpRequest` to download video file as binary from JsonCut using `outputFileId`.  
    - Connect from `If Success` true branch.  

21. **Add `Save Video in NocoDB` Node**  
    - Type: `nocoDb` create operation.  
    - Map fields: Theme, Setting, Style from form inputs; Output as binary video file.  
    - Provide NocoDB API Token credentials.  
    - Connect from `Download Image`.  

22. **Add `calculate audio duration` Node** (Python code)  
    - Type: `code` node (Python).  
    - Input: binary audio data from `Generate audio`.  
    - Output: duration_seconds, duration_minutes, bitrate, file_size_bytes in JSON.  
    - Connect from `Generate audio` before file upload.  
    - Connect `Upload Voice Audio` from this node’s output.  

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The JsonCut Community Node can be used as an alternative to the HTTP request nodes for JsonCut API interaction | https://github.com/jsoncut/n8n-nodes-jsoncut                                                        |
| Example form input: Theme "Overcoming struggles / personal growth"; Setting "Introspective, deep thinking..."; Style "Watercolor painting with muted colors" | Sticky Note4 in the workflow                                                                       |
| Example output video link preview: [Google Drive Video Preview](https://drive.google.com/file/d/1Cl0KwgRgcuBPVdGgL-nqAcheyvfVXttD/preview) | Sticky Note8                                                                                        |
| Background audio is sourced from Openverse API under Creative Commons licenses CC0, BY, BY-SA                   | API: https://api.openverse.engineering/v1/audio/                                                    |

---

This documentation provides a detailed and structured overview of the workflow “Create viral storytelling videos with GPT, Gemini & JsonCut from text prompts”. It enables advanced users and AI agents to fully understand, reproduce, and modify the automation pipeline with confidence, while anticipating potential failure modes and integration points.