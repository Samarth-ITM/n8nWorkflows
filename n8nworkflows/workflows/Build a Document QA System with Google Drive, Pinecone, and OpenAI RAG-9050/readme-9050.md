Build a Document QA System with Google Drive, Pinecone, and OpenAI RAG

https://n8nworkflows.xyz/workflows/build-a-document-qa-system-with-google-drive--pinecone--and-openai-rag-9050


# Build a Document QA System with Google Drive, Pinecone, and OpenAI RAG

---

### 1. Workflow Overview

This workflow implements a Document Question-Answering (QA) system using Google Drive, Pinecone vector database, and OpenAI’s Retrieval-Augmented Generation (RAG) capabilities. The workflow automates ingestion of new documents from a specified Google Drive folder, processes and indexes their content into Pinecone, and then answers user queries interactively by retrieving relevant document vectors and generating responses with OpenAI GPT models.

The workflow is structured into the following logical blocks:

- **1.1 Document Ingestion & Storage:**  
  Watches a Google Drive folder for new files, downloads them, processes their text content, generates embeddings, and stores these embeddings in the Pinecone vector store.

- **1.2 Embeddings Generation:**  
  Creates vector embeddings from document text chunks using OpenAI embeddings.

- **1.3 Vector Store Management:**  
  Inserts embeddings into Pinecone and retrieves relevant vectors during query time.

- **1.4 User Query and AI Processing:**  
  Listens for chat messages (user queries), uses Pinecone to retrieve relevant document vectors, and leverages OpenAI chat models with memory and agent functionality to generate informed answers.

- **1.5 Auxiliary and Configuration Notes:**  
  Sticky notes provide setup instructions, credential information, and workflow annotations.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Document Ingestion & Storage

**Overview:**  
This block triggers upon new file creation in a specific Google Drive folder, downloads the file, splits its content into manageable text chunks, generates embeddings, and stores those embeddings in Pinecone.

**Nodes Involved:**  
- Google Drive Trigger  
- Download file  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings OpenAI  
- Pinecone Vector Store  

**Node Details:**

- **Google Drive Trigger**  
  - *Type:* Trigger node for Google Drive events  
  - *Configuration:* Watches a specific folder for new files created; polling every minute  
  - *Key Variables:* Folder ID specified by user (parameter: `folderToWatch`)  
  - *Connections:* Output to “Download file” node  
  - *Failure Modes:* Auth errors if Google OAuth2 credentials are missing/expired; missing folder ID; API quota limits  

- **Download file**  
  - *Type:* Google Drive file downloader  
  - *Configuration:* Downloads the newly created file using the file ID from the trigger  
  - *Key Variables:* File ID taken from the trigger event  
  - *Connections:* Output to “Pinecone Vector Store” for insertion  
  - *Failure Modes:* File not found; permissions errors; download timeouts  

- **Recursive Character Text Splitter**  
  - *Type:* Text splitter (recursive character-based)  
  - *Configuration:* Splits document text into chunks with 100-character overlap to preserve context  
  - *Connections:* Output to “Default Data Loader” node  
  - *Failure Modes:* Input data missing or malformed text; chunk size misconfiguration  

- **Default Data Loader**  
  - *Type:* Document loader node (handles various document types)  
  - *Configuration:* Uses custom text splitting mode; loads processed text chunks  
  - *Connections:* Output to “Pinecone Vector Store” as document input  
  - *Failure Modes:* Unsupported file format; parsing errors  

- **Embeddings OpenAI**  
  - *Type:* Embedding generator using OpenAI API  
  - *Configuration:* Default embedding model; no additional options set  
  - *Connections:* Outputs embeddings to both “Pinecone Vector Store” and “Pinecone Vector Store1”  
  - *Failure Modes:* API key missing or invalid; rate limits; network issues  

- **Pinecone Vector Store**  
  - *Type:* Vector database insertion node for Pinecone  
  - *Configuration:* Inserts document embeddings into a specified Pinecone index (user-defined)  
  - *Key Variables:* Pinecone index ID configured by user  
  - *Connections:* Receives document, embeddings, and text chunk inputs  
  - *Failure Modes:* API key invalid; index not found; network errors  

---

#### 2.2 Embeddings Generation

**Overview:**  
This block handles generating vector embeddings from document text chunks using OpenAI embeddings, feeding these vectors into Pinecone for storage and later retrieval.

**Nodes Involved:**  
- Embeddings OpenAI  
- Pinecone Vector Store  
- Pinecone Vector Store1  

**Node Details:**

- **Embeddings OpenAI**  
  - (Discussed above in ingestion)  

- **Pinecone Vector Store**  
  - Inserts embeddings for document storage (insertion mode)  

- **Pinecone Vector Store1**  
  - *Type:* Pinecone vector store used for retrieval as a tool within the AI agent  
  - *Configuration:* Set to mode “retrieve-as-tool” with a tool description “Use this as a knowledge base to answer the user message”  
  - *Key Variables:* Index chosen from a list, cached under “databases”  
  - *Connections:* Serves as an AI tool input to “AI Agent” node  
  - *Failure Modes:* Retrieval errors, stale cache, or index misconfiguration  

---

#### 2.3 Vector Store Management

**Overview:**  
Manages insertion and retrieval of vector embeddings to/from Pinecone, acting as the knowledge base for document search during queries.

**Nodes Involved:**  
- Pinecone Vector Store (insertion)  
- Pinecone Vector Store1 (retrieval)  

**Node Details:**  
- Covered above in sections 2.1 and 2.2  

---

#### 2.4 User Query and AI Processing

**Overview:**  
Handles incoming chat messages, uses Pinecone to retrieve relevant document vectors, and runs the AI agent with OpenAI chat model and simple memory to generate context-aware responses.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- OpenAI Chat Model  
- Simple Memory  
- Pinecone Vector Store1  

**Node Details:**

- **When chat message received**  
  - *Type:* Chat trigger node (webhook-based)  
  - *Configuration:* Listens for chat messages from external clients via webhook ID  
  - *Connections:* Triggers the “AI Agent” node  
  - *Failure Modes:* Webhook misconfiguration; timeouts  

- **AI Agent**  
  - *Type:* LangChain Agent node integrating multiple AI functions  
  - *Configuration:* Uses system message “You are a helpful assistant”; output parser enabled  
  - *Connections:*  
    - Inputs: Chat messages, memory, language model, and Pinecone tool  
    - Outputs: Final AI response  
  - *Failure Modes:* Parsing errors; tool retrieval failures; model API errors  

- **OpenAI Chat Model**  
  - *Type:* Language model node using OpenAI chat completions  
  - *Configuration:* Model set to “gpt-4.1-mini” (likely GPT-4 variant)  
  - *Connections:* Feeds into “AI Agent” as language model input  
  - *Failure Modes:* API key or quota issues; model unavailability  

- **Simple Memory**  
  - *Type:* Buffer window memory node  
  - *Configuration:* Default settings, stores recent interactions to provide context  
  - *Connections:* Feeds into “AI Agent” as memory input  
  - *Failure Modes:* Memory overflow or corruption unlikely but possible  

- **Pinecone Vector Store1**  
  - Used here as a retrieval tool for relevant document vectors during query processing (discussed above)  

---

#### 2.5 Auxiliary and Configuration Notes

**Overview:**  
Sticky Notes provide guidance, context, and important setup instructions directly inside the workflow canvas.

**Nodes Involved:**  
- Sticky Note (multiple instances)  

**Key Notes:**

- **Credential Setup (Sticky Note5):**  
  Detailed instructions on creating credentials for Google Drive OAuth2, Pinecone API, and OpenAI API with links to official n8n docs.

- **Workflow Builder Credit (Sticky Note7):**  
  Attribution to workflow builder Abdullah Ahmed with YouTube link.

- **RAG System Explanation (Sticky Note6):**  
  High-level explanation of the Retrieval-Augmented Generation concept and advice on documentation style.

- **Functional Block Headers (Sticky Notes 0,1,2,3,4):**  
  Visual grouping and labels for document inserting, embeddings, storing, testing, and database retrieval.

---

### 3. Summary Table

| Node Name                 | Node Type                                  | Functional Role                              | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                      |
|---------------------------|--------------------------------------------|----------------------------------------------|------------------------|---------------------------|------------------------------------------------------------------------------------------------|
| Google Drive Trigger       | n8n-nodes-base.googleDriveTrigger           | Watches Google Drive folder for new files    | -                      | Download file              | Credential Setup (also applies to Download file)                                               |
| Download file             | n8n-nodes-base.googleDrive                   | Downloads new file from Google Drive          | Google Drive Trigger    | Pinecone Vector Store      | Credential Setup                                                                                |
| Recursive Character Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Splits document text into chunks               | - (implicit from flow)  | Default Data Loader        | Embeddings block sticky note covers this area                                                 |
| Default Data Loader       | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Loads and prepares document text chunks       | Recursive Character Text Splitter | Pinecone Vector Store | Document inserting, Storing sticky notes                                                       |
| Embeddings OpenAI         | @n8n/n8n-nodes-langchain.embeddingsOpenAi   | Generates OpenAI text embeddings               | -                      | Pinecone Vector Store, Pinecone Vector Store1 | Embeddings sticky note                                                                        |
| Pinecone Vector Store     | @n8n/n8n-nodes-langchain.vectorStorePinecone | Inserts embeddings into Pinecone index         | Download file, Default Data Loader, Embeddings OpenAI | -           | Storing sticky note                                                                             |
| When chat message received| @n8n/n8n-nodes-langchain.chatTrigger        | Webhook trigger for user chat messages         | -                      | AI Agent                  | Test here sticky note                                                                           |
| AI Agent                 | @n8n/n8n-nodes-langchain.agent               | Processes user query using AI & vector store  | When chat message received, Simple Memory, OpenAI Chat Model, Pinecone Vector Store1 | - | Test here sticky note                                                                           |
| OpenAI Chat Model         | @n8n/n8n-nodes-langchain.lmChatOpenAi       | Language model for AI Agent                     | -                      | AI Agent                  | Credential Setup                                                                                |
| Simple Memory             | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores recent conversation context             | -                      | AI Agent                  |                                                                                                |
| Pinecone Vector Store1    | @n8n/n8n-nodes-langchain.vectorStorePinecone | Retrieves vectors as a tool for AI Agent       | Embeddings OpenAI       | AI Agent                  | Get the Database sticky note                                                                    |
| Sticky Note (multiple)    | n8n-nodes-base.stickyNote                     | Documentation and guidance within workflow     | -                      | -                         | Various, as detailed in section 2.5                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node:**  
   - Type: Google Drive Trigger  
   - Event: `fileCreated`  
   - Polling interval: every minute  
   - Folder to watch: specify the Google Drive folder ID to monitor  
   - Configure Google Drive OAuth2 credentials  

2. **Create Download file node:**  
   - Type: Google Drive  
   - Operation: `download`  
   - File ID: taken dynamically from the Google Drive Trigger node output  
   - Connect Google Drive Trigger → Download file  
   - Use same Google Drive OAuth2 credentials  

3. **Create Recursive Character Text Splitter node:**  
   - Type: LangChain Recursive Character Text Splitter  
   - Chunk Overlap: 100 characters  
   - Connect Download file → Recursive Character Text Splitter (passing downloaded file content)  

4. **Create Default Data Loader node:**  
   - Type: LangChain Document Default Data Loader  
   - Text Splitting Mode: custom  
   - Connect Recursive Character Text Splitter → Default Data Loader  

5. **Create Embeddings OpenAI node:**  
   - Type: LangChain OpenAI Embeddings  
   - Use default embedding model (no special parameters)  
   - Configure OpenAI API credentials  
   - Connect Default Data Loader → Embeddings OpenAI  

6. **Create Pinecone Vector Store node (Insertion):**  
   - Type: LangChain Vector Store Pinecone  
   - Mode: insert  
   - Pinecone Index: specify your Pinecone index ID  
   - Configure Pinecone API credentials  
   - Connect Download file, Default Data Loader, Embeddings OpenAI → Pinecone Vector Store (ensure proper input mapping: document, text chunks, embeddings)  

7. **Create Pinecone Vector Store1 node (Retrieval Tool):**  
   - Type: LangChain Vector Store Pinecone  
   - Mode: retrieve-as-tool  
   - Tool description: “Use this as a knowledge base to answer the user message”  
   - Pinecone Index: select from your list of indexes (cache as “databases”)  
   - Configure Pinecone API credentials  
   - Connect Embeddings OpenAI → Pinecone Vector Store1  

8. **Create “When chat message received” node:**  
   - Type: LangChain Chat Trigger  
   - Generates webhook for receiving chat messages  
   - Connect to AI Agent node  

9. **Create OpenAI Chat Model node:**  
   - Type: LangChain LM Chat OpenAI  
   - Model: “gpt-4.1-mini” or equivalent GPT-4 variant  
   - Configure OpenAI API credentials  
   - Connect to AI Agent node  

10. **Create Simple Memory node:**  
    - Type: LangChain Memory Buffer Window  
    - Default buffer size  
    - Connect to AI Agent node  

11. **Create AI Agent node:**  
    - Type: LangChain Agent  
    - System message: “You are a helpful assistant”  
    - Enable output parser  
    - Connect inputs from:  
      - When chat message received (main)  
      - Simple Memory (ai_memory)  
      - OpenAI Chat Model (ai_languageModel)  
      - Pinecone Vector Store1 (ai_tool)  

12. **Add Sticky Notes:**  
    - Add sticky notes for documentation on credential setup, block explanations, and credits as per the original workflow  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                                                             |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Credential Setup (Critical) - Setup Google Drive OAuth2, Pinecone API, OpenAI API credentials. Follow official n8n docs for each: Google OAuth2: https://docs.n8n.io/integrations/builtin/credentials/google/oauth-generic/ Pinecone: https://docs.n8n.io/integrations/builtin/credentials/pinecone/ OpenAI: https://docs.n8n.io/integrations/builtin/credentials/openai/ | Provides essential setup for all external API integrations in this workflow                                                                  |
| Workflow builder credit: Abdullah Ahmed - https://www.youtube.com/@gureyosman06                                                                                                                                                                                                                                                                                   | Original workflow builder attribution                                                                                                       |
| RAG system explanation: Summarizes the workflow’s function into easy steps and advises to keep main documentation concise, placing detailed instructions in sticky notes inside the workflow                                                                                                                                                                    | Documentation style advice within the workflow                                                                                              |
| Test here: Note indicating that the chat trigger can be modified as needed for different input sources                                                                                                                                                                                                                                                          | Flexibility for adapting the user input trigger                                                                                             |
| Get the Database: Indicates Pinecone Vector Store1 is used as a retrieval tool for answering user queries                                                                                                                                                                                                                                                        | Clarifies the retrieval role of this node                                                                                                  |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---