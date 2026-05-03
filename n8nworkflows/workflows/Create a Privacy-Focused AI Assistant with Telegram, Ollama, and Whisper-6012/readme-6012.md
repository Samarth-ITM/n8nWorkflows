Create a Privacy-Focused AI Assistant with Telegram, Ollama, and Whisper

https://n8nworkflows.xyz/workflows/create-a-privacy-focused-ai-assistant-with-telegram--ollama--and-whisper-6012


# Create a Privacy-Focused AI Assistant with Telegram, Ollama, and Whisper

### 1. Workflow Overview

This workflow implements a privacy-focused AI assistant that interacts with users via Telegram, leveraging local transcription and AI processing to handle both text and voice messages securely. It is designed for personal use with user authorization, supports audio messages transcribed locally via Whisper ASR, and generates AI responses using the Ollama language model integrated with LangChain agent logic.

Logical blocks in the workflow:

- **1.1 Input Reception:** Captures incoming Telegram messages (text or voice) and checks user authorization.
- **1.2 Message Type Detection:** Differentiates between text and voice messages to route processing accordingly.
- **1.3 Audio Transcription:** Downloads voice messages and transcribes them locally using the Whisper ASR HTTP API.
- **1.4 AI Processing:** Uses LangChain AI Agent with Ollama language model and context memory to generate responses.
- **1.5 Response Delivery:** Sends the AI-generated response back to the Telegram user.
- **1.6 Authorization Handling:** Responds to unauthorized users with an access denial message.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
The workflow starts by listening for Telegram messages. It downloads message content for further processing and verifies if the sender is authorized.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Authorization Check If  
  - Send a text message (Unauthorized response)  
  - Sticky Note (Telegram Trigger annotation)  
  - Sticky Note1 (Authorization IF annotation)

- **Node Details:**  

  - **Telegram Trigger**  
    - Type: Trigger node for Telegram updates  
    - Configuration: Listens to "message" updates; downloads message media automatically  
    - Expressions: Accesses message data such as text and voice file IDs  
    - Input: Webhook trigger from Telegram  
    - Output: Message JSON with user and content details  
    - Edge cases: Telegram API downtime, webhook misconfiguration, missing message data  
    - Credentials: Uses "PersonalAssistant" Telegram API credentials  

  - **Authorization Check If**  
    - Type: Conditional IF node  
    - Configuration: Checks if `message.from.id` equals a predefined authorized user ID (here, null placeholder; should be configured)  
    - Expressions: `{{$json.message.from.id}}` equals authorized user ID number  
    - Input: Output of Telegram Trigger  
    - Output: Yes branch if authorized, No branch if unauthorized  
    - Edge cases: Null or missing user ID, improper authorization config  
    - Failure modes: Missing user ID in the message JSON leads to false negatives  

  - **Send a text message (Unauthorized response)**  
    - Type: Telegram node (message sender)  
    - Configuration: Sends "I am sorry, you have no access to my services." to the unauthorized user's chat ID  
    - Expressions: Uses `{{$json.message.from.id}}` for chat ID  
    - Input: No branch from Authorization Check If  
    - Output: None further  
    - Edge cases: Telegram API errors, user blocking bot, message rate limits  

  - **Sticky Notes**  
    - Provide contextual comments for the trigger and authorization logic blocks.

---

#### 2.2 Message Type Detection

- **Overview:**  
Determines whether the incoming message is text or voice, routing accordingly for AI processing or transcription.

- **Nodes Involved:**  
  - Message Type Switch  
  - Get Voice File  
  - Sticky Note4 (Switch annotation)

- **Node Details:**  

  - **Message Type Switch**  
    - Type: Switch node  
    - Configuration: Two outputs - "text" if `message.text` exists, "voice" if `message.voice.file_id` exists  
    - Expressions: Checks existence of keys within message JSON  
    - Input: Yes branch from Authorization Check If  
    - Outputs:  
      - Output 0: Text messages → AI Agent processing  
      - Output 1: Voice messages → Get Voice File node  
    - Edge cases: Messages without text or voice (e.g., images), unexpected message formats  
    - Failure modes: Expression failure if keys are missing or malformed  

  - **Get Voice File**  
    - Type: Telegram node (get file)  
    - Configuration: Downloads the voice file using `message.voice.file_id`  
    - Expressions: File ID extracted dynamically from incoming message  
    - Input: Voice output of Message Type Switch  
    - Output: Binary audio file for transcription  
    - Edge cases: File not found, Telegram API errors, network failures  
    - Credentials: Same Telegram API credentials as trigger  

---

#### 2.3 Audio Transcription

- **Overview:**  
Takes the downloaded audio file and sends it to a local Whisper ASR HTTP service for transcription, then renames the transcription key for downstream AI processing.

- **Nodes Involved:**  
  - Whisper ASR HTTP Request  
  - Rename Transcription Key  
  - Sticky Note2 (Transcription annotation)

- **Node Details:**  

  - **Whisper ASR HTTP Request**  
    - Type: HTTP Request node  
    - Configuration: POST multipart/form-data to `http://localhost:9000/asr` with audio file binary under `audio_file` parameter  
    - Input: Binary audio from Get Voice File  
    - Output: JSON with transcription result (key named `data`)  
    - Edge cases: Local server down, timeout, malformed audio, network errors  
    - Failure modes: HTTP errors, invalid response format  

  - **Rename Transcription Key**  
    - Type: Rename Keys node  
    - Configuration: Renames `data` key from Whisper response to `message.text` for compatibility with AI Agent input  
    - Input: Output from Whisper ASR HTTP Request  
    - Output: JSON with `message.text` key containing transcription  
    - Edge cases: Missing `data` key, rename failure  

---

#### 2.4 AI Processing

- **Overview:**  
Processes the textual input (either direct text messages or transcribed audio) through a LangChain AI Agent using the Ollama language model and a simple memory buffer for context.

- **Nodes Involved:**  
  - AI Agent  
  - Ollama Chat Model  
  - Simple Memory  
  - Sticky Note5 (AI Agent annotation)

- **Node Details:**  

  - **AI Agent**  
    - Type: LangChain AI Agent node  
    - Configuration: Prompt defines assistant role with input injection:  
      `"You are my personal chatbot assistant.\nMy input: '{{ $json.message.text }}'.\nYour Answer: "`  
    - Inputs:  
      - Main: Text message or transcribed text JSON (`message.text`)  
      - ai_languageModel: Ollama Chat Model  
      - ai_memory: Simple Memory  
    - Outputs: AI-generated answer text  
    - Edge cases: Missing input text, AI service errors, prompt formatting issues  
    - Version: Uses LangChain agent v2  

  - **Ollama Chat Model**  
    - Type: Ollama language model node  
    - Configuration: Uses model `"llama3.2:1b"` locally  
    - Credentials: Requires Ollama API access configured  
    - Edge cases: Model unavailable, Ollama service errors  

  - **Simple Memory**  
    - Type: LangChain memory buffer node  
    - Configuration: Session key based on `message.text` (custom key), maintains context window length of 2 messages  
    - Edge cases: Session key collisions, memory overflow, persistence issues  

---

#### 2.5 Response Delivery

- **Overview:**  
Sends the AI-generated reply back to the Telegram user as a text message.

- **Nodes Involved:**  
  - Send a text message1  
  - Sticky Note3 (Telegram send response annotation)

- **Node Details:**  

  - **Send a text message1**  
    - Type: Telegram node (message sender)  
    - Configuration: Sends the AI Agent's output (`{{$json.output}}`) back to the user; chat ID is dynamically set but empty in JSON (requires correction to use original sender ID)  
    - Input: Output of AI Agent  
    - Edge cases: Telegram API errors, missing chat ID, user blocking bot, rate limiting  
    - Credentials: Uses same Telegram API credentials  

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                         | Input Node(s)           | Output Node(s)           | Sticky Note                                         |
|-------------------------|----------------------------------|---------------------------------------|------------------------|--------------------------|----------------------------------------------------|
| Telegram Trigger        | Telegram Trigger                 | Start on Telegram message receipt     | —                      | Authorization Check If   | ## Telegram Trigger<br>Start when Telegram message is received |
| Authorization Check If  | If                              | Check if user is authorized            | Telegram Trigger       | Message Type Switch, Send a text message (unauthorized) | ## IF<br>Detect if message is sent by authorised user |
| Send a text message     | Telegram                        | Notify unauthorized users              | Authorization Check If | —                        |                                                    |
| Message Type Switch     | Switch                         | Determine if message is text or voice | Authorization Check If | AI Agent (text), Get Voice File (voice) | ## Switch<br>Detect if message is a text or audio message |
| Get Voice File          | Telegram                       | Download voice message file            | Message Type Switch    | Whisper ASR HTTP Request |                                                    |
| Whisper ASR HTTP Request| HTTP Request                   | Send audio file to local Whisper for transcription | Get Voice File         | Rename Transcription Key  | ## Transcription<br>Transcribe audio locally with Whisper API |
| Rename Transcription Key| Rename Keys                    | Rename transcription key for AI input | Whisper ASR HTTP Request| AI Agent                 |                                                    |
| AI Agent                | LangChain AI Agent             | Generate AI response from input text  | Message Type Switch (text), Rename Transcription Key (voice) | Send a text message1        | ## AI Agent <br>Formulate answer from AI Agent    |
| Ollama Chat Model       | Ollama Language Model          | Provides language model to AI Agent   | —                      | AI Agent                 |                                                    |
| Simple Memory           | LangChain Memory Buffer        | Maintain conversational context       | —                      | AI Agent                 |                                                    |
| Send a text message1    | Telegram                      | Send AI response back to user          | AI Agent               | —                        | ## Telegram<br>Send response to the user           |
| Sticky Note             | Sticky Note                   | Annotation for Telegram Trigger        | —                      | —                        | ## Telegram Trigger<br>Start when Telegram message is received |
| Sticky Note1            | Sticky Note                   | Annotation for Authorization IF        | —                      | —                        | ## IF<br>Detect if message is sent by authorised user |
| Sticky Note2            | Sticky Note                   | Annotation for Transcription block     | —                      | —                        | ## Transcription<br>Transcribe audio locally with Whisper API |
| Sticky Note3            | Sticky Note                   | Annotation for Telegram Send Response  | —                      | —                        | ## Telegram<br>Send response to the user           |
| Sticky Note4            | Sticky Note                   | Annotation for Message Type Switch     | —                      | —                        | ## Switch<br>Detect if message is a text or audio message |
| Sticky Note5            | Sticky Note                   | Annotation for AI Agent block          | —                      | —                        | ## AI Agent <br>Formulate answer from AI Agent     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates, enable media download  
   - Credentials: Add Telegram API credentials named `PersonalAssistant`  
   - Position: (0, -20)  

2. **Add "Authorization Check If" Node**  
   - Type: If  
   - Condition: Check if `{{$json.message.from.id}}` equals authorized user ID (replace `null` with actual ID)  
   - Connect Telegram Trigger output main to Authorization Check If input main  

3. **Add "Send a text message" Node (Unauthorized Response)**  
   - Type: Telegram  
   - Parameters: Text = "I am sorry, you have no access to my services."  
   - Chat ID = `{{$json.message.from.id}}`  
   - Credentials: Use same Telegram API credentials  
   - Connect Authorization Check If's No output to this node  

4. **Add "Message Type Switch" Node**  
   - Type: Switch  
   - Rules:  
     - Output "text": `{{$json.message.text}}` exists  
     - Output "voice": `{{$json.message.voice.file_id}}` exists  
   - Connect Authorization Check If's Yes output to Message Type Switch input  

5. **Add "AI Agent" Node**  
   - Type: LangChain AI Agent  
   - Parameters:  
     - Text prompt:  
       ```
       You are my personal chatbot assistant.
       My input: '{{ $json.message.text }}'.
       Your Answer:
       ```  
   - Version: Use LangChain agent v2 or latest available  
   - Position: (1320, -120)  

6. **Add "Ollama Chat Model" Node**  
   - Type: Ollama language model  
   - Parameters: Model = `llama3.2:1b`  
   - Credentials: Add and configure Ollama API credentials  
   - Connect output to AI Agent's `ai_languageModel` input  

7. **Add "Simple Memory" Node**  
   - Type: LangChain Memory Buffer Window  
   - Parameters:  
     - Session key: `{{$json.message.text}}` (or consider user ID for uniqueness)  
     - Context window length: 2  
   - Connect output to AI Agent's `ai_memory` input  

8. **Connect Message Type Switch "text" output to AI Agent main input**  

9. **Add "Get Voice File" Node**  
   - Type: Telegram  
   - Parameters: File ID = `{{$json.message.voice.file_id}}`  
   - Credentials: Use Telegram API credentials  
   - Connect Message Type Switch "voice" output to this node  

10. **Add "Whisper ASR HTTP Request" Node**  
    - Type: HTTP Request  
    - Parameters:  
      - URL: `http://localhost:9000/asr`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body Parameters: `audio_file` from binary data input field `data`  
    - Connect Get Voice File output to this node  

11. **Add "Rename Transcription Key" Node**  
    - Type: Rename Keys  
    - Parameters: Rename `data` key to `message.text`  
    - Connect Whisper ASR HTTP Request output to this node  

12. **Connect Rename Transcription Key output to AI Agent main input**  

13. **Add "Send a text message1" Node**  
    - Type: Telegram  
    - Parameters:  
      - Text: `{{$json.output}}` (AI Agent’s response)  
      - Chat ID: Use original sender ID `{{$json.message.from.id}}` (correct the empty chat ID in original workflow)  
    - Credentials: Telegram API credentials  
    - Connect AI Agent output to this node  

14. **Add Sticky Notes as annotations for each block** as per positions and content described for clarity.  

15. **Verify all credentials are correctly configured (Telegram API, Ollama API)**  

16. **Test workflow with authorized and unauthorized users, with both text and voice messages**  

---

### 5. General Notes & Resources

| Note Content                                                      | Context or Link                                               |
|------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow uses a local Whisper ASR server at http://localhost:9000/asr | Requires local Whisper ASR setup for audio transcription      |
| Ollama language model `"llama3.2:1b"` is used locally or via Ollama API | Requires Ollama API credentials and local Ollama model setup  |
| Authorization check currently compares user ID to `null` - replace with actual Telegram user ID to restrict access | User ID must be configured for security                        |
| The AI Agent uses LangChain integration for advanced chat capabilities | See n8n LangChain docs for advanced configurations             |

---

**Disclaimer:** This document is based solely on the provided n8n workflow JSON and respects all relevant content policies. It contains no illegal or protected data.