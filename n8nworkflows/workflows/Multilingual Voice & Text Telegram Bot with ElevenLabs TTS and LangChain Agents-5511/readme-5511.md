Multilingual Voice & Text Telegram Bot with ElevenLabs TTS and LangChain Agents

https://n8nworkflows.xyz/workflows/multilingual-voice---text-telegram-bot-with-elevenlabs-tts-and-langchain-agents-5511


# Multilingual Voice & Text Telegram Bot with ElevenLabs TTS and LangChain Agents

### 1. Workflow Overview

This workflow implements a **Multilingual Voice & Text Telegram Bot** integrating ElevenLabs Text-to-Speech (TTS) and Speech-to-Text (STT) services, combined with LangChain AI agents for intelligent conversational responses. It supports receiving Telegram messages either as voice or text, processes them accordingly, and replies with either synthesized voice messages or text responses, preserving the user's language preference automatically.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception & Session Initialization:** Receives Telegram updates, extracts chat session ID, and routes messages based on content type (voice vs text).
- **1.2 Voice Message Processing:** Downloads the voice file, transcribes it to text using ElevenLabs STT, flags the message as voice-originated, and prepares text for AI processing.
- **1.3 Text Message Processing:** Directly routes text messages for AI processing without transcription.
- **1.4 AI Processing with LangChain Agents:** Uses LangChain agents with integrated Google Gemini and Groq chat models to generate responses, maintaining session memory.
- **1.5 Output Handling & Response Delivery:** Depending on whether the input was voice or text, sends back a synthesized voice message via ElevenLabs TTS or sends a plain text message via Telegram.
- **1.6 Auxiliary & Documentation Nodes:** Sticky notes explain voice/text routing logic, auto-language detection, and instructions for extending the system with custom tools.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Session Initialization

- **Overview:**  
  This block triggers on incoming Telegram messages, assigns a session ID based on the Telegram chat ID, and routes messages into voice or text processing paths.

- **Nodes Involved:**  
  - Telegram Input  
  - Adds SessionId  
  - Switch

- **Node Details:**

  - **Telegram Input**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry point for Telegram messages using webhook updates limited to "message" events.  
    - *Config:* Listens for new messages; no additional filters configured.  
    - *Connections:* Output → Adds SessionId  
    - *Failures:* Possible webhook misconfiguration or Telegram API failures.

  - **Adds SessionId**  
    - *Type:* Set  
    - *Role:* Adds a `sessionId` field equal to `message.chat.id` to each incoming message for session management.  
    - *Config:* Copies all other fields; assigns `sessionId` as string from Telegram chat ID.  
    - *Connections:* Output → Switch  
    - *Failures:* Expression failure if input JSON missing expected fields.

  - **Switch**  
    - *Type:* Switch  
    - *Role:* Routes messages depending on presence or absence of `message.text` (empty or not).  
    - *Config:* Two outputs — "voice" if `message.text` is empty (voice message), "text" if not empty (text message).  
    - *Connections:*  
      - "voice" → Get Voice File  
      - "text" → Aggregate  
    - *Failures:* Edge cases if message text field is missing or malformed.

---

#### 2.2 Voice Message Processing

- **Overview:**  
  Downloads the voice message file from Telegram, transcribes it into text using ElevenLabs STT, and marks the message as originating from voice input.

- **Nodes Involved:**  
  - Get Voice File  
  - Transcribe ElevenLabs  
  - Edit Fields  
  - Aggregate

- **Node Details:**

  - **Get Voice File**  
    - *Type:* Telegram (Get File)  
    - *Role:* Downloads the voice file using the Telegram `voice.file_id`.  
    - *Config:* Uses dynamic expression for `fileId` from incoming message voice object.  
    - *Connections:* Output → Transcribe ElevenLabs  
    - *Failures:* File ID may be invalid or file download may fail due to network issues.

  - **Transcribe ElevenLabs**  
    - *Type:* ElevenLabs (Speech to Text)  
    - *Role:* Transcribes audio to text using ElevenLabs speech recognition.  
    - *Config:* Speech resource, speechToText operation, default request options.  
    - *Connections:* Output → Edit Fields  
    - *Failures:* API auth errors, transcription latency, unsupported audio format.

  - **Edit Fields**  
    - *Type:* Set  
    - *Role:* Sets `message.text` to the transcribed text, reassigns `sessionId` from Switch node, and sets `voice=true` flag.  
    - *Config:* Assigns fields with expressions referencing prior nodes.  
    - *Connections:* Output → Aggregate  
    - *Failures:* Expression errors if transcription output missing.

  - **Aggregate**  
    - *Type:* Aggregate  
    - *Role:* Aggregates all item data to unify the message representation for downstream nodes.  
    - *Config:* Aggregate all item data into one composite item.  
    - *Connections:* Output → Voice Assistant  
    - *Failures:* Failure if input data is empty.

---

#### 2.3 Text Message Processing

- **Overview:**  
  Directly aggregates the incoming text message for AI processing, bypassing transcription and voice-specific steps.

- **Nodes Involved:**  
  - Aggregate

- **Node Details:**

  - **Aggregate**  
    - *Type:* Aggregate  
    - *Role:* Aggregates incoming text messages for unified AI input structure.  
    - *Config:* Same as in voice path.  
    - *Connections:* Output → Voice Assistant  
    - *Failures:* Same as above.

---

#### 2.4 AI Processing with LangChain Agents

- **Overview:**  
  This block uses LangChain agents powered by Google Gemini and Groq chat models to process input text and generate responses. It maintains a memory buffer keyed by session ID to retain conversational context.

- **Nodes Involved:**  
  - Voice Assistant Memory  
  - Google Gemini Chat Model  
  - Groq Chat Model  
  - Voice Assistant

- **Node Details:**

  - **Voice Assistant Memory**  
    - *Type:* LangChain Memory Buffer Window  
    - *Role:* Stores conversational memory using a windowed buffer keyed by the `sessionId` from data.  
    - *Config:* Custom session key from aggregated data item field `sessionId`.  
    - *Connections:* Memory input → Voice Assistant  
    - *Failures:* Memory persistence failure or mis-keying causing context loss.

  - **Google Gemini Chat Model**  
    - *Type:* LangChain Language Model (Google Gemini)  
    - *Role:* Provides a chat language model for response generation.  
    - *Config:* Model name set to `models/gemini-2.5-flash`.  
    - *Connections:* AI Language Model input → Voice Assistant  
    - *Failures:* API quota, auth, or latency issues.

  - **Groq Chat Model**  
    - *Type:* LangChain Language Model (Groq)  
    - *Role:* Alternative chat model (`deepseek-r1-distill-llama-70b`) integrated for multi-model AI capabilities.  
    - *Config:* Default options.  
    - *Connections:* AI Language Model input → Voice Assistant  
    - *Failures:* Similar API or model availability risks.

  - **Voice Assistant**  
    - *Type:* LangChain Agent  
    - *Role:* Orchestrates AI chat response using inputs from memory and language models. Uses a custom system message defining its role as a multi-source crypto intelligence analyst with integrated agents.  
    - *Config:* Prompt type "define", text input from aggregated data message text, system message specifying AI persona and tools.  
    - *Connections:* Output → If voice  
    - *Failures:* Expression errors, prompt misconfiguration, API errors.

---

#### 2.5 Output Handling & Response Delivery

- **Overview:**  
  This block sends back the AI-generated response either as a voice message (via ElevenLabs TTS and Telegram voice message send) or as a simple Telegram text message.

- **Nodes Involved:**  
  - If voice  
  - ElevenLabs (TTS)  
  - Telegram send voice message  
  - Telegram Send Message

- **Node Details:**

  - **If voice**  
    - *Type:* If  
    - *Role:* Checks if the input was a voice message by testing the boolean `voice` flag in aggregated data.  
    - *Config:* Condition: `voice == true`.  
    - *Connections:*  
      - True → ElevenLabs (TTS)  
      - False → Telegram Send Message  
    - *Failures:* Possible expression evaluation errors.

  - **ElevenLabs (TTS)**  
    - *Type:* ElevenLabs (Text to Speech)  
    - *Role:* Converts AI text output into speech audio.  
    - *Config:*  
      - Text from Voice Assistant output.  
      - Voice set to “Aria” (voice ID `9BWtsMINqrJLrRacOk9x`).  
      - Model: `eleven_multilingual_v2`.  
      - Language code auto-detected from Telegram input sender language code.  
      - Output format: Opus 48kHz 64kbps.  
    - *Connections:* Output → Telegram send voice message  
    - *Failures:* TTS API errors, language code mismatches.

  - **Telegram send voice message**  
    - *Type:* HTTP Request  
    - *Role:* Sends the synthesized voice audio back to Telegram using Telegram Bot API `sendVoice` method.  
    - *Config:*  
      - POST multipart/form-data with fields: `chat_id` from Telegram Input, `voice` as binary data from ElevenLabs TTS.  
      - Authorization via Telegram bot token credential.  
    - *Connections:* None (terminal node)  
    - *Failures:* Telegram API errors, network problems, incorrect token.

  - **Telegram Send Message**  
    - *Type:* Telegram  
    - *Role:* Sends text message reply back to Telegram chat.  
    - *Config:* Text from Voice Assistant output, chat ID from Telegram input.  
    - *Connections:* None (terminal node)  
    - *Failures:* Telegram API errors, token issues.

---

#### 2.6 Auxiliary & Documentation Nodes

- **Overview:**  
  Sticky notes provide important explanations and instructions for users and maintainers regarding language detection, routing logic, and custom tool integration.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2

- **Node Details:**

  - **Sticky Note**  
    - *Content:* Explains auto-language detection using Telegram `message.from.language_code`, recommends ElevenLabs TTS model `eleven_multilingual_v2`, and how to override language code if necessary.  
    - *Position:* Near TTS nodes.

  - **Sticky Note1**  
    - *Content:* Describes the voice/text routing logic: voice messages go through voice file retrieval, STT, flagged as voice, AI processing, TTS, then voice reply; text messages bypass STT/TTS and get direct AI text responses. Specifies Switch node logic based on empty text.  
    - *Position:* Near Switch node.

  - **Sticky Note2**  
    - *Content:* Instructions on integrating custom tools by editing the `systemMessage` of the Voice Assistant node and adding Python-like tool declarations. Suggests tool examples such as database queries, API integrations, and code execution.  
    - *Position:* Near AI processing nodes.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                  | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                 |
|-------------------------|----------------------------------|-------------------------------------------------|-------------------------|--------------------------|---------------------------------------------------------------------------------------------|
| Telegram Input          | Telegram Trigger                  | Entry point for incoming Telegram messages       |                         | Adds SessionId            |                                                                                             |
| Adds SessionId          | Set                              | Adds chat-based sessionId to message             | Telegram Input           | Switch                   |                                                                                             |
| Switch                  | Switch                           | Routes messages by presence of text (voice/text) | Adds SessionId           | Get Voice File, Aggregate | See Sticky Note1: Voice/Text Routing Logic                                                  |
| Get Voice File          | Telegram (Get File)               | Downloads voice message audio file                | Switch (voice output)    | Transcribe ElevenLabs     |                                                                                             |
| Transcribe ElevenLabs   | ElevenLabs (Speech to Text)       | Converts voice audio to text                       | Get Voice File           | Edit Fields              |                                                                                             |
| Edit Fields             | Set                              | Sets transcribed text, sessionId, and voice flag | Transcribe ElevenLabs    | Aggregate                |                                                                                             |
| Aggregate               | Aggregate                        | Aggregates message data for AI processing         | Switch (text), Edit Fields | Voice Assistant           |                                                                                             |
| Voice Assistant Memory  | LangChain Memory Buffer Window   | Maintains conversational memory by session       |                         | Voice Assistant           |                                                                                             |
| Google Gemini Chat Model| LangChain Language Model         | Provides AI chat model (Google Gemini)            |                         | Voice Assistant           |                                                                                             |
| Groq Chat Model         | LangChain Language Model         | Provides AI chat model (Groq)                      |                         | Voice Assistant           |                                                                                             |
| Voice Assistant         | LangChain Agent                  | Orchestrates AI response with defined persona     | Aggregate, Memory, Models | If voice                 | See Sticky Note2: Custom Tools Integration                                                  |
| If voice                | If                               | Determines if input was voice to choose output    | Voice Assistant          | ElevenLabs, Telegram Send Message |                                                                                             |
| ElevenLabs              | ElevenLabs (Text to Speech)       | Converts AI text output to speech audio            | If voice (true)          | Telegram send voice message | See Sticky Note: Auto-language detection and TTS model settings                            |
| Telegram send voice message | HTTP Request                  | Sends synthesized voice message to Telegram       | ElevenLabs               |                          |                                                                                             |
| Telegram Send Message   | Telegram                         | Sends AI text message reply to Telegram           | If voice (false)         |                          |                                                                                             |
| Sticky Note             | Sticky Note                     | Explains auto-language detection                   |                         |                          | Auto-language detection and TTS model recommendation                                       |
| Sticky Note1            | Sticky Note                     | Explains voice/text routing logic                  |                         |                          | Voice/Text Routing Logic                                                                    |
| Sticky Note2            | Sticky Note                     | Instructions for adding custom tools               |                         |                          | Custom Tools Integration Instructions                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Input Node:**  
   - Type: Telegram Trigger  
   - Configure to listen for `message` updates.  
   - Set webhook ID appropriately or create a new webhook.  
   - Connect output to next node.

2. **Add SessionId Node (Set):**  
   - Type: Set  
   - Copy all incoming fields.  
   - Add field `sessionId` as string with value expression: `={{ $json.message.chat.id }}`.  
   - Connect output to Switch node.

3. **Create Switch Node:**  
   - Type: Switch  
   - Add two outputs:  
     - "voice" output if `message.text` is empty (`={{ $json.message.text }}` empty check).  
     - "text" output if `message.text` is not empty.  
   - Connect "voice" output to Get Voice File node.  
   - Connect "text" output to Aggregate node.

4. **Create Get Voice File Node:**  
   - Type: Telegram (Get File)  
   - Parameter `fileId`: `={{ $json.message.voice.file_id }}`.  
   - Connect output to Transcribe ElevenLabs.

5. **Create Transcribe ElevenLabs Node:**  
   - Type: ElevenLabs (Speech to Text)  
   - Resource: Speech  
   - Operation: speechToText  
   - Use default request options.  
   - Connect output to Edit Fields node.

6. **Create Edit Fields Node:**  
   - Type: Set  
   - Assign fields:  
     - `message.text` = `={{ $json.text }}` (transcription result).  
     - `sessionId` = `={{ $('Switch').item.json.sessionId }}`.  
     - `voice` = `true` (boolean).  
   - Connect output to Aggregate node.

7. **Create Aggregate Node:**  
   - Type: Aggregate  
   - Aggregate all item data.  
   - Connect output to Voice Assistant node.

8. **Create Voice Assistant Memory Node:**  
   - Type: LangChain Memory Buffer Window  
   - Set session key: `={{ $json.data[0].sessionId }}`  
   - Session ID type: customKey.  
   - Connect memory output to Voice Assistant.

9. **Create Google Gemini Chat Model Node:**  
   - Type: LangChain LM Chat Google Gemini  
   - Set model name: `models/gemini-2.5-flash`.  
   - Connect AI Language Model output to Voice Assistant.

10. **Create Groq Chat Model Node:**  
    - Type: LangChain LM Chat Groq  
    - Model: `deepseek-r1-distill-llama-70b`.  
    - Connect AI Language Model output to Voice Assistant.

11. **Create Voice Assistant Node:**  
    - Type: LangChain Agent  
    - Text input: `={{ $json.data[0].message.text }}` (aggregated message).  
    - System Message: Custom persona describing multi-source crypto intelligence analyst with integrated tools.  
    - Prompt Type: define.  
    - Connect output to If voice node.

12. **Create If voice Node:**  
    - Type: If  
    - Condition: boolean `voice` field true (`={{ $('Aggregate').item.json.data[0].voice }}`).  
    - True output → ElevenLabs TTS node.  
    - False output → Telegram Send Message node.

13. **Create ElevenLabs TTS Node:**  
    - Type: ElevenLabs (Text to Speech)  
    - Text: `={{ $('Voice Assistant').item.json.output }}`.  
    - Voice: Select "Aria" by ID `9BWtsMINqrJLrRacOk9x`.  
    - Model: `eleven_multilingual_v2`.  
    - Language Code: `={{ $('Telegram Input').item.json.message.from.language_code }}`.  
    - Output format: `opus_48000_64`.  
    - Connect output to Telegram send voice message node.

14. **Create Telegram send voice message Node:**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://api.telegram.org/bot{{$credentials.telegramToken}}/sendVoice`  
    - Content-Type: multipart/form-data  
    - Body Parameters:  
      - `chat_id`: `={{ $('Telegram Input').item.json.message.chat.id }}`  
      - `voice`: form binary data from ElevenLabs output ("data" field)  
    - Header Parameters: Content-Type multipart/form-data  
    - Requires Telegram Bot Token credential.  
    - Terminal node.

15. **Create Telegram Send Message Node:**  
    - Type: Telegram  
    - Text: `={{ $json.output }}` (from Voice Assistant output)  
    - Chat ID: `={{ $('Telegram Input').item.json.message.chat.id }}`  
    - Terminal node.

16. **Add Sticky Notes (Optional for documentation):**  
    - Add notes explaining language detection, routing logic, and custom tool integration at the respective workflow sections.

17. **Credential Setup:**  
    - Configure Telegram API credentials with bot token.  
    - Configure ElevenLabs API credentials (key).  
    - Configure LangChain AI credentials for Google Gemini and Groq models as required.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                  | Context or Link                                                                                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Auto-language detection** uses Telegram's `message.from.language_code` to select TTS voice language automatically.                                                         | Sticky Note near TTS node: Ensure ElevenLabs model `eleven_multilingual_v2` is used for best multilingual support.                                               |
| The **voice/text routing logic** is based on detecting whether the incoming Telegram message text field is empty or not.                                                     | Sticky Note1 near Switch node: Voice messages are transcribed, flagged, AI processed, then TTS/replied; text messages bypass transcription and TTS.              |
| To **add custom tools** to the AI assistant, edit the `systemMessage` in the Voice Assistant node with Python-like tool definitions describing functions and descriptions.     | Sticky Note2 near Voice Assistant node: Useful for adding capabilities like database queries, API calls, or code execution integrated into the AI assistant.       |
| ElevenLabs limits or errors may occur if audio format or language codes are incorrect; fallback or error handling should be considered if deploying in production.            | General caution for TTS/STT nodes.                                                                                                                                 |
| Telegram Bot API requires the bot token with sufficient permissions to send messages and voice messages to users.                                                            | Telegram send voice message node requires valid Telegram Bot token credential.                                                                                     |
| LangChain models require proper API access and quota; multi-model integration enables fallback or enhanced response diversity.                                               | Google Gemini and Groq Chat Model nodes require valid credentials and API access.                                                                                   |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.