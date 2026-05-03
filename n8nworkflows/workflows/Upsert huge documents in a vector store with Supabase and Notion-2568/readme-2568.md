Upsert huge documents in a vector store with Supabase and Notion

https://n8nworkflows.xyz/workflows/upsert-huge-documents-in-a-vector-store-with-supabase-and-notion-2568


# Upsert huge documents in a vector store with Supabase and Notion

### 1. Workflow Overview

This workflow is designed to maintain an up-to-date Retrieval-Augmented Generation (RAG) system based on a Notion knowledge base, embedding its content into a Supabase vector store. It enables real-time question answering against living data by continuously syncing and embedding updated Notion pages.

The workflow is logically divided into two main functional blocks:

- **1.1 Data Synchronization and Embedding Upsert**  
  Periodically polls Notion for recently updated pages, retrieves their full content, removes outdated embeddings from Supabase, generates new embeddings for the updated content, and upserts these embeddings into the Supabase vector store.

- **1.2 Question Answering Chat Interface**  
  Listens for incoming user chat messages, retrieves relevant content from the Supabase vector store based on the query, and uses OpenAI GPT-4 to generate context-aware answers referencing the stored Notion knowledge base.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Synchronization and Embedding Upsert

**Overview:**  
This block ensures that changes in the Notion knowledge base are regularly detected and reflected in the vector store. It deletes outdated embeddings linked to updated pages, fetches the latest content, splits and embeds the text, and stores the embeddings with appropriate metadata in Supabase.

**Nodes Involved:**  
- Schedule Trigger  
- Get updated pages (Notion)  
- Input Reference (NoOp)  
- Loop Over Items (SplitInBatches)  
- Delete old embeddings if exist (Supabase)  
- Limit  
- Get page blocks (Notion)  
- Concatenate to single string (Summarize)  
- Token Splitter (Text Splitter - Token)  
- Default Data Loader (Document loader)  
- Embeddings OpenAI  
- Supabase Vector Store (Insert embeddings)  
- Supabase Vector Store1 (VectorStore for embeddings)  
- Limit1

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Polls every minute to initiate the workflow for updated Notion pages.  
  - Config: Interval set to every 1 minute.  
  - Inputs: None (start trigger).  
  - Outputs: Connects to "Get updated pages".  
  - Failure scenarios: n8n scheduling errors or downtime may delay updates.

- **Get updated pages**  
  - Type: Notion node (databasePage/getAll)  
  - Role: Retrieves all Notion database pages updated exactly one minute ago to capture recent changes.  
  - Config: Filter by "Last edited time" equals (current time minus 1 minute).  
  - Inputs: Trigger node output.  
  - Outputs: Passes updated pages downstream.  
  - Edge cases: No updated pages yield empty results; API rate limits or auth errors possible.

- **Input Reference**  
  - Type: NoOp (placeholder)  
  - Role: Serves as a reference node for downstream connections and expression simplification.  
  - Inputs/Outputs: Passes data unchanged.  
  - No special config.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each updated Notion page sequentially to prevent concurrency issues.  
  - Config: Default batch size (likely 1).  
  - Inputs: List of updated pages.  
  - Outputs: Each page processed individually on main output; optionally passes to delete old embeddings on secondary output.  
  - Edge cases: Large volume of updates may slow processing.

- **Delete old embeddings if exist**  
  - Type: Supabase node (delete operation)  
  - Role: Deletes existing embeddings in Supabase that correspond to the current Notion page ID to prevent duplicates.  
  - Config: Filter on `metadata->>id` equals current page ID.  
  - Inputs: From Loop Over Items secondary output.  
  - Outputs: Passes to Limit node.  
  - Failure modes: Supabase API errors, incorrect filtering resulting in no deletion or partial deletion.

- **Limit**  
  - Type: Limit  
  - Role: Limits data flow to one item at a time, preventing multiple active streams which could cause double-processing.  
  - Inputs: From Delete old embeddings node.  
  - Outputs: To "Get page blocks".  

- **Get page blocks**  
  - Type: Notion node (block/getAll)  
  - Role: Fetches all content blocks of the given Notion page, including nested blocks, to obtain full page content.  
  - Config: Block ID set to current page ID; fetch nested blocks enabled; return all blocks.  
  - Inputs: From Limit node.  
  - Outputs: Passes full page content blocks downstream.  
  - Edge cases: Large pages could result in API rate limits or timeouts.

- **Concatenate to single string**  
  - Type: Summarize node  
  - Role: Aggregates all content blocks into a single concatenated text string, separated by newlines, to prepare for embedding.  
  - Config: Summarize field "content" by concatenation with "\n" separator.  
  - Inputs: From Get page blocks.  
  - Outputs: To Supabase Vector Store.  

- **Token Splitter**  
  - Type: LangChain Text Splitter (Token)  
  - Role: Splits long text into manageable chunks (max 500 tokens) for embedding, with adjustable chunk size and optional overlap.  
  - Config: Chunk size set to 500 tokens; overlap not explicitly set (default 0).  
  - Inputs: From Default Data Loader.  
  - Outputs: To Embeddings OpenAI.  
  - Notes: Chunk size plus overlap must not exceed 8191 tokens for *text-embedding-ada-002*.

- **Default Data Loader**  
  - Type: LangChain Document Default Data Loader  
  - Role: Prepares documents for embedding with metadata, here including Notion page ID and name.  
  - Config: Metadata fields "id" and "name" set from the current input page.  
  - Inputs: From Token Splitter.  
  - Outputs: To Supabase Vector Store (embedding insertion).  

- **Embeddings OpenAI**  
  - Type: LangChain Embeddings OpenAI  
  - Role: Generates vector embeddings for each text chunk using OpenAI embedding models.  
  - Config: Uses OpenAI credentials; model unspecified defaults to *text-embedding-ada-002*.  
  - Inputs: From Default Data Loader.  
  - Outputs: To Supabase Vector Store and Supabase Vector Store1.  
  - Failure modes: API rate limits, invalid auth, or input formatting errors.

- **Supabase Vector Store**  
  - Type: LangChain Vector Store Supabase (insert mode)  
  - Role: Inserts new embedding vectors into the Supabase "documents" table along with metadata.  
  - Config: Insert mode with credentials for Supabase.  
  - Inputs: From Embeddings OpenAI and Default Data Loader.  
  - Outputs: To Limit1.  

- **Supabase Vector Store1**  
  - Type: LangChain Vector Store Supabase (retriever mode)  
  - Role: Provides vector store retrieval interface used downstream for question answering.  
  - Config: Table "documents" in Supabase.  
  - Inputs: From Embeddings OpenAI.  
  - Outputs: To Vector Store Retriever.  

- **Limit1**  
  - Type: Limit  
  - Role: Similar to Limit node, restricts processing to single item streams to avoid concurrency issues.  
  - Inputs: From Supabase Vector Store.  
  - Outputs: None downstream in this path.

---

#### 2.2 Question Answering Chat Interface

**Overview:**  
This block enables users to interact with the knowledge base via a chat interface. Incoming messages trigger a vector search in Supabase, and the retrieved context is passed to an OpenAI GPT-4 chat model to generate answers.

**Nodes Involved:**  
- When chat message received (Chat Trigger)  
- Question and Answer Chain (LangChain Retrieval QA)  
- Vector Store Retriever (LangChain Retriever)  
- OpenAI Chat Model  
- Supabase Vector Store1

**Node Details:**

- **When chat message received**  
  - Type: LangChain Chat Trigger  
  - Role: Webhook node that listens for incoming chat messages from users and triggers the QA process.  
  - Config: Public webhook enabled; webhook ID specified.  
  - Inputs: None (trigger).  
  - Outputs: To Question and Answer Chain.  
  - Edge cases: Webhook connectivity issues, malformed inputs.

- **Question and Answer Chain**  
  - Type: LangChain Chain Retrieval QA  
  - Role: Orchestrates the QA process by taking user input, retrieving relevant documents via vector search, and generating answers.  
  - Config: Default options.  
  - Inputs: From chat trigger, vector store retriever, and OpenAI chat model.  
  - Outputs: Final response to be sent back to the user.  
  - Failure modes: Retrieval failure, model errors, input inconsistencies.

- **Vector Store Retriever**  
  - Type: LangChain Retriever Vector Store  
  - Role: Executes a vector similarity search against the Supabase vector store to find relevant embedding chunks based on the query.  
  - Config: Uses Supabase Vector Store1 for retrieval.  
  - Inputs: From Supabase Vector Store1 embeddings and Question and Answer Chain.  
  - Outputs: To Question and Answer Chain.  
  - Edge cases: Empty or irrelevant results, Supabase API errors.

- **OpenAI Chat Model**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Generates chat completions using GPT-4o model, incorporating retrieved context to answer user queries.  
  - Config: Model set to "gpt-4o"; credentials specified.  
  - Inputs: From Question and Answer Chain.  
  - Outputs: To Question and Answer Chain.  
  - Failure modes: API rate limits, invalid input format.

- **Supabase Vector Store1**  
  - Role: Also used here as retriever source for vector search (see above).

---

### 3. Summary Table

| Node Name                    | Node Type                             | Functional Role                              | Input Node(s)                   | Output Node(s)                      | Sticky Note                                                                                                  |
|------------------------------|-------------------------------------|----------------------------------------------|--------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger                    | Initiates polling for updated Notion pages   | None                           | Get updated pages                  |                                                                                                              |
| Get updated pages             | Notion (databasePage/getAll)       | Retrieves pages updated in last minute       | Schedule Trigger               | Input Reference                   | This placeholder serves as a reference point so it is easier to swap the data source with a different service |
| Input Reference              | NoOp                               | Holds reference data for easier expression use | Get updated pages              | Loop Over Items                   | Select Database: Choose the database representing your Knowledge Base                                         |
| Loop Over Items              | SplitInBatches                     | Processes each updated page sequentially     | Input Reference                | Delete old embeddings if exist, (secondary output empty) | Process each page/document separately                                                                       |
| Delete old embeddings if exist | Supabase (delete)                  | Deletes old embeddings for current page      | Loop Over Items (secondary)    | Limit                            | All chunks of a previous version of the document are deleted by filtering metadata by the given ID           |
| Limit                       | Limit                              | Restricts to 1 stream to prevent double-processing | Delete old embeddings if exist | Get page blocks                 | Reduce active streams/items to just 1 to prevent double-processing                                           |
| Get page blocks             | Notion (block/getAll)               | Retrieves all content blocks of a Notion page | Limit                         | Concatenate to single string      | Retrieve all page contents/blocks                                                                             |
| Concatenate to single string | Summarize                         | Concatenates all blocks into a single string | Get page blocks               | Supabase Vector Store            | Combine all contents into a single text formatted into one line                                              |
| Default Data Loader          | LangChain Document Default Data Loader | Prepares documents with metadata for embedding | Token Splitter                | Supabase Vector Store            | Store additional metadata with each embed, especially Notion ID for later retrieval                          |
| Token Splitter               | LangChain Text Splitter (Token)    | Splits text into chunks for embedding        | Default Data Loader           | Embeddings OpenAI                | Adjust chunk size and overlap for accurate search results; chunk size + overlap ≤ 8191 for text-embedding-ada-002 model |
| Embeddings OpenAI            | LangChain Embeddings OpenAI        | Generates embeddings for text chunks          | Token Splitter                | Supabase Vector Store, Supabase Vector Store1 | Model used for both creating and reading embeddings                                                         |
| Supabase Vector Store        | LangChain Vector Store Supabase (insert) | Inserts embeddings into Supabase vector store | Embeddings OpenAI, Default Data Loader | Limit1                       | Embed item and store in Vector Store; content split into multiple chunks if needed                            |
| Limit1                      | Limit                              | Restricts to 1 stream to prevent double-processing | Supabase Vector Store        | None                            | Reduce active streams/items to just 1 to prevent double-processing                                           |
| Supabase Vector Store1       | LangChain Vector Store Supabase (retriever) | Provides vector retrieval capability          | Embeddings OpenAI             | Vector Store Retriever           |                                                                                                              |
| Vector Store Retriever       | LangChain Retriever Vector Store   | Retrieves relevant embeddings for queries     | Supabase Vector Store1        | Question and Answer Chain        |                                                                                                              |
| When chat message received   | LangChain Chat Trigger             | Listens for incoming chat messages            | None                         | Question and Answer Chain        | Simple chat bot to ask specific questions while accessing Notion Knowledge Base context                      |
| Question and Answer Chain    | LangChain Chain Retrieval QA       | Performs retrieval-augmented question answering | When chat message received, Vector Store Retriever, OpenAI Chat Model | None                  |                                                                                                              |
| OpenAI Chat Model            | LangChain LM Chat OpenAI           | Generates context-aware answers using GPT-4o | Question and Answer Chain     | Question and Answer Chain        |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set it to run every 1 minute.

2. **Add a Notion node ("Get updated pages")**  
   - Operation: databasePage/getAll  
   - Filter by "Last edited time" equals current time minus 1 minute (`{{$now.minus(1, 'minutes').toISO()}}`).  
   - Select your Notion database ID representing your knowledge base.  
   - Connect Schedule Trigger output to this node.

3. **Add a NoOp node ("Input Reference")**  
   - Connect "Get updated pages" output to this node. This serves as a stable reference point for expressions.

4. **Add a SplitInBatches node ("Loop Over Items")**  
   - Default batch size (process pages one by one).  
   - Connect "Input Reference" output to this node.

5. **Add a Supabase node ("Delete old embeddings if exist")**  
   - Operation: Delete  
   - Table: "documents" (or your vector store table)  
   - Filter: `metadata->>id=eq.{{$json["id"]}}` (delete all embeddings with metadata.id matching current page ID)  
   - Connect secondary output of "Loop Over Items" to this node.

6. **Add a Limit node ("Limit")**  
   - Default limit 1.  
   - Connect "Delete old embeddings if exist" output to this node.

7. **Add a Notion node ("Get page blocks")**  
   - Operation: block/getAll  
   - Block ID: `{{$json["id"]}}` (current page ID)  
   - Fetch nested blocks enabled, return all blocks.  
   - Connect "Limit" output to this node.

8. **Add a Summarize node ("Concatenate to single string")**  
   - Field to summarize: "content"  
   - Aggregation: Concatenate with "\n" separator  
   - Connect "Get page blocks" output to this node.

9. **Add a LangChain Document Default Data Loader node ("Default Data Loader")**  
   - Add metadata fields:  
     - id = `{{$json["id"]}}`  
     - name = `{{$json["name"]}}`  
   - Connect "Concatenate to single string" output to this node.

10. **Add a LangChain Text Splitter Token Splitter node ("Token Splitter")**  
    - Chunk size: 500 tokens (adjust as needed)  
    - Connect "Default Data Loader" output to this node.

11. **Add a LangChain Embeddings OpenAI node ("Embeddings OpenAI")**  
    - Credentials: Connect your OpenAI API credentials.  
    - Connect "Token Splitter" output to this node.

12. **Add a LangChain Vector Store Supabase node ("Supabase Vector Store")**  
    - Mode: Insert  
    - Table: "documents" (or your vector store table)  
    - Credentials: Supabase API credentials  
    - Connect "Embeddings OpenAI" output to this node.  
    - Also connect "Default Data Loader" output as input to this node (for metadata).

13. **Add a Limit node ("Limit1")**  
    - Limit 1  
    - Connect "Supabase Vector Store" output to this node.

14. **Add a LangChain Vector Store Supabase node ("Supabase Vector Store1")**  
    - Mode: Retriever (default retrieval mode)  
    - Table: "documents"  
    - Credentials: Supabase API credentials  
    - Connect "Embeddings OpenAI" output to this node.

15. **Create the Chat Interface:**  
    - Add LangChain Chat Trigger node ("When chat message received")  
      - Enable public webhook, note the webhook URL for chat client integration.

16. **Add LangChain Retriever Vector Store node ("Vector Store Retriever")**  
    - Connect "Supabase Vector Store1" output to this node.  
    - Connect this node output to the QA chain.

17. **Add LangChain LM Chat OpenAI node ("OpenAI Chat Model")**  
    - Model: "gpt-4o" (or preferred GPT-4 variant)  
    - Credentials: OpenAI API credentials.

18. **Add LangChain Chain Retrieval QA node ("Question and Answer Chain")**  
    - Connect from "When chat message received", "Vector Store Retriever", and "OpenAI Chat Model" nodes as inputs.  
    - Final output will be the chat response.

19. **Connect the QA chain output as needed for downstream usage (e.g., sending response to user).**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                           | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow adds the capability to build a RAG on living data by syncing Notion pages with a Supabase vector store for up-to-date embeddings.                                         | Purpose of workflow                                                                                       |
| Demo video: [Watch on YouTube](https://youtu.be/ELAxebGmspY)                                                                                                                           | Workflow demo                                                                                            |
| For more accurate polling on n8n cloud plans, switch from Schedule Trigger to native Notion Trigger node to save executions.                                                           | Sticky note near Schedule Trigger / Notion Trigger node                                                  |
| To setup a new Vector Store in Supabase, refer to [Supabase Vector Columns Guide](https://supabase.com/docs/guides/ai/vector-columns)                                                  | Prerequisite for Supabase vector store setup                                                            |
| Adjust chunk size and overlap in the Token Splitter node for optimal embedding quality; chunk size + overlap must not exceed 8191 tokens for *text-embedding-ada-002* model.             | Sticky note near Token Splitter node                                                                     |
| Metadata including Notion Page ID is stored with each embedding to allow efficient deletion and retrieval of all chunks belonging to a single Notion page.                              | Sticky note near Supabase Vector Store nodes                                                             |
| Manual polling approach chosen over webhook triggers for higher accuracy and to avoid missing updates due to Notion caching or event delays.                                           | Sticky note near Schedule Trigger and Get updated pages                                                  |
| The workflow can be adapted to other vector stores like PGVector, Pinecone, or Qdrant by replacing Supabase nodes with appropriate HTTP request nodes.                                 | Workflow description                                                                                      |
| Chunking large documents into smaller pieces enables better embedding and retrieval performance.                                                                                        | Sticky note near Token Splitter and Default Data Loader                                                  |
| The chat interface uses GPT-4o model for generating answers, but any compatible OpenAI chat model can be configured.                                                                    | Sticky note near OpenAI Chat Model                                                                       |

---

This detailed documentation enables advanced users and AI agents to fully understand, reproduce, and modify the workflow, as well as anticipate potential errors related to API limits, authentication, and data consistency.