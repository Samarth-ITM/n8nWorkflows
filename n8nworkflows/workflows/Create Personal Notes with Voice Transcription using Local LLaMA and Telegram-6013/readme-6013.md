Create Personal Notes with Voice Transcription using Local LLaMA and Telegram

https://n8nworkflows.xyz/workflows/create-personal-notes-with-voice-transcription-using-local-llama-and-telegram-6013


# Create Personal Notes with Voice Transcription using Local LLaMA and Telegram

### 1. Workflow Overview

This workflow enables creating personal notes from Telegram messages, supporting both text and voice inputs. Its main purpose is to allow a user to send ideas or notes via Telegram either as text or voice messages, transcribe voice messages locally using a Whisper ASR API, summarize the content with a local LLaMA AI model, and send back the summarized bullet points to the user through Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Listens to incoming Telegram messages, downloads voice files if present, and filters messages by authorized user.
- **1.2 Message Type Detection:** Distinguishes whether the incoming message is a text or voice message.
- **1.3 Voice Transcription:** If voice message, downloads audio and sends it to a local Whisper ASR HTTP service for transcription.
- **1.4 AI Processing:** Uses a local LLaMA model via Ollama API to summarize the text or transcribed audio into concise bullet points.
- **1.5 Output Delivery:** Sends the AI-generated summary back to the user on Telegram.
- **1.6 Access Control:** Verifies if the Telegram user is authorized and responds with an error message if not.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for new Telegram messages, downloads any attached voice files, and initiates user authorization check.

- **Nodes Involved:**  
  - Telegram Trigger  
  - If  
  - Sticky Note (for annotation)

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Trigger node for Telegram updates (message type)  
    - Config: Listens for "message" updates with file download enabled  
    - Credentials: Uses Telegram API credentials "TakeMyNotes"  
    - Input: N/A (trigger)  
    - Output: Emits incoming Telegram messages including file data if available  
    - Edge Cases: Failures if Telegram API credentials invalid or network issues; large voice files may cause timeout.

  - **If**  
    - Type: Conditional node  
    - Config: Checks if the message sender's Telegram user ID equals `1460980649` (authorized user)  
    - Input: Output from Telegram Trigger  
    - Output: Passes authorized user messages to next block; unauthorized users routed to error response  
    - Edge Cases: If user ID is missing or malformed, defaults to unauthorized path.

#### 2.2 Message Type Detection

- **Overview:**  
  Determines if the incoming Telegram message contains text or voice content to route accordingly.

- **Nodes Involved:**  
  - Switch  
  - Sticky Note

- **Node Details:**

  - **Switch**  
    - Type: Switch for routing based on message content  
    - Config:  
      - Checks if `$json.message.text` exists (text message) → outputs on "text" branch  
      - Checks if `$json.message.voice.file_id` exists (voice message) → outputs on "voice" branch  
    - Input: From If node (authorized messages)  
    - Output: Routes to text processing or voice processing  
    - Edge Cases: Messages with neither text nor voice are not explicitly handled; could cause workflow to stall or error.

#### 2.3 Voice Transcription

- **Overview:**  
  Downloads the voice file from Telegram, sends it to a local Whisper ASR service for transcription, and obtains text.

- **Nodes Involved:**  
  - Get Voice File  
  - HTTP Request  
  - Sticky Note

- **Node Details:**

  - **Get Voice File**  
    - Type: Telegram API node to fetch file by ID  
    - Config: Uses `$json.message.voice.file_id` to download file data  
    - Credentials: Telegram API "TakeMyNotes"  
    - Input: Voice branch from Switch node  
    - Output: Binary audio file data for transcription  
    - Edge Cases: Failures if file ID invalid, file missing, or Telegram API issues.

  - **HTTP Request**  
    - Type: HTTP POST request  
    - Config:  
      - URL: `http://localhost:9000/asr` (local Whisper ASR API)  
      - Content-Type: multipart-form-data  
      - Body: Sends audio file as form binary data `"audio_file"`  
    - Input: Binary audio from Get Voice File  
    - Output: JSON transcription result in `.data` field  
    - Edge Cases: Local ASR service down, network failures, or invalid audio format cause errors.

#### 2.4 AI Processing

- **Overview:**  
  Summarizes the text (from text message or transcribed voice) into bullet points using a local LLaMA 3.2 1b model via Ollama API.

- **Nodes Involved:**  
  - Basic LLM Chain (text path)  
  - Ollama Model (text path)  
  - Basic LLM Chain1 (voice transcription path)  
  - Ollama Model1 (voice transcription path)  
  - Sticky Notes

- **Node Details:**

  - **Basic LLM Chain**  
    - Type: LangChain LLM chain node  
    - Config: Prompt instructs model to summarize input text into bullet points with no introduction or explanation  
    - Input: Text message content (`$json.message.text`) passed directly  
    - Output: Summarized bullet points in `.text`  
    - Dependencies: Uses Ollama Model as language model backend  
    - Edge Cases: Empty or malformed text input may cause incomplete summaries.

  - **Ollama Model**  
    - Type: LangChain model node connecting to Ollama API  
    - Config: Uses model `"llama3.2:1b"` with no extra options  
    - Credentials: Ollama API credentials  
    - Input: Connected as AI language model for Basic LLM Chain  
    - Output: Text completion from LLaMA model  
    - Edge Cases: Ollama API downtime, auth failure, or model errors.

  - **Basic LLM Chain1**  
    - Type and config identical to Basic LLM Chain, but input is transcription data (`$json.data`) from ASR service  
    - Connected to Ollama Model1.

  - **Ollama Model1**  
    - Same as Ollama Model, linked to Basic LLM Chain1.

#### 2.5 Output Delivery

- **Overview:**  
  Sends the summarized bullet points back to the user on Telegram.

- **Nodes Involved:**  
  - Send a text message1  
  - Sticky Note

- **Node Details:**

  - **Send a text message1**  
    - Type: Telegram API node to send messages  
    - Config:  
      - Text: Uses summarized text from LLM chain output  
      - Chat ID: Set statically as `"telegramChatId"` (likely a placeholder, should be dynamic)  
      - Additional: No attribution appended  
    - Credentials: Telegram API "TakeMyNotes"  
    - Input: Outputs from both Basic LLM Chain and Basic LLM Chain1  
    - Edge Cases: If chat ID is not dynamically set, message may fail or go to wrong chat; Telegram API errors or rate limits possible.

#### 2.6 Access Control

- **Overview:**  
  Handles unauthorized user messages by sending an access denial message.

- **Nodes Involved:**  
  - Send a text message  
  - Sticky Note

- **Node Details:**

  - **Send a text message**  
    - Type: Telegram API node  
    - Config:  
      - Text: "I am sorry, you have no access to my services."  
      - Chat ID: Uses `$json.message.from.id` dynamically to respond to sender  
    - Credentials: Telegram API "TakeMyNotes"  
    - Input: From If node unauthorized branch  
    - Edge Cases: Telegram API errors, or if user ID missing, message won't send.

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                         | Input Node(s)        | Output Node(s)           | Sticky Note                                                                                 |
|---------------------|----------------------------------|---------------------------------------|----------------------|--------------------------|---------------------------------------------------------------------------------------------|
| Telegram Trigger     | Telegram Trigger                 | Start automation on Telegram messages | None (trigger)       | If                       | ## Telegram Trigger<br>Start automation when receiving a Telegram message                   |
| If                  | If                              | Check user authorization               | Telegram Trigger     | Switch, Send a text message | ## If<br>Detect if message comes from admited user                                         |
| Switch              | Switch                          | Determine if message is text or voice | If                   | Basic LLM Chain, Get Voice File | ## Switch<br>Check if message is a text or audio message                                   |
| Get Voice File       | Telegram                        | Download voice message audio file      | Switch (voice branch) | HTTP Request              |                                                                                             |
| HTTP Request        | HTTP Request                    | Send voice audio to local Whisper ASR | Get Voice File        | Basic LLM Chain1          | ## Transcription<br>Transcribe audio locally with Whisper API                              |
| Basic LLM Chain      | LangChain LLM Chain             | Summarize text messages                | Switch (text branch)  | Send a text message1      | ## AI Agent<br>Generate answer from AI Agent                                               |
| Ollama Model         | LangChain Ollama Model          | Local LLaMA model for text summary    | Basic LLM Chain       | Basic LLM Chain           |                                                                                             |
| Basic LLM Chain1     | LangChain LLM Chain             | Summarize transcribed voice text      | HTTP Request          | Send a text message1      |                                                                                             |
| Ollama Model1        | LangChain Ollama Model          | Local LLaMA model for voice summary   | Basic LLM Chain1      | Basic LLM Chain1          |                                                                                             |
| Send a text message1 | Telegram                       | Send summarized note back to user     | Basic LLM Chain, Basic LLM Chain1 | None                     | ## Telegram <br>Send response with elaborated note to the user                             |
| Send a text message  | Telegram                       | Send access denied message             | If (unauthorized)     | None                     |                                                                                             |
| Sticky Note          | Sticky Note                    | Annotation                           | None                 | None                     | ## Telegram Trigger<br>Start automation when receiving a Telegram message                   |
| Sticky Note1         | Sticky Note                    | Annotation                           | None                 | None                     | ## If<br>Detect if message comes from admited user                                         |
| Sticky Note2         | Sticky Note                    | Annotation                           | None                 | None                     | ## Switch<br>Check if message is a text or audio message                                   |
| Sticky Note3         | Sticky Note                    | Annotation                           | None                 | None                     | ## Transcription<br>Transcribe audio locally with Whisper API                              |
| Sticky Note4         | Sticky Note                    | Annotation                           | None                 | None                     | ## AI Agent<br>Generate answer from AI Agent                                               |
| Sticky Note5         | Sticky Note                    | Annotation                           | None                 | None                     | ## Telegram <br>Send response with elaborated note to the user                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Parameters:  
     - Updates: select "message"  
     - Download: enabled (to download files automatically)  
   - Credentials: Attach Telegram API credentials ("TakeMyNotes")  
   - Position: (0, 280)

2. **Create If node**  
   - Type: If  
   - Parameters:  
     - Condition: Check if `{{$json["message"]["from"]["id"]}}` equals `1460980649` (replace with your authorized Telegram user ID)  
   - Connect Telegram Trigger output to If input  
   - Position: (220, 280)

3. **Create Send a text message node (for unauthorized users)**  
   - Type: Telegram  
   - Parameters:  
     - Text: "I am sorry, you have no access to my services."  
     - Chat ID: `={{$json["message"]["from"]["id"]}}`  
   - Credentials: Telegram API "TakeMyNotes"  
   - Connect If "false" output to this node  
   - Position: (440, 380)

4. **Create Switch node**  
   - Type: Switch  
   - Parameters:  
     - Rule 1 ("text"): Check if `{{$json["message"]["text"]}}` exists (string operation "exists")  
     - Rule 2 ("voice"): Check if `{{$json["message"]["voice"]["file_id"]}}` exists  
   - Connect If "true" output to Switch input  
   - Position: (440, 180)

5. **For Text Message Path:**  
   - Create Ollama Model node  
     - Type: LangChain Ollama Model  
     - Model: `"llama3.2:1b"`  
     - Credentials: Connect Ollama API credentials  
     - Position: (1188, 100)
   
   - Create Basic LLM Chain node  
     - Type: LangChain LLM Chain  
     - Parameters:  
       - Text prompt:  
         ```
         You are my personal assistant that helps me note my ideas. Summarize in bullet points this text to be saved in my notes and do not invent anything (give me no introduction or explanation, just the bullet points with the summary): 
         '{{ $json.message.text }}'.
         ```  
       - Prompt type: define  
     - Connect Ollama Model as AI language model to this node  
     - Connect Switch "text" output to Basic LLM Chain input  
     - Position: (1100, -120)

6. **For Voice Message Path:**  
   - Create Get Voice File node  
     - Type: Telegram (Get file)  
     - Parameters:  
       - File ID: `={{ $json.message.voice.file_id }}`  
       - Resource: file  
     - Credentials: Telegram API "TakeMyNotes"  
     - Connect Switch "voice" output to this node  
     - Position: (660, 380)
   
   - Create HTTP Request node  
     - Type: HTTP Request  
     - Parameters:  
       - URL: `http://localhost:9000/asr` (local Whisper ASR API endpoint)  
       - Method: POST  
       - Content-Type: multipart/form-data  
       - Body parameters:  
         - Name: "audio_file"  
         - Type: formBinaryData  
         - Input data field: "data" (binary from Get Voice File)  
     - Connect Get Voice File output to HTTP Request input  
     - Position: (880, 380)
   
   - Create Ollama Model1 node  
     - Same config as Ollama Model node (model llama3.2:1b)  
     - Credentials: Ollama API  
     - Position: (1188, 600)
   
   - Create Basic LLM Chain1 node  
     - Same config as Basic LLM Chain, but prompt text:  
       ```
       You are my personal assistant that helps me note my ideas. Summarize in bullet points this text to be saved in my notes and do not invent anything (give me no introduction or explanation, just the bullet points with the summary): 
       {{ $json.data }}
       ```  
     - Connect HTTP Request output (transcription) to Basic LLM Chain1 input  
     - Connect Ollama Model1 as AI language model to Basic LLM Chain1  
     - Position: (1100, 380)

7. **Create Send a text message1 node**  
   - Type: Telegram  
   - Parameters:  
     - Text: `={{ $json.text }}` (output from Basic LLM Chain or Basic LLM Chain1)  
     - Chat ID: Use dynamic chat ID from incoming message (should replace `"telegramChatId"` with dynamic expression: `{{$json.message.chat.id}}` or equivalent)  
     - Additional: Disable append attribution  
   - Credentials: Telegram API "TakeMyNotes"  
   - Connect Basic LLM Chain and Basic LLM Chain1 outputs to this node  
   - Position: (1476, 180)

8. **Add Sticky Notes for clarity (optional but recommended):**  
   - Add notes describing each block: Telegram Trigger, If, Switch, Transcription, AI Agent, Telegram Send.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                                                                                      |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| The workflow uses a local Whisper ASR API running on `http://localhost:9000/asr` to transcribe voice files.| Requires local installation of Whisper ASR service exposing a multipart-form-data POST endpoint accepting audio files under `"audio_file"` parameter.              |
| The LLaMA model is accessed via Ollama API with model identifier `"llama3.2:1b"`.                         | Requires Ollama API credentials and an operational local or remote Ollama server running LLaMA 3.2 1b model.                                                       |
| Telegram API credentials must have access to the bot and be configured properly in n8n credentials.       | Bot must have permissions to read messages and send messages to users.                                                                                            |
| The chat ID in the final "Send a text message1" node uses a static string `"telegramChatId"`. Replace with dynamic chat ID expression for proper delivery. | Use: `={{ $json.message.chat.id }}` or `={{ $json.message.from.id }}` depending on Telegram message structure.                                                    |
| User authorization is hardcoded to Telegram user ID `1460980649`. Modify to your own Telegram user ID(s). | Find your Telegram user ID using bots like @userinfobot or similar.                                                                                               |
| Workflow is designed for a single authorized user but could be extended with a list of allowed IDs.       | Consider using a Set node or a database lookup for multiple authorized users.                                                                                      |

---

This documentation fully describes the "Create Personal Notes with Voice Transcription using Local LLaMA and Telegram" workflow, enabling comprehensive understanding, reproduction, and modification.