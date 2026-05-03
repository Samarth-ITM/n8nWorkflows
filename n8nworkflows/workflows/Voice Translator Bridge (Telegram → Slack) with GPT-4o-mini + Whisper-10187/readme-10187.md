Voice Translator Bridge (Telegram → Slack) with GPT-4o-mini + Whisper

https://n8nworkflows.xyz/workflows/voice-translator-bridge--telegram---slack--with-gpt-4o-mini---whisper-10187


# Voice Translator Bridge (Telegram → Slack) with GPT-4o-mini + Whisper

### 1. Workflow Overview

This workflow, titled **Voice Translator Bridge (Telegram → Slack) with GPT-4o-mini + Whisper**, automates the process of receiving voice messages from a Telegram bot, transcribing the audio to text using OpenAI's Whisper model, detecting the source language, translating the text into the opposite language (Japanese ↔ English) using GPT-4o-mini, and then posting the translated message into a specified Slack channel. The key use case is enabling seamless voice message translation between Japanese and English speakers via Telegram and Slack integration.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Validation**: Receives Telegram voice messages and verifies the input type.
- **1.2 Telegram Voice File Retrieval**: Extracts file metadata and builds a direct download URL for the voice message.
- **1.3 Audio Download & Preparation**: Downloads the voice file binary and prepares it for transcription.
- **1.4 Transcription & Language Detection**: Uses Whisper to transcribe audio and detects the source language.
- **1.5 Translation with GPT-4o-mini**: Translates the transcribed text from Japanese to English or vice versa.
- **1.6 Slack Message Construction & Posting**: Creates a formatted Slack message with original and translated texts, posting it to a Slack channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Validation

- **Overview:** Listens for incoming Telegram messages and filters only voice messages to proceed.
- **Nodes Involved:** `Telegram Trigger`, `Is Voice?`

**Node Details:**

- **Telegram Trigger**
  - Type: Telegram Trigger (Webhook)
  - Role: Starts workflow on Telegram message events.
  - Configuration: Listens for `message` updates.
  - Credentials: Telegram Bot API token (`TELEGRAM_BOT_CRED`).
  - Outputs: Full Telegram message JSON including sender, voice, and text fields.
  - Edge Cases: If the bot token is invalid or Telegram API is unreachable, trigger will fail. Only triggers on new messages.
  - Sticky Note: Describes purpose to start on Telegram voice messages, outputs relevant fields like `message.voice.file_id`.

- **Is Voice?**
  - Type: If (Conditional)
  - Role: Checks if incoming message contains the `voice` element.
  - Configuration: Expression evaluates if `message.voice` is defined.
  - Inputs: Telegram Trigger node output.
  - Outputs: Passes only voice messages forward.
  - Edge Cases: Messages without voice elements are filtered out; no further processing.

---

#### 1.2 Telegram Voice File Retrieval

- **Overview:** Extracts necessary metadata from the Telegram message and retrieves the actual file path via Telegram API.
- **Nodes Involved:** `Get File ID`, `Add Bot Token`, `Telegram getFile`, `Build File URL`

**Node Details:**

- **Get File ID**
  - Type: Set
  - Role: Extracts `file_id`, `chat_id`, and `from_username` (falls back to `first_name` or `"unknown"`).
  - Inputs: `Is Voice?` node.
  - Outputs: JSON with extracted fields.
  - Edge Cases: Missing username or first name handled with `"unknown"` default.

- **Add Bot Token**
  - Type: Set
  - Role: Adds the Telegram bot token to the data for subsequent API calls.
  - Inputs: `Get File ID`.
  - Outputs: Adds `bot_token` field.
  - Configuration: User must replace `{{YOUR_TELEGRAM_BOT_TOKEN}}` with actual token.
  - Edge Cases: Missing or invalid bot token breaks API calls.

- **Telegram getFile**
  - Type: HTTP Request
  - Role: Calls Telegram API `getFile` endpoint to get file metadata including `file_path`.
  - Inputs: `Add Bot Token`.
  - Configuration: URL constructed using bot token and `file_id`.
  - Outputs: JSON containing `file_path`.
  - Edge Cases: API errors if file_id is invalid or token unauthorized.

- **Build File URL**
  - Type: Function
  - Role: Constructs the direct download URL for the voice file from Telegram.
  - Inputs: `Telegram getFile`, `Get File ID`, `Add Bot Token`.
  - Outputs: JSON with `file_url`, `chat_id`, and `from_username`.
  - Edge Cases: Requires `file_path` and valid token; missing data causes failure.

---

#### 1.3 Audio Download & Preparation

- **Overview:** Downloads the voice file binary from Telegram and prepares it for transcription by Whisper.
- **Nodes Involved:** `Download Voice File`, `Prepare Whisper Input1`

**Node Details:**

- **Download Voice File**
  - Type: HTTP Request
  - Role: Downloads the voice file as binary data from the constructed URL.
  - Inputs: `Build File URL`.
  - Configuration: GET request to `file_url`; response expected as file binary.
  - Outputs: Binary data under `$binary.data`.
  - Edge Cases: Download failure if URL invalid, network issues, or file size too large.

- **Prepare Whisper Input1**
  - Type: Code
  - Role: Copies binary data from `$binary.data` to `$binary.audio` for Whisper compatibility.
  - Inputs: `Download Voice File`.
  - Outputs: JSON with binary audio keyed as `audio`.
  - Edge Cases: Missing binary data causes transcription failure.

---

#### 1.4 Transcription & Language Detection

- **Overview:** Converts audio to text with Whisper, then detects whether the language is Japanese or English.
- **Nodes Involved:** `Transcribe a recording`, `Extract Transcript`, `Detect Language`

**Node Details:**

- **Transcribe a recording**
  - Type: OpenAI Audio Transcription (Whisper)
  - Role: Transcribes audio binary into text.
  - Inputs: `Prepare Whisper Input1` (binary audio).
  - Credentials: OpenAI API key (`OPENAI_API_KEY_HEADER`).
  - Outputs: JSON with transcription text.
  - Edge Cases: Max file size 25MB; API limits or auth errors possible.

- **Extract Transcript**
  - Type: Set
  - Role: Extracts transcription text and `from_username` for downstream use.
  - Inputs: `Transcribe a recording`, `Build File URL`.
  - Outputs: JSON with `transcript_text` and `from_username`.
  - Edge Cases: Empty or invalid transcription text.

- **Detect Language**
  - Type: Function
  - Role: Heuristically determines source language (Japanese if text contains Kanji or Kana characters, else English).
  - Inputs: `Extract Transcript`.
  - Outputs: JSON with `original_text`, `source_lang` (`ja` or `en`), `target_lang` (opposite), and `from_username`.
  - Edge Cases: Non-Japanese/English text defaults to English source.

---

#### 1.5 Translation with GPT-4o-mini

- **Overview:** Translates the original transcribed text from source to target language using OpenAI GPT-4o-mini chat completion.
- **Nodes Involved:** `Translate (OpenAI)`

**Node Details:**

- **Translate (OpenAI)**
  - Type: HTTP Request
  - Role: Sends transcription and language info to GPT-4o-mini model for translation.
  - Inputs: `Detect Language`.
  - Credentials: OpenAI API key (`OPENAI_HEADER_AUTH`).
  - Configuration:
    - POST to `https://api.openai.com/v1/chat/completions`.
    - Body includes system prompt enforcing exact translation and user message with source/target languages and text.
    - Temperature set to 0.2 for controlled output.
  - Outputs: JSON with translated text under `choices[0].message.content`.
  - Edge Cases: API quota, authentication errors, incomplete translations.

---

#### 1.6 Slack Message Construction & Posting

- **Overview:** Formats the translation and original text into a Slack message and posts it to a target Slack channel.
- **Nodes Involved:** `Build Slack Message`, `Post to Slack`

**Node Details:**

- **Build Slack Message**
  - Type: Function
  - Role: Creates a Slack-formatted message including country flag emojis, Telegram username, original transcript, and translation.
  - Inputs: `Translate (OpenAI)`, `Detect Language`, `Extract Transcript`, `Get File ID`.
  - Outputs: JSON with `slack_text`.
  - Edge Cases: Missing username or texts default to placeholders.

- **Post to Slack**
  - Type: HTTP Request
  - Role: Posts the constructed message text to a specified Slack channel.
  - Inputs: `Build Slack Message`.
  - Credentials: Slack Bot Token (`SLACK_BOT_TOKEN_HEADER`).
  - Configuration:
    - POST to `https://slack.com/api/chat.postMessage`.
    - Channel ID must be set by user in the body (replace `{{YOUR_SLACK_CHANNEL_ID}}`).
  - Edge Cases: Invalid token, missing channel ID, message format errors.

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                             | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                  |
|------------------------|---------------------|--------------------------------------------|------------------------|------------------------|----------------------------------------------------------------------------------------------|
| Telegram Trigger       | Telegram Trigger    | Start workflow on Telegram message event   | —                      | Is Voice?              | Starts on Telegram voice messages; outputs message with voice file_id                        |
| Is Voice?              | If                  | Check if message contains voice             | Telegram Trigger       | Get File ID            | Filters only voice messages                                                                 |
| Get File ID            | Set                 | Extract file_id, chat_id, username          | Is Voice?              | Add Bot Token          | Extracts Telegram file_id and sender info                                                   |
| Add Bot Token          | Set                 | Append Telegram bot token for API calls     | Get File ID            | Telegram getFile       | Adds bot token needed for Telegram API calls                                                |
| Telegram getFile       | HTTP Request        | Get file_path from Telegram API              | Add Bot Token          | Build File URL         | Retrieves file path metadata                                                                |
| Build File URL         | Function            | Build direct download URL for voice file    | Telegram getFile       | Download Voice File    | Constructs actual file URL for download                                                    |
| Download Voice File    | HTTP Request        | Download audio file in binary form           | Build File URL         | Prepare Whisper Input1 | Downloads voice audio as binary data                                                       |
| Prepare Whisper Input1 | Code                | Prepare binary audio data for Whisper        | Download Voice File    | Transcribe a recording | Renames binary key to `audio` for Whisper compatibility                                   |
| Transcribe a recording | OpenAI Audio Node   | Transcribe audio to text using Whisper       | Prepare Whisper Input1 | Extract Transcript     | Transcribes audio to text                                                                   |
| Extract Transcript     | Set                 | Extract transcription text and username     | Transcribe a recording | Detect Language        | Extracts and formats transcript text                                                       |
| Detect Language        | Function            | Detect source language (ja/en)                | Extract Transcript     | Translate (OpenAI)     | Detects language for translation direction                                                |
| Translate (OpenAI)     | HTTP Request        | Translate text with GPT-4o-mini model        | Detect Language        | Build Slack Message    | Translates transcription accurately                                                        |
| Build Slack Message    | Function            | Format message with flags, username, texts  | Translate (OpenAI)     | Post to Slack          | Prepares Slack message content                                                             |
| Post to Slack          | HTTP Request        | Post message to Slack channel                 | Build Slack Message    | —                      | Posts translated message to Slack                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Telegram Trigger` Node:**
   - Type: Telegram Trigger
   - Parameters:
     - Listen for updates: `message`
   - Credentials:
     - Connect your Telegram Bot API token credential.
   - Position: Start node.

2. **Create `Is Voice?` Node (If):**
   - Condition: Check if `{{$json["message"]["voice"] !== undefined ? "voice" : "other"}}` equals `"voice"`.
   - Connect `Telegram Trigger` → `Is Voice?`.

3. **Create `Get File ID` Node (Set):**
   - Extract:
     - `file_id` = `{{$json["message"]["voice"]["file_id"]}}`
     - `chat_id` = `{{$json["message"]["chat"]["id"]}}`
     - `from_username` = `{{$json["message"]["from"]["username"] || $json["message"]["from"]["first_name"] || "unknown"}}`
   - Connect `Is Voice?` (true branch) → `Get File ID`.

4. **Create `Add Bot Token` Node (Set):**
   - Add string field `bot_token` with value: `{{YOUR_TELEGRAM_BOT_TOKEN}}` (replace with your Telegram Bot token).
   - Connect `Get File ID` → `Add Bot Token`.

5. **Create `Telegram getFile` Node (HTTP Request):**
   - URL: `https://api.telegram.org/bot{{$json["bot_token"]}}/getFile?file_id={{$json["file_id"]}}`
   - Method: GET
   - Connect `Add Bot Token` → `Telegram getFile`.

6. **Create `Build File URL` Node (Function):**
   - Code:
     ```js
     const filePath = items[0].json.result.file_path;
     const token = $items("Add Bot Token")[0].json.bot_token;
     const fileUrl = `https://api.telegram.org/file/bot${token}/${filePath}`;
     const chat_id = $items("Get File ID")[0].json.chat_id;
     const from_username = $items("Get File ID")[0].json.from_username;
     return [{ json: { file_url: fileUrl, chat_id, from_username } }];
     ```
   - Connect `Telegram getFile` → `Build File URL`.

7. **Create `Download Voice File` Node (HTTP Request):**
   - URL: `{{$json["file_url"]}}`
   - Method: GET
   - Response format: File (binary)
   - Connect `Build File URL` → `Download Voice File`.

8. **Create `Prepare Whisper Input1` Node (Code):**
   - Code:
     ```js
     return [{
       json: { ...$json },
       binary: { audio: $binary.data }
     }];
     ```
   - Connect `Download Voice File` → `Prepare Whisper Input1`.

9. **Create `Transcribe a recording` Node (OpenAI Audio):**
   - Resource: Audio
   - Operation: Transcribe
   - Binary Property Name: `audio`
   - Credentials: OpenAI API key credential.
   - Connect `Prepare Whisper Input1` → `Transcribe a recording`.

10. **Create `Extract Transcript` Node (Set):**
    - Set `transcript_text` = `{{$json.text}}`
    - Set `from_username` = `{{$items("Build File URL")[0].json.from_username}}`
    - Connect `Transcribe a recording` → `Extract Transcript`.

11. **Create `Detect Language` Node (Function):**
    - Code:
      ```js
      const text = $json.transcript_text || "";
      const jaRegex = /[ぁ-んァ-ン一-龠]/;
      const isJa = jaRegex.test(text);
      return [{ json: { original_text: text, source_lang: isJa ? 'ja' : 'en', target_lang: isJa ? 'en' : 'ja', from_username: $json.from_username } }];
      ```
    - Connect `Extract Transcript` → `Detect Language`.

12. **Create `Translate (OpenAI)` Node (HTTP Request):**
    - Method: POST
    - URL: `https://api.openai.com/v1/chat/completions`
    - Authentication: Header Auth with OpenAI API key.
    - Headers:
      - Authorization: `Bearer {{$credentials.httpHeaderAuth.headerValue}}`
      - Content-Type: `application/json`
    - Body Parameters (raw JSON):
      ```json
      {
        "model": "gpt-4o-mini",
        "temperature": 0.2,
        "messages": [
          { "role": "system", "content": "You are a translation engine. Translate exactly, preserving meaning and tone. Output ONLY the translated text." },
          { "role": "user", "content": "Source language: {{$json.source_lang}}\nTarget language: {{$json.target_lang}}\nText:\n{{$json.original_text}}" }
        ]
      }
      ```
    - Connect `Detect Language` → `Translate (OpenAI)`.

13. **Create `Build Slack Message` Node (Function):**
    - Code:
      ```js
      function flagFor(lang) {
        if (lang === 'ja') return '🇯🇵';
        if (lang === 'en') return '🇺🇸';
        return '🌐';
      }
      const sourceLang = $json.source_lang || 'en';
      const targetLang = $json.target_lang || 'en';
      const src = flagFor(sourceLang);
      const dst = flagFor(targetLang);
      const username = $node["Get File ID"].json.from_username || 'unknown';
      const original = $node["Extract Transcript"].json.transcript_text || '(no transcript)';
      const translated = $node["Translate (OpenAI)"].json.choices[0].message.content || '(no translation)';
      const text = `*${src} → ${dst}* (@${username})\n> ${original}\n\n${translated}\n`;
      return [{ json: { slack_text: text } }];
      ```
    - Connect `Translate (OpenAI)` → `Build Slack Message`.

14. **Create `Post to Slack` Node (HTTP Request):**
    - Method: POST
    - URL: `https://slack.com/api/chat.postMessage`
    - Authentication: Header Auth with Slack Bot token.
    - Body Parameters:
      - `channel`: Replace with your Slack channel ID (e.g., `"C09NT81DQU"`).
      - `text`: `{{$json.slack_text}}`
    - Connect `Build Slack Message` → `Post to Slack`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Max audio file size for Whisper transcription is 25 MB.                                                                                                                                                                    | Important limitation for audio files.                              |
| Slack channel ID must be replaced with your actual channel ID. Use a test channel before production.                                                                                                                        | Configuration detail for `Post to Slack` node.                    |
| Telegram Bot Token and OpenAI API key credentials must be securely stored and connected in n8n credential manager.                                                                                                         | Security and credential management.                               |
| The language detection is a heuristic based on presence of Japanese characters (Kanji, Hiragana, Katakana). For other languages or edge cases, it defaults to English.                                                     | Language detection logic detail.                                  |
| The GPT-4o-mini model is instructed to output ONLY the translated text to avoid formatting issues in Slack messages.                                                                                                       | Prompt design for translation consistency.                        |
| For more information on n8n Telegram integration: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.telegramTrigger/                                                                                      | Official n8n Telegram node docs.                                  |
| For OpenAI Whisper transcription and GPT models usage, see OpenAI API docs: https://platform.openai.com/docs/api-reference/audio/create-transcription and https://platform.openai.com/docs/models/gpt-4o-mini              | OpenAI API reference.                                             |
| Slack chat.postMessage method docs: https://api.slack.com/methods/chat.postMessage                                                                                                                                         | Slack API documentation.                                          |

---

This document fully describes the structure, logic, and configuration of the **Voice Translator Bridge (Telegram → Slack) with GPT-4o-mini + Whisper** workflow, enabling advanced users and AI agents to understand, reproduce, and extend the automation with confidence.