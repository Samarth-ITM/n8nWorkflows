Wikipedia Podcast Generator - AI-Powered Voice Content Creator via Telegram

https://n8nworkflows.xyz/workflows/wikipedia-podcast-generator---ai-powered-voice-content-creator-via-telegram-4496


# Wikipedia Podcast Generator - AI-Powered Voice Content Creator via Telegram

### 1. Workflow Overview

This workflow, titled **Wikipedia Podcast Generator - AI-Powered Voice Content Creator via Telegram**, is designed to create engaging, professionally structured podcast episodes based on Wikipedia topics requested by users through a Telegram bot. It supports both text and voice inputs, transcribes voice messages if necessary, researches the topic automatically via Wikipedia, generates a podcast script using an AI language model with an agent architecture, converts the script into natural speech, and finally sends the audio back to the user through Telegram.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception:** Receives incoming messages (voice or text) from Telegram users.
- **1.2 Input Type Routing:** Differentiates between text and voice messages and routes accordingly.
- **1.3 Voice Message Processing:** Downloads and transcribes voice messages to text.
- **1.4 Text Message Preparation:** Prepares text input for AI processing.
- **1.5 AI Podcast Generation Agent:** Uses AI tools to research Wikipedia and compose a 5-minute podcast script.
- **1.6 Text-to-Speech Conversion:** Converts the podcast script into an AI-generated voice audio file.
- **1.7 Audio Delivery:** Sends the generated voice podcast back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures incoming updates from users interacting with the Telegram bot. It triggers the workflow on new messages.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**  
  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Role: Initiates the workflow upon receiving new user messages (text or voice)  
    - Configuration: Listens for "message" updates only  
    - Credentials: Uses Telegram Bot API credentials linked to the "Wikipedia Podcast Bot"  
    - Input: Incoming Telegram message updates  
    - Output: JSON containing the full message data (text or voice)  
    - Edge Cases: Telegram webhook errors, authorization failures, malformed messages  
    - Notes: Webhook ID set to manage Telegram webhook lifecycle  

---

#### 2.2 Input Type Routing

- **Overview:**  
  This block determines whether the incoming message is text or voice, routing each to the appropriate processing chain.

- **Nodes Involved:**  
  - Text or Voice (Switch)

- **Node Details:**  
  - **Text or Voice (Switch)**  
    - Type: Switch node  
    - Role: Routes data based on message content presence  
    - Configuration:  
      - Checks if message.text is non-empty → routes to "Text" output  
      - Checks if message.voice exists → routes to "Voice" output  
    - Input: JSON from Telegram Trigger  
    - Outputs: Two separate paths — one for text messages, one for voice messages  
    - Edge Cases: Messages without text or voice are not processed; potential for unhandled message types  
    - Notes: Case-sensitive and strict type validation to avoid false positives  

---

#### 2.3 Voice Message Processing

- **Overview:**  
  For voice messages, this block downloads the voice file from Telegram, then transcribes it to text using OpenAI Whisper.

- **Nodes Involved:**  
  - Get Voice Message  
  - Transcribe Voice Message

- **Node Details:**  
  - **Get Voice Message**  
    - Type: Telegram node (file download)  
    - Role: Retrieves the voice file from Telegram servers using the file_id  
    - Configuration: Uses file_id extracted from incoming voice message JSON  
    - Credentials: Telegram Bot API credentials  
    - Input: Voice message metadata from Switch's "Voice" output  
    - Output: Binary audio file data  
    - Edge Cases: File not found, Telegram API limits, network errors  

  - **Transcribe Voice Message**  
    - Type: OpenAI node (audio transcription)  
    - Role: Converts audio file to text using OpenAI Whisper transcription service  
    - Configuration: Resource set to "audio," operation "transcribe"  
    - Credentials: OpenAI API credentials  
    - Input: Binary audio from Get Voice Message  
    - Output: JSON with transcribed text  
    - Edge Cases: Transcription errors, audio format incompatibility, API rate limits  

---

#### 2.4 Text Message Preparation

- **Overview:**  
  For text messages (or after transcription), this block prepares the text payload for the AI podcast generation agent.

- **Nodes Involved:**  
  - Prepare Text Message for AI Agent

- **Node Details:**  
  - **Prepare Text Message for AI Agent**  
    - Type: Set node  
    - Role: Extracts and assigns the text content from the Telegram message JSON to a variable named "text"  
    - Configuration: Assigns `text = {{$json.message.text}}`  
    - Input: JSON from Switch's "Text" output or from Transcribe Voice Message node  
    - Output: JSON with a "text" property containing the topic request  
    - Edge Cases: Empty text values, encoding issues  

---

#### 2.5 AI Podcast Generation Agent

- **Overview:**  
  This core block uses an AI agent architecture with a language model and Wikipedia tool integration to research the topic and generate a structured podcast script.

- **Nodes Involved:**  
  - Anthropic Chat Model  
  - Wikipedia  
  - Think  
  - Wikipedia Podcast Agent

- **Node Details:**  
  - **Anthropic Chat Model**  
    - Type: Langchain Anthropic language model node  
    - Role: Provides the AI language understanding and generation core, configured with Claude 4 Sonnet model  
    - Configuration: Selected model "claude-sonnet-4-20250514" optimized for chat-based generation  
    - Credentials: Anthropic API  
    - Input: AI prompt and context from the agent node  
    - Output: Generated text segments from the AI model  
    - Edge Cases: API limits, timeouts, model unavailability  

  - **Wikipedia**  
    - Type: Langchain Wikipedia tool node  
    - Role: Queries Wikipedia to retrieve background, facts, and details about the requested topic  
    - Configuration: Default setup, integrated as a tool within the agent  
    - Input: Topic queries from the agent  
    - Output: Wikipedia content snippets  
    - Edge Cases: Missing or ambiguous Wikipedia pages, API failures  

  - **Think**  
    - Type: Langchain Tool Think node  
    - Role: Performs intermediate reasoning or summarization steps as part of the agent workflow  
    - Configuration: Default, used for internal AI tool chaining  
    - Input/Output: Connected within agent's toolset  
    - Edge Cases: Expression or reasoning errors  

  - **Wikipedia Podcast Agent**  
    - Type: Langchain agent node  
    - Role: Orchestrates AI generation combining model and tools to create a 5-minute podcast script (~600-750 words)  
    - Configuration:  
      - System message defines podcast host persona and detailed podcast creation methodology and style  
      - Uses Wikipedia and Think tools to research and refine content  
      - Prompt instructs structured output: intro, main body, outro with conversational style  
    - Input: Text topic from Prepare Text Message or transcription output  
    - Output: Podcast text ready for speech synthesis  
    - Edge Cases: Model hallucination, incomplete data, prompt misinterpretation  
    - Version: Requires Langchain agent support (version 2)  

---

#### 2.6 Text-to-Speech Conversion

- **Overview:**  
  Converts the AI-generated podcast script into natural, human-like speech audio using ElevenLabs TTS service.

- **Nodes Involved:**  
  - ElevenLabs Text to Speech

- **Node Details:**  
  - **ElevenLabs Text to Speech**  
    - Type: ElevenLabs TTS node  
    - Role: Synthesizes speech from the podcast text using a selected AI voice  
    - Configuration:  
      - Input text: podcast script from Wikipedia Podcast Agent output  
      - Voice: "Ramona - Calm & Soothing Voice" selected from voice list  
    - Credentials: ElevenLabs API  
    - Input: Podcast text string  
    - Output: Binary audio file (mp3)  
    - Edge Cases: API failures, unsupported characters, long text limits  

---

#### 2.7 Audio Delivery

- **Overview:**  
  Sends the generated podcast audio file back to the Telegram user as a voice message with a caption.

- **Nodes Involved:**  
  - Send Voice Response

- **Node Details:**  
  - **Send Voice Response**  
    - Type: Telegram node (send audio)  
    - Role: Uploads and sends the generated mp3 file to the user’s chat  
    - Configuration:  
      - Chat ID from Telegram Trigger message context  
      - Sends as audio with caption "🎙️ Wikipedia-Podcast: [user's original text]"  
      - File name derived from user's message text, with .mp3 extension  
      - Binary data attachment enabled  
    - Credentials: Telegram Bot API  
    - Input: Binary audio from ElevenLabs TTS, chat metadata from original message  
    - Output: Confirmation message from Telegram API  
    - Edge Cases: File upload failures, chat ID errors, Telegram API limits  

---

### 3. Summary Table

| Node Name                     | Node Type                                 | Functional Role                         | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                                                  |
|-------------------------------|-------------------------------------------|---------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger              | Telegram Trigger                          | Entry point: receives Telegram messages| None                        | Text or Voice               | Welcome note describes workflow sequence and requirements                                                                    |
| Text or Voice                | Switch                                   | Routes message by input type           | Telegram Trigger             | Prepare Text Message for AI Agent, Get Voice Message |                                                                                                                              |
| Prepare Text Message for AI Agent | Set                                     | Extracts and prepares text input       | Text or Voice (Text output)  | Wikipedia Podcast Agent      |                                                                                                                              |
| Get Voice Message            | Telegram (file download)                  | Downloads voice message file            | Text or Voice (Voice output) | Transcribe Voice Message     |                                                                                                                              |
| Transcribe Voice Message     | OpenAI (audio transcription)              | Transcribes voice audio to text         | Get Voice Message            | Wikipedia Podcast Agent      |                                                                                                                              |
| Anthropic Chat Model         | Langchain Anthropic Chat Model            | AI language model for podcast generation| Wikipedia Podcast Agent (ai_languageModel) | Wikipedia Podcast Agent      |                                                                                                                              |
| Wikipedia                   | Langchain Wikipedia Tool                   | Fetches Wikipedia content               | Wikipedia Podcast Agent (ai_tool) | Wikipedia Podcast Agent      |                                                                                                                              |
| Think                       | Langchain Think Tool                       | Intermediate reasoning tool             | Wikipedia Podcast Agent (ai_tool) | Wikipedia Podcast Agent      |                                                                                                                              |
| Wikipedia Podcast Agent     | Langchain Agent                            | Core AI agent to create podcast script | Prepare Text Message for AI Agent, Transcribe Voice Message | ElevenLabs Text to Speech    |                                                                                                                              |
| ElevenLabs Text to Speech   | ElevenLabs Text-to-Speech                  | Converts text podcast to speech audio  | Wikipedia Podcast Agent      | Send Voice Response          |                                                                                                                              |
| Send Voice Response         | Telegram Send Audio                        | Sends generated podcast audio to user  | ElevenLabs Text to Speech    | None                        |                                                                                                                              |
| Sticky Note                 | Sticky Note                               | Documentation note                      | None                        | None                        | Welcome to my Wikipedia Podcast Telegram Agent Workflow! Sequence, credentials, features, and use cases described in detail. |

---

### 4. Reproducing the Workflow from Scratch

To recreate this workflow in n8n:

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Parameters: Listen to "message" updates only  
   - Credentials: Link your Telegram Bot API credentials (Wikipedia Podcast Bot)  
   - Position: Starting point

2. **Add Switch node named "Text or Voice"**  
   - Type: Switch (version 3.2)  
   - Rules:  
     - Output "Text": Condition → message.text is not empty (strict, case-sensitive)  
     - Output "Voice": Condition → message.voice exists (object existence)  
   - Connect Telegram Trigger's main output to this Switch node

3. **Prepare Text Message for AI Agent node**  
   - Type: Set node  
   - Assign variable "text" with expression `{{$json.message.text}}`  
   - Connect Switch node "Text" output to this node

4. **Get Voice Message node**  
   - Type: Telegram node (download file)  
   - Parameters: Use `{{$json.message.voice.file_id}}` for fileId, resource "file"  
   - Credentials: Telegram Bot API  
   - Connect Switch node "Voice" output to this node

5. **Transcribe Voice Message node**  
   - Type: OpenAI node  
   - Resource: Audio  
   - Operation: Transcribe  
   - Credentials: OpenAI API  
   - Connect Get Voice Message output to this node

6. **Wikipedia Podcast Agent node**  
   - Type: Langchain Agent node (version 2)  
   - Parameters:  
     - Text input: `{{$json.text}}` (from Prepare Text Message or transcription output)  
     - System message: Paste the detailed podcast host persona and instructions (as provided in workflow)  
     - Use Wikipedia and Think tools  
     - Language model: Anthropic Chat Model (Claude 4 Sonnet)  
   - Connect Prepare Text Message and Transcribe Voice Message outputs to this node

7. **Anthropic Chat Model node**  
   - Type: Langchain Anthropic Chat Model  
   - Model: "claude-sonnet-4-20250514"  
   - Credentials: Anthropic API  
   - Connect as AI languageModel input to Wikipedia Podcast Agent node

8. **Wikipedia node**  
   - Type: Langchain Wikipedia Tool  
   - Default config  
   - Connect as ai_tool input to Wikipedia Podcast Agent node

9. **Think node**  
   - Type: Langchain Think Tool  
   - Default config  
   - Connect as ai_tool input to Wikipedia Podcast Agent node

10. **ElevenLabs Text to Speech node**  
    - Type: ElevenLabs TTS  
    - Text input: `{{$json.output}}` from Wikipedia Podcast Agent node  
    - Voice: Select "Ramona - Calm & Soothing Voice" or any preferred voice preset  
    - Credentials: ElevenLabs API  
    - Connect Wikipedia Podcast Agent output to this node

11. **Send Voice Response node**  
    - Type: Telegram node (send audio)  
    - Parameters:  
      - Chat ID: `{{$('Telegram Trigger').item.json.message.chat.id}}`  
      - Operation: sendAudio  
      - Attach binary audio data from ElevenLabs node  
      - Caption: `"🎙️ Wikipedia-Podcast: {{$('Telegram Trigger').item.json.message.text}}"`  
      - Filename: `"{{$('Telegram Trigger').item.json.message.text}}.mp3"`  
    - Credentials: Telegram Bot API  
    - Connect ElevenLabs Text to Speech output to this node

12. **Add Sticky Note** (optional)  
    - Add detailed notes about workflow sequence, credentials, use cases, and links (copy content from workflow sticky note)

13. **Set Execution Order**  
    - Ensure the nodes are connected in the order described above, with proper branching at the Switch node

14. **Test and Deploy**  
    - Activate webhook for Telegram Trigger  
    - Provide required API credentials for Telegram, Anthropic, OpenAI, and ElevenLabs  
    - Test with both text and voice message inputs in Telegram

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Telegram Bot API integration documentation                                                                               | https://docs.n8n.io/integrations/builtin/credentials/telegram/                                                            |
| Anthropic API (Claude 4 Sonnet) integration documentation                                                                | https://docs.n8n.io/integrations/builtin/credentials/anthropic/                                                           |
| OpenAI API for voice transcription with Whisper                                                                         | https://docs.n8n.io/integrations/builtin/credentials/openai/                                                              |
| ElevenLabs API for text-to-speech AI voice synthesis                                                                     | https://github.com/n8n-ninja/n8n-nodes-elevenlabs?tab=readme-ov-fil/                                                       |
| Workflow creator LinkedIn contact                                                                                         | https://www.linkedin.com/in/friedemann-schuetzt                                                                            |
| The workflow supports multi-input (text and voice), enabling accessibility and varied user preferences for Wikipedia info |                                                                                                                           |
| Podcast creation instructions embedded in AI prompt include style, structure, and tone guidelines                         | Detailed prompt in Wikipedia Podcast Agent node system message section                                                     |
| The architecture uses Langchain agent framework integrating multiple AI tools and models                                  | Enables modular and extensible AI processing pipeline                                                                      |

---

**Disclaimer:** The provided content is exclusively derived from an automated workflow created using n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.