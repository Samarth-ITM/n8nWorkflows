Create YouTube videos with OpenAI scripts, ElevenLabs voice, Pixabay and Shotstack

https://n8nworkflows.xyz/workflows/create-youtube-videos-with-openai-scripts--elevenlabs-voice--pixabay-and-shotstack-12392


# Create YouTube videos with OpenAI scripts, ElevenLabs voice, Pixabay and Shotstack

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automatically generate a YouTube-ready video by:
- Generating a ~5-minute narration script with OpenAI
- Converting it to an MP3 voiceover with ElevenLabs
- Searching and selecting stock video clips from Pixabay
- Assembling and rendering a final MP4 via Shotstack
- Storing voiceover and final video in Google Drive

**Target use cases:**
- Automated content creation pipelines (faceless channels, explainer videos)
- Rapid prototyping of narrated videos for marketing/education
- Scheduled generation of recurring video content

### Logical blocks
**1.1 Orchestration & Parameters (Entry Point)**  
Schedule-triggered execution and definition of topic/duration/number of clips.

**1.2 Stock Footage Retrieval (Pixabay)**  
Search Pixabay videos by topic and extract usable video URLs with metadata.

**1.3 Script + Voiceover Generation (OpenAI → ElevenLabs → Google Drive)**  
Generate spoken script, synthesize MP3, upload it to Drive, and (attempt to) make it public.

**1.4 Merge + Render Composition (Shotstack JSON build)**  
Merge footage list with voiceover availability and build a Shotstack render payload.

**1.5 Render Submission, Polling, Download, Archival (Shotstack → Drive)**  
Submit render job, poll until complete, download MP4, and save to Drive.

---

## 2. Block-by-Block Analysis

### 2.1 Orchestration & Parameters (Entry Point)

**Overview:** Starts the workflow on a schedule and defines the key parameters used downstream (search query, target duration, number of clips).  
**Nodes involved:** `Schedule Trigger`, `Set Video Parameters1`

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` (starts workflow periodically)
- **Config choices:** Uses an interval rule (not fully specified in JSON; defaults may apply depending on UI state).
- **Outputs:** Sends one execution to **two branches**:
  - to `Set Video Parameters1` (video branch)
  - to `Generate Script2` (voice/script branch)
- **Edge cases/failures:**
  - Misconfigured interval (runs too frequently or not at all).
  - Timezone surprises (instance timezone vs expected).
- **Version notes:** `typeVersion: 1.3`

#### Node: Set Video Parameters1
- **Type / role:** `Set` (defines runtime parameters)
- **Config choices (interpreted):**
  - `topic`: `"productivity business workflow"`
  - `videoDuration`: `"300"` (string)
  - `numberOfClips`: `"10"` (string)
- **Key variables:** `$json.topic`, `$json.videoDuration`, `$json.numberOfClips` used later.
- **Connections:**
  - Input: `Schedule Trigger`
  - Output: `Search Pixabay Videos`
- **Edge cases/failures:**
  - Values stored as strings; downstream nodes/code may assume numbers.
  - `videoDuration` is not actually used by the Shotstack builder (hardcoded to 300 there), so changing this set value won’t change the final length unless code is updated.
- **Version notes:** `typeVersion: 1`

---

### 2.2 Stock Footage Retrieval (Pixabay)

**Overview:** Calls Pixabay’s video search API using the topic, then converts the API response into a list of clip items containing direct video URLs.  
**Nodes involved:** `Search Pixabay Videos`, `Extract Video URLs1`

#### Node: Search Pixabay Videos
- **Type / role:** `HTTP Request` (Pixabay API call)
- **Config choices:**
  - URL: `https://pixabay.com/api/videos/`
  - Method: `POST` (Pixabay examples often use GET; this may still work if the API accepts query params regardless of method)
  - Sends **query parameters** with “Query Auth”:
    - `key` = `API_Key` (literal placeholder; should be replaced by your credential/expression)
    - `q` = `{{ $json.topic }}`
    - `per_page` = `{{ $json.numberOfClips }}`
    - `video_type` = `all`
  - Response format: JSON
- **Connections:**
  - Input: `Set Video Parameters1`
  - Output: `Extract Video URLs1`
- **Edge cases/failures:**
  - Invalid API key → 401/403.
  - If `per_page` is a string, API typically coerces, but could fail or default.
  - Empty results if topic too narrow.
  - Rate limits.
- **Credentials:** Generic credential type with query auth (`genericAuthType: queryAuth`)
- **Version notes:** `typeVersion: 4.1`

#### Node: Extract Video URLs1
- **Type / role:** `Code` (normalizes Pixabay response into individual clip items)
- **Config choices:**
  - Reads: `$input.item.json.hits`
  - For each hit, picks the “best available” URL:
    - `video.videos.large.url` else `medium` else `small`
  - Emits items shaped like:
    - `url`, `duration` (defaults to 15), `id`, `width`, `height`
  - Throws errors if no videos found or no valid URLs.
- **Connections:**
  - Input: `Search Pixabay Videos`
  - Output: `Merge` (Branch 0)
- **Edge cases/failures:**
  - Pixabay schema changes (missing `hits` or `videos`).
  - Execution stops due to thrown error; consider converting errors into empty branch handling if desired.
- **Version notes:** `typeVersion: 2`

---

### 2.3 Script + Voiceover Generation (OpenAI → ElevenLabs → Google Drive)

**Overview:** Generates a conversational 5-minute script with OpenAI, synthesizes voice using ElevenLabs, uploads MP3 to Google Drive, then attempts to share it publicly (loop node present, but see caveat).  
**Nodes involved:** `Generate Script2`, `HTTP Request`, `Upload Voiceover to Drive`, `Make Voiceover Public`, `Loop Over Items`

#### Node: Generate Script2
- **Type / role:** `OpenAI` (Chat completion to generate script)
- **Config choices:**
  - Resource: `chat`
  - Provides a single “assistant” message containing detailed instructions to produce spoken-word script only.
  - `simplifyOutput: false` → output retains full OpenAI response structure (important for the ElevenLabs node which reads `$json.choices[0].message.content`)
- **Connections:**
  - Input: `Schedule Trigger`
  - Output: `HTTP Request` (ElevenLabs)
- **Credentials:** `OpenAi account`
- **Edge cases/failures:**
  - Auth/quotas.
  - Model/provider changes could alter output shape; but the node references `choices[0].message.content` which is typical for chat responses.
  - Script length may exceed desired 5 minutes depending on speaking rate; no post-validation is implemented.
- **Version notes:** `typeVersion: 1`, `alwaysOutputData: true`

#### Node: HTTP Request (ElevenLabs TTS)
- **Type / role:** `HTTP Request` (text-to-speech)
- **Config choices:**
  - URL: `https://api.elevenlabs.io/v1/text-to-speech/kPzsL2i3teMYv0FxEYQ6` (voice ID embedded)
  - Method: POST
  - Headers: `Content-Type: application/json`
  - Body parameters:
    - `text` = `{{ $json.choices[0].message.content }}`
    - `voice_settings.stability` = `0.5`
    - `voice_settings.similarity_boost` = `0.75`
    - `voice_settings.style` = `0.0`
    - `voice_settings.use_speaker_boost` = `true`
  - Auth: HTTP Header Auth credential (likely `xi-api-key`)
- **Connections:**
  - Input: `Generate Script2`
  - Output: `Upload Voiceover to Drive`
- **Edge cases/failures:**
  - If ElevenLabs returns audio bytes, the node must be configured to handle binary audio; current config doesn’t explicitly set “response format: file”. This is a common break point.
  - Wrong/missing voice ID → 404.
  - Content too long → 400/413 depending on ElevenLabs limits.
- **Credentials:** `ElevenLabs API` via header auth
- **Version notes:** `typeVersion: 4.3`

#### Node: Upload Voiceover to Drive
- **Type / role:** `Google Drive` (file upload)
- **Config choices:**
  - Filename: `voiceover_{{ $now.toFormat('yyyy-MM-dd_HH-mm-ss') }}.mp3`
  - Drive: “My Drive”, folder `root`
- **Connections:**
  - Input: ElevenLabs `HTTP Request`
  - Output: `Loop Over Items`
- **Credentials:** Google Drive OAuth2
- **Edge cases/failures:**
  - If the ElevenLabs node didn’t produce binary MP3 data, upload will fail or upload a JSON/text payload.
  - OAuth token expiration / insufficient scopes.
- **Version notes:** `typeVersion: 3`

#### Node: Make Voiceover Public
- **Type / role:** `Google Drive` (share/permissions)
- **Config choices:**
  - Operation: `share`
  - `fileId` = `{{ $json.id }}`
  - Permissions: `anyone` + `reader`
  - `alwaysOutputData: true` (continues even if no data?)
- **Connections:**
  - Input: `Loop Over Items` (but see note below)
  - Output: `Loop Over Items` (creates a loop)
- **Edge cases/failures:**
  - Domain or Drive restrictions may prevent “anyone with the link”.
  - If it receives items not containing `id`, expression fails.
- **Version notes:** `typeVersion: 3`

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` (iterative loop control)
- **Config choices:** Default batch configuration (not specified).
- **Connections (as wired):**
  - Input: `Upload Voiceover to Drive` and also `Make Voiceover Public`
  - Outputs:
    - Output 0 → `Make Voiceover Public`
    - Output 1 → `Merge` (Branch 1)
- **Important wiring caveat:**  
  As connected, `Make Voiceover Public` feeds back into `Loop Over Items`, which can create an unintended loop. Typically, `Split In Batches` should:
  1) take input items,
  2) send a batch to a processing node,
  3) then return to the split node to request the next batch.
  Here, because the voiceover upload likely outputs **a single item**, the split loop is unnecessary. A simpler and safer pattern is: `Upload Voiceover to Drive` → `Make Voiceover Public` → `Merge`.
- **Edge cases/failures:**
  - Infinite looping or premature termination depending on batch settings and item count.
- **Version notes:** `typeVersion: 3`

---

### 2.4 Merge + Render Composition (Shotstack JSON build)

**Overview:** Waits until both the clip list and voiceover branch are ready, then constructs the Shotstack render payload referencing the Google Drive-hosted audio.  
**Nodes involved:** `Merge`, `Build Shotstack JSON1`

#### Node: Merge
- **Type / role:** `Merge` (synchronizes branches)
- **Config choices:** `mode: chooseBranch`
  - Branch 0 receives items from `Extract Video URLs1` (multiple clip items)
  - Branch 1 receives from `Loop Over Items` (intended to signal voiceover is ready)
- **Connections:**
  - Output → `Build Shotstack JSON1`
- **Edge cases/failures:**
  - “Choose Branch” doesn’t combine data; it selects one input depending on execution order/availability. For this workflow, the Code node later directly references the voiceover upload node by name, so it can still work even if merge doesn’t actually merge—however it’s fragile.
  - If branch 1 never emits (loop issues), render block may never run.
- **Version notes:** `typeVersion: 3.2`

#### Node: Build Shotstack JSON1
- **Type / role:** `Code` (creates Shotstack render JSON)
- **Config choices / logic:**
  - Input clips: `$input.all().map(item => item.json)` (expects multiple clip items)
  - Fetches voiceover file ID via node reference:
    - `$('Upload Voiceover to Drive').first().json.id`
  - Constructs voiceover URL as direct download:
    - `https://drive.google.com/uc?export=download&id=${driveFileId}`
  - Timeline:
    - Soundtrack uses `fadeInFadeOut`, volume 1.0
    - One track of video clips with transitions (rotating list)
  - Duration math:
    - `totalDuration = 300` hardcoded
    - `transitionDuration = 0.8`
    - Computes per-clip length to fit total with overlaps
  - Output settings: mp4, hd, fps 25, quality high
- **Connections:**
  - Output → `Submit Render Job1`
- **Edge cases/failures:**
  - Hardcoded `totalDuration` ignores `Set Video Parameters1.videoDuration`.
  - Google Drive “uc?export=download” may not work reliably if Drive blocks hotlinking or requires virus scan confirmation for larger files.
  - If the file isn’t truly public, Shotstack cannot fetch the soundtrack URL.
  - Clip URLs from Pixabay might be temporary or geo-restricted; Shotstack fetch may fail.
- **Version notes:** `typeVersion: 2`

---

### 2.5 Render Submission, Polling, Download, Archival (Shotstack → Drive)

**Overview:** Sends the render request to Shotstack, polls the status until complete, downloads the rendered MP4, and saves it to a specific Google Drive folder.  
**Nodes involved:** `Submit Render Job1`, `Wait for Render1`, `Check Render Status1`, `Is Render Complete?1`, `Download Final Video1`, `Save to Google Drive1`

#### Node: Submit Render Job1
- **Type / role:** `HTTP Request` (creates Shotstack render job)
- **Config choices:**
  - URL: `https://api.shotstack.io/v1/render`
  - Method: POST
  - Body: JSON via `JSON.stringify($json)`
  - Timeout: 30000 ms
  - Auth: HTTP Header Auth credential (Shotstack API key)
  - Header parameters list contains an empty object `{}` (likely a misconfiguration; normally you’d send `x-api-key: ...` via credential plus `Content-Type: application/json`)
- **Connections:** Output → `Wait for Render1`
- **Edge cases/failures:**
  - Missing/incorrect headers → 401.
  - Sandbox endpoint limits, queue delays.
  - 30s timeout usually fine for submission, not for render completion.
- **Version notes:** `typeVersion: 4.1`

#### Node: Wait for Render1
- **Type / role:** `Wait` (delay between polling attempts)
- **Config choices:** Wait 60 seconds
- **Connections:** Output → `Check Render Status1`
- **Edge cases/failures:**
  - Too short → many polls; too long → slow completion.
- **Version notes:** `typeVersion: 1`

#### Node: Check Render Status1
- **Type / role:** `HTTP Request` (poll Shotstack job status)
- **Config choices:**
  - URL: `https://api.shotstack.io/v1/render/{{ $json.response.id }}`
  - Header: `Content-Type: application/json`
  - Auth: HTTP Header Auth (Shotstack)
- **Connections:** Output → `Is Render Complete?1`
- **Edge cases/failures:**
  - Assumes prior node output contains `$json.response.id`. If Shotstack returns a different shape (or error), expression fails.
- **Version notes:** `typeVersion: 4.1`

#### Node: Is Render Complete?1
- **Type / role:** `If` (checks completion)
- **Config choices:**
  - Condition: string `value1 = {{ $json.success }}` contains `"true"`
  - True branch → download
  - False branch → wait again
- **Connections:**
  - True → `Download Final Video1`
  - False → `Wait for Render1` (poll loop)
- **Edge cases/failures:**
  - Shotstack commonly uses fields like `status` (`done`, `rendering`, `failed`)—checking `$json.success` may be incorrect. This could cause never-ending polling or premature download attempts.
- **Version notes:** `typeVersion: 1`

#### Node: Download Final Video1
- **Type / role:** `HTTP Request` (downloads MP4 file)
- **Config choices:**
  - URL: `{{ $json.response.url }}`
  - Response format: `file` (binary)
- **Connections:** Output → `Save to Google Drive1`
- **Edge cases/failures:**
  - If status response doesn’t include `response.url`, download fails.
  - Large file download timeouts (node options not tuned).
- **Version notes:** `typeVersion: 4.1`

#### Node: Save to Google Drive1
- **Type / role:** `Google Drive` (uploads rendered MP4)
- **Config choices:**
  - Name: `final_video_{{ $now.toFormat('yyyy-MM-dd_HH-mm-ss') }}.mp4`
  - Folder: `Archive` (ID: `1ymC_Lf5Zi7wf3EsMIEV687PMclAnDlkV`)
- **Connections:** terminal node
- **Edge cases/failures:**
  - Requires binary data from previous node; otherwise uploads wrong content.
  - Folder permissions, OAuth scope, quota.
- **Version notes:** `typeVersion: 3`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Workflow entry point / scheduler | — | Set Video Parameters1; Generate Script2 | ## Automated YouTube Video Creator with AI Voiceover, Pixabay Videos to Drive (plus “How it works” + “Setup” content with links to Pixabay, Elevenlabs, Shotstack, OpenAI, Google Cloud) |
| Set Video Parameters1 | Set | Defines topic, duration, clip count | Schedule Trigger | Search Pixabay Videos | ## 1. Choose video clips |
| Search Pixabay Videos | HTTP Request | Search stock videos on Pixabay | Set Video Parameters1 | Extract Video URLs1 | ## 1. Choose video clips |
| Extract Video URLs1 | Code | Extract/normalize video clip URLs | Search Pixabay Videos | Merge | ## 1. Choose video clips |
| Generate Script2 | OpenAI | Generate spoken YouTube script | Schedule Trigger | HTTP Request | ## 2. Generate voiceover |
| HTTP Request | HTTP Request | ElevenLabs TTS synthesis | Generate Script2 | Upload Voiceover to Drive | ## 2. Generate voiceover |
| Upload Voiceover to Drive | Google Drive | Upload MP3 to Drive | HTTP Request | Loop Over Items | ## 2. Generate voiceover |
| Loop Over Items | Split In Batches | (Intended) loop controller for sharing | Upload Voiceover to Drive; Make Voiceover Public | Make Voiceover Public; Merge | ## 2. Generate voiceover |
| Make Voiceover Public | Google Drive | Set file permission to “anyone can read” | Loop Over Items | Loop Over Items | ## 2. Generate voiceover |
| Merge | Merge | Synchronize branches (video list + voiceover readiness) | Extract Video URLs1; Loop Over Items | Build Shotstack JSON1 | (General description sticky note applies) |
| Build Shotstack JSON1 | Code | Build Shotstack render payload (timeline/output) | Merge | Submit Render Job1 | ## 3. Create video and store in Google Drive |
| Submit Render Job1 | HTTP Request | Submit render request to Shotstack | Build Shotstack JSON1 | Wait for Render1 | ## 3. Create video and store in Google Drive |
| Wait for Render1 | Wait | Delay between status polls | Submit Render Job1; Is Render Complete?1 (false) | Check Render Status1 | ## 3. Create video and store in Google Drive |
| Check Render Status1 | HTTP Request | Poll Shotstack render status | Wait for Render1 | Is Render Complete?1 | ## 3. Create video and store in Google Drive |
| Is Render Complete?1 | If | Decide whether render is finished | Check Render Status1 | Download Final Video1; Wait for Render1 | ## 3. Create video and store in Google Drive |
| Download Final Video1 | HTTP Request | Download rendered MP4 | Is Render Complete?1 (true) | Save to Google Drive1 | ## 3. Create video and store in Google Drive |
| Save to Google Drive1 | Google Drive | Archive final MP4 to Drive folder | Download Final Video1 | — | ## 3. Create video and store in Google Drive |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. **OpenAI credential** in n8n (API key).
   2. **HTTP Header Auth credential** for **ElevenLabs** (header typically `xi-api-key: <key>`).
   3. **HTTP Header Auth credential** for **Shotstack** (header typically `x-api-key: <key>`; use sandbox or production base URL consistently).
   4. **Google Drive OAuth2** credential:
      - Enable Google Drive API in Google Cloud.
      - Add OAuth consent + scopes (Drive file access).
      - Connect in n8n and verify you can upload a file.

2) **Add “Schedule Trigger”**
   - Node: **Schedule Trigger**
   - Configure interval/cron as desired.
   - This node will feed two branches.

3) **Video branch: parameters + Pixabay**
   1. Add node: **Set** named `Set Video Parameters1`
      - Add fields:
        - `topic` (string) e.g. `productivity business workflow`
        - `videoDuration` (number preferred; if string, adjust code later) e.g. `300`
        - `numberOfClips` (number) e.g. `10`
   2. Add node: **HTTP Request** named `Search Pixabay Videos`
      - Method: GET (recommended) or POST (as in workflow)
      - URL: `https://pixabay.com/api/videos/`
      - Send query parameters:
        - `key`: your Pixabay API key (prefer storing in n8n credential or env var)
        - `q`: `{{$json.topic}}`
        - `per_page`: `{{$json.numberOfClips}}`
        - `video_type`: `all`
      - Response: JSON
   3. Add node: **Code** named `Extract Video URLs1`
      - Use code equivalent to:
        - read `hits`
        - choose `large|medium|small` URL
        - return one item per clip with `url`, `duration`, `width`, `height`, `id`
      - Connect: `Set Video Parameters1` → `Search Pixabay Videos` → `Extract Video URLs1`

4) **Voice branch: OpenAI → ElevenLabs → Drive**
   1. Add node: **OpenAI** named `Generate Script2`
      - Resource: Chat
      - Prompt/messages: paste the script instructions (spoken-only output).
      - Ensure output is not overly simplified so `choices[0].message.content` exists (or adjust mapping).
   2. Add node: **HTTP Request** named `HTTP Request` (ElevenLabs)
      - Method: POST
      - URL: `https://api.elevenlabs.io/v1/text-to-speech/<VOICE_ID>`
      - Headers: `Content-Type: application/json`
      - Auth: select ElevenLabs Header Auth credential
      - Body JSON:
        - `text`: `{{$json.choices[0].message.content}}`
        - include `voice_settings` as desired
      - **Important:** Configure response to return a **file/binary** (MP3). In n8n, set the response format to `File` if available, or ensure “Download”/binary options are enabled so the next node can upload MP3.
   3. Add node: **Google Drive** named `Upload Voiceover to Drive`
      - Operation: Upload
      - Binary property: the binary output from ElevenLabs (commonly `data`; confirm in execution)
      - File name: `voiceover_{{$now.toFormat('yyyy-MM-dd_HH-mm-ss')}}.mp3`
      - Folder: `root` (or your chosen folder)
   4. Add node: **Google Drive** named `Make Voiceover Public`
      - Operation: Share
      - File ID: `{{$json.id}}`
      - Permission: `anyone` + `reader`
   5. (Recommended simplification) **Skip Split In Batches** unless you truly have multiple files. Connect:
      - `Upload Voiceover to Drive` → `Make Voiceover Public`
      - Then send to merge (next section)

5) **Merge branches and build Shotstack render**
   1. Add node: **Merge** named `Merge`
      - If you want strict synchronization, use a merge mode that waits for both inputs (e.g., “Wait for both” / “Combine” depending on n8n version).  
      - Connect:
        - `Extract Video URLs1` → `Merge` (Input 1)
        - `Make Voiceover Public` (or the end of voice branch) → `Merge` (Input 2)
   2. Add node: **Code** named `Build Shotstack JSON1`
      - Build `timeline` and `output` JSON.
      - Reference the Drive file ID from the correct incoming item or by node reference (as in the workflow):
        - `const driveFileId = $('Upload Voiceover to Drive').first().json.id;`
      - Create a publicly accessible soundtrack URL (Drive `uc?export=download`), or use another hosting method if Drive hotlinking is unreliable.
      - Use clips from `$input.all()` to build track clips with transitions.
      - (Optional) Replace hardcoded `totalDuration` with `Number($('Set Video Parameters1').first().json.videoDuration)`.

6) **Submit render to Shotstack and poll**
   1. Add node: **HTTP Request** named `Submit Render Job1`
      - POST `https://api.shotstack.io/v1/render`
      - Auth: Shotstack header credential
      - Send JSON body = the output of `Build Shotstack JSON1`
      - Timeout ~30s
   2. Add node: **Wait** named `Wait for Render1`
      - 60 seconds
   3. Add node: **HTTP Request** named `Check Render Status1`
      - GET `https://api.shotstack.io/v1/render/{{$json.response.id}}`
      - Auth: Shotstack header credential
   4. Add node: **If** named `Is Render Complete?1`
      - Prefer checking a status field (commonly `response.status == "done"`) rather than `$json.success`.
      - True: proceed to download
      - False: loop back to wait

7) **Download final MP4 and save**
   1. Add node: **HTTP Request** named `Download Final Video1`
      - URL: `{{$json.response.url}}`
      - Response format: `File`
   2. Add node: **Google Drive** named `Save to Google Drive1`
      - Upload binary MP4 to your target folder (e.g., “Archive”)
      - Name: `final_video_{{$now.toFormat('yyyy-MM-dd_HH-mm-ss')}}.mp4`

8) **Connect everything**
   - `Schedule Trigger` → `Set Video Parameters1` → `Search Pixabay Videos` → `Extract Video URLs1` → `Merge`
   - `Schedule Trigger` → `Generate Script2` → `ElevenLabs HTTP Request` → `Upload Voiceover to Drive` → `Make Voiceover Public` → `Merge`
   - `Merge` → `Build Shotstack JSON1` → `Submit Render Job1` → `Wait for Render1` → `Check Render Status1` → `Is Render Complete?1`
   - `Is Render Complete?1 (true)` → `Download Final Video1` → `Save to Google Drive1`
   - `Is Render Complete?1 (false)` → `Wait for Render1`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automated YouTube Video Creator with AI Voiceover, Pixabay Videos to Drive | Workflow sticky note description |
| Pixabay videos API | https://pixabay.com/api/docs/ |
| ElevenLabs (speech synthesis) | https://elevenlabs.io/ and https://elevenlabs.io/app/speech-synthesis/speech-to-speech |
| Shotstack API keys | https://dashboard.shotstack.io/keys |
| OpenAI API settings | https://platform.openai.com/settings/ |
| Google Cloud Console (enable Drive API) | https://console.cloud.google.com/ |
| Setup checklist (keys, credentials, prompt, topic) | Included in the main sticky note content |

