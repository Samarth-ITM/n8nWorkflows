Translate and upload dubbed YouTube videos using ElevenLabs and Google Drive

https://n8nworkflows.xyz/workflows/translate-and-upload-dubbed-youtube-videos-using-elevenlabs-and-google-drive-12723


# Translate and upload dubbed YouTube videos using ElevenLabs and Google Drive

## 1. Workflow Overview

**Purpose:**  
This workflow automates the process of downloading a source video from a URL, sending it to **ElevenLabs Dubbing** to generate a dubbed version in a target language, waiting for processing, downloading the final dubbed media, and then uploading it to **Google Drive** and optionally publishing/scheduling it through **Postiz** or uploading via **Upload-Post.com** for YouTube.

**Typical use cases:**
- Translating YouTube Shorts or videos into another language at scale
- Centralizing dubbed outputs in Google Drive
- Pushing localized media to publishing/scheduling platforms

### 1.1 Execution & Input Parameters
Manual start, then a parameter setup step defines the source `video_url` and the target dubbing language `target_audio`.

### 1.2 Video Retrieval
Downloads the source video file via HTTP into binary data for subsequent upload to ElevenLabs.

### 1.3 ElevenLabs Dubbing Job Creation
Submits the downloaded video to the ElevenLabs Dubbing API (multipart upload) and receives a `dubbing_id` plus expected processing time.

### 1.4 Async Wait + Status Retrieval
Waits a computed amount of time based on ElevenLabs’ `expected_duration_sec`, then fetches dubbing job status/details.

### 1.5 Download Dubbed Output
Downloads the final dubbed output from ElevenLabs using `dubbing_id` and the returned target language code.

### 1.6 Distribution (Drive + Publishing Options)
Uploads the dubbed file to:
- **Google Drive**
- **Postiz** (via direct upload endpoint, then a Postiz node schedules/posts)
- **Upload-Post.com** (YouTube upload gateway)

---

## 2. Block-by-Block Analysis

### Block 1 — Execution & Input Parameters

**Overview:** Starts the workflow manually and sets the key inputs used by all subsequent nodes: the source video URL and the desired target language.  
**Nodes involved:**  
- When clicking ‘Execute workflow’
- Set params

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — starts the workflow on demand.
- **Key configuration:** No parameters.
- **Connections:**
  - **Output →** Set params
- **Failure/edge cases:** None (only runs when manually executed).
- **Version notes:** TypeVersion `1` (standard).

#### Node: Set params
- **Type / role:** Set node (`n8n-nodes-base.set`) — defines the variables used downstream.
- **Configuration choices:**
  - Creates two string fields:
    - `video_url` = `VIDEO_URL` (placeholder; must be replaced with a direct downloadable video URL)
    - `target_audio` = `es` (example language code)
- **Key expressions/variables:**
  - Downstream nodes reference:
    - `{{$json.video_url}}`
    - `{{$json.target_audio}}`
- **Connections:**
  - **Input ←** Manual Trigger
  - **Output →** Get video
- **Failure/edge cases:**
  - If `video_url` is not a direct file URL (e.g., a YouTube page URL), the download node may return HTML or fail.
  - Invalid/unsupported `target_audio` may cause ElevenLabs API errors.
- **Version notes:** TypeVersion `3.4`.

---

### Block 2 — Video Retrieval

**Overview:** Downloads the source video so it can be sent as multipart binary to ElevenLabs.  
**Nodes involved:**  
- Get video

#### Node: Get video
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — fetches the remote video content.
- **Configuration choices:**
  - **URL:** `={{ $json.video_url }}`
  - Method defaults to GET (not explicitly set)
  - No explicit “Download”/response format options shown in the JSON; however, the next node expects **binary** data named `data`.
- **Connections:**
  - **Input ←** Set params
  - **Output →** Elevenlabs Dubbing
- **Failure/edge cases:**
  - If the server requires auth/cookies, the request may fail (403) or return an HTML login page.
  - Large videos may hit timeouts unless HTTP node timeout/options are adjusted.
  - If n8n does not store the response as binary in field `data`, the ElevenLabs multipart upload will fail (missing binary property).
- **Version notes:** TypeVersion `4.3`.

---

### Block 3 — ElevenLabs Dubbing Job Creation

**Overview:** Submits the video file to ElevenLabs Dubbing API, creating an async dubbing job.  
**Nodes involved:**  
- Elevenlabs Dubbing

#### Node: Elevenlabs Dubbing
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — multipart POST to ElevenLabs dubbing endpoint.
- **Configuration choices:**
  - **URL:** `https://api.elevenlabs.io/v1/dubbing`
  - **Method:** POST
  - **Content-Type:** `multipart-form-data`
  - **Authentication:** Predefined credential type → `elevenLabsApi`
  - **Body (multipart):**
    - `file`: binary from `inputDataFieldName: data`
    - `target_lang`: `={{ $json.target_audio }}`
- **Key expressions/variables:**
  - Uses `{{$json.target_audio}}` from Set params.
  - Produces `dubbing_id` and `expected_duration_sec` (used later by Wait).
- **Connections:**
  - **Input ←** Get video
  - **Output →** Wait
- **Failure/edge cases:**
  - ElevenLabs credential missing/invalid → 401/403.
  - Wrong binary field name (must match where “Get video” stores the file; expected `data`) → 400 or request formed without file.
  - Unsupported language code → 4xx.
  - File size/format limits enforced by ElevenLabs.
- **Version notes:** TypeVersion `4.3`.
- **Credentials:** `ElevenLabs account` (preconfigured in n8n).

---

### Block 4 — Async Wait + Status Retrieval

**Overview:** Waits a dynamic amount of time based on ElevenLabs’ estimate, then retrieves job status/details using the returned dubbing ID.  
**Nodes involved:**  
- Wait  
- Get Dubbing ID

#### Node: Wait
- **Type / role:** Wait (`n8n-nodes-base.wait`) — pauses execution to let dubbing complete.
- **Configuration choices:**
  - **Amount (seconds):** `={{ $json.expected_duration_sec + 120 }}`
    - Adds a 120-second buffer to ElevenLabs’ expected duration.
- **Key expressions/variables:**
  - `expected_duration_sec` must exist in the incoming JSON from Elevenlabs Dubbing.
- **Connections:**
  - **Input ←** Elevenlabs Dubbing
  - **Output →** Get Dubbing ID
- **Failure/edge cases:**
  - If `expected_duration_sec` is missing/not numeric → expression evaluation error.
  - Dubbing may take longer than estimated; workflow may continue too early and later download step can fail.
- **Version notes:** TypeVersion `1.1`.

#### Node: Get Dubbing ID
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — fetches dubbing job details/status.
- **Configuration choices:**
  - **URL:** `https://api.elevenlabs.io/v1/dubbing/{{ $('Elevenlabs Dubbing').item.json.dubbing_id }}`
  - **Authentication:** Predefined credential type → `elevenLabsApi`
- **Key expressions/variables:**
  - `{{ $('Elevenlabs Dubbing').item.json.dubbing_id }}` explicitly references the earlier node output.
- **Connections:**
  - **Input ←** Wait
  - **Output →** Download final Video
- **Failure/edge cases:**
  - If the referenced item context differs (multiple items) the `.item` reference can point unexpectedly; in batch scenarios, consider using `{{$json.dubbing_id}}` passed through instead.
  - If job not ready yet, API may return status indicating processing; next step might fail.
- **Version notes:** TypeVersion `4.3`.
- **Credentials:** `ElevenLabs account`.

---

### Block 5 — Download Dubbed Output

**Overview:** Downloads the final dubbed media from ElevenLabs using the dubbing ID and the chosen target language returned by the status call.  
**Nodes involved:**  
- Download final Video

#### Node: Download final Video
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — downloads the generated dubbed output.
- **Configuration choices:**
  - **URL:** `https://api.elevenlabs.io/v1/dubbing/{{ $json.dubbing_id }}/audio/{{ $json.target_languages[0] }}`
  - **Authentication:** Predefined credential type → `elevenLabsApi`
  - (Options not shown; downstream nodes expect **binary** `data`.)
- **Key expressions/variables:**
  - `{{$json.dubbing_id}}` from “Get Dubbing ID”
  - `{{$json.target_languages[0]}}` assumes `target_languages` exists and is a non-empty array
- **Connections:**
  - **Input ←** Get Dubbing ID
  - **Outputs →** (fan-out)
    - Upload file (Google Drive)
    - Upload Video to Postiz
    - Youtube Upload-Post
- **Failure/edge cases:**
  - If `target_languages` missing/empty → expression error.
  - If the dubbing isn’t finished yet → 404/409 or an API error; consider polling until status = completed.
  - If response is not configured to be stored as binary, upload nodes expecting `$binary.data` will fail.
- **Version notes:** TypeVersion `4.3`.
- **Credentials:** `ElevenLabs account`.

---

### Block 6 — Distribution (Google Drive + Publishing Options)

**Overview:** Uploads the dubbed file to Google Drive and optionally pushes it to publishing platforms. The workflow currently sends the same binary output to three parallel branches.  
**Nodes involved:**  
- Upload file  
- Upload Video to Postiz  
- Youtube Postiz  
- Youtube Upload-Post  

#### Node: Upload file
- **Type / role:** Google Drive (`n8n-nodes-base.googleDrive`) — uploads the resulting dubbed file to a specific Drive folder.
- **Configuration choices:**
  - **File name:** `={{ $binary.data.fileName }}`
  - **Drive:** “My Drive”
  - **Folder:** ID `1tkCr7xdraoZwsHqeLm7FZ4aRWY94oLbZ` (cached name: `n8n`)
- **Key expressions/variables:**
  - Relies on `$binary.data.fileName` being present.
- **Connections:**
  - **Input ←** Download final Video
  - **Output:** none
- **Failure/edge cases:**
  - Missing/expired Google OAuth2 token → auth errors.
  - Folder permissions/ID wrong → 404/403.
  - Missing binary `data` or missing `fileName` → upload failure.
- **Version notes:** TypeVersion `3`.
- **Credentials:** `Google Drive account (n3w.it)`.

#### Node: Upload Video to Postiz
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — uploads the media file to Postiz via a public upload endpoint.
- **Configuration choices:**
  - **URL:** `https://api.postiz.com/public/v1/upload`
  - **Method:** POST
  - **Content-Type:** `multipart-form-data`
  - **Authentication:** Generic credential type → `httpHeaderAuth` (Postiz)
  - **Body (multipart):**
    - `file`: binary from `inputDataFieldName: data`
- **Connections:**
  - **Input ←** Download final Video
  - **Output →** Youtube Postiz
- **Failure/edge cases:**
  - Wrong/missing header auth → 401/403.
  - Postiz API may require specific filename/mime-type; if absent, could reject upload.
- **Version notes:** TypeVersion `4.2`.
- **Credentials:** `Postiz` (HTTP Header Auth).

#### Node: Youtube Postiz
- **Type / role:** Postiz node (`n8n-nodes-postiz.postiz`) — schedules/posts content using the uploaded media reference.
- **Configuration choices (interpreted):**
  - **Date:** `={{ $now.format('yyyy-LL-dd') }}T{{ $now.format('HH:ii:ss') }}`
    - Uses “now” in a specific timestamp format.
  - **Posts payload:**
    - Uses the upload response fields:
      - `id`: `={{ $json.id }}`
      - `path`: `={{ $json.path }}`
    - Includes placeholder content: `YOUR_CONTENT`
    - Uses placeholder `integrationId`: `XXX` (must be replaced with a real Postiz integration/channel id)
  - **shortLink:** enabled
- **Connections:**
  - **Input ←** Upload Video to Postiz
  - **Output:** none
- **Failure/edge cases:**
  - If upload response does not contain `id`/`path`, post creation fails.
  - Invalid `integrationId` → Postiz rejects request.
  - Timestamp formatting: Postiz may expect timezone or ISO8601; verify required format.
- **Version notes:** TypeVersion `1`.
- **Credentials:** `Postiz account` (Postiz API credential).

#### Node: Youtube Upload-Post
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — uploads video to YouTube through Upload-Post.com gateway.
- **Configuration choices:**
  - **URL:** `https://api.upload-post.com/api/upload`
  - **Method:** POST
  - **Content-Type:** `multipart-form-data`
  - **Authentication:** Generic credential type → `httpHeaderAuth` (Upload-post.com API)
  - **Body (multipart) fields:**
    - `title`: `YOUR_CONTENT` (placeholder)
    - `user`: `YOUR_USERNAME` (placeholder)
    - `platform[]`: `youtube`
    - `video`: binary from `inputDataFieldName: data`
- **Connections:**
  - **Input ←** Download final Video
  - **Output:** none
- **Failure/edge cases:**
  - Placeholders not replaced → upload may be rejected or posted with wrong metadata.
  - Upload-Post API may require additional fields (description, tags, privacy) depending on account configuration.
  - Auth header invalid/expired → 401/403.
- **Version notes:** TypeVersion `4.2`.
- **Credentials:** `Upload-post.com API` (HTTP Header Auth).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Set params |  |
| Set params | Set | Defines `video_url` and `target_audio` | When clicking ‘Execute workflow’ | Get video | ## STEP 1 - Set params; **Configure Input Parameters**; In the *Set params* node, define: `video_url` (direct URL), `target_audio` (language code) |
| Get video | HTTP Request | Downloads source video | Set params | Elevenlabs Dubbing | ## STEP 2- Elevenlabs Dubbing; The video file is sent to the **ElevenLabs Dubbing API**, which initiates audio dubbing in the specified target language. |
| Elevenlabs Dubbing | HTTP Request | Creates ElevenLabs dubbing job (multipart upload) | Get video | Wait | ## STEP 2- Elevenlabs Dubbing; The video file is sent to the **ElevenLabs Dubbing API**, which initiates audio dubbing in the specified target language. |
| Wait | Wait | Pauses based on expected dubbing duration | Elevenlabs Dubbing | Get Dubbing ID | ## STEP 3- Download video; After the wait, it checks the dubbing status using the dubbing_id and retrieves the final dubbed audio file. |
| Get Dubbing ID | HTTP Request | Fetches dubbing job status/details | Wait | Download final Video | ## STEP 3- Download video; After the wait, it checks the dubbing status using the dubbing_id and retrieves the final dubbed audio file. |
| Download final Video | HTTP Request | Downloads dubbed output from ElevenLabs | Get Dubbing ID | Upload file; Upload Video to Postiz; Youtube Upload-Post | ## STEP 4- Upload to Google Drive & Youtube; After the wait, it checks the dubbing status using the dubbing_id and retrieves the final dubbed audio file.; Remember to set the video description parameters in these nodes, choose one of the two platforms for uploading the YouTube video, and disconnect the other from the workflow. |
| Upload file | Google Drive | Stores dubbed file in Drive folder | Download final Video | — | ## STEP 4- Upload to Google Drive & Youtube; After the wait, it checks the dubbing status using the dubbing_id and retrieves the final dubbed audio file.; Remember to set the video description parameters in these nodes, choose one of the two platforms for uploading the YouTube video, and disconnect the other from the workflow. |
| Upload Video to Postiz | HTTP Request | Uploads media asset to Postiz | Download final Video | Youtube Postiz | ## STEP 4- Upload to Google Drive & Youtube; After the wait, it checks the dubbing status using the dubbing_id and retrieves the final dubbed audio file.; Remember to set the video description parameters in these nodes, choose one of the two platforms for uploading the YouTube video, and disconnect the other from the workflow. |
| Youtube Postiz | Postiz | Schedules/posts using Postiz integration | Upload Video to Postiz | — | ## STEP 4- Upload to Google Drive & Youtube; After the wait, it checks the dubbing status using the dubbing_id and retrieves the final dubbed audio file.; Remember to set the video description parameters in these nodes, choose one of the two platforms for uploading the YouTube video, and disconnect the other from the workflow. |
| Youtube Upload-Post | HTTP Request | Uploads to YouTube via Upload-Post.com | Download final Video | — | ## STEP 4- Upload to Google Drive & Youtube; After the wait, it checks the dubbing status using the dubbing_id and retrieves the final dubbed audio file.; Remember to set the video description parameters in these nodes, choose one of the two platforms for uploading the YouTube video, and disconnect the other from the workflow. |
| Sticky Note | Sticky Note | Comment/branding/instructions | — | — | ## Automatically Translate and Upload YouTube Videos Using ElevenLabs AI Dubbing; Includes links: https://iframe.mediadelivery.net/play/580928/c445daec-e3fe-4019-b035-58ac3bf386dd and https://iframe.mediadelivery.net/play/580928/2179db44-e7e2-43e6-82a1-13b12e18ba8b ; Credentials links: https://try.elevenlabs.io/ahkbf00hocnu ; https://affiliate.postiz.com/n3witalia ; https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app |
| Sticky Note1 | Sticky Note | Comment | — | — | ## STEP 1 - Set params (content as above) |
| Sticky Note2 | Sticky Note | Comment | — | — | ## STEP 2- Elevenlabs Dubbing (content as above) |
| Sticky Note3 | Sticky Note | Comment | — | — | ## STEP 3- Download video (content as above) |
| Sticky Note4 | Sticky Note | Comment | — | — | ## STEP 4- Upload to Google Drive & Youtube (content as above) |
| Sticky Note9 | Sticky Note | Comment/branding | — | — | ## MY NEW YOUTUBE CHANNEL; Link: https://youtube.com/@n3witalia ; Image link: https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Translate Videos Using ElevenLabs Dubbing** (or your preferred name).

2. **Add Manual Trigger**
   - Node: **Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`

3. **Add Set node for inputs**
   - Node: **Set**
   - Name: `Set params`
   - Add fields:
     - `video_url` (String): set to a **direct video file URL** (replace placeholder `VIDEO_URL`)
     - `target_audio` (String): set to a language code, e.g. `es`, `en`, `fr`
   - Connect: **Manual Trigger → Set params**

4. **Add HTTP Request to download the video**
   - Node: **HTTP Request**
   - Name: `Get video`
   - Set **URL** to: `{{$json.video_url}}`
   - Configure the node so the response is available as **binary** (commonly stored as `data`).  
     (Exact option labels depend on n8n version; ensure downstream can reference binary field `data`.)
   - Connect: **Set params → Get video**

5. **Add HTTP Request to create dubbing job (ElevenLabs)**
   - Node: **HTTP Request**
   - Name: `Elevenlabs Dubbing`
   - Method: **POST**
   - URL: `https://api.elevenlabs.io/v1/dubbing`
   - Authentication: **Predefined Credential Type → ElevenLabs API**
     - Create credential: **ElevenLabs API**
     - Provide your ElevenLabs API key as required by n8n’s credential type
   - Content Type: **Multipart/Form-Data**
   - Body parameters:
     - `file` → **Binary** from input field name: `data`
     - `target_lang` → value: `{{$json.target_audio}}`
   - Connect: **Get video → Elevenlabs Dubbing**

6. **Add Wait node**
   - Node: **Wait**
   - Name: `Wait`
   - Amount (seconds): `{{$json.expected_duration_sec + 120}}`
   - Connect: **Elevenlabs Dubbing → Wait**

7. **Add HTTP Request to fetch dubbing job status**
   - Node: **HTTP Request**
   - Name: `Get Dubbing ID`
   - Method: GET
   - URL: `https://api.elevenlabs.io/v1/dubbing/{{ $('Elevenlabs Dubbing').item.json.dubbing_id }}`
   - Authentication: **ElevenLabs API credential**
   - Connect: **Wait → Get Dubbing ID**

8. **Add HTTP Request to download final dubbed output**
   - Node: **HTTP Request**
   - Name: `Download final Video`
   - Method: GET
   - URL: `https://api.elevenlabs.io/v1/dubbing/{{$json.dubbing_id}}/audio/{{$json.target_languages[0]}}`
   - Authentication: **ElevenLabs API credential**
   - Configure response to be saved as **binary** in field `data` (so uploads can use it).
   - Connect: **Get Dubbing ID → Download final Video**

9. **Add Google Drive upload**
   - Node: **Google Drive**
   - Name: `Upload file`
   - Operation: upload file (standard upload)
   - File name: `{{$binary.data.fileName}}`
   - Drive: **My Drive**
   - Folder: select your target folder (the original uses folder ID `1tkCr7xdraoZwsHqeLm7FZ4aRWY94oLbZ`)
   - Credentials: **Google Drive OAuth2**
     - Create OAuth2 credential and authorize the Google account
   - Connect: **Download final Video → Upload file**

10. **(Optional branch A) Upload to Postiz via HTTP**
    - Node: **HTTP Request**
    - Name: `Upload Video to Postiz`
    - Method: POST
    - URL: `https://api.postiz.com/public/v1/upload`
    - Content Type: multipart/form-data
    - Authentication: **HTTP Header Auth**
      - Create credential “Postiz” containing required header(s) (typically an API key/bearer token header as required by Postiz)
    - Body:
      - `file` → Binary from `data`
    - Connect: **Download final Video → Upload Video to Postiz**

11. **(Optional branch A ادامه) Create/schedule the Postiz post**
    - Node: **Postiz** (community node: `n8n-nodes-postiz.postiz`)
    - Name: `Youtube Postiz`
    - Credentials: **Postiz API**
    - Date: `{{$now.format('yyyy-LL-dd')}}T{{$now.format('HH:ii:ss')}}`
    - Post content configuration:
      - Use upload response `id` and `path` in the image/media item mapping:
        - `id`: `{{$json.id}}`
        - `path`: `{{$json.path}}`
      - Replace placeholders:
        - content text `YOUR_CONTENT`
        - `integrationId` = your real Postiz integration id
    - Connect: **Upload Video to Postiz → Youtube Postiz**

12. **(Optional branch B) Upload to YouTube via Upload-Post.com**
    - Node: **HTTP Request**
    - Name: `Youtube Upload-Post`
    - Method: POST
    - URL: `https://api.upload-post.com/api/upload`
    - Content Type: multipart/form-data
    - Authentication: **HTTP Header Auth**
      - Create credential “Upload-post.com API” with required API headers
    - Body parameters:
      - `title` = set your title (replace `YOUR_CONTENT`)
      - `user` = your username (replace `YOUR_USERNAME`)
      - `platform[]` = `youtube`
      - `video` = Binary from `data`
    - Connect: **Download final Video → Youtube Upload-Post**

13. **Decide which YouTube publishing path to use**
    - Keep **Postiz** path *or* **Upload-Post.com** path.
    - If using only one, disconnect the other branch to avoid double-posting.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically Translate and Upload YouTube Videos Using ElevenLabs AI Dubbing (example source and result) | Source (Italian Short): https://iframe.mediadelivery.net/play/580928/c445daec-e3fe-4019-b035-58ac3bf386dd |
| Example result (English version) | https://iframe.mediadelivery.net/play/580928/2179db44-e7e2-43e6-82a1-13b12e18ba8b |
| ElevenLabs API link (credential setup reference) | https://try.elevenlabs.io/ahkbf00hocnu |
| Postiz link | https://affiliate.postiz.com/n3witalia |
| Upload-Post.com link | https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app |
| Subscribe link (author’s channel) | https://youtube.com/@n3witalia |
| Cover image used in sticky note | https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg |