Handle clinic WhatsApp support with GPT-4o-mini, Pinecone RAG and Gmail alerts

https://n8nworkflows.xyz/workflows/handle-clinic-whatsapp-support-with-gpt-4o-mini--pinecone-rag-and-gmail-alerts-11605


# Handle clinic WhatsApp support with GPT-4o-mini, Pinecone RAG and Gmail alerts

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Handle clinic WhatsApp support with GPT-4o-mini, Pinecone RAG and Gmail alerts

**Purpose:**  
This workflow acts as a WhatsApp Cloud API customer support agent for a medical clinic (Arabic-first). It receives WhatsApp messages (text/audio/image/document), normalizes them into a single “final text”, builds conversation context + appointment/slot context, queries an AI agent (GPT-4o-mini) with RAG via Pinecone, replies to the user on WhatsApp, extracts structured fields (booking/reschedule/human escalation/leads), and synchronizes operational tables to a Google Sheets CRM. Documents sent by users trigger Gmail alerts with the attachment.

**Target use cases:**
- Automate WhatsApp clinic inquiries in Arabic (services/pricing/booking/rescheduling).
- Detect explicit human escalation requests and notify staff.
- Maintain minimal conversation memory and log it.
- Store/maintain appointment rows per service.
- Capture leads with a short AI summary + category.
- Maintain a RAG knowledge base sourced from a Google Doc and stored in Pinecone.
- Periodically sync n8n Data Tables to Google Sheets.

### 1.1 Webhook Intake & Verification
Receives WhatsApp webhook calls, responds to Meta verification challenge, and routes processing by message type.

### 1.2 Media Handling (Audio / Image / Document)
Downloads audio/image/files from the Graph API, transcribes audio (Arabic), analyzes images (Vision), and emails staff when documents are received.

### 1.3 Text Normalization + Memory + Context Building
Combines (voice > image > text) into one final text, writes to a “memory” table, loads recent history and upcoming appointments, and formats a single JSON context for the AI agent (including available booking slots).

### 1.4 AI Agent + Tools (RAG + Appointments + Memory)
Runs the main GPT agent with (a) Pinecone retrieval tool, (b) appointment table tool, and (c) buffer memory.

### 1.5 Extraction + Appointment Save
Extracts strict JSON signals from the AI response, parses/normalizes them, then upserts appointments per (phone+service) and optionally inserts additional appointment rows when booking multiple services.

### 1.6 Leads + Human Escalation + Logging
Creates leads with summarized intent, triggers Gmail notification + “human queue” record when escalation is needed, and updates the conversation row with the final AI reply. Sends WhatsApp reply.

### 1.7 CRM Sync to Google Sheets (Scheduled)
Every minute, synces Data Tables (slots/leads/human/appointments) into a Google Sheets CRM.

### 1.8 RAG Knowledge Base Load (Manual Run)
Manual branch loads a Google Doc, chunks it, embeds it, and inserts into Pinecone for later retrieval.

---

## 2. Block-by-Block Analysis

### Block 1 — Webhook Intake & Verification
**Overview:** Handles WhatsApp Cloud API webhooks, including verification via `hub.challenge`, and routes messages based on message type for downstream processing.  
**Nodes involved:** `WhatsApp Webhook`, `Verify • WA (hub.challenge)`, `filter text messages`, `If Audio`, `If Image`, `If Video/PDF`

#### Node: WhatsApp Webhook
- **Type / role:** `Webhook` (entry point). Receives inbound WhatsApp Cloud API webhook POSTs (and verification GETs).
- **Configuration (interpreted):**
  - Path: `whatsapp`
  - Response mode: `responseNode` (workflow uses a separate Respond node to respond)
  - Multiple methods enabled (supports verification and messages)
- **Key expressions/vars:**
  - Downstream nodes access deep fields like:
    - `body.entry[0].changes[0].value.messages[0]...`
    - `...contacts[0].wa_id`, `...metadata.phone_number_id`
- **Connections:**
  - Output 1 → `Verify • WA (hub.challenge)` (always)
  - Output 2 → `If Audio`, `filter text messages`, `If Image`, `If Video/PDF` (parallel routing)
- **Failure/edge cases:**
  - Payload structure may differ if WhatsApp sends non-message events (statuses, etc.). Expressions that assume `messages[0]` may fail.
  - Verification requests may not contain `entry/changes/messages`.

#### Node: Verify • WA (hub.challenge)
- **Type / role:** `Respond to Webhook` returns verification token (`hub.challenge`) or `OK`.
- **Configuration:**
  - HTTP 200, `Content-Type: text/plain`
  - Response body: `{{$json.query['hub.challenge'] || 'OK'}}`
- **Connections:** Terminal response node for verification path.
- **Failure/edge cases:** If n8n receives an unexpected method/payload, `query` might be missing; expression safely falls back to `OK`.

#### Node: filter text messages
- **Type / role:** `IF` router for plain text messages.
- **Condition:** message type equals `"text"` using:
  - `{{$json.body.entry[0].changes[0].value.messages[0].type}} == "text"`
- **Connections:** True → `Prepare Text Data`
- **Failure/edge cases:** If `messages[0]` missing, expression can error; add “exists” guard or switch to optional chaining in a Code node.

#### Node: If Audio
- **Type / role:** `IF` router for audio messages.
- **Condition:** type equals `"audio"`
- **Connections:** True → `Get Audio`
- **Failure/edge cases:** Same deep path risk as above.

#### Node: If Image
- **Type / role:** `IF` router for images.
- **Condition:** `...messages[0].image.mime_type` contains `"image"`
- **Connections:** True → `Get Image`
- **Failure/edge cases:** If message is not image, `.image` is undefined; condition evaluation can fail without guard.

#### Node: If Video/PDF
- **Type / role:** `IF` router for documents/video treated as file escalation.
- **Condition (OR):**
  - document mime contains `"application"` OR contains `"video"`
- **Connections:** True → `Get file`
- **Failure/edge cases:** Assumes `messages[0].document.mime_type` exists.

---

### Block 2 — Media Handling (Audio / Image / File)
**Overview:** Retrieves media metadata and downloads content from WhatsApp Graph API. Audio is transcribed to Arabic text; images are analyzed with vision; documents are emailed to staff with attachments.  
**Nodes involved:** `Get Audio`, `Download Audio`, `Transcribe Audio`, `Get Image`, `Download Image`, `Analyze image`, `Edit Fields`, `Get file`, `Download file`, `Prepare Email With Attachment`, `Send Email with attachment`

#### Node: Get Audio
- **Type / role:** `HTTP Request` to Graph API to fetch audio media metadata (includes `url`).
- **Configuration:**
  - URL: `https://graph.facebook.com/v22.0/{{ audio.id }}`
  - Auth: Bearer token credential named `whatsapp`
- **Input:** from `If Audio`
- **Output:** JSON including `url` for download
- **Failure/edge cases:** Token expired/invalid; media id invalid; Graph API version mismatch.

#### Node: Download Audio
- **Type / role:** `HTTP Request` to download the audio binary from `{{$json.url}}`.
- **Configuration:** Bearer auth `whatsapp`
- **Input:** from `Get Audio`
- **Output:** Binary audio for transcription
- **Failure/edge cases:** Missing `url`; download returns non-audio content; ensure node is configured to keep binary.

#### Node: Transcribe Audio
- **Type / role:** `OpenAI` (LangChain OpenAI node) audio transcription (Whisper-like).
- **Configuration:**
  - Resource: `audio`, Operation: `transcribe`
  - Options: language `ar`, temperature `0.6`
  - Credential: `OpenAi account`
- **Input:** Binary audio from `Download Audio`
- **Output:** `json.text` with transcription
- **Failure/edge cases:** Missing binary input; OpenAI quota; file too large; unsupported codec.

#### Node: Get Image
- **Type / role:** `HTTP Request` to fetch image media metadata from Graph API.
- **Configuration:** `https://graph.facebook.com/v22.0/{{ image.id }}`
- **Auth:** Bearer `whatsapp`
- **Output:** JSON including `url`
- **Failure/edge cases:** Same as `Get Audio`.

#### Node: Download Image
- **Type / role:** `HTTP Request` to download image binary using `{{$json.url}}`.
- **Auth:** Bearer `whatsapp`
- **Output:** Binary image
- **Failure/edge cases:** ensure response is stored as binary; large images.

#### Node: Analyze image
- **Type / role:** `OpenAI` image analysis (vision).
- **Configuration:**
  - Model: `gpt-4o-mini`
  - Operation: `analyze`, Resource: `image`
  - Input type: base64 (expects base64 conversion from binary)
  - Prompt: “Describe the image and transcribe any text…”
  - Max tokens: 50
- **Input:** Image content
- **Output:** `json.content` (description / extracted text)
- **Failure/edge cases:** If binary not converted to base64 as expected; OpenAI image limits; maxTokens too low for complex images.

#### Node: Edit Fields
- **Type / role:** `Set` prepares a merged text payload combining image description and the image caption.
- **Configuration:**
  - Sets `text` to:
    - `image description: {{ $json.content }}`
    - `user's message: {{ ...image.caption }}`
- **Input:** from `Analyze image`
- **Output:** `json.text` used by normalization code
- **Failure/edge cases:** caption may not exist; expression errors if missing.

#### Node: Get file
- **Type / role:** `HTTP Request` to fetch document media metadata.
- **Configuration:** `https://graph.facebook.com/v22.0/{{ document.id }}`
- **Auth:** Bearer `whatsapp`
- **Output:** JSON including `url`
- **Failure/edge cases:** document field missing for video messages (this branch checks document mime only).

#### Node: Download file
- **Type / role:** `HTTP Request` to download the file binary from `{{$json.url}}`.
- **Configuration:** authentication field contains `genericAuthType="=httpBearerAuth"` (note the leading `=` looks like a misconfiguration in the JSON; in UI it should be `httpBearerAuth`).
- **Input:** from `Get file`
- **Output:** Binary file (PDF/video/etc.)
- **Failure/edge cases:**
  - Potential auth misconfiguration due to `=httpBearerAuth`.
  - File size limits.

#### Node: Prepare Email With Attachment
- **Type / role:** `Code` builds HTML email and passes through binary for Gmail attachment.
- **Configuration choices:**
  - Reads webhook contact name + phone
  - Ensures one binary key exists, otherwise throws:
    - `"مطلوب name و phone و ملف binary من العقدة السابقة."`
  - Outputs `{json:{subject, html}, binary:{[binaryKey]:...}}`
- **Input:** binary from `Download file`
- **Output:** HTML + binary attachment
- **Failure/edge cases:** If binary key name differs, it still detects first key; if none exists it hard fails.

#### Node: Send Email with attachment
- **Type / role:** `Gmail` sends email with the downloaded file attached.
- **Configuration:**
  - To: `user@example.com` (placeholder)
  - Subject: Arabic “هناك عميل أرسل ملف ”
  - Message body uses `{{$json.html}}`
  - Attachments from binary (UI config shows attachmentsBinary placeholder)
  - Credential: `Gmail account`
- **Failure/edge cases:** Gmail OAuth expired; attachment mapping not set to correct binary property; recipient placeholder not updated.

---

### Block 3 — Text Normalization + Memory + Context Builder
**Overview:** Consolidates message content into a single final text string, logs it, fetches recent conversation + upcoming appointments, and prepares a structured context object for the main AI agent.  
**Nodes involved:** `Prepare Text Data`, `Insert Row Memory`, `Get History`, `Get only users appointment`, `Format Context`

#### Node: Prepare Text Data
- **Type / role:** `Code` normalizes inbound message text from multiple sources.
- **Logic:**
  - Extracts phone: `...messages[0].from`
  - Tries in order:
    1) Transcribed voice from `Transcribe Audio`.json.text  
    2) Image-based text from `Edit Fields`.json.text  
    3) Plain text from webhook `.text.body`
  - Sets `finalText = voice || image || text || ''`
  - Adds `source` indicator: `voice|image|text`
  - Outputs `{phone, text: finalText, source, timestamp}`
- **Input:** triggered either directly from `filter text messages` or after transcription/image handling.
- **Output:** normalized item(s) with `json.text`
- **Failure/edge cases:**
  - Uses try/catch around node lookups to avoid hard failure.
  - If all sources missing, sends empty text (downstream agent may behave oddly).

#### Node: Insert Row Memory
- **Type / role:** `Data Table` insert that logs the user’s message.
- **Configuration:**
  - Table: `test table` (`l6biIqkSE1qLJvaZ`)
  - Columns:
    - `phone` = `contacts[0].wa_id`
    - `sender` = normalized `{{$json.text}}`
  - `executeOnce: true` and `alwaysOutputData: true`
- **Output:** table insertion response includes `id` (used later as `users_id`)
- **Failure/edge cases:** Phone stored as string; make sure consistency with other parts that cast `.toNumber()`.

#### Node: Get History
- **Type / role:** `Data Table` get recent history for same phone.
- **Filter:** `phone == {{ $('Prepare Text Data').item.json.phone }}`
- **executeOnce:** true
- **Output:** multiple items (history)
- **Failure/edge cases:** No history returns empty; downstream context builder handles.

#### Node: Get only users appointment
- **Type / role:** `Data Table` get upcoming appointments for this phone.
- **Filters (match all):**
  - `date >= {{$now.toISO()}}`
  - `phone == {{$json.phone}}` (from history stream; note: `$json.phone` at this point may not exist on history items—works if item includes phone; otherwise risk)
- **Output:** appointments list
- **Failure/edge cases:** Date type mismatch (stored as dateTime vs string). Uses ISO string compare; depends on table field type.

#### Node: Format Context
- **Type / role:** `Code` builds the final AI context JSON.
- **Key computed fields:**
  - `sender`: phone number
  - `text`: normalized message text
  - `wa_id`: `metadata.phone_number_id` (used for Graph endpoint)
  - `name`: contact profile name
  - `available_slots`: pulled from **node reference `Get Slots1`** (but the workflow only has `Get slots`—this is a likely broken reference)
  - `users_appointments`: formats all appointment items from `$input.all()`
  - `formatted_output`: formats last 10 conversation entries from `Get History` as `(timestamp, ai:..., user:...)`
  - `users_id`: inserted row id from `Insert Row Memory`
  - `type`: message source from `Prepare Text Data`
- **Connections:**
  - Input from `Get only users appointment`
  - Output → `Main AI Agent`
- **Failure/edge cases / important notes:**
  - **Broken node reference risk:** It tries `$('Get Slots1').all()` and `$('Get appointment1').all()` but those nodes do not exist in the provided JSON. That means available slots and existing appointment formatting may always error and fall back to `[]`.
  - If webhook missing `contacts[0].profile.name`, this may throw.
  - It assumes `Insert Row Memory` returns `.json.id`.

---

### Block 4 — AI Agent + Tools (RAG + Appointments + Memory)
**Overview:** The main GPT agent answers in Arabic, using Pinecone RAG tool and appointment lookup tool, while also using a short memory buffer.  
**Nodes involved:** `OpenAI Chat Model`, `Main AI Agent`, `Simple Memory`, `Vector Store Tool`, `Pinecone Vector Store (Retrieval)`, `Embeddings OpenAI1`, `Booked Appointment`

#### Node: OpenAI Chat Model
- **Type / role:** LangChain chat LLM configuration node shared by multiple agents/tools.
- **Configuration:** model `gpt-4o-mini`
- **Connected as ai_languageModel to:** `Main AI Agent`, `AI Data Extractor`, `AI Summarizer1`, `Vector Store Tool`
- **Failure/edge cases:** OpenAI credential/quota; model availability.

#### Node: Simple Memory
- **Type / role:** LangChain memory buffer window (very short).
- **Configuration:**
  - session key: `{{$json.sender}}`
  - window length: 1
- **Connection:** provides `ai_memory` to `Main AI Agent`
- **Failure/edge cases:** If `sender` empty, sessions collide.

#### Node: Pinecone Vector Store (Retrieval)
- **Type / role:** Pinecone vector store configured for retrieval.
- **Configuration:**
  - Index: `clinic`
  - Embeddings: from `Embeddings OpenAI1`
- **Connection:** `ai_vectorStore` → `Vector Store Tool`
- **Failure/edge cases:** wrong Pinecone API key, index missing, namespace mismatch.

#### Node: Embeddings OpenAI1
- **Type / role:** embeddings model config for retrieval.
- **Credentials:** OpenAI
- **Connection:** `ai_embedding` → `Pinecone Vector Store (Retrieval)`
- **Failure/edge cases:** embedding model defaults; dimension mismatch with existing Pinecone index.

#### Node: Vector Store Tool
- **Type / role:** Tool wrapper enabling the agent to query Pinecone.
- **Configuration:**
  - Name: `company_documents_tool`
  - Description: “Retrieve information from any company documents”
- **Connection:** `ai_tool` → `Main AI Agent`
- **Failure/edge cases:** If the knowledge base has not been inserted (Block 8), retrieval returns poor/empty results.

#### Node: Booked Appointment
- **Type / role:** `DataTableTool` to allow the agent to fetch appointment data as a tool.
- **Configuration:** `get` from appointments table `XICUlFpXl76eV2Dm`
- **Connection:** `ai_tool` → `Main AI Agent`
- **Failure/edge cases:** Tool returns unfiltered data unless agent supplies constraints; potential privacy/scale concerns.

#### Node: Main AI Agent
- **Type / role:** LangChain agent orchestrating the conversation.
- **Input:** `text = {{$json.text}}` from `Format Context`
- **System message highlights:**
  - Arabic only, no markdown
  - Use Pinecone vector store for clinic knowledge
  - Include memory: `{{$json.formatted_output}}`
  - Include Slots and Appointments contexts
  - Strictly avoid inventing dates
  - Clinic offers three services (as written): `feller`, `عناية بالبشرة`, `تبييض أسنان`
  - Booking response templates:
    - Booking: `تم حجز موعدك! ✅ 📅 [date]|[day] 🕐 [time] 💉 [service]`
    - Reschedule: `تم تعديل موعدك ...`
  - If message type indicates image, answer shortly about the image + caption if present
- **Outputs (parallel):**
  - → `Send via WhatsApp` (reply to user)
  - → `Update Row` (log reply in memory table)
  - → `AI Data Extractor` (structured extraction)
  - → `Get leads` (lead handling)
- **Failure/edge cases:**
  - If `available_slots` is empty due to broken reference, booking accuracy declines.
  - Agent may generate non-JSON despite `hasOutputParser` if parser not configured correctly in node UI.

---

### Block 5 — Extraction + Appointment Save
**Overview:** Extracts structured booking/reschedule/human escalation signals from the AI’s response, normalizes to stable fields, and writes appointments to the Data Table with upsert/insert logic.  
**Nodes involved:** `AI Data Extractor`, `Output Format Text`, `Check Appointment1`, `save appointment - same service`, `If Appointment is ready`, `save appointment - different service`

#### Node: AI Data Extractor
- **Type / role:** LangChain agent used as a strict JSON extractor over the AI reply.
- **Input:** `{{$json.output}}` (coming from `Main AI Agent` output)
- **System message:** Very detailed JSON schema with rules:
  - has_appointment true only if confirmation phrases exist
  - needs_human true only for explicit human request, anger, emergency, repeated bot failure
  - lead_category constrained to enumerated list
  - Date handling: if day mentioned but no date, pick nearest correct date
- **Output:** `json.output` containing JSON (string or object)
- **Failure/edge cases:** If main agent reply doesn’t contain enough signals; extractor may hallucinate fields unless constrained; ensure output parser enforces valid JSON.

#### Node: Output Format Text
- **Type / role:** `Code` parses extractor output safely and enriches it.
- **Key operations:**
  - Parses `$input.first().json.output`
  - If parse fails: strips ```json fences and extracts `{...}` via regex; else fallback default structure.
  - Enriches with:
    - `phone` and `customer_name` from `Format Context`
    - `created_at` now
    - `has_appointment_text`, `is_reschedule_text`, `needs_human_text` as `YES/NO`
  - Converts null/undefined to empty strings
- **Output:** single enriched JSON item
- **Failure/edge cases:** If `Format Context` missing, phone/name become empty.

#### Node: Check Appointment1
- **Type / role:** `IF` gates appointment write.
- **Condition:** `{{$json.has_appointment}}` is boolean true.
- **Connection:** True → `save appointment - same service`
- **Failure/edge cases:** Extractor must output boolean, not string. (Strict typeValidation is enabled.)

#### Node: save appointment - same service
- **Type / role:** `Data Table` upsert appointment keyed by (phone + service).
- **Configuration:**
  - Table: `appointments` (`XICUlFpXl76eV2Dm`)
  - Match type: allConditions
  - Filters:
    - `phone == {{$json.phone}}`
    - `service == {{$json.service}}`
  - Upsert columns:
    - day/date/time/service + customer name
- **Failure/edge cases:** `phone` column type in schema is `number`, but other places use string; conversion issues possible.

#### Node: If Appointment is ready
- **Type / role:** `IF` checks if a different service was booked compared to previous “checked service”.
- **Condition:** `{{$('Output Format Text').item.json.service}} != {{$('Check Appointment1').item.json.service}}`
- **Connection:** True → `save appointment - different service`
- **Important:** This comparison looks logically suspicious: `Check Appointment1` passes through the same item; it does not represent “previous booking”. It likely always equals and thus prevents multi-service insert, unless `Check Appointment1` item differs due to multi-item flows.
- **Failure/edge cases:** This may never trigger as intended.

#### Node: save appointment - different service
- **Type / role:** `Data Table` insert a new appointment row (no match filters).
- **Configuration:** inserts day/date/time/service + name/phone.
- **Failure/edge cases:** Duplicates possible; no dedupe logic.

---

### Block 6 — Leads + Human Escalation + Logging
**Overview:** Summarizes and stores leads, detects escalation to humans (email + queue), logs AI response to memory table, and sends WhatsApp reply.  
**Nodes involved:** `Get leads`, `Check if Lead`, `AI Summarizer1`, `Format output`, `Insert Lead`, `Check Human is called`, `Send Email Notification If Human is called`, `Add row for Human Call`, `Update Row`, `Send via WhatsApp`

#### Node: Get leads
- **Type / role:** `Data Table` get existing leads by phone.
- **Filter:** `phone == {{ $('Format Context').item.json.sender }}`
- **Output:** may return 0..n items
- **Failure/edge cases:** Phone type in leads table is `number` (later); here treated as string—may not match.

#### Node: Check if Lead
- **Type / role:** `IF` decides whether to create a lead.
- **Condition:** uses `notExists` on `{{$json.name}}` with rightValue `{{ $('Format Context').item.json.sender.toNumber() }}`.
- **Important:** This condition is likely incorrect:
  - `notExists` checks existence of left value, not equality.
  - `$json.name` depends on lead table output shape, not guaranteed.
- **Connection:** True → `AI Summarizer1`
- **Failure/edge cases:** Lead dedupe may not work; may create leads repeatedly or never.

#### Node: AI Summarizer1
- **Type / role:** LangChain agent producing compact JSON summary/category.
- **Input text:** `{{$('Format Context').item.json.formatted_output}}` (note: it summarizes history, not the latest message; might be intentional but usually you want latest text)
- **System message:** returns only:
  - `{"summary":"...", "category":"..."}`
  - Categories: appointment/pricing/discount/teeth_whitening/skincare/staff_request/out_of_scope
  - Output language equals user language
- **Output:** `json.output` or parsed fields depending on parser
- **Failure/edge cases:** If formatted_output empty, summary meaningless.

#### Node: Format output
- **Type / role:** `Code` parses summarizer output into `{summary, category}`.
- **Behavior:**
  - Accepts either object with summary/category or string JSON
  - Removes ```json fences
  - On error outputs `{summary:"Error parsing response", category:"error"}`
- **Failure/edge cases:** If summarizer returns additional text, parse fails.

#### Node: Insert Lead
- **Type / role:** `Data Table` insert lead row.
- **Columns:**
  - name = `Format Context.name`
  - phone = `Format Context.sender`
  - short_sum = `{{$json.summary}}`
  - lead_category = `{{$json.category}}`
- **Failure/edge cases:** Phone schema expects number; source is string.

#### Node: Check Human is called
- **Type / role:** `IF` gates human escalation.
- **Condition:** `{{$json.needs_human}}` boolean true
- **Connection:** True → `Send Email Notification If Human is called`

#### Node: Send Email Notification If Human is called
- **Type / role:** `Gmail` sends a rich HTML alert to staff.
- **Configuration:**
  - To: `user@example.com` (placeholder)
  - Subject: Arabic “هناك عميل يحتااج تواصل بشري ”
  - HTML template uses:
    - `{{$json.customer_name}}`, `{{$json.phone}}`, `{{$json.description}}`
  - Contains placeholders like `{{VIEW_LINK}}`, `{{UNSUBSCRIBE_LINK}}` that are not populated by this workflow (will appear literally unless replaced).
- **Failure/edge cases:** OAuth; placeholders not replaced; HTML size.

#### Node: Add row for Human Call
- **Type / role:** `Data Table` upsert into “human” queue table.
- **Configuration:**
  - Operation: `upsert`
  - Filters: `phone == {{ $('Output Format Text').item.json.phone }}`
  - Columns: name/phone (description field removed in schema)
- **Failure/edge cases:** If you need description stored, schema currently removes it (`descreption` removed).

#### Node: Update Row
- **Type / role:** `Data Table` upsert to update conversation/memory row with AI reply (intended).
- **Configuration issues:**
  - Columns value is `{}` (no fields mapped), and schema fields are marked `removed:true`.
  - Filter uses only `keyValue: {{ users_id }}` without specifying `keyName`.
  - As written, this node likely does not update anything meaningful.
- **Failure/edge cases:** This is likely broken; needs proper `filters.conditions[0].keyName = 'id'` and set `reciever` to the AI output.

#### Node: Send via WhatsApp
- **Type / role:** `HTTP Request` sends reply to WhatsApp Cloud API.
- **Configuration:**
  - URL: `https://graph.facebook.com/v22.0/{{ wa_id }}/messages`
  - Body includes:
    - `to`: `{{ sender }}` (phone)
    - text body: `{{ JSON.stringify($json.output) }}`
  - Auth: Bearer credential `whatsapp`
- **Failure/edge cases:**
  - Uses `JSON.stringify($json.output)`; if output already string, it will add quotes.
  - WhatsApp API expects text.body string; ensure not exceeding limits.
  - `wa_id` must be the phone_number_id, not user wa_id.

---

### Block 7 — CRM Sync to Google Sheets (Scheduled)
**Overview:** Every minute, reads operational n8n Data Tables and upserts rows into Google Sheets CRM tabs.  
**Nodes involved:** `Schedule Trigger`, `Get slots`, `Append or update slot`, `Get lead`, `Append or update lead`, `Get human call`, `Append or update human call`, `Get human appointment`, `Append or update appointment`

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger`
- **Configuration:** every 1 minute
- **Outputs:** triggers 4 parallel gets:
  - `Get slots`, `Get lead`, `Get human call`, `Get human appointment`
- **Failure/edge cases:** frequent sync can hit Google API quota; consider batching or less frequent schedule.

#### Node: Get slots
- **Type / role:** `Data Table` get all rows from `slots` table (`DySLjqEPX2i5RBdj`).
- **Output →** `Append or update slot`

#### Node: Append or update slot
- **Type / role:** `Google Sheets` appendOrUpdate into `CRM` spreadsheet, sheet `slots`.
- **Matching column:** `service`
- **Columns written:** `day`, `time`, `service`
- **Credential:** Google Sheets OAuth2
- **Failure/edge cases:** matching only by service means one row per service; if multiple times per service exist, they overwrite.

#### Node: Get lead
- **Type / role:** `Data Table` get all leads (`16bAoJicIDEPak46`)
- **Output →** `Append or update lead`

#### Node: Append or update lead
- **Type / role:** `Google Sheets` appendOrUpdate into sheet `leads`.
- **Matching column:** `phone`
- **Columns written:** name, phone, short sum, lead category
- **Failure/edge cases:** phone formatting differences (string vs number) cause duplicates.

#### Node: Get human call
- **Type / role:** `Data Table` get all rows from human queue (`ygicufEiNou6tVCb`)
- **Output →** `Append or update human call`

#### Node: Append or update human call
- **Type / role:** `Google Sheets` appendOrUpdate into sheet `human call`.
- **Matching column:** `phone`
- **Columns:** name, phone, description (maps from `$json.descreption` which is misspelled and may be removed in human table schema)
- **Failure/edge cases:** description likely blank due to schema removal/mismatch.

#### Node: Get human appointment
- **Type / role:** `Data Table` get all appointments (`XICUlFpXl76eV2Dm`)
- **Output →** `Append or update appointment`

#### Node: Append or update appointment
- **Type / role:** `Google Sheets` appendOrUpdate into sheet `appointments`.
- **Matching column:** `phone` (this will collapse multiple services per phone into one row—likely wrong).
- **Columns:** day/date/name/time/phone/service
- **Failure/edge cases:** If multiple appointments per phone (different services), they overwrite due to matching on phone only.

---

### Block 8 — RAG Knowledge Base Load (Manual Run)
**Overview:** Manually executed path that pulls a Google Doc, chunks it, embeds it, and inserts vectors into Pinecone for RAG retrieval.  
**Nodes involved:** `When clicking ‘Execute workflow’`, `Get a document`, `Pinecone Vector Store`, `Embeddings OpenAI`, `Recursive Character Text Splitter`, `Default Data Loader`

#### Node: When clicking ‘Execute workflow’
- **Type / role:** `Manual Trigger`
- **Use:** run to refresh Pinecone knowledge base

#### Node: Get a document
- **Type / role:** `Google Docs` get document content.
- **Configuration:** Document URL hardcoded to a specific Google Doc.
- **Failure/edge cases:** Doc permissions; Google credential missing.

#### Node: Pinecone Vector Store
- **Type / role:** Pinecone insert mode.
- **Configuration:**
  - Mode: insert
  - Index: `n8n` (note: retrieval uses index `clinic`, so this is inconsistent)
  - Namespace: `"1"`
  - Embeddings from `Embeddings OpenAI`
  - Documents from `Default Data Loader`
- **Failure/edge cases:** Index mismatch (insert to `n8n` but retrieve from `clinic` means RAG won’t find data). This should be aligned.

#### Node: Embeddings OpenAI
- **Type / role:** embeddings provider for insert pipeline.
- **Credentials:** OpenAI

#### Node: Recursive Character Text Splitter
- **Type / role:** splits the document into chunks for embedding.
- **Configuration:** chunk overlap 100 (chunk size not shown; defaults apply)
- **Connection:** `ai_textSplitter` → `Default Data Loader`

#### Node: Default Data Loader
- **Type / role:** converts input document into LangChain documents, using the text splitter.
- **Connection:** `ai_document` → `Pinecone Vector Store`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| WhatsApp Webhook | Webhook | Entry point for WhatsApp Cloud API | — | Verify • WA (hub.challenge); If Audio; filter text messages; If Image; If Video/PDF | Section 1 — Webhook & Verification: Receives WhatsApp webhooks, responds to `hub.challenge` verification, and routes incoming messages by type. |
| Verify • WA (hub.challenge) | Respond to Webhook | Responds to Meta verification challenge | WhatsApp Webhook | — | Section 1 — Webhook & Verification: Receives WhatsApp webhooks, responds to `hub.challenge` verification, and routes incoming messages by type. |
| filter text messages | IF | Route text messages | WhatsApp Webhook | Prepare Text Data | Section 2 — Media Handling (Audio / Image / File) Fetches media metadata from Graph API, downloads files, transcribes audio, analyzes images, and emails staff when a document/PDF is received. |
| If Audio | IF | Route audio messages | WhatsApp Webhook | Get Audio | Section 2 — Media Handling (Audio / Image / File) Fetches media metadata from Graph API, downloads files, transcribes audio, analyzes images, and emails staff when a document/PDF is received. |
| Get Audio | HTTP Request | Fetch audio media metadata | If Audio | Download Audio | Section 2 — Media Handling (Audio / Image / File) Fetches media metadata from Graph API, downloads files, transcribes audio, analyzes images, and emails staff when a document/PDF is received. |
| Download Audio | HTTP Request | Download audio binary | Get Audio | Transcribe Audio | Section 2 — Media Handling (Audio / Image / File) Fetches media metadata from Graph API, downloads files, transcribes audio, analyzes images, and emails staff when a document/PDF is received. |
| Transcribe Audio | OpenAI (LangChain) | Transcribe Arabic audio to text | Download Audio | Prepare Text Data | Section 2 — Media Handling (Audio / Image / File) Fetches media metadata from Graph API, downloads files, transcribes audio, analyzes images, and emails staff when a document/PDF is received. |
| If Image | IF | Route image messages | WhatsApp Webhook | Get Image | Section 2 — Media Handling (Audio / Image / File) Fetches media metadata from Graph API, downloads files, transcribes audio, analyzes images, and emails staff when a document/PDF is received. |
| Get Image | HTTP Request | Fetch image media metadata | If Image | Download Image | Section 2 — Media Handling (Audio / Image / File) Fetches media metadata from Graph API, downloads files, transcribes audio, analyzes images, and emails staff when a document/PDF is received. |
| Download Image | HTTP Request | Download image binary | Get Image | Analyze image | Section 2 — Media Handling (Audio / Image / File) Fetches media metadata from Graph API, downloads files, transcribes audio, analyzes images, and emails staff when a document/PDF is received. |
| Analyze image | OpenAI (LangChain) | Vision analysis + OCR-like extraction | Download Image | Edit Fields | Section 2 — Media Handling (Audio / Image / File) Fetches media metadata from Graph API, downloads files, transcribes audio, analyzes images, and emails staff when a document/PDF is received. |
| Edit Fields | Set | Merge image analysis + caption to text | Analyze image | Prepare Text Data | Section 2 — Media Handling (Audio / Image / File) Fetches media metadata from Graph API, downloads files, transcribes audio, analyzes images, and emails staff when a document/PDF is received. |
| If Video/PDF | IF | Route document/video | WhatsApp Webhook | Get file | Section 2 — Media Handling (Audio / Image / File) Fetches media metadata from Graph API, downloads files, transcribes audio, analyzes images, and emails staff when a document/PDF is received. |
| Get file | HTTP Request | Fetch file media metadata | If Video/PDF | Download file | Section 2 — Media Handling (Audio / Image / File) Fetches media metadata from Graph API, downloads files, transcribes audio, analyzes images, and emails staff when a document/PDF is received. |
| Download file | HTTP Request | Download file binary | Get file | Prepare Email With Attachment | Section 2 — Media Handling (Audio / Image / File) Fetches media metadata from Graph API, downloads files, transcribes audio, analyzes images, and emails staff when a document/PDF is received. |
| Prepare Email With Attachment | Code | Build Gmail HTML + pass-through binary | Download file | Send Email with attachment | Section 2 — Media Handling (Audio / Image / File) Fetches media metadata from Graph API, downloads files, transcribes audio, analyzes images, and emails staff when a document/PDF is received. |
| Send Email with attachment | Gmail | Notify staff with attachment | Prepare Email With Attachment | — | Section 2 — Media Handling (Audio / Image / File) Fetches media metadata from Graph API, downloads files, transcribes audio, analyzes images, and emails staff when a document/PDF is received. |
| Prepare Text Data | Code | Normalize final text (voice>image>text) | filter text messages / Transcribe Audio / Edit Fields | Insert Row Memory | Section 3 — Memory & Context Builder … |
| Insert Row Memory | Data Table | Store inbound message in memory table | Prepare Text Data | Get History | Section 3 — Memory & Context Builder … |
| Get History | Data Table | Fetch recent conversation by phone | Insert Row Memory | Get only users appointment | Section 3 — Memory & Context Builder … |
| Get only users appointment | Data Table | Fetch upcoming appointments by phone | Get History | Format Context | Section 3 — Memory & Context Builder … |
| Format Context | Code | Build AI-ready context JSON | Get only users appointment | Main AI Agent | Section 3 — Memory & Context Builder … |
| OpenAI Chat Model | LM Chat OpenAI | Shared LLM config | — | AI Summarizer1; AI Data Extractor; Main AI Agent; Vector Store Tool | Section 4 — AI Agent + Tools (RAG + Appointments + Memory) … |
| Simple Memory | Memory Buffer Window | Minimal per-user memory | — | Main AI Agent (ai_memory) | Section 4 — AI Agent + Tools (RAG + Appointments + Memory) … |
| Pinecone Vector Store (Retrieval) | Vector Store Pinecone | Retrieval backend for RAG | — | Vector Store Tool | Section 4 — AI Agent + Tools (RAG + Appointments + Memory) … |
| Embeddings OpenAI1 | Embeddings OpenAI | Embeddings for retrieval | — | Pinecone Vector Store (Retrieval) | Section 4 — AI Agent + Tools (RAG + Appointments + Memory) … |
| Vector Store Tool | Tool Vector Store | Agent tool for document retrieval | Pinecone Vector Store (Retrieval) | Main AI Agent (ai_tool) | Section 4 — AI Agent + Tools (RAG + Appointments + Memory) … |
| Booked Appointment | DataTableTool | Agent tool to query appointments | — | Main AI Agent (ai_tool) | Section 4 — AI Agent + Tools (RAG + Appointments + Memory) … |
| Main AI Agent | LangChain Agent | Generate Arabic clinic response (with tools) | Format Context | Send via WhatsApp; Update Row; AI Data Extractor; Get leads | Section 4 — AI Agent + Tools (RAG + Appointments + Memory) … |
| AI Data Extractor | LangChain Agent | Extract strict JSON booking/escalation fields | Main AI Agent | Output Format Text | Section 5 — Extraction + Appointment Save … |
| Output Format Text | Code | Parse+normalize extractor JSON + metadata | AI Data Extractor | Check Appointment1; Check Human is called | Section 5 — Extraction + Appointment Save … |
| Check Appointment1 | IF | Gate appointment saving | Output Format Text | save appointment - same service | Section 5 — Extraction + Appointment Save … |
| save appointment - same service | Data Table | Upsert appointment by phone+service | Check Appointment1 | If Appointment is ready | Section 5 — Extraction + Appointment Save … |
| If Appointment is ready | IF | Insert new row if different service | save appointment - same service | save appointment - different service | Section 5 — Extraction + Appointment Save … |
| save appointment - different service | Data Table | Insert additional appointment row | If Appointment is ready | — | Section 5 — Extraction + Appointment Save … |
| Get leads | Data Table | Check existing leads by phone | Main AI Agent | Check if Lead | Section 6 — Leads + Human Escalation + Logging … |
| Check if Lead | IF | Decide lead insertion | Get leads | AI Summarizer1 | Section 6 — Leads + Human Escalation + Logging … |
| AI Summarizer1 | LangChain Agent | Summarize + categorize in JSON | Check if Lead | Format output | Section 6 — Leads + Human Escalation + Logging … |
| Format output | Code | Parse summarizer output | AI Summarizer1 | Insert Lead | Section 6 — Leads + Human Escalation + Logging … |
| Insert Lead | Data Table | Insert lead row | Format output | — | Section 6 — Leads + Human Escalation + Logging … |
| Check Human is called | IF | Gate escalation flow | Output Format Text | Send Email Notification If Human is called | Section 6 — Leads + Human Escalation + Logging … |
| Send Email Notification If Human is called | Gmail | Email staff for escalation | Check Human is called | Add row for Human Call | Section 6 — Leads + Human Escalation + Logging … |
| Add row for Human Call | Data Table | Upsert into human queue | Send Email Notification If Human is called | — | Section 6 — Leads + Human Escalation + Logging … |
| Update Row | Data Table | Intended: log AI reply in memory row | Main AI Agent | — | Section 6 — Leads + Human Escalation + Logging … |
| Send via WhatsApp | HTTP Request | Send AI response back to user | Main AI Agent | — | Section 6 — Leads + Human Escalation + Logging … |
| Schedule Trigger | Schedule Trigger | Periodic CRM sync | — | Get slots; Get lead; Get human call; Get human appointment | Section 7 — CRM Sync to Google Sheets … (note: sticky note disabled in workflow) |
| Get slots | Data Table | Read slots table for sync | Schedule Trigger | Append or update slot | Section 7 — CRM Sync to Google Sheets … (note: sticky note disabled in workflow) |
| Append or update slot | Google Sheets | Upsert slots sheet | Get slots | — | Section 7 — CRM Sync to Google Sheets … (note: sticky note disabled in workflow) |
| Get lead | Data Table | Read leads table for sync | Schedule Trigger | Append or update lead | Section 7 — CRM Sync to Google Sheets … (note: sticky note disabled in workflow) |
| Append or update lead | Google Sheets | Upsert leads sheet | Get lead | — | Section 7 — CRM Sync to Google Sheets … (note: sticky note disabled in workflow) |
| Get human call | Data Table | Read human queue table for sync | Schedule Trigger | Append or update human call | Section 7 — CRM Sync to Google Sheets … (note: sticky note disabled in workflow) |
| Append or update human call | Google Sheets | Upsert human call sheet | Get human call | — | Section 7 — CRM Sync to Google Sheets … (note: sticky note disabled in workflow) |
| Get human appointment | Data Table | Read appointments table for sync | Schedule Trigger | Append or update appointment | Section 7 — CRM Sync to Google Sheets … (note: sticky note disabled in workflow) |
| Append or update appointment | Google Sheets | Upsert appointments sheet | Get human appointment | — | Section 7 — CRM Sync to Google Sheets … (note: sticky note disabled in workflow) |
| When clicking ‘Execute workflow’ | Manual Trigger | Start RAG ingestion | — | Get a document | Section 8 — RAG Knowledge Base (Manual Run) … |
| Get a document | Google Docs | Load knowledge doc | Manual Trigger | Pinecone Vector Store | Section 8 — RAG Knowledge Base (Manual Run) … |
| Recursive Character Text Splitter | Text Splitter | Chunk doc for embeddings | — | Default Data Loader | Section 8 — RAG Knowledge Base (Manual Run) … |
| Default Data Loader | Document Loader | Prepare documents for vector insert | Recursive Character Text Splitter | Pinecone Vector Store | Section 8 — RAG Knowledge Base (Manual Run) … |
| Embeddings OpenAI | Embeddings OpenAI | Embeddings for insert pipeline | — | Pinecone Vector Store | Section 8 — RAG Knowledge Base (Manual Run) … |
| Pinecone Vector Store | Vector Store Pinecone | Insert embeddings into Pinecone | Get a document + Default Data Loader + Embeddings OpenAI | — | Section 8 — RAG Knowledge Base (Manual Run) … |
| Sticky Note | Sticky Note | Comment | — | — | ## SENDING new rows to google sheet |
| Sticky Note1 | Sticky Note | Comment | — | — | ## RAG System |
| Sticky Note2 | Sticky Note | Comment | — | — | Section 3 — Memory & Context Builder … |
| Sticky Note3 | Sticky Note (disabled) | Comment | — | — | Section 7 — CRM Sync to Google Sheets … + link: https://docs.google.com/spreadsheets/d/1HCl3CvMnzILIrcjnFK-AwLqWuKfMwezM1yXWkJ4KG9Q/edit?usp=sharing |
| Sticky Note4 | Sticky Note | Comment | — | — | 🏥🤖 Medical Clinic WhatsApp Agent + CRM Sync (full description) |
| Sticky Note6 | Sticky Note | Comment | — | — | Section 1 — Webhook & Verification … |
| Sticky Note7 | Sticky Note | Comment | — | — | Section 5 — Extraction + Appointment Save … |
| Sticky Note8 | Sticky Note | Comment | — | — | Section 2 — Media Handling (Audio / Image / File) … |
| Sticky Note10 | Sticky Note | Comment | — | — | Section 6 — Leads + Human Escalation + Logging … |
| Sticky Note11 | Sticky Note | Comment | — | — | Section 8 — RAG Knowledge Base (Manual Run) … |
| Sticky Note12 | Sticky Note | Comment | — | — | Section 4 — AI Agent + Tools (RAG + Appointments + Memory) … |

---

## 4. Reproducing the Workflow from Scratch

1) **Create required credentials**
   1. **HTTP Bearer Auth** credential named `whatsapp` containing your WhatsApp Cloud API token.
   2. **OpenAI API** credential `OpenAi account`.
   3. **Gmail OAuth2** credential `Gmail account`.
   4. **Google Sheets OAuth2** credential `Google Sheets account`.
   5. **Google Docs** credential (if separate in your n8n instance).
   6. **Pinecone** credentials (configured inside Pinecone vector store nodes in n8n environment).

2) **Create Data Tables in n8n**
   - `test table` (memory): fields at least `phone` (string), `sender` (string), `reciever` (string), plus auto `id`, `createdAt`.
   - `appointments`: fields `name` (string), `phone` (prefer string for consistency), `date` (dateTime), `day` (string), `times` (string), `service` (string).
   - `slots`: fields `day`, `time`, `service`.
   - `leads`: fields `name`, `phone`, `short_sum`, `lead_category`.
   - `human`: fields `name`, `phone`, `descreption` (if you want to sync description to Sheets; keep spelling consistent).

3) **Section 1: Webhook & verification**
   1. Add **Webhook** node named `WhatsApp Webhook`:
      - Path: `whatsapp`
      - Response mode: **Respond with Respond to Webhook node**
      - Enable multiple methods.
   2. Add **Respond to Webhook** named `Verify • WA (hub.challenge)`:
      - Respond with: Text
      - Body: `{{$json.query['hub.challenge'] || 'OK'}}`
      - Header Content-Type: text/plain
   3. Add IF nodes:
      - `filter text messages` condition: message type == `text`
      - `If Audio` condition: message type == `audio`
      - `If Image` condition: image mime contains `image`
      - `If Video/PDF` OR condition: document mime contains `application` OR `video`
   4. Connect `WhatsApp Webhook` output(s) to these routers and to verification response node.

4) **Section 2: Media handling**
   - **Audio path**
     1. `Get Audio` (HTTP Request GET) to `https://graph.facebook.com/v22.0/{{ audio.id }}` with Bearer token.
     2. `Download Audio` (HTTP Request GET) to `{{$json.url}}` with Bearer token and binary response.
     3. `Transcribe Audio` (OpenAI node) resource audio → transcribe, language `ar`.
     4. Connect to `Prepare Text Data`.
   - **Image path**
     1. `Get Image` GET `https://graph.facebook.com/v22.0/{{ image.id }}` (Bearer).
     2. `Download Image` GET `{{$json.url}}` (Bearer) as binary.
     3. `Analyze image` OpenAI image analyze (gpt-4o-mini), input base64, max tokens 50.
     4. `Edit Fields` set `text` = `image description: {{ $json.content }} ... caption ...`
     5. Connect to `Prepare Text Data`.
   - **Document path**
     1. `Get file` GET `https://graph.facebook.com/v22.0/{{ document.id }}` (Bearer).
     2. `Download file` GET `{{$json.url}}` (Bearer) binary.
     3. `Prepare Email With Attachment` code node (use the provided logic; ensure it outputs `{json:{subject,html}, binary:{...}}`).
     4. `Send Email with attachment` Gmail node:
        - To: your staff email
        - Subject: appropriate Arabic text
        - Body: `{{$json.html}}`
        - Attachments: map the binary property from previous node.

5) **Section 3: Normalize + memory + context**
   1. Add `Prepare Text Data` (Code) to select voice > image > text and output `{phone, text, source, timestamp}`.
   2. Add `Insert Row Memory` (Data Table) insert into memory table: `phone`, `sender`.
   3. Add `Get History` (Data Table) get rows where phone equals the current phone.
   4. Add `Get only users appointment` (Data Table) get appointments where:
      - phone equals the user phone
      - date >= now
   5. Add `Format Context` (Code) to build final context JSON.
      - **Important:** Fix node references inside the code:
        - Replace `$('Get Slots1')` with the actual slots getter node name if you want live slots in context.
        - Remove/replace `$('Get appointment1')` if unused.
      - Ensure it outputs: `{sender,text,available_slots,formatted_output,name,wa_id,users_appointments,users_id,type}`.

6) **Section 4: AI agent + tools**
   1. Add `OpenAI Chat Model` (gpt-4o-mini).
   2. Add `Embeddings OpenAI1` and `Pinecone Vector Store (Retrieval)` configured to the **same index/namespace you will insert into**.
   3. Add `Vector Store Tool` pointing to the Pinecone retrieval node.
   4. Add `Booked Appointment` DataTableTool for appointments.
   5. Add `Simple Memory` buffer window (session key = phone).
   6. Add `Main AI Agent`:
      - Input text: `{{$json.text}}`
      - System message: include clinic rules, the formatted history, slots, and appointment list.
      - Attach the LLM, tools, and memory via n8n LangChain connections.
   7. Add `Send via WhatsApp` HTTP Request POST to:
      - `https://graph.facebook.com/v22.0/{{phone_number_id}}/messages`
      - JSON body with `"to": "{{user_wa_id}}"` or `"to": "{{from}}"` depending on your payload mapping.
      - Use Bearer token auth.

7) **Section 5: Extract + save appointment**
   1. Add `AI Data Extractor` agent node with strict JSON schema prompt.
   2. Add `Output Format Text` code node to parse and normalize booleans and add metadata.
   3. Add `Check Appointment1` IF: `has_appointment == true`.
   4. Add `save appointment - same service` Data Table upsert by (phone+service).
   5. Add `If Appointment is ready` IF for multi-service logic.
      - Consider redesign: compare against existing appointments from a table query rather than comparing to the same item.
   6. Add `save appointment - different service` insert node.

8) **Section 6: Leads + escalation + logging**
   1. Add `Get leads` Data Table get by phone.
   2. Add `Check if Lead` IF:
      - Implement a correct check, e.g., “if no items returned” then create lead.
   3. Add `AI Summarizer1` to produce `{summary, category}`.
   4. Add `Format output` code node to parse it.
   5. Add `Insert Lead` Data Table insert.
   6. Add `Check Human is called` IF `needs_human == true`.
   7. Add `Send Email Notification If Human is called` Gmail node (fill real links or remove placeholders).
   8. Add `Add row for Human Call` Data Table upsert keyed by phone, include description if desired.
   9. Add `Update Row` Data Table update/upsert the memory row:
      - Filter by `id == users_id`
      - Set `reciever` (AI reply) to the agent output.

9) **Section 7: Google Sheets sync**
   1. Add `Schedule Trigger` every minute.
   2. Add 4 Data Table get nodes: slots/leads/human/appointments.
   3. Add 4 Google Sheets appendOrUpdate nodes mapping fields accordingly.
   4. Choose dedupe keys carefully (appointments should likely match by phone+service+date, not phone only).

10) **Section 8: RAG ingestion (manual)**
   1. Add `Manual Trigger`.
   2. Add `Google Docs` get document.
   3. Add `Recursive Character Text Splitter` + `Default Data Loader`.
   4. Add `Embeddings OpenAI`.
   5. Add `Pinecone Vector Store` in insert mode to the **same** index/namespace used by retrieval.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google Sheets CRM referenced in comments | https://docs.google.com/spreadsheets/d/1HCl3CvMnzILIrcjnFK-AwLqWuKfMwezM1yXWkJ4KG9Q/edit?usp=sharing |
| Workflow’s own description sticky note (high-level behavior and requirements) | “🏥🤖 Medical Clinic WhatsApp Agent + CRM Sync” (included in workflow as Sticky Note4) |
| Important consistency warning | Pinecone insert uses index `n8n`, retrieval uses index `clinic` → align indexes/namespaces or RAG retrieval will not work. |
| Important code reference warning | `Format Context` references `Get Slots1` and `Get appointment1` which do not exist in the provided workflow JSON → update those references or slots/appointments context will be empty. |
| WhatsApp sending payload warning | `Send via WhatsApp` stringifies `$json.output`; ensure you’re sending the actual reply text string and not quoted JSON. |

