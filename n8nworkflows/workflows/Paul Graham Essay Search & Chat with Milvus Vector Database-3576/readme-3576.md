Paul Graham Essay Search & Chat with Milvus Vector Database

https://n8nworkflows.xyz/workflows/paul-graham-essay-search---chat-with-milvus-vector-database-3576


# Paul Graham Essay Search & Chat with Milvus Vector Database

### 1. Workflow Overview

This workflow implements a Retrieval-Augmented Generation (RAG) system focused on Paul Graham’s essays, leveraging the Milvus vector database for semantic search and AI chat capabilities. It is designed for users who want to ingest a corpus of essays, embed them into a vector store, and then interactively query and discuss the content via an AI chat interface.

The workflow is logically divided into two main blocks:

- **1.1 Scrape & Load**: Automatically scrapes the list of Paul Graham essays from his website, fetches the full text of a limited number of essays, extracts and processes the text, generates vector embeddings using OpenAI, and inserts these embeddings into a Milvus vector database collection.

- **1.2 Chat Interface**: Listens for incoming chat messages, uses the Milvus vector store as a retrieval tool to semantically search the essay embeddings, and generates AI-powered conversational responses based on the retrieved content.

---

### 2. Block-by-Block Analysis

#### 2.1 Scrape & Load

**Overview:**  
This block scrapes Paul Graham’s essays from his website, extracts essay URLs, fetches the essay contents, processes the text, generates embeddings, and loads them into a Milvus vector database collection.

**Nodes Involved:**  
- When clicking "Execute Workflow" (Manual Trigger)  
- Fetch Essay List (HTTP Request)  
- Extract essay names (HTML Extract)  
- Split out into items (Split Out)  
- Limit to first 3 (Limit)  
- Fetch essay texts (HTTP Request)  
- Extract Text Only (HTML Extract)  
- Recursive Character Text Splitter1 (Text Splitter)  
- Default Data Loader (Document Loader)  
- Embeddings OpenAI (Embeddings)  
- Milvus Vector Store (Vector Store Insert)  
- Sticky Note3 (Note)  
- Sticky Note5 (Note)  
- Sticky Note (Setup Instructions Note)

**Node Details:**

- **When clicking "Execute Workflow"**  
  - Type: Manual Trigger  
  - Role: Entry point to start the scraping and loading process manually.  
  - Inputs: None  
  - Outputs: Triggers "Fetch Essay List"  
  - Edge cases: None expected; user must manually trigger.

- **Fetch Essay List**  
  - Type: HTTP Request  
  - Role: Downloads the HTML page listing all Paul Graham essays from http://www.paulgraham.com/articles.html  
  - Configuration: Simple GET request without special headers or authentication.  
  - Inputs: Trigger from manual node  
  - Outputs: HTML content to "Extract essay names"  
  - Edge cases: Network errors, site downtime, or HTML structure changes.

- **Extract essay names**  
  - Type: HTML Extract  
  - Role: Parses the fetched HTML to extract all essay URLs from nested tables using CSS selector `table table a` and extracts the `href` attribute.  
  - Configuration: Returns an array of URLs relative to the base site.  
  - Inputs: HTML content from "Fetch Essay List"  
  - Outputs: Array of essay URLs to "Split out into items"  
  - Edge cases: Changes in page structure may cause extraction failure.

- **Split out into items**  
  - Type: Split Out  
  - Role: Splits the array of essay URLs into individual items for sequential processing.  
  - Inputs: Array of URLs from "Extract essay names"  
  - Outputs: Single essay URL per item to "Limit to first 3"  
  - Edge cases: Empty array input.

- **Limit to first 3**  
  - Type: Limit  
  - Role: Limits processing to the first 3 essays only, to reduce load and runtime.  
  - Inputs: Individual essay URLs from "Split out into items"  
  - Outputs: Limited essay URLs to "Fetch essay texts"  
  - Edge cases: Less than 3 essays available.

- **Fetch essay texts**  
  - Type: HTTP Request  
  - Role: Fetches the full HTML content of each essay by constructing URL `http://www.paulgraham.com/{{ $json.essay }}` where `{{ $json.essay }}` is the essay path.  
  - Inputs: Essay URL from "Limit to first 3"  
  - Outputs: HTML content of essay to "Extract Text Only"  
  - Edge cases: Network errors, 404 if essay URL invalid.

- **Extract Text Only**  
  - Type: HTML Extract  
  - Role: Extracts the main textual content from the essay HTML using CSS selector `body`, skipping `img` and `nav` elements to avoid non-text content.  
  - Inputs: HTML from "Fetch essay texts"  
  - Outputs: Clean essay text to "Milvus Vector Store" and "Recursive Character Text Splitter1"  
  - Edge cases: HTML structure changes, extraction failure.

- **Recursive Character Text Splitter1**  
  - Type: Text Splitter (LangChain)  
  - Role: Splits large essay text into chunks of max 6000 characters for embedding processing.  
  - Inputs: Essay text from "Default Data Loader" (connected downstream)  
  - Outputs: Text chunks to "Default Data Loader"  
  - Edge cases: Very large essays, splitting logic errors.

- **Default Data Loader**  
  - Type: Document Loader (LangChain)  
  - Role: Loads the text chunks into document format suitable for embedding and vector store insertion.  
  - Inputs: Text chunks from "Recursive Character Text Splitter1"  
  - Outputs: Documents to "Milvus Vector Store"  
  - Edge cases: Invalid text data.

- **Embeddings OpenAI**  
  - Type: Embeddings (OpenAI)  
  - Role: Generates vector embeddings for the essay text chunks using OpenAI embeddings API.  
  - Inputs: Documents from "Default Data Loader"  
  - Outputs: Embeddings to "Milvus Vector Store"  
  - Credentials: Requires OpenAI API credentials  
  - Edge cases: API rate limits, authentication errors.

- **Milvus Vector Store**  
  - Type: Vector Store (Milvus)  
  - Role: Inserts the generated embeddings into the Milvus collection named `n8n_test`. The collection is cleared before insertion (`clearCollection: true`).  
  - Inputs: Embeddings from "Embeddings OpenAI" and documents from "Default Data Loader"  
  - Outputs: None (terminal node for this flow)  
  - Credentials: Requires Milvus API credentials  
  - Edge cases: Connection errors, collection not found, authentication failures.

- **Sticky Notes**  
  - Provide visual documentation and instructions within the workflow canvas for user guidance.

---

#### 2.2 Chat Interface

**Overview:**  
This block enables users to interact with the AI agent via chat. Incoming chat messages trigger semantic search queries against the Milvus vector store, and the AI agent generates responses based on retrieved essay content.

**Nodes Involved:**  
- When chat message received (Chat Trigger)  
- AI Agent (LangChain Agent)  
- Milvus Vector Store as tool (Vector Store Retrieval Tool)  
- OpenAI Chat Model (Chat Language Model)  
- Embeddings OpenAI1 (Embeddings for retrieval)  
- Sticky Note1 (Note)  
- Sticky Note2 (Note)

**Node Details:**

- **When chat message received**  
  - Type: Chat Trigger (LangChain)  
  - Role: Listens for incoming chat messages via webhook and triggers the AI Agent.  
  - Inputs: External chat messages  
  - Outputs: Message data to "AI Agent"  
  - Edge cases: Webhook connectivity, malformed messages.

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Central AI orchestrator that uses the Milvus vector store as a retrieval tool and OpenAI chat model to generate conversational responses.  
  - Inputs: Chat messages from "When chat message received"  
  - Outputs: AI-generated chat responses  
  - Configuration: Uses "Milvus Vector Store as tool" as a retrieval tool and "OpenAI Chat Model" as the language model.  
  - Edge cases: API errors, retrieval failures, timeout.

- **Milvus Vector Store as tool**  
  - Type: Vector Store Retrieval Tool (Milvus)  
  - Role: Provides semantic search capabilities to the AI Agent by querying the `n8n_test` Milvus collection.  
  - Inputs: Queries from "AI Agent"  
  - Outputs: Retrieved documents to "AI Agent"  
  - Credentials: Milvus API credentials required  
  - Edge cases: Connection issues, empty results.

- **OpenAI Chat Model**  
  - Type: Chat Language Model (OpenAI)  
  - Role: Generates natural language responses based on retrieved documents and user queries.  
  - Inputs: Query and context from "AI Agent"  
  - Outputs: Chat responses to "AI Agent"  
  - Credentials: OpenAI API credentials required  
  - Edge cases: Rate limits, authentication errors.

- **Embeddings OpenAI1**  
  - Type: Embeddings (OpenAI)  
  - Role: Used internally by "Milvus Vector Store as tool" for embedding queries during retrieval.  
  - Inputs/Outputs: Connected to "Milvus Vector Store as tool"  
  - Credentials: OpenAI API credentials  
  - Edge cases: Same as other OpenAI embedding nodes.

- **Sticky Notes**  
  - Provide user guidance on starting chat and workflow steps.

---

### 3. Summary Table

| Node Name                      | Node Type                               | Functional Role                         | Input Node(s)                    | Output Node(s)                 | Sticky Note                                                                                   |
|-------------------------------|---------------------------------------|---------------------------------------|---------------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger                        | Starts scraping and loading process   | None                            | Fetch Essay List               | ## Step 1 1. Set up a Milvus server based on [this guide](https://milvus.io/docs/install_standalone-docker-compose.md). And then create a collection named `n8n_test`. 2. Click this workflow to load scrape and load Paul Graham essays to Milvus collection. |
| Fetch Essay List               | HTTP Request                          | Fetches essays list HTML page          | When clicking "Execute Workflow" | Extract essay names            |                                                                                              |
| Extract essay names           | HTML Extract                         | Extracts essay URLs from HTML          | Fetch Essay List                | Split out into items           |                                                                                              |
| Split out into items          | Split Out                           | Splits essay URLs into individual items | Extract essay names             | Limit to first 3               |                                                                                              |
| Limit to first 3              | Limit                              | Limits to first 3 essays                | Split out into items            | Fetch essay texts              |                                                                                              |
| Fetch essay texts             | HTTP Request                       | Fetches full essay HTML content         | Limit to first 3                | Extract Text Only              |                                                                                              |
| Extract Text Only             | HTML Extract                      | Extracts main essay text content        | Fetch essay texts               | Milvus Vector Store, Recursive Character Text Splitter1 |                                                                                              |
| Recursive Character Text Splitter1 | Text Splitter (LangChain)          | Splits text into chunks for embedding   | Default Data Loader             | Default Data Loader            |                                                                                              |
| Default Data Loader           | Document Loader (LangChain)        | Loads text chunks as documents           | Recursive Character Text Splitter1 | Milvus Vector Store           |                                                                                              |
| Embeddings OpenAI             | Embeddings (OpenAI)                | Generates embeddings for essay chunks   | Default Data Loader             | Milvus Vector Store            |                                                                                              |
| Milvus Vector Store           | Vector Store Insert (Milvus)       | Inserts embeddings into Milvus collection | Embeddings OpenAI, Default Data Loader | None                        | ## Load into Milvus vector database                                                         |
| When chat message received    | Chat Trigger (LangChain)           | Receives chat messages                   | External webhook               | AI Agent                      | ## Step 2 Start to chat with the AI Agent with Milvus tool                                  |
| AI Agent                     | LangChain Agent                   | Orchestrates retrieval and chat response | When chat message received, Milvus Vector Store as tool, OpenAI Chat Model | None                        |                                                                                              |
| Milvus Vector Store as tool   | Vector Store Retrieval Tool (Milvus) | Provides semantic search tool for AI Agent | Embeddings OpenAI1             | AI Agent                      |                                                                                              |
| OpenAI Chat Model             | Chat Language Model (OpenAI)       | Generates AI chat responses              | AI Agent                      | AI Agent                      |                                                                                              |
| Embeddings OpenAI1            | Embeddings (OpenAI)                | Embeddings for retrieval queries         | Milvus Vector Store as tool    | Milvus Vector Store as tool   |                                                                                              |
| Sticky Note3                 | Sticky Note                       | Visual note: Scrape latest Paul Graham essays | None                        | None                         |                                                                                              |
| Sticky Note5                 | Sticky Note                       | Visual note: Load into Milvus vector database | None                        | None                         |                                                                                              |
| Sticky Note                  | Sticky Note                       | Visual note: Setup instructions          | None                        | None                         | ## Step 1 1. Set up a Milvus server based on [this guide](https://milvus.io/docs/install_standalone-docker-compose.md). And then create a collection named `n8n_test`. 2. Click this workflow to load scrape and load Paul Graham essays to Milvus collection. |
| Sticky Note1                 | Sticky Note                       | Visual note: Start chat with AI Agent    | None                        | None                         | ## Step 2 Start to chat with the AI Agent with Milvus tool                                  |
| Sticky Note2                 | Sticky Note                       | Empty visual note                         | None                        | None                         |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start the scraping and loading process manually.

2. **Add HTTP Request Node "Fetch Essay List"**  
   - URL: `http://www.paulgraham.com/articles.html`  
   - Method: GET  
   - Connect output of Manual Trigger to this node.

3. **Add HTML Extract Node "Extract essay names"**  
   - Operation: Extract HTML content  
   - Extraction Values:  
     - Key: `essay`  
     - CSS Selector: `table table a`  
     - Attribute: `href`  
     - Return as Array: true  
   - Connect output of "Fetch Essay List" to this node.

4. **Add Split Out Node "Split out into items"**  
   - Field to Split Out: `essay`  
   - Connect output of "Extract essay names" to this node.

5. **Add Limit Node "Limit to first 3"**  
   - Max Items: 3  
   - Connect output of "Split out into items" to this node.

6. **Add HTTP Request Node "Fetch essay texts"**  
   - URL: Expression: `http://www.paulgraham.com/{{ $json.essay }}`  
   - Method: GET  
   - Connect output of "Limit to first 3" to this node.

7. **Add HTML Extract Node "Extract Text Only"**  
   - Operation: Extract HTML content  
   - Extraction Values:  
     - Key: `data`  
     - CSS Selector: `body`  
     - Skip Selectors: `img,nav`  
   - Connect output of "Fetch essay texts" to this node.

8. **Add Recursive Character Text Splitter Node "Recursive Character Text Splitter1"**  
   - Chunk Size: 6000 characters  
   - Connect output of "Default Data Loader" (to be created next) to this node’s input.

9. **Add Document Loader Node "Default Data Loader"**  
   - JSON Mode: Expression Data  
   - JSON Data: Expression referencing `Extract Text Only` node’s `data` field: `={{ $('Extract Text Only').item.json.data }}`  
   - Connect output of "Recursive Character Text Splitter1" to this node.

10. **Add Embeddings Node "Embeddings OpenAI"**  
    - Provider: OpenAI  
    - Credentials: Configure OpenAI API credentials  
    - Connect output of "Default Data Loader" to this node.

11. **Add Milvus Vector Store Node "Milvus Vector Store"**  
    - Mode: Insert  
    - Options: Clear collection before insert (true)  
    - Collection: Select or enter collection name `n8n_test`  
    - Credentials: Configure Milvus API credentials  
    - Connect outputs of "Embeddings OpenAI" and "Default Data Loader" to this node.

12. **Create Chat Trigger Node "When chat message received"**  
    - Type: LangChain Chat Trigger  
    - Configure webhook as needed for receiving chat messages.

13. **Add LangChain Agent Node "AI Agent"**  
    - Connect input from "When chat message received"  
    - Configure to use:  
      - Language Model: "OpenAI Chat Model" (to be created)  
      - Tool: "Milvus Vector Store as tool" (to be created)

14. **Add Vector Store Retrieval Tool Node "Milvus Vector Store as tool"**  
    - Mode: Retrieve as tool  
    - Tool Name: `milvus_knowledge_base`  
    - Tool Description: "useful when you need to retrieve information"  
    - Collection: `n8n_test`  
    - Credentials: Milvus API credentials  
    - Connect output to "AI Agent" as tool input.

15. **Add OpenAI Chat Model Node "OpenAI Chat Model"**  
    - Model: `gpt-4o-mini` (or preferred OpenAI chat model)  
    - Credentials: OpenAI API credentials  
    - Connect output to "AI Agent" as language model input.

16. **Add Embeddings Node "Embeddings OpenAI1"**  
    - Provider: OpenAI  
    - Credentials: OpenAI API credentials  
    - Connect output to "Milvus Vector Store as tool" as embedding input.

17. **Connect "AI Agent" output to wherever chat response should be sent (e.g., webhook response).**

18. **Add Sticky Notes** (optional) for user guidance and documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Set up a Milvus server following the official installation guide: https://milvus.io/docs/install_standalone-docker.md  | Milvus server installation prerequisite for this workflow                                            |
| Collection name used in this workflow: `n8n_test`                                                                     | Must be created in Milvus before running the workflow                                                 |
| Paul Graham essays source: http://www.paulgraham.com/articles.html                                                     | Source website for scraping essays                                                                    |
| Workflow demonstrates a RAG system combining web scraping, vector embedding, vector database, and AI chat              | Useful reference for building similar semantic search and chat applications                           |
| OpenAI API credentials required for embedding generation and chat model                                                | Ensure valid API keys with sufficient quota                                                           |
| Milvus API credentials required for vector store operations                                                            | Ensure connectivity and authentication with Milvus server                                            |

---

This documentation provides a detailed, stepwise understanding of the workflow structure, node configurations, and operational logic, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.