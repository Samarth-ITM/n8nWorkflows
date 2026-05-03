Create active learning notes from YouTube videos with Gemini and Notion

https://n8nworkflows.xyz/workflows/create-active-learning-notes-from-youtube-videos-with-gemini-and-notion-12372


# Create active learning notes from YouTube videos with Gemini and Notion

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Convert a YouTube video URL into a **Notion database entry** containing an **Active Learning worksheet** (ICAP-based) generated from an **AI transcript + analysis** (Gemini).  
**Primary use case:** Students/learners who want structured learning goals, Cornell-style prompts, a Feynman explanation, and a hands-on simulation task from any YouTube video.

### 1.1 Input Reception (Form → Video ID)
Receives a YouTube link via an n8n Form Trigger and extracts the 11-character YouTube video ID.

### 1.2 Fetch Video Metadata (YouTube API → Normalization)
Calls the YouTube Data API to fetch title/channel/thumbnail/publish date, then normalizes fields for downstream use.

### 1.3 Audio Retrieval & Transcription (RapidAPI → Binary file → Gemini Audio)
Downloads the video audio via RapidAPI, fetches the audio file as binary, forces the correct MIME type, and transcribes it with Gemini.

### 1.4 Transcript Guard + Data Merge (IF → Merge)
Ensures a transcript exists before continuing, then merges transcript + metadata into a single item.

### 1.5 Active Learning AI Generation (LangChain Agent + Gemini Chat + Output Parser + Memory)
Uses a Gemini chat model via an Agent prompt to produce **structured JSON** (micro-goals, Cornell notes, Feynman explanation, simulation task, key topics).

### 1.6 Notion Page Construction & Storage (Code → Notion API)
Transforms the structured AI JSON and YouTube metadata into a Notion page payload (properties + rich block children) and creates a page in a Notion database.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Form → Video ID)
**Overview:** Collects a YouTube URL from a user-facing form and extracts the video ID required for API calls.  
**Nodes involved:**  
- On form submission  
- SET - Get YT ID

#### Node: **On form submission**
- **Type / Role:** `Form Trigger` — entry point; hosts a simple form and triggers workflow on submission.
- **Key configuration:**
  - Form title: *YouTube to Active Learning Notes*
  - Single required field: **URL** (YouTube Video URL)
  - Custom button label: “Generate Learning Notes”
  - Attribution disabled
- **Input/Output:**
  - **Output:** JSON containing `URL` (user input).
- **Edge cases / failures:**
  - Users may paste non-YouTube links; downstream regex may fail and pass the full URL as “videoId”, causing YouTube node errors.

#### Node: **SET - Get YT ID**
- **Type / Role:** `Set` — derives `VIdeoID` from the submitted URL.
- **Key configuration:**
  - Keeps other fields (`includeOtherFields: true`)
  - Extracts video ID using:
    - `{{$json.URL.match(/(?:v=|youtu.be\/)([a-zA-Z0-9_-]{11})/)?.[1] || $json.URL }}`
- **Input/Output:**
  - **Input:** `URL`
  - **Output:** adds `VIdeoID`
- **Edge cases / failures:**
  - Regex does not handle all formats (e.g., `youtube.com/shorts/…`, `youtube.com/embed/…`, additional parameters without `v=`).
  - If regex fails, `VIdeoID` becomes the entire URL, breaking the YouTube API call.

---

### Block 2 — Fetch Video Metadata (YouTube API → Normalization)
**Overview:** Retrieves video metadata from YouTube and reshapes it into a consistent structure for later merging and Notion rendering.  
**Nodes involved:**  
- YouTube - Fetch Video Details  
- Set - Prepare Video Metadata

#### Node: **YouTube - Fetch Video Details**
- **Type / Role:** `YouTube` — calls YouTube Data API to get video resource details.
- **Key configuration:**
  - Resource: **video**
  - Operation: **get**
  - Video ID: `={{ $json.VIdeoID }}`
- **Credentials / Requirements:**
  - Requires YouTube Data API credentials configured in n8n.
- **Input/Output:**
  - **Input:** `VIdeoID`
  - **Output:** YouTube API response (includes `id`, `snippet.*`)
- **Edge cases / failures:**
  - Invalid/unknown video ID → 404 / empty result.
  - Quota limits / API key restrictions.
  - Private/region-blocked videos can fail or return limited data.

#### Node: **Set - Prepare Video Metadata**
- **Type / Role:** `Set` — maps YouTube response into fields used downstream.
- **Key configuration (assignments):**
  - `videoId = $json.id`
  - `title = $json.snippet.title`
  - `description = $json.snippet.description`
  - `publishedAt = $json.snippet.publishedAt`
  - `thumbnailUrl = maxres || high || default`
  - `videoUrl = 'https://www.youtube.com/watch?v=' + $json.id`
  - `channelTitle = $json.snippet.channelTitle`
- **Input/Output:**
  - **Input:** YouTube video object
  - **Outputs:**
    - To **Merge - Attach Metadata + Transcript** (metadata branch)
    - To **HTTP - Get YouTube Audio** (audio branch)
- **Edge cases / failures:**
  - `maxres` may not exist; fallback logic handles this.
  - If `snippet` is missing due to API limitations, expressions may produce `undefined`.

---

### Block 3 — Audio Retrieval & Transcription (RapidAPI → Gemini Audio)
**Overview:** Downloads YouTube audio via RapidAPI, retrieves the file as binary, ensures correct MIME type, then transcribes with Gemini.  
**Nodes involved:**  
- HTTP - Get YouTube Audio  
- HTTP - Download Audio File  
- Set Audio MimeType  
- Transcribe a recording

#### Node: **HTTP - Get YouTube Audio**
- **Type / Role:** `HTTP Request` — calls RapidAPI endpoint to obtain a downloadable audio URL.
- **Key configuration:**
  - URL: `https://youtube-video-fast-downloader-24-7.p.rapidapi.com/download_audio/{{videoId}}?quality=140`
  - Headers include:
    - `x-rapidapi-host`
    - `x-rapidapi-key` = **PASTE_YOUR_RAPIDAPI_KEY_HERE** (must be replaced)
    - `Content-Type: application/json`
- **Input/Output:**
  - **Input:** `videoId`
  - **Output:** expected to include a field like `file` (used next)
- **Edge cases / failures:**
  - Missing/invalid RapidAPI key → 401/403.
  - Quota exceeded / rate limited.
  - Provider response shape changes (e.g., `file` key renamed) breaks downstream.

#### Node: **HTTP - Download Audio File**
- **Type / Role:** `HTTP Request` — downloads the audio file as **binary**.
- **Key configuration:**
  - URL: `={{ $json.file }}`
  - Response format: **file** (binary)
- **Input/Output:**
  - **Input:** RapidAPI response containing `file` URL
  - **Output:** item with `binary.data` containing audio
- **Edge cases / failures:**
  - `file` missing/null → expression error or invalid URL.
  - Large files may cause timeout/memory issues depending on n8n limits.
  - Remote host may require redirects; if not handled, download may fail.

#### Node: **Set Audio MimeType**
- **Type / Role:** `Code` — forces correct binary metadata for Gemini audio ingestion.
- **Key configuration:**
  - For each incoming item with `item.binary.data`:
    - `mimeType = 'audio/mp4'`
    - `fileExtension = 'm4a'`
    - `fileName = 'audio_downloaded.m4a'`
- **Input/Output:**
  - **Input:** binary audio file
  - **Output:** same binary, corrected metadata
- **Edge cases / failures:**
  - If binary property name isn’t `data` (depends on HTTP node settings), code won’t update it.
  - If RapidAPI serves a different format, forcing `audio/mp4` may reduce transcription quality or fail.

#### Node: **Transcribe a recording**
- **Type / Role:** `Google Gemini (LangChain)` — transcribes audio to text.
- **Key configuration:**
  - Resource: **audio**
  - Input type: **binary**
  - Model: `models/gemini-2.5-flash`
  - Retries enabled; waits 5000 ms between tries
- **Credentials / Requirements:**
  - Requires Google Gemini API credentials configured in n8n.
- **Input/Output:**
  - **Input:** binary audio
  - **Output:** typically structured response with transcript at `content.parts[0].text`
- **Edge cases / failures:**
  - Audio length limits / payload size limits (provider-dependent).
  - Empty transcript for silent/unsupported audio.
  - API errors due to billing/quota.

---

### Block 4 — Transcript Guard + Data Merge
**Overview:** Validates transcript presence and merges transcript with previously prepared metadata into one combined item for the AI agent.  
**Nodes involved:**  
- IF - Skip if No Transcript  
- Merge - Attach Metadata + Transcript

#### Node: **IF - Skip if No Transcript**
- **Type / Role:** `IF` — ensures transcription text exists before continuing.
- **Key configuration:**
  - Condition: `{{$json.content?.parts?.[0]?.text}}` **is not empty**
- **Input/Output:**
  - **Input:** transcription output
  - **Output (true path used here):** goes to Merge input index 1
- **Edge cases / failures:**
  - If Gemini response structure differs, condition may incorrectly evaluate false.
  - **Current workflow wiring:** the *false* branch is not connected, so items without transcript effectively stop and no Notion page is created (silent “drop” behavior).

#### Node: **Merge - Attach Metadata + Transcript**
- **Type / Role:** `Merge` — combines two streams: metadata (index 0) + transcript (index 1).
- **Key configuration:**
  - Mode: **combine**
  - Combine by: **position**
  - `includeUnpaired: true`
- **Input/Output:**
  - **Input 0:** from *Set - Prepare Video Metadata*
  - **Input 1:** from *IF - Skip if No Transcript* (true)
  - **Output:** combined object used by AI Agent
- **Edge cases / failures:**
  - If either branch produces multiple items, “position” pairing can misalign.
  - `includeUnpaired` may allow metadata-only items through if transcript branch is missing; the AI prompt then uses fallback `'No transcript available'`, potentially producing low-quality output.

---

### Block 5 — Active Learning AI Generation (Agent + Model + Parser + Memory)
**Overview:** Uses a LangChain Agent with a Gemini chat model to generate a **strict JSON structure** for active learning outputs.  
**Nodes involved:**  
- Google Gemini Chat Model  
- Memory - Conversation Buffer  
- Output Parser - Structured JSON  
- AI Agent - Active Learning

#### Node: **Google Gemini Chat Model**
- **Type / Role:** `LM Chat (Google Gemini)` — provides the language model backing the agent.
- **Key configuration:** default options (no explicit model ID shown here; model selection may be set in credentials or node defaults depending on n8n version).
- **Input/Output connections:**
  - Connected to Agent via **ai_languageModel** channel.
- **Edge cases / failures:**
  - If a default model is not configured, execution may fail.
  - Model may refuse/limit very large prompts (long transcripts).

#### Node: **Memory - Conversation Buffer**
- **Type / Role:** `Memory Buffer Window` — retains short conversation context.
- **Key configuration:**
  - `sessionKey = active-learning-session`
  - Session type: custom key
- **Input/Output connections:**
  - Connected to Agent via **ai_memory** channel.
- **Edge cases / failures:**
  - With a fixed session key, multiple users/runs may share memory unintentionally (cross-run contamination). Consider using a per-run session ID.

#### Node: **Output Parser - Structured JSON**
- **Type / Role:** `Structured Output Parser` — constrains output to a schema-like JSON example.
- **Key configuration:**
  - Provides example schema with keys:
    - `micro_goals` (array of strings)
    - `cornell_notes` (array of objects: `trigger`, `concept`)
    - `feynman_explanation` (string)
    - `simulation_task` (string)
    - `key_topics` (array of strings)
- **Input/Output connections:**
  - Connected to Agent via **ai_outputParser**
- **Edge cases / failures:**
  - If model outputs non-JSON or violates schema, parser/agent may error or produce partial output.

#### Node: **AI Agent - Active Learning**
- **Type / Role:** `LangChain Agent` — orchestrates prompt + model + parser to produce structured learning worksheet content.
- **Key configuration:**
  - Prompt includes:
    - `Title: {{ $json.title }}`
    - `Channel: {{ $('Set - Prepare Video Metadata').item.json.channelTitle }}`
    - `Transcript: {{ $json.content?.parts?.[0]?.text || 'No transcript available' }}`
  - System message: “Elite Productivity Strategist & Coach…” (action-oriented, non-generic)
  - Output parser enabled
- **Input/Output:**
  - **Input:** merged metadata + transcript
  - **Output:** parsed JSON (often available under `$json.output` or directly in `$json`)
  - Connected to: **Build Page Structure**
- **Edge cases / failures:**
  - Expression dependency on `Set - Prepare Video Metadata` using `$('...').item` can fail if item pairing differs; safer to use merged fields directly when possible.
  - Very long transcript can exceed model context; consider chunking/summarizing transcript first if needed.

---

### Block 6 — Notion Page Construction & Storage
**Overview:** Converts AI structured JSON into Notion page properties and rich blocks, then creates the page via Notion API.  
**Nodes involved:**  
- Build Page Structure  
- Create Notion Page

#### Node: **Build Page Structure**
- **Type / Role:** `Code` — builds a Notion `/v1/pages` payload (database parent, properties, children blocks).
- **Key configuration choices:**
  - Reads AI output from: `const rawAi = $input.item.json.output || $input.item.json;`
  - Parses JSON if string.
  - Pulls fallback metadata from:
    - `$('YouTube - Fetch Video Details').first()`
    - `$('Set - Prepare Video Metadata').first()`
  - **Database ID placeholder:** `const database_id = "PASTE_YOUR_NOTION_DATABASE_ID_HERE";` (must be replaced)
  - Sets Notion properties:
    - **Name** (title)
    - **URL** (url)
    - **Channel** (select)
    - **Topics** (multi_select)
    - **Status** (status = “🟢 Processed”)
    - **Summary** (rich_text = feynman explanation)
    - **Published Date** (date)
    - **Thumbnail** (files external url)
  - Creates rich content blocks:
    - Callout “Active Learning Mode…”
    - Heading + To-do list for micro-goals
    - Cornell notes as toggles with child paragraphs
    - Feynman quote + reflection prompt
    - Simulation callout + to-do confirmation
- **Input/Output:**
  - **Input:** AI agent output item
  - **Output:** JSON body ready for Notion API
- **Edge cases / failures:**
  - If Notion database doesn’t have matching property names/types (Name/URL/Channel/Topics/Status/Summary/Published Date/Thumbnail), Notion API returns validation errors.
  - `channelName.replace(/,/g, '')` prevents commas; if select option doesn’t exist, Notion usually creates it depending on permissions/settings, but can fail in some workspaces.
  - Thumbnail URL may be undefined; Notion external file block may error if URL missing.

#### Node: **Create Notion Page**
- **Type / Role:** `HTTP Request` — creates a Notion page in a database.
- **Key configuration:**
  - POST `https://api.notion.com/v1/pages`
  - Authentication: **Notion API credential** (`notionApi`)
  - Header: `Notion-Version: 2022-06-28`
  - Body: `={{ $json }}` (expects payload from previous code node)
- **Input/Output:**
  - **Input:** Notion page payload
  - **Output:** Notion API response (created page object)
- **Edge cases / failures:**
  - Notion credential not connected → 401.
  - Integration not connected to the database → 403 “insufficient permissions”.
  - Property schema mismatch → 400 validation error.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | n8n-nodes-base.formTrigger | Collect YouTube URL via form and start workflow | — | SET - Get YT ID | ## 🚀 Overview & Setup; **Turn YouTube videos into Active Learning study sheets.**; Prereqs: Notion Template link + API keys (RapidAPI, Gemini, YouTube Data API); Before running: paste RapidAPI key + Database ID. |
| SET - Get YT ID | n8n-nodes-base.set | Extract video ID from URL | On form submission | YouTube - Fetch Video Details | ## 📥 Step 1: Get Video Data; Separates Video ID and fetches Title/Channel/Thumbnail. |
| YouTube - Fetch Video Details | n8n-nodes-base.youTube | Fetch title/channel/thumb/publish date from YouTube API | SET - Get YT ID | Set - Prepare Video Metadata | ## 📥 Step 1: Get Video Data; Separates Video ID and fetches Title/Channel/Thumbnail. |
| Set - Prepare Video Metadata | n8n-nodes-base.set | Normalize YouTube metadata for downstream use | YouTube - Fetch Video Details | Merge - Attach Metadata + Transcript; HTTP - Get YouTube Audio | ## 📥 Step 1: Get Video Data; Separates Video ID and fetches Title/Channel/Thumbnail. |
| HTTP - Get YouTube Audio | n8n-nodes-base.httpRequest | Call RapidAPI to obtain downloadable audio link | Set - Prepare Video Metadata | HTTP - Download Audio File | ## 🎧 Step 2: Listen & Note-take; Download audio via RapidAPI; transcribe with Gemini Flash. |
| HTTP - Download Audio File | n8n-nodes-base.httpRequest | Download audio as binary file | HTTP - Get YouTube Audio | Set Audio MimeType | ## 🎧 Step 2: Listen & Note-take; Download audio via RapidAPI; transcribe with Gemini Flash. |
| Set Audio MimeType | n8n-nodes-base.code | Force correct MIME metadata for Gemini audio input | HTTP - Download Audio File | Transcribe a recording | ## 🎧 Step 2: Listen & Note-take; Download audio via RapidAPI; transcribe with Gemini Flash. |
| Transcribe a recording | @n8n/n8n-nodes-langchain.googleGemini | Transcribe audio binary to text | Set Audio MimeType | IF - Skip if No Transcript | ## 🎧 Step 2: Listen & Note-take; Download audio via RapidAPI; transcribe with Gemini Flash. |
| IF - Skip if No Transcript | n8n-nodes-base.if | Stop/guard if transcript is empty | Transcribe a recording | Merge - Attach Metadata + Transcript (index 1) | ## 🎧 Step 2: Listen & Note-take; (Implicitly) prevents continuing without transcript. |
| Merge - Attach Metadata + Transcript | n8n-nodes-base.merge | Combine metadata + transcript into one item | Set - Prepare Video Metadata (index 0); IF - Skip if No Transcript (index 1) | AI Agent - Active Learning | ## 🤖 Step 3: "Active Learning" Analysis; AI creates micro-goals, Cornell notes, Feynman test, simulation task. |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM backend for the agent | — (model provider connection) | AI Agent - Active Learning (ai_languageModel) | ## 🤖 Step 3: "Active Learning" Analysis; AI creates micro-goals, Cornell notes, Feynman test, simulation task. |
| Memory - Conversation Buffer | @n8n/n8n-nodes-langchain.memoryBufferWindow | Provides session memory to agent | — | AI Agent - Active Learning (ai_memory) | ## 🤖 Step 3: "Active Learning" Analysis; AI creates micro-goals, Cornell notes, Feynman test, simulation task. |
| Output Parser - Structured JSON | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured JSON output fields | — | AI Agent - Active Learning (ai_outputParser) | ## 🤖 Step 3: "Active Learning" Analysis; AI creates micro-goals, Cornell notes, Feynman test, simulation task. |
| AI Agent - Active Learning | @n8n/n8n-nodes-langchain.agent | Generate ICAP-based Active Learning worksheet JSON | Merge - Attach Metadata + Transcript | Build Page Structure | ## 🤖 Step 3: "Active Learning" Analysis; AI creates micro-goals, Cornell notes, Feynman test, simulation task. |
| Build Page Structure | n8n-nodes-base.code | Convert AI JSON + metadata into Notion page payload | AI Agent - Active Learning | Create Notion Page | ## 💾 Step 4: Build Notion Page; Uses custom JS to create Notion blocks and sends to Notion DB. |
| Create Notion Page | n8n-nodes-base.httpRequest | Create page in Notion database via API | Build Page Structure | — | ## 🛠️ Checklist Before Running (Must Check!); Template link; how to find Database ID; RapidAPI key link; Gemini key link; ensure Notion integration connected to DB. |
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Documentation note | — | — | (This node is itself a note) ## 🚀 Overview & Setup with template/API prerequisites and setup steps. |
| Sticky Note - Fetch | n8n-nodes-base.stickyNote | Documentation note | — | — | (This node is itself a note) ## 📥 Step 1: Get Video Data explanation. |
| Sticky Note - Audio | n8n-nodes-base.stickyNote | Documentation note | — | — | (This node is itself a note) ## 🎧 Step 2: Listen & Note-take explanation. |
| Sticky Note - AI | n8n-nodes-base.stickyNote | Documentation note | — | — | (This node is itself a note) ## 🤖 Step 3: "Active Learning" Analysis explanation. |
| Sticky Note - Storage | n8n-nodes-base.stickyNote | Documentation note | — | — | (This node is itself a note) ## 💾 Step 4: Build Notion Page explanation. |
| Sticky Note - Security | n8n-nodes-base.stickyNote | Documentation note | — | — | (This node is itself a note) ## 🛠️ Checklist Before Running with links and connection instructions. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add trigger node:** `Form Trigger`
   - Title: *YouTube to Active Learning Notes*
   - Description: *Transform any YouTube video into structured Active Learning study sheets powered by AI*
   - Field:
     - Name: `URL` (required)
     - Label: “YouTube Video URL”
   - Button label: “Generate Learning Notes”
   - Disable attribution if desired.

3. **Add node:** `Set` → name it **SET - Get YT ID**
   - Enable “Include Other Fields”.
   - Add field `VIdeoID` (string) with expression:
     - `{{$json.URL.match(/(?:v=|youtu.be\/)([a-zA-Z0-9_-]{11})/)?.[1] || $json.URL }}`

4. **Add node:** `YouTube` → name it **YouTube - Fetch Video Details**
   - Resource: **Video**
   - Operation: **Get**
   - Video ID: `={{ $json.VIdeoID }}`
   - **Credentials:** configure YouTube Data API credentials in n8n and attach them to this node.

5. **Add node:** `Set` → name it **Set - Prepare Video Metadata**
   - Create fields:
     - `videoId` = `{{$json.id}}`
     - `title` = `{{$json.snippet.title}}`
     - `description` = `{{$json.snippet.description}}`
     - `publishedAt` = `{{$json.snippet.publishedAt}}`
     - `thumbnailUrl` = `{{$json.snippet.thumbnails.maxres?.url || $json.snippet.thumbnails.high?.url || $json.snippet.thumbnails.default.url}}`
     - `videoUrl` = `{{'https://www.youtube.com/watch?v=' + $json.id}}`
     - `channelTitle` = `{{$json.snippet.channelTitle}}`

6. **Add node:** `HTTP Request` → name it **HTTP - Get YouTube Audio**
   - Method: GET
   - URL expression:
     - `{{ 'https://youtube-video-fast-downloader-24-7.p.rapidapi.com/download_audio/' + $json.videoId + '?quality=140' }}`
   - Send headers = true; add:
     - `x-rapidapi-host`: `youtube-video-fast-downloader-24-7.p.rapidapi.com`
     - `x-rapidapi-key`: *(your RapidAPI key)*
     - `Content-Type`: `application/json`

7. **Add node:** `HTTP Request` → name it **HTTP - Download Audio File**
   - URL: `={{ $json.file }}`
   - Response: set **Response Format = File** (binary)

8. **Add node:** `Code` → name it **Set Audio MimeType**
   - Paste logic that sets:
     - `item.binary.data.mimeType = 'audio/mp4'`
     - `fileExtension = 'm4a'`, `fileName = 'audio_downloaded.m4a'`
   - Ensure the binary property name matches your prior HTTP node (commonly `data`).

9. **Add node:** `Google Gemini` (LangChain) → name it **Transcribe a recording**
   - Resource: **Audio**
   - Input type: **Binary**
   - Model: `models/gemini-2.5-flash`
   - **Credentials:** configure Gemini API key credentials in n8n.
   - Optional: enable retry with wait.

10. **Add node:** `IF` → name it **IF - Skip if No Transcript**
   - Condition: transcript text **not empty**
   - Use expression: `{{$json.content?.parts?.[0]?.text}}`
   - Connect **true** output forward; (optional) connect **false** to a notification/log node.

11. **Add node:** `Merge` → name it **Merge - Attach Metadata + Transcript**
   - Mode: **Combine**
   - Combine by: **Position**
   - Include unpaired: enabled (match existing behavior)

12. **Add LangChain support nodes:**
   - **LM Chat (Google Gemini)** → name it **Google Gemini Chat Model**
   - **Memory Buffer Window** → name it **Memory - Conversation Buffer**
     - Session key: `active-learning-session` (consider dynamic session key if multi-user)
   - **Structured Output Parser** → name it **Output Parser - Structured JSON**
     - Provide a JSON schema example with keys: `micro_goals`, `cornell_notes`, `feynman_explanation`, `simulation_task`, `key_topics`.

13. **Add node:** `AI Agent` (LangChain) → name it **AI Agent - Active Learning**
   - Prompt type: “Define”
   - Paste the main prompt (Title/Channel/Transcript + ICAP instructions + required fields).
   - Add a system message (action-oriented).
   - Enable structured output parsing.
   - Wire connections:
     - `Google Gemini Chat Model` → Agent via **ai_languageModel**
     - `Memory - Conversation Buffer` → Agent via **ai_memory**
     - `Output Parser - Structured JSON` → Agent via **ai_outputParser**

14. **Add node:** `Code` → name it **Build Page Structure**
   - Implement:
     - Load AI result from `$json.output` (or `$json`)
     - Parse JSON if string
     - Set `database_id = "YOUR_NOTION_DATABASE_ID"`
     - Build `properties` with matching Notion database property names
     - Build `children` blocks (callouts, headings, todos, toggles, quote)
   - Ensure property names match your database exactly.

15. **Add node:** `HTTP Request` → name it **Create Notion Page**
   - Method: POST
   - URL: `https://api.notion.com/v1/pages`
   - Authentication: **Notion API credential** (predefined credential type)
   - Header: `Notion-Version: 2022-06-28`
   - Body: JSON = `={{ $json }}`

16. **Connect nodes in this order:**
   - Form Trigger → SET - Get YT ID → YouTube - Fetch Video Details → Set - Prepare Video Metadata
   - Set - Prepare Video Metadata → Merge (input 0)
   - Set - Prepare Video Metadata → HTTP - Get YouTube Audio → HTTP - Download Audio File → Set Audio MimeType → Transcribe a recording → IF - Skip if No Transcript (true) → Merge (input 1)
   - Merge → AI Agent - Active Learning → Build Page Structure → Create Notion Page

17. **Credentials checklist:**
   - YouTube node: YouTube Data API credentials present and quota available
   - Gemini nodes: Gemini API key present (billing/quota OK)
   - Notion node: Notion integration connected and has access to the target database
   - RapidAPI key inserted in header

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Notion template to duplicate before running | https://www.notion.so/YouTube-to-Notion-Active-Learning-Assistant-Beginner-2db1fc5a596381b1b5dec683689d3d1d?source=copy_link |
| RapidAPI “youtube-video-fast-downloader-24-7” key + quota | https://rapidapi.com/ytdlfree/api/youtube-video-fast-downloader-24-7 |
| Gemini API key creation | https://aistudio.google.com/app/apikey |
| Required manual edits before first run | Replace `PASTE_YOUR_RAPIDAPI_KEY_HERE` in **HTTP - Get YouTube Audio** and `PASTE_YOUR_NOTION_DATABASE_ID_HERE` in **Build Page Structure** |
| Notion integration must be connected to the database | In Notion DB page: `...` → Connections → Connect to your n8n integration; then ensure n8n credential shows “Connected” |

