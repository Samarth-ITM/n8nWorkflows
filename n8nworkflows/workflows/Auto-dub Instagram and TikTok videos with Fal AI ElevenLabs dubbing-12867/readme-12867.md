Auto-dub Instagram and TikTok videos with Fal AI ElevenLabs dubbing

https://n8nworkflows.xyz/workflows/auto-dub-instagram-and-tiktok-videos-with-fal-ai-elevenlabs-dubbing-12867


# Auto-dub Instagram and TikTok videos with Fal AI ElevenLabs dubbing

## 1. Workflow Overview

**Purpose:**  
This workflow automates **video dubbing** for short-form content (Instagram/TikTok) by sending a source video URL to **Fal AI’s ElevenLabs Dubbing endpoint**, polling until the job is complete, downloading the dubbed video file, then uploading it to:
- **TikTok** via **Upload-Post** (HTTP API)
- **Instagram** via **Postiz** (upload file + create/publish post)

**Primary use cases:**
- Localizing viral videos into another language quickly (e.g., EN → ES)
- Scaling multi-language social publishing for creators/marketing teams
- Automating repetitive “dub → download → upload” steps with minimal manual input

### 1.1 Input Reception & Parameter Setup
Manual trigger + a Set node where the user specifies `video_url` and `target_audio`.

### 1.2 Dubbing Job Submission (Fal AI / ElevenLabs)
Sends `video_url` and `target_lang` to Fal queue endpoint and receives a `request_id`.

### 1.3 Job Polling & Retrieval
Waits, checks the job status repeatedly until `COMPLETED`, then retrieves metadata containing the final dubbed video URL.

### 1.4 Download & Publishing (TikTok + Instagram)
Downloads the dubbed video as binary, then uploads it to TikTok (Upload-Post) and uploads to Postiz, then creates an Instagram post via Postiz node.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Parameter Setup

**Overview:**  
Starts the workflow manually and defines the two key inputs: the source video URL and the target dubbing language code.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Set params

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`manualTrigger`) — entry point for manual runs.
- **Configuration choices:** No parameters; starts execution on demand in n8n UI.
- **Inputs / outputs:** No input. Output flows to **Set params**.
- **Edge cases:** None (only runs when manually executed).
- **Version notes:** TypeVersion 1.

#### Node: Set params
- **Type / role:** Set node (`set`) — creates/overrides fields used later.
- **Configuration choices:**
  - Sets `video_url` (string) default value: `VIDEO_URL` (placeholder to replace).
  - Sets `target_audio` (string) default value: `es` (Spanish).
- **Key variables created:**
  - `$json.video_url`
  - `$json.target_audio`
- **Inputs / outputs:** Input from Manual Trigger. Output to **Video Dubbing**.
- **Potential failures / edge cases:**
  - If `video_url` is not a direct downloadable video URL, Fal may fail to fetch it.
  - Invalid language code may cause dubbing API validation errors.
- **Version notes:** TypeVersion 3.4.

**Sticky note(s) covering this block (content preserved):**
- “## STEP 1 - Set params … define `video_url` … `target_audio` …”

---

### Block 2 — Dubbing Job Submission (Fal AI / ElevenLabs)

**Overview:**  
Submits the dubbing request to Fal’s queue endpoint for ElevenLabs dubbing and receives a job `request_id`.

**Nodes involved:**
- Video Dubbing

#### Node: Video Dubbing
- **Type / role:** HTTP Request (`httpRequest`) — starts Fal dubbing job.
- **Configuration choices (interpreted):**
  - **Method:** POST
  - **URL:** `https://queue.fal.run/fal-ai/elevenlabs/dubbing`
  - **Body:** JSON (explicit)
    - `video_url`: `{{ $json.video_url }}`
    - `target_lang`: `{{ $json.target_audio }}`
  - **Headers:** `Content-Type: application/json`
  - **Auth:** Generic credential type → **HTTP Header Auth** credential named **“Fal.run API”**
- **Key expressions / variables:**
  - Uses `$json.video_url` and `$json.target_audio` from **Set params**.
- **Inputs / outputs:** Input from **Set params**. Output to **Wait 30 sec.**
- **Expected output shape:**
  - Must include something like `request_id` (later referenced by polling nodes).
- **Potential failures / edge cases:**
  - 401/403 if Fal API key header is missing/invalid.
  - 400 if body schema is wrong or language code unsupported.
  - Fal cannot fetch the video (private URL, geo-blocked, expired signed URL).
  - Large video may exceed service limits/time.
- **Version notes:** TypeVersion 4.3.

**Sticky note(s) covering this block:**
- “## STEP 2- AL AI Elevenlabs Dubbing … initiates audio dubbing …”

---

### Block 3 — Job Polling & Retrieval

**Overview:**  
Waits 30 seconds, checks job status, loops until `COMPLETED`, then fetches the final response containing the dubbed video URL and downloads the file.

**Nodes involved:**
- Wait 30 sec.
- Get status
- Completed?
- Get final video url
- Get final video file

#### Node: Wait 30 sec.
- **Type / role:** Wait (`wait`) — introduces delay between status checks.
- **Configuration choices:** Wait amount = 30 seconds.
- **Inputs / outputs:** Input from **Video Dubbing** or from **Completed?** “false” branch loop. Output to **Get status**.
- **Potential failures / edge cases:**
  - Too short wait may cause more polling loops (rate limits).
  - Too long wait slows throughput for short videos.
- **Version notes:** TypeVersion 1.1.

#### Node: Get status
- **Type / role:** HTTP Request — checks Fal job status endpoint.
- **Configuration choices:**
  - **URL (expression):**  
    `https://queue.fal.run/fal-ai/elevenlabs/requests/{{ $('Video Dubbing').item.json.request_id }}/status `
    - Note: there is a **trailing space** after `status ` in the configured URL string. Many servers tolerate it, but it can cause subtle request issues depending on HTTP client normalization.
  - **Auth:** HTTP Header Auth credential **“Fal.run API”**
  - **Send Query:** enabled; adds a query parameter `Content-Type=application/json` (unusual placement—typically this should be a header).
- **Key expressions / variables:**
  - References `$('Video Dubbing').item.json.request_id` (pulls the `request_id` from the earlier node item).
- **Inputs / outputs:** Input from **Wait 30 sec.** Output to **Completed?**
- **Potential failures / edge cases:**
  - If `Video Dubbing` returns no `request_id`, expression fails.
  - 401/403 auth issues.
  - 404 if request_id invalid/expired.
  - Rate limiting if polled too frequently.
- **Version notes:** TypeVersion 4.2.

#### Node: Completed?
- **Type / role:** If (`if`) — routes based on completion status.
- **Configuration choices:**
  - Condition: `{{ $json.status }}` equals `COMPLETED`
  - True branch → **Get final video url**
  - False branch → **Wait 30 sec.** (poll loop)
- **Inputs / outputs:** Input from **Get status**. Two outputs:
  - **True:** to **Get final video url**
  - **False:** to **Wait 30 sec.**
- **Potential failures / edge cases:**
  - If Fal uses different terminal states (e.g., `FAILED`, `CANCELED`), this workflow will loop forever unless externally stopped.
  - If `$json.status` missing, condition evaluates false and loops.
- **Version notes:** TypeVersion 2.2.

#### Node: Get final video url
- **Type / role:** HTTP Request — retrieves final Fal request payload (expects final video metadata).
- **Configuration choices:**
  - **URL (expression):** `https://queue.fal.run/fal-ai/elevenlabs/requests/{{ $json.request_id }}`
  - **Headers:** `Content-Type: application/json`
  - **Auth:** HTTP Header Auth credential **“Fal.run API”**
- **Inputs / outputs:** Input from **Completed?** (true). Output to **Get final video file**.
- **Potential failures / edge cases:**
  - Assumes the incoming JSON includes `request_id` at the root. Depending on Fal status response, it may not; you might need to reference the ID from the original node (as in Get status).
  - If response structure changes, downstream `{{ $json.video.url }}` may break.
- **Version notes:** TypeVersion 4.2.

#### Node: Get final video file
- **Type / role:** HTTP Request — downloads the dubbed video file.
- **Configuration choices:**
  - **URL (expression):** `{{ $json.video.url }}`
  - No authentication configured (assumes final URL is publicly downloadable).
- **Inputs / outputs:** Input from **Get final video url**. Output goes to **Upload to TikTok** and **Upload to Postiz** in parallel.
- **Potential failures / edge cases:**
  - If `video.url` missing, expression fails.
  - If the URL is signed and expires quickly, download may fail.
  - Depending on n8n settings, you may need to ensure the node outputs **binary** data; the next nodes expect binary field `data`.
- **Version notes:** TypeVersion 4.3.

**Sticky note(s) covering this block:**
- “## STEP 3- Get and Download video … checks the status and retrieves the final dubbed file.”

---

### Block 4 — Upload & Social Publishing

**Overview:**  
Takes the downloaded dubbed video binary and uploads it to TikTok via Upload-Post, and uploads it to Postiz, then triggers an Instagram post creation/publish through Postiz.

**Nodes involved:**
- Upload to TikTok
- Upload to Postiz
- Upload to Instagram

#### Node: Upload to TikTok
- **Type / role:** HTTP Request — uploads video to Upload-Post for TikTok publishing.
- **Configuration choices:**
  - **URL:** `https://api.upload-post.com/api/upload`
  - **Method:** POST
  - **Content-Type:** multipart/form-data
  - **Body fields:**
    - `title` = `XXX` (placeholder)
    - `user` = `YOUR_USERNAME` (placeholder)
    - `platform[]` = `tiktok`
    - `video` = binary file from input field name `data`
  - **Auth:** HTTP Header Auth credential named **“Youtube Transcript Extractor API 1”** (name suggests it may be reused/misnamed; functionally it must contain Upload-Post auth header(s)).
- **Inputs / outputs:** Input from **Get final video file**. No downstream connections configured.
- **Potential failures / edge cases:**
  - If **Get final video file** does not output binary in field `data`, the form upload will fail.
  - Wrong credentials/header for Upload-Post causes 401/403.
  - Missing required fields (`user`, `title`) or wrong `platform[]` value.
  - Upload size limits / timeouts for large videos.
- **Version notes:** TypeVersion 4.2.

#### Node: Upload to Postiz
- **Type / role:** HTTP Request — uploads the dubbed video to Postiz media storage.
- **Configuration choices:**
  - **URL:** `https://api.postiz.com/public/v1/upload`
  - **Method:** POST
  - **Content-Type:** multipart/form-data
  - **Body field:** `file` = binary from input field `data`
  - **Auth:** HTTP Header Auth credential named **“Postiz”**
- **Inputs / outputs:** Input from **Get final video file**. Output to **Upload to Instagram** (Postiz node).
- **Expected output:** Must include media identifiers like `id` and `path` used by the next node.
- **Potential failures / edge cases:**
  - Binary missing/not named `data`.
  - Auth failure if Postiz token invalid.
  - API returns different response fields than expected (`id`, `path`).
- **Version notes:** TypeVersion 4.2.

#### Node: Upload to Instagram
- **Type / role:** Postiz node (`n8n-nodes-postiz.postiz`) — creates/schedules/publishes an Instagram post in Postiz.
- **Configuration choices (interpreted):**
  - **Date:** set to “now” using expressions:  
    `{{$now.format('yyyy-LL-dd')}}T{{$now.format('HH:ii:ss')}}`
  - **Posts payload:**
    - content includes `imageItem` with:
      - `id` = `{{ $json.id }}`
      - `path` = `{{ $json.path }}`
    - `content` = `XXX` (placeholder caption/content)
    - settings include:
      - `__type` = `instagram`
      - `post_type` = `post`
    - `integrationId` = `XXX` (placeholder; must be your Postiz channel/integration id)
  - **shortLink:** enabled
  - **Credentials:** Postiz API credential **“Postiz account”**
- **Inputs / outputs:** Input from **Upload to Postiz** (expects `id`/`path`). No downstream connections configured.
- **Potential failures / edge cases:**
  - If Postiz upload response doesn’t match expected fields, post creation fails.
  - Wrong `integrationId` (channel id) prevents publishing.
  - Using `imageItem` for a video may be mismatched depending on Postiz API schema; confirm Postiz expects a video/media item structure for Instagram video posts/reels.
  - Time format: uses `HH:ii:ss` (in many date libs `mm` is minutes; here `ii` is n8n Luxon-style formatting for minutes is typically `mm`. If `ii` is not valid, time formatting may be wrong).
- **Version notes:** TypeVersion 1.

**Sticky note(s) covering this block:**
- “## STEP 4 - Upload video Upload video to TikTok and Instagram”
- “Set YOUR_USERNAME and TITLE for [Upload-Post]((https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app))”
- “Set Channel_ID and TITLE for [Postiz](https://affiliate.postiz.com/n3witalia) (TikTok, Instagram, Facebook, X, Youtube)”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Set params |  |
| Set params | Set | Define `video_url` + `target_audio` inputs | When clicking ‘Execute workflow’ | Video Dubbing | ## STEP 1 - Set params… define `video_url` and `target_audio`. |
| Video Dubbing | HTTP Request | Submit dubbing job to Fal (ElevenLabs) | Set params | Wait 30 sec. | ## STEP 2- AL AI Elevenlabs Dubbing… initiates audio dubbing. |
| Wait 30 sec. | Wait | Delay between polling attempts | Video Dubbing; Completed? (false) | Get status | ## STEP 3- Get and Download video… checks status and retrieves final file. |
| Get status | HTTP Request | Poll Fal job status | Wait 30 sec. | Completed? | ## STEP 3- Get and Download video… checks status and retrieves final file. |
| Completed? | If | Branch on `status == COMPLETED` | Get status | Get final video url (true); Wait 30 sec. (false) | ## STEP 3- Get and Download video… checks status and retrieves final file. |
| Get final video url | HTTP Request | Fetch final Fal request payload (video URL) | Completed? (true) | Get final video file | ## STEP 3- Get and Download video… checks status and retrieves final file. |
| Get final video file | HTTP Request | Download dubbed video binary | Get final video url | Upload to TikTok; Upload to Postiz | ## STEP 3- Get and Download video… checks status and retrieves final file. |
| Upload to TikTok | HTTP Request | Upload/publish to TikTok via Upload-Post | Get final video file | — | Set YOUR_USERNAME and TITLE for [Upload-Post]((https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app)) |
| Upload to Postiz | HTTP Request | Upload media to Postiz | Get final video file | Upload to Instagram | Set Channel_ID and TITLE for [Postiz](https://affiliate.postiz.com/n3witalia) (TikTok, Instagram, Facebook, X, Youtube) |
| Upload to Instagram | Postiz (postiz) | Create/publish Instagram post in Postiz | Upload to Postiz | — | ## STEP 4 - Upload video Upload video to TikTok and Instagram |
| Sticky Note1 | Sticky Note | Comment | — | — |  |
| Sticky Note2 | Sticky Note | Comment | — | — |  |
| Sticky Note3 | Sticky Note | Comment | — | — |  |
| Sticky Note4 | Sticky Note | Comment | — | — |  |
| Sticky Note5 | Sticky Note | Comment | — | — |  |
| Sticky Note8 | Sticky Note | Comment | — | — |  |
| Sticky Note9 | Sticky Note | Comment | — | — |  |
| Sticky Note | Sticky Note | Comment | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   `Instagram & TikTok Video Dubbing Fal AI`

2. **Add Manual Trigger**
   - Node type: **Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`

3. **Add Set node for inputs**
   - Node type: **Set**
   - Name: `Set params`
   - Add fields:
     - `video_url` (String) → set a placeholder like `VIDEO_URL` (replace when running)
     - `target_audio` (String) → e.g. `es`
   - Connect: **Manual Trigger → Set params**

4. **Add Fal dubbing request node**
   - Node type: **HTTP Request**
   - Name: `Video Dubbing`
   - Method: **POST**
   - URL: `https://queue.fal.run/fal-ai/elevenlabs/dubbing`
   - Authentication: **Generic Credential Type → HTTP Header Auth**
     - Create credential “Fal.run API” that sends Fal auth header(s) required by fal.run (commonly `Authorization: Key …` or `Authorization: Bearer …` depending on Fal configuration).
   - Send Headers: enabled
     - Header: `Content-Type: application/json`
   - Send Body: enabled → Body Content Type: **JSON**
   - JSON body:
     - `video_url` = expression `{{ $json.video_url }}`
     - `target_lang` = expression `{{ $json.target_audio }}`
   - Connect: **Set params → Video Dubbing**

5. **Add Wait node**
   - Node type: **Wait**
   - Name: `Wait 30 sec.`
   - Amount: `30` seconds
   - Connect: **Video Dubbing → Wait 30 sec.**

6. **Add status polling HTTP node**
   - Node type: **HTTP Request**
   - Name: `Get status`
   - Method: GET (default)
   - URL (expression):  
     `https://queue.fal.run/fal-ai/elevenlabs/requests/{{ $('Video Dubbing').item.json.request_id }}/status`
   - Authentication: same **Fal.run API** credential (HTTP Header Auth)
   - (Recommended fix) Put `Content-Type: application/json` as a **header**, not a query parameter.
   - Connect: **Wait 30 sec. → Get status**

7. **Add IF node for completion**
   - Node type: **IF**
   - Name: `Completed?`
   - Condition: String → `{{ $json.status }}` equals `COMPLETED`
   - Connect: **Get status → Completed?**
   - Connect **False** output back to **Wait 30 sec.** (creates polling loop)

8. **Add final payload retrieval node**
   - Node type: **HTTP Request**
   - Name: `Get final video url`
   - URL (expression):  
     `https://queue.fal.run/fal-ai/elevenlabs/requests/{{ $json.request_id }}`
   - Authentication: **Fal.run API** credential
   - Header: `Content-Type: application/json`
   - Connect: **Completed? (true) → Get final video url**
   - (If needed) Adjust expression to use the original request id:  
     `{{ $('Video Dubbing').item.json.request_id }}`

9. **Add video download node**
   - Node type: **HTTP Request**
   - Name: `Get final video file`
   - URL (expression): `{{ $json.video.url }}`
   - Ensure it downloads as **file/binary** (in n8n this may require enabling “Response Format: File” depending on node UI/version) so downstream nodes can read binary field `data`.
   - Connect: **Get final video url → Get final video file**

10. **Add TikTok upload (Upload-Post)**
   - Node type: **HTTP Request**
   - Name: `Upload to TikTok`
   - Method: POST
   - URL: `https://api.upload-post.com/api/upload`
   - Authentication: **HTTP Header Auth** (create an Upload-Post credential; the JSON shows a credential name that appears unrelated—ensure it contains the correct Upload-Post API header token)
   - Body Content Type: **Multipart/Form-Data**
   - Fields:
     - `title` = your title (replace `XXX`)
     - `user` = `YOUR_USERNAME`
     - `platform[]` = `tiktok`
     - `video` = **binary** from input field name `data`
   - Connect: **Get final video file → Upload to TikTok**

11. **Add Postiz upload node**
   - Node type: **HTTP Request**
   - Name: `Upload to Postiz`
   - Method: POST
   - URL: `https://api.postiz.com/public/v1/upload`
   - Authentication: **HTTP Header Auth** credential “Postiz” (must include Postiz public API token header expected by Postiz)
   - Body: Multipart/Form-Data
     - `file` = binary from input field name `data`
   - Connect: **Get final video file → Upload to Postiz**

12. **Add Postiz publish node (Instagram)**
   - Node type: **Postiz**
   - Name: `Upload to Instagram`
   - Credentials: **Postiz account** (Postiz API credential type used by the Postiz node)
   - Date expression: `{{$now.format('yyyy-LL-dd')}}T{{$now.format('HH:ii:ss')}}`
   - Configure the post payload:
     - Use returned `id` and `path` from **Upload to Postiz**
     - Set `integrationId` to your Instagram channel/integration in Postiz
     - Set caption/content (replace `XXX`)
     - Settings: `__type=instagram`, `post_type=post`
   - Connect: **Upload to Postiz → Upload to Instagram**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## Auto-Dub Viral Videos for Instagram & TikTok with ElevenLabs using Fal AI …” (full workflow description + setup steps) | Internal workflow sticky note (overview/instructions) |
| “Set YOUR_USERNAME and TITLE for Upload-Post” | https://www.upload-post.com/?linkId=lp_144414&sourceId=n3witalia&tenantId=upload-post-app |
| “Set Channel_ID and TITLE for Postiz (TikTok, Instagram, Facebook, X, Youtube)” | https://affiliate.postiz.com/n3witalia |
| “MY NEW YOUTUBE CHANNEL … Subscribe … FREE templates for n8n” + image link | https://youtube.com/@n3witalia (image: https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg) |

**Disclaimer (provided by you, preserved):**  
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.