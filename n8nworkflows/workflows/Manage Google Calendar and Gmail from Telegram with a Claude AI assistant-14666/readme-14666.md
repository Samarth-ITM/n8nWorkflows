Manage Google Calendar and Gmail from Telegram with a Claude AI assistant

https://n8nworkflows.xyz/workflows/manage-google-calendar-and-gmail-from-telegram-with-a-claude-ai-assistant-14666


# Manage Google Calendar and Gmail from Telegram with a Claude AI assistant

## 1. Workflow Overview

This workflow creates a Telegram-based personal assistant that accepts text messages, voice notes, and photos, then uses AI to understand the request and either respond directly or act on Google Calendar and Gmail.

Its main use case is a private assistant for a single authorized Telegram user. Once a message is received, the workflow verifies authorization, converts non-text inputs into text context, merges all supported input types into one unified structure, sends that context to an AI agent backed by Claude Haiku via OpenRouter, and lets the agent call Calendar and Gmail tools when needed.

### 1.1 Entry and Access Control
The workflow starts with a Telegram trigger, then checks whether the sender is authorized. Unauthorized users receive a rejection message and do not proceed further.

### 1.2 Input Type Routing
Authorized messages are classified into one of three supported input modes:
- Voice note
- Text message
- Image

### 1.3 Media-to-Text Conversion
Voice notes are downloaded and transcribed with OpenAI. Images are downloaded and analyzed with OpenAI vision. Text messages are passed through directly.

### 1.4 Unified Agent Input Preparation
Outputs from the voice, text, and image branches are merged into a normalized structure so the AI agent can consume them consistently.

### 1.5 AI Agent Orchestration
A LangChain agent receives the normalized input, uses Claude Haiku through OpenRouter as its language model, keeps a 30-message conversational memory per Telegram user, and decides whether to answer normally or invoke Calendar/Gmail tools.

### 1.6 Tool Execution: Google Calendar and Gmail
The agent has access to Google Calendar operations for availability, event creation, reading, updating, and deletion, and Gmail operations for sending, searching, reading, replying, and deleting emails.

### 1.7 Response Delivery
The final agent response is sent back to the Telegram chat.

---

## 2. Block-by-Block Analysis

## 2.1 Entry and Authorization

**Overview:**  
This block receives incoming Telegram messages and restricts workflow usage to an authorized user. If authorization fails, the workflow sends an access-denied message and stops.

**Nodes Involved:**  
- Telegram Trigger
- If
- Send a text message1

### Node Details

#### Telegram Trigger
- **Type and technical role:** `n8n-nodes-base.telegramTrigger`  
  Entry-point webhook node for Telegram bot updates.
- **Configuration choices:**  
  - Listens for `message` updates only.
  - Uses Telegram bot credentials.
- **Key expressions or variables used:**  
  None in parameters, but downstream nodes access fields like:
  - `$json.message.text`
  - `$json.message.voice.file_id`
  - `$json.message.photo`
  - `$json.message.chat.id`
  - `$json.message.from.id`
- **Input and output connections:**  
  - No input
  - Output → `If`
- **Version-specific requirements:**  
  - Type version `1.2`
- **Edge cases or potential failure types:**  
  - Invalid Telegram bot credentials
  - Webhook registration issues
  - Unexpected payloads if Telegram sends message variants not accounted for
- **Sub-workflow reference:**  
  None

#### If
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional authorization gate.
- **Configuration choices:**  
  - Checks whether `$json.authorized` equals `"false"`.
  - Loose type validation is enabled.
- **Key expressions or variables used:**  
  - `={{ $json.authorized }}`
- **Important interpretation note:**  
  This node assumes an `authorized` field already exists in the incoming data, but no earlier node in this workflow creates it. The sticky note says to replace a placeholder with a Telegram numeric user ID, but the actual node configuration shown does not implement that comparison.
- **Input and output connections:**  
  - Input ← `Telegram Trigger`
  - False/true style branching:
    - Unauthorized branch → `Send a text message1`
    - Other branch → `Switch`
- **Version-specific requirements:**  
  - Type version `2.2`
- **Edge cases or potential failure types:**  
  - Authorization logic will not work correctly unless `authorized` is populated somehow
  - If `authorized` is missing, the branch behavior may be unintended
- **Sub-workflow reference:**  
  None

#### Send a text message1
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends an authorization failure message.
- **Configuration choices:**  
  - Sends fixed text: `Authorization Failed - You don't have access to Agent`
  - Chat ID comes from the incoming Telegram message
  - Parse mode set to HTML
  - Attribution disabled
- **Key expressions or variables used:**  
  - `={{ $('Telegram Trigger').item.json.message.chat.id }}`
- **Input and output connections:**  
  - Input ← `If`
  - No downstream node
- **Version-specific requirements:**  
  - Type version `1.2`
- **Edge cases or potential failure types:**  
  - Telegram send failure if chat ID is missing
  - Credential or permission issues with the bot
- **Sub-workflow reference:**  
  None

---

## 2.2 Input Type Routing

**Overview:**  
This block inspects the authorized Telegram message and routes it to one of three processing paths: voice, text, or image.

**Nodes Involved:**  
- Switch

### Node Details

#### Switch
- **Type and technical role:** `n8n-nodes-base.switch`  
  Branch router based on message content.
- **Configuration choices:**  
  - Output 1: `Audio Note` if `message.voice.file_id` exists
  - Output 2: `Text Note` if `message.text` exists
  - Output 3: `Image` if `message.photo` exists
  - Uses strict validation in conditions
  - Output names are explicitly renamed
- **Key expressions or variables used:**  
  - `={{ $json.message.voice.file_id }}`
  - `={{ $json.message.text }}`
  - `={{ $json.message.photo }}`
- **Input and output connections:**  
  - Input ← `If`
  - Output `Audio Note` → `Get Audio`
  - Output `Text Note` → `Edit Fields`
  - Output `Image` → `Telegram2`
- **Version-specific requirements:**  
  - Type version `3.2`
- **Edge cases or potential failure types:**  
  - Messages with unsupported formats, such as documents, stickers, or video, will not match any branch
  - If a message contains multiple supported fields unexpectedly, branch behavior depends on Switch evaluation logic
  - Images with fewer photo sizes may affect later indexing
- **Sub-workflow reference:**  
  None

---

## 2.3 Voice Processing

**Overview:**  
This block downloads a Telegram voice note and transcribes it to text using OpenAI audio transcription.

**Nodes Involved:**  
- Get Audio
- OpenAI

### Node Details

#### Get Audio
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Downloads a Telegram file from a voice message.
- **Configuration choices:**  
  - Resource: `file`
  - File ID taken from the Telegram voice note
- **Key expressions or variables used:**  
  - `={{ $json.message.voice.file_id }}`
- **Input and output connections:**  
  - Input ← `Switch`
  - Output → `OpenAI`
- **Version-specific requirements:**  
  - Type version `1.2`
- **Edge cases or potential failure types:**  
  - Missing or invalid `file_id`
  - Telegram API download errors
  - Binary file retrieval issues
- **Sub-workflow reference:**  
  None

#### OpenAI
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  Transcribes audio to text.
- **Configuration choices:**  
  - Resource: `audio`
  - Operation: `transcribe`
  - Uses OpenAI credentials
- **Key expressions or variables used:**  
  No custom expression shown in the node parameters.
- **Input and output connections:**  
  - Input ← `Get Audio`
  - Output → `Merge`
- **Version-specific requirements:**  
  - Type version `1.8`
- **Edge cases or potential failure types:**  
  - OpenAI credential errors
  - Unsupported file format or corrupted audio
  - Rate limits or transcription timeout
  - Output schema may vary by node version/model behavior
- **Sub-workflow reference:**  
  None

---

## 2.4 Text Message Preparation

**Overview:**  
This block extracts the incoming Telegram text message into a normalized field named `Instructions`.

**Nodes Involved:**  
- Edit Fields

### Node Details

#### Edit Fields
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a normalized text field for downstream processing.
- **Configuration choices:**  
  - Adds a string field `Instructions`
  - Pulls the value from the original Telegram message text
  - `alwaysOutputData` is enabled
- **Key expressions or variables used:**  
  - `={{ $('Telegram Trigger').item.json.message.text }}`
- **Input and output connections:**  
  - Input ← `Switch`
  - Output → `Merge`
- **Version-specific requirements:**  
  - Type version `3.4`
- **Edge cases or potential failure types:**  
  - Empty message text
  - Expression resolution failure if trigger data is unavailable
- **Sub-workflow reference:**  
  None

---

## 2.5 Image Processing

**Overview:**  
This block downloads a Telegram image, analyzes it with OpenAI vision, and formats the resulting description together with the original caption.

**Nodes Involved:**  
- Telegram2
- Analyze image
- fields

### Node Details

#### Telegram2
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Downloads an image file from Telegram.
- **Configuration choices:**  
  - Resource: `file`
  - Uses `message.photo[2].file_id`
- **Key expressions or variables used:**  
  - `={{ $json.message.photo[2].file_id }}`
- **Input and output connections:**  
  - Input ← `Switch`
  - Output → `Analyze image`
- **Version-specific requirements:**  
  - Type version `1.2`
- **Edge cases or potential failure types:**  
  - `message.photo[2]` may not exist if Telegram only provides fewer image size variants
  - Invalid file ID
  - Telegram API download issues
- **Sub-workflow reference:**  
  None

#### Analyze image
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  Performs image understanding with an OpenAI multimodal model.
- **Configuration choices:**  
  - Resource: `image`
  - Operation: `analyze`
  - Input type: `base64`
  - Prompt text: `Explain the image`
  - Model: `chatgpt-4o-latest`
- **Key expressions or variables used:**  
  No additional custom expression in parameters beyond model selection.
- **Input and output connections:**  
  - Input ← `Telegram2`
  - Output → `fields`
- **Version-specific requirements:**  
  - Type version `2`
- **Edge cases or potential failure types:**  
  - OpenAI credential issues
  - Image conversion/base64 handling problems
  - Large image payload limits
  - Vision model response shape may vary
- **Sub-workflow reference:**  
  None

#### fields
- **Type and technical role:** `n8n-nodes-base.set`  
  Extracts the image analysis text and stores the original Telegram caption.
- **Configuration choices:**  
  - Creates `Analysis`
  - Creates `caption`
- **Key expressions or variables used:**  
  - `={{ $json['0'].content[0].text }}`
  - `={{ $('Telegram Trigger').item.json.message.caption }}`
- **Input and output connections:**  
  - Input ← `Analyze image`
  - Output → `Merge`
- **Version-specific requirements:**  
  - Type version `3.4`
- **Edge cases or potential failure types:**  
  - The expression `{{$json['0'].content[0].text}}` assumes a very specific response shape from the image analysis node
  - If the model output format changes, this node will break
  - Caption may be undefined
- **Sub-workflow reference:**  
  None

---

## 2.6 Unified Input Preparation

**Overview:**  
This block combines the three alternative input branches and reshapes them into a common payload for the AI agent.

**Nodes Involved:**  
- Merge
- Input

### Node Details

#### Merge
- **Type and technical role:** `n8n-nodes-base.merge`  
  Collects outputs from voice, text, and image branches.
- **Configuration choices:**  
  - Configured for `numberInputs: 3`
- **Key expressions or variables used:**  
  None directly.
- **Input and output connections:**  
  - Input 0 ← `OpenAI`
  - Input 1 ← `Edit Fields`
  - Input 2 ← `fields`
  - Output → `Input`
- **Version-specific requirements:**  
  - Type version `3.2`
- **Edge cases or potential failure types:**  
  - Merge behavior depends on node execution mode and available incoming items
  - Since only one branch is expected per message, this setup may produce waiting or empty-output issues depending on actual merge semantics in the n8n version used
- **Sub-workflow reference:**  
  None

#### Input
- **Type and technical role:** `n8n-nodes-base.set`  
  Normalizes all possible branch outputs into agent-ready fields.
- **Configuration choices:**  
  - `text_message` = `Instructions || text`
  - `image_description` = `Analysis`
  - `caption` = `caption`
- **Key expressions or variables used:**  
  - `={{ $json.Instructions || $json.text}}`
  - `={{ $json.Analysis }}`
  - `={{ $json.caption }}`
- **Input and output connections:**  
  - Input ← `Merge`
  - Output → `MainAgent`
- **Version-specific requirements:**  
  - Type version `3.4`
- **Edge cases or potential failure types:**  
  - If transcription output uses a field name other than `text`, voice messages may not populate `text_message`
  - Empty fields are possible for some branches
- **Sub-workflow reference:**  
  None

---

## 2.7 AI Agent Core

**Overview:**  
This block is the reasoning center of the workflow. It receives the unified input, uses Claude Haiku through OpenRouter, stores chat memory by Telegram user, and can invoke Calendar or Gmail tools.

**Nodes Involved:**  
- MainAgent
- OpenRouter Chat Model1
- Simple Memory

### Node Details

#### MainAgent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain-style agent node that combines prompt instructions, memory, language model, and callable tools.
- **Configuration choices:**  
  - Prompt type: defined manually
  - Agent text is assembled from:
    - `text_message`
    - `image_description`
    - `caption`
  - System message defines:
    - Persona as a personal assistant for Casonova Brook
    - English-only responses
    - Short, plain-text answers
    - Behavior rules for greeting, general queries, calendar tasks, and email tasks
    - Instruction to call tools instead of exposing JSON or internal tool traces
- **Key expressions or variables used:**  
  Main input text:
  - Builds a combined prompt using:
    - `$json.text_message`
    - `$json.image_description`
    - `$json.caption`
  System prompt includes:
  - `{{ $now }}`
- **Input and output connections:**  
  - Main input ← `Input`
  - Language model input ← `OpenRouter Chat Model1`
  - Memory input ← `Simple Memory`
  - Tool inputs ← all Calendar and Gmail tool nodes via `ai_tool`
  - Main output → `Send a text message`
- **Version-specific requirements:**  
  - Type version `3`
- **Edge cases or potential failure types:**  
  - Poor tool selection if the prompt is ambiguous
  - Tool-call failures bubbling into incomplete responses
  - Prompt conflicts: system prompt says plain text, but outgoing Telegram node uses Markdown parse mode
  - Hallucinated assumptions if users omit critical details
- **Sub-workflow reference:**  
  None

#### OpenRouter Chat Model1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  Provides the chat model used by the agent.
- **Configuration choices:**  
  - Model: `anthropic/claude-haiku-4.5`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Output via `ai_languageModel` → `MainAgent`
- **Version-specific requirements:**  
  - Type version `1`
- **Edge cases or potential failure types:**  
  - OpenRouter auth failures
  - Model availability changes
  - Rate limits or network errors
- **Sub-workflow reference:**  
  None

#### Simple Memory
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  Stores recent conversation history per user.
- **Configuration choices:**  
  - Session key is the Telegram sender ID
  - Session ID type: custom key
  - Context window length: 30 messages
- **Key expressions or variables used:**  
  - `={{ $('Telegram Trigger').item.json.message.from.id }}`
- **Input and output connections:**  
  - Output via `ai_memory` → `MainAgent`
- **Version-specific requirements:**  
  - Type version `1.3`
- **Edge cases or potential failure types:**  
  - Missing Telegram sender ID
  - Memory continuity issues if payload shape changes
- **Sub-workflow reference:**  
  None

---

## 2.8 Google Calendar Tools

**Overview:**  
These nodes are exposed as tools to the AI agent so it can check availability and manage events in Google Calendar.

**Nodes Involved:**  
- Availability
- UpdateEvent
- createEvent
- GetManyEvents
- GetEvent
- DeleteEvent

### Node Details

#### Availability
- **Type and technical role:** `n8n-nodes-base.googleCalendarTool`  
  Calendar availability lookup tool.
- **Configuration choices:**  
  - Resource: `calendar`
  - Accepts AI-provided start and end times
  - Uses a specific calendar ID
- **Key expressions or variables used:**  
  - `={{ $fromAI('Start_Time', '', 'string') }}`
  - `={{ $fromAI('End_Time', '', 'string') }}`
- **Input and output connections:**  
  - Tool output → `MainAgent`
- **Version-specific requirements:**  
  - Type version `1.3`
- **Edge cases or potential failure types:**  
  - Invalid date/time formatting
  - OAuth token expiration
  - Wrong calendar ID
- **Sub-workflow reference:**  
  None

#### UpdateEvent
- **Type and technical role:** `n8n-nodes-base.googleCalendarTool`  
  Updates an existing calendar event.
- **Configuration choices:**  
  - Operation: `update`
  - Requires event ID
  - Only summary/title is explicitly updated in this configuration
- **Key expressions or variables used:**  
  - `={{ $fromAI('Event_ID', '', 'string') }}`
  - `={{ $fromAI('Summary', '', 'string') }}`
- **Input and output connections:**  
  - Tool output → `MainAgent`
- **Version-specific requirements:**  
  - Type version `1.3`
- **Edge cases or potential failure types:**  
  - Missing/incorrect event ID
  - Limited update coverage because start/end/time changes are not configured
- **Sub-workflow reference:**  
  None

#### createEvent
- **Type and technical role:** `n8n-nodes-base.googleCalendarTool`  
  Creates a Google Calendar event.
- **Configuration choices:**  
  - Requires AI-provided `start` and `end`
  - Uses summary in additional fields
  - Adds one attendee field
  - `sendUpdates` is set to `all`
- **Key expressions or variables used:**  
  - `={{ $fromAI('Start', '', 'string') }}`
  - `={{ $fromAI('End', '', 'string') }}`
  - `={{ $fromAI('Summary', '', 'string') }}`
  - `={{ $fromAI('attendees0_Attendees', 'email of the attendes', 'string') }}`
- **Input and output connections:**  
  - Tool output → `MainAgent`
- **Version-specific requirements:**  
  - Type version `1.3`
- **Edge cases or potential failure types:**  
  - Invalid time ranges
  - Missing timezone assumptions
  - Invalid attendee email
  - Event creation failures due to calendar permissions
- **Sub-workflow reference:**  
  None

#### GetManyEvents
- **Type and technical role:** `n8n-nodes-base.googleCalendarTool`  
  Retrieves multiple events within a time range.
- **Configuration choices:**  
  - Operation: `getAll`
  - Time range supplied by AI
  - `returnAll` controlled by AI
- **Key expressions or variables used:**  
  - `={{ $fromAI('After', '', 'string') }}`
  - `={{ $fromAI('Before', '', 'string') }}`
  - `={{ $fromAI('Return_All', '', 'boolean') }}`
- **Input and output connections:**  
  - Tool output → `MainAgent`
- **Version-specific requirements:**  
  - Type version `1.3`
- **Edge cases or potential failure types:**  
  - Large result sets
  - Invalid date filters
  - OAuth/calendar access issues
- **Sub-workflow reference:**  
  None

#### GetEvent
- **Type and technical role:** `n8n-nodes-base.googleCalendarTool`  
  Fetches a single event by ID.
- **Configuration choices:**  
  - Operation: `get`
  - Requires event ID from the agent
- **Key expressions or variables used:**  
  - `={{ $fromAI('Event_ID', '', 'string') }}`
- **Input and output connections:**  
  - Tool output → `MainAgent`
- **Version-specific requirements:**  
  - Type version `1.3`
- **Edge cases or potential failure types:**  
  - Missing or invalid event ID
- **Sub-workflow reference:**  
  None

#### DeleteEvent
- **Type and technical role:** `n8n-nodes-base.googleCalendarTool`  
  Deletes an event from Google Calendar.
- **Configuration choices:**  
  - Operation: `delete`
  - Sends updates to all attendees
- **Key expressions or variables used:**  
  - `={{ $fromAI('Event_ID', '', 'string') }}`
- **Input and output connections:**  
  - Tool output → `MainAgent`
- **Version-specific requirements:**  
  - Type version `1.3`
- **Edge cases or potential failure types:**  
  - Missing event ID
  - Delete permissions
  - Attendee notification side effects
- **Sub-workflow reference:**  
  None

---

## 2.9 Gmail Tools

**Overview:**  
These nodes give the agent email capabilities for sending, searching, reading, replying to, and deleting Gmail messages.

**Nodes Involved:**  
- sendMessage
- getEmails
- getEmail
- replyEmail
- deleteEmail

### Node Details

#### sendMessage
- **Type and technical role:** `n8n-nodes-base.gmailTool`  
  Sends a plain-text email.
- **Configuration choices:**  
  - AI provides recipient, subject, and body
  - Optional `replyTo`
  - Attribution disabled
- **Key expressions or variables used:**  
  - `={{ $fromAI('To', '', 'string') }}`
  - `={{ $fromAI('Subject', '', 'string') }}`
  - `={{ $fromAI('Message', '', 'string') }}`
  - `={{ $fromAI('Send_Replies_To', 'If we want to send reply', 'string') }}`
- **Input and output connections:**  
  - Tool output → `MainAgent`
- **Version-specific requirements:**  
  - Type version `2.1`
- **Edge cases or potential failure types:**  
  - Invalid recipient format
  - Gmail OAuth issues
  - Sending restrictions or quota problems
- **Sub-workflow reference:**  
  None

#### getEmails
- **Type and technical role:** `n8n-nodes-base.gmailTool`  
  Searches Gmail messages using filters.
- **Configuration choices:**  
  - Operation: `getAll`
  - Supports search query, sender, received-after date, and spam/trash inclusion
- **Key expressions or variables used:**  
  - `={{ $fromAI('Search', 'search for the emails', 'string') }}`
  - `={{ $fromAI('Sender', 'If there is any specific sender', 'string') }}`
  - `={{ $fromAI('Received_After', 'Email received after specific email', 'string') }}`
  - `={{ $fromAI('Include_Spam_and_Trash', '', 'boolean') }}`
- **Input and output connections:**  
  - Tool output → `MainAgent`
- **Version-specific requirements:**  
  - Type version `2.1`
- **Edge cases or potential failure types:**  
  - Broad searches returning large datasets
  - Invalid query syntax
  - Gmail permission issues
- **Sub-workflow reference:**  
  None

#### getEmail
- **Type and technical role:** `n8n-nodes-base.gmailTool`  
  Reads a specific email by message ID.
- **Configuration choices:**  
  - Operation: `get`
  - Optional simplified output controlled by AI
- **Key expressions or variables used:**  
  - `={{ $fromAI('Message_ID', 'Specific email id', 'string') }}`
  - `={{ $fromAI('Simplify', '', 'boolean') }}`
- **Input and output connections:**  
  - Tool output → `MainAgent`
- **Version-specific requirements:**  
  - Type version `2.1`
- **Edge cases or potential failure types:**  
  - Missing message ID
  - Message no longer available
- **Sub-workflow reference:**  
  None

#### replyEmail
- **Type and technical role:** `n8n-nodes-base.gmailTool`  
  Replies to an existing email thread.
- **Configuration choices:**  
  - Operation: `reply`
  - Plain text email body
  - Optional reply-to-sender-only mode
  - Attribution disabled
- **Key expressions or variables used:**  
  - `={{ $fromAI('Message_ID', '', 'string') }}`
  - `={{ $fromAI('Message', '', 'string') }}`
  - `={{ $fromAI('Reply_to_Sender_Only', '', 'boolean') }}`
- **Input and output connections:**  
  - Tool output → `MainAgent`
- **Version-specific requirements:**  
  - Type version `2.1`
- **Edge cases or potential failure types:**  
  - Invalid message ID
  - Thread reply failures
  - Gmail permissions
- **Sub-workflow reference:**  
  None

#### deleteEmail
- **Type and technical role:** `n8n-nodes-base.gmailTool`  
  Deletes a Gmail message.
- **Configuration choices:**  
  - Operation: `delete`
  - Requires message ID
- **Key expressions or variables used:**  
  - `={{ $fromAI('Message_ID', '', 'string') }}`
- **Input and output connections:**  
  - Tool output → `MainAgent`
- **Version-specific requirements:**  
  - Type version `2.1`
- **Edge cases or potential failure types:**  
  - Message ID not found
  - Delete permission issues
- **Sub-workflow reference:**  
  None

---

## 2.10 Response Delivery

**Overview:**  
This block sends the AI agent’s final response back to the Telegram user.

**Nodes Involved:**  
- Send a text message

### Node Details

#### Send a text message
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends the assistant’s answer back to the originating Telegram chat.
- **Configuration choices:**  
  - Text = agent output
  - Chat ID from original Telegram message
  - Parse mode set to Markdown
  - Attribution disabled
- **Key expressions or variables used:**  
  - `={{ $json.output }}`
  - `={{ $('Telegram Trigger').item.json.message.chat.id }}`
- **Input and output connections:**  
  - Input ← `MainAgent`
  - No downstream node
- **Version-specific requirements:**  
  - Type version `1.2`
- **Edge cases or potential failure types:**  
  - Markdown parse errors if the agent outputs characters Telegram interprets as malformed Markdown
  - Missing output text
  - Telegram credential/chat delivery failures
- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | n8n-nodes-base.telegramTrigger | Receives incoming Telegram messages |  | If | ## 🤖 Telegram AI Personal Assistant<br>A personal AI assistant that lives in your Telegram. Send it a text, voice note, or photo — it understands all three. It can manage your Google Calendar and Gmail on your behalf, powered by Claude Haiku via OpenRouter.<br><br>### How it works<br>1. A message arrives on Telegram and is checked against an authorized User ID.<br>2. A Switch node routes the input: voice notes are transcribed via OpenAI Whisper, images are analyzed via GPT-4o, and text passes through directly.<br>3. All three branches merge into a single context object fed to the AI agent.<br>4. The agent (Claude Haiku + 30-message memory buffer) decides whether to reply conversationally or call a tool.<br>5. The response is sent back to the user on Telegram.<br><br>### Setup<br>1. **Telegram API** — connect your bot token to the Telegram Trigger and all send nodes.<br>2. **OpenAI API** — used for Whisper transcription and GPT-4o image analysis.<br>3. **OpenRouter API** — used to run Claude Haiku as the agent's language model.<br>4. **Google Calendar OAuth2** — connect and set your calendar ID in all calendar tool nodes.<br>5. **Gmail OAuth2** — connect your Gmail account to all Gmail tool nodes.<br>6. In the `If` node, replace the placeholder with your own Telegram numeric User ID to restrict access. |
| If | n8n-nodes-base.if | Authorization gate | Telegram Trigger | Send a text message1; Switch | ## 🤖 Telegram AI Personal Assistant<br>A personal AI assistant that lives in your Telegram. Send it a text, voice note, or photo — it understands all three. It can manage your Google Calendar and Gmail on your behalf, powered by Claude Haiku via OpenRouter.<br><br>### How it works<br>1. A message arrives on Telegram and is checked against an authorized User ID.<br>2. A Switch node routes the input: voice notes are transcribed via OpenAI Whisper, images are analyzed via GPT-4o, and text passes through directly.<br>3. All three branches merge into a single context object fed to the AI agent.<br>4. The agent (Claude Haiku + 30-message memory buffer) decides whether to reply conversationally or call a tool.<br>5. The response is sent back to the user on Telegram.<br><br>### Setup<br>1. **Telegram API** — connect your bot token to the Telegram Trigger and all send nodes.<br>2. **OpenAI API** — used for Whisper transcription and GPT-4o image analysis.<br>3. **OpenRouter API** — used to run Claude Haiku as the agent's language model.<br>4. **Google Calendar OAuth2** — connect and set your calendar ID in all calendar tool nodes.<br>5. **Gmail OAuth2** — connect your Gmail account to all Gmail tool nodes.<br>6. In the `If` node, replace the placeholder with your own Telegram numeric User ID to restrict access. |
| Send a text message1 | n8n-nodes-base.telegram | Sends unauthorized access message | If |  | ## 🤖 Telegram AI Personal Assistant<br>A personal AI assistant that lives in your Telegram. Send it a text, voice note, or photo — it understands all three. It can manage your Google Calendar and Gmail on your behalf, powered by Claude Haiku via OpenRouter.<br><br>### How it works<br>1. A message arrives on Telegram and is checked against an authorized User ID.<br>2. A Switch node routes the input: voice notes are transcribed via OpenAI Whisper, images are analyzed via GPT-4o, and text passes through directly.<br>3. All three branches merge into a single context object fed to the AI agent.<br>4. The agent (Claude Haiku + 30-message memory buffer) decides whether to reply conversationally or call a tool.<br>5. The response is sent back to the user on Telegram.<br><br>### Setup<br>1. **Telegram API** — connect your bot token to the Telegram Trigger and all send nodes.<br>2. **OpenAI API** — used for Whisper transcription and GPT-4o image analysis.<br>3. **OpenRouter API** — used to run Claude Haiku as the agent's language model.<br>4. **Google Calendar OAuth2** — connect and set your calendar ID in all calendar tool nodes.<br>5. **Gmail OAuth2** — connect your Gmail account to all Gmail tool nodes.<br>6. In the `If` node, replace the placeholder with your own Telegram numeric User ID to restrict access. |
| Switch | n8n-nodes-base.switch | Routes message by type: voice, text, or image | If | Get Audio; Edit Fields; Telegram2 | ## 🤖 Telegram AI Personal Assistant<br>A personal AI assistant that lives in your Telegram. Send it a text, voice note, or photo — it understands all three. It can manage your Google Calendar and Gmail on your behalf, powered by Claude Haiku via OpenRouter.<br><br>### How it works<br>1. A message arrives on Telegram and is checked against an authorized User ID.<br>2. A Switch node routes the input: voice notes are transcribed via OpenAI Whisper, images are analyzed via GPT-4o, and text passes through directly.<br>3. All three branches merge into a single context object fed to the AI agent.<br>4. The agent (Claude Haiku + 30-message memory buffer) decides whether to reply conversationally or call a tool.<br>5. The response is sent back to the user on Telegram.<br><br>### Setup<br>1. **Telegram API** — connect your bot token to the Telegram Trigger and all send nodes.<br>2. **OpenAI API** — used for Whisper transcription and GPT-4o image analysis.<br>3. **OpenRouter API** — used to run Claude Haiku as the agent's language model.<br>4. **Google Calendar OAuth2** — connect and set your calendar ID in all calendar tool nodes.<br>5. **Gmail OAuth2** — connect your Gmail account to all Gmail tool nodes.<br>6. In the `If` node, replace the placeholder with your own Telegram numeric User ID to restrict access. |
| Get Audio | n8n-nodes-base.telegram | Downloads Telegram voice note file | Switch | OpenAI | ##  Data Conversion<br>This section , convert audio or image to text for llm to process. |
| OpenAI | @n8n/n8n-nodes-langchain.openAi | Transcribes audio to text | Get Audio | Merge | ##  Data Conversion<br>This section , convert audio or image to text for llm to process. |
| Edit Fields | n8n-nodes-base.set | Normalizes text message into Instructions field | Switch | Merge | ##  Data Conversion<br>This section , convert audio or image to text for llm to process. |
| Telegram2 | n8n-nodes-base.telegram | Downloads Telegram image file | Switch | Analyze image | ##  Data Conversion<br>This section , convert audio or image to text for llm to process. |
| Analyze image | @n8n/n8n-nodes-langchain.openAi | Analyzes image content with vision model | Telegram2 | fields | ##  Data Conversion<br>This section , convert audio or image to text for llm to process. |
| fields | n8n-nodes-base.set | Extracts image description and caption | Analyze image | Merge | ##  Data Conversion<br>This section , convert audio or image to text for llm to process. |
| Merge | n8n-nodes-base.merge | Combines branch outputs into one stream | OpenAI; Edit Fields; fields | Input |  |
| Input | n8n-nodes-base.set | Maps merged data into agent-ready fields | Merge | MainAgent |  |
| MainAgent | @n8n/n8n-nodes-langchain.agent | Conversational agent with tool use | Input; OpenRouter Chat Model1; Simple Memory; Availability; UpdateEvent; createEvent; GetManyEvents; GetEvent; DeleteEvent; sendMessage; getEmails; getEmail; replyEmail; deleteEmail | Send a text message | ## 🧠 AI Agent<br><br>Claude Haiku processes the unified input with a 30-message memory buffer.<br>Replies conversationally or calls a tool depending on the request. |
| OpenRouter Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Claude Haiku language model provider |  | MainAgent |  |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores per-user chat history |  | MainAgent |  |
| Send a text message | n8n-nodes-base.telegram | Sends final response back to Telegram | MainAgent |  |  |
| Availability | n8n-nodes-base.googleCalendarTool | Checks calendar availability |  | MainAgent | ## 🔧 Agent Tools<br><br>Google Calendar — create, read, update, delete events, check availability.<br>Gmail — send, search, read, reply, and delete emails. |
| UpdateEvent | n8n-nodes-base.googleCalendarTool | Updates an existing calendar event |  | MainAgent | ## 🔧 Agent Tools<br><br>Google Calendar — create, read, update, delete events, check availability.<br>Gmail — send, search, read, reply, and delete emails. |
| createEvent | n8n-nodes-base.googleCalendarTool | Creates calendar events |  | MainAgent | ## 🔧 Agent Tools<br><br>Google Calendar — create, read, update, delete events, check availability.<br>Gmail — send, search, read, reply, and delete emails. |
| GetManyEvents | n8n-nodes-base.googleCalendarTool | Lists calendar events in a time range |  | MainAgent | ## 🔧 Agent Tools<br><br>Google Calendar — create, read, update, delete events, check availability.<br>Gmail — send, search, read, reply, and delete emails. |
| GetEvent | n8n-nodes-base.googleCalendarTool | Reads one event by ID |  | MainAgent | ## 🔧 Agent Tools<br><br>Google Calendar — create, read, update, delete events, check availability.<br>Gmail — send, search, read, reply, and delete emails. |
| DeleteEvent | n8n-nodes-base.googleCalendarTool | Deletes a calendar event |  | MainAgent | ## 🔧 Agent Tools<br><br>Google Calendar — create, read, update, delete events, check availability.<br>Gmail — send, search, read, reply, and delete emails. |
| sendMessage | n8n-nodes-base.gmailTool | Sends an email |  | MainAgent | ## 🔧 Agent Tools<br><br>Google Calendar — create, read, update, delete events, check availability.<br>Gmail — send, search, read, reply, and delete emails. |
| getEmails | n8n-nodes-base.gmailTool | Searches Gmail messages |  | MainAgent | ## 🔧 Agent Tools<br><br>Google Calendar — create, read, update, delete events, check availability.<br>Gmail — send, search, read, reply, and delete emails. |
| getEmail | n8n-nodes-base.gmailTool | Reads one Gmail message |  | MainAgent | ## 🔧 Agent Tools<br><br>Google Calendar — create, read, update, delete events, check availability.<br>Gmail — send, search, read, reply, and delete emails. |
| replyEmail | n8n-nodes-base.gmailTool | Replies to an email |  | MainAgent | ## 🔧 Agent Tools<br><br>Google Calendar — create, read, update, delete events, check availability.<br>Gmail — send, search, read, reply, and delete emails. |
| deleteEmail | n8n-nodes-base.gmailTool | Deletes an email |  | MainAgent | ## 🔧 Agent Tools<br><br>Google Calendar — create, read, update, delete events, check availability.<br>Gmail — send, search, read, reply, and delete emails. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note for workflow purpose and setup |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for media conversion block |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note for AI agent block |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation note for agent tool block |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Telegram Trigger node**
   - Node type: `Telegram Trigger`
   - Configure it to listen for `message` updates.
   - Attach your Telegram bot credential.
   - This is the main entry point.

3. **Add an authorization gate**
   - Node type: `If`
   - Connect `Telegram Trigger` → `If`
   - Recommended configuration:
     - Compare the Telegram sender ID to your authorized numeric user ID.
     - Example logic: `{{$json.message.from.id}}` equals `YOUR_TELEGRAM_USER_ID`
   - Important: the exported workflow currently checks `$json.authorized == "false"`, but no earlier node sets `authorized`. To rebuild this correctly, implement the sender ID comparison directly in the `If` node.

4. **Add an unauthorized response node**
   - Node type: `Telegram`
   - Name it `Send a text message1`
   - Connect the unauthorized branch of `If` to this node.
   - Set:
     - Resource/action to send a text message
     - Text: `Authorization Failed - You don't have access to Agent`
     - Chat ID: `{{$('Telegram Trigger').item.json.message.chat.id}}`
     - Parse mode: `HTML`
     - Disable attribution if desired

5. **Add the message router**
   - Node type: `Switch`
   - Connect the authorized branch of `If` → `Switch`
   - Create 3 outputs:
     1. `Audio Note`: condition checks `{{$json.message.voice.file_id}}` exists
     2. `Text Note`: condition checks `{{$json.message.text}}` exists
     3. `Image`: condition checks `{{$json.message.photo}}` exists

6. **Build the text branch**
   - Add a `Set` node named `Edit Fields`
   - Connect `Switch` text output → `Edit Fields`
   - Add field:
     - `Instructions` = `{{$('Telegram Trigger').item.json.message.text}}`
   - Enable `alwaysOutputData`

7. **Build the voice branch: file download**
   - Add a `Telegram` node named `Get Audio`
   - Connect `Switch` audio output → `Get Audio`
   - Set resource to `file`
   - File ID: `{{$json.message.voice.file_id}}`
   - Use the same Telegram credential

8. **Build the voice branch: transcription**
   - Add an `OpenAI` LangChain node named `OpenAI`
   - Connect `Get Audio` → `OpenAI`
   - Set:
     - Resource: `audio`
     - Operation: `transcribe`
   - Attach your OpenAI credential
   - Make sure the node receives the binary audio from the Telegram file node

9. **Build the image branch: file download**
   - Add a `Telegram` node named `Telegram2`
   - Connect `Switch` image output → `Telegram2`
   - Set resource to `file`
   - File ID: `{{$json.message.photo[2].file_id}}`
   - Important improvement: in a safer rebuild, use the last available photo size instead of fixed index `[2]`, because some Telegram messages may have fewer than 3 variants.

10. **Build the image branch: vision analysis**
    - Add an `OpenAI` LangChain node named `Analyze image`
    - Connect `Telegram2` → `Analyze image`
    - Set:
      - Resource: `image`
      - Operation: `analyze`
      - Input type: `base64`
      - Prompt text: `Explain the image`
      - Model: `chatgpt-4o-latest`
    - Attach OpenAI credentials

11. **Format image analysis output**
    - Add a `Set` node named `fields`
    - Connect `Analyze image` → `fields`
    - Add:
      - `Analysis` = `{{$json['0'].content[0].text}}`
      - `caption` = `{{$('Telegram Trigger').item.json.message.caption}}`
    - Note: this expression depends on the exact structure returned by the image analysis node. Validate it with a sample execution.

12. **Add a Merge node**
    - Node type: `Merge`
    - Name it `Merge`
    - Connect:
      - `OpenAI` → input 1
      - `Edit Fields` → input 2
      - `fields` → input 3
    - Set `numberInputs` to `3`

13. **Normalize agent input**
    - Add a `Set` node named `Input`
    - Connect `Merge` → `Input`
    - Add fields:
      - `text_message` = `{{$json.Instructions || $json.text}}`
      - `image_description` = `{{$json.Analysis}}`
      - `caption` = `{{$json.caption}}`

14. **Add the agent memory**
    - Add a `Simple Memory` node
    - Node type: `Memory Buffer Window`
    - Set:
      - Session ID type: `customKey`
      - Session key: `{{$('Telegram Trigger').item.json.message.from.id}}`
      - Context window length: `30`

15. **Add the language model**
    - Add an `OpenRouter Chat Model` node
    - Name it `OpenRouter Chat Model1`
    - Set model to `anthropic/claude-haiku-4.5`
    - Attach OpenRouter credentials

16. **Add the AI agent**
    - Add an `AI Agent` node
    - Name it `MainAgent`
    - Connect `Input` → `MainAgent`
    - Connect `OpenRouter Chat Model1` to the agent’s language model port
    - Connect `Simple Memory` to the agent’s memory port
    - Set the main text input to a combined expression using normalized fields:
      - Include `text_message` if present
      - Include `image_description` if present
      - Include `caption` if present
   - Configure the system message with the assistant persona and tool behavior rules:
     - Friendly personal assistant for Casonova Brook
     - English only
     - Short plain-text replies
     - Greeting behavior
     - Calendar and email tool usage instructions
     - Use current date/time via `{{$now}}`
     - Do not expose JSON or tool traces

17. **Add Google Calendar tool: Availability**
    - Node type: `Google Calendar Tool`
    - Name: `Availability`
    - Configure:
      - Resource: `calendar`
      - Calendar ID: your target calendar
      - Start time: `{{$fromAI('Start_Time', '', 'string')}}`
      - End time: `{{$fromAI('End_Time', '', 'string')}}`
    - Connect tool output to `MainAgent` tool port
    - Attach Google Calendar OAuth2 credentials

18. **Add Google Calendar tool: UpdateEvent**
    - Name: `UpdateEvent`
    - Operation: `update`
    - Calendar ID: your target calendar
    - Event ID: `{{$fromAI('Event_ID', '', 'string')}}`
    - Update field:
      - Summary: `{{$fromAI('Summary', '', 'string')}}`
    - Connect to `MainAgent`

19. **Add Google Calendar tool: createEvent**
    - Name: `createEvent`
    - Operation: create event
    - Calendar ID: your target calendar
    - Start: `{{$fromAI('Start', '', 'string')}}`
    - End: `{{$fromAI('End', '', 'string')}}`
    - Additional fields:
      - Summary: `{{$fromAI('Summary', '', 'string')}}`
      - Attendee email: `{{$fromAI('attendees0_Attendees', 'email of the attendes', 'string')}}`
      - Send updates: `all`
    - Connect to `MainAgent`

20. **Add Google Calendar tool: GetManyEvents**
    - Name: `GetManyEvents`
    - Operation: `getAll`
    - Calendar ID: your target calendar
    - After: `{{$fromAI('After', '', 'string')}}`
    - Before: `{{$fromAI('Before', '', 'string')}}`
    - Return all: `{{$fromAI('Return_All', '', 'boolean')}}`
    - Connect to `MainAgent`

21. **Add Google Calendar tool: GetEvent**
    - Name: `GetEvent`
    - Operation: `get`
    - Calendar ID: your target calendar
    - Event ID: `{{$fromAI('Event_ID', '', 'string')}}`
    - Connect to `MainAgent`

22. **Add Google Calendar tool: DeleteEvent**
    - Name: `DeleteEvent`
    - Operation: `delete`
    - Calendar ID: your target calendar
    - Event ID: `{{$fromAI('Event_ID', '', 'string')}}`
    - Options: send updates to `all`
    - Connect to `MainAgent`

23. **Add Gmail tool: sendMessage**
    - Node type: `Gmail Tool`
    - Name: `sendMessage`
    - Operation: send email
    - Set:
      - To: `{{$fromAI('To', '', 'string')}}`
      - Subject: `{{$fromAI('Subject', '', 'string')}}`
      - Message: `{{$fromAI('Message', '', 'string')}}`
      - Email type: `text`
      - Reply-To: `{{$fromAI('Send_Replies_To', 'If we want to send reply', 'string')}}`
      - Disable attribution
    - Connect to `MainAgent`
    - Attach Gmail OAuth2 credentials

24. **Add Gmail tool: getEmails**
    - Name: `getEmails`
    - Operation: `getAll`
    - Filters:
      - Query: `{{$fromAI('Search', 'search for the emails', 'string')}}`
      - Sender: `{{$fromAI('Sender', 'If there is any specific sender', 'string')}}`
      - Received After: `{{$fromAI('Received_After', 'Email received after specific email', 'string')}}`
      - Include Spam/Trash: `{{$fromAI('Include_Spam_and_Trash', '', 'boolean')}}`
    - Connect to `MainAgent`

25. **Add Gmail tool: getEmail**
    - Name: `getEmail`
    - Operation: `get`
    - Message ID: `{{$fromAI('Message_ID', 'Specific email id', 'string')}}`
    - Simplify: `{{$fromAI('Simplify', '', 'boolean')}}`
    - Connect to `MainAgent`

26. **Add Gmail tool: replyEmail**
    - Name: `replyEmail`
    - Operation: `reply`
    - Message ID: `{{$fromAI('Message_ID', '', 'string')}}`
    - Message: `{{$fromAI('Message', '', 'string')}}`
    - Email type: `text`
    - Reply to sender only: `{{$fromAI('Reply_to_Sender_Only', '', 'boolean')}}`
    - Disable attribution
    - Connect to `MainAgent`

27. **Add Gmail tool: deleteEmail**
    - Name: `deleteEmail`
    - Operation: `delete`
    - Message ID: `{{$fromAI('Message_ID', '', 'string')}}`
    - Connect to `MainAgent`

28. **Add the final Telegram response node**
    - Add a `Telegram` node named `Send a text message`
    - Connect `MainAgent` → `Send a text message`
    - Set:
      - Text: `{{$json.output}}`
      - Chat ID: `{{$('Telegram Trigger').item.json.message.chat.id}}`
      - Parse mode: `Markdown`
      - Disable attribution
    - Recommended improvement: switch parse mode off or use plain text if you want full consistency with the agent prompt.

29. **Add optional sticky notes**
    - Add descriptive notes for:
      - Overall workflow purpose and setup
      - Data conversion block
      - AI agent block
      - Tool block

30. **Configure credentials**
    - Telegram bot credential for:
      - `Telegram Trigger`
      - `Get Audio`
      - `Telegram2`
      - `Send a text message1`
      - `Send a text message`
    - OpenAI credential for:
      - `OpenAI`
      - `Analyze image`
    - OpenRouter credential for:
      - `OpenRouter Chat Model1`
    - Google Calendar OAuth2 for all calendar tool nodes
    - Gmail OAuth2 for all Gmail tool nodes

31. **Test each branch separately**
    - Send a text message
    - Send a voice note
    - Send an image with and without caption
    - Ask calendar questions
    - Ask Gmail actions

32. **Validate known weak points before production**
    - Fix authorization logic
    - Confirm Merge behavior with only one active branch
    - Verify transcription output field name
    - Verify image analysis response path
    - Replace `photo[2]` with a safer image selection expression if needed
    - Consider Telegram Markdown escaping or parse mode removal

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Telegram AI Personal Assistant: a private assistant in Telegram that understands text, voice notes, and photos, manages Google Calendar and Gmail, and is powered by Claude Haiku via OpenRouter. | Overall workflow concept |
| Setup dependencies listed in the workflow note: Telegram API, OpenAI API, OpenRouter API, Google Calendar OAuth2, Gmail OAuth2, and an authorized Telegram numeric user ID check in the If node. | Implementation prerequisites |
| Data Conversion note: this section converts audio or image into text for LLM processing. | Media preprocessing block |
| AI Agent note: Claude Haiku processes unified input with a 30-message memory buffer and either replies conversationally or calls a tool. | Agent design |
| Agent Tools note: Google Calendar handles create/read/update/delete and availability; Gmail handles send/search/read/reply/delete. | Tool layer |

### Additional Implementation Notes
- The workflow contains **one entry point**: `Telegram Trigger`.
- The workflow contains **no sub-workflow nodes** and does not invoke any external n8n sub-workflow.
- The current exported authorization logic is incomplete as shown; it should be corrected during rebuild.
- The `Merge` node configuration may require adjustment depending on how your n8n version handles partial branch execution.