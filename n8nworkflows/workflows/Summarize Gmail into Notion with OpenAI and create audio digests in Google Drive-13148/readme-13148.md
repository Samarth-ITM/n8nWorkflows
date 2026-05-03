Summarize Gmail into Notion with OpenAI and create audio digests in Google Drive

https://n8nworkflows.xyz/workflows/summarize-gmail-into-notion-with-openai-and-create-audio-digests-in-google-drive-13148


# Summarize Gmail into Notion with OpenAI and create audio digests in Google Drive

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow runs weekly, pulls recent Gmail messages from specific senders, summarizes them with OpenAI into a weekly recap, writes the recap into Notion, generates an audio version (OpenAI TTS and optionally ElevenLabs), uploads the audio to Google Drive, shares it publicly, and appends the public audio link back into Notion.

**Target use cases:**
- Weekly “inbox-to-knowledge” digest for newsletters, ops updates, AI industry monitoring, etc.
- Automated text + audio recap archive in Notion.

### Logical blocks
1.1 **Scheduling & Gmail retrieval** → Trigger weekly, search Gmail, fetch messages, mark them as read.  
1.2 **Per-email summarization (LLM)** → Extract email body and produce short “raw” summaries.  
1.3 **Aggregation & final editorial recap** → Merge raw summaries, enforce length limits, generate final weekly recap.  
1.4 **Publish to Notion** → Append final recap text to a Notion block/page.  
1.5 **Audio generation & Drive publishing** → Convert recap to speech, upload MP3 to Drive, share, build direct download URL, append URL to Notion.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & Gmail retrieval
**Overview:** Runs on a weekly schedule, searches Gmail for messages from selected senders within the last 7 days, then both marks them as read and fetches full message content for summarization.

**Nodes involved:**
- Weekly trigger
- Get many messages
- Mark as read
- Get a message

#### Node: Weekly trigger
- **Type / role:** `Schedule Trigger` — workflow entrypoint.
- **Configuration (interpreted):** Runs every week on **day 6** at **12:32** (n8n schedule uses numeric weekdays; confirm locale mapping in your instance).
- **Outputs:** Emits a single trigger item to start the flow.
- **Edge cases / failures:** Timezone differences (instance timezone vs expectation) can shift execution time.

#### Node: Get many messages
- **Type / role:** `Gmail (getAll)` — searches Gmail and returns many message metadata items.
- **Key configuration:**
  - Filter query: `newer_than:7d (from:xxx@xxx.ai OR from:yyy@yyy.com)`
  - Operation: **Get All**
- **Credentials:** Gmail OAuth2 (“Gmail account”).
- **Inputs:** From Weekly trigger.
- **Outputs / connections:**
  - To **Mark as read** (same items)
  - To **Get a message** (same items)
- **Edge cases / failures:**
  - Gmail auth expiration / revoked consent.
  - Query returns 0 items → downstream nodes may receive no input (nothing is summarized/published).
  - Large result sets: rate limits and long runtimes; consider limiting/max results if needed.

#### Node: Mark as read
- **Type / role:** `Gmail (removeLabels)` — removes the UNREAD label for each message.
- **Key configuration:**
  - `messageId`: `={{ $json.id }}`
  - Remove labelIds: `UNREAD`
- **Inputs:** Each item from Get many messages.
- **Outputs:** Not connected further (side-effect only).
- **Edge cases / failures:**
  - Message already read → label removal is usually safe but may return no-op.
  - Permission issues if mailbox scope is insufficient.

#### Node: Get a message
- **Type / role:** `Gmail (get)` — fetches the full message payload.
- **Key configuration:**
  - `messageId`: `={{ $json.id }}`
  - `simple: false` (expects richer payload, headers/parts)
- **Inputs:** Each item from Get many messages.
- **Outputs:** To **Import body emails**.
- **Edge cases / failures:**
  - Some emails have complex MIME structures; body extraction might be incomplete unless handled by the next step.
  - Very large emails/attachments can increase execution time.

---

### 2.2 Per-email summarization (LLM)
**Overview:** For each fetched email, an LLM agent generates a concise, Notion-friendly summary with strict formatting and length rules.

**Nodes involved:**
- OpenAI Chat Model
- Import body emails

#### Node: OpenAI Chat Model
- **Type / role:** `LangChain ChatOpenAI` — provides the language model to the agent node.
- **Key configuration:**
  - Model: `gpt-5.2`
- **Credentials:** OpenAI API (“OpenAi account”).
- **Connections:** Connected via **ai_languageModel** to **Import body emails** (agent uses this model).
- **Edge cases / failures:**
  - Model availability/name differences across accounts/regions.
  - Token limits if email bodies are long; agent may truncate/lose context.

#### Node: Import body emails
- **Type / role:** `LangChain Agent` — prompt-driven summarizer for each email.
- **Key configuration (prompt intent):**
  - Uses the email text: `{{ $json.text }}`
  - Enforces:
    - UPPERCASE section titles (no `#`/`##`)
    - Bullet points with `-`
    - No tables/code blocks
    - Maximum **1600 characters**
    - Must include the **Monday date** of the week (only once at beginning)
    - Exclude events
- **Inputs:** Full Gmail message item (expected to contain a `text` field; depends on Gmail node output mapping in your n8n version).
- **Outputs:** Produces an `output` field (agent standard) with the per-email summary.
- **Connections:** To **Code in JavaScript**.
- **Edge cases / failures:**
  - If `{{$json.text}}` is missing (Gmail payload didn’t produce `text`), the agent will summarize an empty string. You may need an explicit “extract body” step in some environments.
  - Character limit enforcement is requested but not guaranteed; later JS also enforces an overall cap.

---

### 2.3 Aggregation & final editorial recap
**Overview:** Aggregates all per-email summaries into one text blob with separators and length trimming, then runs a second LLM pass to produce a single polished “Weekly Recap” under 1500 characters.

**Nodes involved:**
- Code in JavaScript
- OpenAI Chat Model1
- Final Summary

#### Node: Code in JavaScript
- **Type / role:** `Code` — merges multiple items into one and applies deterministic length trimming.
- **Key logic:**
  - Reads all incoming items: `const allItems = $input.all();`
  - Merges `item.json.output` from each item with separator `\n\n---\n\n`
  - Hard trims to **1800 characters** (note: higher than later 1500 requirement)
  - Trims at last `.` within the 1800 window to end cleanly
  - Appends: `(Testo troncato per limiti di lunghezza...)`
  - Outputs single item: `{ final_summary: mergedText }`
- **Inputs:** Multiple items from Import body emails.
- **Outputs:** Single item to **Final Summary**.
- **Edge cases / failures:**
  - If any item lacks `json.output`, merged text may contain `undefined` segments.
  - If there are 0 items, `mergedText` becomes `""` and final output is empty.
  - The trimming is character-based, not token-based.

#### Node: OpenAI Chat Model1
- **Type / role:** `LangChain ChatOpenAI` — model provider for the final recap agent.
- **Key configuration:** Model `gpt-5.2`
- **Credentials:** OpenAI API.
- **Connections:** `ai_languageModel` → **Final Summary**.
- **Edge cases:** Same as other OpenAI model node.

#### Node: Final Summary
- **Type / role:** `LangChain Agent` — generates the final consolidated weekly recap.
- **Key configuration (prompt intent):**
  - Consumes: `{{ $json.final_summary }}`
  - Produces a single narrative text
  - Must be **under 1500 characters** including spaces
  - Conversational but structured (bullet points allowed)
  - No Markdown; use UPPERCASE for titles, bold for emphasis (note: “bold” is mentioned but agent also says “Do not use Markdown formatting”—this is slightly contradictory; most models will avoid markdown and may ignore “bold”.)
  - Starts with: `WEEKLY RECAP:` + Monday date
  - Ignore spam/unimportant emails if needed
- **Inputs:** Single item from Code node.
- **Outputs / branching:**
  - To **Create a text block Notion**
  - To **Convert text to speech**
- **Edge cases / failures:**
  - If the merged raw summaries are weak/empty, recap will be generic.
  - Strict char limit may be violated; consider adding a post-check truncation step if required.

---

### 2.4 Publish to Notion
**Overview:** Appends the final recap as a text block into a Notion page/block.

**Nodes involved:**
- Create a text block Notion

#### Node: Create a text block Notion
- **Type / role:** `Notion (block)` — appends a rich text block to a target block/page.
- **Key configuration:**
  - Target `blockId`: provided as a Notion URL (`https://www.notion.so/ZZZZZZZ`)
  - Block content: rich text set to `={{ $json.output }}`
- **Inputs:** From Final Summary (expects `output` from agent).
- **Outputs:** Not connected further (publication side-effect).
- **Edge cases / failures:**
  - Notion permission errors if the integration is not shared with the page.
  - Invalid block URL/ID parsing.
  - Notion rate limits if run on large schedules.

---

### 2.5 Audio generation & Drive publishing
**Overview:** Creates an audio version of the final recap, uploads it to Google Drive, shares it publicly, builds a direct download URL, and appends that link into Notion.

**Nodes involved:**
- Convert text to speech (ElevenLabs)
- Generate audio (OpenAI TTS)
- Upload file (Google Drive)
- Share file (Google Drive)
- Create Google Drive URL
- Create a link to audio file in Notion

#### Node: Convert text to speech
- **Type / role:** `ElevenLabs (speech)` — generates speech audio from the recap text.
- **Key configuration:**
  - Text: `={{ $json.output }}`
  - Voice ID: `SAz9YHcvj6GT2YYXdXww`
  - **onError:** `continueErrorOutput` (workflow continues even if this fails)
- **Inputs:** From Final Summary.
- **Outputs / connections:**
  - Output 0 → **Upload file**
  - Output 1 → **Generate audio**
- **Important note:** In n8n, “continue on fail” typically routes failed items to an error output. Here, the node has two outgoing connections; the second path likely receives error items (depending on node implementation). This creates a fallback: use OpenAI TTS when ElevenLabs fails.
- **Edge cases / failures:**
  - ElevenLabs quota/voice access issues.
  - Text too long or unsupported characters.
  - If ElevenLabs succeeds, the OpenAI fallback path may not run (depending on whether output 1 is strictly error output).

#### Node: Generate audio
- **Type / role:** `OpenAI (audio)` — text-to-speech generation as fallback/alternate.
- **Key configuration:**
  - Input text: `={{ $('Final Summary').item.json.output }}`
  - Model: `tts-1-hd`
  - Voice: `nova`
  - Resource: `audio`
- **Inputs:** Likely receives items from the error branch of ElevenLabs node.
- **Outputs:** To **Upload file**.
- **Edge cases / failures:**
  - If this runs with no items (because ElevenLabs didn’t fail), it does nothing.
  - OpenAI TTS model availability and quota.

#### Node: Upload file
- **Type / role:** `Google Drive` — uploads the produced MP3 audio.
- **Key configuration:**
  - File name: `News{{$now.format("yyyyMMdd")}}.mp3`
  - Destination folder: “Audio News” (folder ID `1pSgLdNHT-v1WVLB1X8vb1g8OPmw7Zx6C`)
  - Drive: “My Drive”
- **Inputs:** Audio binary output from either ElevenLabs or OpenAI TTS node (must be compatible with Drive upload expectations).
- **Outputs:** To **Share file**.
- **Edge cases / failures:**
  - If audio node returns data not in expected binary property, upload fails (may require setting “Binary Property” in Drive node).
  - File name collisions (same day reruns) may create duplicates.

#### Node: Share file
- **Type / role:** `Google Drive (share)` — sets file permissions to public readable.
- **Key configuration:**
  - `fileId`: `={{ $json.id }}`
  - Permissions: `type=anyone`, `role=reader`
- **Inputs:** From Upload file (expects uploaded file metadata with `id`).
- **Outputs:** To **Create Google Drive URL**.
- **Edge cases / failures:**
  - Domain policies may prevent “anyone” sharing.
  - Permission propagation delays (link may not work immediately in rare cases).

#### Node: Create Google Drive URL
- **Type / role:** `Set` — constructs a direct download URL for the uploaded file.
- **Key configuration:**
  - `audio_url`:
    - `https://drive.google.com/uc?export=download&id={{ $('Upload file').item.json.id }}`
- **Inputs:** From Share file.
- **Outputs:** To **Create a link to audio file in Notion**.
- **Edge cases / failures:**
  - Uses `$('Upload file').item.json.id` (cross-node reference). If multiple items exist, `.item` resolution can be ambiguous. Consider using `{{$json.id}}` from the current item if Share file preserves it.
  - Google may sometimes require confirmation for large files; this direct link can fail for very large audio.

#### Node: Create a link to audio file in Notion
- **Type / role:** `Notion (block)` — appends a link block to the Notion page/block.
- **Key configuration:**
  - Target `blockId`: `https://www.notion.so/ZZZZZZ`
  - Block text: `🎧 Audio Summary: {{ $json.audio_url }}`
  - The text is marked as a link (`textLink` set to the same URL)
- **Inputs:** From Create Google Drive URL (expects `audio_url`).
- **Edge cases / failures:**
  - Same Notion permission/block ID issues as the text block node.
  - If audio_url is missing, it will post an invalid link.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly trigger | Schedule Trigger | Weekly entrypoint | — | Get many messages | ## How it works  / (full workflow explanation + setup steps) |
| Get many messages | Gmail | Search and list recent emails from specific senders | Weekly trigger | Mark as read; Get a message | ## Gmail flow  / Retrieve emails with AI subjects. Tag as READ |
| Mark as read | Gmail | Remove UNREAD label | Get many messages | — | ## Gmail flow  / Retrieve emails with AI subjects. Tag as READ |
| Get a message | Gmail | Fetch full email content | Get many messages | Import body emails | ## Gmail flow  / Retrieve emails with AI subjects. Tag as READ |
| OpenAI Chat Model | LangChain ChatOpenAI | LLM provider for per-email summary agent | — | Import body emails (ai_languageModel) | ## Create summary text  / Summarize with LLM |
| Import body emails | LangChain Agent | Summarize each email body to short structured text | Get a message; OpenAI Chat Model | Code in JavaScript | ## Create summary text  / Summarize with LLM |
| Code in JavaScript | Code | Aggregate all summaries and enforce length | Import body emails | Final Summary |  |
| OpenAI Chat Model1 | LangChain ChatOpenAI | LLM provider for final recap agent | — | Final Summary (ai_languageModel) |  |
| Final Summary | LangChain Agent | Produce single weekly recap under 1500 chars | Code in JavaScript; OpenAI Chat Model1 | Create a text block Notion; Convert text to speech |  |
| Create a text block Notion | Notion | Append final recap text to Notion | Final Summary | — |  |
| Convert text to speech | ElevenLabs | Primary TTS generation (continue on fail) | Final Summary | Upload file; Generate audio | ## Audio transcript / Generate an audio transcript and upload to Google Drive |
| Generate audio | OpenAI (Audio/TTS) | Fallback/alternate TTS generation | Convert text to speech (error output) | Upload file | ## Audio transcript / Generate an audio transcript and upload to Google Drive |
| Upload file | Google Drive | Upload MP3 to Drive folder | Convert text to speech or Generate audio | Share file | ## Audio transcript / Generate an audio transcript and upload to Google Drive |
| Share file | Google Drive | Set public reader permissions | Upload file | Create Google Drive URL | ## Audio transcript / Generate an audio transcript and upload to Google Drive |
| Create Google Drive URL | Set | Build direct download URL | Share file | Create a link to audio file in Notion | ## Audio transcript / Generate an audio transcript and upload to Google Drive |
| Create a link to audio file in Notion | Notion | Append public audio link to Notion | Create Google Drive URL | — | ## Audio transcript / Generate an audio transcript and upload to Google Drive |
| Sticky Note | Sticky Note | Comment | — | — | ## Gmail flow  / Retrieve emails with AI subjects. Tag as READ |
| Sticky Note1 | Sticky Note | Comment | — | — | ## Audio transcript / Generate an audio transcript and upload to Google Drive |
| Sticky Note2 | Sticky Note | Comment | — | — | ## How it works (full text includes setup steps) |
| Sticky Note3 | Sticky Note | Comment | — | — | ## Create summary text / Summarize with LLM |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Weekly trigger” (Schedule Trigger)**
- Set interval to **weeks**.
- Set weekday to **6** and time to **12:32** (adjust to your timezone/desired day).

2) **Add “Get many messages” (Gmail → Get All)**
- Connect from Weekly trigger.
- Operation: **Get All**.
- Filters / query: `newer_than:7d (from:xxx@xxx.ai OR from:yyy@yyy.com)`
- Add Gmail OAuth2 credentials.

3) **Add “Mark as read” (Gmail → Remove Labels)**
- Connect from Get many messages.
- Operation: remove labels.
- `messageId`: expression `{{$json.id}}`
- Labels to remove: `UNREAD`

4) **Add “Get a message” (Gmail → Get)**
- Connect from Get many messages.
- Operation: **Get**
- `messageId`: `{{$json.id}}`
- Set `simple` to **false** (to retrieve structured payload)
- Ensure Gmail credentials are set.

5) **Add “OpenAI Chat Model” (LangChain ChatOpenAI)**
- Model: `gpt-5.2`
- Connect OpenAI API credentials.

6) **Add “Import body emails” (LangChain Agent)**
- Connect main input from **Get a message**.
- In the agent prompt, paste and adapt:

  - Uses email text: `{{ $json.text }}`
  - Formatting rules (UPPERCASE titles, `-` bullets, no tables/code)
  - Max 1600 characters
  - Include Monday date once at top
  - Exclude events

- Connect **OpenAI Chat Model** to the agent’s **ai_languageModel** input.

7) **Add “Code in JavaScript” (Code node)**
- Connect from Import body emails.
- Paste logic to merge all `item.json.output`, join with separators, trim to 1800 chars, end on last period, output `final_summary`.

8) **Add “OpenAI Chat Model1” (LangChain ChatOpenAI)**
- Model: `gpt-5.2`
- OpenAI credentials.

9) **Add “Final Summary” (LangChain Agent)**
- Connect from Code in JavaScript (main).
- Prompt should:
  - Consume `{{ $json.final_summary }}`
  - Produce one recap under 1500 characters
  - Start with `WEEKLY RECAP:` + Monday date
  - Avoid Markdown
- Connect **OpenAI Chat Model1** to **Final Summary** via **ai_languageModel**.

10) **Add “Create a text block Notion” (Notion → Block)**
- Connect from Final Summary.
- Resource: **block** (append block).
- Provide target Notion block/page URL.
- Set rich text content to `{{$json.output}}`.
- Configure Notion credentials; ensure the integration is shared with the target page.

11) **Add “Convert text to speech” (ElevenLabs → Speech)**
- Connect from Final Summary.
- Text: `{{$json.output}}`
- Select voice ID.
- Enable “Continue on fail” (so failures route to error output).
- Configure ElevenLabs credentials.

12) **Add “Generate audio” (OpenAI → Audio / TTS)**
- Connect from the **error output** of “Convert text to speech” (second output).
- Input: `{{ $('Final Summary').item.json.output }}`
- Model: `tts-1-hd`
- Voice: `nova`
- OpenAI credentials.

13) **Add “Upload file” (Google Drive → Upload)**
- Connect from:
  - “Convert text to speech” success output, and
  - “Generate audio” output
- File name: `News{{$now.format("yyyyMMdd")}}.mp3`
- Choose Drive and folder (“Audio News”).
- Configure Google Drive OAuth2 credentials.
- Verify which **binary property** the audio node outputs; set Drive upload to read that binary if required.

14) **Add “Share file” (Google Drive → Share)**
- Connect from Upload file.
- Operation: **share**
- `fileId`: `{{$json.id}}`
- Permissions: `anyone` + `reader`

15) **Add “Create Google Drive URL” (Set node)**
- Connect from Share file.
- Create field `audio_url`:
  - `https://drive.google.com/uc?export=download&id={{ $('Upload file').item.json.id }}`
  - (Optionally replace with `{{$json.id}}` if the shared item includes the file id reliably.)

16) **Add “Create a link to audio file in Notion” (Notion → Block)**
- Connect from Create Google Drive URL.
- Append a rich text block containing:
  - Visible text: `🎧 Audio Summary: {{ $json.audio_url }}`
  - Link target: `{{ $json.audio_url }}`
- Use the same Notion credentials and correct target block/page URL.

17) **(Optional) Add sticky notes**
- Gmail flow note around trigger/Gmail nodes.
- Summary note around LLM summarization nodes.
- Audio transcript note around TTS/Drive/Notion link nodes.
- “How it works” note for documentation inside the canvas.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “## Gmail flow Retrieve emails with AI subjects. Tag as READ” | Canvas note describing Gmail retrieval and read-status handling |
| “## Create summary text Summarize with LLM” | Canvas note for the per-email summarization section |
| “## Audio transcript **Generate an audio transcript and upload to Google Drive” | Canvas note for TTS + Drive upload + sharing |
| Full “How it works” explanation + setup steps (as provided in the sticky note) | Canvas note describing end-to-end behavior and initial configuration guidance |