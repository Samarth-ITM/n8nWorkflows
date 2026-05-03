Build Document RAG System with Kimi-K2, Gemini Embeddings and Qdrant

https://n8nworkflows.xyz/workflows/build-document-rag-system-with-kimi-k2--gemini-embeddings-and-qdrant-6574


# Build Document RAG System with Kimi-K2, Gemini Embeddings and Qdrant

---

### 1. Workflow Overview

This workflow builds a Retrieval-Augmented Generation (RAG) document system using three key technologies: the Kimi-K2 language model via Featherless.ai for contextual summarization, Google Gemini Embeddings for vector representation, and Qdrant as the vector store for indexing and retrieval. The target use case is to create a knowledge base from a large, information-dense document (demonstrated here with the UK Highway Code PDF), enabling efficient and context-aware search and question answering.

The workflow is logically divided into the following blocks:

- **1.1 Document Ingestion and Preprocessing:** Download the large PDF document, extract text, and split it into chunks/pages.
- **1.2 Contextual Summarization using Kimi-K2:** For each chunk, generate a contextual summary with surrounding context to improve search relevance.
- **1.3 Embedding Generation with Gemini Embeddings:** Convert the contextual summaries into dense vectors using Google Gemini Embeddings API.
- **1.4 Vector Store Setup and Population with Qdrant:** Create Qdrant collection, index the vectors along with metadata, and build a summary index for text search.
- **1.5 Query Handling and Retrieval:** Accept queries via triggers, embed them, retrieve relevant document chunks from Qdrant, and answer with a specialized UK Highway Code expert agent.
- **1.6 Example Usage and Testing:** Provides sample driving theory questions and demonstrates automated answering with the integrated system.

---

### 2. Block-by-Block Analysis

#### 2.1 Document Ingestion and Preprocessing

- **Overview:** Downloads a large PDF document (UK Highway Code), extracts its text, and splits it into pages for subsequent processing.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Download Large Document (HTTP Request)  
  - Extract from File (PDF Extraction)  
  - Split Pages (Split Out)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually  
    - Inputs: None  
    - Outputs: Triggers "Download Large Document" node  
    - Failure: None typical, manual start

  - **Download Large Document**  
    - Type: HTTP Request  
    - Role: Downloads the official UK Highway Code PDF via fixed URL  
    - Configuration: URL hardcoded to official PDF, no auth  
    - Input: Trigger from manual node  
    - Output: Binary PDF data to "Extract from File"  
    - Failure: Network errors, URL changes or downtime

  - **Extract from File**  
    - Type: Extract from File (PDF)  
    - Role: Extracts text from PDF file pages without joining pages  
    - Configuration: Operation set to PDF, joinPages = false (preserves page boundaries)  
    - Input: Binary PDF file  
    - Output: JSON with text per page to "Split Pages"  
    - Failure: Corrupt PDF, unsupported PDF features

  - **Split Pages**  
    - Type: Split Out  
    - Role: Splits the extracted text array into individual page chunks  
    - Configuration: Field to split out is "text" array from extraction  
    - Input: JSON text array of pages  
    - Output: Each page as separate item to "Kimi-K2 via Featherless.ai"  
    - Failure: Empty or malformed text array

- **Sticky Note:**  
  - Positioned near this block, it emphasizes that this workflow is optimized for large, information-dense documents, and highlights the importance of contextual summaries to retain document structure and relevance.

---

#### 2.2 Contextual Summarization using Kimi-K2

- **Overview:** For each page chunk, generates a succinct contextual summary including relevant surrounding text to improve retrieval quality.

- **Nodes Involved:**  
  - Kimi-K2 via Featherless.ai  
  - Get Response  
  - Loop Over Items  
  - Chunk Ref (NoOp)

- **Node Details:**

  - **Kimi-K2 via Featherless.ai**  
    - Type: Featherless AI Node (LLM interaction)  
    - Role: Uses the Kimi-K2-Instruct model to generate context, title, and keywords for each chunk  
    - Configuration:  
      - Model: moonshotai/Kimi-K2-Instruct  
      - Message: Combines a slice of the full document text (10 pages before to 5 pages after current chunk) plus current chunk text wrapped in `<document>` and `<chunk>` tags with instructions to output only context, title, and keywords.  
      - Temperature: 0.6 (balanced creativity)  
      - System Prompt: Simple assistant with expected output format for parsing  
      - max_tokens dynamically calculated based on prompt length to fit 16384 token limit  
    - Input: Individual page chunk text + full document text context via expressions  
    - Output: Text response containing context, title, keywords  
    - Failure: API errors, token limits exceeded, expression errors  
    - Credentials: Featherless API key required

  - **Get Response**  
    - Type: Set  
    - Role: Parses the LLM response using regex to extract context, title, keywords and stores them as "summary" and "title" fields; also passes original chunk text and page number  
    - Configuration: Regex match on response field to extract data with safety for missing matches  
    - Input: Response from Kimi-K2 node  
    - Output: Structured JSON for next nodes  
    - Failure: Regex mismatch if LLM output format changes

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Controls flow of processed chunks for subsequent embedding and storage  
    - Configuration: Default batch options  
    - Input: Items from "Get Response"  
    - Output: Batches split to feed to "Chunk Ref" and embedding nodes  
    - Failure: Batch processing errors

  - **Chunk Ref**  
    - Type: No Operation (NoOp)  
    - Role: Placeholder node to branch flow toward embeddings generation  
    - Input: From Loop Over Items  
    - Output: To "Retrieval Vectors with Gemini Embeddings 001" node

- **Sticky Note:**  
  - Emphasizes Featherless.ai subscription benefits for unlimited tokens, enabling cost-effective large-scale contextual summarization workloads.

---

#### 2.3 Embedding Generation with Gemini Embeddings

- **Overview:** Converts the contextual summaries into vector embeddings using Google Gemini Embeddings API, specifying task type for retrieval optimization.

- **Nodes Involved:**  
  - Retrieval Vectors with Gemini Embeddings 001 (HTTP Request)  
  - Add Docs To Qdrant Vector Store

- **Node Details:**

  - **Retrieval Vectors with Gemini Embeddings 001**  
    - Type: HTTP Request  
    - Role: Calls Google Gemini Embeddings API with specific parameters  
    - Configuration:  
      - URL: Gemini embedding endpoint  
      - Method: POST  
      - JSON body includes model name, content parts (summary), task_type "RETRIEVAL_DOCUMENT", title, and output dimensionality 768 to reduce vector size  
      - Header: Content-Type application/json  
      - Authentication: Google Palm API credentials  
      - Batching enabled for efficiency  
    - Input: Summary and title from "Chunk Ref"  
    - Output: Embedding vectors with values accessible for storage  
    - Failure: API auth errors, rate limits, schema errors  
    - Sticky Note: Explains new Gemini embedding API features and rationale for using HTTP request over built-in node.

  - **Add Docs To Qdrant Vector Store**  
    - Type: Qdrant Node (upsertPoints)  
    - Role: Inserts embedding vectors as points into Qdrant collection with payload metadata (summary, page number, URL)  
    - Configuration:  
      - Collection: "highwaycode"  
      - Payload fields: content (chunk text), metadata including summary and pageNumber, and source URL  
      - Vector: Uses embedding values in default vector field  
    - Input: Embeddings and metadata from Gemini Embeddings node and Get Response node  
    - Output: To Loop Over Items for next batch or completion  
    - Failure: Qdrant connection or indexing errors  
    - Credentials: Qdrant REST API credentials

- **Sticky Note:**  
  - Instructions on installing and configuring Qdrant node and collection creation steps.

---

#### 2.4 Vector Store Setup and Population with Qdrant

- **Overview:** Creates a Qdrant vector collection, then sets up a payload index on the summary field to enable full-text search capabilities.

- **Nodes Involved:**  
  - Create Collection  
  - Create Summary Index

- **Node Details:**

  - **Create Collection**  
    - Type: Qdrant Node (createCollection)  
    - Role: Defines a new collection named "highwaycode" with vector size 768 and cosine distance metric  
    - Configuration:  
      - Vector params: size 768, distance "Cosine"  
      - Sparse vectors: configured with minicoil IDF modifier (not used further here)  
    - Input: Triggered once after initial batch processing or manually  
    - Output: To "Create Summary Index"  
    - Failure: Collection name conflicts, Qdrant server availability

  - **Create Summary Index**  
    - Type: Qdrant Node (createPayloadIndex)  
    - Role: Creates a text index on the "metadata.summary" field for full-text search  
    - Configuration:  
      - Field Schema: Type "text" with word tokenizer, token length limits, lowercase normalization  
      - Collection: "highwaycode"  
    - Input: From Create Collection  
    - Output: None (setup complete)  
    - Failure: Index creation errors

- **Sticky Note:**  
  - Provides details on Qdrant node installation and collection setup.

---

#### 2.5 Query Handling and Retrieval

- **Overview:** Accepts user queries via chat or subworkflow triggers, generates embeddings for queries, retrieves relevant document chunks from Qdrant, and answers queries using a specialized UK Highway Code expert agent.

- **Nodes Involved:**  
  - When chat message received (LangChain Chat Trigger)  
  - Highway Code Expert (LangChain Agent)  
  - Google Gemini Chat Model1 (LM Chat)  
  - Subworkflow Trigger (Execute Workflow Trigger)  
  - Retrieval Vectors with Gemini-Embeddings-001 (HTTP Request for query embedding)  
  - Query Docs from Qdrant Vector Store  
  - Highway Code Manual, Highway Code Manual1 (LangChain ToolWorkflow)  
  - Call Highway Code Manual Tool (Execute Workflow)  
  - Kimi-K2 via Featherless ai (for final answer generation)

- **Node Details:**

  - **When chat message received**  
    - Type: LangChain Chat Trigger (Webhook)  
    - Role: Entry point for chat queries  
    - Configuration: Webhook ID set, no options  
    - Output: Triggers "Highway Code Expert" node

  - **Highway Code Expert**  
    - Type: LangChain Agent  
    - Role: Expert advisor answering queries about the UK Highway Code, citing page, rule, or legislation  
    - Configuration: System message enforces citations and fallback response if answer not in code  
    - Input: Chat message, plus retrieved chunks as context  
    - Output: Final answer to user  
    - Failure: Model API errors, incomplete retrieval context

  - **Google Gemini Chat Model1**  
    - Type: LangChain LM Chat using Google Gemini  
    - Role: Supports the expert agent with language model capabilities  
    - Credentials: Google Gemini API key

  - **Subworkflow Trigger**  
    - Type: Execute Workflow Trigger  
    - Role: Facilitates modular embedding generation on query text  
    - Inputs: User query string  
    - Outputs: Embedding vectors for query used downstream

  - **Retrieval Vectors with Gemini-Embeddings-001**  
    - Type: HTTP Request  
    - Role: Embeds user queries similarly to document chunks  
    - Input: Query from subworkflow trigger  
    - Output: Embeddings to "Query Docs from Qdrant Vector Store"

  - **Query Docs from Qdrant Vector Store**  
    - Type: Qdrant Node (search queryPoints)  
    - Role: Searches Qdrant collection with query embedding to retrieve top 20 relevant chunks  
    - Configuration: Collection "highwaycode", using default vector field  
    - Output: Points with similarity scores

  - **Highway Code Manual, Highway Code Manual1**  
    - Type: LangChain ToolWorkflows  
    - Role: Provide an interface to query the Highway Code manual workflow internally  
    - Configuration: Accepts query inputs and returns relevant results  
    - Integration: Used by expert agent and other nodes

  - **Call Highway Code Manual Tool**  
    - Type: Execute Workflow  
    - Role: Iterates over questions, calls Kimi-K2 summarization for each question's context and references  
    - Configuration: Synchronous waiting for subworkflow completion per item  
    - Input: Question strings split from test question list

  - **Kimi-K2 via Featherless ai** (used here in answering)  
    - Role: Uses retrieved references and question to select correct answer choice  
    - Configuration: System prompt tailored for UK driving theory test multiple choice questions, returns only answer number  
    - Input: Question text and retrieved references  
    - Output: Selected answer choice number

- **Sticky Notes:**  
  - Notes on MCP Server usage, AI agents, Featherless AI node, and example usage scenarios like automated driving theory test taking.

---

#### 2.6 Example Usage and Testing

- **Overview:** Provides a set of test driving theory questions, feeds them into the retrieval and answering pipeline, and outputs the selected answers.

- **Nodes Involved:**  
  - Test Questions (Set)  
  - Split Out (Split Out)  
  - Call Highway Code Manual Tool  
  - Kimi-K2 via Featherless ai (for answering)

- **Node Details:**

  - **Test Questions**  
    - Type: Set  
    - Role: Contains multiple choice UK driving theory test questions as JSON array  
    - Configuration: Raw JSON with 9 questions, each with 4 answer choices  
    - Output: JSON array to "Split Out"

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits questions array into individual question items  
    - Output: Each question item triggers "Call Highway Code Manual Tool"

  - **Call Highway Code Manual Tool**  
    - As above, executes subworkflow for each question and triggers final answer generation

- **Sticky Note:**  
  - Highlights example usage of Featherless AI node and automated theory test taker demonstration.

---

### 3. Summary Table

| Node Name                          | Node Type                           | Functional Role                              | Input Node(s)             | Output Node(s)                     | Sticky Note                                                                                                                                                                                                                             |
|-----------------------------------|-----------------------------------|----------------------------------------------|---------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger                    | Starts workflow manually                      |                           | Download Large Document          |                                                                                                                                                                                                                                       |
| Download Large Document            | HTTP Request                     | Downloads large PDF document                   | When clicking ‘Execute workflow’ | Extract from File               |                                                                                                                                                                                                                                       |
| Extract from File                 | Extract from File (PDF)           | Extracts text from PDF                         | Download Large Document      | Split Pages                     |                                                                                                                                                                                                                                       |
| Split Pages                      | Split Out                        | Splits extracted text into pages              | Extract from File           | Kimi-K2 via Featherless.ai      | ## 1. Works Best on Large Documents - Emphasizes importance of contextual summaries for large documents ([Highway Code PDF link](https://www.highwaycodeuk.co.uk/uploads/3/2/9/2/3292309/the_official_highway_code_-_10-04-2025_2.pdf)) |
| Kimi-K2 via Featherless.ai       | Featherless AI                   | Generates contextual summaries                 | Split Pages                | Get Response                   | ## 2. Using Kimi-K2 via Featherless.ai for Unlimited Tokens - Featherless subscription benefits for token-intensive summarization                                                                                                   |
| Get Response                    | Set                             | Parses LLM output into structured summary     | Kimi-K2 via Featherless.ai | Loop Over Items                |                                                                                                                                                                                                                                       |
| Loop Over Items                  | Split In Batches                | Controls batch processing of chunks            | Get Response               | Chunk Ref (NoOp), embeddings nodes |                                                                                                                                                                                                                                       |
| Chunk Ref                       | No Operation                   | Branches flow to embedding generation          | Loop Over Items            | Retrieval Vectors with Gemini Embeddings 001 |                                                                                                                                                                                                                                       |
| Retrieval Vectors with Gemini Embeddings 001 | HTTP Request                    | Generates Gemini embeddings from summaries     | Chunk Ref                  | Add Docs To Qdrant Vector Store | ## 3. Gemini-Embeddings-001 for Document Retrieval Embeddings - Details on embedding API and parameters                                                                                                                               |
| Add Docs To Qdrant Vector Store  | Qdrant Node                    | Upserts embeddings and metadata into Qdrant   | Retrieval Vectors with Gemini Embeddings 001 | Loop Over Items                | ## 3.1 Install Qdrant Community Node - Instructions on installing and setting up Qdrant node                                                                                                                                           |
| Create Collection               | Qdrant Node                    | Creates Qdrant collection for vectors          |                           | Create Summary Index            | ## 3.1 Install Qdrant Community Node                                                                                                                                                                                                   |
| Create Summary Index            | Qdrant Node                    | Creates full-text index on summary field       | Create Collection          |                              | ## 3.1 Install Qdrant Community Node                                                                                                                                                                                                   |
| When chat message received       | LangChain Chat Trigger          | Entry point for chat-based user queries        |                           | Highway Code Expert             | ## Example Usage: The Highway Code Q&A Agent - AI Agents documentation link                                                                                                                                                            |
| Highway Code Expert              | LangChain Agent                | Expert agent answering UK Highway Code queries | When chat message received |                              | ## Example Usage: The Highway Code Q&A Agent                                                                                                                                                                                          |
| Google Gemini Chat Model1        | LangChain LM Chat              | Provides LM chat capabilities for expert agent | Highway Code Expert        |                              |                                                                                                                                                                                                                                       |
| Subworkflow Trigger             | Execute Workflow Trigger        | Triggers embedding workflow for queries        |                           | Retrieval Vectors with Gemini-Embeddings-001 | ## 4. Create a Document Retrieval Subworkflow Tool - Subworkflow trigger usage documentation                                                                                                                                           |
| Retrieval Vectors with Gemini-Embeddings-001 | HTTP Request                    | Embeds user queries                             | Subworkflow Trigger        | Query Docs from Qdrant Vector Store |                                                                                                                                                                                                                                       |
| Query Docs from Qdrant Vector Store | Qdrant Node                    | Queries Qdrant vector store for matching docs  | Retrieval Vectors with Gemini-Embeddings-001 | Highway Code Manual, Highway Code Manual1 |                                                                                                                                                                                                                                       |
| Highway Code Manual              | LangChain ToolWorkflow          | Provides query interface to Highway Code manual | Query Docs from Qdrant Vector Store | Highway Code MCP Server         |                                                                                                                                                                                                                                       |
| Highway Code MCP Server          | LangChain MCP Trigger           | MCP server for Highway Code queries             | Highway Code Manual        |                              | ## Example Usage: Highway Code MCP Server                                                                                                                                                                                             |
| Highway Code Manual1             | LangChain ToolWorkflow          | Alternate manual tool for queries                | Query Docs from Qdrant Vector Store | Highway Code Expert            |                                                                                                                                                                                                                                       |
| Call Highway Code Manual Tool    | Execute Workflow                | Calls subworkflow tool for each test question    | Split Out                  | Kimi-K2 via Featherless ai      | ## 4. Create a Document Retrieval Subworkflow Tool                                                                                                                                                                                    |
| Test Questions                  | Set                             | Provides sample driving theory test questions    |                           | Split Out                      | ## Example Usage: The Automated UK Driving Theory Test Taker                                                                                                                                                                          |
| Split Out                      | Split Out                      | Splits question array into individual items       | Test Questions             | Call Highway Code Manual Tool   |                                                                                                                                                                                                                                       |
| Kimi-K2 via Featherless ai      | Featherless AI                 | Answers multiple choice questions based on retrieved references | Call Highway Code Manual Tool |                              | ## Example Usage: The Automated UK Driving Theory Test Taker                                                                                                                                                                          |
| Sticky Note (multiple)           | Sticky Note                    | Documentation and contextual notes                |                           |                              | See detailed sticky note content in relevant blocks                                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger:**
   - Add a "Manual Trigger" node named "When clicking ‘Execute workflow’" to start the workflow manually.

2. **Download Large Document:**
   - Add an "HTTP Request" node named "Download Large Document."
   - Configure it with the URL: `https://www.highwaycodeuk.co.uk/uploads/3/2/9/2/3292309/the_official_highway_code_-_10-04-2025_2.pdf`.
   - Method: GET, no authentication required.
   - Connect "When clicking ‘Execute workflow’" → "Download Large Document."

3. **Extract Text from PDF:**
   - Add "Extract from File" node named "Extract from File."
   - Set operation to "pdf" and disable "joinPages" (false).
   - Input: Binary data from "Download Large Document."
   - Connect "Download Large Document" → "Extract from File."

4. **Split Extracted Text into Pages:**
   - Add "Split Out" node named "Split Pages."
   - Set "fieldToSplitOut" to `text` (array of pages).
   - Connect "Extract from File" → "Split Pages."

5. **Generate Contextual Summaries per Page:**
   - Add "Featherless AI" node named "Kimi-K2 via Featherless.ai."
   - Select model `moonshotai/Kimi-K2-Instruct`.
   - Configure the message with an expression to slice the full document text around the current chunk index (from 10 pages before to 5 pages after), embedding current chunk text as the chunk.
   - Set temperature to 0.6.
   - Add a system prompt instructing the model to output context, title, and keywords in a strict format.
   - Provide Featherless API credentials.
   - Connect "Split Pages" → "Kimi-K2 via Featherless.ai."

6. **Parse LLM Response:**
   - Add "Set" node named "Get Response."
   - Use regex expressions to extract "summary" (context + keywords), "title," "chunk" (text from split page), and "pageNumber" (index + 1).
   - Connect "Kimi-K2 via Featherless.ai" → "Get Response."

7. **Batch Processing of Summaries:**
   - Add "Split In Batches" node named "Loop Over Items."
   - Connect "Get Response" → "Loop Over Items."

8. **No Operation Node for Branching:**
   - Add "NoOp" node named "Chunk Ref" as a placeholder.
   - Connect "Loop Over Items" (second output) → "Chunk Ref."

9. **Generate Embeddings for Summaries:**
   - Add "HTTP Request" node named "Retrieval Vectors with Gemini Embeddings 001."
   - Configure request:
     - URL: Gemini embedding API endpoint.
     - Method: POST.
     - JSON body includes model `models/gemini-embedding-001`, content as current summary, task_type `RETRIEVAL_DOCUMENT`, title, output_dimensionality 768.
     - Headers: Content-Type application/json.
     - Authentication: Google Palm API credentials.
   - Connect "Chunk Ref" → "Retrieval Vectors with Gemini Embeddings 001."

10. **Create Qdrant Collection:**
    - Add "Qdrant" node named "Create Collection."
    - Operation: createCollection.
    - Collection name: "highwaycode."
    - Vector parameters: size 768, distance "Cosine."
    - Sparse vectors: minicoil with IDF modifier.
    - Connect manually or after initial batch to run once.

11. **Create Payload Index on Summary:**
    - Add "Qdrant" node named "Create Summary Index."
    - Operation: createPayloadIndex on field "metadata.summary."
    - Field schema: type "text," tokenizer "word," min token len 2, max token len 10, lowercase true.
    - Connect "Create Collection" → "Create Summary Index."

12. **Add Embeddings to Qdrant:**
    - Add "Qdrant" node named "Add Docs To Qdrant Vector Store."
    - Operation: upsertPoints.
    - Collection: "highwaycode."
    - Point payload includes chunk content, metadata (summary, pageNumber, URL), and vector from Gemini embeddings.
    - Connect "Retrieval Vectors with Gemini Embeddings 001" → "Add Docs To Qdrant Vector Store."
    - Connect "Add Docs To Qdrant Vector Store" → "Loop Over Items" (to continue batch processing).

13. **Set Up Query Input Trigger:**
    - Add LangChain "Chat Trigger" node named "When chat message received."
    - Configure webhook for incoming user chat messages.

14. **Configure Expert Agent:**
    - Add LangChain "Agent" node named "Highway Code Expert."
    - System prompt to act as UK Highway Code expert, requiring citations, fallback if no answer.
    - Connect "When chat message received" → "Highway Code Expert."

15. **Configure Google Gemini Chat Model:**
    - Add LangChain "LM Chat Google Gemini" node.
    - Connect "Highway Code Expert" → "Google Gemini Chat Model1."
    - Provide Google Palm API credentials.

16. **Create Subworkflow Trigger for Query Embedding:**
    - Add "Execute Workflow Trigger" node named "Subworkflow Trigger."
    - Define input parameter "query" (string).

17. **Embed Query Text:**
    - Add HTTP Request node "Retrieval Vectors with Gemini-Embeddings-001."
    - Same configuration as document embeddings but input is query string.
    - Connect "Subworkflow Trigger" → "Retrieval Vectors with Gemini-Embeddings-001."

18. **Query Qdrant Vector Store:**
    - Add Qdrant node named "Query Docs from Qdrant Vector Store."
    - Operation: queryPoints.
    - Collection: "highwaycode."
    - Query vector: embedding values from previous node.
    - Limit: 20 results.
    - Connect "Retrieval Vectors with Gemini-Embeddings-001" → "Query Docs from Qdrant Vector Store."

19. **Set Up LangChain ToolWorkflows:**
    - Create two LangChain "ToolWorkflow" nodes named "Highway Code Manual" and "Highway Code Manual1."
    - Both configured with this workflow ID and input schema accepting "query" string.
    - Connect outputs of Qdrant query to these nodes.
    - Connect "Highway Code Manual1" → "Highway Code Expert."
    - Connect "Highway Code Manual" → "Highway Code MCP Server" (LangChain MCP trigger).

20. **Set Up MCP Server:**
    - Add LangChain MCP Trigger node named "Highway Code MCP Server."
    - Configure webhook path.
    - Connect "Highway Code Manual" → MCP Server.

21. **Automated Driving Theory Test Setup:**
    - Add "Set" node named "Test Questions" with multiple driving theory questions as array.
    - Add "Split Out" node to split questions.
    - Connect "Test Questions" → "Split Out."
    - Add "Execute Workflow" node named "Call Highway Code Manual Tool."
    - Configure to loop over questions, passing each as query input, waiting for subworkflow.
    - Connect "Split Out" → "Call Highway Code Manual Tool."
    - Connect "Call Highway Code Manual Tool" → "Kimi-K2 via Featherless ai" (for final answer selection).
    - Configure "Kimi-K2 via Featherless ai" with system prompt for multiple choice selection, using references from retrieval.

22. **Credentials Setup:**
    - Featherless.ai API Key for Kimi-K2 nodes.
    - Google Palm API credentials for Gemini embedding and chat model nodes.
    - Qdrant REST API credentials for vector storage nodes.

23. **Run Workflow:**
    - Start with manual trigger to download and process document.
    - Once indexed, use chat webhook or test questions to query and get answers.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| ## 1. Works Best on Large Documents: Credits the UK Highway Code Ebook and explains the need for contextual summaries to preserve context lost by simple chunking.                                                                                                                                                                                                                                                                                                                           | [Highway Code PDF](https://www.highwaycodeuk.co.uk/uploads/3/2/9/2/3292309/the_official_highway_code_-_10-04-2025_2.pdf) |
| ## 2. Using Kimi-K2 via Featherless.ai: Describes Featherless.ai’s unlimited token subscription model, making it ideal for token-intensive summarization workloads.                                                                                                                                                                                                                                                                                                                         | https://featherless.ai/register?referrer=HJUUTA6M                                                        |
| ## 3. Gemini-Embeddings-001 for Document Retrieval Embeddings: Notes the latest Google Gemini embedding model specifics, like output dimensionality tuning and task type for retrieval optimization, justifying use of HTTP API calls.                                                                                                                                                                                                                                                      | https://ai.google.dev/gemini-api/docs/embeddings                                                        |
| ## 3.1 Install Qdrant Community Node: Instructions on installing the Qdrant node in n8n and setting up collections and indexes.                                                                                                                                                                                                                                                                                                                                                             | http://api.qdrant.tech/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-qdrant.qdrant |
| ## 4. Create a Document Retrieval Subworkflow Tool: Advocates using subworkflow triggers for embedding generation due to API parameter requirements and reusability in MCP servers and external agents.                                                                                                                                                                                                                                                                                      | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflowtrigger                |
| ## Example Usage: The Highway Code Q&A Agent: Links to documentation about AI agents in n8n.                                                                                                                                                                                                                                                                                                                                                                                                | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent              |
| ## Example Usage: The Automated UK Driving Theory Test Taker: Demonstrates using the system to answer multiple choice questions automatically.                                                                                                                                                                                                                                                                                                                                           | https://www.npmjs.com/package/n8n-nodes-featherless                                                     |
| ## Example Usage: Highway Code MCP Server: Mentions use of MCP servers for scalable multi-agent query handling.                                                                                                                                                                                                                                                                                                                                                                            | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger/                      |
| ## Contextual Retrieval RAG for Less Using Featherless.AI and Gemini-Embeddings-001: Detailed explanation of how the workflow reduces token costs using Featherless.ai, the process steps, and customization suggestions.                                                                                                                                                                                                                                                                   | https://featherless.ai/?referrer=HJUUTA6M                                                              |
| Banner image for branding and visual identity.                                                                                                                                                                                                                                                                                                                                                                                                                                              | https://cdn.subworkflow.ai/n8n-templates/banner_595x311.png#full-width                                  |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, respecting all applicable content policies, containing no illegal, offensive, or protected material. All data processed is legal and public.

---