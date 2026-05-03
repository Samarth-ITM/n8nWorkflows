Translate Telegram audio messages with AI (55 supported languages)

https://n8nworkflows.xyz/workflows/translate-telegram-audio-messages-with-ai--55-supported-languages--2206


# Translate Telegram audio messages with AI (55 supported languages)

### 1. Workflow Overview

This workflow implements a Telegram bot that translates audio voice messages between two languages using AI-powered speech-to-text, language detection, translation, and text-to-speech capabilities. It supports 55 languages with default translation between English and French but can be customized.

**Target use cases:**  
- Users speaking different languages communicate via Telegram voice messages.  
- Language learners or travelers communicate seamlessly.  

**Logical blocks:**

- **1.1 Telegram Input Reception:** Captures incoming audio voice messages from Telegram users.  
- **1.2 Preprocessing & Settings:** Sets translation language preferences and prepares for error handling.  
- **1.3 Audio Download & Transcription:** Downloads the Telegram voice file and transcribes speech to text via OpenAI audio transcription.  
- **1.4 AI Language Detection & Translation:** Detects the language of the transcribed text and translates it to the target language using AI.  
- **1.5 Output Preparation & Reply:** Sends the translated text and synthesized speech audio back to the Telegram user.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input Reception

- **Overview:**  
  Listens for any incoming Telegram updates, capturing voice messages sent to the bot.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**  
  - **Telegram Trigger**  
    - Type: Trigger node for Telegram updates  
    - Configuration: Listens to all update types (`updates: ["*"]`)  
    - Credentials: Telegram API Token required (configured via "Telegram account")  
    - Inputs: None (webhook trigger)  
    - Outputs: Emits update data including voice message file IDs and chat IDs  
    - Edge cases: Missing or expired webhook configuration, invalid Telegram credentials, bot not started, unsupported update types  
    - Notes: Essential entry point, must be publicly accessible webhook URL

#### 2.2 Preprocessing & Settings

- **Overview:**  
  Sets the translation language preferences and prepares basic error handling for input data.

- **Nodes Involved:**  
  - Settings  
  - Input Error Handling

- **Node Details:**  
  - **Settings**  
    - Type: Set node  
    - Configuration: Assigns two string variables: `language_native` (default "english") and `language_translate` (default "french")  
    - Inputs: From Telegram Trigger  
    - Outputs: Emits these language settings for use downstream  
    - Edge cases: None critical; values can be customized as needed

  - **Input Error Handling**  
    - Type: Set node  
    - Configuration: Sets the field `message.text` to the incoming JSON's message text or empty string if absent (`{{$json?.message?.text || ""}}`)  
    - Inputs: From Settings node  
    - Outputs: Prepares normalized input for next nodes  
    - Edge cases: Handles missing text gracefully to avoid null reference errors

#### 2.3 Audio Download & Transcription

- **Overview:**  
  Downloads the voice message audio file from Telegram and transcribes it to text using OpenAI's speech-to-text API.

- **Nodes Involved:**  
  - Telegram1  
  - OpenAI2

- **Node Details:**  
  - **Telegram1**  
    - Type: Telegram node  
    - Configuration: Downloads the voice message file using `file_id` from Telegram Trigger (`{{$json.message.voice.file_id}}`)  
    - Inputs: From Input Error Handling  
    - Outputs: Binary audio data of the voice message  
    - Edge cases: Invalid file_id, network/timeouts errors, Telegram API limits

  - **OpenAI2**  
    - Type: OpenAI node (audio resource)  
    - Operation: Transcribe audio to text  
    - Configuration: Uses OpenAI's audio transcription endpoint on the downloaded audio binary data  
    - Inputs: From Telegram1 (binary audio)  
    - Outputs: JSON with transcribed text  
    - Credentials: OpenAI API key required  
    - Edge cases: Audio format unsupported, transcription failures, API rate limits, authentication errors

#### 2.4 AI Language Detection & Translation

- **Overview:**  
  Detects the language of the transcribed text and automatically translates it between the native and target languages defined in Settings.

- **Nodes Involved:**  
  - Auto-detect and translate  
  - OpenAI Chat Model

- **Node Details:**  
  - **Auto-detect and translate**  
    - Type: Langchain Chain LLM node  
    - Configuration: Defines a prompt that instructs the AI to detect the language of the input text and translate it to the other language:  
      - If text language equals `language_native`, translate to `language_translate`  
      - If text language equals `language_translate`, translate to `language_native`  
      - Output only the translation, no explanation  
    - Key expressions: Uses values from Settings node via expressions (`{{ $('Settings').item.json.language_native }}` etc.)  
    - Inputs: Receives transcribed text from OpenAI2 and AI language model from OpenAI Chat Model node  
    - Outputs: Translated text string  
    - Edge cases: Ambiguous language detection, unsupported languages, AI model downtime or errors

  - **OpenAI Chat Model**  
    - Type: Langchain Chat LLM node  
    - Configuration: Provides the AI language model interface for the chain  
    - Inputs: Connected to Auto-detect and translate node as `ai_languageModel` input  
    - Outputs: AI responses used by Auto-detect and translate  
    - Credentials: OpenAI API key required  
    - Edge cases: API errors, rate limits, model unavailability

#### 2.5 Output Preparation & Reply

- **Overview:**  
  Sends the translated text as a Telegram message and also synthesizes speech audio from the translated text to send as a voice/audio message.

- **Nodes Involved:**  
  - Text reply  
  - OpenAI  
  - Audio reply

- **Node Details:**  
  - **Text reply**  
    - Type: Telegram node  
    - Configuration: Sends text reply using the translated text (`{{$json.text}}`) to the original chat ID (`{{$('Telegram Trigger').item.json.message.chat.id}}`), with Markdown parsing enabled  
    - Inputs: From Auto-detect and translate  
    - Outputs: None (final message output)  
    - Credentials: Telegram API required  
    - Edge cases: Invalid chat ID, message length limits, Telegram API errors

  - **OpenAI**  
    - Type: OpenAI node (audio resource)  
    - Operation: Generate speech audio from text input  
    - Configuration: Uses the translated text as input to generate speech audio binary data  
    - Inputs: From Auto-detect and translate (translated text)  
    - Credentials: OpenAI API key required  
    - Edge cases: Unsupported language voices, API errors, rate limits

  - **Audio reply**  
    - Type: Telegram node  
    - Operation: Send audio file to Telegram chat  
    - Configuration: Sends binary audio data received from OpenAI node to the user chat ID  
    - Inputs: From OpenAI node (audio binary)  
    - Credentials: Telegram API required  
    - Edge cases: Audio format unsupported by Telegram, large file size, API errors

---

### 3. Summary Table

| Node Name             | Node Type                               | Functional Role                               | Input Node(s)           | Output Node(s)                 | Sticky Note                                                                                                                               |
|-----------------------|---------------------------------------|-----------------------------------------------|-------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger       | n8n-nodes-base.telegramTrigger         | Receive incoming Telegram updates and voice   | None                    | Settings                      |                                                                                                                                           |
| Settings              | n8n-nodes-base.set                     | Set native and target translation languages   | Telegram Trigger        | Input Error Handling          |                                                                                                                                           |
| Input Error Handling   | n8n-nodes-base.set                     | Normalize input text field                      | Settings                | Telegram1                    |                                                                                                                                           |
| Telegram1             | n8n-nodes-base.telegram                | Download voice audio file from Telegram        | Input Error Handling    | OpenAI2                      |                                                                                                                                           |
| OpenAI2               | @n8n/n8n-nodes-langchain.openAi       | Transcribe speech audio to text                 | Telegram1               | Auto-detect and translate    |                                                                                                                                           |
| OpenAI Chat Model      | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide AI language model interface             | None                    | Auto-detect and translate    |                                                                                                                                           |
| Auto-detect and translate | @n8n/n8n-nodes-langchain.chainLlm    | Detect text language and translate accordingly | OpenAI2, OpenAI Chat Model | Text reply, OpenAI           | Sticky Note1: Translation - Converts from speech to text; Translates native to target language as specified in Settings node               |
| Text reply             | n8n-nodes-base.telegram                | Send translated text back to Telegram user     | Auto-detect and translate | None                         | Sticky Note: Telegram output - Provides output in text; supports many languages including English, French, German, Spanish, Chinese, Japanese |
| OpenAI                 | @n8n/n8n-nodes-langchain.openAi       | Generate speech audio from translated text     | Auto-detect and translate | Audio reply                  |                                                                                                                                           |
| Audio reply            | n8n-nodes-base.telegram                | Send synthesized speech audio to Telegram user | OpenAI                  | None                         |                                                                                                                                           |
| Sticky Note            | n8n-nodes-base.stickyNote              | Informational notes                            | None                    | None                         | Sticky Note2: Telegram output - Output in text and speech with supported languages and link to full language list                           |
| Sticky Note1           | n8n-nodes-base.stickyNote              | Informational notes                            | None                    | None                         | See Auto-detect and translate                                                                                                            |
| Sticky Note2           | n8n-nodes-base.stickyNote              | Informational notes                            | None                    | None                         | See Text reply                                                                                                                            |
| Sticky Note (Intro)    | n8n-nodes-base.stickyNote              | Overview and use case description               | None                    | None                         | Multi-lingual AI Powered Universal Translator with Speech ⭐ Key capabilities, use cases, and setup instructions                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Configure credential with your Telegram Bot API token (from BotFather)  
   - Set Updates to listen to `"*"` (all updates)  
   - Position: starting point

2. **Create Settings node:**  
   - Type: Set  
   - Connect from Telegram Trigger  
   - Assign two string variables:  
     - `language_native` = `"english"` (default)  
     - `language_translate` = `"french"` (default)  
   - This node defines the translation language pair

3. **Create Input Error Handling node:**  
   - Type: Set  
   - Connect from Settings  
   - Set `message.text` field to expression: `{{$json?.message?.text || ""}}`  
   - This ensures text field is always present for later nodes

4. **Create Telegram node ("Telegram1") to download audio:**  
   - Type: Telegram  
   - Connect from Input Error Handling  
   - Set operation to default (download file)  
   - Set File ID as expression: `{{$json.message.voice.file_id}}`  
   - Credential: Telegram API  
   - This node downloads the voice message file for transcription

5. **Create OpenAI node ("OpenAI2") for transcription:**  
   - Type: OpenAI  
   - Connect from Telegram1  
   - Resource: Audio  
   - Operation: Transcribe  
   - Credential: OpenAI API key  
   - Use binary data input from Telegram1  
   - This node transcribes speech audio to text

6. **Create OpenAI Chat Model node:**  
   - Type: Langchain Chat LLM  
   - No direct input connections  
   - Credential: OpenAI API key  
   - Provides the AI model for language detection and translation

7. **Create Auto-detect and translate node:**  
   - Type: Langchain Chain LLM  
   - Connect main input from OpenAI2 (transcribed text)  
   - Connect AI language model input from OpenAI Chat Model node  
   - Configure Prompt Type: Define  
   - Prompt text:  
     ```
     Detect the language of the text that follows.  
     - If it is {{ $('Settings').item.json.language_native }} translate to {{ $('Settings').item.json.language_translate }}.  
     - If it is in {{ $('Settings').item.json.language_translate }} translate to {{ $('Settings').item.json.language_native }}.  
     - In the output just provide the translation and do not explain it. Just provide the translation without anything else.  

     Text:  
     {{ $json.text }}
     ```
   - This node detects and translates text automatically

8. **Create Text reply node:**  
   - Type: Telegram  
   - Connect from Auto-detect and translate  
   - Text: Expression `{{$json.text}}` (translated text)  
   - Chat ID: Expression `{{$('Telegram Trigger').item.json.message.chat.id}}`  
   - Enable Markdown parse mode in additional fields  
   - Credential: Telegram API  
   - Sends translated text back to user

9. **Create OpenAI node ("OpenAI") for speech synthesis:**  
   - Type: OpenAI  
   - Connect from Auto-detect and translate  
   - Resource: Audio  
   - Operation: Generate speech audio from text input  
   - Input: Expression `{{$json.text}}` (translated text)  
   - Credential: OpenAI API key  
   - This node generates speech audio of the translation

10. **Create Audio reply node:**  
    - Type: Telegram  
    - Connect from OpenAI (speech audio)  
    - Operation: Send Audio  
    - Chat ID: Expression `{{$('Telegram Trigger').item.json.message.chat.id}}`  
    - Send binary data (audio)  
    - Credential: Telegram API  
    - Sends synthesized speech audio back to user

11. **Activate the workflow:**  
    - Deploy the workflow and ensure webhook URL for Telegram Trigger is publicly accessible and registered with Telegram Bot API.

12. **Customize as needed:**  
    - Modify Settings node to change default translation languages  
    - Update AI prompts for more language pairs or different behavior

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                                   |
|----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Full list of supported languages for OpenAI speech-to-text and speech synthesis can be found here:                   | https://platform.openai.com/docs/guides/speech-to-text/supported-languages                                      |
| Setup instructions to obtain Telegram Bot API token: Start a chat with BotFather, create a new bot, copy token       | Telegram BotFather                                                                                               |
| This workflow enables multi-lingual AI powered universal translation with voice input and output on Telegram         | Introductory sticky note describing key capabilities and use cases                                              |
| Translate between English and French by default; users can customize language pairs by editing the Settings node     | Settings node explanation                                                                                        |

---

This document fully describes the workflow structure, node configurations, data flow, and key considerations to reproduce, understand, or extend the Telegram audio message translation workflow.