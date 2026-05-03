Schedule social media posts from local files using UploadToURL, OpenAI, and Buffer

https://n8nworkflows.xyz/workflows/schedule-social-media-posts-from-local-files-using-uploadtourl--openai--and-buffer-13540


# Schedule social media posts from local files using UploadToURL, OpenAI, and Buffer

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensif ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow receives a media asset (either as a remote URL or uploaded binary) plus posting preferences, hosts the asset publicly via **UploadToURL**, generates platform-optimized copy via **OpenAI**, then schedules the post via **Buffer** for **Twitter/X**, **Instagram**, or **LinkedIn**, and returns a unified JSON response.

**Target use cases:**
- Automating social post scheduling from local files or externally hosted media
- Generating AI captions/hashtags/alt-text with consistent JSON output
- Routing and scheduling posts across multiple social platforms using Buffer

### 1.1 Input Reception & Validation
Webhook intake → normalize/validate payload → decide upload path (remote URL vs binary).

### 1.2 Asset Hosting (UploadToURL)
Upload via community node (remote-link branch vs binary branch) → normalize returned URL.

### 1.3 AI Copy Generation & Payload Assembly
OpenAI generates structured JSON copy → workflow assembles final caption, hashtags, alt-text, and computes schedule time.

### 1.4 Platform Routing & Buffer Scheduling + Response
Switch routes by platform → Buffer API schedules the post → response node returns final scheduling summary.

---

## 2. Block-by-Block Analysis

### Block 1 — Intake & Validation
**Overview:**  
Receives a POST request, validates required fields and platform choice, enriches metadata (character limits, content type, cleaned asset name), and routes execution based on whether the input is a remote URL.

**Nodes Involved:**
- `Webhook - Receive Asset`
- `Validate & Enrich Payload`
- `Has Remote URL?`

#### Node: Webhook - Receive Asset
- **Type / Role:** `n8n-nodes-base.webhook` — entry point for external requests.
- **Configuration choices:**
  - **Method:** POST
  - **Path:** `social-asset-upload`
  - **CORS:** `allowedOrigins: *` (accepts any origin)
  - **Response mode:** “Response Node” (workflow must end with Respond to Webhook)
- **Inputs/Outputs:**
  - **Output →** `Validate & Enrich Payload`
- **Version notes:** Type version 2.
- **Failure/edge cases:**
  - Missing/incorrect payload structure (e.g., no `body` in webhook JSON)
  - Large binary uploads may hit n8n limits (depending on instance config)

#### Node: Validate & Enrich Payload
- **Type / Role:** `n8n-nodes-base.code` — validates request fields and builds a normalized metadata object.
- **Key logic/config:**
  - Reads request as: `const body = $input.first().json.body || $input.first().json;`
  - Allowed platforms: `instagram`, `twitter`, `linkedin`, `facebook`
  - Requires at least one of: `filename` or `fileUrl`
  - Character limits mapping:
    - Twitter 280, Instagram 2200, LinkedIn 3000, Facebook 63206
  - Derives:
    - `filename` from `body.filename` or last segment of `fileUrl`
    - `contentType` from extension: image/video/file
    - `cleanName` by removing extension, splitting underscores/dashes/camelCase
  - Normalized output includes:
    - `platform`, `tone`, `includeHashtags`, `brand`, `campaignContext`, `scheduleTime`, `timestamp`, etc.
- **Inputs/Outputs:**
  - **Input ←** Webhook
  - **Output →** `Has Remote URL?`
- **Version notes:** Type version 2.
- **Failure/edge cases:**
  - Throws explicit errors for missing filename/fileUrl or invalid platform
  - If binary is sent but no `filename` and no `fileUrl`, request is rejected even if binary exists (caller must provide `filename`)

#### Node: Has Remote URL?
- **Type / Role:** `n8n-nodes-base.if` — routes to remote-URL upload vs binary upload.
- **Condition:** checks `fileUrl` is not empty (`string.notEmpty` on `{{ $json.fileUrl }}`).
- **Outputs/Connections:**
  - **True →** `Upload to URL - From Remote URL`
  - **False →** `Upload to URL - From Binary`
- **Version notes:** Type version 2.
- **Failure/edge cases:**
  - If caller intended remote URL but sends whitespace/invalid URL, the branch still goes “true” (only checks not-empty, not URL validity)

---

### Block 2 — File Hosting via UploadToURL
**Overview:**  
Uploads/hosts the asset using the UploadToURL community node (either from a remote link or from binary), then normalizes the API response into a single `cleanUrl` field for downstream use.

**Nodes Involved:**
- `Upload to URL - From Remote URL`
- `Upload to URL - From Binary`
- `Extract Clean URL`

#### Node: Upload to URL - From Remote URL
- **Type / Role:** `n8n-nodes-uploadtourl.uploadToUrl` — host a remote asset and return a public link.
- **Configuration choices:**
  - Default operation (implicit): “upload from URL” (no explicit operation set)
  - Uses **UploadToURL API credentials**
- **Inputs/Outputs:**
  - **Input ←** IF (true branch)
  - **Output →** `Extract Clean URL`
- **Version notes:** Type version 1 (community node; must be installed).
- **Failure/edge cases:**
  - Credential/auth errors to UploadToURL
  - Remote URL unreachable, blocked, or unsupported content-type
  - API may return different field shapes across versions (handled later)

#### Node: Upload to URL - From Binary
- **Type / Role:** `n8n-nodes-uploadtourl.uploadToUrl` — upload binary file content.
- **Configuration choices:**
  - Operation set to: `uploadFile`
  - Uses same **UploadToURL API credentials**
- **Inputs/Outputs:**
  - **Input ←** IF (false branch)
  - **Output →** `Extract Clean URL`
- **Version notes:** Type version 1.
- **Failure/edge cases:**
  - If webhook doesn’t provide binary in the expected property, upload will fail
  - Large files may exceed UploadToURL limits or n8n memory limits

#### Node: Extract Clean URL
- **Type / Role:** `n8n-nodes-base.code` — normalizes UploadToURL response and merges it with validated metadata.
- **Key logic:**
  - Reads upload response from current input; reads metadata from:
    - `$('Validate & Enrich Payload').first().json`
  - Extracts URL using fallback chain:
    - `url` → `link` → `data.url` → `file.url` → `shortUrl`
  - Throws a descriptive error if no URL found, including the full response payload.
  - Outputs merged object:
    - metadata + `cleanUrl`, `uploadId`, `fileSize`
- **Inputs/Outputs:**
  - **Input ←** either UploadToURL branch
  - **Output →** `OpenAI - Generate Caption`
- **Version notes:** Type version 2.
- **Failure/edge cases:**
  - If UploadToURL changes response fields beyond those checked, node will throw
  - If `Validate & Enrich Payload` didn’t run (shouldn’t happen in normal flow), `$()` reference fails

---

### Block 3 — AI Caption Generation & Post Payload Assembly
**Overview:**  
Generates structured social copy using OpenAI (forced JSON output), then constructs the final caption, appends hashtags, warns on character limit overflow, and computes a schedule time if not provided.

**Nodes Involved:**
- `OpenAI - Generate Caption`
- `Assemble Post Payload`

#### Node: OpenAI - Generate Caption
- **Type / Role:** `@n8n/n8n-nodes-langchain.openAi` — calls OpenAI chat model to produce JSON-formatted copy.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Max tokens: 800
  - Temperature: 0.75
  - System message: “Always return a valid JSON object only — no markdown, no preamble.”
  - User prompt includes:
    - platform, clean asset name, contentType, tone, brand, campaignContext, charLimit, includeHashtags
  - Output schema requested (in prompt): JSON with keys:
    - `caption`, `hashtags[]`, `altText`, `hook`, `emojiUsage`, `bestPostingTimeUTC`, `estimatedEngagement`
- **Inputs/Outputs:**
  - **Input ←** `Extract Clean URL`
  - **Output →** `Assemble Post Payload`
- **Version notes:** Type version 1.5.
- **Failure/edge cases:**
  - OpenAI credential/auth issues or quota exhaustion
  - Model may still return invalid JSON (handled in next node)
  - Prompt asks for `bestPostingTimeUTC` like “14:00-16:00”; downstream parsing only uses the hour before `:`

#### Node: Assemble Post Payload
- **Type / Role:** `n8n-nodes-base.code` — parses OpenAI JSON, builds final caption and schedule timestamp.
- **Key logic:**
  - Tries to locate model text in multiple shapes:
    - `message.content`, `choices[0].message.content`, `content`, `text`
  - Parses JSON; throws on parse failure.
  - Builds hashtag string if enabled:
    - adds `#` prefix if missing; joins with spaces; appended after two newlines.
  - Character limit handling:
    - If over limit: logs `console.warn` (does not truncate or fail)
  - Schedule computation:
    - Uses `scheduleTime` from request if present
    - Else parses hour from `bestPostingTimeUTC` (defaults to “14:00”)
    - Sets next UTC time at that hour; if already in the past, schedules for next day
  - Outputs payload fields used later:
    - `assetUrl`, `platform`, `fullCaption`, `altText`, `scheduleAt`, `estimatedEngagement`, etc.
- **Inputs/Outputs:**
  - **Input ←** OpenAI node
  - Uses referenced data from `Extract Clean URL`
  - **Output →** `Route by Platform`
- **Version notes:** Type version 2.
- **Failure/edge cases:**
  - Invalid JSON from OpenAI → hard failure
  - If `bestPostingTimeUTC` is a range like `14:00-16:00`, parsing hour via `.split(':')[0]` yields `14` (works), but other formats (e.g. “2pm”) yield `NaN` → schedules at `NaN` hours (Date becomes invalid). Consider adding validation.
  - Caption over platform limit is only warned; Buffer/Twitter may reject or truncate depending on API behavior.

---

### Block 4 — Platform Routing, Buffer Scheduling, and Webhook Response
**Overview:**  
Routes the post by platform, calls Buffer’s “create update” endpoint with platform-specific media fields, then returns a consistent JSON response to the webhook caller.

**Nodes Involved:**
- `Route by Platform`
- `Buffer - Schedule Twitter/X`
- `Buffer - Schedule Instagram`
- `Buffer - Schedule LinkedIn`
- `Build Final Response`
- `Respond to Webhook`

#### Node: Route by Platform
- **Type / Role:** `n8n-nodes-base.switch` — routes based on `platform`.
- **Configuration choices:**
  - Rules:
    - `twitter` → output “Twitter/X”
    - `instagram` → output “Instagram”
    - `linkedin` → output “LinkedIn”
  - Fallback output: `extra`
- **Inputs/Outputs:**
  - **Input ←** Assemble Post Payload
  - **Outputs →** respective Buffer HTTP Request nodes
  - **Important:** The fallback (`extra`) is not connected in this workflow.
- **Version notes:** Type version 3.
- **Failure/edge cases:**
  - If platform is `facebook` (allowed earlier) it will go to fallback `extra` and then **stop** (no response is produced). This is a functional mismatch: validation allows facebook, routing does not.
  - Any unexpected platform value leads to the same dead-end.

#### Node: Buffer - Schedule Twitter/X
- **Type / Role:** `n8n-nodes-base.httpRequest` — calls Buffer API to schedule an update.
- **Endpoint:** `POST https://api.bufferapp.com/1/updates/create.json`
- **Authentication:** Generic credential type → `httpHeaderAuth`
  - **Note:** credential name shown is “Stripe Header Auth”, but it is used as a generic header auth. It must contain a valid Buffer token header (see reproduction section).
- **JSON body (built via expression):**
  - `text`: `$json.fullCaption`
  - `media.link`: `$json.assetUrl`
  - `media.description`: `$json.altText`
  - `scheduled_at`: `$json.scheduleAt`
  - `profile_ids`: `[$vars.BUFFER_PROFILE_TWITTER]`
- **Connections:**
  - **Input ←** Switch output “Twitter/X”
  - **Output →** `Build Final Response`
- **Version notes:** Type version 4.2.
- **Failure/edge cases:**
  - Wrong/expired Buffer token → 401/403
  - Missing/incorrect `profile_ids` variable → Buffer error
  - Twitter-specific constraints may cause Buffer rejection (e.g., media requirements)

#### Node: Buffer - Schedule Instagram
- **Type / Role:** HTTP Request to Buffer create update.
- **Differences vs Twitter:**
  - Adds `media.photo`: set to asset URL only if `contentType === 'image'`, else empty string.
- **Uses:** `$vars.BUFFER_PROFILE_INSTAGRAM`
- **Connections:** Switch “Instagram” → Buffer node → Build Final Response
- **Failure/edge cases:**
  - Instagram posting via Buffer may require additional setup/permissions and may reject non-image assets or empty `photo`
  - For videos, payload sets `photo` to `""`; Buffer may error. Consider conditional removal of the field instead of empty string.

#### Node: Buffer - Schedule LinkedIn
- **Type / Role:** HTTP Request to Buffer create update.
- **Differences:**
  - Adds `media.title`: uses `$json.filename`
- **Uses:** `$vars.BUFFER_PROFILE_LINKEDIN`
- **Connections:** Switch “LinkedIn” → Buffer node → Build Final Response
- **Failure/edge cases:**
  - LinkedIn post formatting/media constraints through Buffer can cause errors; returned errors will be surfaced in final response.

#### Node: Build Final Response
- **Type / Role:** `n8n-nodes-base.code` — normalizes Buffer response and merges with assembled post data for the caller.
- **Key logic:**
  - Reads Buffer response from input
  - Reads post data via: `$('Assemble Post Payload').first().json`
  - Success condition:
    - `success = !bufferResponse.error && bufferResponse.success !== false`
  - Outputs:
    - `success`, `message`, `scheduleId` (tries `update.id` then `data.id`), plus asset and caption fields
- **Connections:**
  - **Input ←** any Buffer node
  - **Output →** `Respond to Webhook`
- **Failure/edge cases:**
  - Buffer may return non-standard error shapes; success heuristic might misclassify some responses
  - If Buffer returns an HTTP error that stops execution (depending on HTTP Request node settings), this node may never run

#### Node: Respond to Webhook
- **Type / Role:** `n8n-nodes-base.respondToWebhook` — returns final JSON payload.
- **Configuration choices:**
  - HTTP 200
  - Header `Content-Type: application/json`
  - Response body: `{{ $json }}` (entire item JSON)
- **Connections:**
  - **Input ←** Build Final Response
- **Version notes:** Type version 1.1.
- **Failure/edge cases:**
  - If the workflow stops earlier (e.g., Switch fallback not connected), caller gets no proper response (request may hang/time out depending on n8n config)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Overview | Sticky Note | Documentation/overview | — | — | AI Social Media Auto-Scheduler via UploadToURL… Set variables: BUFFER_PROFILE_TWITTER, BUFFER_PROFILE_INSTAGRAM, BUFFER_PROFILE_LINKEDIN. |
| Section 1 — Intake | Sticky Note | Documentation/block label | — | — | ## 1 — Intake & validation… routes based on whether `fileUrl` is set. |
| Webhook - Receive Asset | Webhook | Entry point (POST intake) | — | Validate & Enrich Payload | ## 1 — Intake & validation… routes based on whether `fileUrl` is set. |
| Validate & Enrich Payload | Code | Validate input + enrich metadata | Webhook - Receive Asset | Has Remote URL? | ## 1 — Intake & validation… routes based on whether `fileUrl` is set. |
| Has Remote URL? | IF | Branch: remote URL vs binary upload | Validate & Enrich Payload | Upload to URL - From Remote URL; Upload to URL - From Binary | ## 1 — Intake & validation… routes based on whether `fileUrl` is set. |
| Section 2 — Upload | Sticky Note | Documentation/block label | — | — | ## 2 — File hosting via UploadToURL… normalises the response… |
| Upload to URL - From Remote URL | UploadToURL (community) | Host asset from remote URL | Has Remote URL? (true) | Extract Clean URL | ## 2 — File hosting via UploadToURL… normalises the response… |
| Upload to URL - From Binary | UploadToURL (community) | Host uploaded binary file | Has Remote URL? (false) | Extract Clean URL | ## 2 — File hosting via UploadToURL… normalises the response… |
| Extract Clean URL | Code | Normalize UploadToURL response to `cleanUrl` | Upload to URL - From Remote URL; Upload to URL - From Binary | OpenAI - Generate Caption | ## 2 — File hosting via UploadToURL… normalises the response… |
| Section 3 — Caption | Sticky Note | Documentation/block label | — | — | ## 3 — AI caption generation… calculates the schedule time… |
| OpenAI - Generate Caption | OpenAI (LangChain) | Generate structured caption JSON | Extract Clean URL | Assemble Post Payload | ## 3 — AI caption generation… calculates the schedule time… |
| Assemble Post Payload | Code | Build `fullCaption`, schedule time, final payload | OpenAI - Generate Caption | Route by Platform | ## 3 — AI caption generation… calculates the schedule time… |
| Section 4 — Scheduling | Sticky Note | Documentation/block label | — | — | ## 4 — Platform routing & scheduling… add more platforms… |
| Route by Platform | Switch | Route by platform to correct Buffer request | Assemble Post Payload | Buffer - Schedule Twitter/X; Buffer - Schedule Instagram; Buffer - Schedule LinkedIn | ## 4 — Platform routing & scheduling… add more platforms… |
| Buffer - Schedule Twitter/X | HTTP Request | Create scheduled Buffer update for Twitter/X | Route by Platform | Build Final Response | ## 4 — Platform routing & scheduling… add more platforms… |
| Buffer - Schedule Instagram | HTTP Request | Create scheduled Buffer update for Instagram | Route by Platform | Build Final Response | ## 4 — Platform routing & scheduling… add more platforms… |
| Buffer - Schedule LinkedIn | HTTP Request | Create scheduled Buffer update for LinkedIn | Route by Platform | Build Final Response | ## 4 — Platform routing & scheduling… add more platforms… |
| Build Final Response | Code | Normalize Buffer result to unified response | Buffer - Schedule Twitter/X; Buffer - Schedule Instagram; Buffer - Schedule LinkedIn | Respond to Webhook | ## 4 — Platform routing & scheduling… add more platforms… |
| Respond to Webhook | Respond to Webhook | Return JSON response to caller | Build Final Response | — | ## 4 — Platform routing & scheduling… add more platforms… |

---

## 4. Reproducing the Workflow from Scratch

1) **Install prerequisite community node**
   1. In n8n, install community node: `n8n-nodes-uploadtourl`.
   2. Restart n8n if required so the node appears in the node picker.

2) **Create workflow variables (recommended)**
   - In n8n, set workflow (or instance) variables:
     - `BUFFER_PROFILE_TWITTER`
     - `BUFFER_PROFILE_INSTAGRAM`
     - `BUFFER_PROFILE_LINKEDIN`
   These should be Buffer profile IDs (one per channel/account).

3) **Create credentials**
   1. **UploadToURL credential**
      - Create UploadToURL API credential (as required by the community node).
   2. **OpenAI credential**
      - Create OpenAI API credential for the LangChain OpenAI node.
   3. **Buffer credential (HTTP Header Auth)**
      - Create an **HTTP Header Auth** credential.
      - Configure header to include your Buffer access token, typically:
        - Header name: `Authorization`
        - Header value: `Bearer <YOUR_BUFFER_TOKEN>`
      - (The credential name in the imported workflow is “Stripe Header Auth”, but it should contain Buffer auth.)

4) **Add nodes and configure them (in order)**

5) **Node 1: Webhook - Receive Asset (Webhook)**
   - Add **Webhook** node.
   - Set:
     - HTTP Method: `POST`
     - Path: `social-asset-upload`
     - Response Mode: `Using 'Respond to Webhook' node`
     - Allowed Origins: `*` (optional)
   - Expected request payload fields (JSON body):
     - `fileUrl` (optional) OR `filename` (required if uploading binary)
     - `platform` (optional; defaults to instagram)
     - `tone`, `brand`, `campaignContext` (optional)
     - `hashtags` (optional boolean; `false` disables)
     - `scheduleTime` (optional ISO timestamp)

6) **Node 2: Validate & Enrich Payload (Code)**
   - Add **Code** node and paste logic that:
     - Reads `body` from webhook
     - Validates `filename` or `fileUrl`
     - Validates platform in `[instagram,twitter,linkedin,facebook]`
     - Sets `charLimit`, `cleanName`, `contentType`, etc.
   - Connect: **Webhook → Validate & Enrich Payload**

7) **Node 3: Has Remote URL? (IF)**
   - Add **IF** node:
     - Condition: String → `notEmpty` on `{{$json.fileUrl}}`
   - Connect: **Validate & Enrich Payload → Has Remote URL?**

8) **Node 4a: Upload to URL - From Remote URL (UploadToURL community node)**
   - Add **UploadToURL** node.
   - Keep default operation (upload from URL).
   - Select **UploadToURL credential**.
   - Connect: **Has Remote URL? (true) → Upload to URL - From Remote URL**

9) **Node 4b: Upload to URL - From Binary (UploadToURL community node)**
   - Add another **UploadToURL** node.
   - Set operation to **uploadFile**.
   - Select **UploadToURL credential**.
   - Connect: **Has Remote URL? (false) → Upload to URL - From Binary**

10) **Node 5: Extract Clean URL (Code)**
   - Add **Code** node that:
     - Extracts a usable URL from possible response fields (`url`, `link`, `data.url`, etc.)
     - Merges it with the metadata from “Validate & Enrich Payload” using `$()`
     - Throws if URL not found
   - Connect:
     - **Upload to URL - From Remote URL → Extract Clean URL**
     - **Upload to URL - From Binary → Extract Clean URL**

11) **Node 6: OpenAI - Generate Caption (OpenAI / LangChain)**
   - Add **OpenAI (LangChain)** node.
   - Configure:
     - Model: `gpt-4.1-mini`
     - Temperature: `0.75`
     - Max tokens: `800`
     - Messages:
       - System: enforce “valid JSON object only”
       - User: include platform, cleanName, contentType, tone, brand, campaignContext, charLimit, includeHashtags, and specify required JSON keys
   - Select **OpenAI credential**.
   - Connect: **Extract Clean URL → OpenAI - Generate Caption**

12) **Node 7: Assemble Post Payload (Code)**
   - Add **Code** node that:
     - Parses OpenAI JSON from common response shapes
     - Builds `fullCaption` + hashtags (optional)
     - Computes `scheduleAt` from request or `bestPostingTimeUTC`
     - Outputs `assetUrl`, `platform`, `fullCaption`, `altText`, `scheduleAt`, etc.
   - Connect: **OpenAI - Generate Caption → Assemble Post Payload**

13) **Node 8: Route by Platform (Switch)**
   - Add **Switch** node:
     - Rule 1: if `{{$json.platform}} == "twitter"` → output “Twitter/X”
     - Rule 2: if `{{$json.platform}} == "instagram"` → output “Instagram”
     - Rule 3: if `{{$json.platform}} == "linkedin"` → output “LinkedIn”
     - Fallback output: `extra` (optional, but recommended to connect to an error response)
   - Connect: **Assemble Post Payload → Route by Platform**

14) **Node 9a/9b/9c: Buffer scheduling (HTTP Request nodes)**
   Create three **HTTP Request** nodes with:
   - Method: `POST`
   - URL: `https://api.bufferapp.com/1/updates/create.json`
   - Authentication:
     - “Generic credential type” → choose **HTTP Header Auth** credential containing Buffer token
   - Send body: JSON (expression-built)

   **Twitter/X body fields:**
   - `text`: `{{$json.fullCaption}}`
   - `media.link`: `{{$json.assetUrl}}`
   - `media.description`: `{{$json.altText}}`
   - `scheduled_at`: `{{$json.scheduleAt}}`
   - `profile_ids`: `[{{$vars.BUFFER_PROFILE_TWITTER}}]`

   **Instagram differences:**
   - Add `media.photo`: `{{$json.contentType === 'image' ? $json.assetUrl : ''}}`
   - `profile_ids`: `[{{$vars.BUFFER_PROFILE_INSTAGRAM}}]`

   **LinkedIn differences:**
   - Add `media.title`: `{{$json.filename}}`
   - `profile_ids`: `[{{$vars.BUFFER_PROFILE_LINKEDIN}}]`

   Connect:
   - Switch “Twitter/X” → Twitter HTTP Request
   - Switch “Instagram” → Instagram HTTP Request
   - Switch “LinkedIn” → LinkedIn HTTP Request

15) **Node 10: Build Final Response (Code)**
   - Add **Code** node that:
     - Determines success from Buffer response
     - Extracts `scheduleId` from `update.id` or `data.id`
     - Returns unified fields: `success`, `message`, `assetUrl`, `platform`, `scheduledAt`, `caption`, `hashtags`, `altText`, `estimatedEngagement`, etc.
   - Connect each Buffer node → **Build Final Response**

16) **Node 11: Respond to Webhook (Respond to Webhook)**
   - Add **Respond to Webhook** node.
   - Configure:
     - Respond with: JSON
     - Body: `{{$json}}`
     - Status: 200
     - Header: `Content-Type: application/json`
   - Connect: **Build Final Response → Respond to Webhook**

17) **(Recommended) Fix the “facebook” mismatch**
   - Either:
     - Remove `facebook` from `allowedPlatforms`, **or**
     - Add a Switch rule + a Buffer node for Facebook, **or**
     - Connect fallback `extra` to an error response that returns “platform not implemented”.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI Social Media Auto-Scheduler via UploadToURL — workflow uses UploadToURL to host assets, OpenAI to generate copy, and Buffer to schedule posts. | From the “📋 Overview” sticky note |
| Setup requirements: install `n8n-nodes-uploadtourl`; add UploadToURL/OpenAI/Buffer credentials; set variables `BUFFER_PROFILE_TWITTER`, `BUFFER_PROFILE_INSTAGRAM`, `BUFFER_PROFILE_LINKEDIN`. | From the “📋 Overview” sticky note |
| Intake expects `fileUrl` or binary + `platform`, `tone`, `brand`, `campaignContext`, optional `scheduleTime`. | From “Section 1 — Intake” sticky note |
| UploadToURL response normalization is handled to accommodate field-name differences across API versions; errors include raw response for debugging. | From “Section 2 — Upload” sticky note |
| OpenAI response is intended to be `json_object`-like structured output to avoid free-text parsing; workflow still defensively JSON-parses and fails if invalid. | From “Section 3 — Caption” sticky note |
| To add more platforms, add a new Switch output and duplicate/adjust a Buffer node. | From “Section 4 — Scheduling” sticky note |