Document Q&A System with Voyage-Context-3 Embeddings and MongoDB Atlas

https://n8nworkflows.xyz/workflows/document-q-a-system-with-voyage-context-3-embeddings-and-mongodb-atlas-6819


# Document Q&A System with Voyage-Context-3 Embeddings and MongoDB Atlas

### 1. Workflow Overview

This workflow implements a comprehensive Document Q&A System using Voyage-Context-3 contextual embeddings and MongoDB Atlas as a vector store. It ingests a research paper (PDF) from a specified URL, processes and chunks the document pages, generates contextual embeddings with the Voyage-Context-3 model, and stores these embeddings alongside metadata in MongoDB Atlas. The workflow further supports a Retrieval-Augmented Generation (RAG) Q&A agent, allowing users to query the document interactively via chat, including clarifying questions to refine search context.

The workflow is logically divided into two major parts:

**1.1 Document Ingestion and Embedding Generation**  
- Define document URL and clear existing MongoDB entries  
- Download and extract PDF content by pages  
- Chunk pages into text segments suitable for embedding  
- Generate contextual embeddings using Voyage-Context-3  
- Store embeddings and full-text pages with metadata in MongoDB Atlas  

**1.2 Interactive Q&A Agent with Vector Search**  
- Receive user chat input and generate clarifying questions  
- Collect user answers to clarifying questions  
- Generate query embedding with Voyage-Context-3  
- Perform vector similarity search in MongoDB Atlas  
- Use retrieved documents as context for OpenAI GPT-4.1-Mini to answer user query  
- Respond to user via chat with generated answers  

Supporting features include sub-workflows for scalable processing, batch handling for large documents, and multi-turn chat with human-in-the-loop clarifying questions to improve search relevance.

---

### 2. Block-by-Block Analysis

#### Block 1: Initialization and Document Preparation

- **Overview:**  
  This block sets the initial document URL, clears any existing related data from the MongoDB collection to avoid duplicates, downloads the PDF, and extracts text split by pages.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Set Variables  
  - Clear Collection  
  - Import Research Paper (HTTP Request)  
  - Extract from File  
  - Split Pages  
  - Add Page Number  

- **Node Details:**  
  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow manually  
    - Inputs: None  
    - Outputs: Triggers "Set Variables"  

  - **Set Variables**  
    - Type: Set  
    - Role: Define document URL (`https://arxiv.org/pdf/2402.06196`)  
    - Outputs: URL variable used downstream in HTTP request and database query  
    - Edge Cases: URL must be accessible and return a valid PDF  

  - **Clear Collection**  
    - Type: MongoDB node (delete operation)  
    - Role: Deletes existing documents in collection where metadata.url matches current URL  
    - Configuration: Query deletes all documents with `"metadata.url": "document URL"`  
    - Edge Cases: MongoDB connection/auth errors; ensures no stale data remains  

  - **Import Research Paper**  
    - Type: HTTP Request  
    - Role: Downloads PDF from the URL set in variables  
    - Configuration: URL is dynamic, sourced from "Set Variables" node  
    - Edge Cases: Network failures, 404 or invalid content-type errors  

  - **Extract from File**  
    - Type: Extract from File (PDF operation)  
    - Role: Extracts text content from the downloaded PDF, returning text split by pages  
    - Configuration: `joinPages` set to false to keep pages separate  
    - Edge Cases: Corrupt PDF, extraction failures  

  - **Split Pages**  
    - Type: Split Out  
    - Role: Splits the extracted file text into individual page items for further processing  
    - Field: Splits on `text` field  
    - Outputs: Individual page JSON objects  

  - **Add Page Number**  
    - Type: Set  
    - Role: Adds a `pageNumber` field derived from the item index + 1 for proper page tracking  
    - Outputs: Page number metadata alongside page text  

---

#### Block 2: Chunking and Embedding Generation

- **Overview:**  
  Processes each page by chunking its text into 1000-character segments, then sends these chunks in batches of 3 pages to the Voyage-Context-3 embedding API. The embeddings are stored with associated text and metadata.

- **Nodes Involved:**  
  - Batch 10 (splitInBatches, batch size 10)  
  - Call Embeddings Subworkflow (executeWorkflow)  
  - Subworkflow Trigger (executeWorkflowTrigger)  
  - Loop Over Items (splitInBatches, batch size 3)  
  - Page Ref (No Operation)  
  - Chunk Page Text (Code)  
  - Voyage-Context-3 Embeddings (HTTP Request)  
  - Split Out  
  - Combine Content & Metadata (Set)  
  - Insert Document Page (MongoDB insert)  
  - Split Out1  
  - Combine Content & Vectors (Set)  
  - Insert Documents Vectors (MongoDB insert)  
  - For Each Group (splitInBatches with reset option)  
  - No Operation, do nothing (NoOp)  
  - Aggregate1 (aggregate all item data)  
  - Done (Set)  

- **Node Details:**  
  - **Batch 10**  
    - Splits pages into batches of 10 for manageable processing  
    - Edge Cases: Large documents may require tuning of batch size  

  - **Call Embeddings Subworkflow**  
    - Executes a sub-workflow (the same workflow, recursively) that handles embedding generation for batches  
    - Waits for subworkflow completion before continuing  
    - Inputs: URL, text, pageNumber for each page chunk  

  - **Subworkflow Trigger**  
    - Entry point for the subworkflow invoked by "Call Embeddings Subworkflow"  
    - Inputs: text, url, pageNumber  

  - **Loop Over Items**  
    - Processes items in batches of 3 pages (recommended chunk size for contextual embeddings)  
    - Edge Cases: Ensures memory stability by processing smaller groups  

  - **Page Ref**  
    - NoOp node used to pass page metadata forward  

  - **Chunk Page Text**  
    - Code node chunks page text into 1000-character segments without overlap (overlap=0) as recommended  
    - Key expression: uses JavaScript substring on page text  
    - Edge Cases: Empty or very short pages may produce fewer chunks  

  - **Voyage-Context-3 Embeddings**  
    - HTTP Request to Voyage.ai’s contextual embedding API  
    - Sends all chunks in a single request for contextualized embeddings  
    - Auth: HTTP Header via credentials for Voyage.ai API  
    - Payload: "inputs" is an array of chunk arrays; model: "voyage-context-3"; input_type: "document"  
    - Edge Cases: API rate limits, network errors, invalid chunk formatting  

  - **Split Out**  
    - Splits embedding results array into individual embedding items for further processing  

  - **Combine Content & Metadata**  
    - Sets combined object with text chunk, embedding vector, and metadata (pageNumber, url) for Mongo insertion  

  - **Insert Document Page**  
    - Inserts full page text and metadata as a document into MongoDB (allows retrieval by page for deep dives)  

  - **Split Out1**  
    - Splits embedded chunk data from for-each group processing  

  - **Combine Content & Vectors**  
    - Prepares individual chunk with text, vector embedding, and metadata for database storage  

  - **Insert Documents Vectors**  
    - Inserts chunk embeddings with metadata into MongoDB vector store collection  

  - **For Each Group**  
    - Manages batch processing of chunks with reset flag to ensure all data processed sequentially  

  - **No Operation, do nothing**  
    - NoOp node to facilitate merge branches  

  - **Aggregate1**  
    - Aggregates all inserted chunk data for batch completion  

  - **Done**  
    - Final node setting response "ok" indicating ingestion completion  

---

#### Block 3: Interactive Q&A Chat with Clarifying Questions and RAG Agent

- **Overview:**  
  Enables interactive chat input from user, generates clarifying questions to refine query context, uses Voyage-Context-3 to embed the refined query, performs similarity search on MongoDB vector store, and answers the user with OpenAI GPT-4.1-Mini leveraging retrieved documents as context.

- **Nodes Involved:**  
  - When chat message received (Chat Trigger)  
  - Get Query (Set)  
  - Generate Clarifying Questions (Information Extractor)  
  - Split Questions (Split Out)  
  - Loop Over Questions (splitInBatches)  
  - Wait for Answer (Chat)  
  - Aggregate Answers (Aggregate)  
  - Quick Confirmation (Chat)  
  - Voyage-Context-3 Embeddings1 (HTTP Request)  
  - Perform Similarity Search (MongoDB aggregate)  
  - Aggregate (Aggregate)  
  - Quick Update (Chat)  
  - RAG Agent (OpenAI GPT-4.1-Mini)  
  - Respond to User (Chat)  
  - Query Ref (NoOp)  
  - Fetch Document By Page Number (MongoDB Tool)  
  - Combine Content & Vectors (Set)  
  - Insert Documents Vectors (MongoDB insert)  

- **Node Details:**  
  - **When chat message received**  
    - Chat Trigger node to receive user message via webhook (public)  
    - Edge Cases: Requires published workflow to be publicly accessible  

  - **Get Query**  
    - Extracts user's chat input into a `query` variable  

  - **Generate Clarifying Questions**  
    - Uses Langchain Information Extractor node with system prompt to generate 2 clarifying questions to refine user query  
    - Output schema expects array of strings `questions`  

  - **Split Questions**  
    - Splits generated clarifying questions into individual items for user response  

  - **Loop Over Questions**  
    - Sends each clarifying question to user and waits for their answers  

  - **Wait for Answer**  
    - Chat node that waits for user reply to each clarifying question  

  - **Aggregate Answers**  
    - Aggregates all user responses to clarifying questions into one array  

  - **Quick Confirmation**  
    - Sends a non-blocking acknowledgement message to user indicating search is starting  

  - **Voyage-Context-3 Embeddings1**  
    - Embeds the combined user query plus clarifying answers using Voyage-Context-3 model for similarity search  
    - Sends query embedding as `"inputs": [[ query + answers ]]` with `input_type: "query"`  

  - **Perform Similarity Search**  
    - MongoDB aggregate query with `$vectorSearch` stage to find top 10 closest documents based on embedding vector  
    - Projects text, metadata, and score fields  
    - Edge Cases: MongoDB vector index must be configured, query vector must be valid, connection/auth issues  

  - **Aggregate**  
    - Aggregates retrieved documents for downstream processing  

  - **Quick Update**  
    - Sends a quick, randomized progress message to user indicating number of results found  

  - **RAG Agent**  
    - OpenAI GPT-4.1-Mini node uses retrieved documents as context and user query plus clarifying answers to generate final answer  
    - System prompt instructs to use only the `<documents>` context  
    - Outputs JSON answer content  

  - **Respond to User**  
    - Sends final answer back to user chat, non-blocking  

  - **Query Ref**  
    - NoOp node used during loop for flow control  

  - **Fetch Document By Page Number**  
    - MongoDB Tool node allows fetching full document pages by pageNumber on demand for deeper context if needed  
    - Uses AI tool input to get `pageNumber` parameter  
    - Edge Cases: PageNumber must exist, embedding key must not exist for filtering  

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                                   | Input Node(s)                          | Output Node(s)                         | Sticky Note                                                                                                  |
|-----------------------------|----------------------------------|-------------------------------------------------|--------------------------------------|--------------------------------------|-------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Start the ingestion process                       | —                                    | Set Variables                        | ## 1. Starting Fresh To begin, we'll define our document URL to process...                                   |
| Set Variables               | Set                              | Define document URL variable                      | When clicking ‘Execute workflow’     | Clear Collection                    | ## 1. Starting Fresh To begin, we'll define our document URL to process...                                   |
| Clear Collection            | MongoDB (delete)                 | Clear existing documents for the URL              | Set Variables                       | Import Research Paper               | ## 1. Starting Fresh To begin, we'll define our document URL to process...                                   |
| Import Research Paper       | HTTP Request                    | Download PDF from URL                             | Clear Collection                    | Extract from File                  | ## 2. Download Paper and Split Into Pages Read more about the HTTP node https://docs.n8n.io/integrations... |
| Extract from File           | Extract from File (PDF)           | Extract text split by pages from PDF              | Import Research Paper               | Split Pages                       | ## 2. Download Paper and Split Into Pages Read more about the HTTP node https://docs.n8n.io/integrations... |
| Split Pages                | Split Out                        | Split extracted text into individual pages        | Extract from File                  | Add Page Number                   | ## 2. Download Paper and Split Into Pages Read more about the HTTP node https://docs.n8n.io/integrations... |
| Add Page Number             | Set                              | Add pageNumber metadata to each page               | Split Pages                       | Batch 10                         | ## 2. Download Paper and Split Into Pages Read more about the HTTP node https://docs.n8n.io/integrations... |
| Batch 10                   | Split In Batches                 | Batch pages into groups of 10 for processing      | Add Page Number                   | Call Embeddings Subworkflow       | ## 3. For Large Documents, Use Subworkflows for Better Performance Learn more about Subworkflows https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow |
| Call Embeddings Subworkflow | Execute Workflow                 | Process batch of pages for embeddings              | Batch 10                         | Wait                            | ## 3. For Large Documents, Use Subworkflows for Better Performance Learn more about Subworkflows https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow |
| Wait                       | Wait                            | Pause for subworkflow processing                    | Call Embeddings Subworkflow       | Loop Over Items                 | ## 3. For Large Documents, Use Subworkflows for Better Performance Learn more about Subworkflows https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow |
| Subworkflow Trigger         | Execute Workflow Trigger         | Entry point for subworkflow handling embeddings    | —                                | Loop Over Items                 | ## 3. For Large Documents, Use Subworkflows for Better Performance Learn more about Subworkflows https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow |
| Loop Over Items             | Split In Batches                 | Process pages in groups of 3 for embedding context | Subworkflow Trigger             | Page Ref (main branch), No output (other branch) | ## 4. Contextual Embeddings Using Voyage-Context-3 Learn more about Voyage-Context-3 https://blog.voyageai.com/2025/07/23/voyage-context-3/ |
| Page Ref                   | No Operation                     | Pass page metadata                                | Loop Over Items                 | Chunk Page Text, Combine Content & Metadata | ## 4. Contextual Embeddings Using Voyage-Context-3 Learn more about Voyage-Context-3 https://blog.voyageai.com/2025/07/23/voyage-context-3/ |
| Chunk Page Text             | Code                            | Chunk page text into 1000-char segments without overlap | Page Ref                        | Voyage-Context-3 Embeddings      | ## 4. Contextual Embeddings Using Voyage-Context-3 Learn more about Voyage-Context-3 https://blog.voyageai.com/2025/07/23/voyage-context-3/ |
| Voyage-Context-3 Embeddings | HTTP Request                    | Generate embeddings for chunks                      | Chunk Page Text                 | Split Out                       | ## 4. Contextual Embeddings Using Voyage-Context-3 Learn more about Voyage-Context-3 https://blog.voyageai.com/2025/07/23/voyage-context-3/ |
| Split Out                  | Split Out                        | Split embeddings results into individual items     | Voyage-Context-3 Embeddings      | For Each Group                 | ## 4. Contextual Embeddings Using Voyage-Context-3 Learn more about Voyage-Context-3 https://blog.voyageai.com/2025/07/23/voyage-context-3/ |
| For Each Group             | Split In Batches                 | Batch process embedding chunks                       | Split Out                      | No Operation, do nothing (main branch), Split Out1 (other branch) | ## 4. Contextual Embeddings Using Voyage-Context-3 Learn more about Voyage-Context-3 https://blog.voyageai.com/2025/07/23/voyage-context-3/ |
| No Operation, do nothing    | No Operation                    | Placeholder to merge branches                        | For Each Group                 | Merge                         | ## 4. Contextual Embeddings Using Voyage-Context-3 Learn more about Voyage-Context-3 https://blog.voyageai.com/2025/07/23/voyage-context-3/ |
| Split Out1                 | Split Out                        | Split grouped chunk data for further processing     | For Each Group                 | Combine Content & Vectors     | ## 4. Contextual Embeddings Using Voyage-Context-3 Learn more about Voyage-Context-3 https://blog.voyageai.com/2025/07/23/voyage-context-3/ |
| Combine Content & Vectors   | Set                             | Combine chunk text, embeddings, and metadata        | Split Out1                    | Insert Documents Vectors      | ## 5. Store Vectors and Full Page Text for Advanced RAG Search Learn about MongoDB node https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.mongodb |
| Insert Documents Vectors    | MongoDB (insert)                | Insert embedding chunks and metadata into MongoDB   | Combine Content & Vectors     | Aggregate1                   | ## 5. Store Vectors and Full Page Text for Advanced RAG Search Learn about MongoDB node https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.mongodb |
| Aggregate1                 | Aggregate                       | Aggregate inserted chunk data                        | Insert Documents Vectors      | For Each Group               | ## 5. Store Vectors and Full Page Text for Advanced RAG Search Learn about MongoDB node https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.mongodb |
| Combine Content & Metadata  | Set                             | Combine full page text and metadata                  | Page Ref                      | Insert Document Page         | ## 5. Store Vectors and Full Page Text for Advanced RAG Search Learn about MongoDB node https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.mongodb |
| Insert Document Page        | MongoDB (insert)                | Insert full page text document into MongoDB          | Combine Content & Metadata    | Merge                       | ## 5. Store Vectors and Full Page Text for Advanced RAG Search Learn about MongoDB node https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.mongodb |
| Merge                      | Merge (chooseBranch)            | Merge branches from NoOp and Insert Document Page    | No Operation, do nothing, Insert Document Page | Done                       |                                                                                                             |
| Done                       | Set                             | Set response "ok" after ingestion complete           | Merge                        | Loop Over Items              |                                                                                                             |
| When chat message received  | Langchain Chat Trigger          | Receive user query via chat webhook                   | —                            | Get Query                   | ## 6. Asking Clarifying Questions For Contextual Search Read more about Respond To Chat https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.chat |
| Get Query                  | Set                             | Extract query text from user chat input                | When chat message received   | Generate Clarifying Questions | ## 6. Asking Clarifying Questions For Contextual Search Read more about Respond To Chat https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.chat |
| Generate Clarifying Questions | Langchain Information Extractor | Generate 2 clarifying questions to refine user query | Get Query                    | Split Questions             | ## 6. Asking Clarifying Questions For Contextual Search Read more about Respond To Chat https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.chat |
| Split Questions            | Split Out                       | Split questions array into individual question items   | Generate Clarifying Questions | Loop Over Questions         | ## 6. Asking Clarifying Questions For Contextual Search Read more about Respond To Chat https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.chat |
| Loop Over Questions        | Split In Batches                | Ask clarifying questions one by one and collect answers | Split Questions              | Aggregate Answers, Query Ref | ## 6. Asking Clarifying Questions For Contextual Search Read more about Respond To Chat https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.chat |
| Wait for Answer            | Langchain Chat                  | Wait for user to answer clarifying questions           | Loop Over Questions          | Loop Over Questions         | ## 6. Asking Clarifying Questions For Contextual Search Read more about Respond To Chat https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.chat |
| Aggregate Answers          | Aggregate                       | Aggregate all user answers to clarifying questions     | Loop Over Questions          | Quick Confirmation          | ## 6. Asking Clarifying Questions For Contextual Search Read more about Respond To Chat https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.chat |
| Quick Confirmation         | Langchain Chat                  | Send quick non-blocking acknowledgement to user         | Aggregate Answers            | Voyage-Context-3 Embeddings1 | ## 6. Asking Clarifying Questions For Contextual Search Read more about Respond To Chat https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.chat |
| Voyage-Context-3 Embeddings1 | HTTP Request                  | Generate query embedding including clarifying answers    | Quick Confirmation          | Perform Similarity Search   | ## 2. MongoDB Atlas Vector Search Using Voyage-Context-3 Learn more about MongoDB $vectorSearch queries https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/ |
| Perform Similarity Search  | MongoDB Aggregate              | Vector similarity search for top 10 nearest document chunks | Voyage-Context-3 Embeddings1 | Aggregate                  | ## 2. MongoDB Atlas Vector Search Using Voyage-Context-3 Learn more about MongoDB $vectorSearch queries https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/ |
| Aggregate                  | Aggregate                       | Aggregate retrieved documents                            | Perform Similarity Search    | Quick Update                | ## 2. MongoDB Atlas Vector Search Using Voyage-Context-3 Learn more about MongoDB $vectorSearch queries https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/ |
| Quick Update               | Langchain Chat                  | Send progress message with number of results             | Aggregate                   | RAG Agent                  | ## 2. MongoDB Atlas Vector Search Using Voyage-Context-3 Learn more about MongoDB $vectorSearch queries https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/ |
| RAG Agent                  | Langchain OpenAI GPT-4.1-Mini  | Generate final answer using retrieved documents context   | Quick Update                | Respond to User            | ## 3. Q&A Agent using OpenAI GPT-4.1-Mini Read more about the OpenAI node https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.openai |
| Respond to User            | Langchain Chat                  | Send final answer to user                                 | RAG Agent                   | —                          | ## 3. Q&A Agent using OpenAI GPT-4.1-Mini Read more about the OpenAI node https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.openai |
| Query Ref                  | No Operation                   | Control flow during question loop                        | Loop Over Questions         | Wait for Answer             |                                                                                                             |
| Fetch Document By Page Number | MongoDB Tool                 | Fetch full document page by pageNumber on demand         | —                          | RAG Agent                  |                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Execute workflow’"  

2. **Set Variables Node:**  
   - Type: Set  
   - Name: "Set Variables"  
   - Define a variable `url` with the research paper PDF URL, e.g. `https://arxiv.org/pdf/2402.06196`  
   - Connect output from Manual Trigger  

3. **Clear MongoDB Collection Node:**  
   - Type: MongoDB  
   - Operation: Delete  
   - Collection: `documents`  
   - Query: `{ "metadata.url": "{{ $json.url }}" }`  
   - Credentials: Configure MongoDB credentials  
   - Connect output from "Set Variables"  

4. **Import Research Paper Node:**  
   - Type: HTTP Request  
   - URL: `={{ $('Set Variables').first().json.url }}` (dynamic)  
   - Method: GET (default)  
   - Connect output from "Clear Collection"  

5. **Extract from File Node:**  
   - Type: Extract from File  
   - Operation: PDF  
   - Options: `joinPages = false` (to keep pages separate)  
   - Connect output from "Import Research Paper"  

6. **Split Pages Node:**  
   - Type: Split Out  
   - Field To Split Out: `text`  
   - Connect output from "Extract from File"  

7. **Add Page Number Node:**  
   - Type: Set  
   - Add field: `pageNumber` = `{{$itemIndex + 1}}`  
   - Include other fields  
   - Connect output from "Split Pages"  

8. **Batch 10 Node:**  
   - Type: Split In Batches  
   - Batch Size: 10  
   - Connect output from "Add Page Number"  

9. **Call Embeddings Subworkflow Node:**  
   - Type: Execute Workflow  
   - Workflow ID: The current workflow (self-reference)  
   - Wait For SubWorkflow: Enabled  
   - Workflow Inputs: Map `url`, `text`, `pageNumber` from input  
   - Connect output from "Batch 10"  

10. **Wait Node:**  
    - Type: Wait  
    - Default Parameters  
    - Connect output from "Call Embeddings Subworkflow"  

11. **Subworkflow Entry Point:**  
    - Add Execute Workflow Trigger node  
    - Inputs: `text`, `url`, `pageNumber`  
    - This node acts as subworkflow start for embedding generation  
    - Connect this node to the next nodes below  

12. **Loop Over Items Node:**  
    - Type: Split In Batches  
    - Batch Size: 3 (process 3 pages at a time)  
    - Connect output from Subworkflow Trigger  

13. **Page Ref Node:**  
    - Type: No Operation  
    - Used to hold page metadata  
    - Connect output from "Loop Over Items"  

14. **Chunk Page Text Node:**  
    - Type: Code  
    - JavaScript code: chunk page text into 1000-character segments without overlap, removing newlines  
    - Input: page text from "Page Ref"  
    - Connect output from "Page Ref"  

15. **Voyage-Context-3 Embeddings Node:**  
    - Type: HTTP Request  
    - URL: `https://api.voyageai.com/v1/contextualizedembeddings`  
    - Method: POST  
    - Authentication: HTTP Header with Voyage.ai credentials  
    - JSON Body:
      ```json
      {
        "inputs": $input.all().map(item => item.json.chunks.compact()),
        "input_type": "document",
        "model": "voyage-context-3"
      }
      ```
    - Connect output from "Chunk Page Text"  

16. **Split Out Node:**  
    - Type: Split Out  
    - Field To Split Out: `data` (embedding results)  
    - Connect output from "Voyage-Context-3 Embeddings"  

17. **For Each Group Node:**  
    - Type: Split In Batches  
    - Reset: `={{ $('For Each Group').context.done }}`  
    - Connect output from "Split Out"  

18. **No Operation, do nothing Node:**  
    - Type: No Operation  
    - Connect main output from "For Each Group"  

19. **Split Out1 Node:**  
    - Type: Split Out  
    - Field To Split Out: `data`  
    - Connect second output from "For Each Group"  

20. **Combine Content & Vectors Node:**  
    - Type: Set  
    - Assign:  
      - `text` = chunk text from original chunk array  
      - `embeddings` = embedding vector from `$json.embedding`  
      - `metadata` = object with `pageNumber` and `url` from "Page Ref"  
    - Connect output from "Split Out1"  

21. **Insert Documents Vectors Node:**  
    - Type: MongoDB Insert  
    - Collection: `documents`  
    - Fields: `text`, `embeddings`, `metadata`  
    - Credentials: MongoDB credentials  
    - Connect output from "Combine Content & Vectors"  

22. **Aggregate1 Node:**  
    - Type: Aggregate  
    - Aggregate All Item Data  
    - Connect output from "Insert Documents Vectors"  

23. **Combine Content & Metadata Node:**  
    - Type: Set  
    - Assign:  
      - `text` = full page text  
      - `metadata` = object with `pageNumber` and `url`  
    - Connect output from "Page Ref"  

24. **Insert Document Page Node:**  
    - Type: MongoDB Insert  
    - Collection: `documents`  
    - Fields: `text`, `metadata`  
    - Credentials: MongoDB credentials  
    - Connect output from "Combine Content & Metadata"  

25. **Merge Node:**  
    - Type: Merge (chooseBranch mode)  
    - Connect outputs from "No Operation, do nothing" and "Insert Document Page"  

26. **Done Node:**  
    - Type: Set  
    - Assign `response` = "ok"  
    - Execute once  
    - Connect output from "Merge"  

---

**Q&A Agent Setup**

27. **When chat message received Node:**  
    - Type: Langchain Chat Trigger (public webhook)  
    - Connect outputs to "Get Query"  

28. **Get Query Node:**  
    - Type: Set  
    - Assign `query` = `{{$json.chatInput}}`  
    - Connect output to "Generate Clarifying Questions"  

29. **Generate Clarifying Questions Node:**  
    - Type: Langchain Information Extractor  
    - System Prompt: Generates 2 clarifying questions for user's query  
    - Input: `query`  
    - Connect output to "Split Questions"  

30. **Split Questions Node:**  
    - Type: Split Out  
    - Field To Split Out: `output.questions`  
    - Connect output to "Loop Over Questions"  

31. **Loop Over Questions Node:**  
    - Type: Split In Batches  
    - No batch size (defaults to 1)  
    - Connect outputs to "Aggregate Answers" and "Query Ref"  

32. **Wait for Answer Node:**  
    - Type: Langchain Chat  
    - Message: `{{$json.question}}` (each clarifying question)  
    - Wait for user reply enabled  
    - Connect output to "Loop Over Questions"  

33. **Aggregate Answers Node:**  
    - Type: Aggregate  
    - Aggregate all answers into array `answers`  
    - Connect output to "Quick Confirmation"  

34. **Quick Confirmation Node:**  
    - Type: Langchain Chat  
    - Message: "Thanks. Please wait whilst I search the relevant document."  
    - Wait user reply disabled  
    - Connect output to "Voyage-Context-3 Embeddings1"  

35. **Voyage-Context-3 Embeddings1 Node:**  
    - Type: HTTP Request  
    - URL: `https://api.voyageai.com/v1/contextualizedembeddings`  
    - Method: POST  
    - Auth: HTTP Header (Voyage.ai credentials)  
    - JSON Body: Embeds combined user query plus all clarifying answers as query input  
    - Connect output to "Perform Similarity Search"  

36. **Perform Similarity Search Node:**  
    - Type: MongoDB Aggregate  
    - Query: MongoDB Atlas `$vectorSearch` with embedding vector, limit 10  
    - Collection: `documents`  
    - Credentials: MongoDB  
    - Connect output to "Aggregate"  

37. **Aggregate Node:**  
    - Type: Aggregate  
    - Aggregate all retrieved documents  
    - Connect output to "Quick Update"  

38. **Quick Update Node:**  
    - Type: Langchain Chat  
    - Message: Randomized progress message indicating number of results found  
    - Connect output to "RAG Agent"  

39. **RAG Agent Node:**  
    - Type: Langchain OpenAI GPT-4.1-Mini  
    - System prompt: Use only `<documents>` context to answer  
    - Messages include retrieved documents and user query plus clarifying answers  
    - Credentials: OpenAI  
    - Connect output to "Respond to User"  

40. **Respond to User Node:**  
    - Type: Langchain Chat  
    - Message: Generated answer content  
    - Wait for reply disabled  

41. **Fetch Document By Page Number Node (Optional):**  
    - Type: MongoDB Tool  
    - Query: Filter by metadata.pageNumber with AI tool input, filtering documents without embeddings key  
    - Credentials: MongoDB  
    - Connect output to "RAG Agent" for deep dive context if required  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| ## 1. Starting Fresh To begin, we'll define our document URL to process. Since we breaking down the document in full and don't want duplicate entries in our database the next time we run this ingestion step, we'll clear our MongoDB Collection and start fresh.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note on Initialization Block                                                                |
| ## 2. Download Paper and Split Into Pages [Read more about the HTTP node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest) A common way to extract from a PDF is to use the "Extract from File" node. This will return the file's metadata as well as the text split into pages - which is exactly what we need for this particular flow. Note however, if charts and images are also required to be searchable then you may need to use a vision model to properly parse these elements.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Sticky Note on Document Download and Extraction                                                  |
| ## 3. For Large Documents, Use Subworkflows for Better Performance [Learn more about Subworkflows](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow) For practical applications, smaller executions are generally preferred to help reduce out-of-memory issues in n8n. This is especially so when working with sizable documents and embedding vectors. In this particular setup, we'll process each page separately and in sequence. Though this will take longer, it can ensure our instance's stability for other workflows.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note on Subworkflow Use                                                                    |
| ## 4. Contextual Embeddings Using Voyage-Context-3 [Learn more about Voyage-Context-3](https://blog.voyageai.com/2025/07/23/voyage-context-3/) Voyage-Context-3 is a new contextual chunk embedding model which allows you to include document context to improve retrieval accuracy. Whereas previously you may have had to manually augment incoming chunks, Voyage-Context-3 does this automatically by allowing your to bulk upload chunks and encoding context from the aggregate of all. For this demonstration, we won't use the full document - that's a lot of data! - but rather, we'll embed sets of 3 sequential pages. This should be enough to cover at least "chapter"-level context.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky Note on Embedding Model and Chunking                                                      |
| ## 5. Store Vectors and Full Page Text for Advanced RAG Search [Learn about MongoDB node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.mongodb) We'll insert these context-aware embedding chunks into our MongoDB Atlas Vector Store along with their equivalent text content and metadata. Additionally, we can also store the full page text in Mongo as well - as we're able to later filter by page number, this can be a handy way of expanding on chunks later on in our searches.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note on MongoDB Storage                                                                   |
| ## 6. Asking Clarifying Questions For Contextual Search [Read more about Respond To Chat](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.chat) **Respond to Chat** is a new Human-in-the-loop node where by the "Human" isn't an external source but the active user instead! This can make for really interesting multi-turn chat interactions which can feel more personalised and thus improving the user experience. For this demonstration, we'll implement a "clarifying questions" loop where a set of questions are presented to the user to answer. These questions can help refine the context of the user's query and produce better search results so it's a really useful technique to know. Once the questions are answered, we can again use the "respond to chat" node to give a quick acknowledgement but this time with the "wait for reply" toggled off - this essentially becomes a "send message" operation. | Sticky Note on Clarifying Questions and Human-In-The-Loop Chat Interaction                       |
| ## 2. MongoDB Atlas Vector Search Using Voyage-Context-3 [Learn more about MongoDB $vectorSearch queries](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/) There's not an official MongoDB vector store node support so we'll have to write raw queries using the MongoDB node. Good thing that they're not really that hard to write once you get over the initial learning curve. Again we'll use Voyage-Context-3 on our query to match the embeddings in our vector store. Once the documents are matched, we can use the "respond to chat" node to send a quick progress message to the user. These micro-updates can help break up long pauses between user questions and agent responses and provide a feeling of responsiveness. | Sticky Note on MongoDB Vector Search Query                                                     |
| ## 3. Q&A Agent using OpenAI GPT-4.1-Mini [Read more about the OpenAI node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.openai) Using the retrieved documents with our AI agent completes our Q&A agent flow and let's us respond to the user's query with unmatched relevancy and accuracy - or so we've been promised! Of course, you can't really tell unless you try it out for yourself. In testing this workflow, I did find the ranking of retrieved documents to be better than naive document chunking. I could definitely recommend this contextual embeddings approach for the more demanding RAG requirements. | Sticky Note on RAG Q&A Agent Using OpenAI GPT-4.1-Mini                                     |
| ![](https://cdn.subworkflow.ai/n8n-templates/banner_595x311.png#full-width)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Branding banner image                                                                           |
| **Contextual Chunk Embeddings Using Voyage-Context-3 and Mongo Atlas** On my never-ending quest to find the best embeddings model, I was intrigued to come across [Voyage-Context-3](https://blog.voyageai.com/2025/07/23/voyage-context-3/) by MongoDB and was excited to give it a try. This template implements the embedding model on a Arxiv research paper and stores the results in a Vector store. It was only fitting to use Mongo Atlas from the same parent company. This template also includes a RAG-based Q&A agent which taps into the vector store as a test to helps qualify if the embeddings are any good and if this is even noticeable. How it works This template is split into 2 parts. The first part being the import of a research document which is then chunked and embedded into our vector store. The second part builds a RAG-based Q&A agent to test the vector store retrieval on the research paper. How to use * First ensure you create a Voyage account [voyageai.com](https://voyageai.com) and a MongoDB database ready. * Start with Step 1 and fill in the "Set Variables" node and Click on the Manual Execute Trigger. This will take care of populating the vector store with the research paper. * To use the Q&A agent, it is required to publish the workflow to access the public chat interface. This is because "Respond to Chat" works best in this mode and not in editor mode. * To use for your own document, edit the "Set Variables" node to define the URL to your own document. * This embeddings approach should work best on larger documents. Requirements * [Voyageai.com](https://voyageai.com) account for embeddings. You may need to add credit to get a reasonable RPM for this workflow. * MongoDB database either self-hosted or online at [https://www.mongodb.com](https://www.mongodb.com). * OpenAI account for RAG Q&A agent. Need Help? Join the [Discord](https://discord.com/invite/XPKeKXeB7d) or ask in the [Forum](https://community.n8n.io/)! Happy Hacking! | Workflow overview, usage instructions, project credits, and support links |

---

# Disclaimer

The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.