Travel Planning Assistant with MongoDB Atlas, Gemini LLM and Vector Search

https://n8nworkflows.xyz/workflows/travel-planning-assistant-with-mongodb-atlas--gemini-llm-and-vector-search-3577


# Travel Planning Assistant with MongoDB Atlas, Gemini LLM and Vector Search

### 1. Workflow Overview

This workflow implements a **Travel Planning Assistant** leveraging MongoDB Atlas for long-term memory and vector search, OpenAI embeddings for vectorizing data, and Google Gemini LLM for conversational AI. It is designed to demonstrate how to build an agentic AI system with native n8n nodes that handle memory persistence, document ingestion, vector similarity search, and AI orchestration seamlessly.

The workflow consists of two main logical blocks:

- **1.1 Data Ingestion Flow:**  
  This flow receives travel-related points of interest (POIs) via a webhook, processes and splits the text, generates vector embeddings using OpenAI, and stores them in MongoDB Atlas’s vector store collection (`points_of_interest`). This enables efficient vector similarity search later.

- **1.2 AI Agent Flow:**  
  This flow triggers on chat messages, maintains conversational memory in MongoDB Atlas, and uses the Gemini LLM to generate responses. When relevant, it queries the vector store for contextual POI information to enrich the conversation, enabling a knowledgeable travel assistant.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion Flow

- **Overview:**  
  Handles incoming data about travel points of interest, converts textual data into vector embeddings, and stores them in MongoDB Atlas’s vector search-enabled collection for later retrieval.

- **Nodes Involved:**  
  - Webhook  
  - MongoDB Atlas Vector Store1 (Insert mode)  
  - Embeddings OpenAI1  
  - Default Data Loader  
  - Recursive Character Text Splitter  
  - Sticky Note1  
  - Sticky Note2  

- **Node Details:**

  - **Webhook**  
    - *Type:* HTTP Webhook (n8n-nodes-base.webhook)  
    - *Role:* Entry point for POST requests containing POI data.  
    - *Configuration:* Path `ingestData`, accepts raw JSON body, HTTP method POST.  
    - *Input:* External HTTP POST with JSON payload describing a point of interest (title and description).  
    - *Output:* Raw JSON data forwarded downstream.  
    - *Edge Cases:* Invalid JSON, missing fields, or incorrect data structure may cause failures.  
    - *Sticky Note:* Provides a cURL example for sending data to this webhook.

  - **Default Data Loader**  
    - *Type:* Document Data Loader (Langchain node)  
    - *Role:* Converts incoming JSON data into a document format suitable for embedding.  
    - *Configuration:* Uses expression to concatenate `title` and `description` from webhook JSON into a single string.  
    - *Input:* JSON from webhook.  
    - *Output:* Document object with combined text.  
    - *Edge Cases:* Missing or malformed fields in input JSON.  
    - *Expression:* `={{ $json.body.raw_body.point_of_interest.title }} - {{ $json.body.raw_body.point_of_interest.description }}`

  - **Recursive Character Text Splitter**  
    - *Type:* Text Splitter (Langchain node)  
    - *Role:* Splits large text documents into smaller chunks for better embedding quality.  
    - *Configuration:* Default splitting options.  
    - *Input:* Document from Default Data Loader.  
    - *Output:* Array of text chunks.  
    - *Edge Cases:* Very large or empty text inputs.

  - **Embeddings OpenAI1**  
    - *Type:* OpenAI Embeddings Node (Langchain)  
    - *Role:* Generates vector embeddings for the text chunks.  
    - *Configuration:* Uses OpenAI API credentials, default embedding model.  
    - *Input:* Text chunks from splitter.  
    - *Output:* Embeddings vectors.  
    - *Credentials:* OpenAI API key required.  
    - *Edge Cases:* API rate limits, invalid credentials, network timeouts.

  - **MongoDB Atlas Vector Store1**  
    - *Type:* MongoDB Atlas Vector Store (Langchain)  
    - *Role:* Inserts documents with embeddings into the `points_of_interest` collection.  
    - *Configuration:* Mode set to `insert`, collection `points_of_interest`, vector index `vector_index`, metadata field `description`.  
    - *Input:* Embeddings from OpenAI node and document metadata.  
    - *Output:* Confirmation of insertion.  
    - *Credentials:* MongoDB Atlas account with proper access.  
    - *Edge Cases:* Connection errors, index misconfiguration, batch size issues.

  - **Sticky Note1**  
    - *Content:* Provides a cURL command example to POST data to the ingestion webhook.  
    - *Context:* Helps users test ingestion easily.

  - **Sticky Note2**  
    - *Content:* Explains the ingestion flow and alternative data loading methods.  
    - *Context:* Clarifies ingestion purpose and flexibility.

---

#### 2.2 AI Agent Flow

- **Overview:**  
  Listens for chat messages, maintains conversation history in MongoDB Atlas, uses Gemini LLM for generating responses, and queries the vector store as a tool to retrieve relevant POI information dynamically.

- **Nodes Involved:**  
  - When chat message received  
  - AI Traveling Planner Agent  
  - MongoDB Chat Memory  
  - Google Gemini Chat Model  
  - MongoDB Atlas Vector Store  
  - Embeddings OpenAI  
  - Sticky Note  

- **Node Details:**

  - **When chat message received**  
    - *Type:* Chat Trigger (Langchain)  
    - *Role:* Entry point for chat messages to start the AI agent flow.  
    - *Configuration:* Default options, webhook ID assigned.  
    - *Input:* Incoming chat message via webhook.  
    - *Output:* Triggers AI agent node.  
    - *Edge Cases:* Missing or malformed chat payload.

  - **AI Traveling Planner Agent**  
    - *Type:* Agent Node (Langchain)  
    - *Role:* Orchestrates conversation logic, integrates memory and vector search tool.  
    - *Configuration:* Max 10 iterations, system message defines assistant role and vector search usage.  
    - *Input:* Chat message, memory, language model, vector search tool.  
    - *Output:* AI-generated chat response.  
    - *Edge Cases:* Exceeding iteration limits, tool failures, ambiguous queries.

  - **MongoDB Chat Memory**  
    - *Type:* MongoDB Chat Memory Node (Langchain)  
    - *Role:* Stores and retrieves conversation history in MongoDB for context continuity.  
    - *Configuration:* Database name `test`, uses MongoDB credentials.  
    - *Input:* Chat messages and AI responses.  
    - *Output:* Conversation memory context for agent.  
    - *Credentials:* MongoDB Atlas account.  
    - *Edge Cases:* Connection issues, data consistency.

  - **Google Gemini Chat Model**  
    - *Type:* Language Model Node (Langchain)  
    - *Role:* Generates conversational responses using Google Gemini LLM.  
    - *Configuration:* Model `models/gemini-2.0-flash`, Google PaLM API credentials.  
    - *Input:* Prompt and context from agent.  
    - *Output:* Text response.  
    - *Credentials:* Google Gemini API key.  
    - *Edge Cases:* API quota limits, network errors.

  - **MongoDB Atlas Vector Store**  
    - *Type:* MongoDB Atlas Vector Store (Langchain)  
    - *Role:* Vector search tool to retrieve relevant POIs based on query embeddings.  
    - *Configuration:* Mode `retrieve-as-tool`, top K=10 results, collection `points_of_interest`, vector index `vector_index`, metadata field `description`.  
    - *Input:* Query embeddings generated internally by agent.  
    - *Output:* Relevant POI data as tool results.  
    - *Credentials:* MongoDB Atlas account.  
    - *Edge Cases:* Index misconfiguration, no results found.

  - **Embeddings OpenAI**  
    - *Type:* OpenAI Embeddings Node (Langchain)  
    - *Role:* Provides embeddings for agent queries when vector search is invoked.  
    - *Configuration:* Uses OpenAI API credentials.  
    - *Input:* Text queries from agent.  
    - *Output:* Embeddings vectors.  
    - *Credentials:* OpenAI API key.  
    - *Edge Cases:* API failures, rate limits.

  - **Sticky Note**  
    - *Content:*  
      - Explains the overall AI traveling agent architecture.  
      - Lists setup steps for Google Gemini, OpenAI, MongoDB Atlas, and vector index creation.  
      - Provides links to MongoDB Atlas Vector Search and n8n docs.  
    - *Context:* High-level guidance and prerequisites.

---

### 3. Summary Table

| Node Name                   | Node Type                                  | Functional Role                                | Input Node(s)                | Output Node(s)                   | Sticky Note                                                                                              |
|-----------------------------|--------------------------------------------|------------------------------------------------|------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------|
| When chat message received  | @n8n/n8n-nodes-langchain.chatTrigger       | Entry point for chat messages                   |                              | AI Traveling Planner Agent       |                                                                                                        |
| AI Traveling Planner Agent  | @n8n/n8n-nodes-langchain.agent              | Orchestrates AI conversation with tools        | When chat message received, MongoDB Chat Memory, Google Gemini Chat Model, MongoDB Atlas Vector Store, Embeddings OpenAI |                              |                                                                                                        |
| MongoDB Chat Memory         | @n8n/n8n-nodes-langchain.memoryMongoDbChat | Stores and retrieves chat conversation memory  | When chat message received    | AI Traveling Planner Agent       |                                                                                                        |
| Google Gemini Chat Model    | @n8n/n8n-nodes-langchain.lmChatGoogleGemini| Generates AI chat responses                      | AI Traveling Planner Agent    | AI Traveling Planner Agent       |                                                                                                        |
| MongoDB Atlas Vector Store  | @n8n/n8n-nodes-langchain.vectorStoreMongoDBAtlas | Vector search tool for points of interest       | Embeddings OpenAI             | AI Traveling Planner Agent       |                                                                                                        |
| Embeddings OpenAI           | @n8n/n8n-nodes-langchain.embeddingsOpenAi  | Generates embeddings for queries                 | AI Traveling Planner Agent    | MongoDB Atlas Vector Store       |                                                                                                        |
| Sticky Note                 | n8n-nodes-base.stickyNote                   | Provides architecture and setup guidance         |                              |                                 | Explains AI agent architecture and setup steps with links to MongoDB Atlas Vector Search and n8n docs. |
| Webhook                    | n8n-nodes-base.webhook                       | Receives POST requests with POI data             |                              | Default Data Loader              | Provides cURL example for data ingestion.                                                              |
| Default Data Loader         | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Converts JSON to document format for embedding   | Webhook                      | Recursive Character Text Splitter |                                                                                                        |
| Recursive Character Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Splits text into chunks for embedding            | Default Data Loader           | Embeddings OpenAI1               |                                                                                                        |
| Embeddings OpenAI1          | @n8n/n8n-nodes-langchain.embeddingsOpenAi  | Generates embeddings for ingestion data          | Recursive Character Text Splitter | MongoDB Atlas Vector Store1      |                                                                                                        |
| MongoDB Atlas Vector Store1 | @n8n/n8n-nodes-langchain.vectorStoreMongoDBAtlas | Inserts embedded documents into vector store     | Webhook                      |                                 |                                                                                                        |
| Sticky Note1                | n8n-nodes-base.stickyNote                   | Provides cURL command example for ingestion      |                              |                                 | Shows example cURL command to POST data to ingestion webhook.                                         |
| Sticky Note2                | n8n-nodes-base.stickyNote                   | Explains ingestion flow and alternative methods |                              |                                 | Describes vector search data ingestion and alternatives.                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: `Webhook`  
   - Parameters:  
     - HTTP Method: POST  
     - Path: `ingestData`  
     - Options: Enable raw body parsing  
   - Purpose: Receive JSON payloads with points of interest data.

2. **Create Default Data Loader Node**  
   - Type: `Document Default Data Loader` (Langchain)  
   - Parameters:  
     - JSON Mode: Expression Data  
     - JSON Data Expression: `={{ $json.body.raw_body.point_of_interest.title }} - {{ $json.body.raw_body.point_of_interest.description }}`  
   - Connect input from Webhook node.

3. **Create Recursive Character Text Splitter Node**  
   - Type: `Recursive Character Text Splitter` (Langchain)  
   - Parameters: Use default options.  
   - Connect input from Default Data Loader node.

4. **Create Embeddings OpenAI Node (for ingestion)**  
   - Type: `Embeddings OpenAI` (Langchain)  
   - Credentials: Configure with valid OpenAI API key.  
   - Parameters: Default embedding model.  
   - Connect input from Recursive Character Text Splitter node.

5. **Create MongoDB Atlas Vector Store Node (Insert mode)**  
   - Type: `Vector Store MongoDB Atlas` (Langchain)  
   - Credentials: Configure with MongoDB Atlas credentials (connection string, access).  
   - Parameters:  
     - Mode: `insert`  
     - Collection: `points_of_interest`  
     - Vector Index Name: `vector_index`  
     - Metadata Field: `description`  
     - Embedding Batch Size: 1 (default)  
   - Connect input from Embeddings OpenAI node.

6. **Create Chat Trigger Node**  
   - Type: `Chat Trigger` (Langchain)  
   - Parameters: Default options.  
   - Purpose: Entry point for chat messages to start AI agent flow.

7. **Create MongoDB Chat Memory Node**  
   - Type: `Memory MongoDB Chat` (Langchain)  
   - Credentials: MongoDB Atlas credentials (same or separate).  
   - Parameters:  
     - Database Name: `test` (or your chosen DB)  
   - Connect input from Chat Trigger node.

8. **Create Google Gemini Chat Model Node**  
   - Type: `LM Chat Google Gemini` (Langchain)  
   - Credentials: Google PaLM API key configured.  
   - Parameters:  
     - Model Name: `models/gemini-2.0-flash`  
   - Connect input from AI agent node (see next step).

9. **Create Embeddings OpenAI Node (for agent queries)**  
   - Type: `Embeddings OpenAI` (Langchain)  
   - Credentials: OpenAI API key.  
   - Parameters: Default.  
   - Connect input to MongoDB Atlas Vector Store node (tool).

10. **Create MongoDB Atlas Vector Store Node (Retrieve mode)**  
    - Type: `Vector Store MongoDB Atlas` (Langchain)  
    - Credentials: MongoDB Atlas credentials.  
    - Parameters:  
      - Mode: `retrieve-as-tool`  
      - Top K: 10  
      - Collection: `points_of_interest`  
      - Vector Index Name: `vector_index`  
      - Metadata Field: `description`  
      - Tool Name: `PointofinterestKB`  
      - Tool Description: "The list of Points of Interest from the database."  
    - Connect input from Embeddings OpenAI node.

11. **Create AI Traveling Planner Agent Node**  
    - Type: `Agent` (Langchain)  
    - Parameters:  
      - Max Iterations: 10  
      - System Message: "You are a helpful assistant for a trip planner. You have a vector search capability to locate points of interest, Use it and don't invent much."  
    - Connect inputs:  
      - Main input from Chat Trigger node  
      - AI Memory from MongoDB Chat Memory node  
      - AI Language Model from Google Gemini Chat Model node  
      - AI Tool from MongoDB Atlas Vector Store node (retrieve mode)  
      - AI Embedding from Embeddings OpenAI node (for queries)  
    - Output: AI-generated chat response.

12. **Connect Chat Trigger node output to AI Traveling Planner Agent node input**  
    - This triggers the agent on chat messages.

13. **Add Sticky Notes (optional but recommended)**  
    - Add notes with cURL ingestion example on the webhook.  
    - Add notes explaining vector search ingestion and AI agent architecture with setup instructions and links.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| MongoDB Atlas Vector Search requires creating a vector index on the `points_of_interest` collection with the following configuration: `{ "fields": [ { "type": "vector", "path": "embedding", "numDimensions": 1536, "similarity": "cosine" } ] }`. Adjust `numDimensions` if using a different embedding model.                                                                                                                                                                                                                                                                                                                                                         | https://www.mongodb.com/docs/atlas/atlas-vector-search/tutorials/vector-search-quick-start/?utm=n8n.io                                      |
| For the Gemini LLM, set up Google PaLM API credentials and use the model `models/gemini-2.0-flash` for chat generation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | https://cloud.google.com/vertex-ai/docs/generative-ai/learn/vertex-ai-generative-ai-overview                                                   |
| OpenAI API key is required for generating embeddings both during ingestion and agent query time.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | https://platform.openai.com/docs/api-reference/embeddings                                                                                     |
| MongoDB Atlas cluster must have IP access configured (e.g., `0.0.0.0/0` for testing) and proper user credentials with read/write permissions on the target database and collections.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | https://www.mongodb.com/docs/atlas/tutorial/create-new-cluster/                                                                               |
| The agent uses a maximum of 10 iterations per query to avoid infinite loops or excessive API calls.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |                                                                                                                                            |
| The ingestion webhook expects JSON payloads with the structure: `{ "raw_body": { "point_of_interest": { "title": "Eiffel Tower", "description": "Iconic iron lattice tower located on the Champ de Mars in Paris, France." } } }`.                                                                                                                                                                                                                                                                                                                                                                                                                                           | See Sticky Note1 in workflow                                                                                                                 |
| The workflow demonstrates how to combine memory, vector search, and LLM orchestration natively in n8n without custom code, simplifying agentic AI development.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |                                                                                                                                            |

---

This document fully describes the "Travel Planning Assistant with MongoDB Atlas, Gemini LLM and Vector Search" workflow, enabling reproduction, modification, and troubleshooting by advanced users and automation agents alike.