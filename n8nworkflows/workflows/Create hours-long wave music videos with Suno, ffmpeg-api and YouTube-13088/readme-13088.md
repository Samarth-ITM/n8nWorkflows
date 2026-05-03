Create hours-long wave music videos with Suno, ffmpeg-api and YouTube

https://n8nworkflows.xyz/workflows/create-hours-long-wave-music-videos-with-suno--ffmpeg-api-and-youtube-13088


# Create hours-long wave music videos with Suno, ffmpeg-api and YouTube

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Create hours-long wave music videos with Suno, ffmpeg-api and YouTube  
**Workflow name (n8n):** Wave Music Video Generation

This workflow generates long-form “wave/sleep/ambient” style music videos end-to-end: it collects a theme and a background video URL from a user form, generates multiple instrumental tracks via a Suno-compatible API (Kie.ai), uploads assets to **ffmpeg-api** storage, concatenates audio tracks and merges them with the background video, then uploads and publishes the final video to YouTube via **Blotato**. It also logs song metadata to Google Sheets and generates Vietnamese SEO metadata (title/description/hashtags) using a Gemini LLM agent.

### 1.1 Input Reception & Workspace Initialization
Receives user inputs (music theme, background video URL, number of tracks) and creates a working directory in ffmpeg-api.

### 1.2 Background Video Acquisition & Upload
Downloads the provided video URL as a binary file, allocates an ffmpeg-api storage path, uploads the file to that path, and retrieves the uploaded file reference.

### 1.3 AI Prompt Engineering & Music Generation (Suno/Kie.ai)
Uses a LangChain AI Agent (Gemini) to transform the user theme into structured prompts, splits them into separate requests, creates Suno generation tasks, and polls until tasks are complete. Logs resulting track metadata into Google Sheets.

### 1.4 Audio Download, Upload, Concatenation & Final Merge (ffmpeg-api)
Downloads each generated track audio, uploads to ffmpeg-api, aggregates all uploaded audio paths, builds an ffmpeg job (video loop + audio concat), and runs the ffmpeg-api processing to produce a final MP4.

### 1.5 SEO Metadata + Upload & Publish to YouTube (Blotato)
Uploads the final MP4 to Blotato media storage, generates Vietnamese SEO title/description/hashtags via Gemini, and publishes the video to YouTube through Blotato.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Workspace Initialization

**Overview:** Collects user inputs and creates a dedicated ffmpeg-api directory for all assets produced during the run.

**Nodes involved:**
- Sticky Note (Input Data & Directory Initialization)
- Suno Wave Music Input Form
- Create Working Directory (ffmpeg-api)

#### Node: Sticky Note
- **Type / role:** Sticky Note; documentation overlay.
- **Config:** Describes block purpose: input collection + init working directory.
- **Edge cases:** None (non-executable).

#### Node: Suno Wave Music Input Form
- **Type / role:** `formTrigger`; entry point that starts execution when a user submits the form.
- **Configuration choices:**
  - **Form title:** “Song Input”
  - **Description:** “Provide music details for the background video”
  - **Fields:**
    - “Music theme” (required, text)
    - “Video URL” (configured as `file` type in the JSON but used as a URL string later; see edge case)
    - “Number of tracks” (required, text/number)
- **Key variables/expressions used later:**
  - `$('Suno Wave Music Input Form').item.json['Music theme']`
  - `$('Suno Wave Music Input Form').item.json['Number of tracks']`
  - `$('Suno Wave Music Input Form').item.json['Video URL']`
- **Outputs:** Connects to “Create Working Directory (ffmpeg-api)”.
- **Version notes:** TypeVersion `2.3` (Form Trigger v2).
- **Edge cases / failures:**
  - **Field name mismatch later:** SEO agent references `['Music Theme']` (capital T) but the form defines “Music theme”. This will yield empty/undefined unless corrected.
  - **Video URL field type:** Form field is `file` type but used as URL string in an HTTP Request. If the form provides binary upload rather than a URL, the download step will fail.

#### Node: Create Working Directory (ffmpeg-api)
- **Type / role:** `httpRequest` POST to create an ffmpeg-api directory (workspace).
- **Configuration choices:**
  - **URL:** `https://api.ffmpeg-api.com/directory`
  - **Method:** POST
  - **Auth:** HTTP Header Auth via credential **“Vbee API”**
- **Inputs:** From Form Trigger.
- **Outputs:** To “Prepare Video Path (ffmpeg-api)”.
- **Edge cases / failures:**
  - Credential missing/invalid header token → 401/403.
  - API downtime/timeouts.
  - If response schema changes and `directory.id` is missing, downstream expressions fail (`$json.directory.id`).

---

### Block 2 — Background Video Acquisition & Upload

**Overview:** Reserves a storage location on ffmpeg-api, downloads the background video from the user-provided URL, uploads it to ffmpeg-api object storage, then retrieves metadata (notably `file_info.file_path`) for later ffmpeg processing.

**Nodes involved:**
- Sticky Note1 (Download Background Video)
- Prepare Video Path (ffmpeg-api)
- Download Background Video
- Upload Video to Workspace (ffmpeg-api)
- Retrieve Uploaded Video

#### Node: Sticky Note1
- **Type / role:** Sticky Note; documentation overlay.
- **Config:** Explains download, upload, and retrieval of processed video reference.

#### Node: Prepare Video Path (ffmpeg-api)
- **Type / role:** `httpRequest` POST to create a file placeholder and get a pre-signed upload URL.
- **Configuration choices:**
  - **URL:** `https://api.ffmpeg-api.com/file`
  - **Method:** POST
  - **Auth:** HTTP Header Auth (same credential type)
  - **Body parameters:**
    - `file_name = "Video_Youtube"`
    - `dir_id = {{ $json.directory.id }}`
- **Inputs:** From “Create Working Directory (ffmpeg-api)”.
- **Outputs:** To “Download Background Video”.
- **Edge cases / failures:**
  - Missing `directory.id` → expression error.
  - ffmpeg-api may enforce unique file names; repeated runs could overwrite or error depending on API behavior.

#### Node: Download Background Video
- **Type / role:** `httpRequest` to fetch remote video as binary.
- **Configuration choices:**
  - **URL expression:** `{{ $('Suno Wave Music Input Form').item.json['Video URL'] }}`
  - **Response:** `responseFormat: file`, output property name: `Video`
- **Inputs:** From “Prepare Video Path (ffmpeg-api)” (sequencing dependency), but the URL is sourced from the form.
- **Outputs:** To “Upload Video to Workspace (ffmpeg-api)”.
- **Edge cases / failures:**
  - Invalid/expired URL, redirects, geo blocks.
  - Very large files can exceed n8n memory/binary limits or request timeout.
  - If form field returns file metadata instead of URL string, this node fails.

#### Node: Upload Video to Workspace (ffmpeg-api)
- **Type / role:** `httpRequest` PUT to a pre-signed storage URL.
- **Configuration choices:**
  - **URL:** `{{ $('Prepare Video Path (ffmpeg-api)').item.json.upload.url }}`
  - **Method:** PUT
  - **Body:** binary data from field `Video`
  - **Content-Type:** `binaryData`
- **Inputs:** From “Download Background Video”.
- **Outputs:** To “Retrieve Uploaded Video”.
- **Edge cases / failures:**
  - Pre-signed URL expiration (note in sample data: `expiresInSeconds: 300`).
  - Binary field name mismatch (must be exactly `Video`).
  - Cloud storage may require correct PUT headers; ffmpeg-api usually provides them.

#### Node: Retrieve Uploaded Video
- **Type / role:** `httpRequest` GET to ffmpeg-api to obtain `file_info` for the uploaded video.
- **Configuration choices:**
  - **URL:** `https://api.ffmpeg-api.com/file/{{ file_path }}` using:
    - `{{ $('Prepare Video Path (ffmpeg-api)').item.json.file.file_path }}`
  - **Auth:** HTTP Header Auth
- **Inputs:** From “Upload Video to Workspace (ffmpeg-api)”.
- **Outputs:** To “Generate Music Prompts (AI Agent)”.
- **Edge cases / failures:**
  - If upload failed but workflow continues, retrieval may return 404.
  - If `file.file_path` missing → expression error.

---

### Block 3 — AI Prompt Engineering & Music Generation (Suno/Kie.ai)

**Overview:** Uses Gemini via LangChain agent to create 5 structured prompts, splits them into individual items, triggers Suno/Kie.ai music generation tasks, polls until each task is successful, then logs song metadata to Google Sheets.

**Nodes involved:**
- Sticky Note2 (AI Music Generation)
- Gemini LLM
- Music Prompt Output Parser
- Generate Music Prompts (AI Agent)
- Split Music Requests
- Create Song (Suno API)
- Wait for Song Rendering
- Get Generated Song
- Check Song Status
- Log Song Metadata

#### Node: Sticky Note2
- **Type / role:** Sticky Note; documentation overlay.

#### Node: Gemini LLM
- **Type / role:** Google Gemini chat model used by the agent.
- **Configuration choices:**
  - **Model:** `models/gemini-3-flash-preview`
- **Connections:**
  - Connected as **AI Language Model** input to “Generate Music Prompts (AI Agent)”.
- **Edge cases / failures:**
  - Missing Gemini credentials in n8n.
  - Model name availability changes (preview model may be deprecated).

#### Node: Music Prompt Output Parser
- **Type / role:** Structured output parser that forces the agent output into a JSON shape.
- **Configuration choices:**
  - Expects schema example:
    - `{ "songs": [ { "song": 1, "song_prompt": "string" } ] }`
- **Connections:**
  - Connected as **AI Output Parser** to “Generate Music Prompts (AI Agent)”.
- **Edge cases / failures:**
  - Agent system prompt instructs “Return ONLY a JSON array …” but parser expects an **object with `songs`**. Depending on the agent+parser behavior, this mismatch can cause parsing failures.
  - If parsing fails, `Split Music Requests` cannot find `output.songs`.

#### Node: Generate Music Prompts (AI Agent)
- **Type / role:** LangChain Agent that generates multiple Suno prompts based on user theme.
- **Configuration choices:**
  - **User text input:**  
    `make a song with {{ Number of tracks }} songs` plus “Reference Music Prompt”.
  - **System message:** Forces EXACTLY 5 unique song prompts in JSON.
  - **Has output parser:** yes (connected to Music Prompt Output Parser).
- **Inputs:** From “Retrieve Uploaded Video” (workflow sequencing).
- **Outputs:** To “Split Music Requests”.
- **Edge cases / failures:**
  - The prompt says “with N songs” but system says “exactly 5 songs”. Conflicting requirement; actual output likely 5.
  - If user enters non-numeric “Number of tracks”, the text still passes but logic doesn’t enforce it.

#### Node: Split Music Requests
- **Type / role:** `splitOut` converts array into individual items.
- **Configuration choices:**
  - **Field to split out:** `output.songs`
- **Inputs:** From “Generate Music Prompts (AI Agent)”.
- **Outputs:** To “Create Song (Suno API)”.
- **Edge cases / failures:**
  - If agent output is just an array (not `{output:{songs:...}}`), this field path is wrong → empty split or node error.

#### Node: Create Song (Suno API)
- **Type / role:** `httpRequest` POST to Kie.ai generate endpoint (Suno wrapper).
- **Configuration choices:**
  - **URL:** `https://api.kie.ai/api/v1/generate`
  - **Method:** POST
  - **Auth:** HTTP Header Auth (credential not shown by name here; uses generic)
  - **Body parameters:**
    - `prompt = {{ $json.song_prompt }}`
    - `customMode = false`
    - `instrumental = true`
    - `model = V5`
    - `callBackUrl = https://api.example.com/callback` (placeholder)
- **Inputs:** From split prompts.
- **Outputs:** To “Wait for Song Rendering”.
- **Edge cases / failures:**
  - Invalid API key / insufficient credits.
  - Callback URL is placeholder; this workflow actually polls, so callback is not required, but provider may validate it.
  - Rate limits when generating 5 tracks in parallel.

#### Node: Wait for Song Rendering
- **Type / role:** `wait` node used for polling/looping.
- **Configuration choices:**
  - Has a webhookId, but no explicit “wait until webhook called” configuration shown; in this topology it’s used as a delay/loop anchor.
- **Inputs:** From “Create Song (Suno API)” and from IF false branch (re-queue).
- **Outputs:** To “Get Generated Song”.
- **Edge cases / failures:**
  - Without a configured wait duration or webhook trigger specifics, behavior depends on n8n Wait node defaults/settings. It may pause indefinitely awaiting a resume event if configured as “wait for webhook”.
  - For polling, consider configuring a time-based wait (e.g., 30–60 seconds).

#### Node: Get Generated Song
- **Type / role:** `httpRequest` GET (query) to retrieve generation status/details.
- **Configuration choices:**
  - **URL:** `https://api.kie.ai/api/v1/generate/record-info ` (note trailing space)
  - **Query parameter:** `taskId = {{ $json.data.taskId }}`
  - **Auth:** HTTP Header Auth
- **Inputs:** From “Wait for Song Rendering”.
- **Outputs:** To “Check Song Status”.
- **Edge cases / failures:**
  - **Trailing space in URL** can break requests depending on n8n/version; should be removed.
  - If provider returns `PENDING/PROCESSING`, downstream IF handles it by looping back.

#### Node: Check Song Status
- **Type / role:** `if` node to branch on completion.
- **Condition:** `$json.data.status == "SUCCESS"`
- **True output:** to “Log Song Metadata”
- **False output:** to “Wait for Song Rendering” (poll again)
- **Edge cases / failures:**
  - If provider uses different status values or nesting changes, condition never matches and workflow loops indefinitely.
  - Consider adding max retries / timeout guard.

#### Node: Log Song Metadata
- **Type / role:** Google Sheets append row, used as a “fan-out to fan-in” anchor for track info.
- **Configuration choices:**
  - **Operation:** Append
  - **Document:** “Suno Log” (Spreadsheet ID `1TAcHusI...`)
  - **Sheet:** “Trang tính1”
  - **Columns mapped:**
    - `Duration = {{ $json.data.response.sunoData[0].duration }}`
    - `Song_URL = {{ $json.data.response.sunoData[0].audioUrl }}`
    - `Tilte_Song = {{ $json.data.response.sunoData[0].title }}{{ ...title }}` (title duplicated; likely a mistake)
    - Column “Theme” exists in schema but is **not** mapped.
- **Inputs:** From IF success branch.
- **Outputs:** To “Prepare Audio Path (ffmpeg-api)”.
- **Edge cases / failures:**
  - Google Sheets auth/permissions errors.
  - Duplicate title concatenation produces odd filenames later.
  - Missing `sunoData[0]` causes expression errors (if API returns empty).

---

### Block 4 — Audio Download, Upload, Concatenation & Final Merge (ffmpeg-api)

**Overview:** For each logged song, creates an ffmpeg-api file slot, downloads the MP3, uploads it, retrieves the stored `file_path`. Then aggregates all audio paths and constructs a single ffmpeg job that loops the background video and concatenates audio tracks into one final MP4.

**Nodes involved:**
- Sticky Note3 (Download Songs & Merge Audio + Video)
- Prepare Audio Path (ffmpeg-api)
- Download Song Audio
- Upload Song to Workspace (ffmpeg-api)
- Retrieve Uploaded Song
- Aggregate Audio Tracks
- Concatenate Audio Tracks
- Merge Audio with Video (ffmpeg-api)

#### Node: Sticky Note3
- **Type / role:** Sticky Note; documentation overlay.

#### Node: Prepare Audio Path (ffmpeg-api)
- **Type / role:** `httpRequest` POST to allocate storage and get pre-signed upload URL for each audio track.
- **Configuration choices:**
  - **URL:** `https://api.ffmpeg-api.com/file`
  - **Method:** POST
  - **Auth:** HTTP Header Auth
  - **Body parameters:**
    - `file_name = {{ $json.Tilte_Song }}_{{ $execution.id }}`
    - `dir_id = {{ $('Create Working Directory (ffmpeg-api)').item.json.directory.id }}`
- **Inputs:** From “Log Song Metadata”.
- **Outputs:** To “Download Song Audio”.
- **Edge cases / failures:**
  - If `Tilte_Song` includes forbidden filesystem chars, ffmpeg-api may reject.
  - Execution ID is numeric/string; OK for uniqueness.

#### Node: Download Song Audio
- **Type / role:** `httpRequest` to download MP3 as binary.
- **Configuration choices:**
  - **URL:** `{{ $('Log Song Metadata').item.json.Song_URL }}`
  - **Response:** file; **outputPropertyName is an expression**: `{{ $json.file.file_name }}`
    - This aims to name the binary property dynamically.
- **Inputs:** From “Prepare Audio Path (ffmpeg-api)”.
- **Outputs:** To “Upload Song to Workspace (ffmpeg-api)”.
- **Edge cases / failures:**
  - Output property name expression depends on `Prepare Audio Path` response being present in the current item as `$json.file.file_name`. If not, it resolves incorrectly.
  - Large MP3 sizes can hit memory limits.

#### Node: Upload Song to Workspace (ffmpeg-api)
- **Type / role:** `httpRequest` PUT binary MP3 to pre-signed URL.
- **Configuration choices:**
  - **URL:** `{{ $('Prepare Audio Path (ffmpeg-api)').item.json.upload.url }}`
  - **Method:** PUT
  - **Content-Type:** binaryData
  - **inputDataFieldName:** **hard-coded** `=Cellular DawnCellular Dawn_727`
- **Inputs:** From “Download Song Audio”.
- **Outputs:** To “Retrieve Uploaded Song”.
- **Major issue / edge case:**
  - The binary field name is hard-coded to a specific track name from pinned data. On real runs, audio binary property names will differ, causing “No binary data found” errors.  
  - Recommended: set `inputDataFieldName` to the actual binary property (`Video`-like fixed name) or to a stable property name (e.g., always output to `audio` in the previous node).

#### Node: Retrieve Uploaded Song
- **Type / role:** `httpRequest` GET ffmpeg-api file metadata for uploaded audio.
- **Configuration choices:**
  - **URL:** `https://api.ffmpeg-api.com/file/{{ $('Prepare Audio Path (ffmpeg-api)').item.json.file.file_path }}`
  - **Auth:** HTTP Header Auth
- **Inputs:** From “Upload Song to Workspace (ffmpeg-api)”.
- **Outputs:** To “Aggregate Audio Tracks”.
- **Edge cases / failures:**
  - 404 if upload didn’t succeed.
  - Expression error if file_path missing.

#### Node: Aggregate Audio Tracks
- **Type / role:** `aggregate` gathers multiple items into a single item with an array of audio paths.
- **Configuration choices:**
  - Aggregates field: `file_info.file_path`
- **Inputs:** From “Retrieve Uploaded Song” (multiple items).
- **Outputs:** To “Concatenate Audio Tracks”.
- **Edge cases / failures:**
  - Output field naming is critical: downstream code expects `$input.first().json.file_path` (but aggregation is configured on `file_info.file_path`). Depending on n8n Aggregate output structure, this may not match and break the code node.

#### Node: Concatenate Audio Tracks
- **Type / role:** `code` node that builds an ffmpeg-api job spec (JSON) to merge video + concatenated audio.
- **Key logic/config:**
  - Reads background video path from:
    - `$('Retrieve Uploaded Video').first().json.file_info.file_path`
  - Reads aggregated songs from:
    - `const songs = $input.first().json.file_path;` (likely should be something like `json['file_info.file_path']` or whatever Aggregate outputs)
  - Builds `inputs`:
    - Video input first, with `-stream_loop -1` to loop indefinitely.
    - Audio inputs next.
  - Builds `filter_complex`:
    - Concats `[1:a][2:a]...[n:a]concat=n=<songs.length>:v=0:a=1[aout]`
  - Outputs MP4:
    - Maps video from input 0 and audio from `[aout]`
    - Encodes H.264, `-preset veryfast`, `-shortest` to stop at end of audio.
  - Produces JSON object: `{ ffmpeg_job: { task: {...} } }`
- **Inputs:** From “Aggregate Audio Tracks”.
- **Outputs:** To “Merge Audio with Video (ffmpeg-api)”.
- **Edge cases / failures:**
  - If `songs` is not an array, filter generation fails.
  - If any audio lacks an audio stream, concat fails at ffmpeg level.
  - Duration mismatches are fine due to concat; video loops until audio ends due to `-shortest`.

#### Node: Merge Audio with Video (ffmpeg-api)
- **Type / role:** `httpRequest` POST to run ffmpeg job on ffmpeg-api.
- **Configuration choices:**
  - **URL:** `https://api.ffmpeg-api.com/ffmpeg/process`
  - **Method:** POST
  - **Auth:** HTTP Header Auth
  - **Body:** JSON stringified from `$json.ffmpeg_job`
- **Inputs:** From code node.
- **Outputs:** To “Upload Media On Blotato”.
- **Edge cases / failures:**
  - ffmpeg processing can be slow; ensure HTTP node timeout accommodates long jobs.
  - If ffmpeg-api rejects the job schema or input paths, returns error.
  - Output name fixed: `final_sleep_music_video.mp4` (may overwrite if same dir and API doesn’t namespace by job).

---

### Block 5 — SEO Metadata + Upload & Publish to YouTube

**Overview:** Takes the merged video download URL, uploads it to Blotato media storage, generates Vietnamese SEO metadata using Gemini, and publishes to YouTube with “contains synthetic media” enabled.

**Nodes involved:**
- Sticky Note4 (Auto Upload to YouTube)
- Upload Media On Blotato
- Gemini LLM1
- YouTube Metadata Parser
- Generate SEO Title & Description
- Publish YouTube Video

#### Node: Sticky Note4
- **Type / role:** Sticky Note; documentation overlay.

#### Node: Upload Media On Blotato
- **Type / role:** Blotato node (`resource: media`) to ingest a media URL and host it for publishing.
- **Configuration choices:**
  - **mediaUrl:** `{{ $json.result[0].download_url }}` (from ffmpeg-api result)
- **Inputs:** From “Merge Audio with Video (ffmpeg-api)”.
- **Outputs:** To “Generate SEO Title & Description”.
- **Edge cases / failures:**
  - If ffmpeg-api result array is empty, `result[0]` is undefined.
  - Blotato auth/account not configured.

#### Node: Gemini LLM1
- **Type / role:** Google Gemini model for SEO agent.
- **Configuration:** `models/gemini-3-flash-preview`
- **Connections:** AI Language Model input to “Generate SEO Title & Description”.

#### Node: YouTube Metadata Parser
- **Type / role:** Structured output parser expecting:
  - `{ "title": "string", "description": "string", "hashtags": "string" }`
- **Connections:** AI Output Parser input to “Generate SEO Title & Description”.
- **Edge cases / failures:**
  - The SEO agent system message demands “EXACTLY 3 sections” in plain text, not JSON. Parser expects JSON. This mismatch can cause parsing failure unless the agent implicitly formats to parser expectations.

#### Node: Generate SEO Title & Description
- **Type / role:** LangChain Agent for YouTube SEO metadata generation (Vietnamese).
- **Configuration choices:**
  - **Input text expression:**
    - `REFERENCE TITLE: {{ $('Suno Wave Music Input Form').item.json['Music Theme'] }}`
  - **System message:** Produces optimized title (<70 chars), multi-paragraph description, and semicolon-separated hashtags; Vietnamese; no emojis.
  - **Has output parser:** yes.
- **Inputs:** From “Upload Media On Blotato” (sequencing).
- **Outputs:** To “Publish YouTube Video”.
- **Edge cases / failures:**
  - **Field name mismatch:** uses `Music Theme` but form uses `Music theme`.
  - Output format mismatch vs parser (sections vs JSON).

#### Node: Publish YouTube Video
- **Type / role:** Blotato node to publish a YouTube post/video.
- **Configuration choices:**
  - **Platform:** youtube
  - **Account:** `xAI Giang ( GiangVT)` (accountId `22282`)
  - **Title:** `{{ $json.output.title }}`
  - **Text (description + hashtags):**
    - `{{ $json.output.description }}\n{{ $json.output.hashtags }}`
  - **Media URL:** `{{ $('Upload Media On Blotato').item.json.url }}`
  - **Contains synthetic media:** true
- **Inputs:** From “Generate SEO Title & Description”.
- **Outputs:** End of workflow.
- **Edge cases / failures:**
  - If SEO parser fails, `$json.output.*` may be undefined.
  - YouTube publishing restrictions, copyright/verification issues, quota.
  - Blotato account permissions or token expiry.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation | — | — | ## Input Data & Directory Initialization<br>Collect user input from the form and initialize a working directory using ffmpeg-api. This directory will be used to store videos, audio files, and intermediate assets throughout the workflow |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation | — | — | ## Download Background Video<br>Generate a storage path, download the background video from the provided URL, upload it to ffmpeg-api storage, and retrieve the processed video reference for later merging. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation | — | — | ## AI Music Generation (Suno)<br>Use an AI agent to convert the user’s music theme into structured Suno prompts, generate multiple songs, poll for completion, and log song metadata for downstream processing. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation | — | — | ## Download Songs & Merge Audio + Video<br>Download generated songs, upload them to ffmpeg-api storage, concatenate multiple tracks into a long-form audio file, and merge the final audio with the background video |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation | — | — | ## Auto Upload to YouTube |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation / Setup notes | — | — | # 🛠️ Workflow Setup Guide<br>Author: [GiangxAI](https://www.youtube.com/@giangxai.official)<br>Setup guide [n8n](https://n8n.partnerlinks.io/giangxai)<br>Kie ai: https://kie.ai?ref=f8cec88ea15f9ecbff52ccbafa41dd6e<br>FFmpeg-API: https://ffmpeg-api.com<br>Blotato: https://blotato.com/?ref=giang9s |
| Suno Wave Music Input Form | n8n-nodes-base.formTrigger | User input entry point | — | Create Working Directory (ffmpeg-api) | ## Input Data & Directory Initialization<br>Collect user input from the form and initialize a working directory using ffmpeg-api. This directory will be used to store videos, audio files, and intermediate assets throughout the workflow |
| Create Working Directory (ffmpeg-api) | n8n-nodes-base.httpRequest | Create ffmpeg-api directory | Suno Wave Music Input Form | Prepare Video Path (ffmpeg-api) | ## Input Data & Directory Initialization<br>Collect user input from the form and initialize a working directory using ffmpeg-api. This directory will be used to store videos, audio files, and intermediate assets throughout the workflow |
| Prepare Video Path (ffmpeg-api) | n8n-nodes-base.httpRequest | Reserve video file path + get pre-signed upload URL | Create Working Directory (ffmpeg-api) | Download Background Video | ## Download Background Video<br>Generate a storage path, download the background video from the provided URL, upload it to ffmpeg-api storage, and retrieve the processed video reference for later merging. |
| Download Background Video | n8n-nodes-base.httpRequest | Download remote video as binary | Prepare Video Path (ffmpeg-api) | Upload Video to Workspace (ffmpeg-api) | ## Download Background Video<br>Generate a storage path, download the background video from the provided URL, upload it to ffmpeg-api storage, and retrieve the processed video reference for later merging. |
| Upload Video to Workspace (ffmpeg-api) | n8n-nodes-base.httpRequest | PUT upload to pre-signed URL | Download Background Video | Retrieve Uploaded Video | ## Download Background Video<br>Generate a storage path, download the background video from the provided URL, upload it to ffmpeg-api storage, and retrieve the processed video reference for later merging. |
| Retrieve Uploaded Video | n8n-nodes-base.httpRequest | Retrieve uploaded video file_info | Upload Video to Workspace (ffmpeg-api) | Generate Music Prompts (AI Agent) | ## Download Background Video<br>Generate a storage path, download the background video from the provided URL, upload it to ffmpeg-api storage, and retrieve the processed video reference for later merging. |
| Gemini LLM | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM for music prompt agent | — (AI connection) | Generate Music Prompts (AI Agent) | ## AI Music Generation (Suno)<br>Use an AI agent to convert the user’s music theme into structured Suno prompts, generate multiple songs, poll for completion, and log song metadata for downstream processing. |
| Music Prompt Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parse for music prompts | — (AI connection) | Generate Music Prompts (AI Agent) | ## AI Music Generation (Suno)<br>Use an AI agent to convert the user’s music theme into structured Suno prompts, generate multiple songs, poll for completion, and log song metadata for downstream processing. |
| Generate Music Prompts (AI Agent) | @n8n/n8n-nodes-langchain.agent | Generate 5 Suno prompts from theme | Retrieve Uploaded Video | Split Music Requests | ## AI Music Generation (Suno)<br>Use an AI agent to convert the user’s music theme into structured Suno prompts, generate multiple songs, poll for completion, and log song metadata for downstream processing. |
| Split Music Requests | n8n-nodes-base.splitOut | Split prompts array into items | Generate Music Prompts (AI Agent) | Create Song (Suno API) | ## AI Music Generation (Suno)<br>Use an AI agent to convert the user’s music theme into structured Suno prompts, generate multiple songs, poll for completion, and log song metadata for downstream processing. |
| Create Song (Suno API) | n8n-nodes-base.httpRequest | Start Suno/Kie.ai generation task | Split Music Requests | Wait for Song Rendering | ## AI Music Generation (Suno)<br>Use an AI agent to convert the user’s music theme into structured Suno prompts, generate multiple songs, poll for completion, and log song metadata for downstream processing. |
| Wait for Song Rendering | n8n-nodes-base.wait | Polling delay/loop anchor | Create Song (Suno API); Check Song Status (false) | Get Generated Song | ## AI Music Generation (Suno)<br>Use an AI agent to convert the user’s music theme into structured Suno prompts, generate multiple songs, poll for completion, and log song metadata for downstream processing. |
| Get Generated Song | n8n-nodes-base.httpRequest | Fetch task status/details | Wait for Song Rendering | Check Song Status | ## AI Music Generation (Suno)<br>Use an AI agent to convert the user’s music theme into structured Suno prompts, generate multiple songs, poll for completion, and log song metadata for downstream processing. |
| Check Song Status | n8n-nodes-base.if | Branch when status == SUCCESS | Get Generated Song | Log Song Metadata (true); Wait for Song Rendering (false) | ## AI Music Generation (Suno)<br>Use an AI agent to convert the user’s music theme into structured Suno prompts, generate multiple songs, poll for completion, and log song metadata for downstream processing. |
| Log Song Metadata | n8n-nodes-base.googleSheets | Append track metadata to sheet | Check Song Status (true) | Prepare Audio Path (ffmpeg-api) | ## AI Music Generation (Suno)<br>Use an AI agent to convert the user’s music theme into structured Suno prompts, generate multiple songs, poll for completion, and log song metadata for downstream processing. |
| Prepare Audio Path (ffmpeg-api) | n8n-nodes-base.httpRequest | Reserve audio file path + upload URL | Log Song Metadata | Download Song Audio | ## Download Songs & Merge Audio + Video<br>Download generated songs, upload them to ffmpeg-api storage, concatenate multiple tracks into a long-form audio file, and merge the final audio with the background video |
| Download Song Audio | n8n-nodes-base.httpRequest | Download MP3 as binary | Prepare Audio Path (ffmpeg-api) | Upload Song to Workspace (ffmpeg-api) | ## Download Songs & Merge Audio + Video<br>Download generated songs, upload them to ffmpeg-api storage, concatenate multiple tracks into a long-form audio file, and merge the final audio with the background video |
| Upload Song to Workspace (ffmpeg-api) | n8n-nodes-base.httpRequest | PUT upload MP3 to pre-signed URL | Download Song Audio | Retrieve Uploaded Song | ## Download Songs & Merge Audio + Video<br>Download generated songs, upload them to ffmpeg-api storage, concatenate multiple tracks into a long-form audio file, and merge the final audio with the background video |
| Retrieve Uploaded Song | n8n-nodes-base.httpRequest | Retrieve uploaded audio file_info | Upload Song to Workspace (ffmpeg-api) | Aggregate Audio Tracks | ## Download Songs & Merge Audio + Video<br>Download generated songs, upload them to ffmpeg-api storage, concatenate multiple tracks into a long-form audio file, and merge the final audio with the background video |
| Aggregate Audio Tracks | n8n-nodes-base.aggregate | Collect all audio file paths | Retrieve Uploaded Song | Concatenate Audio Tracks | ## Download Songs & Merge Audio + Video<br>Download generated songs, upload them to ffmpeg-api storage, concatenate multiple tracks into a long-form audio file, and merge the final audio with the background video |
| Concatenate Audio Tracks | n8n-nodes-base.code | Build ffmpeg job JSON (loop video + concat audio) | Aggregate Audio Tracks | Merge Audio with Video (ffmpeg-api) | ## Download Songs & Merge Audio + Video<br>Download generated songs, upload them to ffmpeg-api storage, concatenate multiple tracks into a long-form audio file, and merge the final audio with the background video |
| Merge Audio with Video (ffmpeg-api) | n8n-nodes-base.httpRequest | Run ffmpeg processing job | Concatenate Audio Tracks | Upload Media On Blotato | ## Download Songs & Merge Audio + Video<br>Download generated songs, upload them to ffmpeg-api storage, concatenate multiple tracks into a long-form audio file, and merge the final audio with the background video |
| Upload Media On Blotato | @blotato/n8n-nodes-blotato.blotato | Upload final MP4 to Blotato media storage | Merge Audio with Video (ffmpeg-api) | Generate SEO Title & Description | ## Auto Upload to YouTube |
| Gemini LLM1 | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM for SEO agent | — (AI connection) | Generate SEO Title & Description | ## Auto Upload to YouTube |
| YouTube Metadata Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Structured parse for SEO metadata | — (AI connection) | Generate SEO Title & Description | ## Auto Upload to YouTube |
| Generate SEO Title & Description | @n8n/n8n-nodes-langchain.agent | Generate VN title/description/hashtags | Upload Media On Blotato | Publish YouTube Video | ## Auto Upload to YouTube |
| Publish YouTube Video | @blotato/n8n-nodes-blotato.blotato | Publish to YouTube | Generate SEO Title & Description | — | ## Auto Upload to YouTube |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n named **“Wave Music Video Generation”**.

2) **Add Form Trigger**
   - Node: **Form Trigger**
   - Title: “Song Input”
   - Fields:
     - Text: **Music theme** (required)
     - Text: **Video URL** (recommended to be text/URL, not file upload)
     - Number/Text: **Number of tracks** (required)

3) **Create ffmpeg-api directory**
   - Node: **HTTP Request**
   - Name: “Create Working Directory (ffmpeg-api)”
   - Method: **POST**
   - URL: `https://api.ffmpeg-api.com/directory`
   - Authentication: **Header Auth** credential (create one with your ffmpeg-api key/token header as required by ffmpeg-api)
   - Connect: Form Trigger → Create Working Directory

4) **Reserve background video file path**
   - Node: **HTTP Request**
   - Name: “Prepare Video Path (ffmpeg-api)”
   - Method: **POST**
   - URL: `https://api.ffmpeg-api.com/file`
   - Auth: same ffmpeg-api header credential
   - Send body: **ON**
   - Body params:
     - `file_name`: `Video_Youtube`
     - `dir_id`: expression referencing directory id from previous node (e.g., `{{$json.directory.id}}`)
   - Connect: Create Working Directory → Prepare Video Path

5) **Download background video**
   - Node: **HTTP Request**
   - Name: “Download Background Video”
   - URL: expression from the form: `{{ $('Suno Wave Music Input Form').item.json['Video URL'] }}`
   - Response: **File**
   - Output binary property: set to a fixed name, e.g. `Video`
   - Connect: Prepare Video Path → Download Background Video

6) **Upload background video to ffmpeg-api storage**
   - Node: **HTTP Request**
   - Name: “Upload Video to Workspace (ffmpeg-api)”
   - Method: **PUT**
   - URL: `{{ $('Prepare Video Path (ffmpeg-api)').item.json.upload.url }}`
   - Send body: **ON**
   - Body content type: **Binary Data**
   - Binary property: `Video`
   - Connect: Download Background Video → Upload Video to Workspace

7) **Retrieve uploaded video metadata**
   - Node: **HTTP Request**
   - Name: “Retrieve Uploaded Video”
   - Method: **GET**
   - URL: `https://api.ffmpeg-api.com/file/{{ $('Prepare Video Path (ffmpeg-api)').item.json.file.file_path }}`
   - Auth: ffmpeg-api header credential
   - Connect: Upload Video → Retrieve Uploaded Video

8) **Add Gemini LLM for music prompts**
   - Node: **Google Gemini Chat Model (LangChain)**
   - Name: “Gemini LLM”
   - Model: `models/gemini-3-flash-preview` (or a stable Gemini model you have access to)
   - Configure Gemini credentials in n8n (Google AI / Gemini).

9) **Add Structured Output Parser for music prompts**
   - Node: **Structured Output Parser**
   - Name: “Music Prompt Output Parser”
   - Schema: an object with `songs: [{song:number, song_prompt:string}]`
   - (Recommended) Adjust the agent system prompt to output this object shape.

10) **Add AI Agent for prompt generation**
   - Node: **AI Agent (LangChain)**
   - Name: “Generate Music Prompts (AI Agent)”
   - Connect AI model: Gemini LLM → Agent (ai_languageModel)
   - Connect output parser: Music Prompt Output Parser → Agent (ai_outputParser)
   - Text input: include form fields:
     - Theme: `{{ $('Suno Wave Music Input Form').item.json['Music theme'] }}`
     - Track count: `{{ $('Suno Wave Music Input Form').item.json['Number of tracks'] }}`
   - Connect: Retrieve Uploaded Video → Generate Music Prompts

11) **Split prompts into items**
   - Node: **Split Out**
   - Field: `output.songs`
   - Connect: Agent → Split Out

12) **Create song generation tasks (Kie.ai / Suno)**
   - Node: **HTTP Request**
   - Name: “Create Song (Suno API)”
   - Method: POST
   - URL: `https://api.kie.ai/api/v1/generate`
   - Auth: Header Auth credential with Kie.ai API key
   - Body parameters:
     - prompt: `{{$json.song_prompt}}`
     - instrumental: `true`
     - model: `V5`
     - customMode: `false`
   - Connect: Split Out → Create Song

13) **Polling loop until SUCCESS**
   - Node: **Wait**
   - Configure as time-based delay (recommended 30–60s) to avoid indefinite pause.
   - Connect: Create Song → Wait
   - Node: **HTTP Request** “Get Generated Song”
     - URL: `https://api.kie.ai/api/v1/generate/record-info` (remove trailing space)
     - Query: `taskId = {{$json.data.taskId}}`
     - Auth: Kie.ai header auth
   - Connect: Wait → Get Generated Song
   - Node: **IF** “Check Song Status”
     - Condition: `{{$json.data.status}}` equals `SUCCESS`
   - Connect: Get Generated Song → IF
   - IF false → back to Wait (optionally with a max retry guard)
   - IF true → continue

14) **Log metadata to Google Sheets**
   - Node: **Google Sheets**
   - Operation: Append
   - Spreadsheet: create/select your sheet
   - Map columns: title, duration, audioUrl, (and optionally theme)
   - Connect: IF true → Google Sheets

15) **Reserve audio path + download + upload each MP3 to ffmpeg-api**
   - HTTP Request “Prepare Audio Path (ffmpeg-api)” (POST `/file`)
     - file_name: `{{ $json.Tilte_Song }}_{{ $execution.id }}`
     - dir_id: from working directory node
   - HTTP Request “Download Song Audio”:
     - URL: from sheet item `Song_URL`
     - Response: file, set **fixed** binary property name (recommended `audio`)
   - HTTP Request “Upload Song to Workspace (ffmpeg-api)”:
     - PUT to `{{ $('Prepare Audio Path (ffmpeg-api)').item.json.upload.url }}`
     - Binary property: `audio` (do not hard-code a track-specific name)
   - HTTP Request “Retrieve Uploaded Song” (GET `/file/{file_path}`)
   - Connect in sequence: Sheets → Prepare Audio Path → Download → Upload → Retrieve

16) **Aggregate all uploaded audio paths**
   - Node: **Aggregate**
   - Aggregate: `file_info.file_path` into an array
   - Connect: Retrieve Uploaded Song → Aggregate

17) **Build ffmpeg job**
   - Node: **Code**
   - Read:
     - video path from Retrieve Uploaded Video (`file_info.file_path`)
     - audio paths from the Aggregate node output (ensure the field name matches)
   - Output object with `ffmpeg_job.task.inputs`, `filter_complex`, and `outputs` as required by ffmpeg-api
   - Connect: Aggregate → Code

18) **Run ffmpeg merge**
   - Node: **HTTP Request**
   - Name: “Merge Audio with Video (ffmpeg-api)”
   - Method: POST
   - URL: `https://api.ffmpeg-api.com/ffmpeg/process`
   - Auth: ffmpeg-api header credential
   - JSON body: the `ffmpeg_job` object produced by Code node
   - Connect: Code → Merge

19) **Upload final video to Blotato**
   - Node: **Blotato**
   - Resource: `media`
   - mediaUrl: `{{ $json.result[0].download_url }}`
   - Configure Blotato credentials/account in n8n
   - Connect: Merge → Upload Media On Blotato

20) **Generate SEO metadata (Gemini)**
   - Add Gemini model node (or reuse): “Gemini LLM1”
   - Add structured parser: “YouTube Metadata Parser” expecting JSON `{title, description, hashtags}`
   - Add AI Agent: “Generate SEO Title & Description”
     - Fix the input field reference to `Music theme` (not `Music Theme`)
     - Ensure the agent outputs JSON matching the parser (recommended).
   - Connect: Upload Media On Blotato → SEO Agent

21) **Publish to YouTube via Blotato**
   - Node: **Blotato**
   - Platform: youtube
   - AccountId: select your connected YouTube account
   - Title: from SEO agent output
   - Description text: combine description + hashtags
   - Media URL: from Upload Media On Blotato (`url`)
   - Set “contains synthetic media”: true (as in workflow)
   - Connect: SEO Agent → Publish

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Author: GiangxAI | https://www.youtube.com/@giangxai.official |
| Setup guide n8n link | https://n8n.partnerlinks.io/giangxai |
| Suno API provider used (Kie.ai) | https://kie.ai?ref=f8cec88ea15f9ecbff52ccbafa41dd6e |
| ffmpeg-api service | https://ffmpeg-api.com |
| Blotato (upload/publish to YouTube) | https://blotato.com/?ref=giang9s |
| Workflow runs end-to-end without manual editing once configured | From Sticky Note5 |

