Generate AI camera moves with Seedance and build a previs review board

https://n8nworkflows.xyz/workflows/generate-ai-camera-moves-with-seedance-and-build-a-previs-review-board-14891


# Generate AI camera moves with Seedance and build a previs review board

### 1. Workflow Overview

This workflow implements an **AI-powered virtual cinematography previs pipeline** for VFX/film production. A supervisor fills out a structured web form describing a shot — including script snippet, lens/camera specs, a plate image URL, and move complexity. GPT-4o then translates that creative intent into three distinct camera choreography briefs. Each brief is sent to the **Seedance** video generation API (with the plate image attached as a visual reference), and the resulting short videos are polled until rendering completes. Once all three options are ready, key frames are tagged, a comprehensive previs board package is compiled, and the results are simultaneously published to **Slack**, **Jira**, **ClickUp**, and **Telegram** for supervisor review. Rendered videos are also archived to **Google Drive** as lighting references.

**Logical Blocks:**

| Block | Name | Purpose |
|-------|------|---------|
| 1 | Brief Intake & AI Choreography | Collects the shot brief via a web form, normalizes fields, and calls GPT-4o to generate three camera choreography options |
| 2 | Seedance Video Generation & Polling | Builds Seedance API requests for each option, submits them, and polls every 20 seconds until rendering succeeds |
| 3 | Key Frame Extraction & Board Assembly | Tags key frames at three timecodes per move, then compiles all options into a single structured previs board package |
| 4 | Supervisor Delivery | Publishes the previs board to Slack, Jira, ClickUp, and Telegram; archives each rendered video to Google Drive |
| 5 | Error Handling | Catches AI agent failures and general workflow errors, alerting the team via Telegram or Slack |

---

### 2. Block-by-Block Analysis

---

#### Block 1 — Brief Intake & AI Choreography

**Overview:** This block is the entry point. A production supervisor submits a shot brief through an n8n-hosted web form. The form data is normalized into clean fields (including a complexity mapping), then forwarded to an AI Agent powered by Azure OpenAI GPT-4o-mini. The agent generates three camera choreography options as structured JSON. If the agent fails, a Telegram alert is sent and the flow retries from form extraction.

**Nodes Involved:**
- Form: Previs Brief Input1
- Extract & Map Form Fields
- AI Agent: Generate Camera Options
- Azure OpenAI: GPT-4o Mini
- Parse AI Response → Seedance Items
- Telegram: Alert on AI Agent Failure

---

**Form: Previs Brief Input1**

- **Type:** `n8n-nodes-base.formTrigger` (v2.2)
- **Technical Role:** Webhook-based form trigger; auto-generates a public URL where supervisors fill out the shot brief.
- **Configuration:**
  - Title: "AI Previs — Virtual Cinematography Request"
  - Description: Instructs the user to describe a complex shot for AI translation into camera choreography.
  - Fields (all required unless noted):
    1. **Shot Code** — text field, placeholder `SQ080_SH010`
    2. **Script Snippet** — textarea, placeholder describes camera movement narration
    3. **Lens / Camera Specs** — text field, placeholder `14mm anamorphic, drone rig, 240fps slowmo`
    4. **Plate Image URL** — text field, placeholder `https://your-server.com/location_plate.jpg`
    5. **Move Complexity** — dropdown with four options: "Simple — single axis move", "Medium — multi-axis choreography", "Complex — impossible / virtual camera", "Hero — signature shot / oner"
    6. **Supervisor Email** — email field, placeholder `user@example.com`
- **Input:** None (entry point, triggered by HTTP POST to the webhook URL).
- **Output:** A single item containing all form field values keyed by their labels.
- **Edge Cases:**
  - Webhook URL must be reachable from the supervisor's browser.
  - If the n8n instance restarts, the webhook ID (`33e2daee-8506-4c17-aa56-1d76660bdc4d`) is regenerated; the old URL becomes invalid.
  - Required field validation is handled client-side by the form; missing required fields prevent submission.

---

**Extract & Map Form Fields**

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** JavaScript code node that normalizes raw form field names into a clean, predictable schema and maps the complexity dropdown value to a short code.
- **Configuration:**
  - Maps form fields to output fields: `shotCode`, `scriptSnippet`, `lensSpecs`, `plateImageUrl`, `complexity`, `supervisorEmail`
  - Derives `sequenceCode` by splitting `shotCode` on `_` and taking the first segment.
  - Adds `requestTimestamp` as an ISO 8601 string.
  - Complexity mapping:
    - "Simple — single axis move" → `simple`
    - "Medium — multi-axis choreography" → `medium`
    - "Complex — impossible / virtual camera" → `complex`
    - "Hero — signature shot / oner" → `hero`
    - Fallback: `medium`
  - Default for `lensSpecs` is `"unspecified"` if not provided.
- **Input:** Single item from the form trigger.
- **Output:** Single item with normalized field structure.
- **Edge Cases:**
  - If `Shot Code` is empty after trim, `shotCode` and `sequenceCode` will be empty strings.
  - If the complexity dropdown value is not in the map (e.g., form schema changed), it defaults to `medium`.

---

**AI Agent: Generate Camera Options**

- **Type:** `@n8n/n8n-nodes-langchain.agent` (v2.1)
- **Technical Role:** LangChain Agent node that constructs a prompt from the normalized form data, sends it to the connected LLM, and returns the AI's response.
- **Configuration:**
  - Prompt type: `define` (explicit prompt template)
  - Prompt template: Injects all six form fields and instructs the model to return exactly 3 camera choreography options as a JSON object with fields: `shotIntent`, and an array `moves` each containing `moveId`, `moveName`, `moveIcon`, `moveStyle`, `supervisorNote`, `seedancePrompt`.
  - System message: "You are a virtual cinematography expert for a VFX production pipeline. You translate director intent and technical shot briefs into precise camera choreography options. Return ONLY valid raw JSON — no markdown, no backticks, no preamble."
  - Error handling: `continueErrorOutput` — errors are routed to the error output instead of halting the workflow.
- **Input:** Single item from Extract & Map Form Fields.
- **Output (success):** AI response text on the main output.
- **Output (error):** Error object routed to the error output, which connects to the Telegram alert node.
- **Edge Cases:**
  - The LLM may return markdown-wrapped JSON despite the instruction; downstream parsing handles this.
  - Azure OpenAI rate limits or credential misconfiguration will trigger the error output.
  - Timeout on long prompts (complex shots with verbose script snippets).

---

**Azure OpenAI: GPT-4o Mini**

- **Type:** `@n8n/n8n-nodes-langchain.lmChatAzureOpenAi` (v1)
- **Technical Role:** Language model provider connected to the AI Agent via the `ai_languageModel` connection type.
- **Configuration:**
  - Model: `gpt-4o-mini`
  - Options: default (no temperature, max_tokens, or other overrides specified)
- **Input:** Connected as `ai_languageModel` to the AI Agent (not a data-flow input).
- **Output:** Provides chat completion responses to the Agent node.
- **Edge Cases:**
  - The deployment name in the Azure credential must match `gpt-4o-mini`; if your Azure deployment uses a different name, update this field.
  - Azure OpenAI credentials must be configured in n8n with the correct endpoint, API key, and deployment name.
  - No custom temperature is set — the Azure default applies (typically 1.0).

---

**Parse AI Response → Seedance Items**

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** JavaScript code node that extracts raw JSON from the AI response, handles markdown/backtick contamination, and fans out each of the 3 camera moves into separate items for parallel Seedance processing.
- **Configuration:**
  - Attempts to read AI output from `$input.first().json.text`, `.output`, or `.content`.
  - Retrieves the original form data from the `Extract & Map Form Fields` node via `$('Extract & Map Form Fields').first().json`.
  - Cleans the response: strips `` ```json `` and `` ``` `` markers, replaces newlines with spaces, and attempts `JSON.parse`.
  - If parsing fails, attempts a regex fallback to extract the first `{...}` block.
  - Maps each move to an output item with: all form data fields + `moveId`, `moveName`, `moveIcon`, `moveStyle`, `supervisorNote`, `shotIntent`, `safePrompt`, `totalMoves`.
  - `safePrompt` is constructed as: the Seedance prompt + shot code + lens specs + `--duration 5 --camerafixed false`, JSON-stringified then trimmed of surrounding quotes.
- **Input:** Single item from the AI Agent's main (success) output.
- **Output:** Up to 3 items (one per camera move), each carrying full form context plus move-specific AI-generated data.
- **Edge Cases:**
  - If the AI returns no valid JSON at all, the node throws an error with a truncated preview of the raw output.
  - If fewer than 3 moves are returned, fewer items are emitted; downstream nodes must handle variable item counts.
  - The regex fallback is greedy (`[\s\S]*`) and could capture too much if the AI output contains multiple JSON objects.

---

**Telegram: Alert on AI Agent Failure**

- **Type:** `n8n-nodes-base.telegram` (v1.1)
- **Technical Role:** Sends a Telegram notification when the AI Agent encounters an error.
- **Configuration:**
  - Message: "⚠️ AI Previs: AI Agent Error — The camera choreography agent failed to return valid JSON. The workflow has retried from form extraction. Time: {ISO timestamp}"
  - Chat ID: `YOUR_TELEGRAM_CHAT_ID` (placeholder — must be replaced)
  - Parse mode: Markdown
- **Input:** Error output from the AI Agent node.
- **Output:** Routes back to **Extract & Map Form Fields** for an automatic retry (the form data is reprocessed).
- **Edge Cases:**
  - If Telegram credentials are misconfigured, this alert will also fail, potentially causing a silent retry loop.
  - The retry loop (Telegram Alert → Extract & Map → AI Agent) has no max-retry guard — a persistent AI failure will loop indefinitely.

---

#### Block 2 — Seedance Video Generation & Polling

**Overview:** For each camera move option, this block constructs a Seedance API request (with the plate image as visual reference), submits it, captures the job ID, then polls the task status every 20 seconds until the render succeeds. The polling loop uses a Wait node and an If node to implement retry logic.

**Nodes Involved:**
- Build Seedance API Request
- Seedance: Submit Camera Move Job
- Store Job ID + Move Metadata
- Poll: Check Move Render Status
- Move Render Complete?
- Wait 20s Before Retry

---

**Build Seedance API Request**

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** JavaScript code node that assembles the Seedance API request body from the move data.
- **Configuration:**
  - Constructs a JSON body with:
    - `model`: `"seedance-1-5-pro-251215"`
    - `content`: array of two items — a text prompt (`safePrompt`) and an image URL (`plateImageUrl`) as `image_url` type
    - `generate_audio`: `false`
    - `ratio`: `"16:9"`
    - `duration`: `5`
    - `watermark`: `false`
  - Stringifies the body and adds it as `requestBody` alongside all existing fields.
- **Input:** One item per camera move (from Parse AI Response).
- **Output:** Same item with an added `requestBody` field (JSON string).
- **Edge Cases:**
  - If `plateImageUrl` is empty or invalid, the Seedance API may reject the request.
  - The model identifier `seedance-1-5-pro-251215` is hardcoded; if Seedance updates model names, this must be updated.

---

**Seedance: Submit Camera Move Job**

- **Type:** `n8n-nodes-base.httpRequest` (v4.3)
- **Technical Role:** Sends an HTTP POST request to the Seedance/BytePlus API to start a video generation task.
- **Configuration:**
  - URL: `https://ark.ap-southeast.bytepluses.com/api/v3/contents/generations/tasks`
  - Method: `POST`
  - Body: JSON, parsed from the `requestBody` expression
  - Headers:
    - `Authorization`: `Bearer YOUR_TOKEN_HERE` (placeholder — must be replaced with actual Seedance API key; should be stored as an HTTP Header Auth credential)
    - `Content-Type`: `application/json`
- **Input:** Items from Build Seedance API Request.
- **Output:** API response containing at minimum an `id` field (the task/job ID).
- **Edge Cases:**
  - 401 Unauthorized if the bearer token is invalid or expired.
  - 429 Rate limiting if too many jobs are submitted simultaneously.
  - Network timeout if the Seedance endpoint is slow to respond.
  - The hardcoded token `YOUR_TOKEN_HERE` must be replaced before use — never commit real tokens to the workflow definition.

---

**Store Job ID + Move Metadata**

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Merges the Seedance API response (specifically the job `id`) with the original move metadata for downstream use.
- **Configuration:**
  - Reads the HTTP result from `$input.first().json`.
  - Reads the original move data from the `Build Seedance API Request` node via `$('Build Seedance API Request').first().json`.
  - Returns a single item combining all fields, adding `id` from the API response.
- **Input:** Single item from Seedance: Submit Camera Move Job.
- **Output:** Single item with all move metadata + `id` (Seedance job ID).
- **Edge Cases:**
  - If the Seedance API response does not contain an `id` field, the node will set `id` to `undefined`, causing the polling URL to be malformed.

---

**Poll: Check Move Render Status**

- **Type:** `n8n-nodes-base.httpRequest` (v4.3)
- **Technical Role:** Sends a GET request to the Seedance API to check whether a video generation task has completed.
- **Configuration:**
  - URL: `https://ark.ap-southeast.bytepluses.com/api/v3/contents/generations/tasks/{{ $json.id }}` (dynamic, uses the job ID)
  - Method: `GET`
  - Headers:
    - `Authorization`: `Bearer YOUR_TOKEN_HERE` (same placeholder as submit node)
- **Input:** Items from Store Job ID + Move Metadata, or from Wait 20s Before Retry (loop).
- **Output:** API response containing `status` field and, on success, `content.video_url`.
- **Edge Cases:**
  - If the job ID is invalid, the API may return a 404.
  - The `status` field may have values other than `succeeded` and `failed` (e.g., `processing`, `queued`, `failed`); only `succeeded` breaks the loop.

---

**Move Render Complete?**

- **Type:** `n8n-nodes-base.if` (v2)
- **Technical Role:** Conditional node that checks whether the Seedance render has completed successfully.
- **Configuration:**
  - Condition: `$json.status` equals `"succeeded"` (string, strict type validation, case-insensitive)
  - True output → Collect Move + Tag Key Frames
  - False output → Wait 20s Before Retry
- **Input:** Single item from Poll: Check Move Render Status.
- **Output (true):** Item passes through to Block 3.
- **Output (false):** Item passes to the Wait node for retry.
- **Edge Cases:**
  - If the Seedance task status is `"failed"`, this node will route to the retry loop, which will poll forever on a permanently failed job. There is no maximum retry count or failure detection.
  - If the status field is missing entirely, the condition will evaluate to false, triggering unnecessary retries.

---

**Wait 20s Before Retry**

- **Type:** `n8n-nodes-base.wait` (v1.1)
- **Technical Role:** Pauses execution for 20 seconds before re-polling the Seedance API.
- **Configuration:**
  - Wait amount: 20 seconds
  - Webhook ID: `previs-wait-001`
- **Input:** False branch from Move Render Complete?
- **Output:** Loops back to Poll: Check Move Render Status.
- **Edge Cases:**
  - If the n8n instance restarts during the wait, the webhook-based resumption may fail, causing the execution to hang or be lost.
  - No exponential backoff — every retry is 20 seconds regardless of how many have occurred.

---

#### Block 3 — Key Frame Extraction & Board Assembly

**Overview:** Once a render completes, this block extracts the video URL, tags key frames at three predefined timecodes (opening, peak, landing), and compiles all three camera options into a single previs board package. It also downloads the rendered video and uploads it to Google Drive as a lighting reference.

**Nodes Involved:**
- Collect Move + Tag Key Frames
- Compile Previs Board Package1
- Download Lighting Reference Video
- Google Drive: Archive Lighting Ref

---

**Collect Move + Tag Key Frames**

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Processes the polling result, extracts the video URL, and adds three key frame metadata entries.
- **Configuration:**
  - Reads the poll result from `$input.first().json` and the original move data from the `Store Job ID + Move Metadata` node.
  - Extracts `videoUrl` from `pollResult.content.video_url`; falls back to a "Not found" message with the job ID.
  - Defines three key frames:
    1. Frame 1, timecode `00:00:00:01` — "Opening frame — camera start position"
    2. Frame 48, timecode `00:00:02:00` — "Peak move — maximum camera energy"
    3. Frame 120, timecode `00:00:05:00` — "Landing frame — final composition"
  - Returns a single item with all move metadata, video URL, job ID, resolution, duration, key frames, and a timestamp.
- **Input:** Single item from the true branch of Move Render Complete?
- **Output:** Single enriched item per move.
- **Edge Cases:**
  - The video URL extraction assumes a specific response schema (`content.video_url`); if Seedance changes its API response structure, this will fail silently (returns "Not found" string instead of raising an error).
  - Key frame positions are hardcoded and assume a 5-second, 24fps video.

---

**Compile Previs Board Package1**

- **Type:** `n8n-nodes-base.code` (v2)
- **Technical Role:** Aggregates all camera move items (there should be 3) into a single consolidated previs board package, generating Slack-formatted move cards, Jira-formatted descriptions, and an HTML lookbook table.
- **Configuration:**
  - Collects all input items via `$input.all().map(i => i.json)`.
  - Generates `moveCards`: Slack-formatted markdown with option letters (A, B, C), move icons, names, styles, supervisor notes, video links, and key frame labels.
  - Generates `jiraDesc`: Jira-formatted text with option letters, names, supervisor notes, and video URLs.
  - Generates `lookbookHtml`: A full HTML document with a table of options and a key frame guide list.
  - Returns a single item containing all shared fields (from the first move), `allMoves` (array of all move objects), `moveCards`, `jiraDesc`, `lookbookHtml`, `totalMoves`, and a `generatedAt` timestamp.
- **Input:** All items from Collect Move + Tag Key Frames (expects 3 items, one per camera move).
- **Output:** Single aggregated item.
- **Edge Cases:**
  - If fewer than 3 moves reach this node (due to a Seedance failure), the board will still compile with whatever items are available.
  - The HTML lookbook uses basic `<table>` markup — no CSS is included; rendering depends on the consumer.

---

**Download Lighting Reference Video**

- **Type:** `n8n-nodes-base.httpRequest` (v4.3)
- **Technical Role:** Downloads the rendered video file from the Seedance CDN URL as a binary file for upload to Google Drive.
- **Configuration:**
  - URL: `={{ $json.videoUrl }}` (dynamic)
  - Response format: File (binary)
- **Input:** Items from Collect Move + Tag Key Frames.
- **Output:** Binary data (the MP4 video file) attached to the item.
- **Edge Cases:**
  - If `videoUrl` is the "Not found" fallback string, this request will fail with a DNS or HTTP error.
  - Large video files may exceed n8n's default binary size limits.
  - CDN URLs may be temporary and expire; this node should execute soon after render completion.

---

**Google Drive: Archive Lighting Ref**

- **Type:** `n8n-nodes-base.googleDrive` (v3)
- **Technical Role:** Uploads the downloaded video file to a designated Google Drive folder as a lighting reference archive.
- **Configuration:**
  - Operation: Upload file (implicit — node type defaults to upload)
  - File name: `{{ $json.shotCode }}_lighting_{{ $now.toFormat('yyyyMMdd_HHmmss') }}.mp4` (dynamic, includes shot code and timestamp)
  - Folder ID: `YOUR_GOOGLE_DRIVE_FOLDER_ID` (placeholder — must be replaced)
  - Drive: "My Drive"
- **Input:** Items with binary attachment from Download Lighting Reference Video.
- **Output:** Google Drive file metadata.
- **Edge Cases:**
  - Google Drive OAuth2 credential must be configured with write access to the target folder.
  - If the folder ID is invalid or the credential lacks permission, the upload will fail.
  - File name collisions are avoided by the timestamp suffix, but rapid re-runs within the same second could overwrite.

---

#### Block 4 — Supervisor Delivery

**Overview:** The compiled previs board package is published simultaneously to four channels: Slack (with A/B/C pick prompt), Jira (review task), ClickUp (production record), and Telegram (full formatted message to the supervisor). Each delivery node uses the aggregated board data.

**Nodes Involved:**
- Slack: Publish Previs Board1
- Jira: Create Previs Review Task
- ClickUp: Create Previs Production Record
- Telegram: Deliver Previs to Supervisor

---

**Slack: Publish Previs Board1**

- **Type:** `n8n-nodes-base.slack` (v2.3)
- **Technical Role:** Posts the previs board as a formatted Slack message to a specified channel.
- **Configuration:**
  - Operation: Post message
  - Authentication: OAuth2
  - Channel ID: `YOUR_SLACK_CHANNEL_ID` (placeholder — must be replaced)
  - Message: Formatted markdown including shot code, sequence, complexity, shot intent, script snippet, lens specs, all three camera option cards (moveCards), and a call-to-action for the supervisor to reply with A, B, or C.
- **Input:** Single item from Compile Previs Board Package1.
- **Output:** Slack API response.
- **Edge Cases:**
  - Slack OAuth2 token must have `chat:write` scope for the target channel.
  - Message may exceed Slack's 40,000 character limit if the script snippet is very long.
  - If the channel ID is invalid, the API will return a `channel_not_found` error.

---

**Jira: Create Previs Review Task**

- **Type:** `n8n-nodes-base.jira` (v1)
- **Technical Role:** Creates a Jira task for the previs review, with all shot metadata in the description.
- **Configuration:**
  - Operation: Create issue
  - Project ID: `YOUR_JIRA_PROJECT_ID` (placeholder)
  - Issue Type ID: `YOUR_JIRA_ISSUE_TYPE_ID` (placeholder, default name "Task")
  - Summary: `[AI Previs] {shotCode} – {totalMoves} Camera Options | Supervisor Pick Required`
  - Description: Structured text with shot intent, script, lens, complexity, camera options (`jiraDesc`), status, and timestamp.
- **Input:** Single item from Compile Previs Board Package1.
- **Output:** Jira issue metadata (including issue key).
- **Edge Cases:**
  - Jira Cloud credential must be configured with project write access.
  - If the project or issue type ID is invalid, creation will fail.
  - The `jiraDesc` field may contain Slack-specific markdown (bold with `*`) that Jira does not render — format mismatch.

---

**ClickUp: Create Previs Production Record**

- **Type:** `n8n-nodes-base.clickUp` (v1)
- **Technical Role:** Creates a ClickUp task as a production record for the previs shot.
- **Configuration:**
  - Operation: Create task
  - Team: `YOUR_CLICKUP_TEAM_ID` (placeholder)
  - Space: `YOUR_CLICKUP_SPACE_ID` (placeholder)
  - Folder: `YOUR_CLICKUP_FOLDER_ID` (placeholder)
  - List: `YOUR_CLICKUP_LIST_ID` (placeholder)
  - Task name: `[AI Previs] {shotCode}`
  - Content: Full structured description with shot metadata, camera options, and video links.
- **Input:** Single item from Compile Previs Board Package1.
- **Output:** ClickUp task metadata.
- **Edge Cases:**
  - ClickUp credential must have workspace-level access.
  - All four ID placeholders (team, space, folder, list) must be valid and correctly ordered in the hierarchy.
  - If any ID is wrong, the API will return a 400/404 error.

---

**Telegram: Deliver Previs to Supervisor**

- **Type:** `n8n-nodes-base.telegram` (v1.1)
- **Technical Role:** Sends the full previs board as a formatted Telegram message to the supervisor's chat.
- **Configuration:**
  - Message: Markdown-formatted text with shot code, sequence, complexity, shot intent, script, lens, supervisor email, all move details (icon, name, style, note, resolution, video URL), plate reference, and a call-to-action to reply with the chosen option.
  - Chat ID: `YOUR_TELEGRAM_CHAT_ID` (placeholder)
  - Parse mode: Markdown
- **Input:** Single item from Compile Previs Board Package1.
- **Output:** Telegram API response.
- **Edge Cases:**
  - Telegram bot must be configured and the chat ID must correspond to an active chat with the bot.
  - Markdown parsing may fail if the AI-generated content contains unescaped special characters (e.g., `_`, `*`, `[`).
  - Message may exceed Telegram's 4096 character limit for a single message; long script snippets could cause truncation.

---

#### Block 5 — Error Handling

**Overview:** This block catches two categories of errors: (1) AI Agent failures, which trigger a Telegram alert and automatic retry, and (2) general workflow errors, which trigger a Slack alert to the ops channel.

**Nodes Involved:**
- On Workflow Error
- Slack: Error Alert

(Note: Telegram: Alert on AI Agent Failure was documented in Block 1 as it is part of the AI agent retry loop.)

---

**On Workflow Error**

- **Type:** `n8n-nodes-base.errorTrigger` (v1)
- **Technical Role:** Workflow-level error trigger; fires whenever any node in the workflow fails (and the failure is not caught by a node's own error handling).
- **Configuration:** No parameters — listens for all workflow errors.
- **Input:** None (trigger node).
- **Output:** Error object containing `message` and other error metadata.
- **Edge Cases:**
  - Only catches errors that are not handled by individual node error outputs (e.g., the AI Agent's error output is handled separately and does not trigger this node).

---

**Slack: Error Alert**

- **Type:** `n8n-nodes-base.slack` (v2.3)
- **Technical Role:** Posts an error notification to a Slack ops channel.
- **Configuration:**
  - Authentication: OAuth2
  - Channel ID: `C0ANFAL4WJ2` (hardcoded — appears to be a real channel ID for "social")
  - Message: "❌ AI-Assisted Clean Plate & Object Removal — Error: {error message} — Time: {ISO timestamp}"
- **Input:** Error object from On Workflow Error.
- **Output:** Slack API response.
- **Edge Cases:**
  - The message text references "AI-Assisted Clean Plate & Object Removal" — this appears to be a copy-paste artifact from another workflow and does not match this pipeline's purpose. Consider updating the message text.
  - The hardcoded channel ID `C0ANFAL4WJ2` may not exist in all Slack workspaces.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview: AI Previs Pipeline | stickyNote | Documentation — overall pipeline overview and setup instructions | — | — | ## 🎬 AI Previs — Virtual Cinematography Pipeline — How it works: A supervisor fills out a web form describing a complex shot — script snippet, lens specs, and move complexity. GPT-4o translates that intent into three distinct camera choreography briefs, then Seedance generates a short video for each. All three options land in Slack, Jira, ClickUp, and Telegram so the supervisor can simply pick A, B, or C. The plate image you supply is attached to every Seedance generation as a visual reference, keeping all options grounded in the real location. Rendered videos are also archived to Google Drive for later use as lighting references. Setup steps: 1. Form Trigger — the webhook URL is auto-generated. Share it with your production team. 2. Azure OpenAI — connect your Azure OpenAI credential and confirm the deployment name matches gpt-4o-mini (or update it to your deployment). 3. Seedance API — replace the Authorization bearer token with your own key, stored as an HTTP Header Auth credential. 4. Slack — connect Slack via OAuth2 and update the channelId to your target channel. 5. Jira — connect your Jira Cloud credential; update project and issueType IDs to match your board. 6. ClickUp — connect your ClickUp credential and update team, space, folder, and list IDs. 7. Google Drive — connect via OAuth2 and update the folderId to your previs archive folder. 8. Telegram — connect your bot credential and confirm the chatId is correct. 9. Do a test run using a simple shot description before going live. |
| Section: Brief Intake & AI | stickyNote | Documentation — brief intake and AI choreography section | — | — | ## 📋 Brief Intake & AI Choreography — Collects the shot brief via a structured form, maps it to clean fields, then sends it to GPT-4o to generate three distinct camera choreography options — each with a Seedance-ready prompt, style description, and a note for the supervisor on when to choose it. |
| Section: Seedance Generation & Polling | stickyNote | Documentation — Seedance video generation and polling section | — | — | ## 🎥 Seedance Video Generation — Builds a Seedance API request for each camera option — plate image attached as visual reference — then submits and polls every 20 seconds until the render completes. Runs in parallel for all three options. |
| Section: Key Frame & Board Compile | stickyNote | Documentation — key frame extraction and board assembly section | — | — | ## 🗂️ Key Frame Extraction & Board Assembly — Once all moves are rendered, key frames are tagged at three timecodes (open, peak, landing). All options are compiled into a single previs board package — formatted for Slack, Jira, ClickUp, and Confluence — ready to send in one pass. |
| Section: Supervisor Delivery | stickyNote | Documentation — supervisor delivery section | — | — | ## 📤 Supervisor Delivery — Publishes the previs board simultaneously to Slack (with A/B/C pick prompt), Jira (review task), ClickUp (production record), and Telegram. The Google Drive step archives each rendered video as a lighting reference for the comp team. |
| Security: Credentials Note | stickyNote | Documentation — security and credential management best practices | — | — | ## 🔐 Credentials & Security — Use OAuth2 for Slack, Google Drive, and Jira. Store the Seedance and ClickUp API keys as named n8n credentials — never paste raw tokens into node parameters. Replace all personal IDs, folder paths, and chat IDs with your own values before sharing. |
| Section: Error Handler | stickyNote | Documentation — error handling section | — | — | ## ⚠️ Error Handler — Catches any failure across the entire workflow and immediately sends a Slack alert to the ops channel. Wire this to every sub-workflow or critical node to ensure no silent failures. |
| Form: Previs Brief Input1 | formTrigger | Entry point — web form for supervisor shot brief submission | — | Extract & Map Form Fields | ## 📋 Brief Intake & AI Choreography — Collects the shot brief via a structured form, maps it to clean fields, then sends it to GPT-4o to generate three distinct camera choreography options — each with a Seedance-ready prompt, style description, and a note for the supervisor on when to choose it. |
| Extract & Map Form Fields | code | Normalizes form data into clean schema with complexity mapping | Form: Previs Brief Input1, Telegram: Alert on AI Agent Failure | AI Agent: Generate Camera Options | ## 📋 Brief Intake & AI Choreography — Collects the shot brief via a structured form, maps it to clean fields, then sends it to GPT-4o to generate three distinct camera choreography options — each with a Seedance-ready prompt, style description, and a note for the supervisor on when to choose it. |
| AI Agent: Generate Camera Options | @n8n/n8n-nodes-langchain.agent | LangChain agent that generates 3 camera choreography options via GPT-4o | Extract & Map Form Fields | Parse AI Response → Seedance Items (success), Telegram: Alert on AI Agent Failure (error) | ## 📋 Brief Intake & AI Choreography — Collects the shot brief via a structured form, maps it to clean fields, then sends it to GPT-4o to generate three distinct camera choreography options — each with a Seedance-ready prompt, style description, and a note for the supervisor on when to choose it. |
| Azure OpenAI: GPT-4o Mini | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | LLM provider for the AI Agent (connected via ai_languageModel) | AI Agent: Generate Camera Options (ai_languageModel) | AI Agent: Generate Camera Options (ai_languageModel) | ## 📋 Brief Intake & AI Choreography — Collects the shot brief via a structured form, maps it to clean fields, then sends it to GPT-4o to generate three distinct camera choreography options — each with a Seedance-ready prompt, style description, and a note for the supervisor on when to choose it. |
| Parse AI Response → Seedance Items | code | Parses AI JSON output and fans out 3 camera move items | AI Agent: Generate Camera Options | Build Seedance API Request | ## 📋 Brief Intake & AI Choreography — Collects the shot brief via a structured form, maps it to clean fields, then sends it to GPT-4o to generate three distinct camera choreography options — each with a Seedance-ready prompt, style description, and a note for the supervisor on when to choose it. |
| Telegram: Alert on AI Agent Failure | telegram | Sends Telegram alert on AI Agent error, then retries from form extraction | AI Agent: Generate Camera Options (error) | Extract & Map Form Fields | ## 📋 Brief Intake & AI Choreography — Collects the shot brief via a structured form, maps it to clean fields, then sends it to GPT-4o to generate three distinct camera choreography options — each with a Seedance-ready prompt, style description, and a note for the supervisor on when to choose it. |
| Build Seedance API Request | code | Assembles the Seedance API request body with prompt and plate image | Parse AI Response → Seedance Items | Seedance: Submit Camera Move Job | ## 🎥 Seedance Video Generation — Builds a Seedance API request for each camera option — plate image attached as visual reference — then submits and polls every 20 seconds until the render completes. Runs in parallel for all three options. |
| Seedance: Submit Camera Move Job | httpRequest | Submits video generation task to Seedance API | Build Seedance API Request | Store Job ID + Move Metadata | ## 🎥 Seedance Video Generation — Builds a Seedance API request for each camera option — plate image attached as visual reference — then submits and polls every 20 seconds until the render completes. Runs in parallel for all three options. |
| Store Job ID + Move Metadata | code | Merges Seedance job ID with move metadata for downstream use | Seedance: Submit Camera Move Job | Poll: Check Move Render Status | ## 🎥 Seedance Video Generation — Builds a Seedance API request for each camera option — plate image attached as visual reference — then submits and polls every 20 seconds until the render completes. Runs in parallel for all three options. |
| Poll: Check Move Render Status | httpRequest | GET request to check Seedance task status | Store Job ID + Move Metadata, Wait 20s Before Retry | Move Render Complete? | ## 🎥 Seedance Video Generation — Builds a Seedance API request for each camera option — plate image attached as visual reference — then submits and polls every 20 seconds until the render completes. Runs in parallel for all three options. |
| Move Render Complete? | if | Checks if render status equals "succeeded" | Poll: Check Move Render Status | Collect Move + Tag Key Frames (true), Wait 20s Before Retry (false) | ## 🎥 Seedance Video Generation — Builds a Seedance API request for each camera option — plate image attached as visual reference — then submits and polls every 20 seconds until the render completes. Runs in parallel for all three options. |
| Wait 20s Before Retry | wait | 20-second pause before re-polling render status | Move Render Complete? (false) | Poll: Check Move Render Status | ## 🎥 Seedance Video Generation — Builds a Seedance API request for each camera option — plate image attached as visual reference — then submits and polls every 20 seconds until the render completes. Runs in parallel for all three options. |
| Collect Move + Tag Key Frames | code | Extracts video URL and tags key frames at three timecodes | Move Render Complete? (true) | Compile Previs Board Package1, Download Lighting Reference Video | ## 🗂️ Key Frame Extraction & Board Assembly — Once all moves are rendered, key frames are tagged at three timecodes (open, peak, landing). All options are compiled into a single previs board package — formatted for Slack, Jira, ClickUp, and Confluence — ready to send in one pass. |
| Compile Previs Board Package1 | code | Aggregates all moves into a single board package with Slack, Jira, and HTML formats | Collect Move + Tag Key Frames | Slack: Publish Previs Board1, Jira: Create Previs Review Task, ClickUp: Create Previs Production Record, Telegram: Deliver Previs to Supervisor | ## 🗂️ Key Frame Extraction & Board Assembly — Once all moves are rendered, key frames are tagged at three timecodes (open, peak, landing). All options are compiled into a single previs board package — formatted for Slack, Jira, ClickUp, and Confluence — ready to send in one pass. |
| Download Lighting Reference Video | httpRequest | Downloads the rendered video as a binary file | Collect Move + Tag Key Frames | Google Drive: Archive Lighting Ref | ## 🗂️ Key Frame Extraction & Board Assembly — Once all moves are rendered, key frames are tagged at three timecodes (open, peak, landing). All options are compiled into a single previs board package — formatted for Slack, Jira, ClickUp, and Confluence — ready to send in one pass. |
| Google Drive: Archive Lighting Ref | googleDrive | Uploads the video file to Google Drive for comp team reference | Download Lighting Reference Video | — | ## 🗂️ Key Frame Extraction & Board Assembly — Once all moves are rendered, key frames are tagged at three timecodes (open, peak, landing). All options are compiled into a single previs board package — formatted for Slack, Jira, ClickUp, and Confluence — ready to send in one pass. |
| Slack: Publish Previs Board1 | slack | Posts the previs board message with A/B/C pick prompt | Compile Previs Board Package1 | — | ## 📤 Supervisor Delivery — Publishes the previs board simultaneously to Slack (with A/B/C pick prompt), Jira (review task), ClickUp (production record), and Telegram. The Google Drive step archives each rendered video as a lighting reference for the comp team. |
| Jira: Create Previs Review Task | jira | Creates a Jira review task with shot metadata and camera options | Compile Previs Board Package1 | — | ## 📤 Supervisor Delivery — Publishes the previs board simultaneously to Slack (with A/B/C pick prompt), Jira (review task), ClickUp (production record), and Telegram. The Google Drive step archives each rendered video as a lighting reference for the comp team. |
| ClickUp: Create Previs Production Record | clickUp | Creates a ClickUp production record task | Compile Previs Board Package1 | — | ## 📤 Supervisor Delivery — Publishes the previs board simultaneously to Slack (with A/B/C pick prompt), Jira (review task), ClickUp (production record), and Telegram. The Google Drive step archives each rendered video as a lighting reference for the comp team. |
| Telegram: Deliver Previs to Supervisor | telegram | Sends the full previs board to the supervisor's Telegram chat | Compile Previs Board Package1 | — | ## 📤 Supervisor Delivery — Publishes the previs board simultaneously to Slack (with A/B/C pick prompt), Jira (review task), ClickUp (production record), and Telegram. The Google Drive step archives each rendered video as a lighting reference for the comp team. |
| On Workflow Error | errorTrigger | Catches unhandled workflow errors | — | Slack: Error Alert | ## ⚠️ Error Handler — Catches any failure across the entire workflow and immediately sends a Slack alert to the ops channel. Wire this to every sub-workflow or critical node to ensure no silent failures. |
| Slack: Error Alert | slack | Posts an error notification to the ops Slack channel | On Workflow Error | — | ## ⚠️ Error Handler — Catches any failure across the entire workflow and immediately sends a Slack alert to the ops channel. Wire this to every sub-workflow or critical node to ensure no silent failures. |

---

### 4. Reproducing the Workflow from Scratch

**Step 1 — Create the workflow and add the Form Trigger**

1. Create a new workflow named "Generate AI Camera Moves with Seedance and Build Previs Review Board".
2. Add a **Form Trigger** node.
   - Title: "AI Previs — Virtual Cinematography Request"
   - Description: "Describe your complex shot. AI will translate your intent into technical camera choreography briefs, then generate multiple video options for supervisor selection."
   - Add fields:
     - **Shot Code** — text input, required, placeholder `SQ080_SH010`
     - **Script Snippet** — textarea, required, placeholder describing camera movement
     - **Lens / Camera Specs** — text input, not required, placeholder `14mm anamorphic, drone rig, 240fps slowmo`
     - **Plate Image URL** — text input, required, placeholder `https://your-server.com/location_plate.jpg`
     - **Move Complexity** — dropdown, required, options: "Simple — single axis move", "Medium — multi-axis choreography", "Complex — impossible / virtual camera", "Hero — signature shot / oner"
     - **Supervisor Email** — email input, not required, placeholder `user@example.com`

**Step 2 — Add Extract & Map Form Fields (Code node)**

3. Add a **Code** node named "Extract & Map Form Fields".
   - Language: JavaScript
   - Paste the complexity mapping logic that converts raw form fields to `shotCode`, `scriptSnippet`, `lensSpecs`, `plateImageUrl`, `complexity`, `supervisorEmail`, `sequenceCode`, and `requestTimestamp`.
   - Connect: **Form Trigger** → **Extract & Map Form Fields**

**Step 3 — Configure Azure OpenAI credential and LLM node**

4. In n8n credential settings, create an **Azure OpenAI** credential with your Azure endpoint, API key, and deployment name.
5. Add an **Azure OpenAI Chat Model** node named "Azure OpenAI: GPT-4o Mini".
   - Model: `gpt-4o-mini`
   - Select the Azure OpenAI credential created above.

**Step 4 — Add the AI Agent node**

6. Add an **AI Agent** node named "AI Agent: Generate Camera Options".
   - Prompt type: Define
   - Prompt: Template injecting all form fields and requesting 3 camera choreography options as raw JSON (see workflow for full prompt text).
   - System message: "You are a virtual cinematography expert for a VFX production pipeline. You translate director intent and technical shot briefs into precise camera choreography options. Return ONLY valid raw JSON — no markdown, no backticks, no preamble."
   - Error handling: Continue on error (route to error output)
   - Connect: **Extract & Map Form Fields** → **AI Agent: Generate Camera Options** (main input)
   - Connect: **Azure OpenAI: GPT-4o Mini** → **AI Agent: Generate Camera Options** (ai_languageModel connection)

**Step 5 — Add Parse AI Response code node**

7. Add a **Code** node named "Parse AI Response → Seedance Items".
   - Language: JavaScript
   - Paste logic that reads AI output, cleans markdown contamination, parses JSON, and fans out each move as a separate item with `safePrompt` constructed.
   - Connect: **AI Agent: Generate Camera Options** (success output) → **Parse AI Response → Seedance Items**

**Step 6 — Add Telegram alert for AI Agent failure**

8. In n8n credential settings, create a **Telegram** bot credential (Bot API token from @BotFather).
9. Add a **Telegram** node named "Telegram: Alert on AI Agent Failure".
   - Chat ID: Your Telegram chat ID
   - Message: "⚠️ AI Previs: AI Agent Error — The camera choreography agent failed to return valid JSON. The workflow has retried from form extraction. Time: {ISO timestamp}"
   - Parse mode: Markdown
   - Connect: **AI Agent: Generate Camera Options** (error output) → **Telegram: Alert on AI Agent Failure**
   - Connect: **Telegram: Alert on AI Agent Failure** → **Extract & Map Form Fields** (to form a retry loop)

**Step 7 — Add Build Seedance API Request code node**

10. Add a **Code** node named "Build Seedance API Request".
    - Language: JavaScript
    - Paste logic that builds the Seedance request body with model, text prompt, image URL, and generation parameters.
    - Connect: **Parse AI Response → Seedance Items** → **Build Seedance API Request**

**Step 8 — Configure Seedance credential and HTTP Request node**

11. In n8n credential settings, create an **HTTP Header Auth** credential named "Seedance API Auth" with header name `Authorization` and value `Bearer YOUR_SEEDANCE_API_KEY`.
12. Add an **HTTP Request** node named "Seedance: Submit Camera Move Job".
    - Method: POST
    - URL: `https://ark.ap-southeast.bytepluses.com/api/v3/contents/generations/tasks`
    - Body: JSON, expression `{{ JSON.parse($json.requestBody) }}`
    - Headers: `Authorization` = `Bearer YOUR_TOKEN_HERE` (or select the credential from step 11), `Content-Type` = `application/json`
    - Connect: **Build Seedance API Request** → **Seedance: Submit Camera Move Job**

**Step 9 — Add Store Job ID code node**

13. Add a **Code** node named "Store Job ID + Move Metadata".
    - Language: JavaScript
    - Paste logic that merges `httpResult.id` with the move data from the Build Seedance API Request node.
    - Connect: **Seedance: Submit Camera Move Job** → **Store Job ID + Move Metadata**

**Step 10 — Add Poll HTTP Request node**

14. Add an **HTTP Request** node named "Poll: Check Move Render Status".
    - Method: GET
    - URL: `https://ark.ap-southeast.bytepluses.com/api/v3/contents/generations/tasks/{{ $json.id }}`
    - Headers: `Authorization` = `Bearer YOUR_TOKEN_HERE` (same as step 11)
    - Connect: **Store Job ID + Move Metadata** → **Poll: Check Move Render Status**

**Step 11 — Add If node for render completion check**

15. Add an **If** node named "Move Render Complete?".
    - Condition: `$json.status` equals `succeeded` (string, strict)
    - Connect: **Poll: Check Move Render Status** → **Move Render Complete?**
    - True output → **Collect Move + Tag Key Frames** (step 13)
    - False output → **Wait 20s Before Retry** (step 12)

**Step 12 — Add Wait node**

16. Add a **Wait** node named "Wait 20s Before Retry".
    - Wait amount: 20 seconds
    - Connect: **Wait 20s Before Retry** → **Poll: Check Move Render Status** (loop back to step 10)

**Step 13 — Add Collect Move + Tag Key Frames code node**

17. Add a **Code** node named "Collect Move + Tag Key Frames".
    - Language: JavaScript
    - Paste logic that extracts `videoUrl` from the poll result and adds three key frame metadata entries.
    - Connect: **Move Render Complete?** (true) → **Collect Move + Tag Key Frames**

**Step 14 — Add Compile Previs Board Package code node**

18. Add a **Code** node named "Compile Previs Board Package1".
    - Language: JavaScript
    - Paste logic that aggregates all move items, generates `moveCards` (Slack markdown), `jiraDesc` (Jira text), and `lookbookHtml` (HTML table).
    - Connect: **Collect Move + Tag Key Frames** → **Compile Previs Board Package1**

**Step 15 — Add Download Lighting Reference Video HTTP Request node**

19. Add an **HTTP Request** node named "Download Lighting Reference Video".
    - URL: `{{ $json.videoUrl }}`
    - Response format: File
    - Connect: **Collect Move + Tag Key Frames** → **Download Lighting Reference Video**

**Step 16 — Configure Google Drive credential and upload node**

20. In n8n credential settings, create a **Google Drive OAuth2** credential with write access.
21. Add a **Google Drive** node named "Google Drive: Archive Lighting Ref".
    - Operation: Upload
    - File name: `{{ $json.shotCode }}_lighting_{{ $now.toFormat('yyyyMMdd_HHmmss') }}.mp4`
    - Folder ID: Your Google Drive folder ID
    - Drive: My Drive
    - Select the Google Drive credential from step 20
    - Connect: **Download Lighting Reference Video** → **Google Drive: Archive Lighting Ref**

**Step 17 — Configure Slack credential and delivery node**

22. In n8n credential settings, create a **Slack OAuth2** credential with `chat:write` scope.
23. Add a **Slack** node named "Slack: Publish Previs Board1".
    - Operation: Post message
    - Authentication: OAuth2, select credential from step 22
    - Channel ID: Your Slack channel ID
    - Message: Formatted text with shot metadata and `$json.moveCards` plus the A/B/C pick prompt
    - Connect: **Compile Previs Board Package1** → **Slack: Publish Previs Board1**

**Step 18 — Configure Jira credential and task creation node**

24. In n8n credential settings, create a **Jira Cloud** credential with project write access.
25. Add a **Jira** node named "Jira: Create Previs Review Task".
    - Operation: Create issue
    - Project ID: Your Jira project ID
    - Issue Type ID: Your Jira issue type ID (e.g., Task)
    - Summary: `[AI Previs] {{ $json.shotCode }} – {{ $json.totalMoves }} Camera Options | Supervisor Pick Required`
    - Description: Structured text with shot metadata and `$json.jiraDesc`
    - Select the Jira credential from step 24
    - Connect: **Compile Previs Board Package1** → **Jira: Create Previs Review Task**

**Step 19 — Configure ClickUp credential and task creation node**

26. In n8n credential settings, create a **ClickUp** credential with workspace access.
27. Add a **ClickUp** node named "ClickUp: Create Previs Production Record".
    - Operation: Create task
    - Team ID, Space ID, Folder ID, List ID: Your ClickUp hierarchy IDs
    - Task name: `[AI Previs] {{ $json.shotCode }}`
    - Content: Full structured description with shot metadata, camera options, and video links
    - Select the ClickUp credential from step 26
    - Connect: **Compile Previs Board Package1** → **ClickUp: Create Previs Production Record**

**Step 20 — Configure Telegram delivery node (reuse credential from step 8)**

28. Add a **Telegram** node named "Telegram: Deliver Previs to Supervisor".
    - Chat ID: Your Telegram chat ID
    - Message: Full formatted previs board with all move details and A/B/C pick prompt
    - Parse mode: Markdown
    - Select the Telegram credential from step 8
    - Connect: **Compile Previs Board Package1** → **Telegram: Deliver Previs to Supervisor**

**Step 21 — Add Workflow Error Handler**

29. Add an **Error Trigger** node named "On Workflow Error".
30. Add a **Slack** node named "Slack: Error Alert".
    - Authentication: OAuth2, select the Slack credential from step 22
    - Channel ID: Your error alert channel ID
    - Message: `❌ AI Previs Pipeline — Error: {{ $json.message }} — Time: {{ new Date().toISOString() }}`
    - Connect: **On Workflow Error** → **Slack: Error Alert**

**Step 22 — Activate and test**

31. Activate the workflow.
32. Copy the form trigger webhook URL and share it with your production team.
33. Run a test submission with a simple shot description.
34. Verify that:
    - The AI agent returns valid JSON with 3 moves
    - Seedance jobs are submitted and polling succeeds
    - The previs board appears in Slack, Jira, ClickUp, and Telegram
    - The video is archived in Google Drive
35. Update the Slack: Error Alert message text — the current text references "AI-Assisted Clean Plate & Object Removal" which appears to be a copy-paste artifact; correct it to reference this pipeline.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The Slack: Error Alert node contains a message referencing "AI-Assisted Clean Plate & Object Removal" — this is likely a copy-paste artifact from a different workflow. Update the message text to accurately reference the AI Previs Pipeline. | Slack: Error Alert node |
| The AI Agent retry loop (Telegram Alert → Extract & Map → AI Agent) has no maximum retry counter. A persistent AI failure will cause an infinite loop. Consider adding a counter or a max-retries guard in the Extract & Map Form Fields code node. | Block 1 retry logic |
| The Seedance polling loop has no failure detection for a `status` of `"failed"`. If Seedance returns a failed status, the workflow will poll indefinitely every 20 seconds. Add a condition to detect `"failed"` and route to an error path. | Block 2 polling loop |
| The Jira description uses `*bold*` markdown syntax which is Slack-flavored. Jira uses its own wiki markup. Consider converting the `jiraDesc` format to Jira-compatible markup if rendering is important. | Compile Previs Board Package1 → Jira description |
| The Telegram delivery message may exceed the 4096-character limit for long script snippets or detailed move descriptions. Consider splitting into multiple messages or truncating. | Telegram: Deliver Previs to Supervisor |
| Seedance API base URL: `https://ark.ap-southeast.bytepluses.com/api/v3/contents/generations/tasks` | BytePlus/Seedance API endpoint |
| The Seedance model identifier `seedance-1-5-pro-251215` is hardcoded. If the provider updates the model, this value must be changed manually in the Build Seedance API Request code node. | Build Seedance API Request |
| Key frame timecodes are hardcoded for a 5-second, 24fps video (frames 1, 48, 120). If Seedance generation parameters change (different duration or frame rate), these values should be recalculated. | Collect Move + Tag Key Frames |
| The Compile Previs Board Package1 sticky note mentions "Confluence" as a delivery target, but no Confluence node exists in the workflow. The `lookbookHtml` field is generated but not sent to any node — it is available for future integration if needed. | Section: Key Frame & Board Compile sticky note |
| Azure OpenAI deployment name must match the model field. The node is set to `gpt-4o-mini` — if your Azure deployment uses a different name, update both the credential and the model parameter. | Azure OpenAI: GPT-4o Mini |