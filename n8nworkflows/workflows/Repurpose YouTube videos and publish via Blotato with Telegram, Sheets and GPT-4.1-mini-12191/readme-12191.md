Repurpose YouTube videos and publish via Blotato with Telegram, Sheets and GPT-4.1-mini

https://n8nworkflows.xyz/workflows/repurpose-youtube-videos-and-publish-via-blotato-with-telegram--sheets-and-gpt-4-1-mini-12191


# Repurpose YouTube videos and publish via Blotato with Telegram, Sheets and GPT-4.1-mini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This n8n workflow repurposes a YouTube video into a *new* script and SEO metadata using GPT‑4.1‑mini, logs everything to Google Sheets, notifies via Telegram, and (in a second pipeline) publishes a HeyGen-generated video to YouTube via Blotato once a Google Sheets row is marked `ready`.

**Target use cases**
- Creators who want to quickly rewrite YouTube transcripts into original scripts (same rhythm/structure, different wording/context).
- Teams using Google Sheets as a lightweight “content production queue” (draft → ready → publish).
- Publishing automation via Blotato (upload media + create YouTube post).

**Logical blocks**
1.1 **Telegram Intake & Parsing** → receives a message containing a YouTube URL + instructions.  
1.2 **Configuration + Sheet Logging (Initial Row)** → writes URL/date/description to Sheets.  
1.3 **Transcript Retrieval & Cleanup (RapidAPI)** → extracts videoId, fetches transcript, normalizes text.  
1.4 **AI Generation (Script + SEO) + Sheet Update + Telegram Notify** → produces rewritten script, SEO JSON, updates the row in Sheets, and notifies Telegram.  
1.5 **Publishing Pipeline (Sheets → Drive → Blotato → YouTube → Sheets)** → polls rows with `status=ready`, downloads the HeyGen Drive file, uploads to Blotato, creates a private YouTube post, marks row as `publish`.

**Entry points**
- **Telegram Trigger** (primary intake entry).
- **Workflow Configuration II** (acts like a manual/secondary entry for the publishing pipeline; without a real trigger node, this part typically runs only when executed manually or called by another workflow).

---

## 2. Block-by-Block Analysis

### 2.1 Telegram Intake & Parsing

**Overview:** Listens for Telegram messages and extracts the YouTube URL (line 1) and user instructions (line 2). This becomes the canonical input for downstream logging and AI prompts.

**Nodes involved**
- Telegram Trigger
- Parse Telegram Message

#### Node: Telegram Trigger
- **Type / role:** `telegramTrigger` — webhook-based entry point for Telegram updates.
- **Configuration (interpreted):**
  - Listens to **message** updates only.
  - Uses Telegram credential `Telegram - youtube`.
- **Inputs/Outputs:** No input; outputs Telegram update payload at `item.json.message...`.
- **Potential failures / edge cases:**
  - Credential/token invalid → auth errors.
  - Bot not added to chat / privacy settings → no messages received.
  - Non-text messages (voice/photo) → `message.text` may be undefined later.

#### Node: Parse Telegram Message
- **Type / role:** `code` — transforms raw message into `{ yt_url, user_desc }`.
- **Key logic:**
  - `messageText = $input.item.json.message.text`
  - Split by newline; line 0 → `yt_url`, line 1 → `user_desc`.
- **Output:** `item.json = { yt_url, user_desc }`
- **Edge cases:**
  - Message has only one line → `user_desc` becomes empty string.
  - Message is not text → throws due to `.text` undefined (currently not guarded).
  - Extra lines are ignored.

---

### 2.2 Configuration + Sheet Logging (Initial Row)

**Overview:** Sets required configuration values (language, sheet identifiers, RapidAPI key) and appends/updates a row in Google Sheets with the incoming URL/date/description.

**Nodes involved**
- Workflow Configuration
- Append to Google Sheets

#### Node: Workflow Configuration
- **Type / role:** `set` — stores workflow constants.
- **Configuration choices:**
  - `output_lang = "english"`
  - `sheet_id = <placeholder>`
  - `sheet_tab = <placeholder>`
  - `rapidapi_key = <placeholder>`
  - `includeOtherFields = true` (keeps upstream parsed fields in the item too).
- **Edge cases:**
  - Placeholders not replaced → downstream Google Sheets / RapidAPI will fail.

#### Node: Append to Google Sheets
- **Type / role:** `googleSheets` — writes intake data into a tracking sheet.
- **Operation:** `appendOrUpdate`
- **Document/Sheet:**
  - Document ID from `Workflow Configuration.sheet_id`
  - Tab name from `Workflow Configuration.sheet_tab`
- **Columns written:**
  - `Url = Parse Telegram Message.yt_url`
  - `Date = $now.toISO()`
  - `Description = Parse Telegram Message.user_desc`
- **Matching columns:** `Date` (this is unusual—dates are rarely stable keys).
- **Outputs:** A sheet row representation (used downstream by “Get Video ID” which expects `Url`).
- **Failure modes / edge cases:**
  - OAuth expired / permission denied.
  - Tab name mismatch.
  - Using `Date` as matching key can cause duplicates rather than updates if timestamps differ; can also overwrite if collisions happen.

---

### 2.3 Transcript Retrieval & Cleanup (RapidAPI)

**Overview:** Extracts the YouTube videoId from the sheet URL, calls RapidAPI to fetch transcript data, then consolidates segments into a clean transcript string.

**Nodes involved**
- Get Video ID
- RapidAPI Summarizer
- Clean Text

#### Node: Get Video ID
- **Type / role:** `code` — parses YouTube URL formats into a videoId.
- **Key expressions:**
  - Reads `const url = $input.item.json.Url;`
  - Supports:
    - `youtu.be/<id>`
    - `youtube.com/watch?v=<id>` (and variants with `&v=`).
- **Output:** `{ videoId }`
- **Edge cases:**
  - Shorts URLs (`youtube.com/shorts/<id>`) not handled.
  - Embedded URLs (`/embed/<id>`) not handled.
  - If URL is invalid → `videoId = null`, next HTTP request likely fails.

#### Node: RapidAPI Summarizer
- **Type / role:** `httpRequest` — fetches transcript.
- **Request:**
  - `GET https://youtube-video-summarizer-gpt-ai.p.rapidapi.com/api/v1/get-transcript-v2`
  - Query:
    - `video_id = {{$json.videoId}}`
    - `platform = youtube`
  - Headers:
    - `X-RapidAPI-Key = Workflow Configuration.rapidapi_key`
    - `X-RapidAPI-Host = youtube-video-summarizer-gpt-ai.p.rapidapi.com`
- **Output:** Expects a JSON structure with `data.transcripts` and `data.language_code`.
- **Failure modes:**
  - RapidAPI quota exceeded / billing / invalid key.
  - Video has no transcript / region restrictions.
  - API schema changes → the next “Clean Text” node may break.

#### Node: Clean Text
- **Type / role:** `code` — normalizes transcript segments to a single cleaned string.
- **Important assumptions about input:**
  - `$input.item.json.data` exists.
  - `data.transcripts` is an object keyed by language variants (e.g., `fr_auto`, `en_auto`).
- **Logic highlights:**
  - Preferred language from `data.language_code[0].code` (if present).
  - Chooses transcript language in order:
    1) preferredLang  
    2) `fr_auto`  
    3) `en_auto`  
    4) first available key
  - Chooses segment list priority: `custom` → `auto` → `default`
  - Joins `segment.text` values, cleans whitespace and common HTML entities.
- **Output:** `{ videoId: data.videoId, language, clean_transcript }`
- **Edge cases / failures:**
  - Throws explicit errors when `data` or `data.transcripts` missing.
  - If segments are huge, downstream OpenAI prompt may exceed token limits.
  - If transcript language keys differ from expected, selection may choose wrong language.

---

### 2.4 AI Generation (Script + SEO) + Sheet Update + Telegram Notify

**Overview:** Uses GPT‑4.1‑mini to rewrite the transcript into a new script following strict “same structure/rhythm, different wording” rules, then generates SEO metadata as strict JSON. Results are written back to Google Sheets and a Telegram confirmation is sent.

**Nodes involved**
- Generate Script
- Generate SEO
- update Google Sheets
- Send to Telegram

#### Node: Generate Script
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — LLM generation (freeform text).
- **Model:** `gpt-4.1-mini`
- **Prompting approach:**
  - System: professional video script writer; output in `Workflow Configuration.output_lang`.
  - User content includes:
    - `clean_transcript`
    - user adaptation request from `Parse Telegram Message.user_desc`
    - strict rewrite rules + “no longer than 5 minutes”
  - Output requested: **only** the rewritten script (no commentary).
- **Outputs:** Stored under the node’s structured output (later referenced as `$('Generate Script').item.json.output[0].content[0].text`).
- **Edge cases / failures:**
  - Missing `clean_transcript` (upstream failure).
  - Token/context limit if transcript is long.
  - Model may not perfectly respect “±10% length” or “<= 5 minutes” constraints.

#### Node: Generate SEO
- **Type / role:** OpenAI (LangChain) — structured JSON generation.
- **Model:** `gpt-4.1-mini`
- **Configuration:**
  - Uses **json_schema** response formatting:
    - `title` (<=70 chars)
    - `description` (string)
    - `tags` array of exactly 15 strings
  - Input is the generated script: `={{ $json.output[0].content[0].text }}`
- **Edge cases / failures:**
  - If the script is empty, SEO output becomes low-quality.
  - If model returns invalid JSON despite schema enforcement (rare, but possible depending on node/version).

#### Node: update Google Sheets
- **Type / role:** `googleSheets` — updates the existing row with AI outputs.
- **Operation:** `update`
- **Matching column:** `Url` (more stable than Date; better key).
- **Document/Sheet:** from Workflow Configuration node values.
- **Fields written:**
  - `Url` and `Date` refreshed
  - `Transcript New = Generate Script` output text
  - `Title New / Description New / Tags = Generate SEO` fields
  - `row_number = 0` (note: schema marks it readOnly; setting it may be ignored)
- **Potential issues:**
  - `Tags` is written as `={{ $json.output[0].content[0].text.tags }}` (an array) while the column schema type is string; may serialize oddly (`a,b,c`) or fail depending on Sheets node behavior.
  - If multiple rows share the same Url, update target may be ambiguous.

#### Node: Send to Telegram
- **Type / role:** `telegram` — sends a confirmation message.
- **Text:** `Transcript : done`
- **Chat ID:** `Telegram Trigger.message.chat.id` (replies to the sender chat).
- **Edge cases:**
  - If workflow is triggered from a channel/group, bot permissions may block sending.
  - If upstream execution is not directly from Telegram Trigger (manual run), chat.id may be missing.

---

### 2.5 Publishing Pipeline (Sheets → Drive → Blotato → YouTube → Sheets)

**Overview:** Reads Google Sheets rows where `status=ready`, extracts a Google Drive file ID from a HeyGen URL column, downloads the file, uploads to Blotato media, creates a private YouTube post, then updates the row status to `publish`.

**Nodes involved**
- Workflow Configuration II
- Check Google Sheets
- Get Google Drive ID
- Download file
- Upload Video to BLOTATO
- Youtube
- Update Google Sheets II

#### Node: Workflow Configuration II
- **Type / role:** `set` — provides sheet_id and sheet_tab for publishing pipeline.
- **Important:** This block has **no trigger**; the node is the first node in its chain, so the pipeline usually runs **manually** (or via “Execute workflow” / another workflow call).
- **Edge cases:** placeholders not replaced.

#### Node: Check Google Sheets
- **Type / role:** `googleSheets` — reads rows matching a filter.
- **Filter:** `status` column equals `ready`.
- **Output:** Each matching row becomes an item, including fields like `Url`, `Title New`, `Description New`, `url Heygen`, etc. (depending on sheet columns).
- **Failure modes:**
  - Missing `status` column.
  - No results → downstream nodes receive no items.

#### Node: Get Google Drive ID
- **Type / role:** `set` — extracts Drive file ID from the HeyGen URL.
- **Key expression:**
  - `final_google_drive_url = $json['url Heygen'].match(/https:\/\/drive\.google\.com\/file\/d\/([A-Za-z0-9_-]+)/i)[1]`
- **Edge cases / failures:**
  - If `url Heygen` is empty or not matching that exact pattern → `match(...)` returns null and `[1]` throws.
  - Drive links in other formats (e.g., `open?id=`, shared `uc?id=`) not supported.

#### Node: Download file
- **Type / role:** `googleDrive` — downloads binary content for the Drive file.
- **Operation:** `download`
- **fileId:** from `final_google_drive_url`
- **Output:** Binary data for the video file.
- **Failure modes:**
  - Drive permission denied / file not shared correctly.
  - Large file download timeouts depending on n8n settings.

#### Node: Upload Video to BLOTATO
- **Type / role:** `@blotato/n8n-nodes-blotato.blotato` — uploads media to Blotato library.
- **Resource:** `media`
- **mediaUrl:** constructed as `https://drive.google.com/uc?export=download&id={{ final_google_drive_url }}`
  - Note: This uses a *direct download* URL, not the binary from “Download file” (the prior download is therefore not strictly required unless used for validation).
- **Output:** Expected to include a media URL (used by the next node as `$json.url`).
- **Failure modes:**
  - Blotato auth/API errors.
  - Drive file not publicly downloadable via `uc?export=download` (common if not shared).
  - Rate limits / unsupported media formats.

#### Node: Youtube
- **Type / role:** Blotato — creates a YouTube post via connected account.
- **Platform:** `youtube`
- **Account:** ID `8047` (cached name “DR FIRASS (Dr. Firas)”).
- **Post fields:**
  - `postContentText = Check Google Sheets['Description New']`
  - `postContentMediaUrls = $json.url` (from Blotato upload response)
  - Title: `Check Google Sheets['Title New']`
  - Privacy: `private`
  - Notify subscribers: `false`
- **Edge cases:**
  - Missing Title/Description New in the sheet → YouTube upload may fail validation.
  - Account disconnected in Blotato.
  - Video processing delays; Blotato may return success before YouTube fully processes.

#### Node: Update Google Sheets II
- **Type / role:** `googleSheets` — marks publishing completion.
- **Operation:** `update`
- **Matching column:** `Url`
- **Columns updated:**
  - `Url` (from Check Google Sheets)
  - `status = "publish"`
- **Failure modes:**
  - Same as other Sheets updates; plus ambiguity if multiple rows share the same URL.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | n8n-nodes-base.telegramTrigger | Entry point: receive Telegram messages | — | Parse Telegram Message | # Setup & Configuration Guide; @[youtube](szKueQ7Aen8); Telegram Bot via [@BotFather](https://t.me/botfather); credential `Telegram - youtube`; Google APIs setup; RapidAPI notes; [My Google Sheets : copy](https://docs.google.com/spreadsheets/d/1A-LpZ8OGn8FP692Hx50j3sEuPgIhlyMOrJ1jUNGwHr8/copy) |
| Parse Telegram Message | n8n-nodes-base.code | Extract yt_url and user_desc from message | Telegram Trigger | Workflow Configuration | # Setup & Configuration Guide (same as above) |
| Workflow Configuration | n8n-nodes-base.set | Store constants (language, sheet, RapidAPI key) | Parse Telegram Message | Append to Google Sheets | # Setup & Configuration Guide (same as above) |
| Append to Google Sheets | n8n-nodes-base.googleSheets | Append/update intake row (Url/Date/Description) | Workflow Configuration | Get Video ID | # Setup & Configuration Guide (same as above) |
| Get Video ID | n8n-nodes-base.code | Parse videoId from YouTube URL | Append to Google Sheets | RapidAPI Summarizer | # Setup & Configuration Guide (same as above) |
| RapidAPI Summarizer | n8n-nodes-base.httpRequest | Fetch transcript via RapidAPI | Get Video ID | Clean Text | # Setup & Configuration Guide (same as above) |
| Clean Text | n8n-nodes-base.code | Choose transcript language + merge segments + cleanup | RapidAPI Summarizer | Generate Script | # Setup & Configuration Guide (same as above) |
| Generate Script | @n8n/n8n-nodes-langchain.openAi | Rewrite transcript into new script | Clean Text | Generate SEO | ### OpenAI API; [OpenAI Platform](https://platform.openai.com); credential `n8n free OpenAI API credits`; Model **GPT-4.1-MINI**; ### BLOTATO; [BLOTATO](https://blotato.com/?ref=firas); credential `Blotato account` |
| Generate SEO | @n8n/n8n-nodes-langchain.openAi | Generate SEO JSON (title/description/15 tags) | Generate Script | update Google Sheets | ### OpenAI API (same as above); ### BLOTATO (same as above) |
| update Google Sheets | n8n-nodes-base.googleSheets | Update row with script + SEO fields | Generate SEO | Send to Telegram | ### OpenAI API (same as above); ### BLOTATO (same as above) |
| Send to Telegram | n8n-nodes-base.telegram | Notify completion back to Telegram chat | update Google Sheets | — | # Setup & Configuration Guide (same as above) |
| Workflow Configuration II | n8n-nodes-base.set | Sheet constants for publishing pipeline | — | Check Google Sheets | ## Video Publishing Pipeline; Notion link: https://automatisation.notion.site/YOUTUBE-2d53d6550fd980d0ba16fa054ecb1f95?source=copy_link |
| Check Google Sheets | n8n-nodes-base.googleSheets | Query rows with status=ready | Workflow Configuration II | Get Google Drive ID | ## Video Publishing Pipeline (same as above) |
| Get Google Drive ID | n8n-nodes-base.set | Extract Drive file ID from “url Heygen” | Check Google Sheets | Download file | ## Video Publishing Pipeline (same as above) |
| Download file | n8n-nodes-base.googleDrive | Download video from Google Drive | Get Google Drive ID | Upload Video to BLOTATO | ## Video Publishing Pipeline (same as above) |
| Upload Video to BLOTATO | @blotato/n8n-nodes-blotato.blotato | Upload media to Blotato using Drive direct URL | Download file | Youtube | ## Video Publishing Pipeline (same as above); ### BLOTATO (see OpenAI/BLOTATO note) |
| Youtube | @blotato/n8n-nodes-blotato.blotato | Create YouTube post (private) via Blotato | Upload Video to BLOTATO | Update Google Sheets II | ## Video Publishing Pipeline (same as above); ### BLOTATO (see OpenAI/BLOTATO note) |
| Update Google Sheets II | n8n-nodes-base.googleSheets | Mark row status = publish | Youtube | — | ## Video Publishing Pipeline (same as above) |
| Setup & Configuration Guide | n8n-nodes-base.stickyNote | Documentation/comment | — | — |  |
| Transcript Extraction | n8n-nodes-base.stickyNote | Documentation/comment | — | — |  |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation/comment | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Telegram Trigger**
   - Node: *Telegram Trigger*
   - Updates: `message`
   - Credential: Telegram API OAuth/token credential named **`Telegram - youtube`** (token from @BotFather).

2) **Add Code node: “Parse Telegram Message”**
   - Mode: *Run Once for Each Item*
   - JS: split `message.text` by newline; output:
     - `yt_url = first line`
     - `user_desc = second line`

3) **Add Set node: “Workflow Configuration”**
   - Add fields:
     - `output_lang` (e.g., `english`)
     - `sheet_id` (Google Sheets document ID)
     - `sheet_tab` (tab name)
     - `rapidapi_key` (RapidAPI key)
   - Enable “Include Other Fields”.

4) **Add Google Sheets node: “Append to Google Sheets”**
   - Operation: **Append or Update**
   - Document ID: expression from `Workflow Configuration.sheet_id`
   - Sheet name: expression from `Workflow Configuration.sheet_tab`
   - Map columns:
     - `Url = Parse Telegram Message.yt_url`
     - `Date = $now.toISO()`
     - `Description = Parse Telegram Message.user_desc`
   - Matching column: (as in workflow) `Date` (recommended improvement: use `Url` instead).
   - Credential: **Google Sheets OAuth2 API** named `Google Sheets account`.

5) **Add Code node: “Get Video ID”**
   - Read `$input.item.json.Url`
   - Parse `youtu.be/...` and `watch?v=...` into `videoId`
   - Output `{ videoId }`

6) **Add HTTP Request node: “RapidAPI Summarizer”**
   - Method: GET
   - URL: `https://youtube-video-summarizer-gpt-ai.p.rapidapi.com/api/v1/get-transcript-v2`
   - Query params:
     - `video_id = {{$json.videoId}}`
     - `platform = youtube`
   - Headers:
     - `X-RapidAPI-Key = {{Workflow Configuration.rapidapi_key}}`
     - `X-RapidAPI-Host = youtube-video-summarizer-gpt-ai.p.rapidapi.com`

7) **Add Code node: “Clean Text”**
   - Validate presence of `data` and `data.transcripts`
   - Choose language object (preferred → fr_auto → en_auto → first)
   - Choose segments (custom → auto → default)
   - Join all segment text to `clean_transcript`, normalize whitespace/entities
   - Output `{ videoId, language, clean_transcript }`

8) **Add OpenAI node: “Generate Script” (LangChain OpenAI)**
   - Model: `gpt-4.1-mini`
   - System message: scriptwriter instructions, language from `Workflow Configuration.output_lang`
   - User message: `clean_transcript` + `Parse Telegram Message.user_desc` + rewrite rules
   - Credential: OpenAI API credential named **`n8n free OpenAI API credits`** (or your own).

9) **Add OpenAI node: “Generate SEO”**
   - Model: `gpt-4.1-mini`
   - Enable structured output using JSON schema:
     - `title` (string, <=70 chars by instruction)
     - `description` (string)
     - `tags` (array, exactly 15 strings)
   - Prompt: “Based ONLY on provided script, generate SEO metadata… output valid JSON only”
   - Input content: expression referencing prior script output.

10) **Add Google Sheets node: “update Google Sheets”**
   - Operation: **Update**
   - Document ID / Sheet name from `Workflow Configuration`
   - Matching column: `Url`
   - Update fields:
     - `Transcript New` from Generate Script output
     - `Title New`, `Description New`, `Tags` from Generate SEO output
     - plus `Url` and `Date` if desired
   - Credential: `Google Sheets account`

11) **Add Telegram node: “Send to Telegram”**
   - Operation: send message
   - Chat ID: `Telegram Trigger.message.chat.id`
   - Text: `Transcript : done`

12) **Create publishing pipeline (manual/secondary execution)**
   - Add Set node: **Workflow Configuration II** with `sheet_id`, `sheet_tab` (same as above).
   - Add Google Sheets node: **Check Google Sheets**
     - Filter: `status` equals `ready`
   - Add Set node: **Get Google Drive ID**
     - Extract file ID from column `url Heygen` using regex on `https://drive.google.com/file/d/<ID>`
   - Add Google Drive node: **Download file**
     - Operation: download
     - File ID: `final_google_drive_url`
     - Credential: **Google Drive OAuth2 API** named `Google Drive account`
   - Add Blotato node: **Upload Video to BLOTATO**
     - Resource: media
     - mediaUrl: `https://drive.google.com/uc?export=download&id={{final_google_drive_url}}`
     - Credential: **Blotato API** named `Blotato account`
   - Add Blotato node: **Youtube**
     - Platform: youtube
     - Account: select your connected YouTube account in Blotato
     - Title: from sheet `Title New`
     - Description/Text: from sheet `Description New`
     - Media URLs: from previous Blotato upload output (`$json.url`)
     - Privacy: private; notify subscribers: false
   - Add Google Sheets node: **Update Google Sheets II**
     - Operation: update
     - Matching column: `Url`
     - Set `status = publish`

13) **Connect nodes in order**
   - Intake chain: Telegram Trigger → Parse → Workflow Configuration → Append → Get Video ID → RapidAPI → Clean Text → Generate Script → Generate SEO → update Google Sheets → Send to Telegram
   - Publishing chain: Workflow Configuration II → Check Google Sheets → Get Google Drive ID → Download file → Upload Video to BLOTATO → Youtube → Update Google Sheets II

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Telegram bot creation via BotFather; credential name `Telegram - youtube` | https://t.me/botfather |
| Google Sheets template (copy) | https://docs.google.com/spreadsheets/d/1A-LpZ8OGn8FP692Hx50j3sEuPgIhlyMOrJ1jUNGwHr8/copy |
| RapidAPI subscription: “YouTube Video Summarizer GPT AI” | https://rapidapi.com |
| OpenAI Platform (API key) | https://platform.openai.com |
| Blotato signup and YouTube connection | https://blotato.com/?ref=firas |
| Full documentation on Notion | https://automatisation.notion.site/YOUTUBE-2d53d6550fd980d0ba16fa054ecb1f95?source=copy_link |
| Embedded video reference `@[youtube](szKueQ7Aen8)` | Mentioned in sticky note (internal reference) |