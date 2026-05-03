Convert Telegram Voice Messages to Google Docs with Whisper & GPT-4o Tagging

https://n8nworkflows.xyz/workflows/convert-telegram-voice-messages-to-google-docs-with-whisper---gpt-4o-tagging-8117


# Convert Telegram Voice Messages to Google Docs with Whisper & GPT-4o Tagging

### 1. Workflow Overview

This workflow automates the conversion of Telegram voice messages into formatted Google Docs entries with keyword tagging powered by OpenAI Whisper and GPT-4o. It is designed for users who want to archive voice or text notes received via Telegram into a structured Google Document, enriched with metadata such as timestamps, message type, and user-specific keywords.

Logical blocks:

- **1.1 Input Reception & Audio Detection:** Receives Telegram messages, detects if the message contains a voice note or text.
- **1.2 Audio Retrieval & Transcription:** Downloads the audio file if present and transcribes it using OpenAI Whisper.
- **1.3 Text Preparation & Keyword Tagging:** Prepares the original or transcribed text and generates up to three keywords using GPT-4o.
- **1.4 Formatting & Google Docs Integration:** Formats the enriched text and inserts it into a specified Google Docs document.
- **1.5 User Confirmation:** Sends a confirmation message back to the Telegram user indicating successful saving.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Audio Detection

**Overview:**  
This block captures incoming Telegram messages, determines whether each message contains a voice note or plain text, and branches accordingly.

**Nodes Involved:**  
- Telegram Sprachnachricht Empfang (Telegram Trigger)  
- Check if Audio file (IF Node)

**Node Details:**

- **Telegram Sprachnachricht Empfang**  
  - Type: Telegram Trigger  
  - Role: Listens for new Telegram messages (updates of type "message"). Configured to download attachments automatically.  
  - Key Config: `updates=["message"]`, `download=true`  
  - Credentials: Telegram API account  
  - Inputs: HTTP webhook (Telegram)  
  - Outputs: Emits the Telegram message JSON  
  - Edge cases: Missing download could cause audio unavailability; network or webhook errors possible.

- **Check if Audio file**  
  - Type: IF (Condition) Node  
  - Role: Checks if the incoming Telegram message contains a voice note by verifying the presence of the `message.voice` field.  
  - Condition: `message.voice` is not empty  
  - Inputs: Output of Telegram Trigger  
  - Outputs: Two branches – 'true' if voice note exists, 'false' if not  
  - Edge cases: Messages with no voice or text will fail the condition; malformed message JSON may cause expression errors.

---

#### 1.2 Audio Retrieval & Transcription

**Overview:**  
If the message contains a voice note, this block downloads the audio file and transcribes it with OpenAI Whisper.

**Nodes Involved:**  
- Get a file (Telegram node)  
- OpenAI Whisper Transkription (OpenAI node)

**Node Details:**

- **Get a file**  
  - Type: Telegram node (file resource)  
  - Role: Downloads the voice audio file using its Telegram `file_id`.  
  - Parameters: Uses expression to extract `file_id` from the Telegram message's voice object.  
  - Credentials: Telegram API account  
  - Input: True branch from IF node  
  - Output: Binary audio file data suitable for transcription  
  - Edge cases: File download failure, file ID missing or expired, Telegram API limits.

- **OpenAI Whisper Transkription**  
  - Type: OpenAI node (LangChain integration)  
  - Role: Performs audio transcription using Whisper with German language preset.  
  - Configuration: Operation set to "transcribe", resource "audio", language set to "de" (German).  
  - Credentials: OpenAI API account  
  - Input: Audio binary from 'Get a file' node  
  - Output: Transcribed text in JSON field `text`  
  - Edge cases: API rate limits, audio format unsupported, transcription inaccuracies.

---

#### 1.3 Text Preparation & Keyword Tagging

**Overview:**  
Prepares the text from either the transcribed audio or the original text message. Sends this text to GPT-4o to generate up to three keywords and correct minor transcription errors related to specific terms (e.g., "Claude").

**Nodes Involved:**  
- Set field (to prepare text from Telegram text messages)  
- Message a model (OpenAI GPT-4o)  
- Text formatieren (Function node)

**Node Details:**

- **Set field**  
  - Type: Set node  
  - Role: Assigns the `text` field with the Telegram message text for non-voice messages.  
  - Parameters: `text` = `message.text`  
  - Input: False branch from IF node (non-audio message)  
  - Output: JSON with `text` field ready for keyword extraction  
  - Edge cases: Missing `text` field in message; expression errors.

- **Message a model**  
  - Type: OpenAI node (LangChain chat)  
  - Role: Sends the text to GPT-4o model to generate descriptive keywords (max 3) and correct specific transcription errors.  
  - Configuration: Model ID set to `chatgpt-4o-latest`.  
  - Message prompt includes instructions for keyword extraction, examples, and minor text adjustments focusing on "Claude" recognition errors.  
  - Credentials: OpenAI API account  
  - Input: Output from either Whisper transcription or Set field node  
  - Output: GPT-generated keywords and (possibly) corrected text  
  - Edge cases: API limits, prompt misunderstanding, generation of irrelevant keywords.

- **Text formatieren**  
  - Type: Function node (JavaScript)  
  - Role: Formats the final output text including timestamp (Swiss timezone), message type (voice or text), user name, keywords, and original or transcribed text into a structured string.  
  - Key expressions:  
    - Extracts message date, user first name, keywords from GPT result.  
    - Detects message type (voice/text).  
    - Formats date to German (Switzerland) locale with timezone Europe/Zurich.  
    - Constructs a formatted string with emojis and labels.  
  - Input: Output from 'Message a model' node  
  - Output: JSON with `formattedText`, `originalText`, `keywords`, `timestamp`, `userName`, `messageType`  
  - Edge cases: Missing or malformed input fields, date conversion errors.

---

#### 1.4 Formatting & Google Docs Integration

**Overview:**  
This block inserts the formatted text into a predefined Google Docs document, appending the new content.

**Nodes Involved:**  
- In Google Doc speichern (Google Docs node)

**Node Details:**

- **In Google Doc speichern**  
  - Type: Google Docs node  
  - Role: Inserts the formatted text into a specific Google Docs document by appending it to the document content.  
  - Parameters:  
    - Operation: `update`  
    - Action: Insert text (uses expression to insert `formattedText` from the function node)  
    - Document URL: Hardcoded to a specific Google Docs document  
  - Credentials: Google Docs OAuth2 account  
  - Input: Output from 'Text formatieren' node  
  - Output: Confirmation of document update  
  - Edge cases: OAuth token expiration, document access permission errors, rate limits.

---

#### 1.5 User Confirmation

**Overview:**  
Sends a confirmation message back to the Telegram chat that the message was successfully saved, including the original text snippet.

**Nodes Involved:**  
- Bestätigung senden (Telegram node)

**Node Details:**

- **Bestätigung senden**  
  - Type: Telegram node (send message)  
  - Role: Sends a Telegram message confirming the successful save with a snippet of the original message text.  
  - Parameters:  
    - Text: Static confirmation text with embedded original text snippet using expression.  
    - Chat ID: Extracted dynamically from the original Telegram message chat ID.  
  - Credentials: Telegram API account  
  - Input: Output from 'In Google Doc speichern' node  
  - Output: Confirmation message sent  
  - Edge cases: Chat ID missing, Telegram API errors, rate limits.

---

### 3. Summary Table

| Node Name                   | Node Type                           | Functional Role                     | Input Node(s)                 | Output Node(s)               | Sticky Note                                           |
|-----------------------------|-----------------------------------|-----------------------------------|------------------------------|-----------------------------|-------------------------------------------------------|
| Telegram Sprachnachricht Empfang | Telegram Trigger                  | Receive Telegram messages          | (Webhook)                    | Check if Audio file          |                                                       |
| Check if Audio file          | IF Node                           | Determine if message has voice     | Telegram Sprachnachricht Empfang | Get a file (true branch), Set field (false branch) |                                                       |
| Get a file                  | Telegram Node (file resource)      | Download voice audio file          | Check if Audio file (true)    | OpenAI Whisper Transkription |                                                       |
| OpenAI Whisper Transkription | OpenAI node (Whisper transcription) | Transcribe audio to text           | Get a file                   | Message a model             |                                                       |
| Set field                   | Set Node                         | Prepare text for non-voice messages | Check if Audio file (false)   | Message a model             |                                                       |
| Message a model             | OpenAI node (GPT-4o chat)          | Generate keywords and correct text | OpenAI Whisper Transkription OR Set field | Text formatieren          |                                                       |
| Text formatieren            | Function Node                    | Format final text with metadata    | Message a model              | In Google Doc speichern      |                                                       |
| In Google Doc speichern     | Google Docs Node                 | Insert formatted text into Google Doc | Text formatieren             | Bestätigung senden          |                                                       |
| Bestätigung senden          | Telegram Node (send message)       | Send confirmation back to Telegram | In Google Doc speichern      |                             |                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Name: `Telegram Sprachnachricht Empfang`  
   - Type: Telegram Trigger  
   - Set `Updates` to listen for `"message"`  
   - Enable `Download` in additional fields to retrieve media files automatically  
   - Connect with valid Telegram API credentials  
   - Position: Left side (e.g., X: -480, Y: -192)

2. **Add IF Node to Check for Audio**  
   - Name: `Check if Audio file`  
   - Type: IF  
   - Condition: Check if `message.voice` is not empty (`notEmpty` operator)  
   - Connect input from Telegram Trigger  
   - Two outputs: True branch (voice message), False branch (no voice)

3. **Configure Audio Download Node (Telegram)**  
   - Name: `Get a file`  
   - Type: Telegram node (file resource)  
   - Parameter: Set `fileId` to expression `{{$json.message.voice.file_id}}`  
   - Connect input from IF Node true branch  
   - Use same Telegram credentials

4. **Configure OpenAI Whisper Transcription Node**  
   - Name: `OpenAI Whisper Transkription`  
   - Type: OpenAI node (LangChain)  
   - Operation: `transcribe`  
   - Resource: `audio`  
   - Options: Set `language` to `"de"` (German)  
   - Connect input from `Get a file` node output  
   - Add OpenAI credentials

5. **Configure Set Node for Text Messages**  
   - Name: `Set field`  
   - Type: Set node  
   - Add assignment: `text` = `{{$json.message.text}}` (expression from Telegram message)  
   - Connect input from IF Node false branch (no voice)  

6. **Configure OpenAI GPT-4o Node for Keyword Extraction**  
   - Name: `Message a model`  
   - Type: OpenAI LangChain node (chat)  
   - Model ID: `chatgpt-4o-latest` (select from list)  
   - Message prompt:  
     ```
     Du erhältst den Text einer Nachricht von Julian Reich. Deine Aufgabe ist es, den Text mit maximal drei Schlagworten zusammenzufassen. Beispiele: Arbeit, Idee, Privat, Sport, Ernährung, Schlaf, KI, Projekt, Effizienz, Problem.
     Mache minimale Anpassungen, insbesondere wenn der Begriff "Claude" falsch erkannt wird. Halte dich mit Eingriffen zurück.
     
     Hier der Text: {{ $json.text }}
     ```  
   - Connect input from both `OpenAI Whisper Transkription` and `Set field` nodes (merge logic ensures correct input)  
   - Use OpenAI credentials

7. **Add Function Node to Format Text**  
   - Name: `Text formatieren`  
   - Type: Function node  
   - Insert the provided JavaScript code that:  
     - Extracts date/time from Telegram message, formats it to Swiss German locale  
     - Determines if message is voice or text  
     - Combines keywords, original text, user name into a structured format with emojis  
   - Connect input from `Message a model` node output

8. **Add Google Docs Node to Insert Text**  
   - Name: `In Google Doc speichern`  
   - Type: Google Docs node  
   - Operation: `update`  
   - Action: Insert text with field `={{ $json.formattedText }}`  
   - Document URL: Use your Google Doc sharing/edit URL  
   - Connect input from `Text formatieren` node output  
   - Use valid Google Docs OAuth2 credentials

9. **Add Telegram Node to Send Confirmation**  
   - Name: `Bestätigung senden`  
   - Type: Telegram node (send message)  
   - Text template:  
     ```
     ✅ Nachricht erfolgreich gespeichert!

     📝 Text:

     {{ $('Text formatieren').first().json.originalText }}
     ```  
   - Chat ID: Expression `={{ $('Telegram Sprachnachricht Empfang').item.json.message.chat.id }}`  
   - Connect input from `In Google Doc speichern` node output  
   - Use Telegram API credentials

10. **Connect Nodes According to Logic:**  
    - Telegram Trigger → Check if Audio file  
    - IF true → Get a file → OpenAI Whisper → Message a model  
    - IF false → Set field → Message a model  
    - Message a model → Text formatieren → In Google Doc speichern → Bestätigung senden

11. **Verify Credentials:**  
    - Telegram API account with webhook setup  
    - OpenAI API key with access to Whisper and GPT-4o  
    - Google Docs OAuth2 credentials with edit permissions on the target document

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The workflow handles both voice and text Telegram messages gracefully, converting voice notes via Whisper. | Core functionality of the workflow                                                                        |
| Whisper transcription is set to German language to match typical user context and improve accuracy.      | Node: OpenAI Whisper Transkription                                                                        |
| GPT-4o is instructed to extract up to 3 keywords and fix common transcription errors around "Claude".    | Node: Message a model                                                                                      |
| Date formatting uses Swiss German locale and Europe/Zurich timezone for regional accuracy.               | Node: Text formatieren                                                                                     |
| Google Docs node appends text to an existing document via URL, requiring appropriate access rights.      | Node: In Google Doc speichern                                                                              |
| Confirmation messages help user feedback via Telegram after successful processing.                        | Node: Bestätigung senden                                                                                   |
| For Telegram API, ensure webhook URL is reachable and credentials are properly set to avoid errors.      | Telegram Sprachnachricht Empfang and Telegram nodes                                                        |
| OpenAI API quota and rate limits may affect transcription and keyword generation speed or success.      | OpenAI Whisper Transkription and Message a model nodes                                                    |
| Google Docs API token expiration may interrupt document update; consider token refresh handling.         | In Google Doc speichern                                                                                     |

---

**Disclaimer:** The content here is derived exclusively from an n8n automated workflow. All data handled complies with legal and public data policies. No illegal, offensive, or protected content is involved.