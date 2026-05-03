Chat with GitHub Issues using OpenAI and Redis Vector Search

https://n8nworkflows.xyz/workflows/chat-with-github-issues-using-openai-and-redis-vector-search-10837


# Chat with GitHub Issues using OpenAI and Redis Vector Search

### 1. Workflow Overview

This workflow enables natural language interaction with GitHub repository issues by combining OpenAI’s language models and Redis vector search for retrieval-augmented generation (RAG). It consists of two main logical blocks:

- **1.1 Data Ingestion Workflow:**  
  Fetches all issues from a specified GitHub repository, processes and embeds them using OpenAI embeddings, and stores the embedded representations along with metadata in a Redis vector store. This populates the semantic search database that underpins the conversational AI.

- **1.2 Chat Interface Workflow:**  
  Provides a public-facing chat interface where users can ask questions about the repository’s issues. The AI agent uses GPT-4.1-mini to understand user queries, retrieves relevant issues from Redis vector search, and maintains conversation context using Redis chat memory for multi-turn interactions.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion Workflow

- **Overview:**  
  This block collects all open issues from a GitHub repository, transforms the issues into a structured format, generates embeddings for semantic search, and stores the embeddings in a Redis vector index for later retrieval.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Fetch issues from GitHub (HTTP Request)  
  - Default Data Loader (Document Data Loader)  
  - Embeddings OpenAI (OpenAI Embeddings)  
  - Vectorize and store in Redis (Redis Vector Store)  
  - Sticky Note6, Sticky Note7, Sticky Note8 (Informational notes)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the ingestion process manually.  
    - Configuration: No parameters; triggers the next node on manual execution.  
    - Inputs: None  
    - Outputs: Connects to "Fetch issues from GitHub"  
    - Edge Cases: None (manual trigger)  

  - **Fetch issues from GitHub**  
    - Type: HTTP Request  
    - Role: Requests all open issues from GitHub API for the specified repository.  
    - Configuration:
      - URL template: `https://api.github.com/repos/{{ $parameter.owner }}/{{ $parameter.repository }}/issues?per_page=100&state=open`
      - Pagination enabled using the `Link` header for continuous fetching.
      - Timeout set to 1000 ms.
      - Full HTTP response requested to handle pagination header.
    - Expressions: Uses workflow parameters `owner` and `repository` to build URL dynamically.  
    - Inputs: Triggered by manual trigger node.  
    - Outputs: Passes raw issue data to "Vectorize and store in Redis".  
    - Edge Cases:
      - GitHub rate limiting (see Sticky Note8).  
      - Network timeouts or API errors.  
      - Pagination failures if Link header malformed.  
      - Requires GitHub personal access token for higher rate limits (recommended).  

  - **Default Data Loader**  
    - Type: Document Data Loader (LangChain)  
    - Role: Converts raw JSON issue data into structured documents with metadata for embedding generation.  
    - Configuration:
      - Extracts `title` and `body` from each issue JSON into a JSON document with keys `title` and `details`.
      - Adds metadata fields `url` and `state` from the issue data.
      - Uses expression mode to dynamically map input JSON fields.  
    - Inputs: None directly; integrated internally in vector store node pipeline.  
    - Outputs: Provides structured documents for embeddings.  
    - Edge Cases: Missing fields in issue JSON could cause empty or invalid documents.  

  - **Embeddings OpenAI**  
    - Type: OpenAI Embeddings (LangChain)  
    - Role: Generates vector embeddings from issue documents for semantic search.  
    - Configuration:
      - Uses default OpenAI embeddings model (API key required in credentials).  
    - Inputs: Receives documents from Default Data Loader.  
    - Outputs: Embeddings passed to Redis vector store.  
    - Edge Cases:  
      - API rate limits or quota exceeded.  
      - Network issues.  
      - Invalid input document format causing embedding failure.  

  - **Vectorize and store in Redis**  
    - Type: Redis Vector Store (LangChain)  
    - Role: Inserts documents and embeddings into Redis vector index `github_issues_v1`.  
    - Configuration:
      - Mode: `insert` (adds new vectors).  
      - Redis index: `github_issues_v1`.  
      - Uses credentials configured for Redis server version 8.x or higher.  
    - Inputs: Receives embeddings and documents.  
    - Outputs: None downstream in this block.  
    - Edge Cases:  
      - Redis connection errors.  
      - Index conflicts or schema mismatch.  
      - Data insertion failures.  

  - **Sticky Notes (6,7,8)**  
    - Provide detailed explanations and warnings related to GitHub API rate limits, setup instructions, and links to official docs for Redis Vector Store and GitHub API.

---

#### 2.2 Chat Interface Workflow

- **Overview:**  
  Enables users to interact with the AI agent via a chat interface, leveraging the Redis vector store to perform semantic search on GitHub issues and maintain conversational context using Redis chat memory. The AI agent uses GPT-4.1-mini model with retrieval-augmented generation.

- **Nodes Involved:**  
  - When chat message received (Chat Trigger)  
  - AI Agent using RAG (LangChain Agent)  
  - OpenAI Chat Model (OpenAI GPT-4.1-mini)  
  - Redis Chat Memory (Redis Chat Memory)  
  - Augment with results from Redis (Redis Vector Store in retrieval mode)  
  - Embeddings OpenAI1 (OpenAI Embeddings for retrieval)  
  - Sticky Note (general info)

- **Node Details:**

  - **When chat message received**  
    - Type: Chat Trigger (LangChain)  
    - Role: Listens for incoming chat messages via a public webhook interface.  
    - Configuration:
      - Public access enabled (no auth).  
      - WebhookId assigned.  
    - Inputs: External user chat messages.  
    - Outputs: Connects to "AI Agent using RAG".  
    - Edge Cases:  
      - Unauthorized or spam messages (no auth).  
      - Webhook downtime.  

  - **AI Agent using RAG**  
    - Type: LangChain Agent  
    - Role: Central AI agent orchestrating retrieval and response generation.  
    - Configuration:
      - System message sets assistant role: helpful assistant for exploring GitHub issues.
      - Max iterations: 10 (limits recursive reasoning steps).  
      - Integrates multiple tools: language model, retrieval tool, and memory.  
    - Inputs: Chat messages from trigger node.  
    - Outputs: Final chat response.  
    - Connected Nodes:
      - AI language model: OpenAI Chat Model  
      - AI memory: Redis Chat Memory  
      - AI tool (retrieval): Augment with results from Redis  
    - Edge Cases:  
      - Model generation failures or timeouts.  
      - Retrieval tool returning empty or irrelevant results.  
      - Memory read/write errors.  

  - **OpenAI Chat Model**  
    - Type: OpenAI Chat Completion (LangChain)  
    - Role: Generates natural language responses based on chat context and retrieved info.  
    - Configuration:
      - Uses `gpt-4.1-mini` model variant.  
      - Default options, no additional tools enabled.  
    - Inputs: Messages and context from AI Agent.  
    - Outputs: Generated text responses.  
    - Edge Cases:  
      - API rate limits.  
      - Model unavailability.  

  - **Redis Chat Memory**  
    - Type: Redis Chat Memory (LangChain)  
    - Role: Stores and manages chat conversation history in Redis for multi-turn dialogue continuity.  
    - Configuration: Default settings (Redis server connection required).  
    - Inputs: Chat messages and responses.  
    - Outputs: Provides chat history context to AI Agent.  
    - Edge Cases:  
      - Redis connection failure.  
      - Memory size limits or eviction policies.  

  - **Augment with results from Redis**  
    - Type: Redis Vector Store (LangChain)  
    - Role: Performs semantic search retrieval of relevant GitHub issues from Redis vector index `github_issues_v1`.  
    - Configuration:
      - Mode: `retrieve-as-tool` (acts as a tool invoked by AI Agent).  
      - Redis index: `github_issues_v1`.  
      - Tool description: "The list of issues from the database".  
      - Cached result name configured for performance.  
    - Inputs: Embeddings generated by "Embeddings OpenAI1".  
    - Outputs: Search results passed to AI Agent for response augmentation.  
    - Edge Cases:  
      - Empty or no relevant search results.  
      - Redis query engine errors.  

  - **Embeddings OpenAI1**  
    - Type: OpenAI Embeddings (LangChain)  
    - Role: Converts user query or AI intermediate data to embeddings for retrieval search.  
    - Configuration: Default OpenAI embeddings.  
    - Inputs: From AI Agent retrieval step.  
    - Outputs: Embeddings forwarded to Redis retrieval node.  
    - Edge Cases: Same as Embeddings OpenAI node.  

  - **Sticky Note (general info)**  
    - Provides instructions to use the same embeddings provider and vector store as in ingestion for consistency.

---

### 3. Summary Table

| Node Name                   | Node Type                                   | Functional Role                         | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                          |
|-----------------------------|---------------------------------------------|---------------------------------------|-----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                             | Starts Data Ingestion manually          | None                        | Fetch issues from GitHub     |                                                                                                                                      |
| Fetch issues from GitHub      | HTTP Request                               | Retrieves all open GitHub issues        | When clicking ‘Execute workflow’ | Vectorize and store in Redis | See Sticky Note8 about GitHub rate limits and personal access token.                                                                 |
| Default Data Loader           | Document Data Loader (LangChain)           | Structures issue JSON for embedding     | Internal to Vectorize & store in Redis | Embeddings OpenAI           | See Sticky Note6 and Sticky Note7 for detailed ingestion flow description and Redis vector store docs.                                 |
| Embeddings OpenAI            | OpenAI Embeddings (LangChain)               | Generates embeddings for GitHub issues  | Default Data Loader          | Vectorize and store in Redis |                                                                                                                                      |
| Vectorize and store in Redis | Redis Vector Store (LangChain)              | Stores embedded issues in Redis vector index | Embeddings OpenAI, Fetch issues from GitHub | None                       | See Sticky Note6 for Redis vector store docs.                                                                                        |
| When chat message received   | Chat Trigger (LangChain)                     | Listens for incoming chat messages      | External user messages       | AI Agent using RAG           |                                                                                                                                      |
| AI Agent using RAG           | LangChain Agent                             | Orchestrates AI reasoning with retrieval | When chat message received   | (final chat response)        |                                                                                                                                      |
| OpenAI Chat Model            | OpenAI Chat Completion (LangChain)          | Generates natural language responses    | AI Agent using RAG           | AI Agent using RAG           |                                                                                                                                      |
| Redis Chat Memory            | Redis Chat Memory (LangChain)                | Maintains chat conversation history     | AI Agent using RAG           | AI Agent using RAG           | See Sticky Note for Redis Chat Memory docs.                                                                                          |
| Augment with results from Redis | Redis Vector Store (LangChain)            | Retrieves relevant issues from Redis    | Embeddings OpenAI1           | AI Agent using RAG           | Tool description: "The list of issues from the database".                                                                             |
| Embeddings OpenAI1           | OpenAI Embeddings (LangChain)               | Embeds user queries for vector search   | AI Agent using RAG           | Augment with results from Redis | Must use same embeddings provider as ingestion (see Sticky Note).                                                                     |
| Sticky Note6                 | Sticky Note                                 | Explains Data Ingestion Workflow & Redis Vector Store docs | None                        | None                       | [Read more about the Redis Vector Store node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreredis/) |
| Sticky Note7                 | Sticky Note                                 | Overview and instructions for entire workflow | None                        | None                       | Detailed workflow description and Discord invite: https://discord.com/invite/redis                                                  |
| Sticky Note8                 | Sticky Note                                 | Warning about GitHub API rate limits     | None                        | None                       | GitHub Rate Limits: https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api?apiVersion=2022-11-28             |
| Sticky Note                  | Sticky Note                                 | Explains Chat Interface Workflow & Redis Chat Memory docs | None                        | None                       | [Read more about the Redis Chat Memory node](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memoryredischat/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Purpose: Start ingestion manually.  

2. **Add an HTTP Request node**  
   - Name: "Fetch issues from GitHub"  
   - Connect from Manual Trigger.  
   - Set URL to:  
     `https://api.github.com/repos/{{ $parameter.owner }}/{{ $parameter.repository }}/issues?per_page=100&state=open`  
   - Enable Pagination with:  
     - Mode: Response Contains Next URL  
     - Next URL expression: `{{$response.headers['link'].split(';')[0].slice(1, -1)}}`  
   - Timeout: 1000 ms  
   - Configure authentication with a GitHub personal access token (recommended).  
   - Parameters: Add `owner` and `repository` as workflow parameters.  

3. **Add a Default Data Loader node (LangChain)**  
   - Name: "Default Data Loader"  
   - Connect from "Fetch issues from GitHub".  
   - Configure JSON mode: expressionData.  
   - JSON Data expression:  
     ```
     {
       title: '{{ $json.title }}',
       details: '{{ $json.body }}'
     }
     ```  
   - Metadata fields:  
     - `url` = `{{$json.url}}`  
     - `state` = `{{$json.state}}`  

4. **Add an Embeddings OpenAI node (LangChain)**  
   - Name: "Embeddings OpenAI"  
   - Connect from Default Data Loader.  
   - Use OpenAI credentials configured with API key.  
   - Default embedding model (no special options).  

5. **Add a Redis Vector Store node (LangChain)**  
   - Name: "Vectorize and store in Redis"  
   - Connect inputs from:  
     - "Embeddings OpenAI" (ai_embedding)  
     - "Default Data Loader" (ai_document)  
     - "Fetch issues from GitHub" (main) - to trigger storage for each fetched issue.  
   - Mode: `insert`  
   - Redis index: `github_issues_v1`  
   - Configure Redis credentials to connect to Redis server 8.x or higher.  

6. **Add a Chat Trigger node (LangChain)**  
   - Name: "When chat message received"  
   - Public access enabled.  
   - Configure webhook ID generated by n8n.  

7. **Add a LangChain Agent node**  
   - Name: "AI Agent using RAG"  
   - Connect from "When chat message received".  
   - System message:  
     ```
     You are a helpful assistant for exploring a public GitHub repository. You have a vector search capability to locate explore existing issues in the repository, Use this capability effectively to provide accurate, relevant insights — avoid making assumptions or fabricating information.
     ```  
   - Set max iterations to 10.  

8. **Add an OpenAI Chat Model node (LangChain)**  
   - Name: "OpenAI Chat Model"  
   - Connect to AI Agent's language model input.  
   - Model: `gpt-4.1-mini`  
   - Use OpenAI credentials.  

9. **Add a Redis Chat Memory node (LangChain)**  
   - Name: "Redis Chat Memory"  
   - Connect to AI Agent's memory input.  
   - Use the same Redis credentials as vector store.  

10. **Add an Embeddings OpenAI node (LangChain)**  
    - Name: "Embeddings OpenAI1"  
    - Connect to AI Agent's retrieval tool input.  
    - Use the same OpenAI credentials and embedding model as ingestion.  

11. **Add a Redis Vector Store node (LangChain)**  
    - Name: "Augment with results from Redis"  
    - Connect from "Embeddings OpenAI1".  
    - Mode: `retrieve-as-tool`  
    - Redis index: `github_issues_v1`  
    - Tool description: "The list of issues from the database"  
    - Cached result name: `github_issues_v1`  
    - Connect its output to AI Agent’s tool input.  

12. **Ensure connections:**  
    - "When clicking ‘Execute workflow’" → "Fetch issues from GitHub"  
    - "Fetch issues from GitHub" → "Vectorize and store in Redis"  
    - "Default Data Loader" → "Embeddings OpenAI" → "Vectorize and store in Redis"  
    - "When chat message received" → "AI Agent using RAG"  
    - "OpenAI Chat Model" → "AI Agent using RAG" (language model input)  
    - "Redis Chat Memory" → "AI Agent using RAG" (memory input)  
    - "Embeddings OpenAI1" → "Augment with results from Redis" → "AI Agent using RAG" (tool input)  

13. **Credentials:**  
    - GitHub personal access token for HTTP Request node (recommended).  
    - OpenAI API key configured for embedding and chat model nodes.  
    - Redis credentials for vector store and chat memory nodes.  

14. **Parameters:**  
    - Workflow parameters for GitHub `owner` and `repository` names.  

15. **Test:**  
    - Run manual trigger node to ingest issues.  
    - Use chat interface webhook to send queries and receive AI-generated responses referencing GitHub issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow implements a Retrieval-Augmented Generation (RAG) system for querying GitHub issues with AI.                                  | Overview in Sticky Note7                                                                                         |
| GitHub API rate limits are strict for unauthenticated requests; use a personal access token to improve ingestion speed.                    | https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api?apiVersion=2022-11-28             |
| Redis server version 8.x is required for vector store functionality; older versions need Redis Query Engine module installed separately.    | Sticky Note6                                                                                                    |
| For detailed documentation on Redis Vector Store and Redis Chat Memory nodes, refer to official n8n docs.                                  | Redis Vector Store: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreredis/  
Redis Chat Memory: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memoryredischat/ |
| Join the Redis community Discord for support and discussion.                                                                                | https://discord.com/invite/redis                                                                                 |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.