Generate daily AI reels from Google Drive images with GPT‑5.1, Wavespeed and Submagic

https://n8nworkflows.xyz/workflows/generate-daily-ai-reels-from-google-drive-images-with-gpt-5-1--wavespeed-and-submagic-13110


# Generate daily AI reels from Google Drive images with GPT‑5.1, Wavespeed and Submagic

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate daily AI reels from Google Drive images with GPT‑5.1, Wavespeed and Submagic  
**Workflow name (internal):** 2026 - Image to Video Social Media Reel Autopost

**Purpose:**  
Automatically generate a short-form vertical marketing reel every day from a random image stored in a Google Drive folder. The workflow generates a cinematic image-to-video prompt with GPT‑5.1, sends the image + prompt to Wavespeed (Veo 3.1 Fast) to create an 8-second video with audio, applies captions/overlays via Submagic, requests human approval via Gmail, and then posts the approved reel to Instagram and TikTok using Blotato.

**Main logical blocks**
1. **1.1 Scheduled run & image selection (Google Drive)**
2. **1.2 Asset hosting for downstream tools (Blotato media upload)**
3. **1.3 Prompt generation (GPT‑5.1)**
4. **1.4 Video generation & polling (Wavespeed)**
5. **1.5 Captions/text overlay & polling (Submagic)**
6. **1.6 Human approval gate (Gmail send-and-wait)**
7. **1.7 Social publishing (Blotato → Instagram + TikTok)**
8. **1.8 Retry loops (status polling + non-approval path)**

---

## 2. Block-by-Block Analysis

### 2.1 Scheduled run & image selection (Google Drive)

**Overview:**  
Runs daily at a fixed hour, lists files in a configured Google Drive folder, and randomly selects one file to use as the reel’s base image.

**Nodes involved:**
- Schedule Trigger
- Search files and folders
- Randomizer

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` — starts workflow on a time schedule.
- **Configuration (interpreted):** Triggers at **09:00** (daily by implication of schedule interval usage).
- **Key variables/expressions:** none.
- **Connections:**
  - **Out →** Search files and folders
- **Version notes:** node typeVersion **1.3**.
- **Potential failures / edge cases:**
  - If n8n timezone differs from expected, the 09:00 run may occur at an unexpected local time (instance timezone matters).

#### Node: Search files and folders
- **Type / role:** `Google Drive` (resource: fileFolder) — lists files/folders under a given folder.
- **Configuration (interpreted):**
  - **Resource:** File/Folder search/list
  - **Folder:** provided via a *URL-style* folder reference: `https://drive.google.com/.../folders/REPLACE_WITH_YOUR_FOLDER_ID`
  - **Return all:** enabled (returns entire listing)
- **Key variables/expressions:** folderId is set via URL mode; you must replace the placeholder.
- **Connections:**
  - **In ←** Schedule Trigger
  - **Out →** Randomizer
- **Version notes:** typeVersion **3** (Google Drive node).
- **Potential failures / edge cases:**
  - OAuth not configured / expired token.
  - Folder ID invalid, or Drive permissions prevent listing.
  - Folder contains **0 files** → Randomizer will fail (`files.length` = 0).

#### Node: Randomizer
- **Type / role:** `Code` — selects one random item from the list.
- **Configuration (interpreted):**
  - Reads all incoming items: `const files = $input.all();`
  - Picks random index; returns a single-item array containing that file.
- **Key variables/expressions:** `$input.all()`
- **Connections:**
  - **In ←** Search files and folders
  - **Out →** Upload media
- **Version notes:** typeVersion **2** (Code node).
- **Potential failures / edge cases:**
  - If no files were returned: `Math.random() * 0` leads to `NaN` index and undefined selection.
  - The “file” could be a folder depending on Drive listing behavior; downstream expects an image file.

---

### 2.2 Asset hosting for downstream tools (Blotato media upload)

**Overview:**  
Uploads the selected Google Drive image to Blotato (or a Blotato-managed URL) so it can be used by external services that need a direct media URL.

**Nodes involved:**
- Upload media

#### Node: Upload media
- **Type / role:** `Blotato` (`@blotato/n8n-nodes-blotato.blotato`) — uploads media by URL.
- **Configuration (interpreted):**
  - **Resource:** media upload
  - **mediaUrl:** constructed from the Google Drive file id:  
    `https://drive.google.com/file/d/<FILE_ID>/view`
- **Key variables/expressions:**
  - `{{ $('Search files and folders').item.json.id }}` used to build the Drive URL.
- **Connections:**
  - **In ←** Randomizer
  - **Out →** Prompt Generator
- **Version notes:** typeVersion **2** (community/vendor node).
- **Potential failures / edge cases:**
  - The Drive “/view” URL is not always a direct-download URL; Blotato must be able to fetch it. If the file isn’t public or accessible, upload can fail.
  - Blotato credential missing or invalid.

---

### 2.3 Prompt generation (GPT‑5.1)

**Overview:**  
Generates a single “ready-to-send” cinematic image-to-video prompt, including a short spoken line and voice characteristics, optimized for an 8-second marketing reel.

**Nodes involved:**
- Prompt Generator

#### Node: Prompt Generator
- **Type / role:** `OpenAI (LangChain)` (`@n8n/n8n-nodes-langchain.openAi`) — chat/prompt completion to produce the final video prompt.
- **Configuration (interpreted):**
  - **Model:** `gpt-5.1`
  - **System message:** detailed rules for cinematic prompt engineering, realism, camera, lighting, mood, and a single short spoken line; forbids captions/on-screen text instructions; output must be only the final prompt.
  - **User content:** `Image Name/Description: <random file name>`
- **Key variables/expressions:**
  - `{{ $('Randomizer').item.json.name }}`
  - Output later referenced as: `{{ $json.output[0].content[0].text }}`
- **Connections:**
  - **In ←** Upload media
  - **Out →** Wavespeed Post Request (To generate reels)
- **Version notes:** typeVersion **2.1**.
- **Potential failures / edge cases:**
  - Missing OpenAI credentials / model access not enabled.
  - Output shape assumptions: the downstream node expects `output[0].content[0].text`; if the node output format differs (or changes by version), the expression can break.
  - The filename alone may be insufficient; prompts may be generic unless filenames are descriptive.

---

### 2.4 Video generation & polling (Wavespeed)

**Overview:**  
Submits the image + GPT prompt to Wavespeed’s image-to-video endpoint (Veo 3.1 Fast) for an 8-second vertical video with audio, waits, then polls until completion.

**Nodes involved:**
- Wavespeed Post Request (To generate reels)
- Wait 30 Secs
- GET Result from Wavespeed
- If

#### Node: Wavespeed Post Request (To generate reels)
- **Type / role:** `HTTP Request` — starts a Wavespeed “prediction” job.
- **Configuration (interpreted):**
  - **POST** `https://api.wavespeed.ai/api/v3/google/veo3.1-fast/image-to-video`
  - Sends body parameters:
    - `aspect_ratio = 9:16`
    - `duration = 8`
    - `generate_audio = true`
    - `resolution = 1080p`
    - `image = <uploaded-media-url-from-Blotato>`
    - `prompt = <GPT output text>`
  - **Authentication:** Generic Credential Type → **HTTP Header Auth** (API key likely in headers).
- **Key variables/expressions:**
  - Image: `{{ $('Upload media').item.json.url }}`
  - Prompt: `{{ $json.output[0].content[0].text }}`
- **Connections:**
  - **In ←** Prompt Generator
  - **Out →** Wait 30 Secs
- **Version notes:** typeVersion **4.3**.
- **Potential failures / edge cases:**
  - Incorrect header auth setup (missing/invalid API key).
  - Prompt too long / rejected by provider.
  - Wavespeed returns non-200 or unexpected schema; downstream assumes `data.id`.

#### Node: Wait 30 Secs
- **Type / role:** `Wait` — delay before polling.
- **Configuration:** waits **30 seconds**.
- **Connections:**
  - **In ←** Wavespeed Post Request
  - **Out →** GET Result from Wavespeed
- **Version notes:** typeVersion **1.1**.
- **Potential failures / edge cases:**
  - Too short for generation → extra polling cycles; too long → slower throughput.

#### Node: GET Result from Wavespeed
- **Type / role:** `HTTP Request` — checks job result.
- **Configuration (interpreted):**
  - **GET** `https://api.wavespeed.ai/api/v3/predictions/<id>/result`
  - Uses same **HTTP Header Auth**.
- **Key variables/expressions:**
  - `{{ $json.data.id }}` from the POST response (must exist in the current item).
- **Connections:**
  - **In ←** Wait 30 Secs
  - **Out →** If
- **Version notes:** typeVersion **4.3**.
- **Potential failures / edge cases:**
  - If previous response doesn’t include `data.id`, expression fails.
  - Wavespeed may return statuses like `queued`, `processing`, `failed`.

#### Node: If
- **Type / role:** `IF` — branching on Wavespeed status.
- **Configuration (interpreted):**
  - Condition: `$json.data.status` **equals** `"completed"`
  - **True branch:** proceed to Submagic
  - **False branch:** loop back to wait and poll again
- **Connections:**
  - **In ←** GET Result from Wavespeed
  - **True →** Submagic Post Request
  - **False →** Wait 30 Secs
- **Version notes:** typeVersion **2.3**.
- **Potential failures / edge cases:**
  - If status is `failed`, workflow will currently keep looping (no explicit failure handling/backoff/cap).
  - Missing `data.status` breaks strict validation.

---

### 2.5 Captions/text overlay & polling (Submagic)

**Overview:**  
Creates a Submagic project from the generated video URL, waits, then polls Submagic until the captioned/overlaid result is completed.

**Nodes involved:**
- Submagic Post Request
- 30 Secs
- Submagic Get Result
- If1

#### Node: Submagic Post Request
- **Type / role:** `HTTP Request` — starts a Submagic project.
- **Configuration (interpreted):**
  - **POST** `https://api.submagic.co/v1/projects`
  - Body:
    - `title = "Daily Short"`
    - `language = "en"`
    - `videoUrl = $json.data.outputs[0]` (from Wavespeed result)
    - `templateName = "Hormozi 2"`
  - Auth: **HTTP Header Auth**
- **Key variables/expressions:**
  - `{{ $json.data.outputs[0] }}`
- **Connections:**
  - **In ←** If (true path)
  - **Out →** 30 Secs
- **Version notes:** typeVersion **4.3**.
- **Potential failures / edge cases:**
  - Wavespeed output array missing/empty → invalid videoUrl.
  - Submagic template name invalid or not available to the account.
  - Auth/header issues.

#### Node: 30 Secs
- **Type / role:** `Wait` — delay before polling Submagic.
- **Configuration:** waits **30 seconds**.
- **Connections:**
  - **In ←** Submagic Post Request OR If1 (false path loop)
  - **Out →** Submagic Get Result
- **Version notes:** typeVersion **1.1**.

#### Node: Submagic Get Result
- **Type / role:** `HTTP Request` — fetch project status/output.
- **Configuration (interpreted):**
  - **GET** `https://api.submagic.co/v1/projects/<projectId>`
  - Auth: **HTTP Header Auth**
- **Key variables/expressions:**
  - `{{ $json.id }}` from Submagic Post Request response.
- **Connections:**
  - **In ←** 30 Secs
  - **Out →** If1
- **Version notes:** typeVersion **4.3**.
- **Potential failures / edge cases:**
  - If the current item no longer contains `id` (lost context), polling will fail.
  - Submagic may return error/failed statuses not handled.

#### Node: If1
- **Type / role:** `IF` — branching on Submagic completion.
- **Configuration (interpreted):**
  - Condition: `$json.status` equals `"completed"`
  - **True:** proceed to email approval
  - **False:** wait then poll again
- **Connections:**
  - **In ←** Submagic Get Result
  - **True →** Send message and wait for response
  - **False →** 30 Secs
- **Version notes:** typeVersion **2.3**.
- **Potential failures / edge cases:**
  - If status is `failed`, it loops indefinitely (no cap/backoff).
  - Assumes `status` is a string at the root.

---

### 2.6 Human approval gate (Gmail send-and-wait)

**Overview:**  
Sends an approval email containing the Submagic preview URL and suggested caption/description, then pauses execution until the recipient approves or rejects.

**Nodes involved:**
- Send message and wait for response
- If2

#### Node: Send message and wait for response
- **Type / role:** `Gmail` — sends an email and waits for an approval response (human-in-the-loop).
- **Configuration (interpreted):**
  - Operation: **sendAndWait**
  - To: `REPLACE_WITH_YOUR_EMAIL`
  - Subject: `Daily Short Approval`
  - Message body includes:
    - Reel preview URL: `{{ $json.previewUrl }}`
    - Caption: `{{ $json.description }}`
  - Approval options: **double** (requires stronger confirmation in n8n’s approval mechanism).
- **Connections:**
  - **In ←** If1 (true)
  - **Out →** If2
- **Version notes:** typeVersion **2.2**.
- **Potential failures / edge cases:**
  - Gmail OAuth missing, insufficient scopes, or Google security restrictions.
  - Email lands in spam; approvals never received → execution stays waiting.
  - The node expects n8n’s approval mechanism to be configured/usable in your environment.

#### Node: If2
- **Type / role:** `IF` — checks whether approval was granted.
- **Configuration (interpreted):**
  - Condition: `$json.data.approved` equals `true`
  - **True:** proceed to posting
  - **False:** loop back to image search on next cycle (in practice: immediately continues to “Search files and folders” branch)
- **Connections:**
  - **In ←** Send message and wait for response
  - **True →** Upload to Blotato
  - **False →** Search files and folders
- **Version notes:** typeVersion **2.3**.
- **Potential failures / edge cases:**
  - If the Gmail approval payload differs, `data.approved` may not exist.
  - On rejection, this workflow *immediately* restarts the pipeline from Google Drive search (not “next day” unless you stop it). This can cause repeated attempts in a single run.

---

### 2.7 Social publishing (Blotato → Instagram + TikTok)

**Overview:**  
Uploads the finalized Submagic video to Blotato, then posts it to Instagram and TikTok with the Submagic-generated description as caption text.

**Nodes involved:**
- Upload to Blotato
- Post to Instagram
- Post to Tik Tok

#### Node: Upload to Blotato
- **Type / role:** `Blotato` — uploads finalized video to get a Blotato media URL for posting.
- **Configuration (interpreted):**
  - Resource: media
  - mediaUrl: `{{ $('If1').item.json.directUrl }}`
    - This assumes Submagic completed payload contains a `directUrl` field (often a direct downloadable video).
- **Connections:**
  - **In ←** If2 (true)
  - **Out →** Post to Instagram and Post to Tik Tok (fan-out)
- **Version notes:** typeVersion **2**.
- **Potential failures / edge cases:**
  - If Submagic response doesn’t include `directUrl`, upload fails.
  - Large video sizes, temporary URLs expiring, or Blotato fetch failures.

#### Node: Post to Instagram
- **Type / role:** `Blotato` — publishes to Instagram.
- **Configuration (interpreted):**
  - accountId: placeholder `REPLACE_WITH_ACCOUNT_ID` (must be replaced with your Blotato-connected IG account)
  - Text: `{{ $('Submagic Get Result').item.json.description }}`
  - Media URL(s): `{{ $json.url }}` (from Upload to Blotato output)
- **Connections:**
  - **In ←** Upload to Blotato
  - **Out →** (none)
- **Version notes:** typeVersion **2**.
- **Potential failures / edge cases:**
  - Account not authorized for publishing, or platform permissions missing.
  - Incorrect media type/format for IG Reels.
  - Rate limits / API errors.

#### Node: Post to Tik Tok
- **Type / role:** `Blotato` — publishes to TikTok.
- **Configuration (interpreted):**
  - platform: `tiktok`
  - accountId: placeholder `REPLACE_WITH_ACCOUNT_ID`
  - Text: `{{ $('Submagic Get Result').item.json.description }}`
  - Note: unlike Instagram node, no explicit media URL parameter is set here; it may rely on Blotato node defaults or be incomplete depending on Blotato node requirements.
- **Connections:**
  - **In ←** Upload to Blotato
  - **Out →** (none)
- **Version notes:** typeVersion **2**.
- **Potential failures / edge cases:**
  - If TikTok posting requires explicit media URLs in this node, it will fail or post without video.
  - Account permissions, region restrictions, or upload constraints.

---

### 2.8 Annotation / Documentation (Sticky Notes)

**Overview:**  
Sticky notes document the intent of node groups and provide an external video link and setup/troubleshooting notes.

**Nodes involved:**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

#### Node: Sticky Note
- **Type / role:** Sticky note annotation.
- **Content:** “Pick Base Image File”
- **Applies to:** the image selection area (Schedule Trigger → Drive search → Randomizer).

#### Node: Sticky Note1
- **Type / role:** Sticky note annotation.
- **Content:** “Prompt and Post to Video Generation Model”
- **Applies to:** prompt generation + Wavespeed submission.

#### Node: Sticky Note2
- **Type / role:** Sticky note annotation.
- **Content:** “Post to Text Overlay Tool (Submagic)”
- **Applies to:** Submagic submission/polling (left section).

#### Node: Sticky Note3
- **Type / role:** Sticky note annotation.
- **Content:** “Post to Text Overlay Tool (Submagic)”
- **Applies to:** Submagic submission/polling + approval path (right section).

#### Node: Sticky Note4
- **Type / role:** Large sticky note annotation containing full description, requirements, and troubleshooting.
- **Notable link:** https://www.youtube.com/watch?v=jPOYxQF25ws

#### Node: Sticky Note5
- **Type / role:** Sticky note annotation.
- **Content:** `- [ ] @[youtube](jPOYxQF25ws)` (shorthand reference to the same YouTube video)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Daily scheduled start | — | Search files and folders | Pick Base Image File |
| Search files and folders | n8n-nodes-base.googleDrive | List files from Drive folder | Schedule Trigger; If2 (false) | Randomizer | Pick Base Image File |
| Randomizer | n8n-nodes-base.code | Pick one random file | Search files and folders | Upload media | Pick Base Image File |
| Upload media | @blotato/n8n-nodes-blotato.blotato | Upload chosen image to get accessible URL | Randomizer | Prompt Generator | Pick Base Image File |
| Prompt Generator | @n8n/n8n-nodes-langchain.openAi | Generate cinematic image-to-video prompt | Upload media | Wavespeed Post Request (To generate reels) | Prompt and Post to Video Generation Model |
| Wavespeed Post Request (To generate reels) | n8n-nodes-base.httpRequest | Start Wavespeed image-to-video job | Prompt Generator | Wait 30 Secs | Prompt and Post to Video Generation Model |
| Wait 30 Secs | n8n-nodes-base.wait | Delay before polling Wavespeed | Wavespeed Post Request (To generate reels); If (false) | GET Result from Wavespeed | Prompt and Post to Video Generation Model |
| GET Result from Wavespeed | n8n-nodes-base.httpRequest | Poll Wavespeed job status/result | Wait 30 Secs | If | Prompt and Post to Video Generation Model |
| If | n8n-nodes-base.if | Branch on Wavespeed completion | GET Result from Wavespeed | Submagic Post Request (true); Wait 30 Secs (false) | Prompt and Post to Video Generation Model |
| Submagic Post Request | n8n-nodes-base.httpRequest | Create Submagic caption/overlay project | If (true) | 30 Secs | Post to Text Overlay Tool (Submagic) |
| 30 Secs | n8n-nodes-base.wait | Delay before polling Submagic | Submagic Post Request; If1 (false) | Submagic Get Result | Post to Text Overlay Tool (Submagic) |
| Submagic Get Result | n8n-nodes-base.httpRequest | Poll Submagic project status/result | 30 Secs | If1 | Post to Text Overlay Tool (Submagic) |
| If1 | n8n-nodes-base.if | Branch on Submagic completion | Submagic Get Result | Send message and wait for response (true); 30 Secs (false) | Post to Text Overlay Tool (Submagic) |
| Send message and wait for response | n8n-nodes-base.gmail | Email approval gate (send & wait) | If1 (true) | If2 | Post to Text Overlay Tool (Submagic) |
| If2 | n8n-nodes-base.if | Branch on approval boolean | Send message and wait for response | Upload to Blotato (true); Search files and folders (false) | Post to Text Overlay Tool (Submagic) |
| Upload to Blotato | @blotato/n8n-nodes-blotato.blotato | Upload final video for posting | If2 (true) | Post to Instagram; Post to Tik Tok | Post to Text Overlay Tool (Submagic) |
| Post to Instagram | @blotato/n8n-nodes-blotato.blotato | Publish approved reel to Instagram | Upload to Blotato | — | Post to Text Overlay Tool (Submagic) |
| Post to Tik Tok | @blotato/n8n-nodes-blotato.blotato | Publish approved reel to TikTok | Upload to Blotato | — | Post to Text Overlay Tool (Submagic) |
| Sticky Note | n8n-nodes-base.stickyNote | Annotation | — | — | Pick Base Image File |
| Sticky Note1 | n8n-nodes-base.stickyNote | Annotation | — | — | Prompt and Post to Video Generation Model |
| Sticky Note2 | n8n-nodes-base.stickyNote | Annotation | — | — | Post to Text Overlay Tool (Submagic) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Annotation | — | — | Post to Text Overlay Tool (Submagic) |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation & requirements | — | — | Image to Video Social Media Reel Generator + Autopost… (includes link: https://www.youtube.com/watch?v=jPOYxQF25ws) |
| Sticky Note5 | n8n-nodes-base.stickyNote | Link shorthand | — | — | - [ ] @[youtube](jPOYxQF25ws) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create “Schedule Trigger”**
   - Node type: **Schedule Trigger**
   - Configure to run **daily at 09:00** (adjust timezone/instance settings as needed).
   - Connect to **Google Drive** node.

2. **Create “Search files and folders” (Google Drive)**
   - Node type: **Google Drive**
   - Resource/operation: list/search **files & folders** in a folder
   - Folder: set your target folder (replace placeholder with your folder ID).
   - Enable **Return All**.
   - Credentials: configure **Google Drive OAuth2**.
   - Connect to **Randomizer**.

3. **Create “Randomizer” (Code)**
   - Node type: **Code**
   - Paste logic to pick one random input item:
     - Read all items: `$input.all()`
     - Select a random index
     - Return one item only
   - Connect to **Upload media**.

4. **Create “Upload media” (Blotato)**
   - Node type: **Blotato** (media upload)
   - Resource: **media**
   - `mediaUrl`: build from the Drive file id (e.g., `https://drive.google.com/file/d/<id>/view`)
   - Credentials: configure **Blotato** credentials.
   - Connect to **Prompt Generator**.

5. **Create “Prompt Generator” (OpenAI / LangChain)**
   - Node type: **OpenAI (LangChain)**
   - Model: **gpt-5.1**
   - Add a **system** instruction enforcing:
     - cinematic camera + lighting + realism
     - output only the final prompt
     - include one short spoken line + voice characteristics
     - no on-screen text instructions
   - Add a user message that includes the selected file name (e.g., `Image Name/Description: {{$('Randomizer').item.json.name}}`)
   - Credentials: configure **OpenAI API key**.
   - Connect to **Wavespeed Post Request**.

6. **Create “Wavespeed Post Request (To generate reels)” (HTTP Request)**
   - Node type: **HTTP Request**
   - Method: **POST**
   - URL: `https://api.wavespeed.ai/api/v3/google/veo3.1-fast/image-to-video`
   - Auth: **HTTP Header Auth** (via Generic Credentials); add Wavespeed API key header per Wavespeed docs.
   - Body parameters:
     - aspect_ratio: `9:16`
     - duration: `8`
     - generate_audio: `true`
     - resolution: `1080p`
     - image: URL from **Upload media** output
     - prompt: text from **Prompt Generator** output
   - Connect to **Wait 30 Secs**.

7. **Create polling loop for Wavespeed**
   1) Add **Wait** node “Wait 30 Secs” (30 seconds) → connect from POST node.  
   2) Add **HTTP Request** node “GET Result from Wavespeed”
      - Method: **GET**
      - URL: `https://api.wavespeed.ai/api/v3/predictions/{{$json.data.id}}/result`
      - Same header auth
   3) Add **IF** node “If”
      - Condition: `{{$json.data.status}}` equals `completed`
      - True → proceed
      - False → connect back to **Wait 30 Secs**

8. **Create “Submagic Post Request” (HTTP Request)**
   - Method: **POST**
   - URL: `https://api.submagic.co/v1/projects`
   - Auth: **HTTP Header Auth** (Submagic API key)
   - Body:
     - title: `Daily Short`
     - language: `en`
     - videoUrl: `{{$json.data.outputs[0]}}` (from Wavespeed completed result)
     - templateName: `Hormozi 2`
   - Connect to **Wait** node “30 Secs”.

9. **Create polling loop for Submagic**
   1) Add **Wait** node “30 Secs” (30 seconds).
   2) Add **HTTP Request** “Submagic Get Result”
      - Method: **GET**
      - URL: `https://api.submagic.co/v1/projects/{{$json.id}}`
   3) Add **IF** “If1”
      - Condition: `{{$json.status}}` equals `completed`
      - True → Gmail approval
      - False → connect back to **30 Secs**

10. **Create “Send message and wait for response” (Gmail)**
    - Node type: **Gmail**
    - Operation: **Send and wait**
    - To: your email address
    - Subject: `Daily Short Approval`
    - Body: include `{{$json.previewUrl}}` and `{{$json.description}}`
    - Approval type: **double**
    - Credentials: **Gmail OAuth2**
    - Connect to **If2**.

11. **Create “If2” approval branch**
    - Node type: **IF**
    - Condition: `{{$json.data.approved}}` equals `true`
    - True → **Upload to Blotato**
    - False → connect to **Search files and folders** (retry path)

12. **Create “Upload to Blotato” (final video upload)**
    - Node type: **Blotato** (media upload)
    - mediaUrl: use Submagic’s final downloadable URL (in this workflow: `{{$('If1').item.json.directUrl}}`)
    - Connect output to both posting nodes (fan-out).

13. **Create “Post to Instagram” (Blotato)**
    - Node type: **Blotato** posting
    - Set **accountId** to your Instagram-connected Blotato account.
    - Caption text: `{{$('Submagic Get Result').item.json.description}}`
    - Media URLs: use the uploaded media URL from **Upload to Blotato** (in this workflow: `{{$json.url}}`).
    - No further connections needed.

14. **Create “Post to Tik Tok” (Blotato)**
    - Node type: **Blotato** posting
    - Platform: `tiktok`
    - accountId: your TikTok-connected Blotato account
    - Caption text: same Submagic description expression
    - Ensure the node is configured to include the uploaded media (if required by your Blotato node version).

15. **(Optional but recommended) Add safeguards**
    - Add maximum retry counters for both polling loops (Wavespeed/Submagic).
    - Handle explicit `failed` statuses by notifying you and stopping.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Watch Step By Step Guide on How to Build | https://www.youtube.com/watch?v=jPOYxQF25ws |
| Workflow concept: Google Drive → AI Video Generation → Captions → Approval → Instagram & TikTok | Sticky Note4 (high-level description) |
| Setup requirements: Google Drive OAuth, OpenAI API key, Wavespeed API key, Submagic API key, Gmail OAuth, Blotato account | Sticky Note4 |
| Troubleshooting hints: no images found, video stuck generating (increase wait), approval email issues, posting permissions | Sticky Note4 |
| `- [ ] @[youtube](jPOYxQF25ws)` | Sticky Note5 (shorthand YouTube reference) |