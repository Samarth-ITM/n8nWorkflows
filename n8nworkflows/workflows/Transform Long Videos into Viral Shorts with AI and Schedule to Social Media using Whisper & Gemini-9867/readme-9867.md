Transform Long Videos into Viral Shorts with AI and Schedule to Social Media using Whisper & Gemini

https://n8nworkflows.xyz/workflows/transform-long-videos-into-viral-shorts-with-ai-and-schedule-to-social-media-using-whisper---gemini-9867


# Transform Long Videos into Viral Shorts with AI and Schedule to Social Media using Whisper & Gemini

### 1. Workflow Overview

This workflow automates the transformation of long-form videos into viral short clips optimized for TikTok, Instagram Reels, and YouTube Shorts, leveraging AI transcription (OpenAI Whisper) and content analysis (Google Gemini). It orchestrates video upload, audio extraction, transcription, AI-driven selection of high-engagement video segments, automated cutting via FFmpeg commands, and scheduled posting across multiple social platforms.

Logical blocks:

- **1.1 Intake & Form Submission:** User uploads a long video via a web form.
- **1.2 FFmpeg Audio Extraction Loop:** Uploads video, extracts audio track, and polls job status until complete.
- **1.3 Transcription & Parsing:** Transcribes audio with Whisper to obtain word-level timestamps and prepares data for AI.
- **1.4 AI Selection with Gemini:** Uses Google Gemini and a custom AI agent to identify the most viral short segments with precise timestamps and social media metadata.
- **1.5 FFmpeg Short Clip Generation Loop:** For each selected viral clip, submits cutting jobs to FFmpeg service, polls for completion, and downloads processed short videos.
- **1.6 Scheduling:** Schedules the finalized shorts to TikTok, Instagram, and YouTube with tailored titles and descriptions, spaced one per day.

---

### 2. Block-by-Block Analysis

#### 2.1 Intake & Form Submission

**Overview:**  
Captures user input by uploading a long video file through a web form, triggering the entire process.

**Nodes Involved:**  
- Form: Upload Video  
- Sticky Note (Section: Intake & Form)

**Node Details:**

- **Form: Upload Video**  
  - Type: `formTrigger`  
  - Role: Accepts video file upload from user.  
  - Configuration: Single file input accepting video/* MIME types. Form path and webhook ID configured for API endpoint.  
  - Inputs: External HTTP request (form submission).  
  - Outputs: Binary video data passed downstream.  
  - Edge cases: File size limits, unsupported formats, form submission errors.

- **Sticky Note (Intake & Form)**  
  - Provides section labeling and overview of this block.

---

#### 2.2 FFmpeg Audio Extraction Loop

**Overview:**  
Uploads the video to an external FFmpeg job service to extract a mono 16kHz WAV audio track, then polls the job status until completion.

**Nodes Involved:**  
- FFmpeg: Extract Audio  
- Wait 10s (Audio)  
- Check Audio Job Status  
- Is Audio Job Completed? (If node)  
- Wait 5s & Retry (Audio)  
- Sticky Note (Section: FFmpeg Audio Job Loop)

**Node Details:**

- **FFmpeg: Extract Audio**  
  - Type: `httpRequest`  
  - Role: Submits POST multipart/form-data request to upload video and run FFmpeg command extracting audio (`-map a:0 -vn -ac 1 -ar 16000 -c:a pcm_s16le`).  
  - Config: Uses HTTP Header Authentication credential for API access.  
  - Inputs: Binary video file from form trigger.  
  - Outputs: JSON job info with job_id.  
  - Failures: Network errors, API auth failure, invalid video input.

- **Wait 10s (Audio)**  
  - Type: `wait`  
  - Role: Delays 10 seconds before polling job status to allow processing.  
  - Inputs: After job submission.  
  - Outputs: Triggers next status check.

- **Check Audio Job Status**  
  - Type: `httpRequest`  
  - Role: GET request to check FFmpeg job status using job_id.  
  - Config: Authenticated with same HTTP Header Auth.  
  - Outputs: Job JSON including status field.

- **Is Audio Job Completed?**  
  - Type: `if`  
  - Role: Tests if `status === "finished"`.  
  - True branch: Proceeds to download audio.  
  - False branch: Wait 5s and retry.

- **Wait 5s & Retry (Audio)**  
  - Type: `wait`  
  - Role: Delay before re-checking job status if incomplete.

- **Sticky Note (FFmpeg Audio Job Loop)**  
  - Describes the retry logic and nodes involved.

---

#### 2.3 Transcription & Parsing

**Overview:**  
Transcribes the extracted audio using OpenAI Whisper with verbose JSON output including word-level timestamps. Parses and cleans transcription data for AI processing.

**Nodes Involved:**  
- Download Audio  
- Whisper: Transcribe with Timestamps  
- Parse Whisper Results  
- Sticky Note (Section: Transcription & Parsing)

**Node Details:**

- **Download Audio**  
  - Type: `httpRequest`  
  - Role: Downloads the extracted WAV audio from FFmpeg job service.  
  - Auth: HTTP Header Auth.  
  - Inputs: From "Is Audio Job Completed?" node true branch.  
  - Outputs: Binary audio data.

- **Whisper: Transcribe with Timestamps**  
  - Type: `httpRequest` (OpenAI API)  
  - Role: Sends audio binary to OpenAI Whisper endpoint for transcription with word-level timestamps (`response_format=verbose_json`).  
  - Auth: OpenAI API key credential.  
  - Outputs: JSON with transcription text, word timestamps, duration.

- **Parse Whisper Results**  
  - Type: `code` (JavaScript)  
  - Role: Processes Whisper output to:  
    - Round durations to 3 decimals  
    - Create simplified word timestamp array `{w, s, e}`  
    - Passes cleaned transcript and metadata downstream.  
  - Inputs: Whisper JSON response.  
  - Outputs: Enhanced JSON with `video_duration`, `words_llm`, `text_llm`.

- **Sticky Note (Transcription & Parsing)**  
  - Lists the nodes and purpose of this block.

---

#### 2.4 AI Selection with Gemini

**Overview:**  
Uses Google Gemini Chat Model and a Langchain AI agent to analyze transcription and word timestamps, selecting 3–15 viral short segments optimized for social media with strict timing constraints. Parses AI output into normalized clip metadata and generates FFmpeg commands for cutting.

**Nodes Involved:**  
- AI Agent - Select Viral Clips  
- Google Gemini Chat Model  
- Parse Gemini Analysis  
- Sticky Note (Section: AI Selection)

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: `lmChatGoogleGemini`  
  - Role: Provides Google Gemini LLM access with configured API key.  
  - Inputs: None directly, used as AI model for agent.  
  - Outputs: AI responses to agent.

- **AI Agent - Select Viral Clips**  
  - Type: `agent` (Langchain)  
  - Role: Runs prompt-based AI agent that:  
    - Reads full transcript and word timestamps  
    - Applies strict FFmpeg timing contract (absolute seconds, clip length 15–60s, no mid-word cuts)  
    - Returns JSON object with array of clips including start/end times and social media metadata.  
  - Inputs: Cleaned transcript and timestamps from "Parse Whisper Results".  
  - Outputs: Raw AI JSON response.

- **Parse Gemini Analysis**  
  - Type: `code` (JavaScript)  
  - Role: Parses Gemini AI JSON output, cleans code fences, extracts valid JSON, normalizes clip start/end times (handles percentages if needed), clamps duration 15-60s, constructs FFmpeg cut commands, and prepares individual clip items with binary video for cutting.  
  - Inputs: AI agent JSON output and original video binary from form trigger.  
  - Outputs: Array of clips with cutting commands, timestamps, and descriptions.

- **Sticky Note (AI Selection)**  
  - Describes nodes and their roles.

---

#### 2.5 FFmpeg Short Clip Generation Loop

**Overview:**  
Submits cutting jobs to FFmpeg service for each viral clip, waits and polls for job completion, retries if necessary, then downloads the finished short video.

**Nodes Involved:**  
- FFmpeg: Upload & Cut  
- Wait 10s (Short)  
- Check Short Job Status  
- Is Short Job Completed? (If node)  
- Wait 5s & Retry (Short)  
- Download Short  
- Sticky Note (Section: FFmpeg Short Job Loop)

**Node Details:**

- **FFmpeg: Upload & Cut**  
  - Type: `httpRequest`  
  - Role: Uploads binary video along with FFmpeg cut command to produce short clip (MP4, h264_nvenc, scaled and cropped to 1080x1920).  
  - Auth: HTTP Header Auth.  
  - Inputs: Individual clip data from "Parse Gemini Analysis" (includes binary video and command).  
  - Outputs: Job info with job_id for clip.

- **Wait 10s (Short)**  
  - Type: `wait`  
  - Role: Waits 10 seconds before polling job status.

- **Check Short Job Status**  
  - Type: `httpRequest`  
  - Role: Checks FFmpeg job status for short clip.  
  - Auth: HTTP Header Auth.  
  - Outputs: Job status JSON.

- **Is Short Job Completed?**  
  - Type: `if`  
  - Role: Checks if status is "finished".  
  - True branch: Download short video.  
  - False branch: Wait 5s and retry.

- **Wait 5s & Retry (Short)**  
  - Type: `wait`  
  - Role: Delay before re-checking short job status.

- **Download Short**  
  - Type: `httpRequest`  
  - Role: Downloads the finalized short video binary from FFmpeg service.  
  - Auth: HTTP Header Auth.  
  - Outputs: Binary short video.

- **Sticky Note (FFmpeg Short Job Loop)**  
  - Describes retry logic and involved nodes.

---

#### 2.6 Scheduling

**Overview:**  
Schedules the downloaded short videos to TikTok, Instagram, and YouTube, setting platform-specific titles and descriptions, and scheduling posts daily at 15:00 Europe/Madrid time starting tomorrow.

**Nodes Involved:**  
- Download Short (input node)  
- Schedule to TikTok, Instagram, and YouTube  
- Sticky Note (Section: Scheduling)

**Node Details:**

- **Schedule to TikTok, Instagram, and YouTube**  
  - Type: `uploadPost` (custom node for app.upload-post.com)  
  - Role: Uploads video binary to multiple social platforms with tailored metadata and scheduled date.  
  - Config: Uses expressions to extract titles and descriptions from the parsed Gemini analysis output.  
  - Scheduling: Posts start the day after upload, at 15:00 Madrid time, one clip per day incremented.  
  - Auth: Upload-Post API token credential.  
  - Inputs: Binary short video and metadata from "Download Short".  
  - Outputs: Confirmation of scheduling or upload.

- **Sticky Note (Scheduling)**  
  - Lists scheduling-related nodes.

---

### 3. Summary Table

| Node Name                         | Node Type                         | Functional Role                                   | Input Node(s)                          | Output Node(s)                          | Sticky Note                                                                                                     |
|----------------------------------|----------------------------------|-------------------------------------------------|--------------------------------------|----------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Form: Upload Video                | formTrigger                      | Intake video upload from user                     | (external HTTP)                      | FFmpeg: Extract Audio                  | **Intake & Form** - Form: Upload Video                                                                         |
| FFmpeg: Extract Audio             | httpRequest                     | Upload video and extract audio                    | Form: Upload Video                   | Wait 10s (Audio)                      | **FFmpeg Audio Job Loop** - FFmpeg: Extract Audio                                                              |
| Wait 10s (Audio)                 | wait                            | Delay before polling audio job status             | FFmpeg: Extract Audio                | Check Audio Job Status                 | **FFmpeg Audio Job Loop**                                                                                        |
| Check Audio Job Status            | httpRequest                     | Poll FFmpeg audio job status                       | Wait 10s (Audio), Wait 5s & Retry   | Is Audio Job Completed?               | **FFmpeg Audio Job Loop**                                                                                        |
| Is Audio Job Completed?           | if                              | Check if audio job finished                        | Check Audio Job Status               | Download Audio / Wait 5s & Retry (Audio) | **FFmpeg Audio Job Loop**                                                                                        |
| Wait 5s & Retry (Audio)           | wait                            | Delay before retrying audio job status check      | Is Audio Job Completed? (false)      | Check Audio Job Status                 | **FFmpeg Audio Job Loop**                                                                                        |
| Download Audio                   | httpRequest                     | Download extracted audio track                     | Is Audio Job Completed? (true)        | Whisper: Transcribe with Timestamps   | **Transcription & Parsing** - Download Audio                                                                     |
| Whisper: Transcribe with Timestamps | httpRequest (OpenAI Whisper)  | Transcribe audio with word-level timestamps       | Download Audio                      | Parse Whisper Results                  | **Transcription & Parsing** - Whisper: Transcribe with Timestamps                                               |
| Parse Whisper Results             | code                            | Clean and prepare transcription data              | Whisper: Transcribe with Timestamps | AI Agent - Select Viral Clips          | **Transcription & Parsing**                                                                                        |
| Google Gemini Chat Model          | lmChatGoogleGemini              | Provide AI language model for Gemini               | (no direct input)                   | AI Agent - Select Viral Clips (model) | **AI Selection**                                                                                                  |
| AI Agent - Select Viral Clips     | agent (Langchain)               | Select viral clips and generate JSON output        | Parse Whisper Results, Gemini Model | Parse Gemini Analysis                  | **AI Selection**                                                                                                  |
| Parse Gemini Analysis             | code                            | Parse Gemini output, normalize clips, generate FFmpeg commands | AI Agent - Select Viral Clips       | FFmpeg: Upload & Cut                  | **AI Selection**                                                                                                  |
| FFmpeg: Upload & Cut             | httpRequest                     | Upload video binary and FFmpeg cut command         | Parse Gemini Analysis               | Wait 10s (Short)                      | **FFmpeg Short Job Loop** - FFmpeg: Upload & Cut                                                                 |
| Wait 10s (Short)                | wait                            | Delay before polling short clip job status          | FFmpeg: Upload & Cut                | Check Short Job Status                 | **FFmpeg Short Job Loop**                                                                                         |
| Check Short Job Status            | httpRequest                     | Poll FFmpeg short clip job status                   | Wait 10s (Short), Wait 5s & Retry   | Is Short Job Completed?               | **FFmpeg Short Job Loop**                                                                                         |
| Is Short Job Completed?           | if                              | Check if short clip job finished                    | Check Short Job Status              | Download Short / Wait 5s & Retry (Short) | **FFmpeg Short Job Loop**                                                                                         |
| Wait 5s & Retry (Short)           | wait                            | Delay before retrying short clip job status check  | Is Short Job Completed? (false)      | Check Short Job Status                 | **FFmpeg Short Job Loop**                                                                                         |
| Download Short                  | httpRequest                     | Download completed short clip                       | Is Short Job Completed? (true)       | Schedule to TikTok, Instagram, YouTube | **Scheduling**                                                                                                    |
| Schedule to TikTok, Instagram, and YouTube | uploadPost                   | Schedule and upload shorts to social platforms     | Download Short                     | (workflow end)                       | **Scheduling**                                                                                                    |
| Sticky Note (AUTO-SHORTS GENERATOR) | stickyNote                   | Overview and setup instructions                      | -                                  | -                                    | **AUTO-SHORTS GENERATOR** overview and quick setup instructions                                                  |
| Sticky Note (Intake & Form)       | stickyNote                      | Section label                                       | -                                  | -                                    | **Intake & Form** section                                                                                         |
| Sticky Note (Transcription & Parsing) | stickyNote                  | Section label                                       | -                                  | -                                    | **Transcription & Parsing** section                                                                               |
| Sticky Note (AI Selection)        | stickyNote                      | Section label                                       | -                                  | -                                    | **AI Selection** section                                                                                          |
| Sticky Note (FFmpeg Audio Job Loop) | stickyNote                   | Section label                                       | -                                  | -                                    | **FFmpeg Audio Job Loop** section                                                                                 |
| Sticky Note (FFmpeg Short Job Loop) | stickyNote                   | Section label                                       | -                                  | -                                    | **FFmpeg Short Job Loop** section                                                                                 |
| Sticky Note (Scheduling)          | stickyNote                      | Section label                                       | -                                  | -                                    | **Scheduling** section                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node: "Form: Upload Video"**  
   - Type: `formTrigger`  
   - Set path and webhook ID (e.g., `/352a575b-42cf-4e9b-8d76-6acec4e40402`)  
   - Add a single required file field accepting `video/*` MIME types  
   - This node receives the uploaded video.

2. **Create HTTP Request Node: "FFmpeg: Extract Audio"**  
   - Method: POST  
   - URL: `https://api.upload-post.com/api/uploadposts/ffmpeg/jobs/upload`  
   - Content-Type: multipart-form-data  
   - Body Parameters:  
     - `file` (binary, from form upload)  
     - `full_command`: `ffmpeg -y -i {input} -map a:0 -vn -ac 1 -ar 16000 -c:a pcm_s16le {output}`  
     - `output_extension`: `"wav"`  
   - Authentication: HTTP Header Auth with Upload-post API token  
   - Connect input from "Form: Upload Video"

3. **Create Wait Node: "Wait 10s (Audio)"**  
   - Wait 10 seconds  
   - Connect from "FFmpeg: Extract Audio"

4. **Create HTTP Request Node: "Check Audio Job Status"**  
   - Method: GET  
   - URL: `https://api.upload-post.com/api/uploadposts/ffmpeg/jobs/{{ $json.job_id }}`  
   - Auth: HTTP Header Auth  
   - Connect from "Wait 10s (Audio)" and from retry wait node in step 7

5. **Create If Node: "Is Audio Job Completed?"**  
   - Condition: `$json.status === "finished"`  
   - Connect from "Check Audio Job Status"  
   - True branch: proceed to download audio  
   - False branch: retry loop

6. **Create Wait Node: "Wait 5s & Retry (Audio)"**  
   - Wait 5 seconds  
   - Connect False branch from "Is Audio Job Completed?" to this node  
   - Connect output back to "Check Audio Job Status"

7. **Create HTTP Request Node: "Download Audio"**  
   - Method: GET  
   - URL: `https://api.upload-post.com/api/uploadposts/ffmpeg/jobs/{{ $json.job_id }}/download`  
   - Auth: HTTP Header Auth  
   - Connect True branch from "Is Audio Job Completed?"

8. **Create HTTP Request Node: "Whisper: Transcribe with Timestamps"**  
   - POST to `https://api.openai.com/v1/audio/transcriptions`  
   - Content-Type: multipart-form-data  
   - Body parameters:  
     - `file`: binary audio from previous node  
     - `model`: `whisper-1`  
     - `response_format`: `verbose_json`  
     - `timestamp_granularities[]`: `word`  
   - Authentication: OpenAI API key  
   - Connect from "Download Audio"

9. **Create Code Node: "Parse Whisper Results"**  
   - JavaScript code to: round durations to 3 decimals, create simplified word timestamp array, pass cleaned transcript and duration downstream.  
   - Connect from "Whisper: Transcribe with Timestamps"

10. **Create Google Gemini Chat Model Node: "Google Gemini Chat Model"**  
    - Configure with Google AI Studio API key  
    - No direct input connections; used as LLM for agent node

11. **Create Agent Node: "AI Agent - Select Viral Clips"**  
    - Prompt with detailed instructions for viral clip selection (see node parameters in overview)  
    - Use Google Gemini node as language model  
    - Connect input from "Parse Whisper Results"  
    - Connect output to "Parse Gemini Analysis"

12. **Create Code Node: "Parse Gemini Analysis"**  
    - JavaScript code to parse AI JSON, validate timestamps, normalize durations, and generate FFmpeg cut commands  
    - Connect input from "AI Agent - Select Viral Clips"  
    - Use original video binary from the form submission node (pass binary through context or workflow data)

13. **Create HTTP Request Node: "FFmpeg: Upload & Cut"**  
    - POST multipart-form-data to `https://api.upload-post.com/api/uploadposts/ffmpeg/jobs/upload`  
    - Body parameters:  
      - `file`: binary video clip from "Parse Gemini Analysis"  
      - `full_command`: FFmpeg cut command from parsed data  
      - `output_extension`: "mp4"  
    - Auth: HTTP Header Auth  
    - Connect from "Parse Gemini Analysis"

14. **Create Wait Node: "Wait 10s (Short)"**  
    - Wait 10 seconds  
    - Connect from "FFmpeg: Upload & Cut"

15. **Create HTTP Request Node: "Check Short Job Status"**  
    - Same structure as audio job status check, but for short clip job_id  
    - Connect from "Wait 10s (Short)" and retry wait node

16. **Create If Node: "Is Short Job Completed?"**  
    - Condition: `$json.status === "finished"`  
    - True branch: download short clip  
    - False branch: retry loop

17. **Create Wait Node: "Wait 5s & Retry (Short)"**  
    - Wait 5 seconds  
    - Connect False branch from "Is Short Job Completed?" to this node  
    - Connect output back to "Check Short Job Status"

18. **Create HTTP Request Node: "Download Short"**  
    - GET request to download finished short clip  
    - Auth: HTTP Header Auth  
    - Connect True branch from "Is Short Job Completed?"

19. **Create UploadPost Node: "Schedule to TikTok, Instagram, and YouTube"**  
    - Configure user account and API credentials  
    - Set platform list: TikTok, Instagram, YouTube  
    - Map titles and descriptions from Gemini parsed data fields for each platform  
    - Set scheduledDate with expression to start tomorrow 15:00 Europe/Madrid and increment per item index  
    - Connect input from "Download Short"

20. **Add Sticky Notes** at appropriate places for documentation, labeling blocks exactly as in the original workflow.

21. **Credentials Setup**  
   - OpenAI API key in Whisper node  
   - Google AI Studio API key in Gemini node  
   - Upload-Post API token in HTTP Request nodes and UploadPost scheduling node with HTTP Header Auth credential type.

22. **Test Flow End-to-End**  
   - Upload a test video via form URL  
   - Monitor logs for transcription, AI analysis, clip generation, and scheduling success.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Detailed usage instructions and quick setup steps are documented in the main "AUTO-SHORTS GENERATOR" sticky note. It covers API key setup for OpenAI Whisper and Google Gemini, Upload-Post account creation, and scheduling adjustments.                                                                                   | See Sticky Note "AUTO-SHORTS GENERATOR" in workflow                                                                    |
| Upload-Post platform is key for FFmpeg job execution and multi-platform scheduling. Sign up at https://app.upload-post.com, connect social accounts, and generate API token for credential setup.                                                                                                                        | https://app.upload-post.com                                                                                           |
| The workflow strictly enforces FFmpeg timing contract for clips: absolute seconds (dot decimals), 15-60 seconds length, no mid-word cuts, and uses silent moments for cuts. This ensures high-quality viral-ready shorts.                                                                                                   | AI Agent prompt and "Parse Gemini Analysis" node code                                                                |
| Scheduling posts uses Europe/Madrid timezone and posts start from the day after upload at 15:00, posting one short each day sequentially. Adjust timezone or time in scheduling node as needed.                                                                                                                           | Scheduling node expression                                                                                           |
| This workflow depends on external APIs: OpenAI Whisper for transcription, Google Gemini for AI analysis, and Upload-Post for FFmpeg processing and social media posting. Ensure all credentials and tokens are properly configured to avoid authentication errors.                                                        | Credentials in n8n for OpenAI, Google Palm API, Upload-Post HTTP Header Auth and UploadPost API                      |
| Network failures or API rate limits may cause job failures or delays; retry logic with waits is implemented in both audio extraction and short clip generation loops.                                                                                                                                                | Retry loops with Wait and If nodes for job completion checks                                                         |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.