Create script-to-video avatar videos with GPT-4.1-mini, HeyGen, Google Drive, Gmail and Slack

https://n8nworkflows.xyz/workflows/create-script-to-video-avatar-videos-with-gpt-4-1-mini--heygen--google-drive--gmail-and-slack-11829


# Create script-to-video avatar videos with GPT-4.1-mini, HeyGen, Google Drive, Gmail and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** AI-Powered Script-to-Video Automation Workflow with Loveable UI And HeyGen  
**Declared title (user):** Create script-to-video avatar videos with GPT-4.1-mini, HeyGen, Google Drive, Gmail and Slack

**Purpose:**  
This workflow receives a script request via webhook, rewrites and structures it with an OpenAI chat model (via LangChain nodes), submits it to HeyGen to generate an avatar video, polls HeyGen until the video is ready, downloads the resulting video, uploads it to Google Drive, then generates and sends completion notifications via Gmail and Slack. A separate error-trigger path sends a Slack alert if any node fails.

**Primary use cases:**
- Automated “script → avatar video” production pipelines driven by a UI/form (e.g., Loveable UI)
- Internal content ops: marketing snippets, onboarding videos, product updates
- Standardized notification output (email + Slack) upon delivery

### Logical Blocks
1. **1.1 Input Reception & Normalization**: Webhook receives request and Code node parses/normalizes payload.
2. **1.2 AI Script Rewriting & Structuring (LLM + Memory + Structured Output)**: Agent rewrites script into a HeyGen-ready structure and validates JSON output.
3. **1.3 HeyGen Video Job Submission**: HTTP request submits the generation job to HeyGen.
4. **1.4 Status Polling Loop**: Wait → status check → IF complete? else wait and recheck.
5. **1.5 Video Retrieval & Storage**: Download generated video and upload to Google Drive.
6. **1.6 AI Notification Generation & Dispatch**: Agent produces notification payload; send Slack + Gmail.
7. **1.7 Global Error Handling**: Error Trigger → Slack error alert.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Normalization

**Overview:**  
Receives a script generation request from an external client and converts the inbound payload into the fields expected downstream (script text, metadata, preferences).

**Nodes involved:**
- Receive Script Request 
- Parse Incoming Script Payload

#### Node: Receive Script Request 
- **Type / role:** Webhook (entry point) – receives HTTP requests from external systems.
- **Configuration (interpreted):**
  - Webhook node is present with a dedicated `webhookId`. The JSON does not expose method/path/response settings, so these must be configured in n8n UI (commonly `POST` with JSON body).
- **Key variables/expressions:** Not provided in JSON. Expected to output `body`, `headers`, `query`.
- **Connections:**
  - **Output →** Parse Incoming Script Payload (main)
- **Potential failures / edge cases:**
  - Invalid JSON body / missing required fields (script text, language, avatar settings).
  - Authentication/verification not shown (consider HMAC token or header check).
  - Large payload sizes; ensure webhook and reverse proxy limits fit.
- **Version notes:** typeVersion 2.1.

#### Node: Parse Incoming Script Payload
- **Type / role:** Code node – transforms the webhook input into a normalized internal schema.
- **Configuration (interpreted):**
  - Parameters are empty in JSON; in practice, this node should:
    - Extract script text and any options (tone, duration, audience, CTA, avatar/voice, aspect ratio).
    - Provide defaults when fields are missing.
    - Produce a stable JSON structure for the AI agent.
- **Key variables/expressions:** Not shown; likely uses `items[0].json.body`.
- **Connections:**
  - **Input ←** Receive Script Request 
  - **Output →** Rewrite & Structure Video Script  (main)
- **Potential failures / edge cases:**
  - Runtime errors in JS due to undefined paths.
  - Non-UTF8/escaped content.
  - Missing required fields leading to downstream HeyGen errors.
- **Version notes:** typeVersion 2.

---

### 2.2 AI Script Rewriting & Structuring (LLM + Memory + Structured Output)

**Overview:**  
Uses an OpenAI chat model to rewrite the raw script into a structured, validated JSON format suitable for HeyGen generation (e.g., scenes, voice, pacing), optionally using short-term memory context.

**Nodes involved:**
- Rewrite & Structure Video Script 
- LLM Engine for Script Rewriting
- Apply Script Writing Memory Context
- Validate Structured Script Output

#### Node: LLM Engine for Script Rewriting
- **Type / role:** LangChain Chat Model (OpenAI) – provides the LLM used by the agent.
- **Configuration (interpreted):**
  - Model is implied by the user title (“GPT-4.1-mini”), but the node parameters are empty in JSON; you must set:
    - Provider: OpenAI
    - Model: `gpt-4.1-mini` (or your available equivalent)
    - Temperature and max tokens appropriate for structured JSON.
  - Requires OpenAI credentials in n8n.
- **Connections:**
  - **Output (ai_languageModel) →** Rewrite & Structure Video Script 
- **Potential failures / edge cases:**
  - OpenAI auth/quota errors, rate limits.
  - Output not conforming to expected schema without robust prompting + parser.
- **Version notes:** typeVersion 1.3.

#### Node: Apply Script Writing Memory Context
- **Type / role:** LangChain Memory Buffer Window – supplies recent conversation/context to the agent.
- **Configuration (interpreted):**
  - Buffer window size and memory key must be configured in UI; JSON omits it.
  - Used to keep consistent style/brand voice across requests (within execution context).
- **Connections:**
  - **Output (ai_memory) →** Rewrite & Structure Video Script 
- **Potential failures / edge cases:**
  - Memory not persisted across executions unless using external store; buffer window typically only makes sense within a single run unless configured otherwise.
- **Version notes:** typeVersion 1.3.

#### Node: Validate Structured Script Output
- **Type / role:** LangChain Structured Output Parser – validates/forces agent output into a schema.
- **Configuration (interpreted):**
  - Schema definition is not visible in JSON; must be configured.
  - Expected to enforce a JSON object that HeyGen request builder can use (e.g., `title`, `scenes[]`, `voice_id`, `avatar_id`, `script`).
- **Connections:**
  - **Output (ai_outputParser) →** Rewrite & Structure Video Script 
- **Potential failures / edge cases:**
  - Schema mismatch: agent produces fields not allowed or missing required fields.
  - Parser failures causing workflow error (caught by Error Trigger path).
- **Version notes:** typeVersion 1.3.

#### Node: Rewrite & Structure Video Script 
- **Type / role:** LangChain Agent – orchestrates prompt + memory + model + output parsing to generate structured script.
- **Configuration (interpreted):**
  - Parameters empty in JSON; must define:
    - System + user instructions: rewriting goals, constraints (length, tone), and required JSON schema.
    - Tools (if any) are not shown; likely none.
- **Connections:**
  - **Inputs:**
    - From Parse Incoming Script Payload (main)
    - From LLM Engine for Script Rewriting (ai_languageModel)
    - From Apply Script Writing Memory Context (ai_memory)
    - From Validate Structured Script Output (ai_outputParser)
  - **Output →** Submit Video Generation Request to HeyGen (main)
- **Potential failures / edge cases:**
  - Hallucinated IDs (avatar/voice) that HeyGen rejects.
  - Overlong scripts causing HeyGen limits or large generation time.
- **Version notes:** typeVersion 2.

---

### 2.3 HeyGen Video Job Submission

**Overview:**  
Creates a HeyGen video generation job from the structured script via HTTP request.

**Nodes involved:**
- Submit Video Generation Request to HeyGen

#### Node: Submit Video Generation Request to HeyGen
- **Type / role:** HTTP Request – calls HeyGen API to start video generation.
- **Configuration (interpreted):**
  - Parameters are empty in JSON; must configure in UI:
    - Method: typically `POST`
    - URL: HeyGen “create video” endpoint
    - Auth: HeyGen API key (Header `Authorization` / `x-api-key` depending on HeyGen spec)
    - Body: map from structured script output (agent result) into HeyGen request format.
    - Response: must capture a `video_id` / `task_id` for polling.
- **Connections:**
  - **Input ←** Rewrite & Structure Video Script  (main)
  - **Output →** Wait Before Checking Video Status (main)
- **Potential failures / edge cases:**
  - 401/403 due to invalid API key.
  - 400 due to invalid payload (wrong avatar_id/voice_id, missing script fields).
  - 429 rate limiting, 5xx transient failures.
- **Version notes:** typeVersion 4.2.

---

### 2.4 Status Polling Loop (Wait → Check → IF)

**Overview:**  
Waits a fixed time, checks HeyGen job status, and loops until completion. When completed, proceeds to download.

**Nodes involved:**
- Wait Before Checking Video Status
- Check HeyGen Video Status1
- Wait Before Rechecking Status
- Evaluate Video Completion Status

#### Node: Wait Before Checking Video Status
- **Type / role:** Wait – delays before first status check (reduces immediate polling).
- **Configuration (interpreted):**
  - Parameters not shown; in UI set a duration (e.g., 30–120 seconds) or “wait until”.
  - Has a `webhookId` meaning it can resume via webhook; standard Wait node behavior.
- **Connections:**
  - **Input ←** Submit Video Generation Request to HeyGen
  - **Output →** Check HeyGen Video Status1
- **Potential failures / edge cases:**
  - If using “wait for webhook”, ensure external callback exists; otherwise use time-based wait.
- **Version notes:** typeVersion 1.1.

#### Node: Check HeyGen Video Status1
- **Type / role:** HTTP Request – queries HeyGen for the job status.
- **Configuration (interpreted):**
  - Must configure:
    - Method: `GET` (or `POST` depending on HeyGen)
    - URL: status endpoint including `video_id` from submission response
    - Auth headers same as submission
    - Response parsing: read `status` and possibly `video_url` when ready
- **Connections:**
  - **Input ←** Wait Before Checking Video Status
  - **Output →** Wait Before Rechecking Status
- **Potential failures / edge cases:**
  - Missing/incorrect `video_id` expression.
  - HeyGen returns “processing”, “queued”, “failed”; “failed” should be handled (not shown).
- **Version notes:** typeVersion 4.2.

#### Node: Wait Before Rechecking Status
- **Type / role:** Wait – delays between subsequent status checks.
- **Configuration (interpreted):**
  - Set duration (e.g., 30–120 seconds) to avoid rate limits.
- **Connections:**
  - **Input ←** Check HeyGen Video Status1
  - **Output →** Evaluate Video Completion Status
- **Potential failures / edge cases:**
  - Excessive loop time: consider max attempts guard (not present).
- **Version notes:** typeVersion 1.1.

#### Node: Evaluate Video Completion Status
- **Type / role:** IF – routes execution based on whether the video is done.
- **Configuration (interpreted):**
  - Parameters empty in JSON; must define condition(s), e.g.:
    - `{{$json.status}} == "completed"` (true path)
    - else path loops back to Wait Before Checking Video Status
- **Connections:**
  - **Input ←** Wait Before Rechecking Status
  - **True output (main index 0) →** Download Generated HeyGen Video
  - **False output (main index 1) →** Wait Before Checking Video Status
- **Potential failures / edge cases:**
  - Status field name mismatch (e.g., `data.status` vs `status`).
  - No handling for terminal failure state (e.g., `failed`)—should alert and stop.
- **Version notes:** typeVersion 2.

---

### 2.5 Video Retrieval & Storage

**Overview:**  
Downloads the completed HeyGen video file and uploads it to Google Drive for persistence and sharing.

**Nodes involved:**
- Download Generated HeyGen Video
- Upload Generated Video to Google Drive

#### Node: Download Generated HeyGen Video
- **Type / role:** HTTP Request – fetches the final video content from HeyGen (binary download).
- **Configuration (interpreted):**
  - Must configure:
    - URL: `video_url` from HeyGen status response
    - Response: “File/Binary” mode in n8n
    - Binary property name (e.g., `data`)
- **Connections:**
  - **Input ←** Evaluate Video Completion Status (true path)
  - **Output →** Upload Generated Video to Google Drive
- **Potential failures / edge cases:**
  - Signed URL expiration if not downloaded promptly.
  - Large file sizes causing memory/time limits.
- **Version notes:** typeVersion 4.2.

#### Node: Upload Generated Video to Google Drive
- **Type / role:** Google Drive node – uploads video binary to Drive.
- **Configuration (interpreted):**
  - Must configure:
    - Operation: Upload
    - File: binary property from previous node
    - Destination folder ID/path
    - File name (likely derived from script title or request id)
    - Sharing settings if needed (not shown)
  - Requires Google OAuth2 credentials with Drive scope.
- **Connections:**
  - **Input ←** Download Generated HeyGen Video
  - **Output →** Generate Completion Notifications  (main)
- **Potential failures / edge cases:**
  - OAuth token expiration / insufficient scopes.
  - Upload size limits/timeouts.
  - Folder not found / permission denied.
- **Version notes:** typeVersion 3.

---

### 2.6 AI Notification Generation & Dispatch

**Overview:**  
Generates a structured notification payload (subject/body/slack message + links) using an LLM and then sends notifications to Gmail and Slack.

**Nodes involved:**
- Generate Completion Notifications 
- LLM Engine for Notification Generation
- Apply Notification Context Memory
- Validate Notification Output JSON
- Send Video Ready Slack Notification
- Send Video Ready Email Notification

#### Node: LLM Engine for Notification Generation
- **Type / role:** LangChain Chat Model (OpenAI) – model for composing notification text.
- **Configuration (interpreted):**
  - Set model (likely same `gpt-4.1-mini`) and conservative temperature for predictable formatting.
  - Requires OpenAI credentials.
- **Connections:**
  - **Output (ai_languageModel) →** Generate Completion Notifications 
- **Potential failures / edge cases:** rate limits, schema noncompliance without parser.
- **Version notes:** typeVersion 1.3.

#### Node: Apply Notification Context Memory
- **Type / role:** LangChain Memory Buffer Window – adds context for consistent notification style.
- **Connections:**
  - **Output (ai_memory) →** Generate Completion Notifications 
- **Potential failures / edge cases:** same as earlier memory node (persistence expectations).
- **Version notes:** typeVersion 1.3.

#### Node: Validate Notification Output JSON
- **Type / role:** Structured Output Parser – enforces notification JSON schema.
- **Configuration (interpreted):**
  - Must define schema, e.g.:
    - `email: {to, subject, html/text}`
    - `slack: {channel, text}`
    - `links: {driveUrl, downloadUrl}`
- **Connections:**
  - **Output (ai_outputParser) →** Generate Completion Notifications 
- **Potential failures / edge cases:** parser failure if model outputs invalid JSON.
- **Version notes:** typeVersion 1.3.

#### Node: Generate Completion Notifications 
- **Type / role:** LangChain Agent – creates final notification content using Drive upload output (file id/link).
- **Connections:**
  - **Input ←** Upload Generated Video to Google Drive (main)
  - **Inputs (AI):**
    - LLM Engine for Notification Generation (ai_languageModel)
    - Apply Notification Context Memory (ai_memory)
    - Validate Notification Output JSON (ai_outputParser)
  - **Outputs →**
    - Send Video Ready Email Notification (main)
    - Send Video Ready Slack Notification (main)
- **Potential failures / edge cases:**
  - Missing Drive share link (if file not shared) leading to useless notifications.
  - Agent output fields not matching Gmail/Slack node expectations.
- **Version notes:** typeVersion 2.

#### Node: Send Video Ready Slack Notification
- **Type / role:** Slack node – sends “video ready” message.
- **Configuration (interpreted):**
  - Parameters empty in JSON; must configure:
    - Auth: Slack OAuth/token credentials
    - Channel: static or from agent output
    - Message text: include Drive link and any metadata
- **Connections:**
  - **Input ←** Generate Completion Notifications 
- **Potential failures / edge cases:**
  - Missing permissions (chat:write), wrong channel ID/name.
  - Message formatting errors if expressions reference missing fields.
- **Version notes:** typeVersion 2.3.

#### Node: Send Video Ready Email Notification
- **Type / role:** Gmail node – sends completion email.
- **Configuration (interpreted):**
  - Must configure:
    - Auth: Google OAuth2 (Gmail scopes)
    - To/Subject/Body: from agent output
    - Optionally include Drive link or attach video (attachments not shown)
- **Connections:**
  - **Input ←** Generate Completion Notifications 
- **Potential failures / edge cases:**
  - Gmail sending limits, invalid recipients, missing scopes.
- **Version notes:** typeVersion 2.1.

---

### 2.7 Global Error Handling

**Overview:**  
Catches any workflow execution error and sends a Slack alert.

**Nodes involved:**
- Error Handler Trigger
- Slack: Send Error Alert

#### Node: Error Handler Trigger
- **Type / role:** Error Trigger – runs when the workflow errors.
- **Configuration (interpreted):**
  - Default behavior: triggers on any node failure in this workflow.
- **Connections:**
  - **Output →** Slack: Send Error Alert
- **Potential failures / edge cases:**
  - If Slack node is misconfigured, errors won’t be delivered.
- **Version notes:** typeVersion 1.

#### Node: Slack: Send Error Alert
- **Type / role:** Slack node – posts error details to Slack.
- **Configuration (interpreted):**
  - Must configure channel + message template including error context (node name, error message, execution URL).
- **Connections:**
  - **Input ←** Error Handler Trigger
- **Potential failures / edge cases:** Slack auth/permissions; missing fields from error trigger if template expects specific paths.
- **Version notes:** typeVersion 2.3.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Script Request  | Webhook | Entry point receiving script requests | — | Parse Incoming Script Payload |  |
| Parse Incoming Script Payload | Code | Normalize/shape incoming payload | Receive Script Request  | Rewrite & Structure Video Script  |  |
| Rewrite & Structure Video Script  | LangChain Agent | Rewrite + structure script into validated JSON | Parse Incoming Script Payload; (AI) LLM Engine for Script Rewriting; (AI) Apply Script Writing Memory Context; (AI) Validate Structured Script Output | Submit Video Generation Request to HeyGen |  |
| LLM Engine for Script Rewriting | OpenAI Chat Model (LangChain) | LLM backend for script rewriting | — | (AI) Rewrite & Structure Video Script  |  |
| Apply Script Writing Memory Context | Memory Buffer Window (LangChain) | Context memory for rewriting agent | — | (AI) Rewrite & Structure Video Script  |  |
| Validate Structured Script Output | Structured Output Parser (LangChain) | Enforce schema for rewritten script | — | (AI) Rewrite & Structure Video Script  |  |
| Submit Video Generation Request to HeyGen | HTTP Request | Create HeyGen generation job | Rewrite & Structure Video Script  | Wait Before Checking Video Status |  |
| Wait Before Checking Video Status | Wait | Delay before first status check | Submit Video Generation Request to HeyGen | Check HeyGen Video Status1 |  |
| Check HeyGen Video Status1 | HTTP Request | Query HeyGen job status | Wait Before Checking Video Status | Wait Before Rechecking Status |  |
| Wait Before Rechecking Status | Wait | Delay between polls | Check HeyGen Video Status1 | Evaluate Video Completion Status |  |
| Evaluate Video Completion Status | IF | Branch: completed vs still processing | Wait Before Rechecking Status | (true) Download Generated HeyGen Video; (false) Wait Before Checking Video Status |  |
| Download Generated HeyGen Video | HTTP Request | Download final video as binary | Evaluate Video Completion Status (true) | Upload Generated Video to Google Drive |  |
| Upload Generated Video to Google Drive | Google Drive | Upload video to Drive | Download Generated HeyGen Video | Generate Completion Notifications  |  |
| Generate Completion Notifications  | LangChain Agent | Produce structured Slack+Email notification content | Upload Generated Video to Google Drive; (AI) LLM Engine for Notification Generation; (AI) Apply Notification Context Memory; (AI) Validate Notification Output JSON | Send Video Ready Email Notification; Send Video Ready Slack Notification |  |
| LLM Engine for Notification Generation | OpenAI Chat Model (LangChain) | LLM backend for notification drafting | — | (AI) Generate Completion Notifications  |  |
| Apply Notification Context Memory | Memory Buffer Window (LangChain) | Context memory for notifications | — | (AI) Generate Completion Notifications  |  |
| Validate Notification Output JSON | Structured Output Parser (LangChain) | Enforce schema for notification payload | — | (AI) Generate Completion Notifications  |  |
| Send Video Ready Slack Notification | Slack | Notify stakeholders in Slack | Generate Completion Notifications  | — |  |
| Send Video Ready Email Notification | Gmail | Email stakeholders that video is ready | Generate Completion Notifications  | — |  |
| Error Handler Trigger | Error Trigger | Global failure entry point | — | Slack: Send Error Alert |  |
| Slack: Send Error Alert | Slack | Post error alert to Slack | Error Handler Trigger | — |  |
| Sticky Note8 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note1 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note2 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note3 | Sticky Note | Comment container (empty) | — | — |  |
| Sticky Note4 | Sticky Note | Comment container (empty) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add Webhook node**:  
   - Name: `Receive Script Request `  
   - Method: `POST`  
   - Response: typically “Respond immediately” with a simple JSON ack (optional).  
   - Note the generated webhook URL; your UI/client will call it.
3. **Add Code node**:  
   - Name: `Parse Incoming Script Payload`  
   - Parse `items[0].json.body` and output a normalized object, e.g.:
     - `script_raw`, `title`, `language`, `tone`, `targetDurationSec`, `avatarId`, `voiceId`, etc.  
   - Add defaults when missing.
4. **Add LangChain Chat Model (OpenAI) node**:  
   - Name: `LLM Engine for Script Rewriting`  
   - Credentials: OpenAI API key  
   - Model: `gpt-4.1-mini` (or closest available)  
   - Set temperature low (e.g., 0.2–0.5) for structured output reliability.
5. **Add Memory Buffer Window node**:  
   - Name: `Apply Script Writing Memory Context`  
   - Configure window size (e.g., 3–10). (If you don’t need memory, keep small or remove.)
6. **Add Structured Output Parser node**:  
   - Name: `Validate Structured Script Output`  
   - Define the schema you want the agent to produce (JSON with required fields for HeyGen).
7. **Add LangChain Agent node**:  
   - Name: `Rewrite & Structure Video Script `  
   - Prompt: instruct it to rewrite the script and output **only** valid JSON matching the parser schema.  
   - Connect:
     - `Parse Incoming Script Payload` → Agent (main)
     - `LLM Engine for Script Rewriting` → Agent (AI language model)
     - `Apply Script Writing Memory Context` → Agent (AI memory)
     - `Validate Structured Script Output` → Agent (AI output parser)
8. **Add HTTP Request node**:  
   - Name: `Submit Video Generation Request to HeyGen`  
   - Method: `POST`  
   - URL: HeyGen create-video endpoint (per your HeyGen API version)  
   - Auth: add required header(s) with HeyGen API key  
   - Body: map from agent JSON into HeyGen request format  
   - Ensure response includes a job/video id field (store it in JSON for later nodes).
9. **Add Wait node**:  
   - Name: `Wait Before Checking Video Status`  
   - Configure time-based wait (e.g., 60 seconds).
10. **Add HTTP Request node**:  
   - Name: `Check HeyGen Video Status1`  
   - Method: `GET`  
   - URL: HeyGen status endpoint using expression with the returned id (e.g., `{{$json.video_id}}`)  
   - Auth headers same as submission  
   - Output should expose `status` and when ready, a `video_url`.
11. **Add Wait node**:  
   - Name: `Wait Before Rechecking Status`  
   - Configure polling interval (e.g., 60 seconds).
12. **Add IF node**:  
   - Name: `Evaluate Video Completion Status`  
   - Condition: check HeyGen status equals “completed/ready” (match your API’s exact value).  
   - True branch: proceed to download  
   - False branch: loop back to `Wait Before Checking Video Status`  
   - (Recommended) Add an attempts counter to avoid infinite loops (not present in provided JSON).
13. **Add HTTP Request node (download)**:  
   - Name: `Download Generated HeyGen Video`  
   - Method: `GET`  
   - URL: expression pointing to `video_url` from status response  
   - Response: set to **download file** / output binary (choose a binary property name, e.g. `video`).
14. **Add Google Drive node**:  
   - Name: `Upload Generated Video to Google Drive`  
   - Credentials: Google OAuth2 with Drive scope  
   - Operation: Upload  
   - Binary property: `video` (or your chosen name)  
   - Destination folder: select folder / set folder ID  
   - File name: use expression (e.g., title + timestamp).
15. **Add OpenAI Chat Model node (notifications)**:  
   - Name: `LLM Engine for Notification Generation`  
   - Same OpenAI credentials; model `gpt-4.1-mini`  
   - Temperature low.
16. **Add Memory Buffer Window node (notifications)**:  
   - Name: `Apply Notification Context Memory`
17. **Add Structured Output Parser node (notifications)**:  
   - Name: `Validate Notification Output JSON`  
   - Define a schema containing fields needed by Slack + Gmail nodes (to, subject, body, slackChannel/text, driveLink).
18. **Add LangChain Agent node (notifications)**:  
   - Name: `Generate Completion Notifications `  
   - Prompt: take Drive upload output (file name, file id/link) and produce JSON matching the parser schema.
   - Connect AI ports: model + memory + output parser.
19. **Add Slack node**:  
   - Name: `Send Video Ready Slack Notification`  
   - Credentials: Slack OAuth/token with `chat:write`  
   - Channel: static or from agent output  
   - Message: expression from agent output.
20. **Add Gmail node**:  
   - Name: `Send Video Ready Email Notification`  
   - Credentials: Google OAuth2 with Gmail send scope  
   - To/Subject/Body: expressions from agent output.
21. **Wire the main flow connections** in this order:  
   `Webhook → Code → Script Agent → HeyGen Submit → Wait → Status Check → Wait → IF → (true) Download → Drive Upload → Notification Agent → (Slack + Gmail)`  
   and IF false path back to the first Wait.
22. **Add Error Trigger node**:  
   - Name: `Error Handler Trigger`
23. **Add Slack node for errors**:  
   - Name: `Slack: Send Error Alert`  
   - Channel: ops/errors  
   - Message: include error trigger fields (failed node name, error message, execution link).
24. **Connect**: `Error Handler Trigger → Slack: Send Error Alert`.
25. **Credentials checklist**:
   - OpenAI API key (for both Chat Model nodes)
   - HeyGen API key (HTTP Request headers)
   - Google OAuth2 (Drive + Gmail, may be separate credential entries/scopes)
   - Slack OAuth/token (Slack notifications + Slack error alert)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but contain no text. | No additional embedded guidance was provided in the workflow’s sticky notes. |
| Polling loop has no maximum retry/timeout guard in the provided configuration. | Consider adding an attempts counter + “failed” status handling to prevent infinite loops and to alert on HeyGen job failure. |