Document Q&A System with OpenAI GPT, Pinecone Vector DB & Google Drive Integration

https://n8nworkflows.xyz/workflows/document-q-a-system-with-openai-gpt--pinecone-vector-db---google-drive-integration-5807


# Document Q&A System with OpenAI GPT, Pinecone Vector DB & Google Drive Integration

---

## 1. Workflow Overview

This n8n workflow implements a **Document Question & Answer (Q&A) System** leveraging a combination of Google Drive, Pinecone Vector Database, OpenAI GPT models, and a webhook-based user interface integration. The primary use case is to enable users to query contract and agreement documents stored in Google Drive through natural language questions, obtaining AI-assisted, context-aware answers based on the document content.

The workflow logically splits into three main functional blocks:

**1.1 Document Loading and Vectorization**  
- Downloads contract documents from a specified Google Drive folder.  
- Processes documents by splitting text into manageable chunks and generating vector embeddings using OpenAI.  
- Stores the vector data into a Pinecone vector database for semantic search.

**1.2 Chat Query Handling via Chat Trigger (Test Interface)**  
- Listens for incoming chat queries via a chat trigger node.  
- Uses an AI Agent configured as a legal and contracting expert to search the Pinecone vector database and generate relevant answers.  
- Employs OpenAI GPT chat models and memory buffer for conversational context.

**1.3 Webhook-Based User Interface Query (Lovable UI Integration)**  
- Provides an HTTP POST webhook to receive queries from a web UI form (Lovable).  
- Processes queries with an AI Agent that consults the Pinecone database and responds via the webhook response node.  
- Supports form inputs for user details and freeform query, returning professional, polite answers.

---

## 2. Block-by-Block Analysis

### 2.1 Document Loading and Vectorization

**Overview:**  
This block automates the ingestion of contract documents from Google Drive, downloading them, splitting text content into chunks, generating embeddings with OpenAI, and storing these vectors into Pinecone for semantic search.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Google Drive (Folder list)  
- Google Drive1 (File download)  
- Default Data Loader  
- Recursive Character Text Splitter  
- Embeddings OpenAI  
- Pinecone Vector Store  
- Sticky Note (for documentation)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Initiates the document loading process on demand.  
  - *Inputs:* None  
  - *Outputs:* Connects to Google Drive node  
  - *Edge cases:* Manual action required; no automated scheduling.

- **Google Drive**  
  - *Type:* Google Drive node (List files/folders)  
  - *Role:* Lists files in a specified Google Drive folder (`folderId = 1NgITWoqBgLAVof9bxF0jIrVToQ9c919u`) containing contract documents.  
  - *Credentials:* Google OAuth2 required.  
  - *Input:* Manual trigger  
  - *Output:* File metadata list (IDs) to Google Drive1 node  
  - *Errors:* Auth failure, folder not found, API quota exceeded.

- **Google Drive1**  
  - *Type:* Google Drive node (Download file)  
  - *Role:* Downloads the actual file content based on file ID from previous node.  
  - *Credentials:* Same as above.  
  - *Input:* File metadata from Google Drive node  
  - *Output:* Binary document data to Pinecone Vector Store node  
  - *Errors:* File not found, permission denied, network issues.

- **Default Data Loader**  
  - *Type:* Langchain Document Data Loader  
  - *Role:* Converts raw binary document content into processed text format for vectorization.  
  - *Settings:* Input data type set to binary, custom text splitting mode.  
  - *Input:* Binary data from Google Drive1  
  - *Output:* Structured document text to Recursive Character Text Splitter  
  - *Errors:* Unsupported file format, corrupted files.

- **Recursive Character Text Splitter**  
  - *Type:* Langchain Text Splitter  
  - *Role:* Splits document text into overlapping chunks with chunk overlap set to 100 characters, enabling better context for embeddings.  
  - *Input:* Document text from Default Data Loader  
  - *Output:* Text chunks to Embeddings OpenAI  
  - *Errors:* Text encoding issues, very large documents.

- **Embeddings OpenAI**  
  - *Type:* OpenAI Embeddings node  
  - *Role:* Generates vector embeddings from text chunks using OpenAI’s embedding model (presumably text-embedding-3-small).  
  - *Credentials:* OpenAI API key required.  
  - *Input:* Text chunks from Recursive Character Text Splitter  
  - *Output:* Embeddings to Pinecone Vector Store  
  - *Errors:* API rate limit, invalid API key.

- **Pinecone Vector Store**  
  - *Type:* Pinecone Vector DB node (Insert mode)  
  - *Role:* Inserts vector embeddings into Pinecone index named `package1536` for later semantic search.  
  - *Credentials:* Pinecone API key required.  
  - *Input:* Embeddings from Embeddings OpenAI  
  - *Output:* Confirmation/status output  
  - *Errors:* Pinecone connection failure, index not found, quota exceeded.

- **Sticky Note (Document Loading Description)**  
  - *Content:* Explains the process of loading documents from Google Drive, vectorizing, and storing in Pinecone.  
  - *Role:* Documentation and guidance.

---

### 2.2 Chat Query Handling via Chat Trigger (Test Interface)

**Overview:**  
This block handles natural language queries sent to a chat trigger node, processes them with an AI Agent configured as a legal advisor, and returns responses based on Pinecone vector search results augmented by OpenAI chat models.

**Nodes Involved:**  
- When chat message received (Chat Trigger)  
- AI Agent  
- Simple Memory  
- OpenAI Chat Model  
- Answer questions with a vector store  
- Pinecone Vector Store1  
- Embeddings OpenAI1  
- Sticky Note1 (Documentation)

**Node Details:**

- **When chat message received**  
  - *Type:* Langchain Chat Trigger  
  - *Role:* Webhook-based entry point for chat messages (test or internal interface).  
  - *Input:* Incoming chat messages  
  - *Output:* Queries forwarded to AI Agent  
  - *Errors:* Webhook access issues, message format errors.

- **AI Agent**  
  - *Type:* Langchain AI Agent node  
  - *Role:* Processes chat queries with a system prompt defining a role as a contracting/legal adviser specializing in shipping and forwarding contracts.  
  - *Configuration:*  
    - System message instructs to answer only using Pinecone vector DB data.  
    - Polite and professional tone with optional emojis.  
  - *Input:* Chat messages from trigger  
  - *Output:* To OpenAI Chat Model and the vector store tool node  
  - *Errors:* Model errors, prompt misconfiguration.

- **Simple Memory**  
  - *Type:* Memory Buffer Window  
  - *Role:* Maintains conversational context history for AI Agent to improve dialogue coherence.  
  - *Input/Output:* Linked to AI Agent node  
  - *Errors:* Memory overflow or data loss.

- **OpenAI Chat Model**  
  - *Type:* OpenAI GPT Chat Node  
  - *Role:* Generates natural language responses based on AI Agent inputs.  
  - *Model:* `gpt-4.1-mini`  
  - *Credentials:* OpenAI API key  
  - *Errors:* API rate limits, model unavailability.

- **Answer questions with a vector store**  
  - *Type:* Vector Store Tool Node  
  - *Role:* Uses Pinecone vector DB to analyze queries and retrieve relevant document chunks to inform AI Agent answers.  
  - *Input:* From AI Agent and OpenAI Chat Model  
  - *Output:* Feeds back to AI Agent for answer generation  
  - *Errors:* Vector store connectivity, empty results.

- **Pinecone Vector Store1**  
  - *Type:* Pinecone Vector DB node (Query mode)  
  - *Role:* Performs vector similarity search on index `package1536` to find relevant document text chunks.  
  - *Input:* Queries from the tool vector store node  
  - *Credentials:* Pinecone API key  
  - *Errors:* Connectivity, invalid index.

- **Embeddings OpenAI1**  
  - *Type:* OpenAI Embeddings  
  - *Role:* Generates query embeddings to perform vector similarity search.  
  - *Credentials:* OpenAI API  
  - *Errors:* API failures.

- **Sticky Note1**  
  - *Content:* Mentions this flow node block handles document queries via chat for testing.  
  - *Role:* Informational only.

---

### 2.3 Webhook-Based User Interface Query (Lovable UI Integration)

**Overview:**  
This block exposes an HTTP webhook endpoint to receive user queries from a Lovable web form UI, processes queries through an AI Agent (similar to the chat trigger flow), and responds directly via the webhook response node.

**Nodes Involved:**  
- Webhook (HTTP POST endpoint)  
- AI Agent1  
- OpenAI Chat Model2  
- Simple Memory1 (disabled)  
- Answer questions with a vector store1  
- Pinecone Vector Store2  
- Embeddings OpenAI2  
- OpenAI Chat Model3  
- Respond to Webhook  
- Sticky Note2, Sticky Note5 (Documentation)

**Node Details:**

- **Webhook**  
  - *Type:* HTTP Webhook (POST)  
  - *Role:* Entry point for Lovable UI form submissions.  
  - *Endpoint Path:* `12b44ee5-c43e-430c-a1d4-4fc5ff5e45c4`  
  - *Response Mode:* Uses Respond to Webhook node to return answers.  
  - *Input:* JSON body with user query and form data  
  - *Output:* Passes query to AI Agent1  
  - *Errors:* HTTP errors, webhook security.

- **AI Agent1**  
  - *Type:* Langchain AI Agent  
  - *Role:* Processes Lovable UI queries with the same contracting/legal expert system message as AI Agent in Chat flow.  
  - *Configuration:*  
    - Query text extracted from webhook JSON body (`{{ $json.body.query }}`).  
    - System message emphasizes only answering from Pinecone data.  
  - *Input:* Query from Webhook node  
  - *Output:* To OpenAI Chat Model2 and vector store tool node  
  - *Errors:* Parsing errors, model issues.

- **OpenAI Chat Model2**  
  - *Type:* OpenAI GPT Chat Model  
  - *Role:* Generates conversational response based on AI Agent1 output.  
  - *Model:* `gpt-4.1-mini`  
  - *Credentials:* OpenAI API key  
  - *Errors:* API rate limits.

- **Simple Memory1**  
  - *Type:* Memory Buffer Window  
  - *Role:* Disabled in this flow; no conversational history stored.  
  - *Errors:* None operational.

- **Answer questions with a vector store1**  
  - *Type:* Vector Store Tool Node  
  - *Role:* Performs vector similarity query on Pinecone DB to find relevant document context for answering queries.  
  - *Input:* From AI Agent1 and OpenAI Chat Model2  
  - *Output:* Back to AI Agent1 for final response  
  - *Errors:* Connectivity, empty results.

- **Pinecone Vector Store2**  
  - *Type:* Pinecone Vector DB query node  
  - *Role:* Executes semantic search on the same Pinecone index `package1536`.  
  - *Credentials:* Pinecone API key  
  - *Errors:* Index issues.

- **Embeddings OpenAI2**  
  - *Type:* OpenAI Embeddings node  
  - *Role:* Embeds queries for vector search.  
  - *Credentials:* OpenAI API key  
  - *Errors:* API quota.

- **OpenAI Chat Model3**  
  - *Type:* OpenAI GPT Chat Model  
  - *Role:* Generates final language output.  
  - *Model:* `gpt-4.1-mini`  
  - *Credentials:* OpenAI API key

- **Respond to Webhook**  
  - *Type:* Respond to Webhook node  
  - *Role:* Returns AI Agent1’s generated answer as the HTTP response to the Lovable UI form submission.  
  - *Input:* Response JSON from AI Agent1  
  - *Output:* HTTP response sent to client  
  - *Errors:* Timeout, malformed response.

- **Sticky Note2 & Sticky Note5**  
  - *Content:* Detailed documentation about the webhook integration with Lovable UI, its form inputs, and use cases such as legal contract querying, procurement QA, and customer support automation.  
  - *Role:* Informational and architectural guidance.

---

## 3. Summary Table

| Node Name                         | Node Type                                      | Functional Role                                  | Input Node(s)             | Output Node(s)                     | Sticky Note                                                                                 |
|----------------------------------|------------------------------------------------|-------------------------------------------------|---------------------------|----------------------------------|---------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                                 | Starts document loading process                  | None                      | Google Drive                     |                                                                                             |
| Google Drive                     | Google Drive (List folder)                      | Lists contract documents in Google Drive folder | When clicking ‘Execute workflow’ | Google Drive1                   |                                                                                             |
| Google Drive1                    | Google Drive (Download file)                    | Downloads contract files                          | Google Drive              | Pinecone Vector Store            |                                                                                             |
| Default Data Loader              | Document Data Loader                            | Loads and prepares document text                  | Recursive Character Text Splitter (reverse connection) | Pinecone Vector Store           |                                                                                             |
| Recursive Character Text Splitter| Text Splitter                                  | Splits text into overlapping chunks               | Default Data Loader       | Embeddings OpenAI                |                                                                                             |
| Embeddings OpenAI               | OpenAI Embeddings                              | Generates vector embeddings                        | Recursive Character Text Splitter | Pinecone Vector Store           |                                                                                             |
| Pinecone Vector Store           | Pinecone Vector DB (Insert)                     | Inserts vectors into Pinecone DB                   | Embeddings OpenAI         | None                           |                                                                                             |
| Sticky Note                    | Sticky Note                                    | Documents document loading flow                    | None                      | None                           | Explains document loading and vectorization process                                         |
| When chat message received      | Langchain Chat Trigger                         | Entry for chat queries                             | None                      | AI Agent                       |                                                                                             |
| AI Agent                      | Langchain AI Agent                             | Processes chat queries as legal adviser           | When chat message received | OpenAI Chat Model, Answer questions with a vector store |                                                                                             |
| Simple Memory                 | Memory Buffer Window                           | Maintains conversational context                  | AI Agent                  | AI Agent                       |                                                                                             |
| OpenAI Chat Model             | OpenAI GPT Chat Model                          | Generates natural language responses              | AI Agent                  | AI Agent                       |                                                                                             |
| Answer questions with a vector store | Vector Store Tool                            | Queries vector DB for relevant document chunks    | AI Agent, OpenAI Chat Model| AI Agent                       |                                                                                             |
| Pinecone Vector Store1        | Pinecone Vector DB (Query)                      | Performs vector similarity search                  | Answer questions with vector store | Answer questions with vector store |                                                                                             |
| Embeddings OpenAI1            | OpenAI Embeddings                              | Embeds user queries for vector search             | Pinecone Vector Store1    | Answer questions with a vector store |                                                                                             |
| Sticky Note1                  | Sticky Note                                    | Documents chat query flow                          | None                      | None                           | Mentions chat-based query testing                                                         |
| Webhook                      | HTTP Webhook (POST)                            | Receives queries from Lovable UI                   | None                      | AI Agent1                     |                                                                                             |
| AI Agent1                    | Langchain AI Agent                             | Processes Lovable UI queries as legal adviser     | Webhook                   | OpenAI Chat Model2, Answer questions with vector store1 |                                                                                             |
| OpenAI Chat Model2           | OpenAI GPT Chat Model                          | Generates chatbot response for Lovable queries    | AI Agent1                 | AI Agent1                     |                                                                                             |
| Simple Memory1              | Memory Buffer Window (disabled)                | Disabled conversational memory for Lovable flow  | None                      | None                         |                                                                                             |
| Answer questions with a vector store1 | Vector Store Tool                            | Queries Pinecone vector DB for Lovable UI         | AI Agent1, OpenAI Chat Model2 | AI Agent1                   |                                                                                             |
| Pinecone Vector Store2       | Pinecone Vector DB (Query)                      | Performs vector search on Pinecone                 | Answer questions with vector store1 | Answer questions with vector store1 |                                                                                             |
| Embeddings OpenAI2          | OpenAI Embeddings                              | Embeds Lovable UI queries for vector search       | Pinecone Vector Store2    | Answer questions with vector store1 |                                                                                             |
| OpenAI Chat Model3          | OpenAI GPT Chat Model                          | Final language model for Lovable UI response      | Answer questions with vector store1 | Answer questions with vector store1 |                                                                                             |
| Respond to Webhook          | Respond to Webhook                             | Sends AI Agent1's answer back to Lovable UI       | AI Agent1                 | None                         |                                                                                             |
| Sticky Note2                | Sticky Note                                    | Documents webhook integration with Lovable UI     | None                      | None                         | Explains webhook and Lovable UI integration                                                |
| Sticky Note3                | Sticky Note                                    | Project overview and RAG system explanation        | None                      | None                         | Detailed project description and architecture                                              |
| Sticky Note4                | Sticky Note                                    | Documents chat query flow architecture             | None                      | None                         | Explains chat query flow and components                                                   |
| Sticky Note5                | Sticky Note                                    | Explains Lovable UI webhook flow and use cases     | None                      | None                         | Detailed explanation of Lovable UI integration and business use cases                     |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: Manually start document ingestion.

2. **Add Google Drive Node (List Folder)**  
   - Name: `Google Drive`  
   - Resource: File/Folder  
   - Operation: List  
   - Filter: Folder ID set to `1NgITWoqBgLAVof9bxF0jIrVToQ9c919u` (Contract documents folder)  
   - Credentials: Configure Google Drive OAuth2 credentials with required scopes.

3. **Add Google Drive Node (Download File)**  
   - Name: `Google Drive1`  
   - Operation: Download  
   - File ID: Set dynamically from previous node’s output (`={{ $json.id }}`)  
   - Credentials: Same as above.

4. **Add Default Data Loader Node**  
   - Name: `Default Data Loader`  
   - Data Type: Binary  
   - Text Splitting Mode: Custom (default)  
   - Input: Connect from `Google Drive1`.

5. **Add Recursive Character Text Splitter Node**  
   - Name: `Recursive Character Text Splitter`  
   - Chunk Overlap: 100 characters  
   - Input: Connect from `Default Data Loader`.

6. **Add OpenAI Embeddings Node**  
   - Name: `Embeddings OpenAI`  
   - Model: Use OpenAI’s embedding model (e.g., `text-embedding-3-small`)  
   - Credentials: Configure OpenAI API credentials.

7. **Add Pinecone Vector Store Node (Insert Mode)**  
   - Name: `Pinecone Vector Store`  
   - Mode: Insert  
   - Pinecone Index: `package1536` (create this index in Pinecone dashboard if not exists)  
   - Credentials: Configure Pinecone API credentials.

8. **Wire nodes in order:**  
   Manual Trigger → Google Drive → Google Drive1 → Default Data Loader → Recursive Character Text Splitter → Embeddings OpenAI → Pinecone Vector Store

9. **Set up Chat Query Handling Flow:**  
   - Add `When chat message received` (Langchain Chat Trigger node)  
   - Add `AI Agent` node configured with system message role and instructions as per legal contracting adviser.  
   - Add `Simple Memory` node connected as AI memory to AI Agent.  
   - Add `OpenAI Chat Model` node (model: `gpt-4.1-mini`) connected as AI language model to AI Agent.  
   - Add `Answer questions with a vector store` node connected as AI tool to AI Agent.  
   - Add `Pinecone Vector Store1` node configured for querying `package1536`, connected as vector store to the tool node.  
   - Add `Embeddings OpenAI1` node connected as AI embedding to the Pinecone query node.  
   - Connect nodes as: Chat Trigger → AI Agent → OpenAI Chat Model, Simple Memory, Answer questions with vector store → Pinecone Vector Store1 → Embeddings OpenAI1

10. **Set up Webhook-Based UI Query Flow:**  
    - Add `Webhook` node (POST method, path e.g. `12b44ee5-c43e-430c-a1d4-4fc5ff5e45c4`).  
    - Add `AI Agent1` node with the same legal adviser system message, query text from `{{ $json.body.query }}`.  
    - Add `OpenAI Chat Model2` node (model: `gpt-4.1-mini`).  
    - Add `Answer questions with a vector store1` node.  
    - Add `Pinecone Vector Store2` node for querying `package1536`.  
    - Add `Embeddings OpenAI2` node for query embeddings.  
    - Add `OpenAI Chat Model3` node for final language generation.  
    - Add `Respond to Webhook` node configured to respond with AI Agent1’s output (`={{ $json.output }}`).  
    - Connect nodes as: Webhook → AI Agent1 → OpenAI Chat Model2 & Answer questions with vector store1 → Pinecone Vector Store2 → Embeddings OpenAI2 → OpenAI Chat Model3 → Answer questions with vector store1 → AI Agent1 → Respond to Webhook

11. **Credential Setup:**  
    - Configure Google Drive OAuth2 credentials with access to target folder/files.  
    - Configure OpenAI API credentials with permissions for embeddings and chat completions.  
    - Configure Pinecone API credentials linked to the `package1536` index.

12. **Test the full workflow:**  
    - Trigger manual document ingestion.  
    - Use Chat Trigger node to test chat queries internally.  
    - Use the webhook URL for Lovable UI form submissions to test external queries.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                | Context or Link                                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This project demonstrates a Retrieval-Augmented Generation (RAG) system combining Google Drive, OpenAI embeddings, Pinecone vector DB, and AI Agents.       | Comprehensive project overview available in Sticky Note3 node content.                                             |
| Lovable UI form integration enables users to submit contract queries with personal details, allowing legal, HR, procurement, and customer support use cases.| Detailed explanation in Sticky Note5, describing webhook setup and use cases.                                      |
| Chat queries preserve conversational context through Simple Memory buffers to improve answer relevance and coherence.                                       | Explained in Sticky Note4 node content.                                                                             |
| Pinecone index `package1536` must be created and available, with region `us-east-1` as per project setup.                                                    | Pinecone setup instructions implicit in Pinecone Vector Store nodes configuration.                                  |
| OpenAI GPT model `gpt-4.1-mini` is used for chat completions; update accordingly if newer models become available.                                          | Model version specified in OpenAI Chat Model nodes.                                                                 |
| Ensure API keys for Google Drive, OpenAI, and Pinecone are valid and have sufficient quota to avoid runtime errors.                                         | Common integration failure points across nodes.                                                                     |
| For more on Lovable UI form builder: https://lovable.ai (external resource, not included in workflow)                                                        | External UI integration reference.                                                                                   |

---

**Disclaimer:**  
The provided content is exclusively derived from an n8n automation workflow. It complies with all applicable content policies and contains no illegal, offensive, or protected materials. All processed data is legal and publicly accessible.

---