Summarize YouTube videos in Slack using AssemblyAI transcription and OpenAI

https://n8nworkflows.xyz/workflows/summarize-youtube-videos-in-slack-using-assemblyai-transcription-and-openai-12934


# Summarize YouTube videos in Slack using AssemblyAI transcription and OpenAI

## 1. Workflow Overview

**Title:** Summarize YouTube videos in Slack using AssemblyAI transcription and OpenAI  
**Workflow name (in n8n):** Sift- Youtube Video Summarizer

**Purpose:**  
Accept a YouTube link via a Slack slash command, verify the video is under a maximum duration (10 minutes), download/convert it to MP3 via RapidAPI, transcribe it with AssemblyAI, summarize the transcript with OpenAI (LangChain node), and post the formatted result back into Slack using the command’s `response_url`.

**Primary use cases:**
- Fast “in-channel” summaries of short YouTube videos
- Lightweight knowledge capture for teams without leaving Slack
- Optional “transcript mode” trigger (present as a `mode` field, though the current workflow always generates a summary)

### 1.1 Slack Input Reception & Normalization
Receives a Slack slash command payload, normalizes fields, and extracts the YouTube video ID and URL.

### 1.2 Duration Validation (Guardrail)
Calls a YouTube API (via RapidAPI) to fetch ISO-8601 duration, converts it to minutes, and rejects videos longer than 10 minutes with a Slack message.

### 1.3 Audio Download + AssemblyAI Transcription
Downloads an MP3 from RapidAPI, uploads it to AssemblyAI, submits a transcription job, then polls until completed.

### 1.4 Summarization + Slack Delivery
Uses OpenAI to generate a Slack-formatted summary from the transcript and posts it back to the channel via `response_url`.

---

## 2. Block-by-Block Analysis

### Block 1 — Slack Input Reception & Normalization
**Overview:**  
Triggers on Slack’s slash command webhook, extracts the provided YouTube URL/text, and determines a `mode` (summary vs transcript).

**Nodes involved:**  
- Receive Slack command  
- Normalize Slack payload  
- Extract YouTube video ID

#### Node: Receive Slack command
- **Type / role:** Webhook (entry point). Receives HTTP POST from Slack slash command.
- **Key configuration:**
  - **Path:** `sift`
  - **Method:** `POST`
  - **Immediate response text:** “Filtering the fluff… give me about 2–4 minutes...”
  - **Response mode:** set via `options.responseData` (so Slack gets an immediate acknowledgement).
- **Inputs / outputs:** No inputs. Output goes to **Normalize Slack payload**.
- **Edge cases / failures:**
  - Slack may send `application/x-www-form-urlencoded` by default; this workflow expects `body.text`, `body.command`, `body.response_url`. If Slack is not configured to send JSON or n8n doesn’t parse it as expected, mappings may be empty.
  - If Slack retries (timeouts), you can get duplicate executions unless you add deduplication.

#### Node: Normalize Slack payload
- **Type / role:** Set node. Normalizes Slack fields into consistent names.
- **Key configuration:**
  - Keeps all existing fields (`includeOtherFields: true`).
  - Creates:
    - `videoUrl` = `{{$json["body"]["text"] || ""}}`
    - `mode` = `{{$json.body.command === '/siftf' ? 'transcript' : 'summary'}}`
- **Inputs / outputs:** From **Receive Slack command** to **Extract YouTube video ID**.
- **Edge cases / failures:**
  - If `body.command` is missing, expression still resolves but defaults to `'summary'`.
  - `mode` is later read, but the workflow currently does not branch on it (summary is always produced).

#### Node: Extract YouTube video ID
- **Type / role:** Code node. Extracts a canonical YouTube URL and the 11-character video ID.
- **Key logic:**
  - Strips leading `/command` text: `raw.replace(/^\/\w+\s*/, '')`
  - Extracts first URL-like token or uses the remaining text.
  - Regex supports: `watch?v=`, `youtu.be/`, `shorts/`, `embed/`
- **Outputs fields:**
  - `text` (cleaned input)
  - `videoUrl` (best-effort URL)
  - `videoId` (or `null`)
- **Connections:** Output goes to **Check Video Duration**.
- **Edge cases / failures:**
  - If the input is not a valid YouTube URL/ID, `videoId` becomes `null`, causing downstream API calls to fail (YouTube duration check, MP3 conversion).
  - “Playlist” links or non-standard formats may not match the regex.

---

### Block 2 — Duration Validation (Guardrail)
**Overview:**  
Fetches the video duration via RapidAPI YouTube endpoint, converts ISO-8601 to minutes, and blocks processing if longer than 10 minutes.

**Nodes involved:**  
- Check Video Duration  
- Validate Duration  
- Is video longer than limit?  
- Send duration error to Slack

#### Node: Check Video Duration
- **Type / role:** HTTP Request. Calls a YouTube Data-like endpoint via RapidAPI to get `contentDetails.duration`.
- **Key configuration:**
  - **URL:** `https://youtube-v31.p.rapidapi.com/videos?part=contentDetails&id={{ $('Extract YouTube video ID').item.json.videoId }}`
  - **Auth:** HTTP Header Auth credential “RapidAPI”
  - Sends header: `x-rapidapi-host: youtube-v31.p.rapidapi.com`
- **Connections:** Output → **Validate Duration**
- **Edge cases / failures:**
  - Invalid or null `videoId` → likely 4xx or empty `items`.
  - RapidAPI quota/rate-limit errors (429), invalid key (401/403).
  - Response shape changes: `items[0].contentDetails.duration` missing → code node fails.

#### Node: Validate Duration
- **Type / role:** Code node. Parses ISO-8601 duration and enforces max 10 minutes.
- **Key logic:**
  - Reads: `$input.item.json.items[0].contentDetails.duration`
  - Parses `PT#H#M#S`, computes total minutes.
  - Pulls ID/URL from: `$('Extract YouTube video ID').first()`
  - Returns **non-throwing** error object:
    - If too long: `{ error: true, errorMessage, videoId, videoUrl }`
    - Else: `{ error: false, videoId, videoUrl, durationMinutes }`
- **Connections:** Output → **Is video longer than limit?**
- **Edge cases / failures:**
  - If `items` is empty or `duration` missing, `duration.match(...)` can throw.
  - If regex match returns null, `match[1]` access throws (should be guarded in robust versions).

#### Node: Is video longer than limit?
- **Type / role:** IF node. Routes based on `error === true`.
- **Condition:** `{{$json.error}}` equals `true`
- **Connections:**
  - **True** → Send duration error to Slack
  - **False** → Convert YouTube video to MP3
- **Edge cases / failures:**
  - If `error` is missing/non-boolean, strict validation may cause unexpected behavior; current code always sets it.

#### Node: Send duration error to Slack
- **Type / role:** HTTP Request. Posts an ephemeral response back via Slack `response_url`.
- **Key configuration:**
  - **URL:** `{{$('Receive Slack command').item.json.body.response_url}}`
  - **POST body:** `text = ❌ {{$json.errorMessage}}`
- **Connections:** Terminal node.
- **Edge cases / failures:**
  - `response_url` missing/expired → Slack returns error.
  - If Slack expects JSON but body is form-encoded (depending on node defaults), delivery may fail; in practice Slack response URLs accept JSON payloads—ensure request is JSON if needed.

---

### Block 3 — Audio Download + AssemblyAI Transcription
**Overview:**  
Downloads the YouTube audio as MP3 via RapidAPI, uploads it to AssemblyAI, starts a transcription job, then polls until the job status is `completed`.

**Nodes involved:**  
- Convert YouTube video to MP3  
- Start transcription job  
- Submit audio for transcription  
- Wait for transcription processing  
- Check transcription status  
- Extract transcript text  
- Is transcription complete?

#### Node: Convert YouTube video to MP3
- **Type / role:** HTTP Request. Downloads binary MP3 from RapidAPI converter.
- **Key configuration:**
  - **URL:** `https://youtube-mp3-audio-video-downloader.p.rapidapi.com/download-mp3/{{$json.videoId}}?quality=low`
  - **Auth:** HTTP Header Auth credential “RapidAPI”
  - Headers:
    - `x-rapidapi-host: youtube-mp3-audio-video-downloader.p.rapidapi.com`
    - `accept: application/octet-stream`
  - **Response handling:** `responseFormat = file`, binary saved to property **`mp3`**
  - **Never error:** enabled (`neverError: true`) and `fullResponse: true`
- **Connections:** Output → **Start transcription job**
- **Edge cases / failures:**
  - Because `neverError` is enabled, HTTP errors won’t stop the workflow automatically—downstream nodes may fail due to missing/invalid binary data.
  - Some videos cannot be converted due to restrictions; you may receive HTML/JSON instead of MP3, breaking upload.

#### Node: Start transcription job
- **Type / role:** HTTP Request. Uploads binary audio to AssemblyAI `/v2/upload`.
- **Key configuration:**
  - **Method:** POST
  - **URL:** `https://api.assemblyai.com/v2/upload`
  - **Auth:** HTTP Header Auth credential “AssemblyAI” (typically `authorization: <API_KEY>`)
  - **Body:** `binaryData` from field **`mp3`**; content type `application/octet-stream`
  - **Output:** JSON containing `upload_url`
- **Connections:** Output → **Submit audio for transcription**
- **Edge cases / failures:**
  - Missing binary field `mp3` (from prior step) → request fails.
  - AssemblyAI auth/quota errors, upload size/timeouts.

#### Node: Submit audio for transcription
- **Type / role:** HTTP Request. Creates transcript job in AssemblyAI.
- **Key configuration:**
  - **POST** `https://api.assemblyai.com/v2/transcript`
  - JSON body: `{ "audio_url": "{{$json['upload_url']}}" }`
  - Header: `content-type: application/json`
  - Auth: AssemblyAI header auth
- **Connections:** Output → **Wait for transcription processing**
- **Edge cases / failures:**
  - If `upload_url` missing, job submission fails.
  - Optional parameters (language detection, speaker labels) not set—transcript quality may vary by content.

#### Node: Wait for transcription processing
- **Type / role:** Wait node. Pauses execution for polling.
- **Key configuration:** `amount = 20` (seconds by default in n8n Wait node)
- **Connections:** Output → **Check transcription status**
- **Edge cases / failures:**
  - For longer videos or slow processing, 20 seconds may be insufficient, causing many polling loops (and more API calls).

#### Node: Check transcription status
- **Type / role:** HTTP Request. Fetches status and transcript text.
- **Key configuration:**
  - **GET** `https://api.assemblyai.com/v2/transcript/{{$node["Submit audio for transcription"].json["id"]}}`
  - Response format JSON
  - Auth: AssemblyAI header auth
- **Connections:** Output → **Extract transcript text**
- **Edge cases / failures:**
  - If the transcription failed, status may be `error` with an error message; workflow currently treats anything not `completed` as “keep waiting”.

#### Node: Extract transcript text
- **Type / role:** Code node. Extracts transcript text and carries `mode`.
- **Key logic:**
  - `mode` read from: `$items('Normalize Slack payload', 0, 0)?.json?.mode || 'summary'`
  - Returns: `{ transcript: $json.text ?? '', mode }`
- **Connections:** Output → **Is transcription complete?**
- **Edge cases / failures:**
  - If AssemblyAI response doesn’t include `.text` yet, transcript becomes empty string.
  - If `Normalize Slack payload` produced no items (unusual), mode defaults to summary.

#### Node: Is transcription complete?
- **Type / role:** IF node. Polling loop controller.
- **Condition:** `{{$node["Check transcription status"].json["status"].toLowerCase()}} == "completed"`
- **Connections:**
  - **True** → Generate AI summary
  - **False** → Wait for transcription processing (loop)
- **Edge cases / failures:**
  - If status is `error`, the workflow loops forever (no backoff / max retries / failure branch).
  - If `status` is missing, `.toLowerCase()` can throw.

---

### Block 4 — Summarization + Slack Delivery
**Overview:**  
Summarizes the transcript with OpenAI using strict formatting rules for Slack, then posts the result in-channel via Slack’s `response_url`.

**Nodes involved:**  
- Generate AI summary  
- Post result to Slack

#### Node: Generate AI summary
- **Type / role:** OpenAI (LangChain) node (`@n8n/n8n-nodes-langchain.openAi`). Produces a structured Slack-friendly summary.
- **Key configuration:**
  - **Model:** `gpt-4-turbo`
  - **Messages:**
    - System prompt: strong constraints (neutral, factual, no speculation, Slack Markdown rules, section structure and length caps).
    - User content: transcript text from `Extract transcript text`.
- **Connections:** Output → **Post result to Slack**
- **Version requirements:** Node type version `1.8` (ensure n8n LangChain/OpenAI integration is installed and compatible).
- **Edge cases / failures:**
  - Transcript too large for model context → truncation or failure; no chunking is implemented.
  - API key/permissions issues, rate limits, timeouts.

#### Node: Post result to Slack
- **Type / role:** HTTP Request. Posts final message via Slack `response_url`.
- **Key configuration:**
  - **URL:** `{{$node["Receive Slack command"].json["body"]["response_url"]}}`
  - Body parameters include:
    - `response_type = in_channel`
    - `text` includes:
      - Header `*SIFT Summary*`
      - Original video link: `<url|:film_projector: Original Video>`
      - Summary content pulled from OpenAI output:
        - `{{$node["Generate AI summary"].json["message"]["content"] ?? (...)}}`
  - Adds header: `" Content-Type": "application/json"` (note the leading space in the header name)
- **Connections:** Terminal node.
- **Edge cases / failures:**
  - Header name has a leading space (`" Content-Type"`). Some servers ignore it, others won’t recognize it; safest fix is `Content-Type`.
  - Slack `response_url` expects a JSON payload; ensure the HTTP Request node is actually sending JSON (n8n “Body Parameters” often sends form-encoded unless “JSON/RAW parameters” is selected in some versions).
  - Slack message length limits may truncate large summaries.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Slack command | Webhook | Slack slash-command entry point | — | Normalize Slack payload | Slack command input and payload normalization. |
| Normalize Slack payload | Set | Normalize Slack fields + decide mode | Receive Slack command | Extract YouTube video ID | Slack command input and payload normalization. |
| Extract YouTube video ID | Code | Parse input, extract URL + YouTube videoId | Normalize Slack payload | Check Video Duration | Extracts the YouTube video ID and validates video duration before processing. |
| Check Video Duration | HTTP Request | Fetch YouTube duration via RapidAPI | Extract YouTube video ID | Validate Duration | Extracts the YouTube video ID and validates video duration before processing. |
| Validate Duration | Code | Parse ISO-8601 duration; enforce 10-min limit | Check Video Duration | Is video longer than limit? | Extracts the YouTube video ID and validates video duration before processing. |
| Is video longer than limit? | IF | Route: reject vs continue | Validate Duration | Send duration error to Slack; Convert YouTube video to MP3 | Extracts the YouTube video ID and validates video duration before processing. |
| Send duration error to Slack | HTTP Request | Post duration rejection to Slack | Is video longer than limit? (true) | — | Extracts the YouTube video ID and validates video duration before processing. |
| Convert YouTube video to MP3 | HTTP Request | Download MP3 via RapidAPI (binary) | Is video longer than limit? (false) | Start transcription job | Converts the video to audio and submits it for transcription. |
| Start transcription job | HTTP Request | Upload audio to AssemblyAI | Convert YouTube video to MP3 | Submit audio for transcription | Converts the video to audio and submits it for transcription. |
| Submit audio for transcription | HTTP Request | Create transcription job | Start transcription job | Wait for transcription processing | Converts the video to audio and submits it for transcription. |
| Wait for transcription processing | Wait | Delay for polling loop | Submit audio for transcription; Is transcription complete? (false) | Check transcription status | Waits for transcription completion and retrieves the final transcript. |
| Check transcription status | HTTP Request | Poll AssemblyAI transcript status | Wait for transcription processing | Extract transcript text | Waits for transcription completion and retrieves the final transcript. |
| Extract transcript text | Code | Extract transcript + carry mode | Check transcription status | Is transcription complete? | Waits for transcription completion and retrieves the final transcript. |
| Is transcription complete? | IF | Route: summarize vs wait again | Extract transcript text | Generate AI summary; Wait for transcription processing | Waits for transcription completion and retrieves the final transcript. |
| Generate AI summary | OpenAI (LangChain) | Summarize transcript with strict Slack format | Is transcription complete? (true) | Post result to Slack | Generates an AI summary and sends the result back to Slack. |
| Post result to Slack | HTTP Request | Send summary to Slack via response_url | Generate AI summary | — | Generates an AI summary and sends the result back to Slack. |
| Sticky Note16 | Sticky Note | Documentation block | — | — | ## How it works … (contains overall explanation + setup steps) |
| Sticky Note17 | Sticky Note | Comment | — | — | Slack command input and payload normalization. |
| Sticky Note18 | Sticky Note | Comment | — | — | Extracts the YouTube video ID and validates video duration before processing. |
| Sticky Note19 | Sticky Note | Comment | — | — | Converts the video to audio and submits it for transcription. |
| Sticky Note | Sticky Note | Comment | — | — | Waits for transcription completion and retrieves the final transcript. |
| Sticky Note1 | Sticky Note | Comment | — | — | Generates an AI summary and sends the result back to Slack. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Webhook node**: “Receive Slack command”
- Type: **Webhook**
- Method: **POST**
- Path: `sift`
- Response: set to immediately return text like:  
  `Filtering the fluff… give me about 2–4 minutes. I’ll post the summary here. Sifting... 👨‍🍳`
- Slack setup: create a Slack slash command (e.g., `/sift`) pointing to this webhook URL.

2) **Create Set node**: “Normalize Slack payload”
- Keep other fields enabled.
- Add fields:
  - `videoUrl` (String): `{{$json["body"]["text"] || ""}}`
  - `mode` (String): `{{$json.body.command === '/siftf' ? 'transcript' : 'summary'}}`
- Connect: Webhook → Set

3) **Create Code node**: “Extract YouTube video ID”
- Paste logic to:
  - Remove the slash command text
  - Extract first URL token
  - Extract 11-char YouTube ID via regex
- Output: `text`, `videoUrl`, `videoId`
- Connect: Normalize → Extract ID

4) **Create HTTP Request**: “Check Video Duration”
- URL: `https://youtube-v31.p.rapidapi.com/videos?part=contentDetails&id={{ $('Extract YouTube video ID').item.json.videoId }}`
- Auth: **HTTP Header Auth** credential named “RapidAPI”
  - Must include RapidAPI key header (commonly `x-rapidapi-key`) in the credential.
- Add header: `x-rapidapi-host: youtube-v31.p.rapidapi.com`
- Connect: Extract ID → Check Video Duration

5) **Create Code node**: “Validate Duration”
- Parse `items[0].contentDetails.duration` (ISO-8601) into minutes.
- Enforce `max = 10` minutes:
  - If too long, return `{ error: true, errorMessage, videoId, videoUrl }`
  - Else return `{ error: false, videoId, videoUrl, durationMinutes }`
- Connect: Check Duration → Validate Duration

6) **Create IF node**: “Is video longer than limit?”
- Condition: Boolean equals
  - Left: `{{$json.error}}`
  - Right: `true`
- True path = reject; false path = continue.
- Connect: Validate Duration → IF

7) **Create HTTP Request**: “Send duration error to Slack” (true branch)
- URL: `{{$('Receive Slack command').item.json.body.response_url}}`
- Method: POST
- Send Body: yes
- Body field `text`: `❌ {{$json.errorMessage}}`
- Connect: IF (true) → Send error

8) **Create HTTP Request**: “Convert YouTube video to MP3” (false branch)
- URL: `https://youtube-mp3-audio-video-downloader.p.rapidapi.com/download-mp3/{{$json.videoId}}?quality=low`
- Auth: same **RapidAPI** header credential
- Headers:
  - `x-rapidapi-host: youtube-mp3-audio-video-downloader.p.rapidapi.com`
  - `accept: application/octet-stream`
- Response: **File** (binary), property name: `mp3`
- (Optional but matches workflow) enable “Never Error” and “Full Response”
- Connect: IF (false) → Convert to MP3

9) **Create HTTP Request**: “Start transcription job” (AssemblyAI upload)
- URL: `https://api.assemblyai.com/v2/upload`
- Method: POST
- Auth: **HTTP Header Auth** credential “AssemblyAI” (header usually `authorization: <key>`)
- Body content type: **Binary Data**
  - Binary field: `mp3`
  - Content-Type: `application/octet-stream`
- Connect: Convert MP3 → Upload

10) **Create HTTP Request**: “Submit audio for transcription”
- URL: `https://api.assemblyai.com/v2/transcript`
- Method: POST
- Body: JSON `{ "audio_url": "{{$json.upload_url}}" }`
- Header: `content-type: application/json`
- Auth: AssemblyAI header credential
- Connect: Upload → Submit transcription

11) **Create Wait node**: “Wait for transcription processing”
- Set wait time to **20 seconds**
- Connect: Submit transcription → Wait

12) **Create HTTP Request**: “Check transcription status”
- URL: `https://api.assemblyai.com/v2/transcript/{{$node["Submit audio for transcription"].json["id"]}}`
- Method: GET
- Auth: AssemblyAI header credential
- Connect: Wait → Check status

13) **Create Code node**: “Extract transcript text”
- Output:
  - `transcript: $json.text ?? ''`
  - `mode: $items('Normalize Slack payload', 0, 0)?.json?.mode || 'summary'`
- Connect: Check status → Extract transcript

14) **Create IF node**: “Is transcription complete?”
- Condition (string equals):
  - Left: `{{$node["Check transcription status"].json["status"].toLowerCase()}}`
  - Right: `completed`
- True → summarize; False → loop back to Wait.
- Connect: Extract transcript → IF
- Connect: IF (false) → Wait (creates polling loop)

15) **Create OpenAI (LangChain) node**: “Generate AI summary”
- Node type: `@n8n/n8n-nodes-langchain.openAi`
- Credential: OpenAI API key credential
- Model: `gpt-4-turbo`
- Messages:
  - System: the provided strict summarization + Slack formatting rules
  - User: `{{$node["Extract transcript text"].json["transcript"]}}`
- Connect: IF (true) → Generate AI summary

16) **Create HTTP Request**: “Post result to Slack”
- URL: `{{$node["Receive Slack command"].json["body"]["response_url"]}}`
- Method: POST
- Body:
  - `response_type = in_channel`
  - `text` containing:
    - `*SIFT Summary*`
    - `<{{$node["Extract YouTube video ID"].json["videoUrl"]}}|:film_projector: Original Video>`
    - The model output content from the OpenAI node
- Header: use `Content-Type: application/json` (without leading spaces)
- Connect: Generate AI summary → Post to Slack

**Credentials required**
- **RapidAPI (HTTP Header Auth):** include `x-rapidapi-key` (and optionally other required headers depending on RapidAPI configuration).
- **AssemblyAI (HTTP Header Auth):** `authorization: <ASSEMBLYAI_API_KEY>`
- **OpenAI (OpenAI credential):** API key with access to the chosen model
- **Slack:** no Slack credential is required because replies are sent to `response_url`, but Slack slash command configuration is required.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How it works” + “Setup steps” summary explaining Slack → duration check → MP3 → AssemblyAI → OpenAI → Slack | Included in the workflow sticky note (“How it works” section). |
| Setup steps: create Slack slash command; add credentials for YouTube/RapidAPI, AssemblyAI, OpenAI; adjust max duration; activate and test | Included in the workflow sticky note (“Setup steps” section). |
| Potential operational gap: transcription failures (`status = error`) are not handled; workflow will keep waiting indefinitely | General reliability consideration based on current polling logic. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.