Generate TikTok and Instagram reel clips from YouTube videos with Vizard AI

https://n8nworkflows.xyz/workflows/generate-tiktok-and-instagram-reel-clips-from-youtube-videos-with-vizard-ai-14031


# Generate TikTok and Instagram reel clips from YouTube videos with Vizard AI

# 1. Workflow Overview

This workflow accepts a YouTube URL through an n8n form, sends the video to Vizard AI for automatic clip generation, polls Vizard until processing completes, filters the generated clips by viral score, and then either downloads the selected clips or publishes them to connected social media accounts.

Typical use cases:
- Turning long-form YouTube content into short-form vertical clips
- Building a semi-automated TikTok / Instagram Reels repurposing pipeline
- Filtering AI-generated clips before export based on a quality threshold
- Auto-publishing selected clips when Vizard social accounts are already connected

## 1.1 Input Reception and Runtime Configuration
The workflow starts with a form where a user submits a YouTube video URL. A Set node then prepares runtime settings such as the Vizard API key, export mode, threshold, and schedule time.

## 1.2 Clip Creation Request
The configured payload is sent to Vizard AI to create a new clipping project from the submitted YouTube URL.

## 1.3 Processing Validation and Polling Loop
The workflow first checks whether project creation succeeded. If yes, it waits and repeatedly queries project status until Vizard reports completion or failure.

## 1.4 Clip Filtering and Export Routing
Once processing is complete, the workflow retrieves linked social accounts, filters clips by viral score, and routes output according to the configured export mode.

## 1.5 Download or Social Publishing
If export mode is `download`, the clip file URLs are downloaded directly. If export mode is `auto_publish`, the workflow posts immediately or schedules publication depending on whether a schedule timestamp is present.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Configuration

### Overview
This block captures the YouTube URL from a form submission and prepares all parameters required by downstream Vizard API calls. It acts as the control center for workflow behavior.

### Nodes Involved
- `form_trigger`
- `Configuration`

### Node Details

#### 1. `form_trigger`
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry point of the workflow. Hosts a form and starts execution when submitted.
- **Configuration choices:**
  - Form title: `YouTube Video Clipper`
  - One required field: `YouTube Video Url`
  - Placeholder suggests a standard YouTube watch URL
- **Key expressions or variables used:**
  - Outputs submitted form value under `YouTube Video Url`
- **Input and output connections:**
  - No input; this is the workflow trigger
  - Output goes to `Configuration`
- **Version-specific requirements:**
  - Uses typeVersion `2.2`
- **Edge cases or potential failure types:**
  - Invalid or unsupported YouTube URL may still pass this node because validation is only “required”, not pattern-based
  - If the form is not activated in production, webhook access may be limited depending on n8n mode
- **Sub-workflow reference:** None

#### 2. `Configuration`
- **Type and technical role:** `n8n-nodes-base.set`  
  Defines static and dynamic runtime variables for the rest of the workflow.
- **Configuration choices:**
  - Sets:
    - `vizard_api_key`: placeholder string to replace manually
    - `video_type`: `"2"`
    - `video_length`: `"[0]"`
    - `video_ratio`: `"1"`
    - `video_url`: mapped from form field
    - `video_lang`: `"auto"`
    - `export_mode`: `"download"`
    - `viral_score_threshold`: `5`
    - `schedule_time`: `null`
- **Key expressions or variables used:**
  - `={{ $json['YouTube Video Url'] }}`
- **Input and output connections:**
  - Input from `form_trigger`
  - Output to `AI Clipper`
- **Version-specific requirements:**
  - Uses typeVersion `3.4`
- **Edge cases or potential failure types:**
  - If API key is not replaced, all Vizard API requests will fail with authorization errors
  - Some configured fields (`video_type`, `video_length`, `video_ratio`) are not actually used downstream in the current implementation
  - `schedule_time` is explicitly `null`, so auto-publish defaults to instant posting unless changed
- **Sub-workflow reference:** None

---

## Block 2 — Clip Creation Request

### Overview
This block sends the YouTube URL to Vizard AI to start automatic clip extraction. It then validates whether the create-project call was accepted successfully.

### Nodes Involved
- `AI Clipper`
- `Success?`

### Node Details

#### 3. `AI Clipper`
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Vizard AI project creation endpoint.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/create`
  - JSON body includes:
    - `videoUrl` from configuration
    - `videoType`: `2`
    - `preferLength`: `[1,2]`
    - `projectName`: `test`
    - `lang`: `auto`
  - Sends header `VIZARDAI_API_KEY`
- **Key expressions or variables used:**
  - `{{ $json.video_url }}`
  - `={{ $json.vizard_api_key }}`
- **Input and output connections:**
  - Input from `Configuration`
  - Output to `Success?`
- **Version-specific requirements:**
  - Uses typeVersion `4.2`
- **Edge cases or potential failure types:**
  - Invalid API key or missing key
  - Invalid YouTube URL or unsupported video source
  - Vizard API service errors or rate limiting
  - The node hardcodes values like `videoType`, `projectName`, and `lang`, ignoring some configuration fields that suggest customizability
- **Sub-workflow reference:** None

#### 4. `Success?`
- **Type and technical role:** `n8n-nodes-base.if`  
  Checks whether Vizard returned success code `2000` from project creation.
- **Configuration choices:**
  - Condition: `$json.code == 2000`
- **Key expressions or variables used:**
  - `={{ $json.code }}`
- **Input and output connections:**
  - Input from `AI Clipper`
  - True output goes to `Wait`
  - False output is not connected
- **Version-specific requirements:**
  - Uses typeVersion `2.2`
- **Edge cases or potential failure types:**
  - If response schema changes or `code` is missing, the condition may evaluate unexpectedly
  - Failed creations simply stop because the false branch is not handled
- **Sub-workflow reference:** None

---

## Block 3 — Processing Validation and Polling Loop

### Overview
This block waits for Vizard to process the project, queries project status, and loops until completion. It separates successful completion from “still processing” and generic failure.

### Nodes Involved
- `Wait`
- `Get Status`
- `Check Status`

### Node Details

#### 5. `Wait`
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses execution before re-checking project status.
- **Configuration choices:**
  - No explicit timing parameters are configured in the JSON
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Success?` true branch and from `Check Status` processing branch
  - Output to `Get Status`
- **Version-specific requirements:**
  - Uses typeVersion `1.1`
- **Edge cases or potential failure types:**
  - Because no wait duration is defined, behavior depends on node defaults / n8n wait-node semantics in the running version
  - If not configured appropriately after import, the polling loop may not behave as intended
- **Sub-workflow reference:** None

#### 6. `Get Status`
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries Vizard for the status of the created clipping project.
- **Configuration choices:**
  - Method: default GET
  - URL includes `projectId` from current item:
    - `/project/query/{{ $json.projectId }}`
  - Header `VIZARDAI_API_KEY` from `Configuration`
- **Key expressions or variables used:**
  - `=https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/query/{{ $json.projectId }}`
  - `={{ $('Configuration').item.json.vizard_api_key }}`
- **Input and output connections:**
  - Input from `Wait`
  - Output to `Check Status`
- **Version-specific requirements:**
  - Uses typeVersion `4.2`
- **Edge cases or potential failure types:**
  - If `projectId` is missing from previous response, request URL becomes invalid
  - Auth failures, timeout, API outages
  - Long processing jobs may cause many polling cycles
- **Sub-workflow reference:** None

#### 7. `Check Status`
- **Type and technical role:** `n8n-nodes-base.switch`  
  Routes execution based on Vizard status code.
- **Configuration choices:**
  - Output `Success` when `code == 2000`
  - Output `Processing` when `code == 1000`
  - Fallback output renamed to `Failed`
- **Key expressions or variables used:**
  - `={{ $json.code }}`
- **Input and output connections:**
  - Input from `Get Status`
  - `Success` output to `Retrieve Social Media Account`
  - `Processing` output back to `Wait`
  - `Failed` output is unconnected
- **Version-specific requirements:**
  - Uses typeVersion `3.3`
- **Edge cases or potential failure types:**
  - Unhandled failure branch means unsuccessful jobs stop silently unless n8n captures execution state
  - If Vizard uses additional status codes, they fall into the unconnected fallback output
- **Sub-workflow reference:** None

---

## Block 4 — Clip Filtering and Export Routing

### Overview
After processing completes, this block retrieves available Vizard social accounts, filters generated clips based on viral score, and chooses whether to download or publish.

### Nodes Involved
- `Retrieve Social Media Account`
- `ViralSocre Filter`
- `Switch`

### Node Details

#### 8. `Retrieve Social Media Account`
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Retrieves social publishing accounts connected in Vizard.
- **Configuration choices:**
  - GET `https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/social-accounts`
  - Sends `VIZARDAI_API_KEY` header
- **Key expressions or variables used:**
  - `={{ $('Configuration').item.json.vizard_api_key }}`
- **Input and output connections:**
  - Input from `Check Status` success branch
  - Output to `ViralSocre Filter`
- **Version-specific requirements:**
  - Uses typeVersion `4.3`
- **Edge cases or potential failure types:**
  - If no social accounts are connected, later publish steps may fail
  - This call is made even when export mode is `download`, so it is not strictly required in all paths
- **Sub-workflow reference:** None

#### 9. `ViralSocre Filter`
- **Type and technical role:** `n8n-nodes-base.code`  
  Reads processed clip results and emits only videos whose viral score exceeds the configured threshold.
- **Configuration choices:**
  - JavaScript code:
    - Reads `viral_score_threshold` from `Configuration`
    - Reads `videos` array from `Get Status`
    - Filters where `Number(video.viralScore) > threshold`
    - Returns one item per accepted clip
- **Key expressions or variables used:**
  - `$('Configuration').first().json.viral_score_threshold`
  - `$('Get Status').first().json.videos`
- **Input and output connections:**
  - Input from `Retrieve Social Media Account`
  - Output to `Switch`
- **Version-specific requirements:**
  - Uses typeVersion `2`
- **Edge cases or potential failure types:**
  - Typo in node name (`ViralSocre`) does not break logic but reduces readability
  - If `videos` is undefined or not an array, code will throw
  - Filter uses strict `>` rather than `>=`, so a score exactly equal to threshold is excluded
  - If no videos pass the threshold, no items continue downstream
- **Sub-workflow reference:** None

#### 10. `Switch`
- **Type and technical role:** `n8n-nodes-base.switch`  
  Routes filtered clips based on configured export mode.
- **Configuration choices:**
  - Branch 1: `export_mode == auto_publish`
  - Branch 2: `export_mode == download`
- **Key expressions or variables used:**
  - `={{ $('Configuration').item.json.export_mode }}`
- **Input and output connections:**
  - Input from `ViralSocre Filter`
  - Output 1 to `If`
  - Output 2 to `Download`
- **Version-specific requirements:**
  - Uses typeVersion `3.4`
- **Edge cases or potential failure types:**
  - Any export mode other than the two expected values produces no downstream action
- **Sub-workflow reference:** None

---

## Block 5 — Download or Social Publishing

### Overview
This block either downloads the filtered clips directly or publishes them through Vizard to a connected social account. For publishing, it decides between scheduled and immediate posting.

### Nodes Involved
- `Download`
- `If`
- `Scheduled Publish`
- `Instant Post`

### Node Details

#### 11. `Download`
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the final video from the clip’s `videoUrl`.
- **Configuration choices:**
  - GET request to `videoUrl` from each filtered clip
- **Key expressions or variables used:**
  - `={{ $('ViralSocre Filter').item.json.videoUrl }}`
- **Input and output connections:**
  - Input from `Switch` download branch
  - No downstream connection
- **Version-specific requirements:**
  - Uses typeVersion `4.3`
- **Edge cases or potential failure types:**
  - Large file downloads may timeout or exceed memory limits depending on n8n setup
  - No binary/file handling configuration is explicitly defined, so practical downstream use is limited
  - If URL expires or requires auth, download fails
- **Sub-workflow reference:** None

#### 12. `If`
- **Type and technical role:** `n8n-nodes-base.if`  
  Determines whether a publish timestamp exists.
- **Configuration choices:**
  - Condition checks whether `schedule_time` is not empty
- **Key expressions or variables used:**
  - `={{ $('Configuration').item.json.schedule_time }}`
- **Input and output connections:**
  - Input from `Switch` auto_publish branch
  - True output to `Scheduled Publish`
  - False output to `Instant Post`
- **Version-specific requirements:**
  - Uses typeVersion `2.3`
- **Edge cases or potential failure types:**
  - `schedule_time` is initialized as `null`; depending on expression handling, this correctly routes to instant post
  - No validation that timestamp is a future Unix timestamp
- **Sub-workflow reference:** None

#### 13. `Scheduled Publish`
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Publishes a clip to a connected social account at a scheduled time.
- **Configuration choices:**
  - POST to `/project/publish-video`
  - Body parameters:
    - `finalVideoId` from filtered clip
    - `socialAccountId` from first available publish account
    - `post` empty
    - `title` from clip title
    - `publishTime` from configuration
  - Sends Vizard API key header
- **Key expressions or variables used:**
  - `={{ $('ViralSocre Filter').item.json.videoId }}`
  - `={{ $('Retrieve Social Media Account').item.json.publishAccounts[0].id }}`
  - `= {{ $('ViralSocre Filter').item.json.title }}`
  - `={{ $('Configuration').item.json.schedule_time }}`
- **Input and output connections:**
  - Input from `If` true branch
  - No downstream connection
- **Version-specific requirements:**
  - Uses typeVersion `4.3`
- **Edge cases or potential failure types:**
  - If `publishAccounts` is empty, expression fails
  - Empty `post` parameter may or may not be accepted depending on API expectations
  - Invalid timestamp, expired scheduling window, or unsupported account type may cause API rejection
- **Sub-workflow reference:** None

#### 14. `Instant Post`
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Publishes a clip immediately using the same Vizard publish endpoint.
- **Configuration choices:**
  - POST to `/project/publish-video`
  - Uses the same body structure as Scheduled Publish, including `publishTime`
- **Key expressions or variables used:**
  - `={{ $('ViralSocre Filter').item.json.videoId }}`
  - `={{ $('Retrieve Social Media Account').item.json.publishAccounts[0].id }}`
  - `= {{ $('ViralSocre Filter').item.json.title }}`
  - `={{ $('Configuration').item.json.schedule_time }}`
- **Input and output connections:**
  - Input from `If` false branch
  - No downstream connection
- **Version-specific requirements:**
  - Uses typeVersion `4.3`
- **Edge cases or potential failure types:**
  - It still sends `publishTime`, which will likely be `null`; behavior depends on Vizard API tolerance
  - Same account availability issues as Scheduled Publish
- **Sub-workflow reference:** None

---

## Additional Non-Executable Nodes — Sticky Notes

These are documentation nodes and do not process data, but they provide important context.

### 15. `Main Overview`
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Explains overall workflow purpose, setup, and customization options.
- **Relevant notes:**
  - Get Vizard API key from vizard.ai
  - Replace placeholder API key in Configuration node
  - Choose `download` or `auto_publish`
  - Optional Unix timestamp for delayed posting
  - Adjust threshold and `preferLength`

### 16. `Processing Section`
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Labels the processing and status monitoring area.

### 17. `Export Section`
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Labels filtering and export routing area.

### 18. `Processing Section1`
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Labels input and AI clipping area.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| form_trigger | Form Trigger | Receives YouTube URL submission from user form |  | Configuration | ## 1. Get video URL and AI clipping\n\nSubmits video to Vizard AI |
| Configuration | Set | Defines API key and workflow runtime settings | form_trigger | AI Clipper | ## 1. Get video URL and AI clipping\n\nSubmits video to Vizard AI |
| AI Clipper | HTTP Request | Creates a Vizard clipping project from the YouTube URL | Configuration | Success? | ## 1. Get video URL and AI clipping\n\nSubmits video to Vizard AI |
| Success? | If | Confirms project creation returned Vizard success code 2000 | AI Clipper | Wait | ## 2. AI Video Processing & Status Monitoring\n\nPolls for completion, validates success |
| Wait | Wait | Pauses before polling Vizard project status again | Success?, Check Status | Get Status | ## 2. AI Video Processing & Status Monitoring\n\nPolls for completion, validates success |
| Get Status | HTTP Request | Queries Vizard for processing status and generated clips | Wait | Check Status | ## 2. AI Video Processing & Status Monitoring\n\nPolls for completion, validates success |
| Check Status | Switch | Routes status as success, still processing, or failure | Get Status | Retrieve Social Media Account, Wait | ## 2. AI Video Processing & Status Monitoring\n\nPolls for completion, validates success |
| Retrieve Social Media Account | HTTP Request | Retrieves connected Vizard social publishing accounts | Check Status | ViralSocre Filter | ## 3. Filtering & Export\n\nFilters clips by viral score, routes to download or social publishing |
| ViralSocre Filter | Code | Filters generated clips by viral score threshold | Retrieve Social Media Account | Switch | ## 3. Filtering & Export\n\nFilters clips by viral score, routes to download or social publishing |
| Switch | Switch | Routes clips to download or auto-publish mode | ViralSocre Filter | If, Download | ## 3. Filtering & Export\n\nFilters clips by viral score, routes to download or social publishing |
| If | If | Chooses scheduled versus immediate publishing | Switch | Scheduled Publish, Instant Post | ## 3. Filtering & Export\n\nFilters clips by viral score, routes to download or social publishing |
| Scheduled Publish | HTTP Request | Publishes a clip at a scheduled time | If |  | ## 3. Filtering & Export\n\nFilters clips by viral score, routes to download or social publishing |
| Instant Post | HTTP Request | Publishes a clip immediately | If |  | ## 3. Filtering & Export\n\nFilters clips by viral score, routes to download or social publishing |
| Download | HTTP Request | Downloads the generated clip file from its URL | Switch |  | ## 3. Filtering & Export\n\nFilters clips by viral score, routes to download or social publishing |
| Main Overview | Sticky Note | Workspace documentation and setup guidance |  |  | ## Viral Clip Generator for YouTube Videos\n\nAutomatically extract viral moments from YouTube videos and publish to TikTok, Instagram Reels, and other social platforms using Vizard AI.\n\n### How it works\n\n1. **Submit URL**: Enter a YouTube video link via the form trigger\n2. **AI Analysis**: Vizard AI processes the video and identifies viral-worthy clips\n3. **Quality Filter**: Clips are scored 0-10 for viral potential; only those above your threshold proceed\n4. **Smart Export**: Clips are either downloaded for review or auto-published to your connected social accounts\n\n### Setup\n\n- **Get Vizard API key** from vizard.ai\n- Open **Configuration** node and replace `[ENTER_YOUR_API_KEY_HERE]` with your actual key\n- Set **viral_score_threshold** (default: 5, range: 0-10)\n- Choose **export_mode**: `download` or `auto_publish`\n- Optional: Set **schedule_time** as Unix timestamp for delayed posting\n\n### Customization\n\n- Adjust threshold (7-8 for top clips only, 3-4 for more volume)\n- Modify `preferLength` in AI Clipper node for different clip durations\n- Connect social accounts in Vizard dashboard for auto-publishing |
| Processing Section | Sticky Note | Workspace label for processing area |  |  | ## 2. AI Video Processing & Status Monitoring\n\nPolls for completion, validates success |
| Export Section | Sticky Note | Workspace label for export area |  |  | ## 3. Filtering & Export\n\nFilters clips by viral score, routes to download or social publishing |
| Processing Section1 | Sticky Note | Workspace label for input/clipping area |  |  | ## 1. Get video URL and AI clipping\n\nSubmits video to Vizard AI |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Generate viral TikTok/IG reel clips from YouTube videos with Vizard AI`.

2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Form title: `YouTube Video Clipper`
   - Add one field:
     - Label: `YouTube Video Url`
     - Placeholder: `https://www.youtube.com/watch?v=DB9mjd-65gw`
     - Required: enabled
   - This is the workflow entry point.

3. **Add a Set node named `Configuration`**
   - Connect `form_trigger -> Configuration`
   - Add these fields:
     - `vizard_api_key` = `[ENTER_YOUR_API_KEY_HERE]`
     - `video_type` = `2` as string
     - `video_length` = `[0]` as string
     - `video_ratio` = `1` as string
     - `video_url` = expression: `{{$json['YouTube Video Url']}}`
     - `video_lang` = `auto`
     - `export_mode` = `download`
     - `viral_score_threshold` = `5` as number
     - `schedule_time` = `null` as number/null depending on editor behavior
   - Replace the API key placeholder with your actual Vizard API key before production use.

4. **Add an HTTP Request node named `AI Clipper`**
   - Connect `Configuration -> AI Clipper`
   - Method: `POST`
   - URL: `https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/create`
   - Enable **Send Headers**
   - Add header:
     - `VIZARDAI_API_KEY` = `{{$json.vizard_api_key}}`
   - Enable **Send Body**
   - Body content type: **JSON**
   - JSON body:
     - `videoUrl` = `{{$json.video_url}}`
     - `videoType` = `2`
     - `preferLength` = `[1,2]`
     - `projectName` = `test`
     - `lang` = `auto`

5. **Add an If node named `Success?`**
   - Connect `AI Clipper -> Success?`
   - Condition:
     - Left value: `{{$json.code}}`
     - Operation: equals
     - Right value: `2000`
   - This checks whether project creation succeeded.

6. **Add a Wait node named `Wait`**
   - Connect the **true** output of `Success? -> Wait`
   - Configure an appropriate wait strategy if needed in your n8n version.
   - Practical recommendation: set a polling delay such as 30–60 seconds, because Vizard processing is asynchronous.

7. **Add an HTTP Request node named `Get Status`**
   - Connect `Wait -> Get Status`
   - Method: `GET`
   - URL expression:
     - `https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/query/{{$json.projectId}}`
   - Enable **Send Headers**
   - Header:
     - `VIZARDAI_API_KEY` = `{{$('Configuration').item.json.vizard_api_key}}`

8. **Add a Switch node named `Check Status`**
   - Connect `Get Status -> Check Status`
   - Create output rule 1:
     - Name: `Success`
     - Condition: `{{$json.code}} == 2000`
   - Create output rule 2:
     - Name: `Processing`
     - Condition: `{{$json.code}} == 1000`
   - Set fallback output name to `Failed`
   - Connect:
     - `Success -> Retrieve Social Media Account`
     - `Processing -> Wait`
   - Leave `Failed` unconnected unless you want explicit error handling.

9. **Add an HTTP Request node named `Retrieve Social Media Account`**
   - Connect `Check Status (Success) -> Retrieve Social Media Account`
   - Method: `GET`
   - URL: `https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/social-accounts`
   - Enable **Send Headers**
   - Header:
     - `VIZARDAI_API_KEY` = `{{$('Configuration').item.json.vizard_api_key}}`

10. **Add a Code node named `ViralSocre Filter`**
    - Connect `Retrieve Social Media Account -> ViralSocre Filter`
    - Paste this JavaScript logic:
      ```javascript
      const threshold = $('Configuration').first().json.viral_score_threshold;

      return $('Get Status').first().json.videos
        .filter(video => Number(video.viralScore) > threshold)
        .map(video => ({
          json: video
        }));
      ```
    - This emits one item per clip above threshold.

11. **Add a Switch node named `Switch`**
    - Connect `ViralSocre Filter -> Switch`
    - Rule 1:
      - `{{$('Configuration').item.json.export_mode}}` equals `auto_publish`
    - Rule 2:
      - `{{$('Configuration').item.json.export_mode}}` equals `download`

12. **Add an HTTP Request node named `Download`**
    - Connect `Switch (download branch) -> Download`
    - Method: `GET`
    - URL:
      - `{{$('ViralSocre Filter').item.json.videoUrl}}`
    - If you need actual files in n8n, configure response handling as file/binary according to your n8n version.

13. **Add an If node named `If`**
    - Connect `Switch (auto_publish branch) -> If`
    - Condition:
      - Left value: `{{$('Configuration').item.json.schedule_time}}`
      - Operation: `is not empty` / `notEmpty`
    - True branch = scheduled
    - False branch = instant

14. **Add an HTTP Request node named `Scheduled Publish`**
    - Connect `If (true) -> Scheduled Publish`
    - Method: `POST`
    - URL: `https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/publish-video`
    - Enable **Send Headers**
    - Header:
      - `VIZARDAI_API_KEY` = `{{$('Configuration').item.json.vizard_api_key}}`
    - Enable **Send Body**
    - Add body parameters:
      - `finalVideoId` = `{{$('ViralSocre Filter').item.json.videoId}}`
      - `socialAccountId` = `{{$('Retrieve Social Media Account').item.json.publishAccounts[0].id}}`
      - `post` = empty value
      - `title` = `{{$('ViralSocre Filter').item.json.title}}`
      - `publishTime` = `{{$('Configuration').item.json.schedule_time}}`

15. **Add an HTTP Request node named `Instant Post`**
    - Connect `If (false) -> Instant Post`
    - Method: `POST`
    - URL: `https://elb-api.vizard.ai/hvizard-server-front/open-api/v1/project/publish-video`
    - Enable **Send Headers**
    - Header:
      - `VIZARDAI_API_KEY` = `{{$('Configuration').item.json.vizard_api_key}}`
    - Enable **Send Body**
    - Add body parameters:
      - `finalVideoId` = `{{$('ViralSocre Filter').item.json.videoId}}`
      - `socialAccountId` = `{{$('Retrieve Social Media Account').item.json.publishAccounts[0].id}}`
      - `post` = empty value
      - `title` = `{{$('ViralSocre Filter').item.json.title}}`
      - `publishTime` = `{{$('Configuration').item.json.schedule_time}}`
   - Even though this is the instant branch, the original workflow still sends `publishTime`.

16. **Add workspace sticky notes for clarity**
   - Sticky note 1 near the start:
     - `## 1. Get video URL and AI clipping`
     - `Submits video to Vizard AI`
   - Sticky note 2 near polling:
     - `## 2. AI Video Processing & Status Monitoring`
     - `Polls for completion, validates success`
   - Sticky note 3 near export:
     - `## 3. Filtering & Export`
     - `Filters clips by viral score, routes to download or social publishing`
   - Sticky note 4 as an overall overview:
     - Include setup reminders for API key, threshold, export mode, schedule time, and preferLength customization.

17. **Configure credentials**
   - This workflow does **not** use n8n credential objects in the provided version.
   - Instead, it passes the Vizard API key manually through a header field from the `Configuration` node.
   - Recommended improvement:
     - Store the API key in an environment variable or credential-like secure source rather than hardcoding it in Set.

18. **Test the workflow**
   - Submit a valid public YouTube URL.
   - Verify:
     - `AI Clipper` returns `code: 2000`
     - `Get Status` eventually returns completion
     - `videos` array exists
     - Filtered clips are produced
     - Download or publish path works as expected

19. **Optional hardening improvements**
   - Add validation for YouTube URL format at form level
   - Add explicit handling for failed creation and failed processing branches
   - Add a maximum polling count to avoid infinite loops
   - Skip `Retrieve Social Media Account` when `export_mode = download`
   - Validate `publishAccounts[0]` before posting
   - Change filter from `>` to `>=` if threshold equality should pass
   - Use configuration values consistently in `AI Clipper` instead of hardcoded `videoType`, `lang`, and `preferLength`

### Sub-workflow setup
- No sub-workflows are used.
- No Execute Workflow node is present.
- This workflow has a single entry point: `form_trigger`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Get Vizard API key from Vizard before running the workflow. | `https://vizard.ai` |
| The workflow is designed to extract viral moments from YouTube videos and repurpose them for TikTok, Instagram Reels, and similar platforms. | General workflow purpose |
| Default viral score threshold is 5, with a documented suggested range of 0–10. | Configuration guidance |
| Suggested threshold tuning: use 7–8 for stricter selection, or 3–4 for higher clip volume. | Filtering strategy |
| `export_mode` supports `download` or `auto_publish`. | Configuration guidance |
| `schedule_time` is optional and should be a Unix timestamp for delayed posting. | Publishing behavior |
| `preferLength` in the AI Clipper node can be adjusted to change clip duration preferences. | AI clip generation customization |
| Social accounts must be connected in the Vizard dashboard if using auto-publish. | Vizard publishing prerequisite |