Process Voice, Images & Documents with GPT-4o, MongoDB & Gmail Tools

https://n8nworkflows.xyz/workflows/process-voice--images---documents-with-gpt-4o--mongodb---gmail-tools-6211


# Process Voice, Images & Documents with GPT-4o, MongoDB & Gmail Tools

### 1. Workflow Overview

This n8n workflow implements a **Multi-Modal AI Agent Assistant** that processes diverse input types—voice notes, images, documents, and text messages—received via Telegram. It leverages the advanced OpenAI GPT-4o model for semantic understanding and MongoDB Atlas as a persistent vector memory store to recall contextually relevant information. Additionally, it integrates Gmail tools to enable email sending and searching capabilities through the AI agent.

The workflow is logically divided into two primary functional phases:

**1.1 Input Reception and Preprocessing**  
- Reception of messages from Telegram (voice, images, documents, or text)  
- Conditional routing based on input type  
- Downloading and converting files as needed  
- Transcription of audio, image description generation, and document text extraction  
- Graceful handling of unsupported file types  

**1.2 AI Processing, Memory Management & Response**  
- Formatted text inputs are passed to a Langchain AI Agent node configured with:  
  - MongoDB Atlas Vector Store Tool for retrieval-augmented generation (RAG)  
  - MongoDB Chat Memory for session conversation history  
  - OpenAI GPT-4o Chat Model for response generation  
  - Gmail tools for email sending and searching operations  
- The AI Agent synthesizes responses and sends them back to the Telegram user  

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Routing

**Overview:**  
This block captures incoming Telegram messages and classifies them by type (photo, voice, document, or text) to direct them down appropriate processing paths.

**Nodes Involved:**  
- Telegram Trigger  
- Switch

**Node Details:**  

- **Telegram Trigger**  
  - Type: Telegram Trigger node  
  - Role: Listens for new Telegram messages (all message types)  
  - Config: Listens to "message" updates via webhook, authenticated with Telegram API credentials  
  - Output: Passes incoming message JSON to the next node  
  - Edge cases: Missing or malformed messages; Telegram API connectivity issues  

- **Switch**  
  - Type: Switch node  
  - Role: Routes the message based on content type presence: photo, voice, document, or text  
  - Config: Checks existence of message.photo[3], message.voice, message.document, and message.text fields, outputting to respective branches  
  - Edge cases: Message missing expected fields; multiple types present (prioritizes photo > voice > document > text based on order)  

---

#### 2.2 Photo Processing

**Overview:**  
Processes photo messages by downloading the highest resolution photo, converting it to a standard format, performing AI-powered image analysis, and preparing textual content for AI ingestion.

**Nodes Involved:**  
- Get photo file  
- ConvertAPI HTTP Request  
- Analyze image  
- Get text from Image (Set node)

**Node Details:**  

- **Get photo file**  
  - Type: Telegram node (file download)  
  - Role: Downloads the photo file from Telegram servers using the file_id of the highest resolution photo (index 3)  
  - Config: Uses Telegram API credentials  
  - Output: Returns file data for next processing  
  - Failure modes: File not found; Telegram API rate limits  

- **ConvertAPI HTTP Request**  
  - Type: HTTP Request node  
  - Role: Converts the downloaded image to PNG format using ConvertAPI service for consistent AI input  
  - Config: POST to ConvertAPI endpoint with multipart-form data, includes API key in headers  
  - Output: Converted image URL passed on  
  - Failure modes: External API downtime, authentication failure, file conversion errors  

- **Analyze image**  
  - Type: Langchain OpenAI node (image resource)  
  - Role: Uses GPT-4o model to analyze image content, including object recognition and OCR text extraction, producing descriptive text  
  - Config: Set to analyze operation, with image URL input from previous node  
  - Output: JSON with description content  
  - Edge cases: Model errors, malformed image URLs, API rate limits  

- **Get text from Image (Set node)**  
  - Type: Set node  
  - Role: Formats the AI-generated image description text into a standardized string for AI Agent ingestion  
  - Config: Assigns `text` property with template including AI description content  
  - Output: Standardized text JSON to AI Agent  

---

#### 2.3 Voice Note Processing

**Overview:**  
Handles voice messages by downloading the audio file, transcribing it into text, and formatting the transcription for AI input.

**Nodes Involved:**  
- Get audio file  
- Transcribe a recording  
- Get text from Audio (Set node)

**Node Details:**  

- **Get audio file**  
  - Type: Telegram node (file download)  
  - Role: Downloads voice message audio using file_id  
  - Config: Authenticated Telegram API access  
  - Output: Audio file data for transcription  
  - Edge cases: File missing, Telegram API errors  

- **Transcribe a recording**  
  - Type: Langchain OpenAI node (audio resource)  
  - Role: Uses OpenAI Whisper transcription (via Langchain) to convert audio to text  
  - Config: Transcribe operation, OpenAI API credentials set  
  - Output: Text transcription of audio  
  - Failure modes: Audio format issues, speech recognition errors, API timeouts  

- **Get text from Audio (Set node)**  
  - Type: Set node  
  - Role: Prepares transcribed text in a standardized format for AI Agent input  
  - Config: Assigns `text` property with transcription string  
  - Output: Text JSON for AI Agent  

---

#### 2.4 Document Processing

**Overview:**  
Processes document uploads from Telegram, extracts text from PDFs, and handles unsupported file types with user feedback.

**Nodes Involved:**  
- Get a file  
- If (PDF check)  
- Extract from File  
- Get text from PDF (Set node)  
- Unsupported Input (Set node)

**Node Details:**  

- **Get a file**  
  - Type: Telegram node (file download)  
  - Role: Downloads uploaded document file  
  - Config: Uses file_id from message.document  
  - Output: File data and metadata  
  - Edge cases: Unsupported file formats, download failures  

- **If**  
  - Type: If node  
  - Role: Checks if downloaded file path contains "pdf" to branch processing  
  - Config: String contains condition on `result.file_path` includes "pdf"  
  - Output: True branch to PDF extraction, False to unsupported message  

- **Extract from File**  
  - Type: Extract from File node (n8n utility)  
  - Role: Extracts text content from PDF files  
  - Config: Operation set to pdf extraction  
  - Output: Extracted text content JSON  
  - Edge cases: Encrypted PDFs, scanned PDFs without OCR capability in node  

- **Get text from PDF (Set node)**  
  - Type: Set node  
  - Role: Formats extracted PDF text into a standard textual input for AI Agent  
  - Config: Assigns a template string including document title and content  
  - Output: Text JSON for AI Agent  

- **Unsupported Input (Set node)**  
  - Type: Set node  
  - Role: Returns a user-friendly message indicating unsupported file types (e.g., Word documents)  
  - Config: Static message string assigned to `text`  
  - Output: Text JSON for AI Agent  

---

#### 2.5 Text Message Processing

**Overview:**  
Directly formats incoming text messages from Telegram for AI processing.

**Nodes Involved:**  
- Get text from Message (Set node)

**Node Details:**  

- **Get text from Message**  
  - Type: Set node  
  - Role: Assigns the raw text message from Telegram to a `text` property for AI Agent consumption  
  - Config: Pulls `message.text` directly  
  - Output: JSON with `text` property  

---

#### 2.6 AI Agent Processing & Memory Integration

**Overview:**  
This core block contains the Langchain AI Agent node configured to leverage persistent memory (MongoDB Atlas), advanced GPT-4o language model, and integrated Gmail tools. It generates responses based on retrieved context and sends replies via Telegram.

**Nodes Involved:**  
- AI Agent  
- MongoDB Chat Memory  
- OpenAI Chat Model  
- Respond  
- Send a message in Gmail  
- Search for a messages in Gmail

**Node Details:**  

- **AI Agent**  
  - Type: Langchain Agent node  
  - Role: Central AI processing node that receives formatted text inputs, queries vector store and chat memory, uses GPT-4o to generate responses, and can invoke Gmail tools  
  - Config:  
    - System message instructs assistant to use MongoDB Atlas Vector Store Tool for context retrieval if needed  
    - Session key set by Telegram user ID for personalized memory  
  - Inputs: Receives `text` from various set nodes (image, audio, PDF, text)  
  - Outputs: Text response passed to Telegram Respond node  
  - Edge cases: MongoDB connection failures, OpenAI API errors, invalid queries, tool invocation errors  
  - Sub-workflow: Integrates MongoDB vector memory and Gmail tool nodes as custom tools  

- **MongoDB Chat Memory**  
  - Type: MongoDB memory node for Langchain  
  - Role: Stores and retrieves chat history for session context in AI conversations  
  - Config: Database `chat_messages`, session key from Telegram user ID, context window length 100 messages  
  - Input: Connects to AI Agent’s memory input  
  - Failure modes: MongoDB connectivity issues  

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model node  
  - Role: Provides GPT-4o model capabilities to synthesize AI Agent’s responses  
  - Config: Model set to "gpt-4.1-mini" as representative of GPT-4o; OpenAI API credentials set  
  - Connects as language model input to AI Agent  
  - Edge cases: API limits, model unavailability  

- **Respond**  
  - Type: Telegram node (send message)  
  - Role: Sends AI Agent’s generated text response back to the user on Telegram  
  - Config: Uses chat ID from original Telegram message sender, sends `output` text from AI Agent  
  - Edge cases: Telegram API errors, message too long for Telegram limits  

- **Send a message in Gmail**  
  - Type: Gmail Tool node  
  - Role: Enables AI Agent to send emails on behalf of user if instructed  
  - Config: Uses Gmail OAuth2 credentials  
  - Input: Connected as an AI tool to AI Agent  
  - Failure modes: Gmail API auth errors, quota limits  

- **Search for a messages in Gmail**  
  - Type: Gmail Tool node  
  - Role: Allows AI Agent to search user's Gmail inbox as an external data source  
  - Config: Uses Gmail OAuth2 credentials, configured with search query and limit parameters from AI input  
  - Connected as AI tool to AI Agent  
  - Failure modes: Gmail API errors, malformed queries  

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                                  | Input Node(s)           | Output Node(s)                | Sticky Note                                                                                                         |
|----------------------------|---------------------------------|-------------------------------------------------|------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger            | Telegram Trigger                | Captures incoming Telegram messages             | -                      | Switch                       | General overview of workflow purpose and phases                                                                    |
| Switch                     | Switch                         | Routes based on message content type             | Telegram Trigger       | Get photo file, Get audio file, Get a file, Get text from Message | General overview of workflow purpose and phases                                                                    |
| Get photo file              | Telegram (file download)        | Downloads photo file from Telegram                | Switch (photo)          | ConvertAPI HTTP Request       | Photo processing section                                                                                             |
| ConvertAPI HTTP Request     | HTTP Request                   | Converts image to PNG format                       | Get photo file          | Analyze image                | Photo processing section                                                                                             |
| Analyze image              | Langchain OpenAI (image)        | Analyzes image content with GPT-4o                | ConvertAPI HTTP Request | Get text from Image           | Photo processing section                                                                                             |
| Get text from Image         | Set                           | Formats image description text                     | Analyze image           | AI Agent                     | Photo processing section                                                                                             |
| Get audio file              | Telegram (file download)        | Downloads voice message audio                       | Switch (voice)          | Transcribe a recording        | Audio processing section                                                                                            |
| Transcribe a recording     | Langchain OpenAI (audio)         | Transcribes audio to text                           | Get audio file          | Get text from Audio           | Audio processing section                                                                                            |
| Get text from Audio         | Set                           | Formats transcribed audio text                      | Transcribe a recording  | AI Agent                     | Audio processing section                                                                                            |
| Get a file                 | Telegram (file download)          | Downloads document file                             | Switch (document)       | If                          | Document processing section                                                                                         |
| If                         | If                            | Checks if file is PDF                              | Get a file              | Extract from File, Unsupported Input | Document processing section                                                                                         |
| Extract from File           | Extract from File              | Extracts text from PDF                             | If (true)               | Get text from PDF             | Document processing section                                                                                         |
| Get text from PDF           | Set                           | Formats extracted PDF text                         | Extract from File       | AI Agent                     | Document processing section                                                                                         |
| Unsupported Input           | Set                           | Provides unsupported file type message            | If (false)              | AI Agent                     | Document processing section                                                                                         |
| Get text from Message       | Set                           | Formats plain text messages                        | Switch (text)           | AI Agent                     |                                                                                                                     |
| AI Agent                   | Langchain Agent               | Central AI processing, memory retrieval, tool use | Get text from PDF, Get text from Image, Get text from Audio, Get text from Message, Unsupported Input | Respond                      | AI Agent processing, MongoDB vector store, Gmail tools                                                             |
| MongoDB Chat Memory         | Langchain MongoDB Memory      | Stores and retrieves chat session history          | -                      | AI Agent                     | AI Agent processing, MongoDB vector store, Gmail tools                                                             |
| OpenAI Chat Model           | Langchain OpenAI Chat Model   | GPT-4o model for response generation               | -                      | AI Agent                     | AI Agent processing, MongoDB vector store, Gmail tools                                                             |
| Respond                    | Telegram                       | Sends AI response back to Telegram user            | AI Agent                | -                            | AI Agent processing, MongoDB vector store, Gmail tools                                                             |
| Send a message in Gmail     | Gmail Tool                    | Enables AI Agent to send emails                     | AI Agent (tool input)   | AI Agent                     | AI Agent processing, MongoDB vector store, Gmail tools                                                             |
| Search for a messages in Gmail | Gmail Tool                    | Enables AI Agent to search emails                   | AI Agent (tool input)   | AI Agent                     | AI Agent processing, MongoDB vector store, Gmail tools                                                             |
| Sticky Note                | Sticky Note                   | Documentation note                                 | -                      | -                            | Provides high-level overview of workflow phases                                                                    |
| Sticky Note1               | Sticky Note                   | Documentation note                                 | -                      | -                            | Details photo processing section                                                                                    |
| Sticky Note2               | Sticky Note                   | Documentation note                                 | -                      | -                            | Details document processing and unsupported file handling                                                         |
| Sticky Note3               | Sticky Note                   | Documentation note                                 | -                      | -                            | Details audio processing section                                                                                    |
| Sticky Note4               | Sticky Note                   | Documentation note                                 | -                      | -                            | Details AI agent processing, memory, and Gmail integrations                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Listen to "message" updates  
   - Credentials: Connect Telegram API credentials  
   - Position: Start of workflow  

2. **Add Switch Node**  
   - Type: Switch  
   - Parameters:  
     - Rule 1: Output "photo" if `$json.message.photo[3]` exists  
     - Rule 2: Output "voice" if `$json.message.voice` exists  
     - Rule 3: Output "document" if `$json.message.document` exists  
     - Rule 4: Output "text" if `$json.message.text` exists  
   - Connect Telegram Trigger → Switch  

3. **Photo Processing Branch**  
   a. Add Telegram node "Get photo file"  
      - Parameters: `fileId` = `$json.message.photo[3].file_id`  
      - Credentials: Telegram API  
   b. Add HTTP Request node "ConvertAPI HTTP Request"  
      - Method: POST  
      - URL: `https://v2.convertapi.com/convert/ai/to/png`  
      - Body: multipart-form-data including file from previous node  
      - Headers: Authorization with ConvertAPI Bearer token  
   c. Add Langchain OpenAI node "Analyze image"  
      - Resource: image  
      - Operation: analyze  
      - Model: GPT-4o  
      - Image URL: `$json.Files[0].Url` from ConvertAPI response  
      - Credentials: OpenAI API  
   d. Add Set node "Get text from Image"  
      - Assign field `text` with template: `"I just submitted an images with this description: {{ $json.content }}"`  
   e. Connect nodes: Switch (photo) → Get photo file → ConvertAPI HTTP Request → Analyze image → Get text from Image → AI Agent  

4. **Voice Processing Branch**  
   a. Add Telegram node "Get audio file"  
      - Parameters: `fileId` = `$json.message.voice.file_id`  
      - Credentials: Telegram API  
   b. Add Langchain OpenAI node "Transcribe a recording"  
      - Resource: audio  
      - Operation: transcribe  
      - Credentials: OpenAI API  
   c. Add Set node "Get text from Audio"  
      - Assign field `text` = `$json.text` (transcription)  
   d. Connect nodes: Switch (voice) → Get audio file → Transcribe a recording → Get text from Audio → AI Agent  

5. **Document Processing Branch**  
   a. Add Telegram node "Get a file"  
      - Parameters: `fileId` = `$json.message.document.file_id`  
      - Credentials: Telegram API  
   b. Add If node "If"  
      - Condition: Check if `result.file_path` contains "pdf"  
   c. Add "Extract from File" node  
      - Operation: pdf extraction  
   d. Add Set node "Get text from PDF"  
      - Assign field `text` with template:  
        ```
        I just uploaded a file with details:  
        Name: {{ $json.info.Title }},  
        Content :{{ $json.text }}
        ```  
   e. Add Set node "Unsupported Input"  
      - Assign field `text` = `"User just uploaded the Word file but your system doesn't support such files just yet."`  
   f. Connect nodes: Switch (document) → Get a file → If  
      - If true: → Extract from File → Get text from PDF → AI Agent  
      - If false: → Unsupported Input → AI Agent  

6. **Text Message Branch**  
   a. Add Set node "Get text from Message"  
      - Assign field `text` = `$json.message.text`  
   b. Connect: Switch (text) → Get text from Message → AI Agent  

7. **AI Agent Setup**  
   a. Add Langchain MongoDB Chat Memory node  
      - Database: `chat_messages`  
      - Session key: `$json.message.from.id` from Telegram Trigger  
      - Context window: 100  
      - Credentials: MongoDB connection  
   b. Add Langchain OpenAI Chat Model node  
      - Model: `"gpt-4.1-mini"` (represents GPT-4o)  
      - Credentials: OpenAI API  
   c. Add Langchain Agent node "AI Agent"  
      - Input text from all "Get text from ..." Set nodes and Unsupported Input node  
      - Connect MongoDB Chat Memory as memory input  
      - Connect OpenAI Chat Model as language model input  
      - System message: instruct to query MongoDB Atlas Vector Store Tool for context as needed  
      - Credentials: OpenAI API  
   d. Add Gmail Tool nodes for "Send a message in Gmail" and "Search for a messages in Gmail"  
      - Configure Gmail OAuth2 credentials  
      - Connect both as AI tools input to AI Agent  
   e. Add Telegram node "Respond"  
      - Parameters: text = AI Agent output, chatId = original Telegram sender ID  
      - Credentials: Telegram API  
   f. Connect AI Agent → Respond  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This multi-modal AI assistant workflow combines voice, image, document, and text input processing with advanced AI and persistent memory for intelligent personal assistant capabilities.                                                                                                                                                                                                                     | General workflow purpose                                                                        |
| Image processing uses ConvertAPI to normalize image format before GPT-4o analysis for consistent results.                                                                                                                                                                                                                                                                                                     | Photo processing block                                                                          |
| Document processing supports PDFs with text extraction and gracefully handles unsupported file types with user feedback.                                                                                                                                                                                                                                                                                    | Document processing block                                                                      |
| Voice notes are transcribed with OpenAI Whisper (via Langchain) to make audio input searchable and processable by AI.                                                                                                                                                                                                                                                                                        | Audio processing block                                                                         |
| The AI Agent is configured for RAG via MongoDB Atlas Vector Search, chat memory for context, GPT-4o for generation, and integrates Gmail for email operations, enabling rich, context-aware interaction with users.                                                                                                                                                                                         | AI Agent processing                                                                             |
| For setting up GPT-4o and MongoDB Atlas Vector Store integration, follow OpenAI and MongoDB Atlas official documentation for API keys and index configuration.                                                                                                                                                                                                                                               | External resources (OpenAI, MongoDB Atlas docs)                                                |
| Telegram API credentials require a valid Bot Token and webhook setup to receive message updates.                                                                                                                                                                                                                                                                                                            | Telegram Trigger setup                                                                          |
| Gmail OAuth2 credentials must be configured with appropriate scopes to send and read emails via Gmail API.                                                                                                                                                                                                                                                                                                   | Gmail Tool nodes setup                                                                         |
| ConvertAPI requires an API key and may have request limits; ensure proper authentication and error handling in production.                                                                                                                                                                                                                                                                                  | ConvertAPI HTTP Request node                                                                   |
| See n8n official docs for configuring Langchain nodes, especially for embedding custom tools like MongoDB Vector Store and Gmail Tools for AI Agent.                                                                                                                                                                                                                                                     | n8n Langchain node documentation                                                              |
| Sticky Notes within the workflow provide detailed explanations for each major processing block to facilitate understanding and maintenance.                                                                                                                                                                                                                                                                   | Sticky Notes in workflow                                                                       |

---

**Disclaimer:**  
The provided content is extracted solely from an automated n8n workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.