HR & IT Helpdesk Chatbot with Audio Transcription

https://n8nworkflows.xyz/workflows/hr---it-helpdesk-chatbot-with-audio-transcription-2752


# HR & IT Helpdesk Chatbot with Audio Transcription

### 1. Workflow Overview

This workflow implements an intelligent HR & IT Helpdesk Chatbot that supports both text and audio messages from employees via Telegram. It transcribes audio messages into text, processes queries using AI embeddings and a vector database of internal company policies, and returns relevant, context-aware answers.

The workflow is logically divided into the following blocks:

- **1.1 Knowledge Base Preparation**: Downloading internal policy documents, extracting text, splitting into chunks, embedding, and storing them in a PostgreSQL vector store.
- **1.2 Message Reception and Type Verification**: Capturing incoming Telegram messages, distinguishing between text, audio, and unsupported types.
- **1.3 Audio Transcription**: For voice messages, downloading the audio file and transcribing it to text using OpenAI’s transcription service.
- **1.4 AI Query Processing and Retrieval-Augmented Generation (RAG)**: Using embeddings and vector search to retrieve relevant policy data, maintaining chat memory, and generating AI responses.
- **1.5 Response Delivery**: Sending the AI-generated response back to the employee on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Knowledge Base Preparation

**Overview:**  
This block downloads internal HR/IT policy PDFs, extracts their text content, splits the text into manageable chunks, generates embeddings for these chunks, and stores them in a PostgreSQL vector store. This vector store serves as the knowledge base for answering employee queries.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- HTTP Request  
- Extract from File  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings OpenAI  
- Create HR Policies (Postgres PGVector Vector Store)  
- Sticky Note (instructions)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates the knowledge base setup process on demand.  
  - Inputs: None  
  - Outputs: HTTP Request  
  - Failure: None expected; manual start.

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Downloads the employee handbook PDF from a public URL.  
  - Configuration: URL set to a sample employee handbook PDF.  
  - Inputs: Trigger from manual node.  
  - Outputs: Binary PDF data to Extract from File.  
  - Failure: Network errors, invalid URL, or access denied.

- **Extract from File**  
  - Type: Extract from File  
  - Role: Extracts text content from the downloaded PDF.  
  - Configuration: Operation set to PDF extraction.  
  - Inputs: Binary PDF from HTTP Request.  
  - Outputs: Extracted text JSON to Default Data Loader.  
  - Failure: Corrupted PDF, unsupported format.

- **Recursive Character Text Splitter**  
  - Type: Text Splitter  
  - Role: Splits large text into chunks of 2000 characters for embedding.  
  - Configuration: Chunk size 2000 characters.  
  - Inputs: Text data from Default Data Loader.  
  - Outputs: Text chunks for embedding.  
  - Failure: Empty or malformed text input.

- **Default Data Loader**  
  - Type: Document Data Loader  
  - Role: Loads extracted text into a format suitable for embedding.  
  - Configuration: Uses expression to get text from Extract from File node.  
  - Inputs: Extracted text JSON.  
  - Outputs: Document chunks to Embeddings OpenAI.  
  - Failure: Expression errors if text missing.

- **Embeddings OpenAI**  
  - Type: OpenAI Embeddings  
  - Role: Generates vector embeddings for text chunks.  
  - Configuration: Uses OpenAI API credentials.  
  - Inputs: Document chunks.  
  - Outputs: Embeddings to Create HR Policies.  
  - Failure: API rate limits, invalid credentials.

- **Create HR Policies**  
  - Type: PostgreSQL Vector Store (PGVector)  
  - Role: Inserts embeddings and metadata into PostgreSQL vector store.  
  - Configuration: Insert mode, uses PostgreSQL credentials.  
  - Inputs: Embeddings from OpenAI node.  
  - Outputs: None (data stored).  
  - Failure: DB connection errors, insert failures.

- **Sticky Note**  
  - Provides detailed instructions and links for this block’s setup and purpose.

---

#### 2.2 Message Reception and Type Verification

**Overview:**  
This block listens for incoming Telegram messages, determines if the message is text, audio (voice), or unsupported, and routes accordingly.

**Nodes Involved:**  
- Telegram Trigger  
- Verify Message Type (Switch)  
- Edit Fields (Set)  
- Telegram1 (Telegram node for file download)  
- Unsupported Message Type (Telegram message response)  
- Sticky Note2 (instructions)

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for new Telegram messages (text or voice).  
  - Configuration: Listens to "message" updates.  
  - Inputs: External Telegram messages.  
  - Outputs: To Verify Message Type.  
  - Failure: Webhook misconfiguration, connectivity issues.

- **Verify Message Type**  
  - Type: Switch  
  - Role: Checks if incoming message contains text or voice keys.  
  - Configuration:  
    - Output "Text" if message contains "text" key.  
    - Output "Audio" if message contains "voice" key.  
    - Fallback output for unsupported types.  
  - Inputs: Telegram Trigger output.  
  - Outputs: To Edit Fields (Text), Telegram1 (Audio), Unsupported Message Type (Fallback).  
  - Failure: Expression evaluation errors.

- **Edit Fields**  
  - Type: Set  
  - Role: Extracts text from the message and assigns it to a "text" field for uniform processing.  
  - Configuration: Sets `text = $json.message.text`.  
  - Inputs: Text messages from Verify Message Type.  
  - Outputs: To AI Agent.  
  - Failure: Missing text field in JSON.

- **Telegram1**  
  - Type: Telegram  
  - Role: Downloads the voice message file using file_id.  
  - Configuration: Uses Telegram API credentials, downloads file from `message.voice.file_id`.  
  - Inputs: Audio messages from Verify Message Type.  
  - Outputs: To OpenAI transcription node.  
  - Failure: File not found, API errors.

- **Unsupported Message Type**  
  - Type: Telegram  
  - Role: Sends a fallback message to user if message type is unsupported.  
  - Configuration: Sends text "I'm not able to process this message type." to the chat ID.  
  - Inputs: Fallback from Verify Message Type.  
  - Outputs: None.  
  - Failure: Telegram API errors.

- **Sticky Note2**  
  - Explains the message handling logic and fallback support.

---

#### 2.3 Audio Transcription

**Overview:**  
This block transcribes voice messages into text using OpenAI’s audio transcription API.

**Nodes Involved:**  
- OpenAI (Audio Transcription)  
- Sticky Note (general instructions)

**Node Details:**

- **OpenAI (Audio Transcription)**  
  - Type: OpenAI Audio Transcription  
  - Role: Transcribes audio binary data into text.  
  - Configuration:  
    - Resource: audio  
    - Operation: transcribe  
    - Binary property: `data` (from Telegram1 node)  
    - Uses OpenAI API credentials.  
  - Inputs: Binary audio file from Telegram1.  
  - Outputs: Transcribed text JSON to AI Agent.  
  - Failure: API limits, unsupported audio format, transcription errors.

---

#### 2.4 AI Query Processing and Retrieval-Augmented Generation (RAG)

**Overview:**  
This block processes the user’s query (text or transcribed audio), retrieves relevant information from the vector store, maintains chat memory, and generates a natural language response using OpenAI chat models.

**Nodes Involved:**  
- AI Agent  
- Answer questions with a vector store (Vector Store Tool)  
- Postgres PGVector Store  
- Embeddings OpenAI1  
- OpenAI Chat Model  
- OpenAI Chat Model1  
- Postgres Chat Memory  
- Sticky Note4 (instructions)

**Node Details:**

- **AI Agent**  
  - Type: LangChain AI Agent  
  - Role: Central AI processing node that orchestrates query answering using tools and memory.  
  - Configuration:  
    - Input text from either Edit Fields (text) or OpenAI transcription node (audio).  
    - System message: "You are a helpful assistant for HR and employee policies".  
    - Prompt type: define.  
    - Connected to chat memory, vector store tool, and language models.  
  - Inputs: Text query.  
  - Outputs: Final answer JSON to Telegram node.  
  - Failure: API errors, memory or vector store connection issues.

- **Answer questions with a vector store**  
  - Type: Vector Store Tool  
  - Role: Provides the AI agent access to the HR & IT policies vector store for retrieval.  
  - Configuration: Named "hr_employee_policies" with description.  
  - Inputs: Embeddings and vector store data.  
  - Outputs: To AI Agent.  
  - Failure: DB connectivity, empty vector store.

- **Postgres PGVector Store**  
  - Type: PostgreSQL Vector Store  
  - Role: Vector store used for retrieval during query answering.  
  - Configuration: Uses PostgreSQL credentials.  
  - Inputs: Embeddings from Embeddings OpenAI1.  
  - Outputs: To Vector Store Tool.  
  - Failure: DB errors.

- **Embeddings OpenAI1**  
  - Type: OpenAI Embeddings  
  - Role: Generates embeddings for user queries for vector search.  
  - Configuration: Uses OpenAI API credentials.  
  - Inputs: Query text from AI Agent.  
  - Outputs: To Postgres PGVector Store.  
  - Failure: API limits.

- **OpenAI Chat Model** and **OpenAI Chat Model1**  
  - Type: OpenAI Chat Completion Models  
  - Role: Generate natural language responses based on retrieved data and chat context.  
  - Configuration: Uses OpenAI API credentials.  
  - Inputs: AI Agent and Vector Store Tool outputs.  
  - Outputs: To AI Agent and Answer questions with a vector store.  
  - Failure: API errors.

- **Postgres Chat Memory**  
  - Type: PostgreSQL Chat Memory  
  - Role: Maintains conversation context per user chat ID for multi-turn dialogue.  
  - Configuration: Session key set to Telegram chat ID.  
  - Inputs: AI Agent.  
  - Outputs: To AI Agent.  
  - Failure: DB connection issues.

- **Sticky Note4**  
  - Explains the AI agent’s role, vector store integration, and chat memory for RAG.

---

#### 2.5 Response Delivery

**Overview:**  
This block sends the AI-generated response back to the employee on Telegram.

**Nodes Involved:**  
- Telegram  
- Sticky Note5 (instructions)

**Node Details:**

- **Telegram**  
  - Type: Telegram  
  - Role: Sends the final answer text back to the user’s chat.  
  - Configuration:  
    - Text set to AI Agent output field `output`.  
    - Chat ID from Telegram Trigger node message.  
    - Uses Telegram API credentials.  
  - Inputs: AI Agent output.  
  - Outputs: None.  
  - Failure: Telegram API errors, invalid chat ID.

- **Sticky Note5**  
  - Notes this is the simplest and most important part: sending the message.

---

### 3. Summary Table

| Node Name                    | Node Type                                | Functional Role                                   | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                       |
|------------------------------|-----------------------------------------|--------------------------------------------------|-------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                          | Starts knowledge base setup                       | None                          | HTTP Request                  | The setup needs to be run at the start or when data is changed                                  |
| HTTP Request                 | HTTP Request                            | Downloads internal policy PDF                      | When clicking ‘Test workflow’  | Extract from File             | See Sticky Note for detailed instructions                                                      |
| Extract from File            | Extract from File                       | Extracts text from PDF                             | HTTP Request                  | Default Data Loader           | See Sticky Note for detailed instructions                                                      |
| Default Data Loader          | Document Data Loader                    | Loads extracted text for embedding                | Extract from File             | Embeddings OpenAI             | See Sticky Note for detailed instructions                                                      |
| Recursive Character Text Splitter | Text Splitter                      | Splits text into chunks                            | Default Data Loader           | Embeddings OpenAI             | See Sticky Note for detailed instructions                                                      |
| Embeddings OpenAI            | OpenAI Embeddings                      | Creates vector embeddings for text chunks         | Default Data Loader           | Create HR Policies            | See Sticky Note for detailed instructions                                                      |
| Create HR Policies           | PostgreSQL Vector Store (PGVector)    | Stores embeddings in PostgreSQL vector store       | Embeddings OpenAI             | None                         | See Sticky Note for detailed instructions                                                      |
| Telegram Trigger             | Telegram Trigger                       | Listens for incoming Telegram messages             | External Telegram             | Verify Message Type           | See Sticky Note2 for message handling logic                                                    |
| Verify Message Type          | Switch                                | Routes messages by type (text, audio, unsupported) | Telegram Trigger             | Edit Fields, Telegram1, Unsupported Message Type | See Sticky Note2 for message handling logic                                                    |
| Edit Fields                 | Set                                   | Extracts text from message for uniform processing | Verify Message Type (Text)    | AI Agent                     | See Sticky Note2 for message handling logic                                                    |
| Telegram1                   | Telegram                              | Downloads voice message file                        | Verify Message Type (Audio)   | OpenAI (Audio Transcription) | See Sticky Note2 for message handling logic                                                    |
| Unsupported Message Type    | Telegram                              | Sends fallback message for unsupported types       | Verify Message Type (Fallback)| None                         | See Sticky Note2 for message handling logic                                                    |
| OpenAI (Audio Transcription) | OpenAI Audio Transcription             | Transcribes audio to text                           | Telegram1                    | AI Agent                     |                                                                                                 |
| AI Agent                    | LangChain AI Agent                    | Processes query, integrates vector store and memory | Edit Fields, OpenAI (Audio Transcription) | Telegram                     | See Sticky Note4 for AI agent explanation                                                      |
| Answer questions with a vector store | Vector Store Tool               | Provides vector store access for retrieval         | Postgres PGVector Store       | AI Agent                     | See Sticky Note4 for AI agent explanation                                                      |
| Postgres PGVector Store     | PostgreSQL Vector Store (PGVector)    | Vector store for retrieval                          | Embeddings OpenAI1            | Answer questions with a vector store | See Sticky Note4 for AI agent explanation                                                      |
| Embeddings OpenAI1          | OpenAI Embeddings                      | Embeds user queries for vector search              | AI Agent                     | Postgres PGVector Store      | See Sticky Note4 for AI agent explanation                                                      |
| OpenAI Chat Model           | OpenAI Chat Completion Model           | Generates AI responses                              | AI Agent                     | AI Agent                     | See Sticky Note4 for AI agent explanation                                                      |
| OpenAI Chat Model1          | OpenAI Chat Completion Model           | Generates AI responses                              | Answer questions with a vector store | Answer questions with a vector store | See Sticky Note4 for AI agent explanation                                                      |
| Postgres Chat Memory        | PostgreSQL Chat Memory                 | Maintains conversation context                      | AI Agent                     | AI Agent                     | See Sticky Note4 for AI agent explanation                                                      |
| Telegram                    | Telegram                              | Sends final response to user                        | AI Agent                     | None                         | See Sticky Note5 for response delivery                                                        |
| Sticky Note                 | Sticky Note                           | Instructions for knowledge base setup              | None                         | None                         | See content for detailed instructions                                                         |
| Sticky Note1                | Sticky Note                           | Instructions for vector store creation              | None                         | None                         | See content for detailed instructions                                                         |
| Sticky Note2                | Sticky Note                           | Instructions for message handling logic             | None                         | None                         | See content for detailed instructions                                                         |
| Sticky Note4                | Sticky Note                           | Instructions for AI agent and RAG explanation       | None                         | None                         | See content for detailed instructions                                                         |
| Sticky Note5                | Sticky Note                           | Instructions for response delivery                   | None                         | None                         | See content for detailed instructions                                                         |
| Sticky Note3                | Sticky Note                           | Reminder to run setup at start or data change       | None                         | None                         | See content for detailed instructions                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the knowledge base setup manually.

2. **Create HTTP Request Node**  
   - Connect from Manual Trigger.  
   - Configure URL to download your internal policy PDF (e.g., employee handbook).  
   - No authentication needed if public URL.

3. **Create Extract from File Node**  
   - Connect from HTTP Request.  
   - Set operation to PDF extraction.  
   - Input: Binary PDF from HTTP Request.

4. **Create Default Data Loader Node**  
   - Connect from Extract from File.  
   - Set JSON data to expression: `={{ $('Extract from File').item.json.text }}`.  
   - Purpose: Prepare text for splitting and embedding.

5. **Create Recursive Character Text Splitter Node**  
   - Connect from Default Data Loader.  
   - Set chunk size to 2000 characters.

6. **Create Embeddings OpenAI Node**  
   - Connect from Recursive Character Text Splitter.  
   - Configure with OpenAI API credentials.  
   - Purpose: Generate embeddings for text chunks.

7. **Create PostgreSQL Vector Store Node (Create HR Policies)**  
   - Connect from Embeddings OpenAI.  
   - Set mode to "insert".  
   - Configure PostgreSQL credentials with PGVector extension enabled.  
   - Purpose: Store embeddings and metadata.

8. **Create Telegram Trigger Node**  
   - Configure to listen for "message" updates.  
   - Connect to Verify Message Type node.

9. **Create Verify Message Type (Switch) Node**  
   - Connect from Telegram Trigger.  
   - Add rules:  
     - Output "Text" if message contains "text" key.  
     - Output "Audio" if message contains "voice" key.  
     - Fallback output for unsupported types.

10. **Create Edit Fields (Set) Node**  
    - Connect from Verify Message Type "Text" output.  
    - Set field `text` to `={{ $json.message.text }}`.  
    - Pass all other fields through.

11. **Create Telegram Node (Telegram1)**  
    - Connect from Verify Message Type "Audio" output.  
    - Configure to download file using `fileId = {{$json.message.voice.file_id}}`.  
    - Use Telegram API credentials.

12. **Create OpenAI Audio Transcription Node**  
    - Connect from Telegram1.  
    - Set resource to "audio", operation to "transcribe".  
    - Set binary property name to "data".  
    - Use OpenAI API credentials.

13. **Create Unsupported Message Type Telegram Node**  
    - Connect from Verify Message Type fallback output.  
    - Configure to send message: "I'm not able to process this message type."  
    - Use chat ID from `{{$json.message.chat.id}}`.  
    - Use Telegram API credentials.

14. **Create AI Agent Node**  
    - Connect from Edit Fields and OpenAI Audio Transcription nodes (merge outputs).  
    - Set system message: "You are a helpful assistant for HR and employee policies".  
    - Set prompt type to "define".  
    - Connect AI Agent to:  
      - Postgres Chat Memory node (for chat context).  
      - Answer questions with a vector store node (for retrieval).  
      - OpenAI Chat Model nodes (for response generation).

15. **Create Postgres Chat Memory Node**  
    - Connect to AI Agent as memory.  
    - Set session key to `={{ $('Telegram Trigger').item.json.message.chat.id }}`.  
    - Configure PostgreSQL credentials.

16. **Create Answer Questions with a Vector Store Node**  
    - Connect to AI Agent as tool.  
    - Name it "hr_employee_policies".  
    - Description: "data for HR and employee policies".  
    - Connect to Postgres PGVector Store node.

17. **Create Postgres PGVector Store Node**  
    - Connect from Embeddings OpenAI1 node (which embeds user queries).  
    - Configure PostgreSQL credentials.

18. **Create Embeddings OpenAI1 Node**  
    - Connect from AI Agent (user query embedding).  
    - Use OpenAI API credentials.

19. **Create OpenAI Chat Model and OpenAI Chat Model1 Nodes**  
    - Connect OpenAI Chat Model to AI Agent as language model.  
    - Connect OpenAI Chat Model1 to Answer questions with a vector store node.  
    - Use OpenAI API credentials.

20. **Create Telegram Node (Response Delivery)**  
    - Connect from AI Agent output.  
    - Configure to send message text from AI Agent output field `output`.  
    - Use chat ID from Telegram Trigger message.  
    - Use Telegram API credentials.

21. **Add Sticky Notes**  
    - Add sticky notes at relevant positions with instructions and links as per the original workflow for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Download & Extract Internal Policy Documents using HTTP Request and Extract from File nodes.                                      | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest                            |
| Use PostgreSQL with PGVector extension for production-ready vector storage.                                                      | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreinmemory/ |
| Workflow supports Telegram messages with text and voice, with fallback for unsupported types.                                   |                                                                                                          |
| AI Agent uses Retrieval-Augmented Generation (RAG) with vector store and chat memory for contextual HR & IT helpdesk support.   |                                                                                                          |
| OpenAI Whisper or Google Cloud Speech-to-Text can be used for audio transcription.                                               |                                                                                                          |
| Estimated setup time: 20–25 minutes.                                                                                            |                                                                                                          |
| Example employee handbook PDF used: https://s3.amazonaws.com/scschoolfiles/656/employee_handbook_print_1.pdf                    |                                                                                                          |

---

This documentation provides a detailed, structured reference to understand, reproduce, and maintain the HR & IT Helpdesk Chatbot workflow with audio transcription support in n8n.