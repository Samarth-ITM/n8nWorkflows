Manage Google Calendar via Telegram and get daily AI briefings (OpenAI + Gemini)

https://n8nworkflows.xyz/workflows/manage-google-calendar-via-telegram-and-get-daily-ai-briefings--openai---gemini--13309


# Manage Google Calendar via Telegram and get daily AI briefings (OpenAI + Gemini)

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow turns Telegram into a natural-language (and voice) interface for **Google Calendar management**, and also generates a **daily executive briefing** pushed to Telegram. It uses **OpenAI (chat model)** for intent reasoning + tool calling, **Google Gemini** for **voice transcription**, and Google Calendar nodes/tools to execute calendar actions.

**Target use cases:**
- Create/update/delete/reschedule Google Calendar events by texting or sending a voice note in Telegram.
- Automatically prevent conflicts by retrieving calendar events before changes (enforced via agent instructions).
- Receive a daily productivity briefing (meeting load + free “deep work” windows) at a scheduled time.

### Logical Blocks
**1.1 Telegram Intake & Routing**  
Receives Telegram messages and routes based on whether the message contains a voice note.

**1.2 Voice Processing (Telegram → Audio → Gemini Transcription)**  
Downloads the Telegram voice file and transcribes it with Gemini.

**1.3 AI Scheduling Agent (Reasoning + Memory + Calendar Tools)**  
Uses OpenAI chat model + memory to interpret user intent and call Google Calendar “tool” nodes to create/get/update/delete events.

**1.4 Action Response to Telegram**  
Sends the AI agent’s final summary back to the user in Telegram.

**1.5 Daily Executive Briefing (Schedule → Calendar Fetch → Analytics → Telegram)**  
Runs daily, pulls today’s events, computes meeting load and free windows, then sends a briefing message to Telegram.

---

## 2. Block-by-Block Analysis

### 2.1 Telegram Intake & Routing
**Overview:**  
Listens for incoming Telegram messages and decides whether to process as voice (transcribe first) or text (send directly to the AI agent).

**Nodes involved:**
- Receive Bot Messages
- Voice Note or Text?

#### Node: Receive Bot Messages
- **Type / role:** `telegramTrigger` — entrypoint webhook for Telegram updates.
- **Key configuration:**
  - Updates: `message` only.
  - Uses Telegram bot credentials `MyScedualling_Assistant`.
- **Inputs/Outputs:**
  - **Input:** Telegram webhook event.
  - **Output:** Message payload to **Voice Note or Text?**
- **Failure/edge cases:**
  - Telegram webhook not set / revoked token → trigger won’t fire.
  - Bot privacy settings or missing permissions in groups can prevent receiving messages.
- **Version notes:** TypeVersion `1.2` (Telegram Trigger).

#### Node: Voice Note or Text?
- **Type / role:** `switch` — routes the flow to voice processing or directly to the agent.
- **Key configuration:**
  - Rule “Voice” checks **exists**: `{{$json.message.voice.file_id}}`
  - Rule “Text” checks **notExists**: `{{$json.message.voice.file_id}}`
  - Outputs are renamed to “Voice” and “Text”.
- **Inputs/Outputs:**
  - **Input:** From Receive Bot Messages.
  - **Voice output:** → Transform Audio to mp3 format
  - **Text output:** → AI Agent for Google Calendar
- **Failure/edge cases:**
  - Non-standard Telegram updates (e.g., edited messages, non-message updates) are excluded by trigger config.
  - If Telegram sends a message with neither `voice` nor `text` (e.g., sticker), it will follow “Text” path (because `voice.file_id` not exists) but may give poor agent input.

---

### 2.2 Voice Processing (Telegram → Audio → Gemini Transcription)
**Overview:**  
Downloads the Telegram voice note file and submits it to Gemini for transcription. The transcription is passed into the AI agent.

**Nodes involved:**
- Transform Audio to mp3 format
- Transcribe a recording

#### Node: Transform Audio to mp3 format
- **Type / role:** `telegram` (resource: file) — fetches the voice note file content from Telegram.
- **Key configuration:**
  - Resource: `file`
  - `fileId`: `{{ $('Receive Bot Messages').item.json.message.voice.file_id }}`
- **Inputs/Outputs:**
  - **Input:** Voice output from Voice Note or Text?
  - **Output:** Binary file data → Transcribe a recording
- **Failure/edge cases:**
  - File ID missing/expired → Telegram API error.
  - Large files may exceed platform limits or timeout.
  - Despite the node name, conversion to MP3 is not guaranteed by Telegram node alone; it mainly **retrieves** the file. If Gemini expects a certain codec/container, transcription could fail.

#### Node: Transcribe a recording
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — transcribes audio to text.
- **Key configuration:**
  - Model: `models/gemini-2.0-flash`
  - Resource: `audio`
  - Input type: `binary` (expects audio data from previous node)
- **Inputs/Outputs:**
  - **Input:** Binary audio from Transform Audio to mp3 format
  - **Output:** Transcription text payload → AI Agent for Google Calendar
- **Failure/edge cases:**
  - Invalid/unsupported audio format → transcription error.
  - Gemini credential issues (API key invalid/quota exceeded).
  - Output schema differences: the agent prompt references `{{$json.content.parts[0].text}}`, which assumes a Gemini-style response shape; if node output differs, that expression may be empty.

---

### 2.3 AI Scheduling Agent (Reasoning + Memory + Calendar Tools)
**Overview:**  
Central “agent” interprets user text (or transcription), maintains a short memory window, and calls Google Calendar tool nodes to fulfill requests (create/get/update/delete) while following strict rules (e.g., retrieve event ID before updating/deleting).

**Nodes involved:**
- AI Agent for Google Calendar
- OpenAI Chat Model
- Simple Memory
- Create an event in Google Calendar (tool)
- Get many events in Google Calendar (tool)
- Delete an event in Google Calendar (tool)
- Update an event in Google Calendar (tool)

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` — provides the chat LLM used by the agent.
- **Key configuration:**
  - Model: `gpt-5-nano`
  - Built-in tools: none enabled here (agent uses explicit n8n tools).
- **Inputs/Outputs:**
  - Connected to agent via **ai_languageModel** channel.
- **Failure/edge cases:**
  - OpenAI credential invalid / policy blocks / quota exceeded.
  - Model name availability depends on your OpenAI account/region and n8n node version.

#### Node: Simple Memory
- **Type / role:** `memoryBufferWindow` — short conversational memory for the agent.
- **Key configuration:**
  - SessionId type: `customKey`
  - Session key: `5883770591`
  - Context window length: `10`
- **Inputs/Outputs:**
  - Connected to agent via **ai_memory**.
- **Important implication / edge case:**
  - Using a **fixed custom session key** means *all users share the same memory*. If multiple Telegram users message the bot, context may leak between users. Prefer: `={{ $json.message.chat.id }}` as session key.

#### Node: AI Agent for Google Calendar
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates reasoning and tool calls.
- **Key configuration (interpreted):**
  - PromptType: `define` (custom instruction block).
  - Instructions enforce:
    - Google Calendar as source of truth.
    - Check conflicts before creation.
    - Confirm timezone if missing.
    - ISO 8601 API times.
    - Must retrieve event ID via “get event tool” before update/delete.
    - Uses a “Current time reference” in Asia/Dubai via `DateTime.now().setZone('Asia/Dubai').toISO()`.
  - User input area includes both:
    - `{{ $json.content.parts[0].text }}` (Gemini transcription path)
    - `{{ $json.message.text }}` (Telegram text path)
- **Inputs/Outputs:**
  - **Inputs:**
    - From Switch (text path) or Gemini transcription (voice path).
    - AI language model: OpenAI Chat Model.
    - AI memory: Simple Memory.
    - AI tools: the four Google Calendar tool nodes.
  - **Output:** `output` field used by “Send a summary action message”.
- **Failure/edge cases:**
  - If the incoming item doesn’t contain `content.parts[0].text` (text path), that expression may be undefined but harmless if `$json.message.text` exists (and vice versa).
  - If neither exists (sticker/photo), the agent may receive empty input.
  - Tool-call failures (Google auth, missing required fields) will surface as agent errors or partial responses.

#### Node: Create an event in Google Calendar
- **Type / role:** `googleCalendarTool` — tool node invoked by agent to create events.
- **Key configuration:**
  - Calendar: `user@example.com`
  - Start: `{{$fromAI('Start')}}`
  - End: `{{$fromAI('End')}}`
  - Summary: `{{$fromAI('Summary')}}`
  - Use default reminders: `false`
- **Inputs/Outputs:**
  - **Input:** AI tool invocation from agent (ai_tool channel).
  - **Output:** Returns created event details back to agent.
- **Failure/edge cases:**
  - Missing/invalid ISO timestamps → Google API error.
  - Calendar ID wrong or not shared with OAuth account.
  - If timezone not included in ISO string, Google may assume calendar timezone.

#### Node: Get many events in Google Calendar
- **Type / role:** `googleCalendarTool` (operation `getAll`) — tool node to list events (used for availability checks and event ID retrieval).
- **Key configuration:**
  - Operation: `getAll`
  - Limit: `10`
  - `timeMin`: `{{$fromAI('After')}}`
  - `timeMax`: `{{$fromAI('Before')}}`
  - Optional fields: `{{$fromAI('Fields')}}`
- **Inputs/Outputs:**
  - **Input:** From agent tool call.
  - **Output:** Event list to agent.
- **Failure/edge cases:**
  - AI might omit `After/Before` → tool call can fail or return unexpected range.
  - Fields expression empty may be accepted or may cause API parameter issues depending on node implementation.

#### Node: Delete an event in Google Calendar
- **Type / role:** `googleCalendarTool` (operation `delete`) — deletes an event by ID.
- **Key configuration:**
  - Event ID: `{{$fromAI('Event_ID')}}`
- **Inputs/Outputs:**
  - **Input:** Tool call from agent.
  - **Output:** Deletion confirmation to agent.
- **Failure/edge cases:**
  - Wrong/missing event ID → 404 not found.
  - Recurring events: deleting a single instance vs series may require specific IDs; tool may delete instance only depending on ID.

#### Node: Update an event in Google Calendar
- **Type / role:** `googleCalendarTool` (operation `update`) — updates an event by ID.
- **Key configuration:**
  - Event ID: `{{$fromAI('Event_ID')}}`
  - Update fields:
    - Start: `{{$fromAI('Start')}}`
    - End: `{{$fromAI('End')}}`
    - Summary: `{{$fromAI('Summary')}}`
  - Use default reminders: `false`
- **Inputs/Outputs:**
  - **Input:** Tool call from agent.
  - **Output:** Updated event details to agent.
- **Failure/edge cases:**
  - Updating time without checking conflicts is possible if agent misbehaves; instructions try to prevent this.
  - Partial updates: if AI provides empty Start/End/Summary, it may overwrite fields or error depending on node behavior.

---

### 2.4 Action Response to Telegram
**Overview:**  
Sends the agent’s final “what I did / what I found” response back to the originating Telegram chat.

**Nodes involved:**
- Send a summary action message

#### Node: Send a summary action message
- **Type / role:** `telegram` — sends a message to the user.
- **Key configuration:**
  - Text: `{{ $json.output }}`
  - Chat ID: `{{ $('Receive Bot Messages').item.json.message.chat.id }}`
  - Append attribution: `false`
- **Inputs/Outputs:**
  - **Input:** From AI Agent for Google Calendar (main output).
  - **Output:** None used further.
- **Failure/edge cases:**
  - If agent output is not in `$json.output`, message may be blank.
  - If running from a non-Telegram entrypoint (e.g., manual test), the `Receive Bot Messages` reference may be missing.

---

### 2.5 Daily Executive Briefing (Schedule → Calendar Fetch → Analytics → Telegram)
**Overview:**  
At a scheduled time each day, retrieves today’s calendar events (Dubai timezone), computes meeting hours, free windows between 08:00–18:00, flags overload, and sends the summary to a fixed Telegram chat ID.

**Nodes involved:**
- Scheduled Everyday 8 AM
- Retrive all meeting/event that you have
- Daily Analytics Engine
- Send Summary over Telegram

#### Node: Scheduled Everyday 8 AM
- **Type / role:** `scheduleTrigger` — time-based entrypoint.
- **Key configuration:**
  - Interval rule: triggers at hour `11` (despite node name “8 AM”)
- **Inputs/Outputs:**
  - **Output:** → Retrive all meeting/event that you have
- **Edge case / mismatch:**
  - The label says **8 AM**, but configuration triggers at **11** (likely server timezone dependent). If you want 8 AM Dubai, verify:
    - n8n instance timezone
    - schedule trigger settings
    - adjust to 8 and/or set instance timezone appropriately.

#### Node: Retrive all meeting/event that you have
- **Type / role:** `googleCalendar` — fetches today’s events for analytics.
- **Key configuration:**
  - Operation: `getAll`
  - Calendar: `user@example.com`
  - Limit: `20`
  - timeMin: `DateTime.now().setZone('Asia/Dubai').startOf('day').toISO()`
  - timeMax: `DateTime.now().setZone('Asia/Dubai').endOf('day').toISO()`
  - `alwaysOutputData: true` (ensures output even if no events)
- **Inputs/Outputs:**
  - **Input:** From schedule trigger.
  - **Output:** → Daily Analytics Engine
- **Failure/edge cases:**
  - OAuth token expired / revoked.
  - All-day events use `start.date`/`end.date` (handled later in code).
  - If no events, still outputs (by configuration), enabling consistent briefing.

#### Node: Daily Analytics Engine
- **Type / role:** `code` — computes totals and free windows.
- **Key configuration (logic summary):**
  - Converts Google event times to UAE time using a fixed offset `UTC+4`.
  - Work window considered: **08:00 to 18:00 UAE**.
  - Computes:
    - `totalHours` (meeting minutes / 60, 1 decimal)
    - `meetingCount`
    - `freeWindows` (gaps between meetings inside 08–18)
    - `overloaded` flag if `meetingCount > 4`
  - If no meetings: freeWindows becomes `["08:00 AM - 06:00 PM"]`.
- **Inputs/Outputs:**
  - **Input:** Events list items from Google Calendar.
  - **Output:** Single JSON object → Send Summary over Telegram
- **Edge cases / correctness risks:**
  - Uses a hard-coded `UAE_OFFSET = 4` and manual UTC hour shifting; this can break if:
    - input timestamps already include timezone offsets
    - DST changes (UAE does not use DST, but if reused elsewhere it matters)
  - Mixing all-day events (`date`) with timed events can yield durations that represent midnight-to-midnight blocks.

#### Node: Send Summary over Telegram
- **Type / role:** `telegram` — sends daily briefing message.
- **Key configuration:**
  - Chat ID: hard-coded `"123456789"`
  - Text template includes meetingCount, totalHours, freeWindows join, and overload status.
  - Append attribution: `false`
- **Inputs/Outputs:**
  - **Input:** From Daily Analytics Engine.
- **Failure/edge cases:**
  - Hard-coded chat ID must match your user/group/channel; otherwise message delivery fails.
  - Message includes emoji characters; generally fine, but some environments may require UTF-8 consistency.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Receive Bot Messages | telegramTrigger | Entry point: receive Telegram messages | — | Voice Note or Text? | ## AI Scheduling Layer<br>Handles Telegram input, transcription, AI intent parsing, and Google Calendar execution. |
| Voice Note or Text? | switch | Route message to voice transcription or direct AI | Receive Bot Messages | Transform Audio to mp3 format; AI Agent for Google Calendar | ## AI Scheduling Layer<br>Handles Telegram input, transcription, AI intent parsing, and Google Calendar execution. |
| Transform Audio to mp3 format | telegram | Download Telegram voice file as binary | Voice Note or Text? (Voice) | Transcribe a recording | ## Voice Processing<br>Retrieves Telegram voice notes and transcribes them before AI interpretation. |
| Transcribe a recording | googleGemini (LangChain) | Audio transcription (voice → text) | Transform Audio to mp3 format | AI Agent for Google Calendar | ## Voice Processing<br>Retrieves Telegram voice notes and transcribes them before AI interpretation. |
| OpenAI Chat Model | lmChatOpenAi (LangChain) | LLM powering the agent | — (ai_languageModel link) | AI Agent for Google Calendar (as model) | ## AI Scheduling Layer<br>Handles Telegram input, transcription, AI intent parsing, and Google Calendar execution. |
| Simple Memory | memoryBufferWindow (LangChain) | Conversation memory | — (ai_memory link) | AI Agent for Google Calendar (as memory) | ## AI Scheduling Layer<br>Handles Telegram input, transcription, AI intent parsing, and Google Calendar execution. |
| AI Agent for Google Calendar | langchain.agent | Intent parsing + tool calling + final response | Voice Note or Text? (Text) / Transcribe a recording | Send a summary action message | ## AI Scheduling Layer<br>Handles Telegram input, transcription, AI intent parsing, and Google Calendar execution. |
| Create an event in Google Calendar | googleCalendarTool | Tool: create event | AI Agent for Google Calendar (ai_tool) | AI Agent for Google Calendar (tool return) | ## Google Calendar Actions<br>Creates, updates, retrieves, and deletes events with validation. |
| Get many events in Google Calendar | googleCalendarTool | Tool: list events (availability / lookup IDs) | AI Agent for Google Calendar (ai_tool) | AI Agent for Google Calendar (tool return) | ## Google Calendar Actions<br>Creates, updates, retrieves, and deletes events with validation. |
| Delete an event in Google Calendar | googleCalendarTool | Tool: delete event by ID | AI Agent for Google Calendar (ai_tool) | AI Agent for Google Calendar (tool return) | ## Google Calendar Actions<br>Creates, updates, retrieves, and deletes events with validation. |
| Update an event in Google Calendar | googleCalendarTool | Tool: update event by ID | AI Agent for Google Calendar (ai_tool) | AI Agent for Google Calendar (tool return) | ## Google Calendar Actions<br>Creates, updates, retrieves, and deletes events with validation. |
| Send a summary action message | telegram | Reply to user with agent output | AI Agent for Google Calendar | — | ## AI Scheduling Layer<br>Handles Telegram input, transcription, AI intent parsing, and Google Calendar execution. |
| Scheduled Everyday 8 AM | scheduleTrigger | Entry point: daily scheduled run | — | Retrive all meeting/event that you have | ## Daily Executive Briefing<br>Scheduled analysis of today’s events with deep work and overload detection. |
| Retrive all meeting/event that you have | googleCalendar | Fetch today’s calendar events | Scheduled Everyday 8 AM | Daily Analytics Engine | ## Daily Executive Briefing<br>Scheduled analysis of today’s events with deep work and overload detection. |
| Daily Analytics Engine | code | Compute totals + free windows + overload flag | Retrive all meeting/event that you have | Send Summary over Telegram | ## Daily Executive Briefing<br>Scheduled analysis of today’s events with deep work and overload detection. |
| Send Summary over Telegram | telegram | Send daily briefing to a fixed chat ID | Daily Analytics Engine | — | ## Daily Executive Briefing<br>Scheduled analysis of today’s events with deep work and overload detection. |
| Sticky Note | stickyNote | Documentation / workspace annotation | — | — | ## AI Calendar Assistant + Daily Executive Briefing<br>### This workflow turns your Google Calendar into an intelligent scheduling system powered by Telegram and AI...<br>Setup steps include Google Calendar OAuth, Telegram Bot, OpenAI key, Gemini API, Asia/Dubai timezone. |
| Sticky Note1 | stickyNote | Documentation / workspace annotation | — | — |  |
| Sticky Note2 | stickyNote | Documentation / workspace annotation | — | — |  |
| Sticky Note3 | stickyNote | Documentation / workspace annotation | — | — |  |
| Sticky Note4 | stickyNote | Documentation / workspace annotation | — | — |  |

> Note: Sticky notes are annotations; they do not participate in execution. The main large sticky note provides overall intent and setup/customization guidance.

---

## 4. Reproducing the Workflow from Scratch

### Credentials you must create in n8n
1. **Telegram API** credential: connect your bot token (BotFather).
2. **Google Calendar OAuth2** credential: OAuth client configured; grant access to the calendar account.
3. **OpenAI API** credential: API key with access to the chosen model.
4. **Google Gemini (PaLM)** credential: API key for Gemini transcription node.

### Build steps (node-by-node)

#### A) Telegram scheduling assistant (text + voice)
1. **Add Trigger Node:** *Telegram Trigger*  
   - Name: **Receive Bot Messages**  
   - Updates: **message**  
   - Select your Telegram credentials.

2. **Add Switch Node:** *Switch*  
   - Name: **Voice Note or Text?**  
   - Add rule **Voice**:
     - Condition: *exists* on expression `{{$json.message.voice.file_id}}`
   - Add rule **Text**:
     - Condition: *not exists* on expression `{{$json.message.voice.file_id}}`
   - Connect: **Receive Bot Messages → Voice Note or Text?**

3. **Add Telegram Download Node (voice path):** *Telegram*  
   - Name: **Transform Audio to mp3 format**  
   - Resource: **File** (download/get file)  
   - File ID: `{{ $('Receive Bot Messages').item.json.message.voice.file_id }}`  
   - Connect: **Voice Note or Text? (Voice) → Transform Audio to mp3 format**

4. **Add Gemini Transcription Node:** *Google Gemini (LangChain)*  
   - Name: **Transcribe a recording**  
   - Resource: **audio**  
   - Input type: **binary**  
   - Model: `models/gemini-2.0-flash`  
   - Connect: **Transform Audio to mp3 format → Transcribe a recording**

5. **Add OpenAI Model Node:** *OpenAI Chat Model (LangChain)*  
   - Name: **OpenAI Chat Model**  
   - Model: `gpt-5-nano` (or pick an available model in your account)  
   - Select OpenAI credentials.

6. **Add Memory Node:** *Simple Memory / Buffer Window*  
   - Name: **Simple Memory**  
   - Session ID type: **Custom key**  
   - Session key: (recommended) `={{ $json.message.chat.id }}`  
     - (Original workflow uses a fixed value `5883770591`, which can mix user contexts.)
   - Context window length: **10**

7. **Add Agent Node:** *@n8n/n8n-nodes-langchain.agent*  
   - Name: **AI Agent for Google Calendar**  
   - Prompt type: **Define**  
   - Paste an instruction set similar to the provided one, ensuring it includes:
     - “Google Calendar as single source of truth”
     - “Check conflicts before creating”
     - “Retrieve event ID before update/delete”
     - Time reference using `DateTime.now().setZone('Asia/Dubai').toISO()`
   - In the “User Input” section include both:
     - `{{ $json.content.parts[0].text }}` (voice transcription path)
     - `{{ $json.message.text }}` (text path)
   - Connect:
     - **Voice Note or Text? (Text) → AI Agent for Google Calendar**
     - **Transcribe a recording → AI Agent for Google Calendar**
   - Attach:
     - **OpenAI Chat Model** to agent’s **Language Model** input.
     - **Simple Memory** to agent’s **Memory** input.

8. **Add Google Calendar Tool Nodes (for the agent)**
   - Add *Google Calendar Tool* node: **Create an event in Google Calendar**
     - Operation: create (default for tool)
     - Calendar: your calendar (e.g., primary or email)
     - Start: `{{$fromAI('Start')}}`
     - End: `{{$fromAI('End')}}`
     - Summary: `{{$fromAI('Summary')}}`
     - Use default reminders: off (optional)
   - Add *Google Calendar Tool* node: **Get many events in Google Calendar**
     - Operation: **getAll**
     - Limit: 10
     - timeMin: `{{$fromAI('After')}}`
     - timeMax: `{{$fromAI('Before')}}`
     - Fields (optional): `{{$fromAI('Fields')}}`
   - Add *Google Calendar Tool* node: **Delete an event in Google Calendar**
     - Operation: **delete**
     - Event ID: `{{$fromAI('Event_ID')}}`
   - Add *Google Calendar Tool* node: **Update an event in Google Calendar**
     - Operation: **update**
     - Event ID: `{{$fromAI('Event_ID')}}`
     - Update fields: Start/End/Summary from `$fromAI(...)`
   - For each tool node, select **Google Calendar OAuth2** credentials.
   - Connect each tool node to the agent using the **AI Tool** connection (in n8n UI: “Connect to Agent” as tool).

9. **Add Telegram Send Node (agent response):** *Telegram*  
   - Name: **Send a summary action message**  
   - Chat ID: `{{ $('Receive Bot Messages').item.json.message.chat.id }}`  
   - Text: `{{ $json.output }}`  
   - Append attribution: off  
   - Connect: **AI Agent for Google Calendar → Send a summary action message**

#### B) Daily executive briefing (scheduled)
10. **Add Trigger Node:** *Schedule Trigger*  
    - Name: **Scheduled Everyday 8 AM**  
    - Configure to the intended hour.  
      - If you want **08:00 Asia/Dubai**, ensure instance timezone and schedule hour match your environment (the provided config triggers at hour 11).

11. **Add Google Calendar Node:** *Google Calendar*  
    - Name: **Retrive all meeting/event that you have**  
    - Operation: **getAll**  
    - Calendar: same calendar  
    - timeMin: `{{ DateTime.now().setZone('Asia/Dubai').startOf('day').toISO() }}`  
    - timeMax: `{{ DateTime.now().setZone('Asia/Dubai').endOf('day').toISO() }}`  
    - Limit: 20  
    - Enable “Always Output Data” (or equivalent) to handle days with no events.
    - Connect: **Scheduled Everyday 8 AM → Retrive all meeting/event that you have**

12. **Add Code Node:** *Code*  
    - Name: **Daily Analytics Engine**  
    - Paste logic to compute:
      - total meeting hours
      - count of meetings
      - free windows between 08:00 and 18:00 (Dubai)
      - overloaded flag (count > 4)
    - Connect: **Retrive all meeting/event that you have → Daily Analytics Engine**

13. **Add Telegram Send Node:** *Telegram*  
    - Name: **Send Summary over Telegram**  
    - Chat ID: set to your target (user, group, or channel)  
    - Message template using fields from the code node:
      - `{{$json.meetingCount}}`, `{{$json.totalHours}}`, `{{$json.freeWindows.join('\n')}}`, `{{$json.overloaded ? ... : ...}}`
    - Connect: **Daily Analytics Engine → Send Summary over Telegram**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI Calendar Assistant + Daily Executive Briefing… creates/updates/deletes/reschedules via Telegram; daily morning analytics briefing.” | Sticky note (overall workflow intent) |
| Setup steps mentioned: Google Calendar OAuth, Telegram Bot credentials, OpenAI API key, Gemini API key, ensure timezone Asia/Dubai. | Sticky note (setup guidance) |
| Customization suggested: adjust work hours in analytics code; change overload threshold (default: >4 meetings); add CRM tagging for revenue meetings. | Sticky note (customization ideas) |
| Important mismatch to verify: schedule node name says “8 AM” but configuration triggers at hour 11; confirm instance timezone and desired briefing time. | Observed configuration risk |
| Memory session key is static in provided workflow; for multi-user safety, bind memory session to `chat.id`. | Observed configuration risk |