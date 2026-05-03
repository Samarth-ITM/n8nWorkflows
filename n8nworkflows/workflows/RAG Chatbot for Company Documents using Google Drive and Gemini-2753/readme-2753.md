RAG Chatbot for Company Documents using Google Drive and Gemini

https://n8nworkflows.xyz/workflows/rag-chatbot-for-company-documents-using-google-drive-and-gemini-2753


# RAG Chatbot for Company Documents using Google Drive and Gemini

### 1. Workflow Overview

This workflow implements a Retrieval Augmented Generation (RAG) chatbot designed to answer employee questions based on company documents stored in a dedicated Google Drive folder. It automatically indexes new or updated documents into a Pinecone vector database, enabling the chatbot to retrieve relevant information and generate accurate, context-aware responses using Google’s Gemini AI models.

The workflow is logically divided into two main functional blocks:

**1.1 Document Indexing Block**  
Monitors a specific Google Drive folder for new or updated files, downloads these documents, processes their content by splitting into manageable chunks, generates embeddings using Google Gemini, and indexes these embeddings into a Pinecone vector store.

**1.2 Chat Interaction Block**  
Handles incoming user questions via a chat interface, retrieves relevant document chunks from Pinecone using vector similarity search, and generates comprehensive answers with Google Gemini chat models. It also maintains short-term conversational memory for context-aware interactions.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Indexing Block

**Overview:**  
This block automates the ingestion and indexing of company documents stored in Google Drive. It triggers on file creation or updates, downloads the files, extracts and splits their content, generates embeddings, and inserts them into the Pinecone vector database.

**Nodes Involved:**  
- Google Drive File Created  
- Google Drive File Updated  
- Download File From Google Drive  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings Google Gemini  
- Pinecone Vector Store  
- Sticky Note (comment on document indexing)

**Node Details:**

- **Google Drive File Created**  
  - *Type:* Trigger node  
  - *Role:* Detects new files added to a specific Google Drive folder  
  - *Configuration:* Watches a specified folder (configured by folder ID) with polling every minute  
  - *Input:* None (trigger)  
  - *Output:* File metadata (including file ID and name)  
  - *Failure modes:* API rate limits, folder permission errors  
  - *Notes:* Requires Google Drive OAuth2 credentials

- **Google Drive File Updated**  
  - *Type:* Trigger node  
  - *Role:* Detects file updates in the same Google Drive folder  
  - *Configuration:* Watches the same folder as above, polling every minute  
  - *Input:* None (trigger)  
  - *Output:* Updated file metadata  
  - *Failure modes:* Same as above

- **Download File From Google Drive**  
  - *Type:* Google Drive node  
  - *Role:* Downloads the detected file content for processing  
  - *Configuration:* Uses file ID and name from trigger nodes to download the file binary  
  - *Input:* File metadata from triggers  
  - *Output:* Binary file data  
  - *Failure modes:* File access errors, download timeouts, large file handling issues  
  - *Credentials:* Google Drive OAuth2

- **Recursive Character Text Splitter**  
  - *Type:* Text splitter node  
  - *Role:* Splits document content into smaller overlapping chunks for embedding  
  - *Configuration:* Chunk overlap set to 100 characters, default chunk size (recursive splitting)  
  - *Input:* Document text (binary converted to text by Default Data Loader)  
  - *Output:* Array of text chunks  
  - *Failure modes:* Improper text extraction, encoding issues

- **Default Data Loader**  
  - *Type:* Document loader node  
  - *Role:* Converts binary file data into text documents for further processing  
  - *Configuration:* Loads data from a specific binary field  
  - *Input:* Binary file data from Google Drive download  
  - *Output:* Text document(s)  
  - *Failure modes:* Unsupported file formats, corrupted files

- **Embeddings Google Gemini**  
  - *Type:* Embeddings generation node  
  - *Role:* Generates vector embeddings for each text chunk using Google Gemini’s text-embedding-004 model  
  - *Configuration:* Model set to "models/text-embedding-004"  
  - *Input:* Text chunks from splitter  
  - *Output:* Embeddings vectors  
  - *Failure modes:* API quota limits, network errors, invalid input text  
  - *Credentials:* Google Gemini (PaLM) API key

- **Pinecone Vector Store**  
  - *Type:* Vector store node  
  - *Role:* Inserts embeddings and associated text chunks into the Pinecone index named "company-files"  
  - *Configuration:* Mode set to "insert"  
  - *Input:* Embeddings and documents from Embeddings Google Gemini and Default Data Loader  
  - *Output:* Confirmation of insertion  
  - *Failure modes:* Pinecone API errors, index misconfiguration, network issues  
  - *Credentials:* Pinecone API key

- **Sticky Note (Add documents to vector store)**  
  - *Role:* Provides a descriptive comment on the indexing process  
  - *Content:* "Add documents to vector store when updating or creating new documents in Google Drive"

---

#### 2.2 Chat Interaction Block

**Overview:**  
This block handles user queries via a chat interface, retrieves relevant document chunks from Pinecone using vector similarity search, and generates answers using Google Gemini chat models. It also maintains short-term memory for context-aware conversations.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- Vector Store Tool  
- Pinecone Vector Store (Retrieval)  
- Embeddings Google Gemini (retrieval)  
- Google Gemini Chat Model (retrieval)  
- Window Buffer Memory  
- Sticky Note (Chat with company documents)

**Node Details:**

- **When chat message received**  
  - *Type:* Chat trigger node  
  - *Role:* Receives user chat messages via webhook  
  - *Configuration:* Webhook ID configured, no additional options  
  - *Input:* External chat messages  
  - *Output:* User question text  
  - *Failure modes:* Webhook connectivity, malformed input

- **AI Agent**  
  - *Type:* Agent node  
  - *Role:* Orchestrates the question answering process using tools and language models  
  - *Configuration:*  
    - System message defines the assistant as a helpful HR assistant answering based on company policies  
    - Uses a tool named "company_documents_tool" (Vector Store Tool) to retrieve document info  
  - *Input:* User question from chat trigger  
  - *Output:* Generated answer  
  - *Failure modes:* Tool invocation errors, model response errors, memory handling issues  
  - *Version:* 1.7

- **Vector Store Tool**  
  - *Type:* Tool node  
  - *Role:* Provides access to the Pinecone vector store for document retrieval  
  - *Configuration:* Named "company_documents_tool" with description for retrieving company document info  
  - *Input:* Query embeddings from AI Agent  
  - *Output:* Retrieved relevant document chunks  
  - *Failure modes:* Pinecone query errors, embedding mismatches

- **Pinecone Vector Store (Retrieval)**  
  - *Type:* Vector store node  
  - *Role:* Queries the Pinecone index "company-files" to retrieve relevant document chunks based on embeddings  
  - *Configuration:* Uses the same Pinecone index as indexing node  
  - *Input:* Embeddings from Embeddings Google Gemini (retrieval)  
  - *Output:* Retrieved documents for Vector Store Tool  
  - *Failure modes:* Query timeouts, index unavailability  
  - *Credentials:* Pinecone API key

- **Embeddings Google Gemini (retrieval)**  
  - *Type:* Embeddings generation node  
  - *Role:* Generates embeddings for the user question using the same text-embedding-004 model  
  - *Input:* User question text  
  - *Output:* Query embeddings for Pinecone retrieval  
  - *Failure modes:* API errors, invalid input  
  - *Credentials:* Google Gemini (PaLM) API key

- **Google Gemini Chat Model (retrieval)**  
  - *Type:* Language model node  
  - *Role:* Generates the final answer using the "gemini-2.0-flash-exp" chat model based on retrieved documents and user query  
  - *Input:* Context and question from AI Agent and Vector Store Tool  
  - *Output:* Chatbot answer  
  - *Failure modes:* API quota, response latency  
  - *Credentials:* Google Gemini (PaLM) API key

- **Window Buffer Memory**  
  - *Type:* Memory node  
  - *Role:* Maintains short-term conversational memory to provide context-aware answers  
  - *Input:* Conversation history from AI Agent  
  - *Output:* Updated memory state for AI Agent  
  - *Failure modes:* Memory overflow, data corruption

- **Sticky Note (Chat with company documents)**  
  - *Role:* Describes the chat interaction block  
  - *Content:* "Chat with company documents"

---

### 3. Summary Table

| Node Name                       | Node Type                                   | Functional Role                            | Input Node(s)                      | Output Node(s)                   | Sticky Note                                      |
|--------------------------------|---------------------------------------------|--------------------------------------------|----------------------------------|---------------------------------|-------------------------------------------------|
| Google Drive File Created       | Google Drive Trigger                         | Detect new files in Google Drive folder    | None                             | Download File From Google Drive  |                                                 |
| Google Drive File Updated       | Google Drive Trigger                         | Detect updated files in Google Drive folder| None                             | Download File From Google Drive  |                                                 |
| Download File From Google Drive | Google Drive                                | Download file binary content                | Google Drive File Created, Updated| Pinecone Vector Store            |                                                 |
| Recursive Character Text Splitter| Text Splitter                              | Split document text into chunks             | Default Data Loader              | Default Data Loader              |                                                 |
| Default Data Loader             | Document Loader                             | Convert binary file to text document        | Recursive Character Text Splitter| Pinecone Vector Store            |                                                 |
| Embeddings Google Gemini        | Embeddings Generator                        | Generate embeddings for document chunks     | Default Data Loader              | Pinecone Vector Store            |                                                 |
| Pinecone Vector Store           | Vector Store                                | Insert embeddings into Pinecone index       | Embeddings Google Gemini         | None                           |                                                 |
| Google Drive File Created       | Google Drive Trigger                         | Detect new files                            | None                             | Download File From Google Drive  |                                                 |
| Google Drive File Updated       | Google Drive Trigger                         | Detect updated files                        | None                             | Download File From Google Drive  |                                                 |
| Sticky Note                    | Sticky Note                                 | Comment on indexing process                  | None                             | None                           | "Add documents to vector store when updating or creating new documents in Google Drive" |
| When chat message received      | Chat Trigger                               | Receive user chat messages                   | None                             | AI Agent                       |                                                 |
| AI Agent                      | Agent                                      | Orchestrate question answering               | When chat message received       | None                           |                                                 |
| Vector Store Tool              | Tool (Vector Store)                         | Retrieve relevant document chunks            | Pinecone Vector Store (Retrieval)| AI Agent                      |                                                 |
| Pinecone Vector Store (Retrieval)| Vector Store                             | Query Pinecone index for relevant chunks    | Embeddings Google Gemini (retrieval)| Vector Store Tool           |                                                 |
| Embeddings Google Gemini (retrieval)| Embeddings Generator                   | Generate embeddings for user query           | When chat message received       | Pinecone Vector Store (Retrieval)|                                                 |
| Google Gemini Chat Model (retrieval)| Language Model                         | Generate answer based on retrieved docs      | Vector Store Tool                | None                           |                                                 |
| Window Buffer Memory           | Memory Buffer                              | Maintain short-term conversational memory   | AI Agent                        | AI Agent                       |                                                 |
| Sticky Note2                   | Sticky Note                                 | Comment on chat interaction block            | None                             | None                           | "Chat with company documents"                    |
| Sticky Note1                   | Sticky Note                                 | Setup instructions and prerequisites         | None                             | None                           | See section 5 for full content                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Triggers**  
   - Add two Google Drive Trigger nodes:  
     - One configured for `fileCreated` event, watching the specific Google Drive folder by folder ID.  
     - One configured for `fileUpdated` event, watching the same folder.  
   - Set polling interval to every minute.  
   - Assign Google Drive OAuth2 credentials.

2. **Download File Node**  
   - Add a Google Drive node configured to `download` operation.  
   - Use expression to set `fileId` from trigger node output (`{{$json.id}}`).  
   - Use file name from trigger for `fileName` option.  
   - Connect both Google Drive Trigger nodes to this node.  
   - Assign Google Drive OAuth2 credentials.

3. **Recursive Character Text Splitter**  
   - Add a Recursive Character Text Splitter node.  
   - Set chunk overlap to 100 characters (default chunk size).  
   - Connect output of Default Data Loader node to this node (see next step).

4. **Default Data Loader**  
   - Add a Default Data Loader node.  
   - Configure to load data from the binary field containing the downloaded file.  
   - Connect output of Download File node to this node.  
   - Connect output of Recursive Character Text Splitter node to this node.

5. **Embeddings Google Gemini (Indexing)**  
   - Add an Embeddings Google Gemini node.  
   - Set model to `models/text-embedding-004`.  
   - Connect output of Default Data Loader (text chunks) to this node.  
   - Assign Google Gemini (PaLM) API credentials.

6. **Pinecone Vector Store (Insert Mode)**  
   - Add a Pinecone Vector Store node.  
   - Set mode to `insert`.  
   - Select your Pinecone index named `company-files`.  
   - Connect output of Embeddings Google Gemini node to this node.  
   - Assign Pinecone API credentials.

7. **Chat Trigger Node**  
   - Add a Chat Trigger node.  
   - Configure webhook ID (auto-generated or custom).  
   - No special options needed.

8. **AI Agent Node**  
   - Add an AI Agent node.  
   - Configure system message to define assistant role and usage of the tool named `company_documents_tool`.  
   - Connect Chat Trigger node output to AI Agent input.  
   - Connect Window Buffer Memory node to AI Agent for memory management.

9. **Vector Store Tool Node**  
   - Add a Vector Store Tool node.  
   - Name it `company_documents_tool`.  
   - Provide a description indicating it retrieves information from company documents.  
   - Connect Pinecone Vector Store (Retrieval) node output to this node.  
   - Connect this tool node to AI Agent’s tool input.

10. **Embeddings Google Gemini (Retrieval)**  
    - Add another Embeddings Google Gemini node for query embedding generation.  
    - Use the same model `models/text-embedding-004`.  
    - Connect Chat Trigger node output (user question) to this node.  
    - Assign Google Gemini API credentials.

11. **Pinecone Vector Store (Retrieval Mode)**  
    - Add a Pinecone Vector Store node configured for retrieval/query mode.  
    - Select the same `company-files` index.  
    - Connect Embeddings Google Gemini (retrieval) output to this node.  
    - Assign Pinecone API credentials.

12. **Google Gemini Chat Model (Retrieval)**  
    - Add a Google Gemini Chat Model node.  
    - Set model to `models/gemini-2.0-flash-exp`.  
    - Connect Vector Store Tool output to this node.  
    - Assign Google Gemini API credentials.

13. **Window Buffer Memory**  
    - Add a Window Buffer Memory node.  
    - Connect it to AI Agent node’s memory input and output to maintain conversation context.

14. **Connect Outputs**  
    - Connect AI Agent output to the chat response output (e.g., webhook response).  
    - Ensure all nodes are properly connected as per the logical flow.

15. **Credentials Setup**  
    - Configure and assign credentials for:  
      - Google Drive OAuth2 API  
      - Google Gemini (PaLM) API key  
      - Pinecone API key

16. **Folder and Index Configuration**  
    - Update Google Drive Trigger nodes to watch the dedicated company documents folder.  
    - Configure Pinecone Vector Store nodes to use the `company-files` index.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Setup steps include creating a Google Cloud project, enabling Vertex AI API, obtaining Google AI API key, Pinecone account and index.   | See Sticky Note1 in workflow for detailed setup instructions.                                   |
| Workflow uses Google Gemini AI models for both embeddings (`text-embedding-004`) and chat generation (`gemini-2.0-flash-exp`).            | Requires Google Gemini (PaLM) API credentials.                                                 |
| Pinecone index named `company-files` must be created and configured prior to workflow use.                                               | Pinecone dashboard: https://app.pinecone.io/                                                   |
| Google Drive folder must be dedicated for company documents and accessible via OAuth2 credentials.                                        | Google Drive folder URL example: https://drive.google.com/drive/folders/1evDIoHePhjw_LgVFZXSZyK1sZm2GHp9W |
| The AI Agent uses a tool named `company_documents_tool` to retrieve relevant document chunks from Pinecone during chat interactions.    |                                                                                               |
| The Window Buffer Memory node enables short-term memory for more natural conversations.                                                  |                                                                                               |

---

This documentation provides a comprehensive understanding of the workflow’s structure, node configurations, and operational logic to enable reproduction, modification, and troubleshooting.