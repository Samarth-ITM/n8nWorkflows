Telegram AI Chatbot with Document-Based Answers using OpenAI and PGVector RAG

https://n8nworkflows.xyz/workflows/telegram-ai-chatbot-with-document-based-answers-using-openai-and-pgvector-rag-4799


# Telegram AI Chatbot with Document-Based Answers using OpenAI and PGVector RAG

### 1. Workflow Overview

This workflow implements a **Telegram AI Chatbot** that provides document-based answers by integrating **OpenAI language models** with a **PostgreSQL PGVector vector store** for retrieval-augmented generation (RAG). It supports dynamic ingestion of documents via Google Drive triggers, processes document content into vector embeddings, stores them in PGVector, and uses this indexed knowledge to answer user queries contextually.

The workflow is logically divided into the following blocks:

- **1.1 Document Ingestion and Processing:** Triggered by Google Drive file events, downloads files, extracts text, splits text into chunks, generates embeddings, and stores them in PGVector.
- **1.2 Vector Store Management:** Manages the connection between document embeddings and PGVector, including deleting old records and updating with new data.
- **1.3 Query Handling and AI Agent:** Receives chat messages (Telegram or other triggers), queries PGVector for relevant context, and uses OpenAI chat models to generate contextual answers.
- **1.4 Auxiliary Components:** Includes memory management for chat conversations, utility nodes for batching, setting variables, and sub-workflow execution.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion and Processing

- **Overview:**  
  This block handles file creation or updates on Google Drive, downloads the file, extracts text content, splits it recursively into manageable chunks, creates contextual embeddings, and prepares data for vector storage.

- **Nodes Involved:**  
  File Created, File Updated, Loop Over Items, Set File ID, Delete Old Doc Records, Download File, Extract Document Text, Create Chunks From Doc, Chunks To List, Recursive Character Text Splitter, Default Data Loader, Embeddings OpenAI1

- **Node Details:**

  - **File Created (Google Drive Trigger)**  
    - Type: Trigger node listening for new files in Google Drive.  
    - Configuration: Default trigger on file creation events.  
    - Inputs: None (trigger).  
    - Outputs: Emits new file metadata.  
    - Edge Cases: Permissions errors, latency in Drive events, missing files.  

  - **File Updated (Google Drive Trigger)**  
    - Type: Trigger node listening for file updates in Google Drive.  
    - Configuration: Default trigger on file update events.  
    - Inputs: None (trigger).  
    - Outputs: Emits updated file metadata.  
    - Edge Cases: Similar to File Created, plus handling partial updates.  

  - **Loop Over Items (SplitInBatches)**  
    - Type: Batch processing node to iterate over multiple files.  
    - Configuration: Default batch size (usually 1 or configurable).  
    - Inputs: From triggers.  
    - Outputs: Processes one file at a time downstream.  
    - Edge Cases: Large batch sizes may cause timeouts or memory issues.  

  - **Set File ID (Set)**  
    - Type: Utility node to store the current file ID for downstream operations.  
    - Configuration: Sets variable(s) based on incoming data (e.g., file ID).  
    - Inputs: Batch file data.  
    - Outputs: Passes data on with added context.  
    - Edge Cases: Missing or incorrect file IDs may cause errors later.  

  - **Delete Old Doc Records (Postgres)**  
    - Type: SQL execution node for PGVector database.  
    - Configuration: Runs SQL command to delete existing document records matching the file ID to avoid duplicates.  
    - Inputs: File ID from Set node.  
    - Outputs: Confirmation of deletion; triggers next download.  
    - Edge Cases: Connection errors, SQL errors, transaction deadlocks.  

  - **Download File (Google Drive)**  
    - Type: File download node.  
    - Configuration: Downloads the file content using file ID.  
    - Inputs: File ID.  
    - Outputs: File binary data for extraction.  
    - Edge Cases: File not found, permission denied, large file size causing timeout.  

  - **Extract Document Text (Extract From File)**  
    - Type: Content extraction node.  
    - Configuration: Extracts text from the binary file (supports multiple formats like PDF, DOCX, etc.).  
    - Inputs: File binary from Download File.  
    - Outputs: Extracted raw text.  
    - Edge Cases: Unsupported file formats, extraction failures, encoding issues.  

  - **Create Chunks From Doc (Code)**  
    - Type: Function (Code) node.  
    - Configuration: Custom JavaScript code to prepare text data for splitting.  
    - Inputs: Extracted text.  
    - Outputs: Text formatted for splitting.  
    - Edge Cases: Empty text, malformed input.  

  - **Chunks To List (SplitOut)**  
    - Type: Node to convert split chunks into list items.  
    - Configuration: Standard splitting.  
    - Inputs: Output from code node.  
    - Outputs: List of text chunks.  
    - Edge Cases: Empty chunk list.  

  - **Recursive Character Text Splitter (LangChain Text Splitter)**  
    - Type: Recursive text splitter to divide large texts into smaller chunks recursively by characters.  
    - Configuration: Uses default parameters optimized for context window sizes.  
    - Inputs: Raw or chunked text.  
    - Outputs: Smaller text chunks suitable for embedding.  
    - Edge Cases: Very large documents, incomplete splits.  

  - **Default Data Loader (LangChain Document Loader)**  
    - Type: Prepares documents for vector storage.  
    - Configuration: Uses default settings to convert chunks into document objects.  
    - Inputs: Text chunks.  
    - Outputs: Document objects for embeddings.  
    - Edge Cases: Empty documents, malformed document structure.  

  - **Embeddings OpenAI1 (LangChain Embeddings OpenAI)**  
    - Type: Embeddings generation using OpenAI API.  
    - Configuration: Uses OpenAI embedding model (likely text-embedding-ada-002).  
    - Inputs: Document objects.  
    - Outputs: Vector embeddings.  
    - Edge Cases: API rate limits, auth errors, text too long.

---

#### 2.2 Vector Store Management

- **Overview:**  
  This block manages the PostgreSQL PGVector vector store by inserting new embeddings and associating them with documents. It facilitates efficient retrieval during query time.

- **Nodes Involved:**  
  Postgres PGVector Store, Get Values

- **Node Details:**

  - **Postgres PGVector Store (LangChain Vector Store PGVector)**  
    - Type: Vector store integration node for PostgreSQL with PGVector extension.  
    - Configuration: Inserts or updates vector embeddings and associated metadata.  
    - Inputs: Embeddings and document data from Embeddings OpenAI1 and Default Data Loader.  
    - Outputs: Confirmation and metadata of inserted vectors.  
    - Edge Cases: Database connection failures, vector dimension mismatches, SQL errors.  

  - **Get Values (Set)**  
    - Type: Utility node that extracts or sets values before vector storage.  
    - Configuration: Sets key-value pairs or cleans data.  
    - Inputs: Output from Generate Contextual Text or embedding nodes.  
    - Outputs: Cleaned data for Postgres PGVector Store.  
    - Edge Cases: Missing expected values causing downstream errors.

---

#### 2.3 Query Handling and AI Agent

- **Overview:**  
  This block receives chat messages, either via webhook or internal execution, uses the RAG AI Agent with the PGVector store and OpenAI chat models to generate accurate, context-aware answers.

- **Nodes Involved:**  
  When chat message received, When Executed by Another Workflow, TestData, RAG AI Agent, Docs RAG Tool, Embeddings, OpenAI Chat Model, OpenAI Chat Model3, Chat Memory

- **Node Details:**

  - **When chat message received (LangChain Chat Trigger)**  
    - Type: Webhook trigger for chat messages (Telegram or other chatbot platform).  
    - Configuration: Disabled by default, can be enabled for live listening.  
    - Inputs: Incoming chat messages.  
    - Outputs: Chat message data.  
    - Edge Cases: Webhook URL misconfiguration, message parsing errors.  

  - **When Executed by Another Workflow (Execute Workflow Trigger)**  
    - Type: Trigger node to start this workflow when called from another workflow.  
    - Configuration: Default.  
    - Inputs: Execution requests.  
    - Outputs: Starts the TestData node.  
    - Edge Cases: Missing call parameters.  

  - **TestData (Set - Disabled)**  
    - Type: Utility node with preset data for testing without triggers.  
    - Configuration: Disabled by default, can be enabled for test runs.  
    - Inputs: Manual or triggered by Execute Workflow node.  
    - Outputs: Test data passed to RAG AI Agent.  
    - Edge Cases: None when disabled.  

  - **RAG AI Agent (LangChain Agent)**  
    - Type: Core AI agent node combining language model, vector store, and tools for RAG.  
    - Configuration: Uses OpenAI Chat Model as language model, Docs RAG Tool as retrieval tool, and Chat Memory for session context.  
    - Inputs: Chat messages, tools, embeddings, and memory.  
    - Outputs: AI-generated chat responses.  
    - Edge Cases: API failures, memory load errors, timeouts, no relevant context found.  

  - **Docs RAG Tool (LangChain Vector Store PGVector)**  
    - Type: Retrieval tool for RAG, querying PGVector database.  
    - Configuration: Configured to query PGVector store with chat queries to retrieve relevant document vectors.  
    - Inputs: Query from RAG AI Agent.  
    - Outputs: Retrieved documents or chunks as context.  
    - Edge Cases: Empty results, database connectivity issues.  

  - **Embeddings (LangChain Embeddings OpenAI)**  
    - Type: Embeddings generator used by RAG Tool for query embedding.  
    - Configuration: Uses OpenAI embeddings, separate from document ingestion embeddings.  
    - Inputs: Query text.  
    - Outputs: Query embedding vector.  
    - Edge Cases: API errors, rate limits.  

  - **OpenAI Chat Model (LangChain lmChatOpenAi)**  
    - Type: OpenAI chat completion model for generating responses.  
    - Configuration: Used by RAG AI Agent for final answer generation.  
    - Inputs: Prompt plus context from vector store.  
    - Outputs: Chatbot reply.  
    - Edge Cases: API limits, malformed prompt.  

  - **OpenAI Chat Model3 (LangChain lmChatOpenAi)**  
    - Type: Auxiliary chat model node used for generating contextual text during chunk processing.  
    - Configuration: Similar to main chat model but used in a different part of the pipeline.  
    - Inputs: Text chunks.  
    - Outputs: Generated contextual text.  
    - Edge Cases: Similar to main chat model.  

  - **Chat Memory (LangChain Memory Postgres Chat)**  
    - Type: Persistent chat memory stored in PostgreSQL.  
    - Configuration: Stores conversation history for context retention.  
    - Inputs: Chat messages and AI responses.  
    - Outputs: Chat history context for RAG AI Agent.  
    - Edge Cases: DB connectivity, memory size limits.

---

#### 2.4 Auxiliary Components

- **Overview:**  
  Supporting nodes for batching, setting variables, and sticky notes for documentation.

- **Nodes Involved:**  
  Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note5

- **Node Details:**  
  - Sticky Note nodes are for visual documentation and contain no functional logic. No parameters other than content text. Positioned to annotate respective workflow areas.

---

### 3. Summary Table

| Node Name                  | Node Type                               | Functional Role                   | Input Node(s)             | Output Node(s)             | Sticky Note                      |
|----------------------------|---------------------------------------|---------------------------------|---------------------------|----------------------------|---------------------------------|
| File Created               | Google Drive Trigger                   | Trigger on new file creation    | -                         | Loop Over Items             |                                 |
| File Updated               | Google Drive Trigger                   | Trigger on file update          | -                         | Loop Over Items             |                                 |
| Loop Over Items            | SplitInBatches                        | Process files one by one        | File Created, File Updated | Set File ID, (empty branch) |                                 |
| Set File ID                | Set                                   | Store current file ID           | Loop Over Items            | Delete Old Doc Records      |                                 |
| Delete Old Doc Records     | Postgres                             | Delete old document records     | Set File ID                | Download File              |                                 |
| Download File             | Google Drive                         | Download file content           | Delete Old Doc Records     | Extract Document Text       |                                 |
| Extract Document Text      | Extract From File                     | Extract text from file          | Download File              | Create Chunks From Doc      |                                 |
| Create Chunks From Doc     | Code                                 | Prepare text chunks             | Extract Document Text      | Chunks To List              |                                 |
| Chunks To List             | SplitOut                             | Convert chunks to list items    | Create Chunks From Doc     | Generate Contextual Text    |                                 |
| Recursive Character Text Splitter | LangChain Text Splitter         | Split text recursively          | Default Data Loader        | Default Data Loader         |                                 |
| Default Data Loader        | LangChain Document Loader             | Prepare documents for embeddings | Recursive Character Text Splitter | Postgres PGVector Store  |                                 |
| Embeddings OpenAI1         | LangChain Embeddings OpenAI           | Generate embeddings             | Default Data Loader        | Postgres PGVector Store     |                                 |
| Postgres PGVector Store    | LangChain Vector Store PGVector       | Store embeddings in PGVector    | Embeddings OpenAI1, Default Data Loader | Loop Over Items      |                                 |
| Get Values                 | Set                                   | Prepare values for vector store | Generate Contextual Text   | Postgres PGVector Store     |                                 |
| Generate Contextual Text   | LangChain ChainLlm                    | Generate contextual text        | Chunks To List             | Get Values                 |                                 |
| OpenAI Chat Model3         | LangChain lmChatOpenAi                 | Generate contextual text with AI | Generate Contextual Text | Generate Contextual Text    |                                 |
| When chat message received | LangChain Chat Trigger                 | Trigger for incoming chat msgs  | -                         | (none)                     | Disabled by default              |
| When Executed by Another Workflow | Execute Workflow Trigger           | Trigger workflow from another   | -                         | TestData                   |                                 |
| TestData                  | Set (disabled)                       | Test data without trigger       | When Executed by Another Workflow | RAG AI Agent             | Disabled by default              |
| RAG AI Agent               | LangChain Agent                       | AI agent for RAG chatbot        | TestData, Chat Memory, Docs RAG Tool, OpenAI Chat Model | -                   |                                 |
| Docs RAG Tool              | LangChain Vector Store PGVector       | Retrieval tool for RAG          | Embeddings                | RAG AI Agent               |                                 |
| Embeddings                 | LangChain Embeddings OpenAI           | Generate embeddings for queries | RAG AI Agent              | Docs RAG Tool              |                                 |
| OpenAI Chat Model          | LangChain lmChatOpenAi                 | Main chat model                 | RAG AI Agent              | RAG AI Agent               |                                 |
| Chat Memory                | LangChain Memory Postgres Chat         | Store chat history              | RAG AI Agent              | RAG AI Agent               |                                 |
| Sticky Note1               | Sticky Note                          | Documentation                   | -                         | -                          |                                 |
| Sticky Note2               | Sticky Note                          | Documentation                   | -                         | -                          |                                 |
| Sticky Note3               | Sticky Note                          | Documentation                   | -                         | -                          |                                 |
| Sticky Note5               | Sticky Note                          | Documentation                   | -                         | -                          |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive File Triggers**  
   - Create two Google Drive Trigger nodes:  
     - One for “File Created” events.  
     - One for “File Updated” events.  
   - Connect both to a SplitInBatches node "Loop Over Items" to process files individually.

2. **Loop Over Items**  
   - Add a SplitInBatches node named "Loop Over Items" connected from both triggers.

3. **Set File ID**  
   - Add a Set node named "Set File ID" to store the current file’s ID from the batch item.

4. **Delete Old Document Records in PGVector**  
   - Add a Postgres node "Delete Old Doc Records" configured with SQL to delete existing vectors for the file ID.  
   - Connect "Set File ID" output to this node’s input.

5. **Download File**  
   - Add a Google Drive node "Download File" configured to download the file by ID from "Delete Old Doc Records".  
   - Connect "Delete Old Doc Records" output to "Download File".

6. **Extract Text from File**  
   - Add an "Extract From File" node "Extract Document Text" to extract raw text from the downloaded file binary.  
   - Connect "Download File" output to "Extract Document Text".

7. **Create Chunks from Extracted Text**  
   - Add a Code node "Create Chunks From Doc" with JavaScript code that processes the extracted text into chunks.  
   - Connect "Extract Document Text" to "Create Chunks From Doc".

8. **Convert Chunks to List Items**  
   - Add a SplitOut node "Chunks To List" to convert chunked text into a list.  
   - Connect "Create Chunks From Doc" to "Chunks To List".

9. **Generate Contextual Text for Each Chunk**  
   - Add a LangChain ChainLlm node "Generate Contextual Text" to process each chunk with AI.  
   - Connect "Chunks To List" output to "Generate Contextual Text".

10. **Generate Embeddings for Contextual Text**  
    - Add a Set node "Get Values" if needed to prepare data.  
    - Connect "Generate Contextual Text" to "Get Values".

11. **Store Embeddings in PGVector**  
    - Add a LangChain Vector Store PGVector node "Postgres PGVector Store".  
    - Connect "Get Values" output and "Default Data Loader" output as inputs for document and embedding data.  
    - Configure credentials to connect to PostgreSQL with PGVector extension.

12. **Add Recursive Text Splitter**  
    - Add LangChain Recursive Character Text Splitter node connected to the "Default Data Loader" to split text recursively.

13. **Set up Default Data Loader**  
    - Add LangChain Default Data Loader to prepare documents for embedding ingestion.  
    - Connect output of Recursive Character Text Splitter to Default Data Loader.

14. **Add Embeddings Generation (Documents)**  
    - Add LangChain Embeddings OpenAI node "Embeddings OpenAI1" to generate embeddings for documents.  
    - Connect Default Data Loader output to "Embeddings OpenAI1".

15. **Connect Embeddings to PGVector Store**  
    - Connect "Embeddings OpenAI1" output to "Postgres PGVector Store" input.

16. **Configure RAG AI Agent for Queries**  
    - Add LangChain Agent node "RAG AI Agent".  
    - Add LangChain Vector Store PGVector node "Docs RAG Tool".  
    - Add LangChain Embeddings OpenAI node "Embeddings" for query embeddings.  
    - Add LangChain lmChatOpenAi node "OpenAI Chat Model" for chat completion.  
    - Add LangChain Memory Postgres Chat node "Chat Memory" for conversation history.

17. **Connect Query Embeddings and Vector Store to RAG Agent**  
    - Connect "Embeddings" to "Docs RAG Tool" input.  
    - Connect "Docs RAG Tool", "OpenAI Chat Model", and "Chat Memory" to "RAG AI Agent" inputs appropriately.

18. **Set up Chat Message Trigger (Optional)**  
    - Add LangChain Chat Trigger node "When chat message received" (enable as needed).  
    - Connect it to "RAG AI Agent" or to an Execute Workflow Trigger node if using sub-workflows.

19. **Add Execute Workflow Trigger and Test Data Node**  
    - Include Execute Workflow Trigger node "When Executed by Another Workflow" connected to a Set node "TestData" for manual testing.  
    - Connect "TestData" output to "RAG AI Agent".

20. **Configure Credentials**  
    - Set up Google Drive OAuth2 credentials for file triggers and downloads.  
    - Configure OpenAI API credentials for embeddings and chat models.  
    - Configure PostgreSQL credentials for PGVector vector store and chat memory.

21. **Add Sticky Notes**  
    - Add Sticky Note nodes near functional blocks to document logic or instructions for maintenance.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow integrates LangChain nodes with OpenAI and PGVector to enable a powerful document-based chatbot. | n8n official LangChain integration documentation: https://docs.n8n.io/integrations/builtin/nodes/langchain/ |
| PostgreSQL with PGVector extension is required for vector store operations.                                | PGVector Project: https://pgvector.org/                                                                  |
| Google Drive credentials must have permission to access the target folders and files.                      | Google Drive API docs: https://developers.google.com/drive/api/v3/about-sdk                               |
| OpenAI API key with permission for chat and embeddings endpoints is required.                              | OpenAI API docs: https://platform.openai.com/docs/api-reference                                           |
| Chat memory persistence ensures contextual conversations but requires database management.                  | https://docs.n8n.io/integrations/builtin/nodes/langchain/#memory                                         |

---

This document fully describes the Telegram AI Chatbot workflow with document-based answers using OpenAI and PGVector RAG, enabling both reproduction and advanced modifications.