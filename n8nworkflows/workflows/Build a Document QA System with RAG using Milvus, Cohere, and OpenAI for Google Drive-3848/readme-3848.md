Build a Document QA System with RAG using Milvus, Cohere, and OpenAI for Google Drive

https://n8nworkflows.xyz/workflows/build-a-document-qa-system-with-rag-using-milvus--cohere--and-openai-for-google-drive-3848


# Build a Document QA System with RAG using Milvus, Cohere, and OpenAI for Google Drive

### 1. Workflow Overview

This workflow implements a Retrieval Augmented Generation (RAG) AI agent that integrates Google Drive, Cohere embeddings, Milvus vector database, and OpenAI to enable intelligent question answering over user documents. It automates ingestion of PDF files from a monitored Google Drive folder, extracts and processes their text content into vector embeddings stored in Milvus, and then allows users to query these documents conversationally through a chat interface powered by OpenAI’s GPT-4o model.

**Logical Blocks:**

- **1.1 Document Ingestion and Embedding Generation:**  
  Watches a specific Google Drive folder for new PDF files, downloads them, extracts text, splits into chunks, generates embeddings via Cohere, and inserts these into Milvus.

- **1.2 RAG Chat Agent:**  
  Listens for chat messages, retrieves relevant document chunks from Milvus based on the query, manages conversation memory, and generates context-aware responses using OpenAI GPT-4o.

- **1.3 Supporting Nodes:**  
  Includes nodes for embedding generation for retrieval, memory management, and a sticky note with important contextual information.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion and Embedding Generation

**Overview:**  
This block automates monitoring a Google Drive folder for new PDFs, downloading and extracting their text, chunking the text for embedding, generating vector embeddings with Cohere, and inserting them into a Milvus vector database.

**Nodes Involved:**  
- Watch New Files  
- Download New  
- Extract from File  
- Set Chunks  
- Default Data Loader  
- Embeddings Cohere  
- Insert into Milvus

**Node Details:**

- **Watch New Files**  
  - Type: Google Drive Trigger  
  - Role: Watches a specific Google Drive folder for new file creation events (targeting PDFs).  
  - Configuration: Polls every minute, triggers only on a specified folder (folder ID provided).  
  - Inputs: None (trigger node)  
  - Outputs: File metadata including file ID.  
  - Edge Cases: Folder access permission errors, API rate limits, missing folder.  
  - Credentials: Google Drive OAuth2.

- **Download New**  
  - Type: Google Drive  
  - Role: Downloads the newly detected file using its file ID.  
  - Configuration: Operation set to “download” using the file ID from the trigger node.  
  - Inputs: File ID from “Watch New Files” node.  
  - Outputs: Binary data of the downloaded PDF file.  
  - Edge Cases: File not found, download failures, permission errors.  
  - Credentials: Google Drive OAuth2.

- **Extract from File**  
  - Type: Extract from File  
  - Role: Extracts text content from the downloaded PDF binary.  
  - Configuration: Operation set to PDF extraction.  
  - Inputs: Binary PDF file from “Download New”.  
  - Outputs: Extracted plain text content.  
  - Edge Cases: Corrupted PDF, extraction errors, unsupported PDF features.

- **Set Chunks**  
  - Type: Recursive Character Text Splitter  
  - Role: Splits extracted text into chunks of 700 characters with 60-character overlap to optimize embedding quality.  
  - Configuration: Chunk size 700, overlap 60, default splitting options.  
  - Inputs: Text from “Default Data Loader”.  
  - Outputs: Array of text chunks.  
  - Edge Cases: Very short documents, empty text.

- **Default Data Loader**  
  - Type: LangChain Document Default Data Loader  
  - Role: Prepares documents for embedding by applying the text splitter.  
  - Configuration: Default options (no special parameters).  
  - Inputs: Text chunks from “Set Chunks”.  
  - Outputs: Document objects ready for embedding.  
  - Edge Cases: Empty documents, malformed text.

- **Embeddings Cohere**  
  - Type: LangChain Embeddings (Cohere)  
  - Role: Generates vector embeddings for each text chunk using Cohere’s multilingual embedding model “embed-multilingual-v3.0”.  
  - Configuration: Model name set to “embed-multilingual-v3.0”.  
  - Inputs: Document chunks from “Default Data Loader”.  
  - Outputs: Vector embeddings with metadata.  
  - Edge Cases: API rate limits, invalid API key, network errors.  
  - Credentials: Cohere API key.

- **Insert into Milvus**  
  - Type: LangChain Vector Store (Milvus)  
  - Role: Inserts generated embeddings and metadata into a specified Milvus collection.  
  - Configuration: Insert mode, collection name dynamically selected, no clearing of existing collection.  
  - Inputs: Embeddings from “Embeddings Cohere” and documents from “Default Data Loader”.  
  - Outputs: Confirmation of insertion.  
  - Edge Cases: Milvus connection failures, authentication errors, collection not found.  
  - Credentials: Milvus API key.

---

#### 2.2 RAG Chat Agent

**Overview:**  
This block handles incoming chat messages, retrieves relevant document vectors from Milvus, manages conversation memory, and generates AI responses with OpenAI GPT-4o, integrating retrieved context for enhanced answers.

**Nodes Involved:**  
- When chat message received  
- RAG Agent  
- Retrieve from Milvus  
- Memory  
- OpenAI 4o  
- Cohere embeddings (for retrieval embeddings)

**Node Details:**

- **When chat message received**  
  - Type: LangChain Chat Trigger  
  - Role: Entry point for user chat messages to the RAG agent.  
  - Configuration: Default options, webhook-based trigger.  
  - Inputs: Incoming chat messages.  
  - Outputs: Chat message data.  
  - Edge Cases: Webhook connectivity issues, malformed input.

- **RAG Agent**  
  - Type: LangChain Agent  
  - Role: Orchestrates the RAG process by invoking retrieval and language model nodes.  
  - Configuration: Default options, integrates retrieval tool and language model.  
  - Inputs: Chat messages from “When chat message received”.  
  - Outputs: AI-generated chat responses.  
  - Edge Cases: Agent logic errors, missing tool outputs.

- **Retrieve from Milvus**  
  - Type: LangChain Vector Store (Milvus)  
  - Role: Retrieves top 10 most relevant document chunks from Milvus based on the user query.  
  - Configuration: Retrieval mode “retrieve-as-tool”, topK=10, tool named “vector_store” with descriptive prompt.  
  - Inputs: Query embeddings from “Cohere embeddings”.  
  - Outputs: Retrieved document chunks as context.  
  - Edge Cases: Milvus connection or query failures, empty results.  
  - Credentials: Milvus API key.

- **Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversation history window to provide context and optimize cost/response speed.  
  - Configuration: Default buffer window options.  
  - Inputs: Chat messages and AI responses from “RAG Agent”.  
  - Outputs: Updated conversation memory.  
  - Edge Cases: Memory overflow, data corruption.

- **OpenAI 4o**  
  - Type: LangChain Chat OpenAI  
  - Role: Generates AI responses using OpenAI GPT-4o model, enhanced with retrieved context.  
  - Configuration: Model set to “gpt-4o”, default options.  
  - Inputs: Contextualized prompt from “RAG Agent”.  
  - Outputs: AI-generated text responses.  
  - Edge Cases: API rate limits, invalid API key, network errors.  
  - Credentials: OpenAI API key.

- **Cohere embeddings**  
  - Type: LangChain Embeddings (Cohere)  
  - Role: Generates embeddings for user queries to enable vector search in Milvus.  
  - Configuration: Model “embed-multilingual-v3.0”.  
  - Inputs: User chat messages from “When chat message received”.  
  - Outputs: Query embeddings for retrieval.  
  - Edge Cases: API errors, invalid input.  
  - Credentials: Cohere API key.

---

#### 2.3 Supporting Nodes

**Overview:**  
Additional nodes provide documentation and notes for users.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Provides detailed contextual information about Milvus advantages, setup instructions, usage tips, and contact info.  
  - Configuration: Positioned on canvas, color-coded, sized for readability.  
  - Inputs/Outputs: None.  
  - Edge Cases: None.

---

### 3. Summary Table

| Node Name               | Node Type                                    | Functional Role                         | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                                              |
|-------------------------|----------------------------------------------|---------------------------------------|------------------------|-------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Watch New Files          | Google Drive Trigger                         | Monitors Google Drive folder for PDFs | None                   | Download New            |                                                                                                                                          |
| Download New            | Google Drive                                 | Downloads new PDF file                 | Watch New Files         | Extract from File        |                                                                                                                                          |
| Extract from File        | Extract from File                            | Extracts text from PDF                 | Download New            | Insert into Milvus       |                                                                                                                                          |
| Set Chunks              | Recursive Character Text Splitter            | Splits text into chunks for embedding | Default Data Loader     | Default Data Loader      |                                                                                                                                          |
| Default Data Loader      | LangChain Document Loader                    | Prepares documents for embedding      | Set Chunks              | Insert into Milvus       |                                                                                                                                          |
| Embeddings Cohere       | LangChain Embeddings (Cohere)                | Generates vector embeddings            | Default Data Loader     | Insert into Milvus       |                                                                                                                                          |
| Insert into Milvus      | LangChain Vector Store (Milvus)               | Inserts embeddings into Milvus         | Extract from File, Default Data Loader, Embeddings Cohere | None                    |                                                                                                                                          |
| When chat message received | LangChain Chat Trigger                      | Receives user chat messages            | None                   | RAG Agent                |                                                                                                                                          |
| RAG Agent               | LangChain Agent                              | Orchestrates retrieval and response    | When chat message received, Retrieve from Milvus, Memory, OpenAI 4o | Memory, OpenAI 4o        |                                                                                                                                          |
| Retrieve from Milvus    | LangChain Vector Store (Milvus)               | Retrieves relevant document chunks     | Cohere embeddings      | RAG Agent                |                                                                                                                                          |
| Memory                  | LangChain Memory Buffer Window                | Manages conversation history           | RAG Agent              | RAG Agent                |                                                                                                                                          |
| OpenAI 4o               | LangChain Chat OpenAI                         | Generates AI chat responses             | RAG Agent              | RAG Agent                |                                                                                                                                          |
| Cohere embeddings       | LangChain Embeddings (Cohere)                | Embeds user queries for retrieval       | When chat message received | Retrieve from Milvus     |                                                                                                                                          |
| Sticky Note             | Sticky Note                                  | Provides setup and usage info           | None                   | None                     | Why Milvus: Milvus is often more performant and scalable than Supabase. Setup requires Zilliz account. See https://zilliz.com and https://1node.ai |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node ("Watch New Files")**  
   - Type: Google Drive Trigger  
   - Configure to trigger on file creation in a specific folder (set folder ID).  
   - Poll every minute.  
   - Set credentials to Google Drive OAuth2.

2. **Create Google Drive Node ("Download New")**  
   - Type: Google Drive  
   - Operation: Download  
   - File ID: Use expression to get file ID from trigger node (`{{$json["id"]}}`).  
   - Connect input from "Watch New Files".  
   - Set credentials to Google Drive OAuth2.

3. **Create Extract from File Node ("Extract from File")**  
   - Type: Extract from File  
   - Operation: PDF extraction  
   - Input: Binary data from "Download New".  
   - Connect input from "Download New".

4. **Create Recursive Character Text Splitter Node ("Set Chunks")**  
   - Type: Text Splitter (Recursive Character)  
   - Chunk Size: 700 characters  
   - Chunk Overlap: 60 characters  
   - Connect input from "Default Data Loader" (created next).

5. **Create Default Data Loader Node ("Default Data Loader")**  
   - Type: LangChain Document Default Data Loader  
   - Use default options.  
   - Connect input from "Set Chunks".  
   - Connect output to "Insert into Milvus".

6. **Create Embeddings Node ("Embeddings Cohere")**  
   - Type: LangChain Embeddings (Cohere)  
   - Model: embed-multilingual-v3.0  
   - Connect input from "Default Data Loader".  
   - Connect output to "Insert into Milvus".  
   - Set credentials to Cohere API key.

7. **Create Milvus Vector Store Node ("Insert into Milvus")**  
   - Type: LangChain Vector Store (Milvus)  
   - Mode: Insert  
   - Collection: Select or enter your Milvus collection name.  
   - Clear Collection: False  
   - Connect inputs from "Extract from File", "Default Data Loader", and "Embeddings Cohere".  
   - Set credentials to Milvus API key.

8. **Create Chat Trigger Node ("When chat message received")**  
   - Type: LangChain Chat Trigger  
   - Use default webhook settings.  
   - This is the entry point for user queries.

9. **Create Embeddings Node for Query ("Cohere embeddings")**  
   - Type: LangChain Embeddings (Cohere)  
   - Model: embed-multilingual-v3.0  
   - Connect input from "When chat message received".  
   - Set credentials to Cohere API key.

10. **Create Milvus Vector Store Node for Retrieval ("Retrieve from Milvus")**  
    - Type: LangChain Vector Store (Milvus)  
    - Mode: Retrieve-as-tool  
    - TopK: 10  
    - Tool Name: vector_store  
    - Tool Description: "You are an AI agent that responds based on information received from a vector database."  
    - Connect input from "Cohere embeddings".  
    - Connect output to "RAG Agent".  
    - Set credentials to Milvus API key.

11. **Create Memory Node ("Memory")**  
    - Type: LangChain Memory Buffer Window  
    - Use default options.  
    - Connect input and output to "RAG Agent".

12. **Create OpenAI Chat Node ("OpenAI 4o")**  
    - Type: LangChain Chat OpenAI  
    - Model: gpt-4o  
    - Connect input from "RAG Agent".  
    - Set credentials to OpenAI API key.

13. **Create LangChain Agent Node ("RAG Agent")**  
    - Type: LangChain Agent  
    - Use default options.  
    - Connect input from "When chat message received".  
    - Connect inputs from "Retrieve from Milvus", "Memory", and "OpenAI 4o".  
    - Connect outputs to "Memory" and "OpenAI 4o".

14. **Add Sticky Note Node**  
    - Add a sticky note with setup instructions, Milvus advantages, and contact info for user reference.

15. **Activate the Workflow**  
    - Ensure all credentials are configured correctly.  
    - Test by adding a PDF to the monitored Google Drive folder and sending a chat message to the webhook.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                             | Context or Link                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Milvus is often considered more performant and scalable than Supabase for vector search, especially for large datasets and multilingual support. Zilliz provides managed Milvus cloud infrastructure, eliminating the need for self-hosted containers. | https://zilliz.com/                               |
| To calculate RAG costs for running Milvus on your own server with n8n, use Zilliz’s cost calculator.                                                                                                                                                       | https://zilliz.com/rag-cost-calculator/           |
| For implementing a RAG AI agent tailored to your company, contact 1Node AI for consulting and development services.                                                                                                                                       | https://1node.ai                                  |
| This workflow requires API keys and credentials for Google Drive, Milvus, Cohere, and OpenAI. Ensure these are set up in n8n before activation.                                                                                                         | n8n Credential Setup                              |
| The workflow currently supports PDF files only; consider extending support to other document types such as DOCX, TXT, CSV, or web pages for broader applicability.                                                                                        | Suggested Improvement                             |
| Implement error handling and notification nodes (e.g., email, Slack) to alert users on failures like download errors, extraction issues, or Milvus insertion problems for production readiness.                                                             | Suggested Improvement                             |

---

This document provides a comprehensive understanding of the workflow’s structure, logic, and configuration to enable reproduction, modification, and troubleshooting by advanced users and automation agents alike.