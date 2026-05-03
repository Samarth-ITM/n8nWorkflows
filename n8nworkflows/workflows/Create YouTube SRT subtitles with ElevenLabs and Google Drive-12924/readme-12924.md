Create YouTube SRT subtitles with ElevenLabs and Google Drive

https://n8nworkflows.xyz/workflows/create-youtube-srt-subtitles-with-elevenlabs-and-google-drive-12924


# Create YouTube SRT subtitles with ElevenLabs and Google Drive

## 1. Workflow Overview

**Title:** Create YouTube SRT subtitles with ElevenLabs and Google Drive  
**Workflow name (internal):** YouTube Video to SRT Subtitles via ElevenLabs  
**Purpose:** Given a video URL (intended for YouTube), the workflow downloads the video content, sends it to ElevenLabs Speech-to-Text, converts the transcription into a properly time-coded **SRT (SubRip)** subtitle file, and uploads the `.srt` file to a chosen **Google Drive** folder.

### 1.1 Trigger & Input Reception
Manual execution, then a node sets the `video_url` to process.

### 1.2 Video Fetching
Downloads the video (as binary) from the provided URL via HTTP.

### 1.3 Speech-to-Text (ElevenLabs)
Sends the downloaded audio/video to ElevenLabs Speech-to-Text to obtain timestamped transcription data.

### 1.4 SRT Formatting & File Creation
Transforms ElevenLabs output into SRT text, then converts that text into a binary `.srt` file.

### 1.5 Delivery (Google Drive Upload)
Uploads the generated subtitle file to a specific Google Drive folder.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Video URL Setup

**Overview:** Starts the workflow manually and defines the input video URL in a structured JSON field used by downstream nodes.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Set Video Url

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — entry point.
- **Configuration (interpreted):** No parameters; user runs workflow from the editor/UI.
- **Inputs/Outputs:**  
  - **Input:** None  
  - **Output:** Triggers **Set Video Url**
- **Edge cases / failures:** None (except user not executing, or workflow disabled—this workflow is marked `active: false`).
- **Version:** typeVersion 1.

#### Node: Set Video Url
- **Type / role:** Set (`n8n-nodes-base.set`) — creates a field `video_url`.
- **Configuration (interpreted):**
  - Adds a string field:
    - `video_url` = `XXX` (placeholder)
  - No “keep only set” behavior specified; default behavior applies.
- **Key variables/expressions:**
  - `video_url` is used later as `{{ $json.video_url }}`
- **Inputs/Outputs:**
  - **Input:** Manual Trigger output
  - **Output:** Sends JSON to **Get Video**
- **Edge cases / failures:**
  - If `video_url` is not a valid downloadable URL (or YouTube blocks direct download), the next HTTP request will fail.
  - If left as `XXX`, HTTP request will error (“Invalid URL”).
- **Version:** typeVersion 3.4.

---

### Block 2 — Video Fetching (HTTP Download)

**Overview:** Retrieves the video content from the provided URL. This is expected to produce binary data used by the transcription node.

**Nodes involved:**
- Get Video

#### Node: Get Video
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — downloads content from `video_url`.
- **Configuration (interpreted):**
  - URL is set dynamically from the Set node: `{{ $json.video_url }}`
  - No explicit method shown → defaults to **GET**
  - No explicit “download” options shown in parameters; behavior depends on n8n defaults and response headers (binary vs JSON/text).
- **Key variables/expressions:**
  - `url: {{ $json.video_url }}`
- **Inputs/Outputs:**
  - **Input:** JSON containing `video_url`
  - **Output:** Connected to **Transcribe audio or video**
- **Edge cases / failures:**
  - **YouTube URLs commonly won’t return raw video bytes** via plain GET due to redirects, signature requirements, region/cookie checks, bot protections, etc.
  - Might return HTML instead of video binary (ElevenLabs STT will then fail).
  - Potential issues: 403/429, large file timeouts, redirect handling, response size limits.
- **Version:** typeVersion 4.3.
- **Practical note:** If the intent is “YouTube link in → video bytes out”, you typically need a dedicated downloader step/service or a node that resolves YouTube to a downloadable stream.

---

### Block 3 — Speech-to-Text with ElevenLabs

**Overview:** Sends the downloaded audio/video content to ElevenLabs Speech-to-Text and obtains timestamped transcript data.

**Nodes involved:**
- Transcribe audio or video

#### Node: Transcribe audio or video
- **Type / role:** ElevenLabs node (`@elevenlabs/n8n-nodes-elevenlabs.elevenLabs`) — Speech-to-Text operation.
- **Configuration (interpreted):**
  - **Resource:** `speech`
  - **Operation:** `speechToText`
  - **Additional options:**
    - `diarize: false` (no speaker separation)
  - `requestOptions` empty (defaults used)
- **Credentials:**
  - Uses **ElevenLabs API** credential (“ElevenLabs account”).
- **Inputs/Outputs:**
  - **Input:** Output from **Get Video** (expected to contain binary audio/video)
  - **Output:** Transcript data to **From Elevenlabs to Srt**
- **Edge cases / failures:**
  - Auth errors (invalid/expired API key).
  - If input is not valid audio/video binary (e.g., HTML from YouTube), transcription fails.
  - File size/duration limits on the ElevenLabs side.
  - Language/accuracy issues; diarization disabled.
- **Version:** typeVersion 1.
- **Output expectation (required by downstream code):**
  - Downstream assumes an array (or single object) of “segments” where each segment includes:
    - `segment.text` (string)
    - `segment.words` (array), with entries containing at least:
      - `type` (expects `'word'`)
      - `start` (seconds)
      - `end` (seconds)

---

### Block 4 — Convert ElevenLabs Transcript to SRT + Build Binary File

**Overview:** Converts timestamped transcript segments into SRT blocks, then encodes that SRT text into a binary `.srt` file for upload.

**Nodes involved:**
- From Elevenlabs to Srt
- From Json to Binary

#### Node: From Elevenlabs to Srt
- **Type / role:** Code (`n8n-nodes-base.code`) — transforms ElevenLabs transcript JSON into SRT text.
- **Configuration (interpreted):**
  - JavaScript generates:
    - `secondsToSrtTime(seconds)` → `HH:MM:SS,mmm`
    - `splitTextSmart(text, minLen=20, maxLen=30)`:
      - Splits on punctuation `. ! ?` into chunks
      - Further wraps chunks longer than `maxLen` by words
      - Note: `minLen` exists but is not actually enforced in the current logic
  - Iterates through transcript segments and builds SRT entries with computed timings.
- **Key expressions/variables used:**
  - Reads input from `items[0].json`
  - If not an array, wraps into array: `if (!Array.isArray(input)) input = [input];`
  - For each segment:
    - `segment.words.filter(w => w.type === 'word')`
    - Uses first word’s `start` and last word’s `end` for segment timing
    - Splits `segment.text` into `textParts`
    - Assigns each part an equal fraction of the segment duration
  - Outputs JSON: `{ srt: '...' }`
- **Inputs/Outputs:**
  - **Input:** ElevenLabs STT output
  - **Output:** JSON containing `srt` to **From Json to Binary**
- **Edge cases / failures:**
  - If ElevenLabs output structure differs (no `words`, different `type` values, nested fields), SRT may be empty because the code `continue`s when fields are missing.
  - Timing accuracy: splitting a segment into equal durations may misalign subtitles if speech pacing varies.
  - If `textParts.length` becomes 0 (unlikely due to filters, but possible with unusual punctuation/empty text), division by zero would occur.
- **Version:** typeVersion 2.

#### Node: From Json to Binary
- **Type / role:** Code (`n8n-nodes-base.code`) — converts SRT text into binary file data.
- **Configuration (interpreted):**
  - Reads `items[0].json.srt`
  - Creates a UTF-8 buffer and returns base64 binary under `binary.data`
  - Sets:
    - `mimeType: 'application/x-subrip'`
    - `fileName: 'subtitles.srt'`
- **Inputs/Outputs:**
  - **Input:** JSON with `srt`
  - **Output:** Binary item to **Upload file**
- **Edge cases / failures:**
  - If `srt` is undefined/empty, the uploaded file may be empty (still valid but useless).
  - Large SRT could increase memory usage (generally manageable).
- **Version:** typeVersion 2.

---

### Block 5 — Upload SRT to Google Drive

**Overview:** Uploads the generated `.srt` binary file to a specific folder in Google Drive.

**Nodes involved:**
- Upload file

#### Node: Upload file
- **Type / role:** Google Drive (`n8n-nodes-base.googleDrive`) — file upload.
- **Configuration (interpreted):**
  - Upload name is dynamic: `{{ $binary.data.fileName }}`
  - Drive: “My Drive”
  - Folder: ID `1tkCr7xdraoZwsHqeLm7FZ4aRWY94oLbZ` (cached name “n8n”)
- **Credentials:**
  - Google Drive OAuth2 credential (“Google Drive account (n3w.it)”)
- **Inputs/Outputs:**
  - **Input:** Binary `data` produced by **From Json to Binary**
  - **Output:** No downstream nodes connected
- **Edge cases / failures:**
  - OAuth token expired/invalid, missing scopes (Drive file create).
  - Folder ID invalid or not accessible by the authenticated account.
  - If binary property name differs (must be `data`), upload will fail.
- **Version:** typeVersion 3.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Set Video Url |  |
| Set Video Url | Set | Defines `video_url` | When clicking ‘Execute workflow’ | Get Video | ## STEP 1 - Set Video URL<br>Update the “Set Video Url” node with a valid YouTube link or configure it to receive URLs dynamically. |
| Get Video | HTTP Request | Downloads video/audio from URL | Set Video Url | Transcribe audio or video | ## STEP 1 - Speech to Text<br>Add your [ElevenLabs API ](https://try.elevenlabs.io/ahkbf00hocnu) key in the transcription node and configure Google Drive OAuth2 credentials for file uploads |
| Transcribe audio or video | ElevenLabs (Speech-to-Text) | Transcribes media to timestamped text | Get Video | From Elevenlabs to Srt | ## STEP 1 - Speech to Text<br>Add your [ElevenLabs API ](https://try.elevenlabs.io/ahkbf00hocnu) key in the transcription node and configure Google Drive OAuth2 credentials for file uploads |
| From Elevenlabs to Srt | Code | Builds SRT text with timestamps | Transcribe audio or video | From Json to Binary | ## STEP 3 - Subtitle Formatting<br>These segments are formatted into standard SRT (SubRip) format with precise timing, converted to a binary file, |
| From Json to Binary | Code | Encodes SRT text as `.srt` binary | From Elevenlabs to Srt | Upload file | ## STEP 3 - Subtitle Formatting<br>These segments are formatted into standard SRT (SubRip) format with precise timing, converted to a binary file, |
| Upload file | Google Drive | Uploads `.srt` to Drive folder | From Json to Binary | — | ## STEP 4 - Google Drive<br>Upload file to Google Drive |

Sticky notes present but not directly tied to a single node (global/branding):
- **Automated YouTube Video to SRT Subtitles via ElevenLabs & Google Drive** (overall description + setup)
- **MY NEW YOUTUBE CHANNEL** with link and image: https://youtube.com/@n3witalia

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: Manual Trigger**
   - Node type: **Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`
3. **Add node: Set**
   - Node type: **Set**
   - Name: `Set Video Url`
   - Add field:
     - **String** `video_url` = (your video URL; replace placeholder `XXX`)
   - Connect: `When clicking ‘Execute workflow’` → `Set Video Url`
4. **Add node: HTTP Request**
   - Node type: **HTTP Request**
   - Name: `Get Video`
   - Method: **GET**
   - URL: `{{ $json.video_url }}`
   - Connect: `Set Video Url` → `Get Video`
   - Important: ensure the node is configured to output downloadable content as binary if required by your ElevenLabs node/version (depending on your n8n defaults and the ElevenLabs node expectations).
5. **Add node: ElevenLabs**
   - Node type: **ElevenLabs**
   - Name: `Transcribe audio or video`
   - Resource: **speech**
   - Operation: **speechToText**
   - Additional option: **Diarize = false**
   - Credentials:
     - Create/configure **ElevenLabs API** credentials (API key) and select it here.
   - Connect: `Get Video` → `Transcribe audio or video`
6. **Add node: Code** (SRT creation)
   - Node type: **Code**
   - Name: `From Elevenlabs to Srt`
   - Paste logic that:
     - Reads ElevenLabs segments/words
     - Splits text into subtitle-sized chunks (max ~30 chars as implemented)
     - Formats `HH:MM:SS,mmm` timestamps
     - Outputs `{ json: { srt } }`
   - Connect: `Transcribe audio or video` → `From Elevenlabs to Srt`
7. **Add node: Code** (JSON to binary)
   - Node type: **Code**
   - Name: `From Json to Binary`
   - Configure to:
     - Read `items[0].json.srt`
     - Create binary `data` with:
       - `mimeType = application/x-subrip`
       - `fileName = subtitles.srt`
   - Connect: `From Elevenlabs to Srt` → `From Json to Binary`
8. **Add node: Google Drive**
   - Node type: **Google Drive**
   - Name: `Upload file`
   - Operation: **Upload** (file upload)
   - File name: `{{ $binary.data.fileName }}`
   - Drive: **My Drive**
   - Folder: select the target folder (or paste folder ID)
   - Credentials:
     - Create/configure **Google Drive OAuth2** credentials and select it here.
   - Connect: `From Json to Binary` → `Upload file`
9. **Run the workflow**
   - Update `video_url` to a real downloadable media URL.
   - Execute manually; verify that:
     - ElevenLabs returns word-level timestamps
     - SRT content is non-empty
     - File appears in the selected Drive folder.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automated YouTube Video to SRT Subtitles via ElevenLabs & Google Drive. Generates time-coded SRT from a video URL, transcribes with ElevenLabs, formats SRT, uploads to Drive. | ElevenLabs link included in note: https://try.elevenlabs.io/ahkbf00hocnu |
| MY NEW YOUTUBE CHANNEL — Subscribe for videos/Shorts with practical content and free n8n templates. | https://youtube.com/@n3witalia (image link shown in the sticky note) |
| Practical integration concern: a plain HTTP GET to a YouTube page URL often won’t yield raw video bytes. You may need a dedicated downloader/connector or a preprocessing step that resolves YouTube → direct media stream URL. | Applies to the “Get Video” → “Transcribe audio or video” chain |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.