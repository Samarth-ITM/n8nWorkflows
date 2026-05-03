Create Voice Assistant Interface with OpenAI GPT-4o-mini and Text-to-Speech

https://n8nworkflows.xyz/workflows/create-voice-assistant-interface-with-openai-gpt-4o-mini-and-text-to-speech-6399


# Create Voice Assistant Interface with OpenAI GPT-4o-mini and Text-to-Speech

---

### 1. Workflow Overview

This workflow implements a **Voice Assistant Interface** that leverages OpenAI's GPT-4o-mini model for conversational AI with integrated text-to-speech voice responses. It enables users to interact with an AI assistant through a browser-based voice interface. The workflow is designed to:

- Serve a visually engaging frontend voice interface with a clickable orb that activates speech recognition.
- Receive spoken input converted to text by the browser’s Web Speech API.
- Process the transcribed input with a conversational AI agent that maintains session memory.
- Generate a voice response using OpenAI's text-to-speech capabilities.
- Return audio responses directly to the user's browser for playback.

The workflow is logically divided into two main blocks:

- **1.1 Frontend Interface Delivery**: Serving the interactive HTML voice assistant page and responding to UI requests.
- **1.2 Backend Audio Processing and AI Interaction**: Handling incoming transcribed text, processing it with AI memory-enhanced conversation, generating voice responses, and returning audio to the client.

---

### 2. Block-by-Block Analysis

#### 1.1 Frontend Interface Delivery

**Overview:**  
This block serves a custom HTML page featuring an animated orb UI that users interact with to record speech. It handles HTTP GET requests to provide the interactive voice interface.

**Nodes Involved:**  
- Voice Interface Endpoint (Webhook)  
- Voice Assistant UI (HTML)  
- Send HTML Interface (Respond to Webhook)  
- Sticky Note (Voice Assistant Interface)

**Node Details:**  

- **Voice Interface Endpoint**  
  - *Type:* Webhook  
  - *Role:* Entry point for frontend requests, exposing the `/voice-assistant` HTTP endpoint.  
  - *Configuration:* Responds via the "Send HTML Interface" node; listens for GET requests by default.  
  - *Connections:* Outputs to "Voice Assistant UI".  
  - *Edge cases:*  
    - Network issues may prevent page delivery.  
    - Webhook URL must be correctly set in the frontend script for audio processing.  
  - *Version:* n8n Webhook v2.

- **Voice Assistant UI**  
  - *Type:* HTML node  
  - *Role:* Provides the full HTML, CSS, and JavaScript code for the voice assistant UI.  
  - *Configuration:* Contains embedded HTML with CSS animations for the orb, JavaScript for Web Speech API integration, and fetch calls to the audio processing webhook.  
  - *Key expressions:* Static HTML content; requires manual insertion of the actual audio processing webhook URL replacing `YOUR_WEBHOOK_URL_HERE`.  
  - *Connections:* Outputs HTML to "Send HTML Interface".  
  - *Edge cases:*  
    - Browser compatibility issues with `webkitSpeechRecognition`.  
    - Requires HTTPS and microphone permissions.  
    - If the webhook URL is incorrect or unreachable, audio response playback will fail.  
  - *Version:* HTML v1.2.

- **Send HTML Interface**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends the HTML content as a text response to the frontend client.  
  - *Configuration:* Responds with content-type text/html, delivering the HTML stored in the incoming JSON under `$json.html`.  
  - *Connections:* Final node in frontend delivery chain.  
  - *Edge cases:*  
    - Malformed HTML or encoding issues could break UI rendering.  
  - *Version:* Respond to Webhook v1.1.

- **Sticky Note (Voice Assistant Interface)**  
  - *Purpose:* Documentation note describing this block as delivering the interactive orb frontend via webhook.  
  - *Content:* "This webhook serves the HTML interface with the interactive orb that users click to speak with the AI assistant. Access this webhook URL in your browser to use the voice assistant."  

---

#### 1.2 Backend Audio Processing and AI Interaction

**Overview:**  
This block receives the transcribed text from the frontend, processes it through an AI conversation agent with memory, generates an audio voice response, and returns it as binary audio data to the frontend for playback.

**Nodes Involved:**  
- Audio Processing Endpoint (Webhook)  
- Process User Query (AI Agent)  
- Conversation Memory (Memory Buffer)  
- Generate Voice Response (OpenAI Text-to-Speech) [Disabled]  
- Send Audio Response (Respond to Webhook)  
- Sticky Note1 (Backend Processing)  
- Sticky Note (General Setup Instructions and Customization)  

**Node Details:**  

- **Audio Processing Endpoint**  
  - *Type:* Webhook  
  - *Role:* Receives POST requests containing JSON with a `question` field (transcribed user input).  
  - *Configuration:* Path `/process-audio`, HTTP method POST, responds via "Send Audio Response".  
  - *Connections:* Outputs to "Process User Query".  
  - *Edge cases:*  
    - Invalid/malformed JSON input.  
    - Missing `question` property.  
    - High latency or server overload causing timeouts.  
  - *Version:* Webhook v2.

- **Process User Query**  
  - *Type:* Langchain Agent  
  - *Role:* Processes the user’s question using the GPT-4o-mini model with a system prompt for a friendly, conversational assistant.  
  - *Configuration:*  
    - Input text: Expression `={{ $json.body.question }}` pulls the user query from webhook payload.  
    - System message: "You are a helpful AI assistant. Respond in a friendly and conversational manner."  
    - Prompt type: Defined prompt.  
  - *Connections:* Outputs to "Generate Voice Response" (currently disabled).  
  - *AI Memory:* Connected to "Conversation Memory" node for context-aware responses.  
  - *Edge cases:*  
    - API authentication errors.  
    - Model rate limits or unavailability.  
    - Empty or nonsensical input.  
  - *Version:* Langchain Agent v1.8.

- **Conversation Memory**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Maintains conversational context for the AI agent using a session key `voice-assistant-session`.  
  - *Configuration:*  
    - Session ID type: Custom key (static in this case).  
    - Context window length: 30 messages for conversation history.  
  - *Connections:* Provides memory context to "Process User Query".  
  - *Edge cases:*  
    - Memory buffer overflow or excessive context.  
    - Session key collisions if scaled with multiple users (single static key may limit multi-user use).  
  - *Version:* Langchain Memory v1.3.

- **Generate Voice Response** [Disabled]  
  - *Type:* Langchain OpenAI (Audio resource)  
  - *Role:* Converts text AI output to voice audio using OpenAI Text-to-Speech API.  
  - *Configuration:*  
    - Input: AI agent's response text.  
    - Voice: "onyx" voice selected for deep authoritative tone.  
    - Disabled: This node is currently disabled, likely requiring enabling and proper credentials.  
  - *Connections:* Outputs binary audio data to "Send Audio Response".  
  - *Edge cases:*  
    - Disabled state prevents voice response generation.  
    - API key or permission issues.  
    - Voice availability or quota limits.  
  - *Version:* Langchain OpenAI v1.8.

- **Send Audio Response**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends the generated audio response back to the frontend as binary data for playback.  
  - *Configuration:* Responds with binary data payload linked from the previous node (audio).  
  - *Connections:* Final node in backend processing.  
  - *Edge cases:*  
    - If input binary data missing (e.g., due to disabled voice generation), response fails or is empty.  
    - Large audio payloads may cause delays.  
  - *Version:* Respond to Webhook v1.1.

- **Sticky Note1 (Backend Processing)**  
  - *Purpose:* Documentation note explaining backend logic steps.  
  - *Content:*  
    "This section handles:  
    1. Receiving transcribed speech from the frontend  
    2. Processing through AI with conversation memory  
    3. Converting response to speech  
    4. Sending audio back to the browser"

- **Sticky Note (Setup Instructions)**  
  - *Content:* Provides detailed setup steps around credential configuration, webhook URL replacement in the frontend, and testing instructions.

- **Sticky Note (Customization Options)**  
  - *Content:* Outlines configurable options such as language settings in the HTML, voice selection, and UI theme modifications.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                             | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                                             |
|-------------------------|----------------------------------|---------------------------------------------|----------------------------|---------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Voice Interface Endpoint | Webhook                          | Frontend entry point serving voice UI       |                            | Voice Assistant UI        | This webhook serves the HTML interface with the interactive orb that users click to speak with the AI assistant.         |
| Voice Assistant UI       | HTML                             | Provides interactive orb UI HTML + JS        | Voice Interface Endpoint   | Send HTML Interface       |                                                                                                                         |
| Send HTML Interface      | Respond to Webhook               | Sends HTML page response to client           | Voice Assistant UI         |                           |                                                                                                                         |
| Audio Processing Endpoint| Webhook                          | Receives user speech transcription (POST)   |                            | Process User Query        | This section handles: 1. Receiving transcribed speech 2. AI processing with memory 3. Voice conversion 4. Audio return  |
| Process User Query       | Langchain Agent                  | Conversational AI processing with memory    | Audio Processing Endpoint, Conversation Memory | Generate Voice Response (disabled) |                                                                                                                         |
| Conversation Memory      | Langchain Memory Buffer Window   | Maintain conversation context/session        |                            | Process User Query (memory) |                                                                                                                         |
| Generate Voice Response  | Langchain OpenAI (Audio) [Disabled] | Convert text response to speech audio        | Process User Query         | Send Audio Response       |                                                                                                                         |
| Send Audio Response      | Respond to Webhook               | Sends audio data back to frontend             | Generate Voice Response    |                           |                                                                                                                         |
| Sticky Note              | Sticky Note                     | Documentation for frontend block              |                            |                           | This webhook serves the HTML interface with the interactive orb that users click to speak with the AI assistant.         |
| Sticky Note1             | Sticky Note                     | Documentation for backend processing block    |                            |                           | This section handles: 1. Receiving transcribed speech 2. AI processing with memory 3. Voice conversion 4. Audio return  |
| Template Description     | Sticky Note                     | General overview, audience, setup, demo link |                            |                           | General workflow overview and instructions. See https://youtu.be/0bMdJcRMnZY                                              |
| Setup Instructions       | Sticky Note                     | Setup steps for credentials and webhook URLs |                            |                           | Setup instructions for OpenAI credentials and webhook URL replacement in the UI.                                        |
| Customization Options    | Sticky Note                     | Customization hints for language, voice, UI  |                            |                           | Language and voice options, visual theme customization details.                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Voice Interface Endpoint"**  
   - Type: Webhook  
   - Path: `voice-assistant` (GET default)  
   - Response Mode: Response Node  
   - Purpose: Serve the frontend voice assistant UI.  

2. **Create HTML Node: "Voice Assistant UI"**  
   - Type: HTML  
   - Paste the full HTML content that includes:  
     - Animated orb UI with CSS styles.  
     - JavaScript using `webkitSpeechRecognition` for voice input.  
     - Beep sound on start.  
     - Fetch POST to audio processing webhook (placeholder URL).  
   - Note: Replace `YOUR_WEBHOOK_URL_HERE` in JavaScript with actual audio processing webhook URL from step 4.  

3. **Connect "Voice Interface Endpoint" → "Voice Assistant UI"**

4. **Create Respond to Webhook Node: "Send HTML Interface"**  
   - Type: Respond to Webhook  
   - Respond With: Text  
   - Response Body: Expression `={{ $json.html }}` to send the HTML content.  

5. **Connect "Voice Assistant UI" → "Send HTML Interface"**

6. **Create Webhook Node: "Audio Processing Endpoint"**  
   - Type: Webhook  
   - Path: `process-audio`  
   - HTTP Method: POST  
   - Response Mode: Response Node  
   - Purpose: Receive transcribed user speech text.  

7. **Create Langchain Memory Buffer Window Node: "Conversation Memory"**  
   - Session Key: `voice-assistant-session` (static key)  
   - Session ID Type: Custom Key  
   - Context Window Length: 30 messages  

8. **Create Langchain Agent Node: "Process User Query"**  
   - Text: Expression `={{ $json.body.question }}`  
   - System Message: "You are a helpful AI assistant. Respond in a friendly and conversational manner."  
   - Prompt Type: Define  
   - Connect AI Memory input to "Conversation Memory" node.  

9. **Connect "Audio Processing Endpoint" → "Process User Query"**  
   - Connect "Conversation Memory" → AI Memory input of "Process User Query"  

10. **Create Langchain OpenAI Node: "Generate Voice Response" (Optional / Currently Disabled)**  
    - Resource: Audio  
    - Input: Expression `={{ $json.output }}` (output text from AI agent)  
    - Voice: `onyx` (or choose preferred voice)  
    - Credentials: OpenAI API key with TTS access  
    - Note: Enable when ready to convert text to speech.  

11. **Connect "Process User Query" → "Generate Voice Response"**

12. **Create Respond to Webhook Node: "Send Audio Response"**  
    - Respond With: Binary  
    - Purpose: Return audio binary data to frontend.  

13. **Connect "Generate Voice Response" → "Send Audio Response"**

14. **Connect "Audio Processing Endpoint" → "Process User Query"**  
    - Ensure that the webhook node is configured to accept JSON POST containing `question` field.  

15. **Update Frontend HTML Node**  
    - Replace placeholder webhook URL in JavaScript fetch call with the full URL of the "Audio Processing Endpoint" webhook.  

16. **Add Credentials:**  
    - OpenAI API credentials for both "Process User Query" and "Generate Voice Response" nodes.  
    - Ensure API keys have access to GPT-4o-mini and Text-to-Speech APIs.  

17. **Test:**  
    - Open the `voice-assistant` webhook URL in a modern browser (Chrome recommended).  
    - Click the orb, allow microphone access, speak a query.  
    - Confirm that response audio plays back from the AI.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| This workflow creates a voice-activated AI assistant interface running directly in the browser, combining Web Speech API, OpenAI GPT-4o-mini, and Text-to-Speech for interactive voice conversations.                                           | General workflow purpose                       |
| Requires modern browsers supporting `webkitSpeechRecognition` (Chrome, Edge, Safari).                                                                                                                                                        | Browser requirements                           |
| The workflow uses a static session key for memory, which may limit multi-user scalability; consider dynamic session keys for multi-user deployments.                                                                                         | Conversation Memory limitation                 |
| Demo video of the workflow in action: https://youtu.be/0bMdJcRMnZY                                                                                                                                                                            | Demo video link                                |
| Voice options include: alloy, echo, fable, onyx, nova, shimmer (different voice styles selectable in TTS node).                                                                                                                              | Voice customization in Generate Voice Response|
| Frontend UI customization possible via CSS in the HTML node to modify colors, animations, and orb size.                                                                                                                                       | UI customization                               |
| Setup instructions emphasize adding OpenAI credentials and updating webhook URL in the frontend HTML before testing.                                                                                                                         | Setup instructions                             |
| The "Generate Voice Response" node is currently disabled, so voice output will not be generated unless enabled and configured properly with credentials.                                                                                    | Important operational note                      |
| The frontend fetch call expects binary audio response from the backend webhook, so ensure response mode and content type are correctly set to avoid playback errors.                                                                        | Integration detail                             |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---