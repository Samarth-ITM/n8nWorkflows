RAG Starter Template using Simple Vector Stores, Form trigger and OpenAI

https://n8nworkflows.xyz/workflows/rag-starter-template-using-simple-vector-stores--form-trigger-and-openai-5010


# RAG Starter Template using Simple Vector Stores, Form trigger and OpenAI

---

### 1. Workflow Overview

This workflow is a Retrieval-Augmented Generation (RAG) starter template designed to demonstrate how to upload documents, transform them into vector embeddings, store them in a simple in-memory vector store, and then use these embeddings to answer user queries via chat. It integrates file ingestion, vector embedding creation, vector store management, and AI-driven chat query handling using OpenAI models.

The workflow is logically divided into two main blocks:

- **1.1 Load Data Flow:** Handles user file uploads, processes documents into embeddings, and inserts those embeddings into an in-memory vector store.
- **1.2 Retriever Flow:** Listens for chat messages, retrieves relevant document embeddings from the vector store as context, and uses an AI agent powered by OpenAI to generate answers.

---

### 2. Block-by-Block Analysis

#### 2.1 Load Data Flow

- **Overview:**  
  This block allows users to upload files (PDF or CSV), loads these files as documents, converts the text data into vector embeddings using OpenAI embeddings, then inserts these vectors into an in-memory vector store for later retrieval.

- **Nodes Involved:**  
  - Upload your file here (Form Trigger)  
  - Insert Data to Store (Vector Store In-Memory)  
  - Default Data Loader (Document Loader)  
  - Embeddings OpenAI (Embedding Node)  
  - Sticky Note (instructions and guidance)  
  - Sticky Note1 (section title)  
  - Sticky Note3 (embedding explanation)  

- **Node Details:**

  1. **Upload your file here**  
     - *Type:* Form Trigger  
     - *Role:* Entry point for users to upload files through a web form.  
     - *Configuration:* Accepts files with extensions `.pdf` and `.csv`. The form is titled "Upload your data to test RAG" with a single required file upload field.  
     - *Inputs:* Webhook trigger from form submission.  
     - *Outputs:* Sends uploaded file data downstream.  
     - *Failures:* Could fail if unsupported file types are uploaded or if no file is provided.  
     - *Version:* 2.2  

  2. **Default Data Loader**  
     - *Type:* Document Default Data Loader  
     - *Role:* Converts uploaded binary file data into document format suitable for embedding.  
     - *Configuration:* Processes binary data from upload.  
     - *Inputs:* Receives binary file data from "Insert Data to Store" node connection chain.  
     - *Outputs:* Outputs documents for embedding.  
     - *Failures:* Could fail with corrupt or unsupported file formats.  
     - *Version:* 1.1  

  3. **Embeddings OpenAI**  
     - *Type:* OpenAI Embeddings Node  
     - *Role:* Converts documents into vector embeddings using OpenAI's embedding models.  
     - *Configuration:* No additional options specified, uses credentials for OpenAI API. Credentials are linked to an OpenAI account.  
     - *Inputs:* Receives documents from Default Data Loader.  
     - *Outputs:* Provides vector embeddings for storage and retrieval.  
     - *Failures:* Possible failures include API key issues, rate limits, or incorrect data format.  
     - *Version:* 1.2  

  4. **Insert Data to Store**  
     - *Type:* Vector Store In-Memory Node  
     - *Role:* Inserts embeddings into an in-memory vector store under a key named "vector_store_key".  
     - *Configuration:* Mode is set to "insert."  
     - *Inputs:* Receives embeddings from Embeddings OpenAI node and document data from Default Data Loader. Also triggered by file upload.  
     - *Outputs:* Stores data for later retrieval.  
     - *Failures:* Memory limitations or corrupted data could cause failures.  
     - *Version:* 1.2  

  5. **Sticky Note**  
     - *Type:* Sticky Note  
     - *Role:* Provides user instructions including a quick start guide and a link to official documentation on RAG in n8n: https://docs.n8n.io/advanced-ai/rag-in-n8n/  
     - *Content:* Explains the workflow usage: load data then use chat retriever.  
     - *Version:* 1  

  6. **Sticky Note1**  
     - *Type:* Sticky Note  
     - *Role:* Marks the "📚 Load Data Flow" section visually.  
     - *Version:* 1  

  7. **Sticky Note3**  
     - *Type:* Sticky Note  
     - *Role:* Explains the importance of using the same embedding node for insert and retrieve operations to avoid inconsistency.  
     - *Version:* 1  

#### 2.2 Retriever Flow

- **Overview:**  
  This block listens for incoming chat messages, retrieves relevant vector embeddings from the in-memory store as context, and queries an AI agent powered by OpenAI's chat model to generate responses based on the knowledge base.

- **Nodes Involved:**  
  - When chat message received (Chat Trigger)  
  - AI Agent  
  - OpenAI Chat Model  
  - Query Data Tool (Vector Store In-Memory)  
  - Sticky Note2 (section title)  

- **Node Details:**

  1. **When chat message received**  
     - *Type:* Chat Trigger  
     - *Role:* Webhook that listens for incoming chat messages to initiate the retrieval and response process.  
     - *Configuration:* No specific options configured; uses webhook ID to receive chat messages.  
     - *Inputs:* External chat message trigger.  
     - *Outputs:* Starts AI Agent node.  
     - *Failures:* Could fail if webhook is unreachable or if message format is invalid.  
     - *Version:* 1.1  

  2. **Query Data Tool**  
     - *Type:* Vector Store In-Memory Node  
     - *Role:* Retrieves relevant embeddings from the vector store as a tool named "knowledge_base" to provide context for answering queries.  
     - *Configuration:* Mode is "retrieve-as-tool," toolName is "knowledge_base," with description guiding usage to answer user questions. Uses the same "vector_store_key" memory key used in the Load Data Flow.  
     - *Inputs:* Receives embedding requests from AI Agent.  
     - *Outputs:* Provides retrieved vectors to AI Agent.  
     - *Failures:* Could fail if vector store is empty or corrupted.  
     - *Version:* 1.2  

  3. **AI Agent**  
     - *Type:* Langchain Agent Node  
     - *Role:* Orchestrates the retrieval and language model to generate answers based on context from the vector store.  
     - *Configuration:* Default options, connected to Query Data Tool and OpenAI Chat Model nodes as a tool and language model respectively.  
     - *Inputs:* Triggered by chat message, uses retrieved context and language model to produce response.  
     - *Outputs:* Final chat response sent back to user.  
     - *Failures:* Could fail on API issues, malformed input, or retrieval errors.  
     - *Version:* 2  

  4. **OpenAI Chat Model**  
     - *Type:* OpenAI Chat Language Model Node  
     - *Role:* Provides the underlying GPT-4o-mini model to generate chat completions for the AI Agent.  
     - *Configuration:* Model set explicitly to "gpt-4o-mini".  
     - *Inputs:* Receives prompts from AI Agent.  
     - *Outputs:* Returns generated chat completions to AI Agent.  
     - *Failures:* API key issues, rate limits, or model availability could cause failure.  
     - *Version:* 1.2  

  5. **Sticky Note2**  
     - *Type:* Sticky Note  
     - *Role:* Visually marks the "🐕 2. Retriever Flow" section.  
     - *Version:* 1  

---

### 3. Summary Table

| Node Name              | Node Type                                | Functional Role                        | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                                                                                                    |
|------------------------|----------------------------------------|-------------------------------------|----------------------------|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Upload your file here   | Form Trigger                           | Entry point for file upload          | (Webhook trigger)           | Insert Data to Store      |                                                                                                                                                                               |
| Default Data Loader     | Document Default Data Loader           | Loads uploaded file into documents   | Insert Data to Store (via chain) | Insert Data to Store      |                                                                                                                                                                               |
| Embeddings OpenAI      | OpenAI Embeddings                      | Converts documents to vector embeddings | Default Data Loader         | Insert Data to Store, Query Data Tool | See Sticky Note3: "The Insert and Retrieve operation use the same embedding node to ensure embeddings consistency."                                                          |
| Insert Data to Store    | Vector Store In-Memory                 | Inserts embeddings into vector store | Upload your file here, Default Data Loader, Embeddings OpenAI |                            |                                                                                                                                                                               |
| Query Data Tool         | Vector Store In-Memory                 | Retrieves embeddings as a tool for AI Agent | AI Agent                   | AI Agent                 |                                                                                                                                                                               |
| AI Agent               | Langchain Agent                       | Coordinates retrieval and language model | When chat message received, Query Data Tool, OpenAI Chat Model |                          |                                                                                                                                                                               |
| When chat message received | Chat Trigger                         | Entry point for chat messages        | (Webhook trigger)           | AI Agent                 |                                                                                                                                                                               |
| OpenAI Chat Model       | OpenAI Chat Language Model            | Provides GPT-4o-mini chat completions | AI Agent                   | AI Agent                 |                                                                                                                                                                               |
| Sticky Note            | Sticky Note                          | Instructions and documentation       |                            |                          | See Sticky Note: Quick start guide and link to https://docs.n8n.io/advanced-ai/rag-in-n8n/                                                                                      |
| Sticky Note1           | Sticky Note                          | Section title: Load Data Flow        |                            |                          |                                                                                                                                                                               |
| Sticky Note2           | Sticky Note                          | Section title: Retriever Flow        |                            |                          |                                                                                                                                                                               |
| Sticky Note3           | Sticky Note                          | Embeddings usage explanation         |                            |                          | See Sticky Note3: Embeddings must be consistent between insert and retrieval.                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node:**  
   - Name: `Upload your file here`  
   - Configure form title: "Upload your data to test RAG"  
   - Add a required file upload field accepting `.pdf` and `.csv` files.  
   - Save webhook URL for external triggering.

3. **Add a Vector Store In-Memory node:**  
   - Name: `Insert Data to Store`  
   - Set mode to `insert`.  
   - Set `memoryKey` to `"vector_store_key"` (list mode).  
   - Connect this node’s input to the Form Trigger node output.

4. **Add a Document Default Data Loader node:**  
   - Name: `Default Data Loader`  
   - Set `dataType` to `binary`.  
   - Connect its input to the output of `Insert Data to Store` node (or directly from Form Trigger if preferred).  
   - Connect its output back into the `Insert Data to Store` node to provide documents for embedding.

5. **Add an OpenAI Embeddings node:**  
   - Name: `Embeddings OpenAI`  
   - Use OpenAI credentials (set up your OpenAI API key in Credentials).  
   - No additional options needed.  
   - Connect the input from the `Default Data Loader` node output.  
   - Connect the output to the `Insert Data to Store` node’s embedding input.  

6. **Add another Vector Store In-Memory node:**  
   - Name: `Query Data Tool`  
   - Set mode to `retrieve-as-tool`.  
   - Set tool name to `"knowledge_base"` and description to `"Use this knowledge base to answer questions from the user"`.  
   - Use the same `memoryKey` `"vector_store_key"`.  

7. **Add a Chat Trigger node:**  
   - Name: `When chat message received`  
   - This node listens for incoming chat messages (webhook).  
   - Save its webhook URL for chat clients.

8. **Add an OpenAI Chat Model node:**  
   - Name: `OpenAI Chat Model`  
   - Set model to `"gpt-4o-mini"` (or any preferred OpenAI chat model).  
   - Use OpenAI credentials.

9. **Add an AI Agent node:**  
   - Name: `AI Agent`  
   - Connect the Chat Trigger node as input to AI Agent.  
   - Configure the AI Agent to use:  
     - Language Model: Connect to `OpenAI Chat Model` node.  
     - AI Tool: Connect to `Query Data Tool` node.

10. **Connect outputs:**  
    - `Query Data Tool` output → `AI Agent` tool input.  
    - `OpenAI Chat Model` output → `AI Agent` language model input.  
    - `AI Agent` output returns the chat response.

11. **Add Sticky Notes (optional for clarity):**  
    - Add notes for instructions and section titles as per overview.

12. **Configure credentials:**  
    - Add OpenAI API credentials in n8n credentials manager for both Embeddings and Chat Model nodes.  
    - Ensure OAuth2 or API key credentials are valid and have sufficient quota.

13. **Test the workflow:**  
    - Upload a PDF or CSV file via the form trigger URL.  
    - Once data is inserted, send chat messages to the chat trigger URL to query the AI agent.  
    - Confirm responses are generated based on uploaded document content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                    | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Quick start guide and detailed documentation for RAG workflows in n8n.                                                                                        | https://docs.n8n.io/advanced-ai/rag-in-n8n/                                                             |
| Important to use the same embeddings node for both insert and retrieve operations to ensure consistent vector representations and avoid errors.               | Sticky Note3 in workflow                                                                               |
| This workflow uses an in-memory vector store which is suitable for demonstration and testing but may not be persistent or scalable for production scenarios.   | General consideration                                                                                    |

---

Disclaimer: The provided text is exclusively generated from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---