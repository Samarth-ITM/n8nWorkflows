Build & Query RAG System with Google Drive, OpenAI GPT-4o-mini, and Pinecone

https://n8nworkflows.xyz/workflows/build---query-rag-system-with-google-drive--openai-gpt-4o-mini--and-pinecone-4501


# Build & Query RAG System with Google Drive, OpenAI GPT-4o-mini, and Pinecone

### 1. Workflow Overview

This workflow, titled **Instant RaG Builder**, automates the process of building and querying a Retrieval-Augmented Generation (RAG) system by integrating Google Drive, OpenAI GPT-4o-mini, and Pinecone vector database. It is designed to:

- Automatically watch a specific Google Drive folder for new files.
- Download newly added files.
- Split the file content into manageable text chunks.
- Generate embeddings for these chunks using OpenAI.
- Store these embeddings in a Pinecone vector store for efficient semantic search.
- Provide an AI chat agent interface that allows querying the stored knowledge base in real-time.

The workflow logically divides into two main blocks:

**1.1 Data Ingestion and Indexing Block:**  
Responsible for detecting new files in Google Drive, downloading them, processing their content into embeddings, and storing these embeddings in Pinecone.

**1.2 Query and AI Chat Agent Block:**  
Handles chat message triggers, processes user queries through an AI agent that leverages the Pinecone vector store as a knowledge base, and responds using OpenAI GPT-4o-mini.

--- 

### 2. Block-by-Block Analysis

#### 1.1 Data Ingestion and Indexing Block

- **Overview:**  
This block listens for new files added to a specific Google Drive folder, downloads the file, processes the content by splitting text and creating embeddings, and inserts these embeddings into a Pinecone vector store for later retrieval.

- **Nodes Involved:**  
  - Google Drive Trigger  
  - Google Drive  
  - Recursive Character Text Splitter  
  - Default Data Loader  
  - Embeddings OpenAI  
  - Pinecone Vector Store

- **Node Details:**

1. **Google Drive Trigger**  
   - *Type & Role:* Trigger node that monitors a Google Drive folder for new files.  
   - *Configuration:* Watches a specific folder (`folderToWatch` set to folder ID `1uf6zZN51rgAuQgid4-Oi314f6mJIQdiB`), triggers on `fileCreated` event, polling every minute.  
   - *Input/Output:* No input, outputs metadata about the new file.  
   - *Failure Modes:* Authentication errors if OAuth token expired; missing folder permissions; delays in file detection due to polling interval.  
   - *Credentials:* Google Drive OAuth2.

2. **Google Drive**  
   - *Type & Role:* Downloads the file identified by the trigger node.  
   - *Configuration:* Downloads file using the file ID from the trigger node’s output.  
   - *Input:* Receives file metadata from Google Drive Trigger.  
   - *Output:* Binary file content.  
   - *Failure Modes:* File not found if deleted before download; permission errors.  
   - *Credentials:* Google Drive OAuth2.

3. **Recursive Character Text Splitter**  
   - *Type & Role:* Splits the downloaded text into smaller, manageable chunks recursively to optimize embedding quality.  
   - *Configuration:* Default options, likely splitting by characters with recursive logic.  
   - *Input:* Connected indirectly via Default Data Loader node (which handles binary to text conversion).  
   - *Output:* Array of text chunks.  
   - *Failure Modes:* Improper text extraction from binary data; empty or binary files causing errors.

4. **Default Data Loader**  
   - *Type & Role:* Loads and converts the binary file content into a format suitable for text splitting and embedding generation.  
   - *Configuration:* Set to process binary data type.  
   - *Input:* Receives binary file data from Google Drive download node.  
   - *Output:* Document format acceptable by text splitter and embedding nodes.  
   - *Failure Modes:* Unsupported file types; corrupted files.

5. **Embeddings OpenAI**  
   - *Type & Role:* Generates vector embeddings for the text chunks using OpenAI embedding models.  
   - *Configuration:* Default embedding options without additional parameters.  
   - *Input:* Receives text chunks from the splitter/loader nodes.  
   - *Output:* Embeddings passed to Pinecone insertion.  
   - *Failure Modes:* API quota exceeded; network issues; invalid input data.  
   - *Credentials:* OpenAI API key.

6. **Pinecone Vector Store**  
   - *Type & Role:* Inserts embeddings into the Pinecone index named `ragfile`.  
   - *Configuration:* Mode set to "insert," targeting the `ragfile` Pinecone index.  
   - *Input:* Embeddings from OpenAI node and document metadata from data loader/splitter.  
   - *Output:* Confirmation of vector insertion.  
   - *Failure Modes:* Pinecone API errors; index not found; authorization failures.  
   - *Credentials:* Pinecone API key.

---

#### 1.2 Query and AI Chat Agent Block

- **Overview:**  
This block listens for incoming chat messages, uses an AI agent powered by OpenAI GPT-4o-mini to process queries, retrieves relevant knowledge from Pinecone vector store as a tool, and returns answers.

- **Nodes Involved:**  
  - When chat message received  
  - AI Agent  
  - OpenAI Chat Model  
  - Pinecone Vector Store1 (used as a retrieval tool)  
  - Embeddings OpenAI1 (used for query embedding)

- **Node Details:**

1. **When chat message received**  
   - *Type & Role:* Webhook trigger node that activates when a chat message is received (entry point for user queries).  
   - *Configuration:* No additional parameters; listens on webhook ID `fa37c7db-d78c-4a86-9de3-7cb6805de74f`.  
   - *Input:* External chat message events.  
   - *Output:* Passes chat input to AI Agent.  
   - *Failure Modes:* Webhook down; invalid or malformed chat messages.

2. **AI Agent**  
   - *Type & Role:* Core AI agent node coordinating language model and retrieval tool for response generation.  
   - *Configuration:* No custom options, uses connected nodes for embedding, retrieval, and language modeling.  
   - *Input:* Chat messages from trigger node; embeddings and retrieval results as tools.  
   - *Output:* AI-generated chat responses.  
   - *Failure Modes:* Timeout on model calls; tool connection failures; malformed queries.

3. **OpenAI Chat Model**  
   - *Type & Role:* Language model node using OpenAI GPT-4o-mini for generating chat responses.  
   - *Configuration:* Model explicitly set to `gpt-4o-mini`.  
   - *Input:* Receives prompts from AI Agent.  
   - *Output:* Generated text completions or chat responses.  
   - *Failure Modes:* API rate limits; network issues.  
   - *Credentials:* OpenAI API key.

4. **Pinecone Vector Store1**  
   - *Type & Role:* Acts as a retrieval tool for the AI Agent, fetching relevant documents from Pinecone index `ragfile`.  
   - *Configuration:* Mode set to "retrieve-as-tool" with tool name `knowledge_base`, described as a database access tool.  
   - *Input:* Receives query embeddings from Embeddings OpenAI1.  
   - *Output:* Returns relevant document vectors to AI Agent for contextual response generation.  
   - *Failure Modes:* Pinecone indexing errors; retrieval failures; auth issues.  
   - *Credentials:* Pinecone API key.

5. **Embeddings OpenAI1**  
   - *Type & Role:* Produces embeddings for the incoming chat queries to facilitate semantic search in Pinecone.  
   - *Configuration:* Default embedding options.  
   - *Input:* Chat messages from trigger node or AI Agent.  
   - *Output:* Query embeddings fed into Pinecone Vector Store1.  
   - *Failure Modes:* API quota limits; invalid input.

---

### 3. Summary Table

| Node Name                   | Node Type                                     | Functional Role                                  | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                           |
|-----------------------------|-----------------------------------------------|-------------------------------------------------|---------------------------|--------------------------|-----------------------------------------------------------------------------------------------------|
| Google Drive Trigger         | n8n-nodes-base.googleDriveTrigger             | Watches Google Drive folder for new files       | -                         | Google Drive             | ## Instant RaG Builder: Drive to Pinecone **Author** David Olusola Set Up :✅ Connect credentials: Google Drive, OpenAI, Pinecone Upload file to: Google Drive folder (auto-watched) Workflow does the rest: - Downloads file - Splits text - Creates embeddings - Stores in Pinecone Send a chat message to the Agent → Query stored knowledge instantly! |
| Google Drive                | n8n-nodes-base.googleDrive                     | Downloads new file from Drive                    | Google Drive Trigger       | Pinecone Vector Store    | See above                                                                                           |
| Default Data Loader          | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Converts binary file to document for processing | Recursive Character Text Splitter  | Pinecone Vector Store    | See above                                                                                           |
| Recursive Character Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Splits text into chunks                          | Default Data Loader        | Embeddings OpenAI        | See above                                                                                           |
| Embeddings OpenAI            | @n8n/n8n-nodes-langchain.embeddingsOpenAi     | Creates embeddings from text chunks             | Recursive Character Text Splitter | Pinecone Vector Store    | See above                                                                                           |
| Pinecone Vector Store        | @n8n/n8n-nodes-langchain.vectorStorePinecone | Inserts embeddings into Pinecone index           | Google Drive, Default Data Loader, Embeddings OpenAI | -                        | See above                                                                                           |
| When chat message received   | @n8n/n8n-nodes-langchain.chatTrigger           | Receives user chat queries                       | -                         | AI Agent                 | See above                                                                                           |
| AI Agent                    | @n8n/n8n-nodes-langchain.agent                 | Coordinates chat input, retrieval & response    | When chat message received, Pinecone Vector Store1, OpenAI Chat Model | OpenAI Chat Model        | See above                                                                                           |
| OpenAI Chat Model            | @n8n/n8n-nodes-langchain.lmChatOpenAi          | Generates chat responses using GPT-4o-mini      | AI Agent                  | AI Agent                 | See above                                                                                           |
| Pinecone Vector Store1       | @n8n/n8n-nodes-langchain.vectorStorePinecone  | Retrieves relevant vectors for AI Agent          | Embeddings OpenAI1        | AI Agent                 | See above                                                                                           |
| Embeddings OpenAI1           | @n8n/n8n-nodes-langchain.embeddingsOpenAi     | Creates embeddings for query text                | When chat message received | Pinecone Vector Store1   | See above                                                                                           |
| Sticky Note                 | n8n-nodes-base.stickyNote                       | Documentation note                               | -                         | -                        | ## Instant RaG Builder: Drive to Pinecone **Author** David Olusola Set Up :✅ Connect credentials: Google Drive, OpenAI, Pinecone Upload file to: Google Drive folder (auto-watched) Workflow does the rest: - Downloads file - Splits text - Creates embeddings - Stores in Pinecone Send a chat message to the Agent → Query stored knowledge instantly! |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node**  
   - Type: Google Drive Trigger  
   - Set trigger event: `fileCreated`  
   - Poll interval: every minute  
   - Folder to watch: specify folder ID `1uf6zZN51rgAuQgid4-Oi314f6mJIQdiB`  
   - Connect Google Drive OAuth2 credentials.

2. **Create Google Drive node**  
   - Type: Google Drive  
   - Operation: Download file  
   - File ID: set to `={{ $json.id }}` from trigger output  
   - Connect Google Drive OAuth2 credentials.  
   - Connect input from Google Drive Trigger node.

3. **Create Default Data Loader node**  
   - Type: Document Default Data Loader  
   - Data type: Binary  
   - Connect input from Google Drive node output.

4. **Create Recursive Character Text Splitter node**  
   - Type: Recursive Character Text Splitter  
   - Use default options for splitting text recursively into chunks.  
   - Connect input from Default Data Loader node output.

5. **Create Embeddings OpenAI node**  
   - Type: Embeddings OpenAI  
   - Use default embedding options.  
   - Connect OpenAI credentials.  
   - Connect input from Recursive Character Text Splitter output.

6. **Create Pinecone Vector Store node (insertion mode)**  
   - Type: Pinecone Vector Store  
   - Mode: Insert  
   - Pinecone index: `ragfile`  
   - Connect Pinecone API credentials.  
   - Connect inputs from Google Drive node (file metadata), Default Data Loader, and Embeddings OpenAI outputs.  
   - Connect input from Google Drive node (main) and Embeddings OpenAI (ai_embedding).

7. **Create When chat message received node**  
   - Type: Chat Trigger  
   - No additional parameters required  
   - Connect webhook credentials (automatically generated).  

8. **Create Embeddings OpenAI1 node**  
   - Type: Embeddings OpenAI  
   - Default options  
   - Connect OpenAI credentials.  
   - Connect input from When chat message received node.

9. **Create Pinecone Vector Store1 node (retrieval mode)**  
   - Type: Pinecone Vector Store  
   - Mode: Retrieve as tool  
   - Tool name: `knowledge_base`  
   - Tool description: "call this tool to access the database"  
   - Pinecone index: `ragfile`  
   - Connect Pinecone API credentials.  
   - Connect input from Embeddings OpenAI1 node.

10. **Create OpenAI Chat Model node**  
    - Type: OpenAI Chat Model  
    - Model: `gpt-4o-mini`  
    - Connect OpenAI credentials.  

11. **Create AI Agent node**  
    - Type: AI Agent  
    - Connect inputs from:  
      - When chat message received (main input)  
      - OpenAI Chat Model (ai_languageModel)  
      - Pinecone Vector Store1 (ai_tool)  
    - Connect OpenAI Chat Model output back to AI Agent for response generation.

12. **Connect AI Agent output to OpenAI Chat Model input**  
    - Ensures the AI Agent uses the language model for generating responses.

13. **Add Sticky Note for documentation**  
    - Content:  
      ```
      ## Instant RaG Builder: Drive to Pinecone

      Author: David Olusola

      Set Up: ✅ Connect credentials: Google Drive, OpenAI, Pinecone

      Upload file to: Google Drive folder (auto-watched)

      Workflow does the rest:  
      - Downloads file  
      - Splits text  
      - Creates embeddings  
      - Stores in Pinecone

      Send a chat message to the Agent → Query stored knowledge instantly!
      ```

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                            |
|------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Workflow authored by David Olusola.                                                            | Author attribution.                                        |
| Workflow requires three credentials setup: Google Drive OAuth2, OpenAI API key, Pinecone API key. | Credential setup instructions.                             |
| Google Drive folder ID to watch: `1uf6zZN51rgAuQgid4-Oi314f6mJIQdiB`                            | Folder ID where files should be uploaded for processing.  |
| OpenAI model used: `gpt-4o-mini`                                                               | Lightweight GPT-4 variant optimized for chat generation.  |
| Pinecone index name: `ragfile`                                                                 | Vector store index used for storing and retrieving vectors.|
| Polling interval for Google Drive trigger: every minute                                        | Delay in detection of new files depends on this setting.  |

---

**Disclaimer:**  
The text provided above is extracted and analyzed exclusively from an automated workflow created with n8n, a workflow automation tool. The processing respects all current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.