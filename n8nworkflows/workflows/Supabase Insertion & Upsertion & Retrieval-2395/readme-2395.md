Supabase Insertion & Upsertion & Retrieval

https://n8nworkflows.xyz/workflows/supabase-insertion---upsertion---retrieval-2395


# Supabase Insertion & Upsertion & Retrieval

### 1. Workflow Overview

This workflow demonstrates a complete cycle of embedding, storing, updating, retrieving, and querying documents using Supabase as a vector database integrated with AI models (OpenAI embeddings and chat models). It is designed to showcase how documents can be ingested from external sources (e.g., Google Drive), embedded with consistent AI models, upserted into Supabase with vector search capabilities, and queried via chat interface using a retrieval-augmented generation approach.

Logical blocks in the workflow:

- **1.1 Document Ingestion and Embedding**: Downloading and loading documents, splitting text, embedding content, and inserting into Supabase vector table.
- **1.2 Upserting Documents**: Preparing new or updated document content, embedding it, and updating existing records in Supabase.
- **1.3 Retrieval and Chat Query**: Handling chat input, embedding queries, retrieving relevant documents from Supabase, generating AI chat responses, and formatting output.
- **1.4 Supporting Notes and Metadata**: Sticky notes provide instructions on setup, database preparation, and deletion strategies.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion and Embedding

**Overview:**  
This block downloads a document from Google Drive, loads its content, splits it into manageable text chunks, generates embeddings using OpenAI, and inserts these embeddings along with metadata into Supabase.

**Nodes Involved:**  
- Google Drive  
- Default Data Loader  
- Recursive Character Text Splitter1  
- Embeddings OpenAI Insertion  
- Insert Documents  
- Sticky Note (Insert guidance)

**Node Details:**  

- **Google Drive**  
  - *Type:* File download node for Google Drive  
  - *Role:* Downloads a document from a provided Google Drive URL (fileId extracted from URL)  
  - *Config:* Operation set to "download," fileId configured for target file  
  - *Input/output:* No input; outputs binary file to Default Data Loader  
  - *Failures:* Auth errors, file not found, or permission issues  
  - *Notes:* The fileId is set via URL parsing mode  

- **Default Data Loader**  
  - *Type:* Document loader (LangChain integration)  
  - *Role:* Loads the downloaded file (epub format by default) into text data for processing  
  - *Config:* Loader type set to "epubLoader," input data is binary from Google Drive  
  - *Input/output:* Input binary file; outputs document text chunks  
  - *Failures:* Unsupported file format, corrupted files  

- **Recursive Character Text Splitter1**  
  - *Type:* Text splitter for chunking documents  
  - *Role:* Splits loaded text into chunks suitable for embedding  
  - *Config:* Default options for recursive character splitting  
  - *Input/output:* Input document text; outputs array of text chunks  

- **Embeddings OpenAI Insertion**  
  - *Type:* Embeddings node using OpenAI  
  - *Role:* Generates vector embeddings for the text chunks  
  - *Config:* Model "text-embedding-3-small" selected for consistency with Supabase vector dimension (1536)  
  - *Input/output:* Input text chunks; outputs embeddings  

- **Insert Documents**  
  - *Type:* Supabase vector store insertion node (LangChain integration)  
  - *Role:* Inserts new documents with embeddings into Supabase table "Kadampa"  
  - *Config:* Mode "insert," table name set to "Kadampa," no additional options  
  - *Input/output:* Input embeddings and document metadata; output confirms insertion  
  - *Failures:* DB connection, schema mismatch, embedding size mismatch  
  - *Notes:* Requires pgvector extension and proper table setup (see Sticky Note2)  

- **Sticky Note (INSERTING)**  
  - *Content:* Emphasizes the importance of using the same embedding model across insertion, upsertion, and retrieval to maintain vector dimension consistency.

---

#### 2.2 Upserting Documents

**Overview:**  
This block prepares new or updated document content, generates embeddings, and updates existing records in the Supabase vector table using an upsert operation.

**Nodes Involved:**  
- Placeholder (File/Content to Upsert)  
- Embeddings OpenAI Upserting  
- Update Documents  
- Sticky Note1 (Upsserting)  
- Sticky Note2 (Preparation in Supabase)

**Node Details:**  

- **Placeholder (File/Content to Upsert)**  
  - *Type:* Set node for generating or holding document content JSON  
  - *Role:* Provides raw JSON data including timestamp fields for upserting  
  - *Config:* JSON output mode with dynamic date/time using n8n expressions  
  - *Input/output:* No input; outputs JSON object for update  

- **Embeddings OpenAI Upserting**  
  - *Type:* OpenAI embeddings node  
  - *Role:* Generates embeddings for the upsert JSON content  
  - *Config:* Model "text-embedding-3-small" for dimension consistency  
  - *Input/output:* Inputs JSON content; outputs embeddings for update  

- **Update Documents**  
  - *Type:* Supabase vector store upsert node (LangChain integration)  
  - *Role:* Updates existing records or inserts new documents based on ID and embedding match function  
  - *Config:* Mode "update," queryName "match_documents" (custom function), table "n8n"  
  - *Input/output:* Inputs embeddings and metadata; outputs confirmation  
  - *Failures:* Supabase permission issues, query function errors, embedding mismatch  
  - *Notes:* Table "n8n" must have pgvector and custom function setup (Sticky Note2)  

- **Sticky Note1 (UPSERTING)**  
  - *Content:* Highlights the upserting process, implying the importance of keeping embedding models consistent and using the custom match_documents function in Supabase.

- **Sticky Note2 (PREPARATION in Supabase)**  
  - *Content:* Provides detailed instructions on preparing Supabase: enable pgvector extension, create necessary columns, set policies, and create the match_documents SQL function to enable vector similarity queries.

---

#### 2.3 Retrieval and Chat Query

**Overview:**  
This block handles incoming chat messages, embeds queries, retrieves relevant documents from Supabase based on vector similarity, uses an AI chat model for response generation, and formats the output for delivery.

**Nodes Involved:**  
- When chat message received  
- OpenAI Chat Model  
- Question and Answer Chain  
- Vector Store Retriever  
- Retrieve by Query  
- Embeddings OpenAI Retrieval  
- Customize Response  
- Retrieve Rows from Table (supporting node)  
- Sticky Note3 (Retrieval)  
- Sticky Note4 (Deletion guidance)

**Node Details:**  

- **When chat message received**  
  - *Type:* LangChain chat trigger webhook  
  - *Role:* Entry point for chat messages, triggering workflow on message receipt  
  - *Config:* Public webhook, initial greeting message set, response mode "lastNode"  
  - *Input/output:* Receives chat input; triggers downstream nodes  

- **Embeddings OpenAI Retrieval**  
  - *Type:* OpenAI embeddings node  
  - *Role:* Embeds the incoming chat query for vector search  
  - *Config:* Default options, model consistent with insertion/upsertion  
  - *Input/output:* Input chat text; output vector embedding  

- **Retrieve by Query**  
  - *Type:* LangChain Supabase vector store retrieval node  
  - *Role:* Queries Supabase using custom function "match_documents" for top relevant documents  
  - *Config:* Table "Kadampa," queryName "match_documents," topK set via retriever  
  - *Input/output:* Input query embedding; output matched documents  

- **Vector Store Retriever**  
  - *Type:* LangChain retriever node  
  - *Role:* Receives retrieved documents and passes them to QA chain  
  - *Config:* topK=10 (max documents to consider)  
  - *Input/output:* Input retrieved docs; outputs context for QA  

- **Question and Answer Chain**  
  - *Type:* LangChain retrieval-based QA chain  
  - *Role:* Uses documents + chat input to generate an AI answer via OpenAI Chat Model  
  - *Config:* No custom params, uses chat model as LM  
  - *Input/output:* Inputs retriever context + chat prompt; outputs AI response  

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI chat model node  
  - *Role:* Language model generating conversational answers  
  - *Config:* Default model, options empty  
  - *Input/output:* Input prompt; outputs text answer  

- **Customize Response**  
  - *Type:* Set node  
  - *Role:* Extracts and structures the AI response text for final output  
  - *Config:* Assigns variable "output" with AI response text from chain  
  - *Input/output:* Input raw AI chain output; outputs formatted text  

- **Retrieve Rows from Table**  
  - *Type:* Standard Supabase node  
  - *Role:* Retrieves all rows from "n8n" table, potentially for deletion or reference  
  - *Config:* Operation "getAll," returnAll true, tableId "n8n"  
  - *Input/output:* Outputs full table rows  

- **Sticky Note3 (RETRIEVAL)**  
  - *Content:* Marks the retrieval block section in the workflow for clarity.

- **Sticky Note4 (DELETION)**  
  - *Content:* Explains that n8n lacks built-in vector deletion support for Supabase; recommends using an HTTP Request node with proper API keys and headers to perform DELETE operations securely. Includes API endpoint format and permission notes. References official n8n docs for HTTP Request node usage.

---

### 3. Summary Table

| Node Name                | Node Type                                        | Functional Role                     | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                                                   |
|--------------------------|-------------------------------------------------|-----------------------------------|---------------------------|--------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Google Drive             | n8n-nodes-base.googleDrive                       | Downloads document from Google Drive | None                      | Default Data Loader       |                                                                                                                              |
| Default Data Loader      | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Loads downloaded document content  | Google Drive              | Insert Documents          |                                                                                                                              |
| Recursive Character Text Splitter1 | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Splits document text into chunks   | None                      | Default Data Loader       |                                                                                                                              |
| Embeddings OpenAI Insertion | @n8n/n8n-nodes-langchain.embeddingsOpenAi      | Embeds text chunks for insertion   | Default Data Loader        | Insert Documents          |                                                                                                                              |
| Insert Documents         | @n8n/n8n-nodes-langchain.vectorStoreSupabase    | Inserts embeddings into Supabase vector table | Google Drive, Embeddings OpenAI Insertion | None                     | # INSERTING: Use same embedding model consistently                                                                         |
| Placeholder (File/Content to Upsert) | n8n-nodes-base.set                           | Provides JSON content for upserting | None                      | Update Documents          |                                                                                                                              |
| Embeddings OpenAI Upserting | @n8n/n8n-nodes-langchain.embeddingsOpenAi      | Embeds content for upserting       | Placeholder (File/Content to Upsert) | Update Documents          |                                                                                                                              |
| Update Documents         | @n8n/n8n-nodes-langchain.vectorStoreSupabase    | Updates existing records in Supabase | Placeholder (File/Content to Upsert), Embeddings OpenAI Upserting | None                     | # UPSERTING: Maintain embedding model; use match_documents function                                                          |
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger            | Starts workflow on chat message    | None                      | Question and Answer Chain |                                                                                                                              |
| Embeddings OpenAI Retrieval | @n8n/n8n-nodes-langchain.embeddingsOpenAi      | Embeds chat query                  | When chat message received | Retrieve by Query         |                                                                                                                              |
| Retrieve by Query        | @n8n/n8n-nodes-langchain.vectorStoreSupabase    | Retrieves relevant docs from Supabase | Embeddings OpenAI Retrieval | Vector Store Retriever    |                                                                                                                              |
| Vector Store Retriever   | @n8n/n8n-nodes-langchain.retrieverVectorStore   | Retrieves topK documents for QA   | Retrieve by Query          | Question and Answer Chain |                                                                                                                              |
| Question and Answer Chain | @n8n/n8n-nodes-langchain.chainRetrievalQa        | Generates AI answer using docs    | When chat message received, Vector Store Retriever | Customize Response        |                                                                                                                              |
| OpenAI Chat Model        | @n8n/n8n-nodes-langchain.lmChatOpenAi            | Language model for chat response  | Question and Answer Chain  | Question and Answer Chain |                                                                                                                              |
| Customize Response       | n8n-nodes-base.set                               | Formats AI response output        | Question and Answer Chain  | None                     |                                                                                                                              |
| Retrieve Rows from Table | n8n-nodes-base.supabase                          | Retrieves all table rows (support) | None                      | None                     | # DELETION: Use HTTP Request node for deleting records in Supabase vector DB; see detailed instructions                        |
| Sticky Note              | n8n-nodes-base.stickyNote                        | Instructional notes                | None                      | None                     | # INSERTING: Use consistent embedding model                                                                                   |
| Sticky Note1             | n8n-nodes-base.stickyNote                        | Instructional notes on upserting  | None                      | None                     | # UPSERTING                                                                                                                   |
| Sticky Note2             | n8n-nodes-base.stickyNote                        | Supabase preparation instructions | None                      | None                     | # PREPARATION (Supabase setup, pgvector extension, custom SQL function)                                                      |
| Sticky Note3             | n8n-nodes-base.stickyNote                        | Marks retrieval section           | None                      | None                     | # RETRIEVAL                                                                                                                  |
| Sticky Note4             | n8n-nodes-base.stickyNote                        | Deletion instructions             | None                      | None                     | # DELETION: Use HTTP Request node for Supabase deletions with API key and JWT authorization                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Node**  
   - Type: Google Drive (n8n built-in)  
   - Operation: Download  
   - FileId: Set to the target Google Drive file URL (use "url" mode to parse)  
   - Connect output to Default Data Loader  

2. **Create Default Data Loader Node**  
   - Type: LangChain documentDefaultDataLoader  
   - Loader: epubLoader (or suitable for your file type)  
   - Input: Binary from Google Drive node  
   - Connect output to Recursive Character Text Splitter  

3. **Create Recursive Character Text Splitter Node**  
   - Type: LangChain textSplitterRecursiveCharacterTextSplitter  
   - Options: Default  
   - Input: Document text from Default Data Loader  
   - Connect output to Embeddings OpenAI Insertion  

4. **Create Embeddings OpenAI Insertion Node**  
   - Type: LangChain embeddingsOpenAi  
   - Model: text-embedding-3-small (must match Supabase vector dimension)  
   - Input: Text chunks from text splitter  
   - Connect output to Insert Documents  

5. **Create Insert Documents Node**  
   - Type: LangChain vectorStoreSupabase  
   - Mode: Insert  
   - TableName: Select or enter your Supabase table name (e.g., "Kadampa")  
   - Credentials: Set Supabase credentials with API key and URL  
   - Input: Embeddings from previous node  
   - No further output needed  

6. **Create Placeholder (File/Content to Upsert) Node**  
   - Type: Set node  
   - Mode: Raw  
   - JSON Output: Include fields like Date and Time with dynamic expressions (e.g., `{{$now.format('dd MMM yyyy')}}`)  
   - Connect output to Embeddings OpenAI Upserting  

7. **Create Embeddings OpenAI Upserting Node**  
   - Type: LangChain embeddingsOpenAi  
   - Model: text-embedding-3-small (same as above)  
   - Input: JSON content from Placeholder  
   - Connect output to Update Documents  

8. **Create Update Documents Node**  
   - Type: LangChain vectorStoreSupabase  
   - Mode: Update  
   - TableName: Set to your Supabase table (e.g., "n8n")  
   - QueryName: "match_documents" (custom SQL function)  
   - Credentials: Use same Supabase credentials  
   - Input: Embeddings from Embeddings OpenAI Upserting and content from Placeholder  

9. **Create When Chat Message Received Node**  
   - Type: LangChain chatTrigger  
   - Webhook: Set public to true, configure webhook settings  
   - Initial Messages: Provide a greeting prompt  
   - Connect output to Question and Answer Chain  

10. **Create Embeddings OpenAI Retrieval Node**  
    - Type: LangChain embeddingsOpenAi  
    - Model: text-embedding-3-small (consistent)  
    - Input: Chat message from trigger  
    - Connect output to Retrieve by Query  

11. **Create Retrieve by Query Node**  
    - Type: LangChain vectorStoreSupabase  
    - TableName: "Kadampa" (or your table)  
    - QueryName: "match_documents"  
    - Credentials: Supabase credentials  
    - Input: Embeddings from Embeddings OpenAI Retrieval  
    - Connect output to Vector Store Retriever  

12. **Create Vector Store Retriever Node**  
    - Type: LangChain retrieverVectorStore  
    - topK: 10 (default or adjust as needed)  
    - Input: Retrieved documents from Retrieve by Query  
    - Connect output to Question and Answer Chain  

13. **Create Question and Answer Chain Node**  
    - Type: LangChain chainRetrievalQa  
    - Input: Chat trigger and retriever outputs  
    - Connect AI language model input to OpenAI Chat Model  
    - Connect output to Customize Response  

14. **Create OpenAI Chat Model Node**  
    - Type: LangChain lmChatOpenAi  
    - Options: Default or set as needed  
    - Input: QA chain prompts  
    - Connect output to Question and Answer Chain  

15. **Create Customize Response Node**  
    - Type: Set node  
    - Assignments: Set variable "output" as `{{$json["response"]["text"]}}` to format response text  
    - Input: From Question and Answer Chain  
    - Output: Final response for chat  

16. **Create Retrieve Rows from Table Node** (Optional, for deletion support)  
    - Type: Supabase node  
    - Operation: getAll  
    - TableId: "n8n" (or your table)  
    - Use for retrieving IDs for deletion logic if needed  

17. **Setup Sticky Notes** for guidance and instructions as per workflow content.

18. **Credentials Setup:**  
    - Supabase: Set API URL, Key, and enable pgvector extension in your database.  
    - OpenAI: Set API key and select embedding and chat models compatible with vector dimensions.  
    - Google Drive: OAuth2 credentials configured for file access.  

19. **Supabase Preparation:**  
    - Ensure pgvector extension enabled.  
    - Create table with vector(1536), metadata JSONB, and content TEXT fields.  
    - Create custom SQL function `match_documents` as per Sticky Note2 instructions.  
    - Set appropriate RLS policies for insert, update, and read operations.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                               |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Use the same embedding model consistently across insertion, upsertion, and retrieval to maintain vector dimension compatibility.                                                                                                                                                                                                                                  | Sticky Note on INSERTING and UPSERTING                                                                       |
| Supabase database must have `pgvector` extension enabled. Table schema must include `embedding VECTOR(1536)`, `metadata JSONB`, and `content TEXT` columns.                                                                                                                                                                                                       | Sticky Note2 (PREPARATION in Supabase)                                                                        |
| Create a custom SQL function `match_documents` in Supabase for vector similarity queries using the `<=>` operator.                                                                                                                                                                                                                                                | Sticky Note2 (PREPARATION in Supabase)                                                                        |
| n8n does not have a native Supabase deletion node for vector records. Use the HTTP Request node to send authorized DELETE requests to Supabase REST endpoints with proper headers and query filters.                                                                                                                                                               | Sticky Note4 (DELETION)                                                                                        |
| For any questions or feedback, contact [ria@n8n.io](mailto:ria@n8n.io).                                                                                                                                                                                                                                                                                             | Workflow Description                                                                                           |
| Supabase API endpoint format for deletion: `https://<your-supabase-ref>.supabase.co/rest/v1/<your-vector-table>` with headers including `apikey` and `Authorization: Bearer <JWT>`.                                                                                                                                                                                   | Sticky Note4 (DELETION)                                                                                        |
| Official n8n documentation on the HTTP Request node for API calls: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/                                                                                                                                                                                                                | Sticky Note4 (DELETION)                                                                                        |
| Initial chatbot greeting message: "Hi there! 🙏 You can ask me anything about Venerable Geshe Kelsang Gyatso's Book - 'How To Transform Your Life'. What would you like to know?"                                                                                                                                                                                      | When chat message received node configuration                                                                |

---

This documentation provides a detailed, structured explanation of the Supabase Insertion, Upsertion, and Retrieval workflow, enabling reproduction, modification, and error anticipation for advanced users and AI agents alike.