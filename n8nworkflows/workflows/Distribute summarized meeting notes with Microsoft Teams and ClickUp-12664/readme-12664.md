Distribute summarized meeting notes with Microsoft Teams and ClickUp

https://n8nworkflows.xyz/workflows/distribute-summarized-meeting-notes-with-microsoft-teams-and-clickup-12664


# Distribute summarized meeting notes with Microsoft Teams and ClickUp

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Distribute summarized meeting notes with Microsoft Teams and ClickUp  
**Workflow name (in n8n):** My workflow  
**Purpose:** Convert a meeting recording (or notes) into a transcript, summarize it with an LLM, extract action items, create a ClickUp task for archival/tracking, and notify participants via Microsoft Teams. If a failure is detected early (bad transcription response) or later (missing summary), a Teams error message is sent.

### 1.1 Logical Blocks
1. **1.1 Trigger & Input Initialization**
   - Manual start + set meeting metadata + submit transcription job + basic status gate.
2. **1.2 Transcription Retrieval & Text Shaping**
   - Wait briefly, fetch transcript, normalize to plain text while keeping routing metadata.
3. **1.3 AI Summarization & Validation**
   - Send transcript to OpenAI Chat Completions, parse/normalize result, validate summary presence.
4. **1.4 Action Items Normalization & Merge**
   - Ensure action items are an array and merge with summary payload.
5. **1.5 Storage (ClickUp) & Notification (Teams)**
   - Create ClickUp task, then send formatted Teams message.
6. **1.6 Error Handling (Teams)**
   - Build and send a Teams alert when upstream checks fail.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Trigger & Input Initialization
**Overview:** Starts the workflow manually, defines meeting metadata and routing IDs, posts a transcription request, and branches immediately if the transcription request appears unsuccessful.

**Nodes involved:**
- Start Workflow
- Set Meeting Info
- Send to Transcription API
- Transcription Request OK?
- (Sticky note) Trigger & Input
- (Sticky note) 📋 Meeting Notes Distributor

#### Node: Start Workflow
- **Type / role:** `Manual Trigger` — entry point for ad-hoc runs.
- **Configuration:** No parameters; user clicks “Execute workflow”.
- **Connections:**
  - **Output →** Set Meeting Info
- **Edge cases / failures:** None (manual trigger is deterministic).

#### Node: Set Meeting Info
- **Type / role:** `Set` — provides required fields for downstream API calls and routing.
- **Configuration choices (interpreted):**
  - The JSON provided does not show explicit fields, but downstream code expects these keys to exist:
    - `meetingTitle`
    - `participants` (likely emails or Teams identifiers)
    - `clickupListId`
    - `teamsChannelId`
    - likely a `recordingUrl` (needed by transcription request)
- **Connections:**
  - **Input ←** Start Workflow
  - **Output →** Send to Transcription API
- **Key variables:** `$json.meetingTitle`, `$json.participants`, `$json.clickupListId`, `$json.teamsChannelId`
- **Edge cases / failures:**
  - Missing `recordingUrl` (or equivalent) would make transcription POST invalid.
  - Missing list/channel/chat IDs will break ClickUp/Teams later.

#### Node: Send to Transcription API
- **Type / role:** `HTTP Request` — submits the recording URL to a transcription service.
- **Configuration choices (interpreted):**
  - **POST** to `https://api.exampletranscript.com/v1/transcribe`
  - Uses **predefined credentials** of type `genericApiAuth` (API key/header-based).
  - Payload/body not shown in JSON; must include recording URL and possibly language/speaker diarization settings.
- **Connections:**
  - **Input ←** Set Meeting Info
  - **Output →** Transcription Request OK?
- **Edge cases / failures:**
  - Auth failure (401/403) due to incorrect `genericApiAuth`.
  - 4xx due to missing/invalid recording URL.
  - Timeouts for large media.
  - Provider-specific response may not include `id` needed later.

#### Node: Transcription Request OK?
- **Type / role:** `IF` — branches based on status code to success vs error handling.
- **Configuration choices:**
  - Condition (number): `{{$json.statusCode || 200}} <= 299`
  - This is **optimistic**: if `statusCode` is absent, it assumes 200.
- **Connections:**
  - **Input ←** Send to Transcription API
  - **True →** Wait for Processing
  - **False →** Prepare Error Message
- **Edge cases / failures:**
  - If the HTTP node does not include `statusCode` in its output, failures could pass as success (because of `|| 200`).
  - Provider may return 200 but with an error payload (needs explicit JSON error checking).

---

### Block 1.2 — Transcription Retrieval & Text Shaping
**Overview:** Waits briefly for the transcription job to complete, polls the transcript endpoint, then reshapes the response into a clean `transcriptText` plus metadata needed downstream.

**Nodes involved:**
- Wait for Processing
- Fetch Transcript
- Extract Transcript Text
- (Sticky note) Processing & Summarization

#### Node: Wait for Processing
- **Type / role:** `Wait` — delays execution.
- **Configuration choices:**
  - Unit: seconds (but **no duration value is shown**; in n8n a duration is typically required).
- **Connections:**
  - **Input ←** Transcription Request OK? (true)
  - **Output →** Fetch Transcript
- **Version-specific considerations:** Wait node v1 supports “Wait for X time” or “Wait until”; ensure a duration is configured in the UI.
- **Edge cases / failures:**
  - If duration is not configured, the node may not behave as intended.
  - Fixed wait can be insufficient for long recordings; polling with retries is usually safer.

#### Node: Fetch Transcript
- **Type / role:** `HTTP Request` — retrieves the completed transcript.
- **Configuration choices:**
  - URL expression: `{{ 'https://api.exampletranscript.com/v1/transcript/' + $json.id }}`
  - Uses `genericApiAuth` predefined credentials.
- **Connections:**
  - **Input ←** Wait for Processing
  - **Output →** Extract Transcript Text
- **Key variables:**
  - `$json.id` must come from the transcription submission response.
- **Edge cases / failures:**
  - Missing `$json.id` (transcription API returned different schema).
  - Transcript not ready yet (provider may return 202/404/empty); no retry logic is implemented.
  - Auth/permissions issues.

#### Node: Extract Transcript Text
- **Type / role:** `Code` — normalizes provider response into plain text + preserves metadata.
- **Configuration (code behavior):**
  - Reads transcript provider response as `data = items[0].json`.
  - Outputs:
    - `meetingTitle: $json.meetingTitle`
    - `participants: $json.participants`
    - `transcriptText: data.transcript || data.text || ''`
    - `clickupListId: $json.clickupListId`
    - `teamsChannelId: $json.teamsChannelId`
- **Connections:**
  - **Input ←** Fetch Transcript
  - **Output →** AI Summarize
- **Important implementation note (edge case):**
  - The code references `$json.meetingTitle` etc. while also reading `items[0].json` into `data`. In n8n Code nodes, `$json` typically refers to the current item’s JSON (which is `data` here). If the transcript response does not include `meetingTitle/participants/...`, these fields will become `undefined`.
  - Safer pattern: use `data` for transcript fields and `items[0].json` from previous node via paired items or merge, or use `$('Set Meeting Info').first().json.meetingTitle`.
- **Edge cases / failures:**
  - Transcript field not named `transcript` or `text`.
  - Empty transcript leads to weak/failed summarization downstream.

---

### Block 1.3 — AI Summarization & Validation
**Overview:** Sends transcript text to OpenAI Chat Completions, parses structured JSON if returned, and checks whether a summary exists before proceeding.

**Nodes involved:**
- AI Summarize
- Extract Summary JSON
- Summary Exists?
- (Sticky note) Processing & Summarization

#### Node: AI Summarize
- **Type / role:** `HTTP Request` — calls OpenAI chat completion endpoint.
- **Configuration choices (interpreted):**
  - POST `https://api.openai.com/v1/chat/completions`
  - Uses predefined credentials type `openAiApi`
  - Body not shown; must include `model`, `messages`, and should instruct the model to output strict JSON.
- **Connections:**
  - **Input ←** Extract Transcript Text
  - **Output →** Extract Summary JSON
- **Edge cases / failures:**
  - Invalid OpenAI credentials / revoked key.
  - Missing required request fields (model/messages) if not configured.
  - Token limits for long transcripts; requires chunking or a larger-context model.
  - Non-JSON model output despite instructions.

#### Node: Extract Summary JSON
- **Type / role:** `Code` — parses the LLM response into `summary` and `actionItems`.
- **Configuration (code behavior):**
  - Extracts: `resp.choices?.[0]?.message?.content || ''`
  - Tries `JSON.parse(content)`:
    - `summary = parsed.summary || parsed.notes || ''`
    - `actionItems = parsed.action_items || parsed.actions || ''`
  - If parsing fails: `summary = content` (raw text)
  - Outputs also: `meetingTitle`, `participants`, `clickupListId`, `teamsChannelId` via `$json...` (same caveat as above)
- **Connections:**
  - **Input ←** AI Summarize
  - **Output →** Summary Exists?
- **Edge cases / failures:**
  - If OpenAI output is not JSON, actionItems will remain empty string; later code attempts to split strings into bullets.
  - Same metadata-loss risk: `$json.meetingTitle` may be absent at this stage unless carried through properly.

#### Node: Summary Exists?
- **Type / role:** `IF` — validates that summary is present.
- **Configuration choices:**
  - Condition: string `{{$json.summary}} isEmpty`
  - **Important:** If condition is “isEmpty”, then:
    - **True branch means summary is empty**.
    - **False branch means summary exists**.
  - The current wiring sends:
    - Output 0 → Generate Action Items
    - Output 1 → Prepare Error Message
  - Depending on how n8n maps outputs for IF (true/false), this may be inverted.
- **Connections:**
  - **Input ←** Extract Summary JSON
  - **Branch A →** Generate Action Items
  - **Branch B →** Prepare Error Message
- **Edge cases / failures:**
  - Logic inversion risk: workflow might proceed when summary is empty and error when it exists (verify in UI: “true” output vs “false” output).
  - Summary could be whitespace; consider trimming before check.

---

### Block 1.4 — Action Items Normalization & Merge
**Overview:** Ensures action items are an array of strings, then merges summary and actions into a single payload for storage and notification.

**Nodes involved:**
- Generate Action Items
- Merge Summary & Actions

#### Node: Generate Action Items
- **Type / role:** `Code` — converts action items into a list.
- **Configuration (code behavior):**
  - `actions = $json.actionItems`
  - If string: splits on newline or bullet characters: `/\n|•|-|\*/`
  - Trims and filters empty entries
  - Outputs: `{ summary, actionItems: actions, meetingTitle, participants, clickupListId, teamsChannelId }`
- **Connections:**
  - **Input ←** Summary Exists? (success branch)
  - **Output →** Merge Summary & Actions
- **Edge cases / failures:**
  - If actionItems is already an object/array of objects, this code only handles string vs other; might pass unexpected structures through.
  - Splitting on “-” can over-split hyphenated text.

#### Node: Merge Summary & Actions
- **Type / role:** `Merge` — combines streams.
- **Configuration choices:**
  - Mode: `mergeByIndex`
- **Connections:**
  - **Input ←** Generate Action Items
  - **Output →** Prepare ClickUp Task
- **Edge cases / failures:**
  - As wired, it appears to receive only one input, so merge is effectively redundant unless a second input is added later.
  - `mergeByIndex` requires aligned item indices when two inputs exist.

---

### Block 1.5 — Storage (ClickUp) & Notification (Teams)
**Overview:** Formats a ClickUp task from the merged content, creates it, then formats and sends a Teams HTML message containing the summary and link.

**Nodes involved:**
- Prepare ClickUp Task
- Create ClickUp Task
- Prepare Teams Message
- Send Summary to Teams
- (Sticky note) Notifications & Storage

#### Node: Prepare ClickUp Task
- **Type / role:** `Set` — maps merged data to ClickUp task fields.
- **Configuration choices (interpreted):**
  - Downstream expects:
    - `taskName`
    - `assignees`
    - likely `description` (or similar) for task body
  - The JSON does not show fields; must be configured in node UI.
- **Connections:**
  - **Input ←** Merge Summary & Actions
  - **Output →** Create ClickUp Task
- **Edge cases / failures:**
  - Missing `clickupListId` or not applied to the ClickUp node (list selection is not shown in parameters).
  - Incorrect assignee format (ClickUp expects numeric user IDs, not emails).

#### Node: Create ClickUp Task
- **Type / role:** `ClickUp` — creates a task to store summary/action items.
- **Configuration choices:**
  - Task `name` = `{{$json.taskName}}`
  - Additional field `assignees` = `{{$json.assignees}}`
  - **List/space/workspace selection is not visible in JSON**; must be set in UI (often required).
- **Connections:**
  - **Input ←** Prepare ClickUp Task
  - **Output →** Prepare Teams Message
- **Edge cases / failures:**
  - Missing ClickUp credentials.
  - Wrong workspace/list permissions.
  - Assignees invalid → 400 error.

#### Node: Prepare Teams Message
- **Type / role:** `Set` — builds HTML message payload.
- **Configuration choices (interpreted):**
  - Must create `messageHtml` used by Teams node.
  - Typically includes summary, action items, and a URL to the created ClickUp task (from ClickUp node output).
- **Connections:**
  - **Input ←** Create ClickUp Task
  - **Output →** Send Summary to Teams
- **Edge cases / failures:**
  - If ClickUp output doesn’t include a usable URL, message will lack the link.
  - Mention formatting in Teams differs between chat and channel.

#### Node: Send Summary to Teams
- **Type / role:** `Microsoft Teams` — sends the final message.
- **Configuration choices:**
  - Resource: `chatMessage`
  - Content type: `html`
  - `message`: `{{$json.messageHtml}}`
  - `chatId` is empty in JSON (must be configured)
  - Node has a `teamsChannelId` elsewhere, but this node is configured for **chatId** specifically; channel posting may require a different resource or parameters.
- **Connections:**
  - **Input ←** Prepare Teams Message
  - **Output:** none
- **Edge cases / failures:**
  - Missing Teams credentials/consent scopes.
  - Incorrect chatId (or using channel id in chatId field).
  - HTML not accepted / stripped if malformed.

---

### Block 1.6 — Error Handling (Teams)
**Overview:** Formats an error message and sends it to Teams when early validation fails.

**Nodes involved:**
- Prepare Error Message
- Send Error to Teams
- (Sticky notes) Notifications & Storage; Trigger & Input (conceptually); Processing & Summarization (conceptually)

#### Node: Prepare Error Message
- **Type / role:** `Set` — constructs an alert payload.
- **Configuration choices (interpreted):**
  - Must output `errorText` consumed by Teams.
  - Likely includes meeting title, step that failed, raw API response, and remediation hint.
- **Connections:**
  - **Input ←** Transcription Request OK? (false) OR Summary Exists? (error branch)
  - **Output →** Send Error to Teams
- **Edge cases / failures:**
  - If it references fields not present (meetingTitle, status), expression errors can occur.

#### Node: Send Error to Teams
- **Type / role:** `Microsoft Teams` — pushes error alert.
- **Configuration choices:**
  - Resource: `chatMessage`
  - Content type: `html`
  - Message: `{{$json.errorText}}`
  - `chatId` empty in JSON (must be configured)
- **Connections:**
  - **Input ←** Prepare Error Message
- **Edge cases / failures:**
  - Same Teams chatId/credentials issues as the success notification.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 📋 Meeting Notes Distributor | Sticky Note | Documentation / orientation |  |  | ## How it works… (full note content in workflow) |
| Trigger & Input | Sticky Note | Documentation for input block |  |  | ## Trigger & Input… |
| Processing & Summarization | Sticky Note | Documentation for processing block |  |  | ## Processing & Summarization… |
| Notifications & Storage | Sticky Note | Documentation for output/error block |  |  | ## Notifications & Storage… |
| Start Workflow | Manual Trigger | Entry point |  | Set Meeting Info | ## Trigger & Input… |
| Set Meeting Info | Set | Define meeting metadata and routing ids | Start Workflow | Send to Transcription API | ## Trigger & Input… |
| Send to Transcription API | HTTP Request | Submit transcription job | Set Meeting Info | Transcription Request OK? | ## Trigger & Input… |
| Transcription Request OK? | IF | Gate success vs error branch (status) | Send to Transcription API | Wait for Processing; Prepare Error Message | ## Trigger & Input… |
| Wait for Processing | Wait | Delay before polling transcript | Transcription Request OK? | Fetch Transcript | ## Processing & Summarization… |
| Fetch Transcript | HTTP Request | Poll transcript results | Wait for Processing | Extract Transcript Text | ## Processing & Summarization… |
| Extract Transcript Text | Code | Normalize transcript + keep metadata | Fetch Transcript | AI Summarize | ## Processing & Summarization… |
| AI Summarize | HTTP Request | Call OpenAI to summarize | Extract Transcript Text | Extract Summary JSON | ## Processing & Summarization… |
| Extract Summary JSON | Code | Parse LLM response into summary/actions | AI Summarize | Summary Exists? | ## Processing & Summarization… |
| Summary Exists? | IF | Validate non-empty summary | Extract Summary JSON | Generate Action Items; Prepare Error Message | ## Processing & Summarization… |
| Generate Action Items | Code | Convert action items into array | Summary Exists? | Merge Summary & Actions | ## Processing & Summarization… |
| Merge Summary & Actions | Merge | Combine data streams | Generate Action Items | Prepare ClickUp Task | ## Processing & Summarization… |
| Prepare ClickUp Task | Set | Map fields for ClickUp task creation | Merge Summary & Actions | Create ClickUp Task | ## Notifications & Storage… |
| Create ClickUp Task | ClickUp | Create task in ClickUp | Prepare ClickUp Task | Prepare Teams Message | ## Notifications & Storage… |
| Prepare Teams Message | Set | Build HTML Teams message | Create ClickUp Task | Send Summary to Teams | ## Notifications & Storage… |
| Send Summary to Teams | Microsoft Teams | Send summary to Teams | Prepare Teams Message |  | ## Notifications & Storage… |
| Prepare Error Message | Set | Build error message payload | Transcription Request OK?; Summary Exists? | Send Error to Teams | ## Notifications & Storage… |
| Send Error to Teams | Microsoft Teams | Send error alert to Teams | Prepare Error Message |  | ## Notifications & Storage… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create the workflow**
   - Name it (e.g.) “Distribute summarized meeting notes with Microsoft Teams and ClickUp”.

2. **Add notes (optional but recommended)**
   - Add 4 **Sticky Note** nodes with the provided texts:
     - “📋 Meeting Notes Distributor”
     - “Trigger & Input”
     - “Processing & Summarization”
     - “Notifications & Storage”

3. **Trigger**
   - Add **Manual Trigger** node: **Start Workflow**.

4. **Input metadata**
   - Add a **Set** node: **Set Meeting Info**.
   - Add fields required by later nodes (minimum recommended):
     - `meetingTitle` (string)
     - `participants` (array or string)
     - `recordingUrl` (string; used in transcription POST body)
     - `clickupListId` (string or number)
     - `teamsChatId` (string) *or* `teamsChannelId` depending on where you post
   - Connect: **Start Workflow → Set Meeting Info**.

5. **Submit transcription job**
   - Add **HTTP Request** node: **Send to Transcription API**
     - Method: **POST**
     - URL: `https://api.exampletranscript.com/v1/transcribe` (replace with real provider)
     - Authentication: **Predefined Credential Type**
       - Credential type: `genericApiAuth` (configure with your provider API key/header)
     - Body: include `recordingUrl` (and any provider-required fields).
   - Connect: **Set Meeting Info → Send to Transcription API**.

6. **Gate on transcription request response**
   - Add **IF** node: **Transcription Request OK?**
     - Condition (Number): `{{$json.statusCode || 200}}` **smallerEqual** `299`
   - Connect: **Send to Transcription API → Transcription Request OK?**

7. **Wait**
   - Add **Wait** node: **Wait for Processing**
     - Mode: wait by time
     - Unit: seconds
     - Set a duration (e.g., 20–60 seconds, adjust as needed).
   - Connect **True** output of IF to **Wait for Processing**.

8. **Fetch transcript**
   - Add **HTTP Request** node: **Fetch Transcript**
     - Method: GET
     - URL: `{{ 'https://api.exampletranscript.com/v1/transcript/' + $json.id }}`
     - Authentication: same `genericApiAuth`
   - Connect: **Wait for Processing → Fetch Transcript**
   - Ensure the transcription POST response includes `id` (otherwise adjust expression to match provider schema).

9. **Normalize transcript**
   - Add **Code** node: **Extract Transcript Text**
   - Paste logic equivalent to:
     - Read transcript response (`transcript`/`text`)
     - Output: `transcriptText` and preserve metadata needed later.
   - Connect: **Fetch Transcript → Extract Transcript Text**
   - Recommended improvement when rebuilding: reference meeting info from the Set node explicitly (e.g., `$('Set Meeting Info').first().json.meetingTitle`) to avoid metadata loss.

10. **Summarize with OpenAI**
    - Add **HTTP Request** node: **AI Summarize**
      - Method: POST
      - URL: `https://api.openai.com/v1/chat/completions`
      - Authentication: Predefined credential type `openAiApi` (configure OpenAI credential in n8n).
      - Body must include:
        - `model` (choose)
        - `messages` that instruct JSON output with `summary` and `action_items`
        - Provide `transcriptText` in the prompt.
    - Connect: **Extract Transcript Text → AI Summarize**

11. **Parse LLM output**
    - Add **Code** node: **Extract Summary JSON**
      - Parse `choices[0].message.content`
      - Try JSON.parse; fallback to raw text as summary
      - Extract `summary` + `actionItems`
    - Connect: **AI Summarize → Extract Summary JSON**

12. **Validate summary**
    - Add **IF** node: **Summary Exists?**
      - Condition (String): `{{$json.summary}}` **isEmpty**
    - Connect: **Extract Summary JSON → Summary Exists?**
    - In the UI, ensure the branch mapping matches your intent:
      - If summary is empty → error branch
      - If summary exists → continue branch

13. **Normalize action items**
    - Add **Code** node: **Generate Action Items**
      - If `actionItems` is a string, split into bullet lines and output as array.
    - Connect the “summary exists” branch to **Generate Action Items**.

14. **Merge (optional)**
    - Add **Merge** node: **Merge Summary & Actions**
      - Mode: `mergeByIndex`
    - Connect: **Generate Action Items → Merge Summary & Actions**
    - (If you only have one stream, you can remove this node; keep it only if you later add a second parallel stream.)

15. **Prepare ClickUp fields**
    - Add **Set** node: **Prepare ClickUp Task**
      - Create:
        - `taskName` (e.g., `{{ $json.meetingTitle + ' - Meeting Summary' }}`)
        - `assignees` (array of ClickUp user IDs)
        - `description`/content combining summary + action items
        - any custom fields you need
    - Connect: **Merge Summary & Actions → Prepare ClickUp Task**

16. **Create ClickUp task**
    - Add **ClickUp** node: **Create ClickUp Task**
      - Operation: Create Task
      - Set the List (use `clickupListId` if node supports expression, or select list in UI)
      - Name: `{{$json.taskName}}`
      - Additional fields:
        - Assignees: `{{$json.assignees}}`
      - Configure ClickUp credentials (OAuth2/API token in n8n).
    - Connect: **Prepare ClickUp Task → Create ClickUp Task**

17. **Prepare Teams message**
    - Add **Set** node: **Prepare Teams Message**
      - Build `messageHtml` including:
        - meeting title
        - summary
        - formatted action items
        - link to ClickUp task (from ClickUp output; field name depends on node response)
    - Connect: **Create ClickUp Task → Prepare Teams Message**

18. **Send to Microsoft Teams**
    - Add **Microsoft Teams** node: **Send Summary to Teams**
      - Resource: `chatMessage`
      - Content type: `html`
      - Chat ID: select or set (must not be blank)
      - Message: `{{$json.messageHtml}}`
      - Configure Teams credentials (OAuth2) with appropriate permissions.
    - Connect: **Prepare Teams Message → Send Summary to Teams**

19. **Error branch**
    - Add **Set** node: **Prepare Error Message**
      - Build `errorText` including what failed and relevant IDs/status payload.
    - Add **Microsoft Teams** node: **Send Error to Teams**
      - Same resource/content type; set chatId; message = `{{$json.errorText}}`
    - Connect:
      - **Transcription Request OK? (false) → Prepare Error Message → Send Error to Teams**
      - **Summary validation error branch → Prepare Error Message → Send Error to Teams**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow converts raw meeting recordings or notes into concise, actionable summaries and distributes them automatically… (includes setup steps 1–7) | Sticky note: “📋 Meeting Notes Distributor” |
| Manual Trigger is used for ad-hoc processing; swap Set node for Webhook/form later | Sticky note: “Trigger & Input” |
| Wait then poll transcript; AI returns structured JSON summary; merge for unified payload | Sticky note: “Processing & Summarization” |
| Store in ClickUp and notify via Teams; parallel error notifications | Sticky note: “Notifications & Storage” |