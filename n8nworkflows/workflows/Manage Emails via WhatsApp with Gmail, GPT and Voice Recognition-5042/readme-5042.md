Manage Emails via WhatsApp with Gmail, GPT and Voice Recognition

https://n8nworkflows.xyz/workflows/manage-emails-via-whatsapp-with-gmail--gpt-and-voice-recognition-5042


# Manage Emails via WhatsApp with Gmail, GPT and Voice Recognition

### 1. Workflow Overview

This workflow, titled **"SmartMail Agent – Your AI Email Assistant, Powered by WhatsApp"**, orchestrates a seamless integration between WhatsApp messages, AI-driven email management, and voice recognition technologies. It enables users to manage email-related tasks through WhatsApp, leveraging GPT models for natural language understanding and OpenAI Whisper for voice transcription.

**Target Use Cases:**  
- Receive WhatsApp messages (voice or text) instructing email actions.  
- Transcribe voice messages to text automatically.  
- Understand user intent via AI to send, draft, or reply to emails.  
- Retrieve contact email information from Airtable.  
- Send or draft emails via Gmail.  
- Provide smart feedback back to the user on WhatsApp, including optional voice responses.

**Logical Blocks:**

- **1.1 WhatsApp Message Intake & Voice Transcription:** Captures incoming WhatsApp messages, distinguishes between text and audio, downloads and transcribes audio messages to text for further processing.

- **1.2 Smart Email Agent – Intent Analysis & Email Handling:** Uses AI to interpret the message, accesses contact data, drafts or sends emails via Gmail, and optionally stores session context.

- **1.3 Vocal Response & WhatsApp Feedback:** Generates confirmation or summary messages using AI, optionally converts text to speech, adjusts audio format for WhatsApp compatibility, and sends the response back via WhatsApp.

---

### 2. Block-by-Block Analysis

#### 1.1 WhatsApp Message Intake & Voice Transcription

**Overview:**  
This block listens for WhatsApp messages, splits batch messages if any, identifies message type (audio or text), retrieves audio content if needed, transcribes audio to text using OpenAI Whisper, and formats the output for further AI processing.

**Nodes Involved:**  
- WhatsApp Trigger  
- Split Out  
- Switch  
- WhatsApp Business Cloud  
- HTTP Request  
- OpenAI (Whisper)  
- Edit Fields  

**Node Details:**

- **WhatsApp Trigger**  
  - *Type:* Trigger node for WhatsApp messages  
  - *Role:* Listens to incoming WhatsApp messages in real-time.  
  - *Config:* Listens to "messages" update event; uses WhatsApp OAuth2 credentials.  
  - *Inputs:* Webhook from WhatsApp Business Cloud API  
  - *Outputs:* Raw incoming message JSON  
  - *Failure Modes:* Webhook misconfiguration, OAuth token expiry, WhatsApp API downtime.

- **Split Out**  
  - *Type:* Utility node  
  - *Role:* Splits multiple incoming messages in one payload to process individually.  
  - *Config:* Splits based on JSON field containing messages array.  
  - *Inputs:* WhatsApp Trigger output  
  - *Outputs:* Individual message items  
  - *Failure Modes:* Incorrect message structure, empty payload.

- **Switch**  
  - *Type:* Router node  
  - *Role:* Routes flow based on message type (audio or text).  
  - *Config:* Checks `messages[0].type` property; outputs to 'voice' or 'text' branches.  
  - *Inputs:* Split Out output  
  - *Outputs:* Branches for audio or text processing  
  - *Failure Modes:* Unexpected message types, empty or malformed messages.

- **WhatsApp Business Cloud (mediaUrlGet)**  
  - *Type:* Media retrieval node  
  - *Role:* If message is audio, retrieves media URL using media ID.  
  - *Inputs:* Switch node audio branch output  
  - *Outputs:* Media URL JSON  
  - *Failure Modes:* Media ID invalid, API rate limits, permissions.

- **HTTP Request**  
  - *Type:* HTTP client node  
  - *Role:* Downloads audio file from WhatsApp media URL.  
  - *Config:* Uses WhatsApp API credentials for authentication.  
  - *Inputs:* Media URL JSON  
  - *Outputs:* Binary audio file  
  - *Failure Modes:* Network errors, invalid URL, authentication failures.

- **OpenAI (Whisper)**  
  - *Type:* AI transcription node  
  - *Role:* Transcribes audio file to text.  
  - *Config:* OpenAI Whisper model with audio resource, transcribe operation.  
  - *Inputs:* Binary audio from HTTP Request  
  - *Outputs:* Transcribed text JSON  
  - *Failure Modes:* API quota limits, corrupted audio, transcription inaccuracies.

- **Edit Fields**  
  - *Type:* Data transformation node (Set)  
  - *Role:* Extracts and formats the transcribed text into a standard JSON field `text`.  
  - *Config:* Assigns `text` with `messages[0].text.body` for text messages; for audio, uses transcription output.  
  - *Inputs:* Switch text branch or OpenAI transcription output  
  - *Outputs:* Formatted JSON with unified message text field  
  - *Failure Modes:* Missing text fields, expression errors.

---

#### 1.2 Smart Email Agent – Intent Analysis & Email Handling

**Overview:**  
This block handles AI intent extraction, contact lookup, email drafting/sending, and optional session memory storage. It converts the processed WhatsApp message text into actionable email operations.

**Nodes Involved:**  
- Edit Fields1  
- Email Agent  
- OpenAI Chat Model  
- Get Email (Airtable)  
- Send Email (Gmail)  
- Create Draft (Gmail)  
- Simple Memory (disabled)  

**Node Details:**

- **Edit Fields1**  
  - *Type:* Data transformation node (Set)  
  - *Role:* Prepares message text as `message_type` for AI agent input.  
  - *Config:* Assigns `message_type` from previous `text` field.  
  - *Inputs:* Edit Fields output  
  - *Outputs:* Message formatted for AI processing  
  - *Failure Modes:* Expression or missing field errors.

- **Email Agent**  
  - *Type:* LangChain Agent node  
  - *Role:* Core decision-making AI that interprets the message, decides to send, draft, or request clarification.  
  - *Config:* Uses a detailed system prompt defining its role as an AI email assistant bilingual in French and English with specific instructions on behavior, tone, and formatting.  
  - *Inputs:* Edit Fields1 output, plus AI tool inputs (OpenAI Chat Model, Get Email, Send Email, Create Draft, Simple Memory)  
  - *Outputs:* Two outputs (main for success, error continues)  
  - *Failure Modes:* AI service timeout, ambiguous input, API key issues.

- **OpenAI Chat Model**  
  - *Type:* AI language model node  
  - *Role:* Processes message to extract intent and reasoning for email tasks.  
  - *Config:* Uses GPT-4 Turbo Preview model.  
  - *Inputs:* Email Agent AI language model input  
  - *Outputs:* Structured AI response  
  - *Failure Modes:* Model unavailability, quota limits.

- **Get Email (Airtable)**  
  - *Type:* Airtable node  
  - *Role:* Retrieves recipient email and contact information based on AI output (filter formula).  
  - *Config:* Searches "Auto" table in "Contact List" base, limited to 5 results; uses dynamic filter from AI input.  
  - *Inputs:* Email Agent AI tool input  
  - *Outputs:* Contact data for email addressing  
  - *Failure Modes:* Incorrect filter formula, API key issues, empty results.

- **Send Email (Gmail)**  
  - *Type:* Gmail node  
  - *Role:* Sends an email directly if AI instructs so.  
  - *Config:* Uses OAuth2 Gmail account credentials; sends to email address and body derived from AI output; subject from AI as well.  
  - *Inputs:* Email Agent AI tool input  
  - *Outputs:* Confirmation of send action  
  - *Failure Modes:* Authentication errors, quota, invalid email addresses.

- **Create Draft (Gmail)**  
  - *Type:* Gmail node  
  - *Role:* Creates an email draft when AI decides to save instead of send.  
  - *Config:* Similar to Send Email but resource set to draft, email type HTML.  
  - *Inputs:* Email Agent AI tool input  
  - *Outputs:* Draft creation confirmation  
  - *Failure Modes:* Same as Send Email.

- **Simple Memory (disabled)**  
  - *Type:* LangChain memory buffer window  
  - *Role:* (Optional) Stores session memory keyed by message content for context persistence.  
  - *Config:* Disabled; session key based on message type.  
  - *Inputs:* AI memory input to Email Agent  
  - *Outputs:* N/A  
  - *Failure Modes:* N/A (disabled).

---

#### 1.3 Vocal Response & WhatsApp Feedback

**Overview:**  
Generates textual or audio feedback to user via WhatsApp. If audio, converts AI text to speech, adjusts MIME type, uploads audio media to WhatsApp Cloud, then sends audio or text message as reply.

**Nodes Involved:**  
- If  
- OpenAI1 (Text-to-Speech)  
- Code (MIME Type Fix)  
- WhatsApp Business Cloud2 (Media Upload)  
- WhatsApp Business Cloud3 (Send Audio Message)  
- WhatsApp Business Cloud1 (Send Text Message)  

**Node Details:**

- **If**  
  - *Type:* Conditional node  
  - *Role:* Checks if the reply should be audio (voice) or text based on WhatsApp message type.  
  - *Config:* Compares `messages[0].type` with "audio".  
  - *Inputs:* Email Agent main output  
  - *Outputs:* Two branches: audio and text.  
  - *Failure Modes:* Misclassification, missing type field.

- **OpenAI1 (Text-to-Speech)**  
  - *Type:* LangChain OpenAI TTS node  
  - *Role:* Converts AI-generated text response to voice audio using voice "nova".  
  - *Config:* OpenAI API credentials, resource set to audio, voice specified.  
  - *Inputs:* If node audio branch  
  - *Outputs:* Binary audio data  
  - *Failure Modes:* API errors, unsupported voice, quota.

- **Code**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Fixes audio MIME type from 'audio/mp3' to 'audio/mpeg' for WhatsApp compatibility.  
  - *Config:* Iterates over binary data and updates MIME type.  
  - *Inputs:* OpenAI1 output  
  - *Outputs:* Corrected binary audio  
  - *Failure Modes:* No binary data, unexpected MIME types.

- **WhatsApp Business Cloud2 (Media Upload)**  
  - *Type:* WhatsApp media upload node  
  - *Role:* Uploads audio file to WhatsApp Cloud for sending.  
  - *Config:* Uses WhatsApp API credentials and phone number ID.  
  - *Inputs:* Code node audio output  
  - *Outputs:* Media ID JSON  
  - *Failure Modes:* Upload failures, invalid credentials.

- **WhatsApp Business Cloud3 (Send Audio Message)**  
  - *Type:* WhatsApp message node  
  - *Role:* Sends the audio message to the user.  
  - *Config:* Uses uploaded media ID, sets messageType to audio, sends to target phone number.  
  - *Inputs:* WhatsApp Business Cloud2 output  
  - *Outputs:* Confirmation of send  
  - *Failure Modes:* Send errors, invalid media ID.

- **WhatsApp Business Cloud1 (Send Text Message)**  
  - *Type:* WhatsApp message node  
  - *Role:* Sends textual feedback message if audio is not required.  
  - *Config:* Sends message body from AI output, to user phone number.  
  - *Inputs:* If node text branch  
  - *Outputs:* Confirmation of send  
  - *Failure Modes:* API errors, invalid phone number.

---

### 3. Summary Table

| Node Name                 | Node Type                                  | Functional Role                                   | Input Node(s)                | Output Node(s)                  | Sticky Note                                                                                                                                |
|---------------------------|--------------------------------------------|--------------------------------------------------|------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| WhatsApp Trigger          | n8n-nodes-base.whatsAppTrigger             | Listen for incoming WhatsApp messages             | (Webhook)                    | Split Out                      | Part 1: WhatsApp Message Intake & Voice Transcription                                                                                      |
| Split Out                 | n8n-nodes-base.splitOut                     | Split batch messages into individual items        | WhatsApp Trigger             | Switch                         | Part 1: WhatsApp Message Intake & Voice Transcription                                                                                      |
| Switch                    | n8n-nodes-base.switch                       | Route based on message type (audio/text)           | Split Out                    | WhatsApp Business Cloud, Edit Fields | Part 1: WhatsApp Message Intake & Voice Transcription                                                                                      |
| WhatsApp Business Cloud   | n8n-nodes-base.whatsApp (mediaUrlGet)      | Retrieve media URL for audio messages               | Switch (voice branch)        | HTTP Request                   | Part 1: WhatsApp Message Intake & Voice Transcription                                                                                      |
| HTTP Request              | n8n-nodes-base.httpRequest                  | Download audio file from media URL                   | WhatsApp Business Cloud      | OpenAI (Whisper)               | Part 1: WhatsApp Message Intake & Voice Transcription                                                                                      |
| OpenAI                   | @n8n/n8n-nodes-langchain.openAi (Whisper) | Transcribe audio to text                             | HTTP Request                 | Edit Fields                   | Part 1: WhatsApp Message Intake & Voice Transcription                                                                                      |
| Edit Fields               | n8n-nodes-base.set                          | Format transcribed or text message into field      | Switch (text branch) / OpenAI| Edit Fields1                  | Part 1: WhatsApp Message Intake & Voice Transcription                                                                                      |
| Edit Fields1              | n8n-nodes-base.set                          | Prepare unified message_type for AI agent          | Edit Fields                  | Email Agent                   | Part 2: Smart Email Agent – Read, Draft, Respond                                                                                           |
| Email Agent               | @n8n/n8n-nodes-langchain.agent              | AI-powered email request interpretation & action  | Edit Fields1                 | If (audio check)              | Part 2: Smart Email Agent – Read, Draft, Respond                                                                                           |
| OpenAI Chat Model         | @n8n/n8n-nodes-langchain.lmChatOpenAi       | Extract intent & reasoning from message             | Email Agent (ai_languageModel) | Email Agent (ai_tool)         | Part 2: Smart Email Agent – Read, Draft, Respond                                                                                           |
| Get Email                 | n8n-nodes-base.airtableTool                  | Lookup contact emails from Airtable                 | Email Agent (ai_tool)        | Email Agent (ai_tool)          | Part 2: Smart Email Agent – Read, Draft, Respond                                                                                           |
| Send Email                | n8n-nodes-base.gmailTool                      | Send email directly via Gmail                        | Email Agent (ai_tool)        | Email Agent (ai_tool)          | Part 2: Smart Email Agent – Read, Draft, Respond                                                                                           |
| Create Draft              | n8n-nodes-base.gmailTool                      | Create email draft for review                        | Email Agent (ai_tool)        | Email Agent (ai_tool)          | Part 2: Smart Email Agent – Read, Draft, Respond                                                                                           |
| Simple Memory             | @n8n/n8n-nodes-langchain.memoryBufferWindow | Optional session memory for context                  | Email Agent (ai_memory)      | Email Agent (ai_memory)        | Part 2: Smart Email Agent – Read, Draft, Respond (disabled)                                                                                |
| If                        | n8n-nodes-base.if                            | Decide if response is audio or text                  | Email Agent                  | OpenAI1 (audio), WhatsApp Business Cloud1 (text) | Part 3: Vocal Response & WhatsApp Feedback                                                                                                 |
| OpenAI1                   | @n8n/n8n-nodes-langchain.openAi (TTS)       | Generate voice audio from AI text response           | If (audio branch)            | Code                          | Part 3: Vocal Response & WhatsApp Feedback                                                                                                 |
| Code                      | n8n-nodes-base.code                          | Fix MIME type of audio to 'audio/mpeg'               | OpenAI1                      | WhatsApp Business Cloud2       | Part 3: Vocal Response & WhatsApp Feedback                                                                                                 |
| WhatsApp Business Cloud2  | n8n-nodes-base.whatsApp                       | Upload audio media to WhatsApp Cloud                  | Code                         | WhatsApp Business Cloud3       | Part 3: Vocal Response & WhatsApp Feedback                                                                                                 |
| WhatsApp Business Cloud3  | n8n-nodes-base.whatsApp                       | Send audio message via WhatsApp                        | WhatsApp Business Cloud2     | (End)                         | Part 3: Vocal Response & WhatsApp Feedback                                                                                                 |
| WhatsApp Business Cloud1  | n8n-nodes-base.whatsApp                       | Send text message via WhatsApp                         | If (text branch)             | (End)                         | Part 3: Vocal Response & WhatsApp Feedback                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger node**  
   - Type: `whatsAppTrigger`  
   - Configure OAuth2 credentials for WhatsApp Business Cloud API.  
   - Set updates to listen for `messages`.  
   - Position this as the starting point.

2. **Add Split Out node**  
   - Type: `splitOut`  
   - Configure to split array of messages in incoming webhook payload (field: `messages`).  
   - Connect WhatsApp Trigger output to Split Out input.

3. **Add Switch node**  
   - Type: `switch`  
   - Create two outputs: "voice" and "text".  
   - Set condition to check `messages[0].type` equals `"audio"` for voice, `"text"` for text.  
   - Connect Split Out output to Switch input.

4. **Add WhatsApp Business Cloud node (mediaUrlGet)**  
   - Type: `whatsApp`  
   - Operation: `mediaUrlGet`  
   - Set `mediaGetId` to `{{$json.audio.id}}` (from message).  
   - Use WhatsApp API credentials.  
   - Connect Switch "voice" branch output to this node.

5. **Add HTTP Request node**  
   - Type: `httpRequest`  
   - URL: Set dynamically to media URL from previous node (`{{$json.url}}`).  
   - Authentication: Use WhatsApp API credentials.  
   - Connect WhatsApp Business Cloud output to HTTP Request input.

6. **Add OpenAI node (Whisper transcription)**  
   - Type: LangChain OpenAI node  
   - Resource: `audio`  
   - Operation: `transcribe`  
   - Use OpenAI API credentials with Whisper access.  
   - Connect HTTP Request output (audio binary) to OpenAI input.

7. **Add Edit Fields node**  
   - Type: `set`  
   - Assign a new field `text` as either the transcribed text (from OpenAI) or text body from WhatsApp message for text branch.  
   - Connect Switch "text" branch output directly here; also connect OpenAI transcription output here.

8. **Add Edit Fields1 node**  
   - Type: `set`  
   - Assign `message_type` field from the `text` field.  
   - Connect Edit Fields output to Edit Fields1 input.

9. **Add Email Agent node (LangChain Agent)**  
   - Type: `agent`  
   - Configure with a system prompt defining email assistant behavior in French and English.  
   - Connect Edit Fields1 output to Email Agent input.

10. **Add OpenAI Chat Model node**  
    - Type: `lmChatOpenAi`  
    - Model: `gpt-4-turbo-preview`  
    - Connect Email Agent's AI language model input.

11. **Add Get Email node (Airtable)**  
    - Type: `airtableTool`  
    - Base: Contact List (your Airtable base)  
    - Table: Auto (your contact table)  
    - Operation: Search with filter formula dynamically set from AI output.  
    - Connect Email Agent AI tool input.

12. **Add Send Email node (Gmail)**  
    - Type: `gmailTool`  
    - Operation: Send Email  
    - Credentials: OAuth2 Gmail account.  
    - Set `sendTo`, `message`, and `subject` from AI output fields.  
    - Connect Email Agent AI tool input.

13. **Add Create Draft node (Gmail)**  
    - Type: `gmailTool`  
    - Operation: Create Draft  
    - Email type: HTML  
    - Credentials: Same Gmail OAuth2.  
    - Connect Email Agent AI tool input.

14. **Add If node**  
    - Type: `if`  
    - Condition: Check if original WhatsApp message type is audio (`messages[0].type == "audio"`).  
    - Connect Email Agent main output.

15. **Add OpenAI1 node (Text-to-Speech)**  
    - Type: LangChain OpenAI node  
    - Resource: `audio`  
    - Operation: Text-to-Speech (voice: nova)  
    - Connect If node audio output.

16. **Add Code node**  
    - Type: Code  
    - JavaScript to convert MIME type from `audio/mp3` to `audio/mpeg` for WhatsApp compatibility.  
    - Connect OpenAI1 output.

17. **Add WhatsApp Business Cloud2 node**  
    - Type: WhatsApp media upload  
    - Operation: Upload media (audio)  
    - Use WhatsApp API credentials.  
    - Connect Code node output.

18. **Add WhatsApp Business Cloud3 node**  
    - Type: WhatsApp message send  
    - Operation: Send audio message  
    - Use media ID from previous upload node.  
    - Connect WhatsApp Business Cloud2 output.

19. **Add WhatsApp Business Cloud1 node**  
    - Type: WhatsApp message send  
    - Operation: Send text message  
    - Use AI-generated text response.  
    - Connect If node text output.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                               | Context or Link                                                                                                        |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| This workflow integrates WhatsApp Business Cloud API, OpenAI GPT models for natural language understanding, OpenAI Whisper for speech-to-text transcription, and Gmail API for email handling in a unified AI email assistant.             | Workflow description and node configurations                                                                            |
| Setup prerequisites include WhatsApp Business Cloud API access with webhook, OpenAI API keys, Gmail OAuth2 credentials, Airtable account for contacts, and an n8n instance with HTTPS.                                                     | Sticky Note (Setup Guide)                                                                                               |
| The system prompt in the Email Agent node is critical for guiding AI behavior, including bilingual support, HTML email formatting, and clear instructions not to justify actions in responses.                                             | Email Agent node parameters                                                                                             |
| MIME type adjustment in the Code node is necessary for WhatsApp to accept audio messages; without this, audio playback issues may occur.                                                                                                | Code node JavaScript                                                                                                    |
| Error handling is configured to continue on Email Agent errors to avoid workflow interruption; recommended to monitor logs for failures such as API limits or invalid inputs.                                                             | Email Agent node `onError` set to `continueErrorOutput`                                                                 |
| Optional memory storage is available but disabled; enabling it can provide session context for multi-turn conversations or complex email handling.                                                                                      | Simple Memory node (disabled)                                                                                           |
| For voice responses, ElevenLabs or other TTS providers can be integrated alternatively to OpenAI if preferred.                                                                                                                           | Suggested enhancement (not implemented here)                                                                           |
| Official WhatsApp Business Cloud API documentation: https://developers.facebook.com/docs/whatsapp/cloud-api/                                                                                                                             | Useful link for webhook and API setup                                                                                   |
| OpenAI API documentation for GPT and Whisper: https://platform.openai.com/docs/models/                                                                                                                                                    | Useful link for AI models                                                                                                |
| Gmail API OAuth2 setup guide: https://developers.google.com/identity/protocols/oauth2                                                                                                                                                     | Useful link for Gmail credential setup                                                                                   |
| Airtable API documentation: https://airtable.com/api                                                                                                                                                                                      | Useful link for contact data integration                                                                                 |

---

**Disclaimer:**  
The content provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with prevailing content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.