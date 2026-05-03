Basic RAG chat

https://n8nworkflows.xyz/workflows/basic-rag-chat-5028


# Basic RAG chat

### 1. Workflow Overview

This workflow, titled **Basic RAG chat**, implements a Retrieval-Augmented Generation (RAG) pattern that enables chat interactions enhanced by a custom knowledge base. It is designed for use cases where users want to query documents stored externally and receive AI-generated answers grounded in those documents. The workflow primarily deals with loading external data, chunking and embedding it into an in-memory vector store, and then responding to chat queries by retrieving relevant chunks and formulating answers via a large language model.

The workflow is logically divided into two main blocks:

- **1.1 Data Loading and Index Building:** Fetches a text file from disk, splits it into chunks, creates vector embeddings, and stores them in an in-memory vector database.

- **1.2 Chat Query Handling:** Listens for chat inputs, embeds the query, retrieves relevant document chunks from the vector store, and uses a chat language model to generate answers based on the retrieved data.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Loading and Index Building

**Overview:**  
This block is responsible for loading a text file from disk, splitting the text into manageable chunks, embedding those chunks using Cohere embeddings, and indexing them into an in-memory vector store for efficient retrieval.

**Nodes Involved:**  
- When clicking 'Test Workflow' button  
- Read/Write Files from Disk  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings Cohere  
- In-Memory Vector Store  

**Node Details:**

- **When clicking 'Test Workflow' button**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow manually to initiate data loading.  
  - *Config:* No special parameters.  
  - *Connections:* Outputs to "Read/Write Files from Disk".  
  - *Edge cases:* None; user-triggered.

- **Read/Write Files from Disk**  
  - *Type:* File Reader/Writer  
  - *Role:* Reads the source text file containing the document to be indexed.  
  - *Config:* Reads file from `/tmp/external_data/news.txt`.  
  - *Connections:* Outputs file data to "In-Memory Vector Store" (main input).  
  - *Edge cases:* File not found or access denied errors; binary or encoding issues.

- **Recursive Character Text Splitter**  
  - *Type:* Langchain Text Splitter  
  - *Role:* Splits large documents into smaller chunks with overlap for better context retention.  
  - *Config:* Splitting mode uses "markdown" code, chunk overlap set to 50 characters.  
  - *Connections:* Outputs chunks to "Default Data Loader".  
  - *Edge cases:* Could fail if input text is empty or improperly formatted.

- **Default Data Loader**  
  - *Type:* Langchain Document Data Loader  
  - *Role:* Converts chunked text into documents consumable by embedding models.  
  - *Config:* Input data treated as binary.  
  - *Connections:* Outputs document data to "In-Memory Vector Store" (ai_document input).  
  - *Edge cases:* Input data format mismatches.

- **Embeddings Cohere**  
  - *Type:* Langchain Embedding Node  
  - *Role:* Generates multilingual embeddings using Cohere's `embed-multilingual-v3.0` model.  
  - *Config:* Model name set; uses stored Cohere API credentials.  
  - *Connections:* Outputs embeddings to "In-Memory Vector Store" (ai_embedding input).  
  - *Edge cases:* API authentication failures, rate limits, network issues.

- **In-Memory Vector Store**  
  - *Type:* Langchain Vector Store  
  - *Role:* Stores document embeddings for retrieval. Set to insert mode and clears store on new data load.  
  - *Config:* `mode` set to "insert"; `clearStore` true (clears previous embeddings).  
  - *Connections:*  
    - Inputs:  
      - Main input from "Read/Write Files from Disk" (file data)  
      - ai_embedding from "Embeddings Cohere"  
      - ai_document from "Default Data Loader"  
    - Outputs to "Vector Store Retriever" (in other block).  
  - *Edge cases:* Memory limits if large data; failure to clear store properly.

---

#### 2.2 Chat Query Handling

**Overview:**  
This block handles incoming chat messages, embeds the user query, retrieves relevant chunks from the vector store, and formulates a response using a chat language model.

**Nodes Involved:**  
- When clicking 'Chat' button below  
- Question and Answer Chain  
- Vector Store Retriever  
- In-Memory Vector Store1  
- Embeddings Cohere1  
- Groq Chat Model  

**Node Details:**

- **When clicking 'Chat' button below**  
  - *Type:* Langchain Chat Trigger (webhook)  
  - *Role:* Listens for chat messages from external clients.  
  - *Config:* Webhook ID assigned for external trigger.  
  - *Connections:* Outputs chat input to "Question and Answer Chain".  
  - *Edge cases:* Webhook downtime, malformed payloads.

- **Question and Answer Chain**  
  - *Type:* Langchain Retrieval QA Chain  
  - *Role:* Orchestrates query embedding, document retrieval, and answer generation.  
  - *Config:* Uses default options; receives retriever and language model inputs.  
  - *Connections:*  
    - Inputs:  
      - ai_retriever from "Vector Store Retriever"  
      - ai_languageModel from "Groq Chat Model"  
    - Output: Final chat answer.  
  - *Edge cases:* Failures in subcomponents, model API limits.

- **Vector Store Retriever**  
  - *Type:* Langchain Retriever  
  - *Role:* Retrieves relevant document chunks from the vector store based on query embeddings.  
  - *Config:* Default parameters.  
  - *Connections:*  
    - Input: ai_vectorStore from "In-Memory Vector Store1"  
    - Output: ai_retriever to "Question and Answer Chain".  
  - *Edge cases:* Empty retrieval results, query embedding mismatches.

- **In-Memory Vector Store1**  
  - *Type:* Langchain Vector Store (In-Memory)  
  - *Role:* Provides vector store for retrieval.  
  - *Config:* Default parameters; receives embeddings from "Embeddings Cohere1".  
  - *Connections:*  
    - Input: ai_embedding from "Embeddings Cohere1"  
    - Output: ai_vectorStore to "Vector Store Retriever".  
  - *Edge cases:* Data consistency with main vector store; memory limits.

- **Embeddings Cohere1**  
  - *Type:* Langchain Embeddings  
  - *Role:* Embeds incoming chat queries using same multilingual model as data loader.  
  - *Config:* Model name `embed-multilingual-v3.0`; uses Cohere API credentials.  
  - *Connections:* Outputs embeddings to "In-Memory Vector Store1".  
  - *Edge cases:* API errors, rate limiting.

- **Groq Chat Model**  
  - *Type:* Langchain Chat Model (Groq)  
  - *Role:* Generates chat answers based on retrieved documents.  
  - *Config:* Uses `llama-3.3-70b-versatile` model with options empty; note indicates usage in Traditional Chinese.  
  - *Connections:* Outputs to "Question and Answer Chain" (ai_languageModel input).  
  - *Edge cases:* Model API failures, language support issues.

---

### 3. Summary Table

| Node Name                         | Node Type                                      | Functional Role                            | Input Node(s)                | Output Node(s)              | Sticky Note                                                                                         |
|----------------------------------|------------------------------------------------|--------------------------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| When clicking 'Test Workflow' button | Manual Trigger                                | Initiates data loading workflow            |                             | Read/Write Files from Disk   |                                                                                                   |
| Read/Write Files from Disk        | File Reader/Writer                             | Reads source text file                      | When clicking 'Test Workflow' button | In-Memory Vector Store       |                                                                                                   |
| Recursive Character Text Splitter | Langchain Text Splitter                        | Splits document text into overlapping chunks |                             | Default Data Loader          |                                                                                                   |
| Default Data Loader               | Langchain Document Data Loader                 | Converts text chunks to documents           | Recursive Character Text Splitter | In-Memory Vector Store       |                                                                                                   |
| Embeddings Cohere                | Langchain Embeddings                           | Generates multilingual embeddings            |                             | In-Memory Vector Store       |                                                                                                   |
| In-Memory Vector Store            | Langchain Vector Store                         | Stores and indexes document embeddings      | Read/Write Files from Disk, Embeddings Cohere, Default Data Loader | Vector Store Retriever       |                                                                                                   |
| When clicking 'Chat' button below | Langchain Chat Trigger (Webhook)               | Receives chat queries                        |                             | Question and Answer Chain    |                                                                                                   |
| Question and Answer Chain         | Langchain Retrieval QA Chain                   | Orchestrates retrieval and answer generation | Vector Store Retriever, Groq Chat Model |                             |                                                                                                   |
| Vector Store Retriever            | Langchain Retriever                            | Retrieves relevant document chunks          | In-Memory Vector Store1      | Question and Answer Chain    |                                                                                                   |
| In-Memory Vector Store1           | Langchain Vector Store                         | Provides vector store for retrieval          | Embeddings Cohere1           | Vector Store Retriever       |                                                                                                   |
| Embeddings Cohere1               | Langchain Embeddings                           | Embeds incoming chat queries                 |                             | In-Memory Vector Store1      |                                                                                                   |
| Groq Chat Model                  | Langchain Chat Model                           | Generates chat answers based on context     |                             | Question and Answer Chain    | 使用繁體中文 (Uses Traditional Chinese)                                                           |
| Sticky Note                      | Sticky Note                                    | Notes on loading data into database          |                             |                             | ### Load data into database\nFetch file from Google Drive, split it into chunks and insert into Pinecone index |
| Sticky Note1                     | Sticky Note                                    | Notes on chat with database                   |                             |                             | ### Chat with database\nEmbed the incoming chat message and use it retrieve relevant chunks from the vector store. These are passed to the model to formulate an answer |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking 'Test Workflow' button`  
   - Type: Manual Trigger  
   - No special parameters.

2. **Create File Reader Node**  
   - Name: `Read/Write Files from Disk`  
   - Type: Read/Write File  
   - Parameter: File Selector set to `/tmp/external_data/news.txt`  
   - Connect output from Manual Trigger to this node input.

3. **Create Recursive Character Text Splitter Node**  
   - Name: `Recursive Character Text Splitter`  
   - Type: Langchain Recursive Character Text Splitter  
   - Parameters:  
     - Split code: `markdown`  
     - Chunk overlap: `50`  
   - Connect output from Read/Write Files node to this node input.

4. **Create Default Data Loader Node**  
   - Name: `Default Data Loader`  
   - Type: Langchain Document Default Data Loader  
   - Parameters:  
     - Data type: `binary`  
   - Connect output from Recursive Character Text Splitter to this node input.

5. **Create Embeddings Cohere Node**  
   - Name: `Embeddings Cohere`  
   - Type: Langchain Embeddings Cohere  
   - Parameters:  
     - Model Name: `embed-multilingual-v3.0`  
   - Credentials: Attach valid Cohere API credential.  
   - Connect output from Default Data Loader to this node input.

6. **Create In-Memory Vector Store Node**  
   - Name: `In-Memory Vector Store`  
   - Type: Langchain Vector Store In-Memory  
   - Parameters:  
     - Mode: `insert`  
     - Clear Store: `true` (to reset index on each run)  
   - Connect inputs:  
     - Main input from Read/Write Files from Disk node (file data)  
     - ai_embedding input from Embeddings Cohere node  
     - ai_document input from Default Data Loader node

7. **Create Langchain Chat Trigger Node**  
   - Name: `When clicking 'Chat' button below`  
   - Type: Langchain Chat Trigger  
   - Parameters: none  
   - Configure webhook URL as needed for external chat integration.

8. **Create Embeddings Cohere1 Node**  
   - Name: `Embeddings Cohere1`  
   - Type: Langchain Embeddings Cohere  
   - Parameters:  
     - Model Name: `embed-multilingual-v3.0`  
   - Credentials: Use same Cohere API credentials as previous.  
   - Connect chat input to this node for embedding user queries.

9. **Create In-Memory Vector Store1 Node**  
   - Name: `In-Memory Vector Store1`  
   - Type: Langchain Vector Store In-Memory  
   - Default parameters  
   - Connect ai_embedding input from Embeddings Cohere1 node.

10. **Create Vector Store Retriever Node**  
    - Name: `Vector Store Retriever`  
    - Type: Langchain Retriever Vector Store  
    - Default parameters  
    - Connect ai_vectorStore input from In-Memory Vector Store1 node.  
    - Output connects to retriever input of QA Chain.

11. **Create Groq Chat Model Node**  
    - Name: `Groq Chat Model`  
    - Type: Langchain Chat Model Groq  
    - Parameters:  
      - Model: `llama-3.3-70b-versatile`  
      - Options: empty/default  
    - Credentials: Attach valid Groq API credentials.  
    - Note: configured to support Traditional Chinese.  
    - Output connects to languageModel input of QA Chain.

12. **Create Question and Answer Chain Node**  
    - Name: `Question and Answer Chain`  
    - Type: Langchain Chain Retrieval QA  
    - Default parameters  
    - Inputs:  
      - ai_retriever from Vector Store Retriever  
      - ai_languageModel from Groq Chat Model  
    - Connect webhook output (chat trigger node) main output to QA Chain main input.

13. **Connect In-Memory Vector Store main output** to Vector Store Retriever ai_vectorStore input to close data retrieval loop.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow uses Cohere embeddings model `embed-multilingual-v3.0` for multilingual support. | Cohere API documentation: https://docs.cohere.ai/                                               |
| Groq Chat Model set to `llama-3.3-70b-versatile` supports Traditional Chinese output.          | Groq AI docs: https://docs.groq.ai/                                                             |
| Sticky Notes in the workflow provide high-level explanations of data loading and chat blocks. | Useful to keep for onboarding new users or maintainers.                                         |
| The workflow uses in-memory vector stores which are volatile; consider persistent vector DBs for production. | For production, Pinecone or similar vector DBs recommended.                                     |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.