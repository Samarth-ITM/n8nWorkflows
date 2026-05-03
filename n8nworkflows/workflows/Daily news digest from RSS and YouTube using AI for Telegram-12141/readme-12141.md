Daily news digest from RSS and YouTube using AI for Telegram

https://n8nworkflows.xyz/workflows/daily-news-digest-from-rss-and-youtube-using-ai-for-telegram-12141


# Daily news digest from RSS and YouTube using AI for Telegram

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow runs every morning and creates a **daily news digest** by aggregating **RSS articles** (TechCrunch, The Verge, BBC World) and **recent YouTube videos** about AI/tech. It then uses **Google Gemini** to curate and write a Telegram-ready briefing (HTML) plus an **audio script**, generates an **MP3** via **OpenAI TTS**, and sends both **text + audio** to **Telegram**.

**Target use cases:**
- Personal “morning briefing” automation to Telegram
- Internal team broadcast (tech/global news update)
- Rapid AI-curated digest from heterogeneous sources (RSS + YouTube)

### Logical blocks
1. **1.1 Scheduling & Input Collection**: Scheduled run triggers RSS reads + YouTube search.
2. **1.2 YouTube Normalization & Filtering**: Filters YouTube results by keywords and recency; normalizes fields.
3. **1.3 RSS Aggregation & Source Marking**: Merges RSS feeds and tags items as `sourceType=rss` (YouTube as `sourceType=youtube`).
4. **1.4 Cleaning, Deduplication & Context Building**: Cleans HTML, removes duplicates, filters “deal” content, enforces quotas/mixing ratio, builds `newsContext`.
5. **1.5 AI Editorial (Gemini)**: Produces strict JSON containing `title`, `telegram_body` (Telegram HTML), and `audio_script`.
6. **1.6 Output Delivery (Telegram Text + OpenAI TTS + Telegram Audio)**: Sends text to Telegram; generates MP3 from script; fixes binary metadata; sends audio to Telegram.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & Input Collection

**Overview:** Runs daily and fetches content from RSS feeds and YouTube in parallel.

**Nodes involved:**
- Schedule Trigger
- RSS Feed Read (TechCrunch)
- RSS Feed Read (The Verge)
- RSS Feed Read (BBC World)
- YouTube - Search (Latest 24h)

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` — workflow entry point.
- **Config choices:** Triggers at **07:00** (hour-based schedule). Workflow timezone in settings is **Asia/Ho_Chi_Minh**.
- **Connections:** Outputs to all three RSS nodes and the YouTube search node (parallel fan-out).
- **Edge cases:**
  - Timezone mismatch: node runs at 07:00 in the workflow timezone; the digest title and YouTube time filter use **America/New_York** in expressions, so “day boundaries” can differ.

#### Node: RSS Feed Read (TechCrunch)
- **Type / role:** `RSS Feed Read` — fetch TechCrunch RSS items.
- **Config choices:** URL `https://techcrunch.com/feed/`; `ignoreSSL=true`; **onError=continueRegularOutput** (does not fail whole execution on feed issues); `retryOnFail=false`.
- **Connections:** To **Merge - Tech (Append)** (input 0).
- **Failure modes / edge cases:**
  - Feed downtime / rate limits / malformed XML → node continues (may output empty items), potentially reducing coverage.

#### Node: RSS Feed Read (The Verge)
- **Type / role:** `RSS Feed Read` — fetch The Verge RSS items.
- **Config choices:** URL `https://www.theverge.com/rss/index.xml`; `ignoreSSL=true`; onError continues.
- **Connections:** To **Merge - Tech (Append)** (input 1).
- **Edge cases:** Similar to above; Verge feed sometimes includes boilerplate phrases later stripped downstream.

#### Node: RSS Feed Read (BBC World)
- **Type / role:** `RSS Feed Read` — fetch BBC World RSS items.
- **Config choices:** URL `http://feeds.bbci.co.uk/news/world/rss.xml`; `ignoreSSL=true`; onError continues.
- **Connections:** To **Merge - Add BBC World (Append)** (input 1).
- **Edge cases:** HTTP URL (not HTTPS) could be blocked in some environments; still typically works.

#### Node: YouTube - Search (Latest 24h)
- **Type / role:** `YouTube` — searches recent videos.
- **Config choices:**
  - Resource: `video`
  - Limit: `15`
  - Query: `"AI, Tech News, Artificial Intelligence"`
  - Region: `US`
  - Published after: `{{ $now.setZone('America/New_York').minus({ hours: 24 }).toISO() }}`
  - Order: `date`
- **Credentials:** YouTube OAuth2 (YouTube Data API).
- **Connections:** To **Code - YouTube Normalize + Filter**.
- **Failure modes / edge cases:**
  - OAuth permission/scopes or quota exceeded → node fails and reduces content.
  - If YouTube returns unexpected shape, downstream normalization may drop all items.

**Sticky note context (applies to this block):**
- “## 1. Input Sources … Replace these RSS URLs and YouTube keywords…”

---

### 2.2 YouTube Normalization & Filtering

**Overview:** Converts YouTube search results into a consistent “article-like” schema and filters out low-signal or unwanted topics.

**Nodes involved:**
- Code - YouTube Normalize + Filter
- Set - Mark YouTube

#### Node: Code - YouTube Normalize + Filter
- **Type / role:** `Code` — JS transformation + filtering.
- **Key configuration choices:**
  - Time window: `ONLY_LAST_HOURS = 24` (server time)
  - Inclusion keywords (must match at least one): ai, artificial intelligence, tech, llm, chatgpt, gemini, openai, automation, robotics, machine learning
  - Exclusion keywords: crypto/bitcoin/coin, investment, price prediction, xrp, giveaway, live stream
  - Output schema per item:
    - `title`
    - `link` (constructed `https://www.youtube.com/watch?v=${videoId}`)
    - `description`
    - `source` (`YouTube - <channel>`)
    - `publishedAt`
    - `thumbnail`
    - `channelTitle`, `videoId`
- **Connections:** Input from YouTube search; output to **Set - Mark YouTube**.
- **Edge cases / failures:**
  - Missing `id.videoId` or `snippet.title` → item dropped.
  - `publishedAt` parsing failures won’t crash (it only filters if parse succeeds).
  - Very strict filters can yield **0 videos**, which is acceptable but reduces the mix.

#### Node: Set - Mark YouTube
- **Type / role:** `Set` — adds a classification field.
- **Config choices:** Adds `sourceType = "youtube"`; `includeOtherFields=true` so normalized fields remain.
- **Connections:** To **Merge - Add YouTube (Append)** on **index 1** (second input).
- **Edge cases:** None major; if upstream output empty, merge receives nothing on that branch.

---

### 2.3 RSS Aggregation & Source Marking

**Overview:** Merges RSS feeds together and tags all RSS items with `sourceType=rss` so the later cleaner can mix RSS and YouTube with quotas.

**Nodes involved:**
- Merge - Tech (Append)
- Merge - Add BBC World (Append)
- Set - Mark RSS

#### Node: Merge - Tech (Append)
- **Type / role:** `Merge` (Append) — combines TechCrunch + The Verge item streams.
- **Config choices:** Default merge behavior (Append) in n8n Merge v3.x means it concatenates both inputs.
- **Connections:** Inputs: TechCrunch (index 0), The Verge (index 1). Output to **Merge - Add BBC World (Append)** (index 0).
- **Edge cases:** If one feed errors and returns nothing, append still works with the other input.

#### Node: Merge - Add BBC World (Append)
- **Type / role:** `Merge` (Append) — appends BBC World to the already-merged tech RSS.
- **Connections:** Inputs: Merge-Tech output (index 0), BBC RSS (index 1). Output to **Set - Mark RSS**.
- **Edge cases:** Same as above.

#### Node: Set - Mark RSS
- **Type / role:** `Set` — adds `sourceType=rss`.
- **Config choices:** Adds `sourceType = "rss"`; `includeOtherFields=true`.
- **Connections:** To **Merge - Add YouTube (Append)** on **index 0** (first input).

---

### 2.4 Cleaning, Deduplication & Context Building

**Overview:** Produces a curated “context” string (`newsContext`) for Gemini by cleaning HTML, filtering old/junk items, deduplicating, enforcing a RSS/YouTube mix ratio, and limiting total size.

**Nodes involved:**
- Merge - Add YouTube (Append)
- Code - Clean & Dedup (newsContext)

#### Node: Merge - Add YouTube (Append)
- **Type / role:** `Merge` (Append) — combines RSS-marked items and YouTube-marked items into one list.
- **Connections:** Input 0 from Set-Mark-RSS; input 1 from Set-Mark-YouTube. Output to cleaning code.
- **Edge cases:** If YouTube branch is empty, output is only RSS (and vice versa).

#### Node: Code - Clean & Dedup (newsContext)
- **Type / role:** `Code` — main preparation logic; outputs one item with `newsContext`, `articles[]`, and `stats`.
- **Key configuration choices:**
  - **Recency filter:** `ONLY_LAST_HOURS = 36` (drops items older than 36h when date parse succeeds)
  - **Limits:** `MAX_ITEMS=60`, `MAX_TOTAL_CHARS=28000`, title/desc clipping
  - **Mix ratio:** RSS 70% / YouTube 30% (quota computed; fallback if one side lacks items)
  - **Exclusions:** regex-based filter for deals/prices/“where to buy”, etc.
  - **Cleaning:** HTML stripping + entity decoding; Verge boilerplate stripping.
  - **Dedup key:** prefers link-based (`L:<link>`), otherwise normalized title (`T:<title>`).
  - **Sort:** Each group sorted newest-first (by `publishedAt` if parseable).
  - **Output format in `newsContext`:**
    - One line per item:
      `[Publication | Author | sourceType] Title: Description (Link)`
- **Inputs/outputs:**
  - Input: appended RSS+YT items with at least `title/link/description/...` fields
  - Output: single item:
    - `newsContext` (string)
    - `articles` (array of selected items)
    - `stats` (counts + drops + quota info)
- **Failure modes / edge cases:**
  - RSS items have differing date fields; code tries multiple (`isoDate`, `pubDate`, etc.). If unparseable, item may bypass time filtering and sort as timestamp 0.
  - Extremely large descriptions are clipped; overall context stops once `MAX_TOTAL_CHARS` is exceeded (may yield fewer than `MAX_ITEMS`).
  - If upstream feeds provide unexpected field names, some metadata (author/publication) may be empty but still works.

**Sticky note context (applies to this block):**
- “## 2. Data Preparation … Cleans HTML tags, deduplicates … mixes sources…”

---

### 2.5 AI Editorial (Gemini) + Robust JSON Parsing

**Overview:** Gemini is prompted to select top stories and return strict JSON. A parser node extracts and sanitizes the JSON, ensuring Telegram HTML compatibility.

**Nodes involved:**
- Google Gemini Chat (AI Analysis)
- Code - Robust Parser (Gemini JSON)

#### Node: Google Gemini Chat (AI Analysis)
- **Type / role:** `Google Gemini` (LangChain node) — generates the briefing content.
- **Config choices:**
  - Model: `models/gemini-2.5-flash`
  - Temperature: `0.3`
  - System message: “You are a professional News Editor. You must follow instructions exactly.”
  - User content includes:
    - The full `{{$json.newsContext}}`
    - Requirements to output ONLY JSON (no markdown fences)
    - JSON schema:
      - `title`: date-based, using `$now.setZone('America/New_York').toFormat('yyyy-LL-dd')`
      - `telegram_body`: Telegram HTML only (`<b>`, `<i>`, `<a href='url'>`), under 4000 chars, bullet `•`
      - `audio_script`: speech-friendly, no URLs, avoid special markup chars
- **Node settings:** `jsonOutput=true` and `simplify=false` (still, output is parsed again downstream).
- **Credentials:** Google PaLM / Gemini API credential.
- **Connections:** To **Code - Robust Parser (Gemini JSON)**.
- **Failure modes / edge cases:**
  - Model may still return extra text or invalid JSON → downstream parser throws.
  - Char limit: Telegram body asked to stay <4000 chars, but not guaranteed unless Gemini complies.
  - If `newsContext` is empty/too short, output quality degrades.

#### Node: Code - Robust Parser (Gemini JSON)
- **Type / role:** `Code` — extracts text from Gemini response, slices first `{` to last `}`, parses JSON, and sanitizes Telegram HTML.
- **Key behavior:**
  - Extracts raw text from: `candidates[0].content.parts[].text`
  - Defensive JSON extraction: substring between first `{` and last `}`
  - Parses JSON; throws explicit errors if:
    - No raw text
    - No JSON braces found
    - JSON.parse fails (includes JSON string in error)
  - Telegram HTML cleanup:
    - Converts `<br>` to `\n`
    - Removes `<p>` and converts `</p>` to newline
    - Normalizes `href='...'` to `href="..."`
  - Validates non-empty `telegram_body` and `audio_script`
- **Outputs:** `{ title, telegram_body, audio_script }`
- **Connections:** Two parallel outputs:
  - To **Code - Build TTS Payload (OpenAI)**
  - To **Telegram - Send Briefing Text**
- **Edge cases:**
  - If Gemini returns JSON but with different keys, validation fails.
  - Telegram HTML still may be invalid if Gemini outputs unsupported tags/entities.

**Sticky note context (applies to this block):**
- “## 3. AI Analysis … Edit the Prompt in the Gemini node…”

---

### 2.6 Text Delivery + Audio Generation (OpenAI TTS) + Audio Delivery

**Overview:** Sends the Telegram text briefing, generates an MP3 from the audio script, fixes binary metadata for Telegram compatibility, and sends the audio.

**Nodes involved:**
- Telegram - Send Briefing Text
- Code - Build TTS Payload (OpenAI)
- HTTP - OpenAI TTS (speech)
- Code - Fix Audio Meta (filename + mime)
- Telegram - Send Briefing Audio

#### Node: Telegram - Send Briefing Text
- **Type / role:** `Telegram` — sends message text.
- **Config choices:**
  - `text = {{$json.telegram_body}}`
  - `parse_mode = HTML`
  - `chatId = "INPUT YOUR CHAT ID WITH BOT"` (placeholder)
- **Credentials:** Telegram API credential (bot token).
- **Connections:** Terminal.
- **Failure modes / edge cases:**
  - Wrong chat ID / bot not allowed in chat → send fails.
  - Telegram rejects invalid HTML or >4096 chars (Telegram message limit); prompt says 4000, but no hard enforcement here.

#### Node: Code - Build TTS Payload (OpenAI)
- **Type / role:** `Code` — prepares OpenAI TTS request and metadata.
- **Key configuration choices:**
  - Trims `audio_script`, throws if empty.
  - Truncates to **<=4096 chars** (OpenAI TTS input constraint), truncates to 4050 + `...`.
  - Builds filename: `morning_briefing_YYYY-MM-DD.mp3` using server local date.
  - TTS payload:
    - `model: gpt-4o-mini-tts`
    - `voice: alloy`
    - `instructions: Clear, professional English broadcast voice.`
    - `response_format: mp3`
    - `speed: 1.05`
- **Connections:** To **HTTP - OpenAI TTS (speech)**.
- **Edge cases:**
  - Date/time inconsistency vs title (title uses America/New_York; filename uses server time).
  - If OpenAI changes model availability, request fails.

#### Node: HTTP - OpenAI TTS (speech)
- **Type / role:** `HTTP Request` — calls OpenAI Audio Speech endpoint and stores MP3 as binary.
- **Config choices:**
  - POST `https://api.openai.com/v1/audio/speech`
  - Body: `={{ $json.tts }}`
  - Auth: predefined credential type `openAiApi`
  - Response handling: `responseFormat=file`, stored under binary property **`audio`**
- **Connections:** To **Code - Fix Audio Meta (filename + mime)**.
- **Failure modes / edge cases:**
  - Auth errors (invalid API key), model errors, 429 rate limits.
  - If OpenAI returns JSON error, node may not produce binary `audio` → next node throws.

#### Node: Code - Fix Audio Meta (filename + mime)
- **Type / role:** `Code` — ensures Telegram receives a properly named MP3 binary.
- **Key behavior:**
  - Requires `item.binary.audio` else throws.
  - Sets `binary.audio.fileName` from `item.json.fileName` (fallback `morning_briefing.mp3`)
  - Forces `mimeType = audio/mpeg` and `fileExtension = mp3`
- **Connections:** To **Telegram - Send Briefing Audio**.
- **Edge cases:** If binary property name differs (not `audio`), it fails.

#### Node: Telegram - Send Briefing Audio
- **Type / role:** `Telegram` — sends audio file.
- **Config choices:**
  - Operation: `sendAudio`
  - `chatId = "INPUT YOUR CHAT ID WITH BOT"` (placeholder)
  - `binaryData=true`, `binaryPropertyName="audio"`
  - Caption: `={{ $json.title }}`
- **Credentials:** Telegram API credential.
- **Failure modes / edge cases:**
  - Wrong chat ID, bot permissions.
  - Telegram file size constraints (rare here, but possible with long TTS).

**Sticky note context (applies to this block):**
- “## 4. Audio Generation & Delivery …”
- “### ⚠️ Check Chat ID Ensure `TELEGRAM_CHAT_ID` is set …” (note: workflow currently hardcodes placeholder strings rather than reading an env var).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Daily trigger entry point | — | RSS Feed Read (TechCrunch); RSS Feed Read (The Verge); RSS Feed Read (BBC World); YouTube - Search (Latest 24h) | ## Daily News Digest: Text & Audio Briefing… (full workflow description + setup + customization) |
| RSS Feed Read (TechCrunch) | rssFeedRead | Fetch RSS items | Schedule Trigger | Merge - Tech (Append) | ## 1. Input Sources… Replace RSS URLs and YouTube keywords… |
| RSS Feed Read (The Verge) | rssFeedRead | Fetch RSS items | Schedule Trigger | Merge - Tech (Append) | ## 1. Input Sources… Replace RSS URLs and YouTube keywords… |
| RSS Feed Read (BBC World) | rssFeedRead | Fetch RSS items | Schedule Trigger | Merge - Add BBC World (Append) | ## 1. Input Sources… Replace RSS URLs and YouTube keywords… |
| YouTube - Search (Latest 24h) | youTube | Search recent videos | Schedule Trigger | Code - YouTube Normalize + Filter | ## 1. Input Sources… Replace RSS URLs and YouTube keywords… |
| Merge - Tech (Append) | merge | Append TechCrunch + Verge | RSS Feed Read (TechCrunch); RSS Feed Read (The Verge) | Merge - Add BBC World (Append) | ## 2. Data Preparation… Cleans, dedups, mixes… |
| Merge - Add BBC World (Append) | merge | Append BBC to RSS list | Merge - Tech (Append); RSS Feed Read (BBC World) | Set - Mark RSS | ## 2. Data Preparation… Cleans, dedups, mixes… |
| Set - Mark RSS | set | Add `sourceType=rss` | Merge - Add BBC World (Append) | Merge - Add YouTube (Append) | ## 2. Data Preparation… Cleans, dedups, mixes… |
| Code - YouTube Normalize + Filter | code | Filter + normalize YouTube items | YouTube - Search (Latest 24h) | Set - Mark YouTube | ## 2. Data Preparation… Cleans, dedups, mixes… |
| Set - Mark YouTube | set | Add `sourceType=youtube` | Code - YouTube Normalize + Filter | Merge - Add YouTube (Append) | ## 2. Data Preparation… Cleans, dedups, mixes… |
| Merge - Add YouTube (Append) | merge | Combine RSS + YouTube | Set - Mark RSS; Set - Mark YouTube | Code - Clean & Dedup (newsContext) | ## 2. Data Preparation… Cleans, dedups, mixes… |
| Code - Clean & Dedup (newsContext) | code | Clean HTML, dedup, mix ratio, build context | Merge - Add YouTube (Append) | Google Gemini Chat (AI Analysis) | ## 2. Data Preparation… Cleans, dedups, mixes… |
| Google Gemini Chat (AI Analysis) | googleGemini | Generate briefing JSON | Code - Clean & Dedup (newsContext) | Code - Robust Parser (Gemini JSON) | ## 3. AI Analysis… Edit prompt, item count, max chars… |
| Code - Robust Parser (Gemini JSON) | code | Extract/parse JSON; sanitize Telegram HTML | Google Gemini Chat (AI Analysis) | Code - Build TTS Payload (OpenAI); Telegram - Send Briefing Text | ## 3. AI Analysis… Edit prompt, item count, max chars… |
| Telegram - Send Briefing Text | telegram | Send HTML text briefing | Code - Robust Parser (Gemini JSON) | — | ### ⚠️ Check Chat ID Ensure `TELEGRAM_CHAT_ID` is set… |
| Code - Build TTS Payload (OpenAI) | code | Build OpenAI TTS request + filename | Code - Robust Parser (Gemini JSON) | HTTP - OpenAI TTS (speech) | ## 4. Audio Generation & Delivery… |
| HTTP - OpenAI TTS (speech) | httpRequest | Call OpenAI speech endpoint, get MP3 binary | Code - Build TTS Payload (OpenAI) | Code - Fix Audio Meta (filename + mime) | ## 4. Audio Generation & Delivery… |
| Code - Fix Audio Meta (filename + mime) | code | Ensure MP3 metadata for Telegram | HTTP - OpenAI TTS (speech) | Telegram - Send Briefing Audio | ## 4. Audio Generation & Delivery… |
| Telegram - Send Briefing Audio | telegram | Send MP3 audio briefing | Code - Fix Audio Meta (filename + mime) | — | ### ⚠️ Check Chat ID Ensure `TELEGRAM_CHAT_ID` is set… |
| Sticky Note | stickyNote | Documentation block | — | — | ## Daily News Digest: Text & Audio Briefing… (full content) |
| Sticky Note1 | stickyNote | Documentation: inputs | — | — | ## 1. Input Sources… |
| Sticky Note2 | stickyNote | Documentation: preparation | — | — | ## 2. Data Preparation… |
| Sticky Note3 | stickyNote | Documentation: AI | — | — | ## 3. AI Analysis… |
| Sticky Note4 | stickyNote | Documentation: audio | — | — | ## 4. Audio Generation & Delivery… |
| Sticky Note5 | stickyNote | Documentation: chat id warning | — | — | ### ⚠️ Check Chat ID… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *Daily News Digest: RSS & Youtube to Telegram (Text + Audio) with Gemini & OpenAI*
   - Set workflow **Timezone** to `Asia/Ho_Chi_Minh` (or your preference).

2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Set to run daily at **07:00** (triggerAtHour = 7).

3. **Add RSS feed nodes (3x)**
   - Node: **RSS Feed Read**
   - Configure each:
     - TechCrunch: `https://techcrunch.com/feed/`
     - The Verge: `https://www.theverge.com/rss/index.xml`
     - BBC World: `http://feeds.bbci.co.uk/news/world/rss.xml`
   - In **Options**, set **Ignore SSL Issues** = true.
   - Set node error handling to **Continue on Fail** (equivalent to `onError=continueRegularOutput`).

4. **Merge TechCrunch + The Verge**
   - Node: **Merge**
   - Mode: **Append**
   - Connect:
     - TechCrunch RSS → Merge input 0
     - The Verge RSS → Merge input 1

5. **Append BBC World**
   - Node: **Merge**
   - Mode: **Append**
   - Connect:
     - Merge-Tech output → Merge-BBC input 0
     - BBC RSS → Merge-BBC input 1

6. **Mark RSS items**
   - Node: **Set**
   - Add field: `sourceType` (String) = `rss`
   - Enable **Include Other Fields**.
   - Connect Merge-BBC → Set-Mark-RSS.

7. **Add YouTube search**
   - Node: **YouTube**
   - Resource: **Video**
   - Limit: **15**
   - Filters:
     - `q`: `AI, Tech News, Artificial Intelligence`
     - `regionCode`: `US`
     - `publishedAfter`: `{{ $now.setZone('America/New_York').minus({ hours: 24 }).toISO() }}`
   - Options: `order = date`
   - **Credentials:** configure **YouTube OAuth2** for YouTube Data API.

8. **Normalize + filter YouTube**
   - Node: **Code**
   - Paste the “YouTube Normalize + Filter” JS logic (in the workflow).
   - Connect YouTube → this Code node.

9. **Mark YouTube items**
   - Node: **Set**
   - Add field: `sourceType` (String) = `youtube`
   - Enable **Include Other Fields**.
   - Connect YouTube code → Set-Mark-YouTube.

10. **Merge RSS + YouTube**
    - Node: **Merge**
    - Mode: **Append**
    - Connect:
      - Set-Mark-RSS → Merge input 0
      - Set-Mark-YouTube → Merge input 1

11. **Clean/dedup/build context**
    - Node: **Code**
    - Paste the “Clean & Dedup (newsContext)” JS logic.
    - Connect Merge(RSS+YT) → this Code node.

12. **Add Google Gemini Chat**
    - Node: **Google Gemini Chat** (`@n8n/n8n-nodes-langchain.googleGemini`)
    - Model: `models/gemini-2.5-flash`
    - Temperature: `0.3`
    - System message: “You are a professional News Editor. You must follow instructions exactly.”
    - User message: include the prompt that references `{{$json.newsContext}}` and demands strict JSON with keys `title`, `telegram_body`, `audio_script`.
    - Enable JSON output (node option).
    - **Credentials:** configure **Google Gemini/PaLM API** credentials.
    - Connect Clean/Dedup Code → Gemini.

13. **Parse Gemini output robustly**
    - Node: **Code**
    - Paste the “Robust Parser (Gemini JSON)” JS logic.
    - Connect Gemini → parser.

14. **Send Telegram text**
    - Node: **Telegram**
    - Operation: “Send Message”
    - `chatId`: set your actual chat ID (or map from an env variable you manage)
    - `text`: `{{ $json.telegram_body }}`
    - `parse_mode`: `HTML`
    - **Credentials:** configure Telegram bot credential.
    - Connect parser → Telegram text node.

15. **Build OpenAI TTS payload**
    - Node: **Code**
    - Paste the “Build TTS Payload (OpenAI)” JS logic.
    - Connect parser → TTS payload code.

16. **Call OpenAI TTS**
    - Node: **HTTP Request**
    - Method: POST
    - URL: `https://api.openai.com/v1/audio/speech`
    - Authentication: **OpenAI API credential**
    - Send body: JSON, with body `{{ $json.tts }}`
    - Response: **File**, output binary property: `audio`
    - Connect TTS payload code → HTTP node.

17. **Fix binary metadata**
    - Node: **Code**
    - Paste the “Fix Audio Meta” JS logic.
    - Connect HTTP TTS → Fix Audio Meta.

18. **Send Telegram audio**
    - Node: **Telegram**
    - Operation: `sendAudio`
    - `chatId`: your chat ID
    - Binary data: true
    - Binary property: `audio`
    - Caption: `{{ $json.title }}`
    - Connect Fix Audio Meta → Telegram audio.

19. **(Optional) Add sticky notes**
    - Add sticky notes to label sections (Inputs, Preparation, AI, Delivery) as in the original workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow concept: aggregates RSS + YouTube, cleans/dedups, Gemini curates, OpenAI generates MP3, delivers to Telegram. | Embedded in sticky note “Daily News Digest: Text & Audio Briefing” |
| Setup requirement: configure credentials for Google Gemini, OpenAI, Telegram, YouTube Data API. | Sticky note “Daily News Digest: Text & Audio Briefing” |
| Environment variable suggestion: define `TELEGRAM_CHAT_ID` (but nodes currently contain placeholders). | Sticky note “⚠️ Check Chat ID” |
| Customization: replace RSS URLs and YouTube keywords; adjust Gemini prompt (story count/length/style). | Sticky notes “1. Input Sources” and “3. AI Analysis” |