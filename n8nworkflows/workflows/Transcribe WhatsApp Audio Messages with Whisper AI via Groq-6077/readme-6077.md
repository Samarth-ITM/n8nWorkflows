Transcribe WhatsApp Audio Messages with Whisper AI via Groq

https://n8nworkflows.xyz/workflows/transcribe-whatsapp-audio-messages-with-whisper-ai-via-groq-6077


# Transcribe WhatsApp Audio Messages with Whisper AI via Groq

### 1. Workflow Overview

This workflow automates the transcription of WhatsApp audio messages using Whisper AI via Groq's API. It listens for incoming WhatsApp messages via a webhook, filters audio messages, converts the base64 audio data into a binary file, sends it to the Groq Whisper transcription service, and then replies back to the original WhatsApp chat with the transcribed text using the Evolution API.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Receives WhatsApp message payloads through a webhook.
- **1.2 Data Preparation:** Extracts and structures relevant message information.
- **1.3 Message Type Filtering:** Determines if the incoming message is an audio message.
- **1.4 Audio Conversion:** Converts base64 audio data into a binary audio file for processing.
- **1.5 Transcription Request:** Sends the audio file to Groq Whisper AI for transcription.
- **1.6 Sending Transcription:** Sends the transcribed text back to the WhatsApp sender as a reply.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives incoming WhatsApp messages through an HTTP POST webhook, serving as the entry point for the workflow.

- **Nodes Involved:**  
  - Webhook1

- **Node Details:**

  - **Webhook1**  
    - *Type & Role:* Webhook node, listens for HTTP POST requests from WhatsApp message upstream systems.  
    - *Configuration:*  
      - Path: `/seucaminho/MESSAGES_UPSERT` (customizable endpoint path)  
      - HTTP Method: POST  
    - *Key Expressions:* None (raw payload captured)  
    - *Input & Output:* No input; output is the HTTP request JSON body.  
    - *Version Requirements:* v2  
    - *Edge Cases:* Incorrect webhook URL or method will prevent triggering; malformed payloads may cause downstream errors.  
    - *Sub-workflow:* None

#### 1.2 Data Preparation

- **Overview:**  
  Extracts key fields from the incoming JSON payload to simplify further processing, including sender number, sender name, message text, message type, audio content (base64), and message ownership.

- **Nodes Involved:**  
  - Edit Fields1

- **Node Details:**

  - **Edit Fields1**  
    - *Type & Role:* Set node, used to create simplified, normalized fields for downstream nodes.  
    - *Configuration:*  
      - Assignments:  
        - `numero`: Extracted sender ID before '@' from `remoteJid`  
        - `nome`: Sender's push name  
        - `mensagem`: Text content from conversation message  
        - `tipo_msg`: Message type string  
        - `audio`: Base64 audio content from message  
        - `body.data.key.fromMe`: Boolean indicating if message is sent by the bot/user  
    - *Key Expressions:*  
      - Uses JavaScript expressions to navigate nested JSON and extract/transform data.  
    - *Input & Output:* Input from Webhook1; output to Switch1.  
    - *Version Requirements:* v3.4  
    - *Edge Cases:* Missing or unexpected JSON structure can lead to undefined fields; audio field empty if message is not audio.  
    - *Sub-workflow:* None

#### 1.3 Message Type Filtering

- **Overview:**  
  Filters messages to continue processing only if the message type is `audioMessage`.

- **Nodes Involved:**  
  - Switch1

- **Node Details:**

  - **Switch1**  
    - *Type & Role:* Switch node, conditional routing based on message type.  
    - *Configuration:*  
      - Rule: Checks if `tipo_msg` equals `"audioMessage"`  
    - *Input & Output:* Input from Edit Fields1; output to Convert to File1 only if condition met.  
    - *Version Requirements:* v3.2  
    - *Edge Cases:* Non-audio messages are ignored (no output on main connection), so no transcription occurs.  
    - *Sub-workflow:* None

#### 1.4 Audio Conversion

- **Overview:**  
  Converts the base64-encoded audio string into a binary audio file with a defined filename and MIME type suitable for API upload.

- **Nodes Involved:**  
  - Convert to File1

- **Node Details:**

  - **Convert to File1**  
    - *Type & Role:* ConvertToFile node, converts string to binary file.  
    - *Configuration:*  
      - Source Property: `audio` (base64 string)  
      - File Name: `audio.mp3`  
      - MIME Type: `audio/mpeg`  
    - *Input & Output:* Input from Switch1; output to HTTP Request1.  
    - *Version Requirements:* v1.1  
    - *Edge Cases:* Corrupt or non-base64 data may cause conversion failure; wrong MIME type may affect transcription.  
    - *Sub-workflow:* None

#### 1.5 Transcription Request

- **Overview:**  
  Sends the binary audio file to the Groq Whisper AI transcription API with specific parameters, and receives the transcription text in JSON format.

- **Nodes Involved:**  
  - HTTP Request1

- **Node Details:**

  - **HTTP Request1**  
    - *Type & Role:* HTTP Request node to interact with external transcription API.  
    - *Configuration:*  
      - URL: `https://api.groq.com/openai/v1/audio/transcriptions`  
      - Method: POST  
      - Content Type: multipart/form-data  
      - Body Parameters:  
        - `file`: binary data field named `data` (from Convert to File1)  
        - `model`: `"whisper-large-v3"`  
        - `response_format`: `"json"`  
        - `temperature`: `"0"` (deterministic output)  
        - `language`: `"pt"` (Portuguese)  
      - Headers: Authorization Bearer token from environment variable `GROQ_API_KEY`  
    - *Key Expressions:*  
      - Authorization header uses environment variable interpolation  
    - *Input & Output:* Input from Convert to File1; output JSON with transcription text to Evolution API.  
    - *Version Requirements:* v4.2  
    - *Edge Cases:* API key missing/invalid causes auth failure; malformed audio file causes transcription errors; network timeouts possible.  
    - *Sub-workflow:* None

#### 1.6 Sending Transcription

- **Overview:**  
  Sends the transcribed text as a message reply to the original WhatsApp sender using the Evolution API node.

- **Nodes Involved:**  
  - Evolution API

- **Node Details:**

  - **Evolution API**  
    - *Type & Role:* Custom n8n node for interacting with Evolution's WhatsApp messaging API.  
    - *Configuration:*  
      - Resource: `messages-api`  
      - `remoteJid`: Pulled dynamically from webhook message (`remoteJid`)  
      - `messageText`: Static prefix `"*Mensagem transcrita automaticamente.*\n"` plus the transcribed text from HTTP Request1 (`text` field)  
      - `instanceName`: Taken from incoming webhook data (`body.instance`)  
      - `options_message`: Specifies quoting the original message by ID for reply threading  
    - *Input & Output:* Input from HTTP Request1; no outgoing connections (endpoint).  
    - *Version Requirements:* v1  
    - *Edge Cases:* Evolution API authentication issues, message send failures, invalid quoted message ID causing failure.  
    - *Sub-workflow:* None  
    - *Note:* This node is currently inactive in the workflow (disabled).

---

### 3. Summary Table

| Node Name       | Node Type                  | Functional Role                      | Input Node(s)   | Output Node(s)   | Sticky Note                                                                                              |
|-----------------|----------------------------|------------------------------------|-----------------|------------------|----------------------------------------------------------------------------------------------------------|
| Webhook1        | Webhook                    | Receive incoming WhatsApp messages | None            | Edit Fields1     |                                                                                                          |
| Edit Fields1    | Set                        | Extract and normalize message data | Webhook1        | Switch1          |                                                                                                          |
| Switch1         | Switch                     | Filter for audio messages           | Edit Fields1    | Convert to File1 |                                                                                                          |
| Convert to File1| ConvertToFile              | Convert base64 audio to binary file | Switch1         | HTTP Request1    |                                                                                                          |
| HTTP Request1   | HTTP Request               | Send audio file to Groq Whisper AI | Convert to File1| Evolution API    |                                                                                                          |
| Evolution API   | Evolution API (custom)     | Send transcription back to WhatsApp| HTTP Request1   | None             | Node is currently inactive (disabled) in the workflow.                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: `Webhook1`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `/seucaminho/MESSAGES_UPSERT` (replace with your preferred endpoint)  
   - No authentication required unless your environment demands it  

2. **Add Set Node for Data Preparation:**  
   - Name: `Edit Fields1`  
   - Type: Set  
   - Assign these fields using expressions:  
     - `numero`: `={{ $json.body.data.key.remoteJid.split("@").shift() }}`  
     - `nome`: `={{ $json.body.data.pushName }}`  
     - `mensagem`: `={{ $json.body.data.message.conversation }}`  
     - `tipo_msg`: `={{ $json.body.data.messageType }}`  
     - `audio`: `={{ $json.body.data.message.base64 }}`  
     - `body.data.key.fromMe`: `={{ $json.body.data.key.fromMe }}`  
   - Connect Webhook1 output to Edit Fields1 input  

3. **Add Switch Node for Message Type Filtering:**  
   - Name: `Switch1`  
   - Type: Switch  
   - Add rule:  
     - Condition: `tipo_msg` equals `"audioMessage"`  
   - Connect Edit Fields1 output to Switch1 input  

4. **Add ConvertToFile Node:**  
   - Name: `Convert to File1`  
   - Type: ConvertToFile  
   - Parameters:  
     - Source Property: `audio` (the field set in Edit Fields1)  
     - File Name: `audio.mp3`  
     - MIME Type: `audio/mpeg`  
   - Connect Switch1 audioMessage output to Convert to File1 input  

5. **Add HTTP Request Node for Transcription:**  
   - Name: `HTTP Request1`  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://api.groq.com/openai/v1/audio/transcriptions`  
     - HTTP Method: POST  
     - Content Type: `multipart/form-data`  
     - Body Parameters:  
       - `file`: use binary data field named `data` (from Convert to File1)  
       - `model`: `whisper-large-v3`  
       - `response_format`: `json`  
       - `temperature`: `0`  
       - `language`: `pt` (Portuguese)  
     - Headers: Add `Authorization` header with value `Bearer {{ $env.GROQ_API_KEY }}`  
   - Connect Convert to File1 output to HTTP Request1 input  

6. **Add Evolution API Node to Send Reply (Optional / Disabled):**  
   - Name: `Evolution API`  
   - Type: Evolution API node (custom integration)  
   - Configure with:  
     - Resource: `messages-api`  
     - `remoteJid`: `={{ $('Webhook1').item.json.body.data.key.remoteJid }}`  
     - `messageText`: `=*Mensagem transcrita automaticamente.*\n{{ $('HTTP Request1').item.json.text }}`  
     - `instanceName`: `={{ $('Webhook1').item.json.body.instance }}`  
     - Options: quoted message with `messageId` from webhook data  
   - Connect HTTP Request1 output to Evolution API input  
   - Note: This node is currently inactive, activate when ready and ensure API credentials are set  

7. **Credential Setup:**  
   - Set environment variable `GROQ_API_KEY` with your Groq API key for authentication  
   - Configure Evolution API credentials as needed if enabling the sending node  

8. **Final Connections:**  
   - Webhook1 → Edit Fields1 → Switch1  
   - Switch1 (audioMessage) → Convert to File1 → HTTP Request1 → Evolution API (optional)  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The Evolution API node is disabled; enable it only after configuring credentials and testing the Groq API.    | Ensures no accidental message sending before setup is complete.                                |
| Groq API uses OpenAI-compatible endpoints for Whisper transcription; see https://groq.com/openai for details. | Official Groq API documentation.                                                               |
| Webhook path `/seucaminho/MESSAGES_UPSERT` should be replaced with your actual endpoint and secured properly. | Security best practices for webhook exposure.                                                   |
| The transcribed message is sent with a prefix "*Mensagem transcrita automaticamente.*" indicating auto reply.  | Helps distinguish bot replies from user messages.                                              |

---

This document fully describes the workflow's architecture, node configurations, and reproduction steps, enabling developers or AI agents to understand, recreate, and enhance the WhatsApp audio transcription automation.