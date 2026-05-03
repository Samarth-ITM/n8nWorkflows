AI YouTube Shorts Creator 🤖🎞️: Prompt-Based Clipping, Dubbing & Social Upload

https://n8nworkflows.xyz/workflows/ai-youtube-shorts-creator--------prompt-based-clipping--dubbing---social-upload-14191


# AI YouTube Shorts Creator 🤖🎞️: Prompt-Based Clipping, Dubbing & Social Upload

# 1. Workflow Overview

This workflow automates the creation of a short-form video clip from a YouTube video based on a user prompt. It:

1. receives a YouTube video ID and a natural-language prompt,
2. fetches the transcript,
3. uses an OpenAI model to locate the relevant timestamp range,
4. downloads the source video,
5. uploads the source video to public storage,
6. sends the public URL plus timestamps to Fal.run to trim the clip,
7. polls until the clip is ready,
8. downloads the final clip,
9. stores it and optionally publishes it to social platforms.

The workflow is designed for prompt-based repurposing of long YouTube videos into Shorts/Reels/TikTok-style clips.

## 1.1 Input Reception and Initial Parameters
This block starts the workflow manually and defines the two core inputs:
- `VIDEO ID`
- `PROMPT`

It then splits into two parallel branches:
- transcript retrieval
- source video download

## 1.2 Transcript Retrieval and Preparation
This block calls an external transcript API and extracts the usable transcript array from the response.

## 1.3 AI Timestamp Extraction
This block sends the transcript and user prompt to an LLM, forcing a structured JSON response that identifies:
- whether the requested content was found,
- start time,
- end time,
- duration

## 1.4 Source Video Download and Public Storage
This block downloads the original YouTube video via a RapidAPI service, waits for the file to become available, fetches the actual file, and uploads it to Google Drive and BunnyCDN via FTP.

## 1.5 Clip Creation via Fal.run
This block builds a public video URL, requests trimming from Fal.run, and polls for completion until the job is ready.

## 1.6 Final Clip Retrieval and Distribution
This block downloads the trimmed clip and distributes it to:
- Google Drive
- BunnyCDN FTP
- YouTube via Upload-Post
- TikTok via Upload-Post
- Postiz upload, then Instagram via Postiz

## 1.7 Embedded Notes and Setup Guidance
Several sticky notes provide operational guidance, required credentials, public links, and platform setup notes.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Initial Parameters

### Overview
This block initializes the workflow manually and defines the input values used everywhere else. It is the main entry point and creates two parallel execution paths: transcript analysis and video download.

### Nodes Involved
- `When clicking ‘Execute workflow’`
- `Set params`

### Node Details

#### When clicking ‘Execute workflow’
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry point for testing or ad hoc execution.
- **Configuration choices:** No special parameters; starts when executed manually in the editor.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: none
  - Output: `Set params`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - None functionally; only requires manual execution.
- **Sub-workflow reference:** None.

#### Set params
- **Type and technical role:** `n8n-nodes-base.set`; defines reusable workflow inputs.
- **Configuration choices:** Creates two string fields:
  - `VIDEO ID` = `eLz22nWmiuI`
  - `PROMPT` = `How to obtain Gemini API`
- **Key expressions or variables used:** These values are later referenced using expressions such as:
  - `$('Set params').item.json['VIDEO ID']`
  - `$('Set params').item.json.PROMPT`
- **Input and output connections:**
  - Input: `When clicking ‘Execute workflow’`
  - Outputs:
    - `Generate transcript`
    - `Download Video`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Invalid YouTube video ID causes downstream transcript/video retrieval failures.
  - A vague prompt may produce weak timestamp detection.
- **Sub-workflow reference:** None.

---

## 2.2 Transcript Retrieval and Preparation

### Overview
This block fetches the transcript for the YouTube video and extracts the transcript array expected by the LLM block. It converts the raw API response into a cleaner structure for later processing.

### Nodes Involved
- `Generate transcript`
- `Get transcript`

### Node Details

#### Generate transcript
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls the external transcript API.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://www.youtube-transcript.io/api/transcripts`
  - Authentication: generic header auth
  - Sends JSON body
  - Explicit `Content-Type: application/json`
  - Request body contains:
    - `ids: [VIDEO ID]`
- **Key expressions or variables used:**
  - `{{ $json['VIDEO ID'] }}`
- **Input and output connections:**
  - Input: `Set params`
  - Output: `Get transcript`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Invalid or missing API credential.
  - Transcript unavailable for the video.
  - Video has no captions.
  - Rate limiting or API response shape changes.
  - If `tracks[0]` is absent later, the next node fails.
- **Sub-workflow reference:** None.

#### Get transcript
- **Type and technical role:** `n8n-nodes-base.set`; extracts the transcript array from the API response.
- **Configuration choices:** Assigns:
  - `transcript = $json.tracks[0].transcript`
- **Key expressions or variables used:**
  - `={{ $json.tracks[0].transcript }}`
- **Input and output connections:**
  - Input: `Generate transcript`
  - Output: `Basic LLM Chain`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - `tracks` missing or empty.
  - API payload format different from expected.
  - Transcript not in `tracks[0]`.
- **Sub-workflow reference:** None.

---

## 2.3 AI Timestamp Extraction

### Overview
This block analyzes the transcript using an OpenAI chat model and returns a structured JSON object containing the detected clip boundaries. It is the core intelligence of the workflow.

### Nodes Involved
- `Basic LLM Chain`
- `OpenAI Chat Model`
- `Structured Output Parser`
- `Output found?`

### Node Details

#### Basic LLM Chain
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`; orchestrates prompt + transcript analysis through a connected language model.
- **Configuration choices:**
  - Prompt type: defined manually
  - Main input text combines:
    - requested content from `PROMPT`
    - full transcript serialized with `JSON.stringify($json.transcript)`
  - A system-style instruction defines a “VIDEO TRANSCRIPT TIMESTAMP EXTRACTOR”
  - Requires strict JSON-only output
  - Structured output parser enabled
- **Key expressions or variables used:**
  - `{{ $('Set params').item.json.PROMPT }}`
  - `{{ JSON.stringify($json.transcript) }}`
- **Input and output connections:**
  - Main input: `Get transcript`
  - AI language model input: `OpenAI Chat Model`
  - AI output parser input: `Structured Output Parser`
  - Main output: `Output found?`
- **Version-specific requirements:** Type version `1.9`; requires compatible LangChain nodes in the n8n instance.
- **Edge cases or potential failure types:**
  - Transcript too large for model context window.
  - Hallucinated timestamps if transcript is noisy.
  - Non-JSON model response if parser enforcement fails.
  - Weak prompt matching when user request is ambiguous.
- **Sub-workflow reference:** None.

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; provides the LLM used by the chain.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Uses OpenAI credentials
  - No extra built-in tools configured
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Output via AI connector to `Basic LLM Chain`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - OpenAI auth errors.
  - Model availability changes.
  - Token/context limits for long transcripts.
  - API latency or quota exhaustion.
- **Sub-workflow reference:** None.

#### Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces a schema-like JSON result.
- **Configuration choices:** Example schema:
  - `found` boolean
  - `start_time` integer
  - `end_time` integer
  - `duration` integer
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Output via AI parser connector to `Basic LLM Chain`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Parser may fail if model returns malformed JSON.
  - Null outputs may need explicit handling downstream.
- **Sub-workflow reference:** None.

#### Output found?
- **Type and technical role:** `n8n-nodes-base.if`; checks whether the LLM found a matching segment.
- **Configuration choices:**
  - Condition checks whether `output.found` is `true`
  - Implemented using a boolean “true” operator on `={{ $json.output.found }}`
- **Key expressions or variables used:**
  - `={{ $json.output.found }}`
- **Input and output connections:**
  - Input: `Basic LLM Chain`
  - True output: `Set video url`
  - False output: none
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - If parser output is missing, condition may not behave as intended.
  - If `found` is `false`, the workflow effectively stops for this branch.
- **Sub-workflow reference:** None.

---

## 2.4 Source Video Download and Public Storage

### Overview
This block retrieves the original YouTube video from a downloader API, waits for delayed availability, downloads the actual file, and stores it in both Google Drive and BunnyCDN FTP.

### Nodes Involved
- `Download Video`
- `Wait`
- `Get video`
- `Upload file`
- `Upload to FTP`

### Node Details

#### Download Video
- **Type and technical role:** `n8n-nodes-base.httpRequest`; initiates retrieval of the YouTube video through RapidAPI.
- **Configuration choices:**
  - URL: `https://youtube-video-fast-downloader-24-7.p.rapidapi.com/download_video/{{ VIDEO ID }}`
  - Query parameter: `quality=247`
  - Headers:
    - `x-rapidapi-host`
    - `x-rapidapi-key`
  - Redirect handling enabled
- **Key expressions or variables used:**
  - `{{ $json['VIDEO ID'] }}`
- **Input and output connections:**
  - Input: `Set params`
  - Output: `Wait`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - Invalid RapidAPI key.
  - Requested format/quality unavailable.
  - Downloader service outage.
  - Response may initially point to a not-yet-ready file.
- **Sub-workflow reference:** None.

#### Wait
- **Type and technical role:** `n8n-nodes-base.wait`; delays file retrieval.
- **Configuration choices:** Waits `300` seconds.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Download Video`
  - Output: `Get video`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - If the file still is not ready after 300 seconds, next node may get `404`.
  - Long waits affect execution duration.
- **Sub-workflow reference:** None.

#### Get video
- **Type and technical role:** `n8n-nodes-base.httpRequest`; downloads the actual video file from the file URL returned by the previous API.
- **Configuration choices:**
  - URL comes from `{{$json.file}}`
  - Response settings left mostly default
- **Key expressions or variables used:**
  - `={{ $json.file }}`
- **Input and output connections:**
  - Input: `Wait`
  - Outputs:
    - `Upload file`
    - `Upload to FTP`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - `404` if file still not ready.
  - URL expired.
  - Large file download timeout.
  - If response is not binary as expected, upload nodes may fail.
- **Sub-workflow reference:** None.

#### Upload file
- **Type and technical role:** `n8n-nodes-base.googleDrive`; stores the original downloaded video in Google Drive.
- **Configuration choices:**
  - File name: `VIDEO ID.mp4`
  - Drive: `My Drive`
  - Folder: specific folder ID `1tkCr7xdraoZwsHqeLm7FZ4aRWY94oLbZ`
- **Key expressions or variables used:**
  - `={{$('Set params').item.json['VIDEO ID']}}.mp4`
- **Input and output connections:**
  - Input: `Get video`
  - Output: none
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Google OAuth issues.
  - Missing folder permission.
  - Binary property mismatch if file was not downloaded correctly.
- **Sub-workflow reference:** None.

#### Upload to FTP
- **Type and technical role:** `n8n-nodes-base.ftp`; uploads the original source video to BunnyCDN-backed storage or another FTP endpoint.
- **Configuration choices:**
  - Operation: `upload`
  - Path: `/n3wstorage/test/{{ $json.name }}`
- **Key expressions or variables used:**
  - `=/n3wstorage/test/{{ $json.name }}`
- **Input and output connections:**
  - Input: `Get video`
  - Output: none
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - FTP auth failure.
  - Remote path permissions.
  - `$json.name` may not be defined as expected depending on binary metadata.
  - Uploading may fail if binary data property is missing.
- **Sub-workflow reference:** None.

---

## 2.5 Clip Creation via Fal.run

### Overview
This block constructs a public URL to the uploaded source video, requests trimming from Fal.run using AI-derived timestamps, then polls until the job is complete and retrieves the final clip URL.

### Nodes Involved
- `Set video url`
- `Video Dubbing`
- `Wait 30 sec.`
- `Get status`
- `Completed?`
- `Get final video url`

### Node Details

#### Set video url
- **Type and technical role:** `n8n-nodes-base.set`; creates the public source video URL used by Fal.run.
- **Configuration choices:**
  - Adds `video_url = https://n3wstorage.b-cdn.net/test/{{ VIDEO ID }}.mp4`
- **Key expressions or variables used:**
  - `=https://n3wstorage.b-cdn.net/test/{{ $('Set params').item.json['VIDEO ID'] }}.mp4`
- **Input and output connections:**
  - Input: `Output found?`
  - Output: `Video Dubbing`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Assumes source upload to BunnyCDN succeeded and naming matches exactly.
  - Public CDN propagation delay may make the URL temporarily unavailable.
- **Sub-workflow reference:** None.

#### Video Dubbing
- **Type and technical role:** `n8n-nodes-base.httpRequest`; despite the name, this node requests video trimming from Fal.run.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://queue.fal.run/fal-ai/workflow-utilities/trim-video`
  - Auth: generic header auth
  - Body parameters:
    - `video_url`
    - `start_time`
    - `end_time`
    - `duration`
- **Key expressions or variables used:**
  - `={{ $json.video_url }}`
  - `={{ $('Basic LLM Chain').item.json.output.start_time }}`
  - `={{ $('Basic LLM Chain').item.json.output.end_time }}`
  - `={{ $('Basic LLM Chain').item.json.output.duration }}`
- **Input and output connections:**
  - Input: `Set video url`
  - Output: `Wait 30 sec.`
- **Version-specific requirements:** Type version `4.3`.
- **Edge cases or potential failure types:**
  - Fal auth error.
  - Invalid or inaccessible `video_url`.
  - Start/end times outside source duration.
  - Trim job rejected due to malformed parameters.
- **Sub-workflow reference:** None.

#### Wait 30 sec.
- **Type and technical role:** `n8n-nodes-base.wait`; delays between polling attempts.
- **Configuration choices:** Waits `30` seconds.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Inputs:
    - `Video Dubbing`
    - false branch of `Completed?`
  - Output: `Get status`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Repeated polling increases execution time.
  - No explicit maximum retry count.
- **Sub-workflow reference:** None.

#### Get status
- **Type and technical role:** `n8n-nodes-base.httpRequest`; checks Fal job status.
- **Configuration choices:**
  - URL: `https://queue.fal.run/fal-ai/workflow-utilities/requests/{{ request_id }}/status`
  - Auth: generic header auth
  - Query param incorrectly used for `Content-Type`, though typically harmless for many APIs
- **Key expressions or variables used:**
  - `={{ $('Video Dubbing').item.json.request_id }}`
- **Input and output connections:**
  - Input: `Wait 30 sec.`
  - Output: `Completed?`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Status endpoint unavailable.
  - `request_id` missing.
  - Job may fail or return a status other than `COMPLETED`; no dedicated failed-state branch exists.
- **Sub-workflow reference:** None.

#### Completed?
- **Type and technical role:** `n8n-nodes-base.if`; routes based on Fal job status.
- **Configuration choices:**
  - Checks `{{$json.status}} == "COMPLETED"`
- **Key expressions or variables used:**
  - `={{ $json.status }}`
- **Input and output connections:**
  - Input: `Get status`
  - True output: `Get final video url`
  - False output: `Wait 30 sec.`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Infinite polling loop if status never becomes `COMPLETED`.
  - Failed jobs are retried forever unless manually stopped.
- **Sub-workflow reference:** None.

#### Get final video url
- **Type and technical role:** `n8n-nodes-base.httpRequest`; fetches the final result payload from Fal.run after completion.
- **Configuration choices:**
  - URL: `https://queue.fal.run/fal-ai/workflow-utilities/requests/{{ $json.request_id }}`
  - Auth: generic header auth
  - Sends `Content-Type: application/json`
- **Key expressions or variables used:**
  - `={{ $json.request_id }}`
- **Input and output connections:**
  - Input: `Completed?`
  - Output: `Get video file`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Request ID may not be present in status payload in the expected way.
  - API result shape may change.
- **Sub-workflow reference:** None.

---

## 2.6 Final Clip Retrieval and Distribution

### Overview
This block downloads the generated short clip and pushes it to storage and social publishing services. It is the final asset delivery layer.

### Nodes Involved
- `Get video file`
- `Upload do Drive`
- `Upload to FTP server`
- `Upload to Youtube`
- `Upload to TikTok`
- `Upload to Postiz`
- `Upload to Instagram`

### Node Details

#### Get video file
- **Type and technical role:** `n8n-nodes-base.httpRequest`; downloads the trimmed video from the final URL returned by Fal.run.
- **Configuration choices:**
  - URL: `{{$json.video.url}}`
- **Key expressions or variables used:**
  - `={{ $json.video.url }}`
- **Input and output connections:**
  - Input: `Get final video url`
  - Outputs:
    - `Upload do Drive`
    - `Upload to FTP server`
    - `Upload to Youtube`
    - `Upload to TikTok`
    - `Upload to Postiz`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - URL may be temporary or expired.
  - Binary handling issues if response is not downloaded as expected.
  - Large file timeout.
- **Sub-workflow reference:** None.

#### Upload do Drive
- **Type and technical role:** `n8n-nodes-base.googleDrive`; stores the final trimmed clip in Google Drive.
- **Configuration choices:**
  - File name: `VIDEO ID_clip.mp4`
  - Same Google Drive folder as source upload
- **Key expressions or variables used:**
  - `={{$('Set params').item.json['VIDEO ID']}}_clip.mp4`
- **Input and output connections:**
  - Input: `Get video file`
  - Output: none
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - Google Drive auth or permission errors.
  - Binary property mismatch.
- **Sub-workflow reference:** None.

#### Upload to FTP server
- **Type and technical role:** `n8n-nodes-base.ftp`; uploads the final clip to the FTP endpoint.
- **Configuration choices:**
  - Operation: `upload`
  - Path: `/n3wstorage/test/{{ VIDEO ID }}_clip.mp4`
- **Key expressions or variables used:**
  - `=/n3wstorage/test/{{ $('Set params').item.json['VIDEO ID'] }}_clip.mp4`
- **Input and output connections:**
  - Input: `Get video file`
  - Output: none
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - FTP auth issues.
  - Remote write permissions.
  - Binary property not present.
- **Sub-workflow reference:** None.

#### Upload to Youtube
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends the clip to Upload-Post for YouTube publishing.
- **Configuration choices:**
  - URL: `https://api.upload-post.com/api/upload`
  - Method: `POST`
  - Multipart form-data
  - Header-auth credential
  - Form fields:
    - `title = SET_TITLE`
    - `user = YOUR_USERNAME`
    - `platform[] = youtube`
    - `video` from binary field `data`
- **Key expressions or variables used:**
  - `=SET_TITLE`
- **Input and output connections:**
  - Input: `Get video file`
  - Output: none
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Placeholder values must be replaced.
  - API auth errors.
  - User account/platform mapping incorrect.
  - File size or format restrictions.
- **Sub-workflow reference:** None.

#### Upload to TikTok
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sends the clip to Upload-Post for TikTok publishing.
- **Configuration choices:**
  - URL: `https://api.upload-post.com/api/upload`
  - Method: `POST`
  - Multipart form-data
  - Form fields:
    - `title = SET_TITLE`
    - `user = YOUR_USERNAME`
    - `platform[] = tiktok`
    - `video` from binary field `data`
- **Key expressions or variables used:**
  - `=SET_TITLE`
- **Input and output connections:**
  - Input: `Get video file`
  - Output: none
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Placeholder values must be replaced.
  - Current credential name appears unrelated (`Youtube Transcript Extractor API 1`), so credential misuse is possible.
  - TikTok platform-side restrictions may block uploads.
- **Sub-workflow reference:** None.

#### Upload to Postiz
- **Type and technical role:** `n8n-nodes-base.httpRequest`; uploads the binary media to Postiz to obtain a media asset reference.
- **Configuration choices:**
  - URL: `https://api.postiz.com/public/v1/upload`
  - Method: `POST`
  - Multipart form-data
  - Binary field: `file` from input binary property `data`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Get video file`
  - Output: `Upload to Instagram`
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Auth issues.
  - Binary property mismatch.
  - Postiz may return an unexpected media object shape.
- **Sub-workflow reference:** None.

#### Upload to Instagram
- **Type and technical role:** `n8n-nodes-postiz.postiz`; schedules/posts content to Instagram via Postiz.
- **Configuration choices:**
  - Date/time built from current timestamp:
    - `{{$now.format('yyyy-LL-dd')}}T{{$now.format('HH:ii:ss')}}`
  - Post payload includes:
    - `contentItem`
    - uploaded media reference (`id`, `path`)
    - caption/content placeholder `XXX`
    - `integrationId = XXX`
  - `shortLink` enabled
- **Key expressions or variables used:**
  - `={{ $now.format('yyyy-LL-dd') }}T{{ $now.format('HH:ii:ss') }}`
  - `={{ $json.id }}`
  - `={{ $json.path }}`
- **Input and output connections:**
  - Input: `Upload to Postiz`
  - Output: none
- **Version-specific requirements:** Type version `1`; requires the Postiz community/integration node to be installed and configured.
- **Edge cases or potential failure types:**
  - Placeholder `integrationId` and content must be replaced.
  - Scheduled time formatting may need timezone validation.
  - Instagram account must already be linked in Postiz.
- **Sub-workflow reference:** None.

---

## 2.7 Documentation / Notes Layer

### Overview
These nodes do not affect runtime logic but document setup requirements, credential sources, workflow purpose, and publishing resources.

### Nodes Involved
- `Sticky Note`
- `Sticky Note1`
- `Sticky Note2`
- `Sticky Note3`
- `Sticky Note4`
- `Sticky Note5`
- `Sticky Note6`
- `Sticky Note8`
- `Sticky Note9`

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation.
- **Configuration choices:** Notes that the downloaded video file may take 20 to 300 seconds to become available, otherwise a `404` occurs, and it is only available for 10 minutes.
- **Input and output connections:** none.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** Communicates delayed availability of the downloader output.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** sticky note.
- **Configuration choices:** Documents social upload APIs and links:
  - Upload-Post for TikTok and YouTube
  - Postiz for Instagram
- **Input and output connections:** none.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** Highlights optional external dependencies.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** sticky note.
- **Configuration choices:** Documents RapidAPI key requirement and link for YouTube downloader.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and technical role:** sticky note.
- **Configuration choices:** Documents YouTube Transcript API requirement and source link.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and technical role:** sticky note.
- **Configuration choices:** Explains that `VIDEO ID` and `PROMPT` must be set.
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and technical role:** sticky note.
- **Configuration choices:** Explains transcript search purpose of the LLM block.
- **Sub-workflow reference:** None.

#### Sticky Note6
- **Type and technical role:** sticky note.
- **Configuration choices:** Describes the clip creation section.
- **Sub-workflow reference:** None.

#### Sticky Note8
- **Type and technical role:** sticky note.
- **Configuration choices:** Provides full workflow description and setup steps.
- **Sub-workflow reference:** None.

#### Sticky Note9
- **Type and technical role:** sticky note.
- **Configuration choices:** Promotional note with YouTube link and image.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual workflow entry point |  | Set params |  |
| Set params | Set | Defines `VIDEO ID` and `PROMPT` inputs | When clicking ‘Execute workflow’ | Generate transcript; Download Video | ## PARAMS<br>Set Youtube ID video and prompt (the content you want to search for within the video) |
| Generate transcript | HTTP Request | Fetches transcript metadata and transcript payload | Set params | Get transcript | ## Y-TRANSCRIPT KEY<br>Get [Youtube Transcript API](youtube-transcript.io) |
| Get transcript | Set | Extracts transcript array from API response | Generate transcript | Basic LLM Chain | ## TIMESTAMP EXTRACTOR<br>Search within the video transcript for the content searched for in the set prompt |
| Basic LLM Chain | LangChain LLM Chain | Finds clip timestamps from transcript and prompt | Get transcript | Output found? | ## TIMESTAMP EXTRACTOR<br>Search within the video transcript for the content searched for in the set prompt |
| OpenAI Chat Model | OpenAI Chat Model | Supplies the LLM used by the chain |  | Basic LLM Chain | ## TIMESTAMP EXTRACTOR<br>Search within the video transcript for the content searched for in the set prompt |
| Structured Output Parser | Structured Output Parser | Enforces JSON output schema from the LLM |  | Basic LLM Chain | ## TIMESTAMP EXTRACTOR<br>Search within the video transcript for the content searched for in the set prompt |
| Output found? | If | Continues only when the transcript match was found | Basic LLM Chain | Set video url | ## TIMESTAMP EXTRACTOR<br>Search within the video transcript for the content searched for in the set prompt |
| Download Video | HTTP Request | Starts retrieval of the source YouTube video | Set params | Wait | ## RAPIDAPI KEY<br>Get and set RapidAPI Key for [Youtube video downloader](https://rapidapi.com/nikzeferis/api/youtube-video-fast-downloader-24-7/playground)<br>## IMPORTANT Note<br>The file will soon be ready (from 20 to 300 seconds). Until it is ready, attempting to access it will return a 404 error. The file will be available for download only 10 minute |
| Wait | Wait | Delays before fetching the generated downloadable file | Download Video | Get video | ## IMPORTANT Note<br>The file will soon be ready (from 20 to 300 seconds). Until it is ready, attempting to access it will return a 404 error. The file will be available for download only 10 minute |
| Get video | HTTP Request | Downloads the original source video file | Wait | Upload file; Upload to FTP | ## IMPORTANT Note<br>The file will soon be ready (from 20 to 300 seconds). Until it is ready, attempting to access it will return a 404 error. The file will be available for download only 10 minute |
| Upload file | Google Drive | Stores the original source video in Google Drive | Get video |  | ## IMPORTANT Note<br>The file will soon be ready (from 20 to 300 seconds). Until it is ready, attempting to access it will return a 404 error. The file will be available for download only 10 minute |
| Upload to FTP | FTP | Uploads the original source video to BunnyCDN/FTP storage | Get video |  | ## IMPORTANT Note<br>The file will soon be ready (from 20 to 300 seconds). Until it is ready, attempting to access it will return a 404 error. The file will be available for download only 10 minute |
| Set video url | Set | Builds the public source video URL for Fal.run | Output found? | Video Dubbing | ## CREATE SHORT VIDEO<br>Create a short video clips from a YouTube video based on specific content requested by the user |
| Video Dubbing | HTTP Request | Sends trimming request to Fal.run | Set video url | Wait 30 sec. | ## CREATE SHORT VIDEO<br>Create a short video clips from a YouTube video based on specific content requested by the user |
| Wait 30 sec. | Wait | Polling delay for Fal.run job completion | Video Dubbing; Completed? | Get status | ## CREATE SHORT VIDEO<br>Create a short video clips from a YouTube video based on specific content requested by the user |
| Get status | HTTP Request | Checks Fal.run job status | Wait 30 sec. | Completed? | ## CREATE SHORT VIDEO<br>Create a short video clips from a YouTube video based on specific content requested by the user |
| Completed? | If | Routes based on whether Fal.run finished the job | Get status | Get final video url; Wait 30 sec. | ## CREATE SHORT VIDEO<br>Create a short video clips from a YouTube video based on specific content requested by the user |
| Get final video url | HTTP Request | Retrieves final result payload from Fal.run | Completed? | Get video file | ## CREATE SHORT VIDEO<br>Create a short video clips from a YouTube video based on specific content requested by the user |
| Get video file | HTTP Request | Downloads the final trimmed clip | Get final video url | Upload do Drive; Upload to FTP server; Upload to Youtube; Upload to TikTok; Upload to Postiz | ## CREATE SHORT VIDEO<br>Create a short video clips from a YouTube video based on specific content requested by the user |
| Upload do Drive | Google Drive | Stores the final clip in Google Drive | Get video file |  | ## CREATE SHORT VIDEO<br>Create a short video clips from a YouTube video based on specific content requested by the user |
| Upload to FTP server | FTP | Uploads the final clip to BunnyCDN/FTP storage | Get video file |  | ## CREATE SHORT VIDEO<br>Create a short video clips from a YouTube video based on specific content requested by the user |
| Upload to Youtube | HTTP Request | Publishes the final clip to YouTube through Upload-Post | Get video file |  | ## UPLOAD TO SOCIAL<br>Get [Upload-Post API](https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app) to share video on TikTok and Youtube<br>Get [Postiz API](https://postiz.pro/n3witalia) to share video on Instagram |
| Upload to TikTok | HTTP Request | Publishes the final clip to TikTok through Upload-Post | Get video file |  | ## UPLOAD TO SOCIAL<br>Get [Upload-Post API](https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app) to share video on TikTok and Youtube<br>Get [Postiz API](https://postiz.pro/n3witalia) to share video on Instagram |
| Upload to Postiz | HTTP Request | Uploads clip media to Postiz before Instagram posting | Get video file | Upload to Instagram | ## UPLOAD TO SOCIAL<br>Get [Upload-Post API](https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app) to share video on TikTok and Youtube<br>Get [Postiz API](https://postiz.pro/n3witalia) to share video on Instagram |
| Upload to Instagram | Postiz | Posts/schedules Instagram content using uploaded media | Upload to Postiz |  | ## UPLOAD TO SOCIAL<br>Get [Upload-Post API](https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app) to share video on TikTok and Youtube<br>Get [Postiz API](https://postiz.pro/n3witalia) to share video on Instagram |
| Sticky Note | Sticky Note | Documents downloader delay and temporary file availability |  |  | ## IMPORTANT Note<br>The file will soon be ready (from 20 to 300 seconds). Until it is ready, attempting to access it will return a 404 error. The file will be available for download only 10 minute |
| Sticky Note1 | Sticky Note | Documents social upload providers |  |  | ## UPLOAD TO SOCIAL<br>Get [Upload-Post API](https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app) to share video on TikTok and Youtube<br>Get [Postiz API](https://postiz.pro/n3witalia) to share video on Instagram |
| Sticky Note2 | Sticky Note | Documents RapidAPI downloader setup |  |  | ## RAPIDAPI KEY<br>Get and set RapidAPI Key for [Youtube video downloader](https://rapidapi.com/nikzeferis/api/youtube-video-fast-downloader-24-7/playground) |
| Sticky Note3 | Sticky Note | Documents transcript API setup |  |  | ## Y-TRANSCRIPT KEY<br>Get [Youtube Transcript API](youtube-transcript.io) |
| Sticky Note4 | Sticky Note | Documents parameter setup |  |  | ## PARAMS<br>Set Youtube ID video and prompt (the content you want to search for within the video) |
| Sticky Note5 | Sticky Note | Documents timestamp extraction block purpose |  |  | ## TIMESTAMP EXTRACTOR<br>Search within the video transcript for the content searched for in the set prompt |
| Sticky Note6 | Sticky Note | Documents short creation block purpose |  |  | ## CREATE SHORT VIDEO<br>Create a short video clips from a YouTube video based on specific content requested by the user |
| Sticky Note8 | Sticky Note | High-level workflow description and setup instructions |  |  | ## AI YouTube Shorts Creator: Prompt-Based Clipping, Dubbing & Social Upload<br><br>This workflow automates the process of creating short video clips from a YouTube video based on specific content requested by the user.<br><br>Tis is a complete AI-powered video clipping and distribution system, turning any YouTube video into ready-to-publish short-form content automatically<br><br>### How it works<br><br>This workflow turns a YouTube video into a short clip based on a user’s prompt. It fetches the transcript, uses an OpenAI model to identify the most relevant segment with precise start and end timestamps, downloads the source video, uploads it to BunnyCDN for a public URL, then sends the URL and timecodes to Fal.run to trim the clip. Once generated, the final short is downloaded and distributed automatically to Google Drive, BunnyCDN, TikTok, YouTube, and Instagram through the configured APIs.<br><br>### Setup steps<br><br>Configure all required credentials before running the workflow: RapidAPI for video download, youtube-transcript for transcript retrieval, OpenAI for clip detection, and Fal Run for trimming. Then set up Google Drive OAuth2 and BunnyCDN FTP for storage, optionally connect Postiz for Instagram and Upload-post for TikTok and YouTube publishing, and update the workflow test variables such as `VIDEO ID`, `PROMPT`, destination folder IDs, FTP paths, titles, usernames, and platform-specific integration fields. |
| Sticky Note9 | Sticky Note | Promotional YouTube channel note |  |  | ## MY NEW YOUTUBE CHANNEL<br>👉 [Subscribe to my new **YouTube channel**](https://youtube.com/@n3witalia). Here I’ll share videos and Shorts with practical tutorials and **FREE templates for n8n**.<br><br>[![image](https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg)](https://youtube.com/@n3witalia) |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Create video clip from Youtube video`.
   - Keep execution mode standard.
   - Ensure binary mode supports file handling.

2. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Name: `When clicking ‘Execute workflow’`

3. **Add a Set node for input parameters**
   - Node type: `Set`
   - Name: `Set params`
   - Add two string fields:
     1. `VIDEO ID`
     2. `PROMPT`
   - Example values:
     - `VIDEO ID = eLz22nWmiuI`
     - `PROMPT = How to obtain Gemini API`
   - Connect:
     - `When clicking ‘Execute workflow’` → `Set params`

4. **Add transcript retrieval HTTP node**
   - Node type: `HTTP Request`
   - Name: `Generate transcript`
   - Method: `POST`
   - URL: `https://www.youtube-transcript.io/api/transcripts`
   - Authentication: `Generic Credential Type` → `Header Auth`
   - Add header:
     - `Content-Type: application/json`
   - Enable JSON body
   - Body:
     - `ids` as array containing `{{$json['VIDEO ID']}}`
   - Create credential for transcript API header auth.
   - Connect:
     - `Set params` → `Generate transcript`

5. **Add transcript extraction Set node**
   - Node type: `Set`
   - Name: `Get transcript`
   - Add field:
     - `transcript` as Array
   - Value:
     - `{{$json.tracks[0].transcript}}`
   - Connect:
     - `Generate transcript` → `Get transcript`

6. **Add OpenAI Chat Model node**
   - Node type: `OpenAI Chat Model`
   - Name: `OpenAI Chat Model`
   - Choose model: `gpt-4.1-mini`
   - Configure OpenAI credentials.

7. **Add Structured Output Parser node**
   - Node type: `Structured Output Parser`
   - Name: `Structured Output Parser`
   - Use a JSON schema example like:
     ```json
     {
       "found": true,
       "start_time": 130,
       "end_time": 135,
       "duration": 5
     }
     ```

8. **Add Basic LLM Chain node**
   - Node type: `Basic LLM Chain`
   - Name: `Basic LLM Chain`
   - Prompt type: define manually
   - User/input text:
     - include requested content from `$('Set params').item.json.PROMPT`
     - include transcript with `JSON.stringify($json.transcript)`
   - Add the instruction prompt telling the model to:
     - inspect transcript segments
     - return `found`, `start_time`, `end_time`, `duration`
     - return only valid JSON
     - return null values with `found: false` if not found
   - Enable structured output parser.
   - Connect:
     - `Get transcript` → `Basic LLM Chain`
     - `OpenAI Chat Model` → `Basic LLM Chain` via AI language model connection
     - `Structured Output Parser` → `Basic LLM Chain` via AI output parser connection

9. **Add an If node to validate the LLM result**
   - Node type: `If`
   - Name: `Output found?`
   - Condition:
     - check boolean `{{$json.output.found}}` is true
   - Connect:
     - `Basic LLM Chain` → `Output found?`

10. **Add source video downloader HTTP node**
    - Node type: `HTTP Request`
    - Name: `Download Video`
    - Method: `GET`
    - URL:
      - `https://youtube-video-fast-downloader-24-7.p.rapidapi.com/download_video/{{$json['VIDEO ID']}}`
    - Query parameter:
      - `quality = 247`
    - Headers:
      - `x-rapidapi-host = youtube-video-fast-downloader-24-7.p.rapidapi.com`
      - `x-rapidapi-key = YOUR_KEY`
    - Enable redirect handling.
    - Connect:
      - `Set params` → `Download Video`

11. **Add a Wait node for downloader readiness**
    - Node type: `Wait`
    - Name: `Wait`
    - Wait amount: `300` seconds
    - Connect:
      - `Download Video` → `Wait`

12. **Add HTTP node to fetch the actual source file**
    - Node type: `HTTP Request`
    - Name: `Get video`
    - URL:
      - `{{$json.file}}`
    - Make sure the node downloads the file as binary data.
    - Connect:
      - `Wait` → `Get video`

13. **Add Google Drive upload for the source video**
    - Node type: `Google Drive`
    - Name: `Upload file`
    - Operation: upload file
    - File name:
      - `{{$('Set params').item.json['VIDEO ID']}}.mp4`
    - Select:
      - Drive: `My Drive`
      - Folder: your target folder
    - Configure Google Drive OAuth2 credentials.
    - Connect:
      - `Get video` → `Upload file`

14. **Add FTP upload for the source video**
    - Node type: `FTP`
    - Name: `Upload to FTP`
    - Operation: `upload`
    - Remote path:
      - `/n3wstorage/test/{{ $json.name }}`
    - Configure FTP credentials for BunnyCDN or your FTP storage.
    - Connect:
      - `Get video` → `Upload to FTP`

15. **Add a Set node to build the public source URL**
    - Node type: `Set`
    - Name: `Set video url`
    - Add string field:
      - `video_url`
    - Value:
      - `https://n3wstorage.b-cdn.net/test/{{ $('Set params').item.json['VIDEO ID'] }}.mp4`
    - Connect:
      - `Output found?` true branch → `Set video url`

16. **Add Fal.run trim request node**
    - Node type: `HTTP Request`
    - Name: `Video Dubbing`
    - Method: `POST`
    - URL:
      - `https://queue.fal.run/fal-ai/workflow-utilities/trim-video`
    - Authentication: generic header auth
    - Body parameters:
      - `video_url = {{$json.video_url}}`
      - `start_time = {{ $('Basic LLM Chain').item.json.output.start_time }}`
      - `end_time = {{ $('Basic LLM Chain').item.json.output.end_time }}`
      - `duration = {{ $('Basic LLM Chain').item.json.output.duration }}`
    - Configure Fal.run API credential.
    - Connect:
      - `Set video url` → `Video Dubbing`

17. **Add polling wait node**
    - Node type: `Wait`
    - Name: `Wait 30 sec.`
    - Wait amount: `30` seconds
    - Connect:
      - `Video Dubbing` → `Wait 30 sec.`

18. **Add Fal.run status check node**
    - Node type: `HTTP Request`
    - Name: `Get status`
    - URL:
      - `https://queue.fal.run/fal-ai/workflow-utilities/requests/{{ $('Video Dubbing').item.json.request_id }}/status`
    - Authentication: generic header auth
    - Use the same Fal.run credential.
    - Connect:
      - `Wait 30 sec.` → `Get status`

19. **Add completion check node**
    - Node type: `If`
    - Name: `Completed?`
    - Condition:
      - `{{$json.status}}` equals `COMPLETED`
    - Connect:
      - `Get status` → `Completed?`

20. **Create the polling loop**
    - Connect false output of `Completed?` back to `Wait 30 sec.`
    - This reproduces the original indefinite polling behavior.

21. **Add Fal.run final result retrieval node**
    - Node type: `HTTP Request`
    - Name: `Get final video url`
    - URL:
      - `https://queue.fal.run/fal-ai/workflow-utilities/requests/{{ $json.request_id }}`
    - Authentication: generic header auth
    - Add header:
      - `Content-Type: application/json`
    - Connect:
      - true output of `Completed?` → `Get final video url`

22. **Add HTTP node to download the final clip**
    - Node type: `HTTP Request`
    - Name: `Get video file`
    - URL:
      - `{{$json.video.url}}`
    - Configure it to download binary file data.
    - Connect:
      - `Get final video url` → `Get video file`

23. **Add Google Drive upload for final clip**
    - Node type: `Google Drive`
    - Name: `Upload do Drive`
    - File name:
      - `{{$('Set params').item.json['VIDEO ID']}}_clip.mp4`
    - Same Drive and folder as desired.
    - Connect:
      - `Get video file` → `Upload do Drive`

24. **Add FTP upload for final clip**
    - Node type: `FTP`
    - Name: `Upload to FTP server`
    - Operation: upload
    - Path:
      - `/n3wstorage/test/{{ $('Set params').item.json['VIDEO ID'] }}_clip.mp4`
    - Connect:
      - `Get video file` → `Upload to FTP server`

25. **Add Upload-Post node for YouTube**
    - Node type: `HTTP Request`
    - Name: `Upload to Youtube`
    - Method: `POST`
    - URL: `https://api.upload-post.com/api/upload`
    - Multipart form-data
    - Authentication: generic header auth
    - Fields:
      - `title = SET_TITLE`
      - `user = YOUR_USERNAME`
      - `platform[] = youtube`
      - `video` as binary field from `data`
    - Replace placeholders with real values.
    - Connect:
      - `Get video file` → `Upload to Youtube`

26. **Add Upload-Post node for TikTok**
    - Node type: `HTTP Request`
    - Name: `Upload to TikTok`
    - Method: `POST`
    - URL: `https://api.upload-post.com/api/upload`
    - Multipart form-data
    - Fields:
      - `title = SET_TITLE`
      - `user = YOUR_USERNAME`
      - `platform[] = tiktok`
      - `video` as binary field from `data`
    - Connect:
      - `Get video file` → `Upload to TikTok`

27. **Add Postiz media upload node**
    - Node type: `HTTP Request`
    - Name: `Upload to Postiz`
    - Method: `POST`
    - URL: `https://api.postiz.com/public/v1/upload`
    - Multipart form-data
    - Authentication: generic header auth
    - Field:
      - `file` as binary field from `data`
    - Configure Postiz header auth credential.
    - Connect:
      - `Get video file` → `Upload to Postiz`

28. **Add Instagram posting node**
    - Node type: `Postiz`
    - Name: `Upload to Instagram`
    - Date:
      - `{{$now.format('yyyy-LL-dd')}}T{{$now.format('HH:ii:ss')}}`
    - Build one post entry containing:
      - uploaded media `id = {{$json.id}}`
      - uploaded media `path = {{$json.path}}`
      - text/caption placeholder like `XXX`
      - `integrationId = XXX`
    - Enable short link if desired.
    - Install and configure the Postiz node/credentials if not already available.
    - Connect:
      - `Upload to Postiz` → `Upload to Instagram`

29. **Add optional sticky notes for maintainability**
    - Create notes for:
      - RapidAPI downloader setup
      - Transcript API setup
      - parameters to change
      - social publishing setup
      - warning that downloader files may return `404` for several minutes
      - high-level system description

30. **Configure credentials**
    - Required:
      - YouTube transcript API header auth
      - RapidAPI header auth or direct headers
      - OpenAI API
      - Fal.run header auth
      - Google Drive OAuth2
      - FTP credentials
    - Optional:
      - Upload-Post header auth
      - Postiz header auth
      - Postiz native node credentials

31. **Replace all placeholders before production use**
    - Replace:
      - `XXX` values
      - `SET_TITLE`
      - `YOUR_USERNAME`
      - target Google Drive folder IDs
      - BunnyCDN paths/domain if different
      - Postiz `integrationId`
      - social captions

32. **Validate binary handling**
    - Ensure both `Get video` and `Get video file` return downloadable binary data.
    - Ensure downstream upload nodes expect the same binary property name, commonly `data`.

33. **Test in stages**
    - First test transcript branch only.
    - Then test source download/storage.
    - Then test Fal trimming.
    - Finally test each social upload independently.

34. **Recommended hardening improvements**
    - Add an error branch when transcript is missing.
    - Add a max retry counter for polling.
    - Add explicit handling for Fal statuses like `FAILED`.
    - Add checks that source upload completed before using the CDN URL.
    - Add a fallback if `output.found` is false.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI YouTube Shorts Creator: Prompt-Based Clipping, Dubbing & Social Upload | High-level workflow concept |
| This workflow automates the process of creating short video clips from a YouTube video based on specific content requested by the user. | Workflow description |
| This is a complete AI-powered video clipping and distribution system, turning any YouTube video into ready-to-publish short-form content automatically. | Workflow description |
| Configure all required credentials before running the workflow: RapidAPI for video download, youtube-transcript for transcript retrieval, OpenAI for clip detection, and Fal Run for trimming. Then set up Google Drive OAuth2 and BunnyCDN FTP for storage, optionally connect Postiz for Instagram and Upload-post for TikTok and YouTube publishing, and update the workflow test variables such as `VIDEO ID`, `PROMPT`, destination folder IDs, FTP paths, titles, usernames, and platform-specific integration fields. | Setup guidance |
| RapidAPI downloader provider | https://rapidapi.com/nikzeferis/api/youtube-video-fast-downloader-24-7/playground |
| YouTube transcript provider | youtube-transcript.io |
| Upload-Post API for TikTok and YouTube | https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app |
| Postiz API / platform for Instagram | https://postiz.pro/n3witalia |
| Subscribe to my new YouTube channel | https://youtube.com/@n3witalia |
| Promotional image used in sticky note | https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg |

## Additional implementation notes
- There is only **one actual workflow entry point**: `When clicking ‘Execute workflow’`.
- There are **no sub-workflow nodes** and **no external n8n workflow calls** in this workflow.
- The node named **`Video Dubbing` does not perform dubbing**; it calls a Fal.run video trimming endpoint.
- The workflow assumes that the original uploaded source video becomes publicly reachable at the BunnyCDN URL before Fal.run tries to access it.
- The polling loop has **no stop condition** other than `COMPLETED`, so failed or stuck Fal jobs can loop indefinitely.
- Several social publishing fields are placeholders and must be replaced before the workflow is operational in production.