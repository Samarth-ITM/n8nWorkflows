WhatsApp Customer Support with Claude AI, Google Docs & Multilingual Capabilities

https://n8nworkflows.xyz/workflows/whatsapp-customer-support-with-claude-ai--google-docs---multilingual-capabilities-9027


# WhatsApp Customer Support with Claude AI, Google Docs & Multilingual Capabilities

### 1. Workflow Overview

This workflow implements a multilingual WhatsApp customer support system using Claude AI (via LangChain), Google Docs as a knowledge base, and media processing capabilities. It is designed to handle diverse incoming WhatsApp message types (text, voice notes, images), transcribe or analyze media content, and respond with accurate, concise, and contextually appropriate answers in English or Roman Urdu.

The workflow is logically divided into four blocks:

- **1.1 Input Reception & Classification**: Receives WhatsApp messages and classifies them by type (text, voice, image).
- **1.2 Multi-Modal Content Processing**: Downloads and processes audio and image content, converts them to text prompts.
- **1.3 AI Customer Support Engine**: Runs an AI agent with conversation memory, multilingual support, and Google Docs integration to generate knowledgeable responses.
- **1.4 Response Delivery System**: Sends AI-generated replies back to the user on WhatsApp.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Classification

**Overview:**  
This block listens for incoming WhatsApp messages using a webhook trigger. It then inspects the message content to determine if it is a text message, voice note, or image, routing flow accordingly.

**Nodes Involved:**  
- WhatsApp Trigger  
- Check Input Type (Switch)

**Node Details:**  

- **WhatsApp Trigger**  
  - *Type:* WhatsApp Trigger node  
  - *Role:* Listens for incoming WhatsApp messages via webhook  
  - *Config:* Monitors message updates, uses WhatsApp API credentials  
  - *Key expressions:* None (raw incoming JSON)  
  - *Input:* External webhook events  
  - *Output:* JSON with message details (text, audio, image)  
  - *Edge cases:* Webhook connectivity issues, auth errors, malformed messages

- **Check Input Type**  
  - *Type:* Switch node  
  - *Role:* Classifies incoming message type based on JSON content  
  - *Config:* Checks existence of audio, image, or text body fields to route to Voice, Image, or Text outputs respectively  
  - *Key expressions:* Uses JSON path expressions such as `{{$json.messages[0].audio}}` and `{{$json.messages[0].text.body}}`  
  - *Input:* Output from WhatsApp Trigger  
  - *Output:* Routes flow to three distinct paths based on media type  
  - *Edge cases:* Messages with multiple media types ambiguous; missing expected fields

---

#### 2.2 Multi-Modal Content Processing

**Overview:**  
Processes media-specific content by retrieving media URLs, downloading files, and converting content into textual descriptions or transcriptions usable by the AI agent.

**Nodes Involved:**  
- Get Audio URL  
- Download Audio  
- Transcribe Audio  
- Audio Prompt  
- Get Image URL  
- Download Image  
- Analyze Image  
- Image + Text Prompt  
- Text Only Prompt

**Node Details:**  

- **Get Audio URL**  
  - *Type:* WhatsApp node (media URL retrieval)  
  - *Role:* Retrieves download URL for incoming audio message  
  - *Config:* Uses media ID from WhatsApp Trigger message JSON  
  - *Input:* Voice path from Check Input Type  
  - *Output:* URL JSON for audio file  
  - *Edge cases:* Media unavailable, expired URL, API auth failure

- **Download Audio**  
  - *Type:* HTTP Request node  
  - *Role:* Downloads audio file from provided URL  
  - *Config:* Uses header authentication credentials  
  - *Input:* URL from Get Audio URL  
  - *Output:* Binary audio data  
  - *Edge cases:* Download failure, timeout, invalid auth

- **Transcribe Audio**  
  - *Type:* LangChain OpenAI node (audio transcribe)  
  - *Role:* Converts audio binary to text transcription using Whisper model  
  - *Config:* Uses Sycorda OpenAI API credentials  
  - *Input:* Audio binary from Download Audio  
  - *Output:* Transcribed text JSON  
  - *Edge cases:* Poor audio quality, transcription errors, API limits

- **Audio Prompt**  
  - *Type:* Set node  
  - *Role:* Formats transcription text into prompt field `text` for AI agent  
  - *Config:* Assigns `text` property from transcription output  
  - *Input:* Transcribed text from Transcribe Audio  
  - *Output:* Text prompt JSON  
  - *Edge cases:* Empty transcription results

- **Get Image URL**  
  - *Type:* WhatsApp node (media URL retrieval)  
  - *Role:* Retrieves download URL for incoming image message  
  - *Config:* Uses media ID from WhatsApp Trigger message JSON  
  - *Input:* Image path from Check Input Type  
  - *Output:* URL JSON for image file  
  - *Edge cases:* Media unavailable, expired URL, API auth failure

- **Download Image**  
  - *Type:* HTTP Request node  
  - *Role:* Downloads image file from provided URL  
  - *Config:* Header authentication credentials  
  - *Input:* URL from Get Image URL  
  - *Output:* Binary image data  
  - *Edge cases:* Download failure, timeout, invalid auth

- **Analyze Image**  
  - *Type:* LangChain OpenAI node (image analysis)  
  - *Role:* Analyzes image content using GPT-4 Vision capable model, produces detailed description  
  - *Config:* Uses Sycorda OpenAI API credentials, model "chatgpt-4o-latest"  
  - *Input:* Image binary from Download Image  
  - *Output:* Descriptive text content JSON  
  - *Edge cases:* Poor image quality, analysis errors, API rate limits

- **Image + Text Prompt**  
  - *Type:* Set node  
  - *Role:* Builds prompt text combining image description and user’s image caption or fallback  
  - *Config:* Constructs prompt with image content and original WhatsApp image caption  
  - *Input:* Image description from Analyze Image, WhatsApp Trigger JSON  
  - *Output:* Combined prompt text JSON  
  - *Edge cases:* Missing captions or description

- **Text Only Prompt**  
  - *Type:* Set node  
  - *Role:* Extracts plain text message body for AI prompt  
  - *Config:* Takes text body from WhatsApp Trigger JSON  
  - *Input:* Text path from Check Input Type  
  - *Output:* Text prompt JSON  
  - *Edge cases:* Empty text messages

---

#### 2.3 AI-Powered Customer Support Engine

**Overview:**  
This block powers the conversational AI agent that interprets the input text prompt, maintains conversation memory, retrieves knowledge base info from Google Docs, and generates concise, context-aware multilingual answers.

**Nodes Involved:**  
- AI Agent (LangChain)  
- Simple Memory  
- Get a document in Google Docs  
- OpenRouter Chat Model

**Node Details:**  

- **AI Agent**  
  - *Type:* LangChain Agent node  
  - *Role:* Core conversational AI interpreting prompts, generating responses based on system instruction and retrieved docs  
  - *Config:*  
    - Uses dynamic input `text` from prompt nodes  
    - System message defines customer support role, multilingual and media handling rules, and strict knowledge base usage from Google Docs  
    - Integrates:  
      - Google Docs node as AI tool for document retrieval  
      - OpenRouter Chat Model as language model backend (Claude Sonnet 4)  
      - Simple Memory Buffer for context  
  - *Input:* Text prompt from media processing nodes or Text Only Prompt  
  - *Output:* AI-generated response JSON with `output` text  
  - *Edge cases:* API failures, doc retrieval errors, prompt errors, memory overflow

- **Simple Memory**  
  - *Type:* LangChain Memory Buffer Window node  
  - *Role:* Maintains recent conversation context (last 20 turns) keyed by user phone number  
  - *Config:* Session key based on WhatsApp sender number to isolate conversations  
  - *Input:* Conversation text from AI Agent  
  - *Output:* Context fed back into AI Agent for continuity  
  - *Edge cases:* Memory size limits, session key mismatches

- **Get a document in Google Docs**  
  - *Type:* Google Docs Tool node  
  - *Role:* Fetches company knowledge base document content by fixed URL  
  - *Config:* OAuth2 credentials for Google Docs API, document URL is hardcoded for Sycorda info  
  - *Input:* Invoked by AI Agent as AI tool for information retrieval  
  - *Output:* Document text available to AI Agent for answering queries  
  - *Edge cases:* OAuth token expiry, document access permission issues

- **OpenRouter Chat Model**  
  - *Type:* LangChain OpenRouter Chat Model node  
  - *Role:* Provides Claude Sonnet 4 conversational AI backend for the AI Agent  
  - *Config:* Uses OpenRouter API credentials  
  - *Input:* Receives prompt and context from AI Agent node  
  - *Output:* Generated conversational text response  
  - *Edge cases:* API rate limits, network errors, model unavailability

---

#### 2.4 Response Delivery System

**Overview:**  
This block sends the AI-generated text responses back to the WhatsApp user, ensuring proper formatting and addressing.

**Nodes Involved:**  
- Respond with Text

**Node Details:**  

- **Respond with Text**  
  - *Type:* WhatsApp node (send message)  
  - *Role:* Sends the AI Agent’s response text back to the user’s phone number on WhatsApp  
  - *Config:*  
    - Uses WhatsApp API credentials  
    - Dynamically sets `recipientPhoneNumber` from original message sender  
    - Sends message body `textBody` from AI Agent output `output` property  
  - *Input:* AI Agent response  
  - *Output:* Confirmation of message sending (status)  
  - *Edge cases:* WhatsApp API errors, invalid phone numbers, message size limits

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                       | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                  |
|-------------------------|----------------------------------|-------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| WhatsApp Trigger        | n8n-nodes-base.whatsAppTrigger    | Receives incoming WhatsApp messages | None                        | Check Input Type             | STEP 1: WhatsApp Message Reception & Input Classification - Receives and categorizes incoming messages       |
| Check Input Type        | n8n-nodes-base.switch             | Classifies message type (voice, image, text) | WhatsApp Trigger             | Get Audio URL, Get Image URL, Text Only Prompt | STEP 1: WhatsApp Message Reception & Input Classification                                                    |
| Get Audio URL           | n8n-nodes-base.whatsApp           | Retrieves audio file URL             | Check Input Type (Voice)     | Download Audio               | STEP 2: Multi-Modal Content Processing - Media URL retrieval                                                 |
| Download Audio          | n8n-nodes-base.httpRequest        | Downloads audio file                 | Get Audio URL               | Transcribe Audio             | STEP 2: Multi-Modal Content Processing - Media file download                                                 |
| Transcribe Audio        | @n8n/n8n-nodes-langchain.openAi  | Converts audio to text transcription | Download Audio              | Audio Prompt                | STEP 2: Multi-Modal Content Processing - Audio transcription                                                 |
| Audio Prompt            | n8n-nodes-base.set                | Formats transcription text for AI   | Transcribe Audio            | AI Agent                    | STEP 2: Multi-Modal Content Processing - Prompt formatting                                                   |
| Get Image URL           | n8n-nodes-base.whatsApp           | Retrieves image file URL             | Check Input Type (Image)     | Download Image              | STEP 2: Multi-Modal Content Processing - Media URL retrieval                                                 |
| Download Image          | n8n-nodes-base.httpRequest        | Downloads image file                 | Get Image URL               | Analyze Image               | STEP 2: Multi-Modal Content Processing - Media file download                                                 |
| Analyze Image           | @n8n/n8n-nodes-langchain.openAi  | Describes image content via AI      | Download Image              | Image + Text Prompt         | STEP 2: Multi-Modal Content Processing - Image content analysis                                              |
| Image + Text Prompt     | n8n-nodes-base.set                | Combines image description & caption | Analyze Image              | AI Agent                    | STEP 2: Multi-Modal Content Processing - Prompt formatting                                                   |
| Text Only Prompt        | n8n-nodes-base.set                | Extracts text message for AI prompt | Check Input Type (Text)      | AI Agent                    | STEP 2: Multi-Modal Content Processing - Prompt formatting                                                   |
| AI Agent                | @n8n/n8n-nodes-langchain.agent   | Conversational AI with memory & doc retrieval | Audio Prompt, Image + Text Prompt, Text Only Prompt | Respond with Text           | STEP 3: AI-Powered Customer Support Engine - Core AI with Google Docs & multilingual support                  |
| Simple Memory           | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversation context      | AI Agent                   | AI Agent                    | STEP 3: AI-Powered Customer Support Engine - Conversation memory                                              |
| Get a document in Google Docs | n8n-nodes-base.googleDocsTool   | Retrieves company knowledge base    | AI Agent (AI tool)          | AI Agent                   | STEP 3: AI-Powered Customer Support Engine - Knowledge base integration                                      |
| OpenRouter Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provides Claude Sonnet 4 language model | AI Agent (ai_languageModel) | AI Agent                   | STEP 3: AI-Powered Customer Support Engine - Language model backend                                          |
| Respond with Text       | n8n-nodes-base.whatsApp           | Sends AI response back to WhatsApp user | AI Agent                   | None                       | STEP 4: Response Delivery System - Sends AI-generated replies via WhatsApp                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger node**  
   - Type: WhatsApp Trigger  
   - Configure webhook for incoming message updates  
   - Set credentials for WhatsApp API access  
   - Output: Incoming WhatsApp JSON message

2. **Add Switch node (Check Input Type)**  
   - Type: Switch  
   - Add 3 outputs named: Voice, Image, Text  
   - Configure conditions:  
     - Voice: Check if `messages[0].audio` exists  
     - Image: Check if `messages[0].image` exists  
     - Text: Check if `messages[0].text.body` exists  
   - Connect WhatsApp Trigger output to this node’s input

3. **Voice Path Setup:**  
   - Add WhatsApp node (Get Audio URL)  
     - Operation: mediaUrlGet  
     - MediaGetId: `{{$json.messages[0].audio.id}}`  
     - Credentials: WhatsApp API  
   - Connect Check Input Type Voice output to Get Audio URL input  

   - Add HTTP Request node (Download Audio)  
     - URL: `{{$json.url}}` (from Get Audio URL)  
     - Auth: Header Auth with API key  
   - Connect Get Audio URL output to Download Audio input  

   - Add LangChain OpenAI node (Transcribe Audio)  
     - Resource: Audio  
     - Operation: Transcribe  
     - Credentials: OpenAI API (Whisper capable)  
   - Connect Download Audio output to Transcribe Audio input  

   - Add Set node (Audio Prompt)  
     - Assign variable `text` = `{{$json.text}}` (transcription)  
   - Connect Transcribe Audio output to Audio Prompt input  

   - Connect Audio Prompt output to AI Agent input

4. **Image Path Setup:**  
   - Add WhatsApp node (Get Image URL)  
     - Operation: mediaUrlGet  
     - MediaGetId: `{{$json.messages[0].image.id}}`  
     - Credentials: WhatsApp API  
   - Connect Check Input Type Image output to Get Image URL input  

   - Add HTTP Request node (Download Image)  
     - URL: `{{$json.url}}` (from Get Image URL)  
     - Auth: Header Auth  
   - Connect Get Image URL output to Download Image input  

   - Add LangChain OpenAI node (Analyze Image)  
     - Model: chatgpt-4o-latest or GPT-4 Vision capable  
     - Operation: Analyze image  
     - Input type: base64 (binary data)  
     - Credentials: OpenAI API  
   - Connect Download Image output to Analyze Image input  

   - Add Set node (Image + Text Prompt)  
     - Assign variable `text` =  
       ```
       # The user provided the following image and text.

       ## IMAGE CONTENT:
       {{$json.content}}

       ## USER MESSAGE:
       {{$('WhatsApp Trigger').item.json.messages[0].image.caption || "Describe the image"}}
       ```  
   - Connect Analyze Image output to Image + Text Prompt input  

   - Connect Image + Text Prompt output to AI Agent input

5. **Text Path Setup:**  
   - Add Set node (Text Only Prompt)  
     - Assign variable `text` = `{{$('WhatsApp Trigger').item.json.messages[0].text.body}}`  
   - Connect Check Input Type Text output to Text Only Prompt input  

   - Connect Text Only Prompt output to AI Agent input

6. **AI Agent Setup:**  
   - Add LangChain Agent node  
   - Configure:  
     - Text input: `={{ $json.text }}` (from prompt nodes)  
     - System message: Detailed prompt specifying multilingual support, Google Docs knowledge base usage, tone, language rules (copy from provided system message)  
     - AI Tool: Add Google Docs Tool node (see below)  
     - AI Language Model: Add OpenRouter Chat Model node (Claude Sonnet 4)  
     - AI Memory: Add Simple Memory Buffer node (session key = sender phone number, window length 20)  
   - Connect prompt nodes (Audio Prompt, Image + Text Prompt, Text Only Prompt) to AI Agent input  
   - Connect Simple Memory output to AI Agent ai_memory input  
   - Connect OpenRouter Chat Model output to AI Agent ai_languageModel input  
   - Connect Google Docs Tool output to AI Agent ai_tool input

7. **Google Docs Tool Setup:**  
   - Add Google Docs Tool node  
   - Operation: Get document by URL  
   - Document URL: `1H9ZBePkINDHi2c-IfsxRaVjF5iiGkwimxgJLICyGZco` (Sycorda knowledge base)  
   - Credential: Google OAuth2 API with access rights to the document

8. **OpenRouter Chat Model Setup:**  
   - Add LangChain OpenRouter Chat Model node  
   - Model: `anthropic/claude-sonnet-4`  
   - Credential: OpenRouter API account  

9. **Simple Memory Buffer Setup:**  
   - Add LangChain Memory Buffer Window node  
   - Session Key: `={{ $('WhatsApp Trigger').item.json.messages[0].from }}` (unique user)  
   - Context window length: 20 turns  

10. **Response Delivery Setup:**  
    - Add WhatsApp node (Respond with Text)  
    - Operation: Send message  
    - Text Body: `={{ $json.output }}` (AI Agent output)  
    - Recipient Phone Number: `={{ $('WhatsApp Trigger').item.json.messages[0].from }}`  
    - Credentials: WhatsApp API  
    - Connect AI Agent output to Respond with Text input

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow employs strict language and tone rules for English and Roman Urdu to maintain natural, human-like customer support dialogue. | System message in AI Agent node                                                                |
| Google Docs is used as a single source of truth knowledge base; answers must strictly derive from this document.                          | Google Docs Tool node configuration                                                            |
| Voice notes are transcribed using OpenAI Whisper; images are analyzed with GPT-4 Vision to extract meaningful text content.              | Transcribe Audio and Analyze Image nodes                                                       |
| Conversation memory preserves context up to last 20 exchanges keyed by WhatsApp user phone number.                                         | Simple Memory Buffer node                                                                      |
| Workflow uses OpenRouter API for Claude Sonnet 4 model, requiring proper credential setup to ensure response quality.                    | OpenRouter Chat Model node                                                                     |
| WhatsApp API authentication and media download require valid header authentication credentials, managed separately in HTTP Request nodes. | Nodes: Get Audio/Image URL, Download Audio/Image                                              |
| Sticky notes in the workflow visually group logical blocks and provide step descriptions.                                                  | Visible in workflow editor for maintenance and clarity                                         |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible.