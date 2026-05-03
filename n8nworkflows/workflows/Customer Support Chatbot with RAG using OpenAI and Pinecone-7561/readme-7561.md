Customer Support Chatbot with RAG using OpenAI and Pinecone

https://n8nworkflows.xyz/workflows/customer-support-chatbot-with-rag-using-openai-and-pinecone-7561


# Customer Support Chatbot with RAG using OpenAI and Pinecone

### 1. Workflow Overview

This workflow implements a **Customer Support Chatbot** leveraging Retrieval-Augmented Generation (RAG) by integrating OpenAI's language models and Pinecone vector database. It enables automated ingestion of new documents from a Google Drive folder, processes and indexes them into Pinecone, and then uses this indexed knowledge base to answer user chat queries with contextual, accurate responses based on stored data.

The workflow is logically divided into two main functional blocks:

- **1.1 Document Ingestion and Indexing:** Watches a specific Google Drive folder for new files, downloads them, splits their content, generates embeddings with OpenAI, and inserts these vectors into Pinecone for future retrieval.

- **1.2 Chat Query Handling and AI Response:** Listens for incoming chat messages, retrieves relevant information from Pinecone using semantic search and reranking, applies conversational memory, and generates user responses via OpenAI chat models constrained by retrieved knowledge.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion and Indexing

**Overview:**  
This block automatically triggers when new files are added to a specified Google Drive folder. It downloads the file, processes the document by splitting it into chunks, embeds these chunks using OpenAI embeddings, and finally inserts the vectors into the Pinecone vector store for later retrieval.

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
  - *Role:* Watches a specific folder ("Snapfectly RAG" folder) for any new file creation event, polling every minute  
  - *Configuration:* Trigger on `fileCreated` event in folder with ID `1cDW1qkq76TX5Hr4k_JWBZjMigOf4hdC3`  
  - *Credentials:* Google Drive OAuth2  
  - *Inputs:* None (trigger)  
  - *Outputs:* File metadata to "Download file" node  
  - *Failure Modes:* Google API quota limits, folder permission issues, network errors  

- **Download file**  
  - *Type:* Google Drive file download  
  - *Role:* Downloads the newly created file by file ID from the trigger  
  - *Configuration:* Uses dynamic file ID from trigger node (`{{$json.id}}`)  
  - *Credentials:* Google Drive OAuth2  
  - *Inputs:* Metadata from previous node  
  - *Outputs:* Raw file content passed to "Pinecone Vector Store"  
  - *Failure Modes:* File access errors, network timeouts, invalid file ID  

- **Recursive Character Text Splitter**  
  - *Type:* Langchain text splitter  
  - *Role:* Splits document text into manageable chunks recursively for embedding  
  - *Configuration:* Default options, no custom splitting rules specified  
  - *Inputs:* Document text from "Default Data Loader"  
  - *Outputs:* Text chunks to "Default Data Loader"  
  - *Failure Modes:* Improper input format, very large files causing memory issues  

- **Default Data Loader**  
  - *Type:* Langchain document loader  
  - *Role:* Loads and prepares documents, integrating text splitter for chunking  
  - *Configuration:* Custom text splitting mode enabled  
  - *Inputs:* Text chunks from splitter  
  - *Outputs:* Document chunks to "Pinecone Vector Store"  
  - *Failure Modes:* Document parsing errors, unsupported file types  

- **Embeddings OpenAI**  
  - *Type:* OpenAI embeddings generator  
  - *Role:* Converts document text chunks into vector embeddings  
  - *Configuration:* Default embedding model, no custom options  
  - *Credentials:* OpenAI API (Hostinger)  
  - *Inputs:* Document chunks  
  - *Outputs:* Embeddings to "Pinecone Vector Store"  
  - *Failure Modes:* API quota limits, network failures, invalid inputs  

- **Pinecone Vector Store (Insert Mode)**  
  - *Type:* Pinecone vector database node  
  - *Role:* Inserts embeddings into Pinecone index `snapfectly`  
  - *Configuration:* Insert mode with default options  
  - *Credentials:* Pinecone API  
  - *Inputs:* Embeddings from OpenAI, document chunks  
  - *Outputs:* None (terminal for ingestion flow)  
  - *Failure Modes:* API authentication errors, index unavailability, network issues  

---

#### 2.2 Chat Query Handling and AI Response

**Overview:**  
This block handles incoming user chat messages, employs vector search with reranking to retrieve relevant knowledge from Pinecone, maintains conversational memory context, and uses GPT-4.1-mini chat model to generate precise, source-aware answers based strictly on retrieved data.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- Simple Memory  
- OpenAI Chat Model  
- Vector Store (Pinecone Retrieve Mode)  
- Reranker Cohere  
- Embeddings OpenAI1  

**Node Details:**  

- **When chat message received**  
  - *Type:* Langchain chat trigger  
  - *Role:* Entry point for receiving chat messages from users via webhook  
  - *Configuration:* Default options, webhook ID `df5359c0-5a3e-43e8-8f66-409858dc3988`  
  - *Inputs:* Incoming chat message  
  - *Outputs:* Message to "AI Agent"  
  - *Failure Modes:* Webhook misconfiguration, authentication issues  

- **AI Agent**  
  - *Type:* Langchain agent node  
  - *Role:* Orchestrates user question answering using vector store retrieval and language model response  
  - *Configuration:*  
    - System message defines role and response rules emphasizing retrieval-based answers, no hallucination, and clarity of data source  
    - Connects to memory, language model, vector store, and reranker nodes  
  - *Inputs:* Chat message from trigger, context from memory and vector search  
  - *Outputs:* Final response to user  
  - *Failure Modes:* Agent logic errors, missing data in vector store, language model failures  

- **Simple Memory**  
  - *Type:* Langchain memory buffer window  
  - *Role:* Maintains conversational history context to provide continuity in chat  
  - *Configuration:* Default buffer window size (unspecified here)  
  - *Inputs:* Past chat context  
  - *Outputs:* Context to AI Agent  
  - *Failure Modes:* Memory overflow, state loss on restart  

- **OpenAI Chat Model**  
  - *Type:* Langchain GPT chat model  
  - *Role:* Generates language responses based on prompts and retrieved context  
  - *Configuration:* Model set to `gpt-4.1-mini` (a specific fine-tuned GPT-4 variant or smaller model)  
  - *Credentials:* OpenAI API (Hostinger)  
  - *Inputs:* Prompt from AI Agent  
  - *Outputs:* Generated response text  
  - *Failure Modes:* API limits, network errors, model availability  

- **Vector Store (Retrieve Mode)**  
  - *Type:* Pinecone vector store retrieval node  
  - *Role:* Retrieves top 10 relevant documents from Pinecone based on user query embeddings  
  - *Configuration:*  
    - Mode: retrieve-as-tool  
    - Uses reranker to improve relevance  
    - Pinecone index: `snapfectly`  
    - Tool description provided for agent usage  
  - *Credentials:* Pinecone API  
  - *Inputs:* Query embeddings from Embeddings OpenAI1  
  - *Outputs:* Retrieved documents to AI Agent  
  - *Failure Modes:* Retrieval errors, no results found, API issues  

- **Reranker Cohere**  
  - *Type:* Cohere reranker node  
  - *Role:* Reranks search results from Pinecone to improve relevance before passing to AI agent  
  - *Configuration:* Default reranking options  
  - *Credentials:* Cohere API  
  - *Inputs:* Retrieved documents from Vector Store  
  - *Outputs:* Reordered results to Vector Store node  
  - *Failure Modes:* API quota, reranking failures  

- **Embeddings OpenAI1**  
  - *Type:* OpenAI embeddings generator  
  - *Role:* Converts incoming user query into embeddings for vector search  
  - *Credentials:* OpenAI API (Hostinger)  
  - *Inputs:* User chat message text  
  - *Outputs:* Query embeddings to Vector Store  
  - *Failure Modes:* API errors, invalid input text  

---

### 3. Summary Table

| Node Name                | Node Type                                       | Functional Role                             | Input Node(s)               | Output Node(s)                  | Sticky Note                                                   |
|--------------------------|------------------------------------------------|---------------------------------------------|-----------------------------|--------------------------------|---------------------------------------------------------------|
| Google Drive Trigger      | n8n-nodes-base.googleDriveTrigger               | Watches folder for new files                  | -                           | Download file                  | ## Insert documents into pinecone                             |
| Download file            | n8n-nodes-base.googleDrive                      | Downloads newly added files                    | Google Drive Trigger        | Pinecone Vector Store           | ## Insert documents into pinecone                             |
| Recursive Character Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Splits documents into chunks                  | Default Data Loader         | Default Data Loader             | ## Insert documents into pinecone                             |
| Default Data Loader      | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Loads and prepares document chunks             | Recursive Character Text Splitter | Pinecone Vector Store          | ## Insert documents into pinecone                             |
| Embeddings OpenAI        | @n8n/n8n-nodes-langchain.embeddingsOpenAi      | Generates embeddings for documents             | Default Data Loader         | Pinecone Vector Store           | ## Insert documents into pinecone                             |
| Pinecone Vector Store    | @n8n/n8n-nodes-langchain.vectorStorePinecone   | Inserts vectors into Pinecone index             | Download file, Embeddings OpenAI, Default Data Loader | -                          | ## Insert documents into pinecone                             |
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger           | Receives user chat messages                     | -                           | AI Agent                      | ## Agent                                                     |
| AI Agent                 | @n8n/n8n-nodes-langchain.agent                  | Orchestrates query answering with retrieval    | When chat message received, Simple Memory, OpenAI Chat Model, Vector Store | -                          | ## Agent                                                     |
| Simple Memory            | @n8n/n8n-nodes-langchain.memoryBufferWindow    | Maintains conversation memory                   | -                           | AI Agent                      | ## Agent                                                     |
| OpenAI Chat Model        | @n8n/n8n-nodes-langchain.lmChatOpenAi           | Generates language model responses              | AI Agent                    | AI Agent                      | ## Agent                                                     |
| Vector Store             | @n8n/n8n-nodes-langchain.vectorStorePinecone    | Retrieves relevant documents from Pinecone      | Embeddings OpenAI1, Reranker Cohere | AI Agent                      | ## Agent                                                     |
| Reranker Cohere          | @n8n/n8n-nodes-langchain.rerankerCohere         | Reranks search results for relevance            | Vector Store                | Vector Store                  | ## Agent                                                     |
| Embeddings OpenAI1       | @n8n/n8n-nodes-langchain.embeddingsOpenAi        | Generates embeddings for user query              | When chat message received  | Vector Store                  | ## Agent                                                     |
| Sticky Note              | n8n-nodes-base.stickyNote                        | Visual grouping label                            | -                           | -                            | ## Agent                                                     |
| Sticky Note1             | n8n-nodes-base.stickyNote                        | Visual grouping label                            | -                           | -                            | ## Insert documents into pinecone                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node:**
   - Type: Google Drive Trigger  
   - Configure event to `fileCreated`  
   - Set folder to watch by folder ID (`1cDW1qkq76TX5Hr4k_JWBZjMigOf4hdC3`)  
   - Poll every minute  
   - Attach Google Drive OAuth2 credentials  

2. **Create Download file Node:**
   - Type: Google Drive  
   - Operation: Download  
   - File ID: Set dynamically to `{{$json["id"]}}` from trigger  
   - Attach same Google Drive OAuth2 credentials  
   - Connect from Google Drive Trigger node  

3. **Create Recursive Character Text Splitter Node:**
   - Type: Langchain Recursive Character Text Splitter  
   - Use default options  
   - Connect output to Default Data Loader (see next step)  

4. **Create Default Data Loader Node:**
   - Type: Langchain Default Data Loader  
   - Enable custom text splitting mode  
   - Connect input from Recursive Character Text Splitter node  
   - Connect output to Embeddings OpenAI node  

5. **Create Embeddings OpenAI Node (for documents):**
   - Type: Langchain Embeddings OpenAI  
   - Use default embedding model  
   - Attach OpenAI API credentials (Hostinger)  
   - Connect input from Default Data Loader node  
   - Connect output to Pinecone Vector Store (Insert mode) node  

6. **Create Pinecone Vector Store Node (Insert mode):**
   - Type: Langchain Pinecone Vector Store  
   - Set mode to `insert`  
   - Select Pinecone index named `snapfectly`  
   - Attach Pinecone API credentials  
   - Connect input from Download file, Embeddings OpenAI, and Default Data Loader nodes as per data flow  

7. **Create When chat message received Node:**
   - Type: Langchain Chat Trigger  
   - Enable webhook with unique ID  
   - Connect output to AI Agent node  

8. **Create AI Agent Node:**
   - Type: Langchain Agent  
   - Enter system message specifying role and response rules (see overview for exact text)  
   - Connect AI memory to Simple Memory node  
   - Connect AI language model to OpenAI Chat Model node  
   - Connect AI tool to Vector Store (Retrieve mode) node  

9. **Create Simple Memory Node:**
   - Type: Langchain Memory Buffer Window  
   - Use default buffer window size  
   - Connect output to AI Agent node’s memory input  

10. **Create OpenAI Chat Model Node:**
    - Type: Langchain LM Chat OpenAI  
    - Model: `gpt-4.1-mini`  
    - Attach OpenAI API credentials (Hostinger)  
    - Connect output to AI Agent node’s language model input  

11. **Create Embeddings OpenAI Node (for queries):**
    - Type: Langchain Embeddings OpenAI  
    - Use default embedding model  
    - Attach OpenAI API credentials (Hostinger)  
    - Connect input from When chat message received node  
    - Connect output to Vector Store (Retrieve mode) node  

12. **Create Vector Store Node (Retrieve mode):**
    - Type: Langchain Pinecone Vector Store  
    - Set mode to `retrieve-as-tool`  
    - Set `topK` to 10  
    - Enable reranker usage  
    - Select Pinecone index `snapfectly`  
    - Attach Pinecone API credentials  
    - Provide tool description for agent use  
    - Connect input from Embeddings OpenAI1 and Reranker Cohere node  
    - Connect output to AI Agent node’s tool input  

13. **Create Reranker Cohere Node:**
    - Type: Langchain Reranker Cohere  
    - Use default options  
    - Attach Cohere API credentials  
    - Connect input from Vector Store (Retrieve mode) node  
    - Connect output back to Vector Store node (reranker interface)  

14. **Add Sticky Notes for clarity:**
    - Add a sticky note titled "## Insert documents into pinecone" covering nodes related to document ingestion  
    - Add a sticky note titled "## Agent" covering nodes related to chat message processing and AI response  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                   | Context or Link                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| The system message in the AI Agent node enforces strict adherence to retrieved data, avoiding hallucination, and encourages transparency about data sources.                                    | Critical for maintaining factual answers                      |
| Pinecone index name `snapfectly` is central to both ingestion and retrieval; ensure it is created and accessible in your Pinecone account.                                                    | Pinecone documentation: https://docs.pinecone.io/             |
| Cohere reranker improves retrieval relevance; requires proper API key and quota management.                                                                                                   | Cohere API docs: https://docs.cohere.ai/reference/rerank      |
| The workflow uses OpenAI GPT-4.1-mini, a specific GPT-4 variant; verify availability and credentials for this model.                                                                           | OpenAI API specs: https://platform.openai.com/docs/models     |
| Google Drive folder must have appropriate permissions to allow polling and file download by OAuth2 credentials.                                                                                | Google Drive API docs: https://developers.google.com/drive/api |
| Webhook ID in chat trigger must be exposed and configured correctly in the integration platform used to receive chat messages.                                                                 | n8n Webhook docs: https://docs.n8n.io/nodes/n8n-nodes-base.webhook/ |
| The Recursive Character Text Splitter node is used to split large documents properly, which is essential to avoid token limit issues during embedding and retrieval.                            | Langchain text splitting concepts: https://python.langchain.com/en/latest/modules/data_connection/text_splitters.html |

---

**Disclaimer:** The text provided is extracted exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.