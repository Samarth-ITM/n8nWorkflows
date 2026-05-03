Create Anki language flashcards with GPT-4, DALL-E and ElevenLabs

https://n8nworkflows.xyz/workflows/create-anki-language-flashcards-with-gpt-4--dall-e-and-elevenlabs-12265


# Create Anki language flashcards with GPT-4, DALL-E and ElevenLabs

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Create Anki language flashcards with GPT-4, DALL-E and ElevenLabs

**Purpose:**  
This workflow collects user preferences via an n8n Form, generates a set of language-learning flashcards using OpenAI (GPT-4o), generates an illustration per card using DALL·E 3, generates pronunciation audio with ElevenLabs, compiles everything into a real Anki package (`.apkg`), backs up the card data to Google Sheets, emails the `.apkg` to the user via Gmail, and finally returns a JSON response.

**Typical use cases**
- Rapid creation of Anki decks for vocabulary acquisition (topic-based or user-provided word list).
- Language learning with spaced repetition including images + native-like audio.
- Automated deck delivery and backup for later reuse.

### 1.1 Input Reception (Form)
User submits topic/word list, languages, number of cards, difficulty, image style, reverse-card option, email.

### 1.2 Validation & Configuration
Sanitizes and constrains inputs (e.g., card count), creates deck ID, maps target/native languages to ElevenLabs voice IDs.

### 1.3 Vocabulary Generation (GPT-4o)
Requests a strict JSON schema response that includes per-card: word, translation, readings/romanization, example sentence + translation, and an image prompt.

### 1.4 Media Preparation & Per-Card Loop
Transforms GPT output into per-card items, then loops:
- DALL·E 3 image generation (base64)
- ElevenLabs word TTS (currently disabled in workflow)
- ElevenLabs example sentence TTS
- Converts binary audio to base64 and merges into card data

### 1.5 Anki Package Compilation (.apkg)
Aggregates all card items, creates Anki DB structures (models/decks/notes/cards), writes `collection.anki21`, adds media + mapping, zips into `.apkg` and base64-encodes it.

### 1.6 Backup to Google Sheets
Writes a row per flashcard to a specified spreadsheet/sheet.

### 1.7 Delivery (Gmail) & API Response
Attaches `.apkg` to an email, sends to the user, then returns a JSON response to the original form submission.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Form)

**Overview:** Collects all deck requirements from the user via a hosted n8n form and initiates the workflow.

**Nodes involved:**
- Sticky Note - Main
- Sticky Note1
- Flashcard Form

#### Node: Sticky Note - Main
- **Type / role:** Sticky Note (documentation)
- **Configuration (interpreted):** Describes end-to-end flow and setup checklist (npm packages, credentials, spreadsheet ID, DALL·E API key).
- **Connections:** None (informational only)
- **Edge cases:** None

#### Node: Sticky Note1
- **Type / role:** Sticky Note (documentation)
- **Configuration:** “Step 1: Form – Collects user preferences”
- **Connections:** None

#### Node: Flashcard Form
- **Type / role:** `formTrigger` (entry point)
- **Key configuration:**
  - Form title: “🎴 AI Flashcard Generator Pro”
  - Fields:
    - Email (required)
    - Topic/Word list textarea (required)
    - Native language dropdown (required)
    - Target language dropdown (required)
    - Number of flashcards (required)
    - Difficulty level dropdown (required)
    - Image style dropdown (required)
    - Reverse cards dropdown (required)
  - Custom “submitted” message indicates 2–5 minutes processing time.
- **Outputs:** To **Validate Input**
- **Edge cases / failures:**
  - Users entering unusually long topics/word lists can increase token usage/cost.
  - “Number of Flashcards” is user-entered but later clamped to 1–20.
- **Version notes:** Node is `typeVersion 2.3` (Form Trigger behavior/field UI can differ between n8n versions).

---

### Block 2 — Validate & Map Voices

**Overview:** Normalizes user data, constrains card count, generates a unique deck ID, and maps language selections to ElevenLabs voice IDs.

**Nodes involved:**
- Sticky Note2
- Sticky Note - Config
- Validate Input

#### Node: Sticky Note2
- **Type / role:** Sticky Note
- **Configuration:** “Step 2: Validate – Check inputs & map voices”

#### Node: Sticky Note - Config
- **Type / role:** Sticky Note
- **Configuration:** Provides ElevenLabs voice IDs by language and notes that mapping occurs in Validate Input.

#### Node: Validate Input
- **Type / role:** `code` node (JavaScript transformation + validation)
- **Key configuration choices:**
  - Clamps `numCards` to **1..20** (defaults to 10).
  - Builds `deckId` as `deck_<timestamp>_<random>`.
  - Maps `targetLanguage` and `nativeLanguage` to voice IDs using `voiceMap`.
  - Sets:
    - `generateReverse`: boolean from “Generate Reverse Cards?” dropdown (checks `.includes('Yes')`)
    - `createdAt`: ISO timestamp
- **Key variables produced (output JSON):**
  - `email`, `topic`, `nativeLanguage`, `targetLanguage`, `numCards`, `level`, `imageStyle`, `generateReverse`
  - `deckId`, `targetVoiceId`, `nativeVoiceId`, `createdAt`
- **Inputs:** From **Flashcard Form**
- **Outputs:** To **Generate Flashcards (GPT-4)**
- **Edge cases / failures:**
  - If a language string doesn’t exactly match a `voiceMap` key, fallback voice IDs are used:
    - `targetVoiceId` fallback: English voice `pNInz6obpgDQGcFmaJgB`
    - `nativeVoiceId` fallback: `21m00Tcm4TlvDq8ikWAM` (not listed in the sticky note)
  - The mapping includes both `'Chinese (Mandarin)'` and `'Chinese'`, but the form uses `'Chinese (Mandarin)'` for target and `'Chinese'` for native; this is handled.
- **Version notes:** Code node `typeVersion 2` uses n8n’s newer code execution environment.

---

### Block 3 — GPT Vocabulary Generation

**Overview:** Calls OpenAI Chat Completions to generate the structured deck content (strict JSON schema), including example sentences and image prompts.

**Nodes involved:**
- Sticky Note3
- Generate Flashcards (GPT-4)

#### Node: Sticky Note3
- **Type / role:** Sticky Note
- **Configuration:** “Step 3: GPT-4 – Generate vocabulary”

#### Node: Generate Flashcards (GPT-4)
- **Type / role:** `httpRequest` node calling OpenAI API
- **Authentication:**
  - Uses **predefined credential type**: `openAiApi` (n8n credential)
- **Request details (interpreted):**
  - POST `https://api.openai.com/v1/chat/completions`
  - Model: `gpt-4o`
  - System prompt: language learning expert, include translations/readings/examples, simple image prompts
  - User prompt: injects `numCards`, `targetLanguage`, `topic`, `level`, `nativeLanguage`, plus language-specific rules
  - Uses `response_format` with **json_schema** and `"strict": true`
- **Inputs:** From **Validate Input**
- **Outputs:** To **Prepare Card Data**
- **Edge cases / failures:**
  - Auth errors if OpenAI credential missing/invalid.
  - Schema strictness: if the model fails schema compliance, OpenAI may return an error or a non-parseable structure.
  - Large `topic` text may inflate tokens or cause truncation.
- **Version notes:** Node `typeVersion 4.3` (HTTP Request) may have slightly different auth/JSON behavior across n8n versions.

---

### Block 4 — Prepare Card Data & Split for Loop

**Overview:** Parses the GPT JSON, sanitizes text for safe prompt composition, builds DALL·E prompts, and outputs one item per card for batch processing.

**Nodes involved:**
- Sticky Note4
- Prepare Card Data
- Split Into Cards

#### Node: Sticky Note4
- **Type / role:** Sticky Note
- **Configuration:** “Step 4: Prepare – Format for media gen”

#### Node: Prepare Card Data
- **Type / role:** `code` node (parse + transform)
- **Key configuration choices:**
  - Parses: `JSON.parse(response.choices[0].message.content)`
  - Builds `dallePrompt` using:
    - `formData.imageStyle`
    - `card.translation` and `card.imagePrompt`
    - Enforces “white background, centered, no text”
    - Truncates prompt to 4000 chars
  - Adds placeholders: `imageBase64`, `wordAudioBase64`, `exampleAudioBase64` set to null
  - Includes metadata: `deckId`, voice IDs, `generateReverse`, timestamps
- **Key expressions / references:**
  - Pulls form data via `$('Validate Input').first().json`
- **Inputs:** From **Generate Flashcards (GPT-4)**
- **Outputs:** To **Split Into Cards**
- **Edge cases / failures:**
  - If OpenAI response path changes or content isn’t valid JSON, `JSON.parse` will throw and fail the node.
  - Sanitization replaces quotes and removes control chars; if a language relies on certain punctuation, it may be altered.
- **Version notes:** Code node v2.

#### Node: Split Into Cards
- **Type / role:** `code` node (fan-out items)
- **Behavior:**
  - Converts `deck.cards` array into multiple n8n items (one per card).
  - Carries deck-level metadata onto each card item (`deckName`, languages, level, etc.)
- **Inputs:** From **Prepare Card Data**
- **Outputs:** To **Loop Cards**
- **Edge cases:** If `deck.cards` is missing/empty, outputs empty list, breaking downstream compilation.

---

### Block 5 — Per-Card Loop (Image + Audio)

**Overview:** Iterates through cards using SplitInBatches, generates image and audio, converts audio binaries to base64, and loops until all cards are processed.

**Nodes involved:**
- Sticky Note5
- Sticky Note6
- Sticky Note7
- Sticky Note8
- Sticky Note - NPM
- Loop Cards
- Generate Image (DALL-E)
- TTS Word (ElevenLabs) *(disabled)*
- Convert Word Audio
- TTS Example (ElevenLabs)
- Convert Example Audio
- Merge Card Data

#### Node: Sticky Note5 / 6 / 7 / 8
- **Type / role:** Sticky Notes
- **Configuration:** Steps 5–8 labels (Loop, DALL·E, ElevenLabs word, ElevenLabs example)

#### Node: Sticky Note - NPM
- **Type / role:** Sticky Note
- **Configuration:**
  - Required packages: `jszip`, `sql.js`
  - Docker env hint: `NODE_FUNCTION_ALLOW_EXTERNAL=jszip,sql.js`

#### Node: Loop Cards
- **Type / role:** `splitInBatches` (loop controller)
- **Configuration:** Defaults (batch size not explicitly set; n8n default is typically 1 unless configured)
- **Connections:**
  - Main output 1 → **Generate Image (DALL-E)** (next card in batch)
  - Main output 2 → **Create APKG Data** (when loop completes)
- **Edge cases / failures:**
  - If batch size is changed >1, downstream nodes that assume single item semantics may still work (most use `$input.first()`), but aggregated behavior could be unexpected.

#### Node: Generate Image (DALL-E)
- **Type / role:** `code` node performing direct HTTP call to OpenAI Images API
- **Key configuration choices:**
  - Uses `fetch` to POST `https://api.openai.com/v1/images/generations`
  - Model: `dall-e-3`
  - `response_format: 'b64_json'` (stores image base64)
  - Size: 1024x1024, quality: standard
  - **Hardcoded API key placeholder**: `OPENAI_API_KEY = 'YOUR_OPENAI_API_KEY'`
- **Inputs:** From **Loop Cards** (one item with `card.dallePrompt`)
- **Outputs:** To **TTS Word (ElevenLabs)**
- **Edge cases / failures:**
  - Will fail unless the API key is replaced; also not using n8n OpenAI credentials.
  - DALL·E policy blocks or prompt issues return `result.error`, captured into `card.imageError`.
  - Network/timeouts not retried automatically here.
- **Version notes:** Requires Node.js runtime in n8n that supports `fetch` (modern n8n generally does).

#### Node: TTS Word (ElevenLabs) *(disabled)*
- **Type / role:** ElevenLabs node (`@elevenlabs/n8n-nodes-elevenlabs.elevenLabs`)
- **Status:** **Disabled**, so word audio generation does not run in the current workflow.
- **Expected role if enabled:** Generate pronunciation audio for the vocabulary word (binary output).
- **Connections:** Would output to **Convert Word Audio**
- **Edge cases:** Credential/voice ID mismatch; rate limits; language/voice incompatibility.

#### Node: Convert Word Audio
- **Type / role:** `code` node (binary → base64 merge)
- **Key behavior:**
  - Reads upstream binary audio at `$input.first().binary.data.data`
  - Merges it back into the JSON from `$('Generate Image (DALL-E)').first().json`
  - Sets `card.wordAudioBase64` and `card.wordAudioError`
- **Inputs:** Would come from **TTS Word (ElevenLabs)**, but since TTS Word is disabled, this path is effectively broken unless re-enabled.
- **Outputs:** To **TTS Example (ElevenLabs)**
- **Edge cases / failures:**
  - If no binary data is present, word audio remains null.
  - Because it references **Generate Image (DALL-E)** directly, if multiple items are in play concurrently, cross-item mismatches could occur; the loop design (batch size 1) mitigates this.

#### Node: TTS Example (ElevenLabs)
- **Type / role:** ElevenLabs text-to-speech
- **Configuration (in JSON):**
  - Resource: `textToSpeech`
  - Request options default
  - (Voice ID and text are not shown in the provided parameters; likely set via node UI fields not included here or defaults—this is a critical reproduction point.)
- **Inputs:** From **Convert Word Audio**
- **Outputs:** To **Convert Example Audio**
- **Edge cases / failures:**
  - Missing credentials.
  - If text input is empty or not mapped, audio will be incorrect or node may error.
  - ElevenLabs rate limits / quota exhaustion.

#### Node: Convert Example Audio
- **Type / role:** `code` node (binary → base64 merge)
- **Key behavior:**
  - Takes JSON from `$('Convert Word Audio').first().json`
  - Reads example binary audio from `$input.first().binary.data.data`
  - Sets `card.exampleAudioBase64` and `card.exampleAudioError`
- **Outputs:** To **Merge Card Data**

#### Node: Merge Card Data
- **Type / role:** `code` node (pass-through)
- **Behavior:** Returns the current item unchanged, then loops back to **Loop Cards** for next batch.
- **Connections:** To **Loop Cards**
- **Edge cases:** None (but it’s the join point after media generation).

---

### Block 6 — Build Anki Package (.apkg)

**Overview:** Once all cards are processed, aggregates them into Anki’s SQLite schema and builds a ZIP with the required `collection.anki21` database and `media` mapping.

**Nodes involved:**
- Sticky Note9
- Sticky Note10
- Create APKG Data
- Build APKG ZIP

#### Node: Sticky Note9 / Sticky Note10
- **Type / role:** Sticky Notes
- **Configuration:** Steps 9–10 labels (Merge media, Build APKG)

#### Node: Create APKG Data
- **Type / role:** `code` node (build Anki model/deck/note/card structures + media mapping)
- **Key configuration choices:**
  - Aggregates all processed card items via `$input.all()`
  - Builds:
    - `mediaMapping` (index → filename) and `mediaFiles` (index → base64 data/type)
    - Anki model with fields: Word, Reading, Translation, PartOfSpeech, Example, ExampleReading, ExampleTranslation, Image, WordAudio, ExampleAudio, Notes
    - Templates:
      - Recognition card (always)
      - Production/reverse card (only if `generateReverse` true)
    - Notes and cards arrays with deterministic-ish IDs derived from `Date.now()`
  - Embeds media as:
    - `<img src="img_X.png">`
    - `[sound:word_X.mp3]`, `[sound:example_X.mp3]`
- **Inputs:** From **Loop Cards** “done” output (after all iterations)
- **Outputs:** To **Build APKG ZIP**
- **Edge cases / failures:**
  - If DALL·E or TTS failed, media entries may be missing; deck still builds but without those assets.
  - ID generation based on `Date.now()` could collide in extremely fast reruns; unlikely but possible in concurrent executions.
  - Word audio is likely absent because “TTS Word” is disabled.
- **Version notes:** Requires stable JS execution; large decks increase memory usage.

#### Node: Build APKG ZIP
- **Type / role:** `code` node (SQLite creation + ZIP packing)
- **Key configuration choices:**
  - Requires external npm packages: `jszip`, `sql.js`
  - Creates SQLite tables: `col`, `notes`, `cards`, `revlog`, `graves` + indexes
  - Exports DB to `collection.anki21`
  - Creates ZIP with:
    - `collection.anki21`
    - `media` JSON file
    - numbered media files (0,1,2,...) as required by Anki format
  - Outputs:
    - `apkgBase64`, `apkgFileName`
    - `sheetRows` for Google Sheets backup
- **Inputs:** From **Create APKG Data**
- **Outputs:** To **Split For Sheets**
- **Edge cases / failures:**
  - n8n environment must allow external modules:
    - Set `NODE_FUNCTION_ALLOW_EXTERNAL=jszip,sql.js`
    - Ensure packages are installed in the runtime image/container.
  - Large media (1024x1024 per card) can cause memory pressure and slow zipping.
  - If any base64 media data is invalid, Buffer conversion may throw.
- **Version notes:** Code node external imports depend on n8n deployment mode (cloud vs self-hosted).

---

### Block 7 — Google Sheets Backup

**Overview:** Splits backup rows into individual items, appends them to a Google Sheet, then aggregates back to one item for email delivery.

**Nodes involved:**
- Sticky Note11
- Split For Sheets
- Save to Google Sheets
- Aggregate Results

#### Node: Sticky Note11
- **Type / role:** Sticky Note
- **Configuration:** “Step 11: Sheets – Backup data”

#### Node: Split For Sheets
- **Type / role:** `code` node (fan-out rows)
- **Behavior:**
  - Creates one item per row in `sheetRows`
  - Copies `.apkg` data into `_apkgData` for later re-aggregation
- **Inputs:** From **Build APKG ZIP**
- **Outputs:** To **Save to Google Sheets**
- **Edge cases:** If `sheetRows` empty, no writes happen and downstream aggregation may fail.

#### Node: Save to Google Sheets
- **Type / role:** `googleSheets` node (append operation)
- **Key configuration choices:**
  - Operation: Append
  - Spreadsheet: `YOUR_SPREADSHEET_ID` (must be replaced)
  - Sheet name: `Flashcards`
  - Maps many columns: `word`, `reading`, `romanization`, `translation`, etc.
- **Inputs:** Each item is one row
- **Outputs:** To **Aggregate Results**
- **Edge cases / failures:**
  - OAuth credential missing/expired.
  - Spreadsheet ID or sheet name invalid.
  - Column mismatch: If the sheet doesn’t have headers matching mapped columns, append may still work but data alignment can break depending on configuration.
- **Version notes:** Node `typeVersion 4.4` (Google Sheets) varies in field mapping behavior across versions.

#### Node: Aggregate Results
- **Type / role:** `code` node (fan-in)
- **Behavior:** Uses `$input.all()` and returns:
  - the stored `_apkgData`
  - `savedToSheets: <row_count>`
- **Outputs:** To **Prepare Attachment**
- **Edge cases:** If Google Sheets node errors mid-way, execution halts and no email is sent unless you add error handling.

---

### Block 8 — Email Delivery & Final Response

**Overview:** Converts the `.apkg` base64 into a binary attachment, emails it via Gmail, then returns a JSON response to the initial requester.

**Nodes involved:**
- Sticky Note12
- Sticky Note13
- Prepare Attachment
- Send Email (Gmail)
- Return Response

#### Node: Sticky Note12 / Sticky Note13
- **Type / role:** Sticky Notes
- **Configuration:** Steps 12–13 labels (Email, Response)

#### Node: Prepare Attachment
- **Type / role:** `code` node (create n8n binary attachment)
- **Behavior:**
  - Creates `binary.data` with:
    - `data`: `apkgBase64`
    - `mimeType`: `application/octet-stream`
    - `fileName`: generated `.apkg` name
- **Inputs:** From **Aggregate Results**
- **Outputs:** To **Send Email (Gmail)**
- **Edge cases:** Very large base64 attachments may exceed Gmail limits (~25MB).

#### Node: Send Email (Gmail)
- **Type / role:** `gmail` node (send email with attachment)
- **Key configuration choices:**
  - To: `{{$json.email}}`
  - Subject: “🎴 Your Flashcards Are Ready: {{ $json.deckName }}”
  - HTML body includes import instructions and deck stats
  - Attachments: expects binary attachment from previous node
- **Inputs:** JSON + binary from **Prepare Attachment**
- **Outputs:** To **Return Response**
- **Edge cases / failures:**
  - Gmail OAuth not configured or token expired.
  - Attachment too large; message rejected.
  - API rate limits.

#### Node: Return Response
- **Type / role:** `respondToWebhook` (final HTTP response to form submission)
- **Behavior:**
  - Returns JSON with `success`, message, deck metadata, and `savedToSheets`.
  - Sets `Content-Type: application/json`.
- **Inputs:** From **Send Email (Gmail)**
- **Edge cases:**
  - If earlier nodes take too long and the form/webhook expects quick response, you can hit timeout depending on n8n/webhook setup. (The workflow currently responds only at the end.)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Main | stickyNote | Workflow description & setup notes |  |  | ## AI Flashcard Generator Pro / Setup steps incl. npm packages, credentials, update keys/IDs |
| Sticky Note - Config | stickyNote | ElevenLabs voice ID reference |  |  | ## ⚙️ ElevenLabs Voice IDs (list) + mapping note |
| Sticky Note1 | stickyNote | Step label |  |  | **Step 1: Form** Collects user preferences |
| Sticky Note2 | stickyNote | Step label |  |  | **Step 2: Validate** Check inputs & map voices |
| Sticky Note3 | stickyNote | Step label |  |  | **Step 3: GPT-4** Generate vocabulary |
| Sticky Note4 | stickyNote | Step label |  |  | **Step 4: Prepare** Format for media gen |
| Sticky Note5 | stickyNote | Step label |  |  | **Step 5: Loop** Process each card |
| Sticky Note6 | stickyNote | Step label |  |  | **Step 6: DALL-E** Generate images |
| Sticky Note7 | stickyNote | Step label |  |  | **Step 7: ElevenLabs** Word pronunciation |
| Sticky Note8 | stickyNote | Step label |  |  | **Step 8: ElevenLabs** Example sentence audio |
| Sticky Note9 | stickyNote | Step label |  |  | **Step 9: Merge** Combine all media |
| Sticky Note10 | stickyNote | Step label |  |  | **Step 10: APKG** Build Anki package |
| Sticky Note11 | stickyNote | Step label |  |  | **Step 11: Sheets** Backup data |
| Sticky Note12 | stickyNote | Step label |  |  | **Step 12: Email** Send via Gmail |
| Sticky Note13 | stickyNote | Step label |  |  | **Step 13: Response** Return JSON |
| Flashcard Form | formTrigger | Collect user inputs | — | Validate Input | **Step 1: Form** Collects user preferences |
| Validate Input | code | Validate fields, clamp card count, map voice IDs | Flashcard Form | Generate Flashcards (GPT-4) | **Step 2: Validate** Check inputs & map voices |
| Generate Flashcards (GPT-4) | httpRequest | OpenAI chat completion with strict JSON schema | Validate Input | Prepare Card Data | **Step 3: GPT-4** Generate vocabulary |
| Prepare Card Data | code | Parse GPT JSON, build DALL·E prompts | Generate Flashcards (GPT-4) | Split Into Cards | **Step 4: Prepare** Format for media gen |
| Split Into Cards | code | Fan-out: one item per card | Prepare Card Data | Loop Cards | **Step 5: Loop** Process each card |
| Loop Cards | splitInBatches | Iterate over cards and join at end | Split Into Cards; Merge Card Data | Generate Image (DALL-E); Create APKG Data | **Step 5: Loop** Process each card |
| Generate Image (DALL-E) | code | Call OpenAI Images API (DALL·E 3) and store base64 | Loop Cards | TTS Word (ElevenLabs) | **Step 6: DALL-E** Generate images |
| TTS Word (ElevenLabs) | elevenLabs | Generate word pronunciation audio | Generate Image (DALL-E) | Convert Word Audio | **Step 7: ElevenLabs** Word pronunciation |
| Convert Word Audio | code | Convert binary word audio to base64, merge into card | TTS Word (ElevenLabs) | TTS Example (ElevenLabs) | **Step 7: ElevenLabs** Word pronunciation |
| TTS Example (ElevenLabs) | elevenLabs | Generate example sentence audio | Convert Word Audio | Convert Example Audio | **Step 8: ElevenLabs** Example sentence audio |
| Convert Example Audio | code | Convert binary example audio to base64, merge into card | TTS Example (ElevenLabs) | Merge Card Data | **Step 8: ElevenLabs** Example sentence audio |
| Merge Card Data | code | Pass-through to continue loop | Convert Example Audio | Loop Cards | **Step 9: Merge** Combine all media |
| Create APKG Data | code | Build Anki model/deck/notes/cards + media mapping | Loop Cards (done output) | Build APKG ZIP | **Step 10: APKG** Build Anki package |
| Build APKG ZIP | code | Create SQLite DB + zip into .apkg; create sheetRows | Create APKG Data | Split For Sheets | **Step 10: APKG** Build Anki package |
| Split For Sheets | code | Fan-out rows for Sheets; stash apkg in _apkgData | Build APKG ZIP | Save to Google Sheets | **Step 11: Sheets** Backup data |
| Save to Google Sheets | googleSheets | Append card rows to spreadsheet | Split For Sheets | Aggregate Results | **Step 11: Sheets** Backup data |
| Aggregate Results | code | Re-aggregate and restore apkg payload; count saved rows | Save to Google Sheets | Prepare Attachment | **Step 12: Email** Send via Gmail |
| Prepare Attachment | code | Create binary attachment from base64 | Aggregate Results | Send Email (Gmail) | **Step 12: Email** Send via Gmail |
| Send Email (Gmail) | gmail | Email the .apkg to user | Prepare Attachment | Return Response | **Step 12: Email** Send via Gmail |
| Return Response | respondToWebhook | Return final JSON result | Send Email (Gmail) | — | **Step 13: Response** Return JSON |
| Sticky Note - NPM | stickyNote | Deployment requirements for external modules |  |  | ## 📦 Required npm Packages + `NODE_FUNCTION_ALLOW_EXTERNAL=jszip,sql.js` |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add a Form Trigger** node named **“Flashcard Form”**:
   - Add fields exactly as in the overview (Email, Topic/Word list, Native language, Target language, Number of flashcards, Difficulty, Image style, Reverse cards).
   - Set a friendly “submitted” message indicating processing time.
3. **Add a Code node** named **“Validate Input”** and paste logic that:
   - Reads form fields from `$input.first().json`
   - Clamps number of cards to 1–20
   - Generates `deckId`
   - Maps languages to ElevenLabs voice IDs
   - Outputs normalized JSON: `email, topic, nativeLanguage, targetLanguage, numCards, level, imageStyle, generateReverse, deckId, targetVoiceId, nativeVoiceId, createdAt`
4. **Connect**: Flashcard Form → Validate Input.

5. **Configure OpenAI credential** in n8n:
   - Create/Open **OpenAI API** credential (`openAiApi`) with your API key.
6. **Add an HTTP Request node** named **“Generate Flashcards (GPT-4)”**:
   - Method: POST
   - URL: `https://api.openai.com/v1/chat/completions`
   - Authentication: use the OpenAI predefined credential (`openAiApi`)
   - JSON body:
     - Model `gpt-4o`
     - Messages: system + user prompt including variables like `{{$json.numCards}}`, `{{$json.targetLanguage}}`, etc.
     - `response_format` as strict `json_schema` requiring deck and cards fields.
7. **Connect**: Validate Input → Generate Flashcards (GPT-4).

8. **Add a Code node** named **“Prepare Card Data”**:
   - Parse `response.choices[0].message.content` as JSON.
   - Build per-card `dallePrompt` using selected `imageStyle`.
   - Add placeholders for `imageBase64`, `wordAudioBase64`, `exampleAudioBase64`.
   - Output `{ deck, formData, voiceIds, generateReverse, metadata }`.
9. **Connect**: Generate Flashcards (GPT-4) → Prepare Card Data.

10. **Add a Code node** named **“Split Into Cards”**:
    - Return one n8n item per card (fan-out), carrying deck metadata to each.
11. **Connect**: Prepare Card Data → Split Into Cards.

12. **Add Split In Batches** node named **“Loop Cards”**:
    - Keep default batch size (commonly 1) unless you adjust downstream logic.
13. **Connect**: Split Into Cards → Loop Cards.

14. **Add a Code node** named **“Generate Image (DALL-E)”**:
    - Implement a POST to `https://api.openai.com/v1/images/generations`
    - Model `dall-e-3`, `response_format: b64_json`, use `card.dallePrompt`
    - **Important:** do not hardcode the key in production. Prefer:
      - Using an HTTP Request node with OpenAI credentials, or
      - Using environment variables in the code node.
15. **Connect**: Loop Cards (main output) → Generate Image (DALL-E).

16. **Install and configure ElevenLabs integration**:
    - Ensure `@elevenlabs/n8n-nodes-elevenlabs` is available in your n8n instance.
    - Create ElevenLabs credentials in n8n (API key).
17. **Add ElevenLabs node** named **“TTS Word (ElevenLabs)”**:
    - Resource: Text to Speech
    - Map text to the card’s word (e.g., `{{$json.card.word}}`)
    - Map voice to `{{$json.targetVoiceId}}`
    - Output should be **binary audio**.
    - **Note:** In the provided workflow this node is disabled; enable it if you want word audio.
18. **Connect**: Generate Image (DALL-E) → TTS Word (ElevenLabs).

19. **Add a Code node** named **“Convert Word Audio”**:
    - Read audio binary and set `card.wordAudioBase64`
    - Merge with the JSON from the image node.
20. **Connect**: TTS Word (ElevenLabs) → Convert Word Audio.

21. **Add ElevenLabs node** named **“TTS Example (ElevenLabs)”**:
    - Resource: Text to Speech
    - Text: `{{$json.card.example}}`
    - Voice: `{{$json.targetVoiceId}}` (or a different choice if desired)
    - Output: binary audio
22. **Connect**: Convert Word Audio → TTS Example (ElevenLabs).

23. **Add Code node** named **“Convert Example Audio”**:
    - Convert binary to base64 into `card.exampleAudioBase64`.
24. **Connect**: TTS Example (ElevenLabs) → Convert Example Audio.

25. **Add Code node** named **“Merge Card Data”** (pass-through).
26. **Connect**: Convert Example Audio → Merge Card Data → Loop Cards (to continue).
27. **Connect Loop completion output** (the “done” output) → **Create APKG Data**.

28. **Add Code node** named **“Create APKG Data”**:
    - Aggregate `$input.all()`
    - Build Anki model/deck/notes/cards objects
    - Create `mediaMapping` and numbered media files for Anki
29. **Add Code node** named **“Build APKG ZIP”**:
    - **Install packages** in runtime: `npm install jszip sql.js`
    - If Docker/self-hosted: set `NODE_FUNCTION_ALLOW_EXTERNAL=jszip,sql.js`
    - Build SQLite schema, export DB to `collection.anki21`
    - Create zip with `media` + numbered media files
    - Output base64 `.apkg` and `sheetRows`
30. **Connect**: Create APKG Data → Build APKG ZIP.

31. **Add Code node** named **“Split For Sheets”**:
    - Fan-out `sheetRows` and attach `_apkgData`.
32. **Connect**: Build APKG ZIP → Split For Sheets.

33. **Configure Google Sheets credentials**:
    - OAuth2 Google credential in n8n with Sheets scope.
    - Create a spreadsheet and a `Flashcards` sheet with matching headers (or adjust mapping).
34. **Add Google Sheets node** named **“Save to Google Sheets”**:
    - Operation: Append
    - Document ID: your spreadsheet ID (replace placeholder)
    - Sheet name: `Flashcards`
    - Map columns to JSON fields.
35. **Connect**: Split For Sheets → Save to Google Sheets.

36. **Add Code node** named **“Aggregate Results”**:
    - Use `$input.all()` to restore `_apkgData` and count saved rows.
37. **Connect**: Save to Google Sheets → Aggregate Results.

38. **Add Code node** named **“Prepare Attachment”**:
    - Create `binary.data` from `apkgBase64` with filename `apkgFileName`.
39. **Connect**: Aggregate Results → Prepare Attachment.

40. **Configure Gmail credentials**:
    - Gmail OAuth2 credential in n8n.
41. **Add Gmail node** named **“Send Email (Gmail)”**:
    - To: `{{$json.email}}`
    - Subject/body: include deck info
    - Attachments: include binary from previous node.
42. **Connect**: Prepare Attachment → Send Email (Gmail).

43. **Add Respond to Webhook node** named **“Return Response”**:
    - Respond with JSON summarizing success and deck details.
44. **Connect**: Send Email (Gmail) → Return Response.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install required packages: `npm install jszip sql.js` | Needed by **Build APKG ZIP** code node |
| Docker/self-hosted: `NODE_FUNCTION_ALLOW_EXTERNAL=jszip,sql.js` | Allows requiring external modules in n8n Code node |
| DALL·E node requires updating `OPENAI_API_KEY` | The workflow hardcodes a placeholder in **Generate Image (DALL-E)**; replace with a secure method |
| Replace `YOUR_SPREADSHEET_ID` in Google Sheets node | Required for **Save to Google Sheets** to work |
| ElevenLabs voice IDs are mapped automatically in Validate Input | See “Sticky Note - Config” |

