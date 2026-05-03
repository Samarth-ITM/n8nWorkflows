Send YouTube transcripts to Slack using AssemblyAI

https://n8nworkflows.xyz/workflows/send-youtube-transcripts-to-slack-using-assemblyai-12935


# Send YouTube transcripts to Slack using AssemblyAI

## 1. Workflow Overview

**Title:** Send YouTube transcripts to Slack using AssemblyAI  
**Workflow name (in JSON):** `rippr – YouTube Transcript`

**Purpose:**  
This workflow lets a Slack user submit a YouTube link via a Slack slash command. The workflow extracts the YouTube video ID, checks the video duration (enforcing a max length), converts the video to MP3 via RapidAPI, uploads the audio to AssemblyAI, polls until transcription is complete, then cleans the transcript using OpenAI and posts the final formatted text back into Slack via Slack’s `response_url`.

**Typical use cases:**
- Quickly “rip” a transcript from a YouTube link without leaving Slack.
- Enforce short video limits to control cost/time.
- Produce a readable transcript (cleaned formatting) for sharing in a channel.

### 1.1 Input Reception (Slack)
Receives Slack slash-command payload and normalizes key fields.

### 1.2 YouTube URL Parsing + Duration Guardrail
Extracts the YouTube video ID from the Slack text, calls YouTube Data API (via RapidAPI) to get duration, and blocks videos longer than a configured threshold (10 minutes).

### 1.3 Audio Conversion + AssemblyAI Job Creation
Downloads MP3 audio (RapidAPI), uploads to AssemblyAI, and creates a transcription job.

### 1.4 Polling for Completion + Transcript Retrieval
Waits and repeatedly checks AssemblyAI job status until it is completed, then extracts the transcript text.

### 1.5 Cleanup (OpenAI) + Slack Delivery
Uses OpenAI to reformat transcript for readability (not a summary despite the node name), then posts the result back to Slack.

---

## 2. Block-by-Block Analysis

### Block 1 — Slack Command Intake & Normalization
**Overview:**  
Accepts a Slack slash command request, immediately responds with an “in progress” message, and prepares the payload for downstream nodes.

**Nodes involved:**
- `Receive Slack command`
- `Normalize Slack payload`

#### Node: Receive Slack command
- **Type / role:** Webhook trigger; entrypoint for Slack slash command.
- **Configuration choices:**
  - **HTTP Method:** POST
  - **Path:** `rippr` (Slack slash command should POST to this endpoint)
  - **Immediate response message:** `🗡️ Ripping the full transcript… wait a few minutes, I’ll post it here soon (• v •)`
  - **Response mode:** configured via webhook `options.responseData` (static text)
- **Inputs/Outputs:**
  - **Input:** Slack payload in `body` (typical Slack slash command fields like `text`, `response_url`, etc.)
  - **Output:** Passes request data to `Normalize Slack payload`
- **Failure/edge cases:**
  - Slack may retry if webhook does not respond quickly; ensure n8n is reachable and the webhook responds immediately (this node does).
  - If Slack signing/verification is required, it is **not implemented** here (no signature validation).

#### Node: Normalize Slack payload
- **Type / role:** Set node; maps/creates a `videoUrl` field from Slack’s request body.
- **Configuration choices (interpreted):**
  - Adds/overwrites field `videoUrl` using `{{$json["body"]["text"] || ""}}`
  - **Include other fields:** enabled (passes everything through)
- **Key expressions/variables:**
  - `{{$json["body"]["text"] || ""}}`
- **Inputs/Outputs:**
  - Input from `Receive Slack command`
  - Output to `Extract YouTube video ID`
- **Failure/edge cases:**
  - If Slack payload structure differs (no `body.text`), `videoUrl` becomes empty. Downstream parsing will fail to find a video ID.

**Sticky note (applies to this block’s nodes):**  
“Receives the Slack command and normalizes the incoming payload.”

---

### Block 2 — YouTube ID Extraction & Duration Validation
**Overview:**  
Extracts an 11-character YouTube video ID from multiple possible URL formats, fetches the YouTube video duration via RapidAPI YouTube Data API endpoint, parses ISO-8601 duration, and enforces a 10-minute maximum.

**Nodes involved:**
- `Extract YouTube video ID`
- `Check Video Duration`
- `Parse and validate video duration`
- `Is video longer than limit?`
- `Send duration error to Slack`

#### Node: Extract YouTube video ID
- **Type / role:** Code node (JavaScript); robust URL parsing + ID extraction.
- **Configuration choices (interpreted):**
  - Removes a command prefix `"/rippit "` from the Slack text (note: your webhook path is `rippr`, but the code removes `/rippit`; mismatch is harmless unless users actually type a different command).
  - Extracts first URL from the text (or treats the entire text as URL).
  - Supports forms:
    - `youtube.com/watch?v=VIDEOID`
    - `youtu.be/VIDEOID`
    - `youtube.com/shorts/VIDEOID`
    - `youtube.com/embed/VIDEOID`
- **Key expressions/variables:**
  - Reads Slack text from: `inputJson?.body?.text ?? inputJson?.text ?? ''`
  - Regex: `/(?:youtube\.com\/watch\?v=|youtu\.be\/|shorts\/|embed\/)([A-Za-z0-9_-]{11})/i`
  - Outputs: `{ text, videoUrl, videoId }`
- **Inputs/Outputs:**
  - Input from `Normalize Slack payload`
  - Output to `Check Video Duration`
- **Failure/edge cases:**
  - If `videoId` is `null`, downstream nodes will call APIs with an invalid ID. There is **no explicit guard** for “invalid or missing video ID”.
  - Slack users may paste playlist URLs, channels, or other formats not matching the regex.

#### Node: Check Video Duration
- **Type / role:** HTTP Request; calls YouTube Data API via RapidAPI to get `contentDetails.duration`.
- **Configuration choices (interpreted):**
  - **URL:** `https://youtube-v31.p.rapidapi.com/videos?part=contentDetails&id={{videoId}}`
  - **Auth:** HTTP Header Auth credential named `RapidAPI`
  - **Header:** `x-rapidapi-host: youtube-v31.p.rapidapi.com`
  - (Implicitly requires RapidAPI key header in the credential)
- **Key expressions/variables:**
  - Uses `$('Extract YouTube video ID').item.json.videoId`
- **Inputs/Outputs:**
  - Input: from `Extract YouTube video ID`
  - Output: to `Parse and validate video duration`
- **Failure/edge cases:**
  - RapidAPI quota exceeded, invalid key, or wrong host header.
  - YouTube API may return empty `items` for invalid/private videos; downstream code assumes `items[0]` exists.

#### Node: Parse and validate video duration
- **Type / role:** Code node; parses ISO 8601 duration and applies max duration rule.
- **Configuration choices (interpreted):**
  - Reads duration from: `$input.item.json.items[0].contentDetails.duration`
  - Parses `PT#H#M#S`
  - Computes total minutes
  - Hard-coded limit: **10 minutes**
  - Does not throw errors; returns `{ error: true, errorMessage: ... }` for long videos
- **Key expressions/variables:**
  - Gets original fields from: `$('Extract YouTube video ID').first()`
  - Output includes: `error`, `errorMessage` (if too long), `videoId`, `videoUrl`, `durationMinutes`
- **Inputs/Outputs:**
  - Input: from `Check Video Duration`
  - Output: to `Is video longer than limit?`
- **Failure/edge cases:**
  - If `items[0]` missing or duration not matching regex, `match` may be `null` and `match[1]` will throw.
  - Limit is embedded in code; changing it requires editing JS.

#### Node: Is video longer than limit?
- **Type / role:** IF node; branches based on `error === true`.
- **Configuration choices:**
  - Condition: `{{$json.error}}` equals `true`
- **Inputs/Outputs:**
  - Input: from `Parse and validate video duration`
  - **True path:** `Send duration error to Slack`
  - **False path:** `Convert YouTube video to MP3`
- **Failure/edge cases:**
  - If prior node fails and doesn’t output `error`, condition evaluation may behave unexpectedly.

#### Node: Send duration error to Slack
- **Type / role:** HTTP Request; posts an error message back to Slack via `response_url`.
- **Configuration choices:**
  - **URL:** `{{$('Receive Slack command').item.json.body.response_url}}`
  - **Method:** POST
  - **Body:** `text = ❌ {{$json.errorMessage}}`
- **Inputs/Outputs:**
  - Input: from IF true branch
  - Output: none further (end)
- **Failure/edge cases:**
  - If `response_url` missing (non-Slack invocation), the request fails.
  - Slack `response_url` expires after some time; errors posted too late may be rejected.

**Sticky note (applies to this block’s nodes):**  
“Extracts the YouTube video ID and validates video duration before processing.”

---

### Block 3 — Convert YouTube to MP3 & Start AssemblyAI Transcription
**Overview:**  
Downloads the YouTube audio as an MP3 via RapidAPI, uploads the binary to AssemblyAI to obtain an `upload_url`, then creates a transcription job.

**Nodes involved:**
- `Convert YouTube video to MP3`
- ` Create transcription job`
- `Submit audio for transcription`

#### Node: Convert YouTube video to MP3
- **Type / role:** HTTP Request; fetches MP3 file from RapidAPI downloader endpoint.
- **Configuration choices (interpreted):**
  - **URL:** `https://youtube-mp3-audio-video-downloader.p.rapidapi.com/download-mp3/{videoId}?quality=low`
  - **Auth:** RapidAPI via HTTP Header Auth credential `RapidAPI`
  - Response:
    - `responseFormat: file`
    - output binary property name: `mp3`
    - `fullResponse: true`
    - `neverError: true` (important: even non-2xx responses won’t hard-fail the node)
  - Headers include:
    - `x-rapidapi-host: youtube-mp3-audio-video-downloader.p.rapidapi.com`
    - `accept: application/octet-stream`
    - `content-type : application/json` (note trailing space in header name; may be ignored by some servers)
- **Inputs/Outputs:**
  - Input: from duration check false branch
  - Output: binary `mp3` to ` Create transcription job`
- **Failure/edge cases:**
  - Because `neverError` is enabled, failures may appear as “successful” node executions but produce invalid/empty binary, causing AssemblyAI upload to fail.
  - RapidAPI services can return JSON error bodies instead of binary.

#### Node:  Create transcription job
- **Type / role:** HTTP Request; uploads MP3 binary to AssemblyAI `/upload`.
- **Configuration choices (interpreted):**
  - **URL:** `https://api.assemblyai.com/v2/upload`
  - **Method:** POST
  - **Body type:** binary data
  - **Binary property:** `mp3` (from previous node)
  - **Header:** `content-type: application/octet-stream`
  - **Auth:** HTTP Header Auth credential `AssemblyAI` (expects `authorization: <api-key>` typically)
  - Response format: JSON; expects `upload_url`
- **Inputs/Outputs:**
  - Input: binary from `Convert YouTube video to MP3`
  - Output: JSON including `upload_url` to `Submit audio for transcription`
- **Failure/edge cases:**
  - Missing/invalid AssemblyAI key → 401.
  - Upload size/timeouts for longer videos (even though duration is capped).
  - If MP3 binary is not present at `mp3`, upload will fail.

#### Node: Submit audio for transcription
- **Type / role:** HTTP Request; creates transcription job in AssemblyAI.
- **Configuration choices (interpreted):**
  - **URL:** `https://api.assemblyai.com/v2/transcript`
  - **Method:** POST
  - **JSON body:** `{ "audio_url": "{{$json.upload_url}}" }`
  - **Header:** `content-type: application/json`
  - **Auth:** AssemblyAI header auth
- **Inputs/Outputs:**
  - Input: JSON from upload step
  - Output: JSON containing transcription `id` to `Wait for transcription processing`
- **Failure/edge cases:**
  - `upload_url` missing → job creation fails.
  - AssemblyAI may return `error` fields if input is invalid.

**Sticky note (applies to this block’s nodes):**  
“Converts the video to audio and starts the transcription job.”

---

### Block 4 — Wait/Poll for AssemblyAI Completion & Extract Transcript
**Overview:**  
Implements polling: wait 20 seconds, check job status, loop until completed, then extract transcript text.

**Nodes involved:**
- `Wait for transcription processing`
- ` Check transcription status`
- `Extract transcript text`
- `Is transcription complete?`

#### Node: Wait for transcription processing
- **Type / role:** Wait node; introduces delay between status checks.
- **Configuration choices:**
  - Amount: `20` (seconds)
- **Inputs/Outputs:**
  - Input: from `Submit audio for transcription` (first loop) OR from `Is transcription complete?` false branch (subsequent loops)
  - Output: to ` Check transcription status`
- **Failure/edge cases:**
  - Many concurrent executions can accumulate waiting jobs; consider n8n queue/worker capacity.
  - No maximum retry count; could loop indefinitely if job stays “processing” or “queued” forever.

#### Node:  Check transcription status
- **Type / role:** HTTP Request; polls AssemblyAI transcript status endpoint.
- **Configuration choices (interpreted):**
  - **URL:** `https://api.assemblyai.com/v2/transcript/{{$node["Submit audio for transcription"].json["id"]}}`
  - Response format: JSON (expects `status`, and when complete, `text`)
  - Auth: AssemblyAI header auth
- **Inputs/Outputs:**
  - Input: from `Wait for transcription processing`
  - Output: to `Extract transcript text`
- **Failure/edge cases:**
  - If `Submit audio for transcription` did not run or has no `id`, URL becomes invalid.
  - AssemblyAI errors: `status` could be `error`; the workflow does not explicitly handle this.

#### Node: Extract transcript text
- **Type / role:** Code node; maps transcript text into a clean field.
- **Configuration choices (interpreted):**
  - Outputs:
    - `transcript: $json.text ?? ''` (from AssemblyAI status response)
    - `mode`: reads `mode` from `Normalize Slack payload` if present, else `'summary'`
- **Key expressions/variables:**
  - `($items('Normalize Slack payload', 0, 0)?.json?.mode) || 'summary'`
  - Note: `Normalize Slack payload` does not actually set `mode`, so this defaults to `'summary'` always unless added elsewhere.
- **Inputs/Outputs:**
  - Input: from ` Check transcription status`
  - Output: to `Is transcription complete?`
- **Failure/edge cases:**
  - If transcription is not complete, `$json.text` is often absent; transcript becomes empty (but completion is checked in next node based on `status` from a different node reference).

#### Node: Is transcription complete?
- **Type / role:** IF node; decides whether to proceed or keep waiting.
- **Configuration choices:**
  - Condition checks: `{{$node[" Check transcription status"].json["status"].toLowerCase()}}` equals `completed`
  - True branch → `Generate AI summary`
  - False branch → loops back to `Wait for transcription processing`
- **Inputs/Outputs:**
  - Input: from `Extract transcript text`
  - Branching described above
- **Failure/edge cases:**
  - If `status` is missing, `.toLowerCase()` throws an expression error.
  - No handling for `status: "error"`; it will loop forever.

**Sticky note (applies to this block’s nodes):**  
“Waits for transcription completion and retrieves the final transcript.”

---

### Block 5 — OpenAI Cleanup & Slack Posting
**Overview:**  
Uses OpenAI to reformat transcript into readable paragraphs (despite the node name suggesting summarization), then posts a branded message back to Slack using the original `response_url`.

**Nodes involved:**
- `Generate AI summary`
- `Post result to Slack`

#### Node: Generate AI summary
- **Type / role:** LangChain OpenAI node (`@n8n/n8n-nodes-langchain.openAi`); text transformation.
- **Configuration choices (interpreted):**
  - **Model:** `gpt-4-turbo`
  - **System message:** instructs model to *clean/reformat transcript*, explicitly says **do not summarize**
  - **User content:** transcript from `Extract transcript text`
- **Key expressions/variables:**
  - `{{$node["Extract transcript text"].json["transcript"]}}`
- **Inputs/Outputs:**
  - Input: from `Is transcription complete?` true branch
  - Output: to `Post result to Slack`
- **Failure/edge cases:**
  - OpenAI auth errors, model availability changes, rate limits.
  - Token limits for long transcripts; although video is capped at 10 minutes, transcripts can still be sizeable.
  - Node name is misleading: it performs cleaning, not summarization.

#### Node: Post result to Slack
- **Type / role:** HTTP Request; posts final message to Slack via `response_url`.
- **Configuration choices (interpreted):**
  - **URL:** `{{$node["Receive Slack command"].json["body"]["response_url"]}}`
  - **Method:** POST
  - **Body parameters:**
    - `response_type=in_channel` (broadcasts to channel)
    - `text=` branded header + original video link + OpenAI cleaned transcript
  - Pulls content primarily from:
    - `Generate AI summary` → `message.content`
    - Fallback expression for alternate output shapes
- **Key expressions/variables:**
  - Video link: `<{{$node["Extract YouTube video ID"].json["videoUrl"]}}|:film_projector: Original Video>`
  - Transcript content expression:  
    `{{$node["Generate AI summary"].json["message"]["content"] ?? ($json["message"] ? $json["message"][0]["content"] : "")}}`
- **Inputs/Outputs:**
  - Input: from `Generate AI summary`
  - Output: end
- **Failure/edge cases:**
  - Slack `response_url` may expire; long transcription times can cause posting failure.
  - Slack message length limits; large transcripts may be truncated/rejected. Consider splitting into multiple posts.

**Sticky note (applies to this block’s nodes):**  
“Formats the result and sends it back to Slack.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Slack command | Webhook | Entry point: receives Slack slash command and responds immediately | — | Normalize Slack payload | Receives the Slack command and normalizes the incoming payload. |
| Normalize Slack payload | Set | Maps Slack payload into normalized fields (`videoUrl`) | Receive Slack command | Extract YouTube video ID | Receives the Slack command and normalizes the incoming payload. |
| Extract YouTube video ID | Code | Extracts URL + YouTube 11-char video ID from Slack text | Normalize Slack payload | Check Video Duration | Extracts the YouTube video ID and validates video duration before processing. |
| Check Video Duration | HTTP Request | Calls YouTube Data API via RapidAPI to retrieve ISO duration | Extract YouTube video ID | Parse and validate video duration | Extracts the YouTube video ID and validates video duration before processing. |
| Parse and validate video duration | Code | Parses ISO-8601 duration and enforces 10-minute limit | Check Video Duration | Is video longer than limit? | Extracts the YouTube video ID and validates video duration before processing. |
| Is video longer than limit? | If | Branches: send Slack error vs continue processing | Parse and validate video duration | Send duration error to Slack; Convert YouTube video to MP3 | Extracts the YouTube video ID and validates video duration before processing. |
| Send duration error to Slack | HTTP Request | Posts error back to Slack via `response_url` | Is video longer than limit? (true) | — | Extracts the YouTube video ID and validates video duration before processing. |
| Convert YouTube video to MP3 | HTTP Request | Downloads MP3 via RapidAPI and outputs binary `mp3` | Is video longer than limit? (false) |  Create transcription job | Converts the video to audio and starts the transcription job. |
|  Create transcription job | HTTP Request | Uploads MP3 binary to AssemblyAI `/upload` to get `upload_url` | Convert YouTube video to MP3 | Submit audio for transcription | Converts the video to audio and starts the transcription job. |
| Submit audio for transcription | HTTP Request | Creates AssemblyAI transcript job from `upload_url` |  Create transcription job | Wait for transcription processing | Converts the video to audio and starts the transcription job. |
| Wait for transcription processing | Wait | Delays before polling again (20s) | Submit audio for transcription; Is transcription complete? (false) |  Check transcription status | Waits for transcription completion and retrieves the final transcript. |
|  Check transcription status | HTTP Request | Polls AssemblyAI transcript job status and retrieves text when ready | Wait for transcription processing | Extract transcript text | Waits for transcription completion and retrieves the final transcript. |
| Extract transcript text | Code | Outputs `{transcript, mode}` from AssemblyAI response |  Check transcription status | Is transcription complete? | Waits for transcription completion and retrieves the final transcript. |
| Is transcription complete? | If | If completed → proceed; else loop wait/poll | Extract transcript text | Generate AI summary; Wait for transcription processing | Waits for transcription completion and retrieves the final transcript. |
| Generate AI summary | OpenAI (LangChain) | Cleans/reformats transcript text (no summarization) | Is transcription complete? (true) | Post result to Slack | Formats the result and sends it back to Slack. |
| Post result to Slack | HTTP Request | Posts final transcript back to Slack channel via `response_url` | Generate AI summary | — | Formats the result and sends it back to Slack. |
| Sticky Note16 | Sticky Note | Documentation (“How it works” + setup steps) | — | — |  |
| Sticky Note | Sticky Note | Comment: Slack reception block | — | — |  |
| Sticky Note1 | Sticky Note | Comment: YouTube extraction + duration validation | — | — |  |
| Sticky Note2 | Sticky Note | Comment: conversion + transcription job start | — | — |  |
| Sticky Note3 | Sticky Note | Comment: waiting + completion | — | — |  |
| Sticky Note4 | Sticky Note | Comment: formatting + Slack send | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Webhook trigger**
   1. Add node **Webhook** named `Receive Slack command`.
   2. Set **HTTP Method** = `POST`.
   3. Set **Path** = `rippr`.
   4. In **Response**, set a text response like:  
      `🗡️ Ripping the full transcript… wait a few minutes, I’ll post it here soon (• v •)`

2) **Normalize Slack payload**
   1. Add node **Set** named `Normalize Slack payload`.
   2. Enable **Include Other Fields**.
   3. Add field `videoUrl` (String) with expression:  
      `{{$json.body.text || ""}}`
   4. Connect: `Receive Slack command` → `Normalize Slack payload`.

3) **Extract YouTube video ID**
   1. Add node **Code** named `Extract YouTube video ID`.
   2. Paste logic that:
      - reads Slack text from `body.text`
      - optionally strips `/rippit`
      - extracts first URL
      - extracts video ID using regex for watch/youtu.be/shorts/embed
      - outputs `{ videoUrl, videoId, text }`
   3. Connect: `Normalize Slack payload` → `Extract YouTube video ID`.

4) **RapidAPI: check video duration (YouTube Data API)**
   1. Add node **HTTP Request** named `Check Video Duration`.
   2. Set URL to:  
      `https://youtube-v31.p.rapidapi.com/videos?part=contentDetails&id={{$('Extract YouTube video ID').item.json.videoId}}`
   3. Set **Authentication** to **Generic Credential Type** → **HTTP Header Auth**.
   4. Create credential **RapidAPI** that sends your RapidAPI key header (commonly `X-RapidAPI-Key`).
   5. Add header: `x-rapidapi-host = youtube-v31.p.rapidapi.com`
   6. Connect: `Extract YouTube video ID` → `Check Video Duration`.

5) **Parse duration + enforce max length**
   1. Add node **Code** named `Parse and validate video duration`.
   2. Implement:
      - read `items[0].contentDetails.duration`
      - parse ISO `PT#H#M#S`
      - compute total minutes
      - if `> 10`, output `{ error: true, errorMessage, videoId, videoUrl }`
      - else output `{ error: false, videoId, videoUrl, durationMinutes }`
   3. Connect: `Check Video Duration` → `Parse and validate video duration`.

6) **IF: duration too long**
   1. Add node **IF** named `Is video longer than limit?`
   2. Condition: `{{$json.error}}` **equals** `true`.
   3. Connect: `Parse and validate video duration` → `Is video longer than limit?`

7) **Slack error path**
   1. Add node **HTTP Request** named `Send duration error to Slack`.
   2. Method: POST
   3. URL: `{{$('Receive Slack command').item.json.body.response_url}}`
   4. Body: send `text = ❌ {{$json.errorMessage}}`
   5. Connect IF **true** → `Send duration error to Slack`.

8) **Convert YouTube to MP3 (RapidAPI)**
   1. Add node **HTTP Request** named `Convert YouTube video to MP3`.
   2. URL:  
      `https://youtube-mp3-audio-video-downloader.p.rapidapi.com/download-mp3/{{$json.videoId}}?quality=low`
   3. Authentication: use the same **RapidAPI** HTTP Header Auth credential.
   4. Add headers:
      - `x-rapidapi-host = youtube-mp3-audio-video-downloader.p.rapidapi.com`
      - `accept = application/octet-stream`
   5. Response settings:
      - Response Format = **File**
      - Binary Property = `mp3`
      - (Optional) “Never Error” if you want the same behavior, but note it can mask failures.
   6. Connect IF **false** → `Convert YouTube video to MP3`.

9) **AssemblyAI: upload binary**
   1. Add node **HTTP Request** named ` Create transcription job`.
   2. Method: POST
   3. URL: `https://api.assemblyai.com/v2/upload`
   4. Authentication: **HTTP Header Auth** credential named `AssemblyAI` (send AssemblyAI API key, usually `authorization` header).
   5. Set body type to **Binary Data**, binary field = `mp3`.
   6. Add header: `content-type = application/octet-stream`
   7. Connect: `Convert YouTube video to MP3` → ` Create transcription job`.

10) **AssemblyAI: create transcript job**
   1. Add node **HTTP Request** named `Submit audio for transcription`.
   2. Method: POST
   3. URL: `https://api.assemblyai.com/v2/transcript`
   4. Authentication: `AssemblyAI` credential.
   5. Send JSON body: `{ "audio_url": "{{$json.upload_url}}" }`
   6. Connect: ` Create transcription job` → `Submit audio for transcription`.

11) **Wait + Poll loop**
   1. Add node **Wait** named `Wait for transcription processing`, amount = `20` seconds.
   2. Connect: `Submit audio for transcription` → `Wait for transcription processing`.
   3. Add node **HTTP Request** named ` Check transcription status`.
      - Method: GET
      - URL: `https://api.assemblyai.com/v2/transcript/{{$node["Submit audio for transcription"].json["id"]}}`
      - Auth: `AssemblyAI`
   4. Connect: `Wait for transcription processing` → ` Check transcription status`.

12) **Extract transcript**
   1. Add node **Code** named `Extract transcript text`.
   2. Output: `transcript = $json.text ?? ''` and a `mode` (optional).
   3. Connect: ` Check transcription status` → `Extract transcript text`.

13) **IF: transcription complete**
   1. Add node **IF** named `Is transcription complete?`
   2. Condition: `{{$node[" Check transcription status"].json.status.toLowerCase()}}` equals `completed`
   3. Connect: `Extract transcript text` → `Is transcription complete?`
   4. Connect IF **false** → back to `Wait for transcription processing` (this forms the polling loop).

14) **OpenAI cleanup**
   1. Add node **OpenAI (LangChain)** named `Generate AI summary` (name optional).
   2. Credentials: configure **OpenAI API** credential.
   3. Model: `gpt-4-turbo` (or your preferred equivalent).
   4. Messages:
      - System: instructions to reformat transcript and **not summarize**
      - User: `{{$node["Extract transcript text"].json.transcript}}`
   5. Connect IF **true** → `Generate AI summary`.

15) **Post back to Slack**
   1. Add node **HTTP Request** named `Post result to Slack`.
   2. Method: POST
   3. URL: `{{$node["Receive Slack command"].json.body.response_url}}`
   4. Body:
      - `response_type = in_channel`
      - `text =` include branding + original video link + OpenAI output content
   5. Connect: `Generate AI summary` → `Post result to Slack`.

**Credentials required:**
- **RapidAPI (HTTP Header Auth):** must include your RapidAPI key header (commonly `X-RapidAPI-Key`) used by both RapidAPI endpoints.
- **AssemblyAI (HTTP Header Auth):** typically header `authorization: <ASSEMBLYAI_API_KEY>`.
- **OpenAI:** n8n OpenAI credential.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How it works” + setup checklist is embedded in a sticky note in the workflow canvas. | Internal workflow documentation (Sticky Note16) |
| The workflow enforces a **10-minute** duration limit in code, not in a parameter. | Edit `Parse and validate video duration` to change the limit. |
| Polling loop has **no max retries** and does not handle AssemblyAI `status=error`. | Consider adding an error branch + retry limit to avoid infinite runs. |
| The OpenAI node is named “Generate AI summary” but prompt instructs **cleanup only**, not summarization. | Rename node if you want clarity, or change the system prompt if you truly want summaries. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.