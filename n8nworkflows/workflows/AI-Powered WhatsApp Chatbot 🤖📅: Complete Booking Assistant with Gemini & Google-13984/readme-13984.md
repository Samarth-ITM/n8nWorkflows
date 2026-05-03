AI-Powered WhatsApp Chatbot 🤖📅: Complete Booking Assistant with Gemini & Google

https://n8nworkflows.xyz/workflows/ai-powered-whatsapp-chatbot-------complete-booking-assistant-with-gemini---google-13984


# AI-Powered WhatsApp Chatbot 🤖📅: Complete Booking Assistant with Gemini & Google

# 1. Workflow Overview

This workflow implements an AI-powered booking assistant for a hair salon, accessible through two entry points:

- **WhatsApp** for real customer conversations, supporting both **text and voice notes**
- **n8n chat interface** for testing or embedding a chat-based version

The assistant, named **Emma**, can:

- identify clients
- create or update lightweight client records in Google Sheets
- check appointment availability in Google Calendar
- book, reschedule, or cancel appointments
- escalate unsupported requests through Gmail / human approval tools
- respond in text, or in audio if the incoming WhatsApp message was audio

It also includes:

- a **guardrails layer** to reject unsafe or irrelevant inputs
- **Postgres chat memory** for conversation continuity
- a **workflow-level error handler** that emails execution errors

## 1.1 Input Reception

The workflow has two independent inputs:

- **Chat Trigger** for direct chat messages
- **WhatsApp Trigger** for WhatsApp messages

The WhatsApp path further classifies messages into:
- text
- audio
- unsupported media

## 1.2 Audio Preprocessing

If a WhatsApp message is audio, the workflow:
- retrieves the media URL from WhatsApp
- downloads the audio file
- transcribes it with OpenAI-compatible speech-to-text
- normalizes the transcription into the same message format used by text/chat inputs

## 1.3 Input Normalization and Guardrails

All supported inputs are normalized into a common structure:
- `message`
- `sessionId`
- `type`

The normalized content is then evaluated by a **Guardrails** node powered by Gemini. Safe inputs proceed to the AI agent; rejected content gets a fixed refusal response.

## 1.4 AI Agent and Business Tools

The **Virtual Assistant** agent uses Gemini plus multiple tools to perform business actions:

- Google Sheets lookup for existing clients
- Google Sheets append/update for new clients
- Google Calendar availability lookup
- Google Calendar create/update/delete
- Gmail escalation
- human-in-the-loop escalation tools
- calculator utility
- Postgres memory for chat context

## 1.5 Response Routing

After the AI agent responds, the workflow routes the output depending on channel:

- **Chat input** → sends response to chat
- **WhatsApp text input** → sends text response to WhatsApp
- **WhatsApp audio input** → generates audio from the AI response and sends it back as WhatsApp audio

## 1.6 Error Handling

A dedicated **Error Trigger** catches workflow execution failures and sends an email notification through Gmail.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Chat Input Reception

### Overview
This block receives messages from the n8n chat interface and converts them into the workflow’s normalized message format. It is the non-WhatsApp entry point and is useful for testing or deploying a browser/chat-based assistant.

### Nodes Involved
- When chat message received
- From Chat

### Node Details

#### When chat message received
- **Type / role:** `@n8n/n8n-nodes-langchain.chatTrigger`  
  Entry trigger for chat-based interactions.
- **Configuration choices:**  
  Configured with `responseMode: responseNodes`, meaning downstream response nodes return the reply.
- **Key expressions or variables used:**  
  Uses incoming `chatInput` and `sessionId`.
- **Input / output connections:**  
  No input; outputs to **From Chat**.
- **Version-specific requirements:**  
  Uses node version `1.4`.
- **Edge cases / failures:**  
  Chat UI integration issues, missing session IDs in custom embedding contexts.
- **Sub-workflow reference:**  
  None.

#### From Chat
- **Type / role:** `n8n-nodes-base.set`  
  Maps chat trigger fields into the workflow’s standard structure.
- **Configuration choices:**  
  Creates:
  - `message = {{$json.chatInput}}`
  - `sessionId = {{$json.sessionId}}`
- **Key expressions or variables used:**  
  `{{$json.chatInput}}`, `{{$json.sessionId}}`
- **Input / output connections:**  
  Input from **When chat message received**; output to **Normalize**.
- **Version-specific requirements:**  
  Version `3.4`.
- **Edge cases / failures:**  
  Empty `chatInput`, missing `sessionId`.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Block: WhatsApp Input Reception and Classification

### Overview
This block receives incoming WhatsApp messages and determines whether they are text, voice, or unsupported media. It is the main production input path for customers.

### Nodes Involved
- WhatsApp Trigger
- Input type
- Not supported

### Node Details

#### WhatsApp Trigger
- **Type / role:** `n8n-nodes-base.whatsAppTrigger`  
  Receives inbound WhatsApp Business messages.
- **Configuration choices:**  
  Listens to `messages` updates.
- **Key expressions or variables used:**  
  Downstream nodes read from `messages[0]`.
- **Input / output connections:**  
  No input; outputs to **Input type**.
- **Version-specific requirements:**  
  Version `1`.
- **Edge cases / failures:**  
  Webhook verification issues, expired/invalid WhatsApp OAuth credentials, malformed payloads, unsupported message schemas.
- **Sub-workflow reference:**  
  None.

#### Input type
- **Type / role:** `n8n-nodes-base.switch`  
  Routes WhatsApp payloads by content type.
- **Configuration choices:**  
  Named outputs:
  - `Text` if `messages[0].text.body` exists
  - `Voice` if `messages[0].audio` exists
  - fallback `extra` for unsupported content
- **Key expressions or variables used:**  
  - `{{$json.messages[0].text.body}}`
  - `{{$json.messages[0].audio}}`
- **Input / output connections:**  
  Input from **WhatsApp Trigger**; outputs to:
  - **Message** for text
  - **Get Audio Url** for audio
  - **Not supported** for everything else
- **Version-specific requirements:**  
  Version `3.2`.
- **Edge cases / failures:**  
  WhatsApp payload differences by message type, multiple messages in one webhook, fields missing unexpectedly.
- **Sub-workflow reference:**  
  None.

#### Not supported
- **Type / role:** `n8n-nodes-base.whatsApp`  
  Sends a fallback message for unsupported media types.
- **Configuration choices:**  
  Sends fixed text:
  “You can only send text messages, images, audio files and PDF documents.”
- **Key expressions or variables used:**  
  Recipient uses `{{$('WhatsApp Trigger').item.json.messages[0].from}}`
- **Input / output connections:**  
  Input from **Input type** fallback path; no downstream connection.
- **Version-specific requirements:**  
  Version `1`.
- **Edge cases / failures:**  
  WhatsApp send failures, invalid phone number ID, outbound permission errors.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Block: WhatsApp Audio Retrieval and Transcription

### Overview
This block processes incoming voice notes by resolving the WhatsApp media URL, downloading the audio, transcribing it, and converting it into the same normalized format as text messages.

### Nodes Involved
- Get Audio Url
- Download Audio
- Transcribe Audio
- Audio

### Node Details

#### Get Audio Url
- **Type / role:** `n8n-nodes-base.whatsApp`  
  Retrieves a downloadable media URL for the WhatsApp audio attachment.
- **Configuration choices:**  
  Resource `media`, operation `mediaUrlGet`, using the incoming audio media ID.
- **Key expressions or variables used:**  
  `{{$('WhatsApp Trigger').item.json.messages[0].audio.id}}`
- **Input / output connections:**  
  Input from **Input type** voice output; output to **Download Audio**.
- **Version-specific requirements:**  
  Version `1`.
- **Edge cases / failures:**  
  Expired media IDs, authorization errors, WhatsApp API permission issues.
- **Sub-workflow reference:**  
  None.

#### Download Audio
- **Type / role:** `n8n-nodes-base.httpRequest`  
  Downloads the audio file from the media URL returned by WhatsApp.
- **Configuration choices:**  
  URL is dynamic from `{{$json.url}}`; authentication uses generic credential type with header auth.
- **Key expressions or variables used:**  
  `{{$json.url}}`
- **Input / output connections:**  
  Input from **Get Audio Url**; output to **Transcribe Audio**.
- **Version-specific requirements:**  
  Version `4.2`.
- **Edge cases / failures:**  
  URL expiration, wrong auth header, binary download issues, rate limits.
- **Sub-workflow reference:**  
  None.

#### Transcribe Audio
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi`  
  Speech-to-text transcription node.
- **Configuration choices:**  
  Resource `audio`, operation `transcribe`. Uses an OpenAI-compatible credential named “OpenRouter account”.
- **Key expressions or variables used:**  
  Consumes binary audio from previous node.
- **Input / output connections:**  
  Input from **Download Audio**; output to **Audio**.
- **Version-specific requirements:**  
  Version `1.8`. Compatibility depends on the provider actually supporting audio transcription through the configured OpenAI-compatible endpoint.
- **Edge cases / failures:**  
  Unsupported file format, provider incompatibility, size limits, auth failures, no binary property detected.
- **Sub-workflow reference:**  
  None.

#### Audio
- **Type / role:** `n8n-nodes-base.set`  
  Standardizes transcribed audio into the same format used elsewhere.
- **Configuration choices:**  
  Creates:
  - `message = {{$json.text}}`
  - `sessionId = {{$('WhatsApp Trigger').item.json.messages[0].from}}`
  - `type = whatsapp`
- **Key expressions or variables used:**  
  `{{$json.text}}`, `{{$('WhatsApp Trigger').item.json.messages[0].from}}`
- **Input / output connections:**  
  Input from **Transcribe Audio**; output to **Normalize**.
- **Version-specific requirements:**  
  Version `3.4`.
- **Edge cases / failures:**  
  Empty transcript, poor transcription quality, missing `text` field.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Block: Text Normalization

### Overview
This block converts all supported inputs into a unified schema so that downstream AI and routing logic can operate independently of the source channel.

### Nodes Involved
- Message
- Normalize

### Node Details

#### Message
- **Type / role:** `n8n-nodes-base.set`  
  Standardizes inbound WhatsApp text messages.
- **Configuration choices:**  
  Creates:
  - `message = {{$('WhatsApp Trigger').item.json.messages[0].text.body}}`
  - `sessionId = {{$('WhatsApp Trigger').item.json.messages[0].from}}`
  - `type = whatsapp`
- **Key expressions or variables used:**  
  `{{$('WhatsApp Trigger').item.json.messages[0].text.body}}`
- **Input / output connections:**  
  Input from **Input type** text output; output to **Normalize**.
- **Version-specific requirements:**  
  Version `3.4`.
- **Edge cases / failures:**  
  Missing body text, schema mismatch in WhatsApp payload.
- **Sub-workflow reference:**  
  None.

#### Normalize
- **Type / role:** `n8n-nodes-base.set`  
  Final normalization stage shared by both chat and WhatsApp input paths.
- **Configuration choices:**  
  Copies:
  - `message = {{$json.message}}`
  - `sessionId = {{$json.sessionId}}`
  - `type = {{$json.type}}`
- **Key expressions or variables used:**  
  `{{$json.message}}`, `{{$json.sessionId}}`, `{{$json.type}}`
- **Input / output connections:**  
  Input from **Message**, **Audio**, or **From Chat**; output to **Guardrails**.
- **Version-specific requirements:**  
  Version `3.4`.
- **Edge cases / failures:**  
  If upstream omitted `type`, the later channel switch can fail to route correctly.
- **Sub-workflow reference:**  
  None.

---

## 2.5 Block: Guardrails and Refusal Responses

### Overview
This block screens normalized user input before it reaches the booking agent. Approved messages are sent to the AI agent; rejected ones receive a fixed refusal message through the corresponding channel.

### Nodes Involved
- Guardrails
- Google Gemini Chat Model1
- Send message1
- Chat1

### Node Details

#### Guardrails
- **Type / role:** `@n8n/n8n-nodes-langchain.guardrails`  
  AI safety and relevance gate.
- **Configuration choices:**  
  Evaluates `{{$json.message}}`. Jailbreak detection is enabled with threshold `0.7` and customized system message support.
- **Key expressions or variables used:**  
  `{{$json.message}}`
- **Input / output connections:**  
  Input from **Normalize**.  
  Output 0 → **Virtual Assistant**  
  Output 1 → **Chat1**
- **Version-specific requirements:**  
  Version `2`.
- **Edge cases / failures:**  
  False positives, false negatives, model latency, credential issues with Gemini model.
- **Sub-workflow reference:**  
  None.

#### Google Gemini Chat Model1
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Language model connected to the Guardrails node.
- **Configuration choices:**  
  Uses model `models/gemini-2.5-flash-lite`.
- **Key expressions or variables used:**  
  None directly in parameters beyond model selection.
- **Input / output connections:**  
  Connected to **Guardrails** as `ai_languageModel`.
- **Version-specific requirements:**  
  Version `1`.
- **Edge cases / failures:**  
  Gemini API quota/authentication issues, model availability changes.
- **Sub-workflow reference:**  
  None.

#### Chat1
- **Type / role:** `@n8n/n8n-nodes-langchain.chat`  
  Returns a refusal response for blocked chat inputs.
- **Configuration choices:**  
  Fixed message: “What was communicated does not reflect our policies or is not relevant.”
- **Key expressions or variables used:**  
  Static text.
- **Input / output connections:**  
  Input from blocked branch of **Guardrails**.
- **Version-specific requirements:**  
  Version `1.2`.
- **Edge cases / failures:**  
  Only suitable for chat mode; it is not connected to WhatsApp output.
- **Sub-workflow reference:**  
  None.

#### Send message1
- **Type / role:** `n8n-nodes-base.whatsApp`  
  Intended WhatsApp refusal response.
- **Configuration choices:**  
  Sends fixed text mirroring the refusal.
- **Key expressions or variables used:**  
  Recipient is `{{$('WhatsApp Trigger').item.json.messages[0].from}}`
- **Input / output connections:**  
  **No incoming connection in this workflow JSON.**
- **Version-specific requirements:**  
  Version `1`.
- **Edge cases / failures:**  
  As currently disconnected, it never executes. This is likely an implementation oversight.
- **Sub-workflow reference:**  
  None.

---

## 2.6 Block: AI Agent, Memory, and Business Tools

### Overview
This is the core intelligence block. The Virtual Assistant uses Gemini, memory, and multiple tools to identify the client, inspect availability, manage appointments, and escalate unsupported requests.

### Nodes Involved
- Virtual Assistant
- Google Gemini Chat Model
- Postgres Chat Memory
- New client
- Add client
- Get events
- Create event
- Update event
- Delete event
- Calculator
- Send Email
- Escalation
- Escalation1

### Node Details

#### Virtual Assistant
- **Type / role:** `@n8n/n8n-nodes-langchain.agent`  
  Main conversational agent orchestrating all business logic.
- **Configuration choices:**  
  Uses `{{$json.guardrailsInput}}` as input text.  
  Includes a detailed system prompt defining:
  - assistant identity: Emma from Cuts & Styles
  - mandatory client identification
  - booking/rescheduling/cancel flows
  - date logic
  - escalation rules
  - salon context
- **Key expressions or variables used:**  
  `{{$json.guardrailsInput}}`  
  Prompt references current date/time through `{{$now.toFormat(...)}}`.
- **Input / output connections:**  
  Main input from **Guardrails**; main output to **Switch**.  
  Tool/model/memory connections:
  - **Google Gemini Chat Model**
  - **Postgres Chat Memory**
  - **New client**
  - **Add client**
  - **Get events**
  - **Create event**
  - **Update event**
  - **Delete event**
  - **Calculator**
  - **Escalation1**
- **Version-specific requirements:**  
  Version `3.1`.
- **Edge cases / failures:**  
  Tool misuse by model, ambiguous date interpretation, missing client details, prompt-tool mismatch, malformed tool arguments from AI.
- **Sub-workflow reference:**  
  None.

#### Google Gemini Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Primary LLM for the agent.
- **Configuration choices:**  
  Uses model `models/gemini-3.1-flash-lite-preview`.
- **Key expressions or variables used:**  
  None.
- **Input / output connections:**  
  Connected to **Virtual Assistant** as `ai_languageModel`.
- **Version-specific requirements:**  
  Version `1`.
- **Edge cases / failures:**  
  Preview model stability, quota issues, API auth errors.
- **Sub-workflow reference:**  
  None.

#### Postgres Chat Memory
- **Type / role:** `@n8n/n8n-nodes-langchain.memoryPostgresChat`  
  Stores and retrieves chat history per session.
- **Configuration choices:**  
  Session key is `{{$('Normalize').item.json.sessionId}}`; custom session ID mode.
- **Key expressions or variables used:**  
  `{{$('Normalize').item.json.sessionId}}`
- **Input / output connections:**  
  Connected to **Virtual Assistant** as `ai_memory`.
- **Version-specific requirements:**  
  Version `1.3`.
- **Edge cases / failures:**  
  Database connection problems, missing schema/tables, high latency, inconsistent session keys.
- **Sub-workflow reference:**  
  None.

#### New client
- **Type / role:** `n8n-nodes-base.googleSheetsTool`  
  Tool used by the agent to query the CRM sheet for existing clients.
- **Configuration choices:**  
  Reads from spreadsheet `Appointments`, sheet `Foglio1`. Tool description indicates row retrieval.
- **Key expressions or variables used:**  
  No explicit filter is preconfigured in JSON; the AI tool call is expected to provide lookup context.
- **Input / output connections:**  
  Connected to **Virtual Assistant** as `ai_tool`.
- **Version-specific requirements:**  
  Version `4.7`.
- **Edge cases / failures:**  
  Weak lookup specificity, sheet structure mismatches, auth issues, inefficient searches on larger sheets.
- **Sub-workflow reference:**  
  None.

#### Add client
- **Type / role:** `n8n-nodes-base.googleSheetsTool`  
  Append-or-update client record in Google Sheets.
- **Configuration choices:**  
  Operation `appendOrUpdate`, matching column `Phone Number`.  
  Writes:
  - `Client` from AI
  - `Service` from AI
  - `Phone Number` from `{{$('Normalize').item.json.sessionId}}`
- **Key expressions or variables used:**  
  - `$fromAI('Client', ...)`
  - `$fromAI('Service', ...)`
  - `{{$('Normalize').item.json.sessionId}}`
- **Input / output connections:**  
  Connected to **Virtual Assistant** as `ai_tool`.
- **Version-specific requirements:**  
  Version `4.7`.
- **Edge cases / failures:**  
  Wrong phone-session mapping for non-WhatsApp chat users, missing sheet columns, auth errors.
- **Sub-workflow reference:**  
  None.

#### Get events
- **Type / role:** `n8n-nodes-base.googleCalendarTool`  
  Reads calendar events to check availability and existing bookings.
- **Configuration choices:**  
  Operation `getAll`, return all, timezone `Europe/Madrid`, time window until 30 days from now.  
  Tool description instructs the AI that returned data means the slot is busy.
- **Key expressions or variables used:**  
  `{{$now.plus({ days: 30 }).toFormat('yyyy-MM-dd')}}`
- **Input / output connections:**  
  Connected to **Virtual Assistant** as `ai_tool`.
- **Version-specific requirements:**  
  Version `1.3`.
- **Edge cases / failures:**  
  Timezone ambiguity, large result set, calendar auth errors, no filtering by start/min in node config beyond AI prompt guidance.
- **Sub-workflow reference:**  
  None.

#### Create event
- **Type / role:** `n8n-nodes-base.googleCalendarTool`  
  Creates a calendar appointment.
- **Configuration choices:**  
  Calendar is fixed. AI must provide:
  - `Start`
  - `End`
  - `Summary`
  - `Description`
  
  Tool description tells the model to include client name and service in the event.
- **Key expressions or variables used:**  
  `$fromAI('Start')`, `$fromAI('End')`, `$fromAI('Summary')`, `$fromAI('Description')`
- **Input / output connections:**  
  Connected to **Virtual Assistant** as `ai_tool`.
- **Version-specific requirements:**  
  Version `1.3`.
- **Edge cases / failures:**  
  Invalid ISO datetimes, overlaps not prechecked, missing summary, calendar permission issues.
- **Sub-workflow reference:**  
  None.

#### Update event
- **Type / role:** `n8n-nodes-base.googleCalendarTool`  
  Reschedules an existing appointment.
- **Configuration choices:**  
  Operation `update`. AI must supply:
  - `Event_ID`
  - `Start`
  - `End`
  
  Tool description explicitly requires end time to be 30 minutes after start unless otherwise specified.
- **Key expressions or variables used:**  
  `$fromAI('Event_ID')`, `$fromAI('Start')`, `$fromAI('End')`
- **Input / output connections:**  
  Connected to **Virtual Assistant** as `ai_tool`.
- **Version-specific requirements:**  
  Version `1.3`.
- **Edge cases / failures:**  
  Wrong event selected, missing event ID, invalid datetime formats, race conditions if calendar changed after lookup.
- **Sub-workflow reference:**  
  None.

#### Delete event
- **Type / role:** `n8n-nodes-base.googleCalendarTool`  
  Cancels an existing appointment.
- **Configuration choices:**  
  Operation `delete`, AI must pass `Event_ID`.
- **Key expressions or variables used:**  
  `$fromAI('Event_ID')`
- **Input / output connections:**  
  Connected to **Virtual Assistant** as `ai_tool`.
- **Version-specific requirements:**  
  Version `1.3`.
- **Edge cases / failures:**  
  Wrong event deletion, stale event ID, missing permissions.
- **Sub-workflow reference:**  
  None.

#### Calculator
- **Type / role:** `@n8n/n8n-nodes-langchain.toolCalculator`  
  Utility tool for arithmetic/date-related simple reasoning support.
- **Configuration choices:**  
  Default configuration.
- **Key expressions or variables used:**  
  None.
- **Input / output connections:**  
  Connected to **Virtual Assistant** as `ai_tool`.
- **Version-specific requirements:**  
  Version `1`.
- **Edge cases / failures:**  
  Limited usefulness for actual calendar math versus model-native reasoning.
- **Sub-workflow reference:**  
  None.

#### Send Email
- **Type / role:** `n8n-nodes-base.gmailTool`  
  Tool that emails the team when escalation is needed.
- **Configuration choices:**  
  Sends to `xxx@xxx.com`, subject `Escaletion`, body from AI-generated `Message`. Attribution disabled.
- **Key expressions or variables used:**  
  `$fromAI('Message', ...)`
- **Input / output connections:**  
  Connected as `ai_tool` to both **Escalation** and **Escalation1** rather than directly to the main agent.
- **Version-specific requirements:**  
  Version `2.2`.
- **Edge cases / failures:**  
  Typo in subject, invalid recipient, Gmail quota/auth errors.
- **Sub-workflow reference:**  
  None.

#### Escalation
- **Type / role:** `n8n-nodes-base.whatsAppHitlTool`  
  Human-in-the-loop approval tool for WhatsApp escalation.
- **Configuration choices:**  
  Sends a formatted summary of the intended tool call and parameters. Approval type is `double`. Recipient phone number is expected from AI input.
- **Key expressions or variables used:**  
  `$tool.name`, `$tool.parameters[...]`, `$fromAI('Recipient_s_Phone_Number', ...)`
- **Input / output connections:**  
  Receives **Send Email** as an AI tool, but is **not connected back to Virtual Assistant**.
- **Version-specific requirements:**  
  Version `1.1`.
- **Edge cases / failures:**  
  As configured, it is effectively unused by the main agent. Also depends on correct AI-supplied phone number.
- **Sub-workflow reference:**  
  None.

#### Escalation1
- **Type / role:** `@n8n/n8n-nodes-langchain.chatHitlTool`  
  Human approval tool for chat-based escalation.
- **Configuration choices:**  
  Sends formatted pending action details, double approval, and a maximum wait time of 45 minutes. `blockUserInput` is AI-controlled.
- **Key expressions or variables used:**  
  `$tool.name`, `$tool.parameters[...]`, `$fromAI('Block_User_Input', ..., 'boolean')`
- **Input / output connections:**  
  Connected to **Virtual Assistant** as `ai_tool`.  
  Also connected to **Send Email** as a child tool.
- **Version-specific requirements:**  
  Version `1.2`.
- **Edge cases / failures:**  
  Approval timeout, blocked conversation if no reviewer responds, complex escalation chain.
- **Sub-workflow reference:**  
  None.

---

## 2.7 Block: Response Routing and Delivery

### Overview
This block decides whether the AI response should go back to chat, WhatsApp text, or WhatsApp audio. It preserves modality for WhatsApp audio messages by returning speech.

### Nodes Involved
- Switch
- From audio to audio?
- Send message
- Generate Audio Response
- Fix mimeType for Audio
- Send audio
- Chat

### Node Details

#### Switch
- **Type / role:** `n8n-nodes-base.switch`  
  Routes the AI output by normalized input channel.
- **Configuration choices:**  
  Output `WhatsApp` if `{{$('Normalize').item.json.type}} == whatsapp`  
  Output `Chat` if `... == chat`
- **Key expressions or variables used:**  
  `{{$('Normalize').item.json.type}}`
- **Input / output connections:**  
  Input from **Virtual Assistant**; outputs to:
  - **From audio to audio?** for WhatsApp
  - **Chat** for chat
- **Version-specific requirements:**  
  Version `3.4`.
- **Edge cases / failures:**  
  Missing/incorrect `type` field results in no routing match.
- **Sub-workflow reference:**  
  None.

#### From audio to audio?
- **Type / role:** `n8n-nodes-base.if`  
  Detects whether the original WhatsApp message was audio.
- **Configuration choices:**  
  Checks existence of `{{$('WhatsApp Trigger').item.json.messages[0].audio}}`
- **Key expressions or variables used:**  
  `{{$('WhatsApp Trigger').item.json.messages[0].audio}}`
- **Input / output connections:**  
  Input from **Switch** WhatsApp branch.  
  True → **Generate Audio Response**  
  False → **Send message**
- **Version-specific requirements:**  
  Version `2.2`.
- **Edge cases / failures:**  
  Because it references the trigger directly, this node is tightly coupled to WhatsApp context and would not be reusable elsewhere.
- **Sub-workflow reference:**  
  None.

#### Send message
- **Type / role:** `n8n-nodes-base.whatsApp`  
  Sends plain text reply to WhatsApp.
- **Configuration choices:**  
  Sends `{{$json.output}}` to the sender number from the original WhatsApp message.
- **Key expressions or variables used:**  
  `{{$json.output}}`, `{{$('WhatsApp Trigger').item.json.messages[0].from}}`
- **Input / output connections:**  
  Input from **From audio to audio?** false branch.
- **Version-specific requirements:**  
  Version `1`.
- **Edge cases / failures:**  
  WhatsApp send failure, malformed text, outbound API rate limits.
- **Sub-workflow reference:**  
  None.

#### Generate Audio Response
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi`  
  Text-to-speech generation for WhatsApp audio replies.
- **Configuration choices:**  
  Resource `audio`, input is `{{$('AI Agent1').item.json.output}}`, voice `onyx`.
- **Key expressions or variables used:**  
  `{{$('AI Agent1').item.json.output}}`
- **Input / output connections:**  
  Input from **From audio to audio?** true branch; output to **Fix mimeType for Audio**.
- **Version-specific requirements:**  
  Version `1.8`.
- **Edge cases / failures:**  
  **Important issue:** expression references `AI Agent1`, but the actual agent node is named **Virtual Assistant**. As written, this expression will fail unless a node was renamed previously and the JSON is inconsistent.
- **Sub-workflow reference:**  
  None.

#### Fix mimeType for Audio
- **Type / role:** `n8n-nodes-base.code`  
  Adjusts binary metadata so WhatsApp accepts the generated audio file.
- **Configuration choices:**  
  Iterates over all binary properties and rewrites `audio/mp3` to `audio/mpeg`.
- **Key expressions or variables used:**  
  JavaScript over `$input.all()`
- **Input / output connections:**  
  Input from **Generate Audio Response**; output to **Send audio**.
- **Version-specific requirements:**  
  Version `2`.
- **Edge cases / failures:**  
  If the TTS provider returns a different mime type or no binary data, the code may do nothing.
- **Sub-workflow reference:**  
  None.

#### Send audio
- **Type / role:** `n8n-nodes-base.whatsApp`  
  Sends generated binary audio as a WhatsApp audio message.
- **Configuration choices:**  
  Uses `messageType: audio`, media source `useMedian8n`.
- **Key expressions or variables used:**  
  Recipient is `{{$('Input type').item.json.contacts[0].wa_id}}`
- **Input / output connections:**  
  Input from **Fix mimeType for Audio**.
- **Version-specific requirements:**  
  Version `1`.
- **Edge cases / failures:**  
  Missing binary payload, wrong recipient field, unsupported mime type.
- **Sub-workflow reference:**  
  None.

#### Chat
- **Type / role:** `@n8n/n8n-nodes-langchain.chat`  
  Returns the AI reply to the chat interface.
- **Configuration choices:**  
  Sends `{{$json.output}}`
- **Key expressions or variables used:**  
  `{{$json.output}}`
- **Input / output connections:**  
  Input from **Switch** chat branch.
- **Version-specific requirements:**  
  Version `1.2`.
- **Edge cases / failures:**  
  Empty agent output.
- **Sub-workflow reference:**  
  None.

---

## 2.8 Block: Error Handling

### Overview
This block catches workflow execution errors globally and emails a notification with the error message and last executed node.

### Nodes Involved
- Workflow Error Handler
- Notify: Workflow Error

### Node Details

#### Workflow Error Handler
- **Type / role:** `n8n-nodes-base.errorTrigger`  
  Starts a separate error-handling execution whenever the workflow fails.
- **Configuration choices:**  
  No extra parameters.
- **Key expressions or variables used:**  
  Downstream node uses `execution.error.message` and `execution.lastNodeExecuted`.
- **Input / output connections:**  
  No input; outputs to **Notify: Workflow Error**.
- **Version-specific requirements:**  
  Version `1`.
- **Edge cases / failures:**  
  Trigger only works for workflow errors, not for logically wrong but technically successful runs.
- **Sub-workflow reference:**  
  None.

#### Notify: Workflow Error
- **Type / role:** `n8n-nodes-base.gmail`  
  Sends an email alert on workflow failure.
- **Configuration choices:**  
  Sends to `user@example.com`. HTML body includes the error message and last node.
- **Key expressions or variables used:**  
  - `{{$json.execution.error.message}}`
  - `{{$json.execution.lastNodeExecuted}}`
- **Input / output connections:**  
  Input from **Workflow Error Handler**.
- **Version-specific requirements:**  
  Version `2.1`.
- **Edge cases / failures:**  
  Gmail auth issues. Also the subject/body still reference a different workflow (“LinkedIn PDF → JobAdder Sync”), which should be corrected.
- **Sub-workflow reference:**  
  None.

---

## 2.9 Block: Documentation / Sticky Notes

### Overview
These nodes are non-executable annotations that explain setup, steps, and external resources inside the canvas.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7
- Sticky Note8
- Sticky Note9

### Node Details
All of these are `n8n-nodes-base.stickyNote` nodes. They do not affect execution, but they provide useful context:

- overall workflow description and setup guidance
- block labels for chat input, WhatsApp input, guardrails, AI agent, escalation, response sections, and error handling
- a Google Sheets cloning link
- a YouTube channel link

They have no inputs/outputs and no runtime failure modes.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Update event | n8n-nodes-base.googleCalendarTool | Update existing calendar event | — (AI tool) | Virtual Assistant (AI tool link) | ## STEP 3  - AI Agent<br>[Clone this sheet](https://docs.google.com/spreadsheets/d/1DM24yhANppmCu92D3St6GpMh2He7L2feizajN0aLDrU/edit?usp=sharing) for example of simple database |
| Delete event | n8n-nodes-base.googleCalendarTool | Delete calendar event | — (AI tool) | Virtual Assistant (AI tool link) | ## STEP 3  - AI Agent<br>[Clone this sheet](https://docs.google.com/spreadsheets/d/1DM24yhANppmCu92D3St6GpMh2He7L2feizajN0aLDrU/edit?usp=sharing) for example of simple database |
| Create event | n8n-nodes-base.googleCalendarTool | Create calendar event | — (AI tool) | Virtual Assistant (AI tool link) | ## STEP 3  - AI Agent<br>[Clone this sheet](https://docs.google.com/spreadsheets/d/1DM24yhANppmCu92D3St6GpMh2He7L2feizajN0aLDrU/edit?usp=sharing) for example of simple database |
| Get events | n8n-nodes-base.googleCalendarTool | List calendar events / availability | — (AI tool) | Virtual Assistant (AI tool link) | ## STEP 3  - AI Agent<br>[Clone this sheet](https://docs.google.com/spreadsheets/d/1DM24yhANppmCu92D3St6GpMh2He7L2feizajN0aLDrU/edit?usp=sharing) for example of simple database |
| Add client | n8n-nodes-base.googleSheetsTool | Append or update client record | — (AI tool) | Virtual Assistant (AI tool link) | ## STEP 3  - AI Agent<br>[Clone this sheet](https://docs.google.com/spreadsheets/d/1DM24yhANppmCu92D3St6GpMh2He7L2feizajN0aLDrU/edit?usp=sharing) for example of simple database |
| Workflow Error Handler | n8n-nodes-base.errorTrigger | Trigger on workflow execution error | — | Notify: Workflow Error | ## STEP 5 - Error |
| Notify: Workflow Error | n8n-nodes-base.gmail | Send workflow error email | Workflow Error Handler | — | ## STEP 5 - Error |
| Virtual Assistant | @n8n/n8n-nodes-langchain.agent | Main AI booking agent | Guardrails | Switch | ## STEP 3  - AI Agent<br>[Clone this sheet](https://docs.google.com/spreadsheets/d/1DM24yhANppmCu92D3St6GpMh2He7L2feizajN0aLDrU/edit?usp=sharing) for example of simple database |
| WhatsApp Trigger | n8n-nodes-base.whatsAppTrigger | Receive inbound WhatsApp messages | — | Input type | ## STEP 1 bis - WhatsApp Input<br>Message (text or audio with speech to text) from WhatsApp |
| Download Audio | n8n-nodes-base.httpRequest | Download WhatsApp audio file | Get Audio Url | Transcribe Audio | ## STEP 1 bis - WhatsApp Input<br>Message (text or audio with speech to text) from WhatsApp |
| Transcribe Audio | @n8n/n8n-nodes-langchain.openAi | Transcribe audio to text | Download Audio | Audio | ## STEP 1 bis - WhatsApp Input<br>Message (text or audio with speech to text) from WhatsApp |
| Fix mimeType for Audio | n8n-nodes-base.code | Normalize TTS audio mime type | Generate Audio Response | Send audio | ## STEP 4 - WhatsApp Response |
| Send message | n8n-nodes-base.whatsApp | Send WhatsApp text reply | From audio to audio? | — | ## STEP 4 - WhatsApp Response |
| Send audio | n8n-nodes-base.whatsApp | Send WhatsApp audio reply | Fix mimeType for Audio | — | ## STEP 4 - WhatsApp Response |
| Audio | n8n-nodes-base.set | Normalize transcribed audio input | Transcribe Audio | Normalize | ## STEP 1 bis - WhatsApp Input<br>Message (text or audio with speech to text) from WhatsApp |
| Not supported | n8n-nodes-base.whatsApp | Send unsupported-media fallback | Input type | — | ## STEP 1 bis - WhatsApp Input<br>Message (text or audio with speech to text) from WhatsApp |
| Get Audio Url | n8n-nodes-base.whatsApp | Resolve WhatsApp media URL | Input type | Download Audio | ## STEP 1 bis - WhatsApp Input<br>Message (text or audio with speech to text) from WhatsApp |
| Generate Audio Response | @n8n/n8n-nodes-langchain.openAi | Generate speech from AI reply | From audio to audio? | Fix mimeType for Audio | ## STEP 4 - WhatsApp Response |
| From audio to audio? | n8n-nodes-base.if | Decide between WhatsApp text or audio reply | Switch | Generate Audio Response, Send message | ## STEP 4 - WhatsApp Response |
| Input type | n8n-nodes-base.switch | Classify WhatsApp message type | WhatsApp Trigger | Message, Get Audio Url, Not supported | ## STEP 1 bis - WhatsApp Input<br>Message (text or audio with speech to text) from WhatsApp |
| Calculator | @n8n/n8n-nodes-langchain.toolCalculator | Utility calculation tool for agent | — (AI tool) | Virtual Assistant (AI tool link) | ## STEP 3  - AI Agent<br>[Clone this sheet](https://docs.google.com/spreadsheets/d/1DM24yhANppmCu92D3St6GpMh2He7L2feizajN0aLDrU/edit?usp=sharing) for example of simple database |
| Send Email | n8n-nodes-base.gmailTool | Email escalation tool | — (AI tool via escalation tools) | Escalation, Escalation1 | ## STEP 3 bis - Escalation<br>Escalation with Chatbot or WhatsApp (select one) |
| Escalation | n8n-nodes-base.whatsAppHitlTool | WhatsApp human-in-the-loop escalation | — | — | ## STEP 3 bis - Escalation<br>Escalation with Chatbot or WhatsApp (select one) |
| New client | n8n-nodes-base.googleSheetsTool | Lookup client in sheet | — (AI tool) | Virtual Assistant (AI tool link) | ## STEP 3  - AI Agent<br>[Clone this sheet](https://docs.google.com/spreadsheets/d/1DM24yhANppmCu92D3St6GpMh2He7L2feizajN0aLDrU/edit?usp=sharing) for example of simple database |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM for main agent | — | Virtual Assistant (AI language model link) | ## STEP 3  - AI Agent<br>[Clone this sheet](https://docs.google.com/spreadsheets/d/1DM24yhANppmCu92D3St6GpMh2He7L2feizajN0aLDrU/edit?usp=sharing) for example of simple database |
| Guardrails | @n8n/n8n-nodes-langchain.guardrails | Filter unsafe / irrelevant inputs | Normalize | Virtual Assistant, Chat1 | ## STEP 2  - Guardrails |
| Google Gemini Chat Model1 | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM for guardrails node | — | Guardrails (AI language model link) | ## STEP 2  - Guardrails |
| Send message1 | n8n-nodes-base.whatsApp | WhatsApp policy refusal response | — | — | ## STEP 2  - Guardrails |
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Receive chat-based input | — | From Chat | ## STEP 1 - Chat Input<br>Message from chatbot |
| Postgres Chat Memory | @n8n/n8n-nodes-langchain.memoryPostgresChat | Conversation memory storage | — | Virtual Assistant (AI memory link) | ## STEP 3  - AI Agent<br>[Clone this sheet](https://docs.google.com/spreadsheets/d/1DM24yhANppmCu92D3St6GpMh2He7L2feizajN0aLDrU/edit?usp=sharing) for example of simple database |
| Normalize | n8n-nodes-base.set | Standardize all supported inputs | Audio, Message, From Chat | Guardrails | ## STEP 2  - Guardrails |
| Switch | n8n-nodes-base.switch | Route output to WhatsApp or chat | Virtual Assistant | From audio to audio?, Chat |  |
| Escalation1 | @n8n/n8n-nodes-langchain.chatHitlTool | Chat human-in-the-loop escalation | — (AI tool) | Virtual Assistant (AI tool link) | ## STEP 3 bis - Escalation<br>Escalation with Chatbot or WhatsApp (select one) |
| Chat | @n8n/n8n-nodes-langchain.chat | Return AI reply to chat | Switch | — | ## STEP 4 bis - Chatbot Response |
| Chat1 | @n8n/n8n-nodes-langchain.chat | Return refusal reply to chat | Guardrails | — | ## STEP 2  - Guardrails |
| Message | n8n-nodes-base.set | Normalize WhatsApp text input | Input type | Normalize | ## STEP 1 bis - WhatsApp Input<br>Message (text or audio with speech to text) from WhatsApp |
| From Chat | n8n-nodes-base.set | Normalize chat trigger payload | When chat message received | Normalize | ## STEP 1 - Chat Input<br>Message from chatbot |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## STEP 1 - Chat Input<br>Message from chatbot |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## STEP 1 bis - WhatsApp Input<br>Message (text or audio with speech to text) from WhatsApp |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## STEP 2  - Guardrails |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## STEP 3  - AI Agent<br>[Clone this sheet](https://docs.google.com/spreadsheets/d/1DM24yhANppmCu92D3St6GpMh2He7L2feizajN0aLDrU/edit?usp=sharing) for example of simple database |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## STEP 3 bis - Escalation<br>Escalation with Chatbot or WhatsApp (select one) |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## STEP 4 - WhatsApp Response |
| Sticky Note7 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## STEP 4 bis - Chatbot Response |
| Sticky Note8 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## AI-Powered WhatsApp Chatbot<br>This workflow implements an **AI-powered WhatsApp booking assistant** for a hair salon. The system allows customers to **book, reschedule, or cancel appointments automatically via text or voice messages on WhatsApp**.<br><br>The workflow supports **both text and voice messages**.<br><br>### How it works<br><br>This workflow acts as an **AI-powered WhatsApp booking assistant** that processes both **text and voice messages**. Incoming messages from WhatsApp or a chat interface are captured, and voice notes are automatically downloaded and transcribed using **OpenAI Whisper**. The message then passes through a **Guardrails layer (Google Gemini)** that filters unsafe or policy-violating inputs before reaching the AI agent.<br><br>The **AI Agent (“Emma”)** interprets the user’s intent and decides which tools to call. It checks or creates clients in **Google Sheets**, manages appointments in **Google Calendar** (check availability, create, update, cancel), and escalates unsupported requests via **Gmail or human-in-the-loop channels**. Conversation history is stored in **PostgreSQL** for context. Responses are sent back through WhatsApp as **text or synthesized voice (TTS)** depending on the original message type.<br><br>### Setup steps<br><br>To deploy the workflow, configure integrations and update key resources used by the automation. First, connect required **credentials** including Google Calendar, Google Sheets, Gmail, WhatsApp Business API, OpenAI/OpenRouter for transcription and TTS, Google Gemini for AI processing and guardrails, and a **Postgres database** for conversation memory.<br><br>Next, prepare **Google Sheets** as a lightweight CRM by creating a sheet with columns such as *Phone Number*, *Client*, and *Service*, then insert its **Document ID** in the relevant nodes. Verify the correct **Google Calendar ID** for appointment management. Configure the **WhatsApp Trigger and messaging nodes** with your Phone Number ID, update the escalation **email recipient**, and review the **AI prompts** to customize salon details, assistant behavior, and guardrail thresholds before activating the workflow. |
| Sticky Note9 | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## STEP 5 - Error |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation | — | — | ## MY NEW YOUTUBE CHANNEL<br>👉 [Subscribe to my new **YouTube channel**](https://youtube.com/@n3witalia). Here I’ll share videos and Shorts with practical tutorials and **FREE templates for n8n**.<br><br>[![image](https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg)](https://youtube.com/@n3witalia) |

---

# 4. Reproducing the Workflow from Scratch

1. **Create the chat entry point**
   - Add a **Chat Trigger** node.
   - Set response mode to **Response Nodes**.
   - Name it **When chat message received**.

2. **Create chat payload normalization**
   - Add a **Set** node named **From Chat**.
   - Create fields:
     - `message` = `{{$json.chatInput}}`
     - `sessionId` = `{{$json.sessionId}}`
   - Connect **When chat message received → From Chat**.

3. **Create the WhatsApp entry point**
   - Add a **WhatsApp Trigger** node named **WhatsApp Trigger**.
   - Configure it to listen to **messages** updates.
   - Connect WhatsApp Business credentials.

4. **Classify WhatsApp messages**
   - Add a **Switch** node named **Input type**.
   - Add output rule **Text** where `{{$json.messages[0].text.body}}` exists.
   - Add output rule **Voice** where `{{$json.messages[0].audio}}` exists.
   - Set fallback output to unsupported content.
   - Connect **WhatsApp Trigger → Input type**.

5. **Normalize WhatsApp text**
   - Add a **Set** node named **Message**.
   - Create:
     - `message` = `{{$('WhatsApp Trigger').item.json.messages[0].text.body}}`
     - `sessionId` = `{{$('WhatsApp Trigger').item.json.messages[0].from}}`
     - `type` = `whatsapp`
   - Connect **Input type (Text) → Message**.

6. **Handle unsupported WhatsApp content**
   - Add a **WhatsApp** send node named **Not supported**.
   - Configure:
     - operation: **send**
     - phone number ID: your WhatsApp business sender ID
     - recipient: `{{$('WhatsApp Trigger').item.json.messages[0].from}}`
     - text: `You can only send text messages, images, audio files and PDF documents.`
   - Connect **Input type (fallback) → Not supported**.

7. **Retrieve audio media URL**
   - Add a **WhatsApp** node named **Get Audio Url**.
   - Set:
     - resource: **media**
     - operation: **mediaUrlGet**
     - media ID: `{{$('WhatsApp Trigger').item.json.messages[0].audio.id}}`
   - Connect **Input type (Voice) → Get Audio Url**.

8. **Download WhatsApp audio**
   - Add an **HTTP Request** node named **Download Audio**.
   - URL = `{{$json.url}}`
   - Use the appropriate header-based auth required by the WhatsApp media URL endpoint.
   - Ensure binary download is enabled as needed by your setup.
   - Connect **Get Audio Url → Download Audio**.

9. **Transcribe audio**
   - Add an **OpenAI / OpenAI-compatible LangChain** node named **Transcribe Audio**.
   - Set:
     - resource: **audio**
     - operation: **transcribe**
   - Attach OpenAI-compatible credentials.
   - Connect **Download Audio → Transcribe Audio**.

10. **Normalize transcribed audio**
    - Add a **Set** node named **Audio**.
    - Create:
      - `message` = `{{$json.text}}`
      - `sessionId` = `{{$('WhatsApp Trigger').item.json.messages[0].from}}`
      - `type` = `whatsapp`
    - Connect **Transcribe Audio → Audio**.

11. **Create the shared normalization node**
    - Add a **Set** node named **Normalize**.
    - Create:
      - `message` = `{{$json.message}}`
      - `sessionId` = `{{$json.sessionId}}`
      - `type` = `{{$json.type}}`
    - Connect:
      - **From Chat → Normalize**
      - **Message → Normalize**
      - **Audio → Normalize**

12. **Add the guardrails model**
    - Add a **Google Gemini Chat Model** node named **Google Gemini Chat Model1**.
    - Select model `models/gemini-2.5-flash-lite`.
    - Connect Gemini credentials.

13. **Add the Guardrails node**
    - Add a **Guardrails** node named **Guardrails**.
    - Set text input to `{{$json.message}}`.
    - Enable jailbreak protection.
    - Set threshold to **0.7**.
    - Enable custom system message if you want equivalent behavior.
    - Connect:
      - **Normalize → Guardrails**
      - **Google Gemini Chat Model1 → Guardrails** as language model

14. **Create blocked-response nodes**
    - Add a **Chat** node named **Chat1** with message:
      - `What was communicated does not reflect our policies or is not relevant.`
    - Add a **WhatsApp** node named **Send message1** with the same text and recipient `{{$('WhatsApp Trigger').item.json.messages[0].from}}`.
    - In the provided JSON, only **Chat1** is connected. If you want correct channel-aware refusal behavior, add a routing layer so blocked WhatsApp messages reach **Send message1** too.

15. **Create the main AI model**
    - Add a **Google Gemini Chat Model** node named **Google Gemini Chat Model**.
    - Select model `models/gemini-3.1-flash-lite-preview`.

16. **Add Postgres memory**
    - Add a **Postgres Chat Memory** node named **Postgres Chat Memory**.
    - Configure:
      - session ID type: **custom key**
      - session key: `{{$('Normalize').item.json.sessionId}}`
    - Provide Postgres credentials and required tables/schema.

17. **Create Google Sheets client lookup**
    - Add a **Google Sheets Tool** node named **New client**.
    - Select your spreadsheet and client sheet.
    - Use it as a row retrieval/search tool.
    - Spreadsheet should contain at minimum:
      - `Phone Number`
      - `Client`
      - `Service`

18. **Create Google Sheets client upsert**
    - Add a **Google Sheets Tool** node named **Add client**.
    - Operation: **appendOrUpdate**
    - Match on column: **Phone Number**
    - Map:
      - `Phone Number` = `{{$('Normalize').item.json.sessionId}}`
      - `Client` = AI-provided input
      - `Service` = AI-provided input
    - Use the same spreadsheet/sheet as above.

19. **Create calendar lookup tool**
    - Add a **Google Calendar Tool** node named **Get events**.
    - Operation: **getAll**
    - Calendar: your booking calendar
    - Return all: enabled
    - Timezone: `Europe/Madrid` or your local timezone
    - `timeMax` = `{{$now.plus({ days: 30 }).toFormat('yyyy-MM-dd')}}`
    - Add tool description explaining that existing results mean busy/unavailable slots.

20. **Create calendar booking tool**
    - Add a **Google Calendar Tool** node named **Create event**.
    - Calendar: your booking calendar
    - Operation: create
    - Allow AI-provided:
      - start
      - end
      - summary
      - description
    - Add tool description telling the model to include client name and service in the title.

21. **Create calendar reschedule tool**
    - Add a **Google Calendar Tool** node named **Update event**.
    - Operation: update
    - Allow AI-provided:
      - event ID
      - start
      - end
    - Add instruction that end time should be 30 minutes after start unless specified otherwise.

22. **Create calendar delete tool**
    - Add a **Google Calendar Tool** node named **Delete event**.
    - Operation: delete
    - Require AI-provided event ID.

23. **Add calculator tool**
    - Add a **Calculator Tool** node named **Calculator**.
    - Default config is sufficient.

24. **Create escalation email tool**
    - Add a **Gmail Tool** node named **Send Email**.
    - Configure:
      - recipient: your staff email
      - subject: escalation subject
      - body: AI-provided message
      - disable attribution if desired

25. **Create chat human-approval escalation**
    - Add a **Chat HITL Tool** node named **Escalation1**.
    - Configure:
      - approval type: **double**
      - max wait time: **45 minutes**
      - message body showing requested tool and parameters
      - `blockUserInput` from AI if you want dynamic control
    - Attach **Send Email** as an available tool to this escalation node.

26. **Optionally create WhatsApp human-approval escalation**
    - Add a **WhatsApp HITL Tool** node named **Escalation**.
    - Configure formatted message and double approval.
    - Recipient phone number may be AI-provided, though in practice this is brittle.
    - Attach **Send Email** as an available tool.
    - Note: in the provided workflow this node is not actually connected to the main agent.

27. **Create the AI agent**
    - Add an **AI Agent** node named **Virtual Assistant**.
    - Set input text to `{{$json.guardrailsInput}}`.
    - Add the full business system prompt defining:
      - Emma’s identity
      - mandatory client identification
      - booking, rescheduling, cancellation rules
      - escalation policy
      - current-date logic
      - salon details
    - Enable retry on fail if desired.
    - Connect:
      - **Guardrails → Virtual Assistant**
      - **Google Gemini Chat Model → Virtual Assistant** as language model
      - **Postgres Chat Memory → Virtual Assistant** as memory
      - connect all business tools as AI tools:
        - New client
        - Add client
        - Get events
        - Create event
        - Update event
        - Delete event
        - Calculator
        - Escalation1
      - optionally connect **Escalation** too if you want WhatsApp HITL escalation

28. **Create output channel router**
    - Add a **Switch** node named **Switch**.
    - Route by `{{$('Normalize').item.json.type}}`
      - `whatsapp`
      - `chat`
    - Connect **Virtual Assistant → Switch**.

29. **Create chat reply**
    - Add a **Chat** node named **Chat**.
    - Message = `{{$json.output}}`
    - Connect **Switch (Chat) → Chat**.

30. **Create WhatsApp modality detector**
    - Add an **If** node named **From audio to audio?**
    - Condition: `{{$('WhatsApp Trigger').item.json.messages[0].audio}}` exists
    - Connect **Switch (WhatsApp) → From audio to audio?**

31. **Create WhatsApp text reply**
    - Add a **WhatsApp** send node named **Send message**.
    - Text = `{{$json.output}}`
    - Recipient = `{{$('WhatsApp Trigger').item.json.messages[0].from}}`
    - Connect **From audio to audio? false → Send message**

32. **Create TTS node**
    - Add an **OpenAI / OpenAI-compatible audio generation** node named **Generate Audio Response**.
    - Input should be the agent output.
    - Voice = `onyx`
    - Resource = `audio`
    - Connect **From audio to audio? true → Generate Audio Response**
    - Important: use a valid expression such as `{{$('Virtual Assistant').item.json.output}}`.  
      The JSON currently references `AI Agent1`, which does not exist.

33. **Fix generated audio MIME type**
    - Add a **Code** node named **Fix mimeType for Audio**.
    - Use JavaScript to replace `audio/mp3` with `audio/mpeg` across binary properties.
    - Connect **Generate Audio Response → Fix mimeType for Audio**.

34. **Send WhatsApp audio reply**
    - Add a **WhatsApp** send node named **Send audio**.
    - Operation: send
    - Message type: audio
    - Media source: use n8n binary media
    - Recipient = `{{$('Input type').item.json.contacts[0].wa_id}}`
    - Connect **Fix mimeType for Audio → Send audio**

35. **Create workflow-level error handling**
    - Add an **Error Trigger** node named **Workflow Error Handler**.
    - Add a **Gmail** node named **Notify: Workflow Error**.
    - Configure recipient and HTML body using:
      - `{{$json.execution.error.message}}`
      - `{{$json.execution.lastNodeExecuted}}`
    - Connect **Workflow Error Handler → Notify: Workflow Error**
    - Update the subject/body text so it references this booking workflow, not another workflow.

36. **Add credentials**
    - Configure all required credentials:
      - WhatsApp Trigger OAuth / WhatsApp Business API
      - WhatsApp sending credential
      - Google Sheets OAuth2
      - Google Calendar OAuth2
      - Gmail OAuth2
      - Google Gemini / PaLM API
      - OpenAI-compatible credential for transcription and TTS
      - Postgres
      - optional header-auth credential for media download

37. **Prepare external resources**
    - Google Sheet:
      - create columns `Phone Number`, `Client`, `Service`
    - Google Calendar:
      - choose the calendar dedicated to appointments
    - Postgres:
      - ensure memory storage is ready for chat history
    - WhatsApp:
      - confirm inbound webhook and outbound messaging are enabled

38. **Test all entry points**
    - Test chat input with simple booking requests.
    - Test WhatsApp text booking.
    - Test WhatsApp audio booking.
    - Test blocked input.
    - Test create, reschedule, delete flows.
    - Test escalation behavior.

39. **Fix the known implementation issues before production**
    - Connect blocked WhatsApp path to **Send message1** or a proper channel-aware refusal router.
    - Replace `{{$('AI Agent1').item.json.output}}` with `{{$('Virtual Assistant').item.json.output}}`.
    - Review whether **Escalation** should be connected to the main agent.
    - Correct error email subject/body text.
    - Replace placeholder recipient emails (`xxx@xxx.com`, `user@example.com`) with real values.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI-Powered WhatsApp Chatbot: This workflow implements an AI-powered WhatsApp booking assistant for a hair salon, supporting text and voice messages, using guardrails, Google Sheets, Google Calendar, Gmail escalation, PostgreSQL memory, and WhatsApp text/TTS responses. | Canvas note |
| Clone this sheet for example of simple database | https://docs.google.com/spreadsheets/d/1DM24yhANppmCu92D3St6GpMh2He7L2feizajN0aLDrU/edit?usp=sharing |
| Subscribe to my new YouTube channel. Here I’ll share videos and Shorts with practical videos and free n8n templates. | https://youtube.com/@n3witalia |
| YouTube channel banner image link included in sticky note | https://n3wstorage.b-cdn.net/n3witalia/youtube-n8n-cover.jpg |

## Important implementation observations
| Note Content | Context or Link |
|---|---|
| The WhatsApp guardrails refusal node `Send message1` is disconnected and will never run unless explicitly connected. | Workflow implementation note |
| `Generate Audio Response` references `AI Agent1`, but the actual agent node is `Virtual Assistant`. This expression should be corrected. | Workflow implementation note |
| The error email still references “LinkedIn PDF → JobAdder Sync”, which is unrelated to this workflow and should be renamed. | Workflow implementation note |
| The `Escalation` WhatsApp HITL tool is not connected back to the main agent, while `Escalation1` is. | Workflow implementation note |