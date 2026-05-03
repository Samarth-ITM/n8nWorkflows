Clone and change your voice with Elevenlabs and Telegram

https://n8nworkflows.xyz/workflows/clone-and-change-your-voice-with-elevenlabs-and-telegram-11606


# Clone and change your voice with Elevenlabs and Telegram

### 1. Workflow Overview

This n8n workflow enables users to clone their voice or change their voice using ElevenLabs' AI voice synthesis via Telegram voice messages. It targets users who want to interact with ElevenLabs' voice cloning and speech-to-speech conversion services seamlessly through Telegram. The workflow is divided into logical blocks grouped by functional roles:

- **1.1 Input Reception and Authorization:** Listens for Telegram messages and restricts access to an authorized Telegram user.
- **1.2 Message Type Routing:** Differentiates between text, audio (voice) messages, and images, routing them accordingly.
- **1.3 Voice Cloning Process:** Upon receiving a voice message, uploads the audio to ElevenLabs to create a cloned voice profile and retrieve a unique Voice ID.
- **1.4 Voice Changing Process:** Converts an existing voice audio message using a pre-existing cloned voice from ElevenLabs and outputs the transformed audio.
- **1.5 File Management:** Uploads the generated or cloned audio file to Google Drive for storage and retrieval.
- **1.6 User Guidance and Documentation:** Provides sticky notes with instructions and relevant API key setup information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Authorization

- **Overview:** Listens for incoming Telegram messages, restricts access to a specific Telegram user ID to ensure authorized usage.
- **Nodes Involved:** Telegram Trigger, Sanitaze, Sticky Note (authorization instruction)
  
- **Telegram Trigger**
  - **Type & Role:** Telegram Trigger node; initiates workflow on new Telegram messages.
  - **Configuration:** Listens for all message updates from Telegram.
  - **Credentials:** Uses configured Telegram API credentials.
  - **Connections:** Outputs to Sanitaze node.
  - **Failure modes:** Possible webhook misconfiguration, Telegram API downtime, invalid credentials.
  
- **Sanitaze**
  - **Type & Role:** Code node; filters messages to allow only those from an authorized Telegram user ID.
  - **Configuration:** JavaScript code checks if the sender's Telegram user ID matches a hardcoded ID (placeholder `XXX` to replace).
  - **Key Expressions:** `$input.first().json.message.from.id !== XXX`
  - **Connections:** Outputs to Switch node if authorized; returns an object with `unauthorized: true` if not.
  - **Failure modes:** Missing or incorrect user ID leads to rejection; expression errors if input data structure changes.
  
- **Sticky Note (Authorization Instruction)**
  - **Content:** Instruction to replace `XXX` with the authorized Telegram user ID for access control.

#### 2.2 Message Type Routing

- **Overview:** Differentiates incoming messages by content type (text, voice audio, image) to route them to the appropriate processing branch.
- **Nodes Involved:** Switch
  
- **Switch**
  - **Type & Role:** Switch node; evaluates message content to identify if it contains text, voice audio, or images.
  - **Configuration:** 
    - Routes to “Text” if message.text exists.
    - Routes to “Audio” if message.voice.file_id exists.
    - Routes to “Immagine” if message.photo[0] exists.
  - **Connections:** 
    - “Audio” output connects to “Get audio” node.
    - “Text” and “Immagine” outputs currently have no downstream connections.
  - **Failure modes:** If message structure changes, conditions might fail; unrecognized message types are ignored.
  - **Version:** Uses Switch node version 3.2 with updated condition syntax.

#### 2.3 Voice Cloning Process

- **Overview:** Downloads a voice message from Telegram, then sends it to ElevenLabs to create and save a cloned voice profile, generating a unique Voice ID.
- **Nodes Involved:** Get audio, Create Cloned Voice, Sticky Notes (cloning instructions and voice ID example)
  
- **Get audio**
  - **Type & Role:** Telegram node; downloads the voice message file using the file ID.
  - **Configuration:** Uses `message.voice.file_id` from incoming JSON.
  - **Credentials:** Uses Telegram API credentials.
  - **Connections:** Outputs downloaded audio data to “Create Cloned Voice” node.
  - **Failure modes:** Invalid or expired file ID, Telegram file download errors.
  
- **Create Cloned Voice**
  - **Type & Role:** HTTP Request node; sends audio file to ElevenLabs API to create a new voice clone.
  - **Configuration:** 
    - URL: `https://api.elevenlabs.io/v1/voices/add`
    - Method: POST
    - Content-Type: `multipart/form-data`
    - Body Parameters: `name` (voice name, placeholder `XXX`), and the audio file as binary form data.
    - Authentication: Header Auth with ElevenLabs API key.
  - **Credentials:** ElevenLabs API credentials with header name `xi-api-key`.
  - **Connections:** No downstream connection (could be extended).
  - **Failure modes:** API key issues, API limits, invalid audio format, network errors.
  
- **Sticky Note (Cloning Instructions)**
  - **Content:** Instructions to create ElevenLabs API key and set header authentication; explains voice cloning steps.
  
- **Sticky Note8 (Voice ID Example)**
  - **Content:** JSON snippet showing an example Voice ID and verification requirement flag.

#### 2.4 Voice Changing Process

- **Overview:** Transforms an existing audio voice message using a pre-existing cloned voice from ElevenLabs and produces a modified audio file.
- **Nodes Involved:** Generate cloned audio, Upload file, Sticky Note (voice changer instructions)
  
- **Generate cloned audio**
  - **Type & Role:** HTTP Request node; sends the audio file to ElevenLabs speech-to-speech API to synthesize voice change.
  - **Configuration:** 
    - URL: `https://api.elevenlabs.io/v1/speech-to-speech/XXX` (voice ID placeholder)
    - Method: POST
    - Content-Type: `multipart/form-data`
    - Body: audio file binary data.
    - Query parameter: `output_format=mp3_44100_128`
    - Authentication: Header Auth with ElevenLabs API key.
  - **Credentials:** ElevenLabs API credentials.
  - **Connections:** Outputs converted audio binary to “Upload file”.
  - **Failure modes:** Invalid voice ID, API rate limits, audio format issues, network errors.
  
- **Upload file**
  - **Type & Role:** Google Drive node; uploads the transformed audio file to a specified Google Drive folder.
  - **Configuration:** 
    - Filename: prefixed with “cloned_” plus original filename extracted via regex from ElevenLabs response.
    - Folder ID: preconfigured Google Drive folder for storing files.
  - **Credentials:** Google Drive OAuth2 credentials.
  - **Failure modes:** Google Drive quota issues, OAuth token expiration, misconfigured folder permissions.
  
- **Sticky Note2 (Voice Changer Instructions)**
  - **Content:** Instructions to create ElevenLabs API key, set header authentication, and connect audio node for voice change.

#### 2.5 User Guidance and Documentation

- **Overview:** Sticky notes provide contextual instructions and overall workflow description for users setting up or maintaining the workflow.
- **Nodes Involved:** Sticky Note4, Sticky Note1, Sticky Note2, Sticky Note3
  
- **Sticky Note4**
  - **Content:** High-level description of the workflow purpose, setup instructions for Telegram and ElevenLabs, and usage guidelines.
  
- **Sticky Note1**
  - **Content:** Step-by-step instructions for voice cloning option including API key creation and configuration.
  
- **Sticky Note3**
  - **Content:** Reminder to get audio files from Telegram when sending audio.
  
- **Sticky Note2**
  - **Content:** Instructions for voice changer option including API key setup and connection hints.

---

### 3. Summary Table

| Node Name         | Node Type           | Functional Role                     | Input Node(s)     | Output Node(s)       | Sticky Note                                                                                             |
|-------------------|---------------------|-----------------------------------|-------------------|----------------------|-------------------------------------------------------------------------------------------------------|
| Telegram Trigger  | Telegram Trigger     | Input reception from Telegram     | -                 | Sanitaze             |                                                                                                       |
| Sanitaze          | Code                 | Authorization filter by user ID   | Telegram Trigger  | Switch               | ## STEP 1 - Sanitaze Replace XXXX with your Telegram user ID                                          |
| Switch            | Switch               | Message type routing              | Sanitaze          | Get audio (Audio)    |                                                                                                       |
| Get audio         | Telegram              | Download voice audio file          | Switch (Audio)    | Create Cloned Voice   | ## STEP 2 - Get audio file IF you send an audio file from Telegram, get it                            |
| Create Cloned Voice| HTTP Request         | Send audio to ElevenLabs to clone | Get audio         | -                    | ## OPTION 1  - CLONE YOUR VOICE Clone and save your voice on ElevenLabs by sending a voice message... |
| Generate cloned audio | HTTP Request       | Transform voice with ElevenLabs   | - (No direct trigger in JSON) | Upload file    | ## OPTION 2 - VOICE CHANGER Change your voice to one from your ElevenLabs library and save audio      |
| Upload file       | Google Drive          | Upload transformed audio to Drive | Generate cloned audio | -                   |                                                                                                       |
| Sticky Note8      | Sticky Note           | Voice ID example JSON             | -                 | -                    | ```json [ { "voice_id": "XXXXXX", "requires_verification": false } ] ```                              |
| Sticky Note       | Sticky Note           | Authorization instruction         | -                 | -                    | ## STEP 1 - Sanitaze Replace XXXX with your Telegram user ID                                          |
| Sticky Note1      | Sticky Note           | Cloning instructions              | -                 | -                    | ## OPTION 1  - CLONE YOUR VOICE Clone and save your voice on ElevenLabs by sending a voice message... |
| Sticky Note2      | Sticky Note           | Voice changer instructions        | -                 | -                    | ## OPTION 2 - VOICE CHANGER Change your voice to one from your ElevenLabs library and save audio      |
| Sticky Note3      | Sticky Note           | Audio retrieval reminder          | -                 | -                    | ## STEP 2 - Get audio file IF you send an audio file from Telegram, get it                            |
| Sticky Note4      | Sticky Note           | Workflow overview and setup guide | -                 | -                    | ## Clone and change your voice with Elevenlabs and Telegram ... (full description and setup steps)    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credential**
   - Use BotFather to create Telegram bot.
   - In n8n, create Telegram API credential with bot token.
   
2. **Add Telegram Trigger Node**
   - Set to listen for `message` updates.
   - Assign created Telegram API credential.
   - Position: Start of workflow.
   
3. **Add Code Node "Sanitaze"**
   - Add JavaScript code:
     ```js
     if ($input.first().json.message.from.id !== XXX) { // Replace XXX with your Telegram user ID
         return { unauthorized: true };
     } else {
         return $input.all();
     }
     ```
   - Connect Telegram Trigger output to Sanitaze input.
   
4. **Add Switch Node**
   - Configure 3 outputs:
     - Text: message.text exists
     - Audio: message.voice.file_id exists
     - Image: message.photo[0] exists
   - Connect Sanitaze output to Switch input.
   
5. **Add Telegram Node "Get audio"**
   - Resource: file
   - File ID: `={{ $json.message.voice.file_id }}`
   - Assign Telegram API credential.
   - Connect Switch node Audio output to Get audio input.
   
6. **Add HTTP Request Node "Create Cloned Voice"**
   - Method: POST
   - URL: `https://api.elevenlabs.io/v1/voices/add`
   - Authentication: HTTP Header Auth
     - Header Name: `xi-api-key`
     - Header Value: Your ElevenLabs API Key
   - Content-Type: multipart/form-data
   - Body Parameters:
     - name: Your voice name
     - files: map to binary data from Get audio node
   - Connect Get audio output to Create Cloned Voice input.
   
7. **Add HTTP Request Node "Generate cloned audio"** (For voice changing)
   - Method: POST
   - URL: `https://api.elevenlabs.io/v1/speech-to-speech/VOICE_ID` (Replace VOICE_ID with your ElevenLabs voice ID)
   - Authentication: HTTP Header Auth as above
   - Content-Type: multipart/form-data
   - Query Parameter: output_format=mp3_44100_128
   - Body Parameters:
     - audio: binary data input
   - Connect audio source to this node as required.
   
8. **Add Google Drive Node "Upload file"**
   - Operation: Upload file
   - Name: `=cloned_{{ $json.result.file_path.match(/[^/]+$/)[0] }}` (extracts filename)
   - Folder ID: your Google Drive folder ID for uploads
   - Assign Google Drive OAuth2 credentials.
   - Connect Generate cloned audio output to Upload file input.
   
9. **Add Sticky Notes at relevant points**
   - Authorization instructions near Sanitaze node.
   - Cloning and voice changer instructions near respective HTTP Request nodes.
   - Overall workflow description at the start or a central location.
   
10. **Test Workflow**
    - Send a voice message from your authorized Telegram account.
    - Verify cloning or voice changing works, files upload to Google Drive.
    - Adjust voice ID and user ID placeholders as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow creates a voice AI assistant accessible via Telegram leveraging ElevenLabs voice synthesis technology.             | Sticky Note4 content                                                                            |
| ElevenLabs Developer Portal for API Key creation: https://try.elevenlabs.io/ahkbf00hocnu                                           | Mentioned in Sticky Note1 and Sticky Note2                                                     |
| Replace `XXX` placeholders with your Telegram user ID and desired voice names/IDs for proper authorization and API calls.       | Highlighted in Sanitaze code node and HTTP Request nodes                                       |
| Google Drive OAuth2 credentials must be configured with access to the target folder for audio uploads.                           | Upload file node configuration                                                                 |
| Voice ID example JSON snippet for reference: `[{ "voice_id": "XXXXXX", "requires_verification": false }]`                        | Sticky Note8                                                                                   |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.