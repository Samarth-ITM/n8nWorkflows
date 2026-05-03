Document-Based Chatbot with Memory using OpenAI, Pinecone and Google Drive

https://n8nworkflows.xyz/workflows/document-based-chatbot-with-memory-using-openai--pinecone-and-google-drive-4930


# Document-Based Chatbot with Memory using OpenAI, Pinecone and Google Drive

### 1. Workflow Overview

This n8n workflow implements a **Document-Based Chatbot with Memory** that leverages OpenAI’s language models, Pinecone vector database, and Google Drive document storage to provide intelligent, context-aware customer support on the DGM website. The chatbot dynamically retrieves, processes, and indexes documents from Google Drive, stores conversation memories in Airtable, and uses vector search for knowledge retrieval, all orchestrated through AI agents with conversational memory.

The workflow can be logically divided into the following blocks:

- **1.1 Input Reception:** Receives user chat messages via a LangChain Chat Trigger node.
- **1.2 Memory Retrieval and Management:** Queries Airtable for stored user conversation memories and aggregates them.
- **1.3 Document Retrieval and Processing:** Fetches documents from a designated Google Drive folder, downloads, splits, embeds, and indexes them into Pinecone.
- **1.4 AI Agent Processing:** Main chatbot logic node that integrates language models, memory, and vector store tools to generate responses.
- **1.5 Vector Store Querying:** Uses Pinecone and OpenAI embeddings to retrieve relevant information in response to queries.
- **1.6 Memory Saving:** Persists updated user memories back to Airtable.
- **1.7 Manual Trigger and Testing:** Allows manual execution for document processing and testing purposes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Handles incoming chat messages from users, serving as the workflow entry point for live conversations.

- **Nodes Involved:**
  - `When chat message received`

- **Node Details:**
  - **Type:** LangChain Chat Trigger  
  - **Role:** Webhook to receive user messages publicly.
  - **Configuration:** Public webhook with default options, no authentication required.
  - **Inputs:** External chat messages from users.
  - **Outputs:** Passes messages downstream for memory retrieval.
  - **Version-specific notes:** Uses LangChain Chat Trigger 1.1 for compatibility with LangChain integration.
  - **Failure cases:** Webhook downtime, malformed input, or connectivity issues.

#### 2.2 Memory Retrieval and Management

- **Overview:**  
  Retrieves stored conversation memories for the specific user from Airtable, aggregates them, then merges with the incoming message to provide context for AI response.

- **Nodes Involved:**
  - `Get Memories`
  - `Aggregate`
  - `Merge`
  - `Simple Memory`

- **Node Details:**

  - **Get Memories**  
    - Type: Airtable  
    - Role: Fetches past user memories filtered by user "Astrid".  
    - Config: Uses Airtable credentials, sorts by Created date, searches by formula `{User} = 'Astrid'`.  
    - Input: Trigger from chat message node.  
    - Output: Array of memory records.
    - Edge cases: API limits, authentication errors, no memories found.

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Aggregates all memory records into a single field `memories` to consolidate user context.  
    - Input: Output from Get Memories.  
    - Output: Aggregated memory string for downstream use.

  - **Merge**  
    - Type: Merge  
    - Role: Combines incoming chat message and aggregated memories into a single payload for AI Agent.  
    - Input: From `When chat message received` and `Aggregate`.  
    - Output: Unified context and message.

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Implements conversational memory buffer with context window length of 25 messages to manage session context within AI Agent.  
    - Input: Connected as AI memory for AI Agent.  
    - Output: Provides memory context to AI Agent.  
    - Edge cases: Memory overflow or insufficient memory window size.

#### 2.3 Document Retrieval and Processing

- **Overview:**  
  Retrieves all documents from a specific Google Drive folder, downloads content, splits documents into chunks, embeds them with OpenAI embeddings, and inserts them into Pinecone vector index for semantic search.

- **Nodes Involved:**
  - `When clicking 'Test Workflow' button`
  - `Google Drive`
  - `Get Content`
  - `Loop Over Items`
  - `Default Data Loader`
  - `Recursive Character Text Splitter`
  - `Embeddings OpenAI`
  - `Pinecone Vector Store`

- **Node Details:**

  - **When clicking 'Test Workflow' button**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation to process documents outside of chat flow.

  - **Google Drive**  
    - Type: Google Drive  
    - Role: Lists all files from folder ID `1k0w1u5s_qQKxqLnY5hHujiikoQD7eedT` (named "AI").  
    - Credentials: Google Drive OAuth2.  
    - Output: List of document metadata.

  - **Get Content**  
    - Type: Google Drive  
    - Role: Downloads the content of each file by file ID.  
    - Input: File list from Google Drive node.  
    - Output: Binary content of each document.

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes each document one by one through the workflow for efficient handling.

  - **Default Data Loader**  
    - Type: LangChain Default Data Loader  
    - Role: Converts binary document content to text for processing.  
    - Input: Binary files from Get Content.  
    - Output: Loaded document text.

  - **Recursive Character Text Splitter**  
    - Type: LangChain Text Splitter  
    - Role: Splits loaded text into smaller chunks recursively to optimize embeddings.  
    - Output: Text chunks ready for embedding.

  - **Embeddings OpenAI**  
    - Type: LangChain OpenAI Embeddings  
    - Role: Generates vector embeddings for each text chunk using OpenAI API.  
    - Credentials: OpenAI API key.  
    - Output: Embeddings for Pinecone insertion.

  - **Pinecone Vector Store**  
    - Type: LangChain Pinecone Vector Store  
    - Role: Inserts embeddings into Pinecone index with namespace "DGM" under index "n8n".  
    - Credentials: Pinecone API key.  
    - Output: Confirmation of vector insertion.  
    - Edge cases: API quota limits, connectivity, embedding size limits.

#### 2.4 AI Agent Processing

- **Overview:**  
  The core chatbot logic node that integrates user input, memory, vector store tools, language models, and conversation management to generate intelligent, context-aware responses.

- **Nodes Involved:**
  - `AI Agent`
  - `OpenRouter Chat Model`
  - `Answer questions with a vector store`
  - `OpenAI Chat Model1`
  - `Pinecone Vector Store1`
  - `Embeddings OpenAI1`

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Central AI chatbot node using a detailed system prompt defining personality, conversation flows, memory management, and multilingual support.  
    - Inputs: Unified user message and memories (from Merge), AI memory (Simple Memory), AI tool (Answer questions with vector store).  
    - Language Model: OpenRouter Chat Model (fallback or primary LM).  
    - Output: Chatbot response.  
    - Key expressions: SystemMessage contains extensive instructions and templates with dynamic variables like `{USER_NAME}`, date/time placeholders.  
    - Edge cases: Model API failures, rate limits, unexpected user input.

  - **OpenRouter Chat Model**  
    - Type: LangChain LM Chat OpenRouter  
    - Role: Provides chat completions for AI Agent.  
    - Credentials: OpenRouter API key.

  - **Answer questions with a vector store**  
    - Type: LangChain Tool Vector Store  
    - Role: Tool used by AI Agent to query Pinecone vector store for relevant company info.  
    - Inputs: Calls Pinecone Vector Store1 and OpenAI Chat Model1 for retrieval-augmented generation.  
    - Output: Retrieved knowledge snippets for AI Agent.

  - **OpenAI Chat Model1**  
    - Type: LangChain LM Chat OpenAI GPT-4o-mini  
    - Role: Supports vector store querying with GPT-4o-mini model.  
    - Credentials: OpenAI API key.

  - **Pinecone Vector Store1**  
    - Type: LangChain Pinecone Vector Store  
    - Role: Vector store queried during Q&A with namespace "DGM".  
    - Credentials: Pinecone API key.

  - **Embeddings OpenAI1**  
    - Type: LangChain OpenAI Embeddings  
    - Role: Embeddings provider for vector store queries.

#### 2.5 Memory Saving

- **Overview:**  
  After processing the conversation and generating a response, this block saves updated user memories back to Airtable for persistent context.

- **Nodes Involved:**
  - `Save Memory`

- **Node Details:**

  - **Save Memory**  
    - Type: Airtable Tool  
    - Role: Creates a new record in Airtable table "Table 1" with fields "User" and "Memories".  
    - Input: Memory data from AI Agent’s output (`$fromAI('memory')`) and user name "Astrid".  
    - Credentials: Airtable API token.  
    - Edge cases: Airtable write failures, API quota, data format mismatches.

#### 2.6 Manual Trigger and Testing

- **Overview:**  
  Enables manual initiation of the document processing workflow for testing or updating vector store contents.

- **Nodes Involved:**
  - `When clicking 'Test Workflow' button`

- **Node Details:**

  - **When clicking 'Test Workflow' button**  
    - Type: Manual Trigger  
    - Role: Initiates document retrieval and processing chain starting with Google Drive node.  
    - Output: Starts document ingestion flow.

---

### 3. Summary Table

| Node Name                     | Node Type                                  | Functional Role                               | Input Node(s)                 | Output Node(s)                     | Sticky Note                                                                                       |
|-------------------------------|--------------------------------------------|-----------------------------------------------|------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------|
| When chat message received     | @n8n/n8n-nodes-langchain.chatTrigger       | Entry point for user chat input                | External webhook             | Get Memories                      |                                                                                                 |
| Get Memories                  | n8n-nodes-base.airtable                     | Retrieve user conversation memories            | When chat message received   | Aggregate                        |                                                                                                 |
| Aggregate                    | n8n-nodes-base.aggregate                     | Aggregate memory records                        | Get Memories                 | Merge                            |                                                                                                 |
| Merge                        | n8n-nodes-base.merge                         | Combine memories and current message           | When chat message received, Aggregate | AI Agent                      |                                                                                                 |
| Simple Memory                | @n8n/n8n-nodes-langchain.memoryBufferWindow| Manage conversation memory buffer              | (AI memory input)            | AI Agent                        |                                                                                                 |
| AI Agent                    | @n8n/n8n-nodes-langchain.agent                | Core chatbot logic with personality and tools | Merge, Simple Memory, Answer questions with a vector store | Save Memory                   | ## AI Agent<br>This is where the Chatbot is and all of the tools needed.                        |
| Save Memory                 | n8n-nodes-base.airtableTool                   | Save updated user memories                      | AI Agent (ai_tool output)    | (End)                           |                                                                                                 |
| When clicking 'Test Workflow' button | n8n-nodes-base.manualTrigger            | Manual trigger for document processing         | Manual trigger               | Google Drive                    |                                                                                                 |
| Google Drive                | n8n-nodes-base.googleDrive                     | List files in AI documents folder              | When clicking 'Test Workflow' button | Get Content                 | ## Document Processing <br>This is where the AI will retrieve, download, and process the documents (PDF, CSV,...) to be used by the AI Agent |
| Get Content                 | n8n-nodes-base.googleDrive                     | Download document content                       | Google Drive                | Loop Over Items                 |                                                                                                 |
| Loop Over Items             | n8n-nodes-base.splitInBatches                  | Iterate over each document                      | Get Content                 | Default Data Loader, Pinecone Vector Store (on alternate output) |                                                                                                 |
| Default Data Loader         | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Convert binary file to text                     | Loop Over Items             | Recursive Character Text Splitter |                                                                                                 |
| Recursive Character Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Split text into smaller chunks                  | Default Data Loader         | Embeddings OpenAI               |                                                                                                 |
| Embeddings OpenAI           | @n8n/n8n-nodes-langchain.embeddingsOpenAi     | Generate OpenAI embeddings for text chunks     | Recursive Character Text Splitter | Pinecone Vector Store          |                                                                                                 |
| Pinecone Vector Store       | @n8n/n8n-nodes-langchain.vectorStorePinecone  | Insert embeddings into Pinecone vector index   | Embeddings OpenAI, Loop Over Items (alternate output) | Loop Over Items               |                                                                                                 |
| OpenRouter Chat Model       | @n8n/n8n-nodes-langchain.lmChatOpenRouter      | Language model for AI Agent                     | AI Agent                    | AI Agent                       |                                                                                                 |
| Answer questions with a vector store | @n8n/n8n-nodes-langchain.toolVectorStore | Provides vector store retrieval as AI tool     | OpenAI Chat Model1, Pinecone Vector Store1 | AI Agent                       |                                                                                                 |
| OpenAI Chat Model1          | @n8n/n8n-nodes-langchain.lmChatOpenAi          | Language model used in vector store QA         | Pinecone Vector Store1      | Answer questions with a vector store |                                                                                                 |
| Pinecone Vector Store1      | @n8n/n8n-nodes-langchain.vectorStorePinecone   | Vector store queried for Q&A                    | Embeddings OpenAI1          | OpenAI Chat Model1              |                                                                                                 |
| Embeddings OpenAI1          | @n8n/n8n-nodes-langchain.embeddingsOpenAi      | Embeddings for vector store query               | (Implicit in vector store1) | Pinecone Vector Store1          |                                                                                                 |
| Sticky Note                 | n8n-nodes-base.stickyNote                       | Comment block                                   | N/A                        | N/A                            | ## AI Agent<br>This is where the Chatbot is and all of the tools needed.                       |
| Sticky Note1                | n8n-nodes-base.stickyNote                       | Comment block                                   | N/A                        | N/A                            | ## Document Processing <br>This is where the AI will retrieve, download, and process the documents (PDF, CSV,...) to be used by the AI Agent |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node:**  
   - Name: `When chat message received`  
   - Type: LangChain Chat Trigger (1.1)  
   - Set as public webhook for receiving chat messages.

2. **Create Airtable Node to Get Memories:**  
   - Name: `Get Memories`  
   - Type: Airtable  
   - Operation: Search  
   - Base: Your Airtable base ID for memories  
   - Table: Your memories table  
   - Filter by formula: `{User} = 'Astrid'` (or dynamically set user)  
   - Sort by Created date ascending  
   - Set Airtable credentials.

3. **Create Aggregate Node:**  
   - Name: `Aggregate`  
   - Type: Aggregate  
   - Aggregate all items, include fields: Memories, Created  
   - Destination field: `memories`

4. **Create Merge Node:**  
   - Name: `Merge`  
   - Type: Merge  
   - Mode: Combine All  
   - Connect outputs from `When chat message received` and `Aggregate`

5. **Create Simple Memory Node:**  
   - Name: `Simple Memory`  
   - Type: LangChain Memory Buffer Window  
   - Context window length: 25

6. **Create AI Agent Node:**  
   - Name: `AI Agent`  
   - Type: LangChain Agent  
   - Paste the detailed system prompt describing personality, memory, multilingual handling, conversation flow, and API integration.  
   - Connect inputs:  
     - Main input from `Merge`  
     - AI memory input from `Simple Memory`  
     - AI tool input from `Answer questions with a vector store` (created later)  
   - Connect AI language model input to `OpenRouter Chat Model`.

7. **Create OpenRouter Chat Model Node:**  
   - Name: `OpenRouter Chat Model`  
   - Type: LangChain LM Chat OpenRouter  
   - Set OpenRouter API credentials.  
   - Connect output to AI Agent’s language model input.

8. **Create Vector Store Q&A Tool Nodes:**  
   - Create `Answer questions with a vector store` node: Type LangChain Tool Vector Store.  
   - Set description: "Use this tool to retrieve data (Working hours, contacts, etc) from the files about the DGM website."
   - Connect AI language model to `OpenAI Chat Model1`.
   - Connect vector store to `Pinecone Vector Store1`.

9. **Create OpenAI Chat Model1 Node:**  
   - Name: `OpenAI Chat Model1`  
   - Type: LangChain LM Chat OpenAI  
   - Model: `gpt-4o-mini`  
   - Set OpenAI credentials.

10. **Create Pinecone Vector Store1 Node:**  
    - Name: `Pinecone Vector Store1`  
    - Type: LangChain Pinecone Vector Store  
    - Set index: `n8n`  
    - Namespace: `DGM`  
    - Set Pinecone credentials.

11. **Create Embeddings OpenAI1 Node:**  
    - Name: `Embeddings OpenAI1`  
    - Type: LangChain OpenAI Embeddings  
    - Set OpenAI credentials.

12. **Connect vector store nodes:**  
    - `Embeddings OpenAI1` to `Pinecone Vector Store1`  
    - `Pinecone Vector Store1` to `OpenAI Chat Model1`  
    - `OpenAI Chat Model1` to `Answer questions with a vector store`  
    - `Answer questions with a vector store` to AI Agent.

13. **Create Save Memory Node:**  
    - Name: `Save Memory`  
    - Type: Airtable Tool  
    - Operation: Create  
    - Base and Table: Same as `Get Memories`  
    - Map fields: User = "Astrid" (or dynamic), Memories = `{{$fromAI("memory")}}`  
    - Set Airtable credentials.  
    - Connect AI Agent ai_tool output to Save Memory.

14. **Create Document Processing Nodes:**

    - **Manual Trigger:**  
      - Name: `When clicking 'Test Workflow' button`  
      - Type: Manual Trigger.

    - **Google Drive List Files:**  
      - Name: `Google Drive`  
      - Type: Google Drive  
      - Operation: List files in folder ID `1k0w1u5s_qQKxqLnY5hHujiikoQD7eedT`  
      - Set Google Drive OAuth2 credentials.

    - **Get Content:**  
      - Name: `Get Content`  
      - Type: Google Drive  
      - Operation: Download file by ID.

    - **Loop Over Items:**  
      - Type: Split In Batches to iterate over files.

    - **Default Data Loader:**  
      - Type: LangChain Default Data Loader  
      - Data Type: Binary.

    - **Recursive Character Text Splitter:**  
      - Type: LangChain Text Splitter Recursive Character.

    - **Embeddings OpenAI:**  
      - Type: LangChain OpenAI Embeddings  
      - Set OpenAI credentials.

    - **Pinecone Vector Store:**  
      - Type: LangChain Pinecone Vector Store  
      - Index: `n8n`  
      - Namespace: `DGM`  
      - Set Pinecone credentials.

    - Connect these nodes in order: Manual Trigger → Google Drive → Get Content → Loop Over Items → Default Data Loader → Text Splitter → Embeddings → Pinecone Vector Store → Loop Over Items (to continue batch).

15. **Add Sticky Notes:**  
    - Add two sticky notes with content describing the AI Agent and Document Processing blocks for developer clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                    |
|-------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| The AI Agent system prompt defines a warm, professional customer support personality with multilingual and memory capabilities.    | Embedded in AI Agent node systemMessage parameter |
| Document Processing block handles diverse document types (PDF, CSV, etc.) from Google Drive for indexing in the vector store.      | Sticky Note1 content                               |
| Memory management uses Airtable for persistent storage with fields for user and memories, ensuring historical context retention.    | Airtable nodes configuration                       |
| Integration with Pinecone enables semantic search over company documents to provide accurate, context-aware responses.             | Pinecone vector store nodes                        |
| OpenRouter and OpenAI GPT-4o-mini models are used for language understanding and generation, balancing cost and capability.         | AI Agent and vector store LM nodes                 |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.