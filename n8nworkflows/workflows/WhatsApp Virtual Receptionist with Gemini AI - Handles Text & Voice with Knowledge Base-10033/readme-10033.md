WhatsApp Virtual Receptionist with Gemini AI - Handles Text & Voice with Knowledge Base

https://n8nworkflows.xyz/workflows/whatsapp-virtual-receptionist-with-gemini-ai---handles-text---voice-with-knowledge-base-10033


# WhatsApp Virtual Receptionist with Gemini AI - Handles Text & Voice with Knowledge Base

### 1. Workflow Overview

This workflow implements a **WhatsApp Virtual Receptionist powered by Google Gemini AI** and a knowledge base stored in Pinecone vector store. It handles **both text and voice messages** from WhatsApp users, processes them through AI models for understanding, and replies with relevant, context-aware answers or product recommendations.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception and Message Type Detection:** Captures incoming WhatsApp messages (text or voice), detects message type, and routes accordingly.
- **1.2 Voice Message Processing:** For voice notes, downloads the audio, converts it to Base64, transcribes it using Google Gemini speech-to-text, and prepares the text for AI processing.
- **1.3 AI Processing with Knowledge Base Integration:** Uses LangChain’s AI Agent node powered by Google Gemini chat model, augmented with a Pinecone vector store containing company product and service data, and maintains conversation context with a memory buffer. The AI Agent produces direct, professional responses or product recommendations, which are then sent back to the user via WhatsApp.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Message Type Detection

**Overview:**  
This block receives incoming WhatsApp messages in real-time, detects whether the message contains audio or text, and routes the flow accordingly.

**Nodes Involved:**  
- WhatsApp Trigger  
- Audio/Message (Switch)

**Node Details:**  

- **WhatsApp Trigger**  
  - Type: WhatsApp Trigger (Webhook)  
  - Role: Entry point for all WhatsApp messages (text or audio).  
  - Config: Listens for "messages" updates, authenticated with WhatsApp OAuth credentials.  
  - Input: Incoming WhatsApp messages.  
  - Output: Raw message JSON forwarded to the switch.  
  - Edge Cases: Webhook downtime, credential expiration, unsupported message types.  

- **Audio/Message (Switch)**  
  - Type: Switch node  
  - Role: Routes workflow based on message type presence: Audio or Text.  
  - Config: Checks if `$json.messages[0].audio` exists → route to Audio; else if `$json.messages[0].text` exists → route to Text.  
  - Input: Message JSON from WhatsApp Trigger.  
  - Output: Two branches - Audio and Text.  
  - Edge Cases: Message without audio or text fields, malformed messages.  
  - Notes: Strict existence check for robust routing.  

---

#### 2.2 Voice Message Processing

**Overview:**  
This block downloads the voice message audio file from WhatsApp, converts it to Base64 format, sends it to Google Gemini’s speech-to-text API for transcription, and prepares the transcribed text to be processed by the AI Agent.

**Nodes Involved:**  
- Get Audio URL  
- Audio Download  
- Audio Convert  
- Gemini speech to text  
- Audio Prompt

**Node Details:**  

- **Get Audio URL**  
  - Type: WhatsApp node (mediaUrlGet)  
  - Role: Retrieves a downloadable URL for the audio file from WhatsApp servers.  
  - Config: Uses `mediaGetId` from incoming audio message ID.  
  - Input: Audio message metadata.  
  - Output: JSON containing URL for audio download.  
  - Credentials: WhatsApp API credentials.  
  - Edge Cases: Media not available, expired URL, API rate limits.  

- **Audio Download**  
  - Type: HTTP Request  
  - Role: Downloads the audio file from the URL obtained.  
  - Config: Uses generic HTTP header authentication for authorization.  
  - Input: URL from Get Audio URL node.  
  - Output: Binary audio data.  
  - Credentials: HTTP header authentication with WhatsApp download token.  
  - Edge Cases: Download failures, invalid URL, authentication errors.  

- **Audio Convert**  
  - Type: Code node (JavaScript)  
  - Role: Converts downloaded binary audio to Base64 encoded string for API consumption.  
  - Code: Extracts binary data under key "data", outputs JSON with `base64Audio` and `mimeType`.  
  - Input: Binary audio data from Audio Download.  
  - Output: JSON with Base64 audio and MIME type.  
  - Edge Cases: Missing binary data, unsupported MIME types.  

- **Gemini speech to text**  
  - Type: HTTP Request  
  - Role: Calls Google Gemini speech-to-text API to transcribe audio.  
  - Config: POST request with JSON body including Base64 audio and transcription prompt.  
  - Input: Base64 audio and mime type from Audio Convert.  
  - Output: Transcription result JSON.  
  - Credentials: Uses no explicit credentials configured here (likely public endpoint or inherited).  
  - Edge Cases: API errors, timeout, transcription inaccuracies.  

- **Audio Prompt**  
  - Type: Set node  
  - Role: Extracts the transcribed text from API response and sets it in a variable for the AI Agent input.  
  - Config: Assigns `candidates[0].content.parts[0].text` to the same path in output JSON.  
  - Input: Transcription JSON.  
  - Output: JSON prepared for AI Agent consumption.  
  - Edge Cases: Unexpected transcription response format, missing transcription text.  

---

#### 2.3 AI Processing with Knowledge Base Integration

**Overview:**  
This block uses an AI Agent powered by Google Gemini chat models, enhanced with a Pinecone vector store containing company knowledge and a memory buffer to maintain conversational context. It produces direct, professional responses or product recommendations and sends the reply back via WhatsApp.

**Nodes Involved:**  
- AI Agent  
- Simple Memory  
- Answer questions with a vector store  
- Pinecone Vector Store  
- Embeddings Google Gemini  
- Google Gemini Chat Model  
- Google Gemini Chat Model1  
- Send message

**Node Details:**  

- **AI Agent**  
  - Type: LangChain AI Agent node  
  - Role: Core AI node that generates responses based on input text, memory, and vector store results.  
  - Config:  
    - Input text combines user message and knowledge base content.  
    - System message enforces strict rules: respond only in English, start response immediately with answer or product recommendation, professional tone, and no source acknowledgment.  
    - Uses multiple tools: memory, vector store, and language models.  
  - Inputs:  
    - Text from user or transcribed audio (from Audio Prompt or directly from Text branch).  
    - Context from Simple Memory.  
    - Documents from vector store query.  
  - Outputs: Response JSON with message to send.  
  - Edge Cases: Model API errors, memory overflow, vector store unavailability, inconsistent responses.  

- **Simple Memory**  
  - Type: LangChain memory buffer window  
  - Role: Maintains recent conversational context limited to last 20 messages per user session (keyed by WhatsApp user ID).  
  - Config: Session key derived from WhatsApp user ID.  
  - Inputs/Outputs: Connects to AI Agent as memory provider.  
  - Edge Cases: Memory loss on node restart, session key mismatch.  

- **Answer questions with a vector store**  
  - Type: LangChain tool vector store  
  - Role: Queries Pinecone vector database for documents relevant to the user query.  
  - Config: Description clarifies it returns info about company products and services.  
  - Inputs: From Pinecone vector store output and language model.  
  - Outputs: Documents passed to AI Agent.  
  - Edge Cases: Empty results, Pinecone API errors.  

- **Pinecone Vector Store**  
  - Type: LangChain vector store Pinecone  
  - Role: Connects to Pinecone index "superclean" storing product & service knowledge.  
  - Config: Uses cached "superclean" index.  
  - Credentials: Pinecone API account.  
  - Inputs: Embeddings output.  
  - Outputs: Vector similarity search results.  
  - Edge Cases: Index unavailability, API limits.  

- **Embeddings Google Gemini**  
  - Type: LangChain embeddings node  
  - Role: Converts input text into embeddings suitable for Pinecone vector queries.  
  - Credentials: Google Gemini (PaLM) API.  
  - Inputs: User input text.  
  - Outputs: Embeddings for vector store.  
  - Edge Cases: Embedding API errors, rate limits.  

- **Google Gemini Chat Model**  
  - Type: LangChain language model (chat) node  
  - Role: Provides conversational AI responses, used by AI Agent.  
  - Credentials: Google Gemini (PaLM) API.  
  - Edge Cases: API downtime, quota exhaustion.  

- **Google Gemini Chat Model1**  
  - Type: LangChain language model (chat) node  
  - Role: Used by vector store tool to help answer questions.  
  - Credentials: Google Gemini (PaLM) API.  
  - Edge Cases: Same as above.  

- **Send message**  
  - Type: WhatsApp node (send)  
  - Role: Sends the AI Agent's response back to the user on WhatsApp.  
  - Config: Sends textBody from AI Agent output, uses phoneNumberId and recipientPhoneNumber from WhatsApp Trigger node.  
  - Credentials: WhatsApp API credentials.  
  - Edge Cases: Message send failures, invalid phone number, WhatsApp API limits.  

---

### 3. Summary Table

| Node Name                  | Node Type                                 | Functional Role                                  | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                                     |
|----------------------------|-------------------------------------------|-------------------------------------------------|---------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------|
| WhatsApp Trigger           | WhatsApp Trigger (Webhook)                | Receives incoming WhatsApp messages              | —                         | Audio/Message               | 💬 Workflow Overview: AI-powered WhatsApp agent handling text and voice messages.                             |
| Audio/Message              | Switch                                   | Detects message type (Audio or Text)              | WhatsApp Trigger           | Get Audio URL (Audio), AI Agent (Text) |                                                                                                               |
| Get Audio URL              | WhatsApp (mediaUrlGet)                    | Retrieves downloadable audio URL                   | Audio/Message (Audio)      | Audio Download              | 🎧 Voice Message Handling: Retrieves and transcribes voice notes.                                             |
| Audio Download             | HTTP Request                             | Downloads audio file from WhatsApp URL             | Get Audio URL              | Audio Convert              |                                                                                                               |
| Audio Convert              | Code                                     | Converts audio binary to Base64                     | Audio Download             | Gemini speech to text       |                                                                                                               |
| Gemini speech to text      | HTTP Request                             | Transcribes audio to text via Google Gemini API    | Audio Convert              | Audio Prompt               |                                                                                                               |
| Audio Prompt              | Set                                      | Sets transcription text for AI Agent input         | Gemini speech to text      | AI Agent                   |                                                                                                               |
| AI Agent                  | LangChain AI Agent                       | Processes user query with context and knowledge base | Audio Prompt (Audio), Audio/Message (Text), Simple Memory, Answer questions with a vector store | Send message               | 🧠 AI Agent + Knowledge Base: Uses Gemini, Pinecone, and Memory for professional, context-aware responses.   |
| Simple Memory             | LangChain Memory Buffer                  | Maintains recent conversational context            | —                         | AI Agent                   |                                                                                                               |
| Answer questions with a vector store | LangChain Tool Vector Store             | Retrieves relevant documents from vector DB         | Pinecone Vector Store, Google Gemini Chat Model1 | AI Agent                   |                                                                                                               |
| Pinecone Vector Store     | LangChain Vector Store Pinecone          | Connects to Pinecone index with company knowledge  | Embeddings Google Gemini   | Answer questions with a vector store |                                                                                                               |
| Embeddings Google Gemini  | LangChain Embeddings Google Gemini       | Generates embeddings for vector search              | AI Agent input             | Pinecone Vector Store      |                                                                                                               |
| Google Gemini Chat Model  | LangChain Language Model (Chat)           | Provides conversational AI model for AI Agent       | —                         | AI Agent                   |                                                                                                               |
| Google Gemini Chat Model1 | LangChain Language Model (Chat)           | Provides AI model for vector store tool              | —                         | Answer questions with a vector store |                                                                                                               |
| Send message              | WhatsApp Send Message                      | Sends AI Agent’s response back to WhatsApp user     | AI Agent                   | —                          |                                                                                                               |
| Sticky Note               | Sticky Note                              | Provides overview of workflow                        | —                         | —                          | 💬 Workflow Overview: AI-powered WhatsApp agent that handles text and voice messages with AI integration.     |
| Sticky Note1              | Sticky Note                              | Explains voice message processing                    | —                         | —                          | 🎧 Voice Message Handling: Audio retrieval, transcription, and AI response.                                   |
| Sticky Note2              | Sticky Note                              | Describes AI Agent and knowledge base integration   | —                         | —                          | 🧠 AI Agent + Knowledge Base: AI with context and vector store for accurate, professional responses.          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Parameters: Listen for "messages" updates  
   - Credentials: WhatsApp OAuth2 (configured with valid WhatsApp Business API credentials)  
   - Position: Entry point  

2. **Add Switch Node "Audio/Message"**  
   - Type: Switch  
   - Conditions:  
     - Audio branch: `{{$json.messages[0].audio}}` exists  
     - Text branch: `{{$json.messages[0].text}}` exists  
   - Connect WhatsApp Trigger main output to this switch input  

3. **Voice Message Branch:**  
   a. **Create "Get Audio URL" Node**  
      - Type: WhatsApp (mediaUrlGet)  
      - Parameter: `mediaGetId` set to `{{$json.messages[0].audio.id}}`  
      - Credentials: WhatsApp API credentials  
      - Connect Audio output of Switch node to this node  

   b. **Create "Audio Download" Node**  
      - Type: HTTP Request  
      - Parameter: URL from previous node output `{{$json.url}}`  
      - Authentication: HTTP Header Auth with WhatsApp download token  
      - Connect Get Audio URL node to this node  

   c. **Create "Audio Convert" Node (Code Node)**  
      - Type: Code  
      - JavaScript code snippet:  
        ```javascript  
        const binaryKey = 'data';  
        const binaryData = items[0].binary[binaryKey].data;  
        const mimeType = items[0].binary[binaryKey].mimeType || 'audio/ogg';  
        return [{ json: { base64Audio: binaryData, mimeType } }];  
        ```  
      - Connect Audio Download node to this node  

   d. **Create "Gemini speech to text" Node (HTTP Request)**  
      - Type: HTTP Request  
      - Method: POST  
      - JSON Body:  
        ```json  
        {  
          "contents": [  
            {  
              "parts": [  
                { "inlineData": { "mimeType": "{{$json.mimeType}}", "data": "{{$json.base64Audio}}" } },  
                { "text": "Transcribe this audio and return only the plain spoken words." }  
              ]  
            }  
          ]  
        }  
        ```  
      - Connect Audio Convert node to this node  

   e. **Create "Audio Prompt" Node (Set)**  
      - Type: Set  
      - Assign `candidates[0].content.parts[0].text` from transcription response to the same path in output JSON  
      - Connect Gemini speech to text node to this node  

4. **Text Message Branch:**  
   - Connect Text output of Switch node directly to AI Agent input  

5. **Create AI Processing Nodes:**  

   a. **"Simple Memory" Node**  
      - Type: LangChain Memory Buffer Window  
      - Session key: `{{$('WhatsApp Trigger').item.json.contacts[0].wa_id}}`  
      - Context window length: 20  
      - Connect memory output to AI Agent memory input  

   b. **"Embeddings Google Gemini" Node**  
      - Type: LangChain Embeddings  
      - Credentials: Google Gemini (PaLM) API  
      - Connect input text to this node  

   c. **"Pinecone Vector Store" Node**  
      - Type: LangChain Vector Store Pinecone  
      - Index: "superclean" (cached)  
      - Credentials: Pinecone API account  
      - Connect embeddings output to this node  

   d. **"Google Gemini Chat Model1" Node**  
      - Type: LangChain Language Model (Chat)  
      - Model: "models/gemini-2.0-flash"  
      - Credentials: Google Gemini (PaLM) API  
      - Connect output to "Answer questions with a vector store" node  

   e. **"Answer questions with a vector store" Node**  
      - Type: LangChain Tool Vector Store  
      - Description: Returns documents related to company info, products, services  
      - Connect vector store and language model inputs appropriately  
      - Output connects to AI Agent tool input  

   f. **"Google Gemini Chat Model" Node**  
      - Type: LangChain Language Model (Chat)  
      - Credentials: Google Gemini (PaLM) API  
      - Connect to AI Agent language model input  

   g. **"AI Agent" Node**  
      - Type: LangChain AI Agent  
      - Parameters:  
        - Text to process: `{{$json.messages[0].text.body}} + {{$json.candidates[0].content.parts[0].text}}`  
        - System message enforcing: English only, immediate start with answer, no greetings, professional tone, product recommendations prioritized, no source mention.  
      - Inputs: Text (from Text branch or Audio Prompt), memory, vector store tool, language models.  
      - Output: Generates final response text  

6. **Create "Send message" Node**  
   - Type: WhatsApp Send  
   - Text body: `{{$json.output}}` from AI Agent  
   - PhoneNumberId: Fixed ID (e.g., "723548604171403")  
   - Recipient phone number: `{{$('WhatsApp Trigger').item.json.messages[0].from}}`  
   - Credentials: WhatsApp API credentials  
   - Connect AI Agent output to this node  

7. **Connect Audio Prompt Node to AI Agent node** for voice message flow.  
8. **Connect Text branch from Switch directly to AI Agent node** for text messages.  
9. **Connect AI Agent node output to Send message node** for final delivery.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| 💬 Workflow Overview: AI-powered WhatsApp agent that handles both text and voice messages, automatically detecting type and responding using AI.                                                                                                           | Sticky Note in workflow overview section                                                        |
| 🎧 Voice Message Handling: For voice notes, retrieves audio, converts to Base64, transcribes with Google Gemini, then sends transcription to AI Agent for response generation.                                                                              | Sticky Note near voice processing nodes                                                         |
| 🧠 AI Agent + Knowledge Base: AI Agent integrates Google Gemini chat models with Pinecone vector store of company knowledge and a memory buffer to enable context-aware, professional, and direct responses including product recommendations and quote handling. | Sticky Note near AI Agent node                                                                   |
| Official Pinecone Documentation: https://docs.pinecone.io/                                                                                                                                                                                                  | Useful for vector store setup and management                                                    |
| Google Gemini (PaLM) API Documentation: https://developers.generativeai.google                                                                                                    | Reference for AI and embedding model usage                                                     |
| WhatsApp Business API Documentation: https://developers.facebook.com/docs/whatsapp                                                                                                                                | Reference for WhatsApp Trigger and send message node setup                                     |

---

This documentation fully describes the WhatsApp Virtual Receptionist workflow, enabling developers and AI agents to understand, reproduce, and extend the solution with confidence.