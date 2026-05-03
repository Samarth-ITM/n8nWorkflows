Process audio with ElevenLabs via KIE.AI: transcribe, TTS, and isolate audio

https://n8nworkflows.xyz/workflows/process-audio-with-elevenlabs-via-kie-ai--transcribe--tts--and-isolate-audio-12203


# Process audio with ElevenLabs via KIE.AI: transcribe, TTS, and isolate audio

## 1. Workflow Overview

**Title:** Process audio with ElevenLabs via KIE.AI: transcribe, TTS, and isolate audio

This n8n workflow offers **three independent audio-processing pipelines** powered by **KIE.AI’s job API**, using **ElevenLabs models** behind the scenes:

1. **Speech-to-Text (Transcription)**: Submit an audio URL → poll job status → extract transcribed text.
2. **Text-to-Speech (TTS)**: Submit text → poll job status → extract generated audio URL.
3. **Audio Isolation**: Submit an audio URL → poll job status → extract isolated/denoised audio URL.

Each pipeline uses the same asynchronous pattern:  
**Create task → Wait 5s → Check status → Switch on state → loop until success/fail**.

### 1.1 Entry / Start
- Triggered manually via **Manual Trigger** (only connected to the Transcription branch by default).

### 1.2 Transcription block (Speech-to-Text)
- Provides speech recognition (optionally diarization + audio event tagging).

### 1.3 TTS block (Text-to-Speech)
- Produces speech audio from text using a fixed voice (“Rachel”) and stability/similarity parameters.

### 1.4 Audio isolation block
- Removes background noise / isolates voice or main audio from an audio file.

---

## 2. Block-by-Block Analysis

### Block A — Global notes / context
**Overview:** Documentation and author contact information embedded as sticky notes; not executed.  
**Nodes involved:** `Main Sticky Note`, `Sticky Note7`

#### Node: Main Sticky Note
- **Type / role:** Sticky Note (documentation)
- **Configuration:** Contains explanation, setup steps, and credential naming requirement (“KIE.AI”).
- **Connections:** None
- **Failure modes:** None (non-executing)

#### Node: Sticky Note7
- **Type / role:** Sticky Note (credits/contact)
- **Configuration:** Author bio + links (LinkedIn/portfolio/Upwork), email/phone.
- **Connections:** None
- **Failure modes:** None (non-executing)

---

### Block B — Transcription Section (ElevenLabs speech-to-text via KIE.AI)
**Overview:** Takes an `audio_url`, submits a speech-to-text job, polls until completion, then extracts `transcription`.  
**Nodes involved:**  
`When clicking ‘Execute workflow’` → `Set Audio URL` → `Submit Audio for Transcription` → `Wait for Transcription` → `Check Transcription Status` → `Switch Transcription Status` → `Extract Transcription Text`

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (start node)
- **Configuration:** No parameters
- **Outputs:** To `Set Audio URL`
- **Edge cases:** None

#### Node: Set Audio URL
- **Type / role:** Set node to define input variable(s)
- **Configuration choices:**
  - Sets `audio_url` to a sample MP3 URL (replace with your own).
- **Key variables/expressions:** Produces `$json.audio_url`
- **Input:** Manual Trigger
- **Output:** `Submit Audio for Transcription`
- **Edge cases:** Invalid/expired URL; URL not publicly reachable by KIE.AI.

#### Node: Submit Audio for Transcription
- **Type / role:** HTTP Request (submit async job)
- **Endpoint:** `POST https://api.kie.ai/api/v1/jobs/createTask`
- **Authentication:** HTTP Bearer Auth credential named **KIA.AI** (note the credential object name in JSON is `KIA.AI`; the sticky note suggests `KIE.AI`—ensure your credential name matches what you select in the node UI).
- **Body (JSON):**
  - `model`: `elevenlabs/speech-to-text`
  - `input.audio_url`: pulled from `$('Set Audio URL').item.json.audio_url`
  - `input.tag_audio_events`: `true`
  - `input.diarize`: `true`
- **Output:** to `Wait for Transcription`
- **Likely output fields used later:** `data.taskId`
- **Failure modes:**
  - 401/403 (bad API key)
  - 400 (bad model name, missing input, inaccessible audio URL)
  - Timeouts/network errors

#### Node: Wait for Transcription
- **Type / role:** Wait (delay between polling attempts)
- **Configuration:** 5 seconds
- **Output:** to `Check Transcription Status`
- **Edge cases:** Long-running jobs cause many loops; consider increasing wait time for rate limiting.

#### Node: Check Transcription Status
- **Type / role:** HTTP Request (poll job status)
- **Endpoint:** `GET https://api.kie.ai/api/v1/jobs/recordInfo` (sent as query)
- **Auth:** HTTP Bearer Auth (same credential)
- **Query params:**
  - `taskId = {{ $json.data.taskId }}`
- **Input:** from Wait (which passes along the original job creation response unless altered)
- **Output:** to `Switch Transcription Status`
- **Failure modes:**
  - `data.taskId` missing (if createTask response differs or earlier node failed)
  - 404/400 if taskId invalid
  - 429 if polling too frequently

#### Node: Switch Transcription Status
- **Type / role:** Switch (route by job state)
- **Configuration choices:**
  - Compares `{{ $json.data.state }}` to: `fail`, `success`, `generating`, `queuing`, `waiting`
  - `onError: continueRegularOutput` (prevents workflow from halting on switch evaluation errors; may hide issues)
- **Outputs / routing behavior:**
  - `fail` → loops back to `Submit Audio for Transcription` (as wired)
  - `success` → `Extract Transcription Text`
  - `generating|queuing|waiting` → `Wait for Transcription` (poll again)
- **Edge cases / issues:**
  - The **fail path currently re-submits** the transcription job instead of ending/error handling; this can create repeated tasks and extra cost.
  - If API introduces new states, they will go to default output (not explicitly connected), potentially stopping the loop.

#### Node: Extract Transcription Text
- **Type / role:** Code node (parse result payload)
- **Logic:**  
  - Reads `items[0].json.data.resultJson`
  - `JSON.parse(...)`
  - Returns `transcription: result.resultObject.text`
- **Output:** Final transcription text in `$json.transcription`
- **Failure modes:**
  - `data.resultJson` missing/not valid JSON until job success
  - `resultObject.text` path mismatch if API schema changes

---

### Block C — Text-to-Speech Section (ElevenLabs TTS via KIE.AI)
**Overview:** Takes `text`, submits a TTS job, polls until completion, then extracts a generated `audio_url`.  
**Nodes involved:** `Set Text Input` → `Submit Text for Speech Generation` → `Wait for Speech Generation` → `Check Speech Generation Status` → `Switch Speech Generation Status` → `Extract Audio URL`

> Note: This section is **not connected** to the Manual Trigger in the provided workflow; it must be executed by manually running from its first node or by adding connections.

#### Node: Set Text Input
- **Type / role:** Set node (defines the text to speak)
- **Configuration:** Sets `text = "Enter your text to convert to speech"`
- **Output:** `Submit Text for Speech Generation`
- **Edge cases:** Empty text may cause API validation errors.

#### Node: Submit Text for Speech Generation
- **Type / role:** HTTP Request (submit async job)
- **Endpoint:** `POST https://api.kie.ai/api/v1/jobs/createTask`
- **Auth:** HTTP Bearer Auth credential
- **Body (JSON):**
  - `model`: `elevenlabs/text-to-speech-multilingual-v2`
  - `input.text`: `{{ $json.text }}`
  - `input.voice`: `Rachel`
  - `input.stability`: `0.5`
  - `input.similarity_boost`: `0.75`
  - `input.style`: `0`
  - `input.speed`: `1`
  - `input.timestamps`: `false`
- **Output:** `Wait for Speech Generation`
- **Failure modes:** auth, invalid parameters, unsupported voice/model via KIE.AI, rate limiting.

#### Node: Wait for Speech Generation
- **Type / role:** Wait (poll delay)
- **Configuration:** 5 seconds
- **Output:** `Check Speech Generation Status`

#### Node: Check Speech Generation Status
- **Type / role:** HTTP Request (poll)
- **Endpoint:** `GET https://api.kie.ai/api/v1/jobs/recordInfo`
- **Query:** `taskId = {{ $json.data.taskId }}`
- **Output:** `Switch Speech Generation Status`

#### Node: Switch Speech Generation Status
- **Type / role:** Switch on `data.state`
- **States handled:** `fail`, `success`, `generating`, `queuing`, `waiting`
- **Routing:**
  - `success` → `Extract Audio URL`
  - `generating|queuing|waiting` → `Wait for Speech Generation`
  - `fail` → loops back to `Submit Text for Speech Generation` (same concern: repeated tasks/cost)
- **Failure modes:** missing `data.state`, new/unhandled states.

#### Node: Extract Audio URL
- **Type / role:** Code node (parse result JSON)
- **Logic:** parses `data.resultJson`, returns `audio_url: result.resultUrls[0]`
- **Output:** `$json.audio_url` (generated speech file)
- **Failure modes:** `resultUrls` empty, parse errors, schema changes.

---

### Block D — Audio Isolation Section (ElevenLabs audio isolation via KIE.AI)
**Overview:** Submits an audio isolation job (denoise/background removal), polls until completion, then extracts `isolated_audio_url`.  
**Nodes involved:** `Set Audio URL 1` → `Submit Audio for Isolation` → `Wait for Isolation` → `Check Isolation Status` → `Switch Isolation Status` → `Extract Isolated Audio URL`

> Note: This section is **not connected** to the Manual Trigger in the provided workflow.

#### Node: Set Audio URL 1
- **Type / role:** Set node (defines source audio URL)
- **Configuration:** `audio_url = YOUR_AUDIO_FILE_URL` (placeholder)
- **Output:** `Submit Audio for Isolation`
- **Edge cases:** Placeholder not replaced → guaranteed API failure.

#### Node: Submit Audio for Isolation
- **Type / role:** HTTP Request (submit async job)
- **Endpoint:** `POST https://api.kie.ai/api/v1/jobs/createTask`
- **Auth:** HTTP Bearer Auth credential
- **Body (JSON):**
  - `model`: `elevenlabs/audio-isolation`
  - `input.audio_url`: `{{ $('Set Audio URL 1').item.json.audio_url }}`
- **Output:** `Wait for Isolation`
- **Failure modes:** auth, inaccessible audio URL, unsupported format/length.

#### Node: Wait for Isolation
- **Type / role:** Wait (poll delay)
- **Configuration:** 5 seconds
- **Output:** `Check Isolation Status`

#### Node: Check Isolation Status
- **Type / role:** HTTP Request (poll)
- **Endpoint:** `GET https://api.kie.ai/api/v1/jobs/recordInfo`
- **Query:** `taskId = {{ $json.data.taskId }}`
- **Output:** `Switch Isolation Status`

#### Node: Switch Isolation Status
- **Type / role:** Switch on `data.state`
- **States handled:** `fail`, `success`, `generating`, `queuing`, `waiting`
- **Routing:**
  - `success` → `Extract Isolated Audio URL`
  - `generating|queuing|waiting` → `Wait for Isolation`
  - `fail` → loops back to `Submit Audio for Isolation` (same concern: repeated tasks/cost)
- **Failure modes:** missing fields, unhandled new states.

#### Node: Extract Isolated Audio URL
- **Type / role:** Code node
- **Logic:** parses `data.resultJson`, returns `isolated_audio_url: result.resultUrls[0]`
- **Output:** `$json.isolated_audio_url`
- **Failure modes:** parse errors, missing `resultUrls`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note7 | Sticky Note | Credits/contact information | — | — | ## Muhammad Farooq Iqbal - Automation Expert & n8n Creator / Email, Phone, LinkedIn: https://linkedin.com/in/muhammadfarooqiqbal / Portfolio: https://mfarooqone.github.io/n8n/ / UpWork: https://www.upwork.com/freelancers/~011aeba159896e2eba |
| Main Sticky Note | Sticky Note | Workflow description + setup steps | — | — | ## How it works / Setup steps incl. API key from https://kie.ai/ and HTTP Bearer Auth credential naming guidance |
| Transcription Section | Sticky Note | Section label/documentation | — | — | ## Transcription Section: submit audio URL, poll, extract transcribed text |
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Set Audio URL | ## Transcription Section: submit audio URL, poll, extract transcribed text |
| Set Audio URL | Set | Provide `audio_url` for transcription | When clicking ‘Execute workflow’ | Submit Audio for Transcription | ## Transcription Section: submit audio URL, poll, extract transcribed text |
| Submit Audio for Transcription | HTTP Request | Create STT job on KIE.AI | Set Audio URL | Wait for Transcription | ## Transcription Section: submit audio URL, poll, extract transcribed text |
| Wait for Transcription | Wait | Polling delay | Submit Audio for Transcription; Switch Transcription Status | Check Transcription Status | ## Transcription Section: submit audio URL, poll, extract transcribed text |
| Check Transcription Status | HTTP Request | Poll job status | Wait for Transcription | Switch Transcription Status | ## Transcription Section: submit audio URL, poll, extract transcribed text |
| Switch Transcription Status | Switch | Route by job state | Check Transcription Status | Submit Audio for Transcription; Extract Transcription Text; Wait for Transcription | ## Transcription Section: submit audio URL, poll, extract transcribed text |
| Extract Transcription Text | Code | Parse resultJson → `transcription` | Switch Transcription Status | — | ## Transcription Section: submit audio URL, poll, extract transcribed text |
| Text-to-Speech Section | Sticky Note | Section label/documentation | — | — | ## Text-to-Speech Section: submit text, poll, extract generated audio URL |
| Set Text Input | Set | Provide `text` for TTS | — | Submit Text for Speech Generation | ## Text-to-Speech Section: submit text, poll, extract generated audio URL |
| Submit Text for Speech Generation | HTTP Request | Create TTS job on KIE.AI | Set Text Input | Wait for Speech Generation | ## Text-to-Speech Section: submit text, poll, extract generated audio URL |
| Wait for Speech Generation | Wait | Polling delay | Submit Text for Speech Generation; Switch Speech Generation Status | Check Speech Generation Status | ## Text-to-Speech Section: submit text, poll, extract generated audio URL |
| Check Speech Generation Status | HTTP Request | Poll job status | Wait for Speech Generation | Switch Speech Generation Status | ## Text-to-Speech Section: submit text, poll, extract generated audio URL |
| Switch Speech Generation Status | Switch | Route by job state | Check Speech Generation Status | Submit Text for Speech Generation; Extract Audio URL; Wait for Speech Generation | ## Text-to-Speech Section: submit text, poll, extract generated audio URL |
| Extract Audio URL | Code | Parse resultJson → `audio_url` | Switch Speech Generation Status | — | ## Text-to-Speech Section: submit text, poll, extract generated audio URL |
| Audio Isolation Section | Sticky Note | Section label/documentation | — | — | ## Audio Isolation Section: submit audio URL, poll, extract isolated audio URL |
| Set Audio URL 1 | Set | Provide `audio_url` for isolation | — | Submit Audio for Isolation | ## Audio Isolation Section: submit audio URL, poll, extract isolated audio URL |
| Submit Audio for Isolation | HTTP Request | Create isolation job on KIE.AI | Set Audio URL 1 | Wait for Isolation | ## Audio Isolation Section: submit audio URL, poll, extract isolated audio URL |
| Wait for Isolation | Wait | Polling delay | Submit Audio for Isolation; Switch Isolation Status | Check Isolation Status | ## Audio Isolation Section: submit audio URL, poll, extract isolated audio URL |
| Check Isolation Status | HTTP Request | Poll job status | Wait for Isolation | Switch Isolation Status | ## Audio Isolation Section: submit audio URL, poll, extract isolated audio URL |
| Switch Isolation Status | Switch | Route by job state | Check Isolation Status | Submit Audio for Isolation; Extract Isolated Audio URL; Wait for Isolation | ## Audio Isolation Section: submit audio URL, poll, extract isolated audio URL |
| Extract Isolated Audio URL | Code | Parse resultJson → `isolated_audio_url` | Switch Isolation Status | — | ## Audio Isolation Section: submit audio URL, poll, extract isolated audio URL |

---

## 4. Reproducing the Workflow from Scratch

1. **Create credentials (required for all HTTP nodes)**
   1. In n8n: *Credentials* → **New**
   2. Choose **HTTP Bearer Auth**
   3. Name it (recommendation from notes: `"KIE.AI"`; workflow nodes currently reference a credential object named `"KIA.AI"` in JSON—pick one name and select it in every HTTP Request node)
   4. Paste your **KIE.AI API key** (from https://kie.ai/)

2. **Create Sticky Notes (optional but included)**
   1. Add a **Sticky Note** named `Main Sticky Note`; paste the “How it works / Setup steps” content.
   2. Add a **Sticky Note** named `Sticky Note7`; paste the author contact/links content.
   3. Add three **Sticky Notes** named `Transcription Section`, `Text-to-Speech Section`, `Audio Isolation Section` with their respective section descriptions.

3. **Build Block B: Transcription**
   1. Add **Manual Trigger** node named `When clicking ‘Execute workflow’`.
   2. Add **Set** node named `Set Audio URL`
      - Add field `audio_url` (String)
      - Set it to your publicly accessible audio URL (MP3/WAV, etc.)
   3. Connect: `Manual Trigger` → `Set Audio URL`
   4. Add **HTTP Request** named `Submit Audio for Transcription`
      - Method: **POST**
      - URL: `https://api.kie.ai/api/v1/jobs/createTask`
      - Authentication: **Generic credential type** → **HTTP Bearer Auth**
      - Select your bearer credential
      - Body: **JSON**
      - JSON body fields:
        - `model`: `elevenlabs/speech-to-text`
        - `input.audio_url`: expression referencing Set node output (e.g. `{{$json.audio_url}}` if directly connected; or `{{ $('Set Audio URL').item.json.audio_url }}`)
        - `input.tag_audio_events`: `true`
        - `input.diarize`: `true`
   5. Add **Wait** node named `Wait for Transcription`
      - Wait for: **5 seconds**
   6. Add **HTTP Request** named `Check Transcription Status`
      - Method: **GET**
      - URL: `https://api.kie.ai/api/v1/jobs/recordInfo`
      - Send Query Parameters: **on**
      - Query parameter: `taskId = {{$json.data.taskId}}`
      - Authentication: same bearer credential
   7. Add **Switch** node named `Switch Transcription Status`
      - Switch value: `{{$json.data.state}}` (implemented as rules)
      - Create outputs (rules) for: `fail`, `success`, `generating`, `queuing`, `waiting`
   8. Add **Code** node named `Extract Transcription Text`
      - JS: parse `items[0].json.data.resultJson` and return `{ transcription: result.resultObject.text }`
   9. Connect nodes:
      - `Set Audio URL` → `Submit Audio for Transcription` → `Wait for Transcription` → `Check Transcription Status` → `Switch Transcription Status`
      - Switch routing:
        - `success` → `Extract Transcription Text`
        - `generating|queuing|waiting` → `Wait for Transcription` (loop)
        - `fail` → (to match the provided workflow) `Submit Audio for Transcription` (re-submit). Recommended alternative: route to an error handling node instead.

4. **Build Block C: Text-to-Speech**
   1. Add **Set** node `Set Text Input`
      - Field `text` (String): your desired input text
   2. Add **HTTP Request** `Submit Text for Speech Generation`
      - POST `https://api.kie.ai/api/v1/jobs/createTask`
      - Bearer auth credential
      - JSON body:
        - `model`: `elevenlabs/text-to-speech-multilingual-v2`
        - `input.text`: `{{$json.text}}`
        - `input.voice`: `Rachel`
        - `input.stability`: `0.5`
        - `input.similarity_boost`: `0.75`
        - `input.style`: `0`
        - `input.speed`: `1`
        - `input.timestamps`: `false`
   3. Add **Wait** `Wait for Speech Generation` (5 seconds)
   4. Add **HTTP Request** `Check Speech Generation Status`
      - GET `https://api.kie.ai/api/v1/jobs/recordInfo`
      - Query `taskId = {{$json.data.taskId}}`
      - Bearer auth
   5. Add **Switch** `Switch Speech Generation Status` with the same 5 states.
   6. Add **Code** `Extract Audio URL`
      - Parse `data.resultJson` → return `{ audio_url: result.resultUrls[0] }`
   7. Connect:
      - `Set Text Input` → `Submit Text for Speech Generation` → `Wait for Speech Generation` → `Check Speech Generation Status` → `Switch Speech Generation Status`
      - Switch:
        - `success` → `Extract Audio URL`
        - `generating|queuing|waiting` → `Wait for Speech Generation`
        - `fail` → `Submit Text for Speech Generation` (or error handling)

5. **Build Block D: Audio Isolation**
   1. Add **Set** `Set Audio URL 1`
      - Field `audio_url` with your audio file URL
   2. Add **HTTP Request** `Submit Audio for Isolation`
      - POST `https://api.kie.ai/api/v1/jobs/createTask`
      - Bearer auth
      - JSON body:
        - `model`: `elevenlabs/audio-isolation`
        - `input.audio_url`: reference the set value (e.g. `{{$json.audio_url}}` if directly connected)
   3. Add **Wait** `Wait for Isolation` (5 seconds)
   4. Add **HTTP Request** `Check Isolation Status`
      - GET `https://api.kie.ai/api/v1/jobs/recordInfo`
      - Query `taskId = {{$json.data.taskId}}`
      - Bearer auth
   5. Add **Switch** `Switch Isolation Status` with the same 5 states.
   6. Add **Code** `Extract Isolated Audio URL`
      - Parse `data.resultJson` → return `{ isolated_audio_url: result.resultUrls[0] }`
   7. Connect:
      - `Set Audio URL 1` → `Submit Audio for Isolation` → `Wait for Isolation` → `Check Isolation Status` → `Switch Isolation Status`
      - Switch:
        - `success` → `Extract Isolated Audio URL`
        - `generating|queuing|waiting` → `Wait for Isolation`
        - `fail` → `Submit Audio for Isolation` (or error handling)

6. **(Optional) Add additional entry points**
   - To run TTS or Isolation from the same manual trigger, add connections from the Manual Trigger to `Set Text Input` and/or `Set Audio URL 1`.  
   - If you connect all branches, consider using separate triggers or a Switch/IF to choose which branch to run.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| KIE.AI API key required; create HTTP Bearer Auth credential in n8n | https://kie.ai/ |
| Author / workflow help & customization contact | Email: mfarooqiqbal143@gmail.com / Phone: +923036991118 |
| LinkedIn | https://linkedin.com/in/muhammadfarooqiqbal |
| Portfolio | https://mfarooqone.github.io/n8n/ |
| Upwork profile | https://www.upwork.com/freelancers/~011aeba159896e2eba |