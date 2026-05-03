Create Email Knowledge Base with Gmail, OpenAI, and Qdrant Vector Database

https://n8nworkflows.xyz/workflows/create-email-knowledge-base-with-gmail--openai--and-qdrant-vector-database-6169


# Create Email Knowledge Base with Gmail, OpenAI, and Qdrant Vector Database

---

### 1. Workflow Overview

This workflow creates an Email Knowledge Base by ingesting Gmail emails, embedding their content with OpenAI, storing embeddings in a Qdrant vector database, and enabling Retrieval-Augmented Generation (RAG) chat interactions over the email data.

**Target Use Cases:**  
- Automatically capture incoming Gmail emails and archive their semantic embeddings for later retrieval.  
- Provide a conversational AI agent that answers user queries based on the email knowledge base using vector search and OpenAI chat models.  
- Support batch processing of historical emails to build the initial embedding store.

**Logical Blocks:**

- **1.1 Input Reception & Email Fetching**: Watches Gmail for new emails or manually triggers to fetch many emails.  
- **1.2 Data Extraction & Transformation**: Cleans and structures email data, including splitting text for embedding.  
- **1.3 Embedding Creation & Vector Store Insertion**: Generates OpenAI embeddings of emails and inserts them into Qdrant collections.  
- **1.4 RAG Chat Agent**: Handles user chat input, queries the vector store, and responds using OpenAI's chat model with memory support.  
- **1.5 Batch Processing**: Processes emails in batches for efficient vector insertion.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Email Fetching

**Overview:**  
This block initiates workflow execution by either manual trigger or Gmail trigger for new emails, then fetches full email details.

**Nodes Involved:**  
- Gmail Trigger1  
- Get Mail Data1  
- When clicking ‘Test workflow’  
- Get many messages  

**Node Details:**

- **Gmail Trigger1**  
  - Type: `gmailTrigger`  
  - Role: Watches Gmail inbox for new emails every minute.  
  - Config: No filters, polling mode every minute.  
  - Credentials: Gmail OAuth2 (linked to a specific Gmail account).  
  - Outputs message IDs of new emails.

- **Get Mail Data1**  
  - Type: `gmail` (operation: get)  
  - Role: Fetches full email data for each message ID from trigger.  
  - Config: Retrieves entire email object (not simple).  
  - Input: Message ID from Gmail Trigger1.  
  - Output: Full email JSON including headers, body, subject, etc.  
  - Credentials: Same Gmail OAuth2.  
  - Failure modes: API rate limits, auth errors.

- **When clicking ‘Test workflow’**  
  - Type: `manualTrigger`  
  - Role: Allows manual execution for testing or batch fetching.  
  - Output: Triggers next node.

- **Get many messages**  
  - Type: `gmail` (operation: getAll)  
  - Role: Retrieves all emails from Gmail account for batch processing.  
  - Config: No filters, returns all messages.  
  - Credentials: Gmail OAuth2.  
  - Input: Triggered by manual trigger node.  
  - Possible issues: Large mailbox may cause timeouts or API quota issues.

---

#### 2.2 Data Extraction & Transformation

**Overview:**  
Processes raw email JSON data to extract and clean relevant fields, then formats them for embedding and ingestion into the vector store.

**Nodes Involved:**  
- Code  
- Character Text Splitter  
- Character Text Splitter1  
- Enhanced Default Data Loader3  
- Default Data Loader  

**Node Details:**

- **Code (JS extraction script)**  
  - Type: `code` (JavaScript)  
  - Role: Extracts email fields (to, from, fromName, date, subject, cleaned body, emailId) from raw Gmail data.  
  - Key logic: Removes line breaks, collapses spaces in body text for cleaner embedding input.  
  - Input: Array of emails JSON.  
  - Output: Structured simplified email objects.  
  - Edge cases: Missing or malformed fields handled with fallback empty strings.

- **Character Text Splitter**  
  - Type: `textSplitterCharacterTextSplitter`  
  - Role: Splits email text by the string "Email details:" to fragment large data for document loaders.  
  - Input: Raw email data string with template including headers and body.  
  - Output: Array of text chunks.  
  - Use: Helps manage token limits in embeddings.

- **Character Text Splitter1**  
  - Same as above, applied on differently formatted email details.

- **Enhanced Default Data Loader3**  
  - Type: `documentDefaultDataLoader`  
  - Role: Converts split text chunks into document objects with metadata for vector store ingestion.  
  - Config: Adds metadata fields `data_source=gmail` and `created_at` (email date).  
  - Input: Text chunks from Character Text Splitter.  
  - Output: Document objects ready for embedding.

- **Default Data Loader**  
  - Similar to above, used after Character Text Splitter1 on the cleaned email data from code node.  
  - Adds same metadata fields.

---

#### 2.3 Embedding Creation & Vector Store Insertion

**Overview:**  
Generates OpenAI embeddings from processed email documents and inserts them into Qdrant vector database collections.

**Nodes Involved:**  
- Embeddings OpenAI3  
- Embeddings OpenAI5  
- Embeddings OpenAI11  
- Qdrant Vector Store  
- Qdrant Vector Store1  
- Qdrant Email Vector Store  

**Node Details:**

- **Embeddings OpenAI3 / 5 / 11**  
  - Type: OpenAI embeddings node (LangChain integration)  
  - Role: Generates vector embeddings from document text.  
  - Config: Uses OpenAI API with linked credentials.  
  - Input: Document objects from data loaders.  
  - Output: Embeddings for vector insertion.  
  - Potential issues: API rate limits, embedding size limits.

- **Qdrant Vector Store**  
  - Type: Vector store node for Qdrant  
  - Role: Inserts embeddings into `emails_history` collection or retrieves vectors.  
  - Modes: Insert or retrieve-as-tool depending on context.  
  - Input: Embeddings and documents.  
  - Output: Confirmation or retrieved vectors.  
  - Config: Collection name fixed to `emails_history`.  
  - Failure modes: Connectivity issues, collection misconfigurations.

- **Qdrant Vector Store1**  
  - Similar role as above, used in batch email insertion flow.

- **Qdrant Email Vector Store**  
  - Used in retrieval mode as a tool within RAG Agent to perform vector search queries with time-sensitive instructions in tool description.

---

#### 2.4 RAG Chat Agent

**Overview:**  
Handles chat messages from users, performs vector search via Qdrant, manages conversational memory, and generates AI responses using OpenAI chat model.

**Nodes Involved:**  
- When chat message received  
- RAG Agent  
- OpenAI Chat Model  
- Simple Memory  
- Qdrant Email Vector Store (as retrieval tool)  

**Node Details:**

- **When chat message received**  
  - Type: `chatTrigger` (LangChain)  
  - Role: Listens for incoming chat messages via webhook.  
  - Input: User chat query.  
  - Output: Triggers RAG Agent.  
  - Edge cases: Webhook availability, concurrency.

- **RAG Agent**  
  - Type: LangChain agent node  
  - Role: Coordinates retrieval-augmented generation by querying vector store and calling chat model.  
  - Config: System message instructs to include current date for date-sensitive queries and to add date fields in vector queries.  
  - Inputs: chat message, memory, retrieval tool (Qdrant Email Vector Store), chat model.  
  - Outputs: AI-generated chat responses.  
  - Failure modes: Timeout on API calls, vector store query failures.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI chat model node  
  - Role: Generates conversational text responses based on input prompts and context.  
  - Credentials: OpenAI API credentials linked.  
  - Input: Prompts from RAG Agent with retrieved documents.  
  - Output: Chat response text.

- **Simple Memory**  
  - Type: LangChain memory buffer window  
  - Role: Maintains conversational context window for multi-turn dialogue.  
  - Input/Output: Passes conversation history to/from RAG Agent.

- **Qdrant Email Vector Store (as retrieval tool)**  
  - Provides semantic search over email embeddings with instructions for time-specific queries to improve relevance.

---

#### 2.5 Batch Processing

**Overview:**  
Processes large email datasets in batches to efficiently embed and store emails without overloading API or database.

**Nodes Involved:**  
- batch emails  
- Qdrant Vector Store1  

**Node Details:**

- **batch emails**  
  - Type: `splitInBatches`  
  - Role: Splits incoming email arrays into batches of 50 for sequential processing.  
  - Input: List of email documents or embeddings.  
  - Output: Batches of emails to be processed.  
  - Edge cases: Batch size too large causing timeouts, batch size too small causing inefficiency.

- **Qdrant Vector Store1**  
  - Receives batch embeddings for insertion into vector store.  
  - Loops back to batch emails for next batch processing.

---

### 3. Summary Table

| Node Name                 | Node Type                                              | Functional Role                        | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                          |
|---------------------------|--------------------------------------------------------|-------------------------------------|--------------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| Gmail Trigger1            | gmailTrigger                                           | Watches new Gmail emails             |                          | Get Mail Data1           | # Watch Trigger (Email) - New Email\n## Get new Email -> Extract the text -> Add to Vector Store  |
| Get Mail Data1            | gmail (get)                                           | Fetches full email content           | Gmail Trigger1           | Qdrant Vector Store       |                                                                                                    |
| When clicking ‘Test workflow’ | manualTrigger                                       | Manual trigger for batch email fetch |                          | Get many messages         |                                                                                                    |
| Get many messages         | gmail (getAll)                                        | Fetches all emails for batch process | When clicking ‘Test workflow’ | Code                    | # Get All Emails -> Store embedding in vector db                                                  |
| Code                      | code                                                  | Extracts and cleans email data       | Get many messages         | batch emails             |                                                                                                    |
| batch emails              | splitInBatches                                        | Splits emails into manageable batches| Code, Qdrant Vector Store1 | Qdrant Vector Store1      |                                                                                                    |
| Character Text Splitter   | textSplitterCharacterTextSplitter                      | Splits email text for embedding      |                          | Enhanced Default Data Loader3 |                                                                                                    |
| Character Text Splitter1  | textSplitterCharacterTextSplitter                      | Similar text splitter on cleaned data|                          | Default Data Loader       |                                                                                                    |
| Enhanced Default Data Loader3 | documentDefaultDataLoader                           | Creates document objects with metadata| Character Text Splitter   | Qdrant Vector Store       |                                                                                                    |
| Default Data Loader       | documentDefaultDataLoader                              | Creates document objects with metadata| Character Text Splitter1  | Qdrant Vector Store1      |                                                                                                    |
| Embeddings OpenAI3        | embeddingsOpenAi                                      | Generates embeddings for emails      | Enhanced Default Data Loader3 | Qdrant Email Vector Store |                                                                                                    |
| Embeddings OpenAI5        | embeddingsOpenAi                                      | Generates embeddings                  |                           | Qdrant Vector Store       |                                                                                                    |
| Embeddings OpenAI11       | embeddingsOpenAi                                      | Generates embeddings                  |                           | Qdrant Vector Store1      |                                                                                                    |
| Qdrant Vector Store       | vectorStoreQdrant                                    | Inserts/retrieves embeddings         | Get Mail Data1, Enhanced Default Data Loader3, Embeddings OpenAI5 | batch emails, RAG Agent, Qdrant Email Vector Store |                                                                                                    |
| Qdrant Vector Store1      | vectorStoreQdrant                                    | Inserts embeddings from batches      | batch emails, Default Data Loader, Embeddings OpenAI11 | batch emails                 |                                                                                                    |
| Qdrant Email Vector Store | vectorStoreQdrant                                    | Retrieval tool for vector search     | Embeddings OpenAI3       | RAG Agent                |                                                                                                    |
| When chat message received | chatTrigger                                           | Receives user chat messages          |                          | RAG Agent                |                                                                                                    |
| RAG Agent                 | LangChain agent                                      | Coordinates retrieval & chat response| When chat message received, Simple Memory, OpenAI Chat Model, Qdrant Email Vector Store | OpenAI Chat Model        | # RAG AI Agent                                                                                    |
| OpenAI Chat Model         | lmChatOpenAi                                         | Generates AI chat responses          | RAG Agent                |                          |                                                                                                    |
| Simple Memory             | memoryBufferWindow                                   | Maintains conversational context     |                          | RAG Agent                |                                                                                                    |
| Sticky Note               | stickyNote                                           | Documentation / comments             |                          |                          | # RAG AI Agent                                                                                    |
| Sticky Note4              | stickyNote                                           | Documentation / comments             |                          |                          | # Watch Trigger (Email) - New Email\n## Get new Email -> Extract the text -> Add to Vector Store  |
| Sticky Note11             | stickyNote                                           | Documentation / comments             |                          |                          | # Get All Emails -> Store embedding in vector db                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: `gmailTrigger`  
   - Purpose: Watch for new emails every minute.  
   - Configuration: Set poll mode to every minute, no filters.  
   - Credentials: Connect to your Gmail account using OAuth2.

2. **Create Get Mail Data Node**  
   - Type: `gmail` (operation: get)  
   - Purpose: Retrieve full email details by message ID.  
   - Input: Connect to Gmail Trigger output (message ID).  
   - Credentials: Use same Gmail OAuth2.  
   - Set operation to 'get' and messageId to `={{ $json.id }}`.

3. **Create Manual Trigger Node**  
   - Type: `manualTrigger`  
   - Purpose: Manual start for batch email fetching.

4. **Create Get Many Messages Node**  
   - Type: `gmail` (operation: getAll)  
   - Purpose: Fetch all emails for batch processing.  
   - Input: Connect to manual trigger node.  
   - Credentials: Gmail OAuth2.  
   - Set returnAll=true, no filters.

5. **Create Code Node**  
   - Type: `code` (JavaScript)  
   - Purpose: Extract key fields (to, from, subject, cleaned body) from raw email data.  
   - Input: Connect to Get Many Messages output.  
   - JavaScript: Use provided code to clean and structure email data.

6. **Create Split In Batches Node**  
   - Type: `splitInBatches`  
   - Purpose: Process emails in batches of 50.  
   - Input: Connect from Code node output.  
   - Batch size: 50.

7. **Create Character Text Splitter Node**  
   - Type: `textSplitterCharacterTextSplitter`  
   - Purpose: Split email text by "Email details:" for embedding.  
   - Input: Connect from email text source (depending on flow).

8. **Create Enhanced Default Data Loader Node**  
   - Type: `documentDefaultDataLoader`  
   - Purpose: Convert split texts into documents with metadata (source=gmail, created_at=date).  
   - Input: Connect from Character Text Splitter.  
   - Configure metadata fields accordingly.

9. **Create Embeddings OpenAI Node**  
   - Type: `embeddingsOpenAi`  
   - Purpose: Generate embeddings using OpenAI API.  
   - Input: Connect from Data Loader node.  
   - Credentials: Link your OpenAI API credentials.

10. **Create Qdrant Vector Store Node**  
    - Type: `vectorStoreQdrant`  
    - Purpose: Insert embeddings into Qdrant collection `emails_history`.  
    - Input: Connect from Embeddings OpenAI output.  
    - Collection: Set to `emails_history`.  
    - Configure connection to your Qdrant instance.

11. **Create Qdrant Vector Store Node (Retrieve Mode)**  
    - Type: `vectorStoreQdrant`  
    - Purpose: Retrieve embeddings for RAG agent queries.  
    - Mode: Set to `retrieve-as-tool`.  
    - Tool name and description: Provide instructions to include date filters in queries.  
    - Input: Connect to RAG Agent.

12. **Create When Chat Message Received Node**  
    - Type: `chatTrigger`  
    - Purpose: Webhook to receive user chat messages.  
    - Configure webhook ID.  
    - Output: Connect to RAG Agent.

13. **Create Simple Memory Node**  
    - Type: `memoryBufferWindow`  
    - Purpose: Maintain conversation context.  
    - Connect ai_memory input/output with RAG Agent.

14. **Create OpenAI Chat Model Node**  
    - Type: `lmChatOpenAi`  
    - Purpose: Generate AI chat responses.  
    - Credentials: OpenAI API.  
    - Connect ai_languageModel with RAG Agent.

15. **Create RAG Agent Node**  
    - Type: `agent` (LangChain)  
    - Purpose: Combine chat input, memory, vector retrieval, and OpenAI response.  
    - System Message: Configure instructions including usage of current date and vector search best practices.  
    - Inputs: Connect chat trigger, memory, chat model, and Qdrant retrieve tool.

16. **Connect Batch Processing Loop**  
    - Connect Qdrant Vector Store1 output back to batch emails node to process next batch until all emails are ingested.

17. **Add Sticky Notes**  
    - Add notes to explain workflow sections such as RAG AI Agent and Watch Trigger (Email).

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| # RAG AI Agent: Responsible for orchestrating retrieval-augmented conversational AI over email data using vector search. | Sticky note covering RAG Agent nodes and OpenAI Chat Model.                                       |
| # Watch Trigger (Email) - New Email: Watches for new emails, extracts their content, and adds them to vector store.         | Sticky note associated with Gmail Trigger1 and Get Mail Data1 nodes.                              |
| # Get All Emails -> Store embedding in vector db: Batch processing of emails to build vector database.                      | Sticky note covering batch processing nodes including Get many messages and Code.                 |
| Qdrant collection name is fixed as `emails_history`, ensure your Qdrant instance has this collection created and configured. | Important setup note for vector store integration.                                                |
| OpenAI API credentials require appropriate access to embeddings and chat completions for this workflow to function.         | Credential setup note for OpenAI nodes.                                                           |
| Gmail OAuth2 credentials must have Gmail API access with read permissions for email fetching nodes.                         | Credential setup note for Gmail nodes.                                                             |
| For time-sensitive queries in vector search, always include date ranges explicitly in user prompts for better results.     | Specified in Qdrant Email Vector Store tool description and RAG Agent system message.             |

---

**Disclaimer:**  
The text and workflow provided are generated exclusively from an automated n8n workflow export. The processing strictly respects current content policies and contains no illegal or protected elements. All data handled is legal and publicly accessible.

---