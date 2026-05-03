Travel Planning Agent with Couchbase Vector Search, Gemini 2.0 Flash and OpenAI

https://n8nworkflows.xyz/workflows/travel-planning-agent-with-couchbase-vector-search--gemini-2-0-flash-and-openai-3881


# Travel Planning Agent with Couchbase Vector Search, Gemini 2.0 Flash and OpenAI

### 1. Workflow Overview

This workflow, titled **Travel Planning Agent with Couchbase Vector Search, Gemini 2.0 Flash and OpenAI**, is designed to assist users in deciding travel destinations based on points of interest stored and vectorized in a Couchbase database. It leverages advanced AI models for natural language understanding and retrieval-augmented generation (RAG) to provide intelligent travel recommendations.

The workflow is logically divided into two main functional blocks:

- **1.1 Data Ingestion Block:** Handles the ingestion of travel points of interest data via a webhook, processes the textual data into embeddings using OpenAI, splits large texts recursively, and inserts the vectorized documents into Couchbase for later retrieval.

- **1.2 Chat Application Block:** Listens for chat messages from users, uses the Gemini 2.0 Flash model to generate conversational responses, maintains conversation context with a simple memory buffer, and retrieves relevant points of interest from Couchbase using vector search enhanced by OpenAI embeddings.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion Block

**Overview:**  
This block receives raw travel point of interest data via an HTTP webhook, processes the data by generating embeddings with OpenAI, splits text recursively if needed, and inserts the vectorized documents into a Couchbase collection for future retrieval.

**Nodes Involved:**  
- Webhook  
- Default Data Loader  
- Recursive Character Text Splitter  
- Generate OpenAI Embeddings using text-embedding-3-small1  
- Insert docs with Couchbase Search Vector  
- Sticky Note1 (documentation aid)

**Node Details:**

- **Webhook**  
  - *Type:* HTTP Webhook  
  - *Role:* Entry point for receiving travel point of interest data via POST requests.  
  - *Configuration:* Path set to a unique webhook ID, accepts raw POST body, HTTP method POST.  
  - *Input/Output:* No input; outputs raw request data to Default Data Loader and Insert docs nodes.  
  - *Edge Cases:* Invalid or malformed JSON payloads, network timeouts, unauthorized access if webhook URL is exposed.  
  - *Sticky Note1:* Provides example cURL commands and a shell script link for bulk data ingestion.

- **Default Data Loader**  
  - *Type:* Document Data Loader (LangChain)  
  - *Role:* Converts incoming JSON data into a textual document format for embedding.  
  - *Configuration:* Uses an expression to concatenate the title and description fields from the webhook JSON body into a single string.  
  - *Key Expression:* `={{ $json.body.raw_body.point_of_interest.title }} - {{ $json.body.raw_body.point_of_interest.description }}`  
  - *Input:* Receives webhook data.  
  - *Output:* Sends document text to Recursive Character Text Splitter.  
  - *Edge Cases:* Missing or malformed fields in input JSON may cause expression failures.

- **Recursive Character Text Splitter**  
  - *Type:* Text Splitter (LangChain)  
  - *Role:* Splits long text documents into smaller chunks recursively to optimize embedding quality and indexing.  
  - *Configuration:* Default settings, no custom parameters.  
  - *Input:* Receives document text from Default Data Loader.  
  - *Output:* Passes split text chunks to the OpenAI Embeddings node.  
  - *Edge Cases:* Very large texts may cause performance issues; splitting logic may break semantic coherence if not tuned.

- **Generate OpenAI Embeddings using text-embedding-3-small1**  
  - *Type:* OpenAI Embeddings Node (LangChain)  
  - *Role:* Generates vector embeddings from text chunks for insertion into Couchbase.  
  - *Configuration:* Uses OpenAI credentials; model defaults to `text-embedding-3-small`.  
  - *Input:* Receives split text chunks from Recursive Character Text Splitter.  
  - *Output:* Sends embeddings and original text to Couchbase Insert node.  
  - *Edge Cases:* API rate limits, authentication errors, network timeouts.

- **Insert docs with Couchbase Search Vector**  
  - *Type:* Couchbase Vector Store Node (Community)  
  - *Role:* Inserts vectorized documents into Couchbase vector search index.  
  - *Configuration:*  
    - Mode: Insert  
    - Embedding source: From OpenAI Embeddings node  
    - Text field key: `description`  
    - Bucket, scope, collection, and vector index name configured via credentials and parameters (user must select).  
    - Embedding batch size: 1  
  - *Input:* Receives embeddings and text from OpenAI Embeddings node and raw webhook data.  
  - *Output:* None (terminal node for ingestion).  
  - *Edge Cases:* Couchbase connection failures, permission errors, index misconfiguration, batch size issues.

- **Sticky Note1**  
  - *Type:* Documentation node  
  - *Role:* Provides example cURL commands and a shell script link for bulk data ingestion.  
  - *Content:*  
    - Example cURL POST command to send a point of interest JSON to the webhook.  
    - Link to a shell script for bulk inserting six data points.  
  - *Context:* Assists users in loading data correctly.

---

#### 2.2 Chat Application Block

**Overview:**  
This block listens for chat messages from users, processes them through an AI agent that uses Gemini 2.0 Flash as the chat model, maintains conversation context with a simple memory buffer, and retrieves relevant points of interest from Couchbase vector search enhanced by OpenAI embeddings.

**Nodes Involved:**  
- When chat message received  
- AI Travel Agent  
- Google Gemini Chat Model  
- Simple Memory  
- Retrieve docs with Couchbase Search Vector  
- Generate OpenAI Embeddings using text-embedding-3-small  
- Sticky Note (documentation aid)

**Node Details:**

- **When chat message received**  
  - *Type:* LangChain Chat Trigger  
  - *Role:* Entry point for chat messages from users to start the AI agent workflow.  
  - *Configuration:* Default options; webhook ID assigned.  
  - *Input:* External chat messages.  
  - *Output:* Sends chat input to AI Travel Agent node.  
  - *Edge Cases:* Message format errors, webhook connectivity issues.

- **AI Travel Agent**  
  - *Type:* LangChain Agent Node  
  - *Role:* Core AI agent orchestrating the conversation, integrating memory, language model, and retrieval tools.  
  - *Configuration:*  
    - Max iterations: 10 (limits reasoning steps)  
    - System message: "You are a helpful assistant for a trip planner. You have a vector search capability to locate points of interest, Use it and don't invent much."  
  - *Input:* Receives chat messages, memory context, language model output, and retrieval tool results.  
  - *Output:* Generates final chat responses.  
  - *Edge Cases:* Infinite loops if max iterations not set, tool failures, inconsistent memory state.

- **Google Gemini Chat Model**  
  - *Type:* LangChain Google Gemini Chat Model  
  - *Role:* Provides the conversational language model for generating responses.  
  - *Configuration:* Model name set to `models/gemini-2.0-flash`, requires Google AI credentials.  
  - *Input:* Receives prompts from AI Travel Agent.  
  - *Output:* Returns generated chat completions to AI Travel Agent.  
  - *Edge Cases:* API quota limits, authentication errors, latency.

- **Simple Memory**  
  - *Type:* LangChain Memory Buffer Window  
  - *Role:* Maintains conversation history and context for the AI agent.  
  - *Configuration:* Default buffer window size (not explicitly set).  
  - *Input:* Receives chat messages and AI responses.  
  - *Output:* Provides memory context to AI Travel Agent.  
  - *Edge Cases:* Memory overflow if buffer size is too large, loss of context if reset.

- **Retrieve docs with Couchbase Search Vector**  
  - *Type:* Couchbase Vector Store Node (Community)  
  - *Role:* Retrieves relevant points of interest from Couchbase using vector similarity search.  
  - *Configuration:*  
    - Mode: Retrieve as tool  
    - Top K: 10 (returns top 10 matches)  
    - Tool name: "PointofinterestKB"  
    - Text field key: `description`  
    - Bucket, scope, collection, and vector index name configured via credentials and parameters (user must select).  
  - *Input:* Receives query embeddings from OpenAI Embeddings node.  
  - *Output:* Returns retrieved documents to AI Travel Agent as a tool result.  
  - *Edge Cases:* Couchbase connection issues, empty or irrelevant results, index misconfiguration.

- **Generate OpenAI Embeddings using text-embedding-3-small**  
  - *Type:* OpenAI Embeddings Node (LangChain)  
  - *Role:* Converts user queries into embeddings for vector search.  
  - *Configuration:* Uses OpenAI credentials; model defaults to `text-embedding-3-small`.  
  - *Input:* Receives user query text from AI Travel Agent.  
  - *Output:* Sends embeddings to Couchbase Retrieve node.  
  - *Edge Cases:* API rate limits, authentication errors, network timeouts.

- **Sticky Note**  
  - *Type:* Documentation node  
  - *Role:* Provides setup instructions and important configuration notes for users.  
  - *Content Highlights:*  
    - Setup instructions for Google Gemini and OpenAI credentials.  
    - Couchbase cluster setup including bucket, scope, collection, and vector index import link.  
    - Reminder to send data to the ingestion webhook before querying.  
    - Encouragement to test the agent with example questions.  
  - *Context:* Guides users through initial configuration and testing.

---

### 3. Summary Table

| Node Name                                      | Node Type                                    | Functional Role                           | Input Node(s)                      | Output Node(s)                         | Sticky Note                                                                                                                          |
|------------------------------------------------|----------------------------------------------|-----------------------------------------|----------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received                      | LangChain Chat Trigger                        | Entry point for chat messages            | -                                | AI Travel Agent                       |                                                                                                                                      |
| AI Travel Agent                                | LangChain Agent                              | Core AI agent integrating memory, LM, and tools | When chat message received, Simple Memory, Google Gemini Chat Model, Retrieve docs with Couchbase Search Vector | Google Gemini Chat Model, Simple Memory, Retrieve docs with Couchbase Search Vector |                                                                                                                                      |
| Google Gemini Chat Model                       | LangChain Google Gemini Chat Model           | Generates conversational responses       | AI Travel Agent                  | AI Travel Agent                       | Setup Google API Credentials required                                                                                               |
| Simple Memory                                  | LangChain Memory Buffer Window               | Maintains conversation context           | AI Travel Agent                  | AI Travel Agent                       |                                                                                                                                      |
| Retrieve docs with Couchbase Search Vector    | Couchbase Vector Store Node (Community)      | Retrieves relevant points of interest    | Generate OpenAI Embeddings using text-embedding-3-small | AI Travel Agent                       |                                                                                                                                      |
| Generate OpenAI Embeddings using text-embedding-3-small | OpenAI Embeddings Node (LangChain)           | Converts queries to embeddings            | AI Travel Agent                  | Retrieve docs with Couchbase Search Vector | Setup OpenAI Credentials required                                                                                                   |
| Webhook                                        | HTTP Webhook                                 | Receives data ingestion requests         | -                                | Insert docs with Couchbase Search Vector | Sticky Note1 provides example cURL commands and bulk ingestion script link                                                          |
| Default Data Loader                            | Document Data Loader (LangChain)              | Converts JSON to text document            | Webhook                         | Recursive Character Text Splitter     |                                                                                                                                      |
| Recursive Character Text Splitter             | Text Splitter (LangChain)                      | Splits long text for embedding           | Default Data Loader             | Generate OpenAI Embeddings using text-embedding-3-small1 |                                                                                                                                      |
| Generate OpenAI Embeddings using text-embedding-3-small1 | OpenAI Embeddings Node (LangChain)           | Generates embeddings for insertion       | Recursive Character Text Splitter | Insert docs with Couchbase Search Vector | Setup OpenAI Credentials required                                                                                                   |
| Insert docs with Couchbase Search Vector      | Couchbase Vector Store Node (Community)      | Inserts vectorized documents into Couchbase | Webhook, Default Data Loader, Generate OpenAI Embeddings using text-embedding-3-small1 | -                                     |                                                                                                                                      |
| Sticky Note                                    | Documentation node                            | Setup instructions and configuration notes | -                                | -                                     | Setup Google Gemini, OpenAI, Couchbase cluster, bucket/scope/collection, and vector index import instructions                        |
| Sticky Note1                                   | Documentation node                            | Example cURL and bulk ingestion script   | -                                | -                                     | Example cURL command for data ingestion; link to bulk ingestion shell script                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: HTTP Webhook  
   - Parameters:  
     - Path: Unique identifier (e.g., `3ca6fbdd-a157-4e9d-9042-237048da85b6`)  
     - HTTP Method: POST  
     - Options: Enable raw body capture  
   - Purpose: Receive JSON data for points of interest ingestion.

2. **Create Default Data Loader Node**  
   - Type: LangChain Document Default Data Loader  
   - Parameters:  
     - JSON Mode: Expression Data  
     - JSON Data Expression: `={{ $json.body.raw_body.point_of_interest.title }} - {{ $json.body.raw_body.point_of_interest.description }}`  
   - Connect input from Webhook node output.

3. **Create Recursive Character Text Splitter Node**  
   - Type: LangChain Recursive Character Text Splitter  
   - Parameters: Default (no custom options)  
   - Connect input from Default Data Loader output.

4. **Create OpenAI Embeddings Node for Insertion**  
   - Type: LangChain OpenAI Embeddings  
   - Parameters: Default (model `text-embedding-3-small`)  
   - Credentials: Configure OpenAI API key credentials  
   - Connect input from Recursive Character Text Splitter output.

5. **Create Couchbase Vector Store Node for Insertion**  
   - Type: Couchbase Vector Store (Community)  
   - Parameters:  
     - Mode: Insert  
     - Embedding: Use output from OpenAI Embeddings node  
     - Text Field Key: `description`  
     - Bucket, Scope, Collection: Select your Couchbase bucket (`travel-agent`), scope (`vectors`), and collection (`points-of-interest`)  
     - Vector Index Name: Select the vector index created in Couchbase  
     - Embedding Batch Size: 1  
   - Connect inputs from Webhook (for raw data) and OpenAI Embeddings node.  
   - Credentials: Configure Couchbase credentials with appropriate permissions.

6. **Create Chat Trigger Node**  
   - Type: LangChain Chat Trigger  
   - Parameters: Default  
   - Purpose: Entry point for chat messages.

7. **Create Simple Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Parameters: Default buffer size  
   - Purpose: Maintain conversation context.

8. **Create OpenAI Embeddings Node for Querying**  
   - Type: LangChain OpenAI Embeddings  
   - Parameters: Default (model `text-embedding-3-small`)  
   - Credentials: Use same OpenAI credentials as ingestion embeddings.

9. **Create Couchbase Vector Store Node for Retrieval**  
   - Type: Couchbase Vector Store (Community)  
   - Parameters:  
     - Mode: Retrieve as Tool  
     - Top K: 10  
     - Tool Name: `PointofinterestKB`  
     - Text Field Key: `description`  
     - Bucket, Scope, Collection: Same as insertion node  
     - Vector Index Name: Same as insertion node  
   - Credentials: Use Couchbase credentials.

10. **Create Google Gemini Chat Model Node**  
    - Type: LangChain Google Gemini Chat Model  
    - Parameters:  
      - Model Name: `models/gemini-2.0-flash`  
    - Credentials: Configure Google AI credentials for Gemini.

11. **Create AI Travel Agent Node**  
    - Type: LangChain Agent  
    - Parameters:  
      - Max Iterations: 10  
      - System Message: "You are a helpful assistant for a trip planner. You have a vector search capability to locate points of interest, Use it and don't invent much."  
    - Connect inputs:  
      - Main input from Chat Trigger node  
      - AI Language Model input from Google Gemini Chat Model node  
      - AI Memory input from Simple Memory node  
      - AI Tool input from Couchbase Vector Store Retrieval node  
    - Connect output to Google Gemini Chat Model, Simple Memory, and Couchbase Retrieval nodes as appropriate.

12. **Connect Chat Trigger node output to AI Travel Agent main input.**

13. **Connect Simple Memory output to AI Travel Agent memory input.**

14. **Connect Google Gemini Chat Model output to AI Travel Agent language model input.**

15. **Connect AI Travel Agent output to Google Gemini Chat Model input.**

16. **Connect AI Travel Agent output to Simple Memory input.**

17. **Connect AI Travel Agent output to OpenAI Embeddings (query) input.**

18. **Connect OpenAI Embeddings (query) output to Couchbase Vector Store Retrieval input.**

19. **Connect Couchbase Vector Store Retrieval output to AI Travel Agent tool input.**

20. **Add Sticky Notes**  
    - One with setup instructions for credentials, Couchbase cluster, bucket/scope/collection, and vector index import.  
    - One with example cURL commands and a link to a bulk ingestion shell script.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow uses the `n8n-nodes-couchbase` community package. Community nodes are unverified and usage comes with risks. See [n8n community nodes risks](https://docs.n8n.io/integrations/community-nodes/risks/) for details.           | n8n official documentation on community nodes risks                                                         |
| Instructions for installing n8n community nodes can be found [here](https://docs.n8n.io/integrations/community-nodes/installation/gui-install/).                                                                                     | n8n official documentation on installing community nodes                                                     |
| Couchbase cluster setup requires creating a vector search index. Use the index definition JSON found [here](https://gist.github.com/ejscribner/6f16343d4b44b1af31e8f344557814b0).                                                     | Gist with Couchbase vector index JSON                                                                        |
| Bulk ingestion shell script for loading six points of interest is available [here](https://gist.github.com/ejscribner/355a46a0a383a4878e65e2230b92c6b5).                                                                            | Gist with bulk ingestion shell script                                                                        |
| Recommended Couchbase bucket: `travel-agent`, scope: `vectors`, collection: `points-of-interest`.                                                                                                                                     | Couchbase naming conventions                                                                                  |
| OpenAI credentials are required for embedding generation nodes.                                                                                                                                                                       | n8n OpenAI credential setup: https://docs.n8n.io/integrations/builtin/credentials/openai/                      |
| Google AI credentials are required for Gemini 2.0 Flash chat model node.                                                                                                                                                              | n8n Google AI credential setup: https://docs.n8n.io/integrations/builtin/credentials/googleai/                 |

---

This comprehensive documentation enables advanced users and AI agents to fully understand, reproduce, and customize the Travel Planning Agent workflow with Couchbase vector search, Gemini 2.0 Flash, and OpenAI embeddings.