Create a human-like Evolution API WhatsApp agent with Redis, PostgreSQL and Gemini

https://n8nworkflows.xyz/workflows/create-a-human-like-evolution-api-whatsapp-agent-with-redis--postgresql-and-gemini-13407


# Create a human-like Evolution API WhatsApp agent with Redis, PostgreSQL and Gemini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Create a human-like Evolution API WhatsApp agent with Redis, PostgreSQL and Gemini

**Purpose & target use cases**
This workflow implements a production-grade WhatsApp agent powered by **Evolution API** (WhatsApp gateway), with:
- **Smart message buffering** (debounce rapid texts; group album media) using **Redis**
- **Hybrid memory**: fast “hot cache” in Redis + persistent “cold storage” in **PostgreSQL**
- **Two-stage AI pipeline** with Google Gemini models:
  1) **Context Refiner** compresses/retrieves only relevant facts from history  
  2) **Main AI Agent** generates the user-facing reply in the user’s language

**Logical blocks**
1.1 **Database setup (one-time)** – Creates `chat_history` table in PostgreSQL.  
1.2 **Parallel UX handling (sub-workflow)** – Marks messages as read and sends “typing…” presence without blocking main logic.  
1.3 **Webhook reception & global configuration** – Receives Evolution webhook payload and sets key runtime variables.  
1.4 **Message type routing** – Distinguishes text vs album/media; downloads media when needed.  
1.5 **Media normalization** – Converts downloaded media to binary, detects MIME category, extracts/AI-analyzes content into a unified `message`.  
1.6 **Smart input buffering (Redis)** – Debounces rapid texts and groups album media into one consolidated prompt.  
1.7 **Hybrid memory retrieval** – Reads chat history from Redis first; if empty, loads from PostgreSQL and seeds Redis.  
1.8 **Context refinement & response generation** – Refiner distills context; main agent answers; sends response via Evolution.  
1.9 **Cache lifecycle manager** – Keeps Redis “hot cache” alive per conversation window and clears it when idle.

---

## 2. Block-by-Block Analysis

### 2.1 Database Setup (one-time initialization)
**Overview:** Ensures PostgreSQL has the required `chat_history` table and index. Must be executed once manually before enabling the workflow.

**Nodes involved**
- When clicking ‘Execute workflow’
- Create Table chat_history

**Node details**

**When clicking ‘Execute workflow’**  
- **Type/role:** Manual Trigger; used only for setup runs.
- **Outputs:** Triggers `Create Table chat_history`.
- **Failure modes:** None typical; only runs when manually executed.

**Create Table chat_history**  
- **Type/role:** PostgreSQL node (executeQuery) to create schema objects.
- **Config choices:** Runs a SQL script creating:
  - `chat_history` with FK to `mentour_users(whatsapp_number)` and `ON DELETE CASCADE`
  - index `idx_chat_history_user_timestamp` on `(user_whatsapp_number, message_timestamp)`
- **Edge cases / failures:**
  - FK target table `mentour_users` must exist and have `whatsapp_number` column; otherwise SQL fails.
  - Permissions may prevent `CREATE TABLE/INDEX`.
- **Version notes:** Postgres node v2.6; ensure n8n Postgres credential configured.

---

### 2.2 Parallel UX Handling (sub-workflow, non-blocking)
**Overview:** Improves perceived latency by sending “typing…” and marking messages as read in parallel, via an asynchronous sub-workflow call.

**Nodes involved**
- Parallel Writing status (Execute Workflow)
- When Executed by Another Workflow (entry of sub-workflow)
- Open session
- Read Messages
- Wait
- Writting...

**Node details**

**Parallel Writing status**  
- **Type/role:** Execute Workflow; fires a separate workflow (“Mentour AI copy”) in **fire-and-forget** mode.
- **Key config:**
  - `waitForSubWorkflow: false` (does not block main workflow)
  - Inputs passed from webhook:
    - `ID`: `$('Webhook').item.json.body.data.key.id`
    - `Name`: `$('Webhook').item.json.body.instance`
    - `Number`: `$('Webhook').item.json.body.data.key.remoteJid`
  - `onError: continueRegularOutput` ensures main flow continues if sub-workflow fails.
- **Failure modes:** Wrong workflowId, missing permissions, or sub-workflow errors (ignored due to onError handling).

**When Executed by Another Workflow**  
- **Type/role:** Execute Workflow Trigger (sub-workflow entry point).
- **Inputs:** `Name`, `Number`, `ID` must be provided by caller.
- **Outputs:** Triggers `Open session`, `Read Messages`, and `Wait`.

**Open session**  
- **Type/role:** Evolution API node; changes bot status / opens bot session.
- **Key params:** `remoteJid`, `instanceName`, `evolutionBotId`, operation `evolution-bot`.
- **Failure modes:** Evolution credential invalid, instance not found, remoteJid format errors.

**Read Messages**  
- **Type/role:** Evolution API node; marks message as read.
- **Key params:** messageId, remoteJid, instanceName from inputs.
- **Failure modes:** messageId not found, permission issues.

**Wait**  
- **Type/role:** Wait node; holds execution (no amount configured in parameters here).
- **Connections:** After wait, triggers `Writting...`. (This pattern is commonly used to keep presence updates alive or sequence them.)
- **Failure modes:** Misconfiguration can stall sub-workflow indefinitely.

**Writting...**  
- **Type/role:** Evolution API node; sends presence “typing…”.
- **Key config:** `delay: 10200` (long typing presence), operation `send-presence`.
- **Failure modes:** Evolution API rate limits; long delays can be interrupted by workflow timeouts depending on n8n hosting settings.

---

### 2.3 Webhook & Global Configuration
**Overview:** Receives incoming WhatsApp events from Evolution API and sets global runtime controls for buffering and memory.

**Nodes involved**
- Webhook
- Global Variables

**Node details**

**Webhook**  
- **Type/role:** Webhook trigger (POST).
- **Path:** `a/event/messages-upsert`
- **Expected payload:** Evolution API `messages.upsert` structure, used heavily via expressions:
  - `body.data.key.remoteJid`
  - `body.data.key.id`
  - `body.data.messageType`, `body.data.message.conversation`, etc.
- **Failure modes:**
  - Payload shape differences (missing fields) will break expressions downstream.
  - If Evolution API secret/api key validation is expected, it is not implemented here; consider checking `apikey`.

**Global Variables**  
- **Type/role:** Set node; central config values used across flow.
- **Assignments:**
  - `number`: extracted from `remoteJid.split('@')[0]` (e.g., `573209...`)
  - `wait_buffer`: `"5"` seconds debounce for rapid messages
  - `wait_conversation`: `"300"` seconds hot-cache “conversation active” window
  - `max_chat_history`: `"40"` maximum messages fetched from DB for cold start
- **Edge cases:**
  - `remoteJid` without `@` will break `.split('@')[0]`.
  - Stored values are strings; nodes expecting numeric/date operations must handle conversions (some expressions use them as numbers).

---

### 2.4 Message Type Routing (text vs album/media)
**Overview:** Determines whether the incoming event is a text message, an album message, or other. Text goes to debounce buffer; album/media flow initializes album grouping and downloads media.

**Nodes involved**
- Message Type
- get_message (text)
- Push to Buffer (AlbumGroup)
- Descargar Media

**Node details**

**Message Type**  
- **Type/role:** Switch node.
- **Rules:**
  - Output **Texto** if `conversation` is not empty.
  - Output **AlbumMessage** if `messageType == 'albumMessage'`.
  - Fallback output: `extra`.
- **Edge cases:** Messages like buttons, list replies, reactions, etc., may not populate `conversation`, sending them to fallback (`extra`) and potentially into the media branch due to wiring.

**get_message (text)**  
- **Type/role:** Set node; normalizes text into `message`.
- **Value:** `$('Webhook').item.json.body.data.message.conversation`
- **Downstream:** Feeds into `Push to Buffer` (debounce buffer).

**Push to Buffer (AlbumGroup)**  
- **Type/role:** Redis list push; stores expected media count for album group.
- **List key:** `Media_{{ number }}`
- **MessageData:** total expected images + videos as string.
- **Edge cases:** If expected counts missing, result becomes `NaN` string and completion logic breaks.

**Descargar Media**  
- **Type/role:** Evolution API get media as base64.
- **Key params:** `messageId` = webhook key id; `instanceName` from webhook instance.
- **Failure modes:** Media expired/unavailable; Evolution API auth; large media may hit payload/memory limits.

---

### 2.5 Media Normalization (download → file → type detection → extraction/analysis)
**Overview:** Converts Evolution media base64 into binary, detects MIME category, then extracts text (CSV/PDF/etc.) or uses Gemini to analyze image/audio/video/document, producing a unified `message` describing the media + extracted content.

**Nodes involved**
- Convert to File
- Get Mime Type
- Media Type
- Analyze Image → get_message (Image)
- Analyze audio → get_message (Audio)
- Analyze video → get_message (Video)
- Extract from CSV → Aggregate → Normalize CSV
- HTML Extract Generic1 → Normalize HTML
- Extract from ICS → Normalize ICS
- Extract from JSON → Normalize JSON
- Extract from ODS → Get ODS data → Normalize ODS
- Extract from PDF → Text? → Normalize PDF / (AI branch) Merge → Analyze document → Normalize PDF (AI)
- Extract from RTF → Get RTF data → Normalize RTF
- Extract from File → Normalize text file
- Extract from XML → Normalize XML
- Extract from XLSX → Get RTF data1 → Normalize XLSX
- get_error_message
- get_message (File message)

**Node details (key nodes)**

**Convert to File**  
- **Type/role:** ConvertToFile; converts base64 to binary.
- **Operation:** `toBinary`
- **Source property:** `data.base64` (from `Descargar Media`)
- **Failure modes:** base64 missing/corrupted; huge files may exceed n8n limits.

**Get Mime Type** (Code)  
- **Type/role:** Code node; classifies MIME into `fileTypeCategory`.
- **Key logic:** reads `$('Descargar Media').first().json.data.mimetype` and sets category among:
  `image, audio, video, pdf, spreadsheet, document, text, csv, html, rtf, json, xml, ics, other`
- **Important edge case / bug risk:**
  - It uses `'text file'` in the switch node later, but this code outputs `text` (not `text file`). This mismatch can route plain text to fallback error.
- **Failure modes:** mimetype path differs (comment says verify); then category becomes `other`.

**Media Type** (Switch)  
- **Type/role:** routes by `fileTypeCategory`.
- **Outputs:** image/audio/video/csv/html/ics/json/ods/pdf/rtf/text file/xml/spreadsheet + fallback `extra`.
- **Edge cases:** mismatch above (`text` vs `text file`) and `ods` output key expects `fileTypeCategory == 'ods'` while code sets `spreadsheet` for ODS mimetype. So ODS will go to `spreadsheet` branch, not `ods`.

**Gemini Analyze nodes**  
- **Analyze Image** (Gemini, resource=image, binary) → `get_message (Image)` extracts `content.parts[0].text`
- **Analyze audio** (Gemini, resource=audio, binary) → `get_message (Audio)` extracts candidate text
- **Analyze video** (Gemini, resource=video, binary) → `get_message (Video)` extracts candidate text
- **Analyze document** (Gemini, resource=document, binary) used as fallback for PDFs where text extraction fails
- **Failure modes:** invalid Google credentials; file too large; unsupported format; model limits/timeouts.

**PDF text extraction + AI fallback**
- **Extract from PDF** → **Text?** checks if `$json.text` is not empty
  - If yes: **Normalize PDF** sets `data = $json.text`
  - If no: goes through **Merge (chooseBranch)** then **Analyze document** then **Normalize PDF (AI)** uses `$json.content.parts[0].text`
- **Merge**
  - **Mode:** chooseBranch, `useDataOfInput: 2` (forces using input 2’s data)
  - **Edge case:** Miswiring can cause the wrong branch to be used; ensure PDF binary is present for Analyze document.

**Structured extractors (CSV/HTML/ICS/JSON/XML/XLSX/ODS/RTF)**
- Use ExtractFromFile nodes then normalize to a common `data` string.
- CSV uses **Aggregate** (`aggregateAllItemData`) then **Normalize CSV**.
- ODS/RTF/XLSX go through Code nodes that wrap first item json into `{ data: firstItem.json }`, then normalize.

**get_error_message**
- **Role:** Provides a default message when file type unsupported.
- **Output:** `message = "It was not possible to process the file.File type not supported."`
- **Downstream:** Still goes into input buffering via `Normalize input`, so user gets a response even if unsupported.

**get_message (File message)**  
- **Type/role:** Set node; final “media message” normalization into `message`.
- **Value includes:**
  - caption (if any)
  - fileName
  - `fileTypeCategory`
  - extracted/AI-produced `data`
- **Edge cases:** `fileName` may be missing; expressions referencing it may produce `undefined`.

---

### 2.6 Smart Input Buffering (Redis): debounce texts + group album media
**Overview:** Prevents fragmented responses by consolidating rapid user messages. Also waits for all album items before processing.

**Nodes involved**
- Normalize input
- Check AlbumGroup
- Album?
- Push Media to Buffer
- Get Album From Buffer
- Completed?
- Delete Media Buffer
- Normalize MediaGroup Buffer
- Push to Buffer
- Get From Buffer
- Buffer Route
- Wait For User Other Fast Message
- Delete Buffer
- Normalize Buffer
- No Operation, do nothing

**Node details**

**Normalize input**  
- **Type/role:** Set; ensures downstream reads `message` consistently.
- **Value:** `message = $json.message`
- **Edge cases:** If upstream produced `chatInput` not `message`, later `Get Message` handles that; but here it assumes `message`.

**Album buffering**
- **Check AlbumGroup** (Redis get `Media_{{number}}`)
- **Album?** (IF) checks if `$json.propertyName` array is not empty (album group exists)
  - True → **Push Media to Buffer**
  - False → **Push to Buffer** (treat as normal text)
- **Push Media to Buffer** pushes the normalized `message` into `Media_{{number}}`
- **Get Album From Buffer** reads the entire list
- **Completed?** verifies list length matches expected count:
  - compares `propertyName.length - 1` to `parseInt(propertyName[0])`
- If complete:
  - **Delete Media Buffer**
  - **Normalize MediaGroup Buffer** concatenates items after index 0 into one `message`
- Else:
  - **No Operation, do nothing** (drops until the album is complete)
- **Edge cases:**
  - If expected count entry is missing/corrupt, completion check fails.
  - Concurrent albums from same user number can collide on the same Redis key.

**Text debounce buffering**
- **Push to Buffer** (Redis push to list key `{{number}}`):
  - stores JSON string with:
    - `message` (text or media-constructed)
    - `sessionID` (webhook key id)
    - `date_time` from messageTimestamp converted `toDateTime('s')`
- **Get From Buffer** (Redis get list into `message`)
- **Buffer Route** (Switch) decides:
  - **Ignore:** last sessionID != current webhook id (prevents old executions processing newer input)
  - **Continue:** last date_time is before now minus `wait_buffer` seconds
  - **Wait (fallback):** not old enough yet
- **Wait For User Other Fast Message** waits `wait_buffer` seconds then loops back to `Get From Buffer`
- **Delete Buffer** clears the buffer key once ready
- **Normalize Buffer** concatenates all buffered messages into one string:  
  `message = $json.message.map(m => JSON.parse(m).message).join('\n')`
- **No Operation, do nothing** used as a sink when ignored.

**Failure modes / operational risks**
- Redis list growth if Delete Buffer is never reached (workflow interruptions).
- Date parsing: if `messageTimestamp` missing/unexpected, comparisons in Buffer Route fail.
- `JSON.parse($json.message.last())` requires that list contains valid JSON strings.

---

### 2.7 Hybrid Memory Retrieval (Redis hot cache → PostgreSQL cold storage)
**Overview:** Retrieves recent conversation context. If Redis cache exists, use it. Otherwise query PostgreSQL and seed Redis.

**Nodes involved**
- Get Message
- Get chat_history1
- chat_history In Buffer?
- Get chat_history From DataBase
- Aggregate all Messges
- chat_history in DataBase?
- Push to chat_history
- Push first message to chat_history
- Get chat_history

**Node details**

**Get Message**  
- **Type/role:** Set; final normalization before memory fetch.
- **Value:** `message = $json.message || $json.chatInput`
- **Why:** supports both buffered message (`message`) and other sources (`chatInput`).

**Get chat_history1** (Redis get)  
- **Key:** `Chats_{{number}}`
- **Output:** `propertyName` array (Redis node uses `propertyName` naming).

**chat_history In Buffer?** (IF)  
- Checks `propertyName[0]` not empty:
  - True → go directly to `Get chat_history`
  - False → go to PostgreSQL fetch

**Get chat_history From DataBase** (Postgres select)  
- **Table:** `public.chat_history`
- **Where:** `user_whatsapp_number = number`
- **Limit:** `max_chat_history`
- **Output columns:** `message_text`
- **Options:** `retryOnFail: true`, `alwaysOutputData: true` (prevents hard-stop on empty results)
- **Failure modes:** DB connectivity, missing table, FK constraints earlier not relevant for select.

**Aggregate all Messges** (Aggregate)  
- Aggregates all `message_text` rows into an array `message_text`.

**chat_history in DataBase?** (IF)  
- Checks aggregated `message_text` array not empty:
  - True → **Push to chat_history** (seed Redis with joined history)
  - False → **Push first message to chat_history** (seed Redis with `.` placeholder)

**Push to chat_history** (Redis push)  
- **Key:** `Chats_{{number}}`
- **Value:** `$json.message_text.join('\n')` (note: joins DB rows; each row already contains “User/Agent” lines)

**Push first message to chat_history**  
- Pushes a single dot (`.`) to initialize cache for new users.

**Get chat_history**  
- **Type/role:** Set; slices last N and joins into `history`.
- **Value:** `history = $json.propertyName.slice(-max_chat_history).join('\n')`
- **Downstream:** goes to Context Refiner.

**Edge cases**
- Redis `Chats_{{number}}` is treated as a list of “blocks” (each push is a multi-line string), not as one entry per message; token trimming by slice may be coarse.
- Placeholder `.` can leak into context; refiner should ignore but not guaranteed.

---

### 2.8 Context Refinement & Response Generation
**Overview:** Uses a fast Gemini model to extract only relevant facts from history, then a stronger/primary model to generate the final answer, keeping the user’s language and tone.

**Nodes involved**
- Google Gemini Chat Model
- Context Refiner
- Google Gemini Chat Model1
- AI Agent
- Agent Response
- Send Response

**Node details**

**Google Gemini Chat Model**  
- **Type/role:** LangChain Chat Model (Gemini).
- **Model:** `models/gemini-flash-lite-latest`
- **Temperature:** 0 (deterministic)
- **Connected to:** Context Refiner via `ai_languageModel`.
- **Failure modes:** credential, model availability.

**Context Refiner** (chainLlm)  
- **Role:** Backend context processor; outputs only facts or `NO_CONTEXT`.
- **Input text template:**
  - `<Chat_History>{{ $json.history }}</Chat_History>`
  - `<Current_User_Input>{{ $('Get Message').item.json.message }}</Current_User_Input>`
- **System message:** Strict extraction rules (greeting → `NO_CONTEXT`, link file name to extracted data, resolve pronouns).
- **Output:** placed as `$json.output` (passed forward as `$json.text` in AI Agent prompt).
- **Failure modes:** If history is huge or malformed; model may not follow strict rules (mitigate with temperature 0).

**Google Gemini Chat Model1**  
- **Type/role:** LangChain Chat Model for main agent.
- **Model:** `models/gemini-3-flash-preview`
- **Connected to:** AI Agent via `ai_languageModel`.

**AI Agent** (LangChain agent)  
- **Prompt structure:**
  - `<RETRIEVED_CONTEXT>{{ $json.text }}</RETRIEVED_CONTEXT>`
  - `<USER_MESSAGE>{{ $('Get Message').item.json.message }}</USER_MESSAGE>`
- **System behavior:**
  - Must reply in the same language as user message
  - If `RETRIEVED_CONTEXT` is `NO_CONTEXT`, ignore it
  - Integrate extracted file content naturally (do not mention memory/context)
- **Output:** agent output becomes `$json.output`.

**Agent Response**  
- **Type/role:** Set; maps `reponse = $json.output` (note spelling `reponse`).
- **Edge case:** If agent output field differs, send will be empty.

**Send Response** (Evolution API)  
- **Resource:** `messages-api`
- **remoteJid:** `number` (note: Evolution typically expects full JID like `...@s.whatsapp.net`; here only number is used—verify Evolution API behavior)
- **instanceName:** hard-coded `"Pruebas"`
- **messageText:** `{{ $json.reponse }}`
- **Failure modes:** wrong instanceName, wrong remoteJid format, Evolution credential errors.

---

### 2.9 Persistence + Cache Lifecycle Manager (Redis TTL window)
**Overview:** Saves each interaction to PostgreSQL and Redis, and keeps Redis “hot cache” active for `wait_conversation` seconds after last message. Clears buffers when conversation is inactive.

**Nodes involved**
- Push message chat_history (User And Agent)
- Add Messages to chat_history (User And Agent)
- Push to Buffer (chat_history)
- Get From Buffer(chat_history)
- Buffer (chat_history) Route
- Wait For end of conversation
- Delete Buffer (chat_history)
- Delete Buffer (chat_history)2
- No Operation, do nothing1
- Delete Buffer (chat_history)  *(and also: Delete Buffer (chat_history)2)*
- (Also present but not connected into this lifecycle: Delete Buffer (chat_history) node exists and is used)

**Node details**

**Push message chat_history (User And Agent)** (Redis push)  
- **Key:** `Chats_{{number}}`
- **Value:**  
  `User: <message>\nAgent: <response>`
- **Downstream:** inserts same content into Postgres.

**Add Messages to chat_history (User And Agent)** (Postgres insert)  
- Inserts row into `chat_history` with:
  - `user_whatsapp_number = number`
  - `message_text = "User: ...\nAgent: ..."`
- **Failure modes:** FK constraint if `mentour_users` doesn’t include the number; DB unavailable.

**Push to Buffer (chat_history)** (Redis push)  
- **Key:** `Chats_buffer{{number}}`
- Pushes JSON containing sessionID and date_time (timestamp).
- Purpose: manage “conversation active” timer across messages.

**Get From Buffer(chat_history)** (Redis get)  
- Reads `Chats_buffer{{number}}` list into `message` property for routing.

**Buffer (chat_history) Route** (Switch)  
- Same pattern as message debounce:
  - Ignore if last sessionID != current
  - Continue if last date_time < now - wait_conversation
  - Wait otherwise
- **Outcome:**
  - Continue → delete caches
  - Wait → wait again then re-check

**Wait For end of conversation**  
- Waits `wait_conversation` seconds, then re-checks buffer.

**Delete Buffer (chat_history)** → **Delete Buffer (chat_history)2**  
- Deletes `Chats_{{number}}` (hot cache history)
- Then deletes `Chats_buffer{{number}}` (conversation timer buffer)

**No Operation, do nothing1**  
- Sink for ignored executions.

**Edge cases**
- If workflow execution is terminated, cleanup may not run; Redis may retain stale keys.
- The “sessionID mismatch ignore” is essential to avoid older executions clearing active sessions.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note1 | stickyNote | Documentation/branding |  |  | ## Advanced Evolution API Agent with Redis Buffering & Context Refinement (includes setup checklist; Community Nodes install; credentials; DB init; Evolution instanceName/apikey; global variables; models) |
| When clicking ‘Execute workflow’ | manualTrigger | One-time DB init trigger |  | Create Table chat_history | ## 1. Database Setup… |
| Create Table chat_history | postgres | Create `chat_history` schema objects | When clicking ‘Execute workflow’ |  | ## 1. Database Setup… |
| Sticky Note | stickyNote | Block comment |  |  | ## 1. Database Setup… |
| When Executed by Another Workflow | executeWorkflowTrigger | Sub-workflow entry for UX branch |  | Open session; Read Messages; Wait | ## 2. Parallel UX Handling… |
| Open session | evolutionApi | Evolution bot status/session handling | When Executed by Another Workflow |  | ## 2. Parallel UX Handling… |
| Read Messages | evolutionApi | Mark message as read | When Executed by Another Workflow |  | ## 2. Parallel UX Handling… |
| Wait | wait | Timing/sequence in sub-workflow | When Executed by Another Workflow | Writting... | ## 2. Parallel UX Handling… |
| Writting... | evolutionApi | Send “typing/presence” | Wait |  | ## 2. Parallel UX Handling… |
| Sticky Note3 | stickyNote | Block comment |  |  | ## 2. Parallel UX Handling… |
| Webhook | webhook | Entry point for WhatsApp events |  | Global Variables; Parallel Writing status | ## 3. Webhook & Configuration… |
| Global Variables | set | Set `number`, wait times, history limit | Webhook | Message Type | ## 3. Webhook & Configuration… + ## ⚠️ Core Configuration… |
| Sticky Note4 | stickyNote | Block comment |  |  | ## 3. Webhook & Configuration… |
| Sticky Note2 | stickyNote | Config guidance |  |  | ## ⚠️ Core Configuration… |
| Parallel Writing status | executeWorkflow | Launch UX sub-workflow asynchronously | Webhook |  | ## 2. Parallel UX Handling… |
| Message Type | switch | Route text vs album | Global Variables | get_message (text); Push to Buffer (AlbumGroup); Descargar Media | ## 5. Smart Input Buffering… |
| get_message (text) | set | Normalize text into `message` | Message Type | Push to Buffer | ## 5. Smart Input Buffering… |
| Push to Buffer (AlbumGroup) | redis | Store expected album count | Message Type |  | ## 5. Smart Input Buffering… |
| Descargar Media | evolutionApi | Download media base64 | Message Type | Convert to File | ## 4. Media Normalization… |
| Convert to File | convertToFile | base64 → binary | Descargar Media | Get Mime Type | ## 4. Media Normalization… |
| Get Mime Type | code | Classify MIME into category | Convert to File | Media Type | ## 4. Media Normalization… |
| Media Type | switch | Route by `fileTypeCategory` | Get Mime Type | Analyze Image; Analyze audio; Analyze video; Extractors; get_error_message | ## 4. Media Normalization… |
| Analyze Image | googleGemini (langchain) | Image analysis | Media Type | get_message (Image) | ## 4. Media Normalization… |
| get_message (Image) | set | Map image analysis to `data` | Analyze Image | get_message (File message) | ## 4. Media Normalization… |
| Analyze audio | googleGemini (langchain) | Audio analysis | Media Type | get_message (Audio) | ## 4. Media Normalization… |
| get_message (Audio) | set | Map audio analysis to `data` | Analyze audio | get_message (File message) | ## 4. Media Normalization… |
| Analyze video | googleGemini (langchain) | Video analysis | Media Type | get_message (Video) | ## 4. Media Normalization… |
| get_message (Video) | set | Map video analysis to `data` | Analyze video | get_message (File message) | ## 4. Media Normalization… |
| Extract from CSV | extractFromFile | Parse CSV | Media Type | Aggregate | ## 4. Media Normalization… |
| Aggregate | aggregate | Aggregate CSV parsed output | Extract from CSV | Normalize CSV | ## 4. Media Normalization… |
| Normalize CSV | set | Normalize extracted CSV data | Aggregate | get_message (File message) | ## 4. Media Normalization… |
| HTML Extract Generic1 | html | Extract HTML title/meta/body | Media Type | Normalize HTML | ## 4. Media Normalization… |
| Normalize HTML | set | Normalize HTML extraction | HTML Extract Generic1 | get_message (File message) | ## 4. Media Normalization… |
| Extract from ICS | extractFromFile | Parse ICS | Media Type | Normalize ICS | ## 4. Media Normalization… |
| Normalize ICS | set | Normalize ICS extraction | Extract from ICS | get_message (File message) | ## 4. Media Normalization… |
| Extract from JSON | extractFromFile | Parse JSON | Media Type | Normalize JSON | ## 4. Media Normalization… |
| Normalize JSON | set | Normalize JSON extraction | Extract from JSON | get_message (File message) | ## 4. Media Normalization… |
| Extract from ODS | extractFromFile | Parse ODS | Media Type | Get ODS data | ## 4. Media Normalization… |
| Get ODS data | code | Wrap ODS into `data` | Extract from ODS | Normalize ODS | ## 4. Media Normalization… |
| Normalize ODS | set | Normalize ODS to string | Get ODS data | get_message (File message) | ## 4. Media Normalization… |
| Extract from PDF | extractFromFile | Extract PDF text | Media Type | Text? | ## 4. Media Normalization… |
| Text? | if | Decide PDF text vs AI analysis | Extract from PDF | Normalize PDF; Merge | ## 4. Media Normalization… |
| Normalize PDF | set | Normalize PDF extracted text | Text? | get_message (File message) | ## 4. Media Normalization… |
| Merge | merge | Choose branch for PDF AI fallback | Text?; Media Type(pdf second link) | Analyze document | ## 4. Media Normalization… |
| Analyze document | googleGemini (langchain) | AI document analysis (PDF fallback) | Merge | Normalize PDF (AI) | ## 4. Media Normalization… |
| Normalize PDF (AI) | set | Normalize AI PDF output | Analyze document | get_message (File message) | ## 4. Media Normalization… |
| Extract from RTF | extractFromFile | Parse RTF | Media Type | Get RTF data | ## 4. Media Normalization… |
| Get RTF data | code | Wrap RTF into `data` | Extract from RTF | Normalize RTF | ## 4. Media Normalization… |
| Normalize RTF | set | Normalize RTF to string | Get RTF data | get_message (File message) | ## 4. Media Normalization… |
| Extract from File | extractFromFile | Extract plain text | Media Type | Normalize text file | ## 4. Media Normalization… |
| Normalize text file | set | Normalize text file to string | Extract from File | get_message (File message) | ## 4. Media Normalization… |
| Extract from XML | extractFromFile | Parse XML | Media Type | Normalize XML | ## 4. Media Normalization… |
| Normalize XML | set | Normalize XML to string | Extract from XML | get_message (File message) | ## 4. Media Normalization… |
| Extract from XLSX | extractFromFile | Parse XLSX | Media Type | Get RTF data1 | ## 4. Media Normalization… |
| Get RTF data1 | code | Wrap XLSX into `data` | Extract from XLSX | Normalize XLSX | ## 4. Media Normalization… |
| Normalize XLSX | set | Normalize XLSX to string | Get RTF data1 | get_message (File message) | ## 4. Media Normalization… |
| get_error_message | set | Unsupported file fallback message | Media Type (fallback) | Normalize input | ## 4. Media Normalization… |
| get_message (File message) | set | Build unified file summary into `message` | Normalize* / get_message (Image/Audio/Video) | Normalize input | ## 4. Media Normalization… |
| Sticky Note5 | stickyNote | Block comment |  |  | ## 4. Media Normalization… |
| Normalize input | set | Ensure `message` field for buffering | get_message (File message) / get_error_message | Check AlbumGroup | ## 5. Smart Input Buffering… |
| Check AlbumGroup | redis | Check album buffer existence | Normalize input | Album? | ## 5. Smart Input Buffering… |
| Album? | if | Decide album buffering vs normal | Check AlbumGroup | Push Media to Buffer; Push to Buffer | ## 5. Smart Input Buffering… |
| Push Media to Buffer | redis | Add media message to album list | Album? | Get Album From Buffer | ## 5. Smart Input Buffering… |
| Get Album From Buffer | redis | Read album list | Push Media to Buffer | Completed? | ## 5. Smart Input Buffering… |
| Completed? | if | Verify album complete | Get Album From Buffer | Delete Media Buffer; No Operation, do nothing | ## 5. Smart Input Buffering… |
| Delete Media Buffer | redis | Delete album key | Completed? | Normalize MediaGroup Buffer | ## 5. Smart Input Buffering… |
| Normalize MediaGroup Buffer | set | Join album items into one `message` | Delete Media Buffer | Get Message | ## 5. Smart Input Buffering… |
| Push to Buffer | redis | Debounce buffer push (text/media) | get_message (text) / Album? false | Get From Buffer | ## 5. Smart Input Buffering… |
| Get From Buffer | redis | Read debounce buffer | Push to Buffer | Buffer Route | ## 5. Smart Input Buffering… |
| Buffer Route | switch | Ignore/Continue/Wait for debounce | Get From Buffer | No Operation, do nothing; Delete Buffer; Wait For User Other Fast Message | ## 5. Smart Input Buffering… |
| Wait For User Other Fast Message | wait | Wait debounce window then re-check | Buffer Route (Wait) | Get From Buffer | ## 5. Smart Input Buffering… |
| Delete Buffer | redis | Clear debounce buffer | Buffer Route (Continue) | Normalize Buffer | ## 5. Smart Input Buffering… |
| Normalize Buffer | set | Join buffered messages into one | Delete Buffer | Get Message | ## 5. Smart Input Buffering… |
| No Operation, do nothing | noOp | Sink (album not complete / ignored) | Completed? false / Buffer Route ignore |  | ## 5. Smart Input Buffering… |
| Sticky Note6 | stickyNote | Block comment |  |  | ## 5. Smart Input Buffering… |
| Get Message | set | Final user message normalization | Normalize Buffer / Normalize MediaGroup Buffer | Get chat_history1 | ## 6. Hybrid Memory Retrieval… |
| Get chat_history1 | redis | Load hot cache history | Get Message | chat_history In Buffer? | ## 6. Hybrid Memory Retrieval… |
| chat_history In Buffer? | if | Redis cache present? | Get chat_history1 | Get chat_history; Get chat_history From DataBase | ## 6. Hybrid Memory Retrieval… |
| Get chat_history From DataBase | postgres | Load history from Postgres | chat_history In Buffer? (false) | Aggregate all Messges | ## 6. Hybrid Memory Retrieval… |
| Aggregate all Messges | aggregate | Collect DB rows into array | Get chat_history From DataBase | chat_history in DataBase? | ## 6. Hybrid Memory Retrieval… |
| chat_history in DataBase? | if | DB returned rows? | Aggregate all Messges | Push to chat_history; Push first message to chat_history | ## 6. Hybrid Memory Retrieval… |
| Push to chat_history | redis | Seed hot cache from DB | chat_history in DataBase? true | Get chat_history | ## 6. Hybrid Memory Retrieval… |
| Push first message to chat_history | redis | Initialize hot cache with “.” | chat_history in DataBase? false | Get chat_history | ## 6. Hybrid Memory Retrieval… |
| Get chat_history | set | Build `history` string | Push to chat_history / Push first message… / chat_history In Buffer? true | Context Refiner | ## 6. Hybrid Memory Retrieval… |
| Sticky Note7 | stickyNote | Block comment |  |  | ## 6. Hybrid Memory Retrieval… |
| Google Gemini Chat Model | lmChatGoogleGemini | Model for Context Refiner |  | Context Refiner (ai_languageModel) | ## 7. Context Refinement & Generation… |
| Context Refiner | chainLlm | Extract only relevant facts | Get chat_history | AI Agent | ## 7. Context Refinement & Generation… |
| Google Gemini Chat Model1 | lmChatGoogleGemini | Model for AI Agent |  | AI Agent (ai_languageModel) | ## 7. Context Refinement & Generation… |
| AI Agent | langchain.agent | Generate final reply | Context Refiner | Agent Response | ## 7. Context Refinement & Generation… |
| Agent Response | set | Map agent output to `reponse` | AI Agent | Send Response | ## 7. Context Refinement & Generation… |
| Send Response | evolutionApi | Send WhatsApp reply | Agent Response | Push message chat_history (User And Agent) | ## 7. Context Refinement & Generation… |
| Push message chat_history (User And Agent) | redis | Append exchange to Redis history | Send Response | Add Messages to chat_history (User And Agent) | ## 7. Context Refinement & Generation… |
| Add Messages to chat_history (User And Agent) | postgres | Persist exchange to DB | Push message chat_history… | Push to Buffer (chat_history) | ## 7. Context Refinement & Generation… |
| Sticky Note8 | stickyNote | Block comment |  |  | ## 7. Context Refinement & Generation… |
| Push to Buffer (chat_history) | redis | Update conversation activity buffer | Add Messages to chat_history… | Get From Buffer(chat_history) | ## 8. Cache Lifecycle Manager… |
| Get From Buffer(chat_history) | redis | Read activity buffer | Push to Buffer (chat_history); Wait For end of conversation | Buffer (chat_history) Route | ## 8. Cache Lifecycle Manager… |
| Buffer (chat_history) Route | switch | Decide to wait/cleanup | Get From Buffer(chat_history) | No Operation, do nothing1; Delete Buffer (chat_history); Wait For end of conversation | ## 8. Cache Lifecycle Manager… |
| Wait For end of conversation | wait | Wait TTL then re-check | Buffer (chat_history) Route (Wait) | Get From Buffer(chat_history) | ## 8. Cache Lifecycle Manager… |
| Delete Buffer (chat_history) | redis | Delete hot cache history | Buffer (chat_history) Route (Continue) | Delete Buffer (chat_history)2 | ## 8. Cache Lifecycle Manager… |
| Delete Buffer (chat_history)2 | redis | Delete activity buffer | Delete Buffer (chat_history) |  | ## 8. Cache Lifecycle Manager… |
| No Operation, do nothing1 | noOp | Sink for ignored executions | Buffer (chat_history) Route (Ignore) |  | ## 8. Cache Lifecycle Manager… |
| Sticky Note9 | stickyNote | Block comment |  |  | ## 8. Cache Lifecycle Manager… |
| Delete Buffer (chat_history) (unused extra node name duplication) | redis | (Already documented above) |  |  | ## 8. Cache Lifecycle Manager… |
| fd1118a6 Delete Buffer (chat_history)2 (already) | redis | (Already documented above) |  |  | ## 8. Cache Lifecycle Manager… |
| Get chat_history (set) etc. | set | (Already documented above) |  |  |  |
| Sticky Note15 | stickyNote | Contact/help note |  |  | ## 💡 Need Assistance? Email: [johnsilva11031@gmail.com](mailto:johnsilva11031@gmail.com) • LinkedIn: [John Alejandro Silva Rodríguez](https://www.linkedin.com/in/john-alejandro-silva-rodriguez-48093526b/) |

> Note: Some nodes in the JSON exist but are effectively “documentation-only” (sticky notes) or are present with naming overlaps; the table includes all functional nodes and preserves sticky note content where applicable.

---

## 4. Reproducing the Workflow from Scratch

### 4.1 Prerequisites
1. **Install community node:** `n8n-nodes-evolution-api`  
   - n8n: *Settings → Community Nodes* → install package.
2. Prepare credentials in n8n:
   - **Evolution API** credential (host/apikey/instance as required by node)
   - **Redis** credential (host/port/password/db)
   - **PostgreSQL** credential
   - **Google PaLM / Gemini** credential (GooglePalmApi in n8n)

### 4.2 Create the PostgreSQL schema (one-time)
1. Add **Manual Trigger** node named **When clicking ‘Execute workflow’**.
2. Add **Postgres** node named **Create Table chat_history**:
   - Operation: **Execute Query**
   - Paste SQL (create table + index). Ensure referenced table `mentour_users(whatsapp_number)` exists or adjust FK.
3. Connect Manual Trigger → Create Table chat_history.
4. Run once manually.

### 4.3 Create the main webhook entry
5. Add **Webhook** node:
   - HTTP Method: **POST**
   - Path: `a/event/messages-upsert`
6. Add **Set** node **Global Variables**:
   - `number` = `={{ $('Webhook').item.json.body.data.key.remoteJid.split('@')[0] }}`
   - `wait_buffer` = `5`
   - `wait_conversation` = `300`
   - `max_chat_history` = `40`
7. Connect Webhook → Global Variables.

### 4.4 Add the parallel UX sub-workflow call
8. Add **Execute Workflow** node **Parallel Writing status**:
   - Mode: **each**
   - Options: `waitForSubWorkflow = false`
   - On Error: **continueRegularOutput**
   - Select sub-workflow (create it in 4.5)
   - Map inputs:
     - `ID` = webhook key id
     - `Name` = webhook instance
     - `Number` = webhook remoteJid
9. Connect Webhook → Parallel Writing status (in parallel to Global Variables).

### 4.5 Build the sub-workflow “Mentour AI copy” (UX)
10. Create a new workflow (name arbitrary) and add **Execute Workflow Trigger**:
    - Define inputs: `Name`, `Number`, `ID`
11. Add Evolution API node **Open session**:
    - resource: integrations-api
    - operation: evolution-bot / change status (as in your Evolution node options)
    - remoteJid = `={{ $json.Number }}`
    - instanceName = `={{ $json.Name }}`
    - evolutionBotId = `={{ $json.ID }}`
12. Add Evolution API node **Read Messages**:
    - operation: read-messages
    - messageId = `={{ $json.ID }}`
    - remoteJid = `={{ $json.Number }}`
    - instanceName = `={{ $json.Name }}`
13. Add **Wait** node **Wait** (optional but matching template).
14. Add Evolution API node **Writting...**:
    - operation: send-presence
    - remoteJid = `={{ $('When Executed by Another Workflow').item.json.Number }}`
    - instanceName = `={{ $('When Executed by Another Workflow').item.json.Name }}`
    - delay = `10200`
15. Connect Execute Workflow Trigger → Open session, Read Messages, Wait; then Wait → Writting....

### 4.6 Route message types (text vs album/media)
16. Back in main workflow, add **Switch** node **Message Type**:
    - Rule 1 (Texto): `conversation` not empty
    - Rule 2 (AlbumMessage): `messageType == 'albumMessage'`
    - Fallback: `extra`
17. Connect Global Variables → Message Type.
18. Add **Set** node **get_message (text)**:
    - `message = {{ webhook body.data.message.conversation }}`
19. Connect Message Type (Texto) → get_message (text).

### 4.7 Text debounce buffer (Redis)
20. Add **Redis** node **Push to Buffer**:
    - Operation: **push**
    - List: `={{ $('Global Variables').item.json.number }}`
    - Tail: true
    - MessageData: JSON string containing `message`, `sessionID`, `date_time` (as in workflow)
21. Add **Redis** node **Get From Buffer**:
    - Operation: **get**
    - Key: same number key
    - PropertyName: `message`
22. Add **Switch** node **Buffer Route**:
    - Ignore: last sessionID != current webhook id
    - Continue: last date_time before now - wait_buffer seconds
    - Fallback renamed to **Wait**
23. Add **Wait** node **Wait For User Other Fast Message**:
    - Amount: `={{ wait_buffer }}`
24. Add **Redis** node **Delete Buffer** (delete key number).
25. Add **Set** node **Normalize Buffer**:
    - `message = {{ $json.message.map(m => JSON.parse(m).message).join('\n') }}`
26. Add **NoOp** node **No Operation, do nothing** as sink.
27. Wire:
    - get_message (text) → Push to Buffer → Get From Buffer → Buffer Route
    - Buffer Route:
      - Ignore → NoOp
      - Wait → Wait node → back to Get From Buffer
      - Continue → Delete Buffer → Normalize Buffer

### 4.8 Media download + normalization path
28. From Message Type (AlbumMessage and/or fallback extra), connect to Evolution API **Descargar Media** (get-media-base64).
29. Add **Convert to File** (base64 → binary) using `data.base64`.
30. Add **Code** node **Get Mime Type** to set `fileTypeCategory` based on mimetype.
31. Add **Switch** node **Media Type** routing categories to extract/analyze nodes.
32. For each branch create the extractor/analyzer + normalization Set nodes:
    - image → Gemini image analyze → `data` set
    - audio → Gemini audio analyze → `data` set
    - video → Gemini video analyze → `data` set
    - pdf → ExtractFromFile(pdf) → IF Text? → Normalize PDF OR Gemini document analyze fallback
    - csv/html/ics/json/xml/xlsx/ods/rtf/text similarly using ExtractFromFile + normalization
    - fallback → get_error_message
33. Add **Set** node **get_message (File message)**:
    - Build final `message` including caption, fileName, fileTypeCategory, extracted `data`.
34. Add **Set** node **Normalize input**: `message = $json.message`.
35. Connect every normalization branch → get_message (File message) → Normalize input.

### 4.9 Album grouping buffer
36. Add **Redis** node **Push to Buffer (AlbumGroup)**:
    - List: `Media_{{number}}`
    - push expected count (images+videos)
37. Add **Redis** node **Check AlbumGroup**: get `Media_{{number}}`
38. Add **IF** node **Album?**: array not empty
39. Add **Redis** node **Push Media to Buffer** (push message into Media_{{number}})
40. Add **Redis** node **Get Album From Buffer** (get Media_{{number}})
41. Add **IF** node **Completed?**: `(length-1) == expectedCount`
42. Add **Redis** node **Delete Media Buffer**
43. Add **Set** node **Normalize MediaGroup Buffer**: join slice(1)
44. Wire:
    - Message Type (AlbumMessage) → Push to Buffer (AlbumGroup)
    - Normalize input → Check AlbumGroup → Album?
      - true → Push Media to Buffer → Get Album From Buffer → Completed?
        - true → Delete Media Buffer → Normalize MediaGroup Buffer
        - false → NoOp
      - false → Push to Buffer (debounce path)

### 4.10 Hybrid memory retrieval
45. After Normalize Buffer / Normalize MediaGroup Buffer, add **Set** node **Get Message**: `message = $json.message || $json.chatInput`.
46. Add **Redis** node **Get chat_history1**: get `Chats_{{number}}`
47. Add **IF** node **chat_history In Buffer?**
48. If false:
    - Add **Postgres** node **Get chat_history From DataBase** (select `message_text`, limit max_chat_history, where user number)
    - Add **Aggregate** node **Aggregate all Messges** on `message_text`
    - Add **IF** node **chat_history in DataBase?**
    - True → Redis **Push to chat_history** (push joined text)
    - False → Redis **Push first message to chat_history** (push `.`)
49. Add **Set** node **Get chat_history**: `history = propertyName.slice(-max_chat_history).join('\n')`

### 4.11 AI refinement + response
50. Add **Gemini Chat Model** node (flash-lite) and connect it to **Context Refiner** as language model.
51. Add **Context Refiner** (chainLlm) with the provided system message and input template.
52. Add **Gemini Chat Model1** node (gemini-3-flash-preview) and connect it to **AI Agent** as language model.
53. Add **AI Agent** node with system instructions (language mirroring, NO_CONTEXT rule) and prompt that includes retrieved context + user message.
54. Add **Set** node **Agent Response**: `reponse = $json.output`
55. Add **Evolution API** node **Send Response**:
    - instanceName: set to your Evolution instance (template uses `"Pruebas"`)
    - remoteJid: ensure correct format (number vs full JID depending on Evolution)
    - messageText: `{{$json.reponse}}`

### 4.12 Save interaction + cache lifecycle
56. Add **Redis** node **Push message chat_history (User And Agent)** pushing `"User: ...\nAgent: ..."` into `Chats_{{number}}`.
57. Add **Postgres** node **Add Messages to chat_history (User And Agent)** inserting into `chat_history`.
58. Add **Redis** node **Push to Buffer (chat_history)** pushing sessionID/date_time JSON to `Chats_buffer{{number}}`.
59. Add **Redis** node **Get From Buffer(chat_history)**, **Switch** node **Buffer (chat_history) Route**, and **Wait** node **Wait For end of conversation** (amount = wait_conversation).
60. Add **Redis** deletes:
    - **Delete Buffer (chat_history)** deletes `Chats_{{number}}`
    - **Delete Buffer (chat_history)2** deletes `Chats_buffer{{number}}`
61. Wire:
    - Send Response → Push message chat_history → Postgres insert → Push to Buffer(chat_history) → Get From Buffer(chat_history) → Buffer Route
    - Buffer Route:
      - Wait → Wait node → back to Get From Buffer(chat_history)
      - Continue → Delete Buffer(chat_history) → Delete Buffer(chat_history)2
      - Ignore → NoOp

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install community node `n8n-nodes-evolution-api` via `Settings > Community Nodes` | From workflow notes (setup checklist) |
| Execute the “Create Table chat_history” node once before activating | Database initialization requirement |
| Core buffering/memory knobs: `wait_buffer`, `wait_conversation`, `max_chat_history` | “⚠️ Core Configuration” sticky note |
| Contact for customization help | Email: [johnsilva11031@gmail.com](mailto:johnsilva11031@gmail.com) |
| Author LinkedIn | [John Alejandro Silva Rodríguez](https://www.linkedin.com/in/john-alejandro-silva-rodriguez-48093526b/) |

**Implementation cautions (high-impact)**
- **Potential category mismatch:** `Get Mime Type` outputs `text`, but `Media Type` checks `text file`. Align these values.
- **ODS routing mismatch:** MIME for ODS currently maps to `spreadsheet`, but the switch has both `ods` and `spreadsheet` branches; ensure intended path.
- **Evolution `remoteJid` format:** `Send Response` uses just the number; Evolution often expects full JID (`...@s.whatsapp.net`). Verify and adjust.
- **FK dependency:** Postgres table references `mentour_users`; adjust if your schema differs.