Create a one-minute video from Telegram prompts with Veo 3 and Fal.ai

https://n8nworkflows.xyz/workflows/create-a-one-minute-video-from-telegram-prompts-with-veo-3-and-fal-ai-11540


# Create a one-minute video from Telegram prompts with Veo 3 and Fal.ai

## 1. Workflow Overview

**Purpose:** This workflow turns a single Telegram message (a “video idea”) into a ~1-minute vertical video by:
1) using an AI agent to expand the idea into a script plus **7 standalone Veo prompts** (for 7× ~8-second clips),
2) optionally requiring **human confirmation** (“CONFIRM” / “RETRY”) via Telegram after reviewing a Google Sheet row,
3) generating 7 clips with **Google Gemini Veo 3.1 fast**,
4) uploading each clip to **Google Drive** to obtain public URLs,
5) merging the 7 URLs into one video via **fal.ai ffmpeg merge**,
6) sending the merged video back to the user on Telegram.

### 1.1 Input Reception & User Warning (Telegram)
- Entry point: Telegram trigger receives a message.
- Immediately sends a warning message not to send additional messages.

### 1.2 Prompt/Script Generation (LLM Agent + Structured Parser)
- AI Agent (using OpenAI Chat Model) produces **script + prompts 1..7** in a strict JSON schema validated by a structured output parser.

### 1.3 Human-in-the-loop Review (Google Sheets + Telegram confirm/retry)
- Saves script/prompts to Google Sheets.
- Sends a Telegram “send and wait” message asking user to check the latest row and reply **CONFIRM** or **RETRY**.
- If **RETRY**, loops back to regenerate prompts (agent reruns).

### 1.4 Clip Generation Pipeline (7× Veo → Drive URL labeling → Merge)
- Runs 7 Veo generations (some gated by Wait nodes).
- Each clip is uploaded to a specific Google Drive folder.
- Each upload’s `webContentLink` is stored under a key like `prompt(1)`, `Prompt(2)`, etc.
- A 7-input Merge node combines these into one item for fal.ai merge.

### 1.5 Merge & Delivery (fal.ai polling → Telegram sendVideo)
- Calls fal.ai merge endpoint, then polls `status_url` until `status == COMPLETED`.
- Fetches final result from `response_url`.
- Sends merged video URL to Telegram chat.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Input Reception & “Do not interrupt” warning
**Overview:** Receives a Telegram message and informs the user the process has started and not to send more messages (to avoid interrupting stateful steps like “sendAndWait”).  
**Nodes involved:** `Telegram Trigger`, `Send a text message8`

#### Node: Telegram Trigger
- **Type/Role:** `n8n-nodes-base.telegramTrigger` — workflow entry point for incoming Telegram updates.
- **Config:** Listens to `message` updates.
- **Input/Output:** No input. Outputs the Telegram update payload (`message`, `chat.id`, user info).
- **Failure modes:** Telegram webhook misconfiguration; bot token invalid; n8n URL not reachable.

#### Node: Send a text message8
- **Type/Role:** `n8n-nodes-base.telegram` — sends a warning message to the same chat.
- **Config:** `operation: sendMessage` (implicit by providing `text`), `chatId` from trigger:  
  `={{ $('Telegram Trigger').item.json.message.chat.id }}`
- **Message:** Greets user and warns not to send additional messages.
- **Failure modes:** Invalid chat ID (rare), revoked bot permissions, Telegram rate limits.

---

### Block 2.2 — Prompt & Script generation (Agent + OpenAI model + Structured parsing)
**Overview:** Converts the user’s raw idea into a consistent 7-clip plan: a script plus 7 independent prompts designed to maintain continuity without shared context.  
**Nodes involved:** `AI Agent`, `OpenAI Chat Model`, `Structured Output Parser`

#### Node: OpenAI Chat Model
- **Type/Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM provider for the agent.
- **Config choices:**
  - Model: `gpt-5.1`
  - Timeout: `3600000` ms (1 hour) to avoid long prompt failures.
- **Connections:** Connected into AI Agent via `ai_languageModel`.
- **Failure modes:** Credential issues, model access denied, long responses/timeouts, rate limits.

#### Node: Structured Output Parser
- **Type/Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — validates/parses strict JSON output.
- **Config:** JSON schema example requiring:
  - `items[0].script`, `prompt1`..`prompt7` (strings)
- **Connections:** Connected into AI Agent via `ai_outputParser`.
- **Failure modes:** If agent output is not valid JSON or missing keys, parsing fails (common if prompt formatting drifts).

#### Node: AI Agent
- **Type/Role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates LLM generation with strong formatting rules.
- **Config choices (interpreted):**
  - PromptType: `define` (custom system/task prompt).
  - Strong instructions: “7 independent prompts”, restate descriptions each time, dialogue word limits, etc.
  - Output must match a very strict JSON structure (also enforced by parser).
  - Includes `{{ $json.Input }}` placeholder (expects an `Input` field in incoming JSON).
- **Inputs:**
  - Receives Telegram Trigger output. **But note:** Telegram payload does not naturally provide `$json.Input`.
- **Outputs:**
  - In downstream nodes, values are referenced as:  
    `$('AI Agent').item.json.output.items[0].prompt1` etc.
- **Key risk / edge case:** **Mismatch of expected input field**:
  - The agent prompt references `{{ $json.Input }}` but incoming Telegram message is typically at `$json.message.text`.  
  - Without a mapping Set node, the agent may receive empty input, producing generic outputs or failing schema.
- **Failure modes:** LLM refuses or truncates; JSON invalid; missing keys; very long prompts; timeouts.

---

### Block 2.3 — Store prompts and request human confirmation
**Overview:** Saves the generated script/prompts into Google Sheets, then pauses execution waiting for the user to respond “CONFIRM” or “RETRY”.  
**Nodes involved:** `Save to Google Sheets`, `Send message and wait for response`, `If1`

#### Node: Save to Google Sheets
- **Type/Role:** `n8n-nodes-base.googleSheets` — persists prompts for review and later reuse.
- **Operation:** `appendOrUpdate`
- **Document/Sheet:** Spreadsheet “VEO Videos” / `Sheet1` (gid=0).
- **Column mapping:** Writes:
  - `Script` = `$('AI Agent').item.json.output.items[0].script`
  - `Prompt 1..7` = corresponding `prompt1..prompt7`
- **Matching column:** `Script` used as the key for update vs append.
- **Failure modes:** Google auth expired; Sheets API quota; schema mismatch; missing agent output keys.

#### Node: Send message and wait for response
- **Type/Role:** `n8n-nodes-base.telegram` — `operation: sendAndWait` (stateful).
- **Config:**
  - `chatId` from trigger
  - Message: asks to review sheet and reply `CONFIRM` or `RETRY` (the link is a placeholder text: “(insert sheets link)”).
  - `responseType: freeText`
- **Output:** Response object typically appears under `$json.data.text`.
- **Failure modes:** User never responds (execution remains waiting); user replies with unexpected text; Telegram delivery issues.

#### Node: If1
- **Type/Role:** `n8n-nodes-base.if` — checks whether user typed “RETRY”.
- **Condition:** `={{ $json.data.text }} == "RETRY"`
- **Routing:**
  - **True:** back to `AI Agent` (regenerate prompts and overwrite/append).
  - **False:** starts video generation branches (multiple nodes launched).
- **Edge cases:**
  - User sends “retry”, “Retry”, extra spaces → condition fails due to strict equals.
  - User sends “CONFIRM” → goes to generation path (as intended).
  - Any other text → also goes to generation path (possibly unintended).

---

### Block 2.4 — Generate 7 clips with Veo (Gemini), upload to Drive, label URLs, progress updates
**Overview:** Produces 7 videos from the 7 prompts, uploads each to Google Drive, stores the public link in a normalized field, and sends progress messages to Telegram. Some clips are delayed using Wait nodes.  
**Nodes involved:**  
- Generation: `Generate a video1`, `Generate a video2`, `Generate a video3`, `Generate a video4`, `Generate a video5`, `Generate a video` (Prompt 6), `Generate a video6` (Prompt 7)  
- Drive: `Upload video1`, `Upload video`, `Upload video3`, `Upload video4`, `Upload video5`, `Upload video6`, `Upload video7`  
- Labeling: `Label URL 1..7`  
- Progress: `Send a text message`, `Send a text message2..7`  
- Timing: `Wait5`, `Wait4`, `Wait3`

#### Node: Generate a video1
- **Type/Role:** `@n8n/n8n-nodes-langchain.googleGemini` — generates video via Gemini (Veo).
- **Prompt:** `={{ $('Save to Google Sheets').item.json['Prompt 1'] }}`
- **Model:** `models/veo-3.1-fast-generate-preview`
- **Options:** `aspectRatio: 9:16` (vertical)
- **Outputs:** binary/video metadata including `fileName` (used later).
- **Failure modes:** Gemini/Veo access issues; quota; long prompt errors; model preview instability.

#### Node: Upload video1
- **Type/Role:** `n8n-nodes-base.googleDrive` — uploads generated file to Drive folder.
- **Name:** `={{$now.format('yyyyLLdd')}}{{ $json.fileName }}`
- **Folder:** “VEO VIDEOS” (specific folder ID)
- **Assumption:** Upload operation returns `webContentLink` (download link).
- **Failure modes:** Drive auth/permissions; folder not accessible; file too large; missing binary data.

#### Node: Label URL 1
- **Type/Role:** `n8n-nodes-base.set` — stores the public URL under a stable key.
- **Assignment:** `prompt(1) = {{$json.webContentLink}}`
- **Output:** Item contains `prompt(1)` used later by Merge.
- **Failure modes:** `webContentLink` absent if Drive sharing isn’t configured or API response differs.

#### Node: Send a text message
- **Type/Role:** Telegram sendMessage — progress: “Video 1/7 generated”
- **Edge cases:** Telegram rate limits if many runs.

(Repeat pattern for videos 2–7; key differences below.)

#### Generate a video2 / Upload video / Label URL 2 / Send a text message2
- **Prompt field:** `Prompt 2` from Sheets.
- **Label key:** `Prompt(2)` (note capitalization differs from `prompt(1)`).
- **Risk:** Inconsistent key naming can break merge if referenced incorrectly.

#### Wait5 → Generate a video3 → Upload video3 → Label URL 3 → Send a text message3
- **Wait5:** waits `1.5 minutes` before generating Prompt 3.
- **Purpose:** throttling / quota management / staggering Veo calls.
- **Risk:** Wait nodes share identical webhookId values (see below).

#### Generate a video4 / Upload video4 / Label URL 4 / Send a text message4
- Prompt 4 from Sheets; label `Prompt(4)`.

#### Wait4 → Generate a video5 → Upload video5 → Label URL 5 → Send a text message5
- Wait 1.5 minutes; label `Prompt(5)`.

#### Generate a video (Prompt 6) / Upload video6 / Label URL 6 / Send a text message6
- Prompt 6 from Sheets; label `Prompt(6)`.

#### Wait3 → Generate a video6 (Prompt 7) / Upload video7 / Label URL 7 / Send a text message7
- Wait 1.5 minutes; final progress message says generation complete and combining begins.
- Label `Prompt(7)`.

**Important edge cases in this whole block:**
- **Drive links must be publicly accessible** for fal.ai to fetch them. Merely uploading may not make them public; `webContentLink` may still require auth unless file permissions are changed.
- The workflow does **not** include an explicit “Share file publicly / set permissions” node; it assumes the folder or defaults make links accessible.
- Multiple Wait nodes (`Wait`, `Wait3`, `Wait4`, `Wait5`) reuse the **same webhookId** in the JSON. In n8n this can be problematic or indicate copy/paste; typically each Wait node needs its own internal resume mechanism.

---

### Block 2.5 — Collect URLs (7-input merge) and request fal.ai merge
**Overview:** Combines the seven URL-bearing items into one payload and submits a merge job to fal.ai.  
**Nodes involved:** `Merge`, `HTTP Request`

#### Node: Merge
- **Type/Role:** `n8n-nodes-base.merge` — combines inputs from 7 branches.
- **Mode:** `combineByPosition`, `numberInputs: 7`
- **Effect:** Produces one item containing all URL fields from all 7 inputs (positionally).
- **Failure modes:** If any branch doesn’t produce an item, merge may hang or output incomplete fields.

#### Node: HTTP Request (fal.ai merge-videos)
- **Type/Role:** `n8n-nodes-base.httpRequest` — creates fal queue job to merge videos.
- **Method/URL:** POST `https://queue.fal.run/fal-ai/ffmpeg-api/merge-videos`
- **Headers:**
  - `Authorization: Key (your fal.ai key)` (must be replaced with real key format `Key YOUR_FAL_API_KEY`)
  - `Content-Type: application/json`
- **JSON body:** sends `video_urls` array using fields:
  - `{{ $json["prompt(1)"] }}`, `{{ $json["Prompt(2)"] }}`, `{{ $json['Prompt(3)'] }}` … through `Prompt(7)`
- **Config mismatch / risk:**
  - The workflow defines `Label URL 3` as `Prompt(3)` (capital P) but the HTTP body references `Prompt(3)` (OK).
  - However it references `Prompt(3)` with different quoting styles; must match exactly.
- **Resolution mismatch risk:** body requests `"resolution": "landscape_16_9"` while generation uses `9:16` at least for clip 1. This can cause scaling/cropping/letterboxing.
- **Failure modes:** fal key invalid; job errors if URLs not publicly reachable; merge service rejects unsupported formats.

---

### Block 2.6 — Poll merge status, fetch result, send final video
**Overview:** Polls fal job status until completion, retrieves final merged video URL, sends it via Telegram.  
**Nodes involved:** `HTTP Request1`, `If`, `Wait`, `HTTP Request2`, `Send a video`

#### Node: HTTP Request1 (poll status_url)
- **Type/Role:** HTTP GET to the status endpoint returned by fal.
- **URL:** `={{ $json.status_url }}`
- **Headers:** Authorization and Content-Type set (Content-Type not strictly needed for GET).
- **Failure modes:** status_url missing; job expired; transient HTTP errors.

#### Node: If (status == COMPLETED)
- **Type/Role:** branching poll controller.
- **Condition:** `={{ $json.status }} == "COMPLETED"`
- **True path:** `HTTP Request2` (fetch response_url)
- **False path:** `Wait` then back to `HTTP Request1`

#### Node: Wait (30 seconds)
- **Type/Role:** delays polling loop.
- **Config:** `amount: 30` (defaults to seconds in this node version)
- **Risk:** Uses same webhookId as other Wait nodes in the workflow JSON (possible collision).

#### Node: HTTP Request2 (get response_url)
- **Type/Role:** HTTP GET to fetch final merge output object.
- **URL:** `={{ $json.response_url }}`
- **Expected output:** JSON containing `video.url`.

#### Node: Send a video
- **Type/Role:** `n8n-nodes-base.telegram` — `operation: sendVideo`
- **File:** `={{ $json.video.url }}`
- **chatId:** from trigger chat id.
- **Failure modes:** Telegram cannot fetch remote URL; URL expires; file too large for Telegram limits.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Entry point: receive user prompt | — | Send a text message8, AI Agent | ## Trigger ## |
| Send a text message8 | telegram | Notify user processing started + don’t interrupt | Telegram Trigger | — | ## Trigger ## |
| OpenAI Chat Model | lmChatOpenAi | LLM powering AI Agent | — | AI Agent (ai_languageModel) |  |
| Structured Output Parser | outputParserStructured | Enforce JSON schema for agent output | — | AI Agent (ai_outputParser) |  |
| AI Agent | langchain.agent | Create script + 7 Veo prompts | Telegram Trigger / If1(true) | Save to Google Sheets |  |
| Save to Google Sheets | googleSheets | Persist script/prompts for review | AI Agent | Send message and wait for response |  |
| Send message and wait for response | telegram (sendAndWait) | Human confirmation gate | Save to Google Sheets | If1 |  |
| If1 | if | Branch on RETRY vs proceed | Send message and wait for response | AI Agent (true), Generate a video* & Wait* (false) |  |
| Generate a video1 | googleGemini (video) | Generate clip 1 (Prompt 1) | If1(false) | Upload video1, Send a text message | ## Generation ## |
| Upload video1 | googleDrive | Upload clip 1 to Drive | Generate a video1 | Label URL 1 | ## Generation ## |
| Label URL 1 | set | Store clip1 URL as prompt(1) | Upload video1 | Merge | ## Generation ## |
| Send a text message | telegram | Progress “Video 1/7 generated” | Generate a video1 | — | ## Generation ## |
| Generate a video2 | googleGemini (video) | Generate clip 2 (Prompt 2) | If1(false) | Upload video, Send a text message2 | ## Generation ## |
| Upload video | googleDrive | Upload clip 2 to Drive | Generate a video2 | Label URL 2 | ## Generation ## |
| Label URL 2 | set | Store clip2 URL as Prompt(2) | Upload video | Merge | ## Generation ## |
| Send a text message2 | telegram | Progress “Video 2/7 generated” | Generate a video2 | — | ## Generation ## |
| Wait5 | wait | Delay before clip 3 | If1(false) | Generate a video3 | ## Generation ## |
| Generate a video3 | googleGemini (video) | Generate clip 3 (Prompt 3) | Wait5 | Upload video3, Send a text message3 | ## Generation ## |
| Upload video3 | googleDrive | Upload clip 3 to Drive | Generate a video3 | Label URL 3 | ## Generation ## |
| Label URL 3 | set | Store clip3 URL as Prompt(3) | Upload video3 | Merge | ## Generation ## |
| Send a text message3 | telegram | Progress “Video 3/7 generated” | Generate a video3 | — | ## Generation ## |
| Generate a video4 | googleGemini (video) | Generate clip 4 (Prompt 4) | If1(false) | Upload video4, Send a text message4 | ## Generation ## |
| Upload video4 | googleDrive | Upload clip 4 to Drive | Generate a video4 | Label URL 4 | ## Generation ## |
| Label URL 4 | set | Store clip4 URL as Prompt(4) | Upload video4 | Merge | ## Generation ## |
| Send a text message4 | telegram | Progress “Video 4/7 generated” | Generate a video4 | — | ## Generation ## |
| Wait4 | wait | Delay before clip 5 | If1(false) | Generate a video5 | ## Generation ## |
| Generate a video5 | googleGemini (video) | Generate clip 5 (Prompt 5) | Wait4 | Upload video5, Send a text message5 | ## Generation ## |
| Upload video5 | googleDrive | Upload clip 5 to Drive | Generate a video5 | Label URL 5 | ## Generation ## |
| Label URL 5 | set | Store clip5 URL as Prompt(5) | Upload video5 | Merge | ## Generation ## |
| Send a text message5 | telegram | Progress “Video 5/7 generated” | Generate a video5 | — | ## Generation ## |
| Generate a video | googleGemini (video) | Generate clip 6 (Prompt 6) | If1(false) | Upload video6, Send a text message6 | ## Generation ## |
| Upload video6 | googleDrive | Upload clip 6 to Drive | Generate a video | Label URL 6 | ## Generation ## |
| Label URL 6 | set | Store clip6 URL as Prompt(6) | Upload video6 | Merge | ## Generation ## |
| Send a text message6 | telegram | Progress “Video 6/7 generated” | Generate a video | — | ## Generation ## |
| Wait3 | wait | Delay before clip 7 | If1(false) | Generate a video6 | ## Generation ## |
| Generate a video6 | googleGemini (video) | Generate clip 7 (Prompt 7) | Wait3 | Upload video7, Send a text message7 | ## Generation ## |
| Upload video7 | googleDrive | Upload clip 7 to Drive | Generate a video6 | Label URL 7 | ## Generation ## |
| Label URL 7 | set | Store clip7 URL as Prompt(7) | Upload video7 | Merge | ## Generation ## |
| Send a text message7 | telegram | “Generation complete, combining…” | Generate a video6 | — | ## Generation ## |
| Merge | merge | Combine 7 URL items into 1 | Label URL 1..7 | HTTP Request | ## Fal\n\nFal.AI configuration:\n\nFal is an AI platform and we will be using their ffmpeg video combiner |
| HTTP Request | httpRequest | Submit fal merge job | Merge | HTTP Request1 | ## Fal\n\nFal.AI configuration:\n\nFal is an AI platform and we will be using their ffmpeg video combiner |
| HTTP Request1 | httpRequest | Poll fal status_url | HTTP Request / Wait | If | ## Get Video |
| If | if | Completed? else wait and poll again | HTTP Request1 | HTTP Request2 (true), Wait (false) | ## Get Video |
| Wait | wait | Delay polling 30s | If(false) | HTTP Request1 | ## Get Video |
| HTTP Request2 | httpRequest | Fetch final output from response_url | If(true) | Send a video | ## Get Video |
| Send a video | telegram | Deliver merged video back to user | HTTP Request2 | — | ## Get Video |
| Sticky Note2 | stickyNote | Explanation/setup notes | — | — | ## How It Works … (full note content in workflow) |
| Sticky Note8 | stickyNote | Quota warning (Google API limits) | — | — | ## Notice\n\nDepending on your google API level... |
| Sticky Note11 | stickyNote | Section label | — | — | ## Trigger ## |
| Sticky Note12 | stickyNote | Section label | — | — | ## Generation ## |
| Sticky Note13 | stickyNote | Section label | — | — | ## Get Video |
| Sticky Note | stickyNote | Fal section label | — | — | ## Fal… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Telegram Bot + Credentials**
   1. Create bot via BotFather, copy token.
   2. In n8n, add **Telegram** credentials (bot token).
   3. Add node **Telegram Trigger**:
      - Updates: `message`

2) **Immediate user warning**
   1. Add **Telegram** node “Send a text message8”:
      - Operation: Send Message
      - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`
      - Text: warning message
   2. Connect: `Telegram Trigger → Send a text message8`

3) **(Recommended) Normalize user input for the AI Agent**
   - Add a **Set** node (not present in JSON but needed for correctness):
     - `Input = {{$json.message.text}}`
   - Connect: `Telegram Trigger → Set(Input)` and use this as input to the AI Agent.
   - If you skip this, adjust the AI Agent prompt to use `{{ $json.message.text }}` instead of `{{ $json.Input }}`.

4) **AI Agent with strict JSON output**
   1. Add **OpenAI Chat Model** node:
      - Model: `gpt-5.1`
      - Timeout: 3600000 ms
      - Configure OpenAI credentials.
   2. Add **Structured Output Parser**:
      - Schema example with `items[0].script` and `prompt1..prompt7`.
   3. Add **AI Agent**:
      - PromptType: Define
      - Paste the long task instruction (ensure the final required JSON format matches the parser schema).
      - Ensure it references the normalized input (e.g., `{{ $json.Input }}`).
   4. Connect:
      - OpenAI Chat Model → AI Agent (ai_languageModel)
      - Structured Output Parser → AI Agent (ai_outputParser)
      - Telegram Trigger (or Set(Input)) → AI Agent (main)

5) **Save outputs to Google Sheets**
   1. Create Google Sheets + enable API; add n8n Google credentials.
   2. Create spreadsheet with columns: `Script`, `Prompt 1`..`Prompt 7`.
   3. Add **Google Sheets** node “Save to Google Sheets”:
      - Operation: Append or Update
      - Document: your sheet
      - Matching column: `Script`
      - Map columns from `$('AI Agent').item.json.output.items[0].…`
   4. Connect: `AI Agent → Save to Google Sheets`

6) **Human confirmation gate**
   1. Add **Telegram** node “Send message and wait for response”:
      - Operation: Send and Wait
      - Chat ID from trigger
      - Message instructing to reply CONFIRM/RETRY (+ include your sheets link)
      - Response type: freeText
   2. Add **If** node “If1”:
      - Condition: `{{$json.data.text}} equals "RETRY"`
   3. Connect:
      - Save to Google Sheets → Send and wait
      - Send and wait → If1
      - If1(true) → AI Agent (loop)
      - If1(false) → start generation (next step)

7) **Create 7 Veo generation branches (Gemini)**
   - Add 7 **Google Gemini** nodes (resource: Video), each with:
     - Model: `models/veo-3.1-fast-generate-preview`
     - Prompt:
       - #1: `={{ $('Save to Google Sheets').item.json['Prompt 1'] }}`
       - #2: `Prompt 2`, etc.
     - Optionally set aspect ratio consistently (the JSON only sets `9:16` for the first clip; set for all if desired).
   - Add **Wait** nodes before some generations if you need throttling (as in the JSON: 1.5 min before prompts 3, 5, 7).

8) **Upload each clip to Google Drive**
   - Add 7 **Google Drive** nodes (Upload):
     - Name: `={{$now.format('yyyyLLdd')}}{{ $json.fileName }}`
     - Folder: your “VEO VIDEOS” folder
   - Connect each Generate node to its corresponding Upload node.

9) **Ensure links are publicly accessible**
   - Add a **Google Drive: Share** / **Permission** step per uploaded file (not present in JSON) OR ensure folder defaults produce publicly readable `webContentLink`.
   - Without public access, fal.ai will fail to fetch.

10) **Label each URL for merging**
   - Add 7 **Set** nodes to store:
     - `prompt(1) = {{$json.webContentLink}}`
     - `Prompt(2) … Prompt(7) = {{$json.webContentLink}}`
   - Connect each Upload node → corresponding Label node.

11) **Progress messages**
   - After each Generate node, add a **Telegram sendMessage**:
     - “Video n/7 generated”
   - Final one: “Video generation Complete! Please wait while they are combined.”

12) **Merge 7 URL items**
   - Add **Merge** node:
     - Mode: Combine
     - Combine by position
     - Number of inputs: 7
   - Connect Label URL 1..7 outputs to Merge inputs 0..6.

13) **fal.ai merge request**
   1. Add **HTTP Request** node:
      - POST `https://queue.fal.run/fal-ai/ffmpeg-api/merge-videos`
      - JSON body with `video_urls` array referencing the merged fields
      - Headers:
        - `Authorization: Key YOUR_FAL_API_KEY`
        - `Content-Type: application/json`
   2. Connect: Merge → HTTP Request

14) **Poll fal status until completed**
   1. Add **HTTP Request** node to GET `{{$json.status_url}}`
   2. Add **If** node: status equals `COMPLETED`
   3. Add **Wait** node: 30 seconds
   4. Wire:
      - POST merge → GET status
      - GET status → If
      - If(false) → Wait → GET status
      - If(true) → next

15) **Fetch final response and send video**
   1. Add **HTTP Request** node GET `{{$json.response_url}}`
   2. Add **Telegram** node sendVideo:
      - File: `{{$json.video.url}}`
      - Chat ID from trigger
   3. Connect: If(true) → GET response → SendVideo

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Depending on your Google API level, you can only run this once or twice per day before reaching a limit. If you need more, configure fal.ai nodes instead of Gemini, although it is more expensive. | Sticky note “Notice” |
| Fal.AI configuration: Fal is an AI platform and we will be using their ffmpeg video combiner | Sticky note “Fal” |
| Workflow behavior, human-in-loop, and setup requirements for Telegram/Drive/Sheets/Gemini/OpenAI/fal.ai; fal key format `Key YOUR_FAL_API_KEY`; requires public URLs; no auto-posting | Sticky note “How It Works” |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.