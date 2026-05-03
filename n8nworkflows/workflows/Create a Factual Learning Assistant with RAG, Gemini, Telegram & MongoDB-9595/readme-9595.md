Create a Factual Learning Assistant with RAG, Gemini, Telegram & MongoDB

https://n8nworkflows.xyz/workflows/create-a-factual-learning-assistant-with-rag--gemini--telegram---mongodb-9595


# Create a Factual Learning Assistant with RAG, Gemini, Telegram & MongoDB

---

## 1. Workflow Overview

This workflow implements a **Retrieval-Augmented Generation (RAG) AI Learning Assistant** designed for educational and exam preparation use cases, specifically targeting users like UPSC aspirants. It integrates **Google Gemini AI models**, **MongoDB Atlas Vector Search**, **Telegram messaging**, and **Google Drive** document ingestion to create a factual, source-grounded conversational assistant.

The workflow is logically divided into two main blocks:

### 1.1 Knowledge Ingestion Pipeline  
This block handles the ingestion and indexing of new educational documents. Documents uploaded either via a form upload or Google Drive trigger are downloaded, converted into vector embeddings using Google Gemini embeddings, and stored in a MongoDB Atlas vector store. This builds and maintains the knowledge base required for accurate AI responses.

### 1.2 RAG Chatbot Query Pipeline  
This block listens for user queries sent via Telegram. Upon receiving a question, it uses the RAG method by first retrieving relevant documents from the MongoDB vector store, then passing the retrieved context along with the question to Google Gemini’s chat model. The AI agent generates a precise, well-structured, and factual answer based on the retrieved knowledge and sends it back through Telegram.

---

## 2. Block-by-Block Analysis

### 2.1 Knowledge Ingestion Pipeline

#### Overview  
This block automates the process of detecting new educational content, processing it into embeddings, and inserting those embeddings into a vector search-enabled MongoDB collection. It supports two ingestion triggers: file upload via a form and new file creation in a designated Google Drive folder.

#### Nodes Involved  
- On File Upload  
- File Uploaded (Google Drive Trigger)  
- Download File (Google Drive)  
- Convert Documents to Embeddings (Google Gemini Embeddings)  
- MongoDB Atlas Vector Store - Insert

#### Node Details

- **On File Upload**  
  - Type: Form Trigger  
  - Role: Starts ingestion when a user uploads a file via a web form.  
  - Configuration: Accepts files with extensions .pdf, .csv, .jpg, .jpeg, .png.  
  - Inputs: External user upload through a web form.  
  - Outputs: File data forwarded to MongoDB insert node.  
  - Edge Cases: Upload size limits, unsupported file types, upload failures.

- **File Uploaded (Google Drive Trigger)**  
  - Type: Google Drive Trigger  
  - Role: Watches a specific Google Drive folder for new files created.  
  - Configuration: Triggers every minute, watching a specific folder whose ID is parameterized (`<GOOGLE_DRIVE_FOLDER_ID_WATCH>`).  
  - Inputs: Google Drive folder monitoring.  
  - Outputs: Passes new file metadata to “Download File”.  
  - Edge Cases: Drive API rate limits, access permission errors, missing folder.

- **Download File**  
  - Type: Google Drive Node  
  - Role: Downloads newly uploaded files from Google Drive for processing.  
  - Configuration: Uses file ID from trigger node, outputs binary content.  
  - Inputs: File metadata from Google Drive trigger.  
  - Outputs: Binary file data forwarded to embeddings node.  
  - Edge Cases: File not found, download failure, file access permissions.

- **Convert Documents to Embeddings**  
  - Type: Google Gemini Embeddings Node  
  - Role: Converts document text into vector embeddings for semantic search.  
  - Configuration: Uses Google Gemini API credentials.  
  - Inputs: Binary document data.  
  - Outputs: Embeddings passed to MongoDB insert.  
  - Edge Cases: API authentication errors, document parsing failures, large file size limits.

- **MongoDB Atlas Vector Store - Insert**  
  - Type: MongoDB Atlas Vector Store Node  
  - Role: Inserts document embeddings into a MongoDB collection with vector index enabled.  
  - Configuration: Collection and vector index names parameterized (`<MONGO_ATLAS_COLLECTION_NAME>`, `<MONGO_ATLAS_VECTOR_INDEX_NAME_INSERT>`).  
  - Inputs: Embeddings from previous node or directly from form upload.  
  - Outputs: Confirmation of insertion.  
  - Edge Cases: MongoDB connection or authentication errors, insertion failures, index misconfiguration.

---

### 2.2 RAG Chatbot Query Pipeline

#### Overview  
This block listens for incoming questions on Telegram, retrieves relevant documents from MongoDB using vector similarity search, and uses Google Gemini chat to generate a factual, exam-level response. The answer is then sent back to the user on Telegram.

#### Nodes Involved  
- Listen for Aspirant Question (Telegram Trigger)  
- When Chat Message Received (Langchain Chat Trigger)  
- MongoDB Atlas Vector Store - Retrieve  
- Simple Memory (Memory Buffer)  
- Google Gemini Chat Model  
- RAG Agent (Langchain Agent)  
- Send Answer via Telegram

#### Node Details

- **Listen for Aspirant Question**  
  - Type: Telegram Trigger  
  - Role: Listens for incoming messages in a specified Telegram chat or channel (`@educationalch`).  
  - Configuration: Triggers on new messages only, filtered by chat ID.  
  - Inputs: User messages from Telegram.  
  - Outputs: Passes question text to RAG Agent.  
  - Edge Cases: Telegram API limits, bot permissions, chat ID misconfiguration.

- **When Chat Message Received**  
  - Type: Langchain Chat Trigger  
  - Role: Alternative or complementary trigger for chat messages, possibly for other chat sources or internal testing.  
  - Inputs: Chat message events.  
  - Outputs: Forwards user messages to RAG Agent.  
  - Edge Cases: Message format errors, trigger configuration.

- **MongoDB Atlas Vector Store - Retrieve**  
  - Type: MongoDB Atlas Vector Store Node  
  - Role: Performs semantic retrieval of relevant documents based on the user query embedding.  
  - Configuration: Uses vector index `<MONGO_ATLAS_VECTOR_INDEX_NAME_RETRIEVE>` and collection `<MONGO_ATLAS_COLLECTION_NAME>`.  
  - Inputs: Query text embedded by Google Gemini embeddings (internally handled by RAG Agent).  
  - Outputs: Retrieved relevant document snippets to RAG Agent.  
  - Edge Cases: Query failures, MongoDB timeouts, empty result sets.

- **Simple Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains short-term conversational context to enable multi-turn dialogue coherence.  
  - Configuration: Default buffer window size.  
  - Inputs: Conversation history from previous chat turns.  
  - Outputs: Contextual memory to RAG Agent.  
  - Edge Cases: Memory overload, context truncation.

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini Chat Model Node  
  - Role: Generates natural language answers based on the combined user query and retrieved context.  
  - Configuration: Model name `models/gemini-2.5-flash-lite`, uses Google Gemini API credentials.  
  - Inputs: Contextual prompt from RAG Agent.  
  - Outputs: Generated answer text.  
  - Edge Cases: API quota exceeded, response latency, generation errors.

- **RAG Agent**  
  - Type: Langchain Agent  
  - Role: Core orchestrator combining user input, document retrieval, memory, and language model to produce a grounded answer.  
  - Configuration:  
    - System message defines an expert UPSC exam analyst persona.  
    - Enforces strict RAG usage: answers must be based on retrieved documents only, no hallucination.  
    - Formatting rules for clarity and academic tone.  
  - Inputs: User question (text), memory, retrieval tool, and language model.  
  - Outputs: Final answer text.  
  - Edge Cases: Failure to retrieve relevant documents, prompt construction errors, incomplete answers.

- **Send Answer via Telegram**  
  - Type: Telegram Node  
  - Role: Sends the generated answer back to the user in Telegram chat.  
  - Configuration: Uses Telegram bot token credentials, sends message to `<TELEGRAM_CHAT_ID>`.  
  - Inputs: Answer text from RAG Agent.  
  - Outputs: Telegram message delivery.  
  - Edge Cases: Telegram API errors, chat ID errors, message formatting issues.

---

## 3. Summary Table

| Node Name                            | Node Type                                   | Functional Role                          | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
|------------------------------------|---------------------------------------------|----------------------------------------|--------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Default Data Loader                 | Langchain Document Default Data Loader      | (Unused in main flow) Loads default docs | -                              | MongoDB Atlas Vector Store - Insert |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| When chat message received          | Langchain Chat Trigger                       | Triggers on chat messages for RAG agent | -                              | RAG Agent                     |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| File uploaded                      | Google Drive Trigger                         | Watches specific Google Drive folder for new files | -                              | Download file                 |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| MongoDB Atlas Vector Store - Insert | Langchain MongoDB Atlas Vector Store Insert | Inserts document embeddings into MongoDB | Download file, On File Upload, Default Data Loader | -                             |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| MongoDB Atlas Vector Store - Retrieve | Langchain MongoDB Atlas Vector Store Retrieve | Retrieves relevant documents for AI queries | Retrieve Documents from Embeddings | RAG Agent                     |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Google Gemini Chat Model            | Langchain Google Gemini Chat Model           | Generates AI answer text based on context | RAG Agent                     | RAG Agent                     |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Download file                      | Google Drive Node                           | Downloads files from Google Drive for processing | File uploaded                 | MongoDB Atlas Vector Store - Insert |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| RAG Agent                         | Langchain Agent                             | Combines retrieval, memory, and AI model to answer questions | When chat message received, Listen for Aspirant Question, Simple Memory, Google Gemini Chat Model, MongoDB Atlas Vector Store - Retrieve | Send Answer via Telegram      |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| On File Upload                    | Form Trigger                               | Triggers ingestion on file upload via form | -                              | MongoDB Atlas Vector Store - Insert |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Listen for Aspirant Question        | Telegram Trigger                            | Listens for Telegram messages from users | -                              | RAG Agent                     |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Send Answer via Telegram            | Telegram Node                              | Sends AI-generated answer back to user | RAG Agent                     | -                             |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Convert Documents to Embeddings     | Langchain Google Gemini Embeddings          | Converts documents to vector embeddings | Download file                 | MongoDB Atlas Vector Store - Insert |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Retrieve Documents from Embeddings  | Langchain Google Gemini Embeddings          | (Appears unused in main flow) Converts queries or context for retrieval | -                              | MongoDB Atlas Vector Store - Retrieve |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Simple Memory                     | Langchain Memory Buffer Window               | Maintains conversational context between turns | -                              | RAG Agent                     |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Sticky Note                      | Sticky Note                                 | Explains user audience, workflow purpose, setup instructions | -                              | -                             | Who's it for? This template is perfect for educational institutions, coaching centers (UPSC, GMAT, etc.), corporate knowledge bases, and SaaS companies needing accurate, source-grounded answers. It uses Google Gemini for powerful reasoning combined with a strict RAG approach for factual accuracy. Setup instructions and requirements are detailed in the note.                                                                                                                                                                                     |
| Sticky Note1                     | Sticky Note                                 | Labels Knowledge Ingestion Pipeline block | -                              | -                             | Workflow 1: Knowledge Ingestion Pipeline (Triggers on file upload to form or Google Drive)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Sticky Note2                     | Sticky Note                                 | Labels RAG Chatbot Query Pipeline block | -                              | -                             | Workflow 2: RAG Chatbot Query Pipeline (Triggers on question received via Telegram)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it appropriately (e.g., "RAG-based AI Learning Assistant for Telegram using MongoDB and Google Drive").**

### Knowledge Ingestion Pipeline Setup

2. **Add a Form Trigger node ("On File Upload"):**  
   - Configure the form title as "file upload".  
   - Add a file input field accepting `.pdf, .csv, .jpg, .jpeg, .png`.  
   - No credentials required.

3. **Add a Google Drive Trigger node ("File uploaded"):**  
   - Set event to "fileCreated".  
   - Configure to poll every minute.  
   - Set "Trigger on" to "specificFolder" and specify the folder ID to watch (`<GOOGLE_DRIVE_FOLDER_ID_WATCH>`).  
   - Provide Google Drive credentials.

4. **Add a Google Drive node ("Download file"):**  
   - Set operation to "download".  
   - Map the file ID input from the Google Drive Trigger node.  
   - Use Google Drive credentials.

5. **Add a Google Gemini Embeddings node ("Convert Documents to Embeddings"):**  
   - Leave default options if no customization needed.  
   - Provide Google Gemini API credentials.

6. **Add a MongoDB Atlas Vector Store node ("MongoDB Atlas Vector Store - Insert"):**  
   - Set mode to "insert".  
   - Specify the MongoDB Atlas collection name (`<MONGO_ATLAS_COLLECTION_NAME>`) and vector index name (`<MONGO_ATLAS_VECTOR_INDEX_NAME_INSERT>`).  
   - Provide MongoDB Atlas credentials.  
   - Ensure your MongoDB collection has a vector search index configured.

7. **Connect nodes in this order:**  
   - "On File Upload" → "MongoDB Atlas Vector Store - Insert" (directly for form uploads).  
   - "File uploaded" → "Download file" → "Convert Documents to Embeddings" → "MongoDB Atlas Vector Store - Insert".

### RAG Chatbot Query Pipeline Setup

8. **Add a Telegram Trigger node ("Listen for Aspirant Question"):**  
   - Configure to listen for "message" updates.  
   - Under additional fields, set "chatIds" to your Telegram chat/channel ID (e.g., `@educationalch`).  
   - Provide Telegram Bot credentials.

9. **Optionally, add Langchain Chat Trigger node ("When chat message received")** if supporting other chat sources.

10. **Add a MongoDB Atlas Vector Store node ("MongoDB Atlas Vector Store - Retrieve"):**  
    - Set mode to "retrieve-as-tool".  
    - Specify the same MongoDB collection name (`<MONGO_ATLAS_COLLECTION_NAME>`) and retrieval vector index (`<MONGO_ATLAS_VECTOR_INDEX_NAME_RETRIEVE>`).  
    - Provide MongoDB Atlas credentials.

11. **Add a Langchain Memory Buffer Window node ("Simple Memory"):**  
    - Use default buffer configuration.

12. **Add Google Gemini Chat Model node ("Google Gemini Chat Model"):**  
    - Set model name to `models/gemini-2.5-flash-lite`.  
    - Provide Google Gemini credentials.

13. **Add Langchain Agent node ("RAG Agent"):**  
    - Configure the system prompt with the expert UPSC analyst persona and strict RAG instructions as described.  
    - Set the prompt type to "define".  
    - Connect inputs:  
      - From Telegram Trigger and/or Chat Trigger nodes to main input.  
      - From MongoDB Retrieve node to ai_tool input.  
      - From Simple Memory node to ai_memory input.  
      - From Google Gemini Chat Model node to ai_languageModel input.

14. **Add a Telegram node ("Send Answer via Telegram"):**  
    - Set chat ID to your target Telegram chat (`<TELEGRAM_CHAT_ID>`).  
    - Map the text content from the RAG Agent output.  
    - Provide Telegram Bot credentials.

15. **Connect the final flow:**  
    - Telegram Trigger → RAG Agent → Send Answer via Telegram.

### Additional Setup

16. **Verify credentials:**  
    - Google Gemini API key (for embeddings and chat).  
    - MongoDB Atlas cluster connection with vector search index.  
    - Telegram Bot token and chat/channel IDs.  
    - Google Drive OAuth credentials.

17. **Test the ingestion pipeline:** Upload a document via the form or add a file to the configured Google Drive folder. Ensure it is processed and indexed.

18. **Test the query pipeline:** Send a question via Telegram to the bot and verify the response is factual and based on ingested documents.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This template suits educational institutions, coaching centers (UPSC, GMAT), corporate knowledge bases, and SaaS providers needing instant, accurate, source-grounded answers. It leverages Google Gemini's reasoning power but constrains answers strictly to verified knowledge bases to avoid hallucinations.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note overview in workflow.                                                                |
| Requirements include n8n account, Google Gemini API key, MongoDB Atlas cluster with vector search index, Telegram bot with chat ID, and optionally Google Drive credentials for ingestion.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Setup instructions in sticky note.                                                              |
| The workflow uses a two-phase approach: Knowledge ingestion (via form upload or Google Drive) and AI Query & Response (via Telegram). The RAG Agent is configured with a detailed system prompt to enforce expert-level, factual, and analytical answers suitable for UPSC exam preparation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note explanation.                                                                         |
| MongoDB Atlas Vector Search requires explicit vector index creation on the collection used. Free-tier clusters are sufficient.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | MongoDB Atlas documentation: https://www.mongodb.com/docs/atlas/atlas-search/vector-search/       |
| Google Gemini API credentials are required for both embeddings and chat model nodes, ensure correct scopes and quotas are enabled.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Google Cloud AI Platform documentation.                                                        |
| Telegram Bot setup requires creating a bot via BotFather and obtaining the chat/channel ID where the bot will operate.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Telegram Bot API documentation.                                                                |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. It fully complies with all current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---