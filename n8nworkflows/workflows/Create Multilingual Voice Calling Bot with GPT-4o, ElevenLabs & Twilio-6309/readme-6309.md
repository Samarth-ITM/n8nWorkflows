Create Multilingual Voice Calling Bot with GPT-4o, ElevenLabs & Twilio

https://n8nworkflows.xyz/workflows/create-multilingual-voice-calling-bot-with-gpt-4o--elevenlabs---twilio-6309


# Create Multilingual Voice Calling Bot with GPT-4o, ElevenLabs & Twilio

### 1. Workflow Overview

This workflow implements a **Multilingual AI Voice Calling Bot** integrating Twilio for voice call handling, OpenAI GPT-4o for conversational AI, and ElevenLabs for text-to-speech synthesis. It is designed for service businesses to interact with customers via phone calls, enabling use cases like appointment scheduling, pizza ordering, and other service bookings through natural voice dialogue.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception and Initial Greeting:** Handles incoming Twilio voice calls, checks for speech input, and if none is detected, provides an initial greeting prompting user speech.
- **1.2 AI Conversational Processing:** Sends recognized speech text to OpenAI GPT-4o for generating context-aware, polite, concise responses tailored for voice interaction.
- **1.3 Text-to-Speech Conversion and Response Delivery:** Converts the AI-generated text response into speech audio using ElevenLabs, uploads the audio file, and responds to the caller with TwiML instructions to play the audio and gather further speech input.
- **1.4 Conversation Logging and Appointment Handling:** Logs each conversation interaction into Google Sheets and detects if the user requests an appointment, saving appointment requests separately for follow-up.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initial Greeting

- **Overview:**  
  This block starts the voice call interaction via a Twilio webhook. It checks if any speech input is received in the initial request. If speech is absent, it responds with a predefined greeting and prompts the caller to speak after a tone.

- **Nodes Involved:**  
  - Twilio Voice Webhook  
  - Check Speech Input  
  - Initial Greeting

- **Node Details:**

  - **Twilio Voice Webhook**  
    - Type: Webhook  
    - Role: Entry point triggered by Twilio Voice POST requests when a call is made.  
    - Configuration: HTTP POST method on path `/voice-webhook`, response mode set to respondNode to allow dynamic TwiML responses.  
    - Inputs: Incoming Twilio call payload containing speech results and call metadata.  
    - Outputs: Connected to "Check Speech Input" node.  
    - Edge cases: Missing or malformed webhook requests; Twilio auth not configured here but assumed set in Twilio console.

  - **Check Speech Input**  
    - Type: If (conditional) node  
    - Role: Checks if the JSON field `SpeechResult` exists and is non-empty to determine if caller spoke.  
    - Configuration: Condition checks existence of `{{$json.SpeechResult}}`.  
    - Inputs: From Twilio Voice Webhook.  
    - Outputs:  
      - If true: continues to OpenAI GPT node.  
      - If false: goes to Initial Greeting node.  
    - Edge cases: Speech recognition failures, empty speech inputs.

  - **Initial Greeting**  
    - Type: Respond to Webhook  
    - Role: Sends a TwiML voice greeting prompting the caller to speak, with a speech input gather to receive the next utterance.  
    - Configuration: TwiML includes `<Say>` with voice "alice", `<Gather>` with speech input enabled, 10 seconds timeout, and action pointing back to the webhook URL for continuous interaction.  
    - Inputs: From Check Speech Input (false branch).  
    - Outputs: None (end of this path until new speech input).  
    - Edge cases: Caller does not respond; fallback message and hangup included.

#### 2.2 AI Conversational Processing

- **Overview:**  
  If speech input exists, this block sends the caller’s speech text to OpenAI GPT-4o, instructing it to act as a polite, concise AI assistant for scheduling and ordering use cases. The AI response guides the dialogue step-by-step.

- **Nodes Involved:**  
  - OpenAI GPT-4o Response

- **Node Details:**

  - **OpenAI GPT-4o Response**  
    - Type: OpenAI node  
    - Role: Sends conversation context and user speech to GPT-4o, receives AI-generated text response.  
    - Configuration:  
      - Model: GPT-4o  
      - Max tokens: 150, Temperature: 0.7 (balanced creativity)  
      - System prompt defines assistant behavior for scheduling, ordering, politeness, and voice call brevity (under 100 words).  
      - User prompt: uses either the caller’s speech or a default greeting if none present.  
    - Inputs: From Check Speech Input (true branch).  
    - Outputs: To ElevenLabs Text-to-Speech and Log Conversation.  
    - Credentials: OpenAI API key configured.  
    - Edge cases: API rate limits, token overflow, failed API calls.

#### 2.3 Text-to-Speech Conversion and Response Delivery

- **Overview:**  
  Converts AI text responses into spoken audio via ElevenLabs TTS, uploads the audio file, and sends TwiML response to Twilio to play it back to the caller, then gathers further speech input.

- **Nodes Involved:**  
  - ElevenLabs Text-to-Speech  
  - Upload Audio to Storage  
  - Twilio TwiML Response

- **Node Details:**

  - **ElevenLabs Text-to-Speech**  
    - Type: ElevenLabs node  
    - Role: Synthesizes speech audio from AI text response.  
    - Configuration:  
      - Text: AI response content from OpenAI node.  
      - Model: "eleven_monolingual_v1"  
      - Voice ID: "21m00Tcm4TlvDq8ikWAM" (default English voice)  
      - Voice settings: style=0, stability=0.75, similarity_boost=0.75, speaker boost enabled for naturalness.  
    - Inputs: From OpenAI GPT-4o Response.  
    - Outputs: To Upload Audio node.  
    - Credentials: ElevenLabs API key.  
    - Edge cases: API call failures, unsupported languages or voices.

  - **Upload Audio to Storage**  
    - Type: HTTP Request (upload)  
    - Role: Uploads the generated MP3 audio file to a storage service (endpoint not explicitly configured in JSON).  
    - Configuration:  
      - Operation upload with filename "response.mp3" and content type "audio/mpeg"  
      - Uses binary property "data" from ElevenLabs node output.  
    - Inputs: From ElevenLabs Text-to-Speech.  
    - Outputs: To Twilio TwiML Response.  
    - Edge cases: Upload failures, storage endpoint misconfiguration, latency.

  - **Twilio TwiML Response**  
    - Type: Respond to Webhook  
    - Role: Sends TwiML XML instructing Twilio to play the uploaded audio, gather additional speech, and hang up if no input.  
    - Configuration:  
      - `<Play>` points to `{{$json.audio_url}}` from upload node output.  
      - `<Gather>` configured for speech input, 10 seconds timeout, auto speechTimeout, action loops back to webhook URL for continuous dialogue.  
      - Fallback `<Say>` and `<Hangup>` for no input scenario.  
    - Inputs: From Upload Audio node.  
    - Outputs: None (final response for this turn).  
    - Edge cases: Missing or invalid audio URL, speech gather failures.

#### 2.4 Conversation Logging and Appointment Handling

- **Overview:**  
  Logs call details and conversation to Google Sheets, checks if the AI response contains appointment-related keywords, and if so, saves appointment requests for further processing.

- **Nodes Involved:**  
  - Log Conversation  
  - Check for Appointment  
  - Save Appointment Request

- **Node Details:**

  - **Log Conversation**  
    - Type: Google Sheets node  
    - Role: Appends a new row to a "Call_Logs" sheet recording call SID, caller ID, timestamp, AI response, and speech input.  
    - Configuration:  
      - Document ID: "your-google-sheet-id" (placeholder to be replaced)  
      - Sheet name: "Call_Logs"  
      - Columns mapped explicitly to call metadata and conversation content.  
    - Inputs: From OpenAI GPT-4o Response.  
    - Outputs: To Check for Appointment node.  
    - Credentials: Google Sheets OAuth2.  
    - Edge cases: API quota limits, invalid credentials, missing sheet or document.

  - **Check for Appointment**  
    - Type: If (conditional) node  
    - Role: Inspects AI response text for the keyword "appointment" (case-insensitive).  
    - Configuration: Checks if AI response `.toLowerCase()` contains "appointment".  
    - Inputs: From Log Conversation.  
    - Outputs:  
      - True branch: To Save Appointment Request node.  
      - False branch: No connection (ends here).  
    - Edge cases: False positives/negatives in keyword matching.

  - **Save Appointment Request**  
    - Type: Google Sheets node  
    - Role: Saves appointment request details to a separate "Appointments" sheet with status "pending" for further action.  
    - Configuration:  
      - Document ID: "your-appointments-sheet-id" (placeholder)  
      - Sheet name: "Appointments"  
      - Columns: status, call SID, caller ID, timestamp, and raw speech request details.  
    - Inputs: From Check for Appointment (true branch).  
    - Credentials: Google Sheets OAuth2.  
    - Edge cases: Same as Log Conversation node; plus ensuring request details are parsable.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                         | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                         |
|-----------------------|---------------------|---------------------------------------|------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------|
| Twilio Voice Webhook   | Webhook             | Entry point for incoming Twilio calls | -                      | Check Speech Input         |                                                                                                                     |
| Check Speech Input     | If                  | Checks if speech input exists          | Twilio Voice Webhook    | OpenAI GPT-4o Response (true)<br>Initial Greeting (false) |                                                                                                                     |
| Initial Greeting       | Respond to Webhook   | Sends greeting and gathers speech     | Check Speech Input      | -                         |                                                                                                                     |
| OpenAI GPT-4o Response | OpenAI              | Generates AI conversational response  | Check Speech Input      | ElevenLabs TTS, Log Conversation |                                                                                                                     |
| ElevenLabs Text-to-Speech | ElevenLabs        | Converts AI text to speech audio       | OpenAI GPT-4o Response  | Upload Audio to Storage    |                                                                                                                     |
| Upload Audio to Storage| HTTP Request        | Uploads audio file for playback        | ElevenLabs Text-to-Speech| Twilio TwiML Response     |                                                                                                                     |
| Twilio TwiML Response  | Respond to Webhook   | Sends TwiML to play audio and gather speech | Upload Audio to Storage | -                         |                                                                                                                     |
| Log Conversation      | Google Sheets        | Logs call and conversation data        | OpenAI GPT-4o Response  | Check for Appointment      |                                                                                                                     |
| Check for Appointment  | If                  | Detects appointment requests           | Log Conversation        | Save Appointment Request (true) |                                                                                                                     |
| Save Appointment Request | Google Sheets      | Saves appointment requests             | Check for Appointment   | -                         |                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Twilio Voice Webhook` node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `voice-webhook`  
   - Response Mode: `responseNode`  
   - Purpose: Receive incoming Twilio call webhook requests.

2. **Create `Check Speech Input` node**  
   - Type: If node  
   - Condition: Check if `{{$json.SpeechResult}}` exists and is not empty.  
   - Connect input from `Twilio Voice Webhook` node.

3. **Create `Initial Greeting` node**  
   - Type: Respond to Webhook  
   - TwiML content:  
     ```xml
     <?xml version="1.0" encoding="UTF-8"?>
     <Response>
       <Say voice="alice">Hello! I'm your AI assistant. I can help you schedule appointments, order pizza, or book services. How can I help you today?</Say>
       <Gather input="speech" timeout="10" speechTimeout="auto" action="{{ $node['Twilio Voice Webhook'].getWebhookUrl() }}" method="POST">
         <Say voice="alice">Please speak after the tone.</Say>
       </Gather>
       <Say voice="alice">I didn't hear anything. Please call back. Goodbye!</Say>
       <Hangup/>
     </Response>
     ```  
   - Connect input from `Check Speech Input` node's false branch.

4. **Create `OpenAI GPT-4o Response` node**  
   - Type: OpenAI  
   - Model: GPT-4o  
   - Max Tokens: 150  
   - Temperature: 0.7  
   - Messages:  
     - System: Defines assistant behavior for scheduling, ordering, politeness, brevity (under 100 words).  
     - User: `{{$json.SpeechResult || 'Hello, how can I help you today?' }}`  
   - Credentials: Configure OpenAI API credentials.  
   - Connect input from `Check Speech Input` node's true branch.

5. **Create `ElevenLabs Text-to-Speech` node**  
   - Type: ElevenLabs  
   - Text: `{{$json.choices[0].message.content}}` from OpenAI response  
   - Model ID: `eleven_monolingual_v1`  
   - Voice ID: `21m00Tcm4TlvDq8ikWAM`  
   - Voice Settings: style=0, stability=0.75, similarity_boost=0.75, use_speaker_boost=true  
   - Credentials: Configure ElevenLabs API credentials.  
   - Connect input from `OpenAI GPT-4o Response`.

6. **Create `Upload Audio to Storage` node**  
   - Type: HTTP Request  
   - Operation: Upload  
   - File Name: `response.mp3`  
   - Content Type: `audio/mpeg`  
   - Binary Property Name: `data` (from ElevenLabs node output)  
   - Connect input from `ElevenLabs Text-to-Speech`.

7. **Create `Twilio TwiML Response` node**  
   - Type: Respond to Webhook  
   - TwiML content:  
     ```xml
     <?xml version="1.0" encoding="UTF-8"?>
     <Response>
       <Play>{{ $json.audio_url }}</Play>
       <Gather input="speech" timeout="10" speechTimeout="auto" action="{{ $node.webhook.getWebhookUrl() }}" method="POST">
         <Say voice="alice">Please speak after the tone.</Say>
       </Gather>
       <Say voice="alice">I didn't hear anything. Goodbye!</Say>
       <Hangup/>
     </Response>
     ```  
   - Connect input from `Upload Audio to Storage`.

8. **Create `Log Conversation` node**  
   - Type: Google Sheets (Append)  
   - Document ID: Replace with your Google Sheet ID  
   - Sheet Name: `Call_Logs`  
   - Columns mapping:  
     - call_sid: `{{$json.CallSid}}`  
     - caller_id: `{{$json.From}}`  
     - timestamp: `{{$now}}`  
     - ai_response: `{{$json.choices[0].message.content}}`  
     - speech_input: `{{$json.SpeechResult}}`  
   - Credentials: Configure Google Sheets OAuth2 credentials.  
   - Connect input from `OpenAI GPT-4o Response`.

9. **Create `Check for Appointment` node**  
   - Type: If node  
   - Condition: Check if `{{$json.choices[0].message.content.toLowerCase()}}` contains "appointment" (case-insensitive)  
   - Connect input from `Log Conversation`.

10. **Create `Save Appointment Request` node**  
    - Type: Google Sheets (Append)  
    - Document ID: Replace with your appointments Google Sheet ID  
    - Sheet Name: `Appointments`  
    - Columns mapping:  
      - status: `"pending"`  
      - call_sid: `{{$json.CallSid}}`  
      - caller_id: `{{$json.From}}`  
      - timestamp: `{{$now}}`  
      - request_details: `{{$json.SpeechResult}}`  
    - Credentials: Google Sheets OAuth2 credentials  
    - Connect input from `Check for Appointment` node's true branch.

11. **Connect nodes according to the workflow connections:**  
    - `Twilio Voice Webhook` → `Check Speech Input`  
    - `Check Speech Input` true → `OpenAI GPT-4o Response`  
    - `Check Speech Input` false → `Initial Greeting`  
    - `OpenAI GPT-4o Response` → `ElevenLabs Text-to-Speech` and `Log Conversation`  
    - `ElevenLabs Text-to-Speech` → `Upload Audio to Storage`  
    - `Upload Audio to Storage` → `Twilio TwiML Response`  
    - `Log Conversation` → `Check for Appointment`  
    - `Check for Appointment` true → `Save Appointment Request`

12. **Set workflow to active and test with live Twilio voice calls configured to hit the webhook URL.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                                                         |
|------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| Ensure Twilio webhook URLs are publicly accessible and configured in the Twilio console to receive voice webhook POST requests. | Twilio Voice Webhook Setup                                              |
| Replace placeholder Google Sheets Document IDs with your actual spreadsheet IDs and ensure correct sheet names exist.         | Google Sheets API and OAuth2 setup                                     |
| OpenAI GPT-4o model is used for stateful, context-aware conversations; monitor token usage and rate limits for production use.| OpenAI API Documentation                                               |
| ElevenLabs voice ID and model can be customized for other languages or voices to support multilingual capabilities.            | ElevenLabs TTS API Documentation                                       |
| TwiML `<Gather>` action loops back to webhook URL to maintain conversational state across multiple user inputs.               | Twilio Gather Verb Documentation                                       |
| This workflow assumes the use of OAuth2 credentials for Google Sheets and API keys for OpenAI and ElevenLabs.                  | n8n Credential Management                                              |
| For best accuracy, verify speech recognition accuracy and consider fallback prompts in case of recognition failures.          | Twilio Speech Recognition Guidelines                                  |

---

**Disclaimer:**  
The content provided derives exclusively from an automated workflow created with n8n, respecting current content policies and containing no illegal or offensive material. All processed data is legal and publicly accessible.