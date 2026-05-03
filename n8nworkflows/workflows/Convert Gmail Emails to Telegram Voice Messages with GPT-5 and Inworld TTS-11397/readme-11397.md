Convert Gmail Emails to Telegram Voice Messages with GPT-5 and Inworld TTS

https://n8nworkflows.xyz/workflows/convert-gmail-emails-to-telegram-voice-messages-with-gpt-5-and-inworld-tts-11397


# Convert Gmail Emails to Telegram Voice Messages with GPT-5 and Inworld TTS

### 1. Workflow Overview

This workflow automates the conversion of incoming Gmail emails into Telegram voice messages using advanced AI tools. It is designed for users who want to receive concise spoken briefings of their new emails directly in Telegram, enabling audio-based email consumption on the go.

The workflow is logically divided into four main blocks:

- **1.1 Telegram Chat ID Bootstrap:** Captures and stores the Telegram chat ID the first time a user messages the bot, enabling future voice message deliveries to the correct chat.
- **1.2 Email Intake and Preparation:** Listens for new Gmail messages, extracts relevant email metadata and content, and formats it for AI processing.
- **1.3 AI Summary and Text-to-Speech (TTS):** Processes the email text with GPT-5 to generate a short, natural-sounding summary, then converts this summary into speech audio using Inworld TTS.
- **1.4 Audio Delivery to Telegram:** Downloads the generated audio file and sends it as a voice message to the stored Telegram chat ID.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Chat ID Bootstrap

- **Overview:**  
  This block captures the Telegram chat ID the first time the bot receives a message from a user. It stores the chat ID in a Data Table, which is later used to send voice messages to the correct Telegram chat. This is a one-time setup step that ensures subsequent deliveries are routed properly.

- **Nodes Involved:**  
  - When Telegram Message Received  
  - Save Chat ID to Database  
  - Sticky Note — Chat ID Flow

- **Node Details:**

  - **When Telegram Message Received**  
    - *Type:* Telegram Trigger  
    - *Role:* Listens for new Telegram messages sent to the bot.  
    - *Configuration:* Watches for “message” updates only. No additional filters.  
    - *Credentials:* Uses a configured Telegram API credential with OAuth2.  
    - *Inputs:* Incoming Telegram messages from any user.  
    - *Outputs:* Contains the message JSON, including the user's chat ID.  
    - *Potential Failures:* Connection issues, bot permissions missing, webhook misconfiguration.  
    - *Sub-workflow:* None.

  - **Save Chat ID to Database**  
    - *Type:* Data Table (n8n native node)  
    - *Role:* Stores or updates the Telegram chat ID in a persistent Data Table named “Chat id.”  
    - *Configuration:* Upsert operation keyed by `chat_id` field, converts incoming Telegram message chat ID to number and saves it.  
    - *Expressions:* `={{ $json.message.chat.id }}` extracts chat ID from incoming Telegram message.  
    - *Inputs:* Receives Telegram message JSON from trigger node.  
    - *Outputs:* Confirmation of upsert operation.  
    - *Potential Failures:* Data Table access issues, malformed input data, permissions errors.  
    - *Sub-workflow:* None.

  - **Sticky Note — Chat ID Flow**  
    - *Content:* Documents the purpose of this block and nodes involved for clarity.

---

#### 1.2 Email Intake and Preparation

- **Overview:**  
  Detects new Gmail emails every minute and prepares a compact text representation containing sender, subject, date, and snippet. This text is formatted for subsequent AI summarization.

- **Nodes Involved:**  
  - When Email Received  
  - Prepare Email Data  
  - Sticky Note — Step 1 Email

- **Node Details:**

  - **When Email Received**  
    - *Type:* Gmail Trigger  
    - *Role:* Polls Gmail inbox every minute for new incoming emails.  
    - *Configuration:* No filter criteria, polls regularly at one-minute intervals.  
    - *Credentials:* Uses OAuth2 Gmail account.  
    - *Inputs:* Incoming emails from Gmail.  
    - *Outputs:* Email JSON including From, Subject, snippet, and date fields.  
    - *Potential Failures:* Gmail API rate limits, OAuth token expiration, connectivity issues.  
    - *Sub-workflow:* None.

  - **Prepare Email Data**  
    - *Type:* Code (JavaScript)  
    - *Role:* Extracts and formats key email fields into a single text string for AI processing.  
    - *Configuration:* Uses JavaScript to concatenate “From,” “Subject,” “Date,” and “Snippet” fields into a multiline string named `emailData`.  
    - *Expressions:* Accesses input JSON fields, applies defaults if missing (e.g., “Unknown sender”).  
    - *Inputs:* Email JSON from Gmail trigger node.  
    - *Outputs:* JSON with `emailData` property.  
    - *Potential Failures:* Missing fields, unexpected input format, code errors.  
    - *Sub-workflow:* None.

  - **Sticky Note — Step 1 Email**  
    - *Content:* Explains the purpose of this block and the nodes involved.

---

#### 1.3 AI Summary and Voice Generation

- **Overview:**  
  Generates a natural-sounding, concise 2-3 sentence summary of the email using GPT-5, then converts that summary into speech audio using Inworld TTS API.

- **Nodes Involved:**  
  - Generate Summary with AI  
  - Convert Text to Speech  
  - Sticky Note — Step 2 Summary

- **Node Details:**

  - **Generate Summary with AI**  
    - *Type:* AI/ML API (aimlApi node)  
    - *Role:* Uses OpenAI GPT-5 chat model to generate a brief spoken-style summary of the email text.  
    - *Configuration:*  
      - Model: `openai/gpt-5-1-chat-latest`  
      - Prompt template: instructs the model to produce a 2-3 sentence summary in natural spoken English, returning only the summary text.  
      - Input variable: `emailData` from the previous node.  
    - *Credentials:* Uses configured AI/ML account credential.  
    - *Inputs:* Receives formatted email text.  
    - *Outputs:* JSON containing the AI-generated summary text.  
    - *Potential Failures:* API quota exceeded, invalid prompt, network issues, malformed input.  
    - *Sub-workflow:* None.

  - **Convert Text to Speech**  
    - *Type:* HTTP Request  
    - *Role:* Calls Inworld TTS API to convert AI summary text into speech audio.  
    - *Configuration:*  
      - URL: `https://api.aimlapi.com/v1/tts`  
      - Method: POST  
      - Authenticated with API credentials (same AI/ML account)  
      - Body parameters: specifies model `inworld/tts-1-max` and text as the AI summary.  
      - Response: JSON with audio file URL.  
    - *Credentials:* AI/ML account credentials.  
    - *Inputs:* Receives AI summary text.  
    - *Outputs:* JSON containing URL to the generated audio file.  
    - *Potential Failures:* API authentication errors, TTS service downtime, invalid text input, response parsing errors.  
    - *Sub-workflow:* None.

  - **Sticky Note — Step 2 Summary**  
    - *Content:* Describes the AI summarization and voice generation process.

---

#### 1.4 Audio Delivery to Telegram

- **Overview:**  
  Retrieves the stored Telegram chat ID, downloads the TTS audio file, and sends it as an audio message to the Telegram chat.

- **Nodes Involved:**  
  - Get Telegram Chat ID  
  - Download Audio File  
  - Send Audio to Telegram  
  - Sticky Note — Step 4 Get ID & Download

- **Node Details:**

  - **Get Telegram Chat ID**  
    - *Type:* Data Table (n8n native node)  
    - *Role:* Fetches the stored Telegram chat ID from the “Chat id” Data Table for message delivery.  
    - *Configuration:* Simple get operation on the preconfigured Data Table.  
    - *Inputs:* Receives trigger from previous node (TTS conversion).  
    - *Outputs:* JSON with stored `chat_id`.  
    - *Potential Failures:* Data Table access errors, missing chat ID, format mismatches.  
    - *Sub-workflow:* None.

  - **Download Audio File**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the audio file from the TTS URL to a binary buffer for Telegram upload.  
    - *Configuration:*  
      - URL dynamically set to `{{$('Convert Text to Speech').item.json.audio.url}}`  
      - Response type: file (binary response expected)  
    - *Inputs:* Receives TTS response JSON with audio URL.  
    - *Outputs:* Binary data of the audio file.  
    - *Potential Failures:* URL invalid or expired, network timeout, large file size issues.  
    - *Sub-workflow:* None.

  - **Send Audio to Telegram**  
    - *Type:* Telegram node  
    - *Role:* Sends the downloaded audio as a voice message to the Telegram chat.  
    - *Configuration:*  
      - Chat ID supplied from `{{$json.chat_id}}` fetched from Data Table.  
      - Operation: sendAudio with binary data.  
      - Audio title set dynamically from original email sender and subject:  
        `{{$('When Email Received').item.json.From}} | {{$('When Email Received').item.json.Subject}}`  
      - Binary property name set to the downloaded audio data field.  
    - *Credentials:* Telegram API OAuth2 credentials.  
    - *Inputs:* Binary audio data, chat ID.  
    - *Outputs:* Confirmation of message sent.  
    - *Potential Failures:* Invalid chat ID, Telegram API rate limits, file upload issues, network errors.  
    - *Sub-workflow:* None.

  - **Sticky Note — Step 4 Get ID & Download**  
    - *Content:* Explains the Telegram delivery steps and nodes used.

---

### 3. Summary Table

| Node Name                   | Node Type                | Functional Role                                  | Input Node(s)           | Output Node(s)                | Sticky Note                                                                                         |
|-----------------------------|--------------------------|-------------------------------------------------|-------------------------|------------------------------|---------------------------------------------------------------------------------------------------|
| When Telegram Message Received | Telegram Trigger         | Captures Telegram messages to get chat ID       | —                       | Save Chat ID to Database      | ## **💬 Telegram Chat ID Bootstrap (one-time)** Captures your chat_id the first time you message the bot. Nodes: When Telegram Message Received, Save Chat ID to Database. The stored ID is reused by the main Gmail→Voice automation. |
| Save Chat ID to Database     | Data Table               | Stores Telegram chat ID persistently             | When Telegram Message Received | —                        | ## **💬 Telegram Chat ID Bootstrap (one-time)** Captures your chat_id the first time you message the bot. Nodes: When Telegram Message Received, Save Chat ID to Database. The stored ID is reused by the main Gmail→Voice automation. |
| When Email Received          | Gmail Trigger            | Detects new Gmail emails every minute             | —                       | Prepare Email Data            | ## **📥 Email Intake (Gmail)** Groups nodes that detect new emails and format them for AI processing. Nodes: When Email Received, Prepare Email Data. Produces a compact text block containing sender, subject, date, and snippet. |
| Prepare Email Data           | Code                     | Formats email metadata into compact text block   | When Email Received      | Generate Summary with AI      | ## **📥 Email Intake (Gmail)** Groups nodes that detect new emails and format them for AI processing. Nodes: When Email Received, Prepare Email Data. Produces a compact text block containing sender, subject, date, and snippet. |
| Generate Summary with AI     | AI/ML API (aimlApi)      | Creates concise natural summaries of emails      | Prepare Email Data       | Convert Text to Speech        | ## **🧠 AI Summary & Voice Generation** Creates a short natural-sounding summary and converts it to speech. Nodes: Generate Summary with AI, Convert Text to Speech. The output is a downloadable audio file URL. |
| Convert Text to Speech       | HTTP Request             | Converts summary text into speech audio (TTS)    | Generate Summary with AI | Get Telegram Chat ID          | ## **🧠 AI Summary & Voice Generation** Creates a short natural-sounding summary and converts it to speech. Nodes: Generate Summary with AI, Convert Text to Speech. The output is a downloadable audio file URL. |
| Get Telegram Chat ID         | Data Table               | Retrieves stored Telegram chat ID for delivery   | Convert Text to Speech   | Download Audio File           | ## **📤 Telegram Delivery** Downloads the TTS audio file and sends it to Telegram. Nodes: Get Telegram Chat ID, Download Audio File, Send Audio to Telegram. Uses the stored chat_id to deliver a voice message. |
| Download Audio File          | HTTP Request             | Downloads the TTS audio file as binary data      | Get Telegram Chat ID     | Send Audio to Telegram        | ## **📤 Telegram Delivery** Downloads the TTS audio file and sends it to Telegram. Nodes: Get Telegram Chat ID, Download Audio File, Send Audio to Telegram. Uses the stored chat_id to deliver a voice message. |
| Send Audio to Telegram       | Telegram Node            | Sends the audio message to the Telegram chat     | Download Audio File      | —                            | ## **📤 Telegram Delivery** Downloads the TTS audio file and sends it to Telegram. Nodes: Get Telegram Chat ID, Download Audio File, Send Audio to Telegram. Uses the stored chat_id to deliver a voice message. |
| Sticky Note — Chat ID Flow   | Sticky Note              | Documentation for chat ID capture block           | —                       | —                            | See above for full content                                                                        |
| Sticky Note — Step 1 Email   | Sticky Note              | Documentation for email intake block               | —                       | —                            | See above for full content                                                                        |
| Sticky Note — Step 2 Summary | Sticky Note              | Documentation for AI summary and TTS block        | —                       | —                            | See above for full content                                                                        |
| Sticky Note — Step 4 Get ID & Download | Sticky Note      | Documentation for Telegram delivery block          | —                       | —                            | See above for full content                                                                        |
| Sticky Note                  | Sticky Note              | Overall workflow documentation                      | —                       | —                            | ## **📨 Gmail → AI Summary → Inworld TTS → 🎧 Telegram Voice** This workflow turns new Gmail messages into short spoken briefings delivered to Telegram. It works by detecting incoming emails, generating a concise summary using AI/ML API (GPT-5), converting that summary into natural speech using Inworld TTS-1-Max, and finally sending the audio file to a Telegram chat. The process includes four parts: 1. Gmail polling and preparing a clean text representation of the email. 2. AI-generated 2–3 sentence summary suitable for audio playback. 3. Text-to-speech conversion to produce an audio file. 4. Delivery to a saved Telegram chat ID. A small helper flow captures your Telegram chat_id once and stores it in a Data Table. After that, every new email automatically arrives as a Telegram voice message — making inbox updates easy to listen to on the go. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Chat ID Bootstrap Flow:**

   - Add **Telegram Trigger** node:
     - Name: `When Telegram Message Received`
     - Set updates to `message`
     - Configure Telegram API credentials.
   - Add **Data Table** node:
     - Name: `Save Chat ID to Database`
     - Operation: `upsert`
     - Data Table: Create or use a Data Table named “Chat id” with a `chat_id` column of type number.
     - Mapping: Map `chat_id` to expression `{{$json.message.chat.id}}`
   - Connect the Telegram Trigger node’s output to the Data Table node’s input.

2. **Create Email Intake & Preparation Flow:**

   - Add **Gmail Trigger** node:
     - Name: `When Email Received`
     - Poll interval: every minute
     - Configure Gmail OAuth2 credentials.
   - Add **Code** node:
     - Name: `Prepare Email Data`
     - JavaScript code to extract and format email fields as:

       ```javascript
       const item = $input.first();
       const from = item.json.From || 'Unknown sender';
       const subject = item.json.Subject || 'No subject';
       const snippet = item.json.snippet || '';
       const date = item.json.date || '';

       const emailData = `From: ${from}\nSubject: ${subject}\nDate: ${date}\nSnippet: ${snippet}`;

       return { json: { emailData } };
       ```
   - Connect Gmail Trigger node to Code node.

3. **Create AI Summary & Text-to-Speech Flow:**

   - Add **AI/ML API (aimlApi)** node:
     - Name: `Generate Summary with AI`
     - Model: `openai/gpt-5-1-chat-latest`
     - Prompt:

       ```
       You are a voice assistant that creates brief, natural-sounding email summaries for audio playback. Create a concise summary (2-3 sentences max) in English. Make it sound natural when spoken aloud. Provide ONLY the summary text, nothing else.

       Received email:  
       {{$json.emailData}}
       ```
     - Configure AI/ML credentials.
   - Add **HTTP Request** node:
     - Name: `Convert Text to Speech`
     - Method: POST
     - URL: `https://api.aimlapi.com/v1/tts`
     - Authentication: Use the same AI/ML credentials.
     - Body parameters (JSON or form data):  
       - `model`: `inworld/tts-1-max`  
       - `text`: `{{$json.content}}` (assuming AI response property is `content`)
     - Set response format to JSON.
   - Connect `Prepare Email Data` → `Generate Summary with AI` → `Convert Text to Speech`.

4. **Create Audio Delivery to Telegram Flow:**

   - Add **Data Table** node:
     - Name: `Get Telegram Chat ID`
     - Operation: `get`
     - Data Table: “Chat id”
   - Add **HTTP Request** node:
     - Name: `Download Audio File`
     - Method: GET
     - URL: `={{ $('Convert Text to Speech').item.json.audio.url }}`
     - Response format: File (binary)
   - Add **Telegram** node:
     - Name: `Send Audio to Telegram`
     - Operation: `sendAudio`
     - Chat ID: `={{ $json.chat_id }}`
     - Binary property name: `data` (binary data from Download Audio File node)
     - Additional Fields > Title:  
       `={{ $('When Email Received').item.json.From }} | {{ $('When Email Received').item.json.Subject }}`
     - Configure Telegram credentials.
   - Connect nodes: `Convert Text to Speech` → `Get Telegram Chat ID` → `Download Audio File` → `Send Audio to Telegram`

5. **Add Sticky Notes for Documentation:**

   - Add sticky notes near each logical block describing their purpose, as documented in the workflow overview.

6. **Final Checks:**

   - Verify all credentials (Gmail OAuth2, AI/ML API key, Telegram OAuth2) are properly configured and authorized.
   - Confirm Data Table “Chat id” exists with correct schema.
   - Test Telegram bot permissions to receive messages and send audio.
   - Test Gmail API quota and polling.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses GPT-5 chat model via the aimlApi node for generating natural, concise summaries optimized for spoken playback.                                                                                                                                                                                                                          | AI/ML API node configuration                                                                                                               |
| The Inworld TTS model `inworld/tts-1-max` produces maximum-quality voice synthesis suitable for Telegram voice messages.                                                                                                                                                                                                                                   | Inworld TTS API documentation                                                                                                              |
| Telegram chat ID capturing is a one-time bootstrap step requiring the user to send any message to the bot first. Without this, voice messages cannot be delivered.                                                                                                                                                                                        | Sticky Note — Chat ID Flow                                                                                                                  |
| Gmail polling is set to every minute, but this can be adjusted to reduce API usage or match user needs.                                                                                                                                                                                                                                                     | Gmail Trigger node configuration                                                                                                           |
| For troubleshooting Telegram audio delivery, ensure the bot has `sendAudio` permissions and the chat ID is valid.                                                                                                                                                                                                                                         | Telegram Bot API documentation                                                                                                             |
| The workflow is designed to minimize user interaction after setup, providing hands-free email summaries via Telegram voice messages.                                                                                                                                                                                                                       | Overall workflow design                                                                                                                     |
| Blog post describing this workflow and use case: [https://n8n.io/blog/gmail-to-telegram-voice-notifications](https://n8n.io/blog/gmail-to-telegram-voice-notifications)                                                                                                                                                                                     | External resource                                                                                                                           |
| Video walkthrough of similar workflows for automation use cases can be found at the official n8n YouTube channel: [https://www.youtube.com/c/n8n-io](https://www.youtube.com/c/n8n-io)                                                                                                                                                                       | n8n official YouTube channel                                                                                                               |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n automated workflow. It adheres strictly to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.