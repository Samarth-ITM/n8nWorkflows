Chat with GitHub API Documentation: RAG-Powered Chatbot with Pinecone & OpenAI

https://n8nworkflows.xyz/workflows/chat-with-github-api-documentation--rag-powered-chatbot-with-pinecone---openai-2705


# Chat with GitHub API Documentation: RAG-Powered Chatbot with Pinecone & OpenAI

### 1. Workflow Overview

This workflow implements a Retrieval Augmented Generation (RAG) chatbot that enables conversational querying of the GitHub API OpenAPI specification. It is designed to provide context-aware, accurate answers about the GitHub API by combining vector search with large language models (LLMs). The workflow is structured into two main logical blocks:

**1.1 Data Ingestion and Indexing**  
- Fetches the full GitHub OpenAPI 3 specification JSON from GitHub’s repository.  
- Splits the large specification document into smaller text chunks.  
- Generates vector embeddings for each chunk using OpenAI embedding models.  
- Stores these embeddings and their associated text chunks in a Pinecone vector database index.

**1.2 Chat Interface and Query Processing**  
- Listens for incoming chat messages via a webhook trigger.  
- Generates an embedding for the user’s query.  
- Queries Pinecone to retrieve the most relevant chunks from the indexed API spec.  
- Uses OpenAI’s GPT-4o-mini model to generate a concise, context-aware response based on the retrieved chunks and the user query.  
- Maintains conversation context with a window buffer memory to improve multi-turn interactions.

This design allows easy adaptation to any OpenAPI specification or other large textual knowledge bases, enabling companies to build internal or public API documentation chatbots.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion and Indexing

**Overview:**  
This block is responsible for acquiring the GitHub API specification, processing it into manageable pieces, generating embeddings, and storing them in Pinecone for later retrieval.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- HTTP Request  
- Recursive Character Text Splitter  
- Default Data Loader  
- Generate Embeddings  
- Pinecone Vector Store

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the ingestion process manually.  
  - Configuration: No parameters; triggers workflow execution on demand.  
  - Inputs: None  
  - Outputs: HTTP Request node  
  - Failures: None expected; manual trigger.

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Fetches the raw GitHub OpenAPI specification JSON from the official GitHub repository URL.  
  - Configuration: URL set to `https://raw.githubusercontent.com/github/rest-api-description/refs/heads/main/descriptions/api.github.com/api.github.com.json`  
  - Inputs: Manual Trigger  
  - Outputs: Pinecone Vector Store (insert mode) via downstream nodes  
  - Failures: Network errors, HTTP 404 if URL changes, or rate limiting.

- **Recursive Character Text Splitter**  
  - Type: Text Splitter (Recursive Character)  
  - Role: Splits the large JSON text into smaller chunks suitable for embedding generation.  
  - Configuration: Default options (likely chunk size and overlap)  
  - Inputs: Default Data Loader  
  - Outputs: Default Data Loader  
  - Failures: Improper chunking if input format changes.

- **Default Data Loader**  
  - Type: Document Loader  
  - Role: Prepares documents (text chunks) for embedding generation.  
  - Configuration: Default options  
  - Inputs: Recursive Character Text Splitter  
  - Outputs: Generate Embeddings  
  - Failures: Data format issues.

- **Generate Embeddings**  
  - Type: OpenAI Embeddings  
  - Role: Generates vector embeddings for each text chunk using OpenAI embedding models.  
  - Configuration: Uses OpenAI credentials; default embedding model.  
  - Inputs: Default Data Loader  
  - Outputs: Pinecone Vector Store (insert mode)  
  - Failures: API key errors, rate limits, or model unavailability.

- **Pinecone Vector Store**  
  - Type: Pinecone Vector Store (Insert Mode)  
  - Role: Inserts the generated embeddings and associated text chunks into the Pinecone index named "n8n-demo".  
  - Configuration: Pinecone credentials configured; index "n8n-demo" selected.  
  - Inputs: Generate Embeddings, HTTP Request (via Default Data Loader)  
  - Outputs: None (end of ingestion chain)  
  - Failures: Authentication errors, index not found, network issues.

---

#### 2.2 Chat Interface and Query Processing

**Overview:**  
This block handles user chat input, generates query embeddings, performs semantic search on Pinecone, and generates responses using OpenAI’s GPT-4o-mini model. It also manages conversation memory for context retention.

**Nodes Involved:**  
- When chat message received (Chat Trigger)  
- AI Agent  
- OpenAI Chat Model  
- Window Buffer Memory  
- Vector Store Tool  
- OpenAI Chat Model1  
- Generate User Query Embedding  
- Pinecone Vector Store (Querying)

**Node Details:**

- **When chat message received**  
  - Type: Chat Trigger (Webhook)  
  - Role: Listens for incoming chat messages from users to initiate query processing.  
  - Configuration: Webhook ID set; no additional parameters.  
  - Inputs: External chat messages  
  - Outputs: AI Agent  
  - Failures: Webhook misconfiguration, network issues.

- **AI Agent**  
  - Type: Langchain Agent  
  - Role: Orchestrates the chat logic, integrating memory, tools, and language models.  
  - Configuration: System message set to instruct the agent to provide helpful GitHub API info based on OpenAPI specs.  
  - Inputs: Chat Trigger, OpenAI Chat Model, Vector Store Tool, Window Buffer Memory  
  - Outputs: OpenAI Chat Model1  
  - Failures: Misconfiguration of tools or memory, expression errors.

- **OpenAI Chat Model**  
  - Type: OpenAI Chat Completion  
  - Role: Provides the language model for the AI Agent to generate responses.  
  - Configuration: Uses OpenAI credentials; default GPT model parameters.  
  - Inputs: AI Agent (as language model)  
  - Outputs: AI Agent  
  - Failures: API key issues, rate limits.

- **Window Buffer Memory**  
  - Type: Memory Buffer (Window)  
  - Role: Maintains recent conversation history to provide context for multi-turn dialogue.  
  - Configuration: Default window size.  
  - Inputs: AI Agent  
  - Outputs: AI Agent  
  - Failures: Memory overflow or mismanagement.

- **Vector Store Tool**  
  - Type: Vector Store Tool  
  - Role: Provides the AI Agent with access to the Pinecone vector store for semantic search.  
  - Configuration: Named "GitHub_OpenAPI_Specification" with description indicating it contains OpenAPI v3 specs.  
  - Inputs: Pinecone Vector Store (Querying)  
  - Outputs: AI Agent  
  - Failures: Tool misconfiguration.

- **OpenAI Chat Model1**  
  - Type: OpenAI Chat Completion  
  - Role: Generates final responses based on retrieved context and user query.  
  - Configuration: Uses OpenAI credentials; GPT-4o-mini model.  
  - Inputs: AI Agent  
  - Outputs: Vector Store Tool  
  - Failures: API errors.

- **Generate User Query Embedding**  
  - Type: OpenAI Embeddings  
  - Role: Generates an embedding vector for the user’s query to perform semantic search.  
  - Configuration: Uses OpenAI credentials; default embedding model.  
  - Inputs: AI Agent (user query)  
  - Outputs: Pinecone Vector Store (Querying)  
  - Failures: API key errors, rate limits.

- **Pinecone Vector Store (Querying)**  
  - Type: Pinecone Vector Store (Query Mode)  
  - Role: Queries the Pinecone index "n8n-demo" to find relevant document chunks matching the user query embedding.  
  - Configuration: Pinecone credentials; index "n8n-demo".  
  - Inputs: Generate User Query Embedding  
  - Outputs: Vector Store Tool  
  - Failures: Authentication errors, index not found.

---

### 3. Summary Table

| Node Name                      | Node Type                                | Functional Role                              | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                                            |
|--------------------------------|-----------------------------------------|----------------------------------------------|----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger                          | Starts ingestion process                      | None                             | HTTP Request                    |                                                                                                                        |
| HTTP Request                   | HTTP Request                           | Fetches GitHub OpenAPI spec JSON              | When clicking ‘Test workflow’    | Pinecone Vector Store           |                                                                                                                        |
| Recursive Character Text Splitter | Text Splitter (Recursive Character)   | Splits large spec into chunks                  | Default Data Loader              | Default Data Loader             |                                                                                                                        |
| Default Data Loader            | Document Loader                        | Prepares chunks for embedding                  | Recursive Character Text Splitter | Generate Embeddings             |                                                                                                                        |
| Generate Embeddings            | OpenAI Embeddings                      | Generates vector embeddings for chunks         | Default Data Loader              | Pinecone Vector Store           |                                                                                                                        |
| Pinecone Vector Store          | Pinecone Vector Store (Insert Mode)   | Inserts embeddings and chunks into Pinecone    | Generate Embeddings, HTTP Request | None                          |                                                                                                                        |
| When chat message received     | Chat Trigger (Webhook)                 | Receives user chat messages                     | External                        | AI Agent                       |                                                                                                                        |
| AI Agent                      | Langchain Agent                       | Orchestrates chat logic and response generation | When chat message received, OpenAI Chat Model, Vector Store Tool, Window Buffer Memory | OpenAI Chat Model1             |                                                                                                                        |
| OpenAI Chat Model             | OpenAI Chat Completion                 | Provides LLM for AI Agent                        | AI Agent (language model)        | AI Agent                       |                                                                                                                        |
| Window Buffer Memory          | Memory Buffer (Window)                 | Maintains conversation context                  | AI Agent                       | AI Agent                       |                                                                                                                        |
| Vector Store Tool             | Vector Store Tool                      | Enables semantic search access for AI Agent    | Pinecone Vector Store (Querying) | AI Agent                       |                                                                                                                        |
| OpenAI Chat Model1            | OpenAI Chat Completion                 | Generates final response                         | AI Agent                       | Vector Store Tool              |                                                                                                                        |
| Generate User Query Embedding | OpenAI Embeddings                      | Embeds user query for semantic search           | AI Agent                       | Pinecone Vector Store (Querying) |                                                                                                                        |
| Pinecone Vector Store (Querying) | Pinecone Vector Store (Query Mode)    | Queries Pinecone for relevant chunks             | Generate User Query Embedding    | Vector Store Tool              |                                                                                                                        |
| Sticky Note                  | Sticky Note                           | Documentation note                              | None                           | None                          | ## Indexing content in the vector database\nThis part of the workflow is responsible for extracting content, generating embeddings and sending them to the Pinecone vector store.\n\nIt requests the OpenAPI specifications from GitHub using a HTTP request. Then, it splits the file in chunks, generating embeddings for each chunk using OpenAI, and saving them in Pinecone vector DB. |
| Sticky Note1                 | Sticky Note                           | Documentation note                              | None                           | None                          | ## Querying and response generation \n\nThis part of the workflow is responsible for the chat interface, querying the vector store and generating relevant responses.\n\nIt uses OpenAI GPT 4o-mini to generate responses. |
| Sticky Note2                 | Sticky Note                           | Documentation note                              | None                           | None                          | ## RAG workflow in n8n\n\nThis is an example of how to use RAG techniques to create a chatbot with n8n. It is an API documentation chatbot that can answer questions about the GitHub API. It uses OpenAI for generating embeddings, the gpt-4o-mini LLM for generating responses and Pinecone as a vector database.\n\n### Before using this template\n* create OpenAI and Pinecone accounts\n* obtain API keys OpenAI and Pinecone \n* configure credentials in n8n for both\n* ensure you have a Pinecone index named "n8n-demo" or adjust the workflow accordingly. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the ingestion process.

2. **Create HTTP Request Node**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://raw.githubusercontent.com/github/rest-api-description/refs/heads/main/descriptions/api.github.com/api.github.com.json`  
     - Method: GET (default)  
   - Connect Manual Trigger → HTTP Request.

3. **Create Recursive Character Text Splitter Node**  
   - Type: Text Splitter (Recursive Character)  
   - Parameters: Default chunk size and overlap (adjust if needed).  
   - Connect downstream nodes accordingly (see below).

4. **Create Default Data Loader Node**  
   - Type: Document Default Data Loader  
   - Parameters: Default options.  
   - Connect Recursive Character Text Splitter → Default Data Loader.

5. **Create OpenAI Embeddings Node (Generate Embeddings)**  
   - Type: OpenAI Embeddings  
   - Credentials: Configure OpenAI API credentials with your API key.  
   - Parameters: Default embedding model (e.g., text-embedding-ada-002).  
   - Connect Default Data Loader → Generate Embeddings.

6. **Create Pinecone Vector Store Node (Insert Mode)**  
   - Type: Pinecone Vector Store  
   - Credentials: Configure Pinecone API credentials with your API key.  
   - Parameters:  
     - Mode: Insert  
     - Pinecone Index: "n8n-demo" (or your own index name)  
   - Connect Generate Embeddings → Pinecone Vector Store.  
   - Also connect HTTP Request → Pinecone Vector Store (for document input).

7. **Create Chat Trigger Node**  
   - Type: Chat Trigger (Webhook)  
   - Parameters: Default webhook setup.  
   - This node listens for incoming chat messages.

8. **Create AI Agent Node**  
   - Type: Langchain Agent  
   - Parameters:  
     - System Message: "You are a helpful assistant providing information about the GitHub API and how to use it based on the OpenAPI V3 specifications."  
   - Connect Chat Trigger → AI Agent.

9. **Create OpenAI Chat Model Node**  
   - Type: OpenAI Chat Completion  
   - Credentials: Use OpenAI API credentials.  
   - Parameters: Default GPT model settings.  
   - Connect AI Agent (language model input) → OpenAI Chat Model → AI Agent.

10. **Create Window Buffer Memory Node**  
    - Type: Memory Buffer (Window)  
    - Parameters: Default window size (e.g., last 5 messages).  
    - Connect AI Agent (memory input) → Window Buffer Memory → AI Agent.

11. **Create Vector Store Tool Node**  
    - Type: Vector Store Tool  
    - Parameters:  
      - Name: "GitHub_OpenAPI_Specification"  
      - Description: "Use this tool to get information about the GitHub API. This database contains OpenAPI v3 specifications."  
    - Connect Pinecone Vector Store (Querying) → Vector Store Tool → AI Agent.

12. **Create OpenAI Chat Model1 Node**  
    - Type: OpenAI Chat Completion  
    - Credentials: OpenAI API credentials.  
    - Parameters: Use GPT-4o-mini or similar model for response generation.  
    - Connect AI Agent → OpenAI Chat Model1 → Vector Store Tool.

13. **Create Generate User Query Embedding Node**  
    - Type: OpenAI Embeddings  
    - Credentials: OpenAI API credentials.  
    - Parameters: Default embedding model.  
    - Connect AI Agent (user query) → Generate User Query Embedding → Pinecone Vector Store (Querying).

14. **Create Pinecone Vector Store Node (Query Mode)**  
    - Type: Pinecone Vector Store  
    - Credentials: Pinecone API credentials.  
    - Parameters:  
      - Mode: Query  
      - Pinecone Index: "n8n-demo"  
    - Connect Generate User Query Embedding → Pinecone Vector Store (Querying).

15. **Connect all nodes as per the logical flow:**  
    - Manual Trigger → HTTP Request → Recursive Character Text Splitter → Default Data Loader → Generate Embeddings → Pinecone Vector Store (Insert)  
    - Chat Trigger → AI Agent → OpenAI Chat Model → AI Agent  
    - AI Agent → Window Buffer Memory → AI Agent  
    - AI Agent → Generate User Query Embedding → Pinecone Vector Store (Querying) → Vector Store Tool → AI Agent  
    - AI Agent → OpenAI Chat Model1 → Vector Store Tool

16. **Credential Setup:**  
    - Configure OpenAI API credentials with your API key in n8n.  
    - Configure Pinecone API credentials with your API key in n8n.  
    - Ensure Pinecone index "n8n-demo" exists or create one and update the workflow accordingly.

17. **Test the workflow:**  
    - Run the ingestion part manually to populate Pinecone with embeddings.  
    - Use the chat webhook to send queries and receive responses.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow is a practical example of RAG (Retrieval Augmented Generation) techniques combining OpenAI LLMs and Pinecone vector database for API documentation chatbots.                                                  |                                                                                                   |
| Before using, create OpenAI and Pinecone accounts, obtain API keys, and configure credentials in n8n.                                                                                                                        |                                                                                                   |
| Ensure you have a Pinecone index named "n8n-demo" or adjust the workflow to your index name.                                                                                                                                 |                                                                                                   |
| Setup time is approximately 15-20 minutes.                                                                                                                                                                                    |                                                                                                   |
| The workflow can be adapted to any OpenAPI specification or large textual knowledge base to create custom documentation chatbots.                                                                                           |                                                                                                   |
| Sticky notes within the workflow provide detailed explanations of indexing and querying parts.                                                                                                                               |                                                                                                   |
| For more information on Pinecone, visit: https://www.pinecone.io/                                                                                                                                                             |                                                                                                   |
| For OpenAI API details, see: https://platform.openai.com/docs/api-reference                                                                                                                                                   |                                                                                                   |
| The workflow uses GPT-4o-mini model for cost-effective yet capable response generation.                                                                                                                                        |                                                                                                   |

---

This structured documentation provides a comprehensive understanding of the workflow, enabling advanced users and AI agents to analyze, reproduce, and modify the RAG-powered GitHub API chatbot effectively.