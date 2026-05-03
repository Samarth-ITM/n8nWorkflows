Extract YouTube video clips with Claude and Apify from natural language prompts

https://n8nworkflows.xyz/workflows/extract-youtube-video-clips-with-claude-and-apify-from-natural-language-prompts-13169


# Extract YouTube video clips with Claude and Apify from natural language prompts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Extract YouTube video clips with Claude and Apify from natural language prompts  
**Workflow name (n8n):** Extract Clips from YouTube Video with NLP Prompt  
**Purpose:** Collect a natural-language clipping request (YouTube URL + what to extract), fetch the video transcript via Apify, use Anthropic Claude (via LangChain Agent) to identify relevant clip segments, then call Apify to generate those clips, saving resulting links to Google Sheets and optionally downloading clip files to local storage.

### 1.1 Input Reception
Receives a user request through an n8n Form Trigger.

### 1.2 Configuration / Normalization
A Set node is intended to map form fields into a consistent internal configuration (e.g., YouTube URL, prompt, output settings).

### 1.3 Transcript Retrieval (Apify)
Calls Apify (via HTTP Request) to retrieve a transcript for the YouTube video.

### 1.4 AI Clip Discovery (Claude + Structured Output)
Uses a LangChain Agent with an Anthropic chat model and a Structured Output Parser to convert transcript + user intent into machine-readable clip requests (timestamps, titles, etc.).

### 1.5 Clip Creation + Persistence (Apify + Sheets)
Sends clip requests to Apify to render/export clips; stores returned links in Google Sheets.

### 1.6 Optional: Download & Local Save
Downloads produced clip files and saves them locally via a Code node.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception
**Overview:** Collects the user’s request (likely YouTube URL and a natural-language prompt describing desired clips).  
**Nodes involved:** `YouTube Clip Request Form`

#### Node: YouTube Clip Request Form
- **Type / role:** `n8n-nodes-base.formTrigger` — Entry point; exposes a hosted form and triggers workflow on submission.
- **Configuration choices:** Parameters are empty in JSON, meaning the form fields are not visible here. In practice, you would configure fields such as:
  - `youtubeUrl` (string)
  - `prompt` (string: “Find moments where…“)
  - optional: `maxClips`, `minDuration`, `maxDuration`, `language`, etc.
- **Key variables / expressions:** Not shown (no parameters). Output will be the submitted form payload.
- **Connections:**
  - **Output →** `Workflow Configuration`
- **Edge cases / failures:**
  - Missing/invalid URL input (needs validation in subsequent Set node)
  - Empty prompt → AI may return no clips or irrelevant clips

---

### Block 2 — Configuration / Normalization
**Overview:** Prepares a consistent configuration object for downstream nodes, typically mapping form fields into named JSON properties.  
**Nodes involved:** `Workflow Configuration`

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — Build/normalize fields used throughout the workflow.
- **Configuration choices:** Empty in JSON; intended usage typically includes:
  - Copy form fields into canonical names (e.g., `videoUrl`, `userPrompt`)
  - Define constants like Apify actor/task IDs, desired clip format, and spreadsheet info
- **Key variables / expressions:** Not provided; commonly references form trigger output like `{{$json.<fieldName>}}`.
- **Connections:**
  - **Input ←** `YouTube Clip Request Form`
  - **Output →** `Get Video Transcript from Apify`
- **Edge cases / failures:**
  - Expression errors if form fields are renamed or missing
  - Misconfigured constants (wrong actor ID, missing API token, etc.)

---

### Block 3 — Transcript Retrieval (Apify)
**Overview:** Calls Apify to retrieve a transcript for the requested YouTube video.  
**Nodes involved:** `Get Video Transcript from Apify`

#### Node: Get Video Transcript from Apify
- **Type / role:** `n8n-nodes-base.httpRequest` — Integrates with Apify REST API (likely “run actor / get dataset items” pattern).
- **Configuration choices:** Empty in JSON; typical implementation:
  - HTTP method: `POST` to run an actor/task (or `GET` if using an existing endpoint)
  - Auth: Apify API token (header `Authorization: Bearer <token>` or query `token=...`)
  - Body includes the YouTube URL and transcript options (language, timestamps, etc.)
- **Key variables / expressions:** Usually injects `{{$json.videoUrl}}` or similar from the Set node.
- **Connections:**
  - **Input ←** `Workflow Configuration`
  - **Output →** `Find Clips from Transcript`
- **Edge cases / failures:**
  - Apify auth failure (401/403)
  - Actor run timeout or long runtime
  - Transcript unavailable (no captions, blocked video, region restrictions)
  - Response shape mismatch (agent expects transcript text/segments but receives different JSON)

---

### Block 4 — AI Clip Discovery (Claude + Structured Output)
**Overview:** Uses Claude to analyze the transcript and return structured clip definitions (timestamps, titles, reasoning).  
**Nodes involved:** `Find Clips from Transcript`, `Anthropic Chat Model`, `Structured Output Parser`

#### Node: Find Clips from Transcript
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — Orchestrates LLM reasoning and tool-less extraction into structured output.
- **Configuration choices:** Empty in JSON; typical setup includes:
  - Prompt instructions combining:
    - user request (from form)
    - transcript (from Apify)
    - constraints (max clips, duration limits)
  - Agent configured to use:
    - **Language Model:** `Anthropic Chat Model`
    - **Output Parser:** `Structured Output Parser`
- **Key variables / expressions:** Usually references upstream fields, e.g.:
  - `{{$json.transcript}}` or `{{$json.items}}` (depending on Apify output)
  - `{{$json.userPrompt}}`
- **Connections:**
  - **Input ←** `Get Video Transcript from Apify`
  - **ai_languageModel ←** `Anthropic Chat Model`
  - **ai_outputParser ←** `Structured Output Parser`
  - **Output →** `Prepare Clip Requests`
- **Edge cases / failures:**
  - Transcript too large → token/context overflow (needs chunking or summarization)
  - Model returns output that doesn’t match parser schema → parser failure
  - Ambiguous prompt → unusable timestamps (e.g., missing start/end)
- **Version-specific notes:** Node typeVersion `3` (LangChain Agent). Ensure your n8n includes the `@n8n/n8n-nodes-langchain` package and compatible version.

#### Node: Anthropic Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` — Provides Claude chat model to the agent.
- **Configuration choices:** Empty in JSON; typical required config:
  - Credentials: Anthropic API key
  - Model: e.g., `claude-3-5-sonnet` / `claude-3-opus` depending on availability
  - Temperature (often low for extraction tasks)
- **Connections:**
  - **Output (ai_languageModel) →** `Find Clips from Transcript`
- **Edge cases / failures:**
  - Invalid API key / quota exceeded
  - Model deprecation/name mismatch
  - Rate limiting

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — Enforces a JSON schema-like structure for clips.
- **Configuration choices:** Empty in JSON; typically you define a schema such as:
  - `clips: [{ title, startTime, endTime, description, confidence }]`
- **Connections:**
  - **Output (ai_outputParser) →** `Find Clips from Transcript`
- **Edge cases / failures:**
  - Model returns invalid JSON or wrong fields
  - Timestamps not parseable (e.g., “around 5 minutes”)

---

### Block 5 — Prepare Clip Creation Requests
**Overview:** Converts the AI-produced structured clips into the exact payload required by Apify’s clip-generation actor/task.  
**Nodes involved:** `Prepare Clip Requests`

#### Node: Prepare Clip Requests
- **Type / role:** `n8n-nodes-base.set` — Maps AI output to Apify input format (e.g., array of clip specs).
- **Configuration choices:** Empty in JSON; typical fields set here:
  - `videoUrl` (original)
  - `clips` list with normalized timestamps (seconds) and metadata
  - output format parameters (mp4, resolution, etc.)
- **Connections:**
  - **Input ←** `Find Clips from Transcript`
  - **Output →** `Create Clips via Apify`
- **Edge cases / failures:**
  - Missing clip list (AI returned none)
  - Start/end times reversed or out of bounds
  - Too many clips for Apify limits (payload size)

---

### Block 6 — Create Clips (Apify) + Save Links (Google Sheets)
**Overview:** Sends clip requests to Apify to generate clips, then records resulting links in Google Sheets; also forwards output to optional download branch.  
**Nodes involved:** `Create Clips via Apify`, `Save Clip Links to Google Sheets`

#### Node: Create Clips via Apify
- **Type / role:** `n8n-nodes-base.httpRequest` — Calls Apify actor/task that creates the video clips.
- **Configuration choices:** Empty in JSON; typically:
  - POST to Apify run endpoint with clip specs
  - Wait-for-finish behavior or polling (depending on setup)
  - Extract resulting dataset items containing `clipUrl`/`downloadUrl`
- **Connections:**
  - **Input ←** `Prepare Clip Requests`
  - **Output →**
    - `Save Clip Links to Google Sheets`
    - `Download Clip Files`
- **Edge cases / failures:**
  - Apify actor errors (invalid timestamps, failed download, ffmpeg issues)
  - Long rendering time / timeout
  - Response doesn’t include stable URLs (signed URLs expiry)

#### Node: Save Clip Links to Google Sheets
- **Type / role:** `n8n-nodes-base.googleSheets` — Persists clip metadata/URLs for tracking and sharing.
- **Configuration choices:** Empty in JSON; typical:
  - OAuth2 credentials for Google
  - Spreadsheet ID + sheet name
  - Append rows with columns such as: video URL, prompt, clip title, start/end, clip URL
- **Connections:**
  - **Input ←** `Create Clips via Apify`
  - **Output →** none (terminal)
- **Edge cases / failures:**
  - Auth expired / insufficient permissions
  - Sheet/tab not found
  - Column mismatch if sheet structure changes
- **Version-specific notes:** Node typeVersion `4.7` (ensure n8n is recent enough for this version).

---

### Block 7 — Optional: Download Clip Files and Save Locally
**Overview:** Downloads the clip files from provided URLs and writes them to local disk on the n8n instance.  
**Nodes involved:** `Download Clip Files`, `Save Files Locally`

#### Node: Download Clip Files
- **Type / role:** `n8n-nodes-base.httpRequest` — Fetches binary content (video files).
- **Configuration choices:** Empty in JSON; typical:
  - URL expression from Apify output (e.g., `{{$json.downloadUrl}}`)
  - Response format: `File` / “Download” to binary property (e.g., `data`)
- **Connections:**
  - **Input ←** `Create Clips via Apify`
  - **Output →** `Save Files Locally`
- **Edge cases / failures:**
  - Large files → memory pressure
  - Expired signed URLs
  - 403/404 if clip not ready

#### Node: Save Files Locally
- **Type / role:** `n8n-nodes-base.code` — Writes the binary data to disk using Node.js file APIs (implementation not shown).
- **Configuration choices:** Empty in JSON; expected logic:
  - Read binary from `items[0].binary[...]`
  - Determine a filename (clip title + timestamps)
  - Write to a configured directory path
- **Connections:**
  - **Input ←** `Download Clip Files`
  - **Output →** none (terminal)
- **Edge cases / failures:**
  - n8n Cloud: local filesystem writes may not persist/allowed
  - Self-hosted: container permissions, missing directory
  - Filename sanitization issues (slashes, unicode)
- **Version-specific notes:** Code node typeVersion `2` (uses the newer JS sandbox runtime behavior).

---

### Block 8 — Sticky Notes (Documentation placeholders)
**Overview:** The workflow contains multiple sticky notes, but their content is empty in the provided JSON.  
**Nodes involved:** `Setup Instructions`, `Sticky Note`, `Sticky Note1`, `Sticky Note2`, `Sticky Note3`, `Sticky Note4`

#### Sticky note nodes
- **Type / role:** `n8n-nodes-base.stickyNote` — Canvas documentation only; no execution impact.
- **Configuration choices:** All have empty `content`.
- **Edge cases / failures:** None (non-executing).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| YouTube Clip Request Form | n8n-nodes-base.formTrigger | Entry point (collect user request) | — | Workflow Configuration |  |
| Workflow Configuration | n8n-nodes-base.set | Normalize inputs / constants | YouTube Clip Request Form | Get Video Transcript from Apify |  |
| Get Video Transcript from Apify | n8n-nodes-base.httpRequest | Fetch transcript via Apify API | Workflow Configuration | Find Clips from Transcript |  |
| Find Clips from Transcript | @n8n/n8n-nodes-langchain.agent | LLM agent extracts clip timestamps | Get Video Transcript from Apify; (AI) Anthropic Chat Model; (AI) Structured Output Parser | Prepare Clip Requests |  |
| Prepare Clip Requests | n8n-nodes-base.set | Map AI output to Apify clip payload | Find Clips from Transcript | Create Clips via Apify |  |
| Create Clips via Apify | n8n-nodes-base.httpRequest | Generate clips via Apify API | Prepare Clip Requests | Save Clip Links to Google Sheets; Download Clip Files |  |
| Save Clip Links to Google Sheets | n8n-nodes-base.googleSheets | Persist clip URLs/metadata | Create Clips via Apify | — |  |
| Download Clip Files | n8n-nodes-base.httpRequest | Download rendered clip binaries | Create Clips via Apify | Save Files Locally |  |
| Save Files Locally | n8n-nodes-base.code | Write clip files to disk | Download Clip Files | — |  |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude model provider | — | Find Clips from Transcript (ai_languageModel) |  |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured clip schema | — | Find Clips from Transcript (ai_outputParser) |  |
| Setup Instructions | n8n-nodes-base.stickyNote | Canvas note (empty) | — | — |  |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas note (empty) | — | — |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas note (empty) | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas note (empty) | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas note (empty) | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas note (empty) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: **“Extract Clips from YouTube Video with NLP Prompt”**.

2. **Add Form Trigger**
   - Node: **Form Trigger** (`YouTube Clip Request Form`)
   - Create fields (suggested):
     - `youtubeUrl` (required)
     - `prompt` (required)
     - `maxClips` (optional number, default 3–5)
     - `minDurationSec` / `maxDurationSec` (optional)
   - Save and copy the form URL for testing.

3. **Add Set node for configuration**
   - Node: **Set** (`Workflow Configuration`)
   - Enable “Keep Only Set” (recommended) and set fields such as:
     - `videoUrl` = `{{$json.youtubeUrl}}`
     - `userPrompt` = `{{$json.prompt}}`
     - `maxClips` = `{{$json.maxClips || 5}}`
     - (Optional) `apifyTranscriptActorId`, `apifyClipActorId`
   - Connect: **Form Trigger → Workflow Configuration**

4. **Add HTTP Request to Apify for transcript**
   - Node: **HTTP Request** (`Get Video Transcript from Apify`)
   - Configure (one common Apify pattern):
     - Method: `POST`
     - URL: `https://api.apify.com/v2/acts/<ACTOR_OR_TASK_ID>/runs?waitForFinish=120`
     - Authentication: add Apify token
       - Either as query: `&token=<APIFY_TOKEN>` (not ideal)
       - Or header: `Authorization: Bearer <APIFY_TOKEN>`
     - Body (JSON): include `videoUrl: {{$json.videoUrl}}` and transcript options your actor expects
   - Connect: **Workflow Configuration → Get Video Transcript from Apify**

5. **Add Anthropic Chat Model**
   - Node: **Anthropic Chat Model** (`Anthropic Chat Model`)
   - Create credentials: **Anthropic API Key**
   - Select model (e.g., a Claude 3.x/3.5 model available in your account)
   - Temperature: low (e.g., 0–0.3) for consistent extraction

6. **Add Structured Output Parser**
   - Node: **Structured Output Parser** (`Structured Output Parser`)
   - Define an output schema for clips, for example:
     - `clips` (array)
       - `title` (string)
       - `startSec` (number)
       - `endSec` (number)
       - `reason` (string, optional)
   - Ensure schema matches what you’ll map into Apify.

7. **Add LangChain Agent to find clips**
   - Node: **AI Agent** (`Find Clips from Transcript`)
   - Configure prompt/instructions to include:
     - The user request: `{{$json.userPrompt}}`
     - The transcript text/segments from the Apify response
     - Constraints: max clips, duration bounds, “return JSON matching the schema”
   - Connect AI inputs:
     - **Anthropic Chat Model → Find Clips from Transcript** (ai_languageModel)
     - **Structured Output Parser → Find Clips from Transcript** (ai_outputParser)
   - Connect main flow:
     - **Get Video Transcript from Apify → Find Clips from Transcript**

8. **Add Set node to prepare clip creation payload**
   - Node: **Set** (`Prepare Clip Requests`)
   - Map agent output to Apify clip actor input, e.g.:
     - `videoUrl` = `{{$json.videoUrl}}` (ensure it’s still present; otherwise pass it through from earlier)
     - `clips` = `{{$json.clips}}`
     - Additional fields required by the Apify clip actor (format, resolution, etc.)
   - Connect: **Find Clips from Transcript → Prepare Clip Requests**

9. **Add HTTP Request to Apify for clip creation**
   - Node: **HTTP Request** (`Create Clips via Apify`)
   - Configure similarly to transcript step, but targeting the clip-generation actor/task:
     - POST run endpoint with `clips` array and `videoUrl`
     - Ensure response returns clip URLs (or dataset items you can fetch)
   - Connect: **Prepare Clip Requests → Create Clips via Apify**

10. **Add Google Sheets node to save links**
   - Node: **Google Sheets** (`Save Clip Links to Google Sheets`)
   - Credentials: Google OAuth2 (Sheets scope)
   - Operation: “Append” (typical)
   - Spreadsheet: choose or paste ID
   - Map columns to Apify output fields (e.g., title/start/end/url)
   - Connect: **Create Clips via Apify → Save Clip Links to Google Sheets**

11. **(Optional) Download clip binaries**
   - Node: **HTTP Request** (`Download Clip Files`)
     - Set URL to the clip download URL from Apify output
     - Enable “Download” / return binary
   - Connect: **Create Clips via Apify → Download Clip Files**

12. **(Optional) Save binaries locally**
   - Node: **Code** (`Save Files Locally`)
   - Implement file writing (self-hosted recommended). Ensure:
     - You read from the correct binary property
     - You write into a mounted volume (Docker) or writable path
   - Connect: **Download Clip Files → Save Files Locally**

13. **Run an end-to-end test**
   - Submit the form with a known YouTube video and a clear prompt.
   - Verify:
     - Transcript retrieved
     - Agent outputs valid structured clips
     - Apify generates clips
     - Sheet rows append correctly
     - (Optional) local files are written

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but contain no text in the provided workflow JSON. | Canvas documentation placeholders (`Setup Instructions`, `Sticky Note*`). |
| Local file saving is environment-dependent (n8n Cloud vs self-hosted). | Applies to `Save Files Locally` and filesystem permissions/volumes. |
| Transcript size can exceed LLM context limits; chunking may be required. | Applies to `Find Clips from Transcript` + transcript retrieval strategy. |