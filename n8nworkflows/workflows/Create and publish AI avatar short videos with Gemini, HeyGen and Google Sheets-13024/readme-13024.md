Create and publish AI avatar short videos with Gemini, HeyGen and Google Sheets

https://n8nworkflows.xyz/workflows/create-and-publish-ai-avatar-short-videos-with-gemini--heygen-and-google-sheets-13024


# Create and publish AI avatar short videos with Gemini, HeyGen and Google Sheets

## 1. Workflow Overview

**Workflow name (in n8n):** Automated AI Video Avatar  
**Provided title:** Create and publish AI avatar short videos with Gemini, HeyGen and Google Sheets

**Purpose:**  
This workflow runs end-to-end on a schedule to (1) scrape “viral” TikTok videos from a target profile, (2) store them as an idea queue in Google Sheets, (3) analyze one queued video with Google Gemini, (4) generate a short avatar-ready script (JSON-structured), (5) generate an AI avatar video via HeyGen, then (6) auto-publish the final video to TikTok and Facebook via Blotato, and (7) mark the row as processed in Google Sheets.

**Typical use cases:**
- Automated short-form content repurposing pipeline (“viral to avatar content”).
- Scheduled content calendar driven by Google Sheets statuses (Pending/Done).
- Multi-platform publishing automation.

### 1.1 Viral Idea Engine (collection + queueing)
- Scrapes TikTok videos for a given profile via Apify.
- Normalizes fields (caption, metrics, URLs) and appends them to a Google Sheet with `Status = Pending`.

### 1.2 AI Content Brain (analysis + script generation)
- Pulls the next `Pending` row from the sheet.
- Uses Gemini Video Analysis on the video URL to extract “what works”.
- Uses a LangChain Agent powered by Google Gemini to generate a TikTok script + caption in strict JSON format.

### 1.3 AI Avatar Video Factory (HeyGen generation + polling)
- Sends script text to HeyGen to generate a vertical avatar video.
- Polls HeyGen until the video is no longer `processing`.
- Retrieves the final `video_url`.

### 1.4 Auto Publish & Tracking
- Publishes the rendered video to TikTok and Facebook via Blotato.
- Updates the original Google Sheet row status to `Done`.

---

## 2. Block-by-Block Analysis

### Block 1 — Viral Idea Engine (TikTok scrape → sheet queue)

**Overview:**  
Every 6 hours, the workflow scrapes popular videos from a specific TikTok profile using Apify. It maps the results into a standardized schema and appends them to a Google Sheet as `Pending` items.

**Nodes involved:**
- Cron – Fetch Viral Ideas
- Get Viral Video Dataset
- Save Viral Ideas to Google Sheet
- Save Viral Ideas to Google Sheet1

#### Node: Cron – Fetch Viral Ideas
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) to start ingestion periodically.
- **Config:** Runs every **6 hours**.
- **Outputs:** One execution trigger item.
- **Failure modes:** None typical (unless workflow disabled or n8n scheduler halted).

#### Node: Get Viral Video Dataset
- **Type / role:** Apify node (`@apify/n8n-nodes-apify.apify`) to run an actor and return its dataset.
- **Operation:** “Run actor and get dataset”.
- **Actor:** `clockworks/tiktok-scraper` (via actor ID `GdWCkxBtKWOsKjdch`).
- **Key config choices:**
  - Scrapes **profile** `@sabrina_ramonov`
  - Section: `videos`
  - Sorting: `popular`
  - Results per page: `10`
  - Downloads videos: `shouldDownloadVideos: true`
  - Proxy: `proxyCountryCode: VN`
  - Stores videos in KV store: `"videoKvStoreIdOrName": "my-tiktok-ASMR"`
- **Authentication:** Apify OAuth2.
- **Output:** One item per scraped video (including `text`, `createTimeISO`, counts, `mediaUrls`, `webVideoUrl`, etc.).
- **Failure modes / edge cases:**
  - Apify quota/credit exhaustion.
  - Actor run failure (blocked scraping, proxy issues).
  - Dataset empty → later nodes append nothing.
  - Video download disabled/broken → missing `mediaUrls`.

#### Node: Save Viral Ideas to Google Sheet
- **Type / role:** Set node (`n8n-nodes-base.set`) to remap Apify dataset fields to a sheet schema.
- **Key mappings (expressions):**
  - `Caption = {{$json.text}}`
  - `Date Post = {{$json.createTimeISO.toDateTime().format('dd/LL/yyyy')}}`
  - `Username = {{$json.authorMeta.name}}`
  - `Link user = {{$json.authorMeta.profileUrl}}`
  - `Link video = {{$json.webVideoUrl}}`
  - `VideoURL = {{$json.mediaUrls}}` (array)
  - `Likes = {{$json.diggCount}}`, `Shares`, `View`, `Save`
  - `Hasgtag = {{$json.hashtags}}` (array)
- **Output:** Same items, but with clean top-level fields for Google Sheets insertion.
- **Failure modes / edge cases:**
  - `createTimeISO` missing or not parseable → date formatting fails.
  - `mediaUrls` empty → downstream video analysis may fail.

#### Node: Save Viral Ideas to Google Sheet1
- **Type / role:** Google Sheets append (`n8n-nodes-base.googleSheets`) to create queue entries.
- **Operation:** Append row.
- **Spreadsheet:** Document ID `11mgpwg5pU1IjZugU32oKqzHf21tlU57UbhLhERAoxFE` (cached name “Sabrina viral content”).
- **Sheet/tab:** `gid=0` (cached name “Trang tính1”).
- **Writes (notable):**
  - `Status = "Pending"`
  - `VideoURL = {{$json.VideoURL[0]}}` (first URL only)
  - Metrics fields (`Likes`, `Shares`, `View`, `Save`)
  - Caption, hashtags, username, link fields, date post
- **Important nuance:** Although the earlier Set node stores `VideoURL` as an array, this append writes only the first URL.
- **Failure modes / edge cases:**
  - Google OAuth/credentials missing or expired.
  - Sheet schema mismatch (column renamed, removed).
  - Duplicates: no dedup logic; the same videos can be appended repeatedly across runs.

---

### Block 2 — AI Content Brain (load pending row → analyze → script JSON)

**Overview:**  
Every 6 hours, the workflow pulls the first row with `Status = Pending`, analyzes the linked video with Gemini, then asks a Gemini-powered agent to produce a short script and caption in strict JSON.

**Nodes involved:**
- Cron – Process Viral Row
- Load Viral Idea from Sheet
- Analyze Viral Video
- Google Gemini Chat Model
- Parse Script Structure
- AI Agent – Script & Avatar Director
- Finalize Avatar Script

#### Node: Cron – Process Viral Row
- **Type / role:** Schedule Trigger.
- **Config:** Every **6 hours** (independent from the ingestion cron; they may overlap).
- **Output:** One trigger item.

#### Node: Load Viral Idea from Sheet
- **Type / role:** Google Sheets “read with filter”.
- **Config choices:**
  - Filter: `Status` equals `Pending`
  - `returnFirstMatch: true` (processes a single row per run)
- **Output:** The first matching row, including `row_number` used later for updating.
- **Failure modes / edge cases:**
  - No matching rows → node may output no items; downstream nodes won’t run.
  - If `Status` values differ (e.g., “pending”), filter won’t match.

#### Node: Analyze Viral Video
- **Type / role:** Gemini Video Analysis node (`@n8n/n8n-nodes-langchain.googleGemini`), resource `video`, operation `analyze`.
- **Input config:**
  - Prompt: “Analyze and extract the main content of the video.”
  - Model: `models/gemini-3-flash-preview`
  - `videoUrls = {{$json.VideoURL}}`
- **Important nuance:** From the sheet, `VideoURL` is stored as a string (first URL). This node accepts `videoUrls` and may accept a string or array depending on node implementation; if it strictly expects an array, this becomes a runtime issue.
- **Output:** A Gemini response with `content.parts[0].text` (observed in pinned sample data).
- **Failure modes / edge cases:**
  - Video URL inaccessible (expired Apify KV link, permissions, 404).
  - Model limitations/quotas.
  - Large video size/timeouts.
  - If `content.parts` missing, downstream expression `.parts[0].text` fails.

#### Node: Google Gemini Chat Model
- **Type / role:** LangChain chat model provider (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) used by the agent.
- **Model:** `models/gemini-3-flash-preview`
- **Connections:** Provides `ai_languageModel` input to the agent node.
- **Failure modes:** API key/auth issues, quota errors.

#### Node: Parse Script Structure
- **Type / role:** Structured output parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) enforcing JSON schema.
- **Schema example:**
  ```json
  { "title": "string", "script": "string", "caption": "string" }
  ```
- **Connections:** Feeds the agent’s `ai_outputParser`.
- **Failure modes / edge cases:**
  - If the agent outputs non-JSON or violates schema, parsing fails and stops the workflow.

#### Node: AI Agent – Script & Avatar Director
- **Type / role:** LangChain agent (`@n8n/n8n-nodes-langchain.agent`) generating final script/caption.
- **Prompt strategy:**
  - System message instructs the model to:
    - Write a TikTok script **under 500 characters**
    - Produce a caption **under 120 characters**
    - Replicate structure/energy from the analyzed content
    - Output **ONLY** strict JSON with keys `title`, `script`, `caption`
  - Embeds analyzed text:
    - `{{ $json.content.parts[0].text.replace(/\n+/g, ' ') }}`
- **Dependencies:**
  - Receives main input from **Analyze Viral Video**.
  - Receives language model from **Google Gemini Chat Model**.
  - Receives output parser from **Parse Script Structure**.
- **Output:** Parsed JSON available under `$json.output.title`, `$json.output.script`, `$json.output.caption`.
- **Failure modes / edge cases:**
  - If `content.parts[0].text` missing → expression error.
  - Model may exceed character limits unless enforced well (parser won’t catch character count unless it breaks JSON).
  - If the output parser fails, nothing proceeds.

#### Node: Finalize Avatar Script
- **Type / role:** Set node to rename/flatten agent output into fields used downstream.
- **Mappings:**
  - `Scripts = {{$json.output.script}}`
  - `Title = {{$json.output.title}}`
  - `Caption = {{$json.output.caption}}`
- **Failure modes:** Missing `$json.output.*` (parser failure upstream).

---

### Block 3 — AI Avatar Video Factory (HeyGen generate → poll → retrieve)

**Overview:**  
Transforms the generated script into a HeyGen avatar video (vertical 720×1280). Then it loops polling the HeyGen status endpoint until rendering completes and returns the final `video_url`.

**Nodes involved:**
- Create AI Avatar Video
- Wait for Video Rendering
- Get Rendered Video
- Check Video Status

#### Node: Create AI Avatar Video
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) to HeyGen `POST /v2/video/generate`.
- **Request body (key parts):**
  - `avatar_id: "52774bb2c9ae4b5aa1bf0746784cfc0d"`
  - `voice_id: "beaa78fa0aa04f0f9cd2a130d1f70231"`
  - `input_text: "{{ $json.Scripts }}"`
  - `dimension: { width: 720, height: 1280 }`
- **Auth:** Generic “HTTP Header Auth” (HeyGen API key expected in headers via credential).
- **Headers:** `accept: application/json` (note: no explicit `Content-Type`; n8n usually sets for JSON body).
- **Output:** `data.video_id` used later.
- **Failure modes / edge cases:**
  - Missing/invalid HeyGen API key.
  - HeyGen rejects long text, unsupported characters, or policy blocks.
  - Rate limits or transient 5xx.

#### Node: Wait for Video Rendering
- **Type / role:** Wait node (`n8n-nodes-base.wait`) used as a delay between polls.
- **Config:** No explicit wait duration shown (defaults vary by node behavior/version). It is used in a loop with status checks.
- **Connections:** Receives from Create, then flows to Get Rendered Video.
- **Failure modes / edge cases:**
  - If configured to “wait for webhook” unintentionally, it may stall.
  - Infinite loop risk if status never changes (no max retries configured in workflow).

#### Node: Get Rendered Video
- **Type / role:** HTTP Request to HeyGen `GET /v1/video_status.get`.
- **Query:**
  - `video_id = {{ $('Create AI Avatar Video').item.json.data.video_id }}`
- **Auth:** Same header-auth credential.
- **Output:** `data.status` and when complete, `data.video_url`.
- **Failure modes / edge cases:**
  - If `video_id` missing, request fails.
  - Endpoint version mismatch: generation uses `/v2/...`, status uses `/v1/...` (works only if HeyGen supports this combination).

#### Node: Check Video Status
- **Type / role:** IF node (`n8n-nodes-base.if`) to decide loop vs proceed.
- **Condition:** If `{{$json.data.status}} == "processing"`
  - **True path:** goes back to **Wait for Video Rendering** (poll loop)
  - **False path:** goes to **Publish Video to Tiktok**
- **Failure modes / edge cases:**
  - If HeyGen returns statuses like `queued`, `failed`, `completed`, etc., only `processing` is handled explicitly.
  - If status becomes `failed`, it goes to publishing anyway (because it’s “not processing”), likely breaking later steps due to missing `video_url`.

---

### Block 4 — Auto Publish & Tracking (Blotato publish → sheet update)

**Overview:**  
Once HeyGen returns a finished video URL, the workflow publishes it to TikTok and Facebook via Blotato, then updates the original sheet row to `Done`.

**Nodes involved:**
- Publish Video to Tiktok
- Publish Video to Facebook
- Update row Viral Content

#### Node: Publish Video to Tiktok
- **Type / role:** Blotato node (`@blotato/n8n-nodes-blotato.blotato`) to publish content.
- **Platform:** TikTok
- **Account:** `kytagiang` (account ID `23867`)
- **Text:** 
  - `{{$('Finalize Avatar Script').item.json.Title}}\n{{$('Finalize Avatar Script').item.json.Caption}}`
- **Media URL:** `{{$json.data.video_url}}` (expects input from **Get Rendered Video** on the “not processing” branch)
- **Credentials:** `Blotato GiangxAI`
- **Outputs:** Publishes, then triggers both:
  - Update row Viral Content
  - Publish Video to Facebook
- **Failure modes / edge cases:**
  - TikTok publishing restrictions (account not authorized, video length/format issues, watermark/policy).
  - If `video_url` missing (HeyGen failed) → publish call fails.
  - Rate limits on Blotato API.

#### Node: Publish Video to Facebook
- **Type / role:** Blotato publish to Facebook.
- **Platform:** Facebook
- **Account:** `Giang VT` (account ID `15884`)
- **Page:** `688227101036478`
- **Text:** same Title + Caption.
- **Media URL:** `{{$('Get Rendered Video').item.json.data.video_url}}` (explicitly references Get Rendered Video, not current input).
- **Credentials:** `Blotato GiangxAI`
- **Failure modes / edge cases:**
  - Facebook page permissions/token expiration.
  - Video URL inaccessible to Facebook (must be publicly downloadable).
  - If the node runs in parallel with the update and the execution context differs, expression referencing another node’s item must match the correct item index (usually OK here because single-item flow).

#### Node: Update row Viral Content
- **Type / role:** Google Sheets update to mark processed row.
- **Operation:** Update row (matching column: `row_number`)
- **Update values:**
  - `Status = "Done"`
  - `row_number = {{ $('Load Viral Idea from Sheet').item.json.row_number }}`
- **Sheet:** Same spreadsheet and tab as the queue.
- **Failure modes / edge cases:**
  - If `row_number` missing (sheet read failed), update fails.
  - Concurrency: if two workflow runs pick the same pending row (due to parallel schedules), row updates may conflict.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Cron – Fetch Viral Ideas | Schedule Trigger | Triggers viral idea ingestion every 6h | — | Get Viral Video Dataset | ## Viral Idea Engine<br>Collect viral short-form video ideas on a schedule and store them in Google Sheets as the input queue for the workflow. |
| Get Viral Video Dataset | Apify | Scrape TikTok profile popular videos and return dataset | Cron – Fetch Viral Ideas | Save Viral Ideas to Google Sheet | ## Viral Idea Engine<br>Collect viral short-form video ideas on a schedule and store them in Google Sheets as the input queue for the workflow. |
| Save Viral Ideas to Google Sheet | Set | Normalize scraped fields to sheet schema | Get Viral Video Dataset | Save Viral Ideas to Google Sheet1 | ## Viral Idea Engine<br>Collect viral short-form video ideas on a schedule and store them in Google Sheets as the input queue for the workflow. |
| Save Viral Ideas to Google Sheet1 | Google Sheets | Append new “Pending” rows to queue sheet | Save Viral Ideas to Google Sheet | — | ## Viral Idea Engine<br>Collect viral short-form video ideas on a schedule and store them in Google Sheets as the input queue for the workflow. |
| Cron – Process Viral Row | Schedule Trigger | Triggers processing of one pending row every 6h | — | Load Viral Idea from Sheet | ## AI Content Brain<br>Analyze viral videos and generate short, avatar-ready scripts automatically using AI. |
| Load Viral Idea from Sheet | Google Sheets | Fetch first row where Status=Pending | Cron – Process Viral Row | Analyze Viral Video | ## AI Content Brain<br>Analyze viral videos and generate short, avatar-ready scripts automatically using AI. |
| Analyze Viral Video | Google Gemini (Video Analyze) | Extract main content/structure from the video URL | Load Viral Idea from Sheet | AI Agent – Script & Avatar Director | ## AI Content Brain<br>Analyze viral videos and generate short, avatar-ready scripts automatically using AI. |
| Google Gemini Chat Model | Gemini Chat Model (LangChain) | LLM provider for agent | — | AI Agent – Script & Avatar Director | ## AI Content Brain<br>Analyze viral videos and generate short, avatar-ready scripts automatically using AI. |
| Parse Script Structure | Output Parser (Structured) | Enforce JSON output schema (title/script/caption) | — | AI Agent – Script & Avatar Director (parser input) | ## AI Content Brain<br>Analyze viral videos and generate short, avatar-ready scripts automatically using AI. |
| AI Agent – Script & Avatar Director | LangChain Agent | Generates avatar-ready TikTok script + caption as JSON | Analyze Viral Video (+ model + parser) | Finalize Avatar Script | ## AI Content Brain<br>Analyze viral videos and generate short, avatar-ready scripts automatically using AI. |
| Finalize Avatar Script | Set | Flatten `output.*` into `Title/Scripts/Caption` | AI Agent – Script & Avatar Director | Create AI Avatar Video | ## AI Content Brain<br>Analyze viral videos and generate short, avatar-ready scripts automatically using AI. |
| Create AI Avatar Video | HTTP Request | Call HeyGen to generate avatar video | Finalize Avatar Script | Wait for Video Rendering | ##AI Avatar Video Factory<br>Create AI avatar videos from scripts, wait for rendering, and retrieve the final video from HeyGen. |
| Wait for Video Rendering | Wait | Delay/poll loop control while HeyGen renders | Create AI Avatar Video / Check Video Status (true) | Get Rendered Video | ##AI Avatar Video Factory<br>Create AI avatar videos from scripts, wait for rendering, and retrieve the final video from HeyGen. |
| Get Rendered Video | HTTP Request | Poll HeyGen status endpoint and get `video_url` | Wait for Video Rendering | Check Video Status | ##AI Avatar Video Factory<br>Create AI avatar videos from scripts, wait for rendering, and retrieve the final video from HeyGen. |
| Check Video Status | IF | If status=processing loop; else publish | Get Rendered Video | (true) Wait for Video Rendering; (false) Publish Video to Tiktok | ##AI Avatar Video Factory<br>Create AI avatar videos from scripts, wait for rendering, and retrieve the final video from HeyGen. |
| Publish Video to Tiktok | Blotato | Publish final video to TikTok | Check Video Status (false) | Update row Viral Content; Publish Video to Facebook | ## Auto Publish & Tracking<br>Publish AI avatar videos to social platforms and update Google Sheets with status and results. |
| Publish Video to Facebook | Blotato | Publish final video to Facebook page | Publish Video to Tiktok | — | ## Auto Publish & Tracking<br>Publish AI avatar videos to social platforms and update Google Sheets with status and results. |
| Update row Viral Content | Google Sheets | Mark processed row as Done | Publish Video to Tiktok | — | ## Auto Publish & Tracking<br>Publish AI avatar videos to social platforms and update Google Sheets with status and results. |
| Sticky Note | Sticky Note | Comment block | — | — | ## Viral Idea Engine<br>Collect viral short-form video ideas on a schedule and store them in Google Sheets as the input queue for the workflow. |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## AI Content Brain<br>Analyze viral videos and generate short, avatar-ready scripts automatically using AI. |
| Sticky Note2 | Sticky Note | Comment block | — | — | ##AI Avatar Video Factory<br>Create AI avatar videos from scripts, wait for rendering, and retrieve the final video from HeyGen. |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## Auto Publish & Tracking<br>Publish AI avatar videos to social platforms and update Google Sheets with status and results. |
| Sticky Note4 | Sticky Note | Setup/credits note | — | — | ## I🛠️ Workflow Setup Guide<br><br>Author: [GiangxAI](https://www.youtube.com/@giangxai.official)<br><br>## How it works<br>- Viral video ideas are collected and stored in Google Sheets  <br>- Each idea is analyzed and converted into an AI avatar-ready script  <br>- HeyGen generates an AI avatar video automatically  <br>- The video is published to social platforms  <br>- Google Sheets is updated to track progress and results  <br><br>The entire workflow runs on a schedule and operates end-to-end without manual editing.<br>---<br>## Setup guide<br>- Connect Google Sheets and prepare a sheet for viral ideas and status tracking  <br>- Add credentials for your AI model (e.g. Gemini) and HeyGen  <br>- Configure avatar settings such as voice and language  <br>- Set publishing credentials for your target platforms  <br>- Adjust the schedule to control how often videos are generated and published  <br><br>Setup is quick and requires no coding knowledge. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (inactive by default while configuring).

2. **Add Schedule Trigger** node named **“Cron – Fetch Viral Ideas”**  
   - Interval: every **6 hours**.

3. **Add Apify** node named **“Get Viral Video Dataset”**  
   - Operation: **Run actor and get dataset**  
   - Actor: **TikTok Scraper (clockworks/tiktok-scraper)**  
   - Input JSON (custom body) with:
     - `profiles: ["@sabrina_ramonov"]`
     - `profileScrapeSections: ["videos"]`
     - `profileSorting: "popular"`
     - `resultsPerPage: 10`
     - `shouldDownloadVideos: true`
     - `proxyCountryCode: "VN"`
     - `videoKvStoreIdOrName: "my-tiktok-ASMR"`
   - Credentials: **Apify OAuth2**.
   - Connect: **Cron – Fetch Viral Ideas → Get Viral Video Dataset**.

4. **Add Set** node named **“Save Viral Ideas to Google Sheet”**  
   - Create fields: `Caption`, `Date Post`, `Username`, `Link user`, `Link video`, `VideoURL`, `Likes`, `Shares`, `View`, `Save`, `Hasgtag`  
   - Use expressions as in analysis (map from Apify output).  
   - Connect: **Get Viral Video Dataset → Save Viral Ideas to Google Sheet**.

5. **Add Google Sheets** node named **“Save Viral Ideas to Google Sheet1”**  
   - Operation: **Append**  
   - Select your spreadsheet (viral queue) and sheet tab.  
   - Map columns:
     - `Status = "Pending"`
     - `VideoURL = {{$json.VideoURL[0]}}`
     - plus other mapped fields.
   - Credentials: Google Sheets OAuth2.
   - Connect: **Save Viral Ideas to Google Sheet → Save Viral Ideas to Google Sheet1**.

6. **Add Schedule Trigger** node named **“Cron – Process Viral Row”**  
   - Interval: every **6 hours**.

7. **Add Google Sheets** node named **“Load Viral Idea from Sheet”**  
   - Operation: **Read / Get rows** (with filters)  
   - Filter: `Status` equals `Pending`  
   - Option: **Return first match** enabled  
   - Use same spreadsheet & sheet tab as the queue.
   - Connect: **Cron – Process Viral Row → Load Viral Idea from Sheet**.

8. **Add Gemini Video Analyze** node named **“Analyze Viral Video”**  
   - Resource: **video**, Operation: **analyze**  
   - Model: `models/gemini-3-flash-preview`  
   - Prompt: “Analyze and extract the main content of the video.”  
   - Video URLs: `{{$json.VideoURL}}`  
   - Connect: **Load Viral Idea from Sheet → Analyze Viral Video**.  
   - Configure Google Gemini credentials (Google AI Studio / Vertex credentials depending on your n8n setup).

9. **Add Gemini Chat Model** node named **“Google Gemini Chat Model”**  
   - Model: `models/gemini-3-flash-preview`  
   - This node will connect to the Agent as its **Language Model** input.

10. **Add Output Parser (Structured)** node named **“Parse Script Structure”**  
   - Schema example:
     ```json
     { "title": "string", "script": "string", "caption": "string" }
     ```

11. **Add LangChain Agent** node named **“AI Agent – Script & Avatar Director”**  
   - Prompt type: “Define”  
   - Enable output parser  
   - System message: include the constraints (script < 500 chars; caption < 120 chars; output JSON only) and embed:
     - `{{$json.content.parts[0].text.replace(/\n+/g, ' ')}}`
   - Connect inputs:
     - **Analyze Viral Video → Agent (main)**
     - **Google Gemini Chat Model → Agent (ai_languageModel)**
     - **Parse Script Structure → Agent (ai_outputParser)**

12. **Add Set** node named **“Finalize Avatar Script”**  
   - Set:
     - `Scripts = {{$json.output.script}}`
     - `Title = {{$json.output.title}}`
     - `Caption = {{$json.output.caption}}`
   - Connect: **Agent → Finalize Avatar Script**.

13. **Add HTTP Request** node named **“Create AI Avatar Video”** (HeyGen generate)  
   - Method: POST  
   - URL: `https://api.heygen.com/v2/video/generate`  
   - JSON body: include avatar ID, voice ID, `input_text: {{$json.Scripts}}`, dimension 720×1280.  
   - Authentication: **HTTP Header Auth** (HeyGen API key in header as required by HeyGen).  
   - Connect: **Finalize Avatar Script → Create AI Avatar Video**.

14. **Add Wait** node named **“Wait for Video Rendering”**  
   - Configure as a time delay (recommended) to avoid tight polling (e.g., 30–120 seconds).  
   - Connect: **Create AI Avatar Video → Wait for Video Rendering**.

15. **Add HTTP Request** node named **“Get Rendered Video”** (HeyGen status)  
   - Method: GET  
   - URL: `https://api.heygen.com/v1/video_status.get`  
   - Query param: `video_id = {{$('Create AI Avatar Video').item.json.data.video_id}}`  
   - Same HeyGen header auth.  
   - Connect: **Wait for Video Rendering → Get Rendered Video**.

16. **Add IF** node named **“Check Video Status”**  
   - Condition: `{{$json.data.status}} equals "processing"`  
   - True → back to **Wait for Video Rendering**  
   - False → proceed to publishing  
   - Connect: **Get Rendered Video → Check Video Status**  
   - Connect true branch: **Check Video Status (true) → Wait for Video Rendering**

17. **Add Blotato** node named **“Publish Video to Tiktok”**  
   - Platform: TikTok  
   - Account: select your Blotato TikTok account  
   - Text:
     - `{{$('Finalize Avatar Script').item.json.Title}}\n{{$('Finalize Avatar Script').item.json.Caption}}`
   - Media URL: `{{$json.data.video_url}}`  
   - Connect: **Check Video Status (false) → Publish Video to Tiktok**  
   - Credentials: Blotato API key.

18. **Add Blotato** node named **“Publish Video to Facebook”**  
   - Platform: Facebook  
   - Select account + page  
   - Text: same as TikTok  
   - Media URL: `{{$('Get Rendered Video').item.json.data.video_url}}`  
   - Connect: **Publish Video to Tiktok → Publish Video to Facebook**.

19. **Add Google Sheets** node named **“Update row Viral Content”**  
   - Operation: **Update**  
   - Match on `row_number`  
   - Set:
     - `Status = "Done"`
     - `row_number = {{$('Load Viral Idea from Sheet').item.json.row_number}}`
   - Connect: **Publish Video to Tiktok → Update row Viral Content**.

20. **Activate workflow** after validating credentials and running a manual test.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Author: GiangxAI | https://www.youtube.com/@giangxai.official |
| Workflow concept summary and setup checklist (as provided in sticky note) | Included in “Sticky Note4” content |

--- 

**Notable implementation risks to consider before production:**
- No deduplication for scraped videos → repeated queue rows.
- Poll loop has no max retries and only checks `processing` → handle `failed/queued` explicitly.
- `Analyze Viral Video` expects `VideoURL`; your sheet stores a single URL string—validate node accepts string or convert to array.