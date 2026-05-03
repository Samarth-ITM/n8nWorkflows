Build a WhatsApp Assistant for Text, Audio & Images using GPT-4o & Evolution API

https://n8nworkflows.xyz/workflows/build-a-whatsapp-assistant-for-text--audio---images-using-gpt-4o---evolution-api-11754


# Build a WhatsApp Assistant for Text, Audio & Images using GPT-4o & Evolution API

### 1. Workflow Overview

This workflow implements a **WhatsApp Multimodal AI Assistant** capable of handling text, audio, image, and document messages through WhatsApp using Evolution API and OpenAI GPT-4o models. It supports multimodal input processing, message queuing to accumulate user input before AI response, conversation memory using PostgreSQL, and smart formatting and sending of replies with natural delays.

**Target Use Cases:**  
- Automating WhatsApp customer support or virtual assistant interactions.  
- Processing and transcribing voice/audio messages.  
- Analyzing images and documents sent via WhatsApp.  
- Maintaining conversation context for coherent multi-turn dialogs.  

**Logical Blocks:**  
- **1.1 Input Reception**: Receiving and parsing WhatsApp messages via Evolution API webhook.  
- **1.2 Mark as Read**: Mark incoming messages as read in WhatsApp to improve UX.  
- **1.3 Message Routing**: Routing messages by type (text, audio, image, document, or unsupported).  
- **1.4 Media Processing**: Transcribing audio, analyzing images, extracting text from documents.  
- **1.5 Message Queuing**: Storing incoming messages temporarily in Redis to accumulate multi-part inputs.  
- **1.6 Check & Concatenate**: Detect if new message is the last in a sequence, concatenate queued messages.  
- **1.7 AI Processing**: Use GPT-4o-based AI agent with conversation memory to generate responses.  
- **1.8 Response Formatting**: Format AI-generated long responses into multiple WhatsApp-friendly messages.  
- **1.9 Message Sending**: Sequentially send formatted messages with natural typing delays, including images.  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Receives WhatsApp messages via a webhook from Evolution API and extracts relevant data fields for further processing.  
- **Nodes Involved:** Receive WhatsApp, Parse Message Data, Filter Allowed Numbers, Wait 2s  
- **Node Details:**  
  - **Receive WhatsApp**  
    - Type: Webhook  
    - Configured to listen for POST requests at `/whatsapp-multimodal` path.  
    - Entry point for incoming WhatsApp messages.  
    - Outputs raw webhook JSON containing message data.  
    - Failure Modes: Missing webhook setup, Evolution API connectivity issues, invalid payloads.  
  - **Parse Message Data**  
    - Type: Set  
    - Extracts message event, instance, remoteJid (sender), messageId, pushName, messageText, and messageType from webhook JSON.  
    - Converts nested JSON to flat properties for easier usage downstream.  
    - Potential failure: Missing fields in JSON if Evolution API format changes.  
  - **Filter Allowed Numbers**  
    - Type: Filter  
    - Condition to allow messages only from specified WhatsApp number(s) (replace placeholder `YOUR_ALLOWED_NUMBER`).  
    - Prevents processing of unauthorized senders.  
    - Failure: Misconfiguration leading to filter excluding all messages.  
  - **Wait 2s**  
    - Type: Wait  
    - Introduces a 2-second delay before marking message as read, possibly to handle message ordering or rate limits.

#### 1.2 Mark as Read

- **Overview:** Marks the incoming WhatsApp message as read via Evolution API to acknowledge receipt.  
- **Nodes Involved:** Mark as Read  
- **Node Details:**  
  - **Mark as Read**  
    - Type: Evolution API node (chat-api/read-messages)  
    - Uses messageId, remoteJid, and instance from previous nodes to mark the message read.  
    - Failure: Auth errors, API downtime, invalid messageId.

#### 1.3 Message Routing

- **Overview:** Routes messages to different processing branches depending on their type (text, audio, image, document, unsupported).  
- **Nodes Involved:** Route by Message Type  
- **Node Details:**  
  - **Route by Message Type**  
    - Type: Switch node  
    - Routes on `messageType` field to outputs: text (conversation), audioMessage, imageMessage, documentMessage, or fallback unsupported.  
    - Failure: Unknown message types or malformed data can end in unsupported branch.

#### 1.4 Media Processing

- **Overview:** Processes media inputs by converting base64 media data to binary, then transcribing or analyzing via OpenAI models.  
- **Nodes Involved:**  
  - Text: Extract Text  
  - Audio: Extract Audio Base64 → Convert to Audio → Transcribe Audio → Format Audio Input  
  - Image: Extract Image Base64 → Convert to Image → Analyze Image → Format Image Input  
  - Document: Extract Document Base64 → Convert to Document → Transcribe Document → Format Document Input  
  - Unsupported: Send Unsupported Message → End Unsupported  

- **Node Details:**  
  - **Extract Base64 Nodes (Audio/Image/Document)**  
    - Type: Set  
    - Extract `base64` encoded media from the webhook JSON for further conversion.  
  - **Convert to File nodes**  
    - Type: ConvertToFile  
    - Convert base64 data to binary files with appropriate MIME types (audio/wav, image/png, document mimetype).  
    - Failure: Invalid or corrupted base64 data.  
  - **Transcribe Audio / Transcribe Document**  
    - Type: OpenAI Audio Transcription / Langchain OpenAI  
    - Transcribes speech to text or extracts text from documents using GPT-4o.  
    - Failure: API limits, audio quality issues, unsupported document formats.  
  - **Analyze Image**  
    - Type: Langchain OpenAI Image Analysis  
    - Describes image content and extracts visible text.  
    - Failure: Complex images may lead to inaccurate descriptions.  
  - **Format Input Nodes**  
    - Type: Set  
    - Wrap transcribed or analyzed content inside XML-like tags (`<audio>`, `<image>`, `<document>`) for downstream concatenation and AI input.  
  - **Send Unsupported Message**  
    - Type: Evolution API messages-api  
    - Sends a default message for unsupported media types.  
    - Followed by a NoOp to end flow.

#### 1.5 Message Queuing

- **Overview:** Queues incoming messages in Redis keyed by sender JID to accumulate multiple messages before AI processing.  
- **Nodes Involved:** Unify Input, Queue Message, Wait for More Messages, Get Queued Messages  
- **Node Details:**  
  - **Unify Input**  
    - Type: Set  
    - Standardizes collected inputs under `input` field for queueing.  
  - **Queue Message**  
    - Type: Redis (push to list)  
    - Pushes the current input string to Redis list identified by sender JID.  
    - Failure: Redis connectivity or authentication issues.  
  - **Wait for More Messages**  
    - Type: Wait  
    - Waits 10 seconds to allow more messages before concatenation.  
  - **Get Queued Messages**  
    - Type: Redis (get list)  
    - Retrieves all messages from Redis list for sender.  
    - Failure: Redis errors or key absence.

#### 1.6 Check & Concatenate

- **Overview:** Checks if the current message is the last in the queue and concatenates all queued messages into one input string for AI.  
- **Nodes Involved:** Is Last Message?, Concatenate Messages, Not Last - Stop  
- **Node Details:**  
  - **Is Last Message?**  
    - Type: If  
    - Compares last message in queue with current input to detect message sequence completion.  
  - **Concatenate Messages**  
    - Type: Set  
    - Joins all queued messages with newline separators into a single `finalInput` string.  
  - **Not Last - Stop**  
    - Type: NoOp  
    - Ends workflow branch if more messages expected.

#### 1.7 AI Processing

- **Overview:** Sends concatenated input to GPT-4o powered AI agent with conversation memory stored in PostgreSQL for response generation.  
- **Nodes Involved:** AI Agent, OpenAI Chat Model, Postgres Chat Memory  
- **Node Details:**  
  - **AI Agent**  
    - Type: Langchain Agent  
    - Receives user message and current date.  
    - Uses system prompt to act as a friendly virtual assistant that avoids hallucination.  
  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model (gpt-4o-mini)  
    - Chat completion model invoked by AI Agent.  
  - **Postgres Chat Memory**  
    - Type: Langchain PostgreSQL Memory  
    - Stores conversation history keyed by sender JID.  
    - Context window length set to 20 messages.  
    - Failure: DB connectivity or schema issues.

#### 1.8 Response Formatting

- **Overview:** Formats AI-generated responses into multiple WhatsApp-friendly messages using OpenAI GPT-4o-mini, then splits and loops through them.  
- **Nodes Involved:** Format Response, OpenAI Chat Model1, Structured Output Parser, Split Messages, Loop Messages, Is Image URL?  
- **Node Details:**  
  - **Format Response**  
    - Type: Langchain Chain LLM  
    - Sends AI output to specialized prompt that splits long responses into 3-4 line WhatsApp messages with emojis and logical structure.  
  - **OpenAI Chat Model1**  
    - Type: Langchain OpenAI Chat Model (gpt-4o-mini)  
    - Used by Format Response node for generation.  
  - **Structured Output Parser**  
    - Type: Langchain Structured Output Parser  
    - Parses JSON array of messages from formatted output.  
  - **Split Messages**  
    - Type: SplitOut  
    - Splits JSON array into individual message items for sending.  
  - **Loop Messages**  
    - Type: SplitInBatches  
    - Iterates over each message piece to send sequentially.  
  - **Is Image URL?**  
    - Type: If  
    - Checks if current message ends with image file extensions (.png, .jpg, .jpeg) to decide sending method.  
    - Failure: Misclassification if URLs are malformed.

#### 1.9 Message Sending

- **Overview:** Sends formatted messages back to WhatsApp with delays between messages to simulate natural typing. Supports image and text messages.  
- **Nodes Involved:** Send Image, Send Text, Wait Between Messages, Aggregate, Clear Queue, Done  
- **Node Details:**  
  - **Send Image**  
    - Type: Evolution API messages-api (send-image)  
    - Sends images referenced by URLs in messages.  
  - **Send Text**  
    - Type: Evolution API messages-api (send-text)  
    - Sends text messages with a delay proportional to message length (delay in seconds = message length * 60).  
  - **Wait Between Messages**  
    - Type: Wait  
    - 2-second delay between sending messages for natural pacing.  
  - **Aggregate**  
    - Type: Aggregate  
    - Collects message send outputs (possibly for logging or tracking).  
  - **Clear Queue**  
    - Type: Redis (delete)  
    - Clears Redis message queue after all messages sent to avoid duplicate processing.  
  - **Done**  
    - Type: NoOp  
    - Marks end of workflow.

---

### 3. Summary Table

| Node Name               | Node Type                                  | Functional Role                          | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                                                           |
|-------------------------|--------------------------------------------|----------------------------------------|---------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| Main Overview           | Sticky Note                                | Overview and documentation              |                           |                            | WhatsApp Multimodal AI Assistant overview, setup steps, customization instructions.                                                  |
| Warning - Setup         | Sticky Note                                | Setup requirements and instructions    |                           |                            | ⚠️ Requirements & Setup: Evolution API, Redis, PostgreSQL, OpenAI; webhook setup instructions.                                        |
| Section 1 - Receive     | Sticky Note                                | Marks input reception block             |                           |                            | 1. Receive & Parse Message                                                                                                            |
| Section 2 - Mark Read   | Sticky Note                                | Marks message read block                |                           |                            | 2. Mark as Read                                                                                                                       |
| Section 3 - Router      | Sticky Note                                | Message routing block                   |                           |                            | 3. Route by Message Type                                                                                                              |
| Section 4 - Media Processing | Sticky Note                           | Media processing block                  |                           |                            | 4. Process Media: audio transcription, image analysis, document extraction                                                           |
| Section 5 - Queue       | Sticky Note                                | Message queuing block                   |                           |                            | 5. Message Queue: accumulates messages and waits for more input                                                                       |
| Section 6 - Check       | Sticky Note                                | Check and concatenate messages         |                           |                            | 6. Check & Concatenate                                                                                                                |
| Section 7 - AI          | Sticky Note                                | AI processing block                     |                           |                            | 7. AI Processing                                                                                                                      |
| Section 8 - Format      | Sticky Note                                | Response formatting block               |                           |                            | 8. Format Response                                                                                                                    |
| Section 9 - Send        | Sticky Note                                | Sending messages block                  |                           |                            | 9. Send Messages                                                                                                                      |
| Receive WhatsApp        | Webhook                                   | Entry webhook for WhatsApp messages    |                           | Parse Message Data          |                                                                                                                                       |
| Parse Message Data      | Set                                        | Extract message data fields             | Receive WhatsApp           | Filter Allowed Numbers      |                                                                                                                                       |
| Filter Allowed Numbers  | Filter                                     | Allow only messages from authorized number | Parse Message Data      | Wait 2s                    |                                                                                                                                       |
| Wait 2s                 | Wait                                       | Delay before marking as read            | Filter Allowed Numbers     | Mark as Read               |                                                                                                                                       |
| Mark as Read            | Evolution API                              | Mark WhatsApp message as read           | Wait 2s                   | Route by Message Type       |                                                                                                                                       |
| Route by Message Type   | Switch                                     | Route by message type                   | Mark as Read              | Extract Text, Extract Audio Base64, Extract Image Base64, Extract Document Base64, Send Unsupported Message |                                                                              |
| Extract Text            | Set                                        | Extract text input                      | Route by Message Type (text) | Unify Input             |                                                                                                                                       |
| Extract Audio Base64    | Set                                        | Extract audio base64                    | Route by Message Type (audio) | Convert to Audio          |                                                                                                                                       |
| Convert to Audio        | ConvertToFile                              | Convert base64 string to audio binary  | Extract Audio Base64       | Transcribe Audio           |                                                                                                                                       |
| Transcribe Audio        | OpenAI Audio Transcription                  | Transcribe audio to text                | Convert to Audio           | Format Audio Input         |                                                                                                                                       |
| Format Audio Input      | Set                                        | Format transcribed audio text           | Transcribe Audio           | Unify Input                |                                                                                                                                       |
| Extract Image Base64    | Set                                        | Extract image base64                    | Route by Message Type (image) | Convert to Image         |                                                                                                                                       |
| Convert to Image        | ConvertToFile                              | Convert base64 string to image binary  | Extract Image Base64       | Analyze Image              |                                                                                                                                       |
| Analyze Image           | OpenAI Image Analysis                      | Analyze image content and text         | Convert to Image           | Format Image Input         |                                                                                                                                       |
| Format Image Input      | Set                                        | Format analyzed image text              | Analyze Image              | Unify Input                |                                                                                                                                       |
| Extract Document Base64 | Set                                        | Extract document base64                 | Route by Message Type (document) | Convert to Document     |                                                                                                                                       |
| Convert to Document     | ConvertToFile                              | Convert base64 to document file        | Extract Document Base64    | Transcribe Document        |                                                                                                                                       |
| Transcribe Document     | OpenAI Document Transcription              | Extract text from document              | Convert to Document        | Format Document Input      |                                                                                                                                       |
| Format Document Input   | Set                                        | Format document transcription           | Transcribe Document        | Unify Input                |                                                                                                                                       |
| Send Unsupported Message| Evolution API                              | Notify unsupported message types       | Route by Message Type (fallback) | End Unsupported         |                                                                                                                                       |
| End Unsupported         | NoOp                                       | End unsupported message branch         | Send Unsupported Message   |                            |                                                                                                                                       |
| Unify Input             | Set                                        | Standardize input field                 | Extract Text/Audio/Image/Document Format nodes | Queue Message      |                                                                                                                                       |
| Queue Message           | Redis                                      | Push message to Redis list queue       | Unify Input               | Wait for More Messages     |                                                                                                                                       |
| Wait for More Messages  | Wait                                       | Wait 10 seconds for more input         | Queue Message              | Get Queued Messages        |                                                                                                                                       |
| Get Queued Messages     | Redis                                      | Retrieve all queued messages            | Wait for More Messages     | Is Last Message?           |                                                                                                                                       |
| Is Last Message?        | If                                         | Check if current message is last       | Get Queued Messages        | Concatenate Messages (yes), Not Last - Stop (no) |                                                                                      |
| Concatenate Messages    | Set                                        | Join queued messages into `finalInput` | Is Last Message? (yes)     | AI Agent                  |                                                                                                                                       |
| Not Last - Stop         | NoOp                                       | Stop workflow if not last message      | Is Last Message? (no)      |                            |                                                                                                                                       |
| AI Agent                | Langchain Agent                            | Generate AI response with memory       | Concatenate Messages       | Format Response            |                                                                                                                                       |
| OpenAI Chat Model       | Langchain OpenAI Chat Model (GPT-4o-mini) | Language model for AI Agent             | AI Agent (ai_languageModel) | AI Agent                 |                                                                                                                                       |
| Postgres Chat Memory    | Langchain PostgreSQL Memory                | Store conversation history              | AI Agent (ai_memory)       |                            |                                                                                                                                       |
| Format Response         | Langchain Chain LLM                        | Format AI output into multiple messages| AI Agent                   | Split Messages             |                                                                                                                                       |
| OpenAI Chat Model1      | Langchain OpenAI Chat Model (GPT-4o-mini) | Language model for response formatting | Format Response (ai_languageModel) | Format Response        |                                                                                                                                       |
| Structured Output Parser| Langchain Structured Output Parser         | Parse JSON formatted response           | Format Response (ai_outputParser) | Format Response        |                                                                                                                                       |
| Split Messages          | SplitOut                                   | Split JSON array into individual messages | Format Response           | Loop Messages              |                                                                                                                                       |
| Loop Messages           | SplitInBatches                             | Iterate over messages to send           | Split Messages             | Aggregate, Is Image URL?   |                                                                                                                                       |
| Is Image URL?           | If                                         | Check if message is an image URL        | Loop Messages              | Send Image (yes), Send Text (no) |                                                                                                                                    |
| Send Image              | Evolution API                              | Send image message                      | Is Image URL? (yes)        | Wait Between Messages      |                                                                                                                                       |
| Send Text               | Evolution API                              | Send text message                      | Is Image URL? (no)         | Wait Between Messages      |                                                                                                                                       |
| Wait Between Messages   | Wait                                       | Wait 2 seconds between messages         | Send Text/Send Image       | Loop Messages (next batch) |                                                                                                                                       |
| Aggregate               | Aggregate                                  | Aggregate results of sent messages      | Loop Messages              | Clear Queue                |                                                                                                                                       |
| Clear Queue             | Redis                                      | Delete Redis message queue              | Aggregate                  | Done                      |                                                                                                                                       |
| Done                    | NoOp                                       | Workflow end marker                     | Clear Queue                |                            |                                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Receive WhatsApp"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `whatsapp-multimodal`  
   - Purpose: Entry point for Evolution API webhook messages.

2. **Add Set Node: "Parse Message Data"**  
   - Extract fields:  
     - event: `{{$json.body.event}}`  
     - instance: `{{$json.body.instance}}`  
     - remoteJid: `{{$json.body.data.key.remoteJid}}`  
     - messageId: `{{$json.body.data.key.id}}`  
     - pushName: `{{$json.body.data.pushName}}`  
     - messageText: `{{$json.body.data.message.conversation}}`  
     - messageType: `{{$json.body.data.messageType}}`  
   - Connect from "Receive WhatsApp".

3. **Add Filter Node: "Filter Allowed Numbers"**  
   - Condition: `remoteJid == "YOUR_ALLOWED_NUMBER@s.whatsapp.net"` (replace placeholder)  
   - Connect from "Parse Message Data".

4. **Add Wait Node: "Wait 2s"**  
   - Delay: 2 seconds  
   - Connect from "Filter Allowed Numbers".

5. **Add Evolution API Node: "Mark as Read"**  
   - Resource: chat-api  
   - Operation: read-messages  
   - Parameters:  
     - messageId: `{{$node["Parse Message Data"].json["messageId"]}}`  
     - remoteJid: `{{$node["Filter Allowed Numbers"].json["remoteJid"]}}`  
     - instanceName: `{{$node["Parse Message Data"].json["instance"]}}`  
   - Connect from "Wait 2s".

6. **Add Switch Node: "Route by Message Type"**  
   - Route on `messageType` with outputs:  
     - `"conversation"` → Text branch  
     - `"audioMessage"` → Audio branch  
     - `"imageMessage"` → Image branch  
     - `"documentMessage"` → Document branch  
     - Fallback → Unsupported branch  
   - Connect from "Mark as Read".

7. **Text Branch:**  
   - Add Set node "Extract Text"  
     - Assign `input` = `{{$node["Parse Message Data"].json["messageText"]}}`  
   - Connect from "Route by Message Type" (text output).

8. **Audio Branch:**  
   - Set "Extract Audio Base64"  
     - Assign `data` = `{{$node["Receive WhatsApp"].json.body.data.message.base64}}`  
   - ConvertToFile node "Convert to Audio"  
     - Mime type: audio/wav  
     - Source property: `data`  
   - OpenAI Audio Transcription node "Transcribe Audio"  
   - Set node "Format Audio Input"  
     - Assign `input` = `<audio>\n{{$json.text}}\n</audio>`  
   - Connect nodes sequentially from "Route by Message Type" (audio).

9. **Image Branch:**  
   - Set "Extract Image Base64"  
     - Assign `data` = `{{$node["Receive WhatsApp"].json.body.data.message.base64}}`  
   - ConvertToFile "Convert to Image"  
     - Mime type: image/png  
   - OpenAI Image Analysis node "Analyze Image"  
     - Prompt: Describe image with visible text  
     - Model: gpt-4o  
   - Set "Format Image Input"  
     - Assign `input` = `<image>\n{{$json['0'].content[0].text}}\n</image>`  
   - Connect sequentially from switch image output.

10. **Document Branch:**  
    - Set "Extract Document Base64"  
      - Assign `data` = `{{$node["Receive WhatsApp"].json.body.data.message.base64}}`  
    - ConvertToFile "Convert to Document"  
      - File name: `{{$node["Receive WhatsApp"].json.body.data.message.documentMessage.fileName}}`  
      - Mime type: `{{$node["Receive WhatsApp"].json.body.data.message.documentMessage.mimetype}}`  
    - OpenAI Document Transcription "Transcribe Document"  
      - System prompt: Describe document and transcribe text  
      - Attach file from previous node  
    - Set "Format Document Input"  
      - Assign `input` = `<document>\n{{$json.output[0].content[0].text}}\n</document>`  
    - Connect sequentially from switch document output.

11. **Unsupported Branch:**  
    - Evolution API node "Send Unsupported Message"  
      - Text: "Sorry, I cannot process this type of message yet."  
    - NoOp node "End Unsupported"  
    - Connect from switch fallback output.

12. **Unify Input:**  
    - Set node "Unify Input"  
      - Assign `input` = `{{$json.input}}` from any media/text format nodes.  
    - Connect all media/text format nodes to "Unify Input".

13. **Queue Message:**  
    - Redis node "Queue Message"  
      - Operation: push to list  
      - List key: `{{$node["Parse Message Data"].json["remoteJid"]}}`  
      - Message data: `{{$json.input}}`  
    - Connect from "Unify Input".

14. **Wait for More Messages:**  
    - Wait node "Wait for More Messages"  
      - Delay: 10 seconds  
    - Connect from "Queue Message".

15. **Get Queued Messages:**  
    - Redis node "Get Queued Messages"  
      - Operation: get list  
      - Key: `{{$node["Parse Message Data"].json["remoteJid"]}}`  
    - Connect from "Wait for More Messages".

16. **Check Last Message:**  
    - If node "Is Last Message?"  
      - Condition: last message in Redis list equals current input  
    - Connect from "Get Queued Messages".

17. **Concatenate Messages:**  
    - Set node "Concatenate Messages"  
      - Assign `finalInput` = joined Redis list messages with `\n` separator  
    - Connect from "Is Last Message?" (true output).

18. **Stop if Not Last:**  
    - NoOp node "Not Last - Stop"  
    - Connect from "Is Last Message?" (false output).

19. **AI Agent:**  
    - Langchain Agent node "AI Agent"  
      - Input: `User message: {{ $json.finalInput }}\n\nCurrent date: {{ $now }}`  
      - System message: Friendly assistant, no hallucination  
    - Connect from "Concatenate Messages".

20. **OpenAI Chat Model:**  
    - Langchain OpenAI Chat Model (gpt-4o-mini)  
    - Connect as AI language model for "AI Agent".

21. **Postgres Chat Memory:**  
    - Langchain Postgres Memory node  
    - Table: `whatsapp_chat_memory`  
    - Session key: `{{$node["Parse Message Data"].json["remoteJid"]}}`  
    - Context window length: 20  
    - Connect as AI memory input to "AI Agent".

22. **Format Response:**  
    - Langchain Chain LLM node "Format Response"  
      - Prompt to split long responses into JSON array of WhatsApp messages (3 messages, emojis, line limits)  
    - Connect from "AI Agent".

23. **OpenAI Chat Model1:**  
    - Langchain OpenAI Chat Model (gpt-4o-mini)  
    - Connect as language model for "Format Response".

24. **Structured Output Parser:**  
    - Parses JSON response from "Format Response" into usable array.  
    - Connect from "Format Response".

25. **Split Messages:**  
    - SplitOut node to split `output.respuesta` array into individual messages.  
    - Connect from "Format Response".

26. **Loop Messages:**  
    - SplitInBatches node to iterate over messages in batches (default batch size).  
    - Connect from "Split Messages".

27. **Is Image URL?:**  
    - If node to check if message ends with `.png`, `.jpg`, `.jpeg`.  
    - Connect from "Loop Messages".

28. **Send Image:**  
    - Evolution API messages-api node to send image message.  
    - Media URL: `{{$json['output.respuesta']}}`  
    - Connect from "Is Image URL?" (true output).

29. **Send Text:**  
    - Evolution API messages-api node to send text message.  
    - Message text: `{{$json['output.respuesta']}}`  
    - Delay option: length of message * 60 seconds  
    - Connect from "Is Image URL?" (false output).

30. **Wait Between Messages:**  
    - Wait node with 2 seconds delay between messages.  
    - Connect from both "Send Image" and "Send Text".

31. **Aggregate:**  
    - Aggregate node to collect outputs from message sending for cleanup.  
    - Connect from "Loop Messages".

32. **Clear Queue:**  
    - Redis node to delete message queue with key = sender JID.  
    - Connect from "Aggregate".

33. **Done:**  
    - NoOp node to mark end of workflow.  
    - Connect from "Clear Queue".

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Setup requires Evolution API community node `n8n-nodes-evolution-api` and valid API credentials for Evolution API, OpenAI, Redis, and PostgreSQL. | Setup instructions and community node link are in sticky notes at workflow start.                |
| The workflow enforces a message filtering by allowed WhatsApp numbers to avoid unauthorized access.                           | Replace `"YOUR_ALLOWED_NUMBER"` with your actual WhatsApp number in international format.         |
| Message queueing is implemented with Redis lists keyed by sender JID to accumulate conversation segments before AI response. | Redis instance must be accessible and properly configured in n8n credentials.                      |
| Conversation memory stored in PostgreSQL table `whatsapp_chat_memory` keyed by WhatsApp user ID to support multi-turn dialogs. | PostgreSQL schema must be prepared accordingly; Langchain PostgreSQL memory node depends on it.    |
| OpenAI GPT-4o and GPT-4o-mini models are used for transcription, analysis, AI agent, and response formatting.                  | Ensure API keys have access to these models; watch for token and rate limits in OpenAI usage.     |
| Response formatting prompt enforces JSON output with array of messages optimized for WhatsApp length limits and structure.    | Do not modify the JSON output schema or formatting prompt unless handling different message styles. |
| Natural delays between messages emulate human typing, improving UX and avoiding rate limits.                                   | Wait nodes configured with 2s delay between messages; text message delay scales with message length. |
| For detailed setup and troubleshooting, refer to Evolution API webhook docs and n8n community forums for Redis/PostgreSQL nodes. | Evolution API docs: https://docs.evolutionapi.com/                                                 |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. It strictly follows applicable content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.