Create a WhatsApp Chatbot with GPT-4o, Whisper Transcription and Redis Buffer

https://n8nworkflows.xyz/workflows/create-a-whatsapp-chatbot-with-gpt-4o--whisper-transcription-and-redis-buffer-9440


# Create a WhatsApp Chatbot with GPT-4o, Whisper Transcription and Redis Buffer

---

### 1. Workflow Overview

This n8n workflow implements a WhatsApp chatbot integrating GPT-4o for AI conversational responses, Whisper for audio transcription, and Redis as a message buffer. It handles incoming WhatsApp messages via WasenderAPI, processes different content types (text, audio, image), and replies with context-aware AI-generated answers.

Logical blocks in the workflow:

- **1.1 Input Reception and Field Extraction**: Receives WhatsApp webhook messages, extracts and structures key message fields for downstream processing.
- **1.2 Message Buffering with Redis**: Buffers incoming messages per chat in Redis, managing message grouping and deduplication with timing logic.
- **1.3 Message Parsing and Routing**: Splits buffered messages, safely parses JSON data, and routes messages by content type (text, audio, image).
- **1.4 Content Processing**: Handles each content type specifically:
  - Text messages are prepared directly.
  - Audio messages are decrypted, downloaded, and transcribed using OpenAI Whisper.
  - Image messages are decrypted and analyzed with GPT-4o Vision.
- **1.5 AI Conversation Preparation and Execution**: Aggregates message content, composes a unified chat input, applies a LangChain AI Agent with memory and an OpenAI GPT-4.1-mini model for response generation.
- **1.6 Response Delivery**: Waits a delay to simulate human typing for anti-ban protection, then sends the AI-generated message back to the user via WasenderAPI.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Field Extraction

- **Overview**: Receives WhatsApp messages via webhook (or manual trigger for testing), extracts and normalizes all relevant message fields into a consistent structure.
- **Nodes Involved**:  
  - When clicking 'Execute workflow' (manual trigger)  
  - Set fields1

- **Node Details**:

  - *When clicking 'Execute workflow'*  
    - Type: Manual Trigger  
    - Role: Entry point for manual testing, replaceable by webhook trigger in production  
    - Connections: Output → Set fields1  
    - Edge cases: None, but manual trigger must be replaced with webhook for real use.

  - *Set fields1*  
    - Type: Set  
    - Role: Extracts and normalizes key WhatsApp message fields such as message_id, chat_id, content_type (text/audio/image), content text or caption, timestamps, media metadata (url, mimetype, mediaKey, etc.), and user info.  
    - Configuration: Uses expressions to safely extract nested data, handling optional fields.  
    - Inputs: Raw webhook/manual trigger data  
    - Outputs: Structured JSON with `message` object and media info  
    - Edge cases: Missing or malformed fields could lead to empty strings; robust extraction mitigates this.

#### 1.2 Message Buffering with Redis

- **Overview**: Buffers messages per chat ID in Redis, manages duplicates and timing to group messages for batch processing, improving AI context and efficiency.
- **Nodes Involved**:  
  - Redis  
  - Redis1  
  - Switch  
  - Wait3  
  - Redis2  
  - Split Out

- **Node Details**:

  - *Redis*  
    - Type: Redis node (push operation on list)  
    - Role: Pushes the current message data (structured JSON from Set fields1) into a Redis list keyed by `chat_id_buffer`  
    - Config: Uses `push` operation on list named with chat_id suffix "_buffer"  
    - Edge cases: Redis connection failures, message JSON serialization errors.

  - *Redis1*  
    - Type: Redis node (get operation)  
    - Role: Retrieves the full buffer list of messages for the chat from Redis to assess accumulated messages  
    - Config: Get list by key `chat_id_buffer`, property name "Mensaje"  
    - Edge cases: Redis read failures, empty buffers.

  - *Switch*  
    - Type: Switch  
    - Role: Logic to decide message handling:  
      - Ignore duplicates (based on message_id comparison)  
      - Continue if last message timestamp older than 7 seconds (enough time to batch)  
      - Wait otherwise (more messages might still be arriving)  
    - Config: Uses expression comparisons and DateTime operations  
    - Edge cases: Clock skew, parsing errors in timestamp.

  - *Wait3*  
    - Type: Wait node  
    - Role: Pauses 7 seconds to allow more messages to accumulate before processing  
    - Config: Fixed 7-second wait  
    - Edge cases: Workflow execution timeouts

  - *Redis2*  
    - Type: Redis node (delete operation)  
    - Role: Deletes the Redis buffer list for the chat after processing messages to reset buffer  
    - Edge cases: Redis failures, race conditions if messages arrive during deletion.

  - *Split Out*  
    - Type: Split Out  
    - Role: Splits the JSON array of buffered messages into individual message items for processing  
    - Edge cases: Empty array, malformed JSON.

#### 1.3 Message Parsing and Routing

- **Overview**: Parses each buffered message JSON safely, then routes each message by content type to the appropriate content processing flow.
- **Nodes Involved**:  
  - Code in JavaScript  
  - Switch Type

- **Node Details**:

  - *Code in JavaScript*  
    - Type: Code node (JavaScript)  
    - Role: Parses the `Mensaje` field JSON string into JSON objects with error handling to avoid workflow failure on malformed JSON  
    - Key code: try-catch JSON.parse, returns parsed or error object  
    - Inputs: Split Out items  
    - Outputs: Parsed JSON messages  
    - Edge cases: Malformed or corrupted JSON in buffer.

  - *Switch Type*  
    - Type: Switch  
    - Role: Routes messages by `content_type` field to one of three paths: Text, Audio, or Image  
    - Config: Uses string contains expressions to detect content type  
    - Edge cases: Unknown content types, empty content_type.

#### 1.4 Content Processing

- **Overview**: Processes each message content type with specialized logic to extract meaningful text content for AI input.

- **Nodes Involved**:

  - Text path: Text Content  
  - Audio path: Get the audio → Download the audio → Transcribe a recording → Audio Content  
  - Image path: Get the photo → Analyze image → Image Content

- **Node Details**:

  - *Text Content*  
    - Type: Set  
    - Role: Extracts message text content and timestamp into a consistent format  
    - Config: Assigns `content` and `timestamp` from message fields  
    - Edge cases: Empty or missing text content.

  - *Get the audio*  
    - Type: HTTP Request  
    - Role: Calls WasenderAPI decrypt-media endpoint for audio messages to get a usable URL or file  
    - Config: POST with JSON body containing message media metadata; uses Authorization header with API key  
    - Edge cases: API failures, invalid media metadata, auth errors.

  - *Download the audio*  
    - Type: HTTP Request  
    - Role: Downloads the decrypted audio file from the public URL provided  
    - Edge cases: Network errors, broken URL.

  - *Transcribe a recording*  
    - Type: OpenAI node (audio transcribe operation)  
    - Role: Submits audio to OpenAI Whisper for transcription  
    - Config: Uses OpenAI credentials, transcribes audio resource  
    - Edge cases: API rate limits, audio file issues, transcription errors.

  - *Audio Content*  
    - Type: Set  
    - Role: Formats transcribed text and timestamp into consistent message content  
    - Edge cases: Empty transcription.

  - *Get the photo*  
    - Type: HTTP Request  
    - Role: Calls WasenderAPI decrypt-media endpoint for image messages to get usable image URL  
    - Config: Similar to Get the audio but for images  
    - Edge cases: API failures, invalid image metadata.

  - *Analyze image*  
    - Type: OpenAI node (image analysis)  
    - Role: Sends image URL to GPT-4o Vision for descriptive analysis  
    - Config: Uses GPT-4o model, passes image URLs, asks "What's in the image?"  
    - Edge cases: API errors, unsupported image formats.

  - *Image Content*  
    - Type: Set  
    - Role: Combines user caption and AI image analysis into one content string with timestamp  
    - Edge cases: Missing caption, empty AI result.

#### 1.5 AI Conversation Preparation and Execution

- **Overview**: Aggregates all processed message contents, merges data streams, creates a unified chat input, invokes LangChain AI Agent with memory and OpenAI model to generate a contextual response.

- **Nodes Involved**:  
  - Aggregate  
  - Merge  
  - Chat Input  
  - AI Agent  
  - Simple Memory  
  - OpenAI Chat Model

- **Node Details**:

  - *Aggregate*  
    - Type: Aggregate  
    - Role: Combines all message contents into a single array under field "Content" for the chat session  
    - Edge cases: Empty input arrays.

  - *Merge*  
    - Type: Merge  
    - Role: Merges three input streams (Text, Audio, Image contents) into one unified flow  
    - Config: Number of inputs = 3  
    - Edge cases: Missing inputs, data misalignment.

  - *Chat Input*  
    - Type: Set  
    - Role: Constructs a single string `chat_input` by joining all message contents (or images) with newlines to feed the AI agent  
    - Expression: Joins `Content` array elements' `content` or `Image` fields  
    - Edge cases: Empty content array.

  - *AI Agent*  
    - Type: LangChain Agent node  
    - Role: Processes the chat input text through a LangChain agent with custom prompt  
    - Config: Text input from `chat_input` field; uses defined prompt type "define"  
    - Connections: Receives memory and language model inputs  
    - Edge cases: API errors, prompt failures, memory sync issues.

  - *Simple Memory*  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains conversational context by storing last 10 exchanges per chat session key (chat_id)  
    - Config: Session key from chat_id, custom key type, window length 10  
    - Edge cases: Memory persistence failures.

  - *OpenAI Chat Model*  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides GPT-4.1-mini model for AI Agent's language model backend  
    - Config: OpenAI credentials, GPT-4.1-mini selected  
    - Edge cases: API rate limits, auth errors.

#### 1.6 Response Delivery

- **Overview**: Introduces a delay to simulate human typing and avoid WhatsApp anti-bot detection, then sends AI-generated response to the user via WasenderAPI.

- **Nodes Involved**:  
  - Wait  
  - Send Message to User

- **Node Details**:

  - *Wait*  
    - Type: Wait  
    - Role: Delays 6 seconds before sending response  
    - Purpose: Anti-ban measure to simulate typing latency and prevent WhatsApp account suspension  
    - Edge cases: Workflow timeout, accidental removal risks.

  - *Send Message to User*  
    - Type: HTTP Request  
    - Role: Sends the AI Agent’s output message back to the WhatsApp user via WasenderAPI send-message endpoint  
    - Config: POST with 'to' parameter = chat_id, 'text' = AI output; Authorization header with API key  
    - Edge cases: API errors, invalid chat_id, network issues.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                                   | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                                           |
|-------------------------|----------------------------------|--------------------------------------------------|-----------------------------|---------------------------|-----------------------------------------------------------------------------------------------------------------------|
| When clicking 'Execute workflow' | Manual Trigger                  | Workflow entry point for manual testing          | —                           | Set fields1               | ## Change to Webhook Trigger - Replace manual trigger with webhook for production                                      |
| Set fields1             | Set                              | Extracts and normalizes WhatsApp message fields  | When clicking 'Execute workflow' | Redis                    | ## Set Fields Node - Extracts WhatsApp message data including IDs, content, timestamps, media info                     |
| Redis                   | Redis                            | Pushes message JSON into Redis buffer list       | Set fields1                  | Redis1                    | ## Message Buffer System - Stores messages in Redis list per chat                                                      |
| Redis1                  | Redis                            | Retrieves buffered messages list from Redis      | Redis                       | Switch                    | ## Message Buffer System - Reads accumulated messages to decide next action                                            |
| Switch                  | Switch                           | Logic to ignore duplicates, wait, or continue    | Redis1                      | No Operation, do nothing3 / Redis2 / Wait3 | ## Message Buffer System - Decides to ignore duplicates, continue processing, or wait for more messages                |
| No Operation, do nothing3 | NoOp                             | Dummy node for ignored messages                   | Switch                      | —                         |                                                                                                                       |
| Wait3                   | Wait                             | Pauses 7 seconds to allow message accumulation   | Switch                      | Redis1                    | ## Message Buffer System - Wait period to collect more messages                                                       |
| Redis2                  | Redis                            | Deletes Redis buffer list after processing        | Switch                      | Split Out                 | ## Message Buffer System - Clears buffer after processing                                                             |
| Split Out               | Split Out                       | Splits buffered message array into individual items | Redis2                      | Code in JavaScript         | ## Message Processing - Separates buffered messages for parsing and routing                                            |
| Code in JavaScript      | Code                             | Parses JSON messages safely with error handling  | Split Out                   | Switch Type               | ## Message Processing - Parses JSON and prepares for content routing                                                   |
| Switch Type             | Switch                           | Routes messages by content_type (text/audio/image) | Code in JavaScript           | Text Content / Get the audio / Get the photo | ## Message Processing - Routes by message content type                                                                |
| Text Content            | Set                              | Extracts text content and timestamp               | Switch Type (Text)           | Aggregate                 | ## Content Processing - Prepares text messages                                                                         |
| Get the audio           | HTTP Request                    | Decrypts audio media via WasenderAPI              | Switch Type (Audio)          | Download the audio        | ## Content Processing - Decrypts audio messages                                                                        |
| Download the audio      | HTTP Request                    | Downloads decrypted audio file                      | Get the audio               | Transcribe a recording    |                                                                                                                       |
| Transcribe a recording  | OpenAI (audio transcription)    | Transcribes audio to text via OpenAI Whisper      | Download the audio           | Audio Content             | ## Content Processing - Transcribes audio messages                                                                     |
| Audio Content           | Set                              | Formats transcribed audio text and timestamp      | Transcribe a recording       | Aggregate                 |                                                                                                                       |
| Get the photo           | HTTP Request                    | Decrypts image media via WasenderAPI               | Switch Type (Image)          | Analyze image             | ## Content Processing - Decrypts image messages                                                                        |
| Analyze image           | OpenAI (image analysis)          | Analyzes image content using GPT-4o Vision        | Get the photo               | Image Content             |                                                                                                                       |
| Image Content           | Set                              | Combines user caption and AI image analysis       | Analyze image               | Aggregate                 |                                                                                                                       |
| Aggregate               | Aggregate                       | Combines all processed message contents           | Text Content / Audio Content / Image Content | Merge                      | ## AI Conversation System - Aggregates message contents                                                               |
| Merge                   | Merge                           | Merges data streams from different content types  | Aggregate                   | Chat Input                |                                                                                                                       |
| Chat Input              | Set                              | Creates unified chat input string for AI agent    | Merge                      | AI Agent                  |                                                                                                                       |
| AI Agent                | LangChain Agent                 | Processes chat input with defined prompt and memory | Chat Input / Simple Memory / OpenAI Chat Model | Wait                      | ## AI Conversation System - Context-aware AI agent with memory                                                        |
| Simple Memory           | LangChain Memory Buffer Window  | Maintains last 10 messages per chat for context   | —                           | AI Agent (memory input)   |                                                                                                                       |
| OpenAI Chat Model       | LangChain OpenAI Chat Model     | GPT-4.1-mini model for AI agent                    | —                           | AI Agent (language model) |                                                                                                                       |
| Wait                    | Wait                            | Waits 6 seconds to simulate typing, anti-ban      | AI Agent                   | Send Message to User      | ## Response Delivery - Critical anti-ban protection; simulates human typing                                            |
| Send Message to User    | HTTP Request                    | Sends AI response back to WhatsApp user via WasenderAPI | Wait                      | —                         | ## Response Delivery - Sends final AI-generated message                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Trigger**:  
   - Add a **Manual Trigger** node named "When clicking 'Execute workflow'" for testing (replace with a webhook node for production). No configuration needed.

2. **Set Fields Node**:  
   - Add a **Set** node named "Set fields1".  
   - Configure fields to extract WhatsApp message data from webhook JSON body, including:  
     - `message.message_id` from `body.data.messages.key.id`  
     - `message.chat_id` from `body.data.messages.key.remoteJid`  
     - `content_type` based on presence of text/audio/image in message  
     - `content` from conversation text or image caption  
     - `timestamp` converted to ISO UTC from `body.timestamp`  
     - Media fields: `url`, `mimetype`, `mediaKey`, `fileSha256`, `fileLength`, `jpegThumbnail`  
     - `user.name` from `body.data.messages.pushName`  
   - Connect Manual Trigger output to this node.

3. **Redis Buffer Push**:  
   - Add a **Redis** node named "Redis".  
   - Configure credentials for Redis (free tier recommended).  
   - Operation: `push` to list named `{{ $json.message.chat_id }}_buffer`.  
   - Message data: JSON stringify of all relevant message fields from "Set fields1".  
   - Connect "Set fields1" output to "Redis".

4. **Redis Buffer Read**:  
   - Add a **Redis** node named "Redis1".  
   - Operation: `get` list by key `{{ $json.message.chat_id }}_buffer`.  
   - Property name: "Mensaje".  
   - Connect "Redis" output to "Redis1".

5. **Message Processing Switch**:  
   - Add a **Switch** node named "Switch".  
   - Configure rules:  
     - "Ignore" if last buffered message_id ≠ current message_id (duplicate).  
     - "Continue" if last message timestamp older than 7 seconds (enough time to process).  
     - "Wait" fallback if neither condition met.  
   - Connect "Redis1" output to "Switch".

6. **No Operation Node**:  
   - Add a **No Operation** node "No Operation, do nothing3" for ignored messages.  
   - Connect "Switch" 'Ignore' output to this node.

7. **Wait Node for Buffering**:  
   - Add a **Wait** node "Wait3" with 7 seconds delay.  
   - Connect "Switch" 'Wait' output to this node.  
   - Connect "Wait3" output back to "Redis1" to recheck buffer.

8. **Redis Buffer Clear**:  
   - Add a **Redis** node "Redis2".  
   - Operation: `delete` key `{{ $json.message.chat_id }}_buffer`.  
   - Connect "Switch" 'Continue' output to "Redis2".

9. **Split Buffered Messages**:  
   - Add a **Split Out** node "Split Out" to split the array from Redis buffer.  
   - Field to split: "Mensaje".  
   - Connect "Redis2" output to "Split Out".

10. **Parse JSON Messages**:  
    - Add a **Code** node "Code in JavaScript".  
    - Script: Parse each item’s "Mensaje" JSON string with try-catch to handle errors safely.  
    - Connect "Split Out" output to this node.

11. **Content Type Switch**:  
    - Add a **Switch** node "Switch Type".  
    - Rules: Route by `content_type` field containing "text", "audio", or "image".  
    - Connect "Code in JavaScript" output to this node.

12. **Text Content Processing**:  
    - Add a **Set** node "Text Content".  
    - Assign `content` and `timestamp` from parsed message fields.  
    - Connect "Switch Type" 'Text' output to here.

13. **Audio Content Processing**:  
    - Add a **HTTP Request** node "Get the audio" to call WasenderAPI decrypt-media endpoint with audio message metadata.  
    - Use POST with JSON body referencing media fields (`message_id`, `url`, `mimetype`, etc.).  
    - Add Authorization header with WasenderAPI API Key.  
    - Connect "Switch Type" 'Audio' output to this node.

14. **Download Audio**:  
    - Add **HTTP Request** node "Download the audio" to download audio from decrypted URL.  
    - URL comes from previous node's output field `publicUrl`.  
    - Connect "Get the audio" output to this node.

15. **Transcribe Audio**:  
    - Add **OpenAI** node "Transcribe a recording".  
    - Operation: Transcribe audio resource.  
    - Use OpenAI credentials with Whisper API.  
    - Connect "Download the audio" output to this node.

16. **Format Audio Content**:  
    - Add **Set** node "Audio Content".  
    - Assign `content` from transcription text, `timestamp` from original message.  
    - Connect "Transcribe a recording" output here.

17. **Image Content Processing**:  
    - Add **HTTP Request** node "Get the photo" to decrypt image via WasenderAPI.  
    - Use POST with JSON body for image media metadata, with Authorization header.  
    - Connect "Switch Type" 'Image' output to this node.

18. **Analyze Image**:  
    - Add **OpenAI** node "Analyze image".  
    - Operation: Analyze image resource with GPT-4o Vision model.  
    - Input: Image URLs from previous node.  
    - Connect "Get the photo" output here.

19. **Format Image Content**:  
    - Add **Set** node "Image Content".  
    - Combine user caption and AI image analysis into `Image` field, include timestamp.  
    - Connect "Analyze image" output here.

20. **Aggregate Contents**:  
    - Add **Aggregate** node "Aggregate".  
    - Aggregate all content items into one array under field "Content".  
    - Connect outputs of "Text Content", "Audio Content", and "Image Content" to this node (multiple inputs).

21. **Merge Streams**:  
    - Add **Merge** node "Merge" with 3 inputs to unify text/audio/image content streams.  
    - Connect "Aggregate" output to "Merge".

22. **Create Unified Chat Input**:  
    - Add **Set** node "Chat Input".  
    - Assign `chat_input` string by joining all contents or images with newline separator: `={{ $json.Content.map(msg => msg.content || msg.Image).join('\n') }}`.  
    - Connect "Merge" output here.

23. **Configure AI Agent**:  
    - Add **LangChain Agent** node "AI Agent".  
    - Input text from `chat_input`.  
    - Prompt type: "define" (custom prompt).  
    - Connect "Chat Input" output here.

24. **Configure Memory**:  
    - Add **LangChain Memory Buffer Window** node "Simple Memory".  
    - Session key: `={{ $('Set fields1').item.json.message.chat_id }}`.  
    - Context window length: 10 messages.  
    - Connect memory output to "AI Agent" memory input.

25. **Configure Language Model**:  
    - Add **LangChain OpenAI Chat Model** node "OpenAI Chat Model".  
    - Model: GPT-4.1-mini.  
    - Use OpenAI API credentials.  
    - Connect language model output to "AI Agent" languageModel input.

26. **Delay Before Sending Response**:  
    - Add **Wait** node "Wait" with 6 seconds delay.  
    - Connect "AI Agent" output to this node.

27. **Send Message to User**:  
    - Add **HTTP Request** node "Send Message to User".  
    - POST to WasenderAPI send-message endpoint.  
    - Body parameters: `to` = chat_id, `text` = AI Agent's output.  
    - Set Authorization header with WasenderAPI API key.  
    - Connect "Wait" output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                   | Context or Link                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| WasenderAPI Setup: Signup at [wasenderapi.com](https://wasenderapi.com), create session with WhatsApp number, scan QR, enable webhook for `messages.received`. | Sticky Note near workflow start                              |
| For new WhatsApp numbers, manually use for 7 days before automation to avoid detection.                                                                        | Sticky Note near workflow start                              |
| Redis free tier available at [redis.io](https://redis.io) for message buffering.                                                                              | Sticky Note near Redis nodes                                 |
| OpenAI API key setup at [platform.openai.com](https://platform.openai.com), adding payment method required.                                                   | Sticky Note near AI nodes                                    |
| Approximate costs: $0.006 per minute for audio transcription, $0.01 per image analysis.                                                                         | Sticky Note near AI nodes                                    |
| Response delay (6 seconds) critical to avoid WhatsApp account suspension due to instant bot responses.                                                         | Sticky Note near Wait node before response sending          |

---

**Disclaimer:** This document is generated from an n8n workflow automation. All data processed is compliant with content policies and contains only legal, non-offensive information.

---