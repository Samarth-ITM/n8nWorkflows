Translate and dub YouTube videos using BrowserAct, Telegram, Gemini & ElevenLabs

https://n8nworkflows.xyz/workflows/translate-and-dub-youtube-videos-using-browseract--telegram--gemini---elevenlabs-12361


# Translate and dub YouTube videos using BrowserAct, Telegram, Gemini & ElevenLabs

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Translate and dub YouTube videos using BrowserAct, Telegram, Gemini & ElevenLabs  
**Workflow name (in JSON):** Translate and dub YouTube videos using BrowserAct, Telegrma & Gemini  
**Purpose:** When a user sends a message to a Telegram bot containing a YouTube link, the workflow extracts the URL, scrapes the video transcript via BrowserAct, translates and restructures it into dubbing segments using an LLM (Gemini via OpenRouter + structured output parsing), generates audio for each segment with ElevenLabs, and sends a localized Telegram summary + the dubbed audio files back to the user.

**Target use cases**
- Fast localization of YouTube content for Telegram communities
- Creating dubbed audio segments from a transcript (multi-part, Telegram-friendly delivery)
- Semi-automated “send link → receive translated summary + dubbed audio” flow

### Logical Blocks
**1.1 Telegram Input Reception**
- Entry point: Telegram trigger receives messages.

**1.2 URL Extraction & Early User Feedback**
- LLM agent extracts a clean YouTube URL (or returns `NULL`).
- Sends an “initialization” message to the user in Telegram.
- In parallel, kicks off transcript extraction.

**1.3 Transcript Scraping (BrowserAct) + Target Language Definition**
- BrowserAct workflow scrapes transcript/metadata from YouTube using the extracted URL.
- A Set node defines the target language (default: Spanish).

**1.4 Translation + Dub Script Generation (LLM + Structured Parsing)**
- LLM agent translates transcript, removes timestamps/speaker labels, splits into parts, and generates:
  - a Telegram HTML post (`telegram_post`)
  - a list of dubbing segments (`dubbing_parts[]`)
- Uses an LLM model node (OpenRouter Gemini) + a structured output parser.

**1.5 Delivery to Telegram + Audio Generation Loop**
- Sends the Telegram summary post.
- Splits `dubbing_parts` into items and loops over them.
- For each part: ElevenLabs generates audio → Telegram sends it as an audio message with caption.

---

## 2. Block-by-Block Analysis

### 2.1 Telegram Input Reception
**Overview:** Listens for incoming Telegram messages and forwards the payload into the workflow.

**Nodes Involved**
- **User Sends Message to Bot**

**Node Details**
- **User Sends Message to Bot**
  - **Type / role:** `telegramTrigger` — workflow entry point.
  - **Configuration:** Listens to Telegram `message` updates.
  - **Key data used later:** `{{$json.message.text}}` and `{{$json.message.chat.id}}`.
  - **Outputs:** Main output to **Analyze user Input**.
  - **Potential failures / edge cases:**
    - Telegram webhook misconfiguration or invalid credentials.
    - Message updates that don’t contain `.text` (stickers, photos). This workflow assumes `.message.text` exists.

---

### 2.2 URL Extraction & Early User Feedback
**Overview:** Extracts a YouTube URL from the user’s message via an LLM agent. Immediately notifies the user that processing has started, and triggers transcript extraction.

**Nodes Involved**
- **Validate inputs**
- **Analyze user Input**
- **Process Initialization Alert**

**Node Details**
- **Validate inputs**
  - **Type / role:** `lmChatGoogleGemini` — provides a Gemini chat model used by the agent node.
  - **Configuration choices:** Default options; uses Google Gemini (PaLM) credentials.
  - **Connections:** Supplies the AI language model to **Analyze user Input** via `ai_languageModel`.
  - **Potential failures:** Invalid/expired Google Gemini API key, quota limits, or model availability.

- **Analyze user Input**
  - **Type / role:** `langchain.agent` — performs URL extraction.
  - **Config choices (interpreted):**
    - Input text: `={{ $json.message.text }}`
    - System message strictly enforces: output only the YouTube URL or `NULL`.
    - `hasOutputParser: true` (agent expects structured/parsed behavior, though output is plain URL text per instructions).
  - **Outputs / connections:**
    - Main output goes to:
      - **Process Initialization Alert**
      - **Extract Youtube Transcript**
  - **Edge cases / failures:**
    - If the user message contains no URL, the agent outputs `NULL`. Downstream nodes currently do not explicitly branch on `NULL`, so BrowserAct may run with an invalid URL.
    - Non-YouTube URLs could be incorrectly returned if the model fails instruction-following.
    - If Telegram message is missing `message.text`, expression evaluates to `undefined` and the agent may respond unexpectedly.

- **Process Initialization Alert**
  - **Type / role:** `telegram` — sends a quick status message.
  - **Configuration choices:**
    - Text: `I've got you; give me a minute.`
    - Chat ID: `={{ $('User Sends Message to Bot').first().json.message.chat.id }}`
  - **Potential failures:**
    - Chat ID expression fails if trigger item is absent or structure differs.
    - Telegram rate limits if used heavily.

---

### 2.3 Transcript Scraping (BrowserAct) + Target Language Definition
**Overview:** Uses BrowserAct to scrape a YouTube transcript (and possibly other metadata) from the extracted URL. Sets the target localization language (default Spanish).

**Nodes Involved**
- **Extract Youtube Transcript**
- **Define Language**

**Node Details**
- **Extract Youtube Transcript**
  - **Type / role:** `browserAct` — runs a hosted browser automation workflow.
  - **Configuration choices:**
    - Mode: `WORKFLOW`
    - Timeout: `7200` seconds (2 hours) for long runs
    - BrowserAct workflow ID: `70752502343921927`
    - Input mapping: `Target_Video` ← `={{ $json.output }}`
      - This expects **Analyze user Input** to output the URL under `$json.output`. (If it outputs plain text differently, mapping may break.)
  - **Outputs:** Main output to **Define Language**.
  - **Likely output structure:** The downstream agent references `$('Extract Youtube Transcript').first().json.output.string`, implying BrowserAct returns something like `{ output: { string: "..." } }`.
  - **Potential failures / edge cases:**
    - BrowserAct credentials/workflow ID wrong.
    - YouTube page layout changes can break scraping.
    - Videos without transcripts, disabled transcripts, region restrictions.
    - If input URL is `NULL` or malformed, BrowserAct run fails or returns empty transcript.

- **Define Language**
  - **Type / role:** `set` — defines `Target_Language`.
  - **Configuration choices:**
    - Sets `Target_Language` to `"Spanish"`.
  - **Outputs:** Main output to **Analyze Transcript and Generate Dub**.
  - **Edge cases:** None significant; but it’s static—no user selection implemented.

---

### 2.4 Translation + Dub Script Generation (LLM + Structured Parsing)
**Overview:** Translates transcript into the target language, creates a Telegram HTML summary, and segments dubbing scripts into parts that are later converted to speech.

**Nodes Involved**
- **OpenRouter Chat Model**
- **Structured Output**
- **Analyze Transcript and Generate Dub**
- **Check Output** (connected as the language model provider for the parser)

**Node Details**
- **OpenRouter Chat Model**
  - **Type / role:** `lmChatOpenRouter` — provides the LLM used by the translation/structuring agent.
  - **Configuration choices:**
    - Model: `google/gemini-2.5-pro`
  - **Connections:** Supplies `ai_languageModel` into **Analyze Transcript and Generate Dub**.
  - **Potential failures:** OpenRouter auth/quota issues; model name availability changes.

- **Structured Output**
  - **Type / role:** `outputParserStructured` — enforces JSON output schema and auto-fixes if possible.
  - **Configuration choices:**
    - `autoFix: true` (attempts to repair slightly invalid JSON)
    - Schema example includes:
      - `telegram_post` (HTML string)
      - `dubbing_parts[]` items with `text`, `file_name`, `caption`
  - **Connections:**
    - Provides `ai_outputParser` to **Analyze Transcript and Generate Dub**.
    - Receives `ai_languageModel` from **Check Output** (see next node).
  - **Potential failures / edge cases:**
    - If the agent returns severely malformed JSON, auto-fix may fail.
    - If the agent’s output diverges from schema (missing keys), downstream nodes will break (`output.telegram_post`, `output.dubbing_parts`).

- **Check Output**
  - **Type / role:** `lmChatGoogleGemini` — used here as a language model feeding the structured parser.
  - **Configuration choices:** Default options; Gemini(PaLM) credentials.
  - **Connections:** `ai_languageModel` → **Structured Output**.
  - **Important note:** This is unusual but valid in n8n AI tooling: the output parser can itself use an LLM to help “auto-fix”/coerce structure.

- **Analyze Transcript and Generate Dub**
  - **Type / role:** `langchain.agent` — core transformation: translation, cleanup, splitting, and post generation.
  - **Configuration (interpreted):**
    - Text field is built from two sources:
      - `Target_Language` from **Define Language**
      - Transcript/content from **Extract Youtube Transcript**
    - Expression used:
      - `video_data : {{ $('Define Language').first().json.Target_Language }}`
      - `Target language :{{ $('Extract Youtube Transcript').first().json.output.string }}`
    - **Caution:** The labels appear swapped/misleading: the first line is “video_data” but inserts language; the second is “Target language” but inserts transcript output. The system prompt expects a JSON object with `video_link`, `description`, `transcript`, `target_language`. Current input construction likely does **not** match that expectation unless BrowserAct output already embeds these fields in `.output.string`.
  - **System instructions:** Produce strict JSON with:
    - `telegram_post` (Telegram HTML)
    - `dubbing_parts` (split if >2500 chars; remove timestamps/speaker labels)
  - **Connections / outputs:**
    - Main → **Send Summary Back to Bot**
    - Main → **Split Generated Dubbed Content**
  - **Potential failures / edge cases:**
    - If BrowserAct returns empty transcript, the agent may output minimal/empty dubbing parts.
    - Telegram HTML constraints: invalid tags may cause Telegram parse errors.
    - If transcript is very long, segmentation may still exceed Telegram/ElevenLabs practical limits.
    - Input mismatch (language/transcript swapped) can degrade results significantly.

---

### 2.5 Delivery to Telegram + Audio Generation Loop
**Overview:** Sends the localized summary post first, then iterates over dubbing parts to generate audio and deliver each file to the user.

**Nodes Involved**
- **Send Summary Back to Bot**
- **Split Generated Dubbed Content**
- **Loop Over Generated Items**
- **Convert text to speech**
- **Send Dubbed Audio File**

**Node Details**
- **Send Summary Back to Bot**
  - **Type / role:** `telegram` — sends the summary message.
  - **Configuration:**
    - Text: `={{ $json.output.telegram_post }}`
    - Chat ID: `={{ $('User Sends Message to Bot').first().json.message.chat.id }}`
    - Parse mode: HTML
    - `appendAttribution: false`
  - **Potential failures:**
    - If `output.telegram_post` missing → expression error.
    - HTML invalid → Telegram rejects or renders incorrectly.

- **Split Generated Dubbed Content**
  - **Type / role:** `splitOut` — expands an array into individual items.
  - **Configuration:**
    - Field to split: `output.dubbing_parts`
  - **Output:** Each item becomes one execution item going into **Loop Over Generated Items**.
  - **Edge cases:** If `output.dubbing_parts` is empty or not an array, the flow may stop or error.

- **Loop Over Generated Items**
  - **Type / role:** `splitInBatches` — processes items sequentially (batch loop).
  - **Configuration:** Default options (batch size defaults to 1 in many setups, but not explicitly set here).
  - **Connections:**
    - Output 1 (loop body) → **Convert text to speech**
    - Output 0 (done) → nothing
  - **Edge cases:** If batch size is not 1, you may send audio in groups, affecting captions/filenames.

- **Convert text to speech**
  - **Type / role:** `elevenLabs` — converts each dubbing part text to audio.
  - **Configuration:**
    - Resource: Speech (TTS)
    - Text input: `={{ $json.text }}`
    - Voice: `Liam - Energetic, Social Media Creator` (voice ID `TX3LPaxmHKxFdv7VOQHJ`)
    - Model: `Eleven Flash v2.5` (`eleven_flash_v2_5`)
  - **Output:** Binary audio data (used by Telegram sendAudio).
  - **Potential failures / edge cases:**
    - Text length limits or unsupported characters.
    - Rate limits/quota at ElevenLabs.
    - If `$json.text` is missing (schema mismatch), node fails.

- **Send Dubbed Audio File**
  - **Type / role:** `telegram` — sends an audio file to the same chat.
  - **Configuration:**
    - Operation: `sendAudio`
    - `binaryData: true` (expects binary from ElevenLabs node)
    - Chat ID: from trigger
    - Caption: `={{ $('Loop Over Generated Items').first().json.caption }}`
    - File name: `={{ $('Loop Over Generated Items').first().json.file_name }}`
    - Parse mode: HTML
  - **Connections:** Main → **Loop Over Generated Items** (to request next batch/iteration).
  - **Edge cases / failures:**
    - Using `first()` inside a loop can be risky: it always references the first item of the loop context, not necessarily the current batch item. Prefer `{{$json.caption}}` / `{{$json.file_name}}` if the loop item is passed forward as current item.
    - Telegram limits: audio file size, caption length, HTML validity.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| User Sends Message to Bot | telegramTrigger | Entry point: receive Telegram messages | — | Analyze user Input | ### 🔍 Step 1: Input Analysis & Extraction — The workflow intercepts Telegram messages to identify YouTube links... |
| Validate inputs | lmChatGoogleGemini | LLM provider for URL extraction agent | — | Analyze user Input (ai_languageModel) | ### 🔍 Step 1: Input Analysis & Extraction — The workflow intercepts Telegram messages to identify YouTube links... |
| Analyze user Input | langchain.agent | Extract YouTube URL from message | User Sends Message to Bot; Validate inputs (ai) | Process Initialization Alert; Extract Youtube Transcript | ### 🔍 Step 1: Input Analysis & Extraction — The workflow intercepts Telegram messages to identify YouTube links... |
| Process Initialization Alert | telegram | Send “processing started” message | Analyze user Input | — | ### 🔍 Step 1: Input Analysis & Extraction — The workflow intercepts Telegram messages to identify YouTube links... |
| Extract Youtube Transcript | browserAct | Scrape transcript/metadata from YouTube | Analyze user Input | Define Language | ### 📜 Step 2: Transcript Scrape & Translation — BrowserAct automates a browser to extract the full transcript... |
| Define Language | set | Set target language (Spanish) | Extract Youtube Transcript | Analyze Transcript and Generate Dub | ### 📜 Step 2: Transcript Scrape & Translation — BrowserAct automates a browser to extract the full transcript... |
| OpenRouter Chat Model | lmChatOpenRouter | LLM provider for translation/structuring agent | — | Analyze Transcript and Generate Dub (ai_languageModel) | ### 📜 Step 2: Transcript Scrape & Translation — BrowserAct automates a browser to extract the full transcript... |
| Check Output | lmChatGoogleGemini | LLM used by structured parser auto-fix | — | Structured Output (ai_languageModel) | ### 📜 Step 2: Transcript Scrape & Translation — BrowserAct automates a browser to extract the full transcript... |
| Structured Output | outputParserStructured | Enforce/repair JSON schema for agent output | Check Output (ai) | Analyze Transcript and Generate Dub (ai_outputParser) | ### 📜 Step 2: Transcript Scrape & Translation — BrowserAct automates a browser to extract the full transcript... |
| Analyze Transcript and Generate Dub | langchain.agent | Generate telegram_post + dubbing_parts JSON | Define Language; Extract Youtube Transcript; OpenRouter Chat Model (ai); Structured Output (parser) | Send Summary Back to Bot; Split Generated Dubbed Content | ### 📜 Step 2: Transcript Scrape & Translation — BrowserAct automates a browser to extract the full transcript... |
| Send Summary Back to Bot | telegram | Send localized HTML summary post | Analyze Transcript and Generate Dub | — | ### 🎙️ Step 3: Dubbing Generation & Asset Delivery — The translated text is split into manageable segments... |
| Split Generated Dubbed Content | splitOut | Split dubbing_parts into items | Analyze Transcript and Generate Dub | Loop Over Generated Items | ### 🎙️ Step 3: Dubbing Generation & Asset Delivery — The translated text is split into manageable segments... |
| Loop Over Generated Items | splitInBatches | Sequential loop over dubbing parts | Split Generated Dubbed Content; Send Dubbed Audio File | Convert text to speech | ### 🎙️ Step 3: Dubbing Generation & Asset Delivery — The translated text is split into manageable segments... |
| Convert text to speech | elevenLabs | TTS audio generation per part | Loop Over Generated Items | Send Dubbed Audio File | ### 🎙️ Step 3: Dubbing Generation & Asset Delivery — The translated text is split into manageable segments... |
| Send Dubbed Audio File | telegram | Send audio file to Telegram | Convert text to speech | Loop Over Generated Items | ### 🎙️ Step 3: Dubbing Generation & Asset Delivery — The translated text is split into manageable segments... |
| Documentation | stickyNote | Comment / setup notes | — | — | ## ⚡ Workflow Overview & Setup — Summary, requirements, links to BrowserAct docs |
| Step 1 Explanation | stickyNote | Comment: step 1 | — | — | ### 🔍 Step 1: Input Analysis & Extraction — (content as shown) |
| Step 2 Explanation | stickyNote | Comment: step 2 | — | — | ### 📜 Step 2: Transcript Scrape & Translation — (content as shown) |
| Step 3 Explanation | stickyNote | Comment: step 3 | — | — | ### 🎙️ Step 3: Dubbing Generation & Asset Delivery — (content as shown) |
| Sticky Note | stickyNote | Comment/link/embed | — | — | @[youtube](vGJ2TdfGMpk) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Telegram Trigger**
   - Add node: **Telegram Trigger**
   - Updates: **message**
   - Credentials: connect your **Telegram API** (bot token)
   - This is the entry node.

2) **Add URL extraction agent**
   - Add node: **Google Gemini Chat Model** (LangChain)  
     - Credentials: **Google Gemini (PaLM)** API key
   - Add node: **AI Agent (LangChain Agent)**
     - Text: `{{$json.message.text}}`
     - System message: (use the workflow’s URL-only rules; return URL or `NULL`)
     - Enable output parsing if desired (optional here).
   - Connect: **Telegram Trigger → AI Agent**
   - Connect: **Gemini Chat Model (ai_languageModel) → AI Agent**

3) **Send an initialization message**
   - Add node: **Telegram**
   - Operation: send message
   - Chat ID: `{{$('User Sends Message to Bot').first().json.message.chat.id}}`
   - Text: `I've got you; give me a minute.`
   - Connect: **AI Agent → Telegram (init alert)**

4) **Scrape transcript using BrowserAct**
   - Add node: **BrowserAct**
   - Type: **WORKFLOW**
   - Timeout: **7200**
   - Workflow ID: set to your BrowserAct template workflow ID (expected template: “YouTube Translator & Auto Dubber” per sticky note)
   - Map input variable (example name): **Target_Video**
     - Value expression: from the URL extraction node output (ensure it matches your agent output field; often you’ll use `{{$json.output}}` or `{{$json.text}}`)
   - Connect: **AI Agent → BrowserAct**

5) **Set target language**
   - Add node: **Set**
   - Add field: `Target_Language` = `"Spanish"` (or your preferred default)
   - Connect: **BrowserAct → Set**

6) **Prepare translation + structuring LLM components**
   - Add node: **OpenRouter Chat Model**
     - Credentials: OpenRouter API key
     - Model: `google/gemini-2.5-pro`
   - Add node: **Google Gemini Chat Model** (optional helper model for parser auto-fix, as in the JSON)
     - Credentials: same Gemini credentials
   - Add node: **Structured Output Parser**
     - Enable **Auto-fix**
     - Provide schema example with:
       - `telegram_post` (string)
       - `dubbing_parts` (array of objects with `text`, `file_name`, `caption`)
   - Connect: **Gemini Chat Model (parser helper) → Structured Output Parser (ai_languageModel)**

7) **Add “Analyze Transcript and Generate Dub” agent**
   - Add node: **AI Agent**
   - System message: include rules for:
     - Telegram HTML post under 1000 chars
     - Clean transcript (remove timestamps, labels)
     - Split if long (>2500 chars)
     - Output strict JSON only
   - Text input: pass *both* transcript and target language (ideally as proper fields). For example, construct something like:
     - `target_language: {{$('Define Language').first().json.Target_Language}}`
     - `transcript: {{$('Extract Youtube Transcript').first().json.output.string}}`
     - plus `video_link` and `description` if BrowserAct provides them
   - Connect:
     - **OpenRouter Chat Model (ai_languageModel) → Agent**
     - **Structured Output Parser (ai_outputParser) → Agent**
     - **Set (Define Language) → Agent** (main)
   - Ensure the agent receives the transcript (either via the Set node merging data, or by referencing BrowserAct output via expressions).

8) **Send summary back to Telegram**
   - Add node: **Telegram**
   - Text: `{{$json.output.telegram_post}}`
   - Chat ID: `{{$('User Sends Message to Bot').first().json.message.chat.id}}`
   - Parse mode: **HTML**
   - Connect: **Translation Agent → Telegram (summary)**

9) **Split dubbing parts**
   - Add node: **Split Out**
   - Field to split: `output.dubbing_parts`
   - Connect: **Translation Agent → Split Out**

10) **Loop sequentially**
   - Add node: **Split In Batches**
   - (Optionally set batch size = 1 explicitly)
   - Connect: **Split Out → Split In Batches**

11) **Generate TTS with ElevenLabs**
   - Add node: **ElevenLabs**
   - Resource: **Speech**
   - Text: `{{$json.text}}`
   - Select a voice (e.g., “Liam - Energetic, Social Media Creator”)
   - Model: “Eleven Flash v2.5” (or your chosen model)
   - Credentials: ElevenLabs API key
   - Connect: **Split In Batches → ElevenLabs**

12) **Send audio to Telegram and continue loop**
   - Add node: **Telegram**
   - Operation: **sendAudio**
   - Enable **binaryData**
   - Chat ID: from trigger
   - Caption + filename: preferably use the *current* item:
     - Caption: `{{$json.caption}}`
     - File name: `{{$json.file_name}}`
   - Connect: **ElevenLabs → Telegram (sendAudio)**
   - Connect: **Telegram (sendAudio) → Split In Batches** (to request next item)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Summary:** Takes a YouTube URL from Telegram, scrapes transcript using BrowserAct, translates to target language, generates dubbed audio via ElevenLabs, sends summary + audio back to user. | From sticky note “Documentation” |
| **Requirements:** Credentials for Telegram, BrowserAct, OpenRouter, Google Gemini (PaLM), ElevenLabs. Mandatory: BrowserAct API + template “YouTube Translator & Auto Dubber”. | From sticky note “Documentation” |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| `@[youtube](vGJ2TdfGMpk)` | From sticky note “Sticky Note” (appears to be an embedded reference to a YouTube video ID) |

